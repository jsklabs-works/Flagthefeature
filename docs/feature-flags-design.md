# Feature Flag Platform — Design Plan

Status: Draft v2 · Target: ~40 Spring Boot microservices on GKE, 6 environments (incl. prod)

## 0. Decisions so far

| Topic | Decision |
|---|---|
| Runtime | GKE (Workload Identity Federation for GKE) |
| Scale | ~40 services, 6 environments |
| Store | **Firestore** (Native mode) |
| Targeting | Environment, **region**, **app (service)** and **app version**, plus tenant/user later |
| Other languages | Not now, but the snapshot format and evaluation rules must be language-neutral |
| API standard | Services use the **OpenFeature** Java API; our platform plugs in as a provider |

## 1. Problem

- Several features are developed in parallel, but turning them on or off means a config change and a **restart/redeploy**.
- We need runtime toggles per environment, region and app/version, with an audit trail and a fast kill switch.

## 2. Goals / Non-goals

**Goals**
1. A flag change reaches every running pod in **< 5 s** (target), < 2 min worst case, with no restart.
2. Flag checks are **local and in memory**: no network call per check.
3. Services keep working if the flag platform is down: last-known values, then the GCS snapshot, then code defaults.
4. Targeting on region, app, app version and segments; sticky percentage rollouts.
5. Audit of who changed what, when and why. Prod changes can require approval.
6. One starter dependency for Spring Boot.

**Non-goals (v1)**
- Experimentation analytics.
- Non-Java SDKs (designed for, built later; see §9).
- Replacing regular app config (DB URLs, timeouts, secrets).

## 3. Why custom, on OpenFeature

Spring Cloud Config with `@RefreshScope` avoids restarts but has no targeting, rollouts or audit, and it re-creates beans on refresh. SaaS (LaunchDarkly etc.) and self-hosted OSS (Unleash, Flagsmith, flagd) are mature; a one-week spike against one of them is the benchmark for this build.

**Decision:** build custom, but application code depends only on `dev.openfeature:sdk`. Our platform is an OpenFeature *Provider*, so it can be swapped later without touching feature code. This also gives us ready-made OpenFeature SDKs for other languages later.

## 4. Architecture

