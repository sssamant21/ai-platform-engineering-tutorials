# Chapter 055 — Hands-On Lab: GPU Autoscaling Acceptance

**Part 07 | Chapter 055 | Capstone acceptance plan; live execution pending**

## Goal

Execute or tabletop a complete GPU inference autoscaling acceptance cycle across scheduling, quotas, sharing, HPA/KEDA, node provisioning, warmup, disruption, zone resilience, SLO and GPU FinOps. **This file does not certify a cluster.**

## Guardrails

**[SAFE-READ]** authorized cluster inventory. **[LAB-MUTATE]** approved isolated staging, synthetic prompts, bounded GPU-hours and rollback. **[CHANGE-CONTROL]** production autoscalers, GPU node pools, quota, MIG, taints, PDB, zone failures, IAM and model routing. Never use real customer data or expose secrets.

## Lab 0 — Preflight and evidence contract

Record UTC, context, cloud/region/AZs, Kubernetes/driver/plugin, GPU SKU/MIG, autoscaler ownership, namespaces, model digest, SLO, GPU quota, cost ceiling, approvers and rollback. For every gate record expected, observed, evidence URL, status, reviewer and timestamp.

## Lab 1 — Baseline [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get nodes --show-labels
kubectl get pods -A -o wide
kubectl get resourcequota,limitrange -A
kubectl get hpa -A
kubectl get pdb -A
kubectl get networkpolicy -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get crd | grep -Ei 'scaledobject|nodepool|nodeclaim|karpenter'
```

Document actual GPU resource names, node readiness, allocatable/allocated GPUs and cluster controller versions.

## Lab 2 — Tenant isolation [SAFE-READ / DESIGN]

Inspect approved namespace quota, RBAC, Pod Security, NetworkPolicy, workload identity, model artifact access and token/concurrency limits. Record the GPU sharing strategy. Do not alter security controls without approval.

## Lab 3 — Scheduling and GPU readiness [SAFE-READ]

```bash
kubectl describe node APPROVED_GPU_NODE
kubectl -n APPROVED_NS describe pod APPROVED_GPU_POD
kubectl -n APPROVED_NS get events --sort-by=.lastTimestamp
```

Distinguish admission rejection, Pending scheduler constraints, device-plugin registration and model warmup.

## Lab 4 — HPA/KEDA metrics [SAFE-READ / DESIGN]

Record queue depth, input/output token work, active sequences, KV-cache pressure, TTFT/TPOT and terminal-valid completions. Verify adapter/KEDA existence and one autoscaler owner per target. Do not treat CPU/GPU utilization as sufficient alone.

## Lab 5 — GPU warmup critical path [CALCULATION]

```text
T_ready ≈ scaling decision + GPU node launch
        + node/driver/plugin ready + image pull
        + model download + CUDA warmup + readiness
```

Measure each component from actual staging evidence when approved; otherwise mark not executed. Compare p95 warmup to TTFT objective and minimum warm pool.

## Lab 6 — N-1 zone resilience [CALCULATION]

```text
surviving GPU capacity =
 sum(usable zone capacity) - max(zone capacity)
required = peak GPU demand + maintenance reserve
```

Normalize MIG/time-sliced capacity to model-serving throughput. Verify PDB, topology, stream-safe drain and artifact availability.

## Lab 7 — Synthetic load plan [APPROVAL REQUIRED]

Define short/long prompts, burst and steady phases, RPS/concurrency/token ceilings, expected completions, p95/p99 latency, GPU-hour budget, abort thresholds, cleanup and evidence capture. Do not execute unapproved load.

## Lab 8 — Eight game days [TABLETOP DEFAULT]

01 device plugin loss; 02 quota denial; 03 affinity deadlock; 04 stale metrics; 05 cloud quota; 06 model warmup; 07 stream interruption; 08 zone loss/rollback drift.

## Twenty-four failure checklist

F01 plugin; F02 allocatable; F03 driver; F04 MIG; F05 sharing; F06 GPU quota; F07 CPU/memory; F08 security; F09 taint; F10 affinity; F11 topology; F12 PVC; F13 adapter; F14 KEDA stale; F15 HPA ownership; F16 cloud quota; F17 provisioner; F18 plugin late; F19 model download; F20 warmup; F21 stream; F22 N-1; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions | Pending |
| G02 | Driver/plugin | Pending |
| G03 | Allocatable/allocated | Pending |
| G04 | MIG/sharing | Pending |
| G05 | ResourceQuota | Pending |
| G06 | RBAC/Pod Security/identity | Pending |
| G07 | Taints/affinity | Pending |
| G08 | Topology/PVC | Pending |
| G09 | HPA/KEDA ownership | Pending |
| G10 | Metric correctness | Pending |
| G11 | Node autoscaler | Pending |
| G12 | GPU quota/zone capacity | Pending |
| G13 | Image/model/CUDA warmup | Pending |
| G14 | TTFT/TPOT/goodput | Pending |
| G15 | Stream-safe drain | Pending |
| G16 | PDB/N-1 | Pending |
| G17 | Game days | Pending |
| G18 | Rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Acceptance evidence template

```text
UTC/context/change approval:
Cloud/region/AZ/GPU SKU:
Driver/device plugin/MIG:
Quota/RBAC/Pod Security/IAM:
GPU node inventory and placement:
HPA/KEDA/custom metrics:
CA/Karpenter ownership:
GPU node and model warmup timeline:
TTFT/TPOT/terminal-valid goodput:
PDB/stream drain/N-1:
Eight game-day findings:
GPU-hours/valid completion:
Rollback and GitOps diff:
Gate pass/fail/not-run + reviewer:
Final decision: NOT ACCEPTED until evidence and approvals complete
```

## Cleanup and rollback

Read-only/tabletop work creates no resources. Approved staging tests must restore all owned HPA/KEDA, node pool, quota, placement, PDB and model-serving changes through the approved source of truth; verify no orphaned GPU nodes, validate baseline SLO and reconcile costs. Failed or unexecuted gates remain pending.

**Part 07 written curriculum complete (10/10). Next: Chapter 056 — Golden Signals for LLM Inference.**
