# Chapter 10.3 — Kubernetes RBAC, ServiceAccounts and Workload Identity

**Part 10 — Security | Canonical production tutorial**

## Learning outcomes and safety

Design and audit Kubernetes authorization for TrueFoundry cluster agents, deployment controllers, inference services, jobs and GPU workloads. Implement namespace-scoped RBAC, distinguish Kubernetes API authorization from cloud IAM, secure ServiceAccount tokens and integrate supported workload identity on AWS EKS and Azure AKS. Troubleshoot denied actions without granting cluster-admin.

**Execution labels:** **[SAFE-READ]** inspection only; **[LAB-MUTATE]** isolated staging; **[CHANGE-CONTROL]** production approval, rollback and monitoring. Verify the actual TrueFoundry agent installation, cloud integration, Kubernetes version and identity configuration. Never assume a product agent needs cluster-admin or that a documented integration is enabled in your deployment.

## 1. Identity and authorization layers

```text
Human / CI → TrueFoundry control plane (platform roles)
             ↕ approved agent communication
TrueFoundry cluster agent Pod → Kubernetes ServiceAccount token
                             → Kubernetes API → RBAC authorization
Model-serving Pod → Kubernetes ServiceAccount
                  → projected workload identity token
                  → cloud identity exchange / role trust
                  → S3/Blob/registry/secret manager
Client → inference gateway → application authorization
```

Kubernetes RBAC governs Kubernetes API actions, not cloud storage access. Cloud IAM grants cloud API permissions, not automatically Kubernetes API privileges. A Pod's ServiceAccount name is not evidence that its cloud federation works. Human SSO and inference gateway authentication are separate boundaries.

## 2. RBAC objects and permission evaluation

- **Role:** namespace-scoped set of Kubernetes API permissions.
- **ClusterRole:** cluster-scoped policy; can also be bound to a namespace through a RoleBinding.
- **RoleBinding:** grants Role or ClusterRole permissions within one namespace.
- **ClusterRoleBinding:** grants a ClusterRole at cluster scope.
- **ServiceAccount:** namespaced workload identity for Kubernetes API access.
- **Verb/resource/API group:** explicit authorization tuple. Include subresources such as `pods/log`, `deployments/scale`, `pods/exec` and `serviceaccounts/token` in risk reviews.
- **resourceNames:** narrows selected resource names for certain operations; cannot be treated as a universal guard for create/list/watch operations.
- **Aggregation:** aggregated ClusterRoles can change effective permissions as matching ClusterRoles are added; review aggregation labels and rules.

Kubernetes RBAC permissions are **additive**; there is no native deny rule. Removing one binding does not revoke another binding. `get secrets`, `create pods`, `pods/exec`, `impersonate`, `bind`, `escalate`, `create serviceaccounts/token` and workload mutation may confer indirect privileges. Treat access to arbitrary Pod creation in a namespace as sensitive because it can enable reading mounted credentials and data.

## 3. Read-only discovery [SAFE-READ]

```bash
kubectl config current-context
kubectl get namespaces --show-labels
kubectl get serviceaccounts -A
kubectl get roles,rolebindings -A
kubectl get clusterroles,clusterrolebindings
kubectl get pods -A -o custom-columns='NS:.metadata.namespace,POD:.metadata.name,SA:.spec.serviceAccountName'
kubectl auth can-i --list -n YOUR_NAMESPACE
kubectl auth can-i get pods -n YOUR_NAMESPACE
kubectl auth can-i create clusterrolebindings
```

`kubectl auth can-i` evaluates the caller unless authorized impersonation is used. If `--as=system:serviceaccount:NS:SA` is forbidden, that may mean the **caller cannot impersonate**; it does not establish the target's privileges. Avoid exporting `kubectl get secret -o yaml` and do not store ServiceAccount tokens in tickets.

Inspect the agent's actual ServiceAccount and bindings:
```bash
kubectl -n AGENT_NAMESPACE get deploy AGENT_DEPLOYMENT -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'
kubectl -n AGENT_NAMESPACE get sa AGENT_SA -o yaml
kubectl -n AGENT_NAMESPACE get rolebinding
```
Redact annotations that expose internal account identifiers or URLs when sharing evidence.

## 4. Design least-privilege roles

Start from observed required API calls, namespaces, verbs and resources. Distinguish read-only observability, namespace deployment and cluster-level controllers. Use separate ServiceAccounts for platform agent, inference workload, CronJob and CI automation. Prefer namespace-scoped RoleBindings and explicitly document unavoidable cluster-scoped permissions. Review privileges whenever a new TrueFoundry feature, operator or Kubernetes API resource is introduced.

**Staging example: a read-only application inspector**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tfy-inspector
  namespace: tfy-security-lab
automountServiceAccountToken: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tfy-inspector
  namespace: tfy-security-lab
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tfy-inspector
  namespace: tfy-security-lab
subjects:
  - kind: ServiceAccount
    name: tfy-inspector
    namespace: tfy-security-lab
roleRef:
  kind: Role
  name: tfy-inspector
  apiGroup: rbac.authorization.k8s.io
```

This ServiceAccount is **not mounted by default**. To exercise its Kubernetes API permissions from a Pod, use an approved short-lived projected token with the intended audience, or a controlled impersonation test performed by an authorized reviewer. Do not enable automount broadly merely to make the lab convenient.

## 5. Token handling and projected identities

Modern Kubernetes supports bound projected ServiceAccount tokens with audience and expiration. Avoid legacy long-lived ServiceAccount token Secrets. Set `automountServiceAccountToken: false` on workloads that do not call the Kubernetes API. When an API token is required, use a bounded projection and validate issuer, audience, expiry, rotation and file permissions.

```yaml
spec:
  serviceAccountName: tfy-runtime
  automountServiceAccountToken: false
  volumes:
    - name: k8s-api-token
      projected:
        sources:
          - serviceAccountToken:
              path: token
              audience: https://kubernetes.default.svc
              expirationSeconds: 3600
