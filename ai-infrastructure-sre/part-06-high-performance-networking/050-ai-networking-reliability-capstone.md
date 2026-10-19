# Chapter 050 — AI Networking Reliability Capstone

**AI Infrastructure Engineering & SRE | Part 06 — High Performance Networking | Chapter 050/100**

> **Acceptance boundary:** Publishing this capstone is not equivalent to validating a real GPU network. The synthetic gate, approved staging tests, security review and production signoff are separate milestones.

## Mission and learning outcomes

Integrate Chapters 043–049 into one production-oriented reliability exercise covering GPU/NIC topology, InfiniBand, RoCEv2, AWS EFA, GPUDirect RDMA, NCCL collectives, security, Kubernetes GPU/RDMA scheduling, congestion, failure recovery and SLO-driven incident command. The final deliverable is an evidence-backed go/no-go decision, not just a successful command.

## 1. Enterprise reference architecture

```text
  workload owner / incident commander / change approval
                        |
  Kubernetes + GPU Operator + RDMA/EFA device integration
                        |
       training rank groups and checkpoint coordinator
                        |
        CUDA/NCCL and transport/provider selection
                        |
       GPU -- PCIe/NUMA -- HCA/NIC per node
                        |
    +-------------------+--------------------+
    |                   |                    |
 native InfiniBand   RoCEv2 Ethernet      AWS EFA
 SM / partitions     ECN/PFC/DCQCN       libfabric EFA
    |                   |                AWS OFI NCCL
    +-------------------+--------------------+
                        |
       observability / security / cost / recovery
```

**Do not conflate transports:** InfiniBand uses subnet management; RoCEv2 relies on Ethernet/IP QoS and congestion design; EFA is AWS-specific and not generic RoCE or native IB. GPUDirect RDMA is a conditional GPU-memory access capability, not an automatic consequence of installing NCCL.

## 2. Capstone acceptance architecture

| Layer | Evidence required | Failure condition |
|---|---|---|
| Inventory | GPU UUID, HCA model, firmware, topology, failure domain | incomplete inventory |
| Compatibility | kernel, NVIDIA driver, CUDA, NCCL, libfabric/verbs/plugin | unsupported matrix |
| Fabric | IB SM/partition, RoCE ECN/PFC, or EFA provider/ENI | mismatched transport |
| Kubernetes | GPU and RDMA/EFA allocatable, scheduling, quotas, RBAC | missing resource or unsafe access |
| NCCL | correctness, effective transport, bandwidth and p95 | fallback or failed collective |
| Reliability | N-1, checkpoint RPO/RTO, incident runbook | unrecoverable failure |
| Observability | UTC-aligned GPU, NIC, fabric and rank telemetry | stale/absent signals |
| Governance | owner, security, change, rollback, cost budget | unsigned approval |

All evidence must be scoped to the same cluster revision, timestamped, redacted and independently reviewed.

## 3. Capacity and N-1 model

Define admitted GPU/rank demand, surviving GPU capacity after the largest modeled failure, total network throughput, surviving fabric bandwidth, collective bandwidth floor and storage checkpoint headroom. Example **synthetic** cluster: 3 nodes × 8 GPUs = 24 GPUs; workload requires 16 GPUs; loss of one node leaves 16 GPUs. A nominal 800 Gbps fabric with a modeled 200 Gbps rail loss leaves 600 Gbps; if the required minimum is 500 Gbps, the capacity gate passes **before overhead and actual measurement**.

Do not assert N-1 if an AZ, subnet, switch or shared power failure can remove more than the modeled resources. Compare network capacity with actual NCCL benchmark and workload step latency.

## 4. Production-safe inventory and diagnostics

