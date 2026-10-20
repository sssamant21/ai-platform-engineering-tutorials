# Chapter 057 — Storage IOPS, Throughput and Cost

**AI Infrastructure Engineering & SRE · Part 07 — AI Storage and Data Pipelines · Chapter 057/100**

> **Scope:** Reference architecture, calculations and offline synthetic validation. Provider rates and quotas change; do not use illustrative prices as current quotes. No live benchmarking or cloud-cost verification has been performed.

## Objectives and prerequisites

Measure, size, troubleshoot and optimize storage performance and unit economics for distributed GPU workloads without compromising reliability. Distinguish IOPS, sequential throughput, metadata operations, tail latency, object API requests, checkpoint contention, capacity headroom and cloud egress. Prerequisites: Chapters 051–056 and GPU/network observability fundamentals.

## 1. Storage performance and FinOps reference architecture

```text
versioned object store (S3 / Azure Blob)
  | request rate, p95 GET, egress, storage tier
  v
parallel POSIX tier (FSx for Lustre / Lustre / BeeGFS)
  | OST bandwidth, MDT ops, capacity, striping
  v
GPU-node local NVMe / page cache / prefetch
  | cache hit, eviction, decode, host transfer
  v
PyTorch DataLoader -> GPU training / inference
  | step time, data wait, samples/s, GPU utilization
  +-> coordinated checkpoint writes -> durable archive
        | write bandwidth, age, RPO, restore RTO
  +-> metrics, traces, billing allocation, budgets
```

A fast synthetic storage benchmark does not prove a fast training epoch. Measure end-to-end *useful* throughput, including decoding, augmentation, distributed rank skew and retries.

## 2. Core metrics and interpretation

| Metric | Unit | What it answers |
|---|---|---|
| Read/write IOPS | operations/s | random/small-block capacity |
| Throughput | GB/s or GiB/s | large sequential read/write capacity |
| p95/p99 latency | ms | tail risk and batch stalls |
| Metadata operations | ops/s | small-file and directory pressure |
| Queue depth / utilization | requests | saturation and backlog |
| Object GET/PUT/LIST | requests/s | API overhead and cost |
| Retry/429/503 | % / count | service throttling or overload |
| Cache hit / miss | % | origin-read amplification |
| GPU data-wait | % of step | user-visible impact |
| Capacity / inode headroom | GB / % | exhaustion and failure-domain reserve |
| Checkpoint age | minutes | verified RPO |
| $ per successful epoch | currency/epoch | reliability-aware unit cost |

Keep decimal GB/s distinct from binary GiB/s. IOPS and throughput are related by request size, but not interchangeable: `throughput_Bps ≈ IOPS × average_request_bytes` only under a compatible workload and without other bottlenecks.

## 3. Sizing formulas

```text
origin_read_GBps = workers * worker_read_GBps * cache_miss_fraction * burst
required_IOPS = read_GBps * 1e9 / average_read_bytes
checkpoint_write_GBps = checkpoint_GB / checkpoint_window_seconds
capacity_with_headroom_GB = active_bytes_GB / (1 - headroom_fraction)
cost_per_epoch = (storage + requests + transfer + filesystem + compute_stall) / epochs
```

**Synthetic example:** 32 workers × 0.25 GB/s × 40% miss × 1.25 burst = 4 GB/s origin demand. With 256 KiB average reads, this is roughly 15,259 IOPS. A 2,400 GB checkpoint in 300 s needs another 8 GB/s write bandwidth if it overlaps. If both share a constrained backend, independently verify aggregate contention and separate read/write limits.

## 4. Storage tier tradeoffs

**S3 / Azure Blob:** scalable durable object tier; budget storage class, request charges, lifecycle transitions, retrieval, replication and cross-zone/region transfer. Use immutable manifests and bounded concurrency; do not assume object semantics equal POSIX.