```
┌──────────────────────────── Admin plane (prod project / shared tools project) ───────────┐
│                                                                                          │
│  Engineers ──IAP──► Flag Admin UI ──► flag-service (Spring Boot on GKE, 2+ replicas)     │
│                                          │  CRUD, validation, approvals, rollback        │
│                                          ▼                                               │
│                         Firestore DB "flags-admin"  (only flag-service can read/write)   │
│                           flags, envConfigs, segments, auditLog, changeRequests          │
│                                          │                                               │
│                                 Snapshot publisher (inside flag-service)                 │
│                         ┌────────────────┴───────────────────┐                           │
│                         ▼                                    ▼                           │
│        GCS gs://<org>-flags-{env}/snapshots/{v}.json   Firestore DB "flags-runtime"      │
│        (one bucket per env, immutable versions)        heads/{env} = {version, path,     │
│                                                                 sha256, publishedAt}    │
└─────────────────────────┬────────────────────────────────────┬───────────────────────────┘
                          │ read snapshot                      │ realtime listener (gRPC stream)
┌─────────────────────────┴──── Data plane: every pod ─────────┴───────────────────────────┐
│  flags-spring-boot-starter                                                               │
│   • startup: read heads/{env} → download snapshot from GCS → verify sha256 → compile      │
│   • listen: Firestore snapshot listener on heads/{env}; new version → fetch & swap        │
│   • poll:   every 120 s get heads/{env} (safety net if the listener stalls)               │
│   • AtomicReference<CompiledSnapshot> + local evaluator → OpenFeature provider            │
│   • static context: app, appVersion, region, zone, cluster, namespace (auto-detected)     │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

Key property: **flag-service is not in the data plane.** Pods depend only on Firestore and GCS, which are both managed and highly available. If flag-service is down, only editing is blocked.

### 4.1 Why two Firestore databases

Firestore IAM works per database, not per collection. If pods had read access to the admin database, every service could read all flags, rules, segments and the audit log for every environment.

- `flags-admin`: the source of truth. Only flag-service's service account has access.
- `flags-runtime`: holds only the tiny `heads/{env}` documents. All workloads get `roles/datastore.viewer` on this database only (IAM condition on the database resource name). A head doc reveals only a version number and a path, which is harmless.
- The actual flag content lives in **per-environment GCS buckets**. Each env's workloads get `storage.objectViewer` on their own bucket only, so a dev pod cannot read prod flags.

Put the prod bucket (and ideally prod's `flags-runtime` database) in the prod project if your projects are split by environment.

### 4.2 Change flow

1. An editor saves a change. flag-service validates it (schema, rule compilation, no unknown segments).
2. **Firestore transaction**: write the flag's env config, append an `auditLog` entry, and increment `environments/{env}.pendingVersion`.
3. The publisher builds the full env snapshot (all flags + segments for that env), writes `snapshots/{v}.json` and computes its sha256.
4. It updates `heads/{env}` in `flags-runtime` to `{version: v, path, sha256}`.
5. Pod listeners fire within about a second, download the object, verify the hash and swap in the new snapshot atomically.

**Crash safety:** if flag-service dies between steps 2 and 4, a reconciler (runs on startup and every minute) republishes any env where `pendingVersion > publishedVersion`. Snapshots are immutable and versioned, so a **rollback** simply points `heads/{env}` back at an older version (and records that in the audit log).

### 4.3 Load and cost (rough)

Assume about 40 services × ~10 pods × 6 envs ≈ 2,400 pods.
- Listener: one document read per pod per flag change. Negligible.
- Safety-net poll every 120 s: about 1.7M reads/day across all envs, roughly tens of dollars a month at Firestore list prices. Increase the interval if needed; the listener is the primary path.
- GCS downloads only happen on a version change.
- Firestore's write limit on a single document (about 1/s sustained) is far above how often people edit flags.

## 5. Flag data model

### 5.1 Evaluation context

The starter fills these in automatically at startup:

| Attribute | Source |
|---|---|
| `app` | `spring.application.name` |
| `appVersion` | `BuildProperties` (Spring Boot build-info) or the image tag via env var |
| `region`, `zone` | GKE metadata server / node topology labels via downward API |
| `cluster`, `namespace`, `pod` | Downward API env vars |
| `env` | Starter config (`flags.env`) |

Per-request attributes (`tenantId`, `userId`, …) come from a servlet/WebFlux filter reading the JWT or headers into OpenFeature's transaction context. They are optional today, but the model supports them.

### 5.2 Flag document

```jsonc
// flags-admin: flags/{key}
{
  "key": "checkout.new-pricing-engine",
  "type": "BOOLEAN",                // BOOLEAN | STRING | NUMBER | JSON
  "kind": "RELEASE",                // RELEASE | OPS (kill switch) | PERMISSION
  "owner": "team-payments",
  "description": "Routes pricing to v2 engine",
  "expiresAt": "2027-01-31",
  "apps": ["order-service", "pricing-service"],   // optional: only these apps receive the flag
  "variants": { "on": true, "off": false }
}

