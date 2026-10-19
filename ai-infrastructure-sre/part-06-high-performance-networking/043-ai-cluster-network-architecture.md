# Chapter 043 — AI Cluster Network Architecture

**AI Infrastructure Engineering & SRE | Part 06 — High Performance Networking | Chapter 043/100**

**Status:** Published tutorial; offline tests, live fabric benchmarking, change approval and independent production acceptance are separate pending gates.

## Learning objectives

Design and operate a production AI cluster network for GPU inference and distributed training. Explain front-end, back-end and storage fabrics; leaf–spine topologies; east–west collective traffic; bandwidth oversubscription; NIC, PCIe, NUMA and GPU locality; AWS EFA, Azure InfiniBand/RDMA and RoCE as distinct implementations; congestion, loss and tail latency; Kubernetes network boundaries; network security, observability and failure recovery.

## 1. Enterprise reference architecture

```text
                users / APIs / model gateway
                            |
                   ingress / load balancer
                            |
             FRONT-END IP NETWORK (control)
       API, DNS, service discovery, registry, IAM
                            |
           Kubernetes control + GPU worker nodes
           |                |                |
      GPU node A        GPU node B       GPU node C
       GPU x8           GPU x8          GPU x8
           |                |                |
       NIC(s)           NIC(s)           NIC(s)
           +----------------+----------------+
                            |
            BACK-END TRAINING FABRIC (data)
           leaf switches / spine switches
           NCCL all-reduce, all-gather, reduce-scatter
                            |
              STORAGE / CHECKPOINT NETWORK
              parallel FS / object gateway / NVMe
                            |
            telemetry, security, capacity and SLO
```

Logical fabrics may share physical infrastructure where validated, but production designs must explicitly account for contention, isolation, throughput and failure domains. AWS EFA and Azure RDMA fabrics are not interchangeable with an ordinary VPC/VNet TCP network.

## 2. Workload traffic taxonomy

| Flow | Typical characteristics | Primary risk |
|---|---|---|
| Inference north–south | request/response, low p95/p99 | gateway or network tail latency |
| Training east–west | high-bandwidth collectives, synchronized ranks | one slow rank stalls all |
| Dataset reads | bursty high-throughput storage traffic | storage congestion |
| Checkpoint writes | large synchronized transfers | bandwidth contention, missed RPO |
| Control plane | DNS, kube-apiserver, registry, IAM | cluster liveness or bootstrap failure |
| Observability | metrics, traces, logs | dropped evidence during incident |

Measure both bandwidth and latency. A nominal 400 Gbps link does not guarantee 400 Gbps of usable NCCL throughput after protocol overhead, topology, contention and application behavior.

## 3. Leaf–spine, Clos and oversubscription

Leaf–spine fabrics provide multiple equal-cost paths between racks. Design decisions include per-rack GPU density, NIC ports per server, switch port speed, uplink capacity, number of spines, ECMP hashing, link redundancy and maintenance domains. Compute **oversubscription ratio = total host-facing bandwidth / total usable uplink bandwidth** for a defined rack/fabric boundary. For example, 8 × 200 Gbps downlinks and 4 × 400 Gbps uplinks yield 1:1 nominal bandwidth before overhead and failure derating. During an uplink failure, the effective ratio changes.

Do not infer nonblocking application performance from 1:1 port counts alone. Collectives are sensitive to topology, packet behavior, NIC queues and synchronized incast.

## 4. GPU, CPU, NIC and NUMA locality

PCIe hierarchy, CPU NUMA domains, NIC affinity, NVLink/NVSwitch and GPU placement affect communication efficiency. Use `nvidia-smi topo -m`, `lspci -tv`, `numactl --hardware` and `ethtool -i` on approved hosts to map topology. GPU Direct RDMA eligibility depends on hardware, driver, kernel, IOMMU and fabric stack; validate it explicitly. A GPU pod with allocated devices does not prove optimal NIC locality.

## 5. Transport decision matrix

