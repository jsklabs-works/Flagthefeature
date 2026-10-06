# Feature Flag Platform — Design Plan

Status: Draft · Target: Spring Boot microservices on GCP

## 1. Problem

- Multiple features are developed in parallel, but turning them on/off today means a config change plus a **restart/redeploy**.
- We need to toggle features **at runtime**, per environment, and gradually (by tenant/user/percentage), with an audit trail and a fast kill switch.

## 2. Goals / Non-goals

**Goals**
1. Flip a flag and have every running instance see it in **< 5 s** (target), < 60 s worst case, no restart.
2. Flag checks are **local, in-memory** (sub-microsecond, no network call per evaluation).
3. Services keep working if the flag platform is down (last-known values → code defaults).
4. Targeting: on/off, allow-lists (tenant, user, region…), percentage rollout with sticky bucketing.
5. Audit: who changed what, when, why. Prod changes can require approval.
6. Drop-in for Spring Boot: one starter dependency, one interface.

**Non-goals (v1)**
- A/B experimentation analytics (we expose evaluation events; analysis lives elsewhere).
- Client-side (browser/mobile) SDKs.
- Replacing general app config (DB URLs, timeouts) — that stays in Spring config / Secret Manager.

## 3. Build vs. buy — and why we still build on a standard

| Option | Notes |
|---|---|
| Spring Cloud Config + `@RefreshScope` + Bus | No restart, but no targeting/percentage rollout, bean re-creation side effects, no audit UI. Good for config, weak for flags. |
| SaaS (LaunchDarkly, Split, ConfigCat) | Fastest to adopt; cost scales with seats/MAU; data leaves GCP. |
| Self-hosted OSS (Unleash, Flagsmith, GrowthBook, flagd) | Mature; we operate it. Worth a 1-day spike as a benchmark. |
| **Custom (this doc)** | Full control, GCP-native IAM/audit, no per-seat cost. We own maintenance. |

**Decision:** Build custom, but code services against the **OpenFeature Java API** (`dev.openfeature:sdk`) and ship our platform as an OpenFeature *Provider*. Application code never imports our classes directly, so we can swap to Unleash/flagd/a vendor later without touching feature code.

## 4. Architecture

```
            ┌──────────────────────── Admin plane ────────────────────────┐
  Engineers │  Flag Admin UI  ──(IAP)──►  flag-service (Spring Boot)       │
            │                              │  CRUD, validation, approvals  │
            │                              ▼                               │
            │                       Cloud SQL (Postgres)                   │
            │                       flags, rules, audit_log, versions      │
            │                              │ on commit                     │
            │              ┌───────────────┴──────────────┐                │
            │              ▼                              ▼                │
            │   GCS: snapshots/{env}.json        Pub/Sub topic             │
            │   (versioned, immutable copy)      flags-changed-{env}       │
            └──────────────┬──────────────────────────────┬───────────────┘
                           │                              │ (push hint: "v=1043")
            ┌──────────────┼──────────── Data plane ──────┼───────────────┐
            │              ▼                              ▼                │
            │   ┌─────────────────────────────────────────────────────┐    │
            │   │ Every microservice instance                         │    │
            │   │  flags-spring-boot-starter                          │    │
            │   │   • bootstrap: GET /v1/snapshot/{env} (→ GCS fallback)│  │
            │   │   • listen: Pub/Sub sub → refetch if v > local v     │    │
            │   │   • poll: every 30s with ETag (safety net)           │    │
            │   │   • AtomicReference<Snapshot> + local evaluator      │    │
            │   │   • OpenFeature Provider + Micrometer metrics        │    │
            │   └─────────────────────────────────────────────────────┘    │
            └──────────────────────────────────────────────────────────────┘
```

### 4.1 Components

**flag-service** (Spring Boot, Cloud Run or GKE, 2+ replicas)
- Admin REST API (CRUD, validate, approve, rollback to version N).
- Read API: `GET /v1/snapshot/{env}` → full JSON snapshot with `ETag: "<version>"`; returns `304` when unchanged. This endpoint is cheap and cacheable.
- On every committed change, in the same transaction: bump `env.version`, write `audit_log`. After commit (transactional outbox → publisher): write `gs://<bucket>/snapshots/{env}/{version}.json` + `latest.json`, publish `{env, version}` to Pub/Sub.

**Store: Cloud SQL Postgres** — relational fits rules/approvals/audit well, and we probably already run it. (Firestore is a valid alternative if you want realtime listeners and no DB to manage; see §10.)

**Propagation: "push a hint, pull the data"**
- Pub/Sub message carries only `{env, version}`; the SDK then pulls the snapshot. Messages are tiny, idempotent, and order doesn't matter (SDK just keeps the max version).
- Each instance needs its *own* subscription to get fan-out. The starter creates `flags-{service}-{instanceId}` with an `expirationPolicy` of 1 day and deletes it on graceful shutdown; orphans self-expire.
- **Polling every 30 s with ETag** is always on as a safety net (missed messages, Pub/Sub permissions issues, local dev). Phase 1 can ship with polling only.

