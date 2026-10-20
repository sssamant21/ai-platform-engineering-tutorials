# Lab 069 — OpenTelemetry for AI Infrastructure: Staging Validation Gate

**Chapter:** [069 — OpenTelemetry for AI Infrastructure](../part-09-gpu-observability/069-opentelemetry-ai-infrastructure.md)

**Status:** NOT RUN. All commands and outputs below are templates, not evidence of execution.

## 1. Preconditions and change controls

- Kubernetes **disposable staging** cluster and dedicated \`ai-observability-lab\` namespace.
- Owner-approved NetworkPolicies, OTLP ingress and backend egress; approved telemetry retention and RBAC.
- Reviewed, immutable Collector image digest; Collector binary matching the deployed image for \`validate\`.
- Read-only discovery first. Do not instrument sensitive production traffic or run load tests on shared systems.
- Do not publish bearer tokens, model inputs, prompts, GPU UUIDs, private IPs or full customer metadata in evidence.

## 2. Fourteen-exercise evidence checklist

| ID | Activity | Evidence | Status |
|---|---|---|---|
| 01 | Inventory existing SDKs/Collectors | versions, namespaces, endpoints (redacted) | Pending |
| 02 | Draw signal pipelines | architecture, owners, failure points | Pending |
| 03 | Review attributes | semantic conventions, forbidden fields | Pending |
| 04 | Isolate namespace | namespace and NetworkPolicy manifests | Pending |
| 05 | Validate config | Collector validation + server dry-run output | Pending |
| 06 | Deploy gateway | rollout, pod health, resource usage | Pending |
| 07 | Synthetic trace | trace ID and confirmed backend receipt | Pending |
| 08 | Synthetic metric | unit/type and backend query | Pending |
| 09 | DCGM path check | exporter/Prometheus metric and target health | Pending |
| 10 | Correlate signals | redacted trace/pod/node/GPU timeline | Pending |
| 11 | Failure drill | approved outage, queue/loss/recovery timestamps | Pending |
| 12 | Privacy/cardinality | test outcomes and mitigation | Pending |
| 13 | Capacity budget | peak rate, bytes, retention, CPU/memory | Pending |
| 14 | Acceptance | runbooks, reviewer, scoring | Pending |

## 3. Read-only discovery commands

\`\`\`bash
kubectl get namespaces
kubectl get pods -A -o wide | grep -Ei 'otel|collector|dcgm|prometheus'
kubectl get services -A | grep -Ei 'otel|prometheus'
kubectl get networkpolicy -A
kubectl -n ai-observability-lab get deploy,svc,cm,events
kubectl -n ai-observability-lab describe deploy otel-gateway
kubectl -n ai-observability-lab logs deploy/otel-gateway --tail=100
\`\`\`

Run only commands applicable to the discovered environment; absence of the staging namespace before setup is expected.

## 4. Static configuration and runtime gates

Extract the ConfigMap's \`config.yaml\` as \`config.yaml\` and the Kubernetes resources as \`collector.yaml\`; replace the chapter's non-runnable image placeholder with an approved, pinned digest.

\`\`\`bash
otelcol-contrib validate --config=config.yaml
kubectl -n ai-observability-lab apply --dry-run=server -f collector.yaml
# Only after approved change and network isolation:
kubectl -n ai-observability-lab apply -f collector.yaml
kubectl -n ai-observability-lab rollout status deployment/otel-gateway --timeout=180s
\`\`\`

**Do not mistake success of the Kubernetes dry-run for Collector configuration validation.** Capture both results.

## 5. Synthetic test matrix

| Test | Input | Expected observation | Stop condition |
|---|---|---|---|
| Trace transport | harmless synthetic trace over configured OTLP | received and exported; trace queryable | credentials or payload leak |
| Metric transport | bounded gauge with stable labels | correct type, unit and timestamp | cardinality blow-up |
| Missing backend | staging-only blocked backend | observable retries/queue or explicit drop | impact outside staging |
| Recovery | restore staging backend | queue drains within approved window | sustained loss |
| Redaction | synthetic \`test.secret\` attribute | filtered before external backend | any actual secret exposed |
| GPU correlation | approved staging GPU and synthetic trace | time/node mapping documented | unsupported causal assertion |

A debug exporter is suitable only for **synthetic staging payloads**. For end-to-end backend verification, use an approved OTLP backend exporter and observe actual queryable telemetry.

## 6. Evidence record

\`\`\`yaml
chapter: "069"
environment: "staging"
execution_date_utc: null
cluster: "REDACTED"
reviewer: null
collector_version: null
collector_image_digest: null
sdk_versions: []
otel_protocol: null
tls_and_auth_review: "PENDING"
networkpolicy_review: "PENDING"
synthetic_trace_received: "NOT_RUN"
synthetic_metric_received: "NOT_RUN"
gpu_metrics_reconciled: "NOT_RUN"
sensitive_attributes_blocked: "NOT_RUN"
staging_failure_drill: "NOT_RUN"
accepted_telemetry: null
exported_telemetry: null
dropped_telemetry: null
recovery_seconds: null
evidence_links: []
open_risks: []
score: null
acceptance: "NOT_EVALUATED"
\`\`\`

## 7. Scoring and closeout

Apply Chapter 069's six gates: **15 + 20 + 20 + 15 + 15 + 15 = 100**. Require **≥85**, no critical security/privacy failure, and peer-reviewed evidence. Record offline/static validation, live staging execution and independent review independently in [STATUS-TRACKER](../STATUS-TRACKER.md).

**Stop immediately** for unintended production impact, unapproved telemetry export, sensitive content exposure, uncontrolled cardinality or cross-tenant access. Restore staging state and file an incident/change record where appropriate.

**Next:** Chapter 070 — GPU Health and Xid Alerting.
