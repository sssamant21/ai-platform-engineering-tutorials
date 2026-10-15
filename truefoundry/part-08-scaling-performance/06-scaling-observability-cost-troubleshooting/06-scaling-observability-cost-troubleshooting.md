# 8.6 — Scaling Observability, Cost and Troubleshooting

**Part 08 — Scaling & Performance | Chapter 6 of 7**  
**Audience:** TrueFoundry platform engineers, GPU SREs, inference reliability engineers, FinOps partners  
**Execution labels:** **[SAFE-READ]** observation; **[LAB-MUTATE]** isolated approved staging; **[CHANGE-CONTROL]** production changes.

## 1. Objectives and prerequisites

Build an end-to-end scaling observability model, correlate inference SLOs with HPA/KEDA and GPU node scaling, construct meaningful alerts, calculate GPU cost per useful token, forecast capacity and cost, diagnose failures through **Lowest Proven Healthy Layer (LPHL)** and **First Failed Transition (FFT)**, and produce an incident-ready runbook.

Prerequisites: Chapters 8.1–8.5, a pinned TrueFoundry model/image release, read-only Kubernetes and approved observability access, known TTFT/ITL and error SLOs, GPU quotas, a billing owner, and a rollback plan.

**Principle:** A desired replica count is not serving capacity. Follow every scale decision to a **Ready endpoint successfully completing requests within SLO**.

## 2. End-to-end observability architecture

```text
Clients → Gateway → Service → Ready vLLM Pods
   │        │          │           │
   │        │          │           ├─ queue, TTFT, ITL, tokens/s, KV
   │        │          │           └─ GPU utilization, VRAM, ECC/XID
   │        │          └─ Ready EndpointSlices
   │        └─ 429/5xx, routing, streaming disconnects
   └─ E2E latency, retries, SLO-compliant completions
                   │
          Metrics / logs / traces
                   │
         Prometheus / OpenTelemetry
                   │
           Grafana / alerting
                   │
HPA/KEDA → Deployment → Pending Pods → node autoscaler → GPU nodes
   │          │             │               │             │
 metrics   desired/ready   events        provision time   allocatable
                   │
         Cloud billing + GPU-hours
                   │
         Cost / useful tokens / SLO
```

TrueFoundry, Kubernetes, vLLM and cloud-provider integrations vary by deployed release. Discover actual metrics and supported telemetry endpoints; do not assume a specific dashboard or metric name is installed.

## 3. Four observability planes

| Plane | Key question | Signals |
|---|---|---|
| Client/SLO | Is the user getting useful results? | TTFT/ITL/E2E P95/P99, error rate, stream completion |
| Serving/runtime | Is the model saturated? | Running/waiting requests, queue age, KV/preemption, tokens/s |
| Control/capacity | Is scaling producing Ready capacity? | HPA/KEDA decisions, desired/available replicas, Pending reasons, node provision time |
| Financial | Is capacity cost effective and bounded? | GPU-hours, idle GPU cost, useful tokens, per-model/tenant attribution |

Correlate all planes by **environment, service, model revision, cluster, GPU SKU and time window**. Avoid high-cardinality labels such as prompt text, request IDs, user IDs or token strings in metrics.

## 4. Golden signals and service-level indicators

Define latency at a consistent measurement point:

- **TTFT:** accepted request to first streamed token, by prompt-length bucket.
- **ITL:** interval between streamed output tokens, by output-length/concurrency bucket.
- **E2E:** accepted request to completion.
- **Error rate:** 429, 5xx, timeouts and cancelled streams, with separate causes.
- **Goodput:** requests or tokens completed within the agreed SLO.
- **Saturation:** queue wait/age, KV-cache pressure, GPU/CPU/memory constraints.
- **Capacity:** desired, current, available and **Ready** serving replicas; Ready endpoints; allocatable GPUs.

SLO percentiles require adequate samples. Report counts and histogram windows. Distinguish client-side timeout from engine-side abort.

## 5. Dashboard layout

### Row A — user impact

| Panel | Visualization | Drill-down |
|---|---|---|
| TTFT P50/P95/P99 | Timeseries | Prompt length and model |
| ITL P50/P95/P99 | Timeseries | Output length/concurrency |
| E2E P95/P99 | Timeseries | Streaming vs non-streaming |
| 429/5xx/timeouts | Rate and count | Gateway, admission, runtime |
| Successful stream completion | Percentage | Disconnect/abort reason |
| SLO goodput | Tokens/s or requests/s | Per model/replica |

