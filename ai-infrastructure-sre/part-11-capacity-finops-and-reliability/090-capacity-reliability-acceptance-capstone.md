# Chapter 090 — Capacity and Reliability Acceptance Capstone

**Part 11 — Capacity, FinOps and Reliability | AI Infrastructure Engineering & SRE**

**Status:** DOCUMENTATION PUBLISHED. Execution of the capstone, GPU staging tests, DR game days, cost reconciliation, independent review and operational acceptance remain **PENDING**.

## 1. Mission

Deliver a production-grade, auditable **capacity and reliability acceptance project** for a Kubernetes AI GPU platform. Integrate Chapters 083–089 into one decision package covering demand forecasting, GPU rightsizing, quotas/reservations, N-1 and AZ resilience, unit economics, scaling/admission and disaster recovery.

This capstone is **not passed by publishing Markdown**. Passing requires reproducible evidence, workload-specific SLOs, safe fault injection in an isolated environment, verified costs, security controls, rollback and independent technical approval.

### Target audience

GPU platform SRE, Kubernetes administrators, ML platform engineers, DBRE/data platform owners, FinOps, security, incident commanders and engineering leadership.

### Prerequisites

- Chapter 083 — GPU Demand Forecasting
- Chapter 084 — GPU Rightsizing and Utilization
- Chapter 085 — Capacity Reservation and Quota Governance
- Chapter 086 — N-1 and Multi-Zone GPU Resilience
- Chapter 087 — GPU Cost Allocation and Unit Economics
- Chapter 088 — Scaling and Queue Admission Control
- Chapter 089 — GPU Platform DR and Continuity

## 2. Reference platform

