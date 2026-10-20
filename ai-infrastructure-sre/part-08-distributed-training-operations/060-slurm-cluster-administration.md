# Chapter 060 — Slurm Cluster Administration

**AI Infrastructure Engineering & SRE · Part 08 — Distributed Training Operations · Chapter 060/100**

> **Publication boundary:** Architecture, safe operational examples and an offline synthetic gate. No live Slurm cluster, GPU allocation, scheduler failover or independent security acceptance is implied.

## Objectives and prerequisites

Operate Slurm as a production GPU batch scheduler: controller and accounting HA, MUNGE authentication, slurmd lifecycle, GPU GRES, cgroup isolation, partitions/QOS, associations and fair-share, backfill, reservations, preemption, checkpoint/requeue, observability, incident response and change control. Understand where Slurm differs from Kubernetes and Ray. Prerequisites: Chapter 059, Linux, NVIDIA drivers, CUDA/NCCL, GPU topology, storage and distributed checkpoints.

## 1. Production architecture

```text
GitOps / approved training submission / identity
                    |
              sbatch / srun / squeue
                    |
      slurmctld primary + backup controller
                    |
          slurmdbd + accounting database
                    |
       MUNGE authentication / policy boundary
                    |
 partitions, accounts, QOS, priority, backfill
                    |
      +-------------+--------------+
      | slurmd A    | slurmd B     |
      | 8 GPUs GRES | 8 GPUs GRES  |
      | cgroup      | cgroup       |
      | NCCL / RDMA / topology     |
      +-------------+--------------+
                    |
 dataset FS / checkpoint archive / telemetry
                    |
 on-call / incident response / chargeback / DR
```

Use vendor- and version-appropriate Slurm configuration. Controller HA, slurmdbd and accounting database recovery have distinct failure semantics; do not assume a backup controller automatically protects every dependency.

## 2. Components and configuration ownership

| Component | Responsibility | Critical evidence |
|---|---|---|
| `slurmctld` | scheduling, node/job state | controller health and failover |
| `slurmd` | per-node daemon | registration and health |
| `slurmstepd` | step execution | job logs and process cleanup |
| `slurmdbd` | accounting service | database health and audit |
| MUNGE | authentication | key distribution and permissions |
| `gres.conf` | GPU resource discovery | GPU count and device mapping |
| `cgroup.conf` | CPU/memory/device enforcement | tenant GPU isolation |
| Partitions / QOS / associations | access, limits, priority | admission fairness and cost |
| Prolog / epilog | approved lifecycle hooks | timeouts and failure handling |

Prefer configuration management, peer review and canary node rollouts. Protect MUNGE keys and Slurm configuration; never paste credentials into tickets.

## 3. GPU GRES and cgroups

The scheduler must accurately map requested GPUs to physical devices. Compare `scontrol show node`, `sinfo`, `gres.conf`, NVIDIA GPU UUIDs and container-visible devices. Use supported `AutoDetect` modes and Slurm version-specific GPU isolation configuration. Incorrect GRES accounting can cause oversubscription, silent GPU invisibility or cross-tenant exposure.

Illustrative snippets **only** (adapt to exact Slurm version and hardware):

```ini
# slurm.conf fragment — not a complete deployable configuration
GresTypes=gpu
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory
TaskPlugin=task/cgroup
# NodeName=... Gres=gpu:8 ...  # exact inventory required
```

```bash
sinfo -Nel
scontrol show nodes
scontrol show partitions
squeue -o "%.18i %.9P %.12j %.8u %.2t %.10M %.3D %R"
sacct -S today --format=JobID,User,Account,Partition,State,Elapsed,AllocTRES
```

These are read-only commands; require authorization and appropriate accounting access.

## 4. Job lifecycle and a safe GPU example

```bash
#!/usr/bin/env bash
#SBATCH --job-name=ddp-smoke
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=1
#SBATCH --gres=gpu:4
#SBATCH --cpus-per-task=16
#SBATCH --time=00:10:00
#SBATCH --output=logs/%x-%j.out

set -euo pipefail
# Adapt partition/account/QOS and CUDA environment to the approved cluster.
srun --ntasks=2 bash -lc 'hostname; nvidia-smi -L'
```

This **only** inventories GPUs; it is not a complete DDP launch. Production launch must set correct task/GPU geometry, rendezvous address, master port, rank mapping and NCCL interface selection. Avoid double-spawning ranks with conflicting `srun` and `torchrun` launch models.

## 5. Partitions, QOS, reservations and fair-share

Define partitions by GPU generation, interconnect and reliability class. Accounts/associations govern who may submit; QOS and TRES limits bound GPU, memory, time and concurrency. Use `sprio`, `sshare`, `squeue --start` and `scontrol show job` to distinguish legitimate pending reasons from incidents. Backfill improves utilization but can cause policy surprises; model reservations and maintenance windows.

## 6. Preemption, checkpointing and recovery

Choose preemption mode and signaling policy based on training framework behavior. A signal does not guarantee that all ranks will commit a consistent checkpoint before termination. Define checkpoint grace, durable object-store copy, RPO, restart topology, dataset cursor and requeue idempotency. `scontrol requeue` and node drain/resume are **mutating** operations; use only under change/incident authorization.

## 7. Slurm and Kubernetes interoperability

Slurm may manage dedicated bare-metal GPU nodes, while Kubernetes manages data services or separate GPU pools. Hybrid solutions require an explicit single scheduling authority per physical GPU to prevent double allocation. Document storage, network, workload identity, monitoring and ownership boundaries; do not assume a Slurm node and a Kubernetes GPU pod can independently claim the same device safely.

