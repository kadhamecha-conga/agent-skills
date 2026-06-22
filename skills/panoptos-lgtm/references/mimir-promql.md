# Mimir / Cortex Query Debug — Panoptos

Companion skill to [`lgtm`](../SKILL.md). Loaded only when the work is specifically PromQL / Cortex / Mimir.

## Datasource routing <!-- tags: ds-routing, cortex, mimir-vs-prometheus -->

| Scope | UID | Notes |
|---|---|---|
| RLS-style envs (`rls*`, `cci-*`, `yama-*`, `contracts-*`) — app + infra | `cortex` | "Cortex (Metrics)", actually Mimir. |
| Panoptos platform itself + non-RLS infra | `prometheus` | default DS. |

They hold **disjoint** data. Don't cross-query. Don't fall back.

⚠ `prod` and `staging` exist in Loki but **not in `cortex`** — check `prometheus` UID before concluding "no metrics".

## `service_namespace` — the product axis <!-- tags: namespace-axis, service_namespace, product, namespace-list -->

~40 distinct values. Group mentally:

| Group | values |
|---|---|
| **Core products** | `platform`, `revenue`, `clm`, `billing`, `invoice`, `docgen`, `sign`, `congasign`, `contracts`, `contractssf`, `cci`, `cci-standalone`, `maf`, `conversations`, `core-apps`, `xauthor`, `testauthor` |
| **Platform / shared** | `ai-platform`, `conga-registry`, `cluster-observability`, `rls-app`, `rls-operator` |
| **Infra / system** | `argo-events`, `argo-workflows`, `argocd`, `cattle-fleet-system`, `cattle-monitoring-system`, `cattle-system`, `cloudflare-tunnel`, `default`, `doppler-operator-system`, `external-secrets`, `ingress-nginx`, `nginx-ingress`, `kargo`, `karpenter`, `kong`, `kube-system`, `tidb`, `actions-runner-set`, `actions-runner-system` |
| **Cortex-only extras** | `ccdatasync`, `cert-manager`, `cicd`, `conga-sign`, `esign`, `plat-enablement`, `platform.referenceapp`, `servicepulse-rls04`, `srm` |

**Loki-only namespaces (no metrics)**: `billing`, `cci`, `cci-standalone`, `clm`, `docgen`, `invoice`, `testauthor`, `tidb`, `ingress-nginx`. For these, derive RED-style metrics from logs via Loki `count_over_time` instead.

### Data-quality flags (verified 2026-04)

