# vm-core-service

- **Cloud**: Azure — resource group `VMs`, region `southeastasia`
- **Management state**: manual (no Terraform/Helm management found)
- **Criticality**: TBD — hosts a mix of internal tooling and what look like live product APIs; needs owner input to assess per-service, not as one VM
- **Topology/mode**: single VM (`Standard_E2ds_v4`), all services run as Docker containers or systemd units directly on the host — no orchestration, no isolation between unrelated workloads
- **Owner/contact**: TBD
- **Dependencies**: [MongoDB](mongodb.md) — hosts one of the four discovered Mongo instances (`my_mongo_container`, `mongo:7.0`, backs an internal NodeBB forum — not customer-facing)
- **Open questions**:
  - This VM is a grab-bag of unrelated services accumulated over time — worth deciding whether any of it should be split out / retired before AWS migration, rather than lift-and-shifted as one box
  - `report_web` container is currently unhealthy (per `docker ps` status) — not yet investigated
  - Several containers (`parent_report_api`, `user_activity_report`, `learning_report`, `data-report`-style naming) look like they may duplicate or predate reporting workloads seen in AKS (`data-report-dev` in the `app` namespace) — not yet confirmed
  - No ArgoCD/GitOps tooling here (confirmed) — Jenkins (CI) and n8n (workflow automation) are the only automation tools present, neither overlaps with the AKS `argocd` namespace's responsibility

## Discovery: running services (2026-07-28)

Confirmed via `az vm run-command` (`docker ps` + `systemctl list-units`), not assumed from NSG rules alone.

**Docker containers:**

| Container | Image | Port(s) | Purpose (inferred) |
|---|---|---|---|
| `my_mongo_container` | `mongo:7.0` | 27019→27017 | MongoDB — see [mongodb.md](mongodb.md) |
| `jenkins` | `jenkins_jenkins` | 8081→8080, 50000 | CI |
| `n8n` | `n8nio/n8n` | 5678 | Workflow automation |
| `n8n-extend-api_api_1` | `n8n-extend-api_api` | 18000 | n8n plugin/extension API |
| `wiki_web_node_1` | `wiki_web_node` | 14567→4567 | Internal wiki |
| `redmine_redmine_1` | `bitnami/redmine:6` | 8887→3000 | Project management |
| `redmine_mariadb_1` | `bitnami/mariadb` | 13308→3306 | DB for Redmine |
| `mlflow_server` | `mlflow_server` | 5000 | ML experiment tracking |
| `mlflow_db` | `mysql/mysql-server` | 13307→3306 | DB for MLflow |
| `parent_report_api` | `edu_learn_report_parent_report_api` | 5555→80 | Reporting API (product-facing?) |
| `edu_learn_report_redis_1` | `redis:latest` | 16379→6379 | Cache for reporting stack |
| `report_web_1` | `report_web` | 11333→8000 | Reporting web app — **unhealthy** |
| `user_activity_report` | `user_activity_report:v1.0` | 8006→8080 | Reporting API (product-facing?) |
| `learning_report` | `trunghoang12/mk:learning_report_v2.0` | 8000-8001 | Reporting API (product-facing?) |
| unnamed | (unlabeled) | 8089→80 | Unidentified |

**systemd services (non-default):** `nginx` (reverse proxy), `openvpn-server@server` (VPN).
