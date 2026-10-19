# Chapter 053 — FSx for Lustre and Parallel Filesystems

**AI Infrastructure Engineering & SRE · Part 07 — AI Storage and Data Pipelines · Chapter 053/100**

> **Production boundary:** This is a reference curriculum. Lab evaluation is offline and synthetic. Do not provision, delete, import, export, benchmark or modify production filesystems without approval. Service capabilities, deployment types and pricing vary by region and version.

## Learning outcomes

Architect, operate and troubleshoot high-throughput parallel filesystems for distributed AI training. Understand Lustre clients, metadata targets (MDT), object storage targets (OST), object storage servers (OSS), striping, data repository associations, Amazon FSx for Lustre, Kubernetes CSI, failure domains, telemetry, checkpoint durability, identity and recovery. Compare Lustre, BeeGFS, NFS and cloud-managed options.

**Prerequisites:** Chapters 043–052, Linux, networking, Kubernetes storage classes, object-store dataset manifests and GPU data-loader fundamentals.

## 1. End-to-end architecture

```text
  S3 / durable dataset versions + checkpoint archives
                    |
        controlled import/export workflow
                    |
         FSx for Lustre / parallel POSIX
             |              |
       metadata (MDT)    data (OST)
             |              |
     namespace ops     stripe/IO bandwidth
             +-------+------+
                     |
     VPC/network rails + Lustre clients
                     |
       EKS GPU nodes / CSI / POSIX mount
                     |
    local NVMe cache + GPU training workers
                     |
     metrics / IAM / KMS / backup / DR / SLO
```

Parallel storage separates namespace operations from data I/O. A large sequential training read may be limited by OST/network bandwidth; a million-file directory scan may instead bottleneck on metadata operations. The Lustre control/data paths, failure modes and security boundaries are different from S3 object APIs.

## 2. Technology selection

| Technology | Best fit | Operational tradeoff |
|---|---|---|
| Amazon FSx for Lustre | AWS managed high-throughput POSIX and S3-linked datasets | deployment-type capabilities, cost, networking and client compatibility |
| Self-managed Lustre | specialized HPC/AI clusters | server, MDT/OST, upgrade and recovery ownership |
| BeeGFS | parallel filesystem on managed/self-managed compute | metadata/storage server management and integration |
| NFS/EFS/Azure Files | shared file access with simpler administration | metadata/throughput limits for demanding training |
| Azure Managed Lustre | Azure high-performance parallel workloads where available | regional and configuration dependencies |
| Local NVMe | ephemeral per-node hot cache | not a durable source of truth |

No filesystem is universally optimal. Benchmark workload shape, shard sizes, file counts, rank concurrency, checkpoint bursts and GPU stall time.

## 3. Lustre internals and striping

**Metadata:** MDT/MDS handles names, directories, permissions and namespace metadata. **Data:** OST/OSS stores file data, potentially striped across multiple targets. Clients use the Lustre protocol over supported networking. Stripe count and stripe size influence parallel throughput and load balance, but overstriping tiny files can amplify overhead.

On an authorized mounted Lustre client, inspect read-only:

```bash
mount | grep -i lustre
df -hT /approved/lustre/mount
lfs df -h /approved/lustre/mount
lfs getstripe /approved/lustre/mount/approved-file
lfs osts /approved/lustre/mount
stat /approved/lustre/mount/approved-file
```

`lfs setstripe` and mount changes are **mutations**: use only in an approved staging experiment. File and directory striping behavior depends on Lustre version and inherited layout.

## 4. Amazon FSx for Lustre architecture

Choose deployment type and throughput/capacity based on documented service options and recovery objectives. Review scratch versus persistent deployment characteristics, metadata performance, data repository association (DRA), import/export behavior, backup capabilities, VPC/AZ constraints, security groups and compatible Lustre clients. **Do not assume scratch storage offers the same durability or backup as persistent deployment types.**

When S3 is the durable dataset source, track exact manifest versions and verify files after import. Data repository integration does not automatically make every application write an application-consistent checkpoint in S3.

## 5. Kubernetes GPU integration

Review FSx for Lustre CSI driver availability and supported versions, StorageClass, PVC/PV access modes, mount target reachability, pod security, EKS node placement and client compatibility. Verify IAM/Pod Identity/IRSA only for operations that actually use AWS APIs; filesystem mount security and network access still require their own controls.

Read-only commands:

```bash
kubectl get csidrivers
kubectl get storageclass
kubectl get pv,pvc -A
kubectl get pods -A -o wide
kubectl describe pvc -n APPROVED_NAMESPACE APPROVED_PVC
kubectl get events -A --sort-by=.lastTimestamp
```

A `ReadWriteMany` declaration does not override backend, driver or filesystem limitations.

## 6. Performance and sizing

```text
required_read_GBps = gpu_workers * per_worker_read_GBps * burst_factor
required_checkpoint_GBps = checkpoint_GB / target_write_seconds
normal_headroom = provisioned_GBps - concurrent_peak_GBps
degraded_headroom = surviving_GBps - concurrent_peak_GBps
```

**Synthetic example:** 32 GPU workers × 0.20 GB/s × 1.25 burst = 8 GB/s read demand. A 2,400 GB checkpoint in 300 seconds needs 8 GB/s write bandwidth, excluding metadata, network overhead and contention. If these overlap, independently validate aggregate workload and whether read/write share bottlenecks.

Measure sustained and p95/p99 throughput, metadata operations/s, queue wait, OST imbalance, client retransmits/errors, network utilization, checkpoint completion and GPU data-loader wait. Do not confuse advertised maximum throughput with workload-achieved throughput.

