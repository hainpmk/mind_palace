# MongoDB

- **Cloud**: Azure — resource group `VMS`, region `southeastasia`
- **Management state**: manual (VM `vm-mongo-master`, no Terraform/Helm management found)
- **Criticality**: High — confirmed production data (`edu_app`) and multiple confirmed live consumers (see "Confirmed consumers" below)
- **Topology/mode**: **Standalone** — confirmed via `/etc/mongod.conf` on the VM: the `replication:` section is commented out, so no replica set is configured. Single VM (`vm-mongo-master`, `Standard_D4s_v3`, Linux, zone 3), no companion VM anywhere in the subscription. "master" in the name is a naming leftover, not an actual replica-set role.
- **Owner/contact**: TBD
- **Dependencies**: see "Confirmed consumers" below — both the live `monkey-eks` (AWS) and nonprod `onepercent-aks-v2` (Azure) clusters connect to this instance
- **MongoDB version**: 5.0.26 — **EOL** (MongoDB 5.0 reached end of life October 2024; no more security patches). Combined with the open-to-internet port 27017, this is a real exposure, not just a hygiene issue.
- **Service dependency map**: [mongodb-service-dependencies-2026-07-30.json](../components-raw/mongodb-service-dependencies-2026-07-30.json) — graph-ready (nodes/edges) extraction of every service confirmed connecting to each of the four Mongo instances, for later dependency-graphing.
- **Migration plan**: [vm-mongo-master-migration-plan.md](vm-mongo-master-migration-plan.md) — lift-and-shift to self-managed EC2 (Option B: single-node replica set → add AWS member → failover), same MongoDB version on the AWS target for this first move, action plan and open items tracked there.
- **Open questions**:
  - **No tags set** on the VM (empty `tags: {}`) — no owner/env/cost-center metadata at all
  - No replication = no built-in HA/failover today — resolved as part of the migration plan (becomes a real replica set)

## Confirmed consumers of vm-mongo-master (2026-07-29)

Identified via live established connections on the VM itself (`ss -tn state established`), then cross-referenced each source IP against known infrastructure (AWS NAT gateways via `aws ec2 describe-nat-gateways`, Azure public IPs via `az network public-ip list`) rather than assumed from app code alone:

