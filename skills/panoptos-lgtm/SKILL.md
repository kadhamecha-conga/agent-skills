---
name: panoptos-lgtm
version: 1.0.0
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
| Environment | `deployment_environment` | "env", "environment" | `prod`, `rls03`, `staging`, `dev` |

## Product Namespace Catalog (service_namespace)

### Business Products — Have Cortex Metrics + Loki Logs

| Namespace | Product area | Sample services | Notes |
|---|---|---|---|
| `platform` | Core platform (auth, objectdb, scheduler, search, email, pubsub, datasync, workflows, recyclebin, custom code, licensing, onboarding) | `platform-objectdb-api`, `platform-auth-api`, `platform-search-api`, `platform-scheduler-api`, `platform-email-api`, `platform-datasync-api`, `platform-messaging-api` | ~100 services. Largest namespace. Service names prefixed `platform-*` |
| `revenue` | Revenue management (CPQ, pricing, catalog, orders, renewals, rebates, incentives, config-publisher) | `revenue-quote-api`, `revenue-catalog-api`, `pricing-engine`, `config-engine`, `order-api`, `cart-api` | ~37 services. Mix of `revenue-*` prefixed and standalone names |
| `ai-platform` | AI/ML platform (copilot, extraction, OCR, vectorization, redline, agent services) | `ai-platform-db-api`, `ai-platform-config-service`, `conga-copilot-backend-api`, `ai-platform-usage-api-services` | ~9 services with cortex metrics. Many more workers are Loki-only |
| `contracts` | Contracts lifecycle (batch, cycletime, email-parsing, onboarding) | `conga-contracts-api`, `conga-contracts-batch-worker`, `conga-contracts-lifecycle-worker` | ~7 services. Prefixed `conga-contracts-*` |
| `conga-sign` | Conga Sign product (signing, connectors, callbacks, audit, upgrades) | `sign-api`, `sign-worker`, `sign-audit-api`, `sign-callback-api`, `conga-sign-connector-worker` | ~8 services |
| `congasign` | Legacy Conga Sign namespace | `conga-sign-audit-worker`, `signcallback-worker` | ~3 services. Overlaps with `conga-sign`; check both |
| `core-apps` | Composer, Approvals, Drive, integrations (box, sharepoint, sftp, webhook, workflow) | `approvals-web`, `approvals-worker`, `congadrive-web`, `composerapi`, `boxbasilisk-web` | ~25 services. Animal-themed names (boxbasilisk, datadolphin, workflowwhale, etc.) |
| `xauthor` | XAuthor (Excel/Word/GDocs authoring, templates, review) | `conga-xauthor-api`, `conga-xauthor-gdocs-api`, `conga-review-api`, `conga-xauthor-template-management-api` | ~6 services |
| `docgen` | Document generation (composer, conductor, query builder, scheduler, workflow) | `composerapi`, `conductor`, `core`, `querybuilder`, `weball`, `workflow` | ~10 real services. ⚠️ Has duplicates with `;otelcol-contrib` suffix — filter with `service_name!~".*;otelcol-contrib"` to avoid double-counting |
| `maf` | Managed app framework (billing, invoicing, orders within MAF) | `billing-management`, `invoice-management`, `maf-data-change-worker`, `order-api` | ~7 services |
| `esign` | E-sign integration | `conga-esign-api`, `conga-esign-worker` | 2 services |
| `srm` | Supplier relationship management | `rls-srm-api`, `rls-srm-worker` | 2 services |
| `conversations` | Conversations product | `conga-conversations-api` | 1 service |
| `plat-enablement` | Platform enablement / migration hub | `migrationhub-batch-processor`, `migrationhub-data-ingestion` | 2 services |
| `ccdatasync` | Contracts datasync | `conga.contracts.datasync.api`, `conga-contracts-datasync-worker` | 2 services |

### Business Products — Loki-Only (No Cortex Metrics)

For these, use Loki `count_over_time` for RED-style signals. Query `loki` or `loki-customer`.

| Namespace | Notes |
|---|---|
| `cci`, `cci-standalone` | Conga Contract Intelligence. Many workers (extraction, OCR, search, polling). No cortex metrics at all |
| `clm` | Contract Lifecycle Management |
| `billing` | Billing (has `Billing Management` service_name in cortex under `billing` namespace but very thin) |
| `invoicing` | Invoicing (has `Invoice Management` in cortex but very thin) |
| `testauthor` | Test authoring |
| `approvals` | Approvals (Loki logs only; cortex metrics are under `core-apps` namespace for approvals-web/worker) |
| `tidb` | TiDB shared database. Logs live in one shared home env per stack; discover with `sum by (deployment_environment) (count_over_time({service_namespace="tidb"}[5m]))` |

### Infra Namespaces (Not Product — Don't Confuse)

| Namespace | What it is |
|---|---|
| `cluster-observability` | OTel collectors, prometheus-proxy. In both `loki` and `loki-infra` |
| `sign` | ⚠️ Cluster-level infra for Sign clusters (kube-state-metrics, node-exporter, nginx, etc.) — NOT the Sign product. The Sign product is `conga-sign` or `congasign` |
| `contractssf` | Cluster-level K8s metrics (apiserver, kubelet, kube-state-metrics) for ContractsSF clusters — NOT product application metrics |
| `argocd`, `argo-workflows` | GitOps / CI-CD infra |
| `kube-system`, `cert-manager`, `karpenter`, `external-secrets` | Kubernetes system components |
| `rls-operator`, `rls-app` | RLS cluster operators |
| `kong`, `nginx-ingress`, `ingress-nginx` | Ingress layer. Kong metrics exist in `cortex` but under `kong_http_requests_total` without `service_namespace` — use `route` label instead |

### Loki-Infra Namespace (loki-infra datasource only)

| Namespace | Services |
|---|---|
| `panoptos` | LGTM stack components: `cortex-distributor`, `cortex-ingester`, `cortex-compactor`, `cortex-querier`, `cortex-query-frontend`, `cortex-store-gateway`, `cortex-ruler`, `cortex-alertmanager`, `cortex-nginx`, `loki-distributor`, `loki-ingester`, `loki-querier`, `loki-compactor`, `loki-gateway`, `loki-query-frontend`, `loki-ruler`, `loki-index-gateway`, `loki-infra-*` (self-monitoring variants), `tempo-distributor`, `tempo-ingester`, `tempo-querier`, `tempo-compactor`, `tempo-gateway`, `tempo-query-frontend`, `tempo-metrics-generator`, `grafana`, `grafana-mcp-ops` |
| `cluster-observability` | `otelcol-logs`, `otelcol-metrics`, `otelcol-traces`, `internal-agent-ot-collector-contrib`, `kube-prometheus-stack-*`, `kong-proxy`, `kong-grpc-proxy` |

## Environment Catalog (deployment_environment)

### In Cortex

| Tier | Environments |
|---|---|
| Production | `prod` |
| QA / Staging | `rls04` (QA), `rls03` (dev), `rls05`, `rls06`, `rls07` (perf) |
| Dev | `dev`, `dev1`, `dev2`, `dev3`, `rlsdev`, `contractssf-dev`, `ephemeral`, `local` |
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
