# Chapter 058 — AI Storage Reliability Capstone

**AI Infrastructure Engineering & SRE · Part 07 — AI Storage and Data Pipelines · Chapter 058/100**

> **Capstone status:** A comprehensive production-readiness *design* and offline synthetic acceptance gate. Publishing this chapter does not establish that real cloud resources, distributed GPU jobs, storage benchmarks or disaster-recovery drills have been tested.

## 1. Capstone mission

Prove that an AI training platform can load an immutable dataset, sustain required GPU input throughput, complete and retain globally consistent distributed checkpoints, and recover from storage and infrastructure failures within approved RPO/RTO while maintaining security and predictable unit economics.

**Prerequisites:** Chapters 051–057, Kubernetes/EKS/AKS, object storage, Lustre/FSx, PyTorch distributed training, observability, cloud identity and incident response.

## 2. Reference architecture

```text
immutable dataset versions (S3 / Azure Blob)
        | checksums, manifest, lineage, IAM/KMS
        v
FSx for Lustre / parallel filesystem / POSIX tier
        | metadata, OST/OSS, CSI mounts, network
        v
node NVMe / bounded cache / CPU decode / prefetch
        | shard-aware rank + worker assignment
        v
Kubernetes GPU training (PyTorch distributed / NCCL)
        | step-time, batch-wait and GPU utilization SLO
        v
coordinated distributed checkpoint
        | rank shards + SHA-256 + global commit manifest
        v
durable versioned object archive / independent copy
        | dependency-aware retention / legal hold
        v
isolated restore + optimizer/RNG/sampler validation
        | RPO / RTO / resume correctness
        +-------------------------------+
        | Prometheus / OTel / logs / cost ledger
        +-> SRE on-call, incident command, change review
```

Document blast radius by storage tier, zone, network rail, CSI driver, encryption key and workload identity. A fast single-client filesystem test is not evidence of end-to-end GPU reliability.

## 3. Capstone scenario and acceptance contract

**Synthetic reference environment:** 32 GPU workers, 0.25 GB/s source-read demand per worker, 40% cache miss, 1.25 burst, 256 KiB average reads, a 2,400 GB checkpoint with a 300-second write window, 30-minute RPO and 120-minute end-to-end RTO. These are illustrative **site-defined** budgets, not vendor service guarantees.

```text
origin_read_GBps = 32 * 0.25 * 0.40 * 1.25 = 4.0
read_IOPS ~= 4.0e9 / (256 * 1024) = 15,259
checkpoint_write_GBps = 2400 / 300 = 8.0
degraded_read_headroom = surviving_read_GBps - 4.0
degraded_write_headroom = surviving_write_GBps - 8.0
RPO = incident_time - latest_verified_durable_commit
RTO = detect + triage + provision + transfer + verify + load + resume
```

If read and checkpoint writes overlap, independently measure shared bottlenecks and ensure both demands can be met concurrently. Capacity must include datasets, staging, retention, replicas, inodes and N-1 headroom. Site-approved performance budgets override examples.

## 4. Eight acceptance workstreams

### WS1 — Object store and dataset lineage

Prove immutable manifests, pinned object versions, checksums, schema ownership, least-privilege access, request throttling controls and recovery from object loss. Reject mutable `latest/` prefixes without version pinning.

### WS2 — Parallel filesystem and Kubernetes CSI

Validate Lustre/FSx deployment-type fit, metadata/OST health, striping, per-client bandwidth, Kubernetes PVC/PV access modes, node placement, security groups and driver/client compatibility. Test mount failures only in isolated staging.

### WS3 — Distributed dataset loading

Measure CPU decode, cache hit/miss, prefetch memory, host-to-device transfer, rank skew, GPU data-wait p95 and useful samples/s. Prove rank/worker sharding avoids duplicates and omissions.

### WS4 — Checkpoint publication and retention

Require complete rank shards, SHA-256 verification, globally consistent step, committed immutable manifest, durable archive, protected previous known-good release, dependency-safe delta retention and owner-approved cleanup.

### WS5 — Disaster recovery and N-1

Inject an approved failure-domain loss, restore from an independent durable copy, verify optimizer/scheduler/RNG/sampler state, then measure actual RPO, detect-to-resume RTO and degraded throughput. A filesystem snapshot alone is not application consistency.

### WS6 — Security and governance

Verify scoped workload identity, KMS access, encryption, tenant isolation, auditability, data residency, legal holds, retention owner approval and controlled rollback.

### WS7 — Observability and incident command

Instrument p95/p99 loader wait, filesystem metadata latency, object 429/503, IOPS, read/write GB/s, checkpoint age, restore duration, GPU idle, capacity, cost/epoch and telemetry freshness. Attach UTC incident timelines and RCA/COA.

### WS8 — FinOps and independent signoff

Reconcile object requests, provisioned parallel storage, backup/replication, cross-AZ transfer, NVMe cache, wasted GPU time and unit cost. Require documented tradeoffs and an independent acceptance reviewer.

