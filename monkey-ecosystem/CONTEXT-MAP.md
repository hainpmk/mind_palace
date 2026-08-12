# Monkey Ecosystem — Context Map

Discovery-stage map of backend/product repos under `eduhub123`. One row per repo
(the context-boundary unit for this map — see rationale below). This is **not**
a claim about bounded-context boundaries yet; a repo may turn out to bundle
several domain concepts, or several repos may turn out to be one context split
across services. Boundaries sharpen as each repo gets its own `CONTEXT.md`.

## Conventions

- **Prefix** — the repo's naming family, the only grouping signal we currently
  trust. `edu_*`, `mk_*`, and `hoc10-*` are confirmed distinct product lines
  within the same ecosystem; whether/how they relate to each other is an open
  question, not yet resolved.
- **Status** — `confirmed` (a human or code statement ties it to a named
  product) vs. `unconfirmed` (name/prefix suggests ecosystem membership, not
  yet verified).
- **Purpose (guessed)** — first line pulled from the repo's README, or a
  short note where the README was empty/boilerplate. Not verified against
  code.
- Excluded from this table entirely: all `infra_*` deployment-mirror repos
  (infra concerns live in `bnn`, not here), and generic tooling/boilerplate/
  library/data-infra repos (`docker_image_base`, `angular-schema-form`,
  `angular2-json-schema-form`, `bootstrap-growl`, `nextjs-boilerplate`,
  `ng2-toasty`, `k8s_config`, `kube-prometheus-stack`, `monitoring`,
  `infra_kong`, `iac`, `template-be-laravel10`, `pptx-viewer`,
  `telegram-notifications-plugin`, `wiki_forum`, `api_document`,
  `cryptojs-aes-php`, `aws-sdk-php`, `PhysicsDemo.spritebuilder`,
  `StreamingKit`, `IABReceiptVerification`, `IABV3Example`,
  `nim`, `bbb-greenlight`, `bigbluebutton`, `ETL_Data`).

## `edu_*` line — Monkey Junior ecosystem (confirmed)

