# Loki Query Debug — Panoptos

Companion skill to [`lgtm`](../SKILL.md). Loaded only when the work is specifically Loki/LogQL.

## Label schema (only these 9 exist) <!-- tags: label-schema, logql, loki-schema, severity, deployment_environment -->

`OrganizationId`, `OrganizationFriendlyId`, `SeverityText`, `deployment_environment`, `k8s_cluster_name`, `platform_log_source`, `service_name`, `service_namespace`, `__stream_shard__`

Everything else (trace id, http status, error message, …) is **inside the JSON body** — extract with `| json`.

### `SeverityText` quirks

Commonly uppercase. Use `=~"ERROR|Error|FATAL"` to catch title-case strays.

### Getting the current time (required first step)

LogQL queries need absolute RFC3339 UTC timestamps for `startRfc3339` and `endRfc3339`. **You — the AI agent — must obtain the current time yourself before every query. Do not ask the user.**

**Run this immediately, without asking for permission** (it's read-only and safe):

```bash
# macOS / BSD date
date -u '+%Y-%m-%dT%H:%M:%SZ' && date -u -v-15M '+%Y-%m-%dT%H:%M:%SZ'

# Linux / GNU date (most containers, CI, Codespaces)
date -u '+%Y-%m-%dT%H:%M:%SZ' && date -u -d '15 minutes ago' '+%Y-%m-%dT%H:%M:%SZ'
```

First line → `endRfc3339`. Second line → `startRfc3339`.

**Fallback order if shell is unavailable:**
1. Use a current date/time provided in your system prompt (Claude.ai injects this; Copilot/Cursor/Continue do not).
2. Use the most recent `timestamp` field from any prior MCP response in this session.
3. Only as a last resort — and only if steps 1 and 2 both fail — ask the user.

**Do not** treat "I don't have a clock" as a reason to prompt. Running `date` is part of the skill, not a side request.

### `deployment_environment` values (verified 2026-04)

- **Long-lived**: `prod`, `staging`, `dev`, `dev1`, `dev2`, `dev3`, `cicd`, `ephemeral`
- **RLS**: `rls03`, `rls04`, `rls05`, `rls07`, `rlsaz06`
- **Per-team / preview**: `cci-beta-1030`, `cci-legacy-stage`, `cci-yama-beta-1030`, `yama-dev-1100`, `contracts-dev`, `contractssf-dev`, `max-pu-xajs-engg-401`, `max-pu-xajs-kafka-401`

⚠ `prod` and `staging` exist in Loki but **not in `cortex`** — prod/staging metrics may live in a different Mimir or be filtered out of remote-write.

## JSON body fields (after `| json`) <!-- tags: json-body, traceid, spanid, extract -->

- `traceid`, `spanid` — note: **`traceid`, not `trace_id`** (one word, lowercase)
- `severity` (lowercase mirror of `SeverityText`)
- `body` — the message
- `resources_service_name`, `resources_deployment_environment`, `resources_k8s_cluster_name`, `resources_host_id`, `resources_host_image_version`
- `attributes_TimeStamp`, `attributes_ImageVersion`

## Canonical queries <!-- tags: canonical-logql, error-log, count_over_time, topk, noisy-service -->

For LogQL execution, default missing time ranges automatically: use the last 15 minutes ending at "now". **Never ask the user for the current date/time.** Obtain it yourself:

1. If your system prompt provides a current date/time (e.g. Claude.ai injects "The current date is …"), use that as `now`.
2. Otherwise, shell out — this is the canonical one-liner (macOS/BSD `date`):
```bash
   date -u '+%Y-%m-%dT%H:%M:%SZ' && date -u -v-15M '+%Y-%m-%dT%H:%M:%SZ'
```
   GNU `date` (Linux containers, most CI):
```bash
   date -u '+%Y-%m-%dT%H:%M:%SZ' && date -u -d '15 minutes ago' '+%Y-%m-%dT%H:%M:%SZ'
```
   First line = `endRfc3339`, second = `startRfc3339`.
   
3. If neither a clock nor a shell is available, fall back to the session's most recent known timestamp (e.g. from a previous MCP response's `timestamp` field) — never prompt the user.

```logql
# A. Errors for an env (broad scan, last 5–15m, limit 100)
{deployment_environment="<env>"} | SeverityText=~"ERROR|Error|FATAL"

# B. Errors for one service
{deployment_environment="<env>", service_name="<svc>"} | SeverityText=~"ERROR|Error"

# C. Full picture for one trace (run on ±5m around the error)
{deployment_environment="<env>"} | json | traceid="<id>"
  | line_format "[{{.resources_service_name}}] {{.spanid}} {{.severity}} {{.body}}"

# D. Top noisy services
topk(10, sum by (service_name)
  (count_over_time({deployment_environment="<env>"} | SeverityText=~"ERROR|Error" [15m])))
```

## Loki-specific limits <!-- tags: loki-limits, truncation, 5mb, lookback, max_query_length -->

- **Per-query timeout** ≈ 30s. Wide selectors over `[1h]+` will time out.
- **Result size cap** ≈ 5 MB per response. Symptom of truncation: line count == `limit` exactly + newest entries missing. Drop `limit` to 100–200, narrow window.
- **`max_query_length`** = 30d, **`max_query_lookback`** = 30d. Older = empty result.
- Never query without a label selector. Always use label filters, never line filters as primary scope.

For wide-window requests (>24h raw lines), see [`wide-window-guard`](./wide-window-guard.md).

## Common gotchas <!-- tags: logql-gotcha, line_format, pipe-order, empty-result -->

- `traceid="<id>"` works only **after** `| json`. Without `| json` it's a label match against a label that doesn't exist → empty result.
- `line_format` requires `{{.field}}` Go template syntax with the leading dot.
- Pipe order matters: `{...} | json | traceid="<id>"` not `{...} | traceid="<id>" | json`.
- For large tool results, post-process with `jq -r '.[].line'` instead of re-reading raw JSON.
- **Metric LogQL via `query_loki_logs` drops labels**: `sum by (...) (count_over_time(...))` returns rows with `line=<value>` and `labels=null` — the MCP wrapper flattens every sample to the log-line schema. For top-k by label, pull raw lines and tally: `jq -r '.[].labels.<key>' <result> | sort | uniq -c | sort -rn`. (E.g. `query_D` from canonical queries will misbehave through this MCP — use the jq fallback.)
- **TiDB logs (`service_namespace="tidb"`) live under one shared env per stack**, not per consumer env. Pinning the consumer's `deployment_environment` returns empty even when its apps use TiDB. Discover the home env first: `sum by (deployment_environment) (count_over_time({service_namespace="tidb"}[5m]))`, then re-query without an env filter or pin the discovered env. Sub-services: `service_name` ∈ {`tidb`, `tikv`, `pd`}; format is non-OTel — use `|~ "\\[(WARN|ERROR|FATAL)\\]"` instead of `SeverityText=~`.
