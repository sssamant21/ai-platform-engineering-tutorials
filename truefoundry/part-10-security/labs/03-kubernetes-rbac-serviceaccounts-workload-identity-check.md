# Chapter 10.3 — Hands-on Lab: Kubernetes RBAC, ServiceAccounts and Workload Identity

**Part 10 | Isolated staging only | Execution not yet performed**

## Mission and safety

Prove least-privilege Kubernetes RBAC, ServiceAccount token hygiene and cloud workload identity boundaries for a production-like TrueFoundry deployment. **[SAFE-READ]** inspection; **[LAB-MUTATE]** disposable staging; **[CHANGE-CONTROL]** production approval. Never print or share ServiceAccount tokens, IAM credentials, customer data or Kubernetes Secret contents.

## Lab 0 — Preflight [SAFE-READ]

```bash
kubectl config current-context
kubectl get ns --show-labels
kubectl get sa -A
kubectl get role,rolebinding -A
kubectl get clusterrole,clusterrolebinding
kubectl get pods -A -o custom-columns='NS:.metadata.namespace,POD:.metadata.name,SA:.spec.serviceAccountName'
kubectl auth can-i --list -n YOUR_NAMESPACE
```

Record actual TrueFoundry agent ServiceAccount, required API operations, cluster policy, EKS IRSA/Pod Identity or AKS Workload Identity configuration, identity issuer, IAM owner and GitOps source. If impersonation is forbidden, mark test **blocked** rather than inferring target permissions.

## Lab 1 — Namespace-scoped RBAC [LAB-MUTATE]

Create `tfy-security-lab` only on approved staging. Apply the canonical chapter's inspector ServiceAccount, Role and RoleBinding. Verify `get/list/watch` Pods, Services and Deployments work **through an approved impersonation test or authorized projected-token client**. Verify Secrets read, Pod creation, `pods/exec` and ClusterRoleBinding creation are denied. Capture audit evidence without tokens.

## Lab 2 — ServiceAccount token posture [SAFE-READ]

Inspect `automountServiceAccountToken`, Pod token volumes, issuer/audience/expiration settings and rotation ownership. Do not dump projected token files. In a disposable Pod, compare disabled automount to an explicitly approved bounded token projection.

## Lab 3 — Cloud identity trust [SAFE-READ]

**EKS IRSA:** review OIDC provider, ServiceAccount annotation, trust `sub`/`aud` and role permissions. **EKS Pod Identity:** review agent, association, role trust and SDK support. **AKS:** review OIDC issuer, federated identity subject/audience, managed identity/client ID and required Pod label. Only test the mechanism actually installed.

## Lab 4 — Authorized/denied cloud calls [LAB-MUTATE]

Using disposable storage and a narrowly scoped staging identity, verify access to one permitted object/container and denial for an unapproved one. Confirm the node image-pull identity differs from workload artifact-fetch identity when applicable. Do not create broad wildcard IAM policies.

## Lab 5 — Drift, rollback and tabletop

Review a hypothetical wildcard Role, unexpected ClusterRoleBinding, ServiceAccount token exposure and federated trust mismatch. Document containment, audit evidence, GitOps revert and least-privilege retest. Revert all lab-owned resources and identity associations.

## 28 failure scenarios

| # | Scenario | Evidence focus |
|---|---|---|
| 01 | Role missing resource verb | API authorization tuple |
| 02 | Wrong API group | Role rules |
| 03 | Wrong namespace RoleBinding | Binding scope |
| 04 | Wrong ServiceAccount subject | Binding subject |
| 05 | Pod uses default ServiceAccount | Pod spec |
| 06 | ClusterRoleBinding overly broad | Cluster scope |
| 07 | Aggregated ClusterRole drift | Aggregation labels |
| 08 | `secrets` read unexpectedly allowed | Additive bindings |
| 09 | `pods/exec` unexpectedly allowed | Subresource RBAC |
| 10 | Pod creation grants indirect access | Workload privilege |
| 11 | `impersonate` privilege exposed | Escalation control |
| 12 | `bind` or `escalate` exposed | RBAC privilege boundary |
| 13 | `kubectl auth can-i --as` forbidden | Caller impersonation |
| 14 | Token automount unexpectedly enabled | Pod/SA defaults |
| 15 | Projected token audience mismatch | Token verifier |
| 16 | Projected token expired | Rotation |
| 17 | Legacy static token remains | Token lifecycle |
| 18 | EKS OIDC provider mismatch | IRSA issuer |
| 19 | IRSA trust subject too broad | IAM trust |
| 20 | IRSA STS access blocked | Network/STS |
| 21 | EKS Pod Identity association missing | Association/agent |
| 22 | EKS Pod Identity SDK unsupported | Credential chain |
| 23 | AKS federated subject mismatch | Entra credential |
| 24 | AKS workload identity Pod label missing | Pod mutation |
| 25 | AKS audience/client ID wrong | Token exchange |
| 26 | Cloud object denied despite valid token | Data-plane IAM |
| 27 | Admission rejects workload | Pod security/policy |
| 28 | GitOps restores unsafe RBAC | Desired-state source |

For each record expected/actual action, principal (not token), API response, sanitized audit, root-cause hypothesis, least-privilege fix, rollback and reviewer. Simulate only in staging; otherwise mark tabletop.

## Ten acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Approved staging context and agent identity inventory | Pending |
| G2 | Namespace-scoped Role/RoleBinding positive tests | Pending |
| G3 | Denied Secrets, exec and cluster mutation tests | Pending |
| G4 | Additive bindings and escalation risk reviewed | Pending |
| G5 | ServiceAccount automount/token lifecycle verified | Pending |
| G6 | Installed cloud workload identity mechanism mapped | Pending |
| G7 | Cloud identity federation and scoped object access verified | Pending |
| G8 | Audit/alert and drift detection demonstrated | Pending |
| G9 | GitOps rollback and lab cleanup validated | Pending |
| G10 | Exceptions, security review and sign-off recorded | Pending |

**Pass:** all applicable gates have timestamped evidence and owner approval. Unexecuted or blocked gates are not passes.

## Evidence worksheet

UTC timestamp; cluster/context; namespace; Pod/ServiceAccount; API group/resource/verb; expected/actual decision; role binding; cloud identity mechanism; sanitized audit ID; GitOps commit; rollback; reviewer.

**Next:** Chapter 10.4 — Secrets Management, Encryption and Credential Rotation.
