# Feature Flag Platform — Design Plan

Status: Draft v3 · Target: ~40 Spring Boot microservices on GKE, 6 environments (incl. prod)

## 0. Decisions so far

| Topic | Decision |
|---|---|
| Runtime | GKE (Workload Identity Federation for GKE) |
| Scale | ~40 services, 6 environments |
| Projects | Separate **non-prod** and **prod** GCP projects; each environment is a **namespace** inside one of them (see §4.1) |
| App version | Set by GitOps/Terraform at deploy time → injected into pods as `APP_VERSION` (see §5.1) |
| Distribution | Starter is pulled in through the **shared core module**, so all 40 services get it from one change |
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

### 4.1 Placement across projects and namespaces

Firestore IAM works per database, not per collection. If pods had read access to the admin database, every service could read all flags, rules, segments and the audit log for every environment. So there are two kinds of database:

- `flags-admin`: the source of truth. Only flag-service's service account has access.
- `flags-runtime`: holds only the tiny `heads/{env}` documents. Workloads get `roles/datastore.viewer` on this database only (IAM condition on the database resource name). A head doc reveals only a version number and a path, which is harmless.
- The actual flag content lives in **one GCS bucket per environment**.

Where each piece lives:

| Resource | Non-prod project | Prod project |
|---|---|---|
| flag-service + `flags-admin` DB | – | ✔ (one admin plane for all envs; it writes to non-prod across projects) |
| `flags-runtime` DB | ✔ heads for non-prod envs | ✔ heads for prod-project envs |
| Snapshot buckets | `…-flags-{env}` for each non-prod env | `…-flags-{env}` for each prod-project env |

Rules:
- **Prod pods depend only on resources in the prod project.** A non-prod outage or misconfiguration can't affect prod.
- flag-service sits in the prod project because that is the most locked-down place, and it is the only thing that can write to prod. Its service account gets write access to the non-prod buckets and the non-prod `flags-runtime` database. A small dedicated `tools`/`flags-admin` project is a cleaner alternative if your org allows creating one.
- One admin plane means one UI, one audit log, and "promote qa → stage → prod" across projects.

**Namespace isolation.** Several environments share a project (and probably a cluster), so the boundary between them is the **Kubernetes namespace**. With Workload Identity Federation for GKE, we grant bucket access directly to *every pod in a namespace*, with no Google service accounts to create:

```hcl
resource "google_storage_bucket_iam_member" "flags_reader" {
  bucket = google_storage_bucket.flags["qa"].name
  role   = "roles/storage.objectViewer"
  member = "principalSet://iam.googleapis.com/projects/${var.project_number}/locations/global/workloadIdentityPools/${var.project_id}.svc.id.goog/namespace/qa"
}
```

So pods in the `dev` namespace can read only the dev bucket, even inside the same project and cluster.

> ⚠ Check: this only works if pods use Workload Identity. If any node pool still runs pods as the **node's default service account** (`GKE_METADATA` not enabled on the node pool), all namespaces on that node share one identity and the isolation is lost. Verify with
> `gcloud container node-pools describe POOL --cluster CLUSTER --region REGION --format='value(config.workloadMetadataConfig.mode)'` → must be `GKE_METADATA`.

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
| `appVersion` | `APP_VERSION` env var set by GitOps/Terraform (fallback: Spring Boot `BuildProperties`, then `unknown`) |
| `region`, `zone` | GKE metadata server / node topology labels via downward API |
| `cluster`, `namespace`, `pod` | Downward API env vars |
| `env` | Starter config (`flags.env`) |

**App version from GitOps.** The version is decided in your Terraform, so we pass it straight into the pod rather than relying on the build. In the shared deployment module, add one env var (and the standard label, so dashboards can use it too):

```hcl
# in the Terraform module that renders each service's Deployment
env {
  name  = "APP_VERSION"
  value = var.app_version          # the same value already used for the image tag
}
env {
  name  = "FLAGS_ENV"
  value = var.environment          # dev | qa | ... | prod
}
labels = { "app.kubernetes.io/version" = var.app_version }
```

Because this lives in the shared deployment module, all 40 services get it without per-service changes. If the version doesn't look like semver (e.g. a git SHA), rules use `EQ/IN` instead of `SEMVER_*`, and the UI shows which versions are currently reporting in (from usage reports).

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

**Rollout through the core module.** The starter is its own artifact (`flags-spring-boot-starter`), and the shared core module declares it as a dependency. When services upgrade core, they get flags. The defaults live in the starter, so services need **no config of their own**:

```yaml
# defaults shipped inside the starter; overridable per service
flags:
  enabled: true
  env: ${FLAGS_ENV:}                          # from the Deployment (Terraform)
  project: ${FLAGS_PROJECT:${GOOGLE_CLOUD_PROJECT:}}
  bucket: ${FLAGS_BUCKET:yourco-flags-${FLAGS_ENV}}
  runtime-database: flags-runtime
  poll-interval: 120s
  startup-timeout: 5s
```

