# Chapter 012 — GPU Discovery with nvidia-smi and NVML

**Part 02 — Linux, Servers and GPU Hardware · Chapter 012 of 100**  
**Publication:** Complete. **Runtime execution, real GPU/NVML validation, independent technical review and production acceptance:** Pending.

## Outcomes
Discover GPU inventory and topology, interpret NVIDIA System Management Interface (nvidia-smi) and NVIDIA Management Library (NVML) output, correlate Linux and Kubernetes GPU identity, distinguish health from utilization, design safe polling, and troubleshoot discovery failures with operational runbooks.

## 1. Architecture and identity
```text
GPU firmware / PCIe hardware
           |
Linux NVIDIA kernel driver + device nodes
           |
        NVML library
         /       \
    nvidia-smi   DCGM / exporters / custom tools
         \       /
     inventory and observability
           |
GPU Operator / device plugin -> Kubernetes extended resources
           |
    GPU pods / workload SLOs
```

NVML exposes GPU identity, utilization, memory, clocks, temperature, power, ECC and device state where supported. `nvidia-smi` is a command-line client of NVML; NVML is not a substitute for application telemetry. NVIDIA DCGM adds monitoring/diagnostic capabilities for supported devices.

### Identity mapping
| Identifier | Purpose | Caveat |
|---|---|---|
| GPU index | convenience for one host/session | may change across reboots |
| GPU UUID | stable physical device identity in supported environments | use to join records; not a tenant authorization control |
| PCI bus ID | physical bus placement | may change with platform topology/firmware |
| MIG GPU/compute instance IDs | partition inventory | topology/configuration dependent |
| Kubernetes node name | scheduler placement | not an individual GPU ID |
| `nvidia.com/gpu` | extended resource count | does not describe per-device topology |
| container `CUDA_VISIBLE_DEVICES` | workload-visible mapping | indices inside containers may differ from host |

Never correlate GPUs solely by ordinal index across hosts, containers or restarts.

## 2. Read-only discovery commands
Run on an authorized Linux GPU host. Output may expose hardware identifiers; redact when sharing.
```bash
date -u
uname -r
lspci -nn | grep -Ei 'nvidia|3d controller'
nvidia-smi -L
nvidia-smi --query-gpu=index,uuid,name,pci.bus_id,driver_version,memory.total,memory.used,utilization.gpu,temperature.gpu,power.draw --format=csv
nvidia-smi -q
nvidia-smi topo -m
nvidia-smi --query-compute-apps=gpu_uuid,pid,process_name,used_gpu_memory --format=csv
```
Field availability varies by device, driver and MIG configuration; some process and utilization counters are unsupported or unavailable. Use `nvidia-smi --help-query-gpu` and `--help-query-compute-apps` to discover supported fields. Process lists can contain sensitive metadata.

**Interpreting metrics:** GPU utilization is a sampled busy fraction, not occupancy or achieved FLOPS. `memory.used` includes allocations not necessarily attributable to the visible processes. Power draw may be unsupported or temporarily unavailable. A healthy idle GPU can show 0% utilization; a 100%-busy GPU may still be bottlenecked on memory, kernels or synchronization.

## 3. NVML programming model
Use `pynvml`/NVIDIA-maintained Python bindings appropriate to the installed NVML library and platform. A safe collection pattern is initialize, enumerate, query each handle, catch expected unsupported/not-found exceptions, then shut down. Distinguish an unsupported metric from zero; never silently substitute zero for an unavailable counter.

```python
# Optional live inventory; requires compatible NVIDIA driver and NVML Python bindings.
import pynvml

pynvml.nvmlInit()
try:
    count = pynvml.nvmlDeviceGetCount()
    for index in range(count):
        device = pynvml.nvmlDeviceGetHandleByIndex(index)
        print(index, pynvml.nvmlDeviceGetUUID(device))
finally:
    pynvml.nvmlShutdown()
```
This is an **optional read-only example**, not executed here. Package APIs and return types can differ by binding version. Pin the binding version in your own environment and test it against the installed driver.

## 4. GPU discovery in Kubernetes
```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe node APPROVED_GPU_NODE
kubectl get daemonsets -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```
Compare expected GPU count with the node's **capacity and allocatable** GPU resources, device-plugin health, runtime integration and workload-visible inventory. GPU Operator driver installation must not conflict with a preinstalled host driver. MIG device plugins may expose profile-specific resource names rather than simple full-GPU counts. A Node `Ready` condition alone is not a GPU smoke test.

## 5. Monitoring and cardinality
For production, prefer DCGM exporter plus Prometheus/Grafana for supported GPU telemetry, with node identity, GPU UUID, environment, cluster and approved tenant labels. Avoid attaching unbounded pod IDs, request IDs or raw prompt/model content to high-frequency metrics. Scrape cadence must balance overhead, metric freshness and incident detection. Alert on **missing telemetry** independently of device faults. Preserve UTC timestamps and labels for join with kubelet, NVIDIA driver and application traces.

