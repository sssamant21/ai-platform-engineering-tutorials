# Chapter 044 — RDMA and InfiniBand Fundamentals

**AI Infrastructure Engineering & SRE · Part 06 — High Performance Networking · Chapter 044/100**

> **Publication vs acceptance:** This tutorial and its synthetic lab can be published without proving a live RDMA fabric. Device-level benchmarks, failure injection, security approval and production readiness require separately recorded evidence.

## 1. Learning objectives

Explain RDMA semantics and InfiniBand architecture; distinguish host memory registration, protection domains, queue pairs, completion queues, work requests and completion events; compare send/receive and one-sided read/write; understand reliable connected (RC) versus unreliable datagram (UD) transports; operate HCAs, subnet managers and partitions; troubleshoot GPU-to-GPU collectives across nodes; and build a fail-closed SRE admission gate.

**Prerequisites:** Chapters 021–023 (NVLink/NCCL), 035–037 (AWS GPU/EFA), 043 (cluster fabric), Linux, Kubernetes, PCIe/NUMA and basic routing.

## 2. Enterprise reference architecture

```text
               Kubernetes / Slurm scheduler + workload identity
                                |
                     distributed training job
                   rank 0             rank N
                      |                 |
                 CUDA + NCCL       CUDA + NCCL
                      |                 |
                   verbs/UCX          verbs/UCX
                      |                 |
              GPU -- PCIe -- HCA   GPU -- PCIe -- HCA
                      |                 |
                   IB leaf switches / fabric
                           |
                IB spine / alternate paths
                           |
             subnet manager + fabric telemetry
                           |
           checkpoint storage / recovery services
```

Control-plane IP and data-plane RDMA may share NICs or use distinct fabrics. A subnet manager is essential for native InfiniBand fabric configuration; **AWS EFA is not native InfiniBand** and does not require an InfiniBand subnet manager. RoCEv2 uses Ethernet/IP and different congestion-control requirements.

## 3. RDMA fundamentals

Remote Direct Memory Access enables NIC-driven transfer involving registered application memory with reduced CPU participation and fewer kernel data-path transitions. It does not mean all software operations are zero-copy, nor does it bypass access controls. Memory registration pins or otherwise prepares memory for HCA DMA and creates local/remote access keys subject to protection and transport rules.

| Concept | Purpose | Failure symptom |
|---|---|---|
| HCA | host channel adapter exposing RDMA capabilities | no RDMA device |
| Protection domain (PD) | isolates verbs objects and access | permission/access error |
| Memory region (MR) | registered memory with lkey/rkey | local protection error |
| Queue pair (QP) | send/receive work queues | QP error, timeout |
| Completion queue (CQ) | reports completion status | stalled completion polling |
| Work request (WR) | send, receive, read, write or atomic | rejected or timed-out operation |
| GID/LID | fabric addressing as applicable | unreachable peer |
| Subnet manager (SM) | discovers/configures native IB fabric | missing paths or inactive ports |
| Partition key (P_Key) | native IB partition membership | communication isolation failure |

**Operations:** two-sided SEND/RECV require receiver-posted buffers; one-sided RDMA READ/WRITE require appropriate remote access permissions and valid memory keys. Ordering, visibility and completion semantics vary by operation and transport; application synchronization is still required. Avoid interpreting local completion as proof of durable remote storage.

## 4. InfiniBand fabric components

An InfiniBand deployment has HCAs, switches, links, subnet management, LIDs, routing, virtual lanes and service levels. Subnet managers establish fabric paths; partitioning can restrict membership, but it does not replace workload identity, Kubernetes RBAC or host security. Link speed, lane count, FEC, cable quality and firmware influence effective bandwidth and errors. Record HCA generation and switch topology rather than assuming a speed from product branding.

**Topology:** leaf–spine, rail-optimized and multi-rail fabrics have different path diversity and congestion profiles. Dual rails only improve resilience when the workload/NCCL stack can actually use both and the failure domains are independent.

## 5. RDMA transport comparison

