# Chapter 066 — Distributed Training SRE Capstone

**AI Infrastructure Engineering & SRE · Part 08 — Distributed Training Operations · Chapter 066/100**

> **Evidence boundary:** Enterprise capstone design, reproducible synthetic assessment and staging procedures. Publication is not proof that a Slurm, Ray, Kubernetes or GPU cluster has been deployed, tested or approved.

## Mission and acceptance

Integrate Chapters 059–065 into an enterprise distributed-training platform with Slurm, Ray/KubeRay, PyTorch DDP/FSDP, NCCL, Kubernetes GPU admission, storage/checkpoints, observability, incident response, security, FinOps and disaster recovery. Produce a 100-point reliability acceptance dossier with independently verifiable evidence.

**Prerequisites:** Chapters 001–065, with emphasis on Parts 06–08. All destructive tests require isolated staging, authorization, rollback and incident communication.

## 1. Enterprise reference architecture

```text
tenant / CI / approved model + dataset manifest
                   |
   identity / RBAC / quotas / cost policy
                   |
  queue and gang admission / fair-share / reservation
                   |
    +--------------+-----------------+
    | Slurm        | KubeRay / K8s   |
    | partitions   | RayJob / Train  |
    | QOS / TRES   | GPU operator    |
    +--------------+-----------------+
                   |
       rank rendezvous + membership fencing
                   |
 GPU workers / NVLink / PCIe / NIC NUMA / RDMA
                   |
    PyTorch DDP/FSDP + NCCL collectives
                   |
 dataset cache + parallel FS + object storage
                   |
 consistent distributed checkpoint + archive
                   |
 DCGM / Prometheus / OTel / NIC / job telemetry
                   |
 on-call / SLO / RPO / RTO / chargeback / DR
```

**Boundary rule:** Do not let Slurm and Kubernetes independently schedule the same physical GPU. Establish a single authoritative GPU allocation model for each node pool, with explicit ownership, identity and failure domains.

## 2. Reference workload and reproducibility

Choose a pinned framework/container digest, driver/CUDA/NCCL versions, accelerator SKU, dataset manifest, precision, global batch, gradient accumulation, world size, rank groups, checkpoint format, and storage/transport paths. Run equivalent single-node and multi-node baselines. Preserve configuration hashes, per-rank logs and UTC timestamps.

## 3. Admission and fairness acceptance

Validate queue policy, tenant GPU quotas, reservations, gang scheduling, priority, backfill, fair-share, preemption grace and cloud quota. Demonstrate that a multi-node job is either admitted with its required topology or remains safely queued; partial admission must not strand GPU allocations. Measure queue wait p95 and starvation.

## 4. Slurm and Ray operational acceptance

For Slurm: inspect controller HA, slurmdbd/accounting, MUNGE, GPU GRES, cgroups, partition/QOS, preemption and requeue. For Ray: inspect KubeRay reconciliation, head/GCS recovery, RayJob, GPU advertisements, placement groups, autoscaling, object store spilling and actor retry semantics. Show which control plane owns each job and the operator's recovery path.

## 5. NCCL, rank geometry and fabric acceptance

```text
global_batch = microbatch_per_rank * gradient_accumulation * data_parallel_replicas
scaling_efficiency = observed_throughput / (world_size * one_gpu_throughput)
collective_fraction = collective_p95_ms / step_p95_ms
N-1_fabric_headroom = surviving_fabric_Gbps - required_fabric_Gbps
```

**Synthetic case:** 32 GPUs × 100 samples/s single-GPU baseline; observed 2,400 samples/s yields 75% scaling efficiency. A 160 Gbps required path with 200 Gbps surviving capacity has 40 Gbps headroom. Measure actual collective p95/p99 and stragglers in staging; do not equate synthetic values with production evidence.

## 6. Dataset and checkpoint correctness

Require immutable dataset version, rank-aware sampler, optimizer/RNG/scheduler state, full rank-shard inventory, checksums, atomic global commit manifest, independent durable archive and retention graph. Reject incomplete or corrupted checkpoints. Validate compatible resharding when world size changes; a restart is not automatically replay-safe.

