# Chapter 050 — MIG, GPU Sharing and Multi-Tenant Risks

**Part 07 — Kubernetes GPU Scheduling and Autoscaling | Chapter 050 of 100**

> **Status:** Written tutorial published. Live lab execution, independent technical/security review and operational acceptance pending. MIG availability, GPU sharing semantics and advertised resource names depend on NVIDIA GPU SKU, device-plugin/GPU Operator version, driver and cluster configuration. MIG reconfiguration is disruptive; do not change production devices without approved maintenance.

## Learning outcomes

Distinguish physical GPU allocation, NVIDIA Multi-Instance GPU (MIG), time-slicing and other GPU sharing strategies; plan tenant isolation and VRAM budgets; inspect device plugin resources, MIG profiles and health; troubleshoot scheduling and noisy-neighbor incidents; compare utilization, SLO and FinOps tradeoffs.

## 1. Reference architecture

```text
Tenant A/B/C inference traffic
       │ gateway auth / token quotas / queues
       ▼
Kubernetes namespaces + ResourceQuota
       │
       ▼
GPU scheduling via advertised extended resources
       ├─ full physical GPU
       ├─ MIG hardware GPU instances (supported SKUs)
       └─ time-sliced/shared advertised devices
       │
       ▼
NVIDIA device plugin + GPU Operator (if installed)
       │
       ▼
NVIDIA driver / CUDA / NVML / DCGM
       │
       ▼
GPU VRAM, SMs, cache, memory bandwidth, thermal/power
```

GPU resource requests govern scheduling, not universal isolation guarantees. **Namespaces and time-slicing are not hardware security boundaries.**

## 2. GPU allocation models

| Model | Typical use | Isolation | Operational caveat |
|---|---|---|---|
| whole GPU | large LLM, strict latency | dedicated allocation in common configurations | high idle cost |
| MIG | smaller models on supported hardware | hardware partitioning of supported GPU resources | fixed profiles and disruptive reconfiguration |
| time-slicing | bursty smaller workloads | no hard VRAM isolation | noisy neighbors and memory contention |
| MPS | CUDA multi-process sharing where supported | not equivalent to MIG | version/runtime/security constraints |
| application batching | concurrent requests within one server | application-enforced fairness | KV/cache and token budget risks |

Do not equate one logical advertised GPU share with one full physical GPU. Verify exact plugin sharing configuration and resource accounting.

## 3. MIG fundamentals

MIG partitions compatible GPUs into GPU instances with dedicated slices of compute and memory resources. Available profiles depend on GPU generation, SKU, driver and MIG mode. Profile naming and supported instance combinations must be obtained from the target GPU.

**[SAFE-READ]** On an authorized GPU host:

```bash
nvidia-smi -L
nvidia-smi -q -d MIG
nvidia-smi mig -lgip
nvidia-smi mig -lcip
```

Commands may vary or report unsupported functionality on non-MIG hardware. These are read-only inventory commands; do not use `nvidia-smi mig -cgi` or `-dgi` without maintenance approval.

## 4. Kubernetes MIG resources

NVIDIA device plugin may expose MIG resources such as `nvidia.com/mig-1g.10gb` under some strategies and GPUs. **Do not hardcode profile names:** inspect node Capacity/Allocatable and device-plugin configuration.

```bash
kubectl describe node APPROVED_GPU_NODE
kubectl get nodes --show-labels
kubectl get daemonsets -A
kubectl get pods -A -o wide
```

The exact MIG resource profile can change when GPU partitions are reconfigured; a Pod requesting an unavailable profile remains Pending.

## 5. Device-plugin MIG strategies

NVIDIA device-plugin configurations may support `none`, `single` and `mixed` MIG strategies, subject to version and hardware. `single` and `mixed` can expose different resource keys/semantics. Inspect installed chart/ClusterPolicy and avoid changing a fleet-wide strategy without compatibility testing.

## 6. Time-slicing semantics