| Transport | Network | Key consideration |
|---|---|---|
| Native InfiniBand | IB fabric with subnet manager | verbs, LID/GID, partitions and fabric health |
| RoCEv2 | routable UDP/IP over Ethernet | ECN/PFC design, queue management, loss behavior |
| iWARP | TCP-based RDMA | device/provider support and TCP characteristics |
| AWS EFA | AWS-specific fabric with libfabric | EFA device/provider and AWS OFI NCCL; not IB SM |

**Do not** apply IB subnet-manager runbooks to EFA, or indiscriminately enable PFC on a RoCE fabric. Cloud GPU SKU, image, driver, kernel and container support must be verified per deployment.

## 6. GPU Direct RDMA and topology

GPUDirect RDMA can allow supported HCAs to access supported GPU memory without a host staging copy. Support depends on GPU/HCA generation, PCIe topology, NVIDIA driver, peer-memory or DMA-BUF path, kernel, IOMMU configuration and runtime. Validate `nvidia-smi topo -m` alongside `lspci -tv` and actual NCCL logs. The presence of an HCA and a GPU does not prove GPUDirect is active.

On supported NVIDIA systems, NCCL may use IB verbs, socket transport or a provider plugin. Inspect effective transport and benchmark rather than inferring it from environment variables alone.

## 7. Kubernetes and container runtime integration

RDMA workloads need approved device exposure, node labels/taints, appropriate RDMA shared device plugin or vendor integration, network attachment and permissions according to the chosen CNI and operator. Kubernetes extended resource names and device-plugin settings are **deployment-specific**; inspect live allocatable resources before writing a pod resource request. Avoid blanket `privileged: true`, hostNetwork or hostPath prescriptions. GPU Operator and network operator versions must be jointly validated.

**Read-only discovery:**

```bash
kubectl get nodes -o json | jq -r '.items[] | [.metadata.name, (.status.allocatable | tojson)] | @tsv'
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl get daemonsets -A
```

## 8. Host and fabric diagnostics

Run only on authorized nodes; tools may be absent:

```bash
lspci -nn | grep -Ei 'Mellanox|NVIDIA|InfiniBand|Ethernet'
ibv_devices
ibv_devinfo
ibstat
rdma link show
ip -s link
nvidia-smi topo -m
dmesg --color=never | grep -Ei 'mlx5|rdma|ib_|pcie|nvrm' | tail -n 100
```

Fabric administrator tools (e.g. `sminfo`, `ibnetdiscover`, `perfquery`) require the appropriate installed packages, permissions and operational approval. Never run fabric-wide configuration or reset commands in production as a diagnostic shortcut.

## 9. Safe baseline benchmarks

Use a dedicated staging window and fixed GPU/HCA/firmware/driver/NCCL versions. Compare `ib_write_bw`, `ib_read_bw`, `ib_send_lat` and `nccl-tests` where installed and supported; ensure both peers and port/device selections are correct. These tests may generate substantial traffic, so coordinate with the network owner.

Record message size, iteration count, QP count, ranks, GPU mapping, CPU affinity, transport, measured bandwidth, p50/p95/p99 latency, errors and warm-up. A raw verbs microbenchmark is not a replacement for end-to-end training step time.

## 10. Reliability SLOs and error budgets

Use service-specific SLOs: successful collective completion ratio, p95/p99 collective duration, bandwidth floor relative to measured baseline, queue stalls, link errors, GPU-to-NIC locality, checkpoint RPO/RTO and telemetry freshness. Treat error counters as vendor- and driver-specific; correlate with actual workload impact. Plan for largest rail/switch failure and recompute degraded usable bandwidth and route diversity.

**Synthetic planning example:** 8 hosts × 200 Gbps host-facing = 1,600 Gbps; 5 × 400 Gbps uplinks = 2,000 Gbps; loss of one uplink leaves 1,600 Gbps, so degraded oversubscription is 1:1 before overhead. This is a teaching calculation, not a real throughput guarantee.

## 11. Security and compliance

Apply least-privilege workload identity, Kubernetes namespace isolation, device access controls, InfiniBand P_Key partitions where appropriate, management-plane ACLs, switch firmware integrity and auditable change control. RDMA memory registration and remote access keys are sensitive: do not log them or expose them across trust boundaries. Restrict fabric management tools and subnet-manager privileges. Review multi-tenant threat boundaries independently of GPU partitioning.

