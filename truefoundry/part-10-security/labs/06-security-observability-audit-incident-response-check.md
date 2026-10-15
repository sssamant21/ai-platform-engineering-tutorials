# Chapter 10.6 — Hands-on Lab: Security Observability, Audit and Incident Response

**Part 10 | Authorized staging/tabletop | Live execution pending**

## Mission and guardrails

Map audit coverage across TrueFoundry, IdP, Kubernetes, cloud IAM, CI, registry, inference gateway and GPU runtime. Test detection with **synthetic staging events**, rehearse containment and recovery, and prove evidence integrity. **[SAFE-READ]** observation; **[LAB-MUTATE]** isolated staging only; **[CHANGE-CONTROL]** production authorization. Never export real credentials, prompts, PHI, private model weights or raw customer inference requests.

## Lab 0 — Scope and telemetry inventory [SAFE-READ]

```bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -40
kubectl get rolebinding,clusterrolebinding -A
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations
kubectl get networkpolicy -A
```

Record the actual source/collector/SIEM path for IdP, TrueFoundry platform (if available), Kubernetes API audit, cloud IAM, CI/CD, registry, gateway and CNI. Record time sync, ingest lag, retention, redaction and on-call owner. Missing sources must be marked gaps.

## Lab 1 — Evidence and correlation [SAFE-READ]

Select a benign staging deployment. Correlate Git commit, CI build, platform action, Kubernetes audit request, Pod UID, image/model digest, gateway request ID and cloud object access. Preserve sanitized UTC timeline and hash of exported evidence; store it in an access-controlled evidence location.

## Lab 2 — Detection tests [LAB-MUTATE]

Generate approved synthetic events: denied read, denied deployment, unauthorized staging inference, rejected admission manifest and harmless CI policy violation. Validate alert receipt, enrichment, severity, owner and runbook. Do not create actual privileged bindings, exfiltrate data or deploy malicious artifacts.

## Lab 3 — Audit pipeline failure tabletop

Simulate (on paper or isolated lab collector) ingest delay, dropped events, bad timestamp, retention expiry, missing cloud audit and SIEM outage. Document fallback evidence sources and risk acceptance; do not disable production audit pipelines.

## Lab 4 — Security incident tabletop

Rehearse compromised user token, unexpected cluster-admin grant, unauthorized model promotion, leaked synthetic secret, cross-tenant inference and GPU node anomaly. Assign incident commander, security lead, evidence custodian and communications. Record containment authority and decisions.

## Lab 5 — Recovery validation

Verify credential revocation behavior with disposable identities, authorized vs denied access, immutable image/model digest, GitOps convergence, logging restored and alert recurrence watch. Compromised credentials/artifacts must never be restored as rollback.

## Lab 6 — Cleanup and report

Remove lab-only events/identities/resources, preserve approved sanitized evidence and file detection gaps with owner and deadline. Obtain SRE/security sign-off before marking gates passed.

## Thirty-two scenarios

| # | Scenario | Detection / response focus |
|---|---|---|
| 01 | IdP sign-in anomaly | MFA/session logs |
| 02 | Disabled user activity | Revocation |
| 03 | Platform role escalated | Privilege audit |
| 04 | Production deploy unapproved | Change correlation |
| 05 | Agent identity unexpected | Kubernetes audit |
| 06 | Cluster-admin binding created | RBAC alert |
| 07 | Wildcard Secret access granted | Permission drift |
| 08 | Unexpected `pods/exec` | Interactive access |
| 09 | TokenRequest anomaly | ServiceAccount token |
| 10 | CI federation trust broadened | Cloud IAM |
| 11 | Cloud role assumption denied | Trust mismatch |
| 12 | Storage object accessed unusually | Cloud data events |
| 13 | KMS decrypt anomaly | Key audit |
| 14 | Image digest differs from approved | Artifact integrity |
| 15 | Model checksum mismatch | Model supply chain |
| 16 | Unsigned image admitted | Policy gap |
| 17 | Admission webhook failure | Enforcement |
| 18 | Privileged Pod created | Runtime audit |
| 19 | Unapproved GPU DaemonSet | Node security |
| 20 | Anonymous inference accepted | Authn bypass |
| 21 | Cross-tenant inference | Authz failure |
| 22 | Repeated gateway 401/403 | Brute-force/noise |
| 23 | Alternate hostname bypass | Route coverage |
| 24 | Unexpected egress | Network flow |
| 25 | Secret in application logs | Redaction |
| 26 | Credential rotation failure | Revocation |
| 27 | TLS certificate expires | Expiry alert |
| 28 | Kubernetes audit source absent | Coverage gap |
| 29 | SIEM ingest lag | Pipeline SLO |
| 30 | Evidence timestamps skewed | Correlation |
| 31 | Break-glass action unlogged | Emergency audit |
| 32 | GitOps restores insecure state | Desired-state recovery |

For each scenario: expected signal, actual signal, source, correlation ID, detection delay, severity, responder, containment, recovery, rollback and sanitized evidence. If not safely executable, mark tabletop.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Approved scope, telemetry inventory and owners | Pending |
| G2 | IdP and platform audit coverage verified | Pending |
| G3 | Kubernetes and cloud audit coverage verified | Pending |
| G4 | CI/registry/model artifact provenance correlated | Pending |
| G5 | Gateway/network/GPU runtime signals mapped | Pending |
| G6 | Redaction, retention and evidence integrity reviewed | Pending |
| G7 | Privilege escalation and unauthorized access alerts tested | Pending |
| G8 | Ingest-lag and missing-source alerts validated | Pending |
| G9 | Incident roles, escalation and communications rehearsed | Pending |
| G10 | Credential revocation and containment tabletop completed | Pending |
| G11 | GitOps/artifact recovery and negative tests validated | Pending |
| G12 | Cleanup, gap register and security/SRE sign-off | Pending |

**Pass:** all applicable gates evidenced with timestamp, owner and reviewer. Publishing the lab does not constitute execution.

## Evidence worksheet

UTC timestamp, environment, source/collector, principal category, action, request/correlation ID, artifact digest, detection rule, observed latency, sanitized audit reference, incident commander, containment, rollback commit, review status.

**Next:** Chapter 10.7 — Production Security Hardening and Recovery Capstone.
