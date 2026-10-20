# Chapter 096 — Implement GPU SLOs and Observability

**Part 12 — Enterprise Operations and Final Capstones | AI Infrastructure Engineering & SRE**

**Publication:** PUBLISHED. Deployment, telemetry verification, alert testing, incident drills, independent security review and production acceptance **PENDING**.

## 1. Mission and objectives

Implement an end-to-end observability and SLO program for enterprise GPU infrastructure supporting inference, fine-tuning and distributed training on Kubernetes. Connect **GPU hardware telemetry**, Kubernetes scheduling, network/storage, distributed training progress, inference user experience, FinOps and incident response.

By completion, operators should be able to:
1. Define user-centered SLIs and measurable SLOs with valid denominators.
2. Deploy a supported DCGM Exporter → Prometheus → Grafana monitoring path.
3. Correlate GPU metrics with node, Pod, tenant and workload metadata.
4. Instrument inference TTFT, TPOT, p99, errors and successful tokens.
5. Monitor training step throughput, queue wait, NCCL errors and checkpoint age.
6. Implement burn-rate alerting, on-call runbooks, dashboards and evidence-based acceptance.

**Prerequisites:** Chapters 091–095; secure staging GPU cluster; validated driver/CUDA/NCCL; approved monitoring namespaces, RBAC, telemetry storage and network egress; reviewed alert routes.

## 2. Reference telemetry architecture

