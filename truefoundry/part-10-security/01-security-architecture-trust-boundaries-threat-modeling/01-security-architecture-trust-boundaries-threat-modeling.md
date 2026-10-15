# Chapter 10.1 — TrueFoundry Security Architecture, Trust Boundaries and Threat Modeling

**Part 10 — Security | Canonical tutorial | Production-focused**

## Outcomes and safety contract

Build a verifiable security model for a TrueFoundry deployment covering control plane, cluster agent, Kubernetes runtime, inference endpoints, GPU nodes, cloud identity, secrets, model artifacts and external dependencies. Produce a trust-boundary diagram, data-flow inventory, threat register, security-control mapping, detection plan and rollback criteria.

**Safety:** [SAFE-READ] inventory only; [LAB-MUTATE] disposable staging; [CHANGE-CONTROL] approved production change. Do not assume that a TrueFoundry feature, authentication method or deployment architecture exists in every version. Validate installed components, configuration and current vendor documentation. Never expose credentials, customer prompts, model weights or PHI in screenshots, terminal history or logs.

## 1. Reference security architecture

```text
User / CI / automation
   → identity provider / SSO / API credential
   → TrueFoundry management control plane
       ↔ installed cluster agent (direction and transport must be verified)
           ↔ Kubernetes API (ServiceAccount / RBAC)
               → workload namespace / deployment controller
               → GPU node and container runtime
               → model-serving Pod → ingress / gateway → authorized client
                                  ↘ model registry / object store
                                  ↘ internal data services
                                  ↘ observability pipeline
Cloud IAM / workload identity / secret manager ↔ approved components
```

Treat management-plane, deployment-plane, inference-plane and data-plane paths as **distinct security boundaries**. A successful TLS connection to the control plane does not imply that model-serving authorization or cloud IAM is correct. A private endpoint does not imply it is authenticated.

## 2. Identify assets and adversaries

| Asset | Risk | Required owner and evidence |
|---|---|---|
| Control-plane identity and sessions | privilege escalation/account compromise | SSO/MFA/session settings and audit logs |
| Agent credential and Kubernetes ServiceAccount | cluster takeover | namespace, Role/ClusterRole bindings, token audience |
| Deployment manifests and CI pipeline | supply-chain manipulation | signed/reviewed commits, immutable images |
| Model artifacts and registry | model theft or poisoning | provenance, integrity, storage IAM |
| Inference API and prompt data | unauthorized access/data exfiltration | gateway auth, rate limits, logging policy |
| GPU worker nodes | isolation escape/resource abuse | node patching, runtime policy, admission |
| Cloud identity | lateral movement into storage/secrets | least-privilege IAM and identity federation |
| Secrets and TLS keys | credential disclosure | manager, rotation, access audit |
| Telemetry | sensitive data leakage or missing detection | redaction, retention and alerting |

Threat actors can include external unauthenticated clients, compromised developer credentials, malicious/compromised CI jobs, overprivileged workloads, compromised dependencies and insiders exceeding approved access. Model threats using STRIDE or another organization-approved framework; document likelihood/impact rather than assigning arbitrary severity.

## 3. Discover the actual environment [SAFE-READ]

```bash
kubectl config current-context
kubectl get ns --show-labels
kubectl get sa -A
kubectl get role,rolebinding,clusterrole,clusterrolebinding -A
kubectl get deployments,daemonsets -A
kubectl get ingressclass,ingress -A
kubectl get gatewayclass,gateway,httproute -A 2>/dev/null || true
kubectl get networkpolicy -A
kubectl get pods -A -o wide
```

The cluster-scoped RBAC listing can be large; restrict output and redact identities before sharing. Discover the actual TrueFoundry agent namespace, service account, container image digest, deployment source, network egress and runtime permissions. Avoid `kubectl get secret -o yaml`: base64 is **not** encryption and can reveal credentials.

For any target ServiceAccount:
```bash
kubectl auth can-i --as=system:serviceaccount:AGENT_NAMESPACE:AGENT_SA get pods -A
kubectl auth can-i --as=system:serviceaccount:AGENT_NAMESPACE:AGENT_SA create clusterrolebindings
```
These impersonation checks require caller impersonation privileges and may fail with Forbidden; failure to impersonate is **not** proof the target identity lacks permission. Use a security-approved access-review process.

## 4. Trust-boundary mapping

For every hop, record: initiating principal, authentication method, authorization decision, network source/destination, TLS/mTLS, secrets used, data classification, audit events, expected denial, incident owner and rollback.

1. **User → control plane:** SSO/MFA and organization/project/workspace role mapping; verify actual platform permissions.
2. **Control plane ↔ agent:** verify direction, TLS, certificate trust, credential lifecycle, namespace and endpoint allowlist.
3. **Agent → Kubernetes API:** minimal RBAC verbs/resources/namespaces, token audience, service account mount and audit.
4. **Deployment → registry/storage:** image pull vs Pod artifact fetch; workload identity, checksum/signature and private endpoints.
5. **Client → inference gateway:** DNS/TLS, authenticated route, authorization, request size/rate quotas and tenant isolation.
6. **Serving Pod → data services:** cloud IAM, Kubernetes NetworkPolicy, encryption and secrets rotation.
7. **Serving Pod → telemetry:** redaction, retention, transport encryption and least-privilege collector access.

## 5. Threat model worksheet

