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

## CI OIDC trust was broken from day one — three stacked bugs, found via PR #1's plan (2026-08-11)

Opening the first real PR against `envs/prod`/`modules/**` (the mongo private-DNS-zone work below) surfaced that `prod-plan.yml` had **never once succeeded** — `Not authorized to perform sts:AssumeRoleWithWebIdentity`, every time. Three independent, stacked bugs, found one at a time as each fix exposed the next:

1. **Plan role's trust condition could never match a `pull_request`-triggered token.** It required `sub` to be `repo:eduhub123/iac:ref:refs/pull/*`, but GitHub's real `sub` claim for `pull_request` events is `repo:eduhub123/iac:pull_request` — no `ref:` segment at all for that event type. Fixed in `modules/ci-role`: `allowed_ref = "refs/pull/*"` is now translated to the real `pull_request` claim instead of an unmatchable `ref:` pattern.
2. **Apply role's trust condition broke the same way, for a different reason.** `prod-apply.yml`'s job declares `environment: prod`, and per GitHub's OIDC docs, declaring `environment:` on a job changes `sub` to `repo:OWNER/REPO:environment:NAME` — regardless of whether any protection rules are actually configured on that environment. The apply role expected `ref:refs/heads/main`, which such a job's token can never produce. Fixed via a new `github_environment` variable on the same module.
3. **The real root cause, underneath both: this repo uses GitHub's immutable OIDC subject format.** Created 2026-07-31, after GitHub's 2026-07-15 cutover — repos created on/after that date always mint `sub` as `repo:OWNER@OWNER_ID/REPO@REPO_ID:...`, never the plain name-only form, with no opt-out. Neither fix above accounted for this, so both roles' conditions *still* didn't match any real token even after being individually "fixed" — confirmed by re-testing and hitting the identical `AssumeRoleWithWebIdentity` failure again. Fixed by adding `github_owner_id`/`github_repo_id` (`7600709`/`1318260242`, via `gh api repos/eduhub123/iac --jq '.owner.id,.id'`) into the `sub` claim for all three cases the module handles.