## 7. Benchmarking safely

Use a staging clone or synthetic data with approved I/O budgets. Pin dataset shard count, file size, read/write mix, worker concurrency, striping layout, cache warm/cold state, node placement, network and Lustre/client versions. Run tools such as IOR or mdtest only with authorization; they can saturate shared storage. Capture per-target counters, GPU step-time effects and reproducible configuration. Avoid benchmarking directly against production checkpoints.

## 8. Data durability, checkpoints and DR

A POSIX `close()` or successful write does not by itself prove offsite durable checkpoint recovery. Define application-consistent checkpoint completion, checksum and manifest, export/backup completion, object version, RPO and restore RTO. Test a restore to an isolated filesystem and resume a training job. Protect at least one independently recoverable checkpoint from filesystem corruption, deletion, regional outage and lifecycle expiration.

## 9. Security, governance and cost

Restrict VPC routes and security groups, authorize filesystem clients, enforce POSIX permissions and workload identities, encryption at rest/in transit where supported and required, KMS policy, S3 repository access and audit logs. Apply dataset classification and tenant isolation. Track provisioned capacity/throughput, idle filesystem hours, backup and data transfer, metadata overhead, and cost per completed training job. Retention and cleanup require data-owner approval.

## 10. SLOs and observability

Track filesystem mount success, sustained read/write throughput, metadata p95 latency, capacity and inode pressure, client errors, checkpoint completion, S3 import/export lag, GPU data-loader wait, backup age, telemetry freshness and recovery RTO. Correlate by filesystem ID, job, node, mount, client version and UTC. Page on user-impacting degradation, not isolated benign counter fluctuations.

## 11. Twelve troubleshooting scenarios

1. Lustre client version incompatible with filesystem.
2. MDT saturation due to millions of tiny files.
3. OST imbalance from poor stripe layout.
4. EKS PVC stuck Pending due to CSI or topology.
5. Security group blocks Lustre mount connectivity.
6. GPU jobs starve due to cold dataset cache.
7. Checkpoint write saturates shared storage.
8. S3 repository import lags behind manifest release.
9. Failed OST or server degrades available bandwidth.
10. Metadata operation stalls during directory listing.
11. Storage capacity/quotas block checkpoint creation.
12. Telemetry outage hides I/O errors and RPO breach.

## 12. Ten incident simulations

**INC-053-01:** mount outage. **02:** metadata saturation. **03:** OST hot spot. **04:** CSI regression. **05:** security policy drift. **06:** checkpoint corruption. **07:** backup/export failure. **08:** AZ/network rail loss. **09:** unauthorized data access. **10:** monitoring outage during capacity exhaustion.

For every drill record severity, incident commander, UTC timeline, affected jobs, filesystem and S3 versions, SLO impact, containment, rollback, checkpoint validation and RCA/COA. Inject faults only in approved staging.

## 13. Eight production runbooks

**RB-053-A — Mount failure:** inspect CSI, client compatibility, network routes and security groups.

**RB-053-B — Metadata bottleneck:** measure MDT latency and small-file workload; optimize data layout after approval.

**RB-053-C — OST imbalance:** inspect stripe distribution and per-target I/O; test revised layout in staging.

**RB-053-D — GPU starvation:** correlate data-loader wait, cache misses and filesystem/network throughput.

**RB-053-E — Checkpoint:** preserve last good artifact; verify checksum, export and RPO/RTO.

**RB-053-F — Capacity:** review provisioned storage, inode growth, metadata pressure and forecasts.

**RB-053-G — Security:** contain client access drift, audit identity and restore approved controls.

**RB-053-H — DR/change closure:** perform isolated restore, rollback, cost and owner signoff.

## 14. Eight hands-on labs

1. Draw MDT/OST/client/S3 failure domains.
2. Compare FSx deployment types against RPO/RTO requirements.
3. Inventory read-only Lustre and Kubernetes CSI state.
4. Calculate read demand, checkpoint bandwidth and N-1 headroom.
5. Model small-file metadata pressure and stripe imbalance.
6. Simulate mount, import/export and checkpoint failures.
7. Run companion offline readiness policy and pytest cases.
8. Produce evidence manifest, recovery plan and 100-point scorecard.

**Companion:** [Lab 053 — FSx for Lustre and Parallel Filesystems Gate](../labs/053-fsx-lustre-parallel-filesystems-check.md).

## 15. Enterprise assessment — 100 points

| Domain | Points |
|---|---:|
| Lustre architecture, tier selection and layout | 20 |
| Read/write performance, metadata and capacity | 20 |
| Kubernetes, network, security and tenancy | 20 |
| SLO, checkpoint, incidents and disaster recovery | 20 |
| Tests, staging evidence and independent review | 20 |
| **Total** | **100** |

Require >=90 and zero critical blocker for dataset integrity, mount access, tenant isolation, checkpoint restore, N-1, stale telemetry or rollback. Synthetic policy pass is not production certification.

## References

- [Amazon FSx for Lustre User Guide](https://docs.aws.amazon.com/fsx/latest/LustreGuide/what-is.html)
- [FSx for Lustre CSI driver](https://github.com/kubernetes-sigs/aws-fsx-csi-driver)
- [Lustre documentation](https://wiki.lustre.org/)
- [Azure Managed Lustre](https://learn.microsoft.com/azure/azure-managed-lustre/)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

**Next: Chapter 054 — High Throughput Dataset Loading.**
