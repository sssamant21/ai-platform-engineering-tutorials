# Chapter 021 — NVLink and NVSwitch Topologies

**AI Infrastructure Engineering & SRE · Part 03 — CUDA, GPU Memory and Communication · Chapter 021 of 100**

**Publication status:** Documentation published. Live GPU/NVSwitch qualification, test execution, security review, and production acceptance require separate evidence.

## Learning objectives

By the end of this chapter, an AI Infrastructure SRE should be able to:

1. Explain PCIe, NVLink, NVSwitch, GPU peer-to-peer (P2P), and cross-node networking boundaries.
2. Discover GPU-to-GPU topology and distinguish physical connectivity from usable communication paths.
3. Recognize how GPU generation, node SKU, MIG mode, firmware, drivers, and fabric management affect connectivity.
4. Diagnose topology mismatches, degraded links, fabric registration failures, and NCCL performance regressions.
5. Build topology-aware workload placement, canary, change, incident, and recovery procedures.
6. Verify acceptance through read-only inventory and a synthetic policy validator, then through separately approved hardware tests.

## 1. Architecture: GPU communication is a fabric, not one number

```text
                         AI training / inference application
                         PyTorch / NCCL / CUDA P2P
                                   |
                     CUDA driver / NCCL topology discovery
                                   |
  +------------------------- GPU server --------------------------+
  |                                                               |
  |   CPU socket / NUMA 0                   CPU socket / NUMA 1   |
  |       | PCIe root complex                  | PCIe root        |
  |       +--- NIC / storage                    +--- NIC           |
  |       |                                       |               |
  |     GPU 0 ===== NVLink ===== GPU 1           GPU 4 ...        |
  |        \                         /                             |
  |         +------ NVSwitch fabric --+                            |
  |        /                         \                             |
  |     GPU 2                       GPU 3                          |
  |                                                               |
  +-------------------------- NIC / HCA --------------------------+
                                  |
                   Ethernet / InfiniBand / EFA / RoCE
                                  |
                         Other GPU server(s)
```

This is **conceptual**, not a claim that every GPU server contains an NVSwitch. Some GPU platforms use direct GPU-to-GPU NVLink, others use one or more NVSwitch devices, and others have only PCIe connectivity. Newer systems may extend an NVLink domain beyond a single baseboard or chassis; verify exact vendor architecture.

**PCIe** connects GPUs to the host and other devices through root complexes and switches. **NVLink** provides supported high-bandwidth GPU communication links. **NVSwitch** enables switching across supported GPU topologies. **RDMA, EFA, InfiniBand and RoCE** address inter-node or NIC communication and are not interchangeable with NVLink. **GPUDirect RDMA** enables supported direct data movement between GPU and network devices, subject to IOMMU, PCIe, driver and fabric constraints.

## 2. Topology vocabulary

| Term | Meaning | Operational interpretation |
|---|---|---|
| GPU UUID | Stable device identity across most ordinary reboots | Prefer UUID over ordinal for inventory |
| PCI bus ID | Device PCIe address | May change with hardware/BIOS topology |
| NVLink | GPU interconnect link technology | Presence does not prove healthy throughput |
| NVSwitch | Switching fabric for supported GPU systems | May require Fabric Manager / platform management |
| P2P | GPU-to-GPU direct access capability | Check per GPU pair and application support |
| NUMA | CPU/memory locality | Affects staging, host buffers, NIC access |
| PCIe switch | PCIe fabric component | Not equivalent to NVSwitch |
| NIC/HCA | Network adapter | Cross-node collectives depend on network stack |
| NCCL topology | Collective communication graph | Selected paths can differ from hardware inventory |
| MIG | Partitioned GPU resources | P2P and communication support are architecture/profile dependent |

A topology label such as `NV#`, `PIX`, `PXB`, `PHB`, `NODE`, or `SYS` from `nvidia-smi topo -m` is an **indicator of relative path type**, not a direct measurement of effective bandwidth or guaranteed application performance. Label availability and semantics are driver/platform dependent.

## 3. Read-only inventory and discovery

Run only on an authorized GPU node:

```bash
date -u
hostnamectl
uname -r
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,pci.bus_id,driver_version --format=csv
nvidia-smi topo -m
nvidia-smi topo -mp
nvidia-smi -q -d NVLINK
nvidia-smi -q -d PCI
lspci -nn | grep -Ei 'NVIDIA|Ethernet|InfiniBand'
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|NVLink|NVSwitch|PCIe|AER' | tail -n 120
```

`nvidia-smi topo -mp`, `-d NVLINK`, and some NVLink counters are not supported on all drivers/devices. Confirm with `nvidia-smi --help`. Missing/unsupported data is **unknown**, never healthy or zero.

