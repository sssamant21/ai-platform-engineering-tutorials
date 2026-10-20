# Chapter 098 — Run Distributed Training Failure Recovery

**Part 12 — Enterprise Operations and Final Capstones | AI Infrastructure Engineering & SRE**

**Publication:** PUBLISHED. Staging failure drills, real distributed recovery measurements, checkpoint restore verification, security review and independent production acceptance **PENDING**.

## 1. Mission

Engineer, rehearse and verify **failure recovery for distributed GPU training** on Kubernetes. Handle GPU/node loss, NCCL hangs, rank divergence, launcher failures, network partitions, storage/KMS outages, checkpoint corruption and cluster-wide disruptions while preserving model state, minimizing lost work and controlling recovery cost.

**Prerequisites:** Chapters 091–097; validated CUDA/NCCL stack, secure GPU Kubernetes cluster, distributed training launcher, durable versioned checkpoint store, training progress telemetry and staging fault-injection approval.

### Outcomes

1. Define recovery boundaries, RPO/RTO and consistency invariants for multi-rank training.
2. Distinguish failure detection, coordinated teardown, checkpoint selection and restart.
3. Validate full-state checkpoints (weights, optimizer, scheduler, RNG, sampler, progress).
4. Recover from rank/node, network, storage and control-plane failures.
5. Prove correctness after restart, not merely Pod readiness.
6. Measure downtime, lost steps, checkpoint age, NCCL health and training throughput.
7. Produce repeatable game-day evidence and corrective actions.

## 2. Distributed training failure model