Time-slicing allows multiple workloads to share GPU compute time; it does **not** provide MIG-style memory isolation. Replica counts advertised by the plugin may reflect oversubscription rather than physical devices. Tenant GPU quotas based on these advertised counts may overstate guaranteed capacity.

## 7. MPS versus MIG

CUDA Multi-Process Service (MPS) is a process-sharing mechanism, not a substitute for hardware partitioning. Its support in GPU Operator/device-plugin configurations and tenant-security suitability are version-specific. Require security review and measured performance before enabling MPS.

## 8. VRAM and KV-cache capacity

```text
required VRAM per tenant/model
  = weights + KV cache + activations + CUDA graphs
    + runtime workspace + safety headroom
```

A small MIG profile may not fit the model or long-context KV cache. Time-sliced workloads may contend for the same physical VRAM; even when Pods schedule, CUDA OOM can occur. Measure actual model size, precision, batch/concurrency, token lengths and fragmentation.

## 9. Multi-tenant threat model

| Threat | Example | Mitigation |
|---|---|---|
| availability | tenant exhausts GPU VRAM | dedicated/MIG allocation, admission |
| performance | noisy neighbor spikes TTFT | bounded concurrency, isolation |
| confidentiality | prompt data in logs/metrics | redaction, access controls |
| identity | cross-tenant model artifact access | workload identity/IAM |
| API abuse | excessive tokens/requests | tenant token and RPS quotas |
| privilege | tenant alters GPU Operator | RBAC/admission |
| cost | oversubscribed GPUs or idle MIG | chargeback and alerts |

MIG improves resource isolation on supported hardware but should not be presented as a complete cross-tenant security guarantee. Use defense in depth and vendor-specific security review.

## 10. Isolation tiers

**Tier A:** dedicated physical GPU for regulated or strict-SLO workloads. **Tier B:** supported MIG hardware partition with explicit threat-model approval. **Tier C:** time-sliced/shared GPU for lower-trust-risk or controlled internal workloads after testing. **Tier D:** application-level multiplexing with per-tenant token/concurrency limits. Choose based on security, latency, utilization and cost, not only GPU occupancy.

## 11. Scheduling and quota design

ResourceQuota can limit extended GPU resources but does not guarantee per-tenant VRAM fairness under sharing. Combine quotas with namespace RBAC, admission, node-pool isolation, gateway token budgets and model-runtime limits. Do not assume that a request for `nvidia.com/gpu: 1` always means exclusive physical access.

## 12. Performance testing

Compare whole GPU, MIG and sharing modes under identical model, precision, context distribution, concurrency and GPU SKU. Measure terminal-valid goodput, TTFT p95/p99, TPOT, queue wait, VRAM peak, GPU utilization, OOM rate and GPU-hours per valid completion. Include burst, long-context and noisy-neighbor tests.

## 13. MIG lifecycle and maintenance

```text
Inventory GPU SKU/driver/active Pods
 → compatibility and profile feasibility
 → maintenance approval and rollback
 → drain/evacuate workloads with SLO guardrails
 → change MIG config using supported operator workflow
 → validate advertised resource names and GPU health
 → synthetic CUDA/model inference
 → progressive traffic restore and FinOps audit
```

Changing MIG mode/profiles can disrupt other GPU workloads and may require GPU reset or node replacement depending on platform.

## 14. Observability

Track per-device/MIG instance memory, utilization, ECC/Xid, temperature, throttling, scheduler Pending reasons, model OOM, TTFT/TPOT, tenant token usage, streaming terminal completion and cost attribution. DCGM exporter label/cardinality and MIG support vary by configuration; verify actual metrics.

## 15. Safe-read triage