## 7. Recovery and RPO/RTO

```text
DETECT -> CLASSIFY -> FENCE -> SELECT_LKG
  -> VERIFY_MANIFEST -> RESTORE_ISOLATED
  -> VERIFY_DATA_REPLAY -> RESUME -> OBSERVE -> CLOSE
```

Define checkpoint RPO, full detection-to-resume RTO, preemption grace, node replacement and cross-zone DR. Test loss of rank, GPU node, scheduler head, fabric and object archive independently. Preserve incident timeline, last-known-good and rollback evidence.

## 8. Observability and SLOs

Track GPU SM/memory/Xid, NIC errors, NCCL collective p95, per-rank step skew, data wait, queue p95, placement pending, checkpoint age, restore time, job success, GPU utilization, telemetry freshness and GPU-hours per useful token. Correlate by job, tenant, rank, node, GPU UUID, NIC, dataset manifest and UTC.

## 9. Security, governance and FinOps

Enforce workload identity, tenant RBAC, Kubernetes/Slurm GPU isolation, network policy, scoped storage/KMS access, signed images, change approval and auditable chargeback. Cost optimization must not violate N-1 capacity or recovery commitments.

## 10. Eighteen troubleshooting scenarios

1. **TS-066-01:** Slurm controller failover during multi-node job. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
2. **TS-066-02:** Ray head/GCS failure while actors restart. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
3. **TS-066-03:** Kubernetes gang queue admits partial training workers. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
4. **TS-066-04:** GPU GRES mismatch or device-plugin regression. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
5. **TS-066-05:** NCCL collective hang after one rank exits. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
6. **TS-066-06:** GPU/NIC NUMA mismatch creates stragglers. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
7. **TS-066-07:** Cloud GPU quota blocks autoscaling. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
8. **TS-066-08:** Network partition causes stale rendezvous membership. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
9. **TS-066-09:** Dataset shard replay duplicates training samples. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
10. **TS-066-10:** Checkpoint manifest references missing rank shard. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
11. **TS-066-11:** Incremental retention deletes required base checkpoint. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
12. **TS-066-12:** Preemption grace expires before durable commit. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
13. **TS-066-13:** Cross-zone KMS denial blocks checkpoint restore. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
14. **TS-066-14:** Object-store throttling starves data loader. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
15. **TS-066-15:** GPU thermal throttling masks collective bottleneck. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
16. **TS-066-16:** Tenant bypasses GPU quota and affects fairness. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
17. **TS-066-17:** Stale metrics hide a sustained queue SLO breach. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.
18. **TS-066-18:** GPU-hour chargeback differs from actual allocation. Record symptom, first safe inspection, affected SLI, blast radius, containment, rollback and evidence.

## 11. Ten integrated incident simulations

1. **INC-066-01:** Slurm controller and accounting outage. Run only in authorized staging; capture IC, UTC timeline, queue/job/rank impact, lost GPU-hours, RPO/RTO, SLO and RCA/COA.
2. **INC-066-02:** Ray GCS/head recovery. Run only in authorized staging; capture IC, UTC timeline, queue/job/rank impact, lost GPU-hours, RPO/RTO, SLO and RCA/COA.
3. **INC-066-03:** Gang admission starvation. Run only in authorized staging; capture IC, UTC timeline, queue/job/rank impact, lost GPU-hours, RPO/RTO, SLO and RCA/COA.
4. **INC-066-04:** GPU node failure and rank fencing. Run only in authorized staging; capture IC, UTC timeline, queue/job/rank impact, lost GPU-hours, RPO/RTO, SLO and RCA/COA.
5. **INC-066-05:** NCCL collective deadlock. Run only in authorized staging; capture IC, UTC timeline, queue/job/rank impact, lost GPU-hours, RPO/RTO, SLO and RCA/COA.
6. **INC-066-06:** N-1 fabric degradation. Run only in authorized staging; capture IC, UTC timeline, queue/job/rank impact, lost GPU-hours, RPO/RTO, SLO and RCA/COA.
7. **INC-066-07:** Object-store/dataloader backpressure. Run only in authorized staging; capture IC, UTC timeline, queue/job/rank impact, lost GPU-hours, RPO/RTO, SLO and RCA/COA.
8. **INC-066-08:** Partial checkpoint commit and corrupt shard. Run only in authorized staging; capture IC, UTC timeline, queue/job/rank impact, lost GPU-hours, RPO/RTO, SLO and RCA/COA.
9. **INC-066-09:** Preemption and cross-zone restore. Run only in authorized staging; capture IC, UTC timeline, queue/job/rank impact, lost GPU-hours, RPO/RTO, SLO and RCA/COA.
10. **INC-066-10:** Telemetry/security incident during SLO breach. Run only in authorized staging; capture IC, UTC timeline, queue/job/rank impact, lost GPU-hours, RPO/RTO, SLO and RCA/COA.

