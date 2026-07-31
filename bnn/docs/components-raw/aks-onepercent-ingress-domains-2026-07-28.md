# Ingress domains — onepercent-aks-v2 (2026-07-28)

Full `kubectl get ingress -A` dump from the `onepercent-aks-v2` cluster (nonprod). Two ingress controllers, two public IPs — see [ADR-0003](../adr/0003-cluster-topology-and-per-cluster-argocd.md) for the cluster's role. Every host below currently resolves to one of these two Azure Standard LB IPs and has no CDN/WAF/Front Door in front — all of it needs a DNS cutover (and likely a re-evaluation of the front-door gap) as part of the AWS migration.

- **Kong** (`172.188.210.26`) — 6 hosts, all `*-gateway` API entrypoints (routes to multiple backend services per domain area)
- **nginx-ingress** (`57.155.217.220`) — 34 hosts, one-ingress-per-service direct routing

| Namespace | Ingress | Class | Host | Backend service(s) | LB IP |
|---|---|---|---|---|---|
| app | app-gateway | kong | appdev.monkeyenglish.net | edu-app-v2-dev, edu-app-dev, edu-device-dev, edu-award-dev, edu-lesson-dev, edu-story-dev, edu-app-story-go-dev-edu-story-go, edu-app-platform-go-dev-edu-platform-go, edu-user-writer-dev, edu-user-reader-dev, data-report-dev | 172.188.210.26 |
| app | edu-app-v2-dev | nginx | api.dev.monkeyuni.net | edu-app-v2-dev | 57.155.217.220 |
| argocd | argocd-server-ingress | nginx | cdaz.monkey.edu.vn | argocd-server | 57.155.217.220 |
| class | class-gateway | kong | apiclassdev.monkey.edu.vn | mk-classroom-dev-2, mk-classroom-go-dev, mk-course-dev, mk-course-go-dev, mk-user-dev, edu-websocket-dev | 172.188.210.26 |
| class | mk-classroom-dev-2 | nginx | devclassroom2.monkey.edu.vn | mk-classroom-dev-2 | 57.155.217.220 |
| class | mk-customer-web-dev | nginx | classdev.monkey.edu.vn | mk-customer-web-dev | 57.155.217.220 |
| class | mk-customer-web-dev-2 | nginx | class2dev.monkey.edu.vn | mk-customer-web-dev-2 | 57.155.217.220 |
| cms | cms-gateway | kong | apicmsdev.monkeyenglish.net | edu-cms-dev, edu-platform-dev, edu-story-dev, edu-video-dev-writer, edu-video-dev-reader, edu-product-dev | 172.188.210.26 |
| cms | edu-cms-dev | nginx | cmsms.dev.monkeyuni.net | edu-cms-dev | 57.155.217.220 |
| cms | edu-cms-frontend-dev | nginx | cms.dev.monkeyuni.com | edu-cms-frontend-dev | 57.155.217.220 |
| cms | edu-video-dev-reader | nginx | video.dev.monkeyuni.net | edu-video-dev-reader | 57.155.217.220 |
| cms | edu-video-dev-writer | nginx | video.dev.monkeyuni.net | edu-video-dev-writer | 57.155.217.220 |
| crm | crm-gateway | kong | apicrmdev.monkeyenglish.net | edu-ticket-dev, edu-okr-dev, edu-campaign-dev, edu-log-dev, edu-crm-accountant-dev | 172.188.210.26 |
| crm | edu-campaign-dev | nginx | campaign.dev.monkeyuni.net | edu-campaign-dev | 57.155.217.220 |
| crm | edu-campaign-frontend-dev | nginx | campaigndev.monkey.edu.vn | edu-campaign-frontend-dev | 57.155.217.220 |
| crm | edu-crm-frontend-accountant-dev | nginx | accountant.dev.monkey.edu.vn | edu-crm-frontend-accountant-dev | 57.155.217.220 |
| crm | edu-okr-dev | nginx | okr.dev.monkeyuni.net | edu-okr-dev | 57.155.217.220 |
| crm | edu-ticket-dev | nginx | ticket.dev.monkeyuni.com | edu-ticket-dev | 57.155.217.220 |
| dev | dev-edu-tutoring | nginx | tutoring.dev.monkeyuni.net | dev-edu-tutoring | 57.155.217.220 |
| dev | edu-auth-dev | nginx | auth.dev.monkeyuni.com | edu-auth-dev | 57.155.217.220 |
| dev | edu-crm-dev | nginx | crm.dev.monkeyuni.net | edu-crm-dev | 57.155.217.220 |
| dev | edu-crm-dev | nginx | crmdev.monkeyuni.net | edu-crm-dev | 57.155.217.220 |
| dev | edu-crm-frontend-dev | nginx | crm.dev.monkeyuni.com | edu-crm-frontend-dev | 57.155.217.220 |
| dev | edu-develop-dev-edu-developer | nginx | apiv2.dev.monkeyuni.net | edu-develop-dev-edu-developer | 57.155.217.220 |
| dev | edu-lms-dev | nginx | lms.dev.monkeyuni.net | edu-lms-dev | 57.155.217.220 |
| dev | edu-lms-frontend-dev | nginx | dev.hoc10.vn | edu-lms-frontend-dev | 57.155.217.220 |
| dev | edu-mailsms-dev | nginx | email.dev.monkeyuni.com | edu-mailsms-dev | 57.155.217.220 |
| dev | edu-media-dev | nginx | media.dev.monkeyuni.net | edu-media-dev | 57.155.217.220 |
| dev | edu-web-ai-dev | nginx | ai.beta.monkey.edu.vn | edu-web-ai-dev | 57.155.217.220 |
| dev | edu-web-dev | nginx | beta.monkey.edu.vn | edu-web-dev, edu-phhs | 57.155.217.220 |
| dev | question-service-dev | nginx | question.dev.monkeyuni.net | question-service-dev | 57.155.217.220 |
| dev | service-agent-dev | nginx | agent.dev.monkeyuni.net | service-agent-dev | 57.155.217.220 |
| dev | service-agent-frontend-dev | nginx | affiliatedev.monkey.edu.vn | service-agent-frontend-dev | 57.155.217.220 |
| dev | tutoring-phhs-web-dev-edu-phhs | nginx | devmonkeytutoring.monkey.edu.vn | tutoring-phhs-web-dev-edu-phhs | 57.155.217.220 |
| kong | kong-proxy-ingress | (none) | testkong.monkey.edu.vn | kong-gateway-proxy | 172.188.210.26 |
| stg | app-gateway | kong | stg.monkeyenglish.net | edu-cms-stg, edu-platform-stg, edu-platform-go-stg, edu-story-go-stg, edu-product-stg, edu-lesson-stg, edu-app-stg, edu-story-stg | 172.188.210.26 |
| stg | edu-cms-frontend-stg | nginx | cmsstgmx.dev.monkeyuni.com | edu-cms-frontend-stg | 57.155.217.220 |
| stg | edu-cms-frontend-stg-dev | nginx | cmsstgmx.stgdev.monkeyuni.com | edu-cms-frontend-stg-dev | 57.155.217.220 |
| stg | edu-story-stg | nginx | storystg.monkeyuni.net | edu-story-stg | 57.155.217.220 |

