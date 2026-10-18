# Chapter 011 — NVIDIA Driver Lifecycle and Compatibility

**Part 02 — Linux, Servers and GPU Hardware · Chapter 011 of 100**  
**Publication:** Complete. **Offline lab tests, staging driver upgrade, independent security review and production acceptance:** Pending.

## Learning outcomes
By the end of this chapter, you should be able to inventory the NVIDIA kernel and user-space driver stack; explain driver, CUDA, GPU Operator, container toolkit and kernel compatibility; design an approved upgrade matrix; diagnose driver failures; execute safe canary/rollback workflows; and apply auditable readiness gates.

## 1. Why driver lifecycle management is an SRE responsibility
An NVIDIA GPU is only usable when firmware/hardware, PCIe enumeration, Linux kernel, NVIDIA kernel modules, user-space libraries, container runtime and workload libraries cooperate. A Kubernetes node can report **Ready** while GPU workloads fail to start. A mismatched or missing driver can disrupt inference, distributed training and checkpoint recovery across an entire node pool.

```text
AI application (PyTorch / vLLM / TensorRT-LLM)
                  |
       CUDA runtime / NCCL libraries
                  |
        libcuda.so (driver API)
                  |
     NVIDIA kernel modules / device files
                  |
       Linux kernel + Secure Boot
                  |
          PCIe / GPU / firmware
                  |
     containerd + NVIDIA runtime integration
                  |
       GPU device plugin / GPU Operator
                  |
 Kubernetes scheduling and GPU workloads
```

**Important distinction:** The CUDA Toolkit version installed on a host is not necessarily the CUDA runtime bundled into an application image, and neither is the NVIDIA driver version. Driver compatibility must be evaluated using the exact supported NVIDIA compatibility tables and application requirements. Do not infer support by comparing major version strings alone.

## 2. Component ownership and lifecycle
| Component | Typical owner | Verification | Failure mode |
|---|---|---|---|
| Linux kernel | OS/cloud image team | `uname -r`, package inventory | module ABI/signing mismatch |
| NVIDIA driver | image builder **or** GPU Operator | `nvidia-smi`, loaded modules | driver initialization failure |
| NVIDIA Container Toolkit | host/runtime team | runtime configuration and GPU smoke pod | missing device/library injection |
| GPU Operator | Kubernetes GPU platform | operator and driver DaemonSets | conflicting driver installation |
| Device plugin | Kubernetes GPU platform | node GPU capacity/allocatable | zero schedulable GPUs |
| CUDA user-space | application image owner | app build manifest | unsupported CUDA/driver pairing |
| NCCL | distributed training owner | pinned package + collective test | rank errors or hangs |
| Firmware | hardware/cloud provider | vendor-supported inventory | PCIe/device faults |

Select **one** driver installation authority per node pool: preinstalled in a managed image, configuration management, or operator-managed driver. Never allow multiple controllers to install incompatible versions.

## 3. Compatibility contract
Record these fields for every supported node class:

- OS distribution, release, kernel version and architecture.
- GPU model, compute capability, firmware and virtualization/MIG configuration.
- Driver **branch and exact version**, kernel module flavor and Secure Boot requirements.
- CUDA runtime/toolkit versions used by every supported workload image.
- NCCL, cuDNN and framework versions; container toolkit and runtime versions.
- Kubernetes version, GPU Operator, device plugin and node image versions.
- Vendor-supported matrix link, validation date, approval owner and rollback image digest.

NVIDIA supports documented compatibility mechanisms, including certain minor-version and forward-compatibility paths, but those have constraints. Check the official documentation for the specific driver/CUDA pairing; a newer CUDA application is **not automatically** supported by an older driver.

