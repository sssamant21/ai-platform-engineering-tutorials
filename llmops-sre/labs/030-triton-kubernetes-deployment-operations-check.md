# Chapter 030 — Hands-On Lab: Triton Kubernetes Deployment and Operations

**Part 05 | Chapter 030 | Published lab plan; live execution pending**

## Goal

Design and inspect a secure Triton Kubernetes deployment with GPU scheduling, pinned model/image, probes, internal Services, repository storage, observability, rollout/rollback, HA and FinOps.

## Safety

**[SAFE-READ]** authorized cluster/GPU inventory and sanitized logs. **[LAB-MUTATE]** approved isolated staging, bounded GPU-hours and synthetic models. **[CHANGE-CONTROL]** production Deployment, gateway, GPU node, network policy, PVC, secrets, fault injection and rollback. No PHI, customer tensors or credentials.

## Lab 0 — Preflight

Record UTC, context, namespace, Triton image digest/backend versions, model repository checksum, GPU SKU/UUID, driver/CUDA, GPU quota, storage, gateway, CNI, owner, approvals, cost ceiling and rollback.

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

## Lab 1 — GPU scheduling [SAFE-READ]

```bash
kubectl describe node APPROVED_GPU_NODE
kubectl -n MODEL_NAMESPACE get pods -o wide
kubectl -n MODEL_NAMESPACE get events --sort-by=.lastTimestamp
```

Inspect allocatable GPU resources, node taints/labels, Pod requests/limits, ResourceQuota, device-plugin health and pending reasons. Do not assume fractional GPU support.

## Lab 2 — Deployment security [DESIGN]

Review the chapter's illustrative staging Deployment: immutable image digest, restricted Pod context, no automatic ServiceAccount token, read-only model volume, GPU resource request/limit and internal-only Service. Identify which security settings require compatibility validation for the pinned Triton image.

## Lab 3 — Probes and model readiness [DESIGN]

Define startup/readiness/liveness thresholds from measured cold-start. Distinguish server readiness from model-version readiness and synthetic inference correctness. Avoid expensive inference as a high-frequency liveness probe.

## Lab 4 — Storage and network [DESIGN]

Map model artifact source, checksum, workload identity, PVC/object-store access, DNS and egress. Design default-deny NetworkPolicy with explicit gateway/monitoring/required egress. Review administrative model-control path exposure separately from port exposure.

## Lab 5 — HA and rollout [DESIGN]

Model two GPU replicas across failure domains with PDB, topology spread, cold-start time and capacity for canary/surge. Explain why a one-replica Recreate rollout is not HA and why a PDB cannot manufacture GPU capacity.

## Lab 6 — Metrics and GPU FinOps [SAFE-READ]

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Design client E2E, model readiness, queue/compute, GPU telemetry, SLO burn and GPU-hours per valid completion. Confirm metric units/types and freshness.

## Lab 7 — Eight tabletop game days

01 device plugin failure; 02 security context startup; 03 PVC outage; 04 NetworkPolicy blocks gateway; 05 no surge GPU; 06 GPU node failure; 07 admin API exposure; 08 external model revision not rolled back.

## Twenty-four failure checklist

F01 GPU advertise; F02 Pending; F03 quota; F04 digest; F05 registry auth; F06 CUDA; F07 PVC; F08 permissions; F09 checksum; F10 non-root; F11 probe; F12 model readiness; F13 Service; F14 NetworkPolicy; F15 DNS/egress; F16 metrics exposure; F17 admin exposure; F18 GPU OOM; F19 surge; F20 drain/PDB; F21 device fault; F22 stale metrics; F23 mutable rollback; F24 GPU cost.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Pinned image/backend | Pending |
| G02 | Model provenance | Pending |
| G03 | GPU resources | Pending |
| G04 | Node scheduling | Pending |
| G05 | Storage | Pending |
| G06 | Security context | Pending |
| G07 | Startup probe | Pending |
| G08 | Model readiness | Pending |
| G09 | Synthetic inference | Pending |
| G10 | Internal Service | Pending |
| G11 | TLS/auth | Pending |
| G12 | Admin isolation | Pending |
| G13 | NetworkPolicy | Pending |
| G14 | GPU memory | Pending |
| G15 | HA/failure domain | Pending |
| G16 | Rollout/rollback | Pending |
| G17 | Observability freshness | Pending |
| G18 | SLO | Pending |
| G19 | FinOps/on-call | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/approval:
Triton image/backend/model revision:
GPU SKU/driver/device plugin/quota:
Deployment security/ServiceAccount:
Storage/IAM/model checksum:
Probes/server and model readiness:
Service/gateway/NetworkPolicy:
GPU memory/HA/rollout capacity:
Metrics/SLO/privacy:
GPU-hours/cost:
Game-day mode/findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. For separately approved staging, stop load, remove only owned test resources through authoritative GitOps, close port-forwards and reconcile GPU-hours. All gates remain pending until independently reviewed.

**Next: Chapter 031 — TensorRT-LLM Architecture and Compilation.**
