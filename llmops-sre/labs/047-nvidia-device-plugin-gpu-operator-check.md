# Chapter 047 — Hands-On Lab: NVIDIA Device Plugin and GPU Operator

**Part 07 | Chapter 047 | Published lab plan; live execution pending**

## Goal

Discover NVIDIA GPU management ownership, device plugin registration, drivers, toolkit/CDI, MIG, DCGM and GPU health. Produce a safe upgrade/rollback and incident evidence package.

## Guardrails

**[SAFE-READ]** authorized cluster/host inspection and sanitized logs. **[LAB-MUTATE]** approved isolated staging only. **[CHANGE-CONTROL]** production drivers, GPU Operator, MIG profiles, runtime, privileged DaemonSets, node images and GPU pool maintenance. No PHI, Secrets or unapproved privileged debug Pods.

## Lab 0 — Preflight

Record UTC, cluster/context, Kubernetes/cloud node image, GPU SKU, driver, operator, device-plugin, toolkit/CDI and DCGM versions, GPU workloads, owner, approval, SLO, cost ceiling and rollback image/config.

## Lab 1 — Discover GPU management [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get daemonsets -A
kubectl get pods -A -o wide
kubectl get crd | grep -Ei 'nvidia|clusterpolic'
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

Do not assume GPU Operator is installed. Identify whether cloud-managed images or the operator own the driver/toolkit.

## Lab 2 — Device-plugin registration [SAFE-READ]

```bash
kubectl describe node APPROVED_GPU_NODE
kubectl -n DISCOVERED_NS get daemonset
kubectl -n DISCOVERED_NS logs DEVICE_PLUGIN_POD --tail=100 --since=15m
```

Record Capacity/Allocatable, plugin health, NVML initialization, runtime errors and node events. Redact sensitive log content.

## Lab 3 — GPU host diagnostics [SAFE-READ]

On an approved GPU host or existing authorized diagnostics container:

```bash
nvidia-smi
nvidia-smi -L
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu,temperature.gpu --format=csv
```

If host access is unavailable, use approved DCGM metrics. Never create a privileged Pod without approval.

## Lab 4 — MIG and sharing [DESIGN]

Record GPU SKU/MIG support, advertised resource names, strategy, existing workloads, drain requirements and rollback. Distinguish MIG hardware partitioning from time-slicing and VRAM isolation.

## Lab 5 — Upgrade and rollback [DESIGN]

Build compatibility matrix for kernel/driver/toolkit/device plugin/operator/DCGM, identify image ownership, plan one-node canary, drain, synthetic CUDA/model inference, SLO monitoring and complete rollback.

## Lab 6 — Eight tabletop game days

01 kernel mismatch; 02 plugin registration failure; 03 CDI breakage; 04 operator/cloud image conflict; 05 MIG drift; 06 Xid/ECC; 07 exporter loss; 08 mixed-version rollback.

## Twenty-four failure checklist

F01 operator absent; F02 conflicting drivers; F03 kernel mismatch; F04 driver; F05 toolkit; F06 CDI; F07 plugin absent; F08 crash loop; F09 NVML; F10 kubelet; F11 allocatable; F12 quota; F13 taint; F14 labels; F15 MIG profile; F16 MIG disruption; F17 time-slicing; F18 DCGM; F19 Xid/ECC; F20 cloud image; F21 RBAC; F22 privileged admission; F23 mixed-version rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/version/ownership | Pending |
| G02 | GPU node inventory | Pending |
| G03 | Driver/kernel | Pending |
| G04 | Toolkit/CDI | Pending |
| G05 | Device plugin DaemonSet | Pending |
| G06 | Kubelet registration | Pending |
| G07 | Allocatable GPU | Pending |
| G08 | Quota/taints | Pending |
| G09 | Feature labels | Pending |
| G10 | MIG/sharing | Pending |
| G11 | CUDA smoke | Pending |
| G12 | Model inference | Pending |
| G13 | GPU health | Pending |
| G14 | DCGM telemetry | Pending |
| G15 | Cloud image compatibility | Pending |
| G16 | Security/RBAC | Pending |
| G17 | Upgrade/canary | Pending |
| G18 | Rollback | Pending |
| G19 | FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/approval:
GPU SKU/cloud node image/ownership:
Kernel/driver/operator/plugin/toolkit/CDI versions:
GPU Capacity/Allocatable:
DaemonSet and plugin registration:
NVML/CUDA/MIG/sharing:
Xid/ECC/DCGM metrics:
Security/RBAC/privileged components:
Canary upgrade and rollback:
Synthetic inference and SLO:
GPU-hours/cost:
Eight game-day findings:
Reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Approved staging tests must restore original node image/operator settings, remove only owned test workloads, validate GPU resource registration, reconcile GPU-hours and retain sanitized evidence. No gate is complete without independent review.

**Next: Chapter 048 — GPU Node Pools, Taints, Tolerations and Affinity.**
