# Chapter 006 — AI Infrastructure Reliability, SLOs and Error Budgets

**Part 01 — Foundations and Architecture · Chapter 006 of 100**  
**Status:** Published. Offline lab execution, production telemetry validation, security review and SLO approval pending.

## Learning outcomes
- Define SLIs/SLOs for GPU capacity, Kubernetes scheduling, model serving, distributed training, checkpoint recoverability and platform observability.
- Distinguish user-facing availability from infrastructure health and distinguish failed requests from incomplete token streams.
- Calculate error budgets, burn rates, multi-window alert conditions, N-1 capacity and recovery targets using consistent units.
- Establish incident severity, change freezes, canary/rollback gates and a reliability review.
- Execute an offline Python evaluator and pytest suite without requiring cloud credentials.

## 1. Reliability starts with the user journey
A GPU can be healthy while users experience high time-to-first-token (TTFT), slow time-per-output-token (TPOT), truncated streams or RAG dependency errors. Conversely, an isolated GPU node may fail without violating the user SLO if sufficient healthy capacity exists. Define reliability at multiple levels:

```text
User / training objective
          |
API, admission, model routing
          |
Inference or training service
          |
Kubernetes / Slurm / Ray scheduling
          |
GPU hosts + driver + CUDA/NCCL
          |
network / identity / object store / checkpoints
          |
telemetry / paging / incident response
```

**Golden rule:** Platform SLIs explain service reliability; they do not replace end-to-end user SLIs.

## 2. Service-level indicator catalog
| Service | SLI numerator / denominator | Common pitfall |
|---|---|---|
| Inference availability | correctly completed eligible requests / eligible requests | treating HTTP 200 with truncated stream as success |
| TTFT latency | eligible streams meeting TTFT target / eligible measured streams | excluding timeouts or queue wait |
| TPOT latency | eligible streams meeting token cadence target / eligible measured streams | averaging away long-tail decode |
| GPU allocation | GPU requests scheduled within objective / eligible GPU requests | excluding failed admissions silently |
| GPU capacity readiness | schedulable healthy GPU units / intended GPU units | equating node Ready with GPU allocatable |
| Training job success | jobs completing within policy / eligible completed-or-expired jobs | hiding failed or timed-out jobs |
| Checkpoint restore | successful verified restores / attempted restore drills | counting checkpoint upload as recoverability |
| Observability | required synthetic probes and critical telemetry healthy / scheduled checks | silent missing metrics |

Define the population, exclusions, units, sampling interval, ownership and instrumentation. Document when data is missing; do not silently count unknown as healthy.

## 3. SLOs and error budgets
For a request-based SLO:
```text
SLI = good_events / eligible_events
error_budget_fraction = 1 - SLO_target
allowed_bad_events = floor(eligible_events * error_budget_fraction)
consumed_fraction = bad_events / allowed_bad_events
```
For a 99.9% SLO over 1,000,000 eligible requests, the error budget is **1,000 bad requests**. A 99.95% SLO yields 500. A 99.9% monthly availability SLO corresponds to approximately 43.2 minutes of unavailability over a 30-day month if measured as continuous time, not request counts. These are different measurement models.

**Example:** 700 bad requests in 1,000,000 at 99.9% SLO consumes 70% of the budget. The service can still be within SLO, but aggressive release velocity may be inappropriate if the budget is burning rapidly.

### Error-budget policy
- Below a policy-defined burn threshold: normal change review and canary deployment.
- Elevated burn: prioritize reliability work, restrict risky launches and investigate root causes.
- Exhausted budget: pause nonessential changes; exceptions require accountable approval and documented risk.
- Security incidents, tenant isolation breaches or data corruption are **hard stop conditions** regardless of error-budget remaining.

Do not treat an error budget as permission to cause incidents.

## 4. Burn rates and multi-window alerts
Burn rate compares observed bad-event fraction with the budget fraction:
```text
burn_rate = observed_bad_fraction / (1 - SLO_target)
```
For a 99.9% SLO, a 1% error fraction produces 10× burn. Multi-window alerts reduce noise by requiring sustained burn across a short and a longer window. Example thresholds used in common SRE practices include 14.4× for 1h and 5m, and 6× for 6h and 30m; these are examples, not universal defaults. Use sufficient traffic volume and valid telemetry before paging.

### PromQL illustration
For a counter that marks completed **eligible** inference streams:
```promql
sum(rate(ai_inference_streams_total{result="bad"}[5m]))
/
clamp_min(sum(rate(ai_inference_streams_total[5m])), 1e-9)
```
The instrumentation must count both successful and unsuccessful terminal states and avoid duplicated retries. This ratio alone is not an alerting policy; add low-traffic handling, longer window, burn-rate threshold, missing-telemetry alerts and routing.

## 5. GPU and workload-specific SLO design
**Online inference:** end-to-end availability, p95/p99 TTFT, TPOT, streaming completion, tenant-level throttling, queue wait, GPU memory/KV pressure and cost per successful token.

