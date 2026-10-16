# Chapter 12.2 — Hands-on Lab: GPU Capacity Planning, Quotas and Production Scaling

**Part 12 | Approved staging | Live execution pending**

## Goal and safety

Build a versioned GPU capacity plan, prove quota and scheduling behavior, benchmark synthetic inference, validate scaling delays and estimate GPU costs. [SAFE-READ] inventory; [LAB-MUTATE] bounded staging; [CHANGE-CONTROL] cloud GPU pools, quotas and production scaling. Do not generate unapproved GPU costs or impact shared tenant workloads.

## Lab 0 — Inventory [SAFE-READ]

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe nodes
kubectl get pods -A -o wide
kubectl get resourcequota,limitrange -A
kubectl get hpa -A
kubectl get events -A --sort-by=.lastTimestamp | tail -50
```

Record cluster/region, GPU type/MIG profile, allocatable/allocated GPUs, driver/device plugin, TrueFoundry/vLLM version, model/image digest, node-pool limits, cloud quota, approved benchmark and budget.

## Lab 1 — Workload characterization

Measure synthetic steady/peak RPS, input/output token distributions, TTFT/TPOT, stream duration, output tokens/sec, running/waiting requests, KV cache and GPU memory. Record sample size and hardware/configuration.

## Lab 2 — Replica and reserve calculation

Given synthetic demand 1,200 output tokens/sec, validated 300 tokens/sec per replica and 25% headroom, calculate six replicas. Add separate largest-failure-domain reserve and verify feasible GPU placement, node RAM/CPU and warmup time. Compare estimates with measured queue/latency behavior.

## Lab 3 — Quota and placement tabletop

Review staging ResourceQuota and GPU requests/limits. If authorized, create a disposable namespace/quota and Pod, verify admission and scheduling, then remove. Otherwise inspect existing quota/events. Document cloud GPU quota separately from Kubernetes quota.

## Lab 4 — Autoscaling rehearsal

Use a bounded synthetic load ramp; observe HPA/KEDA (if installed), pending Pods, Cluster Autoscaler/Karpenter, node launch, image pull, model warmup and first successful request. Stop on budget, SLO or shared-resource impact. Record cold vs warm scale-out and stabilization.

## Lab 5 — Failure and cost tabletop

Rehearse GPU node unavailability, zone capacity shortfall, cloud quota exhaustion and high idle GPU spend without disrupting production. Produce capacity/cost estimate, escalation owner and rollback plan.

## Lab 6 — Cleanup

Remove disposable workloads, restore autoscaler/quotas if changed, confirm no orphan GPU nodes or unexpected charges, and capture sanitized UTC evidence and reviewer approval.

## Thirty-two failure scenarios

| # | Scenario | Focus |
|---|---|---|
| 01 | GPU request missing | Resources |
| 02 | GPU resource name wrong | Extended resource |
| 03 | Namespace GPU quota exceeded | ResourceQuota |
| 04 | Cloud account GPU quota exhausted | Provider |
| 05 | Region GPU SKU unavailable | Supply |
| 06 | GPU Pod Pending | Scheduling |
| 07 | Taint/toleration mismatch | Placement |
| 08 | Node affinity impossible | Constraints |
| 09 | CPU/memory blocks GPU Pod | Node fit |
| 10 | MIG profile mismatch | Device topology |
| 11 | NVIDIA device plugin unhealthy | GPU allocatable |
| 12 | CUDA driver/runtime mismatch | Initialization |
| 13 | Model weights exceed memory | Fit |
| 14 | KV cache pressure | Runtime |
| 15 | CUDA OOM | Device memory |
| 16 | Pod OOMKilled | Container memory |
| 17 | GPU utilization high but throughput low | Bottleneck |
| 18 | GPU utilization low and queue high | CPU/network |
| 19 | Input token mix changes | Prefill |
| 20 | TTFT p95 regression | SLO |
| 21 | TPOT regression | Decode |
| 22 | Autoscaler metric stale | HPA/KEDA |
| 23 | HPA max replicas too low | Policy |
| 24 | Scale-out node provisioning slow | Autoscaler |
| 25 | Image pull slow | Startup |
| 26 | Model cold start exceeds SLO | Warm pool |
| 27 | Scale-in drops active streams | Drain |
| 28 | Autoscaler oscillation | Stabilization |
| 29 | Zone failure exhausts capacity | HA |
| 30 | PDB blocks node drain | Maintenance |
| 31 | GPU cost over budget | FinOps |
| 32 | Telemetry missing reports false headroom | Monitoring |

For each scenario record expected/actual, UTC timestamp, safe logs/metrics/events, root-cause hypothesis, mitigation, rollback and reviewer. Unsafe failures are tabletop.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Scope, GPU budget and change approval | Pending |
| G2 | GPU/node/region/driver inventory verified | Pending |
| G3 | Model fit and KV cache baseline measured | Pending |
| G4 | Synthetic workload distribution documented | Pending |
| G5 | SLO-constrained per-replica capacity measured | Pending |
| G6 | Replica/headroom and failure reserve calculated | Pending |
| G7 | Kubernetes and cloud quota validated | Pending |
| G8 | GPU placement and topology verified | Pending |
| G9 | Autoscaler and model warmup timing measured | Pending |
| G10 | Dashboards, alerts and no-data checks verified | Pending |
| G11 | Recovery, rollback, cleanup and cost checked | Pending |
| G12 | SRE/platform/FinOps sign-off | Pending |

Pass requires executed evidence and reviewer approval; tabletop does not count as executed recovery.

**Next:** Chapter 12.3 — High Availability, Resilience and Failure Domains.
