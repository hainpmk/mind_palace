# Infrastructure Inventory

Index of known components. Organized by system/component, not by cloud — cloud is metadata on each entry since the target state is AWS-only (see [ADR-0001](adr/0001-adopt-on-touch-migration.md)).

Each component has a file under `docs/components/`, following [TEMPLATE.md](components/TEMPLATE.md).

Component docs are human-authored/annotated and must not be overwritten by automation. Automated discovery output lives in `docs/components-raw/` instead; see [ADR-0002](adr/0002-discovery-workflow.md) for the bootstrap-then-merge workflow.

## Components

_(populate as components are discovered — one row per file in `docs/components/`)_

| Component | Cloud | Management state | Criticality |
|---|---|---|---|
| [MongoDB](components/mongodb.md) | Azure | manual | High |
| [vm-core-service](components/vm-core-service.md) | Azure | manual | TBD |
| [vm-core-database](components/vm-core-database.md) | Azure | manual | TBD |
| [vm-monkey-staging](components/vm-monkey-staging.md) | Azure | manual | TBD |
| [monkey-eks](components/monkey-eks.md) | AWS | manual (eksctl) | High |
