# Root-Cause Search Playbook — Panoptos

Companion skill to [`lgtm`](../SKILL.md). Loaded when the request is an **investigation / RCA / "why did this break?"** — not a one-off query. It encodes *how to search for a root cause* and *what evidence to gather in a single pass* so the RCA reasoner gets a complete package the first time.

The other reference files tell you **how to query** each signal. This file tells you **what to go looking for**, in what order, and how to package it.

## The single-pass principle <!-- tags: single-pass, evidence-first, no-loop -->

Gather a **complete evidence package in one investigation pass** — do NOT hand back a thin symptom and wait to be asked for more. A weak RCA almost always traces to a missing input, not to weak reasoning. Before you conclude (or hand off to the RCA reasoner), make sure all **five evidence pillars** below are answered or explicitly marked `unknown — not available`:

1. **Symptom** — what is user-visible, where, how bad (always vs a baseline).
2. **Onset** — the timestamp of the *first* anomaly (this is the highest-signal field; see § Onset).
3. **What changed** — any deploy / image bump / restart / scale / config event near onset (see § What changed).
4. **Blast radius** — one service, whole namespace, or cluster-wide (see § Blast radius).
5. **Deepest signal** — the *earliest / lowest* failing component in the call chain, not just the top-level symptom (see § Cascade).

A pillar marked `unknown` is itself a finding — state it; never silently drop it.

## Search order (do NOT start from hypotheses) <!-- tags: search-order, evidence-before-hypothesis -->

Resist the urge to name a cause first and then go confirm it — that bakes in confirmation bias. Search **breadth-first over the five pillars**, *then* let the assembled evidence suggest the cause.

1. **Scope & symptom** — resolve `deployment_environment` + `service_name`/`service_namespace` (see SKILL.md § Service Name Resolution). State the symptom with its baseline.
2. **Find onset** — widen the window and locate the first deviation. Re-scope everything else to a tight window *around onset* (onset−15m .. onset+15m), not "last 1h".
3. **Correlate "what changed" against onset** — this single correlation resolves the majority of incidents (bad deploy, config flip, cert/token expiry, scale event).
4. **Measure blast radius** — disambiguates "this service broke" from "its dependency broke and it's a victim".
5. **Trace to the deepest signal** — follow the failure *down* the call chain to the component closest to the cause. The loudest service is usually the victim, not the culprit.

## Onset — find the first anomaly <!-- tags: onset, first-anomaly, inflection, when-did-it-start -->

The first deviation is closest to the root cause. Find a real timestamp; never eyeball "about an hour ago".

**Logs (works for every namespace, incl. Loki-only):** run a *total* error count bucketed over a wider window, find the first bucket above baseline. No `by()` clause → safe through the MCP metric-LogQL wrapper (see loki-logql § gotchas).

```logql
# R1. Onset via error volume — bucket over the suspected window, find the first spike.
#     Widen window to ~6h; step 5m. Baseline = the flat region before the jump.
sum(count_over_time(
  {deployment_environment="<env>", service_name="<svc>"} | SeverityText=~"ERROR|Error|FATAL" [5m]))
```

**Metrics (Kong/nginx-fronted services):** the inflection in the 5xx rate. (Detect Kong vs nginx first — mimir-promql § Ingress detection.)

```promql
# R2. Onset via 5xx rate — range-query this; the first non-zero/rising point is onset.
sum(rate(kong_http_requests_total{deployment_environment="<env>", code=~"5.."}[5m]))
```

Pin onset as an **absolute IST timestamp**. Then re-run the symptom/changed/radius queries scoped to onset±15m — a tight window around onset is far more decisive than a broad "last hour".

## What changed — the highest-yield correlation <!-- tags: what-changed, deploy-detection, image-version, restart, rollout, config-drift -->

Most production incidents are *triggered* by a change. Look for one within ±30m of onset. **Verify a signal exists before trusting it** (`group by (__name__)`, query H) — don't assume a metric name.