\`\`\`text
                       AI demand forecast / budgets
                                 |
                           policy + approvals
                                 |
          ┌──────────────────────┴──────────────────────┐
          |                                             |
   Region A / primary                            Region B / DR
    AZ-1       AZ-2                              recovery GPU pool
    GPU nodes   GPU nodes                        checkpoint replicas
       |          |                              model artifact mirror
       └── Kubernetes / GPU Operator ───────────────┐
                   |                                |
          ┌────────┴────────┐                       |
          |                 |                       |
     Inference serving   Training queues       DR orchestration
     HPA/KEDA + SLOs     Kueue + gang jobs     RTO/RPO + fencing
          |                 |                       |
          └───────── telemetry / billing / audit ──┘
\`\`\`

**Reference only:** actual region/AZ names, GPU SKUs, cluster and queue APIs, traffic routes, storage replication and security policies must be derived from the target environment. A two-region diagram does not establish that cross-region recovery is feasible.

## 3. Capstone acceptance contract

Define and obtain owner sign-off on:

| Dimension | Required evidence |
|---|---|
| Workloads | named inference and training profiles, owners, demand windows |
| Reliability | inference availability/error/p99, training deadline, RTO/RPO |
| Capacity | SKU/HBM/fabric/zone inventory, reservations and quotas |
| Security | tenant isolation, RBAC, KMS, residency, audit |
| FinOps | approved rate basis, bill reconciliation, useful-work unit costs |
| Change safety | canary plan, blast radius, rollback, approvals |
| Execution | timestamped command output, dashboards, test reports |
| Review | independent reviewer, pass/fail and open corrective actions |

Use **real baseline measurements** for production decisions. Synthetic fixtures in the companion lab are for arithmetic and policy rehearsal only.

## 4. Seven workstreams and deliverables

### WS1 — Forecasting and workload characterization (Chapter 083)

Collect at least one representative demand cycle for training/inference. Segment by GPU SKU, HBM, model version, context length, peak arrival rate, job deadlines and seasonal demand. Record forecast method, confidence bounds, validation error, observed demand and explicit failure headroom.

**Acceptance:** forecast reproduces observed history within preapproved error tolerance and identifies uncertainty, not just a point estimate.

### WS2 — Rightsizing and useful work (Chapter 084)

Correlate allocated GPU-hours, SM activity, HBM occupancy, throughput, p99, errors, training time-to-quality and checkpoint retries. Evaluate alternative GPU SKUs, batching, MIG and full-GPU scheduling; document isolation constraints.

**Acceptance:** staging canary meets model quality and SLO gates; changes have a safe rollback. No downsizing based solely on average GPU utilization.

### WS3 — Quotas, reservations and governance (Chapter 085)

Inventory cloud quotas, physical GPU supply, capacity reservations, billing commitments, namespace ResourceQuotas, queue policies, priorities, tenant floors and borrowing. Verify owners, expiry and audit.

**Acceptance:** no protected workload depends on unverified quota or capacity; P0 priority cannot be self-assigned by untrusted tenants.

### WS4 — N-1 and AZ survivability (Chapter 086)

Calculate **worst-case** node and zone loss by compatible GPU SKU, node bundle, MIG profile, HBM, NIC and storage access. Confirm actual serving placement and training recovery behavior.

**Acceptance:** survivor capacity supports defined protected traffic and p99; checkpointed training resumes within its approved target. A PDB alone is not proof of HA.

### WS5 — Unit economics and cost allocation (Chapter 087)

Reconcile provider invoices, commitment amortization, GPU-hours, failed attempts, storage/network and shared platform overhead to stable tenant/workload owners. Measure cost per successful training step/target quality and per SLO-compliant inference request or million successful tokens.

**Acceptance:** no double-counting, material unexplained variance, false invoice savings or unauthorized chargeback.

### WS6 — Scaling and queue admission (Chapter 088)

Validate HPA/KEDA metric availability, node provisioner compatibility, queue admission and gang scheduling, cold-start delay, bounded overload and safe scale-in. Protect P0 capacity from opportunistic training.

**Acceptance:** realistic peak and overload tests preserve approved SLO/error-budget and fairness policies; scale-in rollback is demonstrated.

### WS7 — DR and business continuity (Chapter 089)

Restore immutable, complete model/checkpoint artifacts, verify hashes and optimizer/RNG/data offsets, confirm recovery GPU SKU and KMS access, test serving traffic promotion and training restart, and rehearse fencing/failback.

**Acceptance:** measured end-to-end RTO/RPO meet approved contracts with no dual writers or unapproved cross-region data movement.

## 5. Quantitative worked example (synthetic)

### Fleet and failure model

- Two AZs, four 8-GPU nodes per AZ: **64 installed GPUs**.
- One node lost: **56 remaining**.
- One AZ lost: **32 remaining**.
- Protected inference demand at failover peak: **24 compatible GPUs**.
- Non-preemptible training demand: **16 GPUs**.

**Result:** 24 inference GPUs fit in the remaining 32 at aggregate level; 24 + 16 = **40 > 32**, so simultaneous non-preemptible training plus protected inference **fails** the AZ-loss aggregate test. Even the inference-only case needs actual placement and load testing.

### Serving capacity

- Synthetic peak traffic: 900 requests/sec.
- Synthetic per-replica capacity at p99 SLO: 150 requests/sec.
- Base: **6 replicas**.
- If one node can host two replicas, a simplified one-node-loss target is **8 ready replicas**.

### Cost

- Training: 8 GPUs × (4 successful + 2 failed hours) = **48 GPU-hours**.
- Fictional rate: $2/GPU-hour; additional costs $12.
- 12,000 successful optimizer steps → **$0.009/step**.
- These calculations do not prove actual cost, capacity or SLO acceptance.

## 6. Staging test plan and gates

| Test | Injection | Pass evidence | Abort condition |
|---|---|---|---|
| Demand spike | synthetic 2× serving load | p99/errors/queue age | customer-like SLO breach |
| GPU rightsizing | canary lower allocation | quality, throughput, headroom | OOM, p99, accuracy regression |
| Quota denial | staging tenant above limit | correct rejection/audit | privilege bypass |
| Queue fairness | P1/P3 competing jobs | policy-compliant admission | P0 starvation |
| Node loss | isolated GPU node unavailable | survivor service and training recovery | missing capacity |
| AZ loss | synthetic isolated zone outage | end-to-end protected SLO | protected traffic failure |
| Cost reconciliation | synthetic/approved invoice ledger | explained variance | double-counting |
| Checkpoint corruption | alter synthetic checkpoint copy | fail-closed checksum gate | invalid restore accepted |
| Regional DR | isolated recovery traffic | measured RTO/RPO, p99 | dual writer, key/residency breach |
| Failback | controlled authority transition | no duplicate outputs | split brain |

**Safety:** do not disable production nodes/AZs, create billable reservations, change live queue quotas or promote real traffic without approved change windows, owners, blast radius and tested rollback.

## 7. Acceptance evidence structure

\`\`\`text
evidence/ch090/
  manifest.yaml
  demand/forecast-baseline.csv
  rightsizing/canary-results.json
  governance/quota-reservation-register.yaml
  resilience/node-az-capacity-matrix.csv
  finops/invoice-reconciliation.csv
  scaling/queue-admission-events.json
  dr/checkpoint-manifest.yaml
  dr/recovery-timeline.csv
  security/review.md
  approvals/acceptance-signoff.md
\`\`\`

Paths are **proposed artifacts**, not claims that evidence exists. Store sensitive billing/customer information only in access-controlled systems, with links/redacted summaries in the repository.

### Evidence manifest template

\`\`\`yaml
chapter: "090"
environment: "STAGING_REQUIRED"
publication: "PUBLISHED"
execution: "NOT_RUN"
gpu_sku: null
clusters: []
forecast_review: "PENDING"
rightsizing_canary: "NOT_RUN"
quota_rbac_review: "PENDING"
n_minus_one_test: "NOT_RUN"
az_loss_test: "NOT_RUN"
invoice_reconciliation: "NOT_RUN"
scaling_overload_test: "NOT_RUN"
checkpoint_restore: "NOT_RUN"
regional_dr_test: "NOT_RUN"
security_review: "PENDING"
independent_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
corrective_actions: []
\`\`\`

## 8. Six enterprise runbooks

**R1 — GPU demand exceeds forecast:** classify traffic/job surge; check queue age, P0 SLO, quotas, reservations and physical stock; enable approved backpressure; escalate procurement; revise forecast.

**R2 — Rightsizing regression:** revert canary, restore GPU/replica floor, verify p99, model quality and training checkpoints; record actual cost and root cause.

**R3 — Tenant quota or queue incident:** distinguish admission denial from scheduling/stock; verify priority/RBAC, borrowing and gang constraints; avoid unsafe preemption.

**R4 — Node/AZ failure:** declare incident, inspect compatible survivor capacity, prioritize inference, restart training from complete checkpoint, validate SLO and recover headroom.

**R5 — GPU cost anomaly:** reconcile rate, commitment, retries, storage/network, workload ownership and successful work; avoid false savings or double-counting.

**R6 — Regional DR activation/failback:** verify incident authority, recovery GPU capacity, artifact hashes, KMS, routing, RTO/RPO, fencing and rollback; document end-to-end evidence.

## 9. Twenty troubleshooting and acceptance scenarios

1. Cloud GPU quota approved but physical stock absent.
2. Capacity reservation matches wrong GPU SKU.
3. Node pool scales but device plugin never registers GPUs.
4. MIG slices stranded while full-GPU job waits.
5. HPA replicas increase while pods remain Pending.
6. Queue admits gang job without compatible topology.
7. Training borrows protected inference headroom.
8. Low GPU utilization hides CPU/data-loader bottleneck.
9. High HBM usage hides long-context p99 regression.
10. GPU-hour savings do not reduce committed invoice.
11. Failed training attempts omitted from cost numerator.
12. Unlabeled workload creates disputed chargeback.
13. N-1 aggregate math passes but node-bundle placement fails.
14. AZ loss leaves insufficient protected GPU capacity.
15. PDB incorrectly treated as involuntary-failure protection.
16. Model artifact mirror lacks KMS permissions.
17. Partial checkpoint accepted without completion marker.
18. World-size change invalidates optimizer resharding.
19. Regional failover exceeds RTO due to model warmup.
20. Failback starts duplicate batch writers.

Each scenario requires: trigger, detection signal, blast radius, root-cause hypothesis, safe mitigation, rollback, evidence and owner.

## 10. Sixteen capstone exercises

1. Approve workload/SLO/DR contracts.
2. Inventory GPU SKU, topology, quotas and reservations.
3. Build and validate a demand forecast.
4. Compute GPU-hour allocation and useful-work baseline.
5. Design and canary rightsizing.
6. Review MIG/time-slicing tenant isolation.
7. Define tenant quota and borrowing governance.
8. Calculate N-1 and worst-AZ survivor capacity.
9. Test serving peak after synthetic node loss.
10. Validate checkpoint-safe training restart.
11. Reconcile invoice and tenant showback.
12. Test queue admission and overload backpressure.
13. Measure GPU provisioning and model warmup.
14. Restore immutable checkpoint and model artifacts.
15. Conduct isolated regional DR/failback tabletop and staged drill.
16. Produce evidence pack, COAs and independent signoff.

## 11. Capstone scoring (100 points)

| Domain | Points |
|---|---:|
| Forecast accuracy and demand coverage | 10 |
| Rightsizing and workload quality/SLO | 15 |
| Quotas, reservations and tenant governance | 15 |
| N-1/AZ survivability and placement | 20 |
| Unit economics and invoice integrity | 10 |
| Scaling, admission and overload safety | 10 |
| DR recovery, checkpoint integrity and RTO/RPO | 15 |
| Audit, rollback and independent review | 5 |
| **Total** | **100** |

**Pass threshold:** ≥85/100 **and every mandatory critical gate passes**: P0 SLO, training integrity, tenant isolation, IAM/KMS, failover capacity, RTO/RPO, rollback, accurate financial claims and independent approval. A score ≥85 cannot override a critical failure.

### Status classification

- **PUBLISHED:** Markdown chapter and companion lab exist.
- **READY_FOR_TEST:** environment, owners, permissions and fixtures approved.
- **TESTED:** timestamped staging execution and evidence collected.
- **REVIEWED:** independent technical/security/FinOps review complete.
- **ACCEPTED:** score and all critical gates passed with authorized signoff.

**Current:** PUBLISHED only. All later statuses are pending.

## 12. References

- [Kubernetes Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Kubernetes Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kueue](https://kueue.sigs.k8s.io/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [FinOps Foundation Framework](https://www.finops.org/framework/)
- [Google SRE Workbook](https://sre.google/workbook/table-of-contents/)

**Part 11 concludes with this chapter.**

**Next:** Chapter 091 — Enterprise AI Infrastructure Reference Architecture (Part 12 — Enterprise Operations and Final Capstones).
