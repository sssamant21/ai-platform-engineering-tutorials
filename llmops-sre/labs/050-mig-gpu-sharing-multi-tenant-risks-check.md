# Chapter 050 — Hands-On Lab: MIG, GPU Sharing and Multi-Tenant Risks

**Part 07 | Chapter 050 | Published lab plan; live execution pending**

## Goal

Discover MIG support, advertised GPU resources and sharing configuration; assess VRAM and tenant fairness; design a bounded synthetic noisy-neighbor test; document safe reconfiguration, rollback and GPU FinOps.

## Safety

**[SAFE-READ]** authorized Kubernetes and GPU inventory. **[LAB-MUTATE]** approved isolated staging only, synthetic traffic and strict GPU-hour caps. **[CHANGE-CONTROL]** MIG mode/profile, GPU Operator, device plugin, time-slicing/MPS, production quotas and driver changes. Never reconfigure active production GPUs from this lab.

## Lab 0 — Preflight

Record UTC, context, GPU SKU, driver/device-plugin/operator versions, MIG support/strategy, resource keys, tenant namespaces, model/tokenizer digests, SLO, change approval, cost ceiling and rollback.

## Lab 1 — Kubernetes GPU resource inventory [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get nodes --show-labels
kubectl describe node APPROVED_GPU_NODE
kubectl get pods -A -o wide
kubectl get resourcequota -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

Record physical GPUs, advertised MIG resource names, allocation, quotas, node labels, taints and Pod scheduling events.

## Lab 2 — GPU/MIG host inventory [SAFE-READ]

On an authorized GPU host:

```bash
nvidia-smi -L
nvidia-smi -q -d MIG
nvidia-smi mig -lgip
nvidia-smi mig -lcip
```

Unsupported MIG commands are expected on some SKUs. Do not run configuration or reset commands.

## Lab 3 — Sharing threat-model worksheet [DESIGN]

| Dimension | Whole GPU | MIG | Time-slicing |
|---|---|---|---|
| hardware availability | pending | pending | pending |
| advertised resource names | pending | pending | pending |
| VRAM isolation | pending | pending | pending |
| noisy-neighbor exposure | pending | pending | pending |
| tenant security review | pending | pending | pending |
| TTFT/TPOT impact | pending | pending | pending |
| GPU-hours/valid completion | pending | pending | pending |

Document actual hardware/plugin behavior rather than assumptions.

## Lab 4 — Bounded synthetic workload [DESIGN]

Plan baseline, burst, long-context, concurrent tenants, cancellation and noisy-neighbor scenarios. Require synthetic prompts, explicit RPS/concurrency/max tokens, duration, GPU-hour ceiling, abort thresholds, privacy controls and cleanup. Execute only in approved isolated staging.

## Lab 5 — MIG reconfiguration rollback [TABLETOP]

Inventory workloads; calculate required GPU profiles; plan drain, supported operator workflow, plugin resource verification, synthetic CUDA/model readiness, telemetry and full rollback. Do not mutate MIG configuration during tabletop.

## Lab 6 — Eight game days

01 missing profile; 02 MIG Manager failure; 03 shared VRAM OOM; 04 KV saturation; 05 TTFT noisy neighbor; 06 Xid/ECC; 07 quota oversubscription; 08 incomplete rollback.

## Twenty-four failure checklist

F01 unsupported SKU; F02 strategy; F03 profile; F04 disruption; F05 Manager; F06 plugin stale; F07 quota; F08 time-slicing assumption; F09 VRAM OOM; F10 KV; F11 TTFT; F12 TPOT; F13 batch fairness; F14 MPS; F15 model fit; F16 CUDA visibility; F17 Xid/ECC; F18 DCGM; F19 IAM; F20 RBAC; F21 token quota; F22 chargeback; F23 rollback; F24 idle MIG.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions | Pending |
| G02 | SKU/MIG support | Pending |
| G03 | Driver/plugin | Pending |
| G04 | MIG strategy | Pending |
| G05 | Advertised resources | Pending |
| G06 | Quota/admission | Pending |
| G07 | Profile feasibility | Pending |
| G08 | Sharing/MPS policy | Pending |
| G09 | VRAM/KV | Pending |
| G10 | Noisy-neighbor SLO | Pending |
| G11 | Token fairness | Pending |
| G12 | IAM/RBAC | Pending |
| G13 | Artifact isolation | Pending |
| G14 | GPU health | Pending |
| G15 | DCGM telemetry | Pending |
| G16 | Safe reconfiguration | Pending |
| G17 | Terminal-valid inference | Pending |
| G18 | Rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/approval:
GPU SKU/driver/plugin/operator:
MIG support/profiles/strategy:
Advertised GPU resources and quotas:
Time-slicing/MPS policy:
VRAM/KV/model fit:
Tenant isolation/IAM/RBAC:
TTFT/TPOT/terminal-validity:
DCGM/Xid/ECC:
Synthetic noisy-neighbor findings:
GPU-hours/cost:
Eight game-day findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop activities create no resources. Approved staging work must restore original GPU configuration via authorized workflow, remove only owned Pods, verify device-plugin registration, reconcile GPU-hours and retain sanitized evidence. All acceptance gates remain pending until independent review.

**Next: Chapter 051 — HPA, KEDA and Custom Inference Metrics.**
