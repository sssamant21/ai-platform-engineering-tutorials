# Chapter 10.6 — Security Observability, Audit and Incident Response

**Part 10 — Security | Canonical enterprise SRE tutorial**

## 1. Objectives and safety

Design detection, investigation, containment and recovery for security incidents affecting TrueFoundry control-plane access, cluster agents, Kubernetes workloads, model-serving endpoints, cloud IAM, GPU nodes and the software/model supply chain. Build auditable evidence pipelines, useful alert rules, incident command, forensics, credential revocation, GitOps recovery and post-incident acceptance.

**Safety labels:** [SAFE-READ] observe and inspect metadata; [LAB-MUTATE] synthetic staging events only; [CHANGE-CONTROL] approved production change/containment. Verify available TrueFoundry audit events, product version, cloud audit integrations, SIEM, CNI and Kubernetes API audit configuration. Do not claim a vendor event or field exists without verifying it. Never export customer prompts, PHI, tokens, secrets or model weights into a shared investigation artifact.

## 2. Security observability architecture

```text
IdP/SSO ──────────────────────┐
TrueFoundry platform audit* ──┤
Kubernetes API audit ─────────┤
Admission/GitOps/CI events ───┤→ secure collection → normalize → SIEM/detections
Cloud IAM + object access ────┤                    ↘ evidence archive / retention
Gateway/WAF/auth decisions ───┤                    ↘ alert → on-call / incident commander
CNI / network flow verdicts ──┤
Node/GPU/runtime signals ─────┘
*Availability and schema depend on deployed platform and edition.
```

A telemetry source is not necessarily enabled. Identify source owner, scope, log transport, expected event fields, retention, time sync, failure mode and test evidence. Security visibility must cover both **allowed privileged actions** and **denied access attempts**.

## 3. Security signals by trust boundary

| Boundary | Events | High-value detection |
|---|---|---|
| Human SSO | sign-in, MFA, session, group change | anomalous privileged login, disabled-user activity |
| Platform | deployment, role/config change, token lifecycle | unapproved production deployment |
| Kubernetes API | RBAC, exec, Secret read, token issuance | new cluster-admin binding, unexpected `pods/exec` |
| Cloud IAM | role assumption, trust-policy change, storage access | cross-account assumption, sensitive object read |
| CI/registry | pipeline identity, signing, image digest | unreviewed artifact promotion |
| Admission/runtime | privileged Pod, hostPath, policy deny | policy bypass or privileged DaemonSet |
| Network/gateway | authn/authz, WAF, egress, route | unauthorized inference, unusual outbound transfer |
| Secrets/TLS | rotation, decrypt, cert expiry | unexpected secret access, failed rotation |
| GPU node | driver/runtime drift, new privileged plugin | unapproved device/plugin change |

Do not equate a single 401 or denied request with a confirmed attack; correlate context, rates, identity, change windows and asset criticality.

## 4. Safe discovery and baseline [SAFE-READ]

```bash
kubectl config current-context
kubectl get events -A --sort-by=.lastTimestamp | tail -60
kubectl get rolebinding,clusterrolebinding -A
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations
kubectl get pods -A -o wide
kubectl get networkpolicy -A
kubectl get deployments,daemonsets -A
```

Identify actual TrueFoundry agent namespace and logs; use bounded, redacted log queries:
```bash
kubectl -n AGENT_NAMESPACE logs deploy/AGENT_DEPLOYMENT --since=15m --tail=100
```
**Review log content before sharing.** For Kubernetes audit logs, verify whether the managed provider exposes them and how to access them through its supported logging service. Do not assume `kubectl logs kube-apiserver` works on EKS or AKS.

## 5. Audit event model and evidence quality

Normalize timestamps (UTC), principal identity/type, source IP or network origin where lawful, action/verb, API group/resource, namespace/workspace, target, decision, request ID, session/trace correlation, cloud account/subscription, artifact digest, deployment revision and policy version.

Preserve raw evidence with access control, retention, integrity checks and chain-of-custody metadata. Redact tokens, request bodies, model inputs/outputs and personal data. Hashing or redaction must not silently destroy facts needed for an authorized investigation; follow legal/security evidence policy.

**Time:** configure reliable NTP; inconsistent timestamps can create false incident timelines. **Sampling:** do not sample away security-critical audit events without risk approval. **Storage:** separate detection pipelines from tamper-resistant evidence storage.

## 6. Detection engineering and alert tuning

Define a detection as: hypothesis, data sources, rule, threshold, suppression, enrichment, owner, severity, runbook, expected false positives and acceptance test. Examples:

1. ClusterRoleBinding created to cluster-admin outside approved change.
2. New `get/list/watch secrets` permission granted to unexpected principal.
3. ServiceAccount token request from unrecognized automation.
4. Repeated denied inference attempts across multiple tenants or routes.
5. Successful inference on a route expected to require authentication.
6. Cloud role trust broadened to wildcard principal/subject.
7. Artifact digest differs from approved deployment manifest.
8. Privileged DaemonSet or hostPath workload appears on GPU node.
9. Secret access after employee disablement or machine identity revocation.
10. SIEM source stops sending expected heartbeat events.

