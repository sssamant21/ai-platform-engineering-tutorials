# Chapter 020 — MIG Partitioning and Isolation

**Part 03 — CUDA, GPU Memory and Communication · Chapter 020 of 100**

**Publication:** Complete. **Offline tests, staging MIG reconfiguration, security review and production acceptance:** Pending.

## Objectives
Explain NVIDIA Multi-Instance GPU (MIG), GPU instances (GI), compute instances (CI), profile placement and device identity; plan secure multitenant capacity; operate MIG through Kubernetes GPU Operator/device plugin; diagnose placement, scheduling, OOM and telemetry faults; implement change, rollback and acceptance runbooks.

## 1. MIG architecture and isolation boundary
```text
Physical MIG-capable NVIDIA GPU (SKU / generation specific)
   |
   +-- GPU instance A: dedicated memory slices + compute partition
   |       +-- compute instance A1 -> workload / device UUID
   |
   +-- GPU instance B: memory + compute partition
           +-- compute instance B1 -> workload / device UUID

Host: NVIDIA driver -> MIG management -> GPU device plugin
                                     -> Kubernetes extended resources
                                     -> CUDA application containers
```
MIG partitions supported GPU resources into instances with hardware-enforced resource partitioning, including memory capacity and portions of compute/cache bandwidth. Exact isolation, partitionable resources, supported profiles, telemetry and interconnect behavior depend on GPU architecture, driver and MIG generation. **MIG is not a substitute for Kubernetes RBAC, network isolation, image security, workload identity, or a complete tenant threat model.**

Distinguish a physical GPU UUID, a MIG device UUID, GPU instance ID, compute instance ID and Kubernetes resource name. IDs can change after reconfiguration: use asset reconciliation and inventory snapshots, not stale integer indexes.

## 2. Profiles, placement and supported hardware
MIG profile names such as `1g.*gb`, `2g.*gb`, `3g.*gb`, `4g.*gb`, and `7g.*gb` are **examples**, not universally available. Availability and co-placement rules vary by SKU and generation; fractional compute/memory geometry does not mean arbitrary combinations fit. Use `nvidia-smi mig -lgip` and `nvidia-smi mig -lcip` when supported, plus the exact NVIDIA MIG guide, to determine legal placements.

| Dimension | What to verify | Common pitfall |
|---|---|---|
| Physical SKU | MIG support, GPU generation, firmware | assuming all NVIDIA GPUs support MIG |
| MIG mode | enabled/disabled state, pending reboot | expecting immediate mode switch |
| GI profile | slice count, framebuffer, placement | overcommitting partition geometry |
| CI profile | compute instance allocation | assuming one CI per GI universally |
| UUID mapping | physical GPU ↔ MIG device ↔ pod | confusing logical ordinal with UUID |
| Device plugin | `single` vs `mixed` strategy | requesting wrong resource name |
| Metrics | DCGM MIG-aware labels | attributing full-GPU metrics to a slice |
| Security | RBAC, namespace, workload identity | treating MIG as full tenant isolation |

## 3. Read-only host inventory
Run on an authorized GPU host:
```bash
date -u
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version,mig.mode.current,mig.mode.pending --format=csv
nvidia-smi mig -lgip
nvidia-smi mig -lcip
nvidia-smi -q | grep -Ei 'MIG Mode|GPU Instance|Compute Instance|UUID|FB Memory' | head -n 120
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|MIG|nvidia' | tail -n 100
```
Command/field support varies by GPU and driver; unsupported output is **unknown**, not zero. The above commands are intended for inspection only. Never create/destroy instances or toggle MIG mode on an active node without a separately approved change and verified workload evacuation.

### Kubernetes inspection
```bash
kubectl config current-context
kubectl get nodes -L nvidia.com/mig.capable,nvidia.com/mig.config,nvidia.com/mig.strategy
kubectl describe node APPROVED_GPU_NODE
kubectl get daemonsets -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl get pdb -A
```
GPU Feature Discovery labels and device-plugin resource names depend on installed versions and configuration. `nvidia.com/gpu` and `nvidia.com/mig-<profile>` can mean different things under `single` and `mixed` strategies; verify actual node allocatable and plugin documentation.

## 4. MIG configuration ownership
Choose exactly one approved controller for MIG lifecycle: NVIDIA MIG Manager through GPU Operator, a supported cluster operator workflow, or authorized bare-metal tooling. Avoid conflicting manual `nvidia-smi mig` mutations and reconciliation controllers.

**Production workflow:** asset/profile plan → compatibility check → workload/tenant approvals → N-1 capacity and checkpoint verification → PDB-aware eviction → approved configuration change → node/device-plugin reconciliation → MIG UUID/profile inventory → CUDA and memory-isolation smoke → telemetry/SLO acceptance → uncordon/re-admit. MIG reconfiguration can disrupt active CUDA contexts and may require GPU reset or reboot depending on platform/driver. A reset is not a harmless live operation.

### Scheduling example (conceptual)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mig-smoke
  namespace: gpu-staging
spec:
  restartPolicy: Never
  containers:
    - name: cuda-smoke
      image: registry.example.invalid/cuda-smoke@sha256:REPLACE_WITH_VERIFIED_DIGEST
      resources:
        limits:
          nvidia.com/mig-1g.10gb: 1