| Fabric | Example environment | Key prerequisites |
|---|---|---|
| TCP/IP + Ethernet | control/inference/storage where appropriate | NIC, MTU, routing, firewall |
| AWS EFA | supported EC2 distributed AI | EFA interface, driver, libfabric, AWS OFI NCCL |
| InfiniBand | supported Azure HPC/GPU or on-prem | HCA, subnet manager, verbs stack, NCCL |
| RoCEv2 | supported Ethernet RDMA fabrics | NIC/firmware, loss/congestion controls, ECN/PFC design |
| NVLink/NVSwitch | intra-node supported GPU systems | GPU topology, NCCL version, health |

EFA uses AWS's supported EFA transport/provider stack; do not assume InfiniBand subnet manager or generic RoCE configuration applies. RoCE PFC/ECN tuning requires network-engineering review; blanket enablement can create pause storms or head-of-line blocking.

## 6. Kubernetes and cloud networking

Kubernetes CNI, pod CIDR/IPAM, kube-proxy or eBPF dataplane, NetworkPolicy, service routing, DNS and node NIC topology influence cluster behavior. For EKS, verify VPC CNI subnet IP headroom, EFA resource exposure where required, security groups and AZ placement. For AKS, verify Azure CNI mode, subnet/pod IP allocation, NSGs, RDMA-capable VM SKU and supported drivers. CNI NetworkPolicy is not a replacement for fabric ACLs, cloud security groups or tenant authorization.

## 7. Capacity and resilience math

Let `H` be total host-facing NIC bandwidth and `U` usable fabric uplink bandwidth. The nominal oversubscription is `H/U` for `U>0`. For N-1 link acceptance, recompute `U` after the largest relevant link failure. Measure collective throughput and training step time under both normal and degraded topologies. Reserve bandwidth for checkpoint bursts and observability; do not admit new distributed training solely from nominal link rate.

**Illustrative scenario:** 16 hosts × 200 Gbps = 3,200 Gbps downlink. Eight 400 Gbps uplinks = 3,200 Gbps, nominal 1:1. Losing one uplink leaves 2,800 Gbps, yielding ~1.14:1 before overhead. These are synthetic capacity inputs, not a measured fabric benchmark.

## 8. Network security and trust boundaries

Separate management, training, storage and external ingress policies. Enforce least-privilege network ACL/SG/NSG, approved peer groups, private storage endpoints, workload identity, encrypted application traffic where supported, controlled egress and audit. Fabric-specific RDMA/EFA traffic may require provider-specific rules; do not open all traffic to `0.0.0.0/0` to make NCCL work. Security reviews must address shared GPU tenancy and potential data leakage through logs or untrusted workloads.

## 9. Observability and SLO engineering

| Signal | Why it matters |
|---|---|
| NIC throughput and link utilization | saturation |
| discards/errors/CRC where available | physical/link failure |
| queue drops and ECN/PFC counters where supported | congestion |
| retransmits and transport-specific errors | path degradation |
| NCCL all-reduce bus bandwidth | effective collective performance |
| p95/p99 training step latency | workload SLO |
| per-rank stall time | straggler/deadlock |
| GPU Xid/ECC and utilization | separate compute from network |
| storage checkpoint duration | network/storage contention |
| telemetry age and topology inventory | reliable diagnosis |

Build workload-specific baselines using fixed message sizes, rank counts, GPU models and software versions. A network SLO should include **successful collective completion**, not just interface up/down.

## 10. Read-only host and Kubernetes discovery