**Resilience: GCS snapshot**
- If flag-service is down at startup, the SDK reads `latest.json` from GCS. If that also fails, it uses defaults declared in code. A running instance just keeps its last-known snapshot.
- Optionally persist the last snapshot to local disk for faster cold starts.

## 5. Flag data model

```jsonc
{
  "key": "checkout.new-pricing-engine",
  "type": "BOOLEAN",                // BOOLEAN | STRING | NUMBER | JSON
  "description": "Routes pricing to v2 engine",
  "owner": "team-payments",
  "kind": "RELEASE",                // RELEASE | OPS (kill switch) | PERMISSION | EXPERIMENT
  "expiresAt": "2026-12-31",        // stale-flag alerts for RELEASE flags
  "variants": { "on": true, "off": false },
  "environments": {
    "prod": {
      "enabled": true,              // master switch; false => offVariant for everyone
      "offVariant": "off",
      "rules": [                    // first match wins
        { "id": "r1", "if": [{ "attr": "tenantId", "op": "IN", "values": ["acme", "globex"] }], "serve": "on" },
        { "id": "r2", "if": [{ "attr": "region",   "op": "EQ", "values": ["eu-west1"] }],
          "rollout": { "bucketBy": "userId", "weights": { "on": 10, "off": 90 } } }
      ],
      "defaultServe": "off"
    }
  }
}
```

Operators: `EQ, NEQ, IN, NOT_IN, STARTS_WITH, MATCHES, GT/LT (numbers, semver), IN_SEGMENT`.
**Segments** (named reusable lists, e.g. `beta-tenants`, `internal-users`) avoid copy-pasting IDs into many flags.

**Percentage rollout** — deterministic & sticky:
`bucket = murmur3_32(flagKey + ":" + salt + ":" + ctx[bucketBy]) mod 10_000` → compare to cumulative weights.
Same user gets the same answer on every service and every instance, and raising 10% → 20% only adds users (never flips existing ones).

### 5.1 Postgres tables (sketch)

```
flag(id, key UNIQUE, type, kind, owner, description, expires_at, archived, created_at)
flag_variant(flag_id, name, value_json)
flag_env_config(flag_id, env, enabled, off_variant, default_serve, rules_json, updated_at, updated_by)
segment(id, key, env, rules_json)
env_state(env PRIMARY KEY, version BIGINT)            -- bumped on every change
audit_log(id, env, flag_key, actor, action, before_json, after_json, reason, ticket, at)
change_request(id, env, flag_key, proposed_json, requested_by, approved_by, status)  -- prod approvals
outbox(id, env, version, published_at NULL)
```

## 6. Client SDK: `flags-spring-boot-starter`

### 6.1 Usage

```java
// build.gradle
implementation "com.yourco.flags:flags-spring-boot-starter:1.x"

# application.yml
flags:
  env: prod
  service-name: order-service
  endpoint: https://flag-service.internal/v1
  snapshot-bucket: yourco-flags-prod
  poll-interval: 30s
  pubsub.enabled: true
```

```java
@Service
class PricingService {
  private final Client flags;   // dev.openfeature.sdk.Client, auto-configured

  Price price(Order o) {
    if (flags.getBooleanValue("checkout.new-pricing-engine", false)) {   // false = safe default
      return v2.price(o);
    }
    return v1.price(o);
  }
}
```

Context (tenantId, userId, region, appVersion, service) is filled automatically per request by a servlet/WebFlux filter from the JWT/headers into OpenFeature's *transaction context*, so most call sites pass nothing.

Optional sugar:
```java
@FeatureToggle(value = "reports.async-export", fallbackMethod = "exportSync")
public Report export(...) { ... }
```
and a `@ConditionalOnFlag` style *router* bean for swapping whole strategy implementations at runtime (beans for both implementations exist; the flag picks one per call — no context refresh).

### 6.2 Internals

- `AtomicReference<CompiledSnapshot>`: rules pre-compiled (regex compiled, `IN` lists → `HashSet`) on load, so evaluation is allocation-light and lock-free. Swapping in a new snapshot is a single atomic set.
- Background `ScheduledExecutorService` for polling; Pub/Sub subscriber via `spring-cloud-gcp-starter-pubsub` (or plain client library).
- Startup: bootstrap with timeout (e.g. 3 s) → GCS → code defaults. Exposed as a Spring Boot **health indicator** (`flags: UP, version=1043, ageSeconds=4`) — degraded, not DOWN, when stale, so it never takes pods out of rotation.
- **Metrics (Micrometer → Cloud Monitoring):** `flags.snapshot.version`, `flags.snapshot.age`, `flags.refresh.failures`, `flags.evaluations{flag,variant}` (sampled / aggregated, no user IDs as tags).
- **Evaluation usage reporting:** SDK batches "flag X evaluated by service Y" counts every minute → flag-service, so the UI shows where each flag is used and which flags are dead.
- **Testing:** `InMemoryProvider` + JUnit 5 extension `@FlagValue(key="...", value="true")`; local dev can point to a `flags-local.yaml` file provider (no GCP needed).