### Row B — serving and GPU

| Panel | Visualization | Drill-down |
|---|---|---|
| Waiting/running requests | Timeseries | Pod/model |
| Queue age | P95/P99 | Endpoint |
| KV-cache occupancy and preemption | Timeseries | Model revision |
| Input/output tokens/s | Timeseries | GPU SKU |
| GPU utilization/VRAM | Timeseries | Node/GPU |
| ECC/XID and GPU health | Events | Node/driver |

### Row C — scaling control

| Panel | Visualization | Drill-down |
|---|---|---|
| HPA desired/current | Timeseries | Conditions/metrics |
| KEDA trigger and readiness | Status | Trigger query/auth |
| Deployment desired/available | Timeseries | Rollout |
| Pending GPU Pods | Count | Scheduler reason |
| GPU allocatable/requests | Stacked capacity | Node pool |
| Node launch → GPU ready → model Ready | Durations | Provisioner/cloud |

### Row D — economics and risk

| Panel | Visualization | Drill-down |
|---|---|---|
| GPU-hours by model/environment | Cost | SKU/region |
| Cost per million useful output tokens | Trend | SLO-qualified only |
| Warm-idle cost | Trend | Reserved headroom |
| Forecast demand vs quota | Projection | Peak/failure domain |
| GPU node max/quota exhaustion | Capacity | Cloud/namespace |
| N-1 and rollout surge headroom | Gauge | Zone/node |

## 6. Prometheus metric contracts

Before writing queries, inventory actual exporter names, labels, scrape intervals, histogram buckets, units, reset behavior and cardinality. Metric names below are **schematic placeholders**, not guaranteed vLLM, TrueFoundry or NVIDIA metric names.

| Logical metric | Example placeholder | Unit |
|---|---|---|
| Completed inference requests | `inference_requests_total` | counter |
| TTFT histogram | `inference_ttft_seconds_bucket` | seconds |
| ITL histogram | `inference_itl_seconds_bucket` | seconds |
| Waiting requests | `inference_waiting_requests` | gauge |
| Output tokens | `inference_output_tokens_total` | counter |
| Ready replicas | `inference_ready_replicas` | gauge |
| GPU-hours | Derived from billing/uptime | GPU-hour |

### Illustrative PromQL patterns — replace metric names and labels

```promql
sum(rate(inference_requests_total{status="success"}[5m]))
```

```promql
histogram_quantile(
  0.95,
  sum by (le, model) (rate(inference_ttft_seconds_bucket[5m]))
)
```

```promql
sum(rate(inference_output_tokens_total[5m]))
```

```promql
sum(inference_waiting_requests)
```

The examples require valid instrumentation and labels. A rate over a counter must handle restarts; histogram buckets must share compatible boundaries. **Do not average P95 values across Pods**—aggregate histogram buckets first, then compute the quantile.

## 7. Scaling efficiency metrics

Track three separate delays:

```text
decision_latency = HPA/KEDA decision - demand threshold crossed
placement_latency = Pod scheduled - Pod created
usable_capacity_latency = first successful inference - demand threshold crossed
```

If HPA desired increases immediately but usable capacity takes eight minutes, the bottleneck is downstream of the scaling decision. Capture cloud node launch, GPU device-plugin registration, model artifact fetch, CUDA initialization and readiness separately.

Define **scaling effectiveness**:

```text
effective_capacity_gain =
  SLO-compliant output tokens/s after scale
  - SLO-compliant output tokens/s before scale
```

Compare identical workload mixes and stable windows. More GPUs without useful capacity gain is a performance/cost regression.

## 8. GPU FinOps and cost allocation

A first-order model:

```text
GPU_cost = Σ(GPU_instance_hour_rate × instance_hours)
total_serving_cost = GPU_cost + CPU/RAM + storage + network
                   + model artifacts + monitoring + platform charges

cost_per_million_useful_output_tokens =
  total_serving_cost / (SLO_compliant_output_tokens / 1,000,000)
```

Clearly label whether costs are **allocated** or **incremental**, and whether rates include reserved/spot discounts, cloud billing adjustments and shared infrastructure. Do not compare cost-per-token across radically different models, context lengths, quality requirements or SLOs without normalization.

