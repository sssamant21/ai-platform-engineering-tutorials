# Chapter 048 — Network Congestion and Packet Loss

**AI Infrastructure Engineering & SRE · Part 06 — High Performance Networking · Chapter 048/100**

> **Safety and scope:** The following commands are read-only unless explicitly marked staging-only. Do not modify production switch QoS, PFC, ECN, MTU, routing, NIC firmware or traffic generators without a reviewed change and rollback. Published curriculum is not evidence of a tested production fabric.

## Outcomes and prerequisites

Diagnose GPU training stalls, packet drops, tail latency and storage incast across Ethernet/RoCE, native InfiniBand and AWS EFA. Distinguish congestion from physical loss, NIC/GPU defects, scheduler skew and NCCL software issues. Build an SRE telemetry, capacity, N-1 and incident-response framework. Prerequisites: Chapters 043–047, Linux networking, NCCL fundamentals, Kubernetes administration.

## 1. Reference architecture and failure domains

```text
         GPU training ranks / NCCL collectives
             |       |       |       |
          GPU+NIC  GPU+NIC  GPU+NIC  GPU+NIC
             |       |       |       |
            leaf switches + NIC queues
                |         |
            spine A     spine B
                |         |
        parallel storage / checkpoints
                |
     counters + traces + GPU metrics + SLO
```

Training all-reduce and all-gather synchronize ranks; a single slow flow or rank can hold the entire step. Dataset fan-out and checkpoint fan-in/fan-out can produce synchronized bursts. ECMP hashing can concentrate flows on a subset of links even if aggregate bandwidth is below nominal capacity. Distinguish link failures, queue congestion, fabric routing, GPU Xid, host CPU starvation and storage bottlenecks.

## 2. Congestion versus packet loss

**Congestion:** offered traffic temporarily exceeds available service rate at a link, queue or endpoint. Symptoms: queue occupancy, tail latency, ECN marking, PFC pause, drops, retransmits, application stalls.

**Physical/link loss:** CRC/FEC, flaps, optic/cable errors, driver resets or NIC failures. These may cause retransmits even when utilization is low.

**Policy-induced loss:** ACL/SG/NSG/NetworkPolicy, MTU mismatch, routing or firewall rules. This may resemble packet loss but requires different remediation.

| Signal | Interpretation | Caveat |
|---|---|---|
| Interface utilization | sustained saturation | averages hide microbursts |
| Queue depth / discards | buffering and loss | ASIC counter semantics differ |
| ECN marks | early congestion indication | marking alone not necessarily bad |
| PFC pause duration | priority backpressure | pause storms can spread |
| CRC/FEC errors | physical quality issues | correlate with link state |
| TCP retransmits | end-to-end loss/reordering | not native RDMA transport metric |
| NCCL p95/p99 step time | user-visible impact | also affected by compute |
| GPU Xid/ECC | GPU hardware failure | may be root cause, not network |

## 3. RoCEv2: PFC, ECN and DCQCN

RoCEv2 runs over routable Ethernet/IP. QoS mappings and per-priority queues must be consistent across NICs, leafs and spines. ECN marks packets before drops under configured conditions; supported NIC congestion control such as DCQCN responds to feedback. PFC can pause a selected priority to reduce loss but can create head-of-line blocking, unfairness or deadlock. **Never globally enable PFC or copy vendor thresholds blindly.** Use a vendor-approved, topology-specific design and verify pause duration, queue occupancy and application SLO.

## 4. Native InfiniBand and AWS EFA

InfiniBand has its own fabric routing, flow control, counters and subnet-management model. Use supported verbs and fabric-admin telemetry. AWS EFA uses a provider-specific libfabric transport and AWS OFI NCCL integration; **do not** diagnose EFA with native InfiniBand subnet-manager assumptions or treat EFA as RoCE. In each environment verify actual NCCL transport and collect provider-appropriate counters.

## 5. Oversubscription and N-1 capacity

Compute host-facing aggregate demand `H`, usable uplink capacity `U`, and lost failure-domain capacity `F`:

```text
normal_ratio = H / U
degraded_ratio = H / (U - F)
headroom = (U - F) - admitted_peak_traffic
```

Require `U > F`; include protocol overhead and workload-specific burst headroom. Example (synthetic): 8 × 200 Gbps = 1,600 Gbps downlinks, 5 × 400 Gbps = 2,000 Gbps uplinks, one uplink lost leaves 1,600 Gbps. Nominal ratio 0.8:1 and degraded 1:1; this does **not** guarantee zero drops or a passing NCCL SLO.

## 6. Safe investigation workflow

1. Declare severity, incident commander and change freeze where needed.
2. Identify impacted jobs, ranks, GPU node groups, racks, AZs, model versions and SLO window.
3. Compare current collective latency and throughput against matched baseline.
4. Inspect GPU Xid/ECC, CPU pressure, disk/checkpoint throughput and scheduling skew.
5. Inspect NIC link/errors, fabric queue counters, ECN/PFC and per-path utilization.
6. Check recent switch/NIC/CNI/security-policy changes.
7. Correlate timestamps and isolate whether loss is physical, congestion or policy.
8. Apply reversible mitigation (admission control, rate limit, job placement or rollback) with owner approval.
9. Verify recovery, checkpoint integrity, error budget and postmortem evidence.

