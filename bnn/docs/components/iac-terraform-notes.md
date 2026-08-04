# `iac` repo: Terraform & CI/CD operating notes

Living context for managing the `eduhub123/iac` repo's Terraform (`envs/prod`, `modules/mongodb`, `modules/bastion`) and its CI workflows — not a migration log. For the mongo migration's own history, see [vm-mongo-master-migration-postmortem.md](vm-mongo-master-migration-postmortem.md). See also `docs/adr/0001-adopt-on-touch-migration.md` and `docs/adr/0004-terraform-ops-flow.md` for the standing design decisions this builds on.

## Current resource quick-reference (as of 2026-08-04)

| Resource | Value |
|---|---|
| Mongo primary instance | `i-0da9f87254f9c2098`, private IP `10.20.1.42`, public DNS `ec2-54-179-59-251.ap-southeast-1.compute.amazonaws.com` |
| Mongo replica set name | `atlas-um8iyc-shard-0` |
| Mongo security group | `sg-07851ed9672124e6b`, name `mongodb-prod-rs-mongo-prod` (kept — see "SG/IAM naming decoupled" below) |
| Mongo IAM role/profile | `mongodb-prod-rs-mongo-prod-ssm` (kept) |
| Mongo data EBS volume | `vol-00548a3c8c0b8ec6b`, 2200GB |
| Bastion instance | `i-01bf08afb6d0fa78e`, private IP `10.20.1.50`, **not Terraform-managed** (see below) |
| Bastion security group | `sg-0617fb0bb31499680` (TF-managed, this one IS real/used) |
| `main` branch | Exists since 2026-08-03 (was `bootstrap-scaffold` before that — see CI/CD section) |

## Terraform state reconciliation (2026-08-03)

Went back to `envs/prod` post-migration to reconcile TF state against everything done out-of-band during the migration. `terraform plan` initially showed **7 to add, 2 to change, 6 to destroy** — most of it dangerous if applied blind:

- `module.mongodb.aws_security_group.mongodb` and the `mongodb_ssm` IAM role/instance-profile all `must be replaced` (destroy+recreate on the live primary) — root cause: `variables.tf`'s `mongodb_replica_set_name` default was already correctly set to `atlas-um8iyc-shard-0`, but that value fed straight into the SG/IAM role's `name` field, and any `name` change forces AWS-level replacement.
- `aws_ebs_volume.data` wanted to shrink `2200 → 1200` — reverting the mid-migration resize; AWS doesn't allow shrinking a volume, so this would have hard-failed.
- `module.bastion.aws_instance.bastion` would be **created** — checked all 16 historical state versions (S3 versioning is enabled on the state bucket) and confirmed a TF-managed bastion was never actually applied, at any point in this project's history (blocked by the EC2 vCPU quota limit at the time). The live `bastion-prod` (`i-01bf08afb6d0fa78e`) has always been a separately, manually-created instance (different AMI, `t4g.micro`/ARM vs the module's `t3.micro`, key pair `aws_data` vs `bastion-prod`, IAM profile `bastion-profile` vs the module's own unused `bastion-prod-ssm`) that got pressed into service and later had the correct SG attached.

**Fixes applied**, all non-destructive:
1. **Bastion left un-adopted**, per ADR-0001 ("adopt-on-touch" — import existing infra only when there's a real reason to change it, not speculatively). Removed `aws_instance.bastion` and everything that only existed to support it (`data.aws_ami.ubuntu`, `instance_type`/`key_name`/`public_subnet_id` vars, `instance_id`/`public_ip` outputs, the root `bastion_public_ip` output, `key_name` var + tfvars entry). The security group (feeds the mongodb module's `bastion_ingress` rule) and the orphaned `bastion_ssm` IAM role/profile are untouched.
2. **`data_volume_size_gb` default fixed** `1200 → 2200` to match the real mid-migration resize.
3. **Decoupled the mongo SG/IAM role's `name` from `replica_set_name`**: added a new `resource_name` variable (default `"rs-mongo-prod"`, the original never-actually-used planned name) used only for the SG/IAM `name` fields, while tags continue to correctly reflect `replica_set_name` (tag updates are safe in-place, unlike `name`). Avoids ever forcing a replacement of the live primary's security group or IAM role for a purely cosmetic naming correction.
4. **`ignore_changes = [user_data]`** added to `aws_instance.mongodb` — confirmed via the plan (no `# forces replacement` marker) and EC2 semantics (`user_data` is only read by cloud-init on first boot; updating it on an already-running instance has no runtime effect) that this diff was cosmetic noise, not a real risk.

Final plan converged to **0 to add, 3 to change, 0 to destroy** — three tag-only updates. Applied — completed in under 3 seconds, no replacement/downtime. Verified immediately after via `db.hello().isWritablePrimary` — still `true`.

### If `bastion-prod` ever needs a real Terraform-managed change