Execute read-only commands on authorized infrastructure:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -o json | jq -r '.items[] | [.metadata.name, (.status.allocatable|tojson)] | @tsv'
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
nvidia-smi -L
nvidia-smi topo -m
lspci -nn
rdma link show
ibv_devices
ip -s link
ethtool -S APPROVED_NIC | grep -Ei 'drop|crc|fec|ecn|pfc|pause|error'
```

Use `fi_info -p efa` only on supported EFA hosts, `ibstat` on native IB hosts, and fabric administrator tools only with approved access. A missing metric is **unknown**, never zero. Redact internal addressing, credentials and tenant identifiers.

## 5. End-to-end staging validation

1. Freeze cluster image digest, firmware, kernel, GPU driver, CUDA, NCCL, network driver and plugin versions.
2. Verify every GPU/NIC/NUMA placement and transport-specific fabric control.
3. Check Kubernetes device resources, workload identity, quotas and security isolation.
4. Run approved single-node NCCL correctness tests to isolate GPU/software issues.
5. Run controlled multi-node all-reduce, all-gather and reduce-scatter; collect `algbw`, `busbw`, p95/p99 and per-rank skew.
6. Confirm effective transport (IB/RoCE/EFA) and GPU-memory registration mechanism where applicable.
7. Exercise N-1 link/node loss in **isolated staging** with explicit traffic budget and network owner approval.
8. Restore checkpoint; validate RPO, RTO and workload correctness.
9. Test telemetry outage, security-policy drift and failed rollout rollback.
10. Compare against matched baseline, calculate error-budget impact and sign the acceptance record.

## 6. Security and change controls

Enforce least-privilege identities, tenant isolation, GPU/RDMA device exposure, switch and management-plane ACLs, approved EFA peer SG rules, network policies, audit retention, firmware provenance and redacted logs. Remote memory keys and fabric credentials must not be exposed. Use staged canaries, change tickets, maintenance windows, rollback artifacts and separation of duties. Avoid blanket privileged pods and broad host networking.

## 7. SLOs and observability

Track collective success rate, p95/p99 step latency, measured collective bandwidth relative to a version-matched baseline, GPU Xid/ECC, NIC/queue errors, ECN/PFC counters where relevant, transport fallback, rank skew, checkpoint age and telemetry freshness. Page on sustained user-visible degradation correlated with supporting signals. SLO targets must be approved per workload, not copied from this tutorial.

## 8. Twelve troubleshooting scenarios

1. Native IB SM unavailable after failover.
2. RoCEv2 ECN/PFC mapping drift.
3. EFA provider/plugin mismatch.
4. NCCL silently falls back to TCP.
5. GPUDirect registration incompatible with driver/kernel.
6. GPU–HCA NUMA misplacement.
7. Physical CRC/FEC errors cause rank stalls.
8. ECMP path imbalance reduces throughput.
9. Kubernetes RDMA device plugin loses allocatable devices.
10. One rank hits GPU Xid/OOM and hangs a collective.
11. Checkpoint storage burst congests shared fabric.
12. Telemetry outage obscures a simultaneous N-1 breach.

## 9. Ten production-style incident simulations

**INC-050-01:** HCA or NIC loss. **02:** spine/rail outage. **03:** RoCE pause storm. **04:** IB partition mismatch. **05:** EFA ENI missing. **06:** CUDA/NCCL plugin incompatibility. **07:** security policy drift. **08:** GPU failure during all-reduce. **09:** checkpoint restore failure. **10:** monitoring outage during congestion.

Each drill must include severity, incident commander, UTC timeline, impacted ranks, blast radius, SLO/error-budget impact, containment, rollback, checkpoint evidence and RCA/COA. Perform destructive drills only in approved staging.

## 10. Eight runbooks

**RB-050-A — Transport discovery:** establish IB/RoCE/EFA and actual NCCL provider.

**RB-050-B — GPU/PCIe health:** inspect GPU Xid/ECC, HCA and NUMA mapping.

**RB-050-C — Congestion:** inspect switch queues, PFC/ECN where relevant, link errors and hot paths.

**RB-050-D — Kubernetes admission:** verify resources, placement, security and plugin health.

**RB-050-E — NCCL hang:** identify first failing rank, communicator and application error.

**RB-050-F — N-1 recovery:** preserve checkpoint, isolate failed domain and test degraded SLO.

**RB-050-G — Security/change:** contain tenant or configuration drift and restore approved policy.

**RB-050-H — Acceptance closure:** review scorecard, evidence manifest, approvals and residual risks.

## 11. Ten hands-on capstone labs

1. Map GPU/NIC/switch/AZ topology and largest shared failure domain.
2. Build transport-specific IB/RoCE/EFA compatibility matrix.
3. Inventory device resources, drivers and fabric counters.
4. Verify Kubernetes GPU/RDMA scheduling and namespace isolation.
5. Establish single-node NCCL correctness baseline.
6. Benchmark multi-node collectives on authorized staging.
7. Simulate link congestion and N-1 loss in isolated staging.
8. Restore from checkpoint and validate RPO/RTO.
9. Run companion offline policy gate and fault tests.
10. Produce security, FinOps, incident and acceptance evidence.

**Companion:** [Lab 050 — AI Networking Reliability Capstone Gate](../labs/050-ai-networking-reliability-capstone-check.md).

## 12. Evidence manifest template

```yaml
assessment_id: CH050-YYYYMMDD
environment: staging
cluster_revision: <immutable-reference>
owner: <team>
transport: <infiniband|roce|efa>
hardware_and_firmware_inventory: <artifact>
driver_cuda_nccl_versions: <artifact>
topology_and_failure_domains: <artifact>
security_review: <approval>
collective_correctness_and_bandwidth: <artifact>
n_minus_one_test: <artifact>
checkpoint_rpo_rto: <artifact>
telemetry_and_slo: <artifact>
change_and_rollback: <ticket>
reviewer_signoff: <identity-and-timestamp>
```

Never populate these fields with invented approvals or live measurements.

## 13. Final 100-point enterprise assessment

| Category | Points |
|---|---:|
| Network architecture, transport and capacity | 20 |
| GPU/NCCL, compatibility and performance | 20 |
| Kubernetes security, tenancy and change governance | 20 |
| SLO, incidents, N-1 and checkpoint recovery | 20 |
| Automated testing, staging evidence and independent review | 20 |
| **Total** | **100** |

**Pass threshold:** >=90/100, all critical gates green, evidence fresh and independent reviewer approval. A synthetic pass is a **lab pass only**, never production certification.

## References

- [NVIDIA NCCL](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [NVIDIA nccl-tests](https://github.com/NVIDIA/nccl-tests)
- [NVIDIA GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/)
- [Linux RDMA subsystem](https://docs.kernel.org/infiniband/index.html)
- [AWS EFA](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)
- [Kubernetes device plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

**Part 06 complete (Chapters 043–050). Next: Chapter 051 — AI Data Storage Architecture, beginning Part 07 — AI Storage and Data Pipelines.**
