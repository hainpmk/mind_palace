# Sprint 1

Focus: kick off the `monkeyuni.net` → Cloudflare public DNS migration. Full task list in `backlog.md`; this is the curated subset to actually work on now.

## Done
- [x] Registrar access + Cloudflare account/API token unblocked.
- [x] Zone export from udag, cross-checked against known hostnames.
- [x] `cloudflare` Terraform provider + `envs/dns-cloudflare` module, own state/backend, own CI roles.
- [x] All 140 records live in Cloudflare, DNS-only, SPF fix applied, `terraform plan` shows `No changes` both locally and in CI.
- [x] Two real bugs found and fixed along the way: Cloudflare's zone-creation auto-scan racing our own record creation (`jump_start=false` + reconciliation, caught and removed a duplicate SPF record it had created), and an OIDC-permission gating bug in `modules/ci-role` (`grant_aws_infra_permissions=false` had incorrectly stripped permissions every instance needs unconditionally).

## Left in this sprint
- [ ] Lower TTLs at udag ahead of cutover, so a rollback propagates fast. **Next actionable step.**
- [ ] Cut over: point registrar NS to `selah.ns.cloudflare.com` / `yoxall.ns.cloudflare.com` via United Domains' "Own nameservers" panel.
- [ ] Post-cutover verification (hostname checklist, mail flow, ALB/ELB metrics) before decommissioning the old zone.

Not in this sprint: the other 3 root domains, Cloudflare proxy decision, and the unrelated `iac`/mongo cleanup backlog items — see `backlog.md`.
