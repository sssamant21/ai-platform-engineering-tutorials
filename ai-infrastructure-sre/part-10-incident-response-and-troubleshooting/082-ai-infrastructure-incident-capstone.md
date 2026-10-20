# Chapter 082 — AI Infrastructure Incident Capstone

**Part 10 — Incident Response and Troubleshooting | Final Enterprise Capstone**

**Status:** Documentation published. Offline lab execution, live staging game day, security review and independent technical acceptance are **pending**. **Part 10 publication is complete; runtime validation is not.**

## 1. Capstone mission

Lead a realistic, multi-layer GPU infrastructure incident from detection through safe restoration, root-cause analysis (RCA), corrective actions (COA) and production readiness acceptance. The assessment integrates Chapters 075–081: GPU node health, CUDA initialization, GPU OOM, NCCL hangs, RDMA/RoCE fabric, storage/checkpoints, Xid/ECC hardware escalation and runbook governance.

The goal is **not** to recover at any cost. The goal is to restore customer SLOs **without losing verified checkpoints, violating change controls or harming shared GPU tenants**, while distinguishing symptoms from causes.

### Learning outcomes

- Declare severity and assign incident commander, GPU/platform, ML workload, network, storage, security and vendor owners.
- Establish the first-failure timeline across Kubernetes, CUDA, NCCL, GPU, fabric and storage.
- Distinguish root cause, trigger, contributing factors, downstream symptoms and unknowns.
- Choose reversible mitigation, verify checkpoint integrity and validate restored SLOs.
- Produce an audit-ready incident package, COA register and 100-point acceptance score.

## 2. Architecture and scenario

