# Lab 081 — Recovery Runbooks and RCA COA Acceptance Gate

**Chapter:** [081 — Recovery Runbooks and RCA COA](../part-10-incident-response-and-troubleshooting/081-recovery-runbooks-rca-coa.md)

**Execution status:** NOT RUN. Offline exercises, staged drills, peer/security review and production acceptance are distinct pending milestones.

## 1. Scope and safety

Run tabletop and synthetic analysis offline; execute recovery only in approved, isolated staging. No production GPU reset, node drain, driver reload, network changes, checkpoint deletion, forced rank termination or vendor repair without explicit change control. Preserve checkpoint integrity and evidence confidentiality.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | Severity and SLO classification | impact worksheet | Pending |
| E02 | IC/owner approval matrix | RACI and handoffs | Pending |
| E03 | Read-only collection | timestamped, redacted outputs | Pending |
| E04 | UTC timeline | ranked causal sequence | Pending |
| E05 | GPU plugin runbook | diagnosis/change/rollback | Pending |
| E06 | CUDA compatibility runbook | pinned matrix and canary | Pending |
| E07 | OOM recovery runbook | allocator/MIG/host classification | Pending |
| E08 | NCCL rank recovery runbook | verified checkpoint semantics | Pending |
| E09 | RDMA/RoCE runbook | fabric owner approval | Pending |
| E10 | Checkpoint outage runbook | checksum/restore/RPO | Pending |
| E11 | Xid/ECC vendor runbook | support bundle | Pending |
| E12 | Synthetic RCA | cause vs contributor vs symptom | Pending |
| E13 | COA register | owner/due date/acceptance | Pending |
| E14 | Rollback tabletop | stop criteria and decision log | Pending |
| E15 | Staging drill | bounded test and alert delivery | Pending |
| E16 | Independent review | score and signed evidence | Pending |

## 3. Read-only evidence commands

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl -n <namespace> describe pod <gpu-pod>
kubectl -n <namespace> logs <gpu-pod> --timestamps=true --since=2h
\`\`\`

Approved GPU host:

\`\`\`bash
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version --format=csv
journalctl -k --since '-2 hours' | grep -Ei 'NVRM|Xid|AER|nvidia'
\`\`\`

## 4. Synthetic multi-layer incident fixture

\`\`\`text
14:00:00Z  driver rollout starts on one GPU node
14:01:12Z  kernel logs: NVIDIA driver/library mismatch
14:01:20Z  device plugin reports GPU registration failure
14:02:05Z  training rank 3 fails CUDA initialization
14:04:00Z  other ranks report NCCL watchdog timeout
14:06:00Z  incident declared SEV2
14:09:00Z  checkpoint status verified from prior successful step
14:14:00Z  approved rollback plan prepared; not executed
\`\`\`

Tasks:
1. Identify first observed failure and distinguish likely trigger, root cause and downstream symptoms.
2. List alternative hypotheses and missing evidence; do not claim confirmed RCA from fixture alone.
3. Choose lowest-risk containment; document why immediate drain/reset is unsafe.
4. Draft vendor/platform handoff and customer status update.
5. Create three COAs: version compatibility gate, GPU registration alert and checkpoint-safe recovery drill.

## 5. Runbook worksheet

\`\`\`yaml
runbook_id: RB-GPU-EXAMPLE
owner: null
trigger: null
severity: null
affected_sli: null
preconditions: []
read_only_diagnostics: []
mitigation_options: []
change_approvers: []
checkpoint_verified: false
blast_radius: null
abort_threshold: null
rollback_artifact: null
recovery_validation: []
vendor_escalation: null
evidence_links: []
last_reviewed_utc: null
\`\`\`

## 6. RCA and COA evidence

\`\`\`yaml
incident_id: INC-081-LAB
execution_date_utc: null
environment: "OFFLINE_OR_STAGING"
reviewer: null
timeline: []
trigger: "HYPOTHESIS"
root_cause: "UNDETERMINED"
confidence: "LOW"
alternative_hypotheses: []
customer_impact: "SYNTHETIC"
checkpoint_status: "NOT_VERIFIED"
coa_register:
  - id: COA-001
    owner: null
    due_date: null
    acceptance_test: null
    status: PROPOSED
staging_drill: "NOT_RUN"
alert_delivery: "NOT_RUN"
security_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 7. Acceptance

Score **15 + 20 + 25 + 15 + 15 + 10 = 100**. Pass at ≥85 only when runbook safety, RCA evidence, COA verification plans, rollback and review are satisfactory. Never equate written documentation or a synthetic tabletop with executed staging recovery.

**Next:** Chapter 082 — AI Infrastructure Incident Capstone.
