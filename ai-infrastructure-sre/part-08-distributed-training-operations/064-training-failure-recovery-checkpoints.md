# Chapter 064 — Training Failure Recovery and Checkpoints

**AI Infrastructure Engineering & SRE · Part 08 — Distributed Training Operations · Chapter 064/100**

> **Validation status:** Production design and offline synthetic policy only. Live GPU recovery, cloud failover and independent operational acceptance remain pending.

## Learning objectives

Design coordinated recovery for multi-node PyTorch DDP/FSDP, Ray and Slurm training; distinguish transient rank failure, node loss, preemption, fabric partitions, data corruption and checkpoint incompatibility. Enforce globally consistent checkpoints, optimizer/RNG/sampler replay, retention dependencies, durable copies, RPO/RTO and SRE incident governance.

**Prerequisites:** Chapters 055–056 and 059–063; Kubernetes GPU operations, Slurm/Ray, NCCL, distributed storage and cloud identity.

## 1. Recovery reference architecture

```text
training job + immutable dataset + pinned image
            |
 rank/process group + rendezvous + watchdog
            |
 GPU/NIC fabric / scheduler / node health
            |
 globally coordinated checkpoint barrier
            |
 per-rank shards -> checksums -> durable object archive
            |                  |
       immutable global commit manifest
            |
 retention graph + legal hold + last-known-good
            |
 incident detection -> fence stale membership
            |
 select last verified checkpoint + isolate restore
            |
 validate optimizer/RNG/sampler/world size
            |
 resume + replay consistency + SLO evidence
```

A checkpoint is recoverable only when **all required shards and metadata are durable, mutually consistent and restorable**. A successful file upload or filesystem snapshot alone does not prove application consistency.

## 2. Failure taxonomy and blast radius

| Failure | Primary symptom | Safe initial action | Recovery constraint |
|---|---|---|---|
| Single rank process | collective hang or exit | preserve first-failure logs | coordinated group restart |
| GPU Xid/ECC | node health alarm | isolate affected device | compatible replacement |
| Fabric partition | NCCL timeout | inspect NIC counters | fence split membership |
| Preemption | termination signal | inspect checkpoint commit | bounded grace and requeue |
| Checkpoint corruption | checksum mismatch | reject candidate | last-known-good fallback |
| Data drift | nonreproducible resume | pin manifest version | deterministic replay |
| Object/KMS outage | archive inaccessible | inspect identity and storage | independent copy and permissions |
| Zone loss | compute and storage unavailable | initiate approved DR | measurable RPO/RTO |

## 3. Checkpoint state contract

Persist the model parameters, optimizer and scheduler, gradient scaler if used, global step/epoch, RNG states for relevant libraries/devices, sampler state or a replay-safe equivalent, immutable dataset manifest, training configuration, code/image digest, world-size and parallelism topology, framework versions, shard digests and a **commit marker written only after complete verification**.

For FSDP, tensor or pipeline parallelism, choose a checkpoint format explicitly supporting required resharding. Never assume arbitrary world-size changes can restore the same optimizer state. Maintain a dependency graph for incremental checkpoints and preserve an independent last-known-good.

## 4. Recovery state machine

```text
RUNNING -> SUSPECT -> FAILED -> FENCED -> SELECT_CHECKPOINT
 -> VERIFY_MANIFEST -> RESTORE_ISOLATED -> VERIFY_REPLAY
 -> REJOIN / RESUME -> OBSERVE -> CLOSED
                   \-> REJECT -> ROLLBACK_TO_LKG
```

The recovery controller must be idempotent: do not allow old ranks to publish new checkpoints after fencing. Use monotonic job generation IDs and immutable commit manifests. Record evidence for each transition.

## 5. PyTorch checkpoint pattern (illustrative)

```python
# Conceptual per-process checkpoint state. Do not use torch.load on untrusted files.
state = {
    "model": model.state_dict(),
    "optimizer": optimizer.state_dict(),
    "scheduler": scheduler.state_dict(),
    "step": global_step,
    "dataset_manifest_sha256": dataset_manifest_sha256,
    "rng_cpu": torch.get_rng_state(),
}
# Use the supported torch.distributed.checkpoint API for sharded,
# coordinated distributed save/load. Verify all ranks and durable
# commit metadata before promoting any checkpoint.
```

In PyTorch, checkpoint format, optimizer state remapping and distributed save/load APIs are version-sensitive. Test exact versions and restoration semantics in staging.

## 6. RPO, RTO and capacity mathematics

