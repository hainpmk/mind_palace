# vm-core-database

- **Cloud**: Azure — resource group `VMs`, region `southeastasia`
- **Management state**: manual (no Terraform/Helm management found)
- **Criticality**: TBD — needs owner input
- **Topology/mode**: single VM (`Standard_D4as_v4`), all services run as Docker containers directly on the host — no orchestration, no isolation between unrelated datastores
- **Owner/contact**: TBD
- **Dependencies**: [MongoDB](mongodb.md) — hosts one of the four discovered Mongo instances (`mongodb_mongodb_1`, `mongo:5.0.26`, EOL, `--auth` enabled). **Confirmed live consumer**: the `data-report` app (live `monkey-eks` cluster, AWS) uses the `edu_data` database here for upload-history tracking and Clevertap campaign scheduling — see [mongodb.md](mongodb.md) for details.
- **Open questions**:
  - Mongo container's root credentials found in plaintext env vars (`docker inspect`), weak/guessable password — see [mongodb.md](mongodb.md) for details. Needs rotating and moving out of plaintext env vars, independent of migration timeline.
  - `mk_classroom_maxwell` (MySQL binlog CDC) suggests this host feeds change-data-capture for the `mk-classroom-go-cdc` deployment seen in AKS `class` namespace — dependency direction not yet confirmed
  - Several ports (MySQL 3306, MongoDB 27017, Redis 6379, RabbitMQ 5672/15672) open to `0.0.0.0/0` via NSG — same exposure pattern as `vm-mongo-master`, worth a security pass independent of migration timeline
  - No tags set — no owner/env/cost-center metadata

## Discovery: running services (2026-07-28)

Confirmed via `az vm run-command` (`docker ps`), not assumed from NSG rules alone.

| Container | Image | Port(s) | Purpose (inferred) |
|---|---|---|---|
| `mongodb_mongodb_1` | `mongo:5.0.26` | 27017 | MongoDB (EOL) — see [mongodb.md](mongodb.md) |
| `mysql_db` | `mysql:5.7.42` | 3306 | MySQL |
| `mk_classroom_maxwell` | `zendesk/maxwell:latest` | — | MySQL binlog CDC — likely feeds `mk-classroom-go-cdc` in AKS |
| `docker-elk_elasticsearch_1` | `docker-elk_elasticsearch` | 9200, 9300 | Elasticsearch |
| `docker-elk_logstash_1` | `docker-elk_logstash` | 5044, 9600, 50000 | Logstash |
| `docker-elk_kibana_1` | `docker-elk_kibana` | 5601 | Kibana |
| `rabbitmq` | `rabbitmq:3-management-alpine` | 5672, 15672 | RabbitMQ (+ management UI) |
| `qdrant` | `qdrant/qdrant:latest` | 6333-6335 | Vector DB (likely AI-feature-related) |
| `node_exporter` | `prom/node-exporter` | — | Host metrics |
