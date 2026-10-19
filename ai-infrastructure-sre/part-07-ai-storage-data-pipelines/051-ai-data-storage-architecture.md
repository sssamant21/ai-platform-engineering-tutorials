# Chapter 051 — AI Data Storage Architecture

**AI Infrastructure Engineering & SRE · Part 07 — AI Storage and Data Pipelines · Chapter 051/100**

> **Publication scope:** Architecture and offline simulation. No live storage benchmark, cloud deployment, recovery drill or independent acceptance has been completed.

## Learning objectives

Design an enterprise data plane for GPU training and inference; select object, file, block and local cache tiers; model dataset ingest, distributed read fan-out and checkpoints; plan Kubernetes storage integration, encryption, access control, capacity, availability, cost, SLOs, failure recovery and operational ownership.

**Prerequisites:** Chapters 001–008, 026–034 and 043–050; Linux filesystem and Kubernetes PVC fundamentals.

## 1. End-to-end reference architecture

```text
source systems / data contracts / lineage / governance
                       |
       object storage: versioned raw + curated data
           (S3 / Azure Blob / compatible)
                       |
    ingest/ETL + metadata + manifests + validation
                       |
      shared high-throughput training file tier
       (FSx for Lustre / Lustre / BeeGFS / Azure
        Managed Lustre where supported)
                       |
     node-local NVMe cache / prefetch / data loader
                       |
          GPU workers / NCCL / training jobs
                       |
     checkpoints / model artifacts / evaluation
                       |
    durable object tier + restore verification
                       |
    observability / security / SLO / FinOps / DR
```

Choose tiers based on access pattern rather than vendor branding. Training can need millions of small metadata operations, large sequential reads, repeated epochs, shuffles, high concurrency and synchronized checkpoint writes. Inference often needs predictable model artifact startup, integrity and cold-start latency.

## 2. Storage classes and tradeoffs

| Tier | Strength | Failure/operations concern |
|---|---|---|
| Object (S3/Blob) | durable scalable datasets and artifacts | request overhead, listing and API costs, not POSIX |
| Parallel POSIX filesystem | shared high-bandwidth training and checkpoints | metadata servers, hot spots, lifecycle, cost |
| Network file/NFS | simpler shared access | server throughput and availability limits |
| Block/PVC | application volumes, stateful workloads | attach topology, access modes, snapshots |
| Local NVMe | high-throughput scratch/cache | ephemeral, node failure, capacity skew |
| Streaming pipeline | continuous ingest | ordering, offsets, replay and backpressure |

Object storage durability and availability are separate from dataset correctness, versioning and recovery. A cache is never the only durable copy.

## 3. Dataset lifecycle and consistency

1. Land source data in an immutable or versioned raw zone.
2. Validate schema, record counts, checksums, PII classification and data contract.
3. Materialize curated training shards and publish a signed manifest.
4. Pin training jobs to a manifest/version rather than mutable prefixes.
5. Prefetch or stage to high-throughput file/cache tier with backpressure.
6. Record data lineage, retention and access approvals.
7. Archive or expire intermediate artifacts under owner-approved policy.
8. Test restore and reproducibility from durable source.

Avoid treating a successful object upload as proof that all readers have a consistent dataset snapshot. Use explicit commit markers and atomic publication strategies appropriate to the backing store.

## 4. Performance engineering

Estimate throughput from aggregate worker demand:

```text
required_read_GBps = workers * per_worker_read_GBps * burst_factor
dataset_epoch_seconds = dataset_size_GB / sustained_read_GBps
checkpoint_seconds = checkpoint_size_GB / sustained_write_GBps
```

**Synthetic example:** 16 workers × 0.25 GB/s × 1.25 burst factor = 5 GB/s required read capacity. A 2 TB dataset at sustained 5 GB/s takes about 400 seconds of transfer, excluding decode, metadata, retries and cache misses. Benchmark p95/p99 read latency, metadata ops, IOPS, throughput, queue depth, cache hit ratio, throttling and GPU data-loader wait.

## 5. Kubernetes integration

StorageClasses, CSI drivers, PVC access modes, PV topology, mount options, workload service accounts and node placement determine correctness and availability. A PVC with `ReadWriteMany` is not automatically supported by every CSI driver. Distinguish Kubernetes volume snapshots from application-consistent checkpoints. Do not mount production object credentials into privileged pods.

Read-only inventory:

```bash
kubectl get storageclass
kubectl get pv,pvc -A
kubectl get csidrivers
kubectl get pods -A -o wide
kubectl describe pvc -n APPROVED_NAMESPACE APPROVED_PVC
kubectl get events -A --sort-by=.lastTimestamp
```

## 6. AWS and Azure patterns

**AWS:** S3 for durable versioned datasets/checkpoints, FSx for Lustre for high-throughput shared training where suitable, EBS for attached block workloads, EFS for managed shared-file use cases and instance-store NVMe for ephemeral caches. FSx for Lustre deployment type, data repository integration, performance and backup capabilities vary by configuration; verify provider documentation and region.

