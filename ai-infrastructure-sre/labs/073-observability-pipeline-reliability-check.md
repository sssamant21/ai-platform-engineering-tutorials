# Lab 073 — Observability Pipeline Reliability Acceptance Gate

**Chapter:** [073 — Observability Pipeline Reliability](../part-09-gpu-observability/073-observability-pipeline-reliability.md)

**Execution status:** NOT RUN. This is a reproducible test plan and evidence template, not a completed validation report.

## 1. Preconditions and safety

Use read-only access for discovery. For any disruption, use a disposable staging namespace with isolated synthetic telemetry and a test notification receiver. Do not stop production Prometheus, delete TSDB/WAL/PVC, block production exporters, or induce GPU failures. Verify security, data residency and telemetry privacy before exporting samples.

## 2. Fourteen-exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | Pipeline inventory | source/backend/version/owner matrix | Pending |
| E02 | Failure-domain map | diagram and blast-radius review | Pending |
| E03 | SLI definitions | numerator, denominator, target, exclusions | Pending |
| E04 | Expected GPU targets | inventory vs discovery | Pending |
| E05 | TSDB/storage health | retention, PVC, disk and WAL status | Pending |
| E06 | Collector telemetry | accepted/refused/exported/queue metrics | Pending |
| E07 | Capacity budget | peak samples, bytes, cardinality | Pending |
| E08 | Collector config validation | pinned version, TLS/auth review | Pending |
| E09 | HA/deduplication | replicas, labels and alert routing | Pending |
| E10 | Staging backend outage | queue, retry, lag and loss timeline | Pending |
| E11 | Staging restart | buffer durability and data receipt | Pending |
| E12 | Synthetic missing-target alert | isolated notification lifecycle | Pending |
| E13 | DR tabletop | RPO/RTO, backup/restore sequence | Pending |
| E14 | Acceptance | reviewer, score and open risks | Pending |

## 3. Read-only command pack

\`\`\`bash
kubectl get pods -A -o wide | grep -Ei 'prometheus|alertmanager|otel|dcgm'
kubectl get servicemonitors,podmonitors,prometheusrules -A
kubectl get pvc -A
kubectl get networkpolicy -A
kubectl get events -A --sort-by=.lastTimestamp | tail -80
kubectl -n <monitoring-namespace> describe pod <prometheus-pod>
kubectl -n <otel-namespace> logs deploy/<collector-deployment> --tail=100
\`\`\`

Prometheus query examples to validate against actual metric names:

\`\`\`promql
up{job=~".*dcgm.*"}
scrape_samples_scraped{job=~".*dcgm.*"}
scrape_duration_seconds{job=~".*dcgm.*"}
count(DCGM_FI_DEV_GPU_UTIL)
\`\`\`

None of these independently proves expected target coverage. Record the authoritative inventory denominator separately.

## 4. Static validation and controlled fault drill

1. Review and pin Collector distribution/version; run \`otelcol-contrib validate --config=config.yaml\` using the matching binary.
2. Verify exporter TLS/auth and no sensitive attributes; test only synthetic payloads.
3. In a staging-only Collector/backend pair, record accepted/exported/refused item counts, queue size, process memory and baseline lag.
4. Disable the **staging test backend only**, for a bounded approved interval. Observe retries, queue behavior and any refused/dropped items.
5. Restore the backend, record recovery time, verify receipt and identify duplicates/gaps.
6. Restart the **staging** Collector and document whether buffered data survives; do not infer persistence from successful startup.
7. Validate a separate synthetic alert through the staging Alertmanager receiver.
8. Stop immediately if the drill affects non-staging traffic, exposes payloads or exceeds approved resource limits.

## 5. Synthetic capacity worksheet

Assume **illustrative only**: 200,000 active series, 30-second scrape interval, 15-day retention.

\`\`\`python
series = 200_000
scrape_interval_seconds = 30
retention_days = 15
samples_per_second = series / scrape_interval_seconds
samples_per_day = samples_per_second * 86400
samples_retained = samples_per_day * retention_days
print({"samples_per_second": round(samples_per_second, 2),
       "samples_per_day": round(samples_per_day),
       "samples_retained": round(samples_retained)})
\`\`\`

Use measured compressed bytes/sample, WAL/index/compaction overhead, HA replicas and remote-write amplification to derive actual storage. Do not use a guessed fixed disk size as an acceptance result.

## 6. Evidence record

\`\`\`yaml
chapter: "073"
environment: "STAGING_OR_OFFLINE"
date_utc: null
reviewer: null
prometheus_version: null
collector_distribution_and_version: null
expected_gpu_targets: null
observed_gpu_targets: null
telemetry_coverage_sli: null
ingest_freshness_p95_seconds: null
otel_accepted_items: null
otel_exported_items: null
otel_refused_items: null
otel_dropped_items: null
max_queue_depth: null
backend_outage_seconds: null
recovery_seconds: null
restart_buffer_persistence: "NOT_RUN"
synthetic_alert_delivery: "NOT_RUN"
dr_tabletop: "NOT_RUN"
tls_auth_review: "PENDING"
privacy_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 7. Acceptance and stop conditions

Score Chapter 073's six gates **15 + 20 + 20 + 15 + 20 + 10 = 100**. Pass at **≥85**, with no critical security/privacy or unintended data-loss failure. Record offline/static validation, live staging results and independent review separately in the status tracker.

**Next:** Chapter 074 — GPU Observability Capstone.
