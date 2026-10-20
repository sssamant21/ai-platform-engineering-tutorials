# Lab 079 — Network and Storage Bottleneck Incidents Acceptance Gate

**Chapter:** [079 — Network and Storage Bottleneck Incidents](../part-10-incident-response-and-troubleshooting/079-network-storage-bottleneck-incidents.md)

**Status:** NOT EXECUTED. Live GPU/network/storage staging tests, offline exercises and peer review remain pending.

## 1. Safety

Read-only diagnostics are permitted only within authorized access. Network stress, \`fio\`, \`iperf3\`, RDMA benchmarks, storage failover, checkpoint writes and tuning require dedicated staging, explicit load ceilings, disposable test data, change approval and rollback. No production traffic generation or destructive filesystem tests.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | Workload SLO baseline | step throughput and p99 | Pending |
| E02 | Cross-layer topology | rank/GPU/NIC/storage map | Pending |
| E03 | UTC incident timeline | changes and earliest symptom | Pending |
| E04 | NIC counter rates | two timestamped samples | Pending |
| E05 | Synthetic TCP loss | retransmit analysis | Pending |
| E06 | Synthetic IB link fault | port/error analysis | Pending |
| E07 | Synthetic RoCE congestion | ECN/PFC/CNP evidence | Pending |
| E08 | GPU/NIC affinity | NUMA/PCIe map | Pending |
| E09 | NCCL collective skew | per-rank timing | Pending |
| E10 | Data-loader stall | CPU/input wait | Pending |
| E11 | Filesystem saturation | latency/IOPS/queue | Pending |
| E12 | Object-store throttling | 429/503 and retries | Pending |
| E13 | Checkpoint restore | staged write and verified read | Pending |
| E14 | Bounded benchmark | approved scope, abort/rollback | Pending |
| E15 | Alerting and mitigation | test notification and recovery | Pending |
| E16 | RCA and review | score and reviewer | Pending |

## 3. Read-only collection

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl -n <namespace> get pods -o wide -l <job-label-selector>
kubectl -n <namespace> get pvc
kubectl get pv
\`\`\`

Approved GPU/network host:

\`\`\`bash
nvidia-smi topo -m
ip -s link
ss -s
ss -ti
rdma link show
df -h
iostat -xz 1 3
vmstat 1 3
\`\`\`

Some utilities may be absent; record \`NOT_AVAILABLE\` rather than assuming healthy signals. Redact host identities and tenant data.

## 4. Offline rate calculation

Use a synthetic NIC counter fixture to validate rate reasoning:

\`\`\`python
first = {"t": 0, "rx_drops": 120, "tx_bytes": 2_000_000_000}
second = {"t": 60, "rx_drops": 180, "tx_bytes": 8_000_000_000}
dt = second["t"] - first["t"]
assert dt > 0
drops_per_second = (second["rx_drops"] - first["rx_drops"]) / dt
throughput_mbps = (second["tx_bytes"] - first["tx_bytes"]) * 8 / dt / 1e6
print("drops/sec:", drops_per_second)
print("throughput Mbps:", throughput_mbps)
\`\`\`

Expected: 1 drop/sec and 800 Mbps. This is arithmetic only; real counters can reset or wrap, and link rate is not necessarily application throughput.

## 5. Staging-only test plan

**Network:** one dedicated staging pair, documented interface, 30-second bounded test, approved rate cap and no shared production fabric. \`iperf3\` tests TCP, not RDMA.

**Storage:** disposable PVC/bucket and uniquely named files, fixed total bytes, concurrency and duration, no destructive overwrite, verify checksums on readback and delete only approved test artifacts after evidence retention.

**Checkpoint:** synthetic training checkpoint to isolated storage, commit marker and restore test, compare RPO/RTO. Never use real production model checkpoints as disposable fixtures.

If no isolated staging is available, mark these as **NOT_RUN**, not PASS.

## 6. Synthetic diagnosis cases

**A — NCCL slowdown:** p99 collective duration doubles; GPU utilization falls; RDMA port error counter increases. Identify fabric hypothesis and confirm against per-rank logs.

**B — Storage metadata storm:** throughput drops while IOPS rises and average request size shrinks. Evaluate small-file/metadata workload.

**C — Object-store throttling:** 429/503 and retry spikes during checkpoint writes. Apply bounded retry/backoff and checkpoint safety; do not amplify request storms.

**D — Topology imbalance:** one rank's GPU/NIC path crosses NUMA boundaries; compare placement before proposing change.

**E — Data-loader stall:** CPU decode saturates while GPU idle; avoid blaming GPU or NCCL.

## 7. Evidence template

\`\`\`yaml
chapter: "079"
environment: "STAGING_OR_OFFLINE"
execution_date_utc: null
reviewer: null
gpu_topology: null
nic_fabric_type: null
storage_backend: null
workload_baseline_steps_per_sec: null
workload_baseline_p99_ms: null
network_counter_samples: []
storage_latency_samples: []
checkpoint_restore: "NOT_RUN"
bounded_network_test: "NOT_RUN"
bounded_storage_test: "NOT_RUN"
synthetic_cases: "NOT_RUN"
alert_delivery: "NOT_RUN"
security_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 8. Acceptance

Score **20 + 20 + 20 + 15 + 15 + 10 = 100**. Pass at ≥85 with no critical data loss, security exposure, unsafe fabric change or production impact. Distinguish offline analysis from live staging evidence.

**Next:** Chapter 080 — GPU Failure Triage and Escalation.
