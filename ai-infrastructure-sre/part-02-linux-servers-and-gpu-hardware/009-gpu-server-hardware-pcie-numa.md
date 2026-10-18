# Chapter 009 — GPU Server Hardware, PCIe and NUMA

**Part 02 — Linux, Servers and GPU Hardware · Chapter 009 of 100**  
**Publication:** Complete. **Offline test execution, real GPU topology measurements, independent review and production acceptance:** Pending.

## Learning objectives
Explain server CPU sockets, NUMA locality, PCIe lanes/root complexes, GPU-to-GPU interconnects and NIC affinity; collect Linux topology evidence; diagnose host-device bottlenecks; compare EKS/AKS and bare-metal GPU configurations; apply fail-closed placement checks and document a safe maintenance runbook.

## 1. Hardware reference architecture
```text
              dual-socket GPU server (illustrative)
       NUMA node 0                       NUMA node 1
   CPU0 + local DRAM                 CPU1 + local DRAM
        | PCIe root                        | PCIe root
     GPU0 / GPU1                       GPU2 / GPU3
        |                                  |
       NIC0                               NIC1
          \       host interconnect       /
                 storage / fabric
```
**Topology is platform-specific.** PCIe switch trees, GPU direct links, NIC attachment and NUMA mappings vary by server SKU and BIOS. NVLink and NVSwitch are available only on supported GPU/server combinations; never infer them from GPU count.

### Key components
| Component | What matters | SRE symptom |
|---|---|---|
| CPU socket / NUMA | memory locality, thread pinning | remote memory latency, CPU bottleneck |
| PCIe generation and width | negotiated link rate, lane count, switch contention | poor H2D/D2H copy throughput |
| GPU memory (HBM/GDDR) | capacity, bandwidth, ECC | OOM, ECC retirement, low utilization |
| GPU interconnect | NVLink/NVSwitch where present | collective communication stalls |
| NIC / RDMA | topology to GPU and fabric, IOMMU requirements | NCCL latency, retransmits |
| Local NVMe | dataset/cache staging | GPU idle waiting for data |
| PSU/cooling | power headroom, fan/thermal design | throttling, node shutdown |
| BIOS/firmware | NUMA, PCIe, ACS/IOMMU settings | device enumeration issues |

## 2. PCIe, bandwidth and NUMA theory
PCIe links are point-to-point; effective payload bandwidth is below theoretical line rate because of protocol overhead, transaction sizes, switch paths and contention. PCIe generation and lane count alone do not prove workload throughput. Example **theoretical one-way encoding-level** x16 bandwidth: PCIe 4.0 ~31.5 GB/s; PCIe 5.0 ~63 GB/s. These are not application benchmarks.

NUMA-aware placement tries to keep CPU threads, memory allocations, GPU and NIC close to the same socket. Linux first-touch memory policy, container CPU pinning, cpusets, huge pages, I/O threads and NIC IRQ affinity can alter results. A Kubernetes GPU request does **not** automatically guarantee optimal CPU, memory or NIC locality; inspect CPU Manager, Topology Manager and NUMA-aware scheduling configuration.

```text
data -> NVMe/object store -> CPU memory -> PCIe -> GPU HBM
                           \-> remote NUMA hop (possible)
GPU <-> NVLink/NVSwitch (only when supported)
GPU <-> PCIe / NIC <-> RDMA fabric <-> remote GPU
```

## 3. Reference deployment patterns
**Single-GPU inference:** optimize model fit, memory headroom, CPU pre/post-processing and cache warmup; avoid oversized host configurations without measured benefit.

**Multi-GPU inference:** verify tensor/pipeline parallelism topology, collective costs and GPU-to-GPU links; map model shards to supported peer-access paths.

**Distributed training:** prioritize measured all-reduce bandwidth, NIC/GPU affinity, local dataset staging and checkpoint throughput. Gang scheduling and placement are operational requirements.

**Kubernetes EKS/AKS:** inventory instance/VM SKU, GPU model, local disks, network capabilities, NUMA layout, device-plugin/operator versions and cloud quota. Cloud providers may virtualize or abstract topology details; verify capabilities on the exact SKU.

**Bare metal/Slurm:** explicit topology-aware rank binding and CPU affinity may be available; validate hardware management, fencing and tenant isolation.

## 4. Read-only hardware discovery lab
Run only on approved Linux GPU nodes:
```bash
uname -r
lscpu -e=CPU,NODE,SOCKET,CORE
numactl --hardware
lspci -tv
lspci -nn | grep -Ei 'nvidia|amd|ethernet|infiniband'
lsblk -o NAME,TYPE,SIZE,ROTA,MODEL
ip -br link
nvidia-smi -L
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,pci.bus_id,memory.total,driver_version,temperature.gpu --format=csv
```
Tools `numactl`, `lspci` and `nvidia-smi` may not be installed. A CPU-only host legitimately lacks NVIDIA devices. `nvidia-smi topo -m` shows host-local topology and CPU/NUMA affinity; it does not certify inter-host RDMA throughput.

### PCIe link verification
On an approved NVIDIA host, inspect `nvidia-smi -q -d PCI` and compare **current versus maximum** negotiated PCIe generation and width under a controlled workload. Idle GPUs may downshift; do not declare a defect from idle current speed alone. Inspect `lspci -vv -s APPROVED_BUS_ID` only when permissions allow. Do not change BIOS, kernel flags or ACS/IOMMU settings without vendor guidance and an approved change.

