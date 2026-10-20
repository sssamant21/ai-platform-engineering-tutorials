# Chapter 092 — Build a Secure GPU Cluster Platform

**Part 12 — Enterprise Operations and Final Capstones | AI Infrastructure Engineering & SRE**

**Status:** DOCUMENTATION PUBLISHED. Cluster provisioning, hands-on lab execution, security tests, GPU validation and independent acceptance remain **PENDING**.

## 1. Mission

Build and verify a **secure, auditable, production-oriented Kubernetes GPU platform** suitable for enterprise AI inference and distributed training. Integrate identity, hardened node pools, GPU device management, admission control, namespace/tenant boundaries, network isolation, artifact integrity, telemetry, backup and operational change management.

**Prerequisites:** Chapter 091 reference architecture, Kubernetes administration, IAM/RBAC, Terraform/GitOps, container supply-chain security, NVIDIA GPU Operator/device plugin, CNI policy enforcement and cloud GPU capacity/quota fundamentals.

### Learning outcomes

1. Translate a reviewed architecture into an approved cluster deployment plan.
2. Separate system, serving, training and research node pools with explicit scheduling policies.
3. Apply least-privilege identity and validate positive and negative RBAC cases.
4. Deploy GPU management components using version-pinned, approved configurations.
5. Enforce workload admission, network segmentation, secrets protection and image provenance.
6. Establish audit, telemetry, patching, rollback and independently verified acceptance.

## 2. Platform threat model

| Threat | Control | Verification |
|---|---|---|
| Untrusted tenant escalates privilege | RBAC, Pod Security Admission, policy engine | deny privileged pod and role escalation |
| Workload accesses another tenant's data | identity, storage ACL, network policy | cross-tenant access denied |
| Malicious container image | pinned digest, signature, provenance, scanning | unsigned image denied |
| GPU host/device exposure | restricted operator privileges, node isolation | hostPath/device access review |
| Exfiltration through egress | deny-by-default network policy, egress gateway | unauthorized destination denied |
| Secrets leaked in manifests/logs | workload identity, KMS, external secrets | no plaintext secrets in Git |
| Priority/queue bypass | admission policy and protected PriorityClasses | unauthorized priority denied |
| Control-plane compromise | MFA/SSO, audit, least privilege, backups | privileged access review |
| Supply-chain drift | GitOps drift detection and signed releases | unapproved change blocked |
| Recovery-region key failure | tested IAM/KMS DR path | restore drill evidence |

**Isolation note:** namespaces, GPU time-slicing and network policies do not automatically provide hard multi-tenant isolation. Threat-model GPU memory sharing, driver privilege, runtime access and side channels. Use dedicated nodes/clusters for tenants requiring stronger boundaries.

## 3. Reference deployment

