# Chapter 14.2 — TrueFoundry Agent Health and Reconciliation Runbook

**Part 14 — SRE Runbooks | Production operations**

## Purpose and scope

Provide a repeatable on-call runbook to detect, diagnose and recover TrueFoundry agent disconnections, failed cluster registration, stalled reconciliation, authentication/authorization errors, agent rollout regressions and stale UI state. Establish whether customer-facing inference is affected separately from control-plane status.

**[SAFE-READ]** approved Kubernetes, TrueFoundry, cloud and synthetic health checks. **[LAB-MUTATE]** isolated staging. **[CHANGE-CONTROL]** production agent restarts/reinstalls, credentials, RBAC, firewall, network policy and platform configuration. Never print or attach tokens, kubeconfigs, secret values, customer prompts or proprietary model artifacts.

## 1. Dependency map and failure boundaries

```text
TrueFoundry control plane / cluster registration
         │ platform endpoint, DNS, TLS, proxy, egress
         ▼
tfy-agent or equivalent installed integration
         │ service account, Kubernetes API, RBAC
         ▼
Reconciliation: desired deployment → observed K8s resources
         │
         ├── Deployments / Pods / Services / routing
         └── inference data plane (can fail independently)
```

TrueFoundry installations and versions vary. Discover actual workload names, namespaces, connection direction, registration workflow and health signals from the approved inventory and vendor documentation. `tfy-agent` is a conceptual name, not a guarantee of a Kubernetes Deployment called `tfy-agent`.

## 2. Detection, SLIs and alert thresholds

| SLI | Source | Meaning | Alert guidance |
|---|---|---|---|
| Agent Pod availability | kube-state-metrics/Events | process and probes healthy | sustained unavailability |
| Agent connection freshness | platform/agent metric or logs | platform channel functioning | age beyond measured expected interval |
| Reconciliation age | platform status/reconcile logs | desired changes applied | sustained stale desired revision |
| Kubernetes API errors | agent sanitized logs | API/RBAC/rate limiting | persistent 401/403/429 or failures |
| Platform endpoint reachability | approved synthetic connectivity | DNS/TLS/egress path | corroborated failures |
| Customer inference SLO | independent synthetic probe | actual serving impact | availability/TTFT/stream completion |
| Observability freshness | scrape/log pipeline health | confidence in signals | absent/stale telemetry |

Do not invent universal heartbeat intervals or assume UI `Connected` implies inference success. Use measured normal cadence and vendor-supported signals. Monitor new deployments and already-serving models separately.

## 3. First five minutes: safe triage

1. Acknowledge alert; record UTC, cluster, region, incident ID, impacted deployment and owner.
2. Run an authorized synthetic inference request through the customer route and verify stream completion.
3. Compare TrueFoundry desired revision, Kubernetes observed revision and last known successful reconciliation.
4. Discover agent Pod/Deployment and inspect status, Events and sanitized recent logs.
5. Check platform endpoint DNS, TLS, proxy/egress and identity/RBAC.
6. Determine whether the issue is one agent, one cluster, one cloud region or shared control plane.
7. Establish severity, escalation, safe mitigation owner and next update.

## 4. Agent inventory and read-only commands

```bash
kubectl config current-context
kubectl get deployments,daemonsets,pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
kubectl get serviceaccounts -A
```

After discovery, use the real namespace and Deployment:

```bash
kubectl -n AGENT_NAMESPACE get deployment AGENT_DEPLOYMENT -o wide
kubectl -n AGENT_NAMESPACE describe deployment AGENT_DEPLOYMENT
kubectl -n AGENT_NAMESPACE get pods -o wide
kubectl -n AGENT_NAMESPACE logs deployment/AGENT_DEPLOYMENT --since=30m --tail=200
```

Deployment logs may select only one replica; inspect each Pod when necessary. Redact all logs before sharing. `kubectl describe pod` can reveal environment names and sensitive operational metadata.

## 5. Symptom-to-layer decision matrix

| Symptom | First check | Likely domain | Escalation |
|---|---|---|---|
| Agent Pending | Pod Events, node selectors, quota | Kubernetes scheduling | K8s SRE |
| CrashLoopBackOff | previous logs, probe, OOM | agent runtime/config | Platform SRE |
| ImagePullBackOff | registry/digest/network | registry/credentials | Platform + security |
| Agent Ready but disconnected | endpoint DNS/TLS/egress | network or identity | Network + platform |
| 401 from platform | approved credential status | authentication | Platform/security |
| 403 from Kubernetes API | service account/RBAC | authorization | K8s/security |
| 429/throttle | API rate, backoff | rate limit/reconciliation | Platform/vendor |
| UI stale but Pods healthy | revision/timestamp and probe | status propagation | Platform |
| New deploy stuck | desired/observed revision | reconciliation | Platform |
| Serving inference down | synthetic, gateway, Pod, GPU | data plane | Model SRE |

Do not infer root cause from a single HTTP code without identifying which API returned it.

## 6. Kubernetes service account and RBAC

