# Chapter 055 — Checkpoint Design and Retention

**AI Infrastructure Engineering & SRE | Part 07 — AI Storage and Data Pipelines | Chapter 055/100**

> **Status:** Published reference and synthetic lab. No live distributed checkpoint, restore, storage benchmark, or production signoff is implied.

## Objectives

Design recoverable training checkpoints covering model parameters, optimizer state, schedulers, mixed-precision scaler, RNG states, data-sampler cursor, distributed topology, dataset manifest and code/configuration version. Compare full, sharded, asynchronous, incremental and application-consistent checkpoints; define RPO/RTO, retention, security, capacity, cost, observability and operational approval.

**Prerequisites:** Chapters 051–054, PyTorch distributed basics, Kubernetes storage, object-store versioning and GPU failure domains.

## 1. Reference architecture

```text
distributed training ranks (model / optimizer / RNG / sampler)
                         |
             coordinated checkpoint barrier
                         |
          rank-local staging and immutable shards
                         |
           checksums + metadata + commit manifest
                         |
     parallel FS / local staging -> durable object store
                         |
      immutable release pointer + replication / backup
                         |
         retention policy + legal hold + audit
                         |
     isolated restore -> validation -> coordinated resume
```

A directory of partially written files is **not** a recoverable checkpoint. Only a committed manifest that references verified, durable shards may be marked restorable. For sharded state, include rank/world-size, partition scheme, tensor shapes/dtypes and compatible resharding or restart instructions.

## 2. What must be checkpointed?

| State | Why it matters | Common failure |
|---|---|---|
| Model weights | learned parameters | missing rank shard |
| Optimizer moments | training trajectory | weights-only restart changes convergence |
| Scheduler and global step | learning-rate correctness | duplicate/skipped steps |
| RNG and mixed-precision scaler | reproducibility and stability | nondeterministic replay |
| Data sampler/iterator | exact or bounded replay | duplicate/omitted samples |
| Dataset manifest and code image | immutable input and software identity | incompatible restore |
| Distributed topology metadata | sharded restore mapping | changed world size unsupported |
| Checksums and commit marker | atomic logical visibility | partial checkpoint accepted |

Inference-only artifacts often need a smaller set of state. Distinguish resumable **training checkpoints** from exported **serving models**.

## 3. Full, sharded, incremental and asynchronous designs

**Full:** straightforward but high storage/write cost. **Sharded:** reduces per-rank memory but requires a complete global manifest. **Incremental/delta:** saves changed state, but recovery depends on an intact base and entire delta chain. **Asynchronous:** overlaps I/O with compute, but must snapshot a consistent state and bound memory pressure. A completed background upload is not proof of a globally consistent distributed step.

Choose the design using measured checkpoint pause, bytes written, upload time, recovery dependency depth and RPO. Verify framework API and version compatibility before copying implementation snippets.

## 4. Two-phase publication protocol

1. Coordinator allocates unique checkpoint ID and target training step.
2. All participating ranks quiesce or produce a framework-supported consistent snapshot.
3. Write shards to a unique staging namespace with bounded I/O.
4. Calculate SHA-256 and size for each shard; include metadata and immutable dataset reference.
5. Verify the expected rank/shard set, storage durability and checksum.
6. Commit an immutable manifest; atomically/conditionally update the latest-known-good pointer.
7. Mark completion in job telemetry only **after** commit and durability verification.
8. Preserve prior checkpoint until new one passes isolated restore validation.

Never assume POSIX rename semantics exist in object storage. Use immutable objects plus a conditional pointer operation appropriate to S3/Blob; handle concurrent writers and retries explicitly.

## 5. RPO, RTO and checkpoint interval

```text
RPO_minutes = now - timestamp_of_latest_verified_durable_checkpoint
RTO_minutes = detect + provision + fetch + validate + rehydrate + resume
checkpoint_write_GBps = total_checkpoint_GB / allowed_write_seconds
capacity_GB = retained_full_GB + retained_delta_GB + staging + headroom
```

**Synthetic sizing:** A 2,400 GB distributed checkpoint written in 300 s requires 8 GB/s sustained end-to-end write bandwidth, excluding protocol overhead and contention. A 15-minute checkpoint interval does not guarantee 15-minute RPO if upload/verification takes longer or a checkpoint fails.

For incremental chains, a missing base or delta invalidates dependent recovery points. Set a maximum chain depth and periodic full checkpoints. Coordinate retention with replication lag, legal hold and rollback windows.

## 6. Kubernetes, storage and failure domains

Review GPU pod scheduling, PVC/CSI semantics, filesystem mount options, local NVMe staging, FSx for Lustre throughput, object-store request quotas, service identity, AZ placement and network contention with NCCL collectives. A Kubernetes VolumeSnapshot is not automatically an application-consistent distributed training checkpoint. Model simultaneous checkpoint bursts across jobs and enforce admission/IO budgets.

## 7. Security and retention

