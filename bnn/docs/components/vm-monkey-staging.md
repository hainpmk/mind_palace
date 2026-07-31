# vm-monkey-staging

- **Cloud**: Azure — resource group `VMs`, region `southeastasia`
- **Management state**: manual (no Terraform/Helm management found)
- **Criticality**: TBD — needs owner input
- **Topology/mode**: single VM (`Standard_B2ms`); this is **not** purely a native-services box — it runs a large mix of native systemd services (nginx, apache2, mongod, mysqld, redis-server) *and* ~18 Docker containers (app stacks). `mongod` itself is native (non-Dockerized), no auth enforced.
- **Owner/contact**: TBD
- **Dependencies**: [MongoDB](mongodb.md) — hosts one of the four discovered Mongo instances (native `mongod`, 7 databases, no auth)
- **Open questions**:
  - **This is a multi-tenant staging/dev host serving ~15 distinct domains** (`hoc10.vn` staging, several `monkeyuni.net`/`monkeyuni.com` dev/staging subdomains, `beta2.monkey.edu.vn`, `n8n.monkey.edu.vn`, `monkeyclass.net`/`.edu.vn`, `sharephoto.monkeyenglish.net`) — far broader than the "Platform/CMS staging VM" characterization from the first pass. See discovery section below for the full domain→port→container map.
  - **All ~18 app-stack Docker containers were found `Exited` at time of discovery** (`docker ps` returned zero running containers). The two most relevant (`edu_platform_web_1`, `edu_cms_web_3_1`) both exited with code 2 at the same instant (`2026-07-31T08:43:26Z`, restart policy `unless-stopped` — so Docker did not auto-restart them, consistent with a manual stop or simultaneous crash). **As of discovery time, none of the app domains below are actually being served** — nginx would return 502/504 for all of them. Needs a live check (or owner confirmation) of whether this is expected downtime or an incident.
  - Given the above, the AKS `stg` namespace's `edu-platform-stg`/`edu-cms-frontend-stg` vs. this VM's `platformstgmx.dev.monkeyuni.net`/`cmsstgmx.dev.monkeyuni.net` redundancy question from the first pass is still open, but now looks more like "this VM's copies are dev/staging endpoints with distinct hostnames" rather than a literal duplicate of the same environment — not yet confirmed which (if either) is authoritative.
  - MongoDB has **no authentication** — `listDatabases` succeeded with zero credentials, and port 27017 is open to `0.0.0.0/0` via NSG. Real (if small, ~5MB) databases: `edu_app`, `edu_backend`, `edu_lesson`, `edu_class_dev`. This is a live exposure, independent of the redundancy question above.
  - Same exposure pattern on MySQL (3306) and Redis (6379) — open to `0.0.0.0/0`
  - Empty tags (`{}`) — no owner/env/cost-center metadata
  - `/etc/letsencrypt/live/` holds certs for domains beyond what's currently wired into nginx (`monkeyclass.edu.vn`, `productstgmx.dev.monkeyuni.net`, `aiface.dev.monkey.edu.vn`, `api.dev.monkeyuni.com`, `kong.dev.monkeyuni.com`, etc.) — some nginx configs also have `_bak` suffixes (`beta2.hoc10.vn.conf_bak`, `phhs.dev.monkeyuni.com.conf_bak`), suggesting historical domains that were later retired/renamed. Not all cert directories necessarily correspond to currently-active vhosts.
  - **This VM has a live, ongoing outbound dependency into AWS, unrelated to the Mongo migration**: an AWS DMS (Database Migration Service) task is actively pulling from this VM's MySQL — see "AWS DMS dependency" section below. If this VM's MySQL is ever migrated/decommissioned, this pipeline needs repointing first or it breaks silently.

## Discovery: running services (2026-07-28)

Confirmed via `az vm run-command` (port/process check) and direct `mongosh` query (no credentials required), not assumed from NSG rules alone.

- **MongoDB** (native `mongod`, port 27017, no auth): `admin`, `config`, `local`, `edu_app`, `edu_backend`, `edu_lesson`, `edu_class_dev` — ~5MB total, all non-empty.
- **MySQL** (3306), **Redis** (6379) — present per NSG rules, not yet individually confirmed live/contents.
- Custom app ports `7012` (labeled "Platform") and `7102` (labeled "CMS") — not yet probed for what's actually listening.

## Discovery: domains, listening ports, and Docker footprint (2026-07-31)

Confirmed via `az vm run-command`: `ss -tlnp` (actual listening sockets + owning process), `docker ps -a`, `nginx -T`-equivalent grep over `/etc/nginx/conf.d/*.conf` (`server_name`/`proxy_pass`/`listen`), `apachectl -S` + `grep ServerName /etc/apache2/`, and `ls /etc/letsencrypt/live/`. No changes made to the VM.

