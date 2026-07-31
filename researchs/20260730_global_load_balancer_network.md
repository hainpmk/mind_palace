# Global Load Balancer Network Architectures: Why NLB Broke ISP-Specific WebSocket Connectivity

Context: `edu_mispronunciation_detection` (MSPEAK) backend, previously fronted by GCP's Global Load Balancer, migrated to AWS EKS behind an ingress-nginx Service exposed via NLB. Some client apps (notably BestHTTP/Unity clients), on specific ISPs/areas, could not establish `wss://` connections to `mspeak2.monkeyenglish.net`.

## Findings: GCP GLB vs. AWS NLB

**GCP Global Load Balancer (previous setup):**
- One **anycast IP**, announced via BGP from hundreds of Google PoPs worldwide simultaneously.
- Client's ISP routes to the topologically closest/healthiest PoP; if a path degrades, BGP convergence shifts traffic elsewhere — usually transparent to the client.
- TLS/HTTP terminates **at the edge PoP**, often inside or very close to the ISP's own network (Google has extensive direct peering, including in-country caches). The uncontrolled public-internet hop is short. From the edge, traffic rides Google's private global backbone to the backend region.
- L7 — understands HTTP/WebSocket semantics, can retry/reroute at the request level.

**AWS NLB (current setup at time of investigation):**
- **Regional, unicast.** Confirmed via `aws elbv2`: exactly 2 static public IPs, one per AZ (`ap-southeast-1a`/`1b`), with cross-zone load balancing disabled (`load_balancing.cross_zone.enabled: false`).
- No edge network — every client's raw TCP/TLS packets must cross the entire public internet path from their ISP to Singapore.
- L4 only — pure TCP passthrough, no HTTP awareness, no request-level retry or failover based on path quality.
- If a specific ISP's transit/peering to `ap-southeast-1` (or specifically to one of the 2 published IPs) is congested, rate-limited, or blackholed, there is **no alternate path** — unlike anycast, that IP is reachable exactly one way.
- Regional "datacenter" IP ranges (like AWS's) are also more often subject to ISP-side reputation-based throttling/filtering than edge/anycast ranges that ISPs treat as trusted eyeball-adjacent infrastructure.

This mismatch — anycast+edge-terminated L7 (GCP GLB) replaced by unicast regional L4 (AWS NLB) — is a strong structural explanation for symptoms isolated to "specific ISP + specific area," since it points at an ISP-to-AWS-region peering/BGP problem rather than general service unavailability.

## ALB as a partial fix

Added an ALB (`alb-k8s-ingress`) in front of the NLB (chained via an IP target group pointed at the NLB's private ENI IPs) to get correct L7 handling and a larger, more elastic IP pool than the NLB's fixed 2.

Root cause of the initial ALB health check failure: the first target group was created as `Protocol=HTTP, Port=80` with `HealthCheckProtocol=HTTPS` — a protocol/port mismatch causing the ALB to send a TLS ClientHello to the NLB's plain-HTTP listener. Fixed by creating a new target group `Protocol=HTTPS, Port=443` (matching the NLB's HTTPS:443 listener → ingress-nginx's TLS listener), with health check matcher widened to `200-499` since ALB target group health checks can't send a custom `Host` header and will land on nginx's default backend (expect 404, not 200).

Also flagged: had the old HTTP:80 target group been used for live traffic, `edu-ai` ingress's `ssl-redirect: true` annotation would have caused nginx to 301-redirect every request (including WS upgrade attempts) — breaking WebSocket handshakes for real clients, not just health checks.

Verified end-to-end via raw TLS + HTTP Upgrade request (`openssl s_client` + manual `GET /ws/v4/... Upgrade: websocket` request) against both the ALB's own DNS name and, after cutover, the real `mspeak2eks.monkeyenglish.net` hostname — both returned `101 Switching Protocols` and the app's welcome frame.

**Caveat:** the ALB is still purely regional (`ap-southeast-1` only) — it improves protocol correctness and IP-pool diversity over the NLB, but does not replicate GCP GLB's anycast+global-backbone model. If the root cause is ISP-to-Singapore peering specifically, the ALB alone may reduce but not eliminate the problem.

## Comparing true anycast/backbone options

| | Anycast edge | Edge→origin path | Notes |
|---|---|---|---|
| GCP GLB (previous) | Yes, ~global PoPs | Google private backbone | What was previously relied on, implicitly |
| AWS ALB | No (regional only) | N/A | Current interim fix; solves protocol bugs, not ISP peering |
| AWS Global Accelerator | Yes, 2 static anycast IPs from all AWS edge locations | AWS private backbone | Closest AWS-native analog to GCP GLB |
| AWS CloudFront | Yes, edge PoPs | AWS backbone from edge | Also supports WebSocket now |
| Cloudflare (proxied DNS / LB) | Yes, anycast from 300+ PoPs, very strong ISP peering incl. in Vietnam/SEA | **Public internet by default**; private backbone only via paid **Argo Smart Routing** add-on | WebSocket supported on all plans by default |

Cloudflare's anycast+edge-peering leg is likely the most relevant fix for this symptom, since the failures are ISP/client-side, not backend-side — Cloudflare's edge absorbs the bad-routing leg before traffic ever needs to cross to Singapore. The edge→origin leg (Cloudflare → AWS ALB) rides the public internet by default unless Argo is enabled, but this leg is between well-peered infrastructure and is not where the observed failures occur.

## Decision: Cloudflare should sit in front of the NLB, not the ALB

**Recommendation: point Cloudflare directly at the NLB, bypassing the ALB.**

Reasoning:
1. **The NLB→nginx chain was never actually broken.** The ISP-specific failures only ever affected *some* clients — most traffic through `mspeak2.monkeyenglish.net` (still pointed straight at the NLB) works fine. That means NLB:443 → nginx's TLS termination + Host-based routing + WebSocket upgrade is already correct. The ALB was introduced specifically to compensate for the NLB's *reachability* limitation (2 static regional IPs, no anycast) — not because the NLB mishandles WS or TLS.
2. **Cloudflare replaces exactly the thing the ALB was compensating for.** The ALB's main value was "more/varied entry IPs than the NLB's fixed 2." Once Cloudflare's anycast edge is the actual internet-facing endpoint, clients never touch AWS's regional IPs at all — Cloudflare's own infra makes the outbound connection to the origin, and Cloudflare's data centers have well-provisioned connectivity to AWS regions (not the same risk category as an arbitrary residential/mobile ISP's peering to Singapore). The ALB stops adding anything once CF is in front.
3. **Fewer hops, less cost, less to maintain.** CF → ALB → NLB → nginx has one more internal proxy layer than CF → NLB → nginx, for no remaining benefit — extra ALB LCU cost and latency with nothing to show for it.
4. **The one tradeoff:** AWS WAF only attaches to ALB/CloudFront, not NLB. Cloudflare's own WAF/DDoS protection covers that role instead, so this isn't a real gap while Cloudflare is the edge.

The ALB built during this investigation stays as a validated, ready-to-use fallback (e.g. if AWS-native WAF or dropping Cloudflare is ever needed later), but does not need to be in the live path.

## Cloudflare setup plan (compute/backend stays on AWS)

Scope: Cloudflare is added purely as the anycast edge/CDN layer in front of the existing AWS stack. **All backend compute (EKS, Triton on GCP, Mongo, Redis, etc.) stays exactly where it is** — this is a DNS/edge-routing change only, not a migration.

Planned setup:
- Add the domain (or just the relevant subdomains, e.g. `mspeak2.monkeyenglish.net`) to Cloudflare, DNS record **proxied** ("orange-clouded") pointing at the **NLB's hostname** (`k8s-ingressn-ingressn-...elb.ap-southeast-1.amazonaws.com`), per the decision above.
- SSL/TLS mode: **Full (Strict)** — Cloudflare validates the origin's real certificate; nginx already serves a valid cert for the relevant hosts via ingress TLS/SNI, so no cert changes needed on the AWS side.
- WebSocket support: on by default on all Cloudflare plans, no extra config needed for the `/ws/v1..v4/{device_id}` routes.
- No "Cloudflare Load Balancer" product needed initially — single origin (the NLB), so a plain proxied DNS record is sufficient. Revisit only if multi-origin failover/steering becomes a requirement.
- Optional follow-up: enable **Argo Smart Routing** if the Cloudflare-edge→origin leg is ever suspected to be a bottleneck (not expected to be, based on current evidence).
- Migration approach: mirror the canary pattern already used for the ALB — cut over `mspeak2eks.monkeyenglish.net` first, verify with affected ISPs/regions, then cut `mspeak2.monkeyenglish.net`.
- AWS-side NLB/ingress-nginx chain remains unchanged and continues to do TLS termination + protocol-correct routing into the cluster; Cloudflare only replaces "how the client reaches AWS."

### Open consideration: fresh domain + registrar on Cloudflare

Currently weighing starting fresh rather than migrating the existing zone: the current DNS registrar is old/slow and the existing zone (`monkeyenglish.net`) carries a large number of unrelated subdomains (see the full ingress list surveyed during this investigation — `app`, `crm`, `class`, `cms`, `hoc10`, etc.), making an in-place migration to Cloudflare-as-registrar riskier and slower than necessary for what is fundamentally a fix scoped to the `mspeak2*` hosts.

Option being considered: register a **new domain**, put it on Cloudflare (DNS + registrar), issue fresh certs there, and move only the MSPEAK/pronunciation-assessment traffic to it — rather than migrating the entire legacy `monkeyenglish.net` zone. Tradeoffs to weigh before deciding:
- Pro: clean slate, no risk to the ~40+ unrelated existing records/subdomains during migration; Cloudflare-as-registrar avoids the old registrar's slow propagation/change process entirely for this service.
- Con: client apps (BestHTTP/Unity) need to be updated to point at the new hostname — this isn't a transparent infra swap, it requires a client-side release/rollout.
- Con: loses the existing `mspeak2`/`mspeak2eks`/`mspeak3` naming continuity and any hardcoded references (certs, mobile app configs, monitoring) tied to the current domain.
- Not yet decided — needs a decision on rollout mechanism (can client apps consume a new host via remote config without an app-store release, or does it require a hard client update).
