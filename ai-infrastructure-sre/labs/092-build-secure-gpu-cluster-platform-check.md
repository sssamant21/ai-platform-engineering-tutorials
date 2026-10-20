# Lab 092 — Build a Secure GPU Cluster Platform Acceptance Gate

**Chapter:** [092 — Build a Secure GPU Cluster Platform](../part-12-enterprise-operations-and-final-capstones/092-build-secure-gpu-cluster-platform.md)

**Execution:** NOT RUN. No cluster has been provisioned or modified. GPU staging, security negative tests, operator validation and independent acceptance are pending.

## 1. Safety

Use an isolated staging cluster and approved identities. Do not change production RBAC, CNI policies, admission controls, GPU drivers, KMS, node pools or cloud reservations. Confirm cost limits and rollback. Never store credentials, customer data or raw billing identifiers in evidence.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | Status |
|---|---|---|---|
| E01 | Threat model/scope | signed risk register | Pending |
| E02 | GPU compatibility | SKU/OS/driver matrix | Pending |
| E03 | IaC controls | plan/policy review | Pending |
| E04 | Identity/audit | access and log proof | Pending |
| E05 | Node-pool isolation | taints/placement | Pending |
| E06 | GPU Operator | allocatable GPUs and security | Pending |
| E07 | Pod Security | restricted denial case | Pending |
| E08 | RBAC | allow/deny cases | Pending |
| E09 | Network isolation | positive/negative flows | Pending |
| E10 | Signed images | admission denial | Pending |
| E11 | Workload identity/KMS | scoped access | Pending |
| E12 | GPU quotas/priority | admission denial | Pending |
| E13 | Scheduling isolation | workload placement | Pending |
| E14 | GPU/security telemetry | dashboards and alerts | Pending |
| E15 | Rollout/rollback | restore evidence | Pending |
| E16 | Independent review | score/COAs | Pending |

## 3. Read-only preflight

\`\`\`bash
kubectl config current-context
kubectl version
kubectl get nodes -o wide
kubectl get namespaces --show-labels
kubectl get priorityclass
kubectl get resourcequota -A
kubectl get networkpolicy -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

Run only against the approved context. Record \`NOT_INSTALLED\` for missing GPU Operator or policy engine; do not infer compliance.

## 4. Offline security manifest review

Use the chapter's example Namespace, Role/RoleBinding, default-deny NetworkPolicy and GPU ResourceQuota. Review with approved tools and, if available, \`kubectl apply --dry-run=server -f <staging-manifest.yaml>\` **only** on an isolated authorized cluster. Dry-run validates API admission/schema; it does not prove policy enforcement, image trust, GPU device isolation or connectivity.

## 5. Negative security test matrix

| Test | Expected result | Evidence |
|---|---|---|
| Unauthorized ClusterRoleBinding | DENIED | RBAC audit |
| Privileged tenant pod | DENIED by policy | admission event |
| Cross-tenant service access | DENIED | flow logs |
| Required DNS/object storage | ALLOWED by explicit rule | connectivity |
| Unsigned image | DENIED by verified admission | policy decision |
| Unapproved high PriorityClass | DENIED | admission audit |
| Secret access by wrong service account | DENIED | IAM/KMS audit |
| GPU host device access | DENIED unless approved | security review |

Tests require real staging identities and policy engines; none are executed by publishing this document.

## 6. GPU device discovery checks

\`\`\`bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'
kubectl get pods -A | grep -Ei 'gpu-operator|device-plugin|dcgm' || true
\`\`\`

An empty GPU count may mean the node has no GPU or the device plugin is unavailable; compare with approved node inventory. Do not use a privileged test pod without security approval.

## 7. Evidence manifest

\`\`\`yaml
chapter: "092"
environment: "STAGING_REQUIRED"
publication: "PUBLISHED"
execution: "NOT_RUN"
execution_date_utc: null
cluster_id: null
platform_owner: null
security_reviewer: null
gpu_sku_driver_matrix: []
iam_rbac_tests: "NOT_RUN"
pod_security_tests: "NOT_RUN"
network_isolation_tests: "NOT_RUN"
image_provenance_tests: "NOT_RUN"
gpu_operator_validation: "NOT_RUN"
workload_identity_kms_tests: "NOT_RUN"
quota_priority_tests: "NOT_RUN"
audit_telemetry_validation: "NOT_RUN"
rollback_test: "NOT_RUN"
independent_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 8. Acceptance

Score **20 + 20 + 20 + 20 + 10 + 10 = 100**. Pass ≥85 with all critical security and rollback gates satisfied. **Publishing manifests is not a deployed or security-validated GPU cluster.**

**Next:** Chapter 093 — Validate Drivers CUDA and NCCL.