### Compatibility evidence matrix (fill with validated versions)
| Pool | GPU SKU | Kernel | Driver | CUDA images | Operator | Status |
|---|---|---|---|---|---|---|
| staging-inference | actual SKU | pinned | pinned | pinned | pinned | pending validation |
| staging-training | actual SKU | pinned | pinned | pinned | pinned | pending validation |
| prod-inference | actual SKU | pinned | pinned | pinned | pinned | requires change |
| prod-training | actual SKU | pinned | pinned | pinned | pinned | requires change |

Do not populate this table with invented compatible version pairs.

## 4. Driver release strategy
1. **Inventory:** capture deployed GPU/node classes, current versions, active workloads and support lifecycle.
2. **Assess:** read vendor security advisories and compatibility documentation, evaluate affected workloads and CVEs.
3. **Build:** pin packages and immutable image; verify package source, signature, SBOM and provenance.
4. **Stage:** reboot a representative GPU node; test kernel module, device enumeration, CUDA context creation, GPU allocation, NCCL and checkpoints.
5. **Canary:** one approved node/failure domain; enforce N-1 capacity, safe training drain and SLO-based stop conditions.
6. **Roll:** sequentially by zone and workload class, verifying each wave before the next.
7. **Close:** record version, health, performance, rollback evidence, ticket, UTC timestamps and owner signoff.

A GPU driver update commonly interrupts processes using the GPU. Plan training checkpoints and graceful workload migration before maintenance. Driver persistence mode is **not** a substitute for workload safety.

## 5. Hands-on Lab A — Read-only host diagnostics
On an approved Linux GPU node:
```bash
date -u
uname -r
cat /etc/os-release
lspci -nn | grep -i nvidia
lsmod | grep -E '^nvidia'
modinfo nvidia 2>/dev/null | grep -E '^(version|vermagic|signer):' || true
nvidia-smi -L
nvidia-smi --query-gpu=index,name,driver_version,pci.bus_id,memory.total --format=csv
nvidia-smi -q -d PERFORMANCE,ECC,PCI
journalctl -k -b --no-pager | grep -Ei 'nvidia|nvrm|xid|nouveau' | tail -n 100
```
A failed `nvidia-smi` is diagnostic evidence, not a reason to reinstall blindly. Compare `lspci` (hardware enumeration), `lsmod`/`modinfo` (module), `journalctl` (kernel) and user-space library versions. Commands may be unavailable on managed/locked-down hosts.

**Security:** kernel logs and device inventories may expose sensitive infrastructure identifiers. Redact before sharing.

## 6. Hands-on Lab B — Kubernetes GPU readiness
```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -A -o wide
kubectl get daemonsets -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe node APPROVED_GPU_NODE
```
Check GPU capacity/allocatable and GPU Operator/device-plugin health. Confirm whether the platform uses operator-managed drivers or preinstalled host drivers before touching DaemonSets. A separate **approved staging smoke job** should use an immutable CUDA image digest to test `nvidia-smi`, CUDA initialization and a minimal kernel, then validate application-specific CUDA/NCCL behavior. The read-only commands do not execute this job.

## 7. Upgrade preflight and rollback gates
**Preflight:** authorized change, owner, pinned previous and candidate images, vendor matrix evidence, GPU inventory, security assessment, checkpoint health, spare/N-1 capacity, monitored SLO, maintenance window and abort contact.

**Hard stops:** unknown driver ownership; unsigned/unapproved image; incompatible kernel; missing restore point; failed CUDA smoke test; GPU count mismatch; tenant isolation regression; insufficient capacity; unobservable canary; Xid/ECC escalation.

**Rollback:** stop rollout, isolate candidate node, preserve diagnostics, restore **previous approved image/configuration** and reboot when required; verify driver/module, GPU count, device plugin, smoke workload, application SLO and security. Some firmware/driver changes may not support a simple in-place downgrade; plan recovery before beginning.

## 8. Production incident simulations
**A — Kernel upgrade breaks module:** kernel ABI or module signing mismatch. Preserve `dmesg`, `modinfo`, Secure Boot state; restore compatible pinned image.

