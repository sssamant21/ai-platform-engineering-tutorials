# Chapter 10.7 — Production Security Hardening and Recovery Capstone

**Part 10 — Security | Final integrated enterprise SRE capstone**

## 1. Mission and completion standard

Design, harden, validate, detect and recover a production-like TrueFoundry AI platform across **human SSO, platform authorization, Kubernetes RBAC, ServiceAccounts, cloud workload identity, secrets, software/model supply chain, GPU runtime, inference gateway, audit and GitOps**. This capstone integrates Chapters 10.1–10.6.

**Scope:** isolated staging or a formally approved production exercise. **[SAFE-READ]** inventory only; **[LAB-MUTATE]** disposable staging; **[CHANGE-CONTROL]** production changes and incident containment. Publication is not a live execution, certification or vendor security assessment. Verify the actual TrueFoundry edition/version, Kubernetes distribution, cloud provider, installed controllers and policy engines before use.

**Outcome:** a signed-off security architecture, minimum-access matrix, reproducible configuration, detection coverage, simulated incidents, tested recovery, evidence package and prioritized gap register.

## 2. Reference architecture and trust boundaries

```text
Corporate IdP + MFA
  └─ TrueFoundry control plane (human roles, workspace/env authorization)
       ├─ CI/CD identity → source/build attestations → image/model registries
       └─ cluster agent → Kubernetes API → scoped RBAC
                             ├─ deployment controller / GitOps
                             ├─ inference gateway → authenticated model service
                             └─ GPU serving Pod (ServiceAccount + cloud federation)
                                       ├─ artifact storage
                                       ├─ approved secret manager
                                       └─ telemetry
IdP / platform / K8s / cloud / CI / gateway / runtime events → SIEM + evidence store
```

Keep management-plane login, Kubernetes API authorization, cloud data-plane IAM and inference endpoint authorization distinct. Private network paths are not substitutes for identity. A signed image is not proof of safe code or model behavior.

## 3. Architecture decisions and baseline inventory

Record: clusters/accounts/subscriptions; namespaces/workspaces; control-plane endpoints; agent and deployment identities; human groups/roles; CI/CD principals; workload identity mechanism (EKS IRSA, EKS Pod Identity or AKS Workload Identity); cloud object/registry/key access; External Secrets/CSI; network boundaries; admission engine; GPU device plugin/driver; audit sources; GitOps ownership; artifact digests; incident owners.

For every component capture **owner, desired state, evidence source, expected allow/deny, rollback and exception expiry**. Mark unsupported/uninstalled components as not applicable with justification, not as passed.

## 4. Hardening workstreams and acceptance evidence

| Workstream | Minimum controls | Evidence |
|---|---|---|
| Human identity | IdP MFA, approved SSO, least privilege, deprovisioning, break-glass | Authn/authz tests, IdP audit |
| Platform permissions | workspace/environment separation, controlled deploy and policy admin | Role matrix, denied deployment |
| Kubernetes RBAC | namespace scope, reviewed cluster permissions, no excess Secret/exec/token privileges | Binding diff, `can-i` tests |
| Workload identity | short-lived federation, exact issuer/subject/audience, narrow IAM | Trust review, scoped object allow/deny |
| Secrets | manager ownership, encryption, token hygiene, rotation, expiry alert | Rotation rehearsal, no values |
| Supply chain | protected source, SBOM, scan gate, immutable image/model digests, provenance | Build/deploy trace |
| Runtime | admission, non-root where compatible, seccomp, constrained GPU exceptions | Policy verdict, Pod spec |
| Network/inference | TLS, authn/authz, tenant isolation, controlled egress | Synthetic request matrix |
| Audit/incident | secure collection, redaction, detection, response roles, evidence | Correlated alert and tabletop |
| Recovery | GitOps restore, approved digests, credentials revoked, security revalidation | Recovery report and sign-off |

## 5. Safe baseline commands [SAFE-READ]

```bash
kubectl config current-context
kubectl get namespaces --show-labels
kubectl get serviceaccounts -A
kubectl get roles,rolebindings -A
kubectl get clusterroles,clusterrolebindings
kubectl get deployments,daemonsets -A -o wide
kubectl get pods -A -o custom-columns='NS:.metadata.namespace,POD:.metadata.name,SA:.spec.serviceAccountName,IMAGE:.spec.containers[*].image'
kubectl get networkpolicy -A
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations
kubectl get events -A --sort-by=.lastTimestamp | tail -50
```

Optional CRDs:
```bash
kubectl api-resources | grep -E 'externalsecret|secretproviderclass|clusterpolicy|constraint|certificate' || true
```

Never dump Kubernetes Secret values, cloud tokens, projected ServiceAccount tokens, private keys, raw prompts or production model weights. `kubectl auth can-i --as` requires caller impersonation privilege; inability to impersonate does not prove the target's authorization.

## 6. Deployment hardening and staged rollout

1. **Freeze baseline:** capture reviewed Git commit, deployed image/model digests, SLOs, owners, current roles and admission policy.
2. **Risk review:** identify changes to identity, cloud trust, Kubernetes API permissions, network, secrets, GPU runtime and CI.
3. **Stage:** apply scoped policies to disposable namespace and identities; confirm the intended positive path still works.
4. **Negative tests:** denied cross-workspace deploy, Secret read, Pod exec, cloud object, unsigned/unapproved artifact and anonymous/cross-tenant inference where applicable.
5. **Canary:** roll out approved artifact digest to staging/canary; monitor auth failures, 5xx, p95/p99 latency, model readiness and GPU health.
6. **Promote:** follow change approval, immutable release evidence and rollback checkpoint.
7. **Watch:** monitor privilege changes, admission rejects, secret sync, workload identity errors, gateway denials and telemetry gaps.
8. **Close:** security/SRE review and exception register; any failed mandatory gate blocks readiness.