### 6.3 Cross-service consistency

A request crossing services A → B → C might see different snapshot versions for a few seconds during a change. If a feature *must* be consistent across a call chain, evaluate it once at the edge and forward the result via OpenTelemetry **baggage** (`ff.checkout.new-pricing-engine=on`); downstream SDKs honour baggage overrides for flags marked `propagate: true`.

## 7. Admin UI & workflow

- Small React/Angular app (or server-side Thymeleaf) served behind **Identity-Aware Proxy**; Google Groups → roles.
- Roles per environment: `viewer`, `editor`, `approver`. Prod edits on `RELEASE` flags → change request + second-person approval. `OPS` kill switches can be flipped immediately by on-call (still audited).
- Features: flag list w/ search & owner filter, rule editor with "evaluate as…" preview (enter a context, see the result), diff view, one-click rollback to any prior version, stale flag report.
- Scheduled changes (e.g. ramp 5% → 25% → 50% → 100% on a schedule) via Cloud Scheduler → flag-service.
- Also expose everything through the API + a small CLI / Terraform provider later so flags can live next to code reviews if desired.

## 8. Security

- Workload Identity: each service's GSA gets `storage.objectViewer` on its env snapshot bucket, `pubsub.subscriber` + subscription create/delete on `flags-changed-{env}`, and `run.invoker` (or internal LB auth) for the read API.
- Separate buckets/topics per environment; dev SDKs physically can't read prod.
- Read API exposes only what SDKs need; no secrets in flags (lint rule + UI validation).
- Audit: our `audit_log` table + Cloud Audit Logs on bucket/topic. Optionally stream audit events to BigQuery and to Slack (#flag-changes).

## 9. Failure modes

| Failure | Behaviour |
|---|---|
| flag-service down | Running pods keep last snapshot; new pods bootstrap from GCS. Only admin edits are blocked. |
| Pub/Sub delay/loss | Polling picks up the change within ≤ 30 s. |
| GCS + flag-service down at cold start | Code defaults (always choose the *safe/old* behaviour as the default). |
| Bad flag config pushed | Server-side schema validation; SDK rejects snapshots that fail to compile and keeps the previous one (+ alert). Rollback button. |
| Huge snapshot | Per-env snapshot compressed; alert if > 1 MB; segments with very large ID lists stored separately. |

## 10. Alternatives considered for propagation

- **Firestore snapshot listeners** instead of Postgres + Pub/Sub: realtime push for free, no per-instance subscriptions; weaker for relational queries/approvals, and every instance holds a listener. Good choice if the team prefers serverless and no Cloud SQL.
- **SSE/gRPC stream from flag-service**: lowest latency, but flag-service now holds a connection per pod and becomes a data-plane dependency.
- **Polling only**: simplest; 15–30 s latency is acceptable for most release flags. ← this is Phase 1.

## 11. Delivery plan

| Phase | Scope | Outcome |
|---|---|---|
| **0. Spike (1 wk)** | Run Unleash or flagd locally against one service to benchmark the UX; confirm build decision. Agree flag naming (`<domain>.<feature>`) and lifecycle policy. | Go/no-go on custom |
| **1. MVP (3–4 wks)** | flag-service with Postgres, boolean flags, env on/off + tenant allow-list, snapshot API + ETag, GCS snapshot, SDK with polling + OpenFeature provider, health/metrics, audit log, minimal UI behind IAP. Pilot on 1–2 services. | Toggle without restart (≤ 30 s) |
| **2. Real-time & targeting (2–3 wks)** | Pub/Sub hint + per-instance subscriptions, percentage rollouts, segments, multivariate/JSON flags, "evaluate as" preview, rollback. | < 5 s propagation, gradual rollouts |
| **3. Governance (2 wks)** | Prod approvals, scheduled ramps, Slack/BigQuery audit stream, usage reporting, stale-flag alerts, `@FeatureToggle` annotation, JUnit extension. | Safe at scale |
| **4. Adoption** | Migrate existing restart-based toggles; add to service template; docs & runbook. | All services on platform |

## 12. Conventions to agree on

- Default value in code = **current/safe behaviour**.
- Every `RELEASE` flag has an owner + expiry; removal ticket created on creation; CI warns on flags past expiry still referenced in code.
- Kill switches (`OPS`) are long-lived and documented in runbooks.
- No flag nesting deeper than one prerequisite; no flags inside hot loops without caching the result for the request.

## 13. Open questions

1. Runtime: GKE, Cloud Run, or both? (affects instance identity for subscriptions and connection limits)
2. Rough scale: number of services, instances per service, environments?
3. Is targeting needed beyond tenant-level (per-user, per-region, app version)?
4. Existing Cloud SQL / Firestore preference, and existing IdP groups for RBAC?
5. Are there non-Java consumers (Node, Python, front-ends) that will need SDKs later?
