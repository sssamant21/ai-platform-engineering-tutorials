# Lab 9.1 — TrueFoundry Networking Architecture and Request Path Check

**Canonical:** [Chapter 9.1](../01-truefoundry-networking-architecture-request-path/01-truefoundry-networking-architecture-request-path.md)  
**Default:** [SAFE-READ]. Changes require [LAB-MUTATE] in approved isolated staging.

## 1. Mission and safety

Trace an authorized model endpoint from client DNS to Ready inference Pod and streaming completion. Separate TrueFoundry management integration from runtime traffic. No production DNS edits, TLS bypass, privileged diagnostic Pods, firewall changes or customer payloads.

## 2. Preflight

```text
Date/time / owner / approval:
Workspace/cluster/namespace and endpoint:
Expected management and runtime paths:
Gateway/Ingress controller, LB and DNS owner:
Service/Pod and immutable model revision:
TLS/SNI/certificate and auth owner:
Known-good rollback and test limits:
```

## 3. Read-only Kubernetes inventory

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods -o wide
kubectl -n <namespace> get svc,ingress -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get networkpolicy
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> describe svc <service>
kubectl -n <namespace> describe pod <pod>
```

Inspect Gateway API resources only when corresponding CRDs exist. Record Not installed where absent.

## 4. End-to-end route map

| Hop | Actual component/address (redact) | Owner | Healthy evidence | Failure signal |
|---|---|---|---|---|
| Client and DNS | | | | |
| Cloud LB | | | | |
| Gateway/Ingress | | | | |
| TLS/auth | | | | |
| Service port/targetPort | | | | |
| EndpointSlice Ready | | | | |
| CNI/NetworkPolicy | | | | |
| Pod listener | | | | |
| vLLM streaming | | | | |
| tfy-agent management path | | | | |

## 5. DNS and TLS validation [SAFE-READ]

From an authorized workstation with a non-sensitive approved host:

```bash
nslookup <approved-host>
curl --head --connect-timeout 5 --max-time 15 https://<approved-host>/
openssl s_client -connect <approved-host>:443 -servername <approved-host> -showcerts
```

A HEAD request may return 405 if the endpoint only supports POST; that is not proof of routing failure. Do not disable TLS verification. Avoid posting tokens or private addresses in reports.

## 6. Service and EndpointSlice mapping

| Check | Expected | Observed | Evidence |
|---|---|---|---|
| Service selector matches Pods | | | |
| Service port and targetPort | | | |
| EndpointSlice Ready count | | | |
| Pod container listener | | | |
| Model-aware readiness | | | |
| Gateway backend reference | | | |
| Gateway upstream protocol/TLS | | | |
| Streaming idle timeout | | | |
| Scale-down drain behavior | | | |

## 7. Optional approved staging probe [LAB-MUTATE]

1. Confirm isolated endpoint, synthetic payload and rollback.
2. Record baseline TLS validity, status code, latency and stream completion.
3. Use an approved existing diagnostic client to compare external route and internal Service reachability.
4. Record DNS, gateway, Service, EndpointSlice and Pod-level evidence without bypassing auth.
5. If approved, perform a bounded lab-only rollout and observe EndpointSlice readiness and stream drain.
6. Stop on unintended external exposure, auth bypass, customer impact or SLO breach.
7. Restore authoritative routing configuration and verify correct model/streaming.

If staging changes are not approved, perform a tabletop and mark the live gate Not run.

## 8. Failure tabletop

| ID | Scenario | LPHL | FFT | Evidence | Safe response |
|---|---|---|---|---|---|
| N1 | NXDOMAIN | | | | |
| N2 | Wrong private/public DNS | | | | |
| N3 | TCP timeout | | | | |
| N4 | TLS SAN mismatch | | | | |
| N5 | Expired chain | | | | |
| N6 | 401/403 | | | | |
| N7 | 404 route mismatch | | | | |
| N8 | No Ready endpoints | | | | |
| N9 | Wrong targetPort | | | | |
| N10 | Cold-model 504 | | | | |
| N11 | NetworkPolicy deny | | | | |
| N12 | Cloud firewall deny | | | | |
| N13 | Artifact egress failure | | | | |
| N14 | DNS egress failure | | | | |
| N15 | Truncated stream | | | | |
| N16 | Gateway buffering | | | | |
| N17 | Drain interrupts stream | | | | |
| N18 | LB target unhealthy | | | | |
| N19 | Service selector mismatch | | | | |
| N20 | Gateway controller absent | | | | |
| N21 | CNI IP exhaustion | | | | |
| N22 | MTU issue | | | | |
| N23 | Upstream TLS mismatch | | | | |
| N24 | tfy-agent control path down | | | | |

LPHL = Lowest Proven Healthy Layer; FFT = First Failed Transition. A management-path fault may coexist with a healthy inference path.

## 9. Acceptance gates

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| A1 Management/runtime paths separated | | |
| A2 DNS/LB/TLS/gateway owner verified | | |
| A3 Service/EndpointSlice/Pod mapping | | |
| A4 CNI/policy/firewall/egress | | |
| A5 Streaming timeout and drain | | |
| A6 N1–N24 LPHL/FFT | | |
| A7 Approved staging probe | | |
| A8 Rollback/model/SLO verification | | |

## 10. Final report

```text
Date/time / owner / approval:
Management path and runtime path:
DNS/LB/Gateway/TLS route inventory:
Service/EndpointSlice/Pod and CNI mapping:
Auth, NetworkPolicy, firewall and egress:
Streaming and drain behavior:
N1–N24 LPHL/FFT and remediation:
A1–A8, rollback, unexecuted gates:
```

**Next:** Chapter 9.2 — Kubernetes Services, EndpointSlices and Service Discovery.
