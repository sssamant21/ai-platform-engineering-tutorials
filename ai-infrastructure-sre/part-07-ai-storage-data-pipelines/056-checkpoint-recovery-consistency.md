# Chapter 056 — Checkpoint Recovery and Consistency

**AI Infrastructure Engineering & SRE · Part 07 — AI Storage and Data Pipelines · Chapter 056/100**

> **Validation boundary:** This chapter and its offline gate are reference material. No GPU restore, cloud failover, production data deletion or independent certification is implied.

## Mission and prerequisites

Recover a distributed training workload from a *verified, durable, globally consistent* checkpoint while meeting measured RPO/RTO and preserving model, optimizer, scheduler, RNG, data-sampler, dataset-manifest and global-step correctness. Build on Chapters 051–055, especially immutable checkpoint manifests and dependency-aware retention.

## 1. Recovery architecture and failure domains

```text
      GPU ranks / distributed process group
          | snapshot and global barrier
   immutable rank shards + SHA-256 checksums
          |          |
  parallel FS      object store / replicated copy
          \          /
          committed manifest and versioned pointer
                      |
       independent integrity / dependency audit
                      |
          isolated restore staging environment
                      |
   framework compatibility + rank/reshard mapping
                      |
   optimizer + RNG + sampler + dataset validation
                      |
     coordinated rendezvous, resume, SLO evidence
```

**Consistency levels:** A filesystem snapshot is not automatically a globally application-consistent training checkpoint. Per-rank successful writes are insufficient if ranks captured different steps. For async snapshots, enforce a framework-supported consistent view and bounded in-flight operations. A commit marker is authoritative only if its referenced objects have been durably verified.

## 2. Recovery state contract

| Component | Acceptance evidence | Unsafe shortcut |
|---|---|---|
| Model tensors | expected shapes, dtypes, hash and shard coverage | load only available ranks |
| Optimizer | state compatible with model/optimizer version | weights-only training restart |
| Scheduler/global step | monotonically correct step and LR | restart scheduler from zero |
| RNG / mixed precision | saved states and reproducibility contract | reuse random seed blindly |
| Sampler/dataloader | committed cursor, epoch, worker/rank assignment | silent duplicate or lost samples |
| Dataset and code | pinned immutable manifest and image digest | read latest mutable prefix |
| Distributed metadata | topology, world size, sharding format | assume automatic resharding |
| Object lineage | exact key, version, checksum, encryption | trust listing order or ETag |

Some algorithms cannot guarantee bit-for-bit replay after world-size changes or nondeterministic kernels. Declare the supported recovery correctness level and test it.

## 3. Fail-closed restore state machine

```text
DISCOVER -> VERIFY_MANIFEST -> VERIFY_SHARDS -> VERIFY_CHAIN
    -> AUTHORIZE -> STAGE -> LOAD -> VALIDATE_STATE
    -> RENDEZVOUS -> RESUME -> PROVE_SLO -> PROMOTE
            \ failure -> QUARANTINE / ROLLBACK
```

1. Freeze affected job and preserve logs, manifest pointer and previous known-good checkpoint.
2. Enumerate committed checkpoint candidates by immutable ID, not directory modification time.
3. Validate manifest signature/provenance, complete rank/shard set, content hashes, sizes and all incremental base/delta dependencies.
4. Verify KMS access, data classification, legal hold, target environment and restore authorization.
5. Stage into isolated storage; record download retries, throughput and per-shard latency.
6. Load model, optimizer, scheduler, RNG, sampler, global step and pinned dataset release.
7. Verify framework/driver/CUDA compatibility and supported sharded-to-new-world-size mapping.
8. Coordinate all ranks through an approved rendezvous; never silently continue surviving ranks when the training framework requires restart.
9. Run a short controlled validation sequence, compare loss/metrics and verify sample accounting.
10. Measure RPO from last *durable committed* checkpoint and RTO from detection to healthy training progress.
11. Promote only after owner and independent review; retain previous known-good checkpoint until rollback window expires.

## 4. RPO/RTO and recovery bandwidth

```text
RPO = incident_time - latest_verified_durable_checkpoint_time
RTO = detection + triage + provisioning + transfer + verification
      + state_load + rendezvous + application_validation
minimum_transfer_GBps = checkpoint_GB / transfer_budget_seconds
```

**Synthetic example:** 2,400 GB restored within a 600-second transfer budget needs 4 GB/s *sustained usable* throughput, before verification, rehydration and rendezvous. A 30-minute overall RTO must reserve time for all phases. Measure p95/p99 per-shard tail latency and rank skew, not only aggregate throughput.

## 5. Cross-node, cross-AZ and cross-region recovery

Plan for GPU SKU differences, CUDA/NCCL compatibility, filesystem and object permissions, object replication lag, checkpoint version availability, dataset co-location, capacity quotas and network egress. An AZ outage can affect storage and training simultaneously; N-1 assumptions must reflect correlated failure domains. Cross-region replica existence does not prove checkpoint consistency or RPO. Test restore on an isolated target, verify checksums, and record actual detect-to-resume duration.

## 6. Kubernetes and PyTorch operations

Read-only triage on authorized infrastructure:

```bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl get jobs -A
kubectl get pv,pvc -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl get nodes -o wide
nvidia-smi -L
```

For PyTorch, use the distributed checkpoint API supported by the pinned version, and verify whether loading across a changed world size is supported for the chosen state representation. Keep restore identity separate from checkpoint writer identity. Do not run `torch.load` on untrusted pickle artifacts; prefer safer formats and provenance verification.