```text
RPO_minutes = incident_time - last_verified_durable_commit_time
RTO_minutes = detect + fence + provision + download + verify + load + resume
checkpoint_write_GBps = checkpoint_GB / allowed_write_seconds
expected_lost_GPU_hours = GPUs * lost_training_minutes / 60
restore_read_GBps = checkpoint_GB / allowed_transfer_seconds
```

**Illustrative case:** 2,400 GB checkpoint; 300-second window implies 8 GB/s sustained write, excluding protocol and metadata overhead. If the latest durable commit is 18 minutes old and policy RPO is 30 minutes, age is within budget. If detection 5 + fencing 5 + provisioning 15 + transfer 30 + verify 10 + load 20 + resume 10 = 95 minutes against 120-minute RTO, the synthetic budget passes. Measure these times during real drills; they are not guarantees.

## 7. Scheduler-specific recovery

**Kubernetes:** GPU node replacement, operator reconciliation, gang admission, PVC/CSI dependencies, termination grace and immutable workload generation. **Slurm:** requeue policy, signal handling, partition compatibility, job accounting and node drain. **Ray:** actor/task retries, GCS lifecycle, idempotent side effects and Ray Train durable checkpoint storage. None automatically guarantees exactly-once training progress.

## 8. Security, observability and FinOps

Protect checkpoint contents using scoped workload identity, KMS, encryption, audit and tenant isolation. Monitor checkpoint age, commit latency, shard failures, archive replication, rank exits, NCCL timeouts, restore p95, RPO/RTO, GPU-hours lost and stale telemetry. Reconcile recovery spend, transfer cost and re-training waste.

## 9. Fifteen troubleshooting scenarios

1. **TS-064-01:** Rank crashes during all-reduce. Identify first-failure evidence, affected SLI, safe containment, checkpoint choice, rollback trigger and owner.
2. **TS-064-02:** NCCL timeout strands surviving workers. Identify first-failure evidence, affected SLI, safe containment, checkpoint choice, rollback trigger and owner.
3. **TS-064-03:** Rendezvous accepts stale membership. Identify first-failure evidence, affected SLI, safe containment, checkpoint choice, rollback trigger and owner.
4. **TS-064-04:** GPU Xid forces node replacement. Identify first-failure evidence, affected SLI, safe containment, checkpoint choice, rollback trigger and owner.
5. **TS-064-05:** Preemption occurs before checkpoint commit. Identify first-failure evidence, affected SLI, safe containment, checkpoint choice, rollback trigger and owner.
6. **TS-064-06:** Rank checkpoint shard is missing. Identify first-failure evidence, affected SLI, safe containment, checkpoint choice, rollback trigger and owner.
7. **TS-064-07:** Checkpoint object checksum mismatches. Identify first-failure evidence, affected SLI, safe containment, checkpoint choice, rollback trigger and owner.
8. **TS-064-08:** Global manifest published before durable upload. Identify first-failure evidence, affected SLI, safe containment, checkpoint choice, rollback trigger and owner.
9. **TS-064-09:** Incremental retention deletes a required base. Identify first-failure evidence, affected SLI, safe containment, checkpoint choice, rollback trigger and owner.
10. **TS-064-10:** Optimizer state incompatible with new world size. Identify first-failure evidence, affected SLI, safe containment, checkpoint choice, rollback trigger and owner.
11. **TS-064-11:** Sampler cursor causes duplicate training examples. Identify first-failure evidence, affected SLI, safe containment, checkpoint choice, rollback trigger and owner.
12. **TS-064-12:** Mixed precision GradScaler state omitted. Identify first-failure evidence, affected SLI, safe containment, checkpoint choice, rollback trigger and owner.
13. **TS-064-13:** KMS permission revoked during cross-zone restore. Identify first-failure evidence, affected SLI, safe containment, checkpoint choice, rollback trigger and owner.
14. **TS-064-14:** Storage 503 throttling extends RTO. Identify first-failure evidence, affected SLI, safe containment, checkpoint choice, rollback trigger and owner.
15. **TS-064-15:** Telemetry disappears during a failed recovery. Identify first-failure evidence, affected SLI, safe containment, checkpoint choice, rollback trigger and owner.

## 10. Ten incident simulations