```bash
date -u
kubectl config current-context
kubectl get nodes --show-labels
kubectl describe node APPROVED_GPU_NODE
kubectl get pods -A -o wide
kubectl get resourcequota -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

For an approved GPU host, inspect `nvidia-smi -L` and `nvidia-smi -q -d MIG`. Do not launch privileged diagnostics without authorization.

## 16. Decision tree

```text
Shared GPU incident?
 ├─ Pod Pending → MIG resource name/profile/quota
 ├─ device absent → plugin/MIG Manager/driver
 ├─ CUDA OOM → weights + KV + shared VRAM
 ├─ TTFT/TPOT regression → noisy neighbor/throttling
 ├─ Xid/ECC → device health
 ├─ tenant starvation → token/concurrency fairness
 ├─ isolation breach → security/identity/logging
 ├─ chargeback mismatch → physical vs logical GPU
 └─ rollback drift → MIG profile/plugin configuration
```

## 17. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | unsupported MIG GPU SKU | inventory |
| F02 | wrong MIG strategy | plugin config |
| F03 | requested MIG profile absent | node allocatable |
| F04 | MIG reconfiguration disrupts Pods | events |
| F05 | MIG Manager reconciliation fails | controller logs |
| F06 | device plugin advertises stale profiles | node status |
| F07 | GPU quota uses wrong resource name | admission |
| F08 | time-slicing misrepresented as exclusive | sharing config |
| F09 | shared VRAM OOM | CUDA/DCGM |
| F10 | KV-cache exhaustion | runtime |
| F11 | noisy-neighbor TTFT spike | latency |
| F12 | noisy-neighbor TPOT spike | decode |
| F13 | tenant monopolizes batch slots | scheduler/runtime |
| F14 | MPS mode unsupported | compatibility |
| F15 | model too large for MIG profile | VRAM |
| F16 | CUDA runtime cannot see MIG device | startup |
| F17 | Xid/ECC device fault | DCGM |
| F18 | DCGM MIG metrics absent | exporter |
| F19 | tenant accesses wrong model artifact | IAM audit |
| F20 | GPU Operator privileges exposed | RBAC |
| F21 | token quota missing | gateway |
| F22 | chargeback treats shares as physical GPUs | FinOps |
| F23 | rollback leaves mixed MIG profiles | fleet audit |
| F24 | idle MIG partitions waste GPU-hours | cost |

## 18. Sixteen operational runbooks

1. Inventory MIG support, SKU and driver.
2. Verify device-plugin MIG strategy.
3. Recover unavailable MIG resource profile.
4. Plan safe MIG reconfiguration and drain.
5. Recover MIG Manager reconciliation.
6. Restore kubelet resource registration.
7. Diagnose extended-resource quota mismatch.
8. Audit time-slicing allocation and oversubscription.
9. Recover shared VRAM and KV OOM.
10. Contain noisy-neighbor TTFT/TPOT regressions.
11. Enforce per-tenant token and concurrency fairness.
12. Validate MPS compatibility and security.
13. Recover CUDA MIG device visibility.
14. Restore DCGM, Xid/ECC and SLO telemetry.
15. Reconcile isolation, IAM and chargeback.
16. Execute complete MIG/sharing rollback and RCA.

## 19. Eight tabletop game days

1. Requested MIG profile disappears after node replacement.
2. MIG Manager fails during profile transition.
3. Time-sliced tenant triggers another tenant's OOM.
4. Long-context workload saturates KV cache.
5. Noisy neighbor doubles p99 TTFT.
6. Xid/ECC fault affects shared GPU workloads.
7. Tenant quota permits unexpected oversubscription.
8. Rollback restores model Pods but not MIG profile.

Default tabletop; live fault injection requires approved isolated staging.

## 20. Twenty acceptance gates

G01 scope/versions; G02 GPU SKU/MIG support; G03 driver/plugin; G04 MIG strategy; G05 advertised resources; G06 quota/admission; G07 profile feasibility; G08 time-slicing/MPS policy; G09 VRAM/KV capacity; G10 noisy-neighbor SLO; G11 tenant token fairness; G12 IAM/RBAC; G13 model artifact isolation; G14 GPU health; G15 DCGM telemetry; G16 safe reconfiguration; G17 terminal-valid inference; G18 rollback; G19 GPU FinOps; G20 independent review.

See [companion lab](../labs/050-mig-gpu-sharing-multi-tenant-risks-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 051 — HPA, KEDA and Custom Inference Metrics.**