**Training:** scheduling wait, useful tokens/s, checkpoint interval, recoverability, lost compute-hours per failure, job deadline attainment, and data cursor/optimizer consistency.

**Platform:** allocatable GPU capacity, device-plugin readiness, driver compatibility, quota, node provisioning time, RDMA fabric health, dataset I/O and control-plane recovery.

**Storage/DBRE:** checkpoint restore success, backup freshness, manifest integrity, object-store latency and vector/index freshness where applicable.

**FinOps:** budget adherence, GPU-hours per completed training objective, GPU utilization at passing SLOs and cost per million delivered tokens.

## 6. N-1 capacity and reliability
Suppose three independent zones each sustain 100 requests/s at the target latency. With one zone unavailable, capacity is 200 requests/s. Reserving 20% gives an admission ceiling of 160 requests/s. Verify safe throughput under real token mixes, prompt lengths, concurrency, model revisions and warmup time. Capacity cannot be inferred from theoretical GPU FLOPS alone.

**Important:** Cross-zone shared dependencies (IAM, KMS, DNS, model registry, object storage, regional quota) can invalidate independence. Test those separately.

## 7. [SAFE-READ] Telemetry and health discovery
Run only in approved contexts:
```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -A -o wide
kubectl get daemonsets -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl get resourcequota -A
nvidia-smi --query-gpu=index,name,utilization.gpu,memory.used,memory.total,temperature.gpu --format=csv
```
`nvidia-smi` requires an NVIDIA GPU and driver. These commands provide point-in-time evidence, not SLO proof. Avoid exposing tenant metadata or private infrastructure details in public issue trackers.

## 8. Production incident simulations
**A — Streaming truncation:** gateway returns HTTP 200 but streams terminate early. Verify correct denominator, terminal events and burn-rate alerts.

**B — GPU node loss:** one zone loses GPU capacity. Validate N-1 safe throughput, queueing, admission and latency SLO.

**C — NCCL hang:** distributed training workers stall. Check rank failures, fabric, checkpoint age and job deadline impact.

**D — Checkpoint corruption:** upload succeeds but restore fails. Treat restore SLI as bad and block unsafe recovery claims.

**E — Telemetry blackout:** DCGM exporter/collector fails. Trigger missing-data alert and independent synthetic probe; never infer service health from absent data.

**F — Retry storm:** failed requests are retried aggressively, inflating GPU demand and cost. Check idempotency, client backoff and SLO event accounting.

## 9. Troubleshooting matrix
| Symptom | Likely cause | Evidence |
|---|---|---|
| High TTFT, normal TPOT | queue or prefill | traces, prompt lengths, GPU |
| Normal TTFT, high TPOT | decode/KV pressure | GPU memory, token cadence |
| GPU Ready, allocation SLO poor | device plugin, quota, scheduling | allocatable, events |
| Training deadline missed | I/O, NCCL, preemption | per-rank traces, job history |
| Checkpoint RPO violated | storage or job cadence | manifest timestamps, restore |
| Error budget jumps | truncated streams, bad deploy | terminal events, canary |
| Missing SLO data | exporter/collector failure | scrape targets, synthetic probes |
| Cost spikes | retry storm or idle reservation | token throughput, GPU-hours |

## 10. Operational runbooks
**RB-006-A — SLO breach:** validate event accounting and missing data, classify impact, declare incident, identify affected tenants and stop unsafe rollout.

**RB-006-B — Fast budget burn:** compute burn rate across multiple windows, page appropriate owner, enforce change restrictions and validate rollback.

**RB-006-C — GPU capacity incident:** inspect zone-safe capacity, quotas, warm replicas, device-plugin health and admission policy; avoid uncontrolled autoscaling.

**RB-006-D — Training reliability:** inspect checkpoint age, rank logs, dataset I/O and NCCL; recover only from verified consistent state.

**RB-006-E — Observability failure:** compare synthetic probes, telemetry ingestion and pager delivery; flag unknown state and restore instrumentation before testing changes.

**RB-006-F — Error-budget review:** produce dated SLO evidence, reliability backlog, incident RCA/COA, exception approvals and next review.

## 11. 100-point acceptance rubric
| Domain | Points |
|---|---:|
| User-journey SLIs and instrumentation | 20 |
| Error budgets and burn-rate math | 20 |
| GPU/training reliability and capacity | 20 |
| Incident/change/rollback governance | 20 |
| Lab, evidence, runbooks and review | 20 |
| **Total** | **100** |

Educational pass >=90 with no security/data-integrity hard stop. The offline lab uses synthetic counters and does not certify live SLOs.

## 12. Companion lab
[Chapter 006 — SLO and Error-Budget Acceptance Lab](../labs/006-ai-infrastructure-reliability-slos-error-budgets-check.md) provides a deterministic Python 3.11+ evaluator and pytest suite.

## References
- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Google SRE Book — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [Prometheus Querying Basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

**Next:** Chapter 007 — Infrastructure Security and Trust Boundaries.
