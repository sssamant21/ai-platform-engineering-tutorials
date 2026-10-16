# Chapter 13.2 — TrueFoundry Agent and Control Plane Troubleshooting

**Part 13 — Troubleshooting | TrueFoundry SRE canonical tutorial**

## Objectives and scope

Diagnose TrueFoundry agent connectivity, cluster registration, authentication, Kubernetes RBAC, reconciliation, platform API reachability, proxy/TLS, deployment state and control-plane versus inference data-plane impact. Build a repeatable, safe diagnostic flow for production incidents.

**[SAFE-READ]** inspect approved namespaces, logs, Events, deployment status and connectivity. **[LAB-MUTATE]** isolated staging. **[CHANGE-CONTROL]** production agent restart/reinstall, credentials, RBAC, network policy, proxy, cluster registration and manual resource edits. Never print tokens, kubeconfigs, secret values, patient data or model prompts in shared logs.

## 1. Understand the failure boundary

```text
TrueFoundry hosted/self-managed control plane (deployment-dependent)
        │ authenticated outbound/inbound connectivity
        ▼
Kubernetes tfy-agent / platform integration (actual name/version varies)
        │ Kubernetes API, service accounts and RBAC
        ▼
Desired state / workload reconciliation
        │
        ├── GPU inference Pods (vLLM/model runtime)
        ├── Services, routing, ingress
        ├── secrets and workload identity references
        └── observability / deployment status
```

Control-plane unavailability can impair new deployments, reconciliation or UI status while already-running inference may continue; **do not assume** either continued service or immediate outage. Verify the specific installation topology, feature dependencies and live data-plane health.

## 2. Agent discovery and baseline

Discover rather than hardcode namespaces and names:

```bash
kubectl config current-context
kubectl get namespaces
kubectl get deployments,daemonsets,pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
kubectl get serviceaccounts -A
```

Identify the installed agent workload from approved platform inventory or documented labels. `tfy-agent` is a common conceptual label, not a guarantee of a Kubernetes Deployment name.

After discovery, set placeholders to real authorized values:

```bash
kubectl -n AGENT_NAMESPACE get deploy AGENT_DEPLOYMENT -o wide
kubectl -n AGENT_NAMESPACE describe deploy AGENT_DEPLOYMENT
kubectl -n AGENT_NAMESPACE get pods -o wide
kubectl -n AGENT_NAMESPACE logs deploy/AGENT_DEPLOYMENT --all-containers=true --since=30m --tail=200
```

`kubectl logs deploy/...` may select a single Pod, not aggregate all replicas. Use per-Pod logs when correlation is needed. Redact log content before attaching to incident records.

## 3. Connectivity diagnostic chain

```text
Agent ready?
 ├─ NO → Pod scheduling, image pull, probes, crash, resource limits
 └─ YES → DNS/TLS/proxy/egress to configured platform endpoint?
           ├─ NO → network policy, NAT/firewall, DNS, certificate, proxy
           └─ YES → authentication/authorization accepted?
                     ├─ NO → identity, token rotation, service account/RBAC
                     └─ YES → reconciliation healthy?
                               ├─ NO → API errors, drift, CRD/schema/version
                               └─ YES → check independent inference path
```

Determine whether the connection is outbound, inbound or bidirectional for the installed TrueFoundry version. Do not assume a universal endpoint, port, protocol or API path. Use vendor-documented connectivity checks and sanitized DNS/TLS diagnostics from an approved diagnostic Pod.

## 4. Kubernetes RBAC and identity

Inspect service account and authorization without exposing tokens:

```bash
kubectl -n AGENT_NAMESPACE get serviceaccount AGENT_SA
kubectl auth can-i get pods -n TARGET_NAMESPACE --as=system:serviceaccount:AGENT_NAMESPACE:AGENT_SA
kubectl auth can-i list deployments -n TARGET_NAMESPACE --as=system:serviceaccount:AGENT_NAMESPACE:AGENT_SA
kubectl -n AGENT_NAMESPACE describe pod AGENT_POD
```

Impersonation checks require appropriate Kubernetes privileges and may be denied; do not broaden RBAC merely to make the command pass. Compare actual required permissions with the installed version's documented role. Check cloud workload identity trust and expiry independently of Kubernetes RBAC.

## 5. Reconciliation and drift