\`\`\`text
GPU nodes → NVIDIA DCGM Exporter ────────┐
Kubernetes → kube-state-metrics / events ┤
Nodes → node exporter / CNI / CSI metrics├─ Prometheus / metrics backend
Training → step, queue, NCCL, checkpoint ┤          │
Serving → request, TTFT, TPOT, tokens ───┘          ├─ Grafana / SLO dashboards
                                                   └─ Alertmanager / paging
Apps → OpenTelemetry SDK → Collector → traces/logs backend
Cloud billing / scheduler admission → FinOps reconciliation
\`\`\`

Prometheus and DCGM provide infrastructure signals; application SLOs need **application instrumentation**. GPU utilization is not a substitute for user-perceived service health. An OTel Collector does not automatically convert every trace into correct request-based SLIs.

## 3. Define the service boundary and SLO contracts

| Service | SLI | Example synthetic target | Notes |
|---|---|---|---|
| Online inference | successful requests within latency threshold / eligible requests | 99.9% over rolling 30 days | define timeouts, cancellations, streaming semantics |
| First-token latency | requests with TTFT ≤ threshold / eligible streaming requests | 99% ≤ 2 s | model, region and prompt-length cohort |
| Token cadence | requests with TPOT ≤ threshold / eligible requests | 99% ≤ 100 ms/token | define per-request TPOT aggregation |
| Training platform | admitted jobs completing within agreed deadline / eligible jobs | 99% / 30 days | distinguish user code from infrastructure failures |
| GPU capacity | protected jobs admitted within queue SLO / eligible jobs | 99% within 5 min | quota vs physical stock |
| Checkpoint recovery | valid restores within RTO / restore drills | 100% of scheduled drills | restore correctness is mandatory |
| Telemetry pipeline | successful metric scrape intervals / expected intervals | 99.9% | separate monitoring reliability SLO |

**All numbers above are illustrative design targets, not measured or approved objectives.** Select SLOs based on service requirements, baseline data, contract boundaries and stakeholder signoff. Segment by model, hardware class, region and customer tier when their performance profiles differ.

## 4. GPU telemetry collection

NVIDIA DCGM Exporter typically exposes Prometheus-format GPU metrics. Exact metric names and available fields depend on exporter/DCGM version, hardware, MIG mode and configured counters.

Common examples to verify in the installed exporter include:
- \`DCGM_FI_DEV_GPU_UTIL\` — sampled GPU activity, not useful application throughput.
- \`DCGM_FI_DEV_FB_USED\` — framebuffer usage, not complete per-process allocation.
- \`DCGM_FI_DEV_GPU_TEMP\` — device temperature.
- \`DCGM_FI_DEV_POWER_USAGE\` — instantaneous power.
- \`DCGM_FI_DEV_XID_ERRORS\` — error signal requiring code-specific interpretation.
- \`DCGM_FI_DEV_ECC_DBE_VOL_TOTAL\` — where supported, double-bit ECC event history.

**Caveat:** counters may be missing or change behavior with MIG, hardware or driver versions. Do not assume a counter is monotonically increasing without confirming its type and exporter behavior. Test alert expressions against real sample data.

### Read-only discovery

\`\`\`bash
kubectl config current-context
kubectl get pods -A | grep -Ei 'dcgm|prometheus|otel|grafana' || true
kubectl get servicemonitors -A 2>/dev/null || true
kubectl get nodes -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -50
\`\`\`

The \`ServiceMonitor\` resource exists only if the Prometheus Operator CRD is installed. Missing commands or permissions are **BLOCKED**, not proof of success.

## 5. Prometheus integration pattern

Use a reviewed DCGM Exporter chart/operator configuration and verify exporter service port and label selectors. Illustrative ServiceMonitor **only after the target service exists**:

\`\`\`yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: gpu-dcgm
  namespace: monitoring
  labels:
    release: monitoring
spec:
  namespaceSelector:
    matchNames: ["gpu-operator"]
  selector:
    matchLabels:
      app: nvidia-dcgm-exporter
  endpoints:
    - port: metrics
      interval: 30s
      scrapeTimeout: 10s
\`\`\`

This is a template: chart versions may use different namespace, labels or named ports. Validate the actual Service and Prometheus Operator selectors before applying. Protect metrics endpoints and prohibit unauthenticated exposure outside approved network boundaries.

### PromQL discovery examples

\`\`\`promql
up{job=~".*dcgm.*"}
DCGM_FI_DEV_GPU_UTIL
DCGM_FI_DEV_FB_USED
count by (instance) (DCGM_FI_DEV_GPU_UTIL)
\`\`\`

Confirm labels and metric availability in your backend. \`up=1\` proves scraping succeeded; it does not prove every GPU metric is complete or accurate.

## 6. Kubernetes, workload and tenancy correlation

Capture:
- Node identity, GPU UUID, device health and topology.
- Pod UID, namespace, workload controller, job ID, model/version and queue.
- Allocatable/requested GPUs, Pending reasons, restart/OOM, eviction and quota denial.
- Queue wait, scheduling latency and GPU resource fragmentation.
- NIC/storage counters and cross-AZ traffic.

Avoid unbounded high-cardinality labels such as raw request IDs, prompt text, full checkpoint paths or tenant secrets in Prometheus. Use traces/logs for individual request correlation and enforce retention, access controls and PII redaction.

**MIG caution:** a MIG device may not map one-to-one to a physical GPU; aggregate physical and partition-level metrics separately to avoid double counting.

## 7. Inference SLIs: instrument the user journey

For each eligible request, record:
- request start, first token, completion, status and cancellation;
- model version, serving region, routing tier and request class;
- input/output token counts, TTFT and TPOT definition;
- p50/p95/p99, errors, throttles and timeout;
- batching, queue delay, model warmup, KV-cache pressure and HBM usage.

Use Prometheus histograms with agreed buckets for latency and a **single clear request denominator**. Streaming requests require explicit treatment of partial responses and client disconnects. Avoid calculating p99 by averaging pod-level p99 values.

### Example request-success SLI

\`\`\`promql
sum(rate(inference_requests_total{result="success"}[5m]))
/
clamp_min(sum(rate(inference_requests_total[5m])), 1e-9)
\`\`\`

The metric and \`result\` label are **application-defined placeholders**. If the metric does not exist, implement instrumentation before using the query. The denominator must reflect the actual eligible request set and not silently exclude failures.

## 8. Training and distributed-system SLIs

Monitor:
- successful training steps/sec, accelerator idle fraction and dataloader wait;
- job admission/queue age and expected-versus-running GPU ranks;
- NCCL collective duration, retries, failures and selected transport;
- checkpoint age, bytes, commit marker and last successful restore;
- model quality/validation gate and cost per useful training step.

Separate infrastructure incidents from user code failures with an auditable classification process. A training Pod remaining Running does not imply that optimizer steps are progressing.

## 9. SLO burn-rate alerting

For a 99.9% availability SLO, error budget is **0.1%** of eligible requests. Burn rate is observed error ratio divided by 0.001. Example: a 1% error ratio burns budget at **10×** the allowed rate.

### Illustrative recording rule

\`\`\`yaml
groups:
  - name: inference-slo
    rules:
      - record: inference:request_error_ratio:rate5m
        expr: |
          sum(rate(inference_requests_total{result!="success"}[5m]))
          /
          clamp_min(sum(rate(inference_requests_total[5m])), 1e-9)
\`\`\`

This assumes a mutually exclusive success/error classification. Define missing telemetry behavior; a zero denominator should not silently mean a healthy service. Use multi-window, multi-burn-rate alerts with independently reviewed thresholds, for example high burn over 5m **and** 1h for fast paging, and lower burn over 30m **and** 6h for slower escalation. Tune from measured traffic and alert noise.

## 10. GPU and platform alert catalog

| Alert | Trigger concept | Action |
|---|---|---|
| GPU Xid/ECC | new critical hardware fault | isolate/replace GPU after triage |
| DCGM scrape missing | absent telemetry beyond grace | investigate exporter/monitoring |
| GPU node allocatable drop | GPU capacity lost | scheduler/driver triage |
| Inference fast burn | SLO error budget consumption | page serving on-call |
| TTFT/TPOT regression | sustained cohort breach | warmup, HBM, queue, model rollback |
| Training stalled | no successful steps | NCCL, storage, code triage |
| Checkpoint overdue | age exceeds recovery budget | storage/data integrity review |
| Queue starvation | protected jobs exceed admission SLO | capacity/priority investigation |
| Network/fabric errors | retransmit/timeout anomaly | fabric on-call |
| Cost per successful unit spike | unit economics regression | investigate retries/idle GPU |

Deduplicate related alerts and route by ownership. Do not page for high GPU utilization alone when throughput and SLOs are healthy.

## 11. Grafana dashboard design

**Executive SLO:** availability, p99, TTFT/TPOT, budget remaining, open incidents.

**GPU fleet:** SKU/AZ, allocatable versus allocated, Xid/ECC, power, temperature, HBM, MIG partition inventory.

**Training:** jobs admitted/running/failed, steps/sec, queue age, NCCL latency, checkpoint age and restore results.

**Inference:** model version, requests/sec, token throughput, p99, warm replicas, throttling, HBM and error rates.

**Platform/FinOps:** storage/network, cluster health, GPU-hours, cost per million successful tokens, cost per successful training step, reservations and forecast.

Add runbook links, owners, units, dashboard time zone and freshness indicators. Dashboards without defined data ownership and alert tests are not production acceptance.

## 12. OpenTelemetry and tracing

Instrument API gateway → model router → serving runtime → approved data access, with trace context propagated across service boundaries. Use sampling suited to error and latency investigations; high-volume token-level spans can be expensive. Redact prompts, personal data, credentials and proprietary artifacts. Correlate with logs and metrics using trace IDs where supported, without turning IDs into high-cardinality metric labels.

## 13. Error budget and change policy

Document the SLO window, exclusions, incident ownership and error-budget policy. For example:
- Normal budget: controlled rollouts permitted with canary gates.
- Rapid burn: freeze risky model/driver/operator promotions.
- Budget exhausted: require reliability remediation and approval for nonessential changes.
- Restore confidence: demonstrate SLI recovery and root-cause controls.

Avoid a blanket freeze that blocks emergency fixes; define exception authority and audit.

## 14. Six production runbooks

**R1 — Inference SLO burn:** validate request denominator and telemetry, identify model/region, inspect queue, HBM and GPU health, throttle or rollback, verify recovery.

**R2 — GPU Xid/ECC:** correlate node/GPU UUID, exact error code and affected Pods; protect workloads, isolate unhealthy node, preserve logs and arrange replacement.

**R3 — Training progress stalls:** compare GPU usage with steps/sec, NCCL transport, dataloader and checkpoint age; restore from verified checkpoint.

**R4 — Monitoring blind spot:** investigate DCGM exporter, scrape targets, relabeling, RBAC, CNI, retention and remote-write; activate alternate health checks.

**R5 — GPU queue saturation:** separate quota from physical stock, identify protected admission SLO, rebalance approved queues and scale compatible capacity.

**R6 — Cost and performance divergence:** reconcile billed GPU-hours against successful steps/tokens, inspect retries and idle reservations; avoid unsafe rightsizing.

## 15. Sixteen hands-on exercises

1. Define service boundaries and approved SLIs.
2. Write synthetic inference/training SLO contracts.
3. Inventory DCGM, Prometheus and OTel components.
4. Validate GPU metric availability and units.
5. Configure a staging ServiceMonitor with verified selectors.
6. Build a GPU fleet dashboard.
7. Correlate Pod/tenant/GPU identities safely.
8. Instrument inference request and token metrics.
9. Validate latency histograms and percentile aggregation.
10. Instrument training steps, NCCL and checkpoint age.
11. Create error-budget recording rules.
12. Configure multi-window burn-rate alerts.
13. Test alert delivery and escalation in staging.
14. Simulate telemetry outage and one workload SLO breach.
15. Rehearse incident recovery and change freeze policy.
16. Assemble evidence and independent SLO acceptance.

## 16. Twelve troubleshooting scenarios

| Symptom | Primary investigation |
|---|---|
| DCGM exporter Running, no metrics | Service/selector/endpoint |
| GPU utilization 0 with active training | job stalled or missing metric |
| \`up=1\`, GPU inventory incomplete | MIG/device mapping |
| p99 appears low despite complaints | missing failed requests/incorrect cohort |
| Error budget jumps after deploy | label/result schema regression |
| TTFT high but GPU usage low | queue/model loading/network |
| GPU utilization high but SLO healthy | avoid false alert |
| NCCL failure not visible | training instrumentation gap |
| Checkpoint overdue without page | missing alert route |
| Grafana and Prometheus disagree | query/step/aggregation |
| OTel trace leaks prompt text | privacy redaction |
| Cost rises while throughput flat | idle GPUs/retries/overprovision |

## 17. Acceptance rubric (100 points)

| Domain | Points |
|---|---:|
| SLI definitions, SLOs and error budgets | 20 |
| DCGM/Prometheus GPU collection | 20 |
| Inference/training instrumentation | 20 |
| Dashboards and cross-layer correlation | 15 |
| Alerting, incident response and resilience | 15 |
| Evidence, security and independent review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 and no critical SLI denominator error, telemetry blind spot, missing GPU health alert, broken paging route, privacy leak, unverified rollback or absent independent signoff. **No tests have been run as part of documentation publication.**

## 18. References

- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
- [Prometheus Monitoring](https://prometheus.io/docs/introduction/overview/)
- [Prometheus Alerting Rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Prometheus Histograms and Summaries](https://prometheus.io/docs/practices/histograms/)
- [OpenTelemetry](https://opentelemetry.io/docs/)
- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Kubernetes Resource Metrics Pipeline](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)

**Next:** Chapter 097 — Run GPU Hardware and Driver Incident Exercises.
