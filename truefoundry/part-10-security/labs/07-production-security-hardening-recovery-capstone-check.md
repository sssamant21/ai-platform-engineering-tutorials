# Chapter 10.7 — Final Capstone Lab: Production Security Hardening and Recovery

**Part 10 | Integrated staging exercise | Execution pending**

## Mission, rules and prerequisites

Integrate all seven security chapters into one reproducible acceptance exercise: SSO/platform roles, Kubernetes RBAC, EKS/AKS workload identity, secrets, cloud IAM, supply chain, GPU runtime, inference authorization, audit, incident command and recovery.

**[SAFE-READ]** inspect; **[LAB-MUTATE]** disposable staging; **[CHANGE-CONTROL]** production approval. Obtain written scope, test identities, synthetic model/payloads, safe rollback, on-call and security reviewer. Never expose real credentials, prompts, PHI, private model weights or customer data. No intentionally malicious images or destructive cluster-wide changes.

## Phase 0 — Freeze baseline

Record Kubernetes context, cloud account/subscription, TrueFoundry version, agent ServiceAccount, GitOps revision, image/model digests, IdP integration, secret manager, workload identity mechanism, admission engine, GPU runtime and telemetry sources.

```bash
kubectl config current-context
kubectl get ns --show-labels
kubectl get sa -A
kubectl get rolebinding,clusterrolebinding -A
kubectl get pods -A -o wide
kubectl get networkpolicy -A
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

## Phase 1 — Identity and authorization

Test disposable SSO user, MFA, viewer/deployer/security roles and group removal. Verify allowed staging deployment and denied production/identity changes. Review agent RBAC and ServiceAccount automount. Authorized impersonation tests only; if not permitted, mark blocked.

## Phase 2 — Cloud and secrets

Inspect EKS IRSA/Pod Identity or AKS Workload Identity as actually installed. Verify scoped synthetic object access and denied out-of-scope object access. Rotate disposable secret; validate ESO/CSI sync, application reload, old-credential revocation and no plaintext evidence. Confirm TLS certificate validity.

## Phase 3 — Supply chain and runtime

Trace reviewed source commit → CI identity → SBOM/scan → signed/pinned image digest (if enforced) → pinned model checksum → GitOps → Pod. Validate existing admission rules using harmless rejected staging manifests. Confirm GPU serving health and documented narrow exceptions.

## Phase 4 — Inference and observability

Use synthetic requests to test valid, missing and wrong-scope credentials; verify tenant isolation and streaming behavior. Correlate IdP, platform (if available), Kubernetes, cloud, CI, registry, gateway and runtime audit events. Validate alert owner, evidence retention and redaction.

## Phase 5 — Four incident simulations

A. Compromised synthetic CI identity: freeze promotion, revoke, verify digests, restore.
B. Unauthorized privilege attempt: denied staging RBAC/admission request, inspect audit, confirm least privilege.
C. Synthetic credential leak/rotation: revoke and replace, confirm old identity denied.
D. Unauthorized inference or artifact integrity failure: block route/artifact, restore reviewed model/image digests, validate SLO.

Record incident commander, UTC timeline, containment decision, evidence reference, rollback, approved recovery and follow-up owner. Use tabletop if unsafe.

## Phase 6 — Restore and cleanup

Reconcile approved GitOps commit; verify positive and negative tests, audit pipeline, credential revocation, GPU readiness and synthetic inference success. Delete only lab-owned resources. Preserve sanitized report and sign-off.

## Thirty-six failure scenarios

| # | Scenario | Primary control |
|---|---|---|
| 01 | SSO issuer mismatch | IdP |
| 02 | MFA bypass or missing requirement | Conditional access |
| 03 | Disabled user retains access | Revocation |
| 04 | Viewer deploys production | Platform RBAC |
| 05 | CI role changes identity policy | Separation of duties |
| 06 | Agent RBAC missing verb | Kubernetes API |
| 07 | Wrong ServiceAccount bound | RoleBinding |
| 08 | Unexpected cluster-admin | Privilege drift |
| 09 | `secrets` read permitted | Kubernetes RBAC |
| 10 | ServiceAccount token auto-mounted | Token posture |
| 11 | EKS IRSA issuer/subject mismatch | IAM trust |
| 12 | EKS Pod Identity association absent | Cloud identity |
| 13 | AKS federated credential wrong | Entra trust |
| 14 | Cloud object access overbroad | Data-plane IAM |
| 15 | KMS/Key Vault decrypt denied | Key authorization |
| 16 | Secret manager unreachable | Network |
| 17 | ESO reconciliation stale | Sync |
| 18 | CSI mount fails | Driver/provider |
| 19 | Application caches old secret | Reload |
| 20 | Revoked credential still accepted | Session/token |
| 21 | TLS certificate expired | Certificate lifecycle |
| 22 | CI artifact digest mismatch | Provenance |
| 23 | Image signature untrusted | Admission |
| 24 | SBOM or scan missing | Supply chain |
| 25 | Model checksum mismatch | Model integrity |
| 26 | Privileged Pod admitted unexpectedly | Runtime |
| 27 | GPU plugin or CUDA mismatch | GPU compatibility |
| 28 | Unapproved privileged DaemonSet | Node security |
| 29 | Anonymous inference accepted | Gateway authn |
| 30 | Cross-tenant inference succeeds | Gateway authz |
| 31 | Unexpected model-serving egress | Network |
| 32 | Cloud audit missing | Telemetry |
| 33 | SIEM ingest delay | Detection |
| 34 | Break-glass action not audited | Emergency control |
| 35 | GitOps restores unsafe policy | Desired state |
| 36 | Recovery reuses compromised credential | Recovery integrity |

Each case must include expected vs actual, scope, UTC timestamp, sanitized evidence, detection owner, containment, recovery, rollback and reviewer. Do not execute risky failures outside isolated approved scope.

## Fifteen acceptance gates

| Gate | Requirement | Status |
|---|---|---|
| G1 | Written scope, architecture and approved rollback | Pending |
| G2 | SSO/MFA and deprovisioning verified | Pending |
| G3 | Platform least-privilege role matrix verified | Pending |
| G4 | Kubernetes RBAC and escalation boundaries verified | Pending |
| G5 | ServiceAccount token posture verified | Pending |
| G6 | Cloud workload identity trust and allow/deny verified | Pending |
| G7 | Secret sync, rotation, revocation and encryption reviewed | Pending |
| G8 | TLS certificate identity/renewal validated | Pending |
| G9 | Image/model provenance and immutable digests verified | Pending |
| G10 | Admission and GPU runtime compatibility validated | Pending |
| G11 | Inference authn/authz and tenant isolation tested | Pending |
| G12 | Audit collection, detection and redaction validated | Pending |
| G13 | Incident command, containment and evidence rehearsed | Pending |
| G14 | GitOps recovery, cleanup and negative tests verified | Pending |
| G15 | Risk exceptions and SRE/security sign-off recorded | Pending |

**Exit criteria:** no critical failed gates, applicable controls evidenced, exceptions documented with expiry and owner, recovery approved by SRE and security. BLOCKED or NOT EXECUTED are never PASS.

## Final report template

- Scope and reviewers:
- Environment, versions and GitOps revision:
- Architecture and identity/IAM matrix:
- Source/build/image/model digests:
- Secrets and certificate evidence (no values):
- Allowed/denied test results:
- Security detection and incident timeline:
- Recovery time and rollback commit:
- Open gaps and compensating controls:
- Gate status G1–G15:
- SRE sign-off / Security sign-off:
- Next review date:

**Part 10 — Security: 7/7 chapters and 7/7 labs published.** Live lab execution and formal reviews remain pending.
