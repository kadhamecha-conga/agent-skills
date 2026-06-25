---
name: panoptos-lgtm
version: 1.1.0
description: "Token-efficient Panoptos LGTM router for Claude. Use for Grafana, Loki logs, Mimir/Cortex metrics, Tempo traces, LogQL, PromQL, TraceQL, errors, latency, 5xxs, or production triage. Load topic files only when needed."
---

# Panoptos LGTM Router

Initial load should stay small. Do not read every topic file up front. Read only the file matching the current user request.

## Base Rules Always In Context

Use `grafana-engr` for Grafana/Loki/Mimir and `tempo-server` for direct trace analysis.

### Datasources

| Use for | UID | Name | Audience |
|---|---|---|---|
| App + infra logs | `loki` | Loki (Logs) | Platform team primary — most used, has all namespaces |
| Product-only logs | `loki-customer` | Loki (Customers) | Product teams — same data as `loki` but scoped for product use |
| Observability stack logs | `loki-infra` | Loki (Infra) | Panoptos infra only — LGTM components, OTel collectors |
| App metrics (Mimir) | `cortex` | Cortex (Metrics) | All product application metrics |
| Panoptos infra metrics | `prometheus` | Prometheus | Observability stack infra metrics only — skip for app queries |
| Traces | `tempo` | Tempo (Traces) | Use `tempo-server` MCP for direct trace analysis |

### Datasource Routing Decision

- User asks about **product/service errors, latency, logs** → `loki` (or `loki-customer`) + `cortex`
- User asks about **LGTM stack health** (cortex-ingester, loki-distributor, tempo, OTel collector) → `loki-infra` + `prometheus`
- User asks about **traces** → `tempo` via `tempo-server` MCP
- Default to `loki` + `cortex` unless explicitly about observability infra

Default time window for ordinary log triage: last 15 minutes, converted to absolute RFC3339 UTC. Do not pass `now-15m` to tools.

Always pin labels before querying. Avoid broad scans.

## Three-Axis Taxonomy

Every application signal (metrics in `cortex`, logs in `loki`/`loki-customer`) is scoped by three orthogonal labels:

| Axis | Label | User says | Examples |
|---|---|---|---|
| Product / domain | `service_namespace` | "product", "team", "domain" | `platform`, `revenue`, `ai-platform` |
| Service | `service_name` | "service", "API", "worker" | `platform-objectdb-api`, `revenue-quote-api` |
| Environment | `deployment_environment` | "env", "environment", "dev"→`rls03`, "qa"→`rls04` | `prod`, `rls04`, `rls03`, `staging` |

## Product Namespace Index (service_namespace)

Compact routing index. **Resolve exact services live** via `list_prometheus_label_values` (see Service Name Resolution) — don't trust hard-coded service lists. For deep per-namespace detail (sample services, product areas, counts), read `references/catalog.md`.

**Cortex metrics + Loki logs:** `platform` (~100 svc, largest, `platform-*`), `revenue`, `ai-platform`, `contracts`, `conga-sign`, `congasign`, `core-apps`, `xauthor`, `docgen`, `maf`, `esign`, `srm`, `conversations`, `plat-enablement`, `ccdatasync`

**Loki-only (no Cortex — use `count_over_time` for RED):** `cci`, `cci-standalone`, `clm`, `billing`, `invoicing`, `testauthor`, `approvals`, `tidb`