## 12. Eight production runbooks

1. **RB-066-A — Incident command and classification:** Assign IC, identify training jobs/ranks and preserve UTC evidence.
2. **RB-066-B — Slurm/Ray/Kubernetes admission:** Inspect controller/GCS/operator, queue, quotas and pending reasons.
3. **RB-066-C — GPU hardware and topology:** Validate GRES/device plugin, GPU UUID, driver, NUMA and NIC.
4. **RB-066-D — NCCL distributed hang:** Find first failed rank, collective, fabric counters and watchdog.
5. **RB-066-E — Checkpoint integrity:** Verify global manifest, all shards, digests and last-known-good.
6. **RB-066-F — Recovery and requeue:** Fence stale workers, restore optimizer/RNG/sampler and verify replay.
7. **RB-066-G — Security and FinOps:** Review workload identity, isolation, quotas, chargeback and spend.
8. **RB-066-H — Reliability acceptance:** Recompute SLO/RPO/RTO, validate rollback, RCA and independent review.

## 13. Fourteen capstone hands-on labs

1. Draw multi-control-plane reference architecture and GPU ownership boundaries.
2. Pin immutable workload, container, dataset and training configuration.
3. Inventory Slurm, Ray/KubeRay and Kubernetes GPU scheduling dependencies.
4. Validate GPU device isolation, NUMA, NVLink and NIC placement.
5. Simulate fair-share, gang admission and quota exhaustion.
6. Measure DDP/FSDP global batch and scaling efficiency.
7. Model NCCL collective and N-1 fabric budgets.
8. Diagnose data-loader and object-store starvation.
9. Validate globally consistent distributed checkpoint manifest.
10. Simulate rank loss, rendezvous fencing and coordinated restart.
11. Simulate Slurm controller and Ray GCS failures.
12. Model preemption, last-known-good and cross-zone RPO/RTO.
13. Execute offline capstone policy and pytest suite.
14. Complete 100-point evidence pack, incident review and signoff.

**Companion:** [Lab 066 — Distributed Training SRE Capstone Gate](../labs/066-distributed-training-sre-capstone-check.md).

## 14. Enterprise 100-point acceptance

| Domain | Points |
|---|---:|
| Architecture, GPU ownership, Slurm/Ray/Kubernetes scheduling | 20 |
| DDP/FSDP, NCCL topology and measured performance | 20 |
| Checkpoint correctness, fencing, recovery and DR | 20 |
| Security, observability, fairness and FinOps | 20 |
| Offline tests, live staging drills, rollback and independent review | 20 |
| **Total** | **100** |

**Required:** >=90, zero critical correctness/security/isolation/recovery blockers, successful live staging acceptance and signed independent review. A passing offline evaluator is only a preliminary policy check.

## References

- [PyTorch Distributed](https://docs.pytorch.org/docs/stable/distributed.html)
- [PyTorch Distributed Checkpoint](https://docs.pytorch.org/docs/stable/distributed.checkpoint.html)
- [NVIDIA NCCL](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [Slurm documentation](https://slurm.schedmd.com/documentation.html)
- [KubeRay](https://docs.ray.io/en/latest/cluster/kubernetes/index.html)
- [Kubernetes scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/)

**Part 08 — Distributed Training Operations complete (8/8 chapters published).**

**Next: Chapter 067 — DCGM Exporter and GPU Metrics (Part 09 — GPU Observability).**
