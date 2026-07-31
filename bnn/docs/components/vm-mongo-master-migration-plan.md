# vm-mongo-master migration plan (Azure → AWS)

Scope: `vm-mongo-master` only. `vm-core-database` (including its `edu_data`) is deferred to a separate later effort — see [mongodb.md](mongodb.md).

**Decision**: the AWS target keeps the **same MongoDB version (5.0.26)** as the source for this first migration. No version upgrade happens as part of this migration — that's deferred to a later, separate rolling-upgrade effort once the instance is settled on AWS. This removes version/driver-compatibility risk from the scope of this plan entirely: every consumer, regardless of its driver version, keeps talking to the same server version it does today.

Downtime budget: **under 30 minutes**. Approach: **Option B** — convert standalone → single-node replica set → add an AWS-hosted member → background initial sync → controlled failover. See [mongodb.md](mongodb.md) for the full discovery record (size, consumers, security findings) this plan is based on.

## Action plan

1. **Update connection strings in all workloads that connect to `vm-mongo-master`.** Confirmed consumers, re-verified 2026-07-31 (see [mongodb.md](mongodb.md), "Confirmed consumers" section):
   - **Live (`monkey-eks`), 37 workloads:**
     - `app` (14): `data-report`, `data-segment`, `edu-ai`, `edu-app-queue` (⚠️ currently 0/3 ready — down for unrelated reasons, still needs its connection string updated), `edu-app-v2`, `edu-app-v2-queue`, `edu-award`, `edu-cms`, `edu-cms-queue`, `edu-device-queue`, `edu-lesson-queue`, `edu-product`, `edu-share-photo`, `edu-story`
     - `crm` (14): `edu-auth`, `edu-auth-queue`, `edu-campaign`, `edu-campaign-queue`, `edu-crm-accountant`, `edu-crm-queue`, `edu-developer`, `edu-developer-queue`, `edu-mailsms-queue`, `edu-media`, `edu-media-queue`, `edu-okr`, `edu-ticket-queue`, `service-agent-queue`
     - `class` (7): `mk-classroom`, `mk-classroom-go`, `mk-classroom-go-cron`, `mk-classroom-go-queue`, `mk-classroom-queue-live`, `mk-course-go`, `mk-course-go-queue`
     - `hoc10` (2): `edu-lms`, `edu-question-service`
     - No StatefulSets involved anywhere — confirmed all-Deployments.
     - Roughly 24 distinct images/repos underneath these (PHP fleet on `php-base:81`, three independent Go services, `data_report`/`data_report_segment`).
   - **VMs (Azure), network-level only — not yet traced to a specific process**: `vm-ai-machine-studio` (`20.6.34.63`), `vm-core-service` (`20.195.10.11`).
   - **Nonprod (`onepercent-aks-v2`), 2 workloads:** `app/data-report-dev`, `stg/edu-story-stg`.
   - Each has its connection string baked into a `.env`/config file at image-build time, not injected via k8s Secret/ConfigMap — this requires an image rebuild + redeploy per service, not a config change.
   - Target state for the connection string: replica-set-aware (add `replicaSet=atlas-um8iyc-shard-0` — see the actual-name correction in the execution log below), so failover later is an automatic driver reconnect, not a manual app redeploy.
   - This is the long pole of the whole migration — budget real time for it, not a same-day task. Given the volume, prioritizing 3-5 services with real user-impact logic first; the rest (mostly audit-log writers) follow after. Owner is gathering usage data to inform exactly which 3-5.
2. Provision the AWS EC2 node: MongoDB **5.0.26** (matching source, per the version decision above), storage sized well above 831GB with headroom for growth.
3. Set up a restricted, temporary network path (VPN/private peering) between the Azure VM and the new EC2 node for the sync period — do not reuse the existing `0.0.0.0/0` exposure.
4. Measure real write throughput over a proper multi-hour window spanning peak usage (not a 30s snapshot) to size the oplog correctly; size generously above that measurement.
5. Convert `vm-mongo-master` standalone → single-node replica set (`rs.initiate()`), with the member host **explicitly set to the existing public FQDN** (`mongo.southeastasia.cloudapp.azure.com`) — do not let it default to the machine's internal hostname, which would break existing connections immediately.
6. Add the AWS node as a replica set member (`rs.add()`); monitor initial sync progress and oplog lag until caught up. Production keeps serving traffic throughout — zero downtime for this phase.
7. Cutover: controlled failover (`rs.stepDown()` on the Azure primary) once lag is ~0. This is the actual downtime — target seconds to low minutes, contingent on step 1 being complete.
8. Keep the Azure node as a rollback replica for a window (hours to a day) before decommissioning.
9. Tear down the temporary cross-cloud network path once the Azure node is retired.
10. Rotate the shared `admin`/weak credential (confirmed reused across at least `data-report` and `edu-story-stg`) as part of, or shortly after, this migration — independent of the version-upgrade question.

## Execution log (2026-07-31)

Real progress against this plan, with several findings the original plan didn't anticipate:

- **`vm-mongo-master` converted to a single-node replica set.** Hit a `security.keyFile is required when authorization is enabled with replica sets` error on first restart — MongoDB requires this whenever both auth and replication are configured, not previously anticipated in this plan. Keyfile generated and added on both sides (checksum-verified identical).
- **The replica set's actual name is `atlas-um8iyc-shard-0`, not the originally planned `rs-mongo-prod`.** `rs.initiate()` failed with "already initialized" — the VM's `local` database already had a stale, orphaned replica set config, and cross-referencing it (`atlas-um8iyc-shard-0`, members named `atlas-um8iyc-shard-00-0X.awttv.mongodb.net`, GCP) shows this data almost certainly originated from **a prior MongoDB Atlas → self-hosted migration**, and the `local` database was copied along with the data at the time (a well-known pitfall — `local` should never be copied between deployments). Client-level deletes on `local.system.replset` are blocked even for `root`; the documented fix is `rs.reconfig(..., {force: true})`, which also can't change the set's `_id` (name) — so rather than fight it, we adopted the existing name. **All future references to this replica set (connection strings, the AWS node's config) must use `atlas-um8iyc-shard-0`.**
- **AWS node had to be recreated in the public subnet.** Original plan assumed a private-subnet instance + Elastic IP would be reachable from Azure — it isn't: a private subnet's route table has no Internet Gateway route, so an EIP there doesn't accept inbound connections regardless of security group rules. Recreated in `prod-public-1a` (`iac` repo's `envs/prod/main.tf` has a `subnet_id` override with a comment marking this temporary — **must move back to the private subnet after cutover**, tracked in the cleanup checklist below).
- **AWS hairpin-NAT limitation**: the AWS node couldn't complete MongoDB's `isSelf` check against its own public IP (EC2 instances generally can't reach their own public IP from inside the VPC — the public IP isn't bound to any local interface). Fixed by using the instance's public DNS hostname as its replica-set member address, with a local `/etc/hosts` override on the AWS instance pointing that same hostname to its private IP — so the instance resolves itself locally while Azure resolves the same name via real DNS to the public IP.
- **Temporary network exposure**: AWS mongo node has a public IP + a security-group rule scoped to exactly `vm-mongo-master`'s IP (`20.198.255.32/32`) on port 27017 — both tagged `Purpose: temp-migration-sync` for easy identification/cleanup. **No TLS in transit** for this sync — accepted as a known residual risk given the timeline, not fixed.
- **Result**: as of this log entry, the AWS node is in `STARTUP2` (initial sync actively running) — first real background-sync progress. Expect several hours given the ~831GB dataset and the throughput math in the "CPU usage investigation" section of `mongodb.md`.

### Cleanup checklist (do after cutover, not before)

1. Move the AWS mongo node back to the private subnet (`prod-private-1a`) — remove the temporary `subnet_id` override in `iac`'s `envs/prod/main.tf`.
2. Release/remove the temporary public-facing exposure: the security-group rule scoped to `vm-mongo-master`'s IP, and the `/etc/hosts` self-reference workaround (no longer needed once there's no public IP).
3. Tear down the Azure side once confident: `vm-mongo-master` itself, its NSG rule set.
4. Consider whether to rename the replica set from `atlas-um8iyc-shard-0` to something reflecting current reality — cosmetic, not urgent.
5. Set up TLS in transit if this pattern (cross-cloud replication) recurs for the deferred `vm-core-database` migration later.

## Connection-string rollout status (AWS Parameter Store, checked 2026-07-31)

Confirmed via the user: the actual source of truth for each service's Mongo connection string is **AWS Parameter Store**, keys prefixed `env-eks-<service-name>` (each service has a base + a `-2` variant — believed to be two environment slots, not yet confirmed which). CI reads these at build time and bakes the value into the image's `.env`/config file (matching what was found earlier by inspecting running containers) — so updating the Parameter Store value and triggering a CI rebuild + CD deploy is the actual mechanism, not editing files in each git repo.

Scanned all 78 `env-eks-*` parameters for any `mongodb://...` value referencing `vm-mongo-master`. **Key format is inconsistent across services** — no single key name is used everywhere (`URI=`, `DB_MONGO_URI=`, `MONGODB_URI=`, `MONGO_URI=`, `MONGODB_SRV=`, YAML `url:`) — so any future scan needs to search for the `mongodb://` substring itself, not a specific key name, or it will silently undercount.

Target value once updated: `mongodb://admin:Admin%401234@mongo.southeastasia.cloudapp.azure.com:27017,ec2-54-179-59-251.ap-southeast-1.compute.amazonaws.com:27017/?replicaSet=atlas-um8iyc-shard-0` (both replica set members listed, explicit `replicaSet=`).

**Re-checked 2026-07-31 (second pass), current state: 26 correct / 37 still old / 15 no-mongo-reference.**