**Actual listening sockets (`ss -tlnp`)** — this is the ground truth; NSG rule *labels* ("Platform"/"CMS") do not match what's on the wire:

| Port | Process | Notes |
|---|---|---|
| 22 | sshd | |
| 80, 443 | nginx (3 worker procs) | reverse proxy for all domains below |
| 8078 | apache2 | serves `beta2.monkey.edu.vn` directly (see below) |
| 3306, 33060 | mysqld | MySQL + X Protocol |
| 6379 | redis-server | |
| 27017 | mongod | native, no auth — see MongoDB discovery above |
| 53 (127.0.0.53) | systemd-resolved | local resolver, not externally relevant |

Notably **nothing was listening on 7011/7012/7101/7102/8080/8092/etc. at scan time** — those are ports nginx *proxies to*, but the backend Docker containers were all down (see below). NSG opening 7012/7102 directly (bypassing nginx) suggests those specific ports are meant for direct external/internal access to the container's secondary port, separate from the nginx-fronted primary port.

**nginx vhosts (`server_name` → `proxy_pass` target), from `/etc/nginx/conf.d/*.conf`:**

| Domain(s) | Proxies to | Backing container (by port mapping) |
|---|---|---|
| `platformstgmx.dev.monkeyuni.net` | `127.0.0.1:7011` | `edu_platform_web_1` (image `edu_platform_web`, maps `7011-7012→7011-7012`) — this is the **"Platform"** app the NSG label refers to; nginx fronts port 7011, NSG's 7012 is the container's second exposed port, not proxied by nginx |
| `cmsstgmx.dev.monkeyuni.net` | `127.0.0.1:7101` | `edu_cms_web_3_1` (image `edu_cms_web_3`, maps `7101→7001`, `7102→7002`) — the **"CMS"** app; nginx fronts 7101, NSG's 7102 is the container's second port |
| `cmsstgmx.dev.monkeyuni.com` | (no proxy_pass found — likely redirect-only or 404 stub) | — |
| `cms.stg.monkeyuni.com` | (no proxy_pass found — likely redirect-only or 404 stub) | — |
| `productstgmx.dev.monkeyuni.net` | `127.0.0.1:8080` | `php_product` (image `edu_product_php_product`, maps `8080→80`) |
| `staging.hoc10.vn`, `www.staging.hoc10.vn` | upstream `hoc10` (named upstream, not resolved here) | likely `edu_lesson`/`edu_app` stack — not confirmed further |
| `api.dev.monkeyuni.com` | `127.0.0.1:8000` | not identified among current containers (Kong listens 8000-8001, but Kong is also exited — needs live check) |
| `kong.dev.monkeyuni.com` | `127.0.0.1:1337` | `konga` (Kong's admin UI, image `pantsel/konga`) |
| `monkeyclass.edu.vn`, `monkeyclass.net`, `www.appv2.monkeyuni.net`, `www.media.monkeyuni.net` | `127.0.0.1:8092` | shared backend on 8092 — container not identified (not in current `docker ps -a` port list); possibly a service not shown or a native process |
| `n8n.monkey.edu.vn` | `127.0.0.1:5678` | `n8n` (Docker, `n8nio/n8n`) |
| `sharephoto.monkeyenglish.net` | `127.0.0.1:8082` | not identified among current containers |
| `beta2.monkey.edu.vn` | `127.0.0.1:8078` | **apache2** (native, not Docker) — confirmed via `apachectl -S`, vhost file `/etc/apache2/sites-enabled/beta2.monkey.edu.vn.conf` |

**TLS**: all above domains have certbot-managed certs under `/etc/letsencrypt/live/<domain>/` (Let's Encrypt), confirming these are the real, currently-configured domains for this box — not speculative. `/etc/letsencrypt/live/` also has entries for domains with no corresponding active nginx vhost found (`monkeyclass.edu.vn` has both a cert and a live vhost; but `productstgmx.dev.monkeyuni.net`, `aiface.dev.monkey.edu.vn`, `api.dev.monkeyuni.com`, `kong.dev.monkeyuni.com` certs exist and do have matching vhosts too — full reconciliation not done, see open questions).

**Docker containers (`docker ps -a`, 2026-07-31)** — **all 18 were `Exited`, none running** at scan time:

| Container | Image | Ports (when up) | Relevance |
|---|---|---|---|
| `edu_platform_web_1` | `edu_platform_web` | 7011-7012 | backs `platformstgmx.dev.monkeyuni.net` (the NSG "Platform" port) |
| `edu_platform_worker_1`, `edu_platform_consumer_1` | `edu_platform_worker`, `edu_platform_consumer` | 7012 (internal) | background workers for the Platform stack |
| `edu_cms_web_3_1` | `edu_cms_web_3` | 7101-7102 | backs `cmsstgmx.dev.monkeyuni.net` (the NSG "CMS" port) |
| `php_product` | `edu_product_php_product` | 8080 | backs `productstgmx.dev.monkeyuni.net` |
| `edu_app_platform_go_web_1` | `edu_app_platform_go_web` | 9101 | not yet mapped to a domain |
| `edu_app_story_go_web_1` | `edu_app_story_go_web` | 9102 | not yet mapped to a domain |
| `php_lesson_1` | `edu_lesson_php_lesson_1` | 9097 | not yet mapped to a domain; likely `edu_lesson` DB owner |
| `n8n` | `n8nio/n8n` | 5678 | backs `n8n.monkey.edu.vn` |
| `qdrant`, `ollama`, `self-hosted-ai-starter-kit-postgres-1` | — | 6333, 11434, 5432 | local AI/vector-DB experimentation stack, unrelated to the app domains |
| `kong`, `konga` | `kong`, `pantsel/konga` | 8000-8001/8443-8444, 1337 | API gateway + its admin UI; `konga` backs `kong.dev.monkeyuni.com` |
| `dev_app_gateway_db_1` | `postgres:9.5` | 5432 | DB for an unidentified "dev app gateway" |
| `node_exporter` | `prom/node-exporter` | — | Prometheus metrics exporter |

All containers with `restart: unless-stopped` were confirmed (via `docker inspect`) to have exited simultaneously at `2026-07-31T08:43:26Z` (exit code 2, e.g. `edu_platform_web_1`, `edu_cms_web_3_1`) and were **not** auto-restarted by Docker — consistent with a manual `docker stop`/compose-down rather than a crash loop, but not confirmed. VM itself has been up 8 days (`uptime`), so this was not a host reboot.

**Net finding**: the ports the original NSG-based note flagged as "Platform" (7012) and "CMS" (7102) are real app stacks (`edu_platform_web`, `edu_cms_web_3`) fronted by nginx via `platformstgmx.dev.monkeyuni.net` and `cmsstgmx.dev.monkeyuni.net` respectively — but this VM is not a single-purpose Platform/CMS staging box, it's a shared dev/staging host for roughly a dozen distinct product/tooling domains, and at discovery time **all of the Docker-backed domains were down** (containers exited, nginx would 502/504). Only the apache-backed `beta2.monkey.edu.vn` and native services (mongod, mysqld, redis) were actually live.

## Discovery: AWS DMS dependency (2026-07-31)

Prompted by checking this VM's outbound/inbound connections for anything reaching AWS production infrastructure (`ss -tn state established`). Found one unexpected remote peer among the many inbound clients: `54.151.196.67` connecting to local port `3306` (MySQL).

- Cross-referenced against AWS's published IP ranges (`ip-ranges.amazonaws.com`) — confirmed AWS-owned, `ap-southeast-1`, EC2 service.
- `aws ec2 describe-network-interfaces --filters Name=association.public-ip,Values=54.151.196.67` resolves it to an ENI tagged `DMSNetworkInterface`, owned by our account (`820883240614`), sitting in a **third VPC** (`vpc-0c8158d6ca5c51928`) not otherwise touched by this project — attached since **2022-03-18**, well before anything else discovered so far.
- `aws dms describe-replication-instances` / `describe-replication-tasks` confirms this is a real, **currently running** AWS DMS pipeline:
  - Replication instance: **`migration-bi-sql-redshift`** (`dms.t3.medium`, status `available`).
  - Task **`crm-redshift-synchronization`**: `full-load-and-cdc`, **`Status: running`**. Source tables: `edu_crm.*` and `edu_agent.*` from this VM's MySQL. Full load completed 2026-07-21 (163 tables); has been doing live CDC (streaming every MySQL change) continuously since.
  - A second task, **`crm-s3-task`**, also exists (same source account/region) — likely the same MySQL data also feeding an S3 data lake target; not fully inspected.

**Assessment**: this is a legitimate, intentional BI/analytics pipeline (MySQL → Redshift/S3 for CRM reporting), not a rogue or suspicious connection — but it's a real, live, previously-undocumented dependency on this VM's MySQL staying up and reachable. Anyone planning to touch, migrate, or decommission this VM's MySQL needs to account for and repoint this DMS task first, or the CDC stream breaks silently. Not related to the MongoDB migration currently in progress; noted here purely as a discovery-time finding.