## 7. Security and data governance

Use least-privilege workload identity, scoped KMS decrypt rights, TLS, encrypted durable replicas, immutable versions, signed manifests, audit logs and protected incident evidence. Restore into an isolated namespace or account with explicit egress controls. Never bypass legal hold, delete corrupted evidence before RCA, or broaden credentials to resolve an outage without approval.

## 8. Observability and SLOs

Track last verified committed checkpoint age, shard completeness, checksum failures, delta-chain integrity, restore attempts, transfer GB/s, per-rank readiness, restore time p95, checkpoint RPO, end-to-end RTO, dataset replay correctness and telemetry freshness. Tag metrics with job, checkpoint ID, release, world size, region and UTC. Missing telemetry is unknown, not success.

## 9. Twelve troubleshooting scenarios

1. A rank disappears before checkpoint barrier.
2. One shard upload succeeds but commit marker is missing.
3. Manifest references a corrupted object checksum.
4. Object version is expired by retention.
5. Optimizer state is incompatible with restored model.
6. Restored RNG/sampler cursor duplicates samples.
7. World-size change lacks a supported reshard plan.
8. Storage credentials or KMS access are revoked.
9. Checkpoint restore bandwidth is below RTO requirement.
10. AZ outage removes staging and one replica.
11. Parallel filesystem returns stale or incomplete metadata.
12. Telemetry is stale during a recovery incident.

## 10. Ten incident simulations

1. **INC-056-01:** Rank crash mid-save.
2. **INC-056-02:** Partial multi-object commit.
3. **INC-056-03:** Manifest corruption.
4. **INC-056-04:** Incremental chain loss.
5. **INC-056-05:** KMS denial.
6. **INC-056-06:** Zone-wide storage outage.
7. **INC-056-07:** World-size incompatibility.
8. **INC-056-08:** Restore throughput collapse.
9. **INC-056-09:** Incorrect sampler replay.
10. **INC-056-10:** Monitoring outage during RPO breach.

For each drill record incident commander, blast radius, UTC timeline, first failed rank, checkpoint ID, last durable step, RPO/RTO, containment, rollback, correctness validation and RCA/COA. Fault injection is staging-only with written authorization.

## 11. Eight production runbooks

1. **RB-056-A — Checkpoint discovery:** Locate latest committed manifest, immutable object versions and ownership.
2. **RB-056-B — Integrity triage:** Verify full rank/shard inventory, checksums and dependency chain.
3. **RB-056-C — Contain partial writes:** Quarantine incomplete candidates; preserve last known-good release.
4. **RB-056-D — Distributed restart:** Validate framework version, rank mapping, topology and coordinated rendezvous.
5. **RB-056-E — State correctness:** Check optimizer, scheduler, RNG, sampler, dataset and global step.
6. **RB-056-F — RPO/RTO response:** Measure durable age and full detect-to-resume elapsed time.
7. **RB-056-G — Security and DR:** Restore least privilege, validate KMS and isolated-region copy.
8. **RB-056-H — Change closure:** Collect artifacts, validate rollback, obtain owner and independent approval.

## 12. Ten hands-on labs

1. Design a checkpoint restore state machine and failure domains.
2. Inventory model, optimizer, RNG, sampler and manifest dependencies.
3. Generate immutable synthetic shard manifests with SHA-256.
4. Reject partial and corrupted checkpoint candidates.
5. Simulate broken base/delta retention chains.
6. Calculate transfer budget, checkpoint RPO and complete restore RTO.
7. Simulate world-size incompatibility and rank rendezvous failure.
8. Simulate KMS denial, stale telemetry and unavailable regional replica.
9. Run the companion offline policy gate and fault tests.
10. Prepare an incident timeline, evidence manifest and acceptance review.

**Companion:** [Lab 056 — Checkpoint Recovery and Consistency Gate](../labs/056-checkpoint-recovery-consistency-check.md).

## 13. Evidence manifest

```yaml
assessment_id: CH056-YYYYMMDD
environment: staging
checkpoint_id: <immutable-id>
job_and_dataset_release: <references>
model_optimizer_rng_sampler_versions: <artifact>
manifest_signature_and_shard_checksums: <artifact>
base_delta_dependency_audit: <artifact>
source_and_target_failure_domains: <artifact>
restore_compatibility_and_world_size: <artifact>
last_durable_step_and_rpo: <measurement>
detect_to_resume_rto: <measurement>
sample_replay_and_loss_validation: <artifact>
security_and_owner_approval: <ticket>
rollback_and_independent_review: <signed-evidence>
```

## 14. Enterprise acceptance — 100 points

| Domain | Points |
|---|---:|
| Manifest, shard and state consistency | 20 |
| Restore performance, capacity and compatibility | 20 |
| Security, governance and failure-domain design | 20 |
| RPO/RTO, incident response and correctness | 20 |
| Offline tests, live staging evidence and independent review | 20 |
| **Total** | **100** |

Pass >=90 with **all critical gates green**, measured staging recovery and signed independent review. Offline policy acceptance alone is **not production certification**.

## References

- [PyTorch Distributed Checkpoint](https://docs.pytorch.org/docs/stable/distributed.checkpoint.html)
- [PyTorch distributed training](https://docs.pytorch.org/docs/stable/distributed.html)
- [Amazon S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html)
- [Azure Blob versioning](https://learn.microsoft.com/azure/storage/blobs/versioning-overview)
- [Kubernetes Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)

**Next: Chapter 057 — Storage IOPS, Throughput and Cost.**