## 5. Performance validation plan
Capture baseline GPU model, bus ID, NUMA node, NIC proximity, negotiated PCIe width/generation, driver, CUDA/NCCL and BIOS versions. Measure separately:
1. Host-to-device and device-to-host copy throughput with pinned/unpinned host memory.
2. GPU-to-GPU peer bandwidth and latency, where supported.
3. NCCL all-reduce and all-gather across intra-node and inter-node topologies.
4. Dataset load throughput and CPU preprocessing utilization.
5. End-to-end inference TTFT/TPOT or training step time at passing SLOs.
6. Thermal/power throttling, ECC/Xid, and stability during sustained load.

Do not compare synthetic numbers to vendor benchmark claims without matching GPU SKU, payload size, concurrency, precision, software stack and topology. Capture warmup and multiple runs.

## 6. Kubernetes topology and placement
Inspect the node's advertised GPU resources, CPU allocation, taints, topology spread, CPU Manager and Topology Manager policy. The GPU device plugin controls GPU resource discovery; Kubernetes Topology Manager can coordinate NUMA hints from supported managers, but its effectiveness depends on configured policies and the runtime/device plugin. For distributed jobs, account for node-level and network-fabric locality.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe node APPROVED_GPU_NODE
kubectl get pods -A -o wide
kubectl get daemonsets -A
kubectl get events -A --sort-by=.lastTimestamp
```
These are read-only but may reveal sensitive metadata; redact before sharing.

## 7. Failure scenarios
**A — PCIe downtraining:** Under load, a GPU negotiates fewer lanes than supported. Check physical slot, BIOS, riser, firmware and error counters. Escalate to hardware owner; do not reseat hardware live.

**B — NUMA remote-memory penalty:** CPU-bound preprocessing threads run on the wrong socket. Correlate CPU/NUMA allocation, memory policy and latency; test approved pinning changes in staging.

**C — NCCL underperformance:** GPU peers use PCIe when expected NVLink path is absent. Verify actual GPU/server support and topology rather than forcing unsupported peer access.

**D — GPU thermally throttles:** Sustained GPU utilization coincides with high temperature or power caps. Check fan, airflow, rack power and firmware with hardware team.

**E — RDMA/NIC misplacement:** GPU and NIC are attached to different NUMA/root complexes. Measure rank performance and NIC counters; adjust placement only after controlled test.

**F — GPU missing after reboot:** Device not enumerated or driver cannot initialize. Check `lspci`, driver logs, kernel compatibility and device plugin; use rollback matrix.

## 8. Troubleshooting matrix
| Symptom | Evidence | Suspected cause | Owner |
|---|---|---|---|
| low GPU utilization | CPU, I/O, GPU metrics | data/CPU bottleneck | GPU/ML platform |
| H2D throughput low | PCIe link and transfer test | lane/gen mismatch | hardware SRE |
| inter-GPU transfer slow | `nvidia-smi topo -m` | topology/peer path | GPU platform |
| NCCL hangs | rank logs, NIC counters | fabric, rank or affinity | HPC/network |
| device absent | `lspci`, driver, plugin | PCIe/driver failure | hardware/GPU SRE |
| frequent Xid/ECC | GPU diagnostics | hardware or driver fault | GPU/vendor |
| node throttling | power/thermal telemetry | cooling/power | datacenter/cloud |

## 9. Operational runbooks
**RB-009-A — Inventory:** capture sanitized node ID, GPU model/count, PCIe bus IDs, NUMA, NIC, firmware, kernel, driver and owner.

**RB-009-B — Baseline:** run safe topology discovery, copy/collective tests under approved load, preserve units and software versions.

**RB-009-C — PCIe regression:** compare negotiated width/gen under load to vendor-supported values, inspect errors, stop unsafe changes and escalate hardware remediation.

**RB-009-D — NUMA regression:** trace CPU, memory, GPU and NIC locality; stage pinning changes, measure SLO impact and maintain rollback.

**RB-009-E — Hardware fault:** cordon/drain only with approved change, preserve Xid/ECC and incident timeline, validate N-1 capacity and return node after burn-in.

**RB-009-F — Acceptance:** verify performance, tenant isolation, health metrics, quotas, recovery, change approvals and cost evidence.

## 10. Executable offline lab
[Chapter 009 — GPU PCIe and NUMA Topology Acceptance Lab](../labs/009-gpu-server-hardware-pcie-numa-check.md) validates synthetic topology inventories, link width/gen expectations and locality risks. It is not a hardware benchmark.

## 11. 100-point assessment
| Category | Points |
|---|---:|
| GPU hardware and PCIe architecture | 20 |
| NUMA and NIC/GPU locality | 20 |
| Linux/Kubernetes discovery | 20 |
| Incident diagnostics and runbooks | 20 |
| Lab, evidence and acceptance | 20 |
| **Total** | **100** |

Pass >=90 with no critical safety, hardware health or tenant isolation blocker. A policy check alone does not certify hardware.

## References
- [NVIDIA System Management Interface](https://docs.nvidia.com/deploy/nvidia-smi/)
- [Linux kernel NUMA documentation](https://docs.kernel.org/admin-guide/mm/numa_memory_policy.html)
- [Kubernetes Topology Manager](https://kubernetes.io/docs/tasks/administer-cluster/topology-manager/)
- [Kubernetes CPU Manager](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/)
- [NVIDIA NCCL Documentation](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)

**Next:** Chapter 010 — Linux GPU Host Provisioning and Baselines.