**Azure:** Blob Storage for object datasets, Azure Managed Lustre where available for parallel training, Azure Files or Azure NetApp Files for supported shared-file workloads, managed disks for block storage and local ephemeral disks for cache. Check GPU VM, networking, region, quota, identity and storage feature availability.

Never assume an AWS and Azure storage tier have identical semantics or performance.

## 7. Security and data governance

Use workload identity (IRSA/EKS Pod Identity or Azure Workload Identity where applicable), least-privilege IAM/RBAC, customer-managed keys where required, TLS, private endpoints, network policy, audit trails, retention/legal hold and data residency. Segregate raw, curated, checkpoint and model artifact permissions. Verify deletion/retention controls before cleanup. Enforce data owner approval for index, dataset or checkpoint lifecycle changes.

## 8. Reliability, SLO and disaster recovery

Set explicit objectives for dataset availability, verified manifest freshness, p95 loader wait, sustained read throughput, checkpoint success, checkpoint RPO and restore RTO. Design for object endpoint throttling, parallel filesystem metadata bottlenecks, cache loss, AZ failure, corruption, accidental deletion and credential revocation. Test restore to an isolated environment with checksums and training-job resume; snapshots alone do not prove recoverability.

## 9. Cost and capacity governance

Measure storage bytes by tier, request/transaction cost, cross-AZ/region data transfer, cache hit ratio, parallel filesystem provisioned throughput, idle mounts, checkpoint frequency and retention. Optimize only after confirming SLO and restore objectives. Avoid aggressive lifecycle expiration of the sole recoverable checkpoint.

## 10. Twelve troubleshooting scenarios

1. GPU utilization collapses due to data-loader starvation.
2. Small-file metadata storm saturates shared filesystem.
3. Object-store 429/503 or throttling causes retries.
4. Cross-AZ placement increases latency and transfer cost.
5. PVC remains Pending due to topology or quota.
6. CSI mount failure blocks distributed ranks.
7. Local NVMe cache fills and evicts hot shards.
8. Checkpoint write latency breaches RPO.
9. Partial dataset publication yields mixed epochs.
10. Corrupted shard checksum breaks reproducibility.
11. IAM or KMS change denies dataset access.
12. Filesystem capacity alert is missing due to stale telemetry.

## 11. Ten incident simulations

**INC-051-01:** object API throttling. **02:** metadata server overload. **03:** worker cache loss. **04:** dataset checksum mismatch. **05:** checkpoint write failure. **06:** AZ/storage endpoint outage. **07:** CSI driver regression. **08:** unauthorized data access. **09:** retention policy accidentally targets active checkpoints. **10:** telemetry outage during GPU data starvation.

Capture UTC timeline, affected jobs, bytes and request rates, user-facing SLO impact, containment, restore evidence, owner approval and RCA/COA.

## 12. Eight production runbooks

**RB-051-A — Dataset ingest:** validate manifest, schema, checksums and publication marker.

**RB-051-B — Throughput:** isolate object API, metadata, cache, network or GPU decode bottleneck.

**RB-051-C — Kubernetes mount:** inspect CSI, PVC/PV, access mode, topology and permissions.

**RB-051-D — Checkpoint:** preserve last verified artifact and validate RPO/RTO.

**RB-051-E — Storage capacity:** forecast bytes, inodes, IOPS and burst headroom.

**RB-051-F — Security:** contain credential drift, audit access and restore least privilege.

**RB-051-G — Regional failure:** switch to approved durable copy and verify dataset version.

**RB-051-H — Change closure:** reconcile data-owner approval, cost, SLO, rollback and evidence.

## 13. Eight hands-on labs

1. Draw the tiered AI dataset and checkpoint architecture.
2. Inventory Kubernetes storage classes and CSI capabilities.
3. Model per-worker demand, burst factor and storage headroom.
4. Create a synthetic immutable dataset manifest and checksum workflow.
5. Simulate cache miss, API throttling and metadata hot spots.
6. Simulate checkpoint failure, stale telemetry and AZ capacity loss.
7. Execute the offline fail-closed readiness gate and tests.
8. Produce a 100-point evidence-backed design review.

**Companion:** [Lab 051 — AI Data Storage Architecture Gate](../labs/051-ai-data-storage-architecture-check.md).

## 14. Enterprise assessment — 100 points

| Category | Points |
|---|---:|
| Tier selection, access patterns and dataset correctness | 20 |
| Performance, caching and capacity modeling | 20 |
| Kubernetes, security and data governance | 20 |
| SLO, incident response, checkpoint and DR | 20 |
| Automated tests, live staging evidence and reviewer approval | 20 |
| **Total** | **100** |

Require >=90, no critical data integrity/security/recovery blocker, and signed production evidence. Offline synthetic acceptance is not live certification.

## References

- [AWS FSx for Lustre](https://docs.aws.amazon.com/fsx/latest/LustreGuide/what-is.html)
- [Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [Azure Managed Lustre](https://learn.microsoft.com/azure/azure-managed-lustre/)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes CSI](https://kubernetes-csi.github.io/docs/)

**Next: Chapter 052 — Object Storage and Dataset Layout.**
