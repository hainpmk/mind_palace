# vm-mongo-master migration postmortem (Azure → AWS)

Status: **migration complete, cutover succeeded 2026-08-03.** This is now a historical log — what was planned, what actually happened, what blocked us, and what we learned — not a living action plan. For ongoing Terraform/CI context for this infra, see [iac-terraform-notes.md](iac-terraform-notes.md). For the discovered service-dependency inventory, see `docs/components-raw/mongodb-service-dependencies-2026-08-04.json`.

Scope: `vm-mongo-master` only. `vm-core-database` (including its `edu_data`) is deferred to a separate later effort — see [mongodb.md](mongodb.md).

## Original plan

**Decision**: the AWS target kept the **same MongoDB version (5.0.26)** as the source for this first migration. No version upgrade happened as part of this migration — deferred to a later, separate rolling-upgrade effort once the instance is settled on AWS. This removed version/driver-compatibility risk from scope entirely: every consumer, regardless of its driver version, kept talking to the same server version.

Downtime budget: **under 30 minutes**. Approach: **Option B** — convert standalone → single-node replica set → add an AWS-hosted member → background initial sync → controlled failover. See [mongodb.md](mongodb.md) for the full discovery record (size, consumers, security findings) this plan was based on.

Planned steps:
1. Update connection strings in all workloads that connect to `vm-mongo-master` (37 k8s workloads across `monkey-eks`, plus 2 nonprod, plus unresolved VM-level connections — see the service-map JSON for the full inventory).
2. Provision the AWS EC2 node: MongoDB 5.0.26, storage sized well above 831GB.
3. Set up a restricted, temporary network path (VPN/private peering) between the Azure VM and the new EC2 node for the sync period.
4. Measure real write throughput over a multi-hour window to size the oplog correctly.
5. Convert `vm-mongo-master` standalone → single-node replica set.
6. Add the AWS node as a replica set member; monitor initial sync + oplog lag.
7. Cutover: controlled failover (`rs.stepDown()`).
8. Keep the Azure node as a rollback replica for a window before decommissioning.
9. Tear down the temporary cross-cloud network path.
10. Rotate the shared `admin`/weak credential.

## Execution log (2026-07-31)