Don't recreate `aws_instance.bastion` speculatively. When there's an actual need, import it (`terraform import module.bastion.aws_instance.bastion i-01bf08afb6d0fa78e`) and reconcile the module's `ami`/`instance_type`/`key_name`/`iam_instance_profile` to match the live instance's real values first (AMI `ami-06f98272c773c9271`, `t4g.micro`, key `aws_data`, profile `bastion-profile`) — otherwise `plan` will propose destroying and recreating a live, in-use bastion that `huy.ngo`'s IAM grant and `docs/dev-guide-mongo-bastion-access.md` are both scoped to by instance ID.

## CI/CD reconciliation: `main` never existed, prod-apply had an unenforced gate (2026-08-03/04)

**`main` branch never existed.** `bootstrap-scaffold` had been this repo's *only* branch, and its actual GitHub default, since the initial scaffold commit — confirmed by listing remote branches (`git ls-remote --heads origin`) and independently via `gh repo view --json defaultBranchRef`. This mattered because `prod-apply.yml` was written to trigger on `push` to `main` — a branch that had never existed — meaning **the apply workflow had never fired, for any change, in this repo's entire history** (confirmed via `gh run list` returning zero runs total).

**Fixed**: renamed `bootstrap-scaffold` → `main` via the GitHub API (a ref-rename, not a `git push`, so — verified — this does not itself trigger any `push`-based workflow), then fixed the local clone to match (`git fetch --prune` to drop the stale `origin/bootstrap-scaffold` tracking ref, `git branch -m ... main`, re-pointed upstream to `origin/main`).

**The `environment: prod` gate on `prod-apply.yml` was never actually enforced.** The workflow's own comment said "gate with required reviewers in repo settings" — but `gh api repos/eduhub123/iac/environments` returned zero environments configured at all; GitHub auto-creates an environment on first reference with no protection rules by default. Attempted to configure required-reviewer protection directly via the API — rejected: `"Please ensure the billing plan supports the required reviewers protection rule"` (this repo is private and org-owned; required-reviewer protection on Environments needs GitHub Team/Enterprise, not the Free tier this org is on). **Net effect**: had a commit ever landed on `main` while the old `push`-triggered config was live, `terraform apply -auto-approve` would have run completely unattended against the live mongo primary, with no real review step.

### Current model: two-phase manual flow (no paid GitHub feature required)

- **`prod-plan.yml`** (phase 1): triggers on `pull_request` (existing) *and* `workflow_dispatch` (run/watch on demand). Plan-only, read-only `terraform-ci-prod-plan` OIDC role, never applies anything.
- **`prod-apply.yml`** (phase 2): `workflow_dispatch` only — no `push` trigger. **Deliberately not chained to phase 1.** Applying is always its own separate, explicit, authorized dispatch — by an engineer directly, or an agent acting under that engineer's real-time authorization for that specific run — never an automatic consequence of a push or merge.
- The `environment: prod` line is kept on the apply job (harmless, future-proofs if billing ever changes) but its comment now states plainly that it is **not** currently enforcing anything — the real safety is the manual-dispatch-only trigger plus this two-phase discipline.
- Explicitly **declined**: an automated plan-scanning guardrail that would refuse to apply anything destroying/replacing mongo-related resources. Decided the two-phase human-authorization model is sufficient without it — revisit if that assumption stops holding.

**How to actually run a change**:
1. Push to a branch, open a PR touching `envs/prod/**` or `modules/**` → `prod-plan.yml` runs automatically, plan visible in the PR.
2. Merge (or, to preview without a PR, dispatch `prod-plan.yml` manually from the Actions tab and read its output).
3. Once satisfied, manually dispatch `prod-apply.yml` from the Actions tab — this is the actual authorization step. Nothing applies before this.

## Deferred: proper internal DNS name for the mongo primary

Currently every connection string (and this migration's own tooling) references `ec2-54-179-59-251.ap-southeast-1.compute.amazonaws.com` — a name that only exists because of the temporary public-IP bridge used during migration, not something actually controlled. The clean fix is a Route 53 **private hosted zone** (e.g. `mongodb-prod.monkey.edu.vn`, VPC-scoped, split-horizon — doesn't affect or require access to the real public `monkey.edu.vn` DNS) giving a stable internal FQDN that can be repointed on IP/subnet changes without touching CoreDNS or any connection string. **Explicitly not done** — introducing a new hostname would mean repeating the entire connection-string rollout (~29 Parameter Store keys, ~20-30 workload redeploys). Revisit as a deliberate, separate cleanup effort — and if/when connection strings get touched again for that purpose, that's also the moment to reconsider the current EC2-hostname-based naming for good.

## Not yet done (operational cleanup, still outstanding)

- Move the AWS mongo node back to the private subnet (`prod-private-1a`) — remove the temporary `subnet_id` override in `envs/prod/main.tf`.
- Release the temporary public IP / the now-obsolete `20.198.255.32/32` SG rule (Azure's IP — Azure side is deallocated, this rule is now dead weight) / the `/etc/hosts` self-reference workaround (no longer needed once there's no public IP).
- Formal Azure-side teardown: delete `vm-mongo-master`'s VM/disk/NSG resources (currently only deallocated, not deleted).
- Set up TLS in transit if a cross-cloud replication pattern recurs for the deferred `vm-core-database` migration.
- Consider whether to rename the replica set from `atlas-um8iyc-shard-0` to something reflecting current reality — cosmetic, not urgent.