**How these got applied**: none of them could be applied via CI, by construction — the very role being fixed can't assume itself to fix itself. The break-glass role (`docs/adr/0004-terraform-ops-flow.md`) was the obvious next path, but turned out to have **no IAM permissions at all** for `ci-role`/`break-glass-role` themselves (its `iam_for_ssm_roles` policy is scoped only to the `*-ssm` instance-role naming pattern used by `mongodb`/`bastion`, not to these roles). Applied directly with an MFA-authenticated session under a devops engineer's own IAM user instead (satisfies the break-glass role's own `aws:MultiFactorAuthPresent` condition, just without needing to actually assume that specific role).

**Immediately after the OIDC layer started working, two more real bugs surfaced in the same pass**, both root-caused by never having a working baseline to test against before:

- **Plan role had no DynamoDB permissions at all**, and `terraform plan` acquires the state lock by default even though it never writes state. Fixed by adding `-lock=false` to `prod-plan.yml`'s plan step, rather than widening the read-only role to cover locking.
- **`terraform plan`/`apply` had no way to supply the three no-default variables** (`mongodb_keyfile_content`, `break_glass_principal_arns`, `break_glass_notification_emails`) — no GitHub repo secrets/variables existed, and the workflow had no `-input=false`, so once the OIDC and lock issues were fixed, the very next run hung for **4.5 hours** prompting on stdin for a value a non-interactive runner can never provide. See the new SSM section below for the fix to the keyfile specifically; `break_glass_principal_arns`/`break_glass_notification_emails` still need an equivalent (they're not secret, just deliberately no-defaulted — GitHub Actions repo *variables* would suffice, not secrets).

**Net effect**: this repo's CI had never successfully planned or applied anything, end to end, at any point before 2026-08-11 — every prior apply (state reconciliation, the mongo migration infra itself) went through local admin credentials or manual `terraform apply`, never through `prod-plan.yml`/`prod-apply.yml` as designed. Worth re-reading `docs/adr/0004-terraform-ops-flow.md` with this in mind — the two-phase design was sound, but had never actually been exercised.

## Internal DNS name for the mongo primary — zone created, not yet wired to any connection string (2026-08-11)

Currently every connection string (and this migration's own tooling) references `ec2-54-179-59-251.ap-southeast-1.compute.amazonaws.com` — a name that only exists because of the temporary public-IP bridge used during migration, not something actually controlled. `modules/dns-private-zone` + `envs/prod`'s `module "dns"` now create a Route 53 **private hosted zone** (`mongodb-prod.monkeyuni.net`, VPC-scoped, split-horizon — doesn't affect or require access to any real public zone) with one record, `primary.mongodb-prod.monkeyuni.net` → the mongo primary's private IP, giving a stable internal FQDN that can be repointed on IP/subnet changes without touching CoreDNS or any connection string. Deliberately scoped to the `mongodb-prod` subdomain rather than the `monkeyuni.net` apex — a private zone at the apex would shadow every other `*.monkeyuni.net` subdomain (e.g. `api.`, `cms.`) for anything resolving inside that VPC, with no fallback to public DNS for names not mirrored in the zone.

**Not yet wired into any connection string** — introducing a new hostname means repeating the entire connection-string rollout (~29 Parameter Store keys, ~20-30 workload redeploys). Revisit as a deliberate, separate cleanup effort — and when connection strings get touched again for that purpose, that's also the moment to reconsider the current EC2-hostname-based naming for good.

## Manual step required: mongo replica-set keyfile lives in SSM, not Terraform

`envs/prod` reads the mongo replica-set internal-auth keyfile from SSM Parameter Store (`/mongo/prod/keyfile`, `SecureString`, default `alias/aws/ssm` key) via a `data "aws_ssm_parameter"` source, not a bare Terraform variable — the old `mongodb_keyfile_content` variable had no default (correctly, since it's a live secret) and CI had no way to supply it at all, which is what caused `prod-plan.yml` to hang for 4.5 hours the first time the OIDC layer actually worked (see the CI reconciliation section below).

**The parameter itself is created/updated manually** (`aws ssm put-parameter --name /mongo/prod/keyfile --type SecureString --overwrite`), not Terraform-managed. If the keyfile ever needs rotating, that command is the only place to do it — Terraform will never see or diff this value.

**TODO, deliberately deferred**: Terraform can't define an "empty" `SecureString` shell without supplying some value via the resource's `value` argument, which would put the real secret into state. The correct fix is a placeholder `aws_ssm_parameter` *resource* (not just a data source) with a throwaway initial value and `lifecycle { ignore_changes = [value] }` — same pattern already used for `aws_instance.mongodb`'s `user_data` in `modules/mongodb`. That would make the parameter's existence/type/key association Terraform-owned while the actual value stays a manual, out-of-band step forever after. Not done yet — plenty of this repo's other resources are still manual/unmanaged too, and this isn't worse than that; revisit as part of a general push to reduce manual bootstrap steps, not as a one-off.

## Not yet done (operational cleanup, still outstanding)

- Move the AWS mongo node back to the private subnet (`prod-private-1a`) — remove the temporary `subnet_id` override in `envs/prod/main.tf`.
- Release the temporary public IP / the now-obsolete `20.198.255.32/32` SG rule (Azure's IP — Azure side is deallocated, this rule is now dead weight) / the `/etc/hosts` self-reference workaround (no longer needed once there's no public IP).
- Formal Azure-side teardown: delete `vm-mongo-master`'s VM/disk/NSG resources (currently only deallocated, not deleted).
- Set up TLS in transit if a cross-cloud replication pattern recurs for the deferred `vm-core-database` migration.
- Consider whether to rename the replica set from `atlas-um8iyc-shard-0` to something reflecting current reality — cosmetic, not urgent.
- Turn `/mongo/prod/keyfile`'s SSM parameter into a real Terraform resource (placeholder value + `lifecycle { ignore_changes = [value] }`) instead of a manually-created, Terraform-invisible one — see the SSM section above.
- Supply `break_glass_principal_arns`/`break_glass_notification_emails` to CI too — not secret (an IAM ARN, an email address), just deliberately no-defaulted, but CI still has no source for them. GitHub Actions repo *variables* (not secrets) would suffice.
- Wire the new `mongodb-prod.monkeyuni.net` private zone into actual connection strings — see the DNS section above.