1. **INC-064-01:** Rank loss mid-step. In isolated staging record UTC timeline, rank/world size, checkpoint age, measured RPO/RTO, lost GPU-hours, RCA and independent reviewer.
2. **INC-064-02:** All-reduce deadlock. In isolated staging record UTC timeline, rank/world size, checkpoint age, measured RPO/RTO, lost GPU-hours, RCA and independent reviewer.
3. **INC-064-03:** Stale rendezvous fencing. In isolated staging record UTC timeline, rank/world size, checkpoint age, measured RPO/RTO, lost GPU-hours, RCA and independent reviewer.
4. **INC-064-04:** Node failure and replacement. In isolated staging record UTC timeline, rank/world size, checkpoint age, measured RPO/RTO, lost GPU-hours, RCA and independent reviewer.
5. **INC-064-05:** Preemption during checkpoint. In isolated staging record UTC timeline, rank/world size, checkpoint age, measured RPO/RTO, lost GPU-hours, RCA and independent reviewer.
6. **INC-064-06:** Partial checkpoint commit. In isolated staging record UTC timeline, rank/world size, checkpoint age, measured RPO/RTO, lost GPU-hours, RCA and independent reviewer.
7. **INC-064-07:** Corrupt checkpoint shard. In isolated staging record UTC timeline, rank/world size, checkpoint age, measured RPO/RTO, lost GPU-hours, RCA and independent reviewer.
8. **INC-064-08:** Broken incremental chain. In isolated staging record UTC timeline, rank/world size, checkpoint age, measured RPO/RTO, lost GPU-hours, RCA and independent reviewer.
9. **INC-064-09:** Cross-zone restore with KMS drift. In isolated staging record UTC timeline, rank/world size, checkpoint age, measured RPO/RTO, lost GPU-hours, RCA and independent reviewer.
10. **INC-064-10:** Monitoring blackout during RPO breach. In isolated staging record UTC timeline, rank/world size, checkpoint age, measured RPO/RTO, lost GPU-hours, RCA and independent reviewer.

## 11. Eight production runbooks

1. **RB-064-A — Rank failure triage:** Identify first failed rank, GPU Xid, watchdog, process exit and blast radius.
2. **RB-064-B — Rendezvous reset:** Fence stale workers and establish supported membership before restart.
3. **RB-064-C — Checkpoint verification:** Verify all shards, manifest commit, digests and durable independent copy.
4. **RB-064-D — Optimizer and sampler restore:** Validate model, optimizer, scheduler, RNG, scaler, data cursor and global step.
5. **RB-064-E — Node replacement:** Drain unhealthy GPU node under authorization and verify compatibility.
6. **RB-064-F — Preemption recovery:** Check termination grace, durable commit, requeue idempotency and lost work.
7. **RB-064-G — Cross-zone disaster recovery:** Restore isolated copy, KMS identity and measured RPO/RTO.
8. **RB-064-H — Incident closure:** Capture UTC evidence, rollback, RCA/COA and independent approval.

## 12. Twelve hands-on labs

1. Draw the recovery state machine and ownership boundaries.
2. Create a synthetic immutable dataset and checkpoint manifest.
3. Calculate checkpoint bandwidth and capacity requirements.
4. Validate rank-shard completeness and global commit.
5. Simulate corrupt checksum and last-known-good fallback.
6. Simulate broken incremental checkpoint retention.
7. Model rank loss, fencing and rendezvous restart.
8. Model preemption grace and lost GPU-hours.
9. Validate optimizer/RNG/sampler replay invariants.
10. Calculate RPO and full detection-to-resume RTO.
11. Run companion fail-closed Python evaluator and pytest suite.
12. Compile evidence, incident report, rollback and independent review.

**Companion:** [Lab 064 — Training Failure Recovery Gate](../labs/064-training-failure-recovery-check.md).

## 13. Enterprise acceptance — 100 points

| Domain | Points |
|---|---:|
| Failure taxonomy, detection and fencing | 20 |
| Checkpoint consistency, retention and dataset replay | 20 |
| RPO/RTO, capacity and cross-zone recovery | 20 |
| Security, observability and incident response | 20 |
| Offline tests, live staging drills and independent signoff | 20 |
| **Total** | **100** |

**Pass:** >=90, no critical correctness/security/recovery blockers, measured staging RPO/RTO, tested rollback and independent signoff. Synthetic policy passing does not establish production acceptance.

## References

- [PyTorch Distributed Checkpoint](https://docs.pytorch.org/docs/stable/distributed.checkpoint.html)
- [PyTorch Distributed](https://docs.pytorch.org/docs/stable/distributed.html)
- [Ray fault tolerance](https://docs.ray.io/en/latest/ray-core/fault_tolerance.html)
- [Slurm preemption](https://slurm.schedmd.com/preempt.html)
- [Kubernetes disruption](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)

**Next: Chapter 065 — Multi-Node Training Performance.**