Where the NVIDIA Fabric Manager service is part of the vendor-certified platform, inspect without modifying it:

```bash
systemctl status nvidia-fabricmanager --no-pager
journalctl -u nvidia-fabricmanager -b --no-pager | tail -n 100
```

The service may be absent by design on platforms that do not need it. Its absence alone does not establish a fault. For Kubernetes:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe node APPROVED_GPU_NODE
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

Capture node, GPU UUIDs, PCI bus IDs, physical topology, driver, GPU Operator, Fabric Manager ownership, NIC and application image versions. Avoid exposing confidential infrastructure inventory or process arguments in shared tickets.

## 4. Hardware connectivity versus functional P2P

Three separate questions must be answered:

1. **Topology:** Are GPUs connected through NVLink, NVSwitch or PCIe?
2. **Capability:** Can this exact pair and driver configuration use CUDA peer access?
3. **Performance:** Does the actual communication workload meet latency/throughput and error-budget targets?

An NVLink label does not establish CUDA P2P support for a MIG device, and P2P support does not establish that NCCL uses the path or that a job reaches expected bandwidth.

### CUDA peer-access probe (staging)

Save as `p2p_probe.cu`:

```cuda
#include <cstdio>
#include <cuda_runtime.h>
int main() {
  int count = 0;
  cudaError_t err = cudaGetDeviceCount(&count);
  if (err != cudaSuccess) {
    std::fprintf(stderr, "CUDA discovery failed: %s\n", cudaGetErrorString(err));
    return 1;
  }
  std::printf("visible_devices=%d\n", count);
  for (int a = 0; a < count; ++a) {
    for (int b = 0; b < count; ++b) {
      if (a == b) continue;
      int supported = 0;
      err = cudaDeviceCanAccessPeer(&supported, a, b);
      if (err != cudaSuccess) {
        std::fprintf(stderr, "P2P %d->%d: %s\n", a, b, cudaGetErrorString(err));
        return 2;
      }
      std::printf("p2p[%d][%d]=%d\n", a, b, supported);
    }
  }
  return 0;
}
```

Run only on a GPU staging node with an approved CUDA toolkit:

```bash
nvcc -O2 p2p_probe.cu -o p2p_probe
./p2p_probe
```

This tests **peer-access capability**, not NVLink health, bandwidth, path selection or isolation. Device ordinals are process-relative; reconcile to UUIDs. A single visible GPU legitimately produces no pairwise checks.

## 5. Topology-aware workload placement

- Prefer validated GPU sets with the required connectivity for tensor/model parallelism and frequent collective operations.
- Consider GPU–NIC affinity and CPU NUMA when crossing nodes; local NVLink does not replace the network fabric.
- Use approved GPU SKU/node-pool constraints and application-level placement evidence. Kubernetes GPU resource counts do not encode all pairwise link requirements.
- Reserve sufficient headroom for N-1 recovery and checkpoint restoration.
- Account for MIG topology, device-plugin strategy, mixed GPU generations and firmware/driver skew.
- Use pinned application images and workload-specific benchmarks rather than a universal bandwidth target.

**Example:** an eight-rank training job requiring eight tightly coupled GPUs may not perform equivalently on eight GPUs spread across multiple PCIe-only nodes. Benchmark the exact topology, rank mapping, NIC fabric, NCCL build and workload before admission.

## 6. NVSwitch and Fabric Manager lifecycle

On platforms requiring Fabric Manager, compatible driver/fabric software and a healthy registered fabric may be necessary for certain CUDA operations. Service names, responsibilities and initialization behavior vary by generation and deployment. Never prescribe a universal “restart Fabric Manager” remediation while active training jobs are running.

Safe maintenance sequence:

1. Inventory node/SKU, NVSwitch topology, driver, Fabric Manager version and ownership.
2. Validate vendor compatibility matrix and known-good image.
3. Confirm checkpoint integrity, tenant notification, PDB/drain feasibility and spare capacity.
4. Drain the workload through an approved process.
5. Perform authorized image/firmware/fabric changes using the platform owner's runbook.
6. Reconcile GPU UUIDs, links, fabric state, CUDA P2P, NCCL and application SLO.
7. Stop promotion on degraded links, unexpected GPU count, Xid/AER, fabric registration failure or performance regression.
8. Restore a **tested** image/configuration pair and reopen only after acceptance.

AWS/Azure managed GPU hosts may not permit direct NVSwitch firmware or fabric changes; use cloud-supported node image, node replacement and vendor escalation.

## 7. Observability and performance

Track GPU UUID, physical/link topology, link state and error counters **where supported**, PCIe AER, Xid, ECC, GPU utilization, NCCL collective latency, communication bandwidth, NIC counters, queue depth, application step time, and telemetry freshness. Different GPU generations expose different counters. Avoid assuming DCGM metrics have uniform availability.