\`\`\`text
Enterprise IdP / SSO / MFA
          |
  CI policy gates → GitOps / IaC → Kubernetes API audit
                                |
                  Admission + RBAC + Pod Security
                                |
               CNI policy / workload identity / KMS
                                |
          ┌─────────────────────┼──────────────────────┐
          |                     |                      |
      System nodes         Serving GPU pool       Training GPU pool
      DNS/metrics          HA and protected        topology-aware
      controllers          p99 capacity            queue + checkpoints
          |                     |                      |
          └─────────────────────┼──────────────────────┘
                                |
                GPU Operator / device plugin
                                |
                  Telemetry + audit + backups
\`\`\`

Use managed Kubernetes where appropriate, but validate provider-specific security controls, GPU SKU availability, quotas, cluster version support and node image/driver compatibility before applying manifests.

## 4. Preflight checklist

- Approved architecture, threat model, service ownership and target region/AZs.
- Kubernetes API version, GPU SKU/HBM, CPU/RAM, NIC/fabric and node image matrix.
- Cloud IAM, VPC/VNet, private endpoints, firewall and egress requirements.
- GPU quota **and** confirmed physical supply/reservations.
- Cluster encryption/KMS, secrets management and audit retention.
- Signed container registry, artifact provenance and vulnerability policies.
- CNI with verified NetworkPolicy enforcement.
- Backup, GitOps recovery, break-glass, change window and rollback.
- Cost guardrails, tagging, node pool limits and budget alerts.
- A dedicated staging environment and test identities for negative security tests.

**Go/no-go:** do not provision until high-risk dependencies and approvals are resolved.

## 5. Infrastructure as code and GitOps layout

\`\`\`text
platform/
  terraform/
    foundation/
    identity/
    kubernetes/
    nodepools/
    storage/
    audit/
  gitops/
    bootstrap/
    system/
    gpu-operator/
    policy/
    tenants/
    observability/
  policies/
    rbac/
    pod-security/
    network/
    image-provenance/
    quotas/
  tests/
    preflight/
    gpu/
    security/
    resilience/
  runbooks/
  evidence/
\`\`\`

This is a proposed directory structure, not a claim that modules or infrastructure have been created. Use remote state locking, scoped CI credentials, plan review, policy checks and protected branches.

## 6. Kubernetes cluster hardening

### Control plane and access

- Private API endpoint or restricted network path, depending on platform constraints.
- SSO/MFA, short-lived identities, minimal admin privileges and emergency access with audit.
- Enable audit logging and monitor privilege changes, service-account token misuse and policy bypass.
- Encrypt sensitive Kubernetes data with supported KMS configuration.
- Define API server/etcd backup and provider-supported recovery procedures.
- Pin cluster/node versions; rehearse upgrade/rollback in staging.

### Node and runtime

- Approved hardened OS/node image and patch cadence.
- Minimal host access; disable unnecessary services and direct SSH where possible.
- Secure container runtime, image verification and restricted privileged workloads.
- Separate GPU management DaemonSets from tenant permissions.
- Monitor GPU Xid/ECC, thermal/power events and node readiness.
- Use topology labels, taints/tolerations and dedicated node pools to prevent accidental cross-class placement.

## 7. GPU Operator and device plugin

The NVIDIA GPU Operator may deploy privileged components to manage drivers, toolkit, device plugins, feature discovery and telemetry. This is a **high-trust platform dependency**, not an ordinary tenant application.

**Deployment gates:**
1. Check Kubernetes, kernel, OS, GPU model, driver, CUDA and Operator compatibility.
2. Decide whether drivers are preinstalled or operator-managed; avoid conflicting installations.
3. Pin reviewed chart/image versions and digests where supported.
4. Limit chart installation and CRD changes to platform administrators.
5. Validate device discovery and \`nvidia.com/gpu\` resource reporting.
6. Review MIG strategy and node labels; prohibit unapproved sharing.
7. Confirm DCGM telemetry and alerting.
8. Test a single-GPU smoke job in staging before production rollout.

**Read-only checks:**

\`\`\`bash
kubectl get nodes -o wide
kubectl describe nodes | grep -E 'nvidia.com/gpu|Capacity:|Allocatable:'
kubectl get pods -A | grep -Ei 'gpu-operator|device-plugin|dcgm'
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

The commands are discovery aids. Do not assume \`nvidia.com/gpu\` is available until drivers/device plugins and allocatable resources are verified.

## 8. Tenant isolation and RBAC

### Example namespace with Pod Security labels

\`\`\`yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ai-research-staging
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
    owner-team: ml-platform
\`\`\`

Pod Security Admission \`restricted\` may conflict with privileged GPU management components; deploy those components in tightly controlled system namespaces and review exemptions. Tenant workloads should not inherit those exemptions.

### Example scoped Role

\`\`\`yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: ai-research-staging
  name: workload-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "events"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: ai-research-staging
  name: workload-reader-binding
subjects:
  - kind: Group
    name: ml-platform-readers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: workload-reader
  apiGroup: rbac.authorization.k8s.io
\`\`\`

The identity group must exist and be mapped by the cluster authenticator. A RoleBinding alone does not establish an authenticated identity or deny other grants.

### Read-only access validation

\`\`\`bash
kubectl auth can-i list pods -n ai-research-staging
kubectl auth can-i create clusterrolebindings
kubectl auth can-i --as=system:serviceaccount:ai-research-staging:default get secrets -n ai-research-staging
\`\`\`

The \`--as\` impersonation check requires permission to impersonate and may fail for that reason; use approved test identities for definitive access-denial evidence.

## 9. Network isolation

A default-deny policy isolates both ingress and egress for selected pods **only if the installed CNI enforces Kubernetes NetworkPolicy**.

\`\`\`yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: ai-research-staging
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
\`\`\`

Before adopting this in staging, design explicit DNS, KMS, registry, object storage, monitoring and approved inference/training traffic allow rules. Applying deny-all without allow rules can break workloads. Validate negative and positive connectivity with disposable test pods, then promote with rollback.

## 10. Resource quotas and GPU admission

\`\`\`yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: research-gpu-quota
  namespace: ai-research-staging
spec:
  hard:
    requests.nvidia.com/gpu: "8"
    pods: "50"
\`\`\`

GPU extended resource quota keys must be validated against the target Kubernetes version and device plugin; for GPUs, request and limit semantics can be constrained by the resource type. ResourceQuota limits **allocation requests**, not physical GPU availability. Kueue/Volcano scheduling, protected serving pools, PriorityClass restrictions and cloud quotas need independent controls.

## 11. Image provenance and secrets

- Require approved registries, immutable digests and signature verification with a supported policy engine.
- Run SBOM/vulnerability scanning with documented severity exceptions.
- Keep model registry and checkpoint access in scoped workload identities.
- Store secrets outside Git; audit rotation and emergency recovery.
- Restrict access to GPU host devices, host namespaces, privileged containers and unsafe mounts.
- Enforce approved data egress and model/data residency.

Admission policies must be **tested with negative cases**; installing a policy engine does not prove it is configured to fail closed.

## 12. Observability and audit

| Domain | Required signals |
|---|---|
| Cluster | node/pod readiness, scheduler failures, API errors |
| GPU | allocatable count, Xid/ECC, HBM, thermal/power |
| Security | denied admission, RBAC changes, privileged workloads |
| Network | denied flows, egress and DNS errors |
| Serving | p99, TTFT/TPOT, errors, warm replicas |
| Training | steps/sec, checkpoint age, NCCL failures |
| Capacity | quota denial, queue age, stock and reservations |
| FinOps | billed GPU-hours, idle commitments, $/useful work |
| DR | backup age, checkpoint integrity, RTO/RPO |

Define ownership, retention, privacy controls, paging thresholds and evidence locations. Monitor security controls themselves for telemetry gaps.

## 13. Safe rollout and rollback

**Rollout:** preflight → IaC plan and policy checks → staging cluster → system services → GPU Operator → security controls → synthetic workload tests → resilience and DR checks → independent signoff → controlled production canary.

**Rollback:** preserve previous cluster/node/operator/chart versions, admission policies, image digests, network allow rules and GitOps commits. Some driver, Kubernetes and CRD upgrades are not safely reversible in place; use tested node replacement or blue/green migration rather than promising a generic rollback.

## 14. Six production runbooks

**R1 — GPU node not ready:** check node OS/driver/device plugin, kernel compatibility, taints and Xid/ECC; cordon/isolate only with approval and restore capacity.

**R2 — Tenant privilege escalation attempt:** preserve audit, deny admission, inspect RBAC/PSA exemptions and rotate compromised identities.

**R3 — Network policy outage:** inspect CNI enforcement and allow rules; restore last known-good policy through approved GitOps rollback.

**R4 — Unsigned or vulnerable image:** block admission, verify registry/signature policy, quarantine artifact and route remediation to owner.

**R5 — GPU quota/scheduling incident:** distinguish ResourceQuota, Kueue admission, node allocatable and cloud stock; protect P0 serving capacity.

**R6 — Compromised platform component:** isolate affected operator/controller, preserve evidence, rotate credentials, restore trusted images and validate workloads.

## 15. Sixteen hands-on exercises

1. Approve threat model and staging deployment scope.
2. Verify GPU SKU/driver/cluster compatibility matrix.
3. Review IaC state, plan and least-privilege CI.
4. Configure cluster identity and audit controls.
5. Build isolated system/serving/training node pools.
6. Validate GPU Operator security and device discovery.
7. Apply restricted tenant Pod Security policy in staging.
8. Validate RBAC allow and deny test identities.
9. Test default-deny and required network allow flows.
10. Enforce approved signed image policy.
11. Validate workload identity and KMS secret access.
12. Test GPU ResourceQuota and priority restrictions.
13. Validate scheduling and training/serving isolation.
14. Verify GPU telemetry, security alerts and log retention.
15. Rehearse safe policy/driver rollout and recovery.
16. Assemble acceptance evidence and independent review.

## 16. Twelve troubleshooting scenarios

| Symptom | Investigation |
|---|---|
| GPU node Ready but no allocatable GPUs | driver/device plugin |
| GPU Operator CrashLoop | kernel/driver/permissions |
| Tenant pod denied by PSA | security context |
| RBAC test unexpectedly succeeds | inherited binding |
| RBAC impersonation check fails | impersonation authorization |
| NetworkPolicy present, traffic still allowed | CNI enforcement |
| Default-deny breaks DNS | missing allow rules |
| Image signature policy bypassed | fail-open admission |
| GPU pod Pending despite quota | node SKU/stock/taints |
| Research workload uses protected priority | policy gap |
| Secrets unavailable after restore | workload identity/KMS |
| Driver upgrade breaks NCCL | compatibility and rollback |

## 17. Security acceptance (100 points)

| Domain | Points |
|---|---:|
| Threat model, IAM and control-plane hardening | 20 |
| GPU node/Operator integrity | 20 |
| Tenant isolation, RBAC and Pod Security | 20 |
| Network and artifact/secret security | 20 |
| Observability, audit and recovery | 10 |
| Evidence and independent review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 and no critical privilege escalation, unverified network isolation, unsigned image bypass, secret leakage, GPU tenant isolation failure or missing rollback. **Documentation publication does not establish secure deployment.**

## 18. References

- [Kubernetes Security](https://kubernetes.io/docs/concepts/security/)
- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [SLSA Supply-chain Security](https://slsa.dev/)
- [Sigstore Cosign](https://docs.sigstore.dev/cosign/)

**Next:** Chapter 093 — Validate Drivers CUDA and NCCL.
