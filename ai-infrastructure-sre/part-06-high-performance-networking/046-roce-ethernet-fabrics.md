# Chapter 046 — RoCE and Ethernet Fabrics

**AI Infrastructure Engineering & SRE | Part 06 — High Performance Networking | Chapter 046/100**

> **Release state:** Curriculum publication only. Real RoCE fabric configuration, packet-loss injection, benchmark validation, security review and production acceptance remain pending.

## Learning objectives

Design, validate and operate an Ethernet-based RDMA fabric for GPU distributed training. Explain RoCEv2 encapsulation, PFC, ECN, DCQCN, QoS, queue behavior, leaf–spine topology, congestion and deadlock risks, GPU/NIC locality, Kubernetes device exposure, security boundaries, N-1 capacity, observability, incident recovery and safe change management.

**Prerequisites:** Chapter 043 network architecture, Chapter 044 RDMA/InfiniBand, CUDA/NCCL fundamentals, Linux NIC diagnostics, Kubernetes GPU administration.

## 1. Production architecture

```text
       scheduler / Kubernetes / Slurm / workload SLO
                          |
      +-------------------+-------------------+
      |                                       |
   GPU worker A                           GPU worker B
   GPU--PCIe--HCA                        GPU--PCIe--HCA
   NCCL/UCX/verbs                        NCCL/UCX/verbs
      |                                       |
   RoCEv2 UDP/IP                          RoCEv2 UDP/IP
      +-------------------+-------------------+
                          |
                 Ethernet leaf switches
                  ECN / PFC / QoS queues
                          |
                    Spine switches
                          |
            Storage / checkpoint endpoints
                          |
          Telemetry, incident, FinOps and DR
```

Separate management/control traffic, RDMA collective traffic and storage traffic logically or physically according to measured contention. A dedicated RoCE priority does not by itself guarantee isolation or eliminate congestion. Validate switch buffer behavior and host NIC queues across the **whole path**.

## 2. RDMA transport comparison

| Transport | Fabric | Operational requirement |
|---|---|---|
| RoCEv1 | Ethernet Layer 2 | limited L2 domain, specialized operations |
| RoCEv2 | UDP/IP Ethernet | routable IP, ECN/PFC/QoS design |
| Native InfiniBand | InfiniBand switches | subnet manager, partitions, verbs |
| AWS EFA | AWS EFA fabric | libfabric EFA provider, AWS OFI NCCL |
| TCP | Ethernet/IP | normal congestion control, higher CPU/latency profile |

**Important:** EFA is not RoCE; native InfiniBand subnet-manager procedures do not apply to RoCEv2. A verbs device may be present without a usable end-to-end RoCE path.

## 3. Ethernet and RoCEv2 packet path

RoCEv2 uses IP/UDP encapsulation for RDMA packets. Its loss sensitivity and NIC transport behavior demand a carefully engineered congestion strategy. Understand ingress classification, DSCP/802.1p mappings where applicable, switch queue selection, ECN marking thresholds, PFC pause thresholds and NIC rate control. ECN can signal congestion without dropping packets; DCQCN is one commonly used RoCE congestion-control approach where supported. PFC can avoid loss for selected priorities but can also cause pause propagation, head-of-line blocking and deadlock. Never enable PFC across all traffic classes by default.

Document the precise switch ASIC, NOS, NIC firmware, driver, DCB configuration and provider-approved reference design. No generic threshold or buffer size is safe for every fabric.

## 4. Fabric topology, bandwidth and N-1

Leaf–spine Clos fabrics offer multiple equal-cost paths. Validate ECMP distribution, hashing entropy, rack-to-rack path symmetry, switch failure domains and oversubscription.

```text
nominal_ratio = aggregate_host_facing_Gbps / aggregate_usable_uplink_Gbps
degraded_ratio = aggregate_host_facing_Gbps /
                 (aggregate_usable_uplink_Gbps - largest_failed_domain_Gbps)
```

Example (synthetic): 8 hosts × 200 Gbps = 1,600 Gbps; 5 uplinks × 400 Gbps = 2,000 Gbps; after losing one 400 Gbps uplink, ratio becomes 1.0:1 before protocol overhead. Recompute with **real** link aggregation, routing and topology rather than assuming every surviving link remains usable.

## 5. GPU/NIC/NUMA and GPUDirect

Map GPU, HCA, PCIe root complexes, NUMA domains, NVLink/NVSwitch and container device assignment. GPUDirect RDMA support is conditional on GPU/HCA generations, drivers, IOMMU, kernel and DMA-BUF or peer-memory integration. GPU and HCA presence alone does not prove zero-copy GPU-memory transfers. Verify actual NCCL transport and application throughput.

```bash
nvidia-smi topo -m
lspci -tv
numactl --hardware
rdma link show
ibv_devinfo
ethtool -i APPROVED_NIC
```

## 6. Network policy and security

RoCEv2 network reachability must be restricted by authorized network segments, switch ACLs, host firewall policies and tenant admission controls. Kubernetes NetworkPolicy and cloud SG/NSG are not replacements for NIC/ToR isolation. Require least-privilege workload identity, approved device plugin exposure, no blanket privileged containers, management-plane RBAC, signed switch/NIC firmware, audit trails and explicit data-residency controls. Treat remote memory keys, fabric management credentials and host debug logs as sensitive.

## 7. Kubernetes and operator integration

Validate Kubernetes node allocatable RDMA devices, GPU device plugin, NIC resource plugins, node affinity, topology manager policies, taints, IPAM, CNI and pod security. Exact resource names vary by device plugin and installation: discover before writing manifests.

```bash
kubectl get nodes -o json | jq -r '.items[] | [.metadata.name, (.status.allocatable | tojson)] | @tsv'
kubectl get pods -A -o wide
kubectl get daemonsets -A
kubectl get events -A --sort-by=.lastTimestamp
```

