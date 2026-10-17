# Chapter 053 — Hands-On Lab: Multi-Zone GPU Capacity and Disruption Budgets

**Part 07 | Chapter 053 | Published lab plan; live execution pending**

## Goal

Inventory GPU capacity by zone, calculate N-1 headroom, inspect topology spread and PDB constraints, design stream-safe draining and zone failover, and document RTO, rollback and FinOps.

## Safety

**[SAFE-READ]** authorized Kubernetes/cloud inventory. **[LAB-MUTATE]** approved isolated staging with bounded GPU-hours. **[CHANGE-CONTROL]** production node drain, PDB, topology, GPU node pools, cloud quota, zone fault injection and autoscaler changes. No PHI, customer prompts, Secrets or unapproved disruptive operations.

## Lab 0 — Preflight

Record UTC, context, provider/region/AZs, GPU SKU, model digests, replicas, per-zone allocatable GPUs, active streams, SLO/RTO, approval, cost ceiling and rollback.

## Lab 1 — Zone and GPU inventory [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get nodes --show-labels
kubectl get pods -A -o wide
kubectl get pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl describe node APPROVED_GPU_NODE
```

Capture zone distribution, node taints, GPU allocatable/allocated and Pod placement.

## Lab 2 — PDB inspection [SAFE-READ]

```bash
kubectl -n APPROVED_NS get pdb
kubectl -n APPROVED_NS describe pdb APPROVED_PDB
kubectl -n APPROVED_NS get pods -o wide
```

Record selector, expected/healthy Pods, `DisruptionsAllowed` and readiness. Do not assume PDB protects involuntary failures.

## Lab 3 — N-1 capacity calculation [CALCULATION]

```text
Zones: A=4 GPUs, B=4 GPUs, C=4 GPUs
Peak GPU-equivalent demand D=7
Maintenance reserve R=1
Survivor capacity = 12 - max(4,4,4) = 8
Required capacity = D + R = 8
Result: nominally sufficient, only if placement/model readiness works.
```

Normalize MIG/time-sliced shares to actual model-serving throughput, not physical GPU count.

## Lab 4 — Topology/PDB design [DESIGN]

```yaml
# DESIGN FRAGMENT — verify labels and capacity
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: llm-inference
```

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: llm-inference
  namespace: approved-staging
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: llm-inference
```

Do not apply until staging owners approve replica counts, zone availability and selectors.

## Lab 5 — Stream-safe drain and failover [TABLETOP]

Define gateway routing removal, active stream tracking, readiness, grace period, replacement GPU capacity, model download, CUDA warmup, TTFT/TPOT recovery and RTO evidence. No live node drain during tabletop.

## Lab 6 — Eight game days

01 AZ outage; 02 backup SKU shortage; 03 topology deadlock; 04 PDB drain block; 05 stream interruption; 06 artifact outage; 07 consolidation churn; 08 incomplete rollback.

## Twenty-four failure checklist

F01 single AZ; F02 SKU; F03 quota; F04 N-1; F05 MIG accounting; F06 spread; F07 affinity; F08 anti-affinity; F09 taint; F10 PDB selector; F11 PDB min; F12 DisruptionsAllowed; F13 involuntary; F14 drain; F15 grace; F16 provisioning; F17 plugin; F18 model cache; F19 KMS/storage; F20 consolidation; F21 spot; F22 TTFT; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions | Pending |
| G02 | Per-zone GPU inventory | Pending |
| G03 | N-1 capacity | Pending |
| G04 | Cloud quota | Pending |
| G05 | GPU SKU compatibility | Pending |
| G06 | Topology spread | Pending |
| G07 | Affinity | Pending |
| G08 | Taints | Pending |
| G09 | PDB selection | Pending |
| G10 | DisruptionsAllowed | Pending |
| G11 | Stream-safe drain | Pending |
| G12 | Termination grace | Pending |
| G13 | Autoscaler capacity | Pending |
| G14 | Driver/plugin | Pending |
| G15 | Artifact/model warmup | Pending |
| G16 | TTFT/TPOT SLO | Pending |
| G17 | Zone failover RTO | Pending |
| G18 | Rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/approval:
Provider/region/AZs:
GPU SKU/allocatable by AZ:
Peak demand/reserve/N-1 math:
Pod placement/labels/taints:
Topology spread/affinity:
PDB selector/healthy/DisruptionsAllowed:
Gateway stream drain/grace:
Cloud quota/autoscaler:
Model artifact/warmup:
TTFT/TPOT/RTO:
GPU-hours/cost:
Eight game-day findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Approved staging work must restore GitOps PDB/topology/node-pool configuration, remove only owned synthetic workloads, verify all zones and model readiness, reconcile GPU-hours and retain sanitized evidence. All gates remain pending until independent review.

**Next: Chapter 054 — GPU Scheduling, Quota and Pending Pod Troubleshooting.**
