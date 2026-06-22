# Wide-Window Protection — Panoptos

Companion skill to [`lgtm`](../SKILL.md). Loaded when a query risks blowing past MCP/Loki/Cortex limits.

## The problem <!-- tags: wide-window-problem, silent-fail -->

Long windows (>1h on Loki raw lines, >24h on Cortex with high cardinality) **silently fail** in three modes:
- **Timeout** (`context deadline exceeded`) at ~30s.
- **Truncation** at ~5 MB result cap — Loki returns a suspiciously round line count = `limit` with newest entries missing.
- **`max_samples` exceeded** on Cortex — a 30d range × `step=15s` = 172,800 samples *per series*; multiplied by series count, you trip the 50M ceiling.

All three leave you with output that *looks* like a clean answer.

## Hard caps the assistant enforces <!-- tags: hard-caps, long-window, wide-window, 30-days, all-errors -->

Refuse / reshape, never bypass:

| Tool | Soft cap (auto-run) | Hard cap (require explicit confirm + narrowing) | Absolute refuse |
|---|---|---|---|
| `query_loki_logs` (raw lines) | ≤ 1h | ≤ 24h **with** `service_name` or `service_namespace` pinned | > 24h raw, or any window without a label selector |
| `query_loki_logs` (metric query — `count_over_time`/`rate`) | ≤ 6h | ≤ 7d with `service_namespace` + `step ≥ 1m` | > 7d, or `[$range]` ≥ window |
| `query_prometheus` (instant) | any | — | — |
| `query_prometheus` (range) | ≤ 24h, `step` auto | ≤ 7d with `step ≥ 1m`; ≤ 30d with `step ≥ 5m` | > 30d, or `step` < 30s on any range > 6h |

## Six rules <!-- tags: six-rules, refuse, aggregate-first, drill-down -->

1. **>24h raw Loki lines = refuse the raw fetch.** Offer an aggregated alternative:
   > *"I can't pull 30 days of error lines (truncation + 30s timeout). I can run a daily error count and show you the top 5 spikes; we then zoom into a 1h window around each spike for actual lines."*
   Run query D-style aggregation with `step ≥ 5m`, then drill down.
2. **Long-window Cortex range queries**: enforce `step ≥ window / 1000` as a floor (≈ 30s for 8h, 5m for 30d).
3. **No `[$range]` longer than the query window.** Cap `[$range]` at `window / 4` for `rate`/`increase`, or use recording rules.
4. **Always pin a label** — even at 30d, `{service_namespace="<ns>", deployment_environment="<env>"}` keeps cardinality finite. Reject any wide query missing both.
5. **Chunk on demand, don't auto-paginate.** Split a 30d window into 1h slices around the spikes identified by step 1 — never run a `for d in $(seq 1 30); do query; done` blind loop.
6. **Prefer ready-made dashboards.** For "last 30 days of 5xx", a Grafana dashboard URL via `generate_deeplink` (see [`trace-debug`](./tempo-traceql.md)) is cheaper, faster, and renders client-side. Hand off, don't compute.

## Symptom → action quick reference <!-- tags: symptom-table, timeout, truncation, max_samples-error -->

| Symptom | Likely cause | Fix |
|---|---|---|
| Empty result, no error, large window | Loki `max_query_lookback` exceeded (>30d) | Cap window at 30d |
| `context deadline exceeded` | Query > 30s | Narrow window, add label, raise `step` |
| Suspiciously round line count = `limit` | 5 MB cap or `limit` cap hit | Lower `limit`, narrow window |
| `too many samples` / 422 from Cortex | `max_samples` exceeded | Raise `step`, scope by `service_namespace` |
| HTTP 429 | Rate limit | Consolidate (one `topk` instead of N queries), back off |

## MCP infrastructure limits (for context) <!-- tags: mcp-limits, concurrency, rate-limit -->

- **Per-query timeout** ≈ 30s (Loki + Cortex).
- **Result size** ≈ 5 MB per tool response.
- **Loki `max_query_length`** = 30d, **`max_query_lookback`** = 30d.
- **Cortex `max_samples`** ≈ 50M.
- **Concurrency**: MCP serializes calls per session — parallel tool calls queue, no real parallelism. Don't fan out 10 queries hoping for speedup.

## Refusal script (paste-ready) <!-- tags: refusal-script, paste-ready-refusal -->

> The window you've asked for would either time out or silently truncate (`{tool}` caps at `{cap}`). Two faster paths:
> 1. **Aggregate first, drill in second** — I run `topk by (service_name)` with `step={recommended}` over your full window to find the top spikes, then pull raw lines for a 1h window around each. (Offered above.)
> 2. **Hand off to a dashboard** — I generate a Grafana deeplink with your filters pre-applied so the UI does the heavy work.
>
> Which would you like?