**Domain count by root**: `monkeyuni.net` (13), `monkeyuni.com` (5), `monkey.edu.vn` (7), `monkeyenglish.net` (5), `hoc10.vn` (1) — 4 distinct root domains need DNS cutover planning, not just subdomain-level changes.

**Open items for migration**:
- No CDN/WAF/Front Door in front of either LB today (confirmed via full-subscription Azure sweep) — decide whether the AWS target adds one, independent of the lift-and-shift of the ingress layer itself.
- `edu-media-dev` has a rule with no `host` set (catch-all) — check what traffic this actually serves before assuming it's droppable.
- **Confirmed 2026-07-28**: checked every Kong-gateway backend Service for existence + ready endpoints (`kubectl get svc`/`endpoints`). Several are dead routes today, not just unconfirmed:
  - Service doesn't exist at all: `app/edu-story-dev`, `cms/edu-platform-dev`, `dev/edu-phhs`
  - Service exists, zero ready pods: `app/edu-app-story-go-dev-edu-story-go`, `class/mk-user-dev`, `cms/edu-cms-dev`, `crm/edu-okr-dev`, `crm/edu-crm-accountant-dev`
  - The other 26 of 34 Kong backend refs have live, ready pods.
  - Net effect: `apiclassdev.monkey.edu.vn`, `apicmsdev.monkeyenglish.net`, `apicrmdev.monkeyenglish.net`, and `appdev.monkeyenglish.net` each route some paths to dead backends right now — fix/prune before migrating these routes to AWS rather than carrying broken routes over as-is.
