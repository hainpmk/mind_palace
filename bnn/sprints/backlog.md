# Backlog

Unprioritized. See `README.md` for how this fits with `sprint-N.md`. Items link back to their source doc where one exists.

## Cloudflare migration: monkeyuni.net public DNS

Source: 2026-08-12 planning session. Full context/rationale in that session's plan; summarized here as tasks.

- [x] Get zone export from udag (United Domains). Confirmed: `dig` reconnaissance missed most of it. Real count: 142 records (48 A, 84 CNAME, 5 MX, 5 TXT). File: `~/Downloads/export_zone_monkeyuni.net.txt`.
  - Confirms both wildcards suspected: `*.monkeyuni.net` and `*.dev.monkeyuni.net`, both CNAME.
  - **New finding, changes prior assumption**: 11 `*live` names (`authlive`, `airflowlive`, `crmlive`, `medialive`, `agentlive`, `apiv2live`, `campaignlive`, `emaillive`, `questionlive`, `ticketlive`, `accountantlive`) all → single IP `4.145.178.221`, Azure range — likely `monkey-aks-live` (per mongodb.md's earlier flag: "`cms.monkey.edu.vn` resolves to an Azure AKS LoadBalancer IP, cluster `monkey-aks-live`... never fully investigated"). Not fully AWS-migrated as previously assumed — real live Azure dependency, must not be dropped.
  - Multi-cloud/vendor spread beyond AWS+Azure: GCP IPs (34.x/35.x), DigitalOcean droplets (159.89.x, 68.183.x, 165.22.x), Vietnamese CDNs (`vncdn.net`, `cmccloud.com.vn`, `kunlunsl.com`, `dnsv1.com`-style), `ladipage.com`.
  - 2 malformed records found (`_f106b55386...test.monkeyuni.net.monkeyuni.net.` and `_557e480585...api.monkeyuni.net.monkeyuni.net.` — double `.monkeyuni.net` suffix, likely a udag data-entry bug from old cert validation) — flag for the user to confirm dead before deciding whether to replicate as-is or drop.
  - ACM-validation CNAMEs (AWS `*.acm-validations.aws.` + Sectigo/Comodo `*.comodoca.com.`) — must preserve exactly, cert-renewal-dependent, not visual noise.
- [x] Generate `cloudflare_record` Terraform resources directly from the real zone export (`modules/cloudflare-records`, `envs/dns-cloudflare/records/parsers/parse_udag_bind_export.py`).
- [x] Registrar access + Cloudflare account/API token — both resolved; token stored at `~/Workspaces/.vault/cf`, `CLOUDFLARE_API_TOKEN`/`CLOUDFLARE_ACCOUNT_ID` set in GitHub Actions.
- [x] `cloudflare` Terraform provider + `envs/dns-cloudflare` (own state/backend, own CI roles). Zone creation is Terraform-managed (`cloudflare_zone`, `jump_start = false` — see below for why).
- [x] All 140 records from the real export created and verified live, DNS-only, matching TTLs from source. `terraform plan` shows `No changes` against real Cloudflare state.
- [x] SPF fix applied and live (`include:_spf.google.com` added).
- [x] Bootstrap reconciliation done: Cloudflare's zone-creation auto-scan (`jump_start` defaults `true`) raced Terraform's own record creation, causing shifting "already exists" conflicts across repeated applies. Fixed by setting `jump_start = false` (delete+recreate the zone — safe, never NS-delegated) and importing the handful of records auto-scan still created before the flag took effect. Also found and deleted a duplicate old SPF record the auto-scan had created (RFC 7208 disallows multiple SPF records — would have defeated the fix) plus 2 harmless duplicates, none of which were ever in Terraform's declared config.
- [x] Fixed a real bug found via `dns-cloudflare-plan.yml`'s first live CI run: `grant_aws_infra_permissions=false` had incorrectly gated `iam:ListOpenIDConnectProviders`/self-role-read too — those are needed unconditionally by every `ci-role` instance, not just AWS-infra-managing ones. Both `prod` and `dns-cloudflare` now show `No changes` in real CI, not just locally.
- [ ] Lower TTLs at udag ahead of cutover, so a rollback propagates fast. **Not yet done — next actionable step.**
- [ ] Cut over: point registrar NS to Cloudflare's assigned nameservers (`selah.ns.cloudflare.com`, `yoxall.ns.cloudflare.com`) via United Domains' "Own nameservers" panel.
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