```

The audience shown is illustrative and must match the target verifier. Projected cloud identity tokens use cloud-specific audience and trust settings. A ServiceAccount token mounted in a Pod is sensitive even when short-lived.

## 6. AWS EKS workload identity

Two common supported approaches have distinct operational requirements:

**IRSA (IAM Roles for Service Accounts):** EKS OIDC issuer/provider, IAM role trust constrained by `sub` (namespace/ServiceAccount) and `aud` (typically `sts.amazonaws.com`), ServiceAccount role annotation, projected token and SDK support. Validate the actual cluster OIDC issuer and role trust. Namespace reuse and wildcards in trust policies can create lateral access.

**EKS Pod Identity:** requires compatible EKS Pod Identity Agent, an association between namespace/ServiceAccount and IAM role, appropriate trust policy and SDK support. Do not assume it is the same as IRSA or that its IAM trust configuration uses identical conditions.

Check S3, ECR, STS, KMS and Secrets Manager permissions independently; **node IAM role** and **Pod IAM role** must not be conflated. An image pull performed by the node/container runtime may use a different credential path than model artifact downloads from a running Pod.

## 7. Azure AKS Workload Identity

Validate AKS OIDC issuer, workload identity enablement, Microsoft Entra federated identity credential with exact subject `system:serviceaccount:NAMESPACE:NAME` and audience `api://AzureADTokenExchange`, managed identity/client ID, ServiceAccount annotation and required Pod label `azure.workload.identity/use: "true"` where mandated by the integration. Confirm supported SDK/credential chain and target Azure RBAC roles for Blob Storage, Key Vault or ACR as appropriate.

A private endpoint controls routing, **not** the identity permission. A `403` from Azure Storage can reflect IAM/data-plane roles, firewall/network restrictions or token audience; investigate the actual service response and cloud logs.

## 8. Admission, escalation and multi-tenancy

Use admission controls to constrain privileged Pods, hostPath/hostNetwork, hostPID, hostIPC, serviceAccountName choices, image registries and token automount as supported. Pod Security Admission, Kyverno/Gatekeeper or cloud-native policies can complement RBAC; verify installation and exemptions. Review RoleBinding/ClusterRoleBinding creation permissions and any escalation/bind/impersonate verbs. Namespace isolation requires RBAC **plus** network policy, storage isolation and cloud IAM.

## 9. Evidence and drift detection

Track changes to Roles, ClusterRoles, bindings, ServiceAccounts, workload identity associations and cloud trust. Correlate Kubernetes audit events with GitOps commit, CI identity, agent actions and cloud IAM logs. Alert on new cluster-admin bindings, wildcard verbs/resources, `secrets` reads, `pods/exec`, `impersonate`, privilege-escalation verbs and unexpected token issuance. Avoid collecting actual token values.

## 10. Twelve production troubleshooting runbooks

### RB-01 — Agent forbidden to list/watch resources
Read the exact API group/resource/verb and namespace from audit and logs; compare to reviewed RBAC. Add only the required permission after staged validation, not cluster-admin.

### RB-02 — RoleBinding points to wrong ServiceAccount
Check subject namespace/name, RoleRef and Pod serviceAccountName; correct binding in GitOps and retest allowed/denied actions.

### RB-03 — ClusterRoleBinding grants unexpected access
Review all additive bindings and aggregation rules. Revoke excess binding under security change control, check audit and confirm essential platform operations.

### RB-04 — ServiceAccount token missing
Check automount policy, projected volume, audience and file permissions. Enable only the required token for the authorized API consumer.

### RB-05 — Kubernetes API rejects token
Check issuer, audience, expiration, projected-token rotation and API server trust; avoid static token fallback.

### RB-06 — IRSA role assumption denied
Inspect OIDC provider/issuer, IAM trust subject/audience, ServiceAccount annotation, STS reachability and SDK credential chain. Correct narrowly scoped trust.

### RB-07 — EKS Pod Identity credentials unavailable
Check Pod Identity Agent health, association, IAM trust and SDK support; do not debug it as an IRSA-only issue.

### RB-08 — AKS Workload Identity token exchange fails
Check federated credential subject/audience/issuer, client ID, Pod label, token projection and Entra errors.

### RB-09 — Cloud object access denied
Separate identity assumption from service-level IAM/data-plane role and network path; grant minimum object/prefix/container scope.

### RB-10 — Pod can read Secrets unexpectedly
Inspect RBAC, workload creation/exec privileges, mounted Secrets and other bindings; treat exposure as security incident if unauthorized.

### RB-11 — CI/CD deployment blocked after hardening
Compare intended deployment verbs, namespace and admission constraints; stage the smallest policy fix and retest negative boundaries.

### RB-12 — GitOps restores unsafe RBAC
Find desired-state commit and reconciler owner, revert unsafe policy in Git, verify effective permissions and audit, and avoid persistent manual drift.

## 11. Change management and acceptance

Before modifying production RBAC or identity trust: inventory consumers, confirm observed required operations, review least privilege, test with disposable staging ServiceAccounts, set abort criteria, document rollback commit, preserve audit, verify both positive and negative tests, and obtain platform/security sign-off. Do not remove the only agent permission path before proving a replacement.

Complete the [companion lab](../labs/03-kubernetes-rbac-serviceaccounts-workload-identity-check.md), including **28 failure scenarios and 10 acceptance gates**. Formal review and live execution are independent of publication.

**Next:** Chapter 10.4 — Secrets Management, Encryption and Credential Rotation.
