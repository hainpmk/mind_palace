# Investigation: `GET /app/api/v5/account/load-update` slowness

- **Date**: 2026-08-01 (instrumentation shipped and root-caused 2026-08-05, delivery fixed and confirmed live 2026-08-06, full instrumentation expansion + second root cause found and fixed 2026-08-07/08, platform-wide blast-radius audit + additional instrumentation 2026-08-09/10)
- **Status**: **Root-caused and largely fixed for `load-update` specifically; turned out to be a platform-wide problem, not an `edu_app`-only one.** Two independent root causes found for `load-update`: (1) DogStatsD delivery gap (fixed 2026-08-06), (2) a dead Sentry APM endpoint on two downstream services adding a fixed ~2s tax to most requests (found + fixed 2026-08-08, see "Second root cause"). v4/v5 average latency dropped ~84-91% after that fix. A 2026-08-09/10 audit, prompted by unrelated Firebase reports on other endpoints, found the **same dead-Sentry-DSN root cause affects at least 13 services platform-wide** (see "Sentry blast-radius audit") — all 5 confirmed same-mechanism services (`edu_story`, `edu_lesson`, `edu_award`, `edu_media`, `mk_course`) now have the **complete** fix (both `SENTRY_TRACES_SAMPLE_RATE=0` and a blanked `SENTRY_LARAVEL_DSN`, closing the request-tax *and* the error-path-tax) rolled out and verified live; 8 more services carry the dead config with narrower (exception-only) exposure, not yet fixed. Separately: a remaining ~500ms gap in v5 is understood to be partly explained by two diagnosed-but-unfixed redundant-fetch bugs in `calculatorPurchasedV2()`'s per-course loop. A real magnitude mismatch between Firebase's client-side p95 (4.54s) and Datadog's server-side timing led to full controller-action + encryption/serialization instrumentation (merged/rolled out 2026-08-10) — **this conclusively ruled out `edu_app`'s own PHP code** (encryption+serialization: <0.5ms; full controller action ≈ inner service call, only ~20ms difference) as the explanation. The remaining ~3.7s gap is now known to live entirely outside `edu_app`, most likely Kong/gateway or client-side SDK measurement — out of scope for further `edu_app` code changes. Two more open, unresolved leads: a recurring off-peak latency swing correlated with (but not conclusively explained by) a brief RDS CPU spike, and `edu_device`/`vnmedia2.monkeyuni.net`'s own unrelated p95 spikes.
- **Author context**: Investigator new to this system; findings below cross-checked against live AWS/EKS state, not just static config

## Correction: Mongo isn't cleanly "still on Azure" — it's a mixed-cloud replica set

Earlier draft of this doc claimed `DB_MONGO_URI`/`DB_MONGO_URI_LESSON` pointed at an Azure host, based on a regex that only captured the first `@host` in the connection string. Re-checked directly on the (single, confirmed) running `edu-app` pod (`edu-app-6d4544548b-xjqf2`, owned by the `edu-app` deployment): the connection string is actually a **replica-set seed list with two hosts**:

```
mongodb://admin:***@mongo.southeastasia.cloudapp.azure.com:27017,ec2-54-179-59-251.ap-southeast-1.compute.amazonaws.com:27017/?replicaSet=atlas-um8iyc-shard-0
```

i.e. one seed still on Azure, one seed on AWS EC2 in `ap-southeast-1`, same replica set (`atlas-um8iyc-shard-0`). No container-level OS env var overrides this (checked — empty), and there's no Laravel config cache (`bootstrap/cache/` doesn't exist) baking in a stale value, so this is the live, effective value. Which physical host actually answers a given query depends on MongoDB's replica-set topology discovery (current primary / nearest secondary per the driver's read preference), not something static config alone resolves — this is consistent with an in-progress or just-completed node-by-node migration of the replica set rather than a single-host cutover.

**Practical implication**: this is exactly why the `datastore` tag on the new metric (below) stays generic (`mongodb`, no `_azure`/`_aws` suffix) — asserting a specific cloud location per call isn't something we can determine from config, and the whole point of shipping this instrumentation is to let real timing data answer that empirically instead.

## Instrumentation added (not yet deployed)

