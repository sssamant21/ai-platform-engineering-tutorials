# Lab 9.2 — Kubernetes Services, EndpointSlices and Service Discovery Check

**Canonical:** [Chapter 9.2](../02-kubernetes-services-endpointslices-service-discovery/02-kubernetes-services-endpointslices-service-discovery.md)  
**Default:** [SAFE-READ]. Isolated staging mutation requires [LAB-MUTATE] approval.

## 1. Mission

Verify a model-serving Service's exposure, DNS, selector, port/targetPort, EndpointSlice readiness and streaming behavior. Trace the first failed transition when traffic does not reach a Ready model Pod. Never expose production via NodePort or external LB for a test.

## 2. Preflight

```text
Date/time / owner / approval:
TrueFoundry workspace / Kubernetes context / namespace:
Model deployment and immutable image/model revision:
Service name, expected DNS and protocol:
Gateway/LB owner, authorized client:
Readiness/termination and streaming SLO:
Known-good rollback and evidence location:
```

## 3. Read-only discovery

```bash
kubectl config current-context
kubectl -n <namespace> get svc,deploy,pods -o wide
kubectl -n <namespace> describe svc <service>
kubectl -n <namespace> get endpointslices -l kubernetes.io/service-name=<service> -o yaml
kubectl -n <namespace> get networkpolicy,pdb
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> describe pod <inference-pod>
```

## 4. Service mapping worksheet

| Property | Expected | Observed | Evidence |
|---|---|---|---|
| Type (ClusterIP/NodePort/LB/etc.) | | | |
| Service DNS name | | | |
| Selector and Pod labels | | | |
| Service port/protocol | | | |
| targetPort and listener | | | |
| EndpointSlice addresses | | | |
| Endpoint ready/serving/terminating | | | |
| Pod model-aware readiness | | | |
| Gateway backend mapping | | | |
| CNI/NetworkPolicy and cloud rules | | | |

## 5. DNS verification

From an authorized **existing** diagnostic client Pod when utilities are present:

```bash
nslookup <service>.<namespace>.svc
getent hosts <service>.<namespace>.svc
```

Record the cluster DNS suffix, namespace search path and headless/non-headless behavior. A successful lookup is not a successful inference request.

## 6. Streaming and termination checklist

| Test | Expected | Observed |
|---|---|---|
| Ready Pod included in EndpointSlice | | |
| Not Ready Pod excluded as configured | | |
| Terminating endpoint conditions | | |
| No new streams to draining Pod | | |
| Existing stream completion | | |
| Gateway idle/upstream timeouts | | |
| Pod termination grace | | |
| PDB behavior | | |

## 7. Optional isolated staging test [LAB-MUTATE]

1. Confirm namespace ownership, approved synthetic payload and resource limits.
2. Apply the illustrative Service manifest from Chapter 9.2 **only** after validating selectors, ports and namespace.
3. Observe selector-to-EndpointSlice reconciliation and Pod readiness.
4. Compare authorized internal and gateway paths without bypassing auth.
5. If separately approved, perform a bounded lab-only rolling update and verify complete streams.
6. Stop on unexpected exposure, wrong model routing, failed TLS/auth or SLO impact.
7. Restore the authoritative TrueFoundry/GitOps release and clean up lab-owned resources.

Mark Not run if mutation is not approved.

## 8. Failure tabletop

| ID | Scenario | LPHL | FFT | Evidence | Recovery |
|---|---|---|---|---|---|
| S1 | DNS NXDOMAIN | | | | |
| S2 | Wrong namespace search | | | | |
| S3 | Selector mismatch | | | | |
| S4 | No EndpointSlices | | | | |
| S5 | Not Ready endpoints | | | | |
| S6 | Named targetPort error | | | | |
| S7 | Numeric targetPort error | | | | |
| S8 | Loopback-only listener | | | | |
| S9 | NetworkPolicy deny | | | | |
| S10 | CNI route failure | | | | |
| S11 | Node-specific failure | | | | |
| S12 | Proxy/eBPF stale state | | | | |
| S13 | Headless DNS cache | | | | |
| S14 | ExternalName TLS | | | | |
| S15 | Internal LB wrong scheme | | | | |
| S16 | Public exposure | | | | |
| S17 | Gateway backend port | | | | |
| S18 | Model cold start | | | | |
| S19 | Terminating endpoint | | | | |
| S20 | No Ready during rollout | | | | |
| S21 | Truncated stream | | | | |
| S22 | PDB prevents drain | | | | |
| S23 | Dual-stack mismatch | | | | |
| S24 | Wrong model revision | | | | |

LPHL = Lowest Proven Healthy Layer; FFT = First Failed Transition.

## 9. Acceptance gates

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| A1 Exposure/type | | |
| A2 Selector/ports | | |
| A3 EndpointSlice conditions | | |
| A4 DNS discovery | | |
| A5 Dataplane/policy | | |
| A6 Stream drain/readiness | | |
| A7 S1–S24 LPHL/FFT | | |
| A8 Staging/rollback | | |

## 10. Final report

```text
Date/time / owner / approval:
Service type/DNS/selector/ports:
EndpointSlice and model readiness:
Dataplane/NetworkPolicy/LB:
Streaming/drain and SLO:
S1–S24 LPHL/FFT:
A1–A8, rollback, unexecuted gates:
```

**Next:** Chapter 9.3 — Ingress, Gateway API and Cloud Load Balancers.