### Suggested SLI signals
| SLI | Source | Caution |
|---|---|---|
| GPU inventory completeness | NVML + expected inventory | MIG profiles complicate counts |
| device health | Xid/ECC, DCGM, driver logs | not every Xid proves failed hardware |
| GPU utilization | NVML/DCGM | sampled, not throughput |
| memory headroom | NVML/DCGM | not identical to framework allocation |
| thermal/power | supported NVML fields | device-specific |
| workload success | app/client telemetry | source of user-facing SLO |
| telemetry freshness | collector/scrape health | stale data must fail closed |

## 6. Hands-on lab sequence
1. Record OS/kernel, driver, GPU UUID, PCI bus ID and topology on an approved node.
2. Build a sanitized CSV snapshot of supported metrics; preserve unavailable values as unavailable.
3. Compare UUIDs with a signed expected inventory, detect duplicates, missing devices and unexpected devices.
4. Correlate node allocatable GPU resources and device plugin/Operator status.
5. Run the offline Python acceptance lab on synthetic JSON records.
6. In approved staging, compare inventory before and after a node reboot, driver upgrade and MIG reconfiguration.
7. Record sampling overhead, missing-metric handling, dashboards and pager tests.
8. Publish a change-controlled inventory with owner and rollback evidence.

## 7. Production incident simulations
**A — `nvidia-smi` cannot communicate with driver:** distinguish PCIe enumeration, kernel module load, Secure Boot signing, user-space mismatch and container runtime; preserve NVRM logs.

**B — GPU missing after reboot:** compare GPU UUID inventory, `lspci`, `nvidia-smi -L`, firmware and driver logs; quarantine node if expected devices are absent.

**C — NVML returns N/A:** identify unsupported metric, permission limitation or MIG restriction; do not convert to 0 or mark device healthy.

**D — Kubernetes shows zero allocatable GPUs:** inspect device plugin, GPU Operator, kubelet registration and resource naming; validate a pinned staging GPU smoke job.

**E — GPU memory apparently occupied with no process:** investigate driver/context state, permissions, MIG and container visibility; avoid resetting GPU on an active node.

**F — Telemetry silently stops:** alert on stale timestamps and exporter health; fall back to independent workload probes and stop unsafe rollout.

**G — GPU UUID mismatch after replacement:** reconcile inventory and asset ownership before admitting workloads; do not trust old GPU index.

**H — Utilization high but throughput low:** correlate power/thermal throttling, memory bandwidth, CPU/data loading, PCIe/NVLink and application profiling.

## 8. Troubleshooting matrix
| Symptom | Evidence | Suspected layer | Owner |
|---|---|---|---|
| no PCIe device | `lspci`, firmware | hardware/cloud | hardware SRE |
| device visible, NVML fails | module/NVRM logs | driver | GPU host SRE |
| NVML works, pod fails | runtime/device plugin | Kubernetes GPU integration | platform |
| duplicate inventory ID | collector joins | telemetry/inventory | observability |
| missing metrics | N/A, scrape age | support/exporter | observability |
| high ECC/Xid | driver and DCGM | GPU health | hardware/vendor |
| wrong MIG resource | MIG config/plugin | GPU partitioning | GPU platform |
| low useful throughput | app trace + GPU metrics | CPU/data/network | ML/HPC |

## 9. Operational runbooks
**RB-012-A — Inventory capture:** collect UTC time, node, GPU UUID, PCI ID, model, driver and owner; sanitize output.
**RB-012-B — NVML failure:** inspect PCIe, module, user-space driver and NVRM logs; preserve evidence and restore pinned image through approval.
**RB-012-C — Kubernetes mismatch:** compare host inventory, device plugin, operator, node resources and smoke-job visibility.
**RB-012-D — MIG inventory:** record physical GPU UUID, MIG configuration, exposed resources and isolation policy; validate after changes.
**RB-012-E — Telemetry outage:** distinguish unavailable metric from zero, restore scrape pipeline and confirm pager delivery.
**RB-012-F — Hardware escalation:** correlate Xid/ECC, power, thermals and workloads; quarantine only with approved capacity/rollback.
**RB-012-G — Post-change acceptance:** verify UUID continuity or documented replacement, application SLOs, GPU inventory, alerting and audit trail.

## 10. Companion lab and acceptance
[Chapter 012 — GPU Discovery Inventory Validator](../labs/012-gpu-discovery-nvidia-smi-nvml-check.md) provides a Python 3.11+ offline parser/validator and pytest suite. The lab validates **synthetic snapshots**, not actual NVML/device health.

### 100-point rubric
| Area | Points |
|---|---:|
| NVIDIA/NVML architecture and identities | 20 |
| Read-only host/Kubernetes discovery | 20 |
| Metric interpretation and monitoring | 20 |
| Incidents, security and runbooks | 20 |
| Executable lab and evidence | 20 |
| **Total** | **100** |

Pass >=90 with no critical device inventory, tenant isolation, telemetry or security blocker.

## References
- [NVIDIA System Management Interface](https://docs.nvidia.com/deploy/nvidia-smi/)
- [NVIDIA Management Library](https://docs.nvidia.com/deploy/nvml-api/)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

**Next:** Chapter 013 — GPU Hardware Health, ECC and Xid Errors.