// flags-admin: flags/{key}/envs/{env}
{
  "enabled": true,                  // master switch; false => offVariant for everyone
  "offVariant": "off",
  "rules": [                        // first match wins
    { "id": "eu-pilot",
      "if": [ { "attr": "region", "op": "IN", "values": ["europe-west1"] },
              { "attr": "appVersion", "op": "SEMVER_GTE", "values": ["2.14.0"] } ],
      "serve": "on" },
    { "id": "us-ramp",
      "if": [ { "attr": "region", "op": "STARTS_WITH", "values": ["us-"] } ],
      "rollout": { "bucketBy": "pod", "weights": { "on": 25, "off": 75 } } }
  ],
  "defaultServe": "off",
  "updatedBy": "alice@yourco.com", "updatedAt": "..."
}
```

Operators: `EQ, NEQ, IN, NOT_IN, STARTS_WITH, MATCHES, GT/LT, SEMVER_EQ/GTE/LT/RANGE, IN_SEGMENT`.
**Segments** are reusable named rules, e.g. `eu-regions`, `canary-pods`, `beta-tenants`.

**Rollouts:** `bucket = murmur3_32(flagKey + ":" + salt + ":" + ctx[bucketBy]) mod 10000`. This is deterministic and sticky, and raising 25% to 50% only adds members.
- `bucketBy: pod` gives a gradual rollout *across instances*, which is useful when there is no user context.
- `bucketBy: tenantId` / `userId` is for user-facing rollouts later.

Snapshot pruning: the starter receives the whole env snapshot, but flags with an `apps` list are skipped for other apps at compile time. Alternatively, the publisher can write per-app snapshots if the size ever matters.

### 5.3 Other collections (flags-admin)

```
environments/{env}           pendingVersion, publishedVersion, requiresApproval
segments/{key}/envs/{env}    rules
auditLog/{autoId}            env, flagKey, actor, action, before, after, reason, ticket, at
changeRequests/{id}          env, flagKey, proposed, requestedBy, approvedBy, status
usage/{env}_{app}_{flagKey}  lastEvaluatedAt, count   (from SDK usage reports)
```

## 6. Client SDK: `flags-spring-boot-starter`

### 6.1 Usage

```yaml
# application.yml (env usually injected via ConfigMap per namespace)
flags:
  env: ${FLAGS_ENV}          # dev | qa | perf | stage | preprod | prod (your 6)
  bucket: yourco-flags-${FLAGS_ENV}
  runtime-database: flags-runtime
  poll-interval: 120s
```

```java
@Service
class PricingService {
  private final Client flags;   // dev.openfeature.sdk.Client, auto-configured

  Price price(Order o) {
    // default in code = current safe behaviour
    if (flags.getBooleanValue("checkout.new-pricing-engine", false)) {
      return v2.price(o);
    }
    return v1.price(o);
  }
}
```

Optional sugar (Phase 3): `@FeatureToggle(value = "...", fallbackMethod = "...")`, and a flag-driven *strategy router* bean that picks between two implementations per call without a context refresh.

### 6.2 Internals

- `AtomicReference<CompiledSnapshot>`: rules are pre-compiled on load (regex compiled, `IN` lists → `HashSet`, semvers parsed). Evaluation is lock-free and allocates almost nothing.
- Static context (app, version, region…) is resolved once. Rules that use only static attributes can be **pre-evaluated at snapshot load**, so most checks become a map lookup.
- Firestore Java client `addSnapshotListener` on `heads/{env}`, with automatic reconnect. A poll every 120 s is the safety net. Versions are applied only if `v > current`.
- Snapshots that fail hash verification or compilation are rejected; the previous one is kept and a metric/alert fires.
- Startup timeout (default 5 s): if the snapshot is unavailable, the app **starts anyway on code defaults** and keeps retrying in the background.
- **Health indicator** `flags: {version, ageSeconds, source}` reports UNKNOWN/degraded when stale, never DOWN, so a flag outage never fails readiness probes.
- **Micrometer → Cloud Monitoring:** `flags.snapshot.version`, `flags.snapshot.age`, `flags.refresh.failures`, `flags.evaluations{flag,variant}` (no user IDs as tags).
- **Usage reporting:** every 5 min, aggregated `{flag, variant, count}` goes to flag-service, which feeds "where is this flag used" and dead-flag detection.
- **Testing:** `InMemoryProvider` + JUnit 5 `@FlagValue(key=..., value=...)`. For local dev, a `flags-local.yaml` file provider means no GCP access is needed.

### 6.3 Cross-service consistency

During a change, a request going A → B → C may briefly see different versions. For flags marked `propagate: true`, evaluate once at the edge and pass the value in OpenTelemetry baggage; downstream providers honour it.

## 7. Admin UI & workflow

- Served by flag-service behind **IAP**. Google Groups map to roles per env: `viewer`, `editor`, `approver`.
- Approval required for `RELEASE` flag changes in prod (configurable per env via `environments/{env}.requiresApproval`). `OPS` kill switches can be flipped immediately by on-call, but are still audited.
- Features:
  - Flag list with filters: owner, app, env, stale.
  - Rule editor with **"evaluate as…"**: pick app, version and region and see the result.
  - Diff before save.
  - **Promote config across envs** (e.g. qa → stage) with a diff.
  - One-click rollback to any snapshot version.
- Scheduled ramps (5% → 25% → 100%) via Cloud Scheduler → flag-service.
- Audit stream: a Firestore trigger (Eventarc) on `auditLog` → Pub/Sub → Slack `#flag-changes` and BigQuery.