\`\`\`text
GPU rank / node / NIC / checkpoint store / launcher failure
                        |
             failure detection + fencing
                        |
           stop or reconcile all affected ranks
                        |
             choose last committed checkpoint
                        |
             restore full state and world topology
                        |
             initialize NCCL / process group
                        |
             verify steps, weights and optimizer
                        |
               resume and measure SLO
\`\`\`

**Critical principle:** an individual rank restarting while peers continue may corrupt synchronization or hang collectives. Recovery behavior depends on framework, elastic-training support and job operator; do not assume generic Kubernetes Pod restarts safely repair a distributed job.

## 3. Recovery objectives and invariants

| Objective | Definition | Illustrative design target |
|---|---|---|
| RPO | maximum successful optimizer steps lost after failure | ≤200 steps |
| RTO | fault detection to verified resumed training | ≤15 minutes |
| Checkpoint age | time since last committed, restorable checkpoint | ≤10 minutes |
| State correctness | weights, optimizer, scheduler, RNG, sampler and progress restored | 100% verified |
| Job success | eligible infrastructure-failed jobs restored within RTO | ≥99% |
| Cost | wasted GPU-hours per failure | tracked and bounded |

These are **synthetic design targets**, not actual SLAs or measurements. Approve targets based on model size, checkpoint bandwidth, job cost and recovery requirements.

### State invariants

- All participating ranks agree on global step and checkpoint generation.
- Checkpoint is committed only after all required shards/metadata are durably written.
- No rank reads a partial checkpoint or incompatible model/config version.
- Resume restores optimizer, LR scheduler, random states, data sampler and gradient-scaler state when applicable.
- Repartitioning to a different world size is allowed **only** when the framework/checkpoint format supports it and is tested.
- Output artifacts are versioned to prevent duplicate publication after retries.

## 4. Failure taxonomy

| Failure | Typical signal | Recovery boundary |
|---|---|---|
| GPU Xid/ECC or CUDA fault | driver logs, rank exception | rank group/job |
| Node loss/preemption | NodeNotReady, Pod lost | job group |
| NCCL timeout/hang | collective timeout, no steps | process group |
| Network partition/RDMA issue | transport errors, stalled all-reduce | group/fabric |
| Launcher/controller crash | orphaned ranks, failed reconciliation | operator/job |
| Checkpoint write failure | incomplete manifest, storage errors | last committed generation |
| Checkpoint corruption | hash mismatch, load failure | older verified generation |
| KMS/IAM outage | access denied/decrypt failure | credentials/access path |
| Dataset consistency fault | sampler/progress mismatch | data pipeline |
| AZ/cluster outage | unavailable GPU pool/control plane | DR environment |

## 5. Kubernetes and training operator architecture

Common options include Kubernetes Jobs with a framework-aware launcher, Kubeflow Training Operator, PyTorch elastic orchestration or other approved distributed schedulers. Confirm actual installed CRDs, API versions and controller behavior. **Do not invent operator-specific YAML without version validation.**

Scheduling requirements:
- consistent GPU SKU and CUDA/NCCL compatibility;
- verified rank-to-GPU mapping and NCCL network interfaces;
- queue/gang admission or equivalent all-rank readiness;
- node topology and GPU/NIC affinity;
- sufficient compatible replacement capacity;
- checkpoint/object store IAM and KMS;
- protected serving capacity and training priority policy.

### Read-only discovery

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get jobs -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl api-resources | grep -Ei 'pytorchjob|training|kueue|workload' || true
\`\`\`

Missing CRDs or permissions are **BLOCKED**, not PASS.

## 6. Checkpoint architecture

Use versioned generations and a **commit-last manifest**:

\`\`\`text
checkpoints/
  run-abc/
    step-001000/
      rank-0000.bin
      rank-0001.bin
      optimizer-*.bin
      rng-sampler.json
      hashes.json
      COMMITTED.json
    step-001200/
      ...
      COMMITTED.json
\`\`\`

This is a conceptual layout; frameworks may shard optimizer/model states differently. A valid checkpoint manifest should include run ID, step, world size, framework/checkpoint schema version, model config/hash, shard list/checksums, dataset snapshot, optimizer/scheduler/RNG/sampler metadata and completion status.

**Durable commit protocol:**
1. Write to unique generation prefix.
2. Upload every rank shard and required metadata.
3. Verify object completion and checksums.
4. Publish immutable completion manifest **last**.
5. Confirm readers can retrieve the committed generation.
6. Retain at least one older verified generation.
7. Enforce KMS access, versioning, retention and lifecycle rules.

For object storage, use provider-supported conditional writes/versioning rather than assuming POSIX rename atomicity. A file named \`COMMITTED\` is not proof of completeness without validation.

## 7. Example manifest validation (offline)

This Python pattern validates a **local staged copy** of a checkpoint. It does not authenticate an object-store manifest or prove framework loadability.

\`\`\`python
import hashlib, json
from pathlib import Path

root = Path("/tmp/staging-checkpoint")
manifest = json.loads((root / "COMMITTED.json").read_text())
assert manifest["status"] == "COMMITTED"
assert isinstance(manifest["step"], int) and manifest["step"] >= 0
for entry in manifest["files"]:
    relative = Path(entry["path"])
    assert not relative.is_absolute() and ".." not in relative.parts
    data = (root / relative).read_bytes()
    assert hashlib.sha256(data).hexdigest() == entry["sha256"]
print("CHECKPOINT_FILE_HASHES_PASS")
\`\`\`

**Security note:** validate paths, symlinks, file sizes, manifest signatures and trust boundaries before using this in production. Hash verification alone does not prove optimizer/sampler consistency or model quality.

## 8. Failure detection and fencing

Detect stalls using **successful optimizer step progress**, not Pod Running status alone. Combine:
- step heartbeat and progress timestamps;
- rank health and collective timeouts;
- GPU Xid/ECC and node conditions;
- NCCL error/timeout logs;
- checkpoint commit age and storage failures;
- controller reconciliation and queue admission.

On suspected split-brain/orphaned ranks, fence stale workers or terminate the affected job group according to the training framework's supported procedure. Do not permit old and new rank groups to write the same checkpoint generation or publish the same model artifact.

## 9. NCCL and rank failure triage

### Evidence capture

\`\`\`bash
kubectl get pods -n <TRAINING_NAMESPACE> -o wide
kubectl get events -n <TRAINING_NAMESPACE> --sort-by=.lastTimestamp
kubectl logs -n <TRAINING_NAMESPACE> <RANK_POD> --tail=200
\`\`\`

Use approved namespace/Pod values. Capture rank, world size, node, GPU UUID, driver/CUDA/NCCL versions, selected network transport, last step, checkpoint ID and exact error.

**Triage sequence:** one GPU → same-node group → multi-node fabric → storage/KMS. Distinguish NCCL deadlock, transport fallback, launcher rank mismatch, application collective mismatch and hardware fault. Setting \`NCCL_IB_DISABLE=1\` may force TCP fallback; it is not a universal remediation.

## 10. Controlled staging recovery procedure

1. Declare exercise, freeze nonessential changes and confirm rollback.
2. Start disposable distributed training with deterministic seed and synthetic dataset.
3. Record baseline steps/sec, world size, model hash and checkpoint generation.
4. Wait for a **verified committed checkpoint**.
5. Inject one approved failure (terminate disposable rank Pod, simulate node loss or block a test-only endpoint).
6. Observe detection, group teardown and controller reconciliation.
7. Verify stale workers are fenced and no partial checkpoint is selected.
8. Restore last committed checkpoint to compatible topology.
9. Validate optimizer/scheduler/RNG/sampler state and resumed global step.
10. Compare loss curve/weights and progress against an approved reference within documented nondeterminism tolerance.
11. Measure RPO, RTO, lost GPU-hours and post-recovery throughput.
12. Preserve logs, close exercise and document COAs.

**Abort:** customer SLO risk, unapproved cross-tenant impact, checkpoint corruption outside disposable data, runaway cloud cost or missing rollback.

## 11. Recovery correctness

A passing recovery requires **all** of:
- checkpoint manifest and shard hashes verified;
- framework can load full distributed state;
- global step and data sampler resume consistently;
- optimizer and scheduler states restored;
- NCCL collectives complete without correctness errors;
- training makes new successful optimizer steps;
- loss/metrics within approved reference tolerance;
- output artifacts not duplicated;
- RPO/RTO and cost measured.

A restarted Pod, Running Job or single successful step is not sufficient.

## 12. Failure injection catalog

| Drill | Safe staging injection | Expected validation |
|---|---|---|
| Rank crash | terminate disposable rank Pod | group recovery |
| Node interruption | approved staging node replacement | rescheduling |
| NCCL timeout | test-only network fault or synthetic timeout | bounded detection |
| GPU fault | simulated Xid or disposable CUDA error | hardware classification |
| Checkpoint incomplete | withhold test commit marker | skip generation |
| Checkpoint hash mismatch | corrupt disposable shard copy | reject generation |
| Object-store access denied | scoped test identity | fail safely and recover |
| KMS failure | test-only permission revocation | no plaintext fallback |
| Queue starvation | artificial test quota contention | bounded admission |
| Launcher restart | restart isolated controller/job launcher | reconcile ranks |
| AZ loss | tabletop or isolated cluster simulation | DR plan |
| Cost overrun | synthetic runaway retries | budget abort |

Never disrupt real customer data, shared KMS keys, production fabric or physical GPUs.

## 13. Performance and recovery economics

Estimate wasted compute:

\`\`\`text
lost_GPU_hours = affected_GPUs × (lost_work_seconds + recovery_seconds) / 3600
wasted_cost = lost_GPU_hours × effective_GPU_hour_rate
\`\`\`

Track checkpoint overhead versus recovery savings. Frequent checkpoints reduce RPO but increase I/O and may reduce useful training throughput. Optimize using actual failure rate, checkpoint duration, model state size, object-store costs and recovery tests.

## 14. Six production runbooks

**R1 — Rank crash/GPU Xid:** identify failed rank/GPU, preserve logs, stop group safely, restore committed checkpoint and verify training progress.

**R2 — NCCL hang:** inspect rank/world size, collectives, topology, network and transport; fence stale group and restart only after fault isolation.

**R3 — Checkpoint corruption:** reject incomplete/invalid generation, select older verified checkpoint, inspect storage integrity and commit protocol.

**R4 — Node/AZ loss:** assess compatible GPU capacity and queue, replace failed nodes or fail over to tested recovery cluster, validate data access.

**R5 — KMS/object-store outage:** distinguish IAM/KMS/network from corruption, restore approved access, avoid plaintext fallback, verify checkpoint reads.

**R6 — Training restart loop/cost spike:** stop retry storm, inspect controller backoff, quotas, checkpoint and GPU allocation; resume with guarded retries.

## 15. Sixteen hands-on exercises

1. Define training RPO/RTO and failure budget.
2. Inventory GPU nodes, framework and launcher.
3. Verify distributed rank/GPU/NCCL topology.
4. Define full-state checkpoint schema.
5. Create and hash a synthetic checkpoint.
6. Test commit-last manifest integrity.
7. Verify restore of optimizer/RNG/sampler state.
8. Monitor successful step heartbeats.
9. Inject disposable rank failure.
10. Test safe group teardown and fencing.
11. Simulate NCCL transport timeout.
12. Reject corrupted/incomplete checkpoint.
13. Recreate job on compatible GPU topology.
14. Validate resumed training and reference loss.
15. Measure RPO/RTO, GPU-hours and COAs.
16. Complete independent reliability acceptance.

## 16. Twelve troubleshooting scenarios

| Symptom | Root-cause focus |
|---|---|
| One rank exits, others hang | group reconciliation |
| Pods Running, no steps | NCCL/data stall |
| NCCL timeout after node change | fabric/topology |
| Training resumes at wrong step | checkpoint metadata |
| Optimizer diverges after restore | missing state |
| Checkpoint files exist but restore fails | incomplete manifest |
| Hash mismatch | corruption/version |
| KMS access denied | workload identity |
| Recovery Pod Pending | GPU stock/taints/queue |
| Different world size fails restore | unsupported resharding |
| Duplicate model artifacts | missing fencing/idempotency |
| GPU-hours spike during retries | restart storm |

## 17. Acceptance rubric (100 points)

| Domain | Points |
|---|---:|
| Failure detection, rank coordination and fencing | 20 |
| Checkpoint durability, integrity and full-state restore | 25 |
| GPU/NCCL/node failure recovery | 20 |
| RPO/RTO, progress and model correctness | 20 |
| Cost, runbooks and incident evidence | 10 |
| Independent review and security | 5 |
| **Total** | **100** |

**Pass:** ≥85/100 and no critical checkpoint corruption, split-brain/orphaned writers, incomplete optimizer state, failed NCCL correctness, unverified RPO/RTO or absent independent review. **Documentation publication is not runtime recovery certification.**

## 18. References

- [PyTorch Distributed Elastic](https://pytorch.org/docs/stable/elastic/index.html)
- [PyTorch Distributed Checkpoint](https://pytorch.org/docs/stable/distributed.checkpoint.html)
- [NVIDIA NCCL User Guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kueue](https://kueue.sigs.k8s.io/)
- [Google SRE Workbook — Disaster Recovery](https://sre.google/workbook/disaster-recovery/)

**Next:** Chapter 099 — Perform Enterprise AI Infrastructure Readiness Review.
