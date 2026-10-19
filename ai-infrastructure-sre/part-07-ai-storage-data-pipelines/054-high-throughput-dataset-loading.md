# Chapter 054 — High Throughput Dataset Loading

**AI Infrastructure Engineering & SRE · Part 07 — AI Storage and Data Pipelines · Chapter 054/100**

> **Scope:** Production-oriented reference material and an offline synthetic policy lab. Published does not mean that PyTorch, cloud storage, GPU performance or recovery tests have been run.

## Outcomes and prerequisites

Diagnose GPU starvation from dataset loading, architect resilient distributed input pipelines and size readers, caches, CPU, memory, network and storage for a training SLO. Design measurable, secure and reproducible PyTorch DataLoader and distributed data pipelines with failure recovery and cost controls. Prerequisites: Chapters 051–053, CUDA/NCCL, Linux, Kubernetes and object-store manifest concepts.

## 1. End-to-end architecture

```text
versioned object store (S3 / Azure Blob)
            | immutable dataset manifest
   ingestion + checksum + schema validation
            |
  FSx for Lustre / parallel POSIX / object client
            |
    bounded prefetch + shard-aware cache
            |
 CPU workers: decode, tokenize, augment, collate
            |
    pinned host buffers + transfer pipeline
            |
    GPU compute + distributed training ranks
            |
 GPU utilization / step time / loader wait / SLO
```

Every stage can impose backpressure. A low GPU-utilization alert is not proof of a slow storage backend: investigate CPU decode, Python workers, memory, network, PCIe, synchronization, GPU kernel execution and rank skew.

## 2. Dataset design and deterministic sharding

Pin each job to an immutable manifest with object version/checksum and data lineage. Partition data across distributed ranks without unintended overlap or omission. For map-style datasets, use an appropriate distributed sampler and call `set_epoch(epoch)` for epoch-dependent shuffle. For iterable datasets, explicitly shard across ranks **and** DataLoader workers; a sampler alone does not prevent duplicate records. Record seed, epoch, world size, dataset version, worker assignment and last committed progress for reproducible restart.

For uneven shard sizes, account for `drop_last`, padding, duplicate samples and rank completion imbalance. Define correctness requirements before optimizing throughput.

## 3. PyTorch DataLoader patterns

```python
import torch
from torch.utils.data import DataLoader, DistributedSampler

# Map-style dataset example. Initialize distributed process group elsewhere.
sampler = DistributedSampler(
    dataset, num_replicas=world_size, rank=rank, shuffle=True, seed=42
)
loader = DataLoader(
    dataset,
    batch_size=64,
    sampler=sampler,
    num_workers=4,
    pin_memory=torch.cuda.is_available(),
    persistent_workers=True,
    prefetch_factor=2,
    drop_last=False,
)
for epoch in range(num_epochs):
    sampler.set_epoch(epoch)
    for batch in loader:
        # Move tensors to the intended device, then train.
        ...
```

This is a pattern, not a complete runnable training program: `dataset`, `world_size`, `rank` and `num_epochs` are supplied by the application. `persistent_workers` and `prefetch_factor` require positive worker counts; for `num_workers=0`, omit these options. Nonblocking device copies only help when tensors and memory-transfer conditions support them. Tune with evidence; excessive workers can worsen CPU contention, memory pressure and file-descriptor exhaustion.

## 4. Capacity model

```text
required_source_GBps = ranks * samples_per_second_per_rank
                       * bytes_per_sample / 1e9 * miss_fraction
required_decode_cores = aggregate_samples_per_second
                        * cpu_seconds_per_sample
prefetch_memory_GB ~= ranks * workers_per_rank
                     * prefetch_batches * batch_bytes / 1e9
```

**Synthetic example:** 16 ranks × 500 samples/s × 0.5 MB/sample × 25% source-cache misses = 1 GB/s source read demand, excluding metadata and retries. With a 1.3 burst factor, provision at least 1.3 GB/s usable read capacity plus operational headroom. This is decimal MB/GB, not MiB/GiB. Check that checkpoint bursts and other tenants do not consume the same bandwidth budget.

## 5. Cache and prefetch strategy

Choose among object-client buffering, shared parallel filesystem, node-local NVMe, page cache and process memory. Use immutable keys and checksums to avoid stale or poisoned caches. Set cache quotas, eviction rules and admission controls. A cache is never the only durable copy. For repeated epochs, compare cold-cache and warm-cache runs; report both instead of publishing a single optimistic throughput number.

## 6. GPU, CPU and storage observability

| Signal | Meaning | Caveat |
|---|---|---|
| GPU SM utilization / step p95 | end-user compute outcome | kernel bottlenecks can coexist |
| batch wait p95/p99 | input pipeline stall | instrument separately from compute |
| samples/s and epoch time | delivered throughput | exclude warmup consistently |
| cache hit ratio | read amplification | high hit rate can hide cold-start pain |
| object 429/503 / retry rate | remote API backpressure | retry storms worsen saturation |
| FS metadata latency | small-file bottleneck | distinguish from data throughput |
| CPU decode saturation | preprocessing bottleneck | check worker CPU quota and NUMA |
| host RAM / pinned memory | prefetch capacity | OOM can kill ranks |
| PCIe transfer time | H2D bottleneck | pinning alone is insufficient |
| rank skew | distributed straggler | investigate per-rank logs |

