# Chapter 081 — Recovery Runbooks and RCA COA

**Part 10 — Incident Response and Troubleshooting | AI Infrastructure Engineering & SRE**

**Status:** Published curriculum. Offline exercises, staging drills, security review and independent technical acceptance are **pending**.

## 1. Mission

Build safe, executable incident recovery runbooks and evidence-based **root cause analysis (RCA)** and **corrective actions (COA)** for GPU infrastructure, Kubernetes, NVIDIA drivers, NCCL, RDMA/RoCE, storage, distributed training and inference serving.

**Outcomes:** distinguish mitigation from permanent correction; declare and manage incidents; preserve checkpoints and customer data; write precise rollback/return-to-service gates; produce blameless causal analysis; prioritize measurable COAs; and verify prevention by staging drills and SLO evidence.

**Prerequisites:** Chapters 075–080; incident command, Kubernetes/GPU administration, SLOs, observability and change management. Production commands below are read-only unless explicitly labeled as gated actions.

## 2. Incident lifecycle and decision rights

\`\`\`text
Detect → Triage → Declare → Contain → Preserve Evidence
  → Mitigate → Validate Service → Recover / Restore
  → RCA → COA → Verify → Close / Track Residual Risk
\`\`\`

| Role | Accountable decision | Handoff evidence |
|---|---|---|
| Incident commander (IC) | severity, coordination, escalation, closure | incident timeline, decisions |
| GPU/platform SRE | GPU/node/driver and runtime diagnosis | UUID, Xid, node state |
| ML workload owner | training/inference impact, checkpoint/restart | checkpoint manifest, model SLO |
| Network/fabric owner | IB/RoCE/NVLink or TCP remediation | link/counter evidence |
| Storage owner | PVC/object store/checkpoint integrity | restore test, latency |
| Security/privacy | access, redaction, incident classification | approved evidence location |
| Vendor/cloud support | supported hardware/firmware repair | case ID, guidance and acceptance |

The IC coordinates; individual owners approve changes in their failure domains. An emergency exception must be recorded with reason, approver, rollback and post-change review.

## 3. Severity and operational guardrails

Define severity from **user impact and recovery urgency**, not only a GPU metric:
- **SEV1:** broad inference unavailability, substantial data-integrity risk or critical training fleet outage.
- **SEV2:** material degradation, lost GPU capacity or failed production jobs with limited redundancy.
- **SEV3:** isolated recoverable failure with no material customer impact.
- **SEV4:** warning or preventive maintenance without service degradation.

These are illustrative; follow the organization's actual severity policy.

**Non-negotiable safety gates:** preserve the last verified checkpoint; protect live model serving replicas and PodDisruptionBudgets; do not force-delete distributed ranks, reset GPUs, reboot nodes, unload drivers, disable ECC or change fabric settings without explicit approval; redact tenant data and credentials; ensure an abort and rollback path.

## 4. First 15 minutes: standard read-only evidence bundle

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl -n <namespace> describe pod <gpu-pod>
kubectl -n <namespace> logs <gpu-pod> --all-containers=true --timestamps=true --since=2h
kubectl -n <namespace> get pod <gpu-pod> -o json
\`\`\`

On an approved GPU host:

\`\`\`bash
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version,memory.used --format=csv
journalctl -k --since '-2 hours' --output=short-iso-precise | grep -Ei 'NVRM|Xid|AER|nvidia|rdma|mlx'
ip -s link
\`\`\`

Collect: UTC timestamps, cluster/context, node, pod UID, GPU UUID/MIG profile, driver/CUDA/NCCL/toolkit versions, image digest, job/rank mapping, recent changes, error-budget consumption, last successful checkpoint and customer-facing SLO. Limit access to raw logs; do not paste secrets into tickets.

## 5. Reusable runbook contract

Every production runbook must contain:
1. **Trigger and impact:** symptoms, severity and affected SLI.
2. **Owner and approvals:** IC, technical owner, security and workload owner.
3. **Preconditions:** identity, context, blast radius, capacity and checkpoints.
4. **Read-only diagnosis:** commands with expected signals and alternatives.
5. **Mitigation options:** ordered by lowest disruption.
6. **Change gate:** exact scope, approver, rollback, maintenance window and abort threshold.
7. **Execution:** idempotent where possible, explicit targets, audit trail.
8. **Validation:** GPU health, training progress, inference p99/error SLO, checkpoint integrity.
9. **Rollback:** trigger, last-known-good artifact and responsible owner.
10. **Escalation:** vendor/team route and handoff bundle.
11. **Closure:** incident timeline, RCA, COA, residual risk and review.

Never present an untested command as universally safe; distro, driver, operator, GPU model and workload topology change the procedure.

## 6. Recovery playbooks by failure domain

### A. GPU node/device plugin failure

**Trigger:** GPU allocatable missing, plugin failure, CUDA unavailable.

**Diagnose:** node conditions, plugin/Operator logs, \`nvidia-smi\`, module state, GPU UUID, recent rollout. **Mitigate:** shift new scheduling to healthy capacity only after policy/owner approval; preserve existing pods. **Gated recovery:** repair Operator/driver/runtime in staging, approve node-specific rollout. **Validate:** expected allocatable GPUs, CUDA smoke, workload SLO and no new Xid.

### B. CUDA driver/runtime incompatibility

**Trigger:** insufficient-driver error, NVML mismatch, container initialization failure.

**Diagnose:** host driver/kernel, runtime image, CUDA matrix, container toolkit ownership. **Mitigate:** select pinned compatible image where possible. **Gated recovery:** driver changes or host reboot only via approved node maintenance. **Validate:** NVML, CUDA device count, bounded kernel test and service health.

### C. GPU OOM and memory fragmentation

**Trigger:** CUDA OOM, rising peak memory or repeated inference admission failure.

**Diagnose:** distinguish \`OOMKilled\` from CUDA OOM; PyTorch allocated/reserved/peak, MIG profile, KV cache, input shapes. **Mitigate:** approved microbatch/concurrency cap or compatible capacity. **Gated recovery:** precision, sharding or allocator changes require numerical and performance acceptance. **Validate:** memory headroom, accuracy, p99 and training continuity.

### D. NCCL hang or distributed rank loss

**Trigger:** collective watchdog timeout, stalled steps, rendezvous error.

**Diagnose:** collect logs from **all ranks**, first error, Xid, network counters, checkpoint status and collective sequence. **Mitigate:** isolate failed job, protect verified checkpoint and preserve logs. **Gated recovery:** coordinated restart from a complete compatible checkpoint; do not kill individual production ranks blindly. **Validate:** all ranks healthy, training step progression, loss continuity and collective latency.

### E. Network/fabric degradation

**Trigger:** collective latency, RDMA link errors, RoCE congestion.

**Diagnose:** GPU↔NIC topology, NIC/RDMA counters, ECN/PFC/CNP, switch events, rank skew. **Mitigate:** use healthy approved placement if available. **Gated recovery:** NIC/MTU/PFC/routing changes require fabric owner. **Validate:** stable link health, collective p99 and no regression.

### F. Storage/checkpoint outage

**Trigger:** stalled data loader, checkpoint write failures, object-store 429/503, filesystem saturation.

**Diagnose:** per-rank I/O latency, throttling, retries, metadata, checkpoint commit markers. **Mitigate:** bounded backoff and workload throttling as approved. **Gated recovery:** restore storage or redirect to compatible backend; preserve last complete checkpoint. **Validate:** real checksum/restore test, training resume and RPO/RTO.

### G. Hardware Xid/ECC/PCIe failure

**Trigger:** Xid 79, repeated ECC/row remap, PCIe AER, missing GPU.

**Diagnose:** GPU UUID, Xid timeline, ECC/PCIe/NVLink and vendor diagnostics. **Mitigate:** protect workloads, coordinate isolation; no automatic reset. **Gated recovery:** vendor-supported repair/replacement or maintenance. **Validate:** GPU enumeration, driver health, CUDA/NCCL smoke, allocatable and service SLO.

## 7. Safe action matrix

| Action | Production default | Required gate |
|---|---|---|
| Inspect pod/node logs | Allowed with RBAC and redaction | approved read access |
| Collect GPU/NIC counters | Allowed with host authorization | security/data handling |
| \`kubectl cordon\` | Not automatic | node owner + capacity review |
| \`kubectl drain\` | Disruptive | checkpoint/PDB/replica approval |
| Restart device plugin/Operator | Potentially disruptive | change window and rollback |
| Restart containerd | Disruptive | platform approval |
| Reset GPU / reboot host | Disruptive and hardware-dependent | vendor/platform + workload owner |
| Modify PFC/ECN/MTU | Fabric-wide risk | network owner |
| Delete/rewrite checkpoints | Data-loss risk | workload/storage owner and retention policy |

## 8. RCA: causal evidence, not guesswork

### 8.1 Timeline

Create an event table with **UTC timestamp, source, observation, confidence, impact and evidence URI**. Distinguish:
- **Trigger:** event that initiated the failure.
- **Root cause:** underlying mechanism supported by evidence.
- **Contributing factors:** conditions that amplified severity or duration.
- **Detection gap:** why alerting did not surface the issue sooner.
- **Recovery gap:** why mitigation or rollback was slow.
- **Unknowns:** evidence not yet collected or alternative hypotheses.

A later NCCL watchdog timeout does not establish NCCL as the root cause; an earlier Xid, rank crash, storage stall or collective mismatch may explain the timeout.

### 8.2 Five whys with safeguards

Use “why” questions to find **systemic mechanisms**, not individuals to blame. Test each claim against logs, traces, deployment history and a counterfactual. If evidence is incomplete, label the cause **probable** or **undetermined**, not confirmed.

### 8.3 RCA template

\`\`\`yaml
incident_id: INC-EXAMPLE
status: "DRAFT"
severity: "UNASSESSED"
start_utc: null
detect_utc: null
mitigate_utc: null
restore_utc: null
customer_impact: null
affected_jobs: []
sli_slo_impact: null
rpo_rto_impact: null
trigger: null
root_cause: "UNDETERMINED"
root_cause_confidence: "LOW"
contributing_factors: []
detection_gaps: []
recovery_gaps: []
evidence_links: []
alternative_hypotheses: []
approvers: []
\`\`\`

## 9. COA: preventive changes with measurable verification

A corrective action must specify **owner, priority, due date, scope, verification test, rollback and effectiveness metric**. “Improve monitoring” is not an acceptable standalone COA.

| Failure mechanism | Example COA | Verification |
|---|---|---|
| Driver/toolkit drift | pinned compatibility policy and CI admission | reject unsupported image/node pairing |
| GPU Xid not paged | GPU UUID-aware health alert | synthetic signal reaches on-call |
| NCCL timeout without rank context | bounded per-rank log/trace retention | reconstruct first-failure timeline |
| Checkpoint corrupt/incomplete | atomic commit marker + restore test | successfully resume staging job |
| RoCE misconfiguration | version-controlled fabric policy and change gate | staged benchmark, no drops/regression |
| GPU OOM from context spikes | bounded admission/context limit | load-test p99 and OOM rate |
| Unsafe drain during training | checkpoint/PDB preflight policy | deny noncompliant drain in tabletop |

### 9.1 COA register

\`\`\`yaml
coa_id: COA-001
incident_id: INC-EXAMPLE
action: "Pin and validate CUDA driver/runtime compatibility"
owner: "platform-sre"
priority: "HIGH"
due_date: null
state: "PROPOSED"
acceptance_test: "staging CUDA smoke and policy test"
rollback: "restore known-good image and node pool"
effectiveness_metric: "incompatible deployment incidents per month"
evidence_links: []
\`\`\`

Track COA states: proposed → approved → in progress → verified → closed. Closure requires evidence, not just a merged PR.

## 10. Communication and incident review

**Initial update:** impact, scope, mitigation owner, next update time, confidence and what is unknown.

**During incident:** update at severity-appropriate cadence; include changes in affected services, mitigation status and customer risk.

**After recovery:** publish timeline, customer impact, verified root-cause confidence, recovery validation, COA owners and dates. Do not include private customer/model details in broadly accessible postmortems.

## 11. Sixteen hands-on exercises

1. Define severity and SLO impact for synthetic GPU outage.
2. Build incident role/approval matrix.
3. Collect Kubernetes and GPU read-only evidence.
4. Construct UTC timeline across rank/GPU/network/storage logs.
5. Draft GPU node/plugin recovery runbook.
6. Draft CUDA compatibility rollback runbook.
7. Draft GPU OOM recovery runbook.
8. Draft NCCL rank failure and checkpoint restart runbook.
9. Draft RDMA/RoCE network incident runbook.
10. Draft storage/checkpoint outage runbook.
11. Draft Xid/ECC hardware escalation runbook.
12. Run a **synthetic** multi-cause RCA and competing hypotheses.
13. Build a COA register with owners and measurable gates.
14. Tabletop change approvals, abort and rollback.
15. Stage a bounded recovery drill and alert-delivery validation.
16. Perform peer review, score and publish evidence record.

## 12. Ten incident scenarios

| # | Scenario | Required distinction |
|---|---|---|
| 1 | GPU allocatable disappears | plugin vs driver vs hardware |
| 2 | CUDA initialization fails after image update | runtime compatibility vs GPU fault |
| 3 | OOM on one MIG slice | capacity vs fragmentation |
| 4 | NCCL watchdog after Xid | symptom vs first cause |
| 5 | Rendezvous fails | rank configuration vs network policy |
| 6 | RoCE congestion during peak training | fabric vs rank straggler |
| 7 | Checkpoint upload succeeded but restore fails | completion vs integrity |
| 8 | GPU Xid 79 after PCIe event | hardware/host evidence |
| 9 | Inference p99 increases after mitigation | rollback gate |
| 10 | COA marked done without test evidence | governance gap |

## 13. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Incident roles, severity and safety | 15 |
| Read-only evidence and causal timeline | 20 |
| Seven recovery runbooks | 25 |
| RCA evidence and competing hypotheses | 15 |
| COA ownership, test and effectiveness | 15 |
| Staging drill, communication and peer review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100, **and** no critical security, data-integrity, unsafe change, missing rollback or unapproved production-impact failure. Staging execution and independent review remain pending.

## 14. References

- [Google SRE — Postmortem Culture](https://sre.google/sre-book/postmortem-culture/)
- [Google SRE — Managing Incidents](https://sre.google/sre-book/managing-incidents/)
- [Kubernetes Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [NVIDIA Xid Errors](https://docs.nvidia.com/deploy/xid-errors/)
- [NVIDIA NCCL Troubleshooting](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/troubleshooting.html)
- [PyTorch Distributed](https://docs.pytorch.org/docs/stable/distributed.html)

**Next:** Chapter 082 — AI Infrastructure Incident Capstone.
