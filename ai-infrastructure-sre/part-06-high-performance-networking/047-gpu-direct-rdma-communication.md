# Chapter 047 — GPU Direct RDMA and Communication

**AI Infrastructure Engineering & SRE · Part 06 — High Performance Networking · Chapter 047/100**

> **Publication is not certification.** This chapter and offline gate document an enterprise acceptance workflow; live GPUDirect RDMA verification, performance tests, security review and operational signoff remain pending.

## Learning objectives

Explain GPUDirect RDMA, distinguish it from NVLink/NVSwitch and ordinary host-memory RDMA, validate GPU–NIC PCIe/NUMA locality, understand memory registration, DMA-BUF and legacy peer-memory paths, interpret NCCL transports, build Kubernetes GPU/RDMA workload placement, troubleshoot fallback, and implement safe SRE change, capacity and recovery controls.

## 1. Architecture

```text
     Distributed training ranks / PyTorch / NCCL
                    |             |
                GPU A           GPU B
                    |             |
               PCIe fabric / IOMMU / root complex
                    |             |
                HCA A           HCA B
                    +------ fabric ------+
                       IB / RoCE / EFA*
                          |
                storage / checkpoint
```

*EFA uses its own supported libfabric/provider integration. Do not assume every EFA instance, NCCL plugin or GPU configuration supports the same GPUDirect RDMA mechanism as a native IB/RoCE verbs deployment. Verify AWS's supported stack and effective runtime path.*

**GPUDirect RDMA** permits supported network adapters to access GPU memory directly across a compatible PCIe path. This can reduce host staging and CPU overhead. It does not guarantee zero-copy for every buffer, eliminate synchronization, or bypass security. **NVLink/NVSwitch** is GPU-to-GPU communication within supported node architectures, not the inter-node RDMA fabric.

## 2. Data-path comparison

| Path | Data movement | Typical concern |
|---|---|---|
| GPU -> host RAM -> NIC | explicit or implicit staging | extra copies, CPU/memory bandwidth |
| GPU -> HCA via GPUDirect RDMA | supported peer DMA to registered GPU memory | peer mapping, compatibility, sync |
| GPU -> NVLink/NVSwitch -> GPU | supported local GPU interconnect | intra-node topology |
| NIC -> host memory RDMA | HCA directly accesses registered host memory | not proof of GPU-direct transfer |

Host buffers and GPU buffers have different ownership and synchronization semantics. Correct completion handling must consider CUDA stream ordering, network completion and memory visibility. A successful collective is necessary but not sufficient to prove the intended peer-memory path.

## 3. Compatibility matrix

Capture GPU SKU, HCA model, firmware, PCIe generation/topology, IOMMU/ACS settings, kernel, NVIDIA driver, CUDA, NCCL, rdma-core/libfabric, network operator, peer-memory or DMA-BUF support, and container runtime. Support varies across versions and vendors. Avoid globally disabling IOMMU or ACS as a troubleshooting shortcut; involve platform security and vendor documentation.

### Peer-memory versus DMA-BUF

Legacy configurations may depend on `nvidia-peermem`; newer supported Linux/RDMA/NVIDIA combinations may use DMA-BUF registration. These are not universally interchangeable. The correct path depends on the supported driver/kernel/HCA stack and can differ by cloud image. Validate actual memory registration and NCCL/provider diagnostics, not just module presence.

## 4. PCIe, NUMA and GPU placement

A NIC near the target GPU can reduce PCIe hops and avoid cross-socket traffic. Inspect `nvidia-smi topo -m`, `lspci -tv`, `numactl --hardware`, `ibv_devinfo` and `ethtool -i` on approved nodes. Check NVLink, PCIe root complexes, GPU UUIDs, HCA NUMA locality and available bandwidth. Record multi-rail topology and per-rank NIC affinity. Benchmark instead of assuming that the closest HCA is always optimal under congestion.

## 5. Kubernetes and GPU/RDMA integration

Kubernetes must expose GPU resources and, where required, RDMA device-plugin resources. Inspect advertised allocatable keys; vendor/operator naming differs. Align pod GPU allocation, CPU/NUMA topology manager, NIC access, namespace quotas, security context, CNI and NetworkPolicy. Use signed immutable container images, nonprivileged defaults and scoped device permissions. Some deployments require additional capabilities; approve exceptions rather than copying a blanket privileged pod manifest.