## 12. Twelve troubleshooting scenarios

1. HCA visible in PCIe but no verbs device.
2. InfiniBand port Down or not Active.
3. Missing or conflicting subnet manager.
4. P_Key partition mismatch prevents peers.
5. GID/LID or path resolution failure.
6. Queue pair transitions to error after link flap.
7. CQ completions stall or time out.
8. Incorrect MR access keys cause protection failures.
9. PCIe/NUMA mismatch reduces effective throughput.
10. GPUDirect RDMA silently falls back to host copies.
11. NCCL transport falls back to sockets or hangs.
12. Multi-rail failure breaks N-1 throughput SLO.

## 13. Ten incident simulations

1. IB leaf switch uplink loss.
2. Subnet-manager failover disruption.
3. HCA firmware regression.
4. Wrong partition membership after tenant migration.
5. Registered-memory exhaustion.
6. QP/CQ resource exhaustion under scale-out.
7. GPUDirect peer-memory incompatibility.
8. NCCL hang with one degraded rank.
9. Storage checkpoint incast congests shared fabric.
10. Telemetry outage masks packet/error counter changes.

For each, capture UTC timeline, rank mapping, HCA and switch inventory, link states, verbs/NCCL error evidence, blast radius, checkpoint status, containment, rollback and RCA/COA. Run destructive fault injection only in approved staging.

## 14. Eight operational runbooks

**RB-044-A — HCA discovery:** inspect PCIe, driver, RDMA devices, firmware and node ownership.

**RB-044-B — Fabric membership:** verify native IB SM, port state, LID/GID, partitions and routes.

**RB-044-C — QP/MR failure:** correlate verbs status, keys, PD permissions and resource exhaustion.

**RB-044-D — GPU locality:** inspect GPU–HCA PCIe/NUMA, GPUDirect support and measured NCCL path.

**RB-044-E — Bandwidth regression:** compare baselines, link errors, queue stalls and oversubscription.

**RB-044-F — Multi-rail outage:** isolate failed rail, verify alternate paths, preserve checkpoint and test N-1.

**RB-044-G — Kubernetes RDMA:** validate device-plugin allocation, network attachments, RBAC and tenant boundary.

**RB-044-H — Incident closure:** restore SLO, confirm error-budget impact, evidence, change approval and rollback.

## 15. Eight hands-on labs

1. Map HCA, GPU, PCIe, NUMA, IB switch and failure-domain topology.
2. Inventory `ibv_devices`, `ibstat` and link state on disposable staging hosts.
3. Trace MR, PD, QP, CQ and WR lifecycle in a diagram or supported verbs example.
4. Compare native IB, RoCE, EFA and TCP transport decision criteria.
5. Inspect RDMA device resources and workload placement in Kubernetes.
6. Benchmark verbs bandwidth/latency and NCCL on an approved isolated fabric.
7. Simulate rail loss, QP errors and stale telemetry using the offline gate.
8. Produce an SRE readiness report with security review and 100-point scorecard.

**Companion:** [Lab 044 — RDMA and InfiniBand Fundamentals Gate](../labs/044-rdma-infiniband-fundamentals-check.md).

## 16. 100-point final assessment

| Domain | Points |
|---|---:|
| RDMA semantics, verbs objects and transport | 20 |
| InfiniBand topology, SM, partitions and resilience | 20 |
| GPU/NIC locality, Kubernetes integration and security | 20 |
| Incident response, troubleshooting and SLOs | 20 |
| Automated tests, live staging evidence and signoff | 20 |
| **Total** | **100** |

Pass >=90 and zero critical fabric reachability, tenant isolation, GPUDirect compatibility, N-1, checkpoint or rollback blocker. A passing synthetic gate alone does not confer production approval.

## 17. Official references

- [NVIDIA NCCL User Guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [Linux kernel RDMA subsystem](https://docs.kernel.org/infiniband/index.html)
- [rdma-core](https://github.com/linux-rdma/rdma-core)
- [NVIDIA Networking documentation](https://docs.nvidia.com/networking/)
- [NVIDIA GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/)
- [AWS EFA](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)
- [Kubernetes device plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

**Next: Chapter 045 — AWS EFA for Distributed AI.**