**FSx for Lustre / Lustre / BeeGFS:** high parallel file bandwidth; size metadata performance, client network, stripe layout, provisioned capacity/throughput and deployment-specific backup/availability. Compare scratch and persistent failure objectives before selecting.

**Block volumes:** size provisioned IOPS/throughput, attachment topology and snapshot recovery. **NVMe cache:** fast but ephemeral; include refill cost and loss of an entire node. **NFS/shared file:** simpler but may bottleneck on one service endpoint.

## 5. Benchmark methodology

1. Define the user-facing SLO (GPU data-wait p95, epoch duration, checkpoint age).
2. Pin GPU SKU, node count, NIC, topology, dataset manifest, shard size, cache state and client versions.
3. Separate sequential reads, random reads, metadata storms, mixed read/write and checkpoint bursts.
4. Benchmark with approved tools (e.g. fio, IOR, mdtest) only on isolated staging storage; use capped duration and I/O volume.
5. Capture cold-cache and warm-cache p50/p95/p99, sustained IOPS, GB/s, per-target skew, retry rates and CPU utilization.
6. Repeat at representative distributed rank concurrency and a defined N-1 failure-domain scenario.
7. Compare useful training samples/s and step-time before and after; report confidence intervals where feasible.
8. Archive configuration, raw metrics, billable dimensions, rollback and reviewer approval.

**Read-only commands** (authorized hosts only):

```bash
iostat -xz 1 3
vmstat 1 3
df -hT /approved/dataset/mount
df -i /approved/dataset/mount
kubectl get pv,pvc -A
kubectl get events -A --sort-by=.lastTimestamp
nvidia-smi --query-gpu=uuid,utilization.gpu,memory.used --format=csv
```

## 6. Reliability-aware cost model

Track object GB-months, request volume, data transfer, parallel filesystem provisioned throughput/capacity, backup replicas, NVMe cache, failed epochs and idle GPU hours caused by I/O. Include *effective compute cost* when storage is slow: saving on storage may increase GPU cost and miss SLOs. Compare unit costs on identical datasets, jobs, regions and reliability requirements.

**Illustrative scenario only:** Storage option A costs $300/job and incurs $700 of GPU idle time; option B costs $500/job and incurs $100 of GPU idle time. The second option is cheaper end-to-end ($600 vs $1,000), assuming equivalent reliability and correctness. These are synthetic, not provider prices.

## 7. Capacity, quotas and failure domains

Forecast dataset growth, checkpoint retention, backup copies, metadata/inode growth, API quotas, network headroom and N-1 surviving throughput/capacity. A single AZ outage may remove multiple correlated clients or storage components. Do not claim N-1 coverage from aggregate normal-state metrics; use an explicit failure-domain inventory and tested degraded measurements.

## 8. Security and governance

Apply workload identity, encryption, tenant isolation, data residency, least privilege, audit logs, owner-approved retention, legal hold and immutable artifact references. Cost optimization cannot silently delete the only recoverable checkpoint or relax access controls. Require rollback plans and review for tier transitions, capacity reductions and lifecycle policies.

## 9. SLO and alerting design

Monitor read throughput versus demand, IOPS, p95 metadata latency, object 429/503, cache miss, GPU data-wait, cost/epoch, checkpoint age, restore RTO and telemetry freshness. Trigger alerts on sustained user-impacting error-budget burn, not isolated high utilization. Attribute to job, dataset version, tenant, region, storage ID and UTC.

## 10. Twelve troubleshooting scenarios

1. GPU utilization drops despite healthy accelerator nodes because data-loader reads stall.
2. Millions of tiny files overwhelm Lustre metadata operations.
3. Object storage GET/HEAD/LIST requests cause throttling and retry amplification.
4. Provisioned bandwidth looks adequate but per-client network links saturate.
5. Random reads hit IOPS limits while sequential throughput remains healthy.
6. Checkpoint writes compete with dataset reads and NCCL collectives.
7. Cache-hit ratio appears high but cold-start p99 misses the SLO.
8. Cross-AZ placement increases latency and transfer charges.
9. Parallel filesystem stripe imbalance creates a hot OST.
10. Burst-credit or tier-specific limits cause performance decay over time.
11. Cost attribution misses shared storage and checkpoint replicas.
12. Telemetry lag hides capacity exhaustion and checkpoint RPO violations.

