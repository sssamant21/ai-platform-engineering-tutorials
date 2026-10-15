# Lab 8.7 — Production Scaling and Recovery Acceptance Capstone

**Canonical:** [Chapter 8.7](../07-production-scaling-recovery-capstone/07-production-scaling-recovery-capstone.md)  
**Default:** [SAFE-READ] and tabletop. Live exercises require [LAB-MUTATE] approval in isolated staging; production changes require [CHANGE-CONTROL].

## 1. Mission and safety

Complete a single integrated operational exercise covering scaling, GPU scheduling, model warmup, TTFT/ITL, goodput, FinOps, failure recovery and rollback. The lab is **not complete** until evidence is reviewed. Never run customer prompts, stress production, force shared GPU node failures or alter cloud quotas without separate approval.

## 2. Preflight and rollback record

```text
Capstone ID/date/owner:
TrueFoundry workspace, cluster, namespace:
Approval and staging isolation:
Immutable model/tokenizer/image and vLLM release:
HPA/KEDA/TrueFoundry ownership and versions:
GPU SKU/driver/plugin, node pool, cloud quota:
TTFT/ITL/E2E/429/5xx/stream-completion SLO:
Synthetic prompt/output distributions:
Hard request/token/time/GPU-cost ceilings:
Stop conditions and authoritative rollback release:
```

## 3. Read-only inventory

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,hpa -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get resourcequota,pdb
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl get nodes -o wide
kubectl describe node <gpu-node>
kubectl -n <namespace> describe pod <inference-pod>
```

Optional only where installed:

```bash
kubectl -n <namespace> get scaledobjects.keda.sh
kubectl get nodepools.karpenter.sh
kubectl get nodeclaims.karpenter.sh
```

Record missing CRDs as Not installed; do not install cluster-wide operators for this exercise.

## 4. Ownership and readiness checklist

| Check | Expected | Observed | Evidence |
|---|---|---|---|
| TrueFoundry/GitOps source of truth | | | |
| HPA/KEDA no competing target | | | |
| GPU node autoscaler and version | | | |
| Device plugin / GPU allocatable | | | |
| Pod affinity/taints/topology | | | |
| Model-aware startup/readiness | | | |
| Stream drain and PDB | | | |
| Immutable rollback revision | | | |

## 5. Capacity and reserve worksheet

```text
serving_replicas = ceil(peak output tokens/s
                        / sustainable output tokens/s per replica at SLO)
