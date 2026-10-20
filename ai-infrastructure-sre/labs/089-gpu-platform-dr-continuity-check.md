# Lab 089 — GPU Platform DR and Continuity Acceptance Gate

**Chapter:** [089 — GPU Platform DR and Continuity](../part-11-capacity-finops-and-reliability/089-gpu-platform-dr-continuity.md)

**Status:** NOT EXECUTED. Offline exercises, isolated restore drills, recovery-region serving tests, independent technical/security review and acceptance remain pending.

## 1. Safety

Use synthetic data and isolated staging only for recovery testing. Never intentionally fail a production region, delete live checkpoints, change DNS traffic, activate billable GPU reservations or alter KMS permissions without authorized change control. Preserve rollback and break-glass audit logs. Cross-region data movement requires residency approval.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | Primary/DR dependency map | region graph | Pending |
| E02 | Failure domains and tiers | recovery classification | Pending |
| E03 | Inference/training RTO/RPO | signed contracts | Pending |
| E04 | Artifact inventory | checkpoint/model manifest | Pending |
| E05 | Checkpoint commit marker | SHA-256 and state list | Pending |
| E06 | Replication/retention review | version inventory | Pending |
| E07 | Isolated restore | hash verification | Pending |
| E08 | Training resume | steps lost, optimizer state | Pending |
| E09 | GPU capacity compatibility | SKU/quota/zone matrix | Pending |
| E10 | Recovery critical path | measured RTO budget | Pending |
| E11 | Serving canary/failback | p99/errors/routing | Pending |
| E12 | Security/residency parity | IAM/KMS/policy review | Pending |
| E13 | Regional outage tabletop | incident timeline | Pending |
| E14 | Corruption recovery | last-good selection | Pending |
| E15 | Fencing/failback | dual-writer prevention | Pending |
| E16 | Independent acceptance | score, evidence and COAs | Pending |

## 3. Offline RTO budget

\`\`\`python
stages_minutes = {
    "detect_and_decide": 8,
    "gpu_provision": 22,
    "restore": 15,
    "warm_and_validate": 12,
    "route_and_verify": 8,
}
target_rto_minutes = 60
total = sum(stages_minutes.values())
print("planned_rto_minutes:", total, "target:", target_rto_minutes)
assert total == 65
assert total > target_rto_minutes
\`\`\`

Expected: **65 minutes**, therefore the synthetic **60-minute RTO FAILS**. This is a planning exercise, not a measured recovery.

## 4. Offline checkpoint integrity test

\`\`\`python
import hashlib
payload = b"synthetic-checkpoint-shard-v1"
expected = hashlib.sha256(payload).hexdigest()
assert hashlib.sha256(payload).hexdigest() == expected
corrupted = payload + b"-corrupt"
assert hashlib.sha256(corrupted).hexdigest() != expected
print("checksum_integrity_gate: PASS (synthetic only)")
\`\`\`

This validates checksum detection on a synthetic byte string. A real training restore must also verify all checkpoint shards, optimizer/scheduler/RNG state, dataset offsets, model quality and actual resumption.

## 5. Read-only staging inventory

\`\`\`bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get pods -A -o wide
kubectl get storageclass,pv,pvc -A
kubectl get resourcequota -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

The combined \`kubectl get storageclass,pv,pvc -A\` output may be formatted differently for cluster-scoped resources. Inspect storage topology and backup controller versions. Never infer cross-region recoverability from a Kubernetes PVC alone.

## 6. Staging recovery test matrix

| Scenario | Injection | Required outcome |
|---|---|---|
| Checkpoint corruption | modify synthetic copy | checksum rejection |
| Training node loss | isolated job interruption | last committed step resumed |
| Registry unavailable | block test mirror endpoint | verified alternate artifact |
| GPU SKU mismatch | select incompatible staging pool | fail closed, no false recovery |
| Inference failover | synthetic traffic to DR region | p99/errors and quality pass |
| IAM/KMS denied | synthetic restricted role | controlled escalation, no bypass |
| Dual-active writers | test fenced job leases | exactly one authoritative writer |
| Primary failback | isolated routing transition | no duplicate outputs |

Record observed end-to-end recovery time, not just the time to download checkpoint files.

## 7. Evidence template

\`\`\`yaml
chapter: "089"
environment: "OFFLINE_OR_STAGING"
execution_date_utc: null
incident_commander: null
platform_owner: null
ml_owner: null
security_reviewer: null
primary_region: null
recovery_region: null
rto_target_minutes: null
rpo_target_steps: null
artifact_manifest_uri: null
checkpoint_hash_verification: "NOT_RUN"
training_resume_test: "NOT_RUN"
gpu_capacity_compatibility: "NOT_VERIFIED"
inference_failover_test: "NOT_RUN"
observed_end_to_end_rto_minutes: null
observed_training_steps_lost: null
iam_kms_residency_review: "PENDING"
fencing_failback_test: "NOT_RUN"
rollback_validation: "NOT_RUN"
independent_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 8. Acceptance

Score **15 + 25 + 20 + 20 + 10 + 10 = 100**. Pass at ≥85 with no critical checkpoint corruption, unsupported recovery capacity, missed protected RTO/RPO, unauthorized cross-region data transfer, dual-writer risk or absent rollback. A documented DR plan is not an executed DR drill.

**Next:** Chapter 090 — Capacity and Reliability Acceptance Capstone.