Use environment-specific baselines and meaningful severity. Alerts should not include secret values, JWTs or raw prompts.

## 7. Incident severity and command

Severity is organization-defined. Illustrative model:
- **SEV-1 security:** confirmed active cross-tenant exposure, broad credential compromise or ongoing exfiltration.
- **SEV-2:** suspicious privileged change or compromised workload with constrained scope.
- **SEV-3:** blocked attack, non-exploited policy gap or degraded audit source.

Assign incident commander, security lead, platform/SRE lead, communications owner, evidence custodian and change approver. Record declaration time, scope, containment authority, customer/privacy/legal escalation thresholds and decision log. If data breach notification rules may apply, involve counsel/privacy teams; do not improvise external statements.

## 8. Investigation workflow

1. **Triage:** establish alert fidelity, affected identity/resource and current exposure.
2. **Scope:** identify impacted clusters, namespaces, projects, models, tokens, CI pipelines, artifacts and cloud accounts.
3. **Preserve:** export authorized, redacted logs and configuration snapshots to controlled evidence store; capture digests and timestamps.
4. **Contain:** disable compromised identity, restrict route/egress, quarantine artifact or pause promotion under incident authority. Avoid destroying forensic state unnecessarily.
5. **Eradicate:** fix root cause, rotate affected credentials, remove unauthorized bindings/artifacts and patch vulnerability.
6. **Recover:** redeploy reviewed immutable image/model digests, reconcile GitOps and validate allowed/denied paths.
7. **Monitor:** enhanced watch for recurrence and new indicators.
8. **Review:** timeline, impact, detection gap, corrective actions, owners, deadlines and control acceptance.

**Compromised credentials:** do not roll back to them for service continuity. Security containment can require downtime.

## 9. Twelve security incident response runbooks

### RB-01 — Suspicious TrueFoundry administrative login
Correlate IdP MFA, device/location context, platform role changes and deployment audit. Revoke affected sessions and rotate credentials as supported; preserve timeline.

### RB-02 — Unauthorized production deployment
Identify user/CI identity, Git commit, image/model digests, affected workspaces and rollout. Pause promotion, revert reviewed desired state and verify runtime.

### RB-03 — Unexpected cluster-admin binding
Inspect binding, creator, audit, related Pod changes and Secrets access. Revoke under incident approval; confirm other additive bindings do not preserve privilege.

### RB-04 — Kubernetes Secret read anomaly
Correlate principal, verb, namespace and workload identity. Treat credential exposure as possible even if read logs omit values; rotate based on scope.

### RB-05 — Compromised CI/CD identity
Revoke tokens/federation trust, pause pipelines, verify artifact provenance and scan recent deployments. Resume only after least-privilege and signing checks.

### RB-06 — Model artifact poisoning suspicion
Quarantine model version, preserve checksum and storage audit, restore approved digest, investigate supply chain and validate inference behavior.

### RB-07 — Anonymous or cross-tenant inference success
Contain endpoint/route, identify data exposure and request IDs, restore authorization and test alternate paths. Engage security/privacy as required.

### RB-08 — Unexpected external egress
Correlate CNI/cloud flow, DNS, Pod identity and storage access; isolate affected workload while preserving forensic evidence.

### RB-09 — Privileged GPU node workload
Inspect driver/plugin DaemonSets, host mounts, image digests, audit and node access; quarantine unauthorized component with platform owner.

### RB-10 — Secret leak into logs or Git
Revoke/rotate, restrict evidence, inspect access history and remove copies per incident policy. Git history rewrite alone does not revoke a secret.

### RB-11 — Audit pipeline failure
Check source enablement, collector health, IAM, retention, backpressure and SIEM ingest. Restore audit coverage and assess the unobserved window.

### RB-12 — GitOps reintroduces insecure configuration
Identify authoritative desired state, malicious/unreviewed commit and reconciler. Correct Git, reconcile, validate policy and watch for reappearance.

## 10. Operational metrics and recovery acceptance

Measure audit source availability and ingest lag, privileged-change detection latency, time-to-acknowledge, containment time, credential revocation time, evidence completeness, denied-action coverage, recurrence and remediation backlog. Define organization-specific SLOs after baselining; do not invent vendor guarantees.

Incident closure requires evidence of scope, owner-approved containment, revoked affected credentials, restored authorized access, denied unauthorized access, verified artifact digests, GitOps convergence, telemetry restoration and incident review actions. Separate operational recovery from legal/security closure.

## 11. Production readiness checklist

Verify IdP/platform/Kubernetes/cloud/gateway/CI/registry/admission logs, secure retention, redaction, correlation IDs, alerts with owners, incident pager, break-glass procedure, immutable recovery artifacts, tabletop schedule, forensic access and tested rollback. Missing evidence is a gap, not a pass.

Complete the [companion lab](../labs/06-security-observability-audit-incident-response-check.md), including **32 security scenarios and 12 acceptance gates**. Publication does not imply live lab execution or formal vendor/security review.

**Next:** Chapter 10.7 — Production Security Hardening and Recovery Capstone.