```bash
kubectl -n AGENT_NAMESPACE get serviceaccount AGENT_SA
kubectl auth can-i get pods -n TARGET_NAMESPACE --as=system:serviceaccount:AGENT_NAMESPACE:AGENT_SA
kubectl auth can-i list deployments -n TARGET_NAMESPACE --as=system:serviceaccount:AGENT_NAMESPACE:AGENT_SA
```

Impersonation checks require permission and may fail independently of the target account's authorization. Validate against version-specific documented minimum roles. Never grant cluster-admin to resolve a generic 403. Inspect cloud workload identity trust and token rotation through approved nonsecret metadata.

## 7. DNS, TLS, proxy and egress

Map configured endpoint category and connection direction. Check resolver health from approved Pod network vantage point, certificate chain/SNI/expiry, proxy settings, NAT/firewall, network policy, cluster clock skew and cloud egress quota. Do not assume fixed TrueFoundry API hostname/port. Do not use `curl -k` as a fix or print `Authorization` headers.

## 8. Reconciliation and drift diagnosis

Classify:
1. Desired revision not submitted.
2. Submitted but not accepted by control plane.
3. Accepted but agent cannot apply (RBAC, API, schema).
4. Applied but Pod/image/model unhealthy.
5. Kubernetes healthy but TrueFoundry status delayed.
6. Manual patch reverted by authoritative reconciliation.

Correlate UTC timestamps, immutable image/model revision and observed Kubernetes generation. A healthy agent Pod is insufficient proof of a working reconciliation loop. Avoid out-of-band patches to TrueFoundry-managed resources.

## 9. Mitigation and rollback decision

Prioritize reversible changes: isolate scope, verify data plane, repair approved endpoint/identity/network configuration, then use supported vendor agent restart/upgrade/rollback only with approval. Before any restart/reinstall capture current version, manifests/config backup, ownership, availability impact, compatibility and rollback procedure. Never delete agent namespace or secrets as a shortcut. If the control plane is down but inference is healthy, protect serving capacity and avoid unnecessary restarts.

## 10. Recovery validation

Confirm all of:
- agent health and documented platform connection;
- successful reconciliation of a harmless staging change;
- desired revision equals observed Kubernetes state;
- no recurring RBAC/401/403/429/TLS errors;
- new deployment status and existing workload health;
- synthetic customer-path inference, TTFT/TPOT and streaming completion;
- telemetry freshness, alert re-arming, owner and incident closure.

## 11. Fourteen operational runbooks

1. Agent discovery and inventory.
2. Agent Pod Pending.
3. Agent CrashLoopBackOff.
4. Agent ImagePullBackOff.
5. Platform endpoint DNS failure.
6. TLS certificate or proxy/egress failure.
7. Agent identity expired/rejected.
8. Kubernetes RBAC forbidden.
9. Kubernetes API throttling.
10. Cluster registration stale.
11. Reconciliation backlog/drift.
12. UI status stale vs data plane.
13. Agent version/upgrade rollback.
14. Recovery validation and incident handoff.

Each runbook requires trigger, impact, safe commands, hypothesis, evidence, mitigation, rollback, customer-path SLO and owner.

## 12. Thirty-six failure scenarios

| # | Scenario | # | Scenario |
|---|---|---|---|
| 01 | Agent not installed | 19 | Kubernetes RBAC denied |
| 02 | Agent Pod Pending | 20 | API throttle |
| 03 | Agent CrashLoopBackOff | 21 | Reconcile backlog |
| 04 | Agent OOMKilled | 22 | CRD/version mismatch |
| 05 | Agent readiness failed | 23 | Desired revision absent |
| 06 | Agent liveness restart | 24 | Observed revision stale |
| 07 | Agent ImagePullBackOff | 25 | Manual patch reverted |
| 08 | Registry authentication | 26 | Cluster registration stale |
| 09 | Platform DNS failure | 27 | UI status delayed |
| 10 | Proxy misconfiguration | 28 | Multi-cluster control plane outage |
| 11 | NO_PROXY mismatch | 29 | Control plane degraded |
| 12 | NAT/firewall blocked | 30 | Data plane healthy despite agent down |
| 13 | NetworkPolicy egress | 31 | Inference unhealthy despite agent Ready |
| 14 | TLS expiry | 32 | Agent version drift |
| 15 | Untrusted CA | 33 | Upgrade regression |
| 16 | Clock skew | 34 | Rollback incompatibility |
| 17 | Platform credential expired | 35 | Metrics missing |
| 18 | Cloud workload identity denied | 36 | Recovery verified only in UI |

## 13. Twelve acceptance gates

G1 approved scope; G2 actual agent inventory; G3 Pod/Events/logs evidence; G4 platform DNS/TLS/egress; G5 identity/RBAC; G6 API throttling/version; G7 desired/observed revision; G8 independent inference SLO; G9 safe mitigation; G10 tested rollback plan; G11 end-to-end reconciliation and alert recovery; G12 reviewer sign-off.

Complete the [companion lab](../labs/02-truefoundry-agent-health-reconciliation-check.md). Formal review and live execution pending.

**Next:** Chapter 14.3 — GPU Node Health and Capacity Runbook.
