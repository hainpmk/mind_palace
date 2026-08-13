---
status: accepted
---

# Terraform ops flow: CI-primary with an MFA-gated break-glass fallback, not local-apply-by-default

Terraform changes in the `iac` repo (see `docs/components/monkey-eks.md`, `docs/components/iac-terraform-notes.md`) could be applied by developers running locally with personal credentials, or exclusively through CI. We chose CI as the primary path: `terraform plan` runs on every PR (reviewable before merge), `terraform apply` runs via a CI service account assuming an AWS role over OIDC — no long-lived static AWS credentials on laptops or in GitHub secrets. This matches bnn's own stated goals (`CONTEXT.md`): safely reproducible, safely testable, semi-automated. Local `terraform plan` for iteration is fine; local `apply` against real environments is not part of the normal flow.

**Correction (2026-08-04)**: `apply` was originally intended to run automatically on merge to `main`, gated by a GitHub Environment's required-reviewers protection. In practice this was never enforced (`main` didn't exist until 2026-08-03, and required-reviewer protection on Environments turned out to need a paid GitHub plan this org doesn't have). Current actual flow is a manual two-phase `workflow_dispatch` for both plan and apply, with apply always a separate, deliberate, authorized trigger — not an automatic consequence of merging. See `docs/components/iac-terraform-notes.md` for the full detail and rationale.

We added a break-glass fallback rather than making CI a hard single point of failure, because GitHub has had recent outages and a devops engineer may need to apply during one. The fallback is a dedicated IAM role — not a standing broad credential — assumable only by named principals, requiring MFA, with a short session duration, and scoped to run the exact same `envs/prod`/`envs/nonprod` Terraform config against the same S3/DynamoDB state backends CI uses (never a separate ad hoc script or parallel state; DynamoDB locking prevents collision with a concurrent CI run). Because it bypasses PR review in the moment, usage must stay visible: a CloudTrail-based alert on assumption of the role, and a norm that any break-glass apply gets a follow-up commit/PR documenting what was applied and why, so git history doesn't silently diverge from real state. The break-glass role itself is provisioned through the normal CI path in advance, not during an outage — no bootstrapping problem.

The specific IAM principals allowed to assume the break-glass role are left as an explicit no-default Terraform variable (matching the pattern already used for `admin_cidr_blocks` in the bastion module) — deliberately not guessed or defaulted open.

No direct pushes to `main`.