**B — `nvidia-smi` fails but PCIe sees GPU:** investigate driver module, device files, user-space/kernel library mismatch and competing installers.

**C — GPU visible on host, absent in pod:** inspect container toolkit, device plugin, operator and Kubernetes allocatable resources.

**D — CUDA error after application rollout:** compare app image CUDA runtime and actual host driver against supported matrix; roll back application or driver through approved path.

**E — NCCL hang after driver rollout:** compare per-rank logs, GPU/NIC topology, driver, NCCL and fabric; recover from verified checkpoint.

**F — Xid/ECC increase:** quarantine node, preserve evidence and follow vendor hardware triage; do not mask a physical fault with repeated driver reinstall.

**G — Canary causes inference SLO burn:** stop next wave, preserve old capacity, execute approved rollback and review admission/queue metrics.

**H — Operator fights preinstalled driver:** detect multiple driver owners and reconcile configuration in staging before production.

## 9. Troubleshooting decision table
| Symptom | First checks | Likely boundary | Escalation |
|---|---|---|---|
| GPU not in `lspci` | PCIe, firmware, cloud instance | hardware/virtualization | provider/vendor |
| GPU in `lspci`, not `nvidia-smi` | module, signing, NVRM logs | host driver | Linux/GPU SRE |
| `nvidia-smi` works, pod fails | toolkit, device plugin, CRI | container integration | Kubernetes GPU |
| CUDA initialization fails | runtime vs driver matrix | application/driver | ML platform |
| GPU count drops | Xid, plugin, node allocatable | hardware/operator | GPU platform |
| Training stalls | NCCL, fabric, version drift | collective/network | HPC/network |
| Canary latency rises | TTFT/TPOT, queues, clocks | workload/capacity | inference SRE |

## 10. Operational runbooks
**RB-011-A — Version inventory:** collect sanitized host/kernel/driver/GPU, operator, container runtime and application CUDA/NCCL versions; link vendor support evidence.

**RB-011-B — Preflight approval:** verify image signatures, compatible matrix, owner, N-1 headroom, checkpoints, monitoring, rollout order and rollback.

**RB-011-C — Driver initialization failure:** correlate hardware, module, kernel, Secure Boot and logs; isolate node and restore approved image.

**RB-011-D — GPU container failure:** verify host health, CRI, toolkit, device plugin, allocatable GPU and pinned smoke image.

**RB-011-E — Controlled canary:** test one node, monitor GPU/Xid/ECC, application SLO and training recovery; stop on hard gates.

**RB-011-F — Incident rollback:** preserve evidence, halt rollout, restore approved artifact, reboot if required and verify full service readiness.

**RB-011-G — Post-change acceptance:** attach versions, test output, dashboards, incident tickets, security review, cost and signoff.

## 11. Companion executable lab
[Chapter 011 — NVIDIA Driver Lifecycle and Compatibility Check](../labs/011-nvidia-driver-lifecycle-compatibility-check.md) implements a synthetic, fail-closed change evaluator with Python 3.11+ and pytest. It does **not** claim to derive vendor compatibility from version numbers.

## 12. 100-point acceptance
| Category | Points |
|---|---:|
| Driver stack architecture and ownership | 20 |
| Compatibility evidence and inventory | 20 |
| Staging/canary/rollback safety | 20 |
| Incident diagnostics and runbooks | 20 |
| Executable lab and evidence | 20 |
| **Total** | **100** |

Pass >=90 with no security, compatibility, recovery or capacity hard stop. **Publication ≠ runtime validation ≠ production approval.**

## Authoritative references
- [NVIDIA CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
- [NVIDIA Data Center GPU Driver Installation](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/)
- [NVIDIA System Management Interface](https://docs.nvidia.com/deploy/nvidia-smi/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

**Next:** Chapter 012 — GPU Discovery with nvidia-smi and NVML.
