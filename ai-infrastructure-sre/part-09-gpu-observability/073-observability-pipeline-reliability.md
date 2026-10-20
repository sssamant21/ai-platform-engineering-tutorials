# Chapter 073 — Observability Pipeline Reliability

**Part 09 — GPU Observability | AI Infrastructure Engineering & SRE**

**Status:** Published documentation; offline validation, live staging exercises, security review and independent production acceptance are pending.

## 1. Learning outcomes

Design reliable GPU and AI workload telemetry pipelines across DCGM Exporter, Prometheus, OpenTelemetry Collector, remote-write systems, tracing/log backends and Grafana. Detect missing telemetry, distinguish ingestion from query outages, size buffers, implement HA without silently duplicating data, and recover observability without compromising production workloads.

**Prerequisites:** Chapters 067–072; read-only Kubernetes, Prometheus and OTel access; isolated staging for controlled fault drills. All configuration examples are templates that must be checked against the actual deployed versions.

## 2. Failure domains and reliability contract

\`\`\`text
GPU / workload / kubelet
   ├─ DCGM exporter ──> Prometheus scrape ──> TSDB / remote_write ─┐
   ├─ OTel SDK ───────> Collector agent/gateway ──> OTLP backend ───┤
   ├─ pod/node logs ──> approved log agent ──> log store ──────────┤
   └─ Kubernetes events ─> event collector ──> event store ────────┤
                                                                     v
                                                  query / dashboard / alerts
                                                   + incident evidence
\`\`\`

A healthy Grafana dashboard does **not** prove all sources are healthy. The critical reliability questions are: (1) was telemetry produced, (2) collected, (3) persisted, (4) queryable within the freshness SLO, and (5) usable by alert evaluation?

| Failure domain | Observable symptoms | Correct response |
|---|---|---|
| Source failure | DCGM exporter down, SDK stopped emitting | compare expected inventory and source health |
| Discovery failure | vanished scrape targets, stale service discovery | independent target inventory, ServiceMonitor checks |
| Collector overload | refused data, OOM, queue pressure | investigate load, memory and backend |
| Transport/auth | TLS failure, 401/403, DNS errors | verify trust and credentials, do not disable TLS |
| Backend ingest | throttling, write errors, lag | control retries and load, escalate capacity |
| Storage | TSDB disk pressure, retention, compaction issues | protect WAL/retention, plan capacity |
| Query path | dashboard empty despite stored data | datasource, tenant, time range and query limits |
| Alerting | rules not evaluated, Alertmanager unreachable | separate alerting-plane SLO and synthetic checks |
| Control plane | Collector config or CRD rejected | validate versions, staged rollout, rollback |

## 3. SLIs, SLOs and error budgets

Specify signal-specific SLOs rather than one ambiguous “telemetry availability” number.

| SLI | Definition | Illustrative target | Measurement note |
|---|---|---|---|
| GPU telemetry coverage | expected healthy GPU-minutes with valid fresh DCGM samples / expected healthy GPU-minutes | ≥99.9% / 30d | independent GPU inventory denominator |
| Prometheus scrape success | successful scrapes / expected attempted scrapes | ≥99.9% / 30d | discovery loss must be counted separately |
| OTLP export success | accepted items successfully exported / accepted items eligible for export | ≥99.5% / 30d | exclude deliberate sampling with explicit policy |
| Freshness | p95 source-to-queryable latency | <60 s | requires source and backend timestamp comparison |
| Alert evaluation | expected evaluation intervals completed on time / expected intervals | ≥99.9% / 30d | independent synthetic verification |
| Incident queryability | successful bounded diagnostic queries / attempted diagnostic queries | ≥99.9% / 30d | query errors and timeouts counted |

These are **design examples**, not validated fleet commitments. For a 99.9% 30-day SLO, the bad-event budget is 0.1% of the defined eligible denominator. Do not count “no metrics arrived” as zero errors: use independent synthetic traffic, target inventory and backend receipt checks.

## 4. Prometheus scrape and storage reliability

### 4.1 Discovery and scrape

\`\`\`promql
# Only catches targets that are still discovered
up{job=~".*dcgm.*"} == 0

# Scrape duration relative to scrape timeout, by target
scrape_duration_seconds{job=~".*dcgm.*"}

# Scrape sample count, useful for unexpected drops
scrape_samples_scraped{job=~".*dcgm.*"}

# Observed GPU series count; NOT independent expected inventory
count(DCGM_FI_DEV_GPU_UTIL)
\`\`\`

A disappeared target may have **no \`up\` series**. Reconcile against a separately maintained expected target list, such as GPU node inventory plus an approved exporter discovery inventory. Beware MIG changes and intentional maintenance exclusions.

### 4.2 WAL, retention and storage

Prometheus uses a write-ahead log and local TSDB; protect persistent storage where required, monitor free bytes/inodes, compaction, block retention, ingest rate and restart time. Do not delete WAL segments, TSDB blocks or PVCs as an emergency “cleanup” without a tested recovery plan. Configure retention by time and/or size with a capacity model and document loss tolerance.

Estimate ingest volume:

\`\`\`text
samples_per_second = active_series / scrape_interval_seconds
raw_samples_per_day = samples_per_second * 86400
storage_budget = measured_bytes_per_sample * raw_samples_per_day * retention_days
               + WAL / index / compaction / head-block overhead
               + replication / remote-write overhead
\`\`\`

Use measured compression and cardinality, not a universal bytes/sample constant. Validate disk alert thresholds against ingest peaks and restart/compaction requirements.

## 5. HA architecture and duplicate handling

- **Prometheus HA:** two independent Prometheus replicas scrape the same targets; remote-write/backend must support replica-label deduplication where appropriate. Independent replicas increase scrape availability but also ingest cost.
- **Alerting HA:** Alertmanager peers and routing redundancy need actual cluster validation. Duplicate suppression depends on consistent external labels, grouping and peer connectivity.
- **OTel gateways:** multiple stateless Collector replicas behind a Service improve availability; **tail sampling and stateful processors require trace affinity or an appropriate routing design**. Random load balancing can split traces and corrupt decisions.
- **Persistent queues:** exporter sending queues and file storage extensions are distribution/version-specific. Validate component availability and mounted durable storage before claiming data survives pod loss.
- **Log agents:** prevent two agents from ingesting the same container stream without documented deduplication.

Do not claim exactly-once delivery. OTLP retries, remote-write replay and HA scrapes may result in duplicates. Choose backend deduplication, idempotent labels and clear data-quality checks.

## 6. OpenTelemetry Collector backpressure

Collector metrics and component names differ across releases. Inspect the actual Collector's internal telemetry endpoint and emitted metrics; do not hardcode a metric name without verification.

| Signal | Meaning | Risk |
|---|---|---|
| Receiver accepted/refused items | input admitted versus rejected | loss or upstream retries |
| Exporter sent/failed items | downstream outcome | backend outage |
| Queue occupancy and capacity | buffered load | overflow and loss |
| Retry duration / attempts | backpressure | sustained delivery lag |
| Process RSS and CPU | resource pressure | OOM and restart |
| Dropped data / sampling | intentional vs unintentional loss | false reliability SLO |
| Ingest-to-query lag | end-to-end freshness | invisible current incidents |

A Collector's in-memory queue is lost on pod termination. If using disk-backed queues, confirm storage extension support, permissions, disk limits, restart semantics and backpressure behavior with the pinned version. Size capacity from measured peak event rate, bytes/item, expected outage duration and headroom.

\`\`\`text
required_buffer_bytes ≈ peak_items_per_second × mean_bytes_per_item
                      × tolerated_backend_outage_seconds × safety_factor
\`\`\`

This is a lower-bound planning approximation, not a guarantee; include batch overhead, indexes, retry amplification and storage availability.

### 6.1 Example minimal pipeline (staging only)

\`\`\`yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 256
    spike_limit_mib: 64
  batch:
    timeout: 5s
    send_batch_size: 512
exporters:
  debug:
    verbosity: basic
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug]
\`\`\`

This example intentionally has **no durable queue and no production backend**. The debug exporter is only for synthetic non-sensitive staging data. Production requires TLS/authenticated exporter, privacy policy, version-pinned Collector and proven restart behavior. Validate with the matching Collector binary, e.g. \`otelcol-contrib validate --config=config.yaml\`, before any deployment.

## 7. Disaster recovery and restoration

Define per-signal RPO/RTO: metrics may tolerate short gaps for dashboards but not for incident evidence; security audit logs may have stricter retention and delivery obligations. Identify which stores support snapshots, object storage, remote-write replay or backup/restore. A replicated Prometheus scrape is not equivalent to durable historical backup.

**Recovery order:** restore independent monitoring of the observability stack → recover authentication/network and ingest → confirm new data receipt → verify alert evaluation → restore historical queryability → reconcile gaps and duplicates → document loss. Preserve WAL and configuration snapshots before invasive actions. Use only vendor-supported procedures for the installed backend.

## 8. Alerting and burn-rate strategy

Design independent alerts for: missing expected GPU targets, scrape failure, Collector exporter failures, queue saturation, disk capacity, ingestion freshness, rule evaluation failures and Alertmanager notification delivery. Use severity routing and multiwindow burn-rate thresholds for SLO-impacting loss, not every transient retry.

Alert on telemetry **blindness** as a separate condition; do not suppress hardware alerts by interpreting missing telemetry as healthy. Alert delivery itself needs synthetic end-to-end tests with a non-production receiver.

## 9. Fourteen hands-on exercises

1. Inventory telemetry sources, backends, versions, replicas and ownership.
2. Draw a failure-domain diagram for metrics, logs, traces and events.
3. Define source-to-query SLIs, denominators and exclusions.
4. Reconcile expected GPU exporter targets with discovered scrape targets.
5. Inspect Prometheus TSDB, WAL, retention and disk headroom.
6. Verify OTel Collector receiver/exporter telemetry and queue signals.
7. Estimate ingest, series cardinality and storage under peak load.
8. Validate a staged Collector config and authenticated export contract.
9. Inspect HA Prometheus and Alertmanager replica/label/deduplication design.
10. Test a **staging-only** exporter/backend outage and record retries/loss.
11. Restart a disposable staging Collector to assess buffer persistence.
12. Test a synthetic missing-target alert and an isolated notification receiver.
13. Tabletop observability backend DR and historical-data restoration.
14. Assemble evidence, runbooks, review and acceptance score.

## 10. Troubleshooting matrix

| Symptom | First checks | Avoid |
|---|---|---|
| Dashboard empty | source, target discovery, backend query, tenant | assuming healthy system |
| Scrapes fail | exporter logs, ServiceMonitor, TLS, DNS | disabling security |
| Collector OOM | batch, queue, CPU/RSS, downstream errors | unbounded memory |
| Remote-write lag | pending samples, retries, backend quotas | blind queue increases |
| Duplicate alerts | HA external labels, Alertmanager peer health | deleting all replicas |
| TSDB disk full | retention, ingest/cardinality, PVC capacity | deleting WAL manually |
| Trace gaps | SDK sampling, context, tail-sampling affinity | treating absence as success |
| DR restored but no alerts | rule evaluation, routing, test receiver | closing incident early |

## 11. Production runbooks

**A — Ingestion loss:** determine source vs discovery vs receiver vs exporter vs backend; measure loss and affected time window; restore safely; verify receipt and record irrecoverable gaps.

**B — Collector saturation:** inspect accepted/refused/exported metrics, queue occupancy, CPU/RSS and backend throttling; limit traffic or scale under approval; validate no unexpected drop after recovery.

**C — Prometheus disk pressure:** quantify WAL/head/blocks and retention, validate PVC capacity and compaction, expand capacity or reduce approved retention via change control; never manually delete data without a recovery plan.

**D — Observability DR:** restore minimal independent alerting, ingestion, historical storage and queries in order; test a synthetic end-to-end alert and document RPO/RTO and gaps.

## 12. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Failure-domain inventory and source coverage | 15 |
| Correct SLI/SLO definitions and error budgets | 20 |
| Prometheus/OTel configuration and capacity review | 20 |
| HA, duplicate and security controls | 15 |
| Staging outage/recovery and DR tabletop | 20 |
| Runbooks, evidence and peer review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 and no critical data-loss, privacy, security or unapproved-production-impact failure. Publication is not proof of validation.

## 13. References

- [Prometheus Storage](https://prometheus.io/docs/prometheus/latest/storage/)
- [Prometheus Remote Write](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write)
- [Prometheus Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [OpenTelemetry Collector Resiliency](https://opentelemetry.io/docs/collector/resiliency/)
- [Prometheus Operator](https://prometheus-operator.dev/)

**Next:** Chapter 074 — GPU Observability Capstone.
