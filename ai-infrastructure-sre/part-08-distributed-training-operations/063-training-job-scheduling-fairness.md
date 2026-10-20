# Chapter 063 — Training Job Scheduling and Fairness

**AI Infrastructure Engineering & SRE · Part 08 — Distributed Training Operations · Chapter 063/100**

> **Scope:** Enterprise design and synthetic policy lab. Publication is not evidence of live GPU scheduler validation.

## Objectives

Design reliable and fair admission for distributed GPU training across Kubernetes, Slurm and Ray. Define queue classes, priority, gang scheduling, tenant quotas, reservations, fair-share, backfill, preemption, topology placement, autoscaler capacity and cost-aware admission. Protect checkpoint RPO/RTO while preventing starvation.

**Prerequisites:** Chapters 059–062, GPU scheduling, NCCL, storage/checkpoints and observability.

## 1. Production reference architecture

```text
Users / CI / Training API
         |
 AuthN/AuthZ + tenant quota + budget
         |
 Queue and policy admission
         |
 Fair-share / priority / reservations / aging
         |
 Gang-aware resource allocation
         |
 Kubernetes + Kueue/Volcano (version-specific)
 or Slurm partitions/QOS / Ray placement groups
         |
 GPU node inventory + topology + N-1 capacity
         |
 Node autoscaler / cloud quota / network / storage
         |
 distributed ranks + checkpoint / restart
         |
 Queue SLO / GPU-hour ledger / incident command
```

Avoid concurrent scheduling ownership of the same physical GPU. Gang scheduling is framework-specific; ordinary Kubernetes scheduling does not automatically provide atomic admission for every distributed rank. Kueue, Volcano, Slurm and Ray offer different semantics; select and validate one orchestration contract per workload.

## 2. Queue and tenant classes

| Queue | Objective | Guardrails |
|---|---|---|
| Critical production | predictable start and completion | reserved capacity, bounded priority |
| Interactive research | responsive small experiments | per-user concurrency, short runtime |
| Batch training | high utilization and throughput | fair-share, backfill, checkpoint |
| Preemptible | opportunistic lower cost | explicit RPO, checkpoint grace |
| Maintenance / validation | safe diagnostics | dedicated quota and isolation |

Define ownership, GPU SKU, maximum GPU-hours, job wall time, retry budget, SLO and escalation for each class. Fairness is not necessarily equal instantaneous GPU share: approved weights, usage history and aging can be applied transparently.

## 3. Scheduling policies and tradeoffs

**FIFO** is simple but suffers head-of-line blocking. **Weighted fair-share** accounts for tenant allocations over a rolling period. **Priority** supports critical work but needs aging/starvation safeguards. **Backfill** fits smaller jobs into idle windows without violating reserved start commitments. **Gang admission** prevents half-started multi-node training. **Preemption** reclaims capacity but consumes checkpoints, recovery time and GPU-hours.

Do not assume `PriorityClass` alone provides gang scheduling, or that Ray placement groups alone reserve Kubernetes nodes. For Slurm, use partitions, accounts, QOS, TRES, `sprio`, `sshare` and `squeue` for inspection.

## 4. Kubernetes scheduling example

Illustrative queue quota policy; exact CRD schema depends on the installed Kueue version:

```yaml
# Conceptual policy, not a ready-to-apply Kueue manifest
queue: gpu-batch
tenant: research-a
resourceFlavor: h100-approved
gpuQuota: 32
maxConcurrentJobs: 4
priority: normal
preemption: checkpoint-first
gangAdmission: required
```

