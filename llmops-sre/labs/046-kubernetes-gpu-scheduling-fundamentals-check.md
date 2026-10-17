# Chapter 046 — Hands-On Lab: Kubernetes GPU Scheduling Fundamentals

**Part 07 | Chapter 046 | Published lab plan; live execution pending**

## Goal

Inspect Kubernetes GPU resources, scheduling constraints, GPU device plugin state, VRAM/topology, namespace quotas and cloud node capacity. Design a safe synthetic GPU Pod test and operational recovery evidence.

## Guardrails

**[SAFE-READ]** authorized Kubernetes inspection and GPU diagnostics. **[LAB-MUTATE]** only approved isolated staging with bounded GPU-hours. **[CHANGE-CONTROL]** production node labels/taints, drivers, device plugins, quotas and privileged diagnostics. No PHI, customer prompts, Secrets or unapproved node changes.

## Lab 0 — Preflight

Record UTC, cluster/context, Kubernetes version, GPU SKU/node pool, device plugin/driver/runtime versions, namespace, ResourceQuota, change approval, SLO, GPU-hour ceiling and rollback.

## Lab 1 — GPU node inventory [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl describe node APPROVED_GPU_NODE
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

Capture Capacity, Allocatable, allocated GPU resources, taints, labels, conditions and node events.

## Lab 2 — Namespace constraints [SAFE-READ]

```bash
kubectl -n APPROVED_NS get resourcequota,limitrange
kubectl -n APPROVED_NS get pods -o wide
kubectl -n APPROVED_NS get events --sort-by=.lastTimestamp
```

Differentiate admission rejection from scheduler Pending and runtime crash.

## Lab 3 — Synthetic GPU Pod design [DESIGN]

```yaml
# ILLUSTRATIVE ONLY; IMAGE DIGEST IS INVALID UNTIL REPLACED
apiVersion: v1
kind: Pod
metadata:
  name: gpu-smoke
  namespace: approved-staging
spec:
  restartPolicy: Never
  containers:
  - name: cuda
    image: registry.example.invalid/cuda@sha256:REPLACE
    command: ["sh", "-c", "nvidia-smi && sleep 5"]
    resources:
      limits:
        nvidia.com/gpu: 1
```

Validate admission and placement only in approved staging after replacing image and namespace with approved values. Record GPU-hours and cleanup.

## Lab 4 — GPU health and topology [SAFE-READ]

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi -L
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu,temperature.gpu --format=csv
```

If host access is not approved, use existing DCGM telemetry. Do not create privileged debug Pods without authorization.

## Lab 5 — Scheduling recovery matrix [DESIGN]

| Symptom | First evidence | Recovery owner |
|---|---|---|
| Pod rejected | quota/admission | platform |
| GPU Pending | scheduler events | platform |
| wrong node | selectors/affinity | application |
| missing GPU | plugin/driver | GPU SRE |
| CUDA OOM | VRAM/runtime | model owner |
| no cloud nodes | autoscaler/quota | cloud SRE |

## Lab 6 — Eight tabletop game days

01 plugin missing; 02 taint drift; 03 quota block; 04 MIG profile drift; 05 same-node GPU shortage; 06 driver mismatch; 07 GPU node loss; 08 idle-cost spike.

## Twenty-four failure checklist

F01 GPU nodes; F02 plugin; F03 driver; F04 toolkit/CDI; F05 quota; F06 devices; F07 taints; F08 selectors; F09 affinity; F10 topology; F11 MIG; F12 time-slicing; F13 multi-GPU; F14 hardware; F15 ECC; F16 CUDA OOM; F17 KV; F18 preemption; F19 cloud quota; F20 image; F21 model storage; F22 telemetry; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions | Pending |
| G02 | GPU node inventory | Pending |
| G03 | Device plugin | Pending |
| G04 | Driver/runtime | Pending |
| G05 | GPU quota | Pending |
| G06 | GPU scheduling | Pending |
| G07 | Taints/tolerations | Pending |
| G08 | Affinity | Pending |
| G09 | Topology | Pending |
| G10 | MIG/sharing | Pending |
| G11 | Multi-GPU placement | Pending |
| G12 | VRAM/KV | Pending |
| G13 | GPU health | Pending |
| G14 | Node autoscaler/quota | Pending |
| G15 | Image/model readiness | Pending |
| G16 | Security/isolation | Pending |
| G17 | Telemetry/SLO | Pending |
| G18 | Rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/change approval:
Kubernetes/driver/device-plugin/runtime versions:
GPU node/SKU/allocatable/allocated:
Labels/taints/affinity/topology:
Namespace quotas and Pod events:
MIG/sharing configuration:
CUDA/VRAM/KV and GPU health:
Autoscaler/cloud GPU quota:
Model readiness and SLO:
Security/telemetry:
GPU-hours/cost:
Game-day findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop activities create no resources. Approved staging tests must delete only owned test Pods, restore GitOps manifests, verify no orphaned GPU nodes and reconcile GPU-hours. No acceptance gate is complete without independent evidence.

**Next: Chapter 047 — NVIDIA Device Plugin and GPU Operator.**