Added a lightweight DogStatsD emitter (`edu_app/service/app/Services/DogStatsdService.php`) — UDP to `DD_AGENT_HOST:DD_DOGSTATSD_PORT` (default port 8125), no new composer dependency. Chosen over installing the `ddtrace` PECL extension because `ddtrace` is **not currently loaded** in the running container (`php -m` confirms it's absent) despite `DD_TRACE_*` env vars being set — installing it is an image/infra change, out of scope for this confirmatory step.

**Socket lifecycle**: one UDP socket per request, not one open/close per metric call. `DogStatsdService` is bound as a container singleton (`AppServiceProvider::register()`), opens its socket once in the constructor (lazily — only the first time something actually resolves the service), and is closed exactly once via `AppServiceProvider::boot()`'s `$this->app->terminating(...)` hook, guarded so it only touches the service if it was actually resolved during the request. Note: this app runs on plain PHP-FPM (no Octane/Swoole), so there's no single long-lived "backend process" spanning multiple requests — each request is its own PHP execution context, so "startup"/"shutdown" here means the start and end of one request's handling, which is the most a non-Octane setup can offer. If this ever moves to Octane/Swoole, the socket would need to survive across requests within a worker instead of reopening each time — worth revisiting then.

Wrapped the highest-confidence external-dependency call sites reached by `refreshData()`, all emitting one metric **`load_update.external_call.duration`** (ms) — renamed from the earlier `load_update.mongo_call.duration` since one of the four call sites is DynamoDB, not Mongo, and the metric needed a name precise enough to cover both. Tagged by `call` and `datastore` so they can be split/compared in Datadog:

| Call site | File | Tag `call` | Tag `datastore` |
|---|---|---|---|
| `userSettingRepository->getByUserID()` (early/version-gated path) | `SyncDataService::syncUser()` | `sync_user.user_setting.probe` | `mongodb` |
| `userSettingRepository->getByUserID()` (fallback path) | `SyncDataService::syncUser()` | `sync_user.user_setting.fallback` | `mongodb` |
| `logsAddDayRepo->getByUserID()` — always executed, not Redis-gated | `PromotionService::addDays()` | `add_days.logs_add_day` | `mongodb` |
| `syncProfileRepository->getByUserID()` (DynamoDB — added as a control/baseline) | `SyncDataService::syncProfileV2()` | `sync_profile_v2.dynamodb` | `dynamodb` |

The `datastore` tag lets us directly compare `mongodb` vs `dynamodb` timing on the same requests once this ships — if Mongo calls run consistently much slower than the DynamoDB baseline, that's strong evidence the replica set is currently being served (for at least some requests) from the Azure-side member or otherwise degraded; if they're comparable, the bottleneck is elsewhere.

Not instrumented in this pass (time-boxed): `PromotionService::checkShowBonusMJ5()`'s deeper calls and `PromotionMSpeakService::addPromotion()`'s conditional `logsAddDayRepo->getByUserAndType()` call — both gated behind business conditions, lower priority than the always-executed paths above.

**Next step**: deploy, let it run against production traffic, then build a Datadog dashboard/query on `load_update.external_call.duration` grouped by `call`/`datastore` to get real numbers. Separately, worth asking the infra owner directly whether the Azure seed in that replica set is meant to still be a live member or is stale/pending removal.

## Datadog metric delivery: root cause found (2026-08-05)

After deploying the instrumentation, **zero data points ever appeared** for `load_update.external_call.duration` (or any suffix/derived metric) in Datadog — not in Metrics Explorer, not in the Custom Metrics usage summary (only 7 custom metrics exist org-wide, none related), not even in the Agent's own live `dogstatsd-stats` view. This took an extensive live debugging pass to root-cause; recording the full chain here so it doesn't need re-discovery.

**Ruled out, in order, each with concrete evidence:**
1. Wrong deployed code — confirmed via `kubectl exec` grep the running pod (`edu-app-7d48545b78-tjdg6`) has the correct instrumented code.
2. Wrong K8s Service routing — confirmed `mx-edu-app` Service selector matches the `edu-app` Deployment's pod labels exactly; Kong's `app-gateway` ingress does route here.
3. No real traffic reaching the endpoint — **false**. Pod stdout access logs (Apache-style, not the app's own `lumen-*.log` which is exception-only) show `/api/v5/account/load-update` (and v2/v7 variants) hit multiple times per second continuously, all `200`, with real multi-KB payloads.
4. Wrong Datadog org / wrong regional site — both explicitly confirmed by the user to match (API key ending `b0dc`, site `app.datadoghq.com`).
5. Datadog custom-metrics governance/allowlist blocking new metric names — checked Org Settings → Data Access Control, no restriction found.
6. Forwarder/backend transmission failing — `agent status` showed `series_v2` with 159,786+ successful transactions, negligible errors (2 total, transient).
7. **A secondary, real-but-not-root-cause finding**: a code-level gate. `SyncDataService::syncUser()` checks Redis (`getSyncUserRedis`) *before* either instrumented Mongo call, and for any user with a fully-synced warm cache (the common case for repeat/active users), **neither wrapped call executes at all** — this is a genuine reason the metric would be rare, but doesn't explain literal zero occurrences across 24h of continuous real traffic, or the fact that raw manual test packets (which bypass this business logic entirely) also never arrived.
8. **Actual root cause**: the cluster's Datadog Agent DaemonSet (deployed via the `DatadogAgent` CRD `datadog/datadog` in namespace `datadog`, managed by the Datadog Operator) exposes its DogStatsD UDP listener as a bare `containerPort: 8125` with **no `hostPort` set**, and `hostNetwork` is not enabled. The agent is therefore only reachable at its own pod IP, never at the node's IP. But every application pod's `DD_AGENT_HOST` is wired via the standard Kubernetes downward-API pattern to the **node's** IP (`status.hostIP`) — which is the officially-documented pattern, but only works when the agent's port has a matching `hostPort`. As a result, **every UDP packet sent by any pod on the cluster following this standard pattern lands on the node's own kernel network stack on a closed port, not on the agent** — it's silently dropped (or occasionally surfaces as a delayed `ECONNREFUSED` on the next `fwrite()`/`socket_sendto()` call, since Linux rate-limits outgoing ICMP "port unreachable" replies, which is why some sends "succeed" with no error yet still never arrive).

**How this was confirmed, concretely:**
- Live-enabled `dogstatsd_metrics_stats_enable` on the (shared, cluster-wide) `DatadogAgent` CRD (`kubectl patch datadogagent -n datadog datadog ...`, rolling-restarted all 5 agent pods) to get a real-time view (`agent dogstatsd-stats`) of exactly what metric names the agent's aggregator is tracking. **Kept enabled** post-investigation (negligible CPU cost, useful going forward) — do not need to re-add it for future debugging.
- Sent manual test packets from `edu-app` (this cluster's node) and independently from `edu-app-v2` (a **different node, different agent pod**) — same `ECONNREFUSED`-alternating failure pattern on both, ruling out "one flaky node/agent."
- Tried both a connected UDP socket (`stream_socket_client(..., STREAM_CLIENT_CONNECT)`, what `DogStatsdService.php` uses) and a raw unconnected `socket_create`/`socket_sendto` (matching how the official `DataDog\DogStatsd` PHP library sends) — **neither reached the agent**, ruling out "wrong PHP socket API" as the cause.
- Directly inspected the agent container: `ss -ulnp` shows it bound to `*:8125` inside its own pod netns (pod IP `10.20.10.151`), while `DD_AGENT_HOST` in application pods is the **node** IP (`10.20.10.149`). Confirmed via `kubectl get pod ... -o json` that the container port spec has no `hostPort` key.
- Confirmed no NetworkPolicy/CiliumNetworkPolicy exists that could otherwise explain a block.

**This means `DogStatsdService.php` and all four wrapped call sites were correct all along** — the app-side DogStatsD implementation is not the bug. **This is a pre-existing, cluster-wide infrastructure gap**, not something introduced by this investigation's changes, and it affects *any* service in the cluster trying to emit custom DogStatsD metrics over UDP via the standard `DD_AGENT_HOST=status.hostIP` pattern — not just `edu_app`. Given the org has only 7 custom metrics total, it's plausible this has silently never worked for anyone.

**Fixed 2026-08-06**: chose option 1 (cluster-wide hostPort fix over per-service UDS migration), since the org's near-zero custom-metrics count suggested other services were likely silently affected the same way, and UDS would have meant fixing services one at a time. Applied via:

```
kubectl patch datadogagent -n datadog datadog --type='merge' \
  -p='{"spec":{"features":{"dogstatsd":{"hostPortConfig":{"enabled":true,"hostPort":8125}}}}}'
```

This is the proper CRD-level fix (not a raw pod-spec patch) — the Datadog Operator regenerates the DaemonSet's pod template from `spec.features.dogstatsd`, and the previous absence of this block meant the operator's default (`hostPortConfig` disabled, favoring UDS) was in effect. The patch triggered a rolling restart of all 5 agent DaemonSet pods (one at a time, ~5 min total). Confirmed post-rollout:
- `hostPort: 8125` present on the `agent` container's `dogstatsdport` in the live pod spec.
- A manual test UDP send that previously failed 100% of the time now succeeds 10/10, with the metric visible immediately in the agent's live `dogstatsd-stats` view.
- `load_update.external_call.duration` confirmed showing up in the Datadog UI (Metrics Explorer) after triggering real-shaped requests against `edu-app`.

**Considered and deferred**: migrating to UDS is still the longer-term Datadog-recommended approach (more reliable, gets automatic origin-detection tagging) — worth revisiting later as a deliberate per-service migration, but not blocking now that the immediate cluster-wide gap is closed.

**`dogstatsd_metrics_stats_enable` was left enabled** on the shared agent config (from the earlier debugging step) — negligible cost, useful for any future DogStatsD delivery debugging via `agent dogstatsd-stats`.

## Summary

Client-side (Firebase Performance) response time for this endpoint is **6.4s over the last 30 days, ~90% slower than before**, correlated with the **Azure → AWS migration on 2026-07-20**.

**Primary hypothesis (medium confidence, needs the new Datadog metric to confirm):** the app's compute moved to AWS (EKS, `ap-southeast-1`), and its MongoDB dependency is a **replica set with mixed seed hosts — one on Azure, one on AWS EC2** (see "Correction" section below) — and this endpoint's code path queries MongoDB on the request's synchronous critical path (on Redis cache miss). If any requests are still being served by the Azure-side member, that's an architectural mismatch introduced by the migration, not a code regression — the code didn't need to change for the symptom to appear. This is now instrumented (see below) rather than purely inferred.

Raw TCP connect latency to the Azure-side seed host from inside the cluster was measured at only ~3–7ms (not itself enough to explain 6.4s), so if that host is involved at all, the mechanism most likely to produce multi-second/20s-class delays is **not simple network RTT** but one of: per-request Mongo auth/TLS handshake cost (no apparent connection pooling across PHP-FPM requests), connection retries/timeouts under load, or intermittent failures at the Azure↔AWS network boundary (firewall/NSG/NAT). This needs the Datadog metric or Mongo-side slow-query logging to fully confirm — flagged as a gap below.

## Evidence

| Fact | Source | Notes |
|---|---|---|
| p90(ish) response time 6.4s, 90% slower than before, over 30d | Reported by investigator (Firebase Performance Monitoring, client-side) | Not server-side timing — includes network/DNS/TLS/gateway time, not just backend processing |
| No server-side latency/breakdown data | Datadog logs (checked with investigator) | Only status + payload logged; no per-call timing. This limits confirmability of the hypothesis below |
| Migration Azure → AWS on 2026-07-20 | Reported by investigator | Working correlation, not yet proven causal |
| `edu-app` deployment running on EKS cluster `monkey-eks`, region `ap-southeast-1` | `kubectl get deploy -n app`, `aws eks list-clusters` (live) | Confirmed AWS compute |
| RDS (MySQL), ElastiCache (Redis), DynamoDB all in `ap-southeast-1` | `aws rds/elasticache/dynamodb` (live) | These dependencies are correctly co-located with AWS compute |
| **`DB_MONGO_URI`/`DB_MONGO_URI_LESSON` are a replica-set seed list with one Azure host (`mongo.southeastasia.cloudapp.azure.com`) and one AWS EC2 host (`ec2-54-179-59-251.ap-southeast-1.compute.amazonaws.com`), same `replicaSet=atlas-um8iyc-shard-0`** | `kubectl exec` into the single running `edu-app` pod (`edu-app-6d4544548b-xjqf2`), read `/var/www/.env` (live, credentials redacted in this doc); confirmed no OS-level env override and no Laravel config cache present | No AWS DocumentDB cluster exists in the account (`aws docdb describe-db-clusters` returned empty) — this is a self-managed/Atlas-style Mongo replica set, not AWS-native. Which seed actually serves a given query depends on replica-set topology discovery, not static config |
| TCP connect to Azure-side Mongo seed: ~3–7ms from the pod; to AWS Redis/MySQL: <1ms | `curl --max-time` connect-time test from inside the `edu-app` pod (live) | Raw network path to the Azure seed specifically is not dramatically slow — points away from pure network distance as the sole cause, if that seed is even the one being used |
| `edu-app-queue` deployment: 0/3 ready, CrashLoopBackOff, 569 restarts over ~8 days | `kubectl get pods -n app` (live) | **Separate issue, not this endpoint's cause** — crash is a PHP `memory_limit` OOM in an unrelated job (`HandleCompleteLessonReview`), confirmed via pod logs. Flagged only because it's a real, currently-broken thing discovered along the way |

## Confirmed request path

```
Client (Firebase-instrumented)
  → Kong (app.monkeyenglish.net/app/api/v5/account/load-update)
    → edu_app service (EKS, ap-southeast-1)
      → Middleware: VerifyTokenApp, ChangeNewFormatPhone
        → v5\LoadUpdateController@refreshData
          → LoadUpdateService::refreshData()
            → ... (see candidates below)
      → (async) job dispatches via RabbitMQ (queue driver = rabbitmq, non-blocking unless broker publish itself hangs)
```

Files:
- Route: `edu_app/service/routes/api_v5.php`
- Controller: `edu_app/service/app/Http/Controllers/v5/LoadUpdateController.php`
- Middleware: `edu_app/service/app/Http/Middleware/VerifyTokenApp.php`
- Core logic: `edu_app/service/app/Services/LoadUpdateService.php` (`refreshData()`, line 186; 2105 lines total, large fan-out)

## Candidates found (grep triage + read of hot files)

### 1. MongoDB queries — ⚠️ top candidate
`refreshData()` calls several services backed by MongoDB repositories (via `mongodb`/`mongodb_lesson` connections, `config/database.php`):
- `SyncDataService::syncUser()` → `UserSettingRepository::getByUserID()` on Redis cache miss (`edu_app/service/app/Services/SyncDataService.php:79-109`)
- `PromotionService` → Mongo `LogsAddDay`, `LogsUserRegister` repositories
- `PromotionMSpeakService` → Mongo `LogsAddDay`, `ProductCondition` repositories

These are gated by Redis cache checks (not every request hits Mongo), but any cache-miss request pays whatever the Azure round trip actually costs. **This is the strongest lead** given the confirmed Azure-hosted Mongo + AWS compute split, but the *mechanism* for a multi-second delay (vs. the observed low raw TCP RTT) is not yet proven — see gaps below.

### 2. `DeviceConnectService::getAppAccountToken()` — low risk
Called unconditionally every request (`edu_app/service/app/Services/LoadUpdateService.php:207`), but checks Redis (`REDIS_APP_ACCOUNT_TOKEN` hash) first; only calls out via HTTP (`CurlService::curlGetData`, 5s timeout) on cache miss. Target: `Config::get('environment.API_SERVICE_DEVICE')` (internal service, not checked live — low priority given cache + timeout).

### 3. `StoryConnectService::getStoryDeepLink()` — low risk
Same pattern: Redis-cached (1hr TTL), 5s-timeout HTTP call on miss only.

### 4. `CurlService` has two **unbounded-timeout** methods (`curlGetDataNotTimeOut`, and untokened branches of `_curlPost`/`_curlPostAuth`) — real risk, but **not in this endpoint's path**
Traced their only callers: `UpgradeFirestoreService` (queued job, not synchronous) and `GiftController`/CRM gift-count endpoints. Neither is reachable from `load-update`. **Ruled out for this specific endpoint**, but worth a separate ticket — any future code path that reuses `curlGetDataNotTimeOut` against a now-cross-cloud dependency is a latent multi-minute-hang risk.

### 5. Async job dispatches (`ConvertAward`, `AddTimeUpdate`, `UpdateAccessTime`) — low risk for this endpoint, but see queue finding above
Queue driver defaults to `rabbitmq` (async, non-blocking) in config. Not deep-dived further since it wouldn't block the response unless the broker publish call itself hangs — no evidence found of that.

## What this spike did NOT cover (explicitly unverified)

- **No distributed tracing** — we could not confirm which specific call within `refreshData()` actually consumes the bulk of the 6.4s. All infra findings are inferred from static code + live topology checks, not a captured slow trace.
- **Mongo-side diagnostics not checked** — no access yet to Mongo slow-query log, connection pool stats, or auth handshake timing on the Azure host itself.
- **Redis cache hit rate for the Mongo-gating checks** (`getSyncUserRedis`, etc.) not measured — if the cache hit rate is very high, Mongo may only be hit for a minority of requests, which would mean it explains a long tail (p99) more than the average (p90/mean) — worth clarifying which percentile "6.4s" actually is.
- **`settingInfoApp`, `calculatorPurchasedV2`, `logsUserRegisterService`, `handleUpdateData`, and other collaborators in `refreshData()`** were not individually read line-by-line — grep triage covered known external-call patterns (curl/Http/Mongo/Dynamo/Redis) but a full read (per the "exhaustive" option we explicitly declined for this pass) could still surface something missed.
- **Kong-level timing** (gateway processing time, any Kong plugin overhead) not checked — request could also be slowed before it even reaches `edu_app`.
- **RabbitMQ/MySQL host `10.20.x.x`** assumed to be AWS-VPC-internal based on context (RDS confirmed in `ap-southeast-1`) but not explicitly verified via VPC CIDR lookup.

## Instrumentation expansion (2026-08-07): from 4 call sites to full coverage across all 7 API versions

The original pass (above) wrapped only 4 call sites in v5's `refreshData()`, as a confirmatory/minimal-risk first step. Once delivery was confirmed working, instrumentation was expanded in a series of small, reviewed PRs — each merged to `master` in `edu_app` before the next branched, so no PR built on unmerged work:

| PR | Scope |
|---|---|
| #2237 | Redis calls in the v4/v5/v6-shared `LoadUpdateService`/`LogsUserRegisterService` code (10 call sites) — the "surface" pass had deliberately excluded Redis, this filled that gap |
| #2238 | v7's `refreshDataMS2()`-specific gap (calls not reachable from v5's `refreshData()`, e.g. `SyncDataService::syncUserMS`, `syncProfileMSVersion`, `UserEventService::checkUserParticipateEvent`) + two shared v5/v7 gaps missed in the original pass (`deviceConnectService->getAppAccountToken`, `identityRepo->listIndentityByuserId`) + redis calls found in adjacent files (`ProfileService`, `DetectIpService`, `ShowKeyService`, `PayInAppService`) |
| #2239 | New metric `edu_app.appapiv5.load_update.request.duration` — wraps the *entire* `refreshData()`/`refreshDataMS2()` execution (not per-call), tagged `api_version:v4\|v5\|v6\|v7`, so total end-to-end time is directly queryable and comparable against the sum of per-call metrics |
| #2240 | v1's `LoadUpdateController` — a fully separate, self-contained controller (no shared `LoadUpdateService`), instrumented independently |
| #2241 | v2's `LoadUpdateController` — same treatment, larger file (860 lines), ~330-line diff |
| #2242 | v3's `LoadUpdateController` — same treatment, smaller file |
| #2243 | Gap-fill: calls inside `settingInfoApp()` and its delegates (`CommonResourceService`, `SettingService`, `LessonReviewService`, `CommonService`, `UpgradeService`, plus two more `LoadUpdateService.php` gaps) that were missed in every prior pass — found while investigating the "unaccounted" ~2.5s gap between `request.duration` and the sum of `external_call.duration` (see below) |

**Net result**: from 4 instrumented call sites to roughly **45+** across all 7 API versions (v4/v5/v6 share code via `LoadUpdateService`; v1/v2/v3 are fully independent controllers each instrumented separately; v7 shares some helpers with v5 but has its own `refreshDataMS2()` entry point). Both metric names carry the same `edu_app.appapiv5.load_update.*` prefix regardless of which version actually emitted them (a known simplification — `api_version`/`call`/`datastore` tags carry the real distinguishing info, see "Open items" below for the naming discussion that was deliberately deferred).

**Scope boundary applied throughout**: only *directly reachable* repo/HTTP/redis calls were wrapped — one hop into tightly-coupled helper services (e.g. `PayInAppService`, `VersionService`, `DetectIpService`, `CommonResourceService`) when they're clearly part of this endpoint's own data-fetching flow, but **not** chased deeper into broad, genuinely-shared services used well beyond `load-update` (e.g. `ProfileCourseService`, `ConvertAccountService`, `UserService`'s internals) — those are treated as opaque units at the call site instead. This is a deliberate scope decision, not an oversight — see "Read-through-cache as a design smell" below for why this makes some calls hard to fully attribute.

Also confirmed and left untouched (dead code, no callers from `load-update`'s reachable graph): `LoadUpdateService::getInfoMonkeyPro`, `calculatorPurchased` (non-V2), `getPathFirstInstallMS2` (commented out), `DetectDeviceService::isDeviceInHouse`, `_timeExpire()` in v1/v2's controllers, `getStatusPurchased` (non-V2, public — used by *other* endpoints like `ProductCourseController`, `UpdatePayClevertapMXService`, `LicenceActiveService`, but not by `load-update`).

## Second root cause found (2026-08-08): dead Sentry APM endpoint, ~2s tax per request

Once real production data accumulated against the full instrumentation, `request.duration` showed v4/v5 averaging **~4.7-4.8s**, while v7 averaged only **~150ms** — a ~30x gap. The per-call breakdown (`external_call.duration` grouped by `call`) showed the actual DB/cache/mysql/mongodb/dynamodb calls were all fast (low-single-digit ms to ~130ms for dynamodb) — but several *unrelated* HTTP calls to different downstream services (`story_connect`, `lesson_connect`, `audio_book_service`, `platform_mx_connect`) all clustered tightly around **~2016-2034ms**, with near-zero variance (e.g. `get_story_deep_link.story_connect`: 2018ms avg across 30 samples, 16ms spread).

**Investigation chain** (each step measured directly, not inferred):
1. DNS resolution for `edu-story.app.svc.cluster.local` from inside the `edu-app` pod: ~6ms — fast, ruled out.
2. TCP connect + curl phase timing (`time_namelookup`/`time_connect`/`time_starttransfer`/`time_total`) against the real endpoint: `starttransfer` (time to first byte) was ~10-20ms even for a real 200 response — but `total` was ~2019ms. The gap is specifically **between first byte and full completion of a 65-byte chunked response** — ruled out DNS, TCP connect, TLS (plain HTTP in-cluster), and payload size.
3. Tested via the Service ClusterIP vs a direct pod-IP connection (bypassing kube-proxy/iptables/conntrack entirely) — **same ~2.01s delay both ways**, ruling out kube-proxy/Service DNAT.
4. Tested an unrelated, known-fast in-cluster service (`kubernetes.default.svc.cluster.local/healthz`) — 18.6ms, no delay — ruling out "all pod-to-pod HTTP is slow."
5. Tested via pure `localhost` inside the `edu-story` pod itself (no network hop at all) — **still ~2.01s**, proving the delay is entirely internal to the PHP request lifecycle, not network/infra.
6. Read `edu_story`'s source (`app/Http/Controllers/App/StoryDeepLinkController.php`, base `Controller.php`, `VerifyTokenServer` middleware, `ApiResponse` trait) — no explicit `sleep()` or obvious delay in application code.
7. `edu_story`'s `bootstrap/app.php` globally registers `Sentry\Laravel\ServiceProvider` and `Sentry\Laravel\Tracing\ServiceProvider`. Checked the live pod's `.env`: **`SENTRY_TRACES_SAMPLE_RATE=1.0`** (100% of requests traced) with `SENTRY_LARAVEL_DSN` pointing at `sentry.monkeyenglish.net` — a **self-hosted, apparently-dead Sentry instance**.
8. Confirmed unreachable from **both** inside the cluster and from an entirely separate external network: DNS resolves fine (`52.237.81.7`, an Azure IP with no reverse DNS), but TCP connect on 443 never completes — a silent black hole, not an active refusal. This points to the Sentry host itself being decommissioned/down, not a cluster-specific egress block.
9. **Live-tested the fix**: temporarily flipped `SENTRY_TRACES_SAMPLE_RATE` to `0` directly in a running `edu-story` pod's `.env` (ephemeral, reverted immediately after). Response time dropped from **~2010ms to ~7ms** — a ~280x improvement, fully reproducing and confirming the theory. Reverted the live pod to its original state afterward since the durable fix needed to go through the proper source of truth.

**Mechanism**: with `traces_sample_rate=1.0`, Sentry's PHP SDK tries to synchronously ship a performance-transaction event to the DSN host as part of Lumen's request-termination lifecycle, *after* the actual response body has already been flushed to the client. Since the host is unreachable, the SDK's own HTTP transport eventually times out (empirically ~2s) before letting the request finish — adding that fixed tax to **every single request**, regardless of what the request actually does. This explains why the delay was so consistent (an SDK-internal timeout, not variable network jitter) and identical across unrelated services (`edu-story` and `edu-lesson` share the same Lumen template and the same dead Sentry config).

**Durable source of truth found**: not in either app's git repo or `infra_edu_story`/`infra_edu_lesson`'s Helm charts (no env/secret refs there) — the `.env` is generated from **AWS SSM Parameter Store**, `SecureString` type, parameters `env-eks-edu-story`, `env-eks-edu-story-2`, `env-eks-edu-lesson`, `env-eks-edu-lesson-2` (the `-2` suffix parameters mirror the primary ones exactly — likely a blue-green or second-cluster pair). Presumably pulled into the `.env` at CI build/deploy time (no `.env`-handling step visible in either Dockerfile from local checkouts — CI pipeline definition not present in local repos).

**Fix applied**: all 4 SSM parameters updated, `SENTRY_TRACES_SAMPLE_RATE=1.0` → `0`, via `aws ssm put-parameter --overwrite` (2026-08-08). Rolled out through the normal CI/build pipeline (not a live kubectl edit) later the same day. Confirmed post-rollout on fresh pods (`SENTRY_TRACES_SAMPLE_RATE=0` present in `/var/www/.env`, response times back to ~7-9ms for the previously-2s-tainted calls).

**Measured production impact** (Datadog `request.duration`, before/after the rollout):

| Version | Before (avg) | After (avg, stable ~1h post-rollout) | Change |
|---|---|---|---|
| v5 | ~4869ms | ~797ms (settled; briefly ~747ms right after rollout) | **-84%** |
| v4 | ~4617ms | ~637ms (settled; briefly ~417ms right after rollout) | **-86%** |
| v7 | ~138ms | ~144ms | unchanged (never hit the affected downstream services) |

`SENTRY_TRACES_SAMPLE_RATE=0` fully disables trace-sending (not just sampling down) — chosen over a partial sample rate as the immediate fix since the destination is entirely dead; error capture (separate from performance tracing) is unaffected. **Not yet resolved**: whether `sentry.monkeyenglish.net` should be brought back, pointed at a new host, or the Sentry integration dropped entirely — flagged as an open decision, not a code/infra task (see "Open items").

## Call graph, N+1 patterns, and structural observations (2026-08-08)

With the Sentry tax removed, v5 request.duration settled at ~797ms average but the sum of all instrumented `external_call.duration` entries for v5's reachable call graph only accounts for roughly ~290ms — leaving **~500ms still unaccounted for** by any of the ~45 instrumented calls. Reading through the code (not just the metrics) surfaced two concrete, fixable causes, both inside `LoadUpdateService::calculatorPurchasedV2()`'s per-course loop:

### Call graph shape (v5, simplified)

```
LoadUpdateController@refreshData (v5)
 └─ LoadUpdateService::refreshData()
     ├─ getUserInfo()                                    [1x, mysql]
     ├─ deviceConnectService->getAppAccountToken()        [1x, http]
     ├─ userReceiverRepo->getByUser()                     [1x, mysql]
     ├─ getPathFirstInstall() / getPathDownload() / ...   [several, mostly redis-cached]
     ├─ calculatorPurchasedV2()                            [see below — the hot loop]
     ├─ settingInfoApp()                                   [~20 sub-calls, see PR #2243]
     ├─ syncDataService->syncUser() / syncProfileV2()      [1x each, mongodb/dynamodb, internally instrumented]
     ├─ logsUserRegisterService->checkVersionRegister()    [1x, mongodb, internally instrumented]
     ├─ promotionService->checkShowBonusMJ5() / addDays()  [1x each]
     ├─ promotionMSpeakService->addPromotion()             [1x]
     └─ getStoryDeepLink()                                 [1x, http via story_connect]

calculatorPurchasedV2(userID, subversion, appID)
 ├─ profileCourseService->getAppHasCourse()[appID]         [pure, in-memory — returns 5 course IDs for APP_ID_LTR]
 ├─ packageService->getListProductByCourse()               [1x, mysql]
 ├─ deviceConnectService->getInfoPackagePayInapp()          [1x, http — fetched ONCE here as $packageIAP]
 ├─ profileCourseService->getExpMonkeyPro()                [1x]
 └─ foreach ($listCourse as $courseID):  ← runs 5x for APP_ID_LTR
     ├─ profileCourseService->getTimeExpProfile()          [redis+mysql — fetched AGAIN inside getFreeDaysUser(), see bug #1]
     ├─ getStatusPurchasedV2()                              [reuses $packageIAP/$packageCod passed in — correctly avoids re-fetch]
     ├─ profileCourseService->getExpNewFromMonkeyPro()      [pure, no I/O]
     ├─ profileCourseService->isActiveUser()                [pure, no I/O]
     ├─ profileCourseService->getFreeDaysUser()             [BUG #1: re-fetches getTimeExpProfile() redundantly]
     ├─ getProductIdMJOld()                                  [mysql, conditional on courseID==COURSE_EE]
     ├─ getProfileTrial() → profileCourseService::getProfileTrial() [mysql, conditional on trial status]
     ├─ getTimeBeginActive()                                [BUG #2: re-fetches getInfoPackagePayInapp() redundantly]
     ├─ promotionMSpeakService->getProductPromotion()        [mongodb, internally instrumented, genuine N+1 by design]
     └─ isCancelAutoRenew() → deviceConnectService->getInfoUserCancelRenew() [http, genuine N+1 by design]
```

### Two concrete redundant-fetch bugs (not yet fixed)

**Bug #1 — `ProfileCourseService::getFreeDaysUser()` re-fetches data the caller already has.**
`calculatorPurchasedV2()` fetches `$timeExp = $this->profileCourseService->getTimeExpProfile($userID, $courseID)` at the top of each loop iteration. A few lines later it calls `getFreeDaysUser($isActive, $courseID, $userID)`, whose implementation (`ProfileCourseService.php:436-443`) does:
```php
public function getFreeDaysUser($isActive, $courseId, $userId) {
    if ($isActive) return self::OUTDATED_TRIAL;
    $timeExp = $this->getTimeExpProfile($userId, $courseId);  // ← re-fetches the same value
    return $this->roundFreeDay($timeExp);
}
```
Whenever `$isActive` is false (any free/trial user — a common case), this is a fully redundant redis+mysql round trip for data already sitting in the caller's `$timeExp` variable. Fires once per course, so up to **5x per request for `APP_ID_LTR`** (5 courses: `[103, 104, 105, 108, 201]`).

**Bug #2 — `LoadUpdateService::getTimeBeginActive()` re-fetches an HTTP call the caller already made.**
`calculatorPurchasedV2()` fetches `$packageIAP = $this->deviceConnectService->getInfoPackagePayInapp($userID, $appID, $subversion)` **once**, at the top of the function, before the loop. Inside the loop, `getTimeBeginActive($isActive, $packageCod, $userID, $appID, $courseID, $subversion)` (`LoadUpdateService.php:662-693`) makes its own call with **identical arguments**:
```php
// N+1: called once per course in calculatorPurchasedV2()'s loop, not cached/deduped across courses
$packageIAP = $this->dogStatsdService->timeCallable(
    'edu_app.appapiv5.load_update.external_call.duration',
    fn() => $this->deviceConnectService->getInfoPackagePayInapp($userID, $appID, $subversion),
    ['call' => 'get_time_begin_active.device_connect', 'datastore' => 'http']
);
```
The code comment already flags this as "N+1", but understates it — this isn't just uncached, it's re-fetching data the *caller already has in scope* (`$packageIAP`) but never passes down (only `$packageCod` is passed to `getTimeBeginActive`). Since this is a real HTTP round-trip (not redis/mysql), it's the more expensive of the two bugs — **5 redundant HTTP calls per `APP_ID_LTR` request**.

**Proposed fix for both** (not yet implemented): pass the already-fetched `$timeExp`/`$packageIAP` values as parameters into `getFreeDaysUser()`/`getTimeBeginActive()` instead of having them re-query. Small, safe, structural fix — explicitly **not** a parallelization/concurrency band-aid (user's explicit direction: sequential-to-parallel conversion introduces its own locking/sync complexity later and doesn't address the actual root cause, which is duplicated work, not serialization).

### Read-through-cache as a design smell

A large fraction of the ~45 instrumented call sites follow the same shape: `redis->get(...)`, on miss call a remote service/repo, then `redis->set(...)` the result. Examples found across this investigation: `getStoryDeepLink`, `getChallenge`, `getProfileKindyCache`, `getRegisterTutoring`, `getVersionF2P`, `getLinkCDN`, `CommonResourceService::getDataCommonResourceByAppId(V2)`, `VersionService::getAllVersionLessonProfile`/`getDataVersionAppInfo`, `UpgradeService::getStatusRequiredUpgrade`, `DetectIpService::getInfoLocationFormIp`, `ShowKeyService::getShowStatus`, `UserEventService::checkUserParticipateEvent`, `ProfileService::getVersionProfile`, and more — roughly 15+ distinct instances of this exact pattern across the files touched.

This is flagged here as a **standing design concern**, not something fixed in this investigation:
- **Hard to reason about current state.** Each call site owns its own cache key, its own TTL (inconsistent — some 1hr, some 3600s literal, some no visible TTL/permanent), and its own miss-handling logic, duplicated ad-hoc rather than through a shared caching abstraction. There's no single place to answer "what's cached, for how long, and why" for this endpoint.
- **Every instance is a potential version of bug #1/#2 above.** The pattern of "fetch once at a higher level, then a nested helper re-derives the same thing via its own redis-then-remote round trip" is structurally easy to introduce precisely *because* each read-through-cache call looks self-contained and side-effect-free from the call site — nothing signals "this data might already be in scope."
- **Likely premature optimization in places.** Some of these caches wrap calls that, per this investigation's own timing data, are already fast (sub-5ms mysql/mongodb reads) — the redis layer adds a network hop, cache-invalidation surface area, and a stale-data risk for latency savings that may not matter at that call's actual frequency. Distinguishing "this cache earns its complexity" from "this cache is defensive copy-paste from another call site" would need per-call hit-rate data (not currently instrumented — see "Open items").
- **Not something to fix as part of this investigation** — noted for future architectural discussion, e.g. a shared "read-through cache" helper/decorator that at least centralizes TTL policy and miss-handling, or a case-by-case audit of which of these ~15 caches are load-bearing vs incidental.

## Sentry blast-radius audit (2026-08-09/10): not an edu_app-only problem

A Firebase Performance report of unrelated endpoints on the same platform — `vnmedia2.monkeyuni.net` (126 req, p95 2.2s), `app.monkeyenglish.net/device/api/v4/location/register` (123 req, p95 4.95s), `app.monkeyenglish.net/award/api/v1/gift/list` (152 req, p95 2.65s), all on 2026-08-09 Vietnam time — prompted a check of whether the dead-Sentry-DSN root cause (see "Second root cause" above) extends beyond `edu_story`/`edu_lesson`.

**Time correlation checked first, found not to cluster**: converting all four data points (including the original `load-update` one) to UTC showed no shared moment (17:07, 01:05, 14:05, 15:01 UTC) — ruling out a single shared-infra event and pointing instead at each service independently carrying its own version of the same underlying config problem.

**Full audit of every `env-eks-*` SSM parameter** (77 parameters, ~40 distinct services): 13 services (26 parameters counting `-2` pairs) still have `SENTRY_TRACES_SAMPLE_RATE=1.0` pointed at the same dead `sentry.monkeyenglish.net` DSN: `edu-app`, `edu-app-v2`, `edu-award`, `edu-developer`, `edu-device`, `edu-lms`, `edu-mail`, `edu-media`, `edu-product`, `edu-ticket`, `mk-classroom`, `mk-course`, `question-service`. (`edu-share-photo` at 0.2 and `mk-classroom-go`/`mk-course-go` at 0.1 use lower, saner rates against the same dead host — not urgent, not touched.)

**Important nuance — not all 13 pay the same "tax on every request"**: the mechanism confirmed on `edu-story`/`edu-lesson` (see above) specifically requires `Sentry\Laravel\Tracing\ServiceProvider` to be registered in `bootstrap/app.php` — that's what creates an automatic transaction and synchronously sends it during Lumen's request-termination lifecycle, on *every* request regardless of outcome. Checked each service's `bootstrap/app.php` (where the repo was available locally):

| Registers Tracing provider (pays tax on every request) | Base ServiceProvider only (exception-triggered only, not a tax on healthy requests) |
|---|---|
| `edu_award` ✅, `edu_media`, `mk_course` | `edu_app`, `edu_app_v2`, `edu_developer`, `edu_device`, `edu_lms`, `edu_product`, `edu_ticket`, `mk_classroom` |

`edu_mail`/`question_service` repos weren't present in the local workspace — not verified either way.

This directly explains the `award/gift/list` report (`edu_award` confirmed has the Tracing provider) but **not** the `device/location/register` report — `edu_device` only has the base provider, so its 4.95s p95 (notably higher magnitude than the ~2s tax anyway) needs its own separate investigation. `vnmedia2.monkeyuni.net` is a different domain entirely (`monkeyuni.net` vs `monkeyenglish.net`) — likely not even this Lumen stack, not covered by this audit, not yet investigated.

Critically: **`edu_app` itself** (the service behind `load-update`) is on the dead-DSN list but only has the base provider — so it does *not* explain the earlier Firebase-vs-Datadog `request.duration` gap via this same "automatic per-request transaction" mechanism. That gap is still most plausibly explained by response encryption (now instrumented, see below) rather than `edu_app`'s own Sentry config. `edu_app`'s dead Sentry DSN is still a real, separate latent problem, though: any actual exception thrown during request handling will trigger a synchronous report attempt to the same unreachable host, adding a stall specifically on unhappy paths — not fixed as part of this investigation.

**Fixed 2026-08-10**: `edu_award`, `edu_media`, `mk_course` — same treatment as `edu_story`/`edu_lesson` (`SENTRY_TRACES_SAMPLE_RATE` flipped to `0` via `aws ssm put-parameter --overwrite` on both the primary and `-2` SSM parameters for each). Rolled out via CI to `edu_award`/`edu_media`/`mk_course` the same day.

### `SENTRY_TRACES_SAMPLE_RATE=0` alone is an incomplete fix — verified on `edu_award` post-rollout

After `edu_award`'s rollout, live-verified via `kubectl exec` + `curl`: a clean 200 request is now fast (~4-10ms, confirming the fix works for healthy traffic). But a request that hit an actual PHP exception (`ErrorException: Undefined array key 1` in `VerifyTokenApp.php:102`, triggered by an intentionally malformed test token) **still took ~2.01s** despite the sample rate being `0`.

**Why**: `SENTRY_TRACES_SAMPLE_RATE` only gates the automatic *performance transaction* sent on every request (the `Sentry\Laravel\Tracing\ServiceProvider` mechanism). Sentry's *exception/error reporting* is a separate code path in the SDK (visible in the stack trace via `Sentry\Laravel\Http\SetRequestIpMiddleware`/`SetRequestMiddleware`, both still active in the pipeline) — it is **not** gated by the sample rate at all. Any real exception still triggers a synchronous report attempt to the same dead `sentry.monkeyenglish.net` and pays the same ~2s SDK-timeout tax. This means `edu_story`/`edu_lesson` (fixed 2026-08-08) carry the same residual gap — only healthy-request latency was actually fixed there, not error-path latency.

**Complete fix**: blank `SENTRY_LARAVEL_DSN` entirely (not just zero the sample rate) — with no valid DSN configured, the SDK no-ops on both the tracing and the error-reporting paths. Applied to `edu_award` only so far (`env-eks-edu-award`/`env-eks-edu-award-2`, `SENTRY_LARAVEL_DSN=` blanked 2026-08-10, pending CI rollout) as a deliberate single-service test before doing the same for `edu_story`, `edu_lesson`, `edu_media`, `mk_course`.

**Not fixed, explicitly deferred**: the 8 base-provider-only services' dead DSN (lower priority — narrower blast radius, only affects error paths) — see "Open items" below.

## Open lead: off-peak latency swing (2026-08-09, unresolved)

After the Sentry fix settled, `request.duration` for v4/v5 wasn't flat at the ~750ms baseline — a full 24h pull showed a distinct pattern:

| Window (UTC, 2026-08-08→09) | v5 avg | Traffic (req/hr) |
|---|---|---|
| 01:00-05:00 | ~4780ms | (pre-rollout, old Sentry-tax pattern, expected) |
| 07:00-16:00 | ~730-850ms | 1000-2400/hr (daytime, high traffic) |
| **17:00-22:00** | **1100-1540ms** | **38-102/hr (lowest of the day)** |
| 23:00-00:00 | ~816-859ms | recovering (252-681/hr) |

The elevated window is counter-intuitive: it's the **quietest** hours of the day, not the busiest — rules out simple worker-pool/request-concurrency contention outright.

**Ruled out, with direct evidence:**
1. **Cold/idle DB connections.** Compared `external_call.duration` grouped by `datastore` between the low-traffic window (17:00-22:00) and a high-traffic window (12:00-15:00) — every datastore's per-call latency was statistically identical (redis 0.53 vs 0.56ms, mongodb 1.16 vs 1.19ms, mysql 2.00 vs 2.53ms, http 15.38 vs 20.18ms, dynamodb 69.28 vs 69.09ms). If connections were going idle and needing fresh handshakes, at least one of these would show elevated latency specifically during the quiet window — none do. The individual instrumented calls are not the source of the slowdown; the entire swing lives in the "unaccounted" portion of `request.duration` (see "Call graph" section above for what that means).
2. **`edu_app`'s own cron.** Only one active scheduled task (`checkStreakSendNotiMailCashback`, `dailyAt('00:00')`); live pod timezone confirmed UTC (`Etc/UTC`, no `date.timezone` override), so this fires at 00:00 UTC — the tail of the *recovery*, not the start of the elevation. Doesn't match.
3. **`edu_app_v2`'s segment-sync jobs** (`SyncDivideSegmentForCustomers`, 4 parallel invocations for `app_id` 2/40/50/51, all at `dailyAt('2:30')`) — live pod also confirmed UTC timezone, so 2:30 UTC, not 17:00 UTC. Also: **no `edu-app-v2` CronJob exists in the cluster at all** (`kubectl get cronjobs -A` lists `edu-app`, `edu-device`, `edu-lesson`, `edu-platform`, `edu-platform-today`, `edu-agent-service-agent`, `edu-crm`, `edu-developer`, `edu-mailsms`, `edu-ticket` — no `edu-app-v2`), so this scheduled task may not even be firing via k8s. Ruled out either way.
4. **`edu_device`/`edu_lesson`'s own schedules** — `edu_device`'s `GetExchangeRate` (`dailyAt('17:30')`) is close in time but almost certainly writes to a separate pricing/device DB, not `app-sql`. `edu_lesson`'s jobs are all `everyMinute`/`everyFiveMinutes`/`hourly` recurring tasks, not a one-time event that would produce a sharp isolated spike.

**Found, but not fully explanatory:** `app-sql` (the RDS instance backing `edu_app`) CloudWatch `CPUUtilization` shows a sharp, isolated spike at **exactly 17:00 UTC (00:00 Vietnam local time)** — average 7.8%, max 26.5%, vs a steady ~1.3% baseline everywhere else in the window. `DatabaseConnections` stayed flat through the same moment (no connection surge), suggesting a compute-heavy operation on existing connections rather than a burst of new traffic. The timing lines up with when `load-update`'s latency started climbing — but the spike itself lasted only ~15 minutes and returned to baseline immediately, while the elevated `load-update` latency persisted for roughly 5 more hours. That mismatch (short spike, long recovery) means the RDS spike may not be the direct cause, or may only be a triggering event (e.g. invalidating a cache that then took hours to naturally re-warm given how little real traffic flows during those overnight hours) — not confirmed either way.

**Where this stalled:** no RDS Performance Insights and no slow-query log available on `app-sql` (`PerformanceInsightsEnabled: False`, `describe-db-log-files` shows only `error/*` logs, no `slowquery/*`), so the exact query/source behind the 17:00 UTC spike can't be identified retroactively. Finding it would require either (a) enabling Performance Insights going forward (so the *next* occurrence is diagnosable) or (b) a much larger manual sweep through every other service's cron/schedule config sharing `app-sql` — neither was done, explicitly deprioritized in favor of documenting this as an open lead rather than continuing to chase it.

**To pick this back up:** enable RDS Performance Insights on `app-sql` (free tier, 7-day retention) and wait for the next occurrence, or check whether this pattern repeats on subsequent nights (would confirm it's a genuine recurring daily event, not a one-off) before investing more time in the cron-sweep approach.

### Client-side (Firebase) vs server-side (Datadog) magnitude mismatch — new instrumentation gap found

Firebase Performance Monitoring (client-side) reported **127 requests at ~00:07 Vietnam time on 2026-08-09 (≈17:07 UTC 2026-08-08) with p95 > 4.54s** — right at the start of the off-peak window above, and close to the RDS CPU spike timing. But Datadog's `request.duration.max` for v5, pulled at 5-minute resolution for that exact window (16:50-17:25 UTC), only reached **1096-1674ms** — nowhere near 4.54s.

This is a real, separate finding: **`request.duration` doesn't measure the full request lifecycle**, only `LoadUpdateService::refreshData()`'s own execution (per its PR #2239 implementation). It excludes:
- Middleware before `refreshData()` runs (`VerifyTokenApp`, routing, container resolution)
- **Response encryption/serialization after `refreshData()` returns** — v5's controller encrypts the response payload; this happens entirely outside the current metric
- Kong/gateway processing time
- Client-side network/DNS/TLS time

That leaves roughly **~2.9-3.4s unaccounted for**, outside everything instrumented in this investigation so far (including the Sentry fix and the two redundant-fetch bugs — neither touches this gap, since both are inside `refreshData()`). Response encryption is the strongest candidate given it's app-controlled, the timing correlates with real request volume (127 requests, not a fluke), and per-call external timing inside `refreshData()` stayed fast/normal in this exact window (ruling out the already-instrumented calls as the cause of *this specific* discrepancy).

**Instrumented 2026-08-10** (revisited sooner than planned, once the Sentry blast-radius audit ruled out `edu_app`'s own Sentry config as the explanation — see below): added to the same open PR (`feat/dd-metrics-settinginfoapp-gaps`, backs PR #2243, pushed as a follow-up commit):
- `EncodeService::encrypt()`/`decrypt()` — split `json_encode()` and the `phpseclib`-based AES-256-CBC encrypt/decrypt into separately-timed spans (`encode_service_encrypt.json_encode`, `encode_service_encrypt.aes_encrypt`, `encode_service_decrypt.aes_decrypt`, tagged `datastore:cpu`). Chosen as the strongest candidate for the missing ~3s: `phpseclib\Crypt\AES` is a **pure-PHP** implementation, not the native `openssl_encrypt()` extension — well known to be dramatically slower (often 10-100x) than OpenSSL's compiled implementation, especially on the large payload `load-update` returns. Covers both v5 and v6 (both call the same shared `EncodeService`; v4 doesn't encrypt, v7 has a different response path).
- New metric **`edu_app.appapiv5.load_update.controller_action.duration`** — wraps the *entire* v5/v6 controller action (request parsing through response building, both before `refreshData()` starts and after it returns), tagged `api_version`. This is the metric to compare directly against Firebase's client-observed p95 going forward — if it closes most of the gap, the remaining difference is genuinely Kong/gateway/network; if it doesn't, the encryption timing data will show whether `phpseclib` AES is the culprit.

### Result (2026-08-10, PR #2243 merged and rolled out): `phpseclib` AES theory conclusively refuted

First real data, within the hour of rollout:

| Metric | v5 |
|---|---|
| `controller_action.duration` (full HTTP action, request-in to response-out) | 842.5ms |
| `request.duration` (inner service call only, previous metric) | 821.6ms |
| `encode_service_encrypt.json_encode` | 0.19ms |
| `encode_service_encrypt.aes_encrypt` | 0.13ms |

Encryption + serialization combined: **under half a millisecond** — completely negligible, not the missing ~3s. And `controller_action.duration` (842.5ms) is nearly identical to `request.duration` (821.6ms) — everything outside the inner service call (middleware, job dispatches, encryption, response building) only adds ~20ms.

**Conclusion**: `edu_app`'s own PHP code, from request-in to response-out, is now fully accounted for and measured. The remaining gap between Firebase's client-observed p95 (4.54s, from the 2026-08-09 report) and this full-action measurement (~842ms) — roughly **~3.7s** — provably lives entirely **outside `edu_app`'s codebase**. The only remaining candidates are Kong/API-gateway processing time, network transit, or how Firebase's client SDK itself measures request duration (e.g. including connection setup time a server-side timer never sees). This is now out of scope for what code-level instrumentation in `edu_app` can resolve — next step, if pursued, would be Kong access logs/timing for the same window, not further `edu_app` changes.

(Caveat: this is very early post-rollout data, n=1-2 samples. Directionally clear given the ~4200x gap between encryption timing and the missing seconds, but worth re-confirming once more traffic accumulates.)

## Open items / explicitly deferred

- **Metric naming does not vary by API version** (`edu_app.appapiv5.load_update.*` used for all of v1-v7's emissions) — raised explicitly, user deferred a decision between (a) keeping one metric name + relying on `api_version`/`call` tags (Datadog-idiomatic, no metric-name cardinality explosion) vs (b) threading a real per-version metric name through every shared call site (bigger refactor, `LoadUpdateService.php` alone serves 4 versions). Not resolved — revisit if tag-based filtering proves insufficient in practice.
- **Read-through-cache hit-rate instrumentation** — not implemented. Would need a cheap counter (not a timer) per cache key to know what fraction of calls are genuinely miss-bound vs redis-served, which would materially inform which of the ~15 read-through-cache instances are worth simplifying/removing.
- **The two redundant-fetch bugs are diagnosed but not fixed** — tracked as sprint tasks, not yet done.
- **Sentry's long-term fate — decided 2026-08-10.** Sentry stays off across the platform; Datadog remains the primary observability tool. If Sentry is ever reintroduced, it needs deliberate planning (working DSN, sane sample rate, explicit decision on which services need it) rather than being silently carried forward from whatever the original per-service template baked in. Not an open question anymore — the remaining work is operational, and the fix has two parts (see "incomplete fix" note above): `SENTRY_TRACES_SAMPLE_RATE=0` (fixes healthy-request latency) and blanking `SENTRY_LARAVEL_DSN` entirely (also fixes error-path latency). Status per service:
  - **All 5 confirmed same-mechanism services — both fixes applied, rolled out, and verified live 2026-08-10**: `edu_award` (pilot — confirmed via `kubectl exec`+`curl`: healthy 200s ~6ms, and the same exception-triggering request that previously took 2.01s now completes in 6-30ms), `edu_story` (re-verified post-rollout: `get-story-deeplink` now ~13-16ms, down from the original ~2018ms), `edu_lesson`, `edu_media`, `mk_course` (all confirmed `SENTRY_LARAVEL_DSN=` blank + `SENTRY_TRACES_SAMPLE_RATE=0` on fresh post-rollout pods). This closes out the Sentry-tax fix for every service known to carry the "pays tax on every request" mechanism.
  - 8 base-provider-only services + 2 unverified (`edu_mail`, `question_service`): dead DSN still fully present, nothing fixed yet — lower priority (see below), still open.
- **8 base-provider-only services still carry the dead DSN** (`edu_app`, `edu_app_v2`, `edu_developer`, `edu_device`, `edu_lms`, `edu_product`, `edu_ticket`, `mk_classroom`) — lower priority since they don't tax healthy requests, but any thrown exception on these services will still stall trying to report to the dead host. Not fixed.
- **`edu_mail`/`question_service`** — Sentry Tracing-provider status unverified (repos not present in local workspace). Need a live-pod check to know if they belong in the "confirmed" or "lower priority" bucket.
- **`edu_device`'s 4.95s p95** — not explained by the confirmed Sentry mechanism (no Tracing provider registered). Separate, unstarted investigation.
- **`vnmedia2.monkeyuni.net`'s 2.2s p95** — different domain (`monkeyuni.net`, not `monkeyenglish.net`), likely a different stack entirely. Not scoped yet.
- **`edu-app-queue` CrashLoopBackOff** (OOM, unrelated to this endpoint) — still open, own ticket.
- **Original Mongo Azure/AWS mixed-replica-set hypothesis** (see "Correction" section above) — never definitively confirmed or ruled out; superseded in practical importance by the root causes actually found, but the replica-set topology question itself is still unresolved.

## Recommended next steps (sprint, as of 2026-08-10)

1. ~~Fix the Datadog Agent `hostPort` gap~~ — **done 2026-08-06**.
2. ~~Expand instrumentation to full coverage, let it run against real traffic, find the actual bottleneck~~ — **done 2026-08-07/08**.
3. ~~Fix the Sentry root cause on `edu_story`/`edu_lesson`~~ — **done 2026-08-08**. ~~Audit the rest of the platform, find and fully fix (sample-rate + DSN) all confirmed same-mechanism services~~ — **done 2026-08-10**, rolled out and verified live on all 5 (`edu_story`, `edu_lesson`, `edu_award`, `edu_media`, `mk_course`). Audit itself complete for all 40 services with an `env-eks-*` parameter.
4. ~~Instrument JSON serialization + AES encryption + full controller-action duration for v5/v6, merge, roll out, confirm~~ — **done 2026-08-10** (PR #2243 merged and rolled out same day). Result: `phpseclib` AES theory conclusively refuted (encryption+serialization <0.5ms combined); the ~3.7s Firebase-vs-server gap is now proven to live entirely outside `edu_app`'s codebase (Kong/gateway/client-SDK, not app code) — no further `edu_app` instrumentation can resolve it further.
5. **Fix bug #1 and bug #2** (redundant `getTimeExpProfile`/`getInfoPackagePayInapp` re-fetches in `calculatorPurchasedV2()`'s loop) — small, safe, not yet done.
6. Check `edu_mail`/`question_service` for the Sentry Tracing provider, fix via SSM if confirmed — not yet done.
7. Decide fate of the 8 base-provider-only services' dead Sentry DSN — not yet done, lower priority.
8. Investigate `edu_device`'s 4.95s p95 (separate cause) — not started.
9. Investigate `vnmedia2.monkeyuni.net`'s 2.2s p95 (different stack, not scoped) — not started.
10. Enable RDS Performance Insights on `app-sql` — makes the next off-peak-swing occurrence diagnosable — not yet done.
11. Consider a read-through-cache audit (see above) — not urgent, flagged as a recurring source of exactly this class of redundant-fetch bug.
12. Separately: `edu-app-queue` CrashLoopBackOff (OOM) should get its own ticket — real broken thing, unrelated to this endpoint.
13. **Investigate the ~3.7s Firebase-vs-Datadog gap itself** — proven (2026-08-10) to live entirely outside `edu_app`'s codebase, but the actual source is still unknown. Candidates: Kong/API-gateway processing time (check Kong access logs/timing for the same request window), network transit, or Firebase Performance Monitoring's own client-side measurement methodology (may include connection setup time a server-side timer never sees). Not started.