- **`vm-mongo-master` converted to a single-node replica set.** Hit a `security.keyFile is required when authorization is enabled with replica sets` error on first restart — MongoDB requires this whenever both auth and replication are configured, not previously anticipated. Keyfile generated and added on both sides (checksum-verified identical).
- **The replica set's actual name is `atlas-um8iyc-shard-0`, not the originally planned `rs-mongo-prod`.** `rs.initiate()` failed with "already initialized" — the VM's `local` database already had a stale, orphaned replica set config, and cross-referencing it (`atlas-um8iyc-shard-0`, members named `atlas-um8iyc-shard-00-0X.awttv.mongodb.net`, GCP) shows this data almost certainly originated from **a prior MongoDB Atlas → self-hosted migration**, and the `local` database was copied along with the data at the time (a well-known pitfall — `local` should never be copied between deployments). Client-level deletes on `local.system.replset` are blocked even for `root`; the documented fix is `rs.reconfig(..., {force: true})`, which also can't change the set's `_id` (name) — so rather than fight it, we adopted the existing name. **All future references to this replica set must use `atlas-um8iyc-shard-0`.**
- **AWS node had to be recreated in the public subnet.** Original plan assumed a private-subnet instance + Elastic IP would be reachable from Azure — it isn't: a private subnet's route table has no Internet Gateway route, so an EIP there doesn't accept inbound connections regardless of security group rules. Recreated in `prod-public-1a` (temporary — moved back post-cutover, see below).
- **AWS hairpin-NAT limitation**: the AWS node couldn't complete MongoDB's `isSelf` check against its own public IP (EC2 instances generally can't reach their own public IP from inside the VPC). Fixed by using the instance's public DNS hostname as its replica-set member address, with a local `/etc/hosts` override on the AWS instance pointing that same hostname to its private IP — so the instance resolves itself locally while Azure resolves the same name via real DNS to the public IP. This same hairpin-NAT class of issue recurred twice more later, on `monkey-product` and (checked but not needed) `staging-monkey` — see below.
- **Temporary network exposure**: AWS mongo node had a public IP + a security-group rule scoped to exactly `vm-mongo-master`'s IP (`20.198.255.32/32`) on port 27017. **No TLS in transit** for this sync — accepted as a known residual risk given the timeline, not fixed.
- **Result**: AWS node entered `STARTUP2` (initial sync actively running) — first real background-sync progress.

## Update (2026-08-01): second full restart, same failure signature — likely systematic, not random

Attempt 2 (started 2026-07-31 17:01:08) ran for **~11h50m**, cloned essentially the entire dataset (including finishing `category_profile`'s 746M-document index build, reaching 99%), and then **failed at 2026-08-01 04:51:07** with:

```
InitialSyncFailure: HostUnreachable: Error cloning collection 'edu_lesson.process_checkpoints'
:: caused by :: network error while attempting to run command 'collStats' on host 'mongo.southeastasia.cloudapp.azure.com:27017'
```

This was **the exact same collection, same command (`collStats`), same error class** as attempt 1's failure (see `researchs/mongodb-initial-sync-mechanism.md`). Two independent ~12-hour attempts, ~12 hours apart in wall-clock start time, both died at the same specific point near the end of the `edu_lesson` database's clone — no longer consistent with ordinary random packet loss. Attempt 3 started immediately (`initialSyncAttempt: 3` of `initialSyncMaxAttempts: 10`).

### Root cause found (2026-08-01): Azure public IP idle timeout, not `process_checkpoints` itself

- `edu_lesson.process_checkpoints` is trivial (`count: 3`, `storageSize: 0.035MB`) — not the actual problem, just the next metadata call issued after the failure trigger.
- `vm-mongo-masterPublicIP` had `idleTimeoutInMinutes: 4` (the Azure default/minimum for a Standard public IP). Any TCP connection through it that goes fully idle for 4 minutes gets reset by Azure's networking layer.
- **The timeline lined up exactly**: in both failed attempts, the last thing before failure was `category_profile` (746.4M docs) finishing its clone and then running a **~45-minute local index build on the AWS side** (WiredTiger external-sorter, purely CPU/disk-bound — `mongod` pinned at ~107% CPU during this phase). During that window, initial sync sends nothing over the sync connection. 4 minutes into the idle window, Azure kills the connection; when the index build finishes and sync tries to resume, it finds a dead connection.
- Deterministic, not random — every attempt was expected to hit the same ~45-minute idle window right after `category_profile`'s index build.

**Applied (2026-08-01, during attempt 3)**: raised `vm-mongo-masterPublicIP`'s `idleTimeoutInMinutes` from 4 to the Azure Standard-SKU max of 30. Confirmed change, zero VM/downtime impact. **Not sufficient alone** — the index build (~45 min) still exceeded the new 30-min ceiling.

## Update (2026-08-01, later): third failure — idle-timeout fix was insufficient, native initial sync abandoned

Attempt 3 (after the idle-timeout bump) reached `category_profile`, cloned it, and built all 5 remaining indexes (after dropping `course_id_1` and the compound `course_id_1_profile_id_1` — see index-cleanup notes below) in only **~23 minutes total** — nowhere near the 30-minute idle threshold. It still failed at **15:51:16** with the identical error.

**This invalidated the idle-timeout theory as a complete explanation.** Comparing total attempt durations:

| Attempt | Duration |
|---|---|
| 1 | ~12.27 hours |
| 2 | ~11.83 hours |
| 3 | ~11.0 hours |

All three failed within a tight **~11–12.3 hour band**, despite attempt 3's index builds being drastically faster. This pattern looks much more like a **fixed maximum connection lifetime** somewhere on the network path (Azure NSG, an intermediate NAT/firewall, or the public IP's connection tracking beyond just idle time) rather than purely an idle-timeout triggered by one slow operation. Never fully diagnosed beyond this.

**Decision: abandon native replica-set initial sync for this migration.** Three full attempts cost ~36 hours cumulative with no success, and there was no confidence a 4th+ attempt would fare differently against an ~11-12h hard ceiling. Actions taken: stopped `mongod` on the AWS instance (halting the in-progress attempt 4 permanently), removed the AWS node from the replica set via `rs.remove()` — `vm-mongo-master` back to a clean single-member PRIMARY.

**Index cleanup performed on `vm-mongo-master` during this investigation** (data-driven, based on `$indexStats` usage — see mongodb.md discussion): dropped `course_id_1` (structurally redundant — `course_id` has exactly 1 distinct value across all 746.4M `category_profile` docs) and the compound `course_id_1_profile_id_1` (verified via `.explain()` that `profile_id_1` alone gives byte-identical query cost, zero docs rejected by residual filter) and `age_1` (zero usage in the ~1.5-day measurement window; `age` has only 2 distinct values, `2` and `100`, likely min/max content-tagging sentinels rather than a real per-document age). Remaining indexes on `category_profile`: `_id_`, `profile_id_1`, `category_id_1`, `order_by_1`, `version_1`.

## Path forward: snapshot + physical file copy

Avoids the fragile single-long-lived-connection initial sync mechanism entirely (see `researchs/mongodb-initial-sync-mechanism.md`):
1. Azure Disk snapshot of `MongoDisk` (2048GB) — `MongoDisk-snapshot-20260801`.
2. Transfer via S3 as an intermediate (`s3://eduhub123-mongo-migration-820883240614`).
3. Import as a native EBS snapshot → create an EBS volume → attach.
4. Mount the copied data (including the real `local.oplog.rs`) as `mongod`'s `dbPath`, start `mongod` — since real oplog continuity comes along, MongoDB recognizes existing data and catches up via **normal incremental oplog tailing** (resumable, unlike initial sync) rather than a fresh initial sync.
5. Source oplog window measured at ~807 days — no meaningful risk of it rotating past the snapshot's point before catch-up, even over a multi-hour transfer.

### Snapshot restore pipeline — execution log (2026-08-01/02)

1. **Azure Disk snapshot** (point-in-time 2026-08-01T08:03:16 UTC) — already existed from earlier prep.
2. **Download to AWS instance** via `azcopy` + SAS URL. Required resizing the instance's EBS volume from 1200GB→2200GB first (the VHD is pre-allocated at its full 2048GB logical size on download, not sparse). Took ~4.5 hours, all 2199023256064 bytes transferred.
3. **Upload to S3** via `aws s3 cp` — required installing `awscli` on the instance and a temporary scoped IAM policy (`mongo-migration-s3-access`, out-of-band, removed post-restore). ~1 hour at ~210-560MB/s. Local VHD copy then deleted.
4. **Import to native EBS snapshot** via `aws ec2 import-snapshot` — required creating the AWS account's `vmimport` IAM service role from scratch (one-time, account-level, left in place rather than treated as temporary).
5. **`import-snapshot` stalled twice, abandoned in favor of a direct raw-block restore.** Both attempts plateaued at exactly 19% "downloading/converting" with no further movement for 30-40+ minutes each. Investigated every plausible customer-side cause before concluding it was an opaque internal AWS backend issue: VHD footer valid (`conectix` cookie, `Data Offset` all-Fs + `Disk Type=2` confirming a proper Fixed VHD), CHS geometry (`65535/16/255`) confirmed as the mathematically correct spec-compliant clamp for any disk over ~127.5GB (not an anomaly), object size exact, S3 encryption plain SSE-S3/AES256 (no KMS), EBS regional gp3 quota nowhere near its limit, GPT/MBR irrelevant since `import-snapshot` doesn't parse partitions. Both attempts stalling at the identical percentage was the deciding signal to stop retrying the same way.
6. **Pivoted to a self-controlled raw-block restore**, bypassing AWS's managed conversion entirely:
   - Created a fresh 2200GB gp3 EBS volume, attached as `/dev/nvme2n1`.
   - Streamed directly from S3 onto the raw device: `aws s3 cp ... - | head -c 2199023255552 | sudo dd of=/dev/nvme2n1 bs=1M` (the `head -c` strips the VHD's trailing 512-byte footer, since a Fixed VHD is just raw disk bytes + footer, no block-allocation-table indirection). Took ~2.8 hours, exact byte match confirmed via `dd`'s own summary.
   - Discovered the partition table, confirmed same layout as source disk, mounted read-only.
   - Found the real data: 831GB, timestamps matching the snapshot's point-in-time exactly.
   - `rsync`'d that subtree into the instance's actual `/mongodb/db`. Needed **3 attempts**: plain `nohup` and `nohup`+`setsid` both got killed by `SIGHUP`/`SIGTERM` when the SSM session ended; running it as a proper systemd transient unit (`systemd-run --unit=mongo-rsync`) finally worked, fully independent of any SSM session. **Lesson**: verify a detached background process survives a session boundary early (seconds/minutes in), not just at the full expected-completion horizon — this caught the detachment bug immediately instead of after an ~80-minute blind wait.
   - Fixed ownership, verified size and readability, unmounted, detached the raw-restore volume (kept briefly as a safety net, removed once `mongod` confirmed healthy).
7. **Started `mongod` on the restored data** — WiredTiger auto-recovered via journal replay (expected, live block-level snapshot not a clean shutdown), briefly came up using its own stale (snapshot-time) replica-set config, contacted `vm-mongo-master`, learned the current authoritative config (no longer listing this host), and transitioned to **`REMOVED`** state — a normal, benign, inert state for a member not in the current config, not corruption.
8. **`rs.add(...)`** on `vm-mongo-master` — the node picked up the new config and, since it already had valid non-empty data + a real oplog, skipped initial sync entirely and went straight to **steady-state oplog catch-up**, replaying the ~26-hour gap. Reached **`SECONDARY`, 0 seconds of replication lag**, essentially immediately.
9. **Data-integrity spot check**: compared document counts across 7 large collections between pre-restore reference counts and the secondary's post-catchup counts — every collection came back at or slightly above its earlier reference, gaps consistent with normal ~1-day write volume. No data-loss or duplication signature.

**Cleanup done as part of this restore**: local VHD copy deleted post-S3-upload; raw-restore EBS volume detached and released once `mongod` confirmed healthy; leftover `snapshot-transfer` scratch directory removed from `/mongodb/db`.

## Cutover complete (2026-08-03): `rs.stepDown()` executed, AWS is now PRIMARY

`rs.stepDown(120)` run on `vm-mongo-master` — succeeded. Confirmed new state immediately after:

| Member | State | Health |
|---|---|---|
| `mongo.southeastasia.cloudapp.azure.com` (Azure, former primary) | SECONDARY | 1 |
| `ec2-54-179-59-251.ap-southeast-1...` (AWS) | **PRIMARY** | 1 |

Production traffic now served from AWS. Pre-stepdown state was verified live immediately beforehand (lag ~2s, both healthy) rather than relying on an earlier check.

**Connection-string rollout, final status before cutover**: what started as 35 Parameter-Store-flagged workloads (2026-07-31 baseline) narrowed dramatically once live pod inspection (not just Parameter Store values) was used as ground truth — several services turned out to already be running the correct string despite a stale-looking Parameter Store entry, and several Parameter Store keys turned out to need updating that weren't obvious from naming alone (`edu-ai` actually reads `env-eks-edu-mspeak`; `mk-classroom-queue-live` reads `env-eks-mk-classroom-go-2`). After updating the real remaining gap (29 Parameter Store keys), CI rolled out rebuilds, and by the final live recheck **zero workloads remained on the old connection string** (excluding `edu-okr`/`edu-campaign`/`edu-campaign-queue`, confirmed unused/deprecated). Full per-key breakdown is in the service-map JSON.

**Also resolved before cutover**: the EC2 standard On-Demand vCPU quota increase (64→128) that was open/blocking earlier is now `CASE_CLOSED`/approved.

**Azure removed from the replica set (2026-08-03, post-cutover)**: `rs.remove("mongo.southeastasia.cloudapp.azure.com:27017")` run against the new AWS primary. Replica set now a clean single-member config. Done deliberately *before* any subnet/networking hardening work on the AWS node — Azure and AWS never had a private network path, so removing Azure first eliminates the risk of the "move AWS back to private subnet" cleanup severing Azure's connectivity mid-replication. Azure's `vm-mongo-master` mongod process kept running with its data fully intact; settled into `REMOVED` state (same benign pattern observed earlier).

**Dev bastion access**: with direct developer connections to the AWS mongo node not possible, a guide was written at `docs/dev-guide-mongo-bastion-access.md`, using AWS SSM Session Manager port-forwarding through `bastion-prod` rather than a literal SSH tunnel.

**Bug found and fixed (2026-08-03)**: initial end-to-end test only verified raw TCP through the tunnel, which passed — but a real `mongosh` connection attempt (auth + wire protocol) failed with `Server selection timed out`. Root cause: the bastion instance had the **wrong security group attached** — a pre-existing legacy SSH-based bastion's SG, not the Terraform-built SSM-only module's own SG (`sg-0617fb0bb31499680`), which is the one mongo's inbound rule actually allows. The SSM tunnel worked fine regardless (SSM's control channel doesn't depend on SGs), masking the problem — but the bastion→mongo hop was silently dropped, zero trace in mongod's connection log. **Fix**: attached the correct SG *alongside* the existing one (additive). **Lesson**: a raw TCP-level tunnel test isn't sufficient to validate this kind of path end-to-end; the application-level handshake needs to be tested too.

**Post-removal verification (2026-08-03)**: confirmed both sides settled correctly, not just that commands returned success — AWS `rs.status()` clean single-member `PRIMARY`; Azure `rs.status()` returns the expected benign "not a member" response.

**Azure wind-down (2026-08-03)**: `mongod` stopped on `vm-mongo-master` — did **not** shut down gracefully (systemd's stop timed out, forced `SIGKILL`; not a data-loss concern, WiredTiger will do crash-consistent recovery if ever restarted). VM **deallocated** (compute billing stopped, disk/data fully preserved, reversible) — full teardown still a separate, later step.

**Dev bastion IAM access provisioned (2026-08-03)**: created managed policy `bastion-ssm-portforward-access` (scoped to `ssm:StartSession` on `bastion-prod` + the port-forwarding document only, plus session self-management) and attached to `huy.ngo@monkey.edu.vn`. Deliberately generic/reusable for future bastion-access grants, not mongo-specific — later reused as-is when `huy.ngo` also needed to reach `staging-monkey` (10.20.10.47), since it was already scoped to "any target reachable from the bastion," not locked to mongo specifically.

### Snapshot-migration temporary resources cleaned up (2026-08-03)

| Resource | Size | Result |
|---|---|---|
| Azure snapshot `MongoDisk-snapshot-20260801` | 2048GB | Deleted |
| S3 object `mongodisk-snapshot.vhd` | 2.0 TiB | Deleted |
| S3 bucket `eduhub123-mongo-migration-820883240614` | — | Deleted (emptied first) |
| Orphaned EBS snapshot `snap-0489a58623377460b` | 2048GB | Deleted — unexpected finding: the second cancelled `import-snapshot` task actually completed successfully on AWS's backend at some point after we'd abandoned it, silently producing this unused snapshot. Safe to delete. |
| Raw-restore EBS volume | 2200GB | Deleted — last "hot" duplicate copy of the database, removed only after confirming replica set + data-integrity checks stable |
| IAM inline policy `mongo-migration-s3-access` | — | Removed |

**Deliberately left in place**: the `vmimport` IAM role — one-time, account-level prerequisite for AWS VM Import/Export in general, not scoped to this migration.

All deletions verified afterward (each resource confirmed actually gone via a follow-up describe/show call, not just trusting the delete command's exit code).

## Post-cutover consumer discovery (see the service-map JSON for full detail)

The k8s-focused Parameter Store rollout missed two legacy standalone EC2 VMs entirely — `monkey-product` and `staging-monkey` — each with actively-broken connection strings still pointing at the now-dead Azure host, found only by tracing their domains from their load balancers. Both fixed. This confirmed a real blind spot flagged as a risk before the stepdown ("infra-side scans wouldn't catch an undocumented internal tool") — worth asking the team directly whether more such VMs exist, rather than assuming the current inventory is complete.

## Largest collections (checked 2026-08-01) — deferred optimization opportunity

While investigating a sync-progress plateau, queried `collStats` across all databases to find what's actually driving sync duration. Sorted by compressed on-disk `storageSize`:

| Collection | Documents | Logical size | Storage size (on disk) |
|---|---|---|---|
| **`edu_backend.history_user_action`** | 93.3M | 2.3TB | **~412GB** |
| `edu_lesson.lesson_profile` | 32.2M | 265GB | ~72GB |
| `edu_lesson.category_profile` | 746.4M | 174GB | ~56GB |
| `edu_lesson.learn_lesson` | 87.9M | 84GB | ~22.7GB |
| `edu_lesson.log_speech` | 28.5M | 60.6GB | ~18.7GB |
| `edu_lesson.log_upgrade` | 54.2M | 46.8GB | ~17.6GB |
| `edu_lesson.report_learn` | 27.0M | 34.9GB | ~16.8GB |
| (13 more, each under 10GB) | | | |

**`edu_backend.history_user_action` alone accounts for ~412GB — roughly half the entire ~831GB dataset.** It's an append-only user-action audit log (93M rows), and by its name/shape is a strong candidate for not needing full historical depth migrated. If the application doesn't actually read old rows in normal operation, filtering this collection (e.g. only recent N months) before/instead of a full clone would be the single highest-leverage way to speed up this and any future full-resync of this database.

**Decision (2026-08-01): kept syncing as-is, revisit this optimization later.** Not blocking, noted here so it isn't lost — and so any *future* full resync (e.g. for the deferred `vm-core-database` effort) starts by asking this question first rather than rediscovering it.

## Network/disk throughput check (2026-08-01)

When sync throughput appeared to be decelerating, measured actual live utilization on both sides — neither was close to any infrastructure limit:

| | Azure source (`vm-mongo-master`) | AWS target |
|---|---|---|
| Live measured throughput | ~6MB/s outbound | ~8MB/s inbound |
| Known caps | disk read ~91.5MB/s (`Standard_D4s_v3`) | EBS: 125MB/s throughput / 3000 IOPS (gp3 defaults); network: up to 10Gbit |

Both sides running at roughly 5-10% of available capacity — the pace was governed by MongoDB's own initial-sync mechanism (collection cloning, document-by-document), not network or disk bandwidth. A disk-usage-based progress proxy (`df -h /mongodb/db`) can appear to plateau even while genuine progress is happening, if the collection currently being cloned has many small documents relative to bytes — `mongod.log`'s `"collection clone progress"` messages (document-count percentage) were a more reliable real-time signal.

## Recommendations for later (not blocking this migration)

Surfaced while investigating sync behavior — should be picked up afterward rather than forgotten.

- **AWS instance is undersized on vCPU for steady-state, not just sync speed.** The target EC2 node is 4 vCPUs. Observed during `category_profile`'s index build: `mongod` pinned at ~107% CPU while 4 small straggler collections sat visibly stalled — not because MongoDB serializes collection cloning (it doesn't; distinct `ReplWriterWorker-N` threads work different collections), but because a single large index build can fully occupy the box's 4 cores, starving everything else. This is a real signal about production capacity: once this node serves live traffic (indexes, aggregations, concurrent connections from 37+ workloads), 4 vCPUs is likely a recurring bottleneck. **Recommendation: size up (e.g. 8 vCPU class).**

- **Data lifecycle / cold storage is a bigger lever than infra tuning, and looks neglected.** A few collections suggest manual partitioning was attempted and abandoned:
  - `edu_lesson.lesson_profile` (32.2M docs, ~72GB) vs `edu_lesson.lesson_profile_3` (seen straggling during sync) — a `_3`-suffixed sibling implies a prior manual sharding/partitioning scheme that was never finished, revisited, or documented. Worth finding out: is `_3` still being written to, or is it an orphaned partition nobody remembers the reason for?
  - `edu_backend.history_user_action` (93.3M docs, ~412GB storage, roughly half the entire dataset) is an append-only audit log by name and shape, with no sign of rotation/archival. Strong candidate for a retention policy: keep N months hot, archive older rows to cold storage (or a TTL index if the application never reads old rows).
  - General recommendation: before the deferred `vm-core-database` migration or any future full resync, do a deliberate pass over collections for this pattern (abandoned partitions, unbounded append-only logs with no TTL/archival).

## Open items / not yet resolved

- Confirm whether the `cal_report/*.py` standalone scripts (in `edu_learn_report`, not imported by the main FastAPI app) are still actively triggered (cron/Lambda) and need their connection string updated too, or are dead.
- Confirm what's connecting from `113.190.232.224` (flagged as a dev proxy) — later needs to sit behind a bastion SSH tunnel rather than direct access, independent of this migration.
- Full driver-compatibility sweep across the ~24 distinct images is only partially done (PHP fleet confirmed modern/compatible; Go services' individual versions noted in mongodb.md) — moot for *this* migration since the server version isn't changing, relevant again once a version-upgrade effort starts.
- Whether there are more legacy single-purpose EC2 VMs like `monkey-product`/`staging-monkey`, outside both the k8s cluster and the Parameter Store convention, that were never swept for connection strings — worth asking the team directly.
- `cms.monkey.edu.vn` resolves to an Azure AKS LoadBalancer IP (cluster `monkey-aks-live`) — a completely different backend from anything else traced in this migration. Never fully investigated (which Ingress/Service/pod serves it, its mongo config) — flagged, not resolved.