## 8. Capacity, SLOs and FinOps

Track GPU-hours allocated vs utilized, queue wait p50/p95, job completion rate, backfill efficiency, fair-share deviation, preemption lost work, checkpoint age, NCCL p95 and $ per successful training step. Attribute usage through Slurm accounting TRES and reconcile with actual cloud/on-prem costs. Define separate SLOs for interactive, reserved and preemptible queues.

## 9. Security and operations

MUNGE key rotation, Slurm service identities, Linux permissions, cgroup device isolation, approved prolog/epilog, container runtime integration, filesystem ACLs and accounting access are production trust boundaries. Limit operator privileges and audit node drains, partition changes, account/QOS changes and credential rotation.

## 10. Fifteen troubleshooting scenarios

1. slurmctld unavailable after controller node failure.
2. slurmdbd outage delays accounting and enforcement.
3. MUNGE key mismatch prevents node registration.
4. GPU GRES inventory differs from nvidia-smi.
5. cgroup device isolation allows wrong GPU visibility.
6. Compute node enters DRAIN due to health-check failure.
7. Job remains PENDING due to association or QOS limit.
8. Backfill and reservation rules starve short GPU jobs.
9. Preemption interrupts a training job before checkpoint commit.
10. Partition topology spans incompatible GPU/NIC fabrics.
11. CUDA/NCCL version drift causes distributed hangs.
12. Node memory pressure triggers OOM despite free GPU memory.
13. Prolog or epilog failure leaves stale allocations.
14. Accounting database lag masks chargeback anomalies.
15. Telemetry outage hides GPU utilization and failed jobs.

For each: symptom, first safe inspection, affected SLI, containment, root-cause hypothesis, recovery and rollback evidence.

## 11. Ten incident simulations

1. **INC-060-01:** Controller failover.
2. **INC-060-02:** Accounting database outage.
3. **INC-060-03:** MUNGE authentication mismatch.
4. **INC-060-04:** GPU GRES misconfiguration.
5. **INC-060-05:** Node drain during active training.
6. **INC-060-06:** Priority/fair-share starvation.
7. **INC-060-07:** Checkpoint/requeue interruption.
8. **INC-060-08:** NCCL collective timeout.
9. **INC-060-09:** Tenant isolation regression.
10. **INC-060-10:** Monitoring outage during queue incident.

Record incident commander, UTC timeline, affected jobs, queue backlog, GPU-hours lost, checkpoint RPO/RTO, security impact and RCA/COA. Disruptive drills are staging-only.

## 12. Eight production runbooks

1. **RB-060-A — Controller and accounting:** Check slurmctld/slurmdbd health, failover, logs and database connectivity.
2. **RB-060-B — GPU node registration:** Compare GRES, cgroups, driver, topology and slurmd status.
3. **RB-060-C — Pending job triage:** Inspect partition, QOS, association, reservations, priority and reason.
4. **RB-060-D — Drain and maintenance:** Preserve checkpoint, cordon/drain affected node, verify safe resumption.
5. **RB-060-E — Distributed hang:** Correlate Slurm allocation, rank mapping, NCCL logs and network counters.
6. **RB-060-F — Preemption and requeue:** Verify signal grace, checkpoint commit and requeue policy.
7. **RB-060-G — Security and fair-share:** Review MUNGE, accounts, tenant isolation and accounting evidence.
8. **RB-060-H — Incident closure:** Measure scheduling SLO and RPO/RTO, validate rollback and review.

## 13. Twelve hands-on labs

1. Diagram controller, accounting, MUNGE and GPU node failure domains.
2. Inventory nodes, partitions, GRES and Slurm service health.
3. Validate GPU UUID/GRES/cgroup mapping in a non-production test cluster.
4. Analyze pending reasons and queue latency.
5. Model fair-share, QOS, backfill and reservation policy.
6. Simulate a controller failover and accounting outage.
7. Simulate GRES mismatch and GPU isolation failure.
8. Model preemption, durable checkpoint RPO and requeue.
9. Simulate distributed NCCL hang and node drain.
10. Calculate N-1 surviving GPUs and queue capacity.
11. Run offline synthetic policy gate and pytest suite.
12. Prepare incident evidence, rollback and independent acceptance review.

**Companion:** [Lab 060 — Slurm Cluster Administration Gate](../labs/060-slurm-cluster-administration-check.md).

## 14. Enterprise acceptance — 100 points

| Domain | Points |
|---|---:|
| Controller, accounting, MUNGE and node lifecycle | 20 |
| GPU GRES, cgroups, partitions and fair-share | 20 |
| Checkpoint, preemption, recovery and SLOs | 20 |
| Security, observability and incident response | 20 |
| Offline tests, live staging validation and independent review | 20 |
| **Total** | **100** |

Require >=90 and no critical HA, GPU isolation, accounting, checkpoint, rollback or reviewer blockers. Offline policy results are not live Slurm validation.

## References

- [Slurm Workload Manager documentation](https://slurm.schedmd.com/documentation.html)
- [Slurm GRES](https://slurm.schedmd.com/gres.html)
- [Slurm cgroups](https://slurm.schedmd.com/cgroups.html)
- [Slurm Accounting](https://slurm.schedmd.com/accounting.html)
- [Slurm Preemption](https://slurm.schedmd.com/preempt.html)

**Next: Chapter 061 — Ray Cluster Operations.**