```bash
kubectl config current-context
kubectl get nodes -o json | jq -r '.items[] | [.metadata.name, (.status.allocatable|tojson)] | @tsv'
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

## 6. Read-only host diagnostics

```bash
nvidia-smi -L
nvidia-smi topo -m
lspci -tv
numactl --hardware
rdma link show
ibv_devices
ibv_devinfo
lsmod | grep -Ei 'nvidia_peermem|mlx5|ib_core|efa'
dmesg --color=never | grep -Ei 'nvrm|xid|rdma|iommu|dma.buf|mlx5|efa' | tail -n 100
```

Commands may not be installed or meaningful for every provider. `nvidia-peermem` being absent is not automatically a failure on a supported DMA-BUF stack.

## 7. Staging benchmark procedure

1. Pin hardware, firmware, kernel, driver, CUDA, NCCL, RDMA provider and plugin versions.
2. Establish a correct multi-node GPU collective with `nccl-tests` using fixed ranks, sizes and warmup.
3. Record NCCL transport/provider selection from approved diagnostic logs.
4. Verify GPU memory registration path using supported driver/provider instrumentation.
5. Compare measured bus bandwidth, all-reduce latency, CPU utilization and training-step p95 with a controlled host-staging baseline.
6. Repeat with different GPU/HCA affinity and an approved rail-failure scenario.
7. Restore from checkpoint and confirm RPO/RTO and rollback.

**Never** run disruptive benchmark traffic, driver resets, IOMMU modifications or fault injection on production without an approved change.

## 8. Reliability and performance model

Collect GPU-to-HCA PCIe effective bandwidth, NCCL `algbw`/`busbw`, collective p95/p99, rank stalls, link errors, GPU Xid/ECC, checkpoint duration and telemetry freshness. Model N-1 as the loss of the largest relevant rail or node failure domain. Calculate nominal and degraded fabric capacity separately; measured collective throughput is the final performance criterion. Track cost per successful training job, including retries and failed checkpoints.

## 9. Security and operational boundaries

Registered GPU memory and remote memory keys must be scoped to authorized workloads. Protect host driver access, RDMA device nodes, workload identity, switch/network policy, tenant isolation, firmware provenance and diagnostic logs. Device plugin exposure alone is not a tenant security boundary. Require approved driver changes, blast-radius analysis, maintenance windows and rollback to a known-good GPU/RDMA stack.

## 10. Twelve troubleshooting scenarios

1. GPU and HCA detected but peer-memory registration fails.
2. Kernel upgrade breaks legacy `nvidia-peermem`.
3. DMA-BUF unsupported by the active HCA/provider.
4. GPU/HCA behind incompatible PCIe topology.
5. NUMA mismatch reduces collective bandwidth.
6. NCCL falls back to host staging or sockets.
7. NCCL plugin and CUDA/driver versions conflict.
8. IOMMU/ACS configuration affects peer DMA.
9. Kubernetes RDMA plugin advertises no devices.
10. EFA-specific transport assumed to be native IB.
11. Multi-rail imbalance causes rank stragglers.
12. Checkpoint RPO breached after collective hang.

## 11. Ten incident simulations

1. Driver upgrade disables peer-memory path.
2. HCA firmware mismatch after node replacement.
3. CUDA/NCCL incompatibility causes collective failures.
4. PCIe AER errors correlate with GPU/NIC degradation.
5. Wrong NUMA placement increases p99 training time.
6. Unexpected socket transport increases CPU load.
7. RDMA resource exhaustion stalls registration.
8. Rail failure exceeds degraded bandwidth SLO.
9. Unauthorized tenant requests privileged RDMA access.
10. Telemetry outage obscures checkpoint recovery failure.

Capture UTC timeline, ranks, GPU UUIDs, HCA/PCIe topology, driver versions, provider/NCCL evidence, SLO, security impact, containment, rollback and RCA/COA.

## 12. Eight operational runbooks

**RB-047-A — Hardware path:** inventory GPU/HCA/PCIe/NUMA and firmware.

**RB-047-B — Registration:** verify supported peer-memory or DMA-BUF path and errors.

**RB-047-C — NCCL transport:** establish actual provider, fallback behavior and correctness.

**RB-047-D — Kubernetes placement:** check GPU/RDMA allocatable, affinity and tenant policy.

**RB-047-E — Bandwidth regression:** compare per-rank performance and GPU/NIC locality.

**RB-047-F — Rail failure:** preserve checkpoint, isolate faulty path, verify N-1.

**RB-047-G — Security:** audit device access, identities, remote memory exposure and logs.

**RB-047-H — Recovery and RCA:** restore compatible versions, validate SLO and approve rollback.

## 13. Eight hands-on labs

1. Draw the GPU–PCIe–HCA memory access path.
2. Build a driver/CUDA/NCCL/HCA compatibility matrix.
3. Inspect GPU/NIC/NUMA topology on approved staging.
4. Check Kubernetes GPU/RDMA device allocation.
5. Benchmark NCCL and verify transport/memory registration.
6. Compare host staging and GPU-direct performance.
7. Inject synthetic incompatibility, rail loss and telemetry faults offline.
8. Produce a signed 100-point readiness report.

**Companion:** [Lab 047 — GPU Direct RDMA and Communication Gate](../labs/047-gpu-direct-rdma-communication-check.md).

## 14. 100-point assessment

| Domain | Points |
|---|---:|
| GPUDirect semantics and memory-registration architecture | 20 |
| Hardware/driver compatibility and locality | 20 |
| Kubernetes, network and tenant security | 20 |
| SLO, N-1, checkpoint and incident recovery | 20 |
| Executable tests, staging evidence and approvals | 20 |
| **Total** | **100** |

Pass >=90 with zero critical correctness, security, transport, recovery or rollback blocker. An offline synthetic pass cannot certify a live GPUDirect data path.

## References

- [NVIDIA GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/)
- [NVIDIA NCCL User Guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [Linux kernel RDMA subsystem](https://docs.kernel.org/infiniband/index.html)
- [rdma-core](https://github.com/linux-rdma/rdma-core)
- [AWS EFA](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)
- [Kubernetes device plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

**Next: Chapter 048 — Network Congestion and Packet Loss.**