Classify: (a) desired state never submitted; (b) accepted but not applied; (c) applied but Pod not healthy; (d) Pod healthy but platform status stale; (e) manual changes overwritten. Correlate timestamps and immutable deployment revision. A successful API response does not prove reconciliation, and a healthy agent Pod does not prove its upstream connection.

Do not manually patch TrueFoundry-managed resources in production without understanding ownership: the controller may revert the patch. Fix desired state through the authoritative platform workflow.

## 6. Certificates, proxy and network

Check DNS resolution, egress policy, cloud NAT, firewall, proxy `NO_PROXY` for cluster endpoints, certificate validity and clock skew. Use approved tools and never run `curl -k` as a permanent TLS workaround. Check whether an outage is cluster-wide, zone-specific or limited to one node.

## 7. Control-plane versus data-plane validation

During platform degradation, independently test an authorized synthetic inference request through the real customer path. Record HTTP success, streaming completion, TTFT/TPOT and errors. Also check new deployment/reconciliation separately. This prevents treating a stale UI as proof of an inference outage or assuming a healthy Pod means customers are unaffected.

## 8. Upgrade and recovery decision

Before agent restart, reinstallation or version rollback: identify current version, release notes, configuration backup, compatibility with Kubernetes and TrueFoundry, credential/identity dependencies, disruption impact and change approval. Prefer supported vendor procedure; do not delete agent namespace or secrets as a troubleshooting shortcut. Restore and verify both reconciliation and customer-serving data plane.

## 9. Twelve production runbooks

1. Agent Pod Pending — scheduler, quota, node selectors, Events.
2. Agent CrashLoopBackOff — previous logs, probes, OOM, config.
3. Agent ImagePullBackOff — registry, digest, egress, auth.
4. Agent platform API unreachable — DNS, proxy, firewall, TLS.
5. TLS trust/certificate failure — trust bundle, expiry, clock skew.
6. Agent identity rejected — token/workload identity rotation and scope.
7. Kubernetes RBAC denied — service account, least-privilege bindings.
8. Reconciliation stalled — API errors, queue, version/schema drift.
9. Manual resource changes reverted — desired-state ownership.
10. Cluster appears disconnected — agent heartbeat, API and synthetic inference.
11. Upgrade incompatibility — version matrix, staged rollback.
12. Control-plane incident recovery — independently verify deployments and inference.

## 10. Thirty-two failure scenarios

| # | Scenario | # | Scenario |
|---|---|---|---|
| 01 | Agent Pod Pending | 17 | Cloud workload identity denied |
| 02 | Agent CrashLoopBackOff | 18 | Service account missing |
| 03 | Agent ImagePullBackOff | 19 | RBAC forbidden |
| 04 | Agent OOMKilled | 20 | CRD schema mismatch |
| 05 | Agent readiness failure | 21 | Reconciliation queue stalled |
| 06 | Agent liveness loop | 22 | Desired state not submitted |
| 07 | Cluster DNS outage | 23 | Status stale in UI |
| 08 | Platform DNS failure | 24 | Manual patch reverted |
| 09 | Proxy misconfigured | 25 | Namespace policy blocks API |
| 10 | NO_PROXY incomplete | 26 | Agent version drift |
| 11 | Firewall egress blocked | 27 | Kubernetes API throttling |
| 12 | NAT exhausted | 28 | Agent cannot list resources |
| 13 | TLS cert expired | 29 | Cluster registration stale |
| 14 | Trust chain invalid | 30 | New deploy blocked |
| 15 | Clock skew | 31 | Existing inference degraded |
| 16 | Token expired | 32 | Recovery verified only in UI |

## 11. Twelve acceptance gates

G1 scope and approval; G2 agent inventory/version; G3 Pod/Events evidence; G4 network/DNS/TLS checks; G5 identity and RBAC review; G6 reconciliation trace; G7 version/CRD compatibility; G8 independent data-plane SLO; G9 sanitized logs; G10 approved mitigation and rollback; G11 post-recovery verification; G12 SRE/platform review.

Complete the [companion lab](../labs/02-truefoundry-agent-control-plane-troubleshooting-check.md). Formal review and live execution pending.

**Next:** Chapter 13.3 — Kubernetes Scheduling, Pending Pods and GPU Capacity Failures.
