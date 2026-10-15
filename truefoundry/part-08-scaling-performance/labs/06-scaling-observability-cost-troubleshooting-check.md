# Lab 8.6 — Scaling Observability, Cost and Troubleshooting Check

**Canonical:** [Chapter 8.6](../06-scaling-observability-cost-troubleshooting/06-scaling-observability-cost-troubleshooting.md)  
**Default:** [SAFE-READ]. Alert deployment or load generation requires [LAB-MUTATE] in isolated approved staging.

## 1. Mission and guardrails

Produce an SRE-ready scaling dashboard, validate inference SLO and control-loop metrics, calculate GPU cost per useful million output tokens, forecast N-1 capacity, classify 24 failures and document rollback. Do not install observability agents, change production alert routing or run unbounded load without approval.

## 2. Preflight

```text
Date/time, owner and approval:
Workspace/cluster/namespace:
Immutable model/image/vLLM/GPU SKU:
Read-only metrics and billing access:
TTFT/ITL/E2E/error SLO:
GPU quota, max nodes and cost ceiling:
Known-good release and rollback owner:
```

## 3. Read-only inventory

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,hpa -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> get resourcequota,pdb
kubectl get nodes -o wide
```

Where installed and authorized, inspect ScaledObjects/Karpenter NodePools; missing CRDs mean Not installed.

## 4. Metric contract and dashboard worksheet

| Plane | Metric | Actual source/units | Freshness | Panel |
|---|---|---|---|---|
| Client | TTFT P95/P99 | | | |
| Client | ITL P95/P99 | | | |
| Client | E2E P95/P99 | | | |
| Client | 429/5xx/stream completion | | | |
| Serving | Waiting/running queue | | | |
| Serving | KV/preemption | | | |
| Serving | Input/output tokens/s | | | |
| GPU | Utilization/VRAM/ECC | | | |
| Control | HPA/KEDA desired | | | |
| Control | Deployment Ready/EndpointSlices | | | |
| Capacity | GPU Pending/allocatable | | | |
| Financial | GPU-hours and useful tokens | | | |

Validate actual metric names, histogram buckets, sample counts, low-cardinality labels and counter resets. Do not average per-Pod P95s.

## 5. Scaling timeline

| Transition | Time | Evidence | Delay |
|---|---|---|---|
| Demand threshold crossed | | | |
| Metric observed | | | |
| KEDA/HPA decision | | | |
| Desired replicas increased | | | |
| Pod Pending | | | |
| GPU node provisioned | | | |
| GPU allocatable | | | |
| Model Ready | | | |
| First successful inference | | | |
| TTFT/ITL SLO recovered | | | |

Mark unavailable measurements Not run. Do not invent timestamps.

## 6. Cost worksheet

```text
GPU_cost = GPU_hours × effective_GPU_hour_rate
total_cost = GPU_cost + allocated_CPU/storage/network/monitoring/platform
cost_per_million_useful_output_tokens =
    total_cost ÷ (SLO_compliant_output_tokens / 1,000,000)
```

| Input | Amount | Source/assumption |
|---|---:|---|
| GPU-hours | | |
| Effective GPU-hour rate | | |
| GPU-only cost | | |
| Other allocated costs | | |
| Total serving cost | | |
| Completed output tokens | | |
| SLO-compliant output tokens | | |
| Cost per million useful tokens | | |
| Warm idle GPU-hours | | |

Hypothetical check: 96 GPU-hours × $2 = $192; 16 million useful tokens → $12 GPU-only per million. Verify exclusions and discounts.

## 7. Capacity forecast

| Input | Baseline | Peak/failover |
|---|---|---|
| Output token demand/s | | |
| Sustainable tokens/s/replica at SLO | | |
| GPUs/Pod | | |
| Steady replicas | | |
| Largest-domain N-1 loss | | |
| Rollout surge | | |
| Node/GPU quota | | |
| Subnet IP and CPU/RAM fit | | |
| Cold-start to Ready | | |
| Forecast confidence/risk | | |

## 8. Alert review

| Alert | SLO/threshold basis | Severity | Owner/runbook | Tested? |
|---|---|---|---|---|
| SLO burn | | | | |
| No Ready endpoints | | | | |
| Desired/Ready gap | | | | |
| Pending GPU Pods | | | | |
| Cloud quota exhaustion | | | | |
| KEDA metric stale | | | | |
| KV preemption/OOM | | | | |
| GPU ECC/XID | | | | |
| Cost anomaly | | | | |
| N-1 reserve shortage | | | | |

Avoid alert fatigue from healthy high GPU utilization.

## 9. Optional staging validation [LAB-MUTATE]

1. Confirm approved isolated deployment, synthetic inputs, quota, test duration and rollback.
2. Validate dashboard metric sources and panel queries.
3. Generate bounded synthetic demand and observe scaling timeline to Ready endpoint.
4. Compare SLO goodput, queue, GPU-hours and cost before/after scaling.
5. Test alert evaluation in a nonpaging staging route if approved.
6. Stop on sustained SLO breach, errors, GPU Pending, OOM or budget ceiling.
7. Restore known-good configuration and verify stream completion, model revision and cleanup.

If unavailable, mark Not run and complete a tabletop.

## 10. Failure tabletop

| ID | Scenario | LPHL | FFT | Evidence | Safe response |
|---|---|---|---|---|---|
| O1 | Metrics absent | | | | |
| O2 | Incorrect P95 aggregation | | | | |
| O3 | High metric cardinality | | | | |
| O4 | Clock skew | | | | |
| O5 | HPA metric unknown | | | | |
| O6 | KEDA stale trigger | | | | |
| O7 | HPA/TrueFoundry conflict | | | | |
| O8 | Desired vs Ready gap | | | | |
| O9 | GPU Pending | | | | |
| O10 | Cloud quota exceeded | | | | |
| O11 | Model cold start | | | | |
| O12 | Queue rising | | | | |
| O13 | Prefill TTFT | | | | |
| O14 | Decode ITL | | | | |
| O15 | GPU ECC/XID | | | | |
| O16 | Scale-down stream loss | | | | |
| O17 | Autoscaler oscillation | | | | |
| O18 | Cost without goodput | | | | |
| O19 | Missing cost attribution | | | | |
| O20 | Spot interruption | | | | |
| O21 | Surge shortage | | | | |
| O22 | False maintenance alert | | | | |
| O23 | Healthy SLO/high GPU | | | | |
| O24 | Wrong model revision | | | | |

LPHL = Lowest Proven Healthy Layer; FFT = First Failed Transition. O23 is not a failure if SLOs are met.

## 11. Acceptance gates

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| A1 Four-plane dashboard | | |
| A2 SLO metrics and goodput validated | | |
| A3 Control loop/Ready/GPU correlation | | |
| A4 Cost per useful million tokens | | |
| A5 Capacity forecast and N-1 | | |
| A6 Alert/runbook review or staging test | | |
| A7 O1–O24 classification | | |
| A8 Rollback and post-recovery SLO | | |

## 12. Final report

```text
Date/time / approval / owner:
Model/image/vLLM and GPU SKU:
Metric sources, histogram validity and dashboard links:
TTFT/ITL/E2E percentiles, errors, stream completion:
Queue/KV/tokens/s and SLO goodput:
HPA/KEDA/Ready/Pending and GPU allocatable:
Node provision/model cold-start timeline:
GPU-hours, total costs, cost per useful million:
Forecast, N-1 reserve, surge and quota:
Alerts, O1–O24 LPHL/FFT:
Rollback, cleanup, A1–A8 and unexecuted gates:
```

**Next:** Chapter 8.7 — Production Scaling and Recovery Capstone.