Coordinate with the network team before changing PFC, ECN, QoS or firmware. Apply versioned IaC, canary switches/nodes, defined rollback and post-change NCCL baselines.

## 8. Observability and SLOs

Track interface throughput, CRC/FEC errors, drops, ECN marks, PFC pause duration/events by priority, switch queue occupancy, NIC congestion-control counters, RDMA retransmission/errors where exposed, NCCL all-reduce bandwidth, per-rank stalls, GPU health, checkpoint time and telemetry freshness. Counter names differ by ASIC/NIC. A PFC counter increase is not automatically an incident: correlate pause duration, queue occupancy and workload impact.

Define SLOs for successful collective completion, p95 training-step latency, minimum measured NCCL bandwidth, recovery after link loss, checkpoint RPO/RTO and telemetry availability. Record baseline topology, GPU SKU, NCCL version and message sizes.

## 9. Read-only troubleshooting commands

```bash
ip -s link show dev APPROVED_NIC
ethtool -S APPROVED_NIC | grep -Ei 'ecn|pfc|pause|drop|discard|crc|error'
ethtool -i APPROVED_NIC
rdma link show
ibv_devices
ibv_devinfo
ss -s
nvidia-smi topo -m
journalctl -k -b --no-pager | grep -Ei 'mlx5|rdma|nvrm|xid' | tail -n 80
```

Do not run bandwidth saturation, packet drops, interface resets or switch tuning in production without an approved change.

## 10. Eight hands-on labs

1. Draw a leaf–spine RoCEv2 topology with GPU/HCA/NUMA and failure domains.
2. Inventory NIC/driver, link speed, MTU, queue and RDMA device state.
3. Compare PFC/ECN/DCQCN semantics and map lossless priorities.
4. Calculate normal and largest-domain-failure oversubscription.
5. Inspect Kubernetes RDMA/GPU allocatable and tenant policy.
6. Run approved `ib_write_bw` and NCCL collective benchmarks in isolated staging.
7. Inject congestion, PFC misconfiguration, stale telemetry and N-1 failures **offline**.
8. Produce evidence, runbooks and a 100-point readiness assessment.

[Lab 046 — RoCE and Ethernet Fabrics Gate](../labs/046-roce-ethernet-fabrics-check.md) contains a fail-closed Python 3.11+ evaluator and pytest suite.

## 11. Twelve troubleshooting scenarios

1. RoCE device appears but peers cannot exchange RDMA traffic.
2. Incorrect DSCP/PCP to priority mapping.
3. ECN disabled or marking threshold unsuitable.
4. PFC pause storm spreads across racks.
5. Head-of-line blocking affects storage and control traffic.
6. Switch queue buffer exhaustion causes packet drops.
7. MTU mismatch between HCA and leaf switch.
8. NIC firmware regression changes congestion behavior.
9. ECMP imbalance overloads a subset of uplinks.
10. GPU–HCA NUMA mismatch degrades NCCL throughput.
11. Kubernetes RDMA device plugin advertises no allocatable resources.
12. N-1 link failure breaches collective latency SLO.

## 12. Ten incident simulations

1. PFC pause storm during synchronized all-reduce.
2. ECN marking stops after switch configuration drift.
3. QoS priority mapping breaks after a firmware update.
4. Spine uplink fails during peak GPU training.
5. Leaf switch buffer pressure causes packet drops.
6. One GPU node falls back from RoCE to sockets.
7. Misplaced rank increases cross-rack traffic.
8. Checkpoint burst causes storage/RDMA contention.
9. Unauthorized tenant gains access to RDMA network.
10. Monitoring pipeline fails during congestion.

Each drill captures UTC timeline, blast radius, queue/ECN/PFC counters, GPU/NCCL evidence, SLO impact, rollback and RCA/COA. Perform disruptive drills only in approved staging.

## 13. Eight operational runbooks

**RB-046-A — Fabric discovery:** verify NIC, RDMA device, link, IP, VLAN and routing.

**RB-046-B — QoS mapping:** inspect DSCP/PCP, traffic class, priority and switch queue mapping.

**RB-046-C — ECN/DCQCN:** compare configured markings, NIC reactions and workload throughput.

**RB-046-D — PFC storm:** isolate affected priority/path, protect control traffic and coordinate network recovery.

**RB-046-E — Packet loss:** inspect switch buffers, drops, MTU, CRC/FEC and oversubscription.

**RB-046-F — GPU/NCCL:** inspect HCA locality, device plugin, transport and rank mapping.

**RB-046-G — N-1 recovery:** preserve checkpoints, reroute/restart in approved staging and validate SLO.

**RB-046-H — Change/incident closure:** restore baseline, review security, budget, owner and rollback.

## 14. 100-point assessment

| Domain | Points |
|---|---:|
| RoCEv2 and Ethernet architecture | 20 |
| PFC/ECN/DCQCN and QoS | 20 |
| GPU/NIC, Kubernetes and security | 20 |
| Reliability, troubleshooting and incident response | 20 |
| Automated tests, live staging evidence and signoff | 20 |
| **Total** | **100** |

Pass >=90 with zero critical congestion, isolation, N-1, collective correctness, checkpoint or rollback blocker. Offline synthetic acceptance is not live production approval.

## References

- [NVIDIA Ethernet software documentation](https://docs.nvidia.com/networking/)
- [NVIDIA NCCL user guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [Linux RDMA subsystem](https://docs.kernel.org/infiniband/index.html)
- [rdma-core](https://github.com/linux-rdma/rdma-core)
- [Kubernetes device plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [NVIDIA GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/)

**Next: Chapter 047 — GPU Direct RDMA and Communication.**