Compare a baseline against the same GPU count, GPU model, message size, rank mapping, CUDA/NCCL versions, network path, clocks and concurrent load. For NCCL benchmarks, capture test version and command, and avoid running saturation tests on production clusters without explicit approval.

## 8. Twelve troubleshooting scenarios

| Symptom | Evidence to collect | Potential cause |
|---|---|---|
| `nvidia-smi topo` shows PCIe instead of expected NVLink | SKU, board layout, topology baseline | wrong node SKU or degraded/missing link |
| NVLink counters unsupported | GPU/driver docs | feature or telemetry limitation |
| fabric initialization failure | Fabric Manager logs, driver version | version skew or platform fault |
| CUDA peer access denied | CUDA probe, MIG mode, GPU pair | unsupported pair/configuration |
| NCCL throughput drops | rank mapping, topology, NIC, clocks | path or workload regression |
| high PCIe AER | kernel logs, bus ID | PCIe fabric fault |
| Xid on NVSwitch-connected GPU | NVRM logs, all GPUs | hardware/driver incident |
| GPU UUID set changes | inventory, node replacement | enumeration or hardware change |
| topology-aware job placed incorrectly | scheduler, node labels | placement policy gap |
| MIG workload has no P2P | profile, architecture docs | capability limitation |
| missing GPU/NVLink metrics | DCGM/exporter support | telemetry outage |
| healthy links but slow training | NCCL algorithm, data loading, network | non-link bottleneck |

## 9. Eight incident game days

1. **Topology regression after node image change:** detect mismatch, stop rollout, revert to qualified image.
2. **Fabric Manager not ready:** hold GPU node admission, inspect platform logs, restore supported service/image.
3. **NVLink path degraded:** correlate link evidence, P2P and application throughput; quarantine if critical.
4. **PCIe AER storm:** investigate root port, GPU bus ID, firmware and node hardware; escalate OEM.
5. **NCCL collective slowdown:** compare rank mapping and cross-node NIC metrics before blaming NVLink.
6. **MIG/P2P incompatibility:** reject workload topology assumptions and place on supported resource profile.
7. **GPU replacement changes mapping:** reconcile UUIDs, labels, DCGM targets and scheduling constraints.
8. **Telemetry silently disappears:** fail closed on canary promotion; restore independent fabric observability.

## 10. Operational runbooks

**RB-021-A — Topology inventory:** Capture physical GPU UUIDs, PCIe bus IDs, `nvidia-smi topo`, link evidence and baseline.

**RB-021-B — Fabric readiness:** Verify whether Fabric Manager is required; compare supported versions and readiness.

**RB-021-C — CUDA P2P failure:** Run staging peer-access probe, check device/MIG mapping and official capability matrix.

**RB-021-D — NCCL performance regression:** Compare exact ranks, collective sizes, network, GPU clocks and topology.

**RB-021-E — NVLink/Xid/AER incident:** Preserve UTC logs, identify blast radius, checkpoint workloads and quarantine as approved.

**RB-021-F — Topology-aware scheduling:** Validate pool SKU, labels, placement, capacity and N-1 recovery.

**RB-021-G — Change and rollback:** Canary approved node image, test link/P2P/NCCL/SLO, stop on hard failures.

**RB-021-H — RCA and production acceptance:** Record topology delta, root cause, corrective actions, evidence and owner approval.

## 11. Companion lab

[Lab 021 — NVLink and NVSwitch Topology Acceptance](../labs/021-nvlink-nvswitch-topologies-check.md) contains a Python 3.11+ synthetic inventory/graph validator and pytest suite. The lab **does not** query GPUs or infer true NVLink topology from fabricated data.

## 12. 100-point assessment

| Domain | Points |
|---|---:|
| NVLink/NVSwitch/PCIe architecture | 20 |
| GPU topology discovery and P2P evidence | 20 |
| Placement, capacity and fabric lifecycle | 20 |
| Incident response and runbooks | 20 |
| Executable tests and staging acceptance | 20 |
| **Total** | **100** |

Pass at least 90/100 and no unresolved compatibility, data integrity, telemetry, security or SLO hard stop.

## References

- [NVIDIA NVLink and NVSwitch overview](https://www.nvidia.com/en-us/data-center/nvlink/)
- [NVIDIA Fabric Manager User Guide](https://docs.nvidia.com/datacenter/tesla/fabric-manager-user-guide/)
- [NVIDIA CUDA peer-to-peer APIs](https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__PEER.html)
- [NVIDIA NCCL documentation](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)
- [Kubernetes device plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

**Next:** Chapter 022 — NCCL Fundamentals and Collective Operations.
