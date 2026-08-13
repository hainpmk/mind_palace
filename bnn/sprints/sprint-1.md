# Sprint 1

Focus: `monkeyuni.net` → Cloudflare public DNS migration. Full task list in `backlog.md`.

## Done — migration complete
- [x] Registrar access + Cloudflare account/API token unblocked.
- [x] Zone export from udag, cross-checked against known hostnames.
- [x] `cloudflare` Terraform provider + `envs/dns-cloudflare` module, own state/backend, own CI roles.
- [x] All 140 records live in Cloudflare, DNS-only, SPF fix applied, `terraform plan` shows `No changes` both locally and in CI.
- [x] Three real bugs found and fixed along the way: Cloudflare's zone-creation auto-scan racing our own record creation (`jump_start=false` + reconciliation, caught and removed a duplicate SPF record it had created), an OIDC-permission gating bug in `modules/ci-role` (`grant_aws_infra_permissions=false` had incorrectly stripped permissions every instance needs unconditionally), and a real DNS record removed for confirmed-decommissioned `vm-core-service` — the latter doubling as the first real end-to-end test of the CI plan→apply pipeline.
- [x] Registrar NS cutover done — confirmed live via public resolvers and Cloudflare API (`status: active`).
- [x] Post-cutover verification: full hostname checklist passing against real-world resolvers; mail flow verified at DNS level (SPF chain resolves cleanly).

## Left, lower priority now (moved detail to backlog.md)
- [ ] Add a DMARC record — found missing during mail-flow verification, pre-existing gap.
- [x] Decommission the old udag zone — decided: leave untouched (no API access, low importance now).

Sprint's main goal (the migration itself) is done. Only DMARC remains — small enough to just track in `backlog.md` going forward rather than needing its own sprint.