steady_GPUs = serving_replicas × GPUs_per_Pod
planned_GPUs = steady_GPUs + largest_domain_reserve + rollout_surge
```

| Input | Baseline | Peak/failover | Evidence |
|---|---|---|---|
| Accepted token demand/s | | | |
| Sustainable throughput/replica | | | |
| GPU/Pod and TP topology | | | |
| Serving replicas | | | |
| Largest node/zone loss | | | |
| N-1 reserve | | | |
| Rollout surge | | | |
| Cloud quota/node max | | | |
| CPU/RAM/subnet fit | | | |
| Model cold-start budget | | | |

## 6. Benchmark worksheet

| Test | Workload | TTFT P95 | ITL P95 | E2E P95 | Errors | SLO goodput |
|---|---|---|---|---|---|---|
| B1 | Short/short, concurrency 1 | | | | | |
| B2 | Short/long, concurrency 8 | | | | | |
| B3 | Long/short, concurrency 8 | | | | | |
| B4 | Long/long, concurrency 16 | | | | | |
| B5 | Long context, concurrency 4 | | | | | |
| B6 | Representative mix, ramp | | | | | |
| B7 | Shared-prefix mix | | | | | |
| B8 | Interactive + batch | | | | | |

All workload values in the chapter are illustrative; reduce to approved model/context and GPU limits. Record request count, arrival rate, token lengths, warm/cold and repeated-run variance.

## 7. T0–T10 scaling timeline

| Milestone | Timestamp | Evidence |
|---|---|---|
| T0 Demand threshold crossed | | |
| T1 Metric updated | | |
| T2 HPA/KEDA desired decision | | |
| T3 Deployment Pod created | | |
| T4 GPU scheduling/Pending | | |
| T5 GPU node launch | | |
| T6 GPU allocatable | | |
| T7 Model/image load | | |
| T8 CUDA warmup | | |
| T9 Ready EndpointSlice | | |
| T10 Client SLO recovery | | |

Compute **T10−T0** from measured times only. Mark Not run if no live scale-up is approved.

## 8. Optional controlled staging run [LAB-MUTATE]

1. Confirm approval, isolated lab-owned resources, max GPU-hours and rollback.
2. Record SLO, metrics, Ready endpoints and baseline GPU utilization.
3. Generate bounded synthetic B1–B8 traffic within request/token/time limits.
4. Observe HPA/KEDA decision and GPU node provisioning, including T0–T10.
5. Compare pre/post-scale TTFT/ITL, queue, errors, stream completion and SLO goodput.
6. Stop for prolonged Pending, OOM, rising 429/5xx, SLO breach or budget ceiling.
7. Perform only separately approved disposable failure drills; otherwise tabletop.
8. Restore known-good configuration via authoritative TrueFoundry/GitOps workflow.
9. Verify correct model revision, stream drain, Ready endpoints, GPU cleanup and baseline costs.

## 9. Failure and recovery tabletop

| ID | Scenario | LPHL | FFT | Mitigation | Recovery evidence |
|---|---|---|---|---|---|
| C1 | HPA metric missing | | | | |
| C2 | KEDA stale/auth denied | | | | |
| C3 | Replica controller conflict | | | | |
| C4 | GPU Pending | | | | |
| C5 | GPU quota exhausted | | | | |
| C6 | GPU instance unavailable | | | | |
| C7 | Device plugin missing | | | | |
| C8 | CUDA/vLLM init failure | | | | |
| C9 | Slow model cold start | | | | |
| C10 | Prefill TTFT regression | | | | |
| C11 | Decode ITL regression | | | | |
| C12 | KV preemption/OOM | | | | |
| C13 | Gateway retry storm | | | | |
| C14 | Scale-down stream drops | | | | |
| C15 | GPU node failure | | | | |
| C16 | Spot interruption | | | | |
| C17 | Surge capacity shortage | | | | |
| C18 | Cost without goodput | | | | |
| C19 | Metrics gap | | | | |
| C20 | Healthy GPU false alert | | | | |
| C21 | Wrong model revision | | | | |
| C22 | GPU affinity/taint mismatch | | | | |
| C23 | Artifact/PVC bottleneck | | | | |
| C24 | Orphan GPU nodes after rollback | | | | |

LPHL = Lowest Proven Healthy Layer; FFT = First Failed Transition. Tabletop classifications are not evidence of live recovery.

## 10. FinOps worksheet

```text
GPU_cost = GPU_hours × effective_GPU_hour_rate
total_cost = GPU_cost + allocated_other_costs
cost_per_million_useful_tokens =
  total_cost / (SLO_compliant_output_tokens / 1_000_000)
```

| Input | Value | Evidence/assumption |
|---|---|---|
| GPU-hours | | |
| Effective GPU-hour rate | | |
| GPU-only cost | | |
| CPU/storage/network/platform | | |
| Total serving cost | | |
| Completed output tokens | | |
| SLO-compliant output tokens | | |
| Cost per million useful tokens | | |
| Warm-idle GPU-hours | | |
| Forecast and approved budget | | |

Hypothetical calculation check: 96 GPU-hours × $2 = $192; 16 million useful tokens → $12 GPU-only per million.

## 11. Acceptance gates

| Gate | Pass / Fail / Not run | Evidence and owner |
|---|---|---|
| A1 Workload/SLO contract | | |
| A2 Immutable release and controller ownership | | |
| A3 GPU capacity, N-1 and surge | | |
| A4 Metric contract and T0–T10 timeline | | |
| A5 B1–B8 benchmark/goodput | | |
| A6 C1–C24 recovery classification | | |
| A7 FinOps and forecast | | |
| A8 Drain, rollback and model correctness | | |
| A9 Alerts, runbook and escalation | | |
| A10 Vendor/platform/SRE signoff | | |

**Go/no-go:** A critical Fail blocks promotion. Not run does not equal Pass. Any waiver must name an owner, risk, mitigation and expiration.

## 12. Final report

```text
Capstone ID/date/approvals:
Workspace/cluster/namespace:
Model/image/vLLM/TrueFoundry and GPU revision:
Autoscaler ownership, metric source and trigger:
Capacity, largest-domain N-1, surge, quota:
B1–B8 benchmark sample counts and SLO results:
T0–T10 timestamps and time-to-usable-capacity:
TTFT/ITL/E2E, errors, queue, KV and stream completion:
C1–C24 LPHL/FFT, recovery/rollback:
GPU-hours, cost per useful million, forecast:
A1–A10 results, risks, owner and signoff:
Unexecuted gates and follow-up actions:
```

**Part 08 publication:** Seven chapters and seven labs complete; formal review and live lab acceptance remain pending.