### Read-only Linux and Kubernetes commands

```bash
ip -brief link
ip -s link show dev APPROVED_NIC
ethtool -i APPROVED_NIC
ethtool -S APPROVED_NIC | grep -Ei 'drop|discard|crc|fec|ecn|pfc|pause|error'
ss -s
nvidia-smi --query-gpu=uuid,utilization.gpu,memory.used --format=csv
nvidia-smi topo -m
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

Counters vary by driver and switch ASIC. A missing counter must be reported as **unknown**, not zero. Collect host and fabric timestamps in UTC.

## 7. Benchmarking and evidence

In approved isolated staging, pin GPU/NIC SKU, driver, firmware, NCCL, plugin, message sizes, rank counts, CPU affinity and topology. Run `nccl-tests` for representative all-reduce sizes and application-level step benchmarks. Record `algbw`, `busbw`, per-rank p95/p99, retransmission/queue counters, checkpoint duration and cost per successful job. Network load tools and fault injection can saturate shared fabrics: do not run them in production.

## 8. SLO, alerting and governance

Define successful collective completion, training-step p95, measured bandwidth floor, queue age, checkpoint RPO/RTO, telemetry freshness and incident recovery targets. Alerts should correlate **user-visible degradation plus network evidence**, rather than page on every ECN mark. Preserve on-call ownership, switch/network escalation, rollback artifact, error-budget decision, workload impact and capacity forecast.

## 9. Twelve troubleshooting scenarios

1. RoCE PFC pause storm spreads across GPU racks.
2. ECN thresholds changed, causing late congestion feedback.
3. Leaf queue drops during synchronized all-reduce.
4. ECMP imbalance overloads a single spine uplink.
5. MTU mismatch silently breaks large payloads.
6. CRC/FEC spike after optics replacement.
7. Kubernetes CNI/IPAM exhaustion blocks rank placement.
8. EFA libfabric provider fallback misdiagnosed as packet loss.
9. InfiniBand link degradation with healthy Ethernet control plane.
10. Checkpoint incast competes with training collectives.
11. GPU Xid stalls a rank despite healthy NIC counters.
12. Telemetry gaps mask a congestion-related SLO breach.

## 10. Ten production-style incident drills

**GD-01:** uplink saturation. **GD-02:** ECN misconfiguration. **GD-03:** PFC pause storm. **GD-04:** physical CRC errors. **GD-05:** ECMP hot path. **GD-06:** MTU mismatch. **GD-07:** NCCL fallback. **GD-08:** checkpoint burst contention. **GD-09:** zone/rail loss violates N-1. **GD-10:** telemetry outage during collective timeout.

For every drill capture UTC start/end, incident commander, topology, ranks, SLO, counters, checkpoint state, mitigation, rollback and RCA/COA. Simulate faults offline first; inject live faults only in approved staging.

## 11. Eight runbooks

**RB-048-A — Saturation:** identify hot queue/link and throttle admissions safely.

**RB-048-B — PFC/ECN:** inspect per-priority counters, verify mapping, restore approved configuration.

**RB-048-C — Physical loss:** investigate CRC/FEC, cable/optic, firmware and link flaps.

**RB-048-D — ECMP/route:** identify asymmetric or hot paths; coordinate fabric team.

**RB-048-E — NCCL:** distinguish provider fallback, rank stalls and GPU failures.

**RB-048-F — Kubernetes:** inspect IPAM, CNI, NetworkPolicy and node placement.

**RB-048-G — N-1 recovery:** preserve checkpoints, fail over to validated paths, test degraded SLO.

**RB-048-H — Postincident:** reconcile error budget, capacity, evidence, change approvals and FinOps.

## 12. Eight hands-on labs

1. Draw GPU/NIC/leaf/spine/storage failure domains.
2. Inventory host and switch counters with read-only commands.
3. Compute normal/N-1 oversubscription and burst headroom.
4. Distinguish physical, congestion and policy loss using sample evidence.
5. Interpret ECN/PFC/queue counters for RoCE without blanket tuning.
6. Benchmark NCCL and checkpoint traffic on approved staging.
7. Run the companion offline failure-gate and incident tests.
8. Complete a 100-point readiness scorecard and rollback review.

[Lab 048 — Network Congestion and Packet Loss Gate](../labs/048-network-congestion-packet-loss-check.md) provides a runnable offline Python evaluator and pytest cases.

## 13. Enterprise acceptance rubric

| Domain | Points |
|---|---:|
| Topology, transport and capacity | 20 |
| Queueing, PFC/ECN and physical loss diagnosis | 20 |
| GPU/NCCL, Kubernetes and security | 20 |
| SLO, incidents, N-1 and checkpoint recovery | 20 |
| Automated tests, staging evidence and signoff | 20 |
| **Total** | **100** |

Pass >=90 and **zero critical blockers** (transport correctness, tenant isolation, N-1, checkpoint, stale telemetry, change/rollback). A synthetic pass is not production acceptance.

## References

- [NVIDIA Networking documentation](https://docs.nvidia.com/networking/)
- [NVIDIA NCCL user guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [Linux RDMA documentation](https://docs.kernel.org/infiniband/index.html)
- [AWS EFA documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)
- [Kubernetes networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)

**Next: Chapter 049 — NCCL Network Performance Troubleshooting.**