Do not blindly apply a non-GPU Pod security baseline to a GPU device plugin or inference server. Document narrow compatibility exceptions with expiry and reviewer.

## 7. Four controlled incident scenarios

### Scenario A — Compromised deployment identity
**Inject (tabletop or disposable staging):** revoke a synthetic CI identity or simulate unapproved promotion. **Detect:** CI/cloud/platform audit and digest mismatch. **Contain:** freeze promotion, revoke trust, quarantine artifacts. **Recover:** verified pipeline identity and approved digest; negative permissions tests. **Evidence:** UTC timeline, CI run, IAM change, Git commit, reviewer.

### Scenario B — Unauthorized Kubernetes privilege
**Inject:** submit a harmless denied RBAC/privileged manifest to existing staging policy, or tabletop unexpected cluster-admin binding. **Detect:** Kubernetes audit/admission. **Contain:** review and remove unauthorized binding under incident authority. **Recover:** GitOps reconciliation and confirm additive bindings do not preserve access.

### Scenario C — Secret or workload identity compromise
**Inject:** rotate a disposable credential or simulate unauthorized Secret access. **Detect:** manager/cloud/Kubernetes audit and app auth errors. **Contain:** revoke affected identity, block access and preserve evidence. **Recover:** issue new scoped credential, resync ESO/CSI, verify app reload and denied old credential. **Never restore compromised credential as rollback.**

### Scenario D — Inference or model supply-chain incident
**Inject:** synthetic invalid inference authorization and harmless checksum mismatch/tabletop poisoned model. **Detect:** gateway denial, provenance gate, model checksum. **Contain:** restrict route and quarantine artifact. **Recover:** approved image/model digests, tenant authorization checks, request completion and SLOs.

## 8. Recovery runbook and abort criteria

**Declare:** incident commander and security lead determine impact and change authority. **Preserve:** authorized sanitized audit/configuration snapshots, immutable artifact IDs and UTC timestamps. **Contain:** compromised identities/artifacts/traffic without destroying necessary evidence. **Eradicate:** correct policy, code, artifact or IAM trust. **Restore:** approved GitOps revision, image/model digests and fresh credentials. **Validate:** authorized requests succeed; unauthorized requests fail; audit flows; GPU runtime and inference SLO recover. **Close:** reviewer approval, incident RCA, action register and watch period.

**Abort/rollback triggers:** unauthorized access succeeds; new cross-tenant exposure; unexplained privilege elevation; production auth outage; critical telemetry loss; persistent inference SLO breach; GPU serving failure. Rollback must never re-enable a known compromise.

## 9. Twelve capstone operational runbooks

1. **SSO outage:** distinguish IdP vs platform; invoke approved break-glass, preserve audit and expire access.
2. **Overprivileged platform user:** scope roles, revoke sessions, validate deployment boundaries.
3. **Agent RBAC denied:** identify exact verb/resource/namespace, restore least-privilege permission.
4. **Cloud federation broken:** verify issuer/subject/audience and role association; no wildcard trust.
5. **Secrets stale after rotation:** inspect ESO/CSI, app reload and connection pool; reissue approved credential.
6. **TLS/mTLS outage:** validate live certificate chain, SAN/SNI and trust rollover; do not disable TLS checks.
7. **Untrusted artifact admitted:** quarantine image/model, restore verified digest and inspect admission gaps.
8. **GPU serving fails after hardening:** inspect driver/device plugin/Pod security exception and canary rollback.
9. **Cross-tenant inference succeeds:** isolate route, preserve request IDs, escalate security/privacy and revalidate auth.
10. **Unexpected privileged DaemonSet:** investigate host access, image provenance, node isolation and audit.
11. **Audit pipeline silent:** restore collection, estimate blind spot and record risk acceptance.
12. **GitOps reintroduces unsafe state:** correct desired state, reconcile, validate denied actions and recurrence alerts.

## 10. Production scorecard

A control passes only with **timestamped evidence, environment, owner, expected/actual behavior, rollback and reviewer**. Mark **PASS / FAIL / BLOCKED / NOT APPLICABLE**; do not count BLOCKED as PASS. Any critical cross-tenant, credential, privilege or artifact-integrity failure blocks readiness.

| Dimension | Example measurable indicator |
|---|---|
| Identity | unauthorized actions denied; MFA/SSO coverage |
| Cloud IAM | least-privilege allow/deny and trust constraints |
| Supply chain | deployed image/model match approved digests |
| Secrets | rotation and revocation observed within agreed window |
| Runtime | admission and GPU serving compatible |
| Detection | alert routed to owner within defined target |
| Recovery | restore verified revision without compromised credentials |
| Evidence | all required gates signed by SRE and security |

Set numeric SLO/MTTR/RTO targets from your own baseline and approved business requirements, not invented platform guarantees.

## 11. Final deliverables and completion

Deliver: architecture/threat model, identity and IAM matrix, RBAC audit, secrets/rotation plan, artifact provenance inventory, admission and GPU exception register, network/inference auth tests, detection coverage matrix, incident timeline, recovery evidence, exception register and owner-approved sign-off.

The [companion acceptance lab](../labs/07-production-security-hardening-recovery-capstone-check.md) includes **36 failure scenarios and 15 acceptance gates**. Completion of the written curriculum is separate from hands-on execution, formal review and production certification.

**Part 10 — Security curriculum publication complete (7/7).** Next curriculum part: **Part 11 — Observability** (confirm exact chapter plan in the repository before publishing).