| Repo | Status | Purpose (guessed) |
|---|---|---|
| `edu_app` | confirmed | Manages user account information within the Monkey ecosystem (register/login/profile/license/orders). Named as the account hub. |
| `edu_device` | confirmed | **Confirmed: exactly 2 contexts bundled in one repo, historical not principled** (confirmed by user; full controller/model survey done). (1) **Device Identity & Local State** — device fingerprint, geo/IP/timezone, pre-login coins wallet, FCM push, "convert device to account" (device→account linking, mirrors `edu_app`'s Merge Account). (2) **In-App Purchase & Payment Verification** — Apple/Google StoreKit receipt verification, refunds, subscriptions, exchange rates, promo-offer signing, purchase event/audit logs. Named connection to MJ; called from `edu_app` via `DeviceConnectService`. |
| `edu_app_platform_go` | confirmed | Go service for the content-catalog/"Platform" domain (lesson/game/word/worksheet/story/activity — matches `edu_app_v2`'s `Platform/*` namespace and `edu_lesson`). Also actively growing a new capability: AI-generated story content (Gemini/OpenAI provider-agnostic, per `docs/superpowers/specs/2026-06-12-story-ai-multi-provider-design.md`) — not just a legacy rewrite. DB connection not confirmed from committed config (env-injected). Named connection to MJ. |
| `edu_app_v2` | confirmed | **Not a "v2" successor — a live, actively-committed monolith sharing the SAME databases** (`edu_app`, `edu_device`, `edu_global`, `edu_platform`, `edu_story` — confirmed via `config/database.php` connection names) as the split-out `edu_app`/`edu_device`/`edu_story`/etc. microservices. Bundles `Story`/`Device`/`Award`/`Crm`/`Platform` namespaces all in one repo — looks like the original monolith these were later split from, but the split was never fully cut over: both read/write the same tables concurrently, not just call each other over HTTP. Last commit 2026-07-20, not frozen. |
| `edu_app_story_go` | unconfirmed | Purpose undocumented (empty README); name suggests Monkey Stories tie-in. |
| `edu_app_tini` | unconfirmed | "Mini App Monkey Tiki" — unclear relation to Monkey Junior line. |
| `edu_agent_app` | unconfirmed | Node.js project, purpose undocumented beyond prerequisites. |
| `edu_api_face_recognition` | unconfirmed | Face recognition API using Qdrant vector database. |
| `edu_auth` | unconfirmed | Lumen PHP framework base — likely the auth service, README is unedited boilerplate. |
| `edu_award` | unconfirmed | Award service ("for optimize load"), REST API, Lumen. |
| `edu_backend_product` | unconfirmed | Purpose undocumented (empty README). |
| `edu_campaign` | unconfirmed | Purpose undocumented beyond name. |
| `edu_cms` | unconfirmed | "Project api config data CMS MS" — content management. |
| `edu_cms_frontend` | unconfirmed | Angular frontend for CMS. |
| `edu_cms_frontend_yolo` | unconfirmed | Angular frontend, likely a variant/experiment of `edu_cms_frontend`. |
| `edu_crm` | unconfirmed | CRM backend, purpose undocumented beyond setup steps. |
| `edu_crm_accountant` | unconfirmed | CRM variant for accountant role. |
| `edu_crm_frontend` | unconfirmed | Angular frontend for CRM. |
| `edu_crm_frontend_accountant` | unconfirmed | Angular frontend for CRM, accountant role. |
| `edu_data_process` | unconfirmed | Data processing functions (Vietnamese README, "Các chức năng chính"). |
| `edu_developer` | unconfirmed | Purpose undocumented (empty README). |
| `edu_learn_report` | confirmed | "MJ 4.0 parent reporting" — explicitly Monkey Junior. |
| `edu_lesson` | unconfirmed | Lesson service ("for optimize load"), REST API, Lumen. |
| `edu_lms` | unconfirmed | Purpose undocumented (empty README); name suggests Learning Management System. |
| `edu_lms_frontend` | unconfirmed | React (CRA) frontend for LMS. |
| `edu_log` | unconfirmed | "Project Log History Management." |
| `edu_mail` | unconfirmed | README title says "go_crm" — mismatched/stale README, purpose unclear. |
| `edu_mailsms` | unconfirmed | Lumen PHP framework base, unedited boilerplate — likely mail/SMS service. |
| `edu_media` | unconfirmed | Media service, REST API, Lumen. |
| `edu_mispronunciation_detection` | confirmed | "MSPEAK v5.0" pronunciation assessment system, explicitly "designed for Monkey Junior." |
| `edu_okr` | unconfirmed | OKR service, REST API, Lumen. |
| `edu_okr_frontend` | unconfirmed | Frontend for OKR, purpose undocumented beyond name. |
| `edu_pbi_reports` | unconfirmed | "repo for DA team" — Power BI reports likely. |
| `edu_personalize_learning` | unconfirmed | Lesson review timing prediction (Ebbinghaus forgetting curve) — no explicit product tie yet, but clearly learning-domain. |
| `edu_platform` | unconfirmed | "Project api platform" — purpose undocumented beyond name. |
| `edu_product` | unconfirmed | Purpose undocumented (empty README). |
| `edu_scheduler_notifier` | unconfirmed | LLM-powered notification system, Go + Python. |
| `edu_share_photo` | unconfirmed | Backend for an Android photo-sharing app. |
| `edu_story` | unconfirmed | README title says "lesson service" (likely copy-pasted from `edu_lesson`) — name suggests Monkey Stories. |
| `edu_ticket` | unconfirmed | Lumen PHP framework base, unedited boilerplate — likely support ticketing. |
| `edu_tutoring` | unconfirmed | React (CRA) frontend, purpose undocumented beyond name. |
| `edu_user` | confirmed | "Golang CQRS microservices." README/go.mod mention Kafka, but it's dead code (unimported); the broker actually wired in (`core/config`) is RabbitMQ. Mirrors `edu_app`'s entire `Users` shape (`max_device_on_active`, `max_profile`, `classification`, `merged_user_id`, `user_lms_id`, `is_user_app`) — likely a CQRS read-model/rewrite fed by `edu_app` events over RabbitMQ. Confirmed connection point to `edu_app`. |
| `edu_video` | unconfirmed | Purpose undocumented (empty README). |
| `edu_web_ai` | unconfirmed | Next.js/React project, purpose undocumented beyond stack. |
| `edu_websocket` | unconfirmed | Purpose undocumented (empty README). |
| `edu_web_v2` | unconfirmed | Purpose undocumented (empty README). |

## `mk_*` line — Monkey Class / Monkey Kindy (confirmed, connected to `edu_app`)

Monkey Class is a management solution for preschools; Monkey Kindy is a preschool
English programme offered either inside Monkey Class or inside the Monkey Junior
super-app. Confirmed connections: (1) `mk_user`'s `students` table links to
`edu_app`'s `profile` table (type `LMS`/`LMS_TEACHER`) — see `edu_app/CONTEXT.md`
→ **MX↔LMS Sync**. (2) `edu_app`'s `PromotionController::addGiftMClass` grants a
free 1-month MJ package as a Monkey Class cross-sell gift — see `edu_app/CONTEXT.md`
→ **"Promotion"/"Gift"**. (3) `edu_app`'s `NotificationController::challengeMC`
surfaces Monkey Class "challenge" notifications per Profile — see
`edu_app/CONTEXT.md` → **Challenge (MC)**. (4) `edu_app`'s
`TicketController::sendTicket` (misleadingly named — a CRM sales-outreach
trigger, not support) flags paying MJ customers not yet linked to a "Monkey
School" for cross-sell outreach into Monkey Class — see `edu_app/CONTEXT.md`
→ **Ticket**.

| Repo | Status | Purpose (guessed) |
|---|---|---|
| `mk_classroom` | unconfirmed | README titled `mk_classroom_2` — manages classroom and school chat groups. |
| `mk_classroom_2` | unconfirmed | Purpose undocumented beyond title match with `mk_classroom`. |
| `mk_classroom_go` | unconfirmed | "mk_class_go - Production Ready Backend" — likely a Go rewrite of `mk_classroom`. Connects to a MySQL DB named `edu_app_3` per `.env.example` — likely a replica/shard of `edu_app`'s own database. |
| `mk_course` | unconfirmed | Lumen PHP framework base, unedited boilerplate. |
| `mk_course_go` | unconfirmed | "mk_course_go - Production Ready Backend" — likely a Go rewrite of `mk_course`. |
| `mk_customer_web` | unconfirmed | Next.js frontend, purpose undocumented beyond stack. |
| `mk_user` | confirmed | Owns the Kindy/Class `Student` entity (own `school_id`, `user_id`, `profile_id`), with a `profile_mx` column linking a Student to its mirrored `edu_app` MX Profile. Confirmed connection point to `edu_app`. |
| `mk_fee_feature` | unconfirmed | Purpose undocumented (empty README); name suggests billing/fees. |
| `mk_document` | unconfirmed | README is generic "Template BE Laravel 10" — likely unedited boilerplate. |

Note: `mk_classroom`/`mk_classroom_go` and `mk_course`/`mk_course_go` each look
like plain-stack + Go-rewrite pairs, mirroring the same pattern as
`edu_app`/`edu_app_v2`. Not merged in this table — worth confirming during
grilling.

## `hoc10-*` line — Hoc10, discontinued but still running (relation to rest of ecosystem unknown)

Electronic version of the "Kite textbook" plus add-ons. No code reference to
`edu_app` (or any other `edu_*`/`mk_*` repo) found — relation, if any, to the
rest of the ecosystem is unconfirmed.

| Repo | Status | Purpose (guessed) |
|---|---|---|
| `hoc10-cms-frontend` | unconfirmed | React (CRA) frontend for a CMS. |
| `hoc10-frontend` | unconfirmed | Next.js frontend, purpose undocumented beyond stack. |
| `hoc10-games` | unconfirmed | Purpose undocumented (empty README); name suggests learning games. |
| `hoc10-question-service` | unconfirmed | Purpose undocumented (empty README); name suggests a question/quiz backend. |

## Ungrouped singles — ecosystem membership unconfirmed

| Repo | Status | Purpose (guessed) |
|---|---|---|
| `live_score` | unconfirmed | Backend for a "Live Scores" Flutter app, modular monolith (FastAPI). |
| `tutoring_phhs` | unconfirmed | Purpose undocumented (empty README); name suggests tutoring product. |
| `tutoring-phhs-web` | unconfirmed | Next.js frontend, paired with `tutoring_phhs`. |
| `Monkey_Crawler` | unconfirmed | Purpose undocumented beyond name. |
| `MonkeyXAssetBunldeBuilder` | unconfirmed | Purpose undocumented (empty README); name suggests an asset-bundling build tool. |
| `agent-mcp-data360` | unconfirmed | "Data 360 dictionary" — MCP server + JSON knowledge base for answering stakeholder business questions. |
| `service_agent` | unconfirmed | REST API ("for optimize load"), Lumen — purpose beyond that undocumented. |
| `python_convert_bundle` | confirmed | Tool to package learning materials — likely cross-product (not tied to one product line). |

## Open questions

- **The "one context = one repo" starting assumption is confirmed broken for
  at least one pair.** `edu_app_v2` shares its actual database tables
  (`edu_app`, `edu_device`, `edu_global`, `edu_platform`, `edu_story`) with
  `edu_app`/`edu_device`/etc. — both are live, both actively committed to.
  Any repo pair sharing a DB-name prefix (`edu_app*`, `mk_classroom*`/
  `mk_classroom_go`, `mk_course*`/`mk_course_go`) should be treated as
  suspect for the same pattern until checked, not assumed to be sequential
  versions.

- ~~How do `mk_*` and `hoc10-*` relate to the `edu_*` (Monkey Junior) line?~~
  Resolved for `mk_*`: Monkey Class is a preschool management solution; Monkey
  Kindy is a preschool English programme offered inside Monkey Class or inside
  the Monkey Junior super-app. `edu_app`'s `profile` table is a shared store —
  `profile.type = PROFILE_LMS`/`PROFILE_LMS_TEACHER` are Kindy student/teacher
  profiles, kept in sync with `mk_user`'s own `students` table by phone number
  (VN-only). See `edu_app/CONTEXT.md` → **MX↔LMS Sync**.
  `hoc10-*` remains unresolved: Hoc10 is an electronic "Kite textbook" +
  add-ons product, discontinued but still running; no code reference to
  `edu_app` found, and its relationship (if any) to the rest of the ecosystem
  is unknown.
- Several `edu_*` repos have empty/boilerplate READMEs (`edu_backend_product`,
  `edu_developer`, `edu_product`, `edu_video`, `edu_websocket`, `edu_web_v2`,
  `edu_lms`) — purpose entirely unconfirmed, name-only.
- `_go` twin pairs (`mk_classroom`/`mk_classroom_go`, `mk_course`/`mk_course_go`)
  and version pairs (`edu_app`/`edu_app_v2`, `edu_cms_frontend`/
  `edu_cms_frontend_yolo`) — likely old-stack/rewrite relationships, not
  yet confirmed which is live.

## Next step

Deep-dive `edu_app` first (flagged as the account/identity hub other repos
plug into) — build its own `CONTEXT.md` under `edu_app/`, starting with
account/user/license vocabulary.
