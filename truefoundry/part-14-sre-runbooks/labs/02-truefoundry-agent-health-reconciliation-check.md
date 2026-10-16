# Chapter 14.2 — Hands-on Lab: TrueFoundry Agent Health and Reconciliation

**Part 14 — SRE Runbooks | Approved staging | Live execution pending**

## Objective and safety

Build an agent-health on-call checklist and validate platform connectivity, Kubernetes RBAC, desired/observed reconciliation and independent inference health. **[SAFE-READ]** inspect; **[LAB-MUTATE]** isolated staging; **[CHANGE-CONTROL]** production agent, identity, RBAC, proxy, firewall or upgrade changes. Do not expose secrets, kubeconfigs, tokens or customer prompts.

## Lab 0 — Discover the installation

```bash
kubectl config current-context
kubectl get deployments,daemonsets,pods -A -o wide
kubectl get serviceaccounts -A
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

Identify actual TrueFoundry integration, namespace, Deployment, service account, version, cluster registration, platform endpoint category, approved connectivity direction, desired/observed model revision and synthetic SLO.

## Lab 1 — Agent process and Events

```bash
kubectl -n AGENT_NAMESPACE describe deployment AGENT_DEPLOYMENT
kubectl -n AGENT_NAMESPACE get pods -o wide
kubectl -n AGENT_NAMESPACE logs deployment/AGENT_DEPLOYMENT --since=30m --tail=200
```

Inspect individual replicas if needed. Capture sanitized restart reason, probe failures, image pull and scheduling Events. Distinguish Pod readiness from actual platform connectivity.

## Lab 2 — Identity, RBAC and network

With approved impersonation permissions:

```bash
kubectl auth can-i get pods -n TARGET_NAMESPACE --as=system:serviceaccount:AGENT_NAMESPACE:AGENT_SA
kubectl auth can-i list deployments -n TARGET_NAMESPACE --as=system:serviceaccount:AGENT_NAMESPACE:AGENT_SA
```

Review least-privilege bindings, cloud workload identity, token expiry metadata, DNS/TLS/proxy/egress path. Do not create a production 403, rotate tokens or disable TLS during a lab.

## Lab 3 — Reconciliation versus serving

Compare TrueFoundry desired revision, Kubernetes observed Deployment/Pod revision, reconciliation timestamp and synthetic inference result. Verify that stale platform status and customer-serving failure are separate hypotheses.

## Lab 4 — Five game days

1. Agent Pod startup failure (staging or tabletop).
2. DNS/proxy/TLS platform endpoint degradation (tabletop unless isolated).
3. Service account/RBAC denial using disposable staging account.
4. Reconciliation drift or stale UI state on a test deployment.
5. Agent/control-plane outage with independently healthy inference and approved recovery.

For each: UTC start/end, expected/actual, customer SLO, sanitized evidence, mitigation, rollback, reviewer, executed/tabletop classification.

## Thirty-six scenarios

01 agent absent; 02 Pending; 03 CrashLoop; 04 OOMKilled; 05 readiness; 06 liveness; 07 ImagePull; 08 registry auth; 09 DNS; 10 proxy; 11 NO_PROXY; 12 NAT/firewall; 13 network policy; 14 TLS expiry; 15 CA trust; 16 clock skew; 17 platform credential; 18 cloud identity; 19 RBAC; 20 API throttle; 21 reconcile queue; 22 CRD mismatch; 23 desired absent; 24 observed stale; 25 manual drift; 26 registration; 27 UI stale; 28 multi-cluster outage; 29 control-plane degraded; 30 serving healthy; 31 serving unhealthy; 32 version drift; 33 upgrade; 34 rollback; 35 telemetry; 36 UI-only recovery.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Approved scope | Pending |
| G2 | Agent name/version/namespace identified | Pending |
| G3 | Pod, Events and logs reviewed | Pending |
| G4 | DNS/TLS/proxy/egress validated | Pending |
| G5 | Identity and least-privilege RBAC | Pending |
| G6 | API throttling and version checked | Pending |
| G7 | Desired vs observed revision | Pending |
| G8 | Independent synthetic inference SLO | Pending |
| G9 | Approved mitigation plan | Pending |
| G10 | Rollback documented | Pending |
| G11 | Reconciliation and alerts restored | Pending |
| G12 | Reviewer sign-off | Pending |

## Evidence template

```text
Exercise ID / approval / UTC start-end:
Cluster / namespace / agent version:
Platform endpoint category / connectivity:
Pod phase, restart reason, Events:
DNS/TLS/proxy/egress, identity, RBAC:
TrueFoundry desired revision / K8s observed:
Reconciliation timestamp / synthetic inference:
Hypothesis / expected / actual:
Mitigation / rollback / SLO:
Executed or tabletop:
Owner / reviewer / outstanding risk:
```

## Cleanup

Restore disposable staging identities and deployments, verify normal reconciliation and synthetic inference, confirm no unauthorized RBAC expansion and close test alerts. Publication does not imply lab execution.

**Next:** Chapter 14.3 — GPU Node Health and Capacity Runbook.