| ID | Boundary | Threat | Impact | Existing control | Evidence | Owner | Residual risk |
|---|---|---|---|---|---|---|---|
| T01 | User/control plane | stolen access token | unauthorized deployment | SSO/MFA, short-lived tokens | IdP audit and revocation test | IAM | Review |
| T02 | Agent/Kubernetes API | excessive ClusterRole | cluster-wide changes | scoped RBAC, audit | role review and negative test | Platform | Review |
| T03 | CI/registry | substituted image/model | code/model compromise | immutable digest, provenance | admission and artifact metadata | Supply chain | Review |
| T04 | Inference endpoint | anonymous access | data exfiltration/cost | authn/authz and rate limits | 401/403 negative test | AppSec | Review |
| T05 | Pod/cloud storage | overprivileged identity | model/data theft | scoped cloud role, private route | access denied test | Cloud | Review |
| T06 | Telemetry | prompt/secret leakage | confidentiality breach | redaction and retention | sanitized sample review | SRE/Security | Review |

Add environment-specific threats such as cross-tenant routing, SSRF via model/tool endpoints, untrusted model downloads, compromised GPU drivers, supply-chain CVEs and shared-node isolation.

## 6. Defense in depth and control ownership

**Identity:** centralized SSO/MFA for humans; short-lived workload credentials and least-privilege service accounts for machines. Review break-glass access and its expiry.

**Authorization:** map TrueFoundry roles, Kubernetes RBAC and cloud IAM independently; privilege in one layer must not imply another. Verify denied actions as well as allowed actions.

**Network:** private endpoints where appropriate, gateway authentication, default-deny policies with required DNS/telemetry allows, cloud SG/NSG/firewall restrictions and explicit outbound controls. Standard Kubernetes NetworkPolicy is L3/L4, not portable FQDN policy.

**Secrets:** use approved secrets manager/external-secret integration; avoid embedding secrets in Git, ConfigMaps, Pod environment dumps or tickets. Rotate compromised credentials under incident procedure.

**Supply chain:** immutable image digests, artifact checksums/signatures where supported, vulnerability scanning, SBOM and admission policies. Do not trust mutable `latest` tags for production release identity.

**Runtime:** hardened security contexts, non-root where compatible, restricted capabilities, seccomp, resource limits, admission and node patching. GPU device plugins/driver stacks need version-specific validation; do not blindly apply CPU-only hardening that breaks GPU workloads.

**Data and observability:** redact prompts/PII/secrets, enforce retention, encrypt transit and storage, audit privileged operations, alert on denied access and unexpected egress.

## 7. Security acceptance tests

Read-only evidence should establish effective identities, role mappings, secret provenance, endpoint exposure, policy enforcement and logging. In staging, test authorized deployment/inference and negative cases: unprivileged deployment denied; unauthorized inference rejected; forbidden cloud object inaccessible; cross-namespace connection blocked; tampered artifact rejected if integrity policy exists. If a control is unavailable, mark **gap** and assign owner rather than claiming pass.

## 8. Ten production security runbooks

### RB-01 — Compromised user or API token
Revoke/disable credential in the actual IdP/platform, review audit logs, terminate active sessions where supported, scope affected resources and rotate downstream secrets. Follow incident command and preserve evidence.

### RB-02 — Overprivileged cluster agent
Inventory effective RBAC and recent actions; stage a least-privilege replacement, test required deployment operations, then change under approval. Avoid abruptly removing permissions that may interrupt reconciliation.

### RB-03 — Unexpected public inference endpoint
Inspect DNS, LB scheme, firewall, gateway routes and auth; contain exposure without disabling legitimate internal access, preserve evidence and assess data access.

### RB-04 — Unauthorized inference succeeds
Treat as a security incident. Identify bypass route/hostname and tenant context; restore enforced authentication and authorization, verify negative tests and rotate affected credentials as required.

### RB-05 — Model artifact integrity concern
Quarantine suspicious artifact, pin last-known-good digest, verify provenance and inspect deployment history. Do not reload untrusted weights into a privileged serving environment.

### RB-06 — Secret leaked into logs or Git
Revoke/rotate immediately per incident policy, restrict exposure, inspect access/audit trail and scrub copies through approved procedures. Deleting a commit alone does not invalidate a leaked credential.

### RB-07 — Cloud IAM access denied after hardening
Compare workload identity binding, role permissions, audience/trust and private endpoint. Restore narrowly scoped required permissions; avoid wildcard administrative grants.

### RB-08 — Unexpected egress or lateral traffic
Correlate CNI flow verdicts, cloud flow logs, DNS, Pod identity and destination; isolate compromised workload under incident approval while preserving evidence.

### RB-09 — CI/CD supply-chain compromise
Pause deployments, revoke affected automation credentials, inspect provenance, pin known-good images and redeploy only reviewed artifacts. Verify no malicious changes remain in GitOps desired state.

### RB-10 — Security logging gap
Validate audit collection, retention, access controls and telemetry path. Restore evidence pipeline; do not declare incident closed without sufficient visibility.

## 9. Production change, rollback and acceptance

Security change record: affected trust boundary, principal/permissions, risk, change owner, Git commit, baseline allowed/denied operations, monitoring, staged rollout, rollback plan and sign-off. Never use a blanket policy deletion or TLS bypass as a routine recovery.

Complete the [Chapter 10.1 companion lab](../labs/01-security-architecture-trust-boundaries-threat-modeling-check.md): 24 scenarios and 10 acceptance gates. **Publication does not imply vendor review, execution or certification.**

**Next:** Chapter 10.2 — Identity, SSO, Authentication and Authorization.