| Source IP | Connections (snapshot) | Identity |
|---|---|---|
| `3.0.234.236` | 205 | AWS NAT gateway, VPC `vpc-02f43d05b165c22c2` — confirmed as **`monkey-eks`'s VPC** (live production EKS cluster) |
| `18.141.88.220` | 121 | AWS NAT gateway, same VPC as above — second AZ's egress for `monkey-eks` |
| `57.155.64.114` | 26 | Azure NAT gateway `aks-dev-nat` (RG `AKS`) — **`onepercent-aks-v2`'s own outbound NAT**, i.e. the nonprod dev/stg cluster |
| `113.190.232.224` | 20 | Not AWS/Azure infra — a dev proxy (not a production consumer; flagged for later — should go through the bastion's SSM Session Manager tunnel rather than direct internet access, independent of the migration; see `docs/components/monkey-eks.md`) |
| `20.6.34.63` | 15 | `vm-ai-machine-studio` (Azure VM) |
| `20.195.10.11` | 8 | `vm-core-service` (Azure VM) |

**Notable**: both the live (`monkey-eks`) and nonprod (`onepercent-aks-v2`) clusters connect to the same instance — the prod/nonprod isolation established in [ADR-0003](../adr/0003-cluster-topology-and-per-cluster-argocd.md) doesn't currently extend to this database. Not yet confirmed whether this is intentional (e.g. shared reference data) or unmanaged sprawl.

Confirmed in application code (`github.com/eduhub123/edu_learn_report`, the `data-report` service on `monkey-eks`) that the plain `URI` env var (this instance) is used by five live FastAPI routers (`mj_2024`, `ms_2025`, `mclass`, `learn_streak`, `cash_back_from_redshift`, all registered via `app.include_router(...)` in `src/app.py`) — plus three standalone `cal_report/*.py` scripts with `if __name__ == "__main__":` blocks that are **not imported by the running web app** (separate execution context, e.g. cron/Lambda — not yet confirmed which, or whether still active).

**Live (`monkey-eks`) consumers confirmed (2026-07-30)**: same bounded search (baked-in `.env`/config-file patterns, all unique workloads) run against the live cluster — **37 of 74 unique workloads** (roughly half of everything running on `monkey-eks`) hardcode a connection to this instance:

`app`: `data-report`, `data-segment`, `edu-ai`, `edu-app-queue`, `edu-app-v2`, `edu-app-v2-queue`, `edu-award`, `edu-cms`, `edu-cms-queue`, `edu-device-queue`, `edu-lesson-queue`, `edu-product`, `edu-share-photo`, `edu-story`
`crm`: `edu-auth`, `edu-auth-queue`, `edu-campaign`, `edu-campaign-queue`, `edu-crm-accountant`, `edu-crm-queue`, `edu-developer`, `edu-developer-queue`, `edu-mailsms-queue`, `edu-media`, `edu-media-queue`, `edu-okr`, `edu-ticket-queue`, `service-agent-queue`
`class`: `mk-classroom`, `mk-classroom-go`, `mk-classroom-go-cron`, `mk-classroom-go-queue`, `mk-classroom-queue-live`, `mk-course-go`, `mk-course-go-queue`
`hoc10`: `edu-lms`, `edu-question-service`

This is a materially bigger scope than the earlier code-level trace of `data-report` alone suggested — the pre-work step of updating connection strings before cutover (replica-set-aware URI, matching credential rotation if that's also done) touches roughly three dozen independently-built-and-deployed images, not one. Plan the pre-work timeline accordingly; this is very unlikely to be a same-day task.

**Nonprod (`onepercent-aks-v2`) consumers confirmed (2026-07-29)**: searched all 67 unique workloads across `app`/`class`/`cms`/`crm`/`dev`/`stg` for baked-in config referencing this instance (common `.env`/config-file patterns, bounded depth — not exhaustive, some Go binaries or non-standard paths could be missed). Found two:
- **`app/data-report-dev`** — same codebase as the live `data-report`, same `.env` pattern, deployed to nonprod too.
- **`stg/edu-story-stg`** — `/var/www/.env`: `DB_MONGO_URI=mongodb://admin:Admin%401234@mongo.southeastasia.cloudapp.azure.com:27017`, `DB_NAME_MONGO=edu_app`. **Same admin credential** (`admin:Admin@1234`) as `data-report`'s `URI` — a root credential shared across independent services, wider blast radius than a single compromised app, and rotating it means coordinating multiple deployments at once.
- **Resolved (2026-07-29)**: `vm-mongo-master`'s `edu_app` is **production**; `vm-core-database`'s `edu_app` is **nonprod/dev** — same name, different environment tier, not duplicates of the same data. Consistent with `vm-mongo-master` being the production instance and `vm-core-database` being the dev-environment instance deferred to a separate later migration effort.

## Other MongoDB instances discovered (2026-07-28 Azure sweep)

The full-subscription Azure resource sweep (`docs/components-raw/azure-sweep-2026-07-28.json`) surfaced two more running `mongod` instances beyond `vm-mongo-master`, neither previously accounted for in this migration's scope. Both confirmed live via `az vm run-command` (process/port check, not assumed from open NSG ports alone). Details TBD pending the same client usage/criticality gathering blocking the rest of this migration.

- **`vm-core-database`** (RG `VMs`, `Standard_D4as_v4`): Dockerized `mongo:5.0.26` (container `mongodb_mongodb_1` / `67000847c537`, same EOL version as `vm-mongo-master`), `--auth` enabled. Runs alongside MySQL, Maxwell (MySQL binlog CDC — likely feeds the `mk-classroom-go-cdc` deployment seen in the AKS `class` namespace), ELK stack, RabbitMQ, and Qdrant on the same VM. Port 27017 open to `0.0.0.0/0` via NSG, same exposure pattern as `vm-mongo-master`.
  - **Bigger than `vm-mongo-master`**: confirmed via `docker exec ... mongosh` (credentials below) — **14 non-empty databases, ~4.8GB total**, including `edu_app` (3.9GB), `edu_lesson` (830MB), `edu_backend`, `edu_class`, `edu_class_dev`, `edu_class_dev_v2`, `edu_data`, `edu_data_2`, `edu_share_photo`, `mk_course_dev`, `mk_course_dev_2`, plus `admin`/`config`/`local`. Database names overlap with the `app`/`class` domain areas in the AKS `onepercent-aks-v2` cluster.
  - **Confirmed live production dependency (2026-07-29)**: the `data-report` app (namespace `app`, live `monkey-eks` cluster on AWS) hardcodes `URI_DATA=mongodb://tuananh:...@20.212.252.28:27017/admin` (this VM's IP) in a `.env` file baked into its container image (source: `github.com/eduhub123/edu_learn_report`). Verified as actively used, not dead code:
    - `HistoryService` (`src/api/upload_data_warehouse/history.py`) connects to `edu_data.history` and is instantiated eagerly at module import time.
    - `Schedulers` (`src/services/scheduler.py`) connects to `edu_data.schedulers`, actively called from the live Clevertap campaign-scheduling feature (`src/api/clevertap/campaign.py`).
    - So `edu_data` specifically (827KB, small) is a real, live production dependency — treat this instance's migration priority accordingly, not as secondary to `vm-mongo-master`. The other 13 databases on this instance are not yet confirmed one way or the other.
  - **Security finding**: root credentials (`MONGO_INITDB_ROOT_USERNAME=early` / `MONGO_INITDB_ROOT_PASSWORD=abcd1234`) sit in plaintext in the container's env vars (visible via `docker inspect`), and the password is weak/guessable. Combined with the open-to-internet port, this is a live, real exposure — independent of the migration timeline, same category as the `vm-mongo-master` findings above.
- **`vm-monkey-staging`** (RG `VMs`, `Standard_B2ms`): native `mongod` (not Dockerized), **no auth** — `listDatabases` succeeded with no credentials. 7 databases (`edu_app`, `edu_backend`, `edu_lesson`, `edu_class_dev`, `admin`, `config`, `local`), ~5MB total. Also open to `0.0.0.0/0` on 27017. Looks like a legacy VM-hosted staging environment for Platform/CMS (custom ports 7012/7102 also open) that may predate the AKS `stg` namespace equivalents — possibly redundant, not yet confirmed.
- **`vm-core-service`** (RG `VMs`): Dockerized `mongo:7.0` (container `my_mongo_container`, port `27019`→`27017`, **no auth**), one of many unrelated services piled onto this VM (Jenkins, n8n, Redmine, MLflow, a reporting stack — see that VM's own component doc). Backs a **NodeBB forum, an internal tool** (not a customer-facing product) — 83MB `nodebb` database, live and actively used: 344 search topics, 353 search posts, 119,167 sessions, 477,327 objects. Not a dormant/test instance, but internal-tooling data rather than product data — likely a different criticality/priority bucket than the other three instances for migration purposes.

**Size verification pass (2026-07-28)**: after finding `vm-mongo-master`'s size was wrong (see Correction under Atlas migration option below), re-checked all three other instances by inspecting their actual data directories, not just trusting `listDatabases` output.
- `vm-core-database`: `/data/db` inside the container is 5.2GB — consistent with the ~4.8GB `listDatabases` total above. No correction needed.
- `vm-monkey-staging`: raw `/var/lib/mongodb` is 502MB, but 301MB is the journal (WAL) and 196MB is `diagnostic.data` (MongoDB's internal metrics, not user data) — actual collections/indexes are only a few MB, consistent with the ~5MB figure above. No correction needed.
- `vm-core-service`: raw `/data/db` is 580MB, same journal (301MB) + diagnostics (196MB) overhead, leaving ~83MB of real data — consistent with the NodeBB figure above. No correction needed.
- Why only `vm-mongo-master` was wrong: these three were measured by asking the live `mongod` process (`listDatabases`, which reports the storage engine's own `sizeOnDisk`), while `vm-mongo-master` was measured by guessing a filesystem path directly — that path turned out to be stale/unused.

## Security findings (from live discovery)

- **MongoDB port 27017 is open to inbound traffic from ANY source (`0.0.0.0/0`)** via NSG rule `AllowAnyMongoDBInbound` (priority 1010, protocol `*`, source `*`). The VM also has a public IP (`20.198.255.32`) and a public FQDN (`mongo.southeastasia.cloudapp.azure.com`). Unless MongoDB auth is enforced and strong, this is a critical exposure — worth verifying/fixing independently of the migration timeline.
- Data lives on the OS disk (`MongoDisk`, 2048GB Premium_LRS) — no separate data disk. Not wrong, but means OS and DB lifecycle are coupled (e.g. OS-level snapshot/restore affects data too).
- SSH (22) and NodeExporter (9100) are restricted to a small allowlist of source IPs — only Mongo's port is open to the world, which points to a specific misconfiguration rather than a generally permissive NSG.

## CPU usage investigation (2026-07-28)

**Symptom reported**: CPU spikes to ~90% for ~15 min, several times per hour. (Note: at the time, the dataset was believed to be a small 304MB — that figure was later found to be wrong, see **Correction** under Atlas migration option below. The actual dataset does not fit in RAM.)

**Root cause**: missing indexes, not resource sizing.
- `space_learning_data` (4.18M docs): `find({profile_id: ...})` had no index — full COLLSCAN every query (`keysExamined: 0`, `docsExamined: 4,179,744`, ~2.3-2.8s/query). 42,948 such slow queries found in one log window.
- `profile_review_games` (113K docs): `find({profile_id, ch_id}).sort({v: -1})` — same pattern, no index. 26,502 occurrences.
- Live at time of investigation: `mongod` was pegging ~400% CPU (all 4 cores) with load average climbing.
- Traffic mix: legitimate connections from `20.6.34.63` (internal AI-workload VM) and `3.0.234.236`/`18.141.88.220` (AWS-hosted apps — expected, per the "MongoDB exposed to internet until AWS migration" interim state). Checked for brute-force/scanning as an alternative explanation: recent auth failures (22 in ~200MB log tail, 12,917 across full 138GB history) were all from `127.0.0.1` with a bad username (`huy`) against `edu_app`/`edu_backend` — local misconfiguration noise, not external attack.

**Fix applied**: two indexes created via MongoDB Compass, non-blocking two-phase build (default since MongoDB 4.2):
```js
db.space_learning_data.createIndex({ profile_id: 1 })                          // completed 2026-07-28T03:49:05Z
db.profile_review_games.createIndex({ profile_id: 1, ch_id: 1, v: -1 })        // completed 2026-07-28T03:51:52Z
```

**Verified fixed**: zero slow-query log entries for either collection in the ~5 minutes following the second index's completion (previously logging thousands/hour). `mongod` CPU dropped from ~400% to ~122% (background baseline); load average decayed steadily afterward (15-min avg 5.63 → 3.54, 5-min avg 4.24 → 1.17 over ~10 min).

**Implication for the AWS architecture decision**: the CPU spikes were an application-level indexing problem, not a capacity/sizing problem. Bigger instances (Atlas M40 or otherwise) would have masked the symptom without fixing it, and cost would have kept scaling with data growth. Recommend re-measuring actual resource usage now that the real bottleneck is fixed, before finalizing target sizing — current usage pattern was not representative of true baseline load.

## Other findings from this investigation (not yet actioned)

- **mongod.log is 138GB and has never been rotated** (running since May 2024). No logrotate config found for it. Should be addressed independent of the migration — unbounded log growth risks filling the disk.
- **Passwordless `sudo` for the `monkey` OS user** on this VM — anyone with that SSH key gets instant root, no second factor.

## Cost estimate (retail, pay-as-you-go)

From Azure Retail Prices API, `southeastasia`, matched to actual SKUs in use:

| Item | SKU | Cost |
|---|---|---|
| Compute | `Standard_D4s_v3`, Linux | ~$182.62/mo |
| Disk | Premium SSD `P40` LRS, 2048GB | ~$259.05/mo |
| Public IP | Standard, static | ~$3.65/mo |
| **Total** | | **~$445/mo** (~$5,340/yr) |

Excludes bandwidth egress (usage-dependent) and any Reserved Instance/Sponsorship discount that may apply to the actual subscription.

## Atlas migration option (AWS)

- **Correction (2026-07-28)**: the original "304MB actual data size" below was measured against the wrong directory (`/var/lib/mongodb`, a stale/unused path). `/etc/mongod.conf` actually sets `dbPath: /mongodb/db` (confirmed against the live `mongod` process), and `du -sh /mongodb/db` shows **831GB**. The over-provisioning conclusion and the Atlas M40 default-storage sizing below were both based on the wrong number — do not treat them as settled; re-derive sizing from 831GB, not 304MB.
- ~~Actual data size: 304MB vs. 2048GB provisioned disk. Massive over-provisioning~~ — superseded, see Correction above. 831GB of 2048GB provisioned is a reasonable ~40% utilization, not over-provisioning.
- Atlas has **no standalone tier** — dedicated clusters are always a replica set with a minimum of 3 data-bearing nodes (confirmed via MongoDB docs). ~~Closest match to the requested 4 vCPU/16GB RAM is M40 (80GB default storage — well within the 304MB actual dataset)~~ — superseded: 80GB default storage is far short of 831GB; storage tier/cost needs to be re-evaluated against the real dataset size.
- Target region: **AWS ap-southeast-1** (Singapore), per decision.
- **Cost: unresolved**, and now more so given the corrected data size. Web searches returned conflicting figures for M40 3-node/ap-southeast-1 monthly cost (~$500/mo vs ~$2,277/mo compute, from unreliable third-party sources) and MongoDB's official pricing calculator is JS-rendered so it couldn't be queried directly. Get the real number from https://www.mongodb.com/pricing/calculator (AWS, ap-southeast-1, appropriate tier for 831GB) before treating any figure here as final.
- Moving to a 3-node replica set also resolves the no-HA gap noted earlier (see Topology/mode above), separate from the EOL-version and public-exposure issues.

## Raw discovery data

- [`azure-vm-mongo-master.json`](../components-raw/azure-vm-mongo-master.json) — `az vm show -d`
- [`azure-vm-mongo-master-nsg-rules.json`](../components-raw/azure-vm-mongo-master-nsg-rules.json) — `az network nsg rule list`
