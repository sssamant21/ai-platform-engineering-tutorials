# Chapter 079 — Network and Storage Bottleneck Incidents

**Part 10 — Incident Response and Troubleshooting | AI Infrastructure Engineering & SRE**

**Publication:** Written curriculum only. Offline lab execution, live GPU staging validation, security review and independent acceptance remain pending.

## 1. Mission and outcomes

Diagnose throughput collapse, NCCL all-reduce slowdown, RDMA/RoCE congestion, InfiniBand link errors, data-loader stalls, parallel filesystem saturation, object-store throttling and checkpoint latency across Kubernetes GPU clusters. Build an evidence-backed causal timeline, choose the smallest safe remediation and validate performance without disrupting production jobs.

**Prerequisites:** Chapters 066 (distributed training capstone), 071–074 (GPU observability), 075–078 (incident response), Kubernetes networking and Linux performance tools. Read-only diagnosis is preferred; traffic generation, network tuning, storage stress, failover and restarts are **staging-only**.

## 2. End-to-end critical path

\`\`\`text
Training step wall time
  ├─ data read / preprocessing / CPU workers
  ├─ host→device transfer / PCIe
  ├─ GPU compute
  ├─ GPU↔GPU collective: NVLink/NVSwitch/PCIe + NIC
  │    ├─ NCCL transport: TCP / InfiniBand / RoCE
  │    └─ switch fabric / routing / congestion / RDMA policy
  └─ checkpoint / metadata / filesystem / object store
\`\`\`

**Key principle:** A low GPU-utilization chart does not by itself prove GPU underprovisioning. A GPU can be idle while waiting for storage, CPU preprocessing, network collectives or scheduler synchronization. Conversely, high NIC utilization does not prove congestion; check drops, retransmits, queue occupancy, flow control and workload demand.

## 3. Incident symptom matrix

| Symptom | Likely domains | First evidence |
|---|---|---|
| All-reduce p99 grows, compute unchanged | RDMA congestion, link errors, topology | per-rank collective timing, NIC counters |
| Throughput drops at checkpoint boundary | filesystem/object-store bandwidth or metadata | checkpoint timing, I/O latency, throttling |
| GPU busy falls and data-loader wait rises | storage reads, CPU throttling, worker count | per-step wait, CPU, disk and cache |
| One rank slow, peers waiting | node/NIC/storage straggler | rank-to-node mapping and per-rank traces |
| NCCL timeout with NIC errors | fabric fault or earlier rank failure | earliest logs, RDMA link events, Xid |
| Object store HTTP 429/503 | throttling/service pressure | request metrics, retry/backoff and error rates |
| High filesystem IOPS but low throughput | small I/O, metadata storms, latency | request sizes, ops and queue depth |
| GPU nodes have uneven bandwidth | NUMA/GPU↔NIC affinity, topology | \`nvidia-smi topo -m\`, CPU/NIC placement |
| High TCP retransmits | packet loss/congestion/host path | \`ss -ti\`, interface errors and pcap policy |

## 4. First 15 minutes — read-only collection

### 4.1 Kubernetes and workload

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl -n <namespace> get pods -o wide -l <job-label-selector>
kubectl -n <namespace> describe pod <rank-pod>
kubectl -n <namespace> logs <rank-pod> --timestamps=true --since=2h
kubectl -n <namespace> get pvc
kubectl get pv
kubectl get storageclasses
\`\`\`

Record job UID, ranks, nodes, GPU UUIDs, NIC names, storage mount/PVC/volume identity, checkpoint destination, recent deployment or network changes, time of regression, and last successful training step.

### 4.2 GPU and topology (approved host)

\`\`\`bash
nvidia-smi topo -m
nvidia-smi --query-gpu=uuid,name,utilization.gpu,memory.used --format=csv
lspci | grep -Ei 'NVIDIA|Mellanox|Ethernet|InfiniBand'
numactl --hardware
\`\`\`

\`numactl\` and \`lspci\` may not be installed or accessible in minimal images. \`nvidia-smi topo -m\` is a topology hint, not an application-level bandwidth measurement.

### 4.3 Network evidence

\`\`\`bash
ip -s link
ss -s
ss -ti
cat /proc/net/dev
# When available and authorized:
rdma link show
ibstat
ethtool -S <nic>
\`\`\`

Capture two or more counter samples with timestamps and calculate rates from deltas; absolute counters alone are often not meaningful. Interpret interface counters by NIC driver and fabric technology. RDMA paths may bypass ordinary TCP counters.

### 4.4 Storage evidence

\`\`\`bash
df -h
mount | grep -E 'nfs|lustre|gpfs|ceph|xfs|ext4'
iostat -xz 1 3
vmstat 1 3
cat /proc/pressure/io
\`\`\`

\`iostat\` may require \`sysstat\`; PSI may be unavailable on some kernels. Avoid reading raw training data or credentials while diagnosing. For object storage, use provider metrics for request latency, throttle/5xx, bandwidth, retry volume and request cost.

## 5. Metrics and SLIs

| Layer | Suggested SLI | Why |
|---|---|---|
| GPU training | steps/sec, step p50/p99 | user-facing training throughput |
| NCCL | collective duration p95/p99, rank variance | synchronization bottlenecks |
| Fabric | link errors, drops, ECN/CNP/PFC where supported | congestion and loss evidence |
| TCP | retransmit rate, RTT | host-network symptoms |
| Storage | read/write throughput, p99 latency, queue depth | data/checkpoint bottleneck |
| Object store | 429/503, request p99, retries | backend throttling |
| CPU/data loader | worker wait, CPU throttle, I/O wait | non-GPU stalls |
| Checkpoints | successful completion, age, duration | recovery point and reliability |

Define labels carefully: job IDs, object keys, GPU UUIDs and file paths can explode cardinality or expose tenant data. Aggregate per tenant/workload tier; retain detailed rank IDs in bounded logs/traces. DCGM exporter and cloud storage metrics depend on versions/configuration; discover actual names rather than assuming availability.

### 5.1 Illustrative PromQL

\`\`\`promql
# Standard node-exporter metrics, subject to exporter labels:
rate(node_network_receive_errs_total[5m])
rate(node_network_transmit_drop_total[5m])
rate(node_disk_read_bytes_total[5m])
rate(node_disk_written_bytes_total[5m])

# Example custom histogram: instrument before using
histogram_quantile(0.99,
  sum by (le, workload) (rate(training_step_duration_seconds_bucket[5m])))
\`\`\`

Never infer RDMA congestion solely from TCP or node-exporter interface counters. Correlate fabric-specific telemetry and workload step latency.

## 6. Network troubleshooting: TCP, InfiniBand, RoCE

### 6.1 TCP path

Check DNS/rendezvous, NetworkPolicies, CNI routes, NIC MTU, link speed, packet drops, retransmits, CPU saturation and kernel socket buffers. A single high RTT is not proof of congestion; inspect time-series and comparable baselines.

### 6.2 InfiniBand

Verify HCA port state, subnet manager/routing, link error counters, P_Key/security partitioning and GPU/NIC placement. Use \`ibstat\`, \`ibv_devinfo\` and switch telemetry where available. Do not reset fabric ports or change subnet manager during a live job without network change approval.

### 6.3 RoCE

Check end-to-end MTU, VLAN/priority mapping, ECN marking, PFC behavior, switch buffer occupancy, congestion notifications and loss. PFC pause frames are not automatically a failure; correlate pause duration and impact. Misconfigured PFC can create head-of-line blocking and fabric-wide issues. Escalate switch changes to the fabric owner.

### 6.4 GPU↔NIC topology

Map rank → pod UID → node → GPU UUID/MIG → NIC → switch/rail. A poor NUMA or PCIe topology can increase latency or consume host interconnect bandwidth. Validate proposed affinity changes against actual device-plugin allocation and Kubernetes scheduling; do not hardcode host GPU index assumptions.

## 7. Storage bottleneck isolation

### 7.1 Data ingest

Differentiate cold-cache from warm-cache runs, sequential vs random reads, small-file metadata storms, CPU decode/decompression, object-store retries and filesystem throughput. Capture bytes/sec, ops/sec, p99 latency, dataset version, input pipeline parallelism and node CPU pressure.

### 7.2 Checkpoint I/O

Checkpoint time includes serialization, metadata, network transfer, storage commit and rank coordination. Measure:
- start/end UTC, bytes, write throughput, per-rank skew;
- atomicity/commit-marker semantics and consistency guarantees;
- retry/backoff, object-store request throttling and network egress;
- whether checkpoint can be restored and resumed, not merely uploaded.

**Never delete old checkpoints** as an immediate response to a storage bottleneck without verifying retention, restore points and recovery objectives.

### 7.3 Parallel filesystem / PVC

Investigate metadata server load, OSS/OST imbalance (for Lustre), Ceph OSD/PG health, NFS server latency, PV topology, PVC expansion and mount options as appropriate to the actual system. Do not assume a generic POSIX filesystem behaves like object storage.

## 8. Controlled staging benchmarks

| Tool | Suitable use | Limit |
|---|---|---|
| \`nccl-tests\` | GPU collective throughput/latency | not application correctness |
| \`iperf3\` | TCP bandwidth baseline | not RDMA performance |
| \`fio\` | block/filesystem I/O under bounded load | can cause data loss or contention if misused |
| \`ib_write_bw\` / \`ib_read_bw\` | RDMA bandwidth where supported | fabric-specific and potentially disruptive |
| Synthetic checkpoint | end-to-end application I/O path | must use disposable storage |

Set maximum duration, rate, bytes, concurrency, test namespace and abort thresholds. **Never run unbounded \`fio\`, \`iperf3\`, RDMA or NCCL stress on production nodes or shared storage.** Do not benchmark real customer datasets.

## 9. Decision tree

\`\`\`text
Training step p99 or throughput regressed?
 ├─ Data-loader wait up → CPU, disk, cache, object-store reads
 ├─ Collective time up → rank skew, GPU health, NIC/fabric/topology
 ├─ Checkpoint time up → serialization, filesystem/object store, metadata
 └─ No clear bottleneck → instrumentation gaps, mixed workload,
                         GPU compute, scheduling, clock skew
\`\`\`

Use at least two independent signals before attributing causality. An incident may have multiple contributing bottlenecks.

## 10. Sixteen hands-on exercises

1. Capture workload throughput/latency baseline and SLO.
2. Map ranks, GPU UUIDs, NICs, storage volumes and topology.
3. Build a UTC incident timeline with deployment/change history.
4. Compare NIC error/drop counters across intervals.
5. Analyze synthetic TCP retransmission spike.
6. Analyze synthetic InfiniBand link degradation.
7. Analyze synthetic RoCE ECN/PFC congestion.
8. Review GPU↔NIC NUMA/PCIe affinity.
9. Compare training collective time across ranks.
10. Profile data-loader wait and CPU decode.
11. Analyze filesystem p99 and queue depth.
12. Analyze object-store 429/503 and retry behavior.
13. Measure checkpoint write and verified restore in staging.
14. Execute one approved bounded network/storage benchmark in staging.
15. Validate alerts and safe mitigation/rollback tabletop.
16. Produce RCA, runbooks, evidence and acceptance score.

## 11. Troubleshooting matrix

| Symptom | First check | Avoid |
|---|---|---|
| Low GPU util + high step time | data loader, collective, checkpoint | GPU scale-first |
| High NIC traffic + slow NCCL | drops, ECN/PFC, rank skew | assuming bandwidth alone is fault |
| TCP clean, NCCL slow | RDMA-specific telemetry | ruling out fabric |
| Storage IOPS high, throughput low | request size, metadata, latency | IOPS-only diagnosis |
| Checkpoint upload complete, resume fails | commit/consistency and restore | assuming upload means valid checkpoint |
| One node slower | GPU/NIC/NUMA and I/O | restarting whole cluster |
| Object-store throttling | 429/503, retries, request shape | unbounded retry storm |
| RDMA errors after network change | link/switch/GID/MTU timeline | ad hoc PFC change |

## 12. Production runbooks

**A — Collective slowdown:** preserve per-rank logs, NCCL timing and fabric counters; identify topology/straggler/transport; stage bounded tests; restore SLO and document network owner sign-off.

**B — RoCE/InfiniBand incident:** inspect port state, errors, ECN/PFC/CNP and switch events; coordinate fabric team; isolate affected rail/node; change configuration only with rollback.

**C — Data-loader bottleneck:** measure CPU, cache, filesystem and object-store latency; optimize input pipeline in staging; validate throughput without compromising data consistency.

**D — Checkpoint/storage outage:** stop unsafe repeated writes, preserve last complete checkpoint, assess backend errors and RPO, restore service, validate a real read/restore test.

**E — Cross-layer incident:** build UTC timeline of training, GPU, NIC, storage and deployment signals; test competing hypotheses; communicate impact and corrective actions.

## 13. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Cross-layer topology and baseline | 20 |
| Network/RDMA/fabric diagnosis | 20 |
| Storage/checkpoint diagnosis | 20 |
| Workload SLI/SLO and causality | 15 |
| Staging benchmarks and recovery safety | 15 |
| Runbooks, evidence and independent review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 and no critical security, data-integrity, unsafe-network-change or production-impact failure. Publication alone is not execution.

## 14. References

- [NVIDIA NCCL Troubleshooting](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/troubleshooting.html)
- [NVIDIA NCCL Tests](https://github.com/NVIDIA/nccl-tests)
- [Linux iproute2](https://wiki.linuxfoundation.org/networking/iproute2)
- [Prometheus Node Exporter](https://github.com/prometheus/node_exporter)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [fio documentation](https://fio.readthedocs.io/en/latest/)
- [PyTorch Distributed](https://docs.pytorch.org/docs/stable/distributed.html)

**Next:** Chapter 080 — GPU Failure Triage and Escalation.