### Worked example — hypothetical

| Variable | Assumption |
|---|---:|
| Four GPUs × 24 hours | 96 GPU-hours |
| GPU-hour effective rate | $2.00 |
| GPU-only cost | $192 |
| SLO-compliant output tokens | 16,000,000 |
| GPU-only cost per million useful tokens | **$12.00** |

This excludes non-GPU costs. If half the tokens breach SLO, the cost per million **useful** tokens doubles, assuming costs stay constant.

## 9. Capacity forecasting

Forecast by **workload shape**, not request count alone. Use daily/weekly seasonality, P95 burst demand, prompt/output token distributions, new model launches and regional failover. Estimate sustainable throughput at the SLO per GPU SKU and model revision.

```text
steady_replicas = ceil(peak_accepted_output_tokens_per_second
                       / sustainable_output_tokens_per_second_per_replica)
GPU_demand = steady_replicas × GPUs_per_replica
```

Add explicit N-1 largest failure-domain reserve, rolling surge, maintenance buffer and cold-start headroom. Check cloud GPU quota, node-pool limits, subnet IPs, CPU/RAM fit and artifact bandwidth. Model uncertainty bands and review forecast error monthly.

## 10. Alert design

| Alert | Suggested condition design | Severity guidance |
|---|---|---|
| User-facing SLO burn | Error/latency burn over short and long windows | Page if sustained impact |
| No Ready endpoints | Serving endpoint unavailable | Page |
| Desired ≫ Ready replicas | Sustained gap with traffic impact | Page or urgent |
| Pending GPU Pods | Sustained unschedulable GPU demand | Urgent |
| GPU quota near limit | Forecasted demand exceeds headroom | Warning |
| KEDA metric stale | Trigger freshness fails | Warning/urgent if affecting scale |
| Node provision failures | Repeated cloud quota/supply/IAM errors | Urgent |
| KV preemption/OOM | Sustained impact or restarts | Urgent |
| Cost anomaly | GPU-hours outside approved budget | FinOps notification |
| N-1 reserve insufficient | Failure-domain loss violates SLO | Urgent planning |

Do not page solely on high GPU utilization when TTFT/ITL and errors are healthy. Set thresholds from observed baselines and SLOs, with routing, deduplication, runbook links and maintenance suppression.

## 11. Read-only operational commands [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,hpa -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> get resourcequota,pdb
kubectl -n <namespace> describe hpa <hpa-name>
kubectl -n <namespace> describe pod <pod>
kubectl get nodes -o wide
kubectl describe node <gpu-node>
```

Only where installed and authorized:

```bash
kubectl -n <namespace> get scaledobjects.keda.sh
kubectl get nodepools.karpenter.sh
kubectl get nodeclaims.karpenter.sh
```

A missing CRD is **Not installed**, not proof of failure. Do not dump secrets, customer prompts or bearer tokens into diagnostics.

## 12. Scaling troubleshooting decision tree

```text
User SLO degraded?
 ├─ No: check cost, reserve and forecast; avoid reflexive scaling
 └─ Yes:
     ├─ Gateway 429/5xx? → quotas/admission/retries
     ├─ Queue rising? → inference demand vs Ready capacity
     │    ├─ Desired not rising → metric/KEDA/HPA ownership
     │    ├─ Desired rising, Pending → GPU scheduler/node quota
     │    └─ Pods Running, not Ready → image/model/CUDA/warmup
     ├─ TTFT high? → queue/prefill/cold start
     ├─ ITL high? → decode/KV/GPU/transport
     └─ New revision? → canary diff and rollback