Correlate UTC timestamps, dataset version, job, rank, node, storage path and Kubernetes pod. Missing telemetry is **unknown**, not healthy.

## 7. Production-safe diagnostics

```bash
kubectl get pods -A -o wide
kubectl top pods -A --containers
kubectl get events -A --sort-by=.lastTimestamp
nvidia-smi --query-gpu=uuid,utilization.gpu,memory.used --format=csv
df -hT /approved/dataset/mount
df -i /approved/dataset/mount
iostat -xz 1 3
vmstat 1 3
```

Commands depend on installed tools and approved access. For staging-only benchmarks, pin GPU SKU, CPU affinity, worker counts, dataset version, batch size, cache state, object request limits and concurrent jobs. Do not stress shared production storage without a reviewed change.

## 8. Kubernetes and security

Size CPU/memory requests, ephemeral storage, file descriptors and node-local cache. Align GPU, NIC and filesystem topology; prevent one noisy tenant from exhausting I/O or shared cache. Use cloud workload identity, private network paths, least privilege, encryption, dataset access audits and checksums. Never embed static object credentials in manifests. Data retention and deletion require owner approval.

## 9. Failure recovery and SLOs

Track successful batches, p95 batch wait, delivered samples/s, GPU idle fraction, cache integrity, telemetry freshness and checkpoint RPO/RTO. On failure: stop retry storms, preserve manifest and training checkpoint, identify the first failing rank, restore verified dataset version, restart with consistent sampler/epoch state, and prove sample correctness and step-time SLO. A recovered process that silently skips or duplicates data is not a successful recovery.

## 10. Twelve troubleshooting scenarios

1. GPU idle because data-loader workers are too few.
2. CPU decode saturates Kubernetes CPU quota.
3. Excessive workers cause RAM/OOM or file-descriptor exhaustion.
4. FSx/Lustre metadata hot spot from tiny files.
5. Object API throttling triggers retry amplification.
6. Cold cache increases first-epoch p99 latency.
7. Iterable dataset duplicates samples across ranks/workers.
8. DistributedSampler epoch seed not advanced.
9. One rank receives disproportionately expensive samples.
10. PCIe host-to-device copy stalls training.
11. Shared checkpoint burst competes with reads.
12. Missing telemetry masks a stale dataset release.

## 11. Ten incident simulations

**INC-054-01:** remote object throttling. **02:** cache loss. **03:** dataset corruption. **04:** CPU worker OOM. **05:** filesystem metadata outage. **06:** GPU rank straggler. **07:** duplicate/omitted samples. **08:** tenant I/O saturation. **09:** checkpoint restore inconsistency. **10:** monitoring outage during loader SLO breach.

Record severity, incident commander, UTC timeline, job/rank, dataset version, SLO/error-budget impact, root-cause evidence, containment, recovery, owner approval and RCA/COA.

## 12. Eight operational runbooks

**RB-054-A — GPU starvation:** compare batch wait, CPU decode, network and GPU traces.

**RB-054-B — Object throttling:** bound concurrency, inspect 429/503, apply jittered backoff.

**RB-054-C — Metadata bottleneck:** inspect file count, MDT latency and shard layout.

**RB-054-D — Cache failure:** invalidate bad objects and refill from checksum-verified source.

**RB-054-E — Distributed correctness:** verify rank/worker sharding, seeds and sample accounting.

**RB-054-F — Kubernetes pressure:** inspect CPU, memory, ephemeral storage and pod restarts.

**RB-054-G — Checkpoint recovery:** restore dataset manifest and sampler state; validate RPO.

**RB-054-H — Change closure:** compare matched benchmarks, cost, SLO and rollback.

## 13. Eight hands-on labs

1. Draw an end-to-end GPU input pipeline and identify bottleneck boundaries.
2. Calculate samples/s, source-read bandwidth and cache-miss demand.
3. Model DataLoader workers, prefetch memory and CPU decode budget.
4. Design rank- and worker-correct dataset sharding with epoch seeds.
5. Simulate object throttling and cold-cache degradation.
6. Simulate checksum failure, checkpoint age and telemetry outage.
7. Run companion offline fail-closed policy evaluator and tests.
8. Produce benchmark evidence, runbooks and a 100-point readiness score.

**Companion:** [Lab 054 — High Throughput Dataset Loading Gate](../labs/054-high-throughput-dataset-loading-check.md).

## 14. Enterprise acceptance (100 points)

| Category | Points |
|---|---:|
| Dataset integrity and distributed sharding | 20 |
| GPU/CPU/storage performance and capacity | 20 |
| Kubernetes tenancy, identity and security | 20 |
| SLO, incidents, checkpoints and recovery | 20 |
| Offline tests, staging benchmarks and reviewer signoff | 20 |
| **Total** | **100** |

Pass >=90 with zero critical integrity, sharding, tenant isolation, recovery, telemetry or rollback blockers. Offline tests are not a substitute for production-equivalent staging evidence.

## References

- [PyTorch DataLoader](https://docs.pytorch.org/docs/stable/data.html)
- [PyTorch DistributedSampler](https://docs.pytorch.org/docs/stable/data.html#torch.utils.data.distributed.DistributedSampler)
- [AWS FSx for Lustre](https://docs.aws.amazon.com/fsx/latest/LustreGuide/what-is.html)
- [Kubernetes resource management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [NVIDIA DCGM](https://developer.nvidia.com/dcgm)

**Next: Chapter 055 — Checkpoint Design and Retention.**
