# Chapter 10.1 — Hands-on Lab: Security Architecture and Threat Modeling

**Part 10 | Isolated staging | No production mutation**

## Goals and safety

Inventory the TrueFoundry control plane, cluster agent, Kubernetes runtime, inference endpoints, identity and cloud services. Create a threat model, validate least privilege and document security incident recovery. Use [SAFE-READ] for inventory, [LAB-MUTATE] for authorized disposable staging tests and [CHANGE-CONTROL] for production approval. Never disclose credentials, model data or customer prompts.

## Lab 0 — Baseline [SAFE-READ]

```bash
kubectl config current-context
kubectl get ns --show-labels
kubectl get sa -A
kubectl get role,rolebinding,clusterrole,clusterrolebinding -A
kubectl get deploy,ds -A
kubectl get ingressclass,ingress -A
kubectl get networkpolicy -A
```

Record actual TrueFoundry version and agent topology, CNI, ingress controller, IdP, cloud IAM, registry, model artifact storage, secret manager and audit owner. Do not export Kubernetes Secret values.

## Lab 1 — Trace trust boundaries [SAFE-READ]

Map human/CI → control plane → agent → Kubernetes API → workload; client → TLS gateway → inference Pod; Pod → artifact storage, private dependencies and telemetry. Record principal, permission, network path, data classification, audit signal and recovery owner for each boundary.

## Lab 2 — Identity and privilege [SAFE-READ]

Review ServiceAccount-to-RoleBinding and ClusterRoleBinding relationships, platform role mappings and cloud identity. Verify denied actions using approved read-only or staging identities. A failure to impersonate another ServiceAccount is not proof of that account's permissions.

## Lab 3 — Secure endpoint and supply chain [SAFE-READ]

Verify TLS/SNI, authentication, authorized vs denied inference, public/private LB exposure, network policy enforcement, immutable image digest and model artifact provenance. Check log redaction and retention without copying sensitive data.

## Lab 4 — Negative tests [LAB-MUTATE]

With disposable staging identities, confirm unprivileged deployment is denied, unauthenticated inference is rejected, forbidden artifact access is blocked, and cross-namespace access is denied when supported by the CNI. Stop immediately if a test affects shared services.

## Lab 5 — Threat workshop and incident tabletop

For each threat capture likelihood, impact, detection, prevention, owner, residual risk and remediation. Rehearse incident command, evidence preservation, credential revocation, containment and known-good GitOps recovery. No live attack simulation is required.

## 24 scenario checks

| # | Scenario | Review focus |
|---|---|---|
| 01 | Human SSO policy gap | IdP configuration |
| 02 | Stolen API credential | Revocation and audit |
| 03 | Excessive platform role | Role mapping |
| 04 | Broad agent RBAC | Kubernetes permissions |
| 05 | ServiceAccount token exposure | Token lifecycle |
| 06 | Overprivileged CI identity | Federation trust |
| 07 | Mutable registry tag | Image digest |
| 08 | Untrusted model artifact | Provenance |
| 09 | Admission control gap | Deployment policy |
| 10 | Credential in Git | Rotation |
| 11 | Credential in logs | Redaction |
| 12 | Anonymous inference accepted | Authentication |
| 13 | Alternate route bypass | Gateway rules |
| 14 | Public access to private endpoint | LB and firewall |
| 15 | TLS identity mismatch | SAN and SNI |
| 16 | Unexpected namespace access | CNI enforcement |
| 17 | Unrestricted outbound traffic | Egress controls |
| 18 | Excessive storage permissions | Cloud IAM |
| 19 | Sensitive prompts in telemetry | Retention and redaction |
| 20 | Tenant isolation gap | Identity and routing |
| 21 | GPU node patch gap | Driver lifecycle |
| 22 | Missing audit events | Collection pipeline |
| 23 | Unreviewed emergency access | Expiration and approval |
| 24 | GitOps reintroduces insecure config | Desired-state recovery |

Mark each scenario **tested**, **tabletop**, **blocked** or **not executed**. Capture evidence and remediation owner.

## Ten acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Approved scope and component inventory | Pending |
| G2 | Management/runtime trust-boundary diagram | Pending |
| G3 | Human and workload identity inventory | Pending |
| G4 | Kubernetes RBAC and cloud IAM review | Pending |
| G5 | TLS, gateway authentication and exposure check | Pending |
| G6 | Secret storage and rotation owner | Pending |
| G7 | CI/model artifact integrity controls | Pending |
| G8 | Authorized and denied network/access tests | Pending |
| G9 | Audit coverage and incident tabletop | Pending |
| G10 | Threat register, rollback and security sign-off | Pending |

**Acceptance:** evidence and reviewer sign-off for every applicable gate. Publication does not mean execution or certification.

**Next:** Chapter 10.2 — Identity, SSO, Authentication and Authorization.