```

Record **LPHL** and **FFT**. Use timestamp correlation rather than blaming the most visible metric.

## 13. Failure scenarios

| ID | Symptom | First failed transition | Evidence |
|---|---|---|---|
| O1 | Dashboard shows no inference metrics | Exporter → scrape | Target health |
| O2 | P95 misleading | Histogram → aggregation | Buckets/labels |
| O3 | High cardinality overwhelms Prometheus | Labels → TSDB | Series count |
| O4 | Clock skew breaks correlation | Timestamp → timeline | NTP/time |
| O5 | HPA metrics unknown | Adapter → HPA | HPA conditions |
| O6 | KEDA trigger stale | Event source → KEDA | Sample age |
| O7 | HPA/TrueFoundry replica conflict | Ownership → Deployment | Reconciliation |
| O8 | Desired grows, Ready does not | Deployment → capacity | Pod events |
| O9 | GPU Pods Pending | Scheduler → node | GPU allocatable |
| O10 | Cloud GPU quota exceeded | Provisioner → cloud | Quota errors |
| O11 | Model cold-start delay | Pod → readiness | Load timeline |
| O12 | Queue rises with normal GPU use | Admission → engine | Waiting requests |
| O13 | TTFT regresses on long prompts | Prefill → first token | Prompt buckets |
| O14 | ITL regresses under concurrency | Decode → stream | KV/preemption |
| O15 | GPU ECC/XID faults | Device → runtime | DCGM/driver |
| O16 | Scale-down drops streams | Drain → client | Disconnects |
| O17 | Autoscaling oscillates | Metrics → policy | HPA history |
| O18 | Cost rises, goodput flat | Scale → useful work | GPU-hours/tokens |
| O19 | Cost allocation missing | Billing → attribution | Tags/labels |
| O20 | Spot interruption exhausts reserve | Node loss → SLO | N-1 timeline |
| O21 | Rollout surge capacity absent | Deployment → GPU fit | Pending events |
| O22 | False alert during maintenance | Change → alerting | Change window |
| O23 | SLO healthy but GPU high | No failed transition | TTFT/ITL |
| O24 | Wrong model after scale | Artifact → inference | Immutable revision |

For each, capture impact, LPHL, FFT, evidence, mitigation, rollback and post-recovery verification. **O23 is intentionally a non-incident.**

## 14. Production incident runbook

1. **Declare impact:** model/endpoint, SLO, TTFT/ITL/E2E, errors, queue, stream completion and affected traffic.
2. **Build timeline:** demand → metrics → KEDA/HPA → Deployment → Pending Pods → GPU nodes → model Ready → client recovery.
3. **Localize FFT:** verify each transition; correlate revision, GPU health, cloud quota, scheduler and metric freshness.
4. **Contain:** preserve known-good warm replicas, bound admission/retries, pause unsafe rollouts or consolidation through approved controls.
5. **Recover:** repair the confirmed layer or restore immutable TrueFoundry/GitOps release; avoid uncontrolled manual scaling.
6. **Validate:** Ready endpoints, correct model, TTFT/ITL, 429/5xx, completed streams, N-1 reserve and cost ceiling.
7. **Review:** SLO burn, cost impact, time-to-usable-capacity, LPHL/FFT, alert gaps and corrective owners.

## 15. Acceptance gates

| Gate | Required evidence |
|---|---|
| A1 | Four-plane observability architecture documented |
| A2 | TTFT/ITL/E2E, goodput and error metrics validated |
| A3 | HPA/KEDA/Ready/GPU capacity correlation |
| A4 | Cost per million useful tokens calculated with exclusions |
| A5 | Forecast and N-1 reserve evaluated |
| A6 | Alerts/runbooks reviewed or tested in staging |
| A7 | O1–O24 classified with LPHL/FFT |
| A8 | Incident rollback and post-recovery SLO documented |

## 16. Evidence template

```text
Date/time / owner / approval:
TrueFoundry workspace / cluster / namespace:
Model/image/vLLM revision and GPU SKU:
Metrics/exporter/adapter and histogram contract:
TTFT/ITL/E2E P50/P95/P99, errors, stream completion:
Queue/KV/preemption, tokens/s, SLO goodput:
KEDA/HPA desired/current, Ready endpoints, Pending:
Node provision, GPU allocatable, cold-start timeline:
GPU-hours, rate assumptions, cost per useful million tokens:
Forecast, cloud quota, N-1 reserve and surge:
Alert evaluation, LPHL/FFT, O1–O24:
Rollback, recovery validation, unexecuted gates:
```

**Hands-on lab:** [8.6 — Scaling Observability, Cost and Troubleshooting Check](../labs/06-scaling-observability-cost-troubleshooting-check.md)  
**Next:** Chapter 8.7 — Production Scaling and Recovery Capstone.

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [KEDA Documentation](https://keda.sh/docs/)
- [Prometheus Histograms](https://prometheus.io/docs/practices/histograms/)
- [Grafana Documentation](https://grafana.com/docs/)
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
- [vLLM Documentation](https://docs.vllm.ai/)