## 8. Failure modes

| Failure | Behaviour |
|---|---|
| flag-service down | No impact on pods; editing is blocked. |
| Firestore listener drops | Client auto-reconnects; the 120 s poll catches up. |
| Firestore unavailable | Pods keep their current snapshot. New pods can fall back to `latest.json` in GCS (written alongside each version). |
| GCS + Firestore unavailable at cold start | Code defaults; retries in the background. |
| Bad config published | Server-side validation; SDK rejects snapshots that don't compile; one-click rollback. |
| Snapshot grows large | Alert at 1 MB; switch to per-app snapshots. |

## 9. Designing for other languages (later)

- **JSON Schema** for the snapshot format, versioned (`"schemaVersion": 1`).
- A **shared conformance suite**: JSON test vectors (`context + snapshot → expected variant`, including hash buckets). The Java SDK must pass it from day one, so Node/Python/Go providers can be validated against the same file.
- Server-side evaluation for front-ends and anything without an SDK: flag-service exposes **OFREP** (OpenFeature Remote Evaluation Protocol). Existing OpenFeature OFREP providers for web, Node, Python, Go and .NET can then call it with no custom SDK at all.

## 10. Delivery plan

| Phase | Scope | Outcome |
|---|---|---|
| **0. Spike (1 wk)** | Benchmark Unleash or flagd on one service. Agree naming (`<domain>.<feature>`), lifecycle policy and env names. Terraform for the Firestore DBs, buckets and IAM. | Go/no-go; infra ready |
| **1. MVP (3–4 wks)** | flag-service + `flags-admin`; boolean flags; env on/off + **region / app / appVersion** rules; snapshot publisher + reconciler; starter with listener + poll + OpenFeature provider + health/metrics; audit log; minimal UI behind IAP. Pilot on 2 services in non-prod. | Toggle without restart, < 5 s |
| **2. Rollouts (2 wks)** | Percentage rollouts, segments, string/number/JSON flags, "evaluate as", rollback, promote-across-envs, conformance test vectors. | Gradual, safe rollouts |
| **3. Governance (2 wks)** | Prod approvals, scheduled ramps, Slack/BigQuery audit stream, usage reporting, stale-flag alerts + CI check, `@FeatureToggle`, JUnit extension. | Safe at 40-service scale |
| **4. Adoption (ongoing)** | Add the starter to the service template; migrate existing restart-based toggles service by service; runbook. | All 40 services onboarded |
| **5. Later** | OFREP endpoint; Node/Python providers using the conformance suite; tenant/user targeting. | Multi-language |

## 11. Conventions

- Default in code = **current/safe behaviour**.
- Every `RELEASE` flag has an owner and an expiry. CI warns when an expired flag key is still referenced in code.
- `OPS` kill switches are long-lived and listed in runbooks.
- At most one prerequisite level; no complex logic hidden in flag rules.

## 12. Remaining open questions

1. Are the 6 environments in separate GCP projects, or one project with separate namespaces/clusters? (This decides where the buckets and `flags-runtime` live.)
2. Is `appVersion` reliably available (Spring Boot build-info or image tag in an env var) for all 40 services?
3. Is there a shared parent POM / Gradle platform where the starter can be added once?
4. Is IAP + Google Groups acceptable for admin auth, or is there a different corporate IdP?