```bash
ip -brief link
ip -s link
ethtool -i eth0                  # replace with approved NIC
ethtool -S eth0 | head -n 80    # counters depend on NIC driver
lspci -tv
numactl --hardware
nvidia-smi topo -m
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

For an approved EFA node: `fi_info -p efa` (when installed). For supported InfiniBand hosts: `ibv_devinfo` (when installed). Do not assume either utility exists or that its success proves NCCL is using the intended fabric.

## 11. Eight hands-on labs

1. Draw rack/zone GPU, NIC, switch and storage failure-domain topology.
2. Inventory NIC speeds, PCIe/NUMA and GPU locality.
3. Compute normal and N-1 oversubscription using synthetic topology data.
4. Validate AWS EFA versus Azure RDMA support matrices.
5. Inspect Kubernetes GPU scheduling, IPAM and network isolation.
6. Benchmark `nccl-tests` all-reduce on approved disposable staging GPUs.
7. Simulate congestion, uplink loss and telemetry staleness **offline**.
8. Document rollback, cost, incident timeline and network acceptance evidence.

[Lab 043 — AI Cluster Network Architecture Gate](../labs/043-ai-cluster-network-architecture-check.md) provides an offline Python 3.11+ fail-closed policy evaluator and pytest suite.

## 12. Twelve troubleshooting scenarios

1. Leaf uplink oversubscription during training.
2. One failed spine path causes tail-latency regression.
3. Wrong NIC/NUMA affinity reduces NCCL throughput.
4. MTU mismatch causes intermittent storage transfers.
5. EFA interface present but libfabric provider unavailable.
6. Azure GPU SKU lacks expected RDMA capability.
7. RoCE pause storm or ECN/PFC misconfiguration.
8. Kubernetes CNI IP exhaustion blocks rank placement.
9. Cloud security-group peer rule blocks EFA traffic.
10. Checkpoint burst competes with all-reduce.
11. GPU Xid misdiagnosed as a network failure.
12. Telemetry gaps hide packet drops during incident.

## 13. Ten production incident simulations

1. Spine uplink failure.
2. Leaf switch overload during GPU scale-out.
3. NIC firmware/driver regression.
4. NUMA mapping change after node replacement.
5. NCCL fallback to TCP.
6. RDMA/EFA peer security drift.
7. RoCE congestion-control regression.
8. Storage checkpoint incast.
9. Cross-zone rank placement degradation.
10. Monitoring pipeline loss during collective hang.

Run disruptive exercises only in authorized staging. Capture UTC timeline, affected racks/ranks, topology, link counters, NCCL logs, workload SLO, containment, rollback and RCA/COA.

## 14. Eight operational runbooks

**RB-043-A — Topology inventory:** verify host, NIC, rack, switch, zone and storage mapping.

**RB-043-B — Bandwidth/oversubscription:** calculate normal/N-1 uplink capacity and compare actual traffic.

**RB-043-C — GPU/NIC locality:** inspect PCIe, NUMA, NVLink, HCA/NIC and NCCL affinity.

**RB-043-D — Fabric discovery:** validate EFA/libfabric, InfiniBand or RoCE prerequisites by provider.

**RB-043-E — Congestion:** inspect drops, ECN/PFC counters where available, tail latency and safe tuning.

**RB-043-F — Kubernetes networking:** diagnose CNI IPAM, DNS, SG/NSG and NetworkPolicy boundaries.

**RB-043-G — Collective recovery:** stop unsafe reruns, preserve checkpoint, isolate bad ranks, retest NCCL.

**RB-043-H — RCA/acceptance:** record SLO, N-1, budget, approved rollback and network-owner signoff.

## 15. 100-point assessment

| Domain | Points |
|---|---:|
| Architecture and topology | 20 |
| Transport, NIC/NUMA and GPU integration | 20 |
| Security, capacity and resilience | 20 |
| Troubleshooting, incident response and runbooks | 20 |
| Executable tests and live staging evidence | 20 |
| **Total** | **100** |

Pass >=90 with no critical fabric reachability, tenant isolation, N-1, workload correctness or rollback blocker.

## References

- [NVIDIA NCCL documentation](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [AWS Elastic Fabric Adapter](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)
- [AWS OFI NCCL](https://github.com/aws/aws-ofi-nccl)
- [Azure HPC VM networking](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes-hpc)
- [Kubernetes networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [NVIDIA GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/)

**Next: Chapter 044 — RDMA and InfiniBand Fundamentals.**