## 11. Ten incident simulations

1. **INC-057-01:** Metadata storm.
2. **INC-057-02:** IOPS ceiling.
3. **INC-057-03:** Throughput collapse.
4. **INC-057-04:** Object API throttling.
5. **INC-057-05:** Cold-cache GPU starvation.
6. **INC-057-06:** Checkpoint-write contention.
7. **INC-057-07:** AZ storage path failure.
8. **INC-057-08:** Unexpected egress bill.
9. **INC-057-09:** Retention cost regression.
10. **INC-057-10:** Monitoring outage.

Capture incident commander, UTC timeline, workload impact, p95/p99 metrics, cost exposure, containment, restore evidence, change ticket and RCA/COA. Stress tests and destructive faults are staging-only.

## 12. Eight production runbooks

1. **RB-057-A — Read/write bottleneck isolation:** Correlate workload latency, queue depth, IOPS, throughput, CPU decode, network and GPU wait.
2. **RB-057-B — Metadata saturation:** Measure operations/s, file count and per-target latency; redesign shards in staging.
3. **RB-057-C — Object throttling:** Inspect 429/503, retry rate, request concurrency and backoff.
4. **RB-057-D — Checkpoint contention:** Prioritize verified RPO, schedule/admit checkpoint bursts and protect NCCL traffic.
5. **RB-057-E — Cache and locality:** Compare cold/warm cache and same-zone/cross-zone runs with consistent workloads.
6. **RB-057-F — Capacity forecast:** Project bytes, inodes, growth, backups, headroom and failure-domain loss.
7. **RB-057-G — FinOps change:** Compare $/epoch, request cost, transfer, provisioned storage and SLO impact.
8. **RB-057-H — Incident and rollback:** Preserve evidence, restore previous configuration and validate user-facing SLO.

## 13. Ten hands-on labs

1. Draw storage tier, network and GPU failure-domain dependencies.
2. Calculate origin throughput and IOPS for a cache-miss workload.
3. Size checkpoint write bandwidth under read/write contention.
4. Model degraded N-1 throughput and capacity headroom.
5. Compare cold/warm cache latency and useful samples/s.
6. Build a synthetic object-request and cross-AZ cost ledger.
7. Simulate storage throttling, metadata latency and telemetry staleness.
8. Test retention approval and last-known-good checkpoint protection.
9. Run the companion offline fail-closed evaluator and pytest suite.
10. Complete a 100-point SRE/FinOps design acceptance packet.

**Companion:** [Lab 057 — Storage IOPS, Throughput and Cost Gate](../labs/057-storage-iops-throughput-cost-check.md).

## 14. Enterprise acceptance — 100 points

| Domain | Points |
|---|---:|
| Performance model and workload SLOs | 20 |
| Capacity, degraded-mode headroom and checkpoints | 20 |
| Cloud unit economics and cost attribution | 20 |
| Security, incidents, governance and rollback | 20 |
| Tests, live staging benchmarks and independent review | 20 |
| **Total** | **100** |

Pass >=90 and zero critical blocker for data integrity, degraded throughput, RPO, retention, security, telemetry or rollback. Synthetic policy pass does **not** certify production readiness.

## References

- [Amazon S3 performance guidelines](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html)
- [Amazon FSx for Lustre](https://docs.aws.amazon.com/fsx/latest/LustreGuide/what-is.html)
- [Azure Blob scalability targets](https://learn.microsoft.com/azure/storage/blobs/scalability-targets)
- [fio documentation](https://fio.readthedocs.io/en/latest/)
- [Kubernetes Storage](https://kubernetes.io/docs/concepts/storage/)

**Next: Chapter 058 — AI Storage Reliability Capstone.**