## 5. Read-only diagnostic inventory

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get pv,pvc -A
kubectl get storageclass
kubectl get events -A --sort-by=.lastTimestamp
df -hT /approved/dataset/mount
df -i /approved/dataset/mount
iostat -xz 1 3
nvidia-smi --query-gpu=uuid,utilization.gpu,memory.used --format=csv
```

Run only with authorization and installed dependencies. No production fault injection, mass reads, deletion, snapshot restores or benchmark saturation without an approved change.

## 6. Fifteen troubleshooting scenarios

1. Object-store version drift invalidates dataset manifest.
2. Dataset checksum mismatch is served from a hot cache.
3. FSx/Lustre metadata storm stalls training ranks.
4. OST imbalance causes p99 latency spikes.
5. CSI mount failure blocks GPU workloads.
6. Object API throttling triggers a retry storm.
7. CPU decode bottleneck mimics slow storage.
8. Cache eviction amplifies source reads and cost.
9. Checkpoint burst starves active dataset readers.
10. Partial rank checkpoint is incorrectly committed.
11. Delta-chain retention removes the only valid base.
12. KMS/identity drift blocks disaster recovery.
13. AZ failure reduces usable throughput below N-1.
14. RPO breach occurs while checkpoint telemetry is stale.
15. Cost anomaly arises from cross-region transfer.

For each scenario: state the symptom, affected SLIs, first safe inspection, hypothesis, containment, recovery action, rollback criterion and required evidence.

## 7. Ten incident simulations

1. **INC-058-01:** Manifest rollback and checksum mismatch.
2. **INC-058-02:** Metadata storm and OST skew.
3. **INC-058-03:** Object request throttling.
4. **INC-058-04:** Cold-cache GPU starvation.
5. **INC-058-05:** Checkpoint partial commit.
6. **INC-058-06:** Broken incremental retention chain.
7. **INC-058-07:** AZ storage path loss.
8. **INC-058-08:** KMS and tenant isolation failure.
9. **INC-058-09:** Cost and capacity shock.
10. **INC-058-10:** Monitoring outage during RPO breach.

Record severity, incident commander, affected job/ranks, UTC timeline, SLO/error-budget impact, RPO/RTO, customer communications, rollback and independent RCA. Inject only in approved staging.

## 8. Eight production runbooks

1. **RB-058-A — Dataset integrity:** Freeze release, verify immutable manifest, object versions and checksums.
2. **RB-058-B — Filesystem and CSI:** Inspect mounts, metadata targets, data targets, PVC topology and client compatibility.
3. **RB-058-C — Loader starvation:** Correlate GPU data wait, CPU decode, cache, network and per-rank skew.
4. **RB-058-D — Object throttling:** Bound request concurrency and jitter retries; preserve manifest consistency.
5. **RB-058-E — Checkpoint recovery:** Verify all shards, committed manifest, durable copy and sampler state.
6. **RB-058-F — DR and N-1:** Restore to isolated failure domain and measure degraded throughput, RPO/RTO.
7. **RB-058-G — Capacity and FinOps:** Review growth, inodes, requests, transfer, checkpoint retention and cost/epoch.
8. **RB-058-H — Change and incident closure:** Validate rollback, incident evidence, owner approval and independent review.

## 9. Twelve hands-on capstone labs

1. Build the storage/GPU dependency and failure-domain diagram.
2. Pin a synthetic immutable dataset manifest and verify checksums.
3. Review Lustre metadata/OST and Kubernetes CSI read-only evidence.
4. Calculate source throughput, IOPS, cache misses and burst demand.
5. Model concurrent checkpoint writes and N-1 surviving headroom.
6. Measure or simulate GPU data-loader p95 latency and rank skew.
7. Validate sharded checkpoint manifest, optimizer/RNG/sampler and durable archive.
8. Simulate a partial checkpoint and reject unsafe promotion.
9. Simulate regional/AZ loss and calculate verified RPO/RTO.
10. Evaluate retention, legal hold, KMS and tenant isolation.
11. Reconcile synthetic storage costs and GPU idle impact.
12. Run companion policy tests, compile the evidence pack and hold a readiness review.

**Companion:** [Lab 058 — AI Storage Reliability Capstone Gate](../labs/058-ai-storage-reliability-capstone-check.md).

## 10. Evidence packet

```yaml
capstone: AI-STORAGE-058
environment: staging
dataset_manifest_and_checksums: <evidence>
filesystem_csi_and_network: <evidence>
gpu_loader_slo_and_rank_sharding: <evidence>
read_iops_throughput_and_n_minus_one: <measurements>
checkpoint_global_commit_and_archive: <evidence>
checkpoint_retention_and_legal_hold: <evidence>
isolated_restore_optimizer_rng_sampler: <evidence>
measured_rpo_and_rto: <measurements>
incident_drill_timeline_and_rca: <evidence>
identity_encryption_tenant_review: <evidence>
cost_per_epoch_and_capacity_forecast: <evidence>
telemetry_freshness: <evidence>
change_rollback_owner_and_independent_signoff: <tickets>
```

## 11. Enterprise acceptance — 100 points

| Category | Points |
|---|---:|
| Dataset integrity, parallel storage and sharding | 20 |
| Read/write IOPS, throughput, GPU loader SLO and N-1 | 20 |
| Checkpoint consistency, retention, RPO and RTO | 20 |
| Security, governance, observability and incident response | 20 |
| FinOps, offline tests, staging evidence and independent signoff | 20 |
| **Total** | **100** |

**Passing policy:** >=90/100, no critical gate failure, production-equivalent staging benchmarks, measured restore and signed review. Offline tests validate policy logic only and do not count as live platform evidence.

## 12. Part 07 completion criteria

Chapters 051–058 cover architecture, object storage, parallel filesystems, dataset loading, checkpoint design, recovery, storage economics and integrated reliability. **Publication completion is distinct from operational acceptance**: live labs, independent security/technical review and DR evidence remain pending.

## References

- [Amazon FSx for Lustre](https://docs.aws.amazon.com/fsx/latest/LustreGuide/what-is.html)
- [Amazon S3 performance](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html)
- [PyTorch Distributed Checkpoint](https://docs.pytorch.org/docs/stable/distributed.checkpoint.html)
- [PyTorch DataLoader](https://docs.pytorch.org/docs/stable/data.html)
- [Kubernetes Storage](https://kubernetes.io/docs/concepts/storage/)
- [OpenTelemetry](https://opentelemetry.io/docs/)

**Next: Chapter 059 — Distributed Training Architecture (Part 08).**