- If `FLAGS_ENV` is missing (local runs, services not yet redeployed with the new Terraform), the starter logs a warning and uses code defaults plus an optional `flags-local.yaml`. **Pulling in core never breaks a service.**
- **Dependency risk:** the Firestore and GCS clients bring gRPC, protobuf and Guava. Pin them with `com.google.cloud:libraries-bom` in core's dependency management, and run all 40 services' builds/tests against the new core before release. This is the main integration risk.

Usage in a service:

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

- Login: **IAP (Identity-Aware Proxy)** in front of flag-service's GKE Ingress/Gateway, with groups mapping to roles per env: `viewer`, `editor`, `approver`. This is pending confirmation (§7.1).
- Approval required for `RELEASE` flag changes in prod (configurable per env via `environments/{env}.requiresApproval`). `OPS` kill switches can be flipped immediately by on-call, but are still audited.
- Features:
  - Flag list with filters: owner, app, env, stale.
  - Rule editor with **"evaluate as…"**: pick app, version and region and see the result.
  - Diff before save.
  - **Promote config across envs** (e.g. qa → stage) with a diff.
  - One-click rollback to any snapshot version.
- Scheduled ramps (5% → 25% → 100%) via Cloud Scheduler → flag-service.
- Audit stream: a Firestore trigger (Eventarc) on `auditLog` → Pub/Sub → Slack `#flag-changes` and BigQuery.

### 7.1 Choosing admin login (IAP or not)

IAP is a Google Cloud feature that puts a Google sign-in in front of a web app, so flag-service never handles passwords. Whether it fits depends on how your engineers sign in to the GCP console today. Ask your platform/cloud admin, or check:

```bash
# 1. Which account type do people use? Company Google accounts (Workspace / Cloud Identity)
#    show up as members like user:alice@yourco.com
gcloud projects get-iam-policy PROD_PROJECT --format='value(bindings.members)' | tr ';' '\n' | sort -u | head

# 2. Do Google Groups exist? (needs org-level read access)
gcloud organizations list
gcloud identity groups search --organization=ORG_ID \
  --labels="cloudidentity.googleapis.com/groups.discussion_forum" --page-size=20

# 3. Is IAP already used for any internal tool? (existing pattern = easy approval)
kubectl get backendconfig -A -o yaml | grep -i -A3 iap         # GKE Ingress style
kubectl get gcpbackendpolicy -A -o yaml | grep -i -A3 iap       # Gateway API style
gcloud iap web get-iam-policy --project=PROD_PROJECT 2>/dev/null

# 4. Federated from Okta / Azure AD / other? (Workforce Identity Federation)
gcloud iam workforce-pools list --location=global --organization=ORG_ID
```

| What you find | Login choice |
|---|---|
| Company Google accounts + groups (1–2 show results) | **IAP + Google Groups**. Simplest; no auth code in flag-service. |
| IAP already used elsewhere (3) | Same; reuse that team's setup and approval path. |
| Okta / Azure AD via workforce federation (4) | IAP still works with workforce identity; groups come from the IdP. |
| None of the above / unsure | **Spring Security OIDC** in flag-service against your corporate IdP (Okta/Azure AD/Keycloak), roles from IdP group claims. Works anywhere, just more code. |

Either way, flag-service's authorization layer (who may edit which env) is the same; only the login front-door differs. Phase 1 can start before this is decided.

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
| **0. Spike (1 wk)** | Benchmark Unleash or flagd on one service. Agree naming (`<domain>.<feature>`), lifecycle policy and env names. Terraform: Firestore DBs + buckets in both projects, namespace-level IAM, `APP_VERSION`/`FLAGS_ENV` in the shared deployment module. Check Workload Identity on node pools; decide admin login (§7.1). | Go/no-go; infra ready |
| **1. MVP (3–4 wks)** | flag-service + `flags-admin`; boolean flags; env on/off + **region / app / appVersion** rules; snapshot publisher + reconciler; starter with listener + poll + OpenFeature provider + health/metrics; audit log; minimal UI behind IAP. Starter added to core module (off-by-default if `FLAGS_ENV` unset); pilot on 2 services in non-prod. | Toggle without restart, < 5 s |
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

1. **Exact env → project/namespace map.** Two projects × two namespaces gives 4 environments, but we counted 6. Please list each env with its project and namespace (e.g. `dev → nonprod/dev`, …, `prod → prod/prod`). Is there anything non-prod (e.g. preprod) living in the prod project?
2. Is one GKE cluster per project shared by all its namespaces, and do all node pools run with Workload Identity (`GKE_METADATA`)? See the check in §4.1.
3. What does the version value look like in Terraform: semver (`2.14.0`) or a git SHA / build number?
4. Admin login: results of the checks in §7.1.
5. Is the core module built with Maven or Gradle, and does it already import `libraries-bom` / `spring-cloud-gcp-dependencies`?