- Dirty `service_namespace` values like `contractssf;cattle-system`, `default;default` (semicolon-joined) — collector relabel bug. Filter them out.
- `revenue_*` metrics are emitted from both `revenue` and `platform` namespaces. Pin both `service_namespace` + `service_name` when chasing a metric family.
- **TiDB is a shared cluster** per stack: `service_namespace="tidb"` logs ship under a **single** `deployment_environment` (the cluster's home env), not under each consumer env. Client-side latency lives under the consumer env via `platform_tidb_*` metrics; server-side logs (`tidb`/`tikv`/`pd`) only under the home env. Discover it with `group by (deployment_environment) ({service_namespace="tidb"})` in Loki before assuming.

## Per-product metric-prefix map <!-- tags: metric-prefixes, platform_, revenue_, kong_, frontend_ -->

| `service_namespace` | Custom-business prefixes | Notes |
|---|---|---|
| `platform` | `platform_auth_*`, `platform_incoming_request_*`, `platform_workflows_*`, `platform_search_*`, `platform_tidb_*`, `platform_objectdb_*`, `platform_messaging_*`, `platform_scheduler_*`, `datasync_api_*`, `migrationmanager_*` | Also emits `revenue_documents_*`, `revenue_query_*`. .NET runtime: `aspnetcore_routing_*`, `process_runtime_dotnet_*`. |
| `revenue` | `revenue_*` (quote, cart, documents, query) | TBD — extend on next investigation |
| `kong` | `kong_http_requests_total{code=~"..."}`, `kong_*` | Primary 5xx source for ingress (queries E/F). |
| _other_ | Discover with query H scoped by `service_namespace`, **never** wildcard `__name__`. | |

OTel-standard prefixes across most .NET/Java services: `http_client_duration_*`, `http_server_active_requests`, `http_request_duration_seconds_*`, `db_client_connections_*`, `otel_sdk_span_*`.

## Ingress detection (Kong vs nginx) <!-- tags: ingress-rollout, kong-vs-nginx, ingress-detect -->

Kong is being rolled out as the ingress proxy. Some envs are on Kong, others still on nginx, a few may be mid-cutover with both active. **Never assume Kong globally** — detect per `deployment_environment` before running ingress-5xx queries.

### One-shot detection probe

```promql
# Returns which ingress metric families have data in this env (last 5m).
# Kong present → __name__ "kong_http_requests_total" returns a count > 0
# Nginx present → __name__ "nginx_ingress_controller_requests" returns a count > 0
count by (__name__) (
  {__name__=~"kong_http_requests_total|nginx_ingress_controller_requests",
   deployment_environment="<env>"}
)
```

### Picking the right metric per ingress

| Ingress | Total counter | Status label | Breakdown label | Notes |
|---|---|---|---|---|
| Kong | `kong_http_requests_total` | `code` (`5..`, `4..`, `2..`) | `route` (parse upstream service from the route name) | Upstream `service_namespace`/`service_name` are NOT carried — must parse `route`. See "Verified metric facts" below. |
| nginx | `nginx_ingress_controller_requests` | `status` (`5..` etc.) | `ingress` and/or `service` / `exported_service` | Has direct upstream labels, no parse needed. Verify exact label names with `list_prometheus_label_names` per env — vary by exporter version. |

### Decision flow

1. Run the detection probe above.
2. **Kong-only** → use queries E/F (below).
3. **Nginx-only** → use the nginx variants of queries E/F:
   ```promql
   # E-nginx. Env-wide 5xx by upstream service
   topk(10, sum by (service)
     (rate(nginx_ingress_controller_requests{deployment_environment="<env>",status=~"5.."}[5m])))

   # F-nginx. 5xx ratio per upstream service
   sum by (service) (rate(nginx_ingress_controller_requests{deployment_environment="<env>",status=~"5.."}[5m]))
   / sum by (service) (rate(nginx_ingress_controller_requests{deployment_environment="<env>"}[5m]))
   ```
   (Confirm `service` is the right label for the env; some exporters use `exported_service` or `ingress` instead.)
4. **Both present (mid-cutover)** → query both, then dedupe by upstream service name when reporting to the user. Note in the response that the env is mid-cutover so numbers from both ingresses are summed.
5. **Neither present** → the env may not have an ingress layer, or metric names differ. Fall back to Loki error-log volume by `service_name` and tell the user no ingress metrics were found.

### Known env mapping (verify per query; rollout is in progress)

The rollout is in motion — always re-detect rather than trust a cached list. If you do need a hint as a starting point, run the probe and let the result drive the query, don't skip it.

## Canonical queries <!-- tags: canonical-promql, 5xx, error-rate, p95, latency, red-metrics, kong-route, discover-metric, __name__, group-by -->

```promql
# E. Env-wide 5xx scan via Kong ingress (best for "what's broken right now")
#    PRECONDITION: confirm Kong is the live ingress in this env via the detection
#    probe in § "Ingress detection (Kong vs nginx)" above. On nginx envs, use E-nginx.
#    `route` = "rls-app.<group>-api-ingress-kong.<service>._.congacloud.io.<port>"
topk(10, sum by (route)
  (rate(kong_http_requests_total{deployment_environment="<env>",code=~"5.."}[5m])))

# F. 5xx ratio per route (Kong)
sum by (route) (rate(kong_http_requests_total{deployment_environment="<env>",code=~"5.."}[5m]))
/ sum by (route) (rate(kong_http_requests_total{deployment_environment="<env>"}[5m]))

# G. p95 latency by service (generic OTel histogram — only some services emit)
histogram_quantile(0.95, sum by (le, service_name)
  (rate(http_request_duration_seconds_bucket{deployment_environment="<env>",service_name="<svc>"}[5m])))

# H. Discover metric families for a product (BEFORE writing a query)
group by (__name__) ({service_namespace="<ns>", deployment_environment="<env>"})

# H'. Narrowed to error/failure families
group by (__name__) ({__name__=~"(?i).*(error|fail|5xx).*", service_namespace="<ns>", deployment_environment="<env>"})
```

## Verified metric facts (rls03, 2026-04) <!-- tags: verified-facts, kong-code-label, http_request_duration -->

- **Kong has the `code` label** (`200`,`400`,`401`,`403`,`404`,`409`,`422`,`499`,`500`,`502`,`504`,…) → primary 5xx source.
- ⚠ **`kong_http_requests_total` does NOT carry the upstream service's `service_namespace`/`service_name`** — those labels are Kong's own (`cluster-observability`/`prometheus-proxy`). Upstream is encoded only in the `route` label. **Always parse `route`, never group by `service_namespace`** on Kong metrics.
- `http_requests_total` is sparse and has **no status_code label** — don't use for error queries.
- `http_server_request_duration_*` does **not** exist; the histogram is `http_request_duration_seconds_*`.
- For RLS infra metrics in `cortex`, the cluster label is **`k8s_cluster_name`** (OTel-style), not `cluster`.
- Custom families: `platform_*`, `revenue_*` quote/cart, `frontend_*`, `kong_*` — discover with query H, never wildcard.

## High-cardinality labels — DO NOT use as matchers or `by()` keys <!-- tags: high-cardinality, label-blocklist, series-fanout, churn -->

Verified on `cortex` (rls03, 2026-04). These labels blow up active-series fan-out and trip `max_samples` / 30s timeout. **Strip them with `without(...)` in every aggregation; never group by them; never use them as a primary matcher.**

| Label | Unique values | Why it's toxic |
|---|---|---|
| `k8s_pod_name` | ~17k | per-pod, churns on deploy |
| `mountpoint` | ~7.8k | filesystem paths, node-fanout |
| `pod` | ~7.7k | per-pod, churns on deploy |
| `name` | ~6k | container/object name, ambiguous |
| `ip` | ~5k | rotates on every pod restart |
| `tid` | ~4.7k | thread id — never aggregate on this |
| `__name__` | ~4.5k | only safe inside `group by (__name__)` discovery (query H), never as filter |
| `service_name` | ~4.2k | OK as **equality** filter, never as `=~".*"` regex |
| `job` | ~4k | OK as equality filter only |
| `pod_template_hash` | ~3.8k | changes every deploy → series churn #1 |
| `k8s_pod_start_time` | ~3.6k | changes every restart → series churn |
| `type` | ~3.1k | overloaded label, prefer `kind`/`resource` |
| `ObjectName` | ~2.6k | Windows perf-counter axis |

### Hard rules the agent MUST follow when writing PromQL

1. **Never** put any label from the table above on the right-hand side of `by(...)` unless the user explicitly asks for per-pod / per-instance breakdown.
2. Always include a `without (pod, k8s_pod_name, instance, ip, tid, pod_template_hash, k8s_pod_start_time, mountpoint, name, ObjectName)` clause on `sum`/`avg`/`max` aggregations over container/node metrics.
3. Always anchor at least one **equality** matcher on a low-card label: `deployment_environment=`, `service_namespace=`, `cluster=` / `k8s_cluster_name=`, or `namespace=`. **No bare `{__name__=~"..."}` queries.**
4. Reject regex matchers on `service_name`, `job`, `pod`, `k8s_pod_name`, `name`, `ip` unless anchored (`=~"prefix-.+"`) AND combined with an equality matcher on env/namespace.
5. For histograms: `by (le, <≤2 low-card labels>)` only. Never include `pod`/`instance` in the `by` of `histogram_quantile`.
6. Prefer `topk(N, …)` over returning the full series set for any exploratory query.

### Safe-label cheat sheet

| Use as **filter** (equality) | OK in `by()` | **Forbidden** in `by()` and as regex filter |
|---|---|---|
| `deployment_environment`, `service_namespace`, `k8s_cluster_name`, `cluster`, `namespace`, `environment`, `code`, `status_code`, `method` | `route`, `service_name`, `container`, `node`, `le` | `pod`, `k8s_pod_name`, `instance`, `ip`, `tid`, `pod_template_hash`, `k8s_pod_start_time`, `mountpoint`, `name`, `ObjectName`, `__name__` |

### Dynamic cardinality cache (preferred over hardcoded table) <!-- tags: cardinality-cache, dynamic-blocklist, ttl, mcp-cost -->

The table above is a **fallback**. When possible, derive the blocklist live, but **do not write it to memory** — keep it in the agent's working context for the duration of the conversation only.

**Refresh policy — probe at most ONCE per env per conversation:**
1. First time an env is referenced in this conversation → run the probe, hold the result in working context.
2. Reuse that in-context result for every subsequent query in the same conversation.
3. Force a re-probe **only** if the previous query failed with `max_samples`, `too_many_series`, or 30s timeout, or the user explicitly asks to refresh.
4. If probing is too expensive or the MCP errors → fall back to the hardcoded table above. **Do not persist anything to `/memories/`.**

**Probe procedure (run once per env per conversation):**
```
1. list_prometheus_label_names(datasource_uid="cortex",
                               matches=['{deployment_environment="<env>"}'],
                               start_rfc3339=now-15m, end_rfc3339=now)
2. For each label name → list_prometheus_label_values(label, same matcher)
3. Bucket by len(values):
     >= 1000  → strip_always
     100..999 → strip_unless_grouped
     <  100   → safe
4. Hold the result in conversation context. Do NOT write to disk or /memories/.
```

**In-context shape (ephemeral, conversation-scoped):**
```
env=<env>
strip_always:        [label: count, ...]   # >= 1000 unique values
strip_unless_grouped:[label: count, ...]   # 100..999
safe:                [label, ...]          # < 100
```

**Per-query agent flow:**
1. If env's blocklist is already in working context → use it (zero MCP calls).
2. Else probe once, then cache in working context.
3. Inject `without(<all keys of strip_always>)` into every aggregation.
4. Reject any user-issued `by(...)` containing keys from `strip_always`. For `strip_unless_grouped`, allow only when the user explicitly asked for that breakdown.
5. Require at least one equality matcher from `safe`.
6. On `max_samples`/timeout → re-probe once and retry.

### Rewrite examples

```promql
# ❌ BAD — returns 17k+ series, will time out
sum by (pod, ip, container) (rate(container_cpu_usage_seconds_total[5m]))

# ✅ GOOD
sum without (pod, k8s_pod_name, instance, ip, pod_template_hash, k8s_pod_start_time)
  (rate(container_cpu_usage_seconds_total{namespace="prod",k8s_cluster_name="rls03"}[5m]))

# ❌ BAD — unanchored regex on high-card label
{service_name=~".*api.*"}

# ✅ GOOD
{service_name=~"cci-.+", deployment_environment="rls03", service_namespace="cci"}
```

## Cortex/Mimir limits <!-- tags: cortex-limits, max_samples, step-floor -->

- **`max_samples`** ≈ 50M per query. High-cardinality `group by (__name__)` over a wide window trips this.
- **Per-query timeout** ≈ 30s.
- For long ranges, enforce `step` minimums: ≤7d → `step ≥ 1m`, ≤30d → `step ≥ 5m`.
- Cap `[$range]` at `window/4` for `rate`/`increase`, or use recording rules.

For wide-window requests, see [`wide-window-guard`](./wide-window-guard.md).