Read-only inspection:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get resourcequota -A
kubectl get priorityclasses
kubectl get events -A --sort-by=.lastTimestamp
kubectl get clusterqueues,localqueues -A  # only if Kueue CRDs are installed
```

Never change quotas, priorities, queues or autoscaler limits in production without change authorization and rollback.

## 5. Capacity and fairness mathematics

```text
gpu_demand = sum(admitted_job_gpu_requests)
available_gpus = ready_compatible_gpus - reserved_gpus
n_minus_one_capacity = available_gpus - largest_failure_domain_gpus
admission_ratio = admitted_gpu_requests / eligible_gpu_requests
queue_wait_p95 = percentile(job_start - eligible_submission, 95)
gpu_utilization = useful_gpu_seconds / allocated_gpu_seconds
weighted_share_i = usage_i / approved_weight_i
lost_preemption_work = elapsed_since_last_durable_checkpoint
```

**Worked synthetic example:** 8 GPU nodes × 8 GPUs = 64 GPUs; one-node failure leaves 56. If 16 are reserved for production, only 40 are available for nonreserved admission in N-1 mode. A 48-GPU batch job must wait or use an approved alternative topology. These numbers are illustrative, not a universal capacity target.

## 6. SLOs and starvation protection

Measure queue wait p50/p95 by class, admission failure rate, pending reason, aging, fair-share deviation, preemption frequency, checkpoint age, completion rate, GPU idle, and cost per successful job. Establish alerts for starvation, impossible resource shape, quota exhaustion, topology fragmentation, autoscaler failure, telemetry gaps and N-1 shortfall.

## 7. Preemption and checkpoint safety

Preemption policies must account for job-wide coordination, termination grace, checkpoint write bandwidth, durable commit verification and restart topology. A signal does not prove a globally consistent checkpoint. Define minimum remaining checkpoint time, maximum tolerated lost work and per-class RPO/RTO.

## 8. Security and FinOps

Enforce tenant identity, namespace/Slurm account isolation, budget limits, audit trails, GPU device boundaries and controlled reservation overrides. Reconcile scheduled GPU-hours with actual utilization and cloud billing; cost-aware admission must not silently violate reliability commitments.

## 9. Twelve troubleshooting scenarios

1. **Scenario 1:** Job stuck pending despite idle GPUs. Inspect scheduling events and queue policy read-only, correlate SLI, identify safe containment and document rollback.
2. **Scenario 2:** Gang scheduler admits only some ranks. Inspect scheduling events and queue policy read-only, correlate SLI, identify safe containment and document rollback.
3. **Scenario 3:** GPU SKU mismatch across nodes. Inspect scheduling events and queue policy read-only, correlate SLI, identify safe containment and document rollback.
4. **Scenario 4:** Tenant quota exhausted unexpectedly. Inspect scheduling events and queue policy read-only, correlate SLI, identify safe containment and document rollback.
5. **Scenario 5:** Priority inversion blocks production job. Inspect scheduling events and queue policy read-only, correlate SLI, identify safe containment and document rollback.
6. **Scenario 6:** Fair-share weights not reflected in queue. Inspect scheduling events and queue policy read-only, correlate SLI, identify safe containment and document rollback.
7. **Scenario 7:** Backfill prevents reservation start. Inspect scheduling events and queue policy read-only, correlate SLI, identify safe containment and document rollback.
8. **Scenario 8:** Preemption loses recent optimizer state. Inspect scheduling events and queue policy read-only, correlate SLI, identify safe containment and document rollback.
9. **Scenario 9:** Autoscaler constrained by regional GPU quota. Inspect scheduling events and queue policy read-only, correlate SLI, identify safe containment and document rollback.
10. **Scenario 10:** Node topology fragmentation causes starvation. Inspect scheduling events and queue policy read-only, correlate SLI, identify safe containment and document rollback.
11. **Scenario 11:** Queue telemetry exporter becomes stale. Inspect scheduling events and queue policy read-only, correlate SLI, identify safe containment and document rollback.
12. **Scenario 12:** Two schedulers contend for the same GPU. Inspect scheduling events and queue policy read-only, correlate SLI, identify safe containment and document rollback.

## 10. Ten incident simulations

1. **INC-063-01:** Gang job partially admitted. Capture UTC timeline, tenant impact, queue wait, GPU-hours, checkpoint loss, incident command and RCA.
2. **INC-063-02:** High-priority job starves research queue. Capture UTC timeline, tenant impact, queue wait, GPU-hours, checkpoint loss, incident command and RCA.
3. **INC-063-03:** GPU quota allows tenant monopoly. Capture UTC timeline, tenant impact, queue wait, GPU-hours, checkpoint loss, incident command and RCA.
4. **INC-063-04:** Karpenter cannot provision requested accelerator. Capture UTC timeline, tenant impact, queue wait, GPU-hours, checkpoint loss, incident command and RCA.
5. **INC-063-05:** Slurm fair-share weights drift. Capture UTC timeline, tenant impact, queue wait, GPU-hours, checkpoint loss, incident command and RCA.
6. **INC-063-06:** Backfill reservation delays critical job. Capture UTC timeline, tenant impact, queue wait, GPU-hours, checkpoint loss, incident command and RCA.
7. **INC-063-07:** Preemption kills uncommitted checkpoint. Capture UTC timeline, tenant impact, queue wait, GPU-hours, checkpoint loss, incident command and RCA.
8. **INC-063-08:** Topology fragmentation prevents placement. Capture UTC timeline, tenant impact, queue wait, GPU-hours, checkpoint loss, incident command and RCA.
9. **INC-063-09:** Cloud GPU quota exhausted. Capture UTC timeline, tenant impact, queue wait, GPU-hours, checkpoint loss, incident command and RCA.
10. **INC-063-10:** Scheduler telemetry stale during queue spike. Capture UTC timeline, tenant impact, queue wait, GPU-hours, checkpoint loss, incident command and RCA.

Only inject disruptions in approved staging.

## 11. Eight production runbooks

1. **RB-063-A — Pending queue:** identify reason, resource shape, quota, reservation and eligibility.
2. **RB-063-B — Gang admission:** reconcile expected ranks, scheduler membership and held allocations.
3. **RB-063-C — Fair-share:** inspect usage history, weights, aging and approved overrides.
4. **RB-063-D — Preemption:** verify durable checkpoint and coordinated restart.
5. **RB-063-E — Capacity:** validate GPU SKU, N-1 domain, autoscaler and cloud quota.
6. **RB-063-F — Security:** investigate tenant quota bypass, isolation and audit logs.
7. **RB-063-G — FinOps:** reconcile GPU-hours, waste, reservations and chargeback.
8. **RB-063-H — Incident closure:** confirm SLO restoration, rollback and independent review.

## 12. Twelve hands-on labs

1. Map queue ownership and scheduler control planes.
2. Classify production, research, batch and preemptible jobs.
3. Calculate compatible GPU inventory and N-1 headroom.
4. Simulate gang admission of a 16-GPU job.
5. Model weighted fair-share and queue aging.
6. Analyze priority inversion and starvation.
7. Evaluate reservations and backfill.
8. Calculate preemption checkpoint loss.
9. Simulate GPU quota and autoscaler exhaustion.
10. Calculate queue p95, GPU utilization and cost.
11. Execute offline acceptance policy and pytest.
12. Assemble incident evidence and independent review.

**Companion:** [Lab 063 — Training Job Scheduling and Fairness Gate](../labs/063-training-job-scheduling-fairness-check.md).

## 13. Enterprise acceptance — 100 points

| Domain | Points |
|---|---:|
| Queue architecture, gang admission and ownership | 20 |
| Quotas, priority, fairness and starvation controls | 20 |
| N-1 capacity, preemption and checkpoint safety | 20 |
| SLOs, security, observability and FinOps | 20 |
| Offline tests, staging evidence and independent signoff | 20 |
| **Total** | **100** |

Require >=90, no critical policy failures, measured live staging behavior and signed independent review. Offline passing does not constitute production acceptance.

## References

- [Kubernetes scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [Kueue](https://kueue.sigs.k8s.io/)
- [Volcano](https://volcano.sh/en/)
- [Slurm priority and multifactor](https://slurm.schedmd.com/priority_multifactor.html)
- [Ray placement groups](https://docs.ray.io/en/latest/ray-core/scheduling/placement-group.html)

**Next: Chapter 064 — Training Failure Recovery and Checkpoints.**
