# Namespace & Service Catalog (detailed)

Load this only when you need deep per-namespace detail (sample service names, product areas, counts) and live MCP discovery isn't enough. For routing decisions, the compact index + gotchas in `SKILL.md` is usually sufficient. **Prefer live discovery** (`list_prometheus_label_values` on `service_name`/`service_namespace`) over trusting these sample lists — they are illustrative, not exhaustive.

## Business Products — Have Cortex Metrics + Loki Logs

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

## Business Products — Loki-Only (No Cortex Metrics)

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

## Infra Namespaces (Not Product — Don't Confuse)

| Namespace | What it is |
|---|---|
| `cluster-observability` | OTel collectors, prometheus-proxy. In both `loki` and `loki-infra` |
| `sign` | ⚠️ Cluster-level infra for Sign clusters (kube-state-metrics, node-exporter, nginx, etc.) — NOT the Sign product. The Sign product is `conga-sign` or `congasign` |
| `contractssf` | Cluster-level K8s metrics (apiserver, kubelet, kube-state-metrics) for ContractsSF clusters — NOT product application metrics |
| `argocd`, `argo-workflows` | GitOps / CI-CD infra |
| `kube-system`, `cert-manager`, `karpenter`, `external-secrets` | Kubernetes system components |
| `rls-operator`, `rls-app` | RLS cluster operators |
| `kong`, `nginx-ingress`, `ingress-nginx` | Ingress layer. Kong metrics exist in `cortex` but under `kong_http_requests_total` without `service_namespace` — use `route` label instead |

## Loki-Infra Namespace (loki-infra datasource only)

| Namespace | Services |
|---|---|
| `panoptos` | LGTM stack components: `cortex-distributor`, `cortex-ingester`, `cortex-compactor`, `cortex-querier`, `cortex-query-frontend`, `cortex-store-gateway`, `cortex-ruler`, `cortex-alertmanager`, `cortex-nginx`, `loki-distributor`, `loki-ingester`, `loki-querier`, `loki-compactor`, `loki-gateway`, `loki-query-frontend`, `loki-ruler`, `loki-index-gateway`, `loki-infra-*` (self-monitoring variants), `tempo-distributor`, `tempo-ingester`, `tempo-querier`, `tempo-compactor`, `tempo-gateway`, `tempo-query-frontend`, `tempo-metrics-generator`, `grafana`, `grafana-mcp-ops` |
| `cluster-observability` | `otelcol-logs`, `otelcol-metrics`, `otelcol-traces`, `internal-agent-ot-collector-contrib`, `kube-prometheus-stack-*`, `kong-proxy`, `kong-grpc-proxy` |
