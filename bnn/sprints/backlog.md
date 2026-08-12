# Backlog

Unprioritized. See `README.md` for how this fits with `sprint-N.md`. Items link back to their source doc where one exists.

## Cloudflare migration: monkeyuni.net public DNS

Source: 2026-08-12 planning session. Full context/rationale in that session's plan; summarized here as tasks.

- [ ] Get the authoritative zone export from udag (current provider, `ns.udag.net`/`.org`/`.de`) — blocking prerequisite, public `dig` reconnaissance alone proved insufficient (missed the entire `*.dev.monkeyuni.net` subtree on first pass).
- [!] Confirm who holds registrar-level access to change NS delegation for `monkeyuni.net` — blocked on the user/team, not technical work.
- [!] Confirm which Cloudflare account to use / get a scoped API token — blocked on the user/team.
- [ ] Add `cloudflare` Terraform provider + new module (e.g. `modules/cloudflare-zone`) to `iac`, own state/backend, separate from the AWS `prod` CI roles.
- [ ] Recreate every record from the real udag export as `cloudflare_record` resources, all DNS-only (`proxied = false`), low TTL for the migration window. Known hostnames to verify present (root `monkeyuni.net` only): apex, `www`, `app` (explicit override), the `*.dev.` subtree (`api.dev`, `crmdev`, `campaign.dev`, `okr.dev`, `media.dev`, `lms.dev`, `question.dev`, `agent.dev`, `tutoring.dev`, `apiv2.dev`, `storystg`), MX (5 records), TXT (site-verification + SPF).
- [ ] Fix the SPF gap in the same change: `v=spf1 include:_smtp.udag.de ~all` → add `include:_spf.google.com` (missing today despite MX → Google Workspace).
- [ ] Verify the new Cloudflare zone in isolation (`dig @<cloudflare-ns> ...`) against the real export, before touching NS.
- [ ] Lower TTLs at udag ahead of cutover, so a rollback propagates fast.
- [ ] Cut over: point registrar NS to Cloudflare's assigned nameservers.
- [ ] Post-cutover verification: same hostname checklist against real-world resolvers, mail flow through Google Workspace, ALB/ELB metrics watch.
- [ ] Decommission the old udag zone once stable (no urgency).
- [ ] Follow-on, explicitly out of scope for this pass: same DNS-cutover-to-Cloudflare treatment for the other 3 root domains bnn flagged (`monkeyuni.com`, `monkey.edu.vn`, `monkeyenglish.net`) — see `docs/components-raw/aks-onepercent-ingress-domains-2026-07-28.md`.
- [ ] Follow-on, deliberately deferred: whether/when to enable Cloudflare's proxy (CDN/WAF) — separate decision from "just move DNS hosting."

## iac/mongo operational cleanup

Source: `docs/components/iac-terraform-notes.md`, "Not yet done" section.

- [ ] Move the AWS mongo node back to the private subnet (`prod-private-1a`) — remove the temporary `subnet_id` override in `envs/prod/main.tf`.
- [ ] Release the temporary public IP / the now-obsolete `20.198.255.32/32` SG rule / the `/etc/hosts` self-reference workaround.
- [ ] Formal Azure-side teardown: delete `vm-mongo-master`'s VM/disk/NSG resources (currently only deallocated).
- [ ] Set up TLS in transit if a cross-cloud replication pattern recurs for the deferred `vm-core-database` migration.
- [ ] Consider renaming the replica set from `atlas-um8iyc-shard-0` — cosmetic, not urgent.
- [ ] Turn `/mongo/prod/keyfile`'s SSM parameter into a real Terraform resource (placeholder value + `lifecycle { ignore_changes = [value] }`) instead of manually-created/Terraform-invisible.
- [ ] Wire the `mongodb-prod.monkeyuni.net` private Route53 zone into actual connection strings (~29 Parameter Store keys, ~20-30 workload redeploys) — separate, larger rollout.
