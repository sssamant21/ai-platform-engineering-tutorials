# Chapter 13.2 — Hands-on Lab: TrueFoundry Agent and Control Plane Troubleshooting

**Part 13 | Approved staging | Live execution pending**

## Safety and purpose

Diagnose agent startup, platform connectivity, identity/RBAC, reconciliation and independent inference availability. [SAFE-READ] inspect; [LAB-MUTATE] approved staging; [CHANGE-CONTROL] production agent/identity/network changes. Never print secret values or disable TLS verification.

## Lab 0 — Discover the actual installation

```bash
kubectl config current-context
kubectl get namespaces
kubectl get deployments,pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

Identify the agent namespace/workload and service account from authoritative platform inventory; names vary by deployment. Record TrueFoundry version, Kubernetes version, cloud, agent image digest, platform endpoint category (not secret), SLO and approvals.

## Lab 1 — Agent startup triage

```bash
kubectl -n AGENT_NAMESPACE describe deployment AGENT_DEPLOYMENT
kubectl -n AGENT_NAMESPACE get pods -o wide
kubectl -n AGENT_NAMESPACE logs deploy/AGENT_DEPLOYMENT --since=30m --tail=200
```

Investigate Pending, image pull, probes, OOM and crash loop. Redact before saving.

## Lab 2 — Connectivity and identity

Map DNS, proxy, firewall, TLS, service account and cloud workload identity. Inspect only approved nonsecret configuration and `kubectl auth can-i` with proper impersonation authorization. Do not rotate or revoke real production tokens during this lab.

## Lab 3 — Reconciliation and data-plane distinction

Compare TrueFoundry desired deployment revision, Kubernetes observed revision and independent synthetic inference response. Demonstrate the distinction between stale UI/control-plane status and serving endpoint health. Label observations and hypotheses.

## Lab 4 — Five game days

1. Agent Pod startup failure (staging or tabletop).
2. Platform endpoint DNS/proxy/TLS outage (tabletop unless isolated).
3. Agent RBAC/identity denied (disposable service account only).
4. Reconciliation drift/stale status (isolated test workload).
5. Control-plane degradation with independent synthetic inference validation.

Each records UTC timeline, actual vs tabletop, scope, SLI, evidence, mitigation, rollback, owner and reviewer.

## Thirty-two scenarios

01 Pending; 02 CrashLoop; 03 ImagePull; 04 OOMKilled; 05 readiness; 06 liveness; 07 cluster DNS; 08 platform DNS; 09 proxy; 10 NO_PROXY; 11 firewall; 12 NAT; 13 TLS expiry; 14 trust chain; 15 clock skew; 16 token expiry; 17 cloud identity; 18 service account; 19 RBAC; 20 CRD mismatch; 21 reconcile queue; 22 desired state absent; 23 UI stale; 24 manual patch reverted; 25 namespace network policy; 26 agent version drift; 27 Kubernetes API throttle; 28 list forbidden; 29 registration stale; 30 new deploy blocked; 31 inference degraded; 32 UI-only recovery claim.

## Twelve acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Authorized scope | Pending |
| G2 | Agent name/version discovered | Pending |
| G3 | Startup/Events triage | Pending |
| G4 | DNS/proxy/TLS path checked | Pending |
| G5 | Identity and least-privilege RBAC reviewed | Pending |
| G6 | Reconciliation path traced | Pending |
| G7 | Version and CRD compatibility checked | Pending |
| G8 | Independent inference SLO validated | Pending |
| G9 | Logs redacted | Pending |
| G10 | Mitigation/rollback documented | Pending |
| G11 | Post-recovery state verified | Pending |
| G12 | Reviewer sign-off | Pending |

## Evidence and cleanup

```text
Exercise ID / UTC start-end:
Cluster, agent namespace and version:
Symptom and customer SLI:
TrueFoundry desired revision / observed Kubernetes revision:
Agent logs and Events (sanitized):
Network, TLS, identity and RBAC findings:
Independent inference probe result:
Hypothesis / test / observed:
Mitigation / rollback / recovery:
Executed or tabletop:
Owner / reviewer / open risk:
```

Remove only disposable staging resources, restore baseline and close test alerts. No execution or sign-off is claimed by publication.

**Next:** Chapter 13.3 — Kubernetes Scheduling, Pending Pods and GPU Capacity Failures.