Encrypt in transit and at rest, restrict checkpoint writer/reader roles, separate training and restore identities, use immutable object versions where supported, audit access and protect secrets/PII in optimizer or training state. Apply tiered retention: recent frequent checkpoints, daily restore points, longer-lived milestone checkpoints and independently recoverable offsite copies as required.

**Deletion guard:** no cleanup without data-owner approval, verified durable alternatives, dependency-chain analysis, legal-hold check, grace period and rollback record. Never delete the only known-good checkpoint.

## 8. SLOs and observability

Track checkpoint completion success, last verified durable age, bytes and duration, upload retries, write bandwidth, per-rank skew, manifest verification errors, storage capacity, retention drift, replication lag, restore drill duration and telemetry freshness. Alert on **verified** RPO/RTO risk, not merely presence of a new directory. Use UTC and link each checkpoint to job, dataset release, code digest and approval record.

## 9. Production-safe inspection

```bash
kubectl get pods -A -o wide
kubectl get pv,pvc -A
kubectl get events -A --sort-by=.lastTimestamp
df -hT /approved/checkpoint/mount
df -i /approved/checkpoint/mount
find /approved/checkpoint/mount -maxdepth 2 -type f | head -50
```

Run only in authorized contexts. Do not inspect sensitive checkpoint contents or execute deletion, restore or large I/O without explicit staging/change approval.

## 10. Twelve troubleshooting scenarios

1. One rank fails before barrier completion.
2. Missing optimizer shard produces invalid resume.
3. Partial upload incorrectly marked committed.
4. Object checksum mismatch after transfer.
5. Incremental base expired while deltas remain.
6. Dataset manifest no longer matches checkpoint.
7. Asynchronous checkpoint consumes GPU host memory.
8. Parallel filesystem metadata saturates on many shards.
9. Checkpoint bursts congest NCCL traffic.
10. Restore world size incompatible with shard layout.
11. KMS/IAM rotation blocks checkpoint reads.
12. Stale telemetry hides RPO breach.

## 11. Ten incident simulations

**INC-055-01:** rank failure during checkpoint. **02:** missing shard. **03:** corrupted manifest. **04:** object-store throttling. **05:** full staging disk. **06:** premature retention deletion. **07:** AZ loss before replication. **08:** KMS denial. **09:** incompatible restart. **10:** monitoring outage during recovery.

Each drill must capture UTC timeline, impacted training jobs, checkpoint ID/step, data-loss window, RPO/RTO, containment, recovery validation, owner approval and RCA/COA. Destructive drills are staging-only.

## 12. Eight production runbooks

**RB-055-A — Checkpoint barrier:** identify first failed rank and abort partial commit.

**RB-055-B — Integrity:** verify all shards, sizes, hashes and manifest.

**RB-055-C — Storage performance:** inspect write throughput, throttling, metadata and competing NCCL traffic.

**RB-055-D — Restore:** select known-good committed checkpoint; test isolated resume.

**RB-055-E — Retention:** verify base/delta dependencies, legal holds and owner approval.

**RB-055-F — Security:** resolve workload identity/KMS failures without broad permissions.

**RB-055-G — RPO/RTO:** calculate verified checkpoint age and end-to-end recovery time.

**RB-055-H — Change closure:** review evidence, cost, rollback and independent signoff.

## 13. Ten hands-on labs

1. Map checkpoint state dependencies and rank shards.
2. Design a versioned checkpoint key layout.
3. Calculate write throughput and staging capacity.
4. Implement an offline manifest integrity check.
5. Model base/delta chain retention and dependency safety.
6. Simulate a failed rank and partial commit.
7. Simulate checkpoint age and RPO breach.
8. Simulate encryption, owner approval and rollback failures.
9. Run the companion policy gate and pytest suite.
10. Produce an evidence manifest and 100-point acceptance review.

**Companion:** [Lab 055 — Checkpoint Design and Retention Gate](../labs/055-checkpoint-design-retention-check.md).

## 14. Acceptance scorecard — 100 points

| Area | Points |
|---|---:|
| State completeness and consistent publication | 20 |
| Throughput, capacity and retention economics | 20 |
| Security, Kubernetes and governance | 20 |
| RPO/RTO, restore and incident response | 20 |
| Automated tests, staging evidence and independent review | 20 |
| **Total** | **100** |

Require >=90 with all critical integrity, dependency, security, RPO, restore, rollback and owner-approval gates green. A synthetic gate pass is not a production acceptance.

## References

- [PyTorch Distributed Checkpoint](https://docs.pytorch.org/docs/stable/distributed.checkpoint.html)
- [PyTorch Saving and Loading Models](https://docs.pytorch.org/tutorials/beginner/saving_loading_models.html)
- [Amazon S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)
- [Azure Blob versioning](https://learn.microsoft.com/azure/storage/blobs/versioning-overview)
- [Kubernetes Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)

**Next: Chapter 056 — Checkpoint Recovery and Consistency.**
