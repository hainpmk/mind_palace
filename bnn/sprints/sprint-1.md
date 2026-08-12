# Sprint 1

Focus: kick off the `monkeyuni.net` → Cloudflare public DNS migration. Full task list in `backlog.md`; this is the curated subset to actually work on now.

## Blocking on the user/team (do these first — everything else in this sprint depends on them)
- [!] Confirm who holds registrar-level access to change NS delegation for `monkeyuni.net`.
- [!] Confirm which Cloudflare account to use / get a scoped API token.

## Once unblocked
- [ ] Get the authoritative zone export from udag. Cross-check against the known-hostname list (apex, `www`, `app`, the `*.dev.` subtree — `api.dev`, `crmdev`, `campaign.dev`, `okr.dev`, `media.dev`, `lms.dev`, `question.dev`, `agent.dev`, `tutoring.dev`, `apiv2.dev`, `storystg` — MX, TXT).
- [ ] Add `cloudflare` Terraform provider + new module to `iac`, own state/backend.
- [ ] Recreate every record from the real export in Cloudflare, DNS-only, low TTL. Bundle the SPF fix (add `include:_spf.google.com`) into this same change.
- [ ] Verify the new zone in isolation against Cloudflare's own nameservers before touching anything live.
- [ ] Lower TTLs at udag, then cut over NS at the registrar.
- [ ] Post-cutover verification (hostname checklist, mail flow, ALB/ELB metrics) before decommissioning the old zone.

Not in this sprint: the other 3 root domains, Cloudflare proxy decision, and the unrelated `iac`/mongo cleanup backlog items — see `backlog.md`.
