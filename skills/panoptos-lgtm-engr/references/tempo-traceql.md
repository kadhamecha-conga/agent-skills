# Tempo Handoff & Cross-Signal Correlation — Panoptos

Companion skill to [`lgtm`](../SKILL.md). Loaded only when the work involves traces, deeplinks, or correlating across logs/metrics/traces.

## MCP tools for traces <!-- tags: tempo-tool, analyse-trace, trace-by-id, tempo-mcp, grafana-tempo-get-trace -->

**Default route**: Use `grafana-engr:tempo_get-trace(datasourceUid, trace_id)` for trace-by-ID lookups. Returns raw spans; suitable for small/medium traces where the user wants to inspect span data directly.

**Escalation route** (large traces, explicit request): Use `tempo-server:analyse_trace(trace_id, query)` only when:
- The user explicitly asks for it (e.g. mentions `@tempo-server`, "analyse", "deep analysis")
- The trace is large enough that `grafana-engr:tempo_get-trace` would fail, truncate, or return an unwieldy blob
- The grafana tool has already been tried and hit a size/timeout limit

`tempo-server:analyse_trace` accepts a natural-language `query` to focus output (e.g. `"slowest spans"`, `"errors"`, `"DB spans"`). Results cached 5 min; pass `force_refresh=true` to bypass.

**Grafana user-facing URL**: `https://engr-telemetry.conga-panoptos.com` (verified 2026-04). Use this URL only for deeplinks shared with users. Never share the internal cluster URL (`grafana.panoptos.svc.cluster.local`).

**Tempo coverage is sparse**: only `platform` and `cluster-observability` currently emit traces. Don't expect a `traceid` pivot for most products yet.

## Correlation key (logs ↔ metrics ↔ traces) <!-- tags: correlation-key, cross-signal, correlate -->

Pin **all three** when crossing signals — pinning only `service_name` is ambiguous.

| Concept | Loki / Cortex label | Tempo attribute (resource scope) |
|---|---|---|
| Environment | `deployment_environment` | `resource.deployment.environment` |
| Service | `service_name` | `resource.service.name` |
| Product / domain | `service_namespace` | `resource.service.namespace` |
| Cluster | `k8s_cluster_name` | `resource.k8s.cluster.name` |
| Tenant | `OrganizationId` (Loki label) | `OrganizationId` (**span scope**, not resource) |

**Naming rule**: OTel collectors flatten dots → underscores when writing to Loki/Mimir, but keep dots in Tempo. Translate when crossing into TraceQL: `service_namespace` → `resource.service.namespace`, etc.

Other rules:
- **`traceid` is the fine-grained join** within a request (Loki JSON body `traceid` ↔ Tempo `trace:id`). The three labels above are the coarse-grained join across signals.
- **Don't correlate by `k8s_cluster_name` alone** — RLS clusters are named `usw2-app-eks-dev-csg1-site01-<env>`; same env can span multiple clusters in HA. Use `deployment_environment` as canonical scope.

## Correlation recipe <!-- tags: correlation-recipe, trace-handoff-steps -->

1. Start from a metric anomaly (e.g., 5xx spike on a Kong route). Note `deployment_environment`, infer `service_name` from the route's `<service>` segment.
2. Pull logs with the **same** `deployment_environment` + `service_name` + `SeverityText=~"ERROR|Error"` over the same window.
3. Extract a `traceid` from a representative error line (`| json`).
4. **Fetch the trace**: default to `grafana-engr:tempo_get-trace(datasourceUid="tempo", trace_id="<id>")` for raw spans. If the trace is too large or the user asks for a focused analysis, escalate to `tempo-server:analyse_trace(trace_id="<id>", query="slowest spans, errors, DB operations")` — this returns span counts, durations, call chain, HTTP status codes, and exception details in one call.
   - If no trace ID is available, build a Tempo Explore deeplink (below) with a TraceQL query scoped to the service + duration threshold.
5. Verify root cause by checking the *upstream* service's RED metrics (filter by its `deployment_environment` + `service_name`) — catches HTTP-status masking.

## `generate_deeplink` — shapes <!-- tags: deeplink-shapes, deeplink, tempo-url, grafana-url, explore-link, traceql, duration -->

### Tempo trace by ID

```json
{
  "resourceType": "explore",
  "datasourceUid": "tempo",
  "queryType": "traceId",
  "queryParams": { "query": "<traceid>" },
  "timeRange": { "from": "<startRfc3339>", "to": "<endRfc3339>" }
}
```

Output:

```json
{ "url": "https://grafana.panoptos.example.com/explore?orgId=1&left=%7B...%7D" }
```

Render in the reply as: `[Open trace <short-id> in Tempo](<url>)`. **Don't paste the URL-encoded blob** — it's noise.

**Grafana user-facing URL**: `https://engr-telemetry.conga-panoptos.com`. Always share links with this scheme/host only. If `generate_deeplink` returns the internal cluster host `http://grafana.panoptos.svc.cluster.local`, replace only the scheme/host before replying; do not expose the internal URL.

### TraceQL search (no known trace ID — find slow spans)

```json
{ "resourceType": "explore", "datasourceUid": "tempo",
  "queryParams": { "query": "{resource.service.name=\"<svc>\" && resource.deployment.environment=\"<env>\" && duration > 5s}", "queryType": "traceql" },
  "timeRange": { "from": "<from>", "to": "<to>" } }
```

### Loki Explore (re-run a LogQL the user can iterate on)

```json
{ "resourceType": "explore", "datasourceUid": "loki", "queryType": "logs",
  "queryParams": { "expr": "<logql>" },
  "timeRange": { "from": "<from>", "to": "<to>" } }
```

### Cortex/Prometheus Explore

```json
{ "resourceType": "explore", "datasourceUid": "cortex",
  "queryType": "metrics", "queryParams": { "expr": "<promql>" },
  "timeRange": { "from": "<from>", "to": "<to>" } }
```

(Use `prometheus` UID for Panoptos platform / non-RLS infra.)

### Dashboard panel

```json
{ "resourceType": "dashboard", "dashboardUid": "<uid>", "panelId": <n>,
  "timeRange": { "from": "<from>", "to": "<to>" } }
```

## When to use which trace tool <!-- tags: tool-vs-deeplink, when-to-use -->

| Situation | Action |
|---|---|
| Have a `traceid`, normal-sized trace | **`grafana-engr:tempo_get-trace`** — default, raw spans |
| User explicitly asks for analysis / mentions `@tempo-server` | **`tempo-server:analyse_trace`** |
| Trace is huge or grafana-engr tool fails/truncates | Escalate to **`tempo-server:analyse_trace`** |
| No trace ID, want to find slow traces | Build TraceQL deeplink with `duration > Xs` filter (above) |
| Returning a query the user will tweak | Include a deeplink alongside the analysis |
| Wide-window trace search | Deeplink only — MCP fetch would be too large |