**✅ Correct (26) — already updated and confirmed matching:**
`env-eks-edu-log`, `env-eks-edu-log-2`, `env-eks-edu-product`, `env-eks-edu-product-2`, `env-eks-edu-app`, `env-eks-edu-app-2`, `env-eks-edu-auth`, `env-eks-edu-auth-2`, `env-eks-edu-media-2`, `env-eks-edu-ticket`, `env-eks-edu-ticket-2`, `env-eks-edu-platform`, `env-eks-edu-platform-2`, `env-eks-edu-app-v2`, `env-eks-edu-app-v2-2`, `env-eks-edu-crm`, `env-eks-edu-crm-2`, `env-eks-edu-device-2`, `env-eks-service-agent`, `env-eks-service-agent-2`, `env-eks-edu-app-platform-go`, `env-eks-edu-app-platform-go-2`, `env-eks-edu-mail`, `env-eks-edu-mail-2`, `env-eks-edu-cms`, `env-eks-edu-cms-2`

**❌ Still old (37) — need the same update:**
`env-eks-edu-app-story-go`, `env-eks-edu-app-story-go-2`, `env-eks-edu-lms`, `env-eks-edu-lms-2`, `env-eks-edu-lesson`, `env-eks-edu-lesson-2`, `env-eks-edu-okr`, `env-eks-edu-okr-2`, `env-eks-edu-story`, `env-eks-edu-story-2`, `env-eks-data-report`, `env-eks-data-report-2`, `env-eks-data-segment`, `env-eks-data-segment-2`, `env-eks-edu-crm-accountant`, `env-eks-edu-crm-accountant-2`, `env-eks-question-service`, `env-eks-question-service-2`, `env-eks-edu-campaign`, `env-eks-edu-campaign-2`, `env-eks-edu-share-photo`, `env-eks-edu-share-photo-2`, `env-eks-mk-course-go`, `env-eks-mk-course-go-2`, `env-eks-mk-course-go-test`, `env-eks-edu-mspeak`, `env-eks-edu-developer`, `env-eks-edu-developer-2`, `env-eks-mk-classroom`, `env-eks-mk-classroom-2`, `env-eks-mk-classroom-go`, `env-eks-mk-classroom-go-2`, `env-eks-mk-classroom-go-test`, `env-eks-edu-device`, `env-eks-edu-award`, `env-eks-edu-award-2`, `env-eks-edu-media`

Note: `data-report`/`data-report-2`/`data-segment`/`data-segment-2` also carry a second key, `URI_DATA=mongodb://tuananh:...@20.212.252.28:27017/admin` (the deferred `vm-core-database` instance) — unaffected by this migration, do not touch when updating the `URI=` key.

**⚠️ Anomalies worth checking before touching**: `env-eks-mk-classroom-go-test` and `env-eks-mk-course-go-test` have a completely different flat, comma-joined structure unlike every other parameter — confirm these are actually live/deployed configs and not stale leftovers before including them in the rollout.

**Base name vs `-2` — which is actually live, confirmed by cross-checking running pods (2026-07-31)**: for `edu-auth` and `edu-ticket` (namespace `crm`), the k8s deployment named without any suffix was found still running the *old* string via live inspection (`kubectl exec ... cat .env`) at a point when Parameter Store's *base-named* parameter (`env-eks-edu-auth`, `env-eks-edu-ticket`) already showed the correct value — meaning those deployments were actually reading from the `-2`-suffixed parameter, not the base one. Updating the `-2` variants resolved it (confirmed via a second Parameter Store check showing `env-eks-edu-auth-2`/`env-eks-edu-ticket-2`/`env-eks-edu-platform-2` flipping to correct). **Do not assume the base name is the "primary" one** — update both base and `-2` for every service; do not treat either as safe to skip.

**No `mongodb://` reference found (15)** — either legitimately don't use Mongo, or use a key pattern this substring search still missed (worth a second look if any of these are later found to actually connect): `env-eks-edu-dialogue`, `env-eks-edu-dialogue-2`, `env-eks-edu-user-reader`, `env-eks-edu-user-reader-2`, `env-eks-edu-user-writer`, `env-eks-edu-user-writer-2`, `env-eks-mk-customer-web`, `env-eks-mk-customer-web-test`, `env-eks-edu-video-writer`, `env-eks-edu-video-reader`, `env-eks-edu-tutoring-phhs`, `env-eks-edu-tutoring-phhs-test`, `/prod/livescore/env-eks-studio-live-score` (confirmed unrelated — Postgres-backed, not Mongo).

## Open items / not yet resolved

- Confirm whether the `cal_report/*.py` standalone scripts (in `edu_learn_report`, not imported by the main FastAPI app) are still actively triggered (cron/Lambda) and need their connection string updated too, or are dead.
- Confirm what's connecting from `113.190.232.224` (flagged as a dev proxy, not urgent now) — later needs to sit behind a bastion SSH tunnel rather than direct access, independent of this migration.
- Full driver-compatibility sweep across the ~24 distinct images is only partially done (PHP fleet confirmed modern/compatible; Go services' individual versions noted in mongodb.md) — moot for *this* migration since the server version isn't changing, but relevant again once the later version-upgrade effort starts.