```
**Do not apply this placeholder manifest.** The resource name is illustrative and may not exist on your hardware. Use a verified digest, a real advertised resource name, approved namespace, security context and staging-only admission policy.

## 5. Capacity and memory economics
MIG partitions are not fungible full GPUs. Plan **profile-specific** capacity, placement fragmentation, reserved headroom, failure-domain spread and recovery of stranded slices. A pod requesting one MIG resource cannot necessarily run on an arbitrary free full-GPU node without a compatible scheduling and plugin strategy.

Measure workload-specific GPU memory, active SM use, p95/p99 inference latency, request concurrency, queue depth and MIG profile. Do not claim a fixed throughput fraction from profile geometry; memory bandwidth, cache partitioning, workload shape and device generation matter. Track per-profile utilization and idle capacity for FinOps.

## 6. Security and multitenancy
- Enforce namespaces, RBAC, admission, signed images, workload identity, secrets isolation and egress controls independently of MIG.
- Audit which tenants share the same physical GPU, and validate approved isolation assumptions with security owners.
- Prevent untrusted workloads from accessing host device nodes or privileged NVIDIA management interfaces.
- Keep MIG reconfiguration privileged and logged; restrict who may create GPU instances.
- Test memory isolation and fault blast radius on staging for the exact hardware/driver; do not infer guarantees from marketing labels.
- MPS and MIG combinations have SKU/driver-specific restrictions; consult NVIDIA documentation.

## 7. Observability and acceptance
Collect physical GPU UUID, MIG UUID, GI/CI identity where available, profile, Kubernetes node/pod/namespace, allocatable and allocated resources, device memory, Xid/ECC, DCGM metrics, device-plugin health, app SLO, stale telemetry and reconfiguration events. Some GPU counters cannot be attributed per MIG instance; mark unavailable metrics as unknown. Alert on resource mismatch, missing MIG devices, sustained OOM, critical Xid and SLO burn, not simply high utilization.

## 8. Ten troubleshooting scenarios
| Symptom | Evidence | Suspected boundary |
|---|---|---|
| no MIG profiles listed | SKU, driver, MIG mode | unsupported/disabled |
| MIG mode pending | current vs pending, active contexts | reboot/reset requirement |
| profile create rejected | GI placement/profile support | geometry/fragmentation |
| pod Pending | advertised extended resources | plugin/strategy mismatch |
| pod sees wrong GPU ordinal | CUDA_VISIBLE_DEVICES, MIG UUID | identity mapping |
| GPU memory OOM inside slice | per-instance memory and allocator | partition too small |
| DCGM metrics missing | exporter MIG support | telemetry gap |
| GPU Operator reverts manual change | controller ownership | reconciliation conflict |
| performance below expectation | profile, clocks, workload SLO | resource contention/fit |
| MIG device disappears after reboot | persistence/reconciliation | lifecycle/driver |

## 9. Eight incident simulations
1. **MIG profile rollout strands inference pods:** reject change until compatible profile capacity exists.
2. **Mixed-strategy resource mismatch:** compare advertised names and manifests; restore known-good strategy.
3. **MIG slice OOM while physical GPU has free VRAM:** right-size profile or workload; avoid global-GPU assumptions.
4. **MIG Manager reconciliation loop:** identify competing manual configuration; stop unauthorized mutation.
5. **Reconfiguration interrupts training:** recover from verified checkpoint; revise drain and approval gates.
6. **GPU UUID changes after recreation:** reconcile asset, DCGM labels and tenant attribution.
7. **Critical Xid on one physical GPU:** assess all colocated MIG tenants and hardware blast radius.
8. **Missing per-instance telemetry:** halt canary promotion until monitoring and SLO proof are restored.

## 10. Operational runbooks
**RB-020-A — MIG inventory:** record physical SKU/UUID, mode, GI/CI profiles, MIG UUIDs, driver, plugin and owner.

**RB-020-B — Profile capacity:** calculate legal placement, tenant demand, fragmentation, N-1 and zone spread.

**RB-020-C — Safe reconfiguration:** obtain change approval, checkpoint, drain, use supported controller and preserve rollback evidence.

**RB-020-D — Pod scheduling:** compare requested resource name with node allocatable, labels, taints and plugin strategy.

**RB-020-E — MIG OOM:** correlate slice memory, model/batch/KV budget, framework allocator and workload SLO.

**RB-020-F — Device loss/Xid:** quarantine affected physical GPU as warranted, evaluate colocated tenants and escalate vendor.

**RB-020-G — Telemetry/security:** verify MIG-aware DCGM, RBAC, workload identity, audit logs and missing metrics.

**RB-020-H — Recovery/RCA:** restore approved profile layout, smoke test, reconcile UUID and cost allocation, record COA.

## 11. Executable companion lab
[Chapter 020 — MIG Partitioning and Isolation Acceptance Lab](../labs/020-mig-partitioning-isolation-check.md) provides an offline Python 3.11+ policy validator and pytest suite for profile-specific capacity, ownership, isolation, health and change gates. It never configures MIG or contacts a GPU.

## 12. 100-point assessment
| Domain | Points |
|---|---:|
| MIG architecture and profile geometry | 20 |
| Kubernetes scheduling and lifecycle | 20 |
| Isolation, capacity and telemetry | 20 |
| Incidents and runbooks | 20 |
| Executable lab and staging evidence | 20 |
| **Total** | **100** |

Pass >=90 with no tenant-isolation, data-loss, unsupported-profile, recovery or SLO hard stop.

## References
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [NVIDIA Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [Kubernetes Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)

**Next:** Chapter 021 — NVLink and NVSwitch Topologies.
