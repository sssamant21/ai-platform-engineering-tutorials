# Lab 096 — GPU SLOs and Observability Acceptance Gate

**Chapter:** [096 — Implement GPU SLOs and Observability](../part-12-enterprise-operations-and-final-capstones/096-gpu-slos-observability.md)

**Execution:** NOT RUN. GPU telemetry installation, metrics validation, staging alert delivery, incident exercises and independent review remain pending.

## 1. Safety

Use approved staging monitoring and test workloads. Do not modify production alert routing, install privileged exporters, change retention, expose Prometheus or generate synthetic customer-facing errors without approval. Never capture raw prompts, credentials or personal data in telemetry.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | Status |
|---|---|---|---|
| E01 | Service/SLI inventory | ownership register | Pending |
| E02 | SLO contracts | approved numerator/denominator | Pending |
| E03 | Exporter inventory | chart/version/pod state | Pending |
| E04 | GPU metric verification | sample metric/units | Pending |
| E05 | ServiceMonitor | scrape/selector evidence | Pending |
| E06 | GPU fleet dashboard | reviewed panels | Pending |
| E07 | Identity correlation | labels/cardinality review | Pending |
| E08 | Inference metrics | counters/histograms | Pending |
| E09 | Percentile validation | histogram query proof | Pending |
| E10 | Training/NCCL/checkpoints | progress metrics | Pending |
| E11 | Error-budget rules | rule tests | Pending |
| E12 | Multi-window alerts | burn-rate simulation | Pending |
| E13 | Alert routing | test page/ack | Pending |
| E14 | Blind spot/SLO drill | incident timeline | Pending |
| E15 | Runbook and freeze policy | recovery evidence | Pending |
| E16 | Independent review | score/COAs | Pending |

## 3. Read-only discovery

\`\`\`bash
kubectl config current-context
kubectl get pods -A | grep -Ei 'dcgm|prometheus|grafana|otel' || true
kubectl get servicemonitors -A 2>/dev/null || true
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

Missing CRDs/permissions are BLOCKED. A healthy exporter Pod is not sufficient to pass GPU coverage or SLO instrumentation.

## 4. Offline SLO arithmetic

\`\`\`python
target = 0.999
eligible_requests = 1_000_000
errors = 1_000
budget = eligible_requests * (1 - target)
observed_error_ratio = errors / eligible_requests
burn_rate = observed_error_ratio / (1 - target)
assert round(budget) == 1000
assert round(burn_rate, 6) == 1.0
print("synthetic allowed errors:", round(budget))
print("synthetic burn rate:", round(burn_rate, 2))
\`\`\`

This is **synthetic arithmetic**, not a measured SLO result. Production SLO windows and budgets require approved definitions and observed eligible traffic.

## 5. PromQL staging checks

\`\`\`promql
up{job=~".*dcgm.*"}
DCGM_FI_DEV_GPU_UTIL
DCGM_FI_DEV_FB_USED
\`\`\`

Use actual installed labels and verify hardware coverage, GPU UUID mapping, metric units and missing data. Check that dashboards do not double-count MIG partitions.

## 6. Alert test matrix

| Scenario | Expected result |
|---|---|
| Synthetic inference error burst | fast-burn alert fires |
| Sustained moderate errors | slow-burn route |
| DCGM scrape failure | telemetry-missing alert |
| Critical Xid/ECC | GPU on-call notified |
| Training steps stall | progress alert |
| Checkpoint age exceeds threshold | recovery alert |
| Alertmanager route unavailable | alternate escalation |
| Prometheus no-data | not silently healthy |

Record alert rule version, synthetic event, time to detect, route, acknowledgment, incident owner and recovery.

## 7. Evidence manifest

\`\`\`yaml
chapter: "096"
environment: "GPU_STAGING_REQUIRED"
publication: "PUBLISHED"
execution: "NOT_RUN"
execution_date_utc: null
cluster: null
dcgm_exporter_version: null
prometheus_version: null
otel_collector_version: null
gpu_metric_coverage: "NOT_VERIFIED"
inference_sli_denominator: "NOT_VERIFIED"
training_progress_metrics: "NOT_VERIFIED"
histogram_quantiles: "NOT_VERIFIED"
slo_burn_rate_rules: "NOT_TESTED"
gpu_hardware_alerts: "NOT_TESTED"
alert_routing: "NOT_TESTED"
telemetry_outage_drill: "NOT_RUN"
incident_runbooks: "NOT_TESTED"
privacy_review: "PENDING"
independent_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
corrective_actions: []
\`\`\`

## 8. Acceptance

Score **20 + 20 + 20 + 15 + 15 + 10 = 100**. Pass ≥85 only after GPU telemetry coverage, correct SLI definitions, alert delivery, privacy, failure drills and independent review meet all critical gates.

**Next:** Chapter 097 — Run GPU Hardware and Driver Incident Exercises.
