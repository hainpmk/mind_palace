# vm-mongo-master migration (Azure → AWS) — index

This migration is complete (cutover 2026-08-03). Content previously here has been split into three focused documents:

- **[vm-mongo-master-migration-postmortem.md](vm-mongo-master-migration-postmortem.md)** — the full timeline: what was planned, what happened, what blocked us (the 3 failed native initial-sync attempts, root causes, the snapshot+physical-copy pivot), gotchas, dead ends, index cleanup, and the cutover itself.
- **[iac-terraform-notes.md](iac-terraform-notes.md)** — ongoing Terraform/CI context for managing this infra (`iac` repo's `envs/prod`, `modules/mongodb`, `modules/bastion`): the state-reconciliation work, the CI/CD gate fixes, and current resource references. Not a migration log — this is the living doc for future infra changes.
- **`docs/components-raw/mongodb-service-dependencies-2026-08-04.json`** — discovered service-to-backend dependency graph (which workloads/VMs call which databases). Marked as a temporary/working artifact, meant to seed a proper future service map, not a final authoritative source.

See also [mongodb.md](mongodb.md) for the original discovery record this migration was based on.