**Infra namespaces (NOT product — don't confuse):** `cluster-observability`, `sign`, `contractssf`, `argocd`, `argo-workflows`, `kube-system`, `cert-manager`, `karpenter`, `external-secrets`, `rls-operator`, `rls-app`, `kong`, `nginx-ingress`, `ingress-nginx`

**loki-infra datasource only:** `panoptos` (LGTM stack: cortex-*, loki-*, tempo-*, grafana), `cluster-observability` (otelcol-*, kube-prometheus-stack-*)

### Critical gotchas (keep in mind when routing)

- `sign` namespace = cluster infra for Sign clusters, **NOT** the Sign product (that's `conga-sign` / `congasign` — check both).
- `congasign` overlaps `conga-sign` — check both for Sign product signals.
- `docgen` has duplicate series with a `;otelcol-contrib` suffix — filter `service_name!~".*;otelcol-contrib"` to avoid double-counting.
- `kong` metrics live in `cortex` as `kong_http_requests_total` **without** `service_namespace` — filter by `route` label instead.
- `approvals` logs are in the `approvals` namespace (Loki-only), but its cortex metrics are under `core-apps`.
- `tidb` logs live in one shared home env per stack — discover with `sum by (deployment_environment) (count_over_time({service_namespace="tidb"}[5m]))`.

## Environment Catalog (deployment_environment)

### Environment Aliases (resolve these first)

When the user names an environment by its informal tier, map it to the literal `deployment_environment` value **before** querying. These aliases are authoritative:

| User says | Use `deployment_environment` |
|---|---|
| "dev", "development" | `rls03` |
| "qa", "QA" | `rls04` |
| "prod", "production" | `prod` |
| "perf", "performance" | `rls07` |
| "staging" | `staging` (Loki-only) |

If the user gives a literal `rls0x`/env value, use it as-is and skip aliasing.

### In Cortex

| Tier | Environments |
|---|---|
| Production | `prod` |
| QA | `rls04` |
| Dev | `rls03` |
| Staging / perf | `rls05`, `rls06`, `rls07` (perf) |
| Other dev | `dev`, `dev1`, `dev2`, `dev3`, `rlsdev`, `contractssf-dev`, `ephemeral`, `local` |
| CCI-specific | `cci-beta-1030`, `cci-legacy-stage`, `cci-yama-beta-1030` |
| Yama | `yama-dev-1100` |
| Azure | `rlsaz06` |
| CICD | `cicd` |

### Loki-Only Environments (not in Cortex)

`staging`, `contracts-dev`, `qa` (in loki-infra), `max-pu-xajs-engg-401`, `max-pu-xajs-kafka-401`

## Service Name Resolution

Users refer to services by informal shorthand (e.g. "objectdb", "connectcrane", "quote api", "extraction worker"). There are 400+ services across 17+ namespaces — do NOT guess the full `service_name` or which `service_namespace` it belongs to.

### One-Call Resolution Pattern

When the user gives a partial/informal service name, resolve it with a **single MCP call** — regex search on `service_name` without pinning any namespace:

```
# Find service_name AND its namespace in one call
list_prometheus_label_values(datasourceUid="cortex", labelName="service_namespace",
  matches=[{filters: [{name: "service_name", type: "=~", value: ".*<shorthand>.*"}]}])
```

This returns the `service_namespace` value(s) that contain matching services. Then if you need the exact service names:

```
# Get exact matching service names (same call, flip the labelName)
list_prometheus_label_values(datasourceUid="cortex", labelName="service_name",
  matches=[{filters: [{name: "service_name", type: "=~", value: ".*<shorthand>.*"}]}])
```

**Examples:**

| User says | Regex | Returns namespace | Returns service_name(s) |
|---|---|---|---|
| "objectdb" | `.*objectdb.*` | `platform` | `platform-objectdb-api`, `platform-objectdb-perftest-api` |
| "connectcrane" | `.*connectcrane.*` | `core-apps` | `connectcrane-web`, `connectcrane-worker` |
| "quote" | `.*quote.*` | `revenue` | `revenue-quote-api` |
| "extraction" | `.*extraction.*` | `ai-platform`, `cci` (if metrics exist) | `ai-platform-extraction-worker`, `cci-extraction-worker`, ... |

### Resolution Rules

1. **Always resolve before querying.** Never hard-code or guess a `service_name` from a shorthand.
2. **If multiple namespaces match** (e.g. "extraction" hits both `ai-platform` and `cci`), ask the user which product they mean.
3. **If zero matches in cortex**, the service may be Loki-only. Retry against Loki:
   ```
   topk(10, sum by (service_name, service_namespace)
     (count_over_time({service_name=~".*<shorthand>.*"}[15m])))
   ```
4. **If the user already specified the product** (e.g. "platform objectdb"), skip the namespace lookup and go straight to the service_name regex scoped to that namespace.

## Other Dynamic Discovery Patterns

```
# List all services for a known namespace
list_prometheus_label_values(datasourceUid="cortex", labelName="service_name",
  matches=[{filters: [{name: "service_namespace", type: "=", value: "<ns>"}]}])

# List environments where a service is active
list_prometheus_label_values(datasourceUid="cortex", labelName="deployment_environment",
  matches=[{filters: [{name: "service_name", type: "=", value: "<svc>"}]}])

# Discover metric families for a product
query_prometheus(datasourceUid="cortex", expr='group by (__name__) ({service_namespace="<ns>", deployment_environment="<env>"})', queryType="instant")
```

## Topic File Map

| User asks about | Read this file |
|---|---|
| Generic "what service is failing?" (no service named, no logs/metrics specified) | logs-first per "Failure triage" below — start with `references/loki-logql.md` |
| LogQL, log labels, error logs, traceid extraction, raw logs | `references/loki-logql.md` |
| PromQL, metrics, RED, latency, Kong/nginx 5xx, Kubernetes metrics | `references/mimir-promql.md` (check ingress rollout first — Kong vs nginx) |
| Trace IDs, Tempo, TraceQL, deeplinks, correlation | `references/tempo-traceql.md` |
| Long windows, 30 days, all errors over a week, timeout/truncation/max_samples | `references/wide-window-guard.md` |
| Deep namespace detail — exact sample services, product areas, per-namespace counts, full LGTM component list | `references/catalog.md` (but prefer live MCP discovery) |
| General triage flow or routing | stay in this file unless detail is needed |

## "Which service is failing?" — generic failure triage

When the user asks the open-ended question *"which service is failing / has more errors / is broken"* without naming a service or specifying logs vs metrics:

**Always logs-first, then metrics.** Reason: log-based error counts capture every failure mode (workers, async jobs, app-level exceptions returning HTTP 200, Loki-only namespaces like `cci`/`clm`/`billing`/`invoicing`/`docgen`/`testauthor`/`tidb`). Ingress metrics only see HTTP-fronted services that transit the ingress.

1. **Loki first** — `topk by service_name` on `SeverityText=~"ERROR|Error|FATAL"` scoped to the env. See `references/loki-logql.md` query D (and its MCP-wrapper gotcha — pull raw lines + tally with jq).
2. **Then ingress metrics** to confirm severity (ratio, rate) and add HTTP-status context. **First detect which ingress is live in this env** — see Ingress rollout note below. Then see `references/mimir-promql.md` queries E/F.
3. **Cross-reference the two lists**:
   - Services topping **both** → real fires, dig in.
   - **Logs-only** top → may be log-spam or non-HTTP failures (workers, jobs).
   - **Metrics-only** top → HTTP-layer failures (timeouts, upstream 5xx) the app isn't logging as ERROR.

### Ingress rollout note (Kong vs nginx)

Kong is being rolled out as the ingress proxy. Some envs are on Kong, others still on nginx, and a few may be mid-cutover with both active. **Never assume Kong globally.** Before running any ingress-5xx query, detect the live ingress for the target `deployment_environment`. See `references/mimir-promql.md` § "Ingress detection (Kong vs nginx)".

## Triage Workflow

1. Scope to `deployment_environment` plus `service_name` or `service_namespace` and a 5-15 minute window.
2. If the question is generic ("what's failing?"), follow the logs-first failure triage above.
3. If logs are needed, read `references/loki-logql.md`.
4. If metrics are needed, read `references/mimir-promql.md`.
5. If a trace ID is available or Tempo is needed, read `references/tempo-traceql.md`.
6. If the requested window is wide, read `references/wide-window-guard.md` before querying.

## Cost Guardrails

- Default log limit: 100.
- Raw Loki lines over 24h: refuse raw fetch; aggregate first.
- Cortex range queries over 24h need careful step sizing.
- Do not run global metric discovery such as `{__name__=~".+"}`.
- If a query returns empty unexpectedly, do one narrow discovery query before concluding no data.

## Sharing & Upload Note

This skill is self-contained — the `references/` subfolder holds all topic files referenced above. Zip the whole `panoptos-lgtm/` folder to share with teammates or upload as a `.skill` bundle. Topic files load only when relevant, keeping the initial context small.