| Change type | Where to look | Signal |
|---|---|---|
| **Deploy / image bump** | Loki `\| json` | `attributes_ImageVersion` or `resources_host_image_version` — a **new value first appearing near onset** = a deploy. This is the most reliable deploy signal we have. |
| **Pod restart / crash loop** | Cortex (discover first) | New/changed `pod_template_hash`, churning `k8s_pod_start_time` (both high-card — observe, don't `by()` them). Discover a restart counter via query H narrowed to `restart\|start`. |
| **Scale event** | Cortex | Replica count change near onset; sudden per-pod load jump (fewer pods serving same traffic). |
| **Config / flag flip** | Loki | Config-reload / "reloaded" / "applied new config" log lines around onset. |
| **Cert / token expiry** | Loki | 401/403 spikes, "certificate expired", "token expired", TLS handshake errors — often *periodic* (renewal cadence). |

```logql
# R3. Did a new image roll out near onset? Distinct image versions seen, oldest→newest.
{deployment_environment="<env>", service_name="<svc>"} | json
  | line_format "{{.attributes_ImageVersion}}{{.resources_host_image_version}}"
```

```promql
# R4. Discover restart/start metric families for this product (don't guess the name).
group by (__name__) ({__name__=~"(?i).*(restart|start|generation|replicas).*",
  service_namespace="<ns>", deployment_environment="<env>"})
```

**If a change lines up with onset → that is your lead hypothesis.** Report the correlation explicitly: *"<symptom> began at <onset IST>; <change> occurred at <change-time IST> — <N>m before."*

## Blast radius — culprit vs victim <!-- tags: blast-radius, scope, victim-vs-culprit -->

How wide the failure spreads tells you where the cause lives.

```logql
# R5. How many services are erroring in this env? (count distinct → radius)
#     Pull raw lines + tally by service_name (metric LogQL drops labels via MCP — see loki gotcha).
topk(25, sum by (service_name)
  (count_over_time({deployment_environment="<env>"} | SeverityText=~"ERROR|Error|FATAL" [15m])))
```

- **One service only** → cause is likely *in* that service (code, config, its own resource limits).
- **Several services in one namespace** → shared dependency (a DB, a cache, an auth service) or a namespace-wide deploy/config change.
- **Cross-namespace / cluster-wide** → infra (node pressure, ingress, networking, cert rotation, a platform-shared service like auth or `tidb`).

The service with the *loudest* symptom is frequently a **victim** of a quieter upstream/downstream failure — let radius + cascade, not volume, pick the culprit.

## Cascade — trace down to the deepest failing component <!-- tags: cascade, dependency-chain, deepest-span, leaf-error -->

Failures propagate **upward** (a slow DB → service times out → API returns 5xx → user sees an error). The user-visible symptom is the *top* of the chain; the root cause is near the *bottom*. Always push past the first error you find.

- **Traces (only `platform` and `cluster-observability` emit them — tempo-traceql):** pivot from a failing request's `traceid` (extract from error logs, loki-logql query C) into Tempo, then find the **deepest / earliest-failing span** — that span's service is the cascade origin, not the root span.
- **No traces (most namespaces):** reconstruct the chain from logs. For each erroring service, check whether its errors are *self-generated* (its own exception/OOM) or *propagated* (timeouts, connection-refused, 5xx-from-upstream). Connection/timeout errors point *outward* — follow them to the next hop.
- **Common cascade shape:** dependency D degrades → callers retry → D overloads → retry storm widens the blast radius. The *first* service to fail (per § Onset, run per-service) is the origin; everything that started failing later is downstream.

## Pattern → signature → confirming evidence <!-- tags: rca-patterns, signatures, confirm-query -->

Recognize these by signature, then gather the confirming evidence in the same pass. (Extends the RCA reasoner's pattern list with the actual signal to pull.)

| Pattern | Signature | Confirm with |
|---|---|---|
| **Bad deploy / release** | 5xx or error spike starting within minutes of a new image version | R2/R3 onset vs image-version change |
| **OOMKilled / memory leak** | Periodic restarts; memory climbs then pod dies; "OOMKilled" in events | R4 restart metric + container memory vs limit; loki "OOM" lines |
| **Resource saturation** | Gradual latency climb, no deploy; CPU/connection-pool near 100% | p95 trend (mimir-promql query G) + CPU/pool gauges; flat onset (no sharp edge) |
| **Dependency failure** | Several services in a namespace fail together; connection-refused/timeout to one host | R5 radius + cascade to the common downstream host |
| **Cert / token / cron expiry** | Sharp onset, often on a round time; 401/403/TLS errors; *recurs* on a cadence | R3-style search for "expired"/TLS; check if onset repeats periodically |
| **Traffic spike** | Request rate jumps *before* errors; errors are saturation downstream of load | request-rate range query leads the error-rate range query |
| **Restart / probe loop** | Repeated start/stop; liveness/readiness-probe-fail log lines; dependency-not-ready | R4 + loki "probe failed"/"not ready" lines |

A genuine cause should explain **all three**: the onset time, the blast radius, *and* the symptom shape. If a hypothesis only explains one, keep it as runner-up, not the verdict.

## Evidence package contract (hand-off to the RCA reasoner) <!-- tags: evidence-package, handoff, rca-input, conclusions-not-raw -->

When you (or the orchestrator) compile evidence for the RCA reasoner, deliver **conclusions, not raw dumps** — and cover all five pillars. Use this shape:

```
SYMPTOM:     <what's user-visible> — <metric/count> vs baseline <normal value>. Outage or degradation?
ONSET:       <first-anomaly timestamp IST> (method: R1/R2). Ongoing | resolved at <t> | intermittent.
WHAT CHANGED:<deploy/restart/scale/config/cert event within ±30m of onset, with its timestamp IST>
             — or "none found in <window>".
BLAST RADIUS:<one svc | N svc in <ns> | cross-ns> — list the affected services (≤25).
DEEPEST SIGNAL:<the earliest/lowest failing component> — self-generated vs propagated error, with
             one representative error line (≤1, trimmed).
CORRELATION: <how the signals line up>, keyed on deployment_environment + service_name + service_namespace.
GAPS:        <any pillar marked unknown and why — e.g. "no traces: namespace doesn't emit to Tempo">.
```

Rules:
- Every number paired with its baseline (`p99 ~35s vs normal ~25ms`), or it's meaningless.
- At most ~10 trimmed log excerpts and ≤25 service candidates — never raw label arrays or raw multi-line dumps (see SKILL.md cost guardrails + each reference's reporting rules).
- All timestamps in **IST** (UTC + 5h30m, append " IST").
- Never fabricate a baseline, an onset, or a change event. `unknown` is a valid, useful answer.

## Wide windows <!-- tags: wide-window, onset-search-cost -->

Onset search (§ R1/R2) deliberately widens the window — keep it cheap. Bucket with `step ≥ 5m`, pin both `deployment_environment` and a service/namespace label, and never raw-fetch lines over the wide window (aggregate first, then drill into onset±15m). See [`wide-window-guard`](./wide-window-guard.md) before any window > 6h.