\`\`\`text
Kubernetes GPU cluster
  ├─ training controller → 8 ranks across 4 GPU nodes
  │   ├─ CUDA + NCCL collectives
  │   ├─ RDMA/RoCE fabric
  │   └─ checkpoint writer → object store / shared filesystem
  ├─ inference deployment → separate serving pool
  ├─ DCGM / Prometheus / logs / traces
  └─ incident response: IC + platform + network + storage + ML owner
\`\`\`

**Synthetic fixture (not a real incident):** A training job loses progress while inference p99 degrades. A recent node image change coincides with CUDA initialization failures on one node; a different rank reports NCCL watchdog timeout; the fabric shows elevated retry counters; checkpoint uploads are intermittently throttled. The assessment requires evidence-based prioritization; **multiple contributing causes are possible**.

### 2.1 UTC incident timeline fixture

| UTC | Observation | Source |
|---|---|---|
| 14:00:00 | GPU node image rollout begins | change log |
| 14:01:10 | GPU device plugin registration warning on node-3 | kubelet/plugin |
| 14:01:35 | rank-5 CUDA initialization error | application log |
| 14:02:10 | NCCL collective progress stops on peers | per-rank trace |
| 14:03:20 | RoCE retry counters rise on a fabric rail | switch/NIC |
| 14:04:00 | checkpoint PUTs receive 503 and retry | object-store telemetry |
| 14:05:30 | training progress alert fires | SLO alert |
| 14:06:15 | inference p99 increases on unrelated serving pool | service telemetry |
| 14:08:00 | incident declared | incident log |
| 14:12:00 | last complete training checkpoint identified | checkpoint manifest |

**Assessment:** The rollout and CUDA failure are an initial hypothesis for the training stall, not confirmed root cause. The inference regression may be a separate incident or shared dependency; investigate independently. The fabric and storage signals need correlation rather than assumption.

## 3. Safety and execution tiers

| Tier | Allowed activity | Evidence |
|---|---|---|
| Offline | synthetic logs, timeline, runbook/RCA tabletop | analytical decisions |
| Staging | bounded GPU smoke, approved fault simulation, checkpoint restore | actual runtime results |
| Production | read-only diagnostics, approved non-disruptive mitigation | audited incident records |
| Restricted | GPU reset, node drain/reboot, driver/firmware, PFC/MTU, rank kill | explicit change/owner approval |

Do not intentionally inject hardware Xid/ECC, fabric congestion or storage failure into production. **Cordon does not evict pods; drain can disrupt running distributed jobs.** Never delete or overwrite the last verified checkpoint. Keep customer model weights, tokens, private topology and tenant identifiers out of public evidence.

## 4. Incident command and escalation

**SEV1:** broad customer-serving outage or critical integrity risk. **SEV2:** material training outage or customer degradation. **SEV3:** isolated recoverable failure. These are examples; the organization severity policy takes precedence.

| Role | Responsibility | Approval boundary |
|---|---|---|
| IC | severity, coordination, updates, closure | incident decisions |
| GPU SRE | device/plugin/driver diagnostics | node maintenance |
| ML owner | model correctness, ranks, checkpoint | training restart |
| Network | RoCE/IB/NIC and switch telemetry | fabric changes |
| Storage | checkpoint durability, object store | restore/storage changes |
| Security | access, log redaction, exposure | evidence sharing |
| Vendor | hardware/firmware diagnostics | supported repair |

Initial incident update must include affected SLO, scope, known/unknown facts, mitigation owner and next update time.

## 5. First 15-minute evidence collection

### Kubernetes and application

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl -n <namespace> get pods -o wide -l <job-label-selector>
kubectl -n <namespace> describe pod <rank-pod>
kubectl -n <namespace> logs <rank-pod> --all-containers=true --timestamps=true --since=2h
kubectl -n <namespace> logs <rank-pod> --previous --timestamps=true
\`\`\`

Capture all ranks, not only the rank that timed out. \`--previous\` may not be available. Record pod UID, node, GPU UUID/MIG, world size, image digest, model version, checkpoint ID and deployment change.

### GPU, fabric and storage (approved host)

\`\`\`bash
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version,memory.used --format=csv
journalctl -k --since '-2 hours' --output=short-iso-precise | grep -Ei 'NVRM|Xid|AER|nvidia|mlx|rdma'
ip -s link
ss -ti
rdma link show
iostat -xz 1 3
\`\`\`

Utilities may be unavailable in restricted environments. Collect two timestamped counter samples to compute rates. RDMA may bypass ordinary TCP counters. Object-store throttling requires provider/application telemetry; disk \`iostat\` cannot measure remote object-store health by itself.

## 6. Hypothesis-driven fault isolation

| Hypothesis | Supporting signal | Falsifying/competing evidence |
|---|---|---|
| Driver/runtime incompatibility after rollout | first CUDA error and image change | healthy CUDA smoke on same exact image/node |
| GPU hardware fault | Xid/ECC/AER or missing GPU | no hardware events, failure follows image |
| Collective mismatch | divergent collective order by rank | all ranks same sequence, one rank crashed first |
| RoCE fabric congestion | retries/ECN/PFC and collective p99 | retries on unrelated rail/time, rank error precedes |
| Storage throttling | checkpoint 503 and retry | training stalled before checkpoint began |
| GPU OOM | CUDA allocation exception, peak memory | CUDA init failure without memory pressure |
| Inference dependency regression | shared network/storage/control plane | serving-only deployment or independent traffic spike |

The earliest timestamp is not necessarily the cause if logs are buffered or clocks skewed. Preserve confidence and missing evidence.

## 7. Recovery plan and change gates

1. **Protect data:** identify complete, compatible checkpoint; verify manifest/commit marker and restoration requirements.
2. **Preserve evidence:** rank logs, kernel Xid/AER, fabric counters, storage errors, changes and UTC timeline.
3. **Contain:** stop new risky rollout or route *new* scheduling to approved healthy capacity; evaluate blast radius.
4. **Choose mitigation:** roll back image/config in controlled canary, repair compatible runtime, shift placement or retry with bounded backoff as evidence warrants.
5. **Approve:** IC and relevant GPU/ML/network/storage owners confirm scope, abort threshold and rollback.
6. **Execute in staging first:** verify CUDA and NCCL smoke, checkpoint restore and inference health.
7. **Recover production:** coordinate full distributed-job restart where required; never kill arbitrary ranks or reset shared GPUs.
8. **Validate:** training step progression, loss/optimizer continuity, checkpoint freshness, inference p99/error rate, GPU/fabric/storage health.
9. **Monitor:** keep heightened monitoring until stable window and SLO recovery.
10. **Review:** publish RCA, COAs, owners and evidence.

**Rollback gate:** stop and revert if canary produces new CUDA/NCCL failures, worsens p99/error rate, loses checkpoint integrity, causes GPU disappearance or affects unrelated tenants.

## 8. Eight required production runbooks

1. **GPU node/device plugin unavailable:** plugin/driver/hardware classification and node return-to-service.
2. **CUDA initialization or compatibility:** version matrix, known-good image and staged rollback.
3. **GPU OOM:** host vs CUDA memory, allocator, MIG and serving KV cache.
4. **NCCL rank/collective hang:** all-rank timeline and coordinated checkpoint-safe restart.
5. **RDMA/RoCE fabric degradation:** NIC/switch telemetry and fabric-owner escalation.
6. **Storage/checkpoint outage:** object-store retries, checksum/commit marker and restore.
7. **Xid/ECC/PCIe hardware:** vendor evidence, quarantine and replacement gate.
8. **Customer-serving SLO regression:** inference error/p99, redundancy and rollback.

Each runbook must contain trigger, owner, preconditions, read-only commands, approval boundary, rollback, validation, escalation and closure.

## 9. Incident RCA and corrective actions

### 9.1 RCA artifact

\`\`\`yaml
incident_id: INC-082-CAPSTONE
status: DRAFT
severity: UNASSESSED
start_utc: null
detect_utc: null
mitigate_utc: null
restore_utc: null
training_impact: null
inference_impact: null
checkpoint_rpo_impact: null
trigger: HYPOTHESIS
root_cause: UNDETERMINED
confidence: LOW
contributing_factors: []
alternative_hypotheses: []
detection_gaps: []
recovery_gaps: []
evidence_links: []
reviewers: []
\`\`\`

Do not state a definitive root cause unless supported by corroborating evidence. Separate training and inference causality if necessary.

### 9.2 COA register

| COA | Preventive action | Owner | Acceptance evidence |
|---|---|---|---|
| 082-01 | GPU image/driver compatibility gate | Platform | CI blocks incompatible version |
| 082-02 | per-rank first-error collection | ML platform | rank timeline reconstructed in drill |
| 082-03 | fabric retry/collective p99 correlation | Network/SRE | synthetic alert delivered |
| 082-04 | checkpoint atomicity and restore validation | Storage/ML | successful staged restore |
| 082-05 | canary and rollback automation | Release/SRE | approved drill passes |
| 082-06 | inference SLO isolation | Serving | independent regression detection |

Each action requires named owner, due date, risk, rollback, acceptance test and verification date. A merged PR alone is not verified prevention.

## 10. Twenty hands-on capstone exercises

1. Inventory cluster, nodes, GPU UUIDs and workload ownership.
2. Define SLOs, severity and incident roles.
3. Capture synthetic Kubernetes node and pod failures.
4. Reconstruct all-rank UTC failure timeline.
5. Classify CUDA initialization versus driver/hardware.
6. Distinguish CUDA OOM from host \`OOMKilled\`.
7. Analyze NCCL collective sequence and first failed rank.
8. Analyze RoCE/RDMA errors and counter deltas.
9. Analyze checkpoint 503 retries and storage latency.
10. Investigate inference p99 as separate/shared dependency.
11. Form competing hypotheses with confidence.
12. Build GPU/node and rank-to-NIC topology map.
13. Verify checkpoint manifest and staged restore.
14. Run bounded CUDA smoke on dedicated staging GPU.
15. Run two-GPU NCCL smoke in approved staging.
16. Tabletop cordon/drain and rollback approvals.
17. Produce all eight recovery runbooks.
18. Draft RCA with evidence and unknowns.
19. Register six COAs with measurable verification.
20. Conduct independent review, score and closure decision.

## 11. Twelve failure-injection/tabletop scenarios

| # | Scenario | Required proof |
|---|---|---|
| 1 | device plugin loses GPU allocatable | driver vs plugin classification |
| 2 | CUDA image incompatible with host driver | safe version rollback |
| 3 | one MIG rank OOMs | slice capacity evidence |
| 4 | Xid 79 precedes NCCL timeout | first cause vs downstream symptom |
| 5 | collective mismatch | rank order comparison |
| 6 | missing rendezvous rank | DNS/port/world-size diagnosis |
| 7 | RoCE congestion | fabric counters and owner escalation |
| 8 | data-loader straggler | CPU/storage wait evidence |
| 9 | checkpoint object-store throttling | bounded retry and RPO |
| 10 | corrupted/incomplete checkpoint | reject restore and protect prior |
| 11 | inference p99 regression during training incident | shared vs independent dependency |
| 12 | rollback causes new GPU registration errors | abort gate and escalation |

Use **synthetic fixtures** for unsafe failures. Do not inject real hardware faults, packet loss or data corruption into production.

## 12. Communication artifacts

**Initial:** impact, affected services, declared severity, owner, mitigation, uncertainty, next update.

**Update:** changes in training progress, inference SLO, checkpoint safety, fabric/device evidence and planned change approval.

**Resolution:** UTC recovery time, restored SLIs, remaining risks, customer impact, RCA confidence and COA follow-up.

## 13. Enterprise acceptance (100 points)

| Gate | Points |
|---|---:|
| Incident command, safety and severity | 10 |
| Cross-layer evidence and UTC timeline | 20 |
| Fault isolation and competing hypotheses | 15 |
| Eight recovery runbooks and rollback | 20 |
| Checkpoint integrity and SLO restoration | 15 |
| RCA, COAs and governance | 10 |
| Staging drills and independent review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 **and** no critical security/privacy, unsafe hardware/network change, data-integrity or unapproved production impact. Publication is not evidence of executed labs. Independent reviewer must verify actual staging artifacts before marking runtime acceptance.

## 14. Part 10 closure

Chapters 075–082 are **published (8/8)**. Part 10 is **publication-complete**, but offline execution, live staging drills, technical/security review and operational acceptance remain open. Track these gates independently; do not claim operational readiness solely from documentation.

## 15. References

- [Google SRE — Managing Incidents](https://sre.google/sre-book/managing-incidents/)
- [Google SRE — Postmortem Culture](https://sre.google/sre-book/postmortem-culture/)
- [NVIDIA Xid Errors](https://docs.nvidia.com/deploy/xid-errors/)
- [NVIDIA NCCL Troubleshooting](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/troubleshooting.html)
- [PyTorch Distributed](https://docs.pytorch.org/docs/stable/distributed.html)
- [Kubernetes Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)

**Next:** Chapter 083 — GPU Demand Forecasting (Part 11 — Capacity, FinOps and Reliability).
