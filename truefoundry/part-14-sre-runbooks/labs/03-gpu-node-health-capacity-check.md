# Chapter 14.3 — Hands-on Lab: GPU Node Health and Capacity

**Part 14 — SRE Runbooks | Approved staging | Live execution pending**

## Objective and guardrails

Practice GPU node inventory, NVIDIA device/driver checks, Kubernetes scheduling, cloud capacity, N-1 headroom, bounded vLLM performance and safe recovery. **[SAFE-READ]** approved telemetry; **[LAB-MUTATE]** disposable staging GPU pool with spend cap; **[CHANGE-CONTROL]** production driver changes, node drains, GPU reset, MIG reconfiguration or cloud quotas.

## Lab 0 — Baseline

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -50
kubectl get resourcequota -A
```

Record GPU SKU/count/MIG, node pool, AZ, driver, device plugin, vLLM version, model digest, replica GPU requests, autoscaler bounds, provider quota, synthetic TTFT/TPOT and cost budget.

## Lab 1 — Hardware, driver and telemetry

On approved GPU-enabled diagnostic environment:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,driver_version,memory.total,memory.used,utilization.gpu --format=csv
```

Cross-check node allocatable, DCGM exporter freshness, XID/ECC, temperature, memory and model serving SLO. Do not interpret missing `nvidia-smi` in an image as proof of hardware failure.

## Lab 2 — Scheduling and provisioning

Inspect Pending Pod Events, GPU extended resources, quota, taints, affinity, topology, MIG profile, autoscaler provisioning and cloud zone quota. Determine whether spare GPU capacity is **schedulable** for the actual model topology.

## Lab 3 — N-1 capacity and cost

Calculate per-node allocatable GPU slots, requested GPU slots, required model replica placement and remaining capacity after one-node loss. Compare synthetic peak workload to TTFT/TPOT and stream completion SLO. Estimate warm spare GPU-hours and cost per successfully served 1,000 output tokens. Do not assume aggregate free GPU count equals usable capacity.

## Lab 4 — Six game days

1. GPU allocatable absent / device plugin failure (tabletop if shared).
2. Pending Pod from quota/taint/affinity.
3. Autoscaler/cloud GPU shortage.
4. XID/ECC or thermal health alert (tabletop).
5. Host OOM vs CUDA OOM vs KV cache classification.
6. One-node loss with safe drain plan, N-1 capacity and rollback (tabletop if unsafe).

For each capture UTC timeline, expected/actual, blast radius, synthetic SLO, budget, mitigation, rollback, executed/tabletop status and reviewer.

## Forty failure scenarios

01 NotReady; 02 kubelet; 03 GPU missing; 04 driver; 05 plugin; 06 allocatable zero; 07 MIG; 08 time-slicing; 09 taint; 10 affinity; 11 topology; 12 quota; 13 GPU slots; 14 cloud quota; 15 autoscaler max; 16 IAM; 17 XID; 18 ECC; 19 thermal; 20 power; 21 host OOM; 22 CUDA OOM; 23 KV cache; 24 fragmentation; 25 TTFT; 26 TPOT; 27 queue; 28 cold start; 29 healthy high utilization; 30 low utilization slow; 31 PDB; 32 drain stream; 33 replacement; 34 zone shortage; 35 spot; 36 subnet IP; 37 N-1; 38 AZ loss; 39 DCGM stale; 40 cost/token.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Approved scope and spend cap | Pending |
| G2 | GPU/node inventory | Pending |
| G3 | Driver/device plugin/health | Pending |
| G4 | Schedulable GPU slots verified | Pending |
| G5 | Autoscaler, cloud quota and limits | Pending |
| G6 | XID/ECC/thermal/DCGM | Pending |
| G7 | Host vs CUDA OOM diagnosis | Pending |
| G8 | TTFT/TPOT and stream SLO | Pending |
| G9 | N-1/topology capacity | Pending |
| G10 | Safe drain and rollback plan | Pending |
| G11 | Synthetic recovery and cost | Pending |
| G12 | Reviewer sign-off | Pending |

## Evidence record

```text
Exercise ID / change approval / UTC:
Cloud/cluster/zone/GPU SKU:
Driver, device plugin, MIG, vLLM, model digest:
Node allocatable vs Pod requests:
DCGM, XID/ECC, temperature, memory:
Autoscaler, quota, instance availability:
N-1 capacity and replica placement:
Synthetic workload, TTFT/TPOT, completion:
Expected/actual, mitigation, rollback:
GPU-hours, cost/token:
Executed/tabletop, owner, reviewer:
```

## Cleanup

Remove only disposable staging workloads, restore autoscaler settings, validate GPU capacity and inference SLO, reconcile budget and close test alerts. Publication does not imply execution or acceptance.

**Next:** Chapter 14.4 — Model Deployment and Rollback Runbook.
