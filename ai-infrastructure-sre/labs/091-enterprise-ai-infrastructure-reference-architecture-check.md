# Lab 091 — Enterprise AI Infrastructure Reference Architecture Acceptance Gate

**Chapter:** [091 — Enterprise AI Infrastructure Reference Architecture](../part-12-enterprise-operations-and-final-capstones/091-enterprise-ai-infrastructure-reference-architecture.md)

**Execution:** NOT RUN. Architecture review, offline modeling, security tests, GPU staging validation and independent acceptance remain pending.

## 1. Safety

This lab designs and audits an enterprise platform. Do not provision billable GPU nodes, expose production data, change IAM, install privileged GPU components, modify live network policies or run fault injection without approved staging scope and change controls. Use synthetic identities and workloads for demonstrations.

## 2. Sixteen-exercise checklist

| ID | Exercise | Required evidence | Status |
|---|---|---|---|
| E01 | Stakeholder/SLO inventory | requirements register | Pending |
| E02 | Logical/physical diagrams | architecture views | Pending |
| E03 | GPU/AZ/fabric inventory | compatibility matrix | Pending |
| E04 | Cluster/node-pool isolation | tenancy decisions | Pending |
| E05 | RBAC/threat model | positive/negative access cases | Pending |
| E06 | Network policies | connectivity matrix | Pending |
| E07 | Artifact and secret supply chain | signing/KMS controls | Pending |
| E08 | Training queue/gang design | admission policy | Pending |
| E09 | Checkpoint integrity | manifest/restore plan | Pending |
| E10 | Serving rollout | canary/p99/rollback | Pending |
| E11 | Observability | SLI and alert ownership | Pending |
| E12 | N-1/AZ resilience | failure-domain calculation | Pending |
| E13 | FinOps unit economics | ledger design | Pending |
| E14 | DR/failback | RTO/RPO/fencing | Pending |
| E15 | Architecture tabletop | 12 failure cases | Pending |
| E16 | Independent signoff | score/COAs/approvals | Pending |

## 3. Offline failure-domain sizing

\`\`\`python
nodes_per_zone = {"az-a": 4, "az-b": 4}
gpus_per_node = 8
installed = sum(nodes_per_zone.values()) * gpus_per_node
after_node_failure = installed - gpus_per_node
after_az_failure = min(
    sum(n*gpus_per_node for zone,n in nodes_per_zone.items() if zone != failed)
    for failed in nodes_per_zone
)
protected_inference = 24
non_preemptible_training = 16
print(installed, after_node_failure, after_az_failure)
assert (installed, after_node_failure, after_az_failure) == (64, 56, 32)
assert protected_inference <= after_az_failure
assert protected_inference + non_preemptible_training > after_az_failure
\`\`\`

Expected: **64 installed, 56 after node loss, 32 after AZ loss**. The aggregate inference-only capacity condition passes; combined inference/training fails. Real topology, SKU/HBM, CPU/network and peak p99 feasibility require staging proof.

## 4. Read-only Kubernetes audit

\`\`\`bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get namespaces
kubectl get priorityclass
kubectl get resourcequota -A
kubectl get networkpolicy -A
kubectl get serviceaccounts -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

Review actual network-policy enforcement in a test namespace, not merely presence of objects. Avoid collecting secret values.

## 5. Architecture review tabletop

| Test | Design question | Evidence |
|---|---|---|
| GPU node failure | compatible spare placement? | capacity matrix |
| AZ outage | survivor SLO and capacity? | failure model |
| Training fabric degradation | NCCL retries and topology? | network plan |
| Checkpoint corruption | committed manifest restore? | integrity design |
| Serving burst | autoscaling + backpressure? | p99 budget |
| Tenant privilege escalation | RBAC/admission denial? | security tests |
| Registry/KMS outage | signed artifacts and DR keys? | dependency graph |
| Invoice anomaly | reconciled unit economics? | ledger policy |

## 6. Evidence manifest

\`\`\`yaml
chapter: "091"
environment: "DESIGN_AND_STAGING"
publication: "PUBLISHED"
execution: "NOT_RUN"
architecture_owner: null
security_owner: null
platform_owner: null
ml_owner: null
finops_owner: null
review_date_utc: null
requirements_approved: false
threat_model_review: "PENDING"
gpu_compatibility_review: "PENDING"
network_isolation_test: "NOT_RUN"
training_admission_test: "NOT_RUN"
checkpoint_restore_test: "NOT_RUN"
inference_slo_test: "NOT_RUN"
n_minus_one_test: "NOT_RUN"
dr_test: "NOT_RUN"
cost_reconciliation: "NOT_RUN"
independent_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
corrective_actions: []
\`\`\`

## 7. Acceptance rubric

| Gate | Points |
|---|---:|
| Requirements and threat model | 15 |
| Physical/GPU/network/data architecture | 20 |
| Kubernetes tenancy, identity and supply chain | 20 |
| Training/inference lifecycle | 15 |
| Observability and operations | 10 |
| Capacity, FinOps, HA and DR | 15 |
| Review evidence/rollback | 5 |
| **Total** | **100** |

Pass ≥85 and every critical security, SLO, checkpoint, data residency, capacity, recovery and rollback gate. **A reference diagram is not an executed deployment or accepted production platform.**

**Next:** Chapter 092 — Build a Secure GPU Cluster Platform.
