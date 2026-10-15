# Lab 5.5 — Services, Endpoints, Routing and Connectivity Check

**Canonical:** [Chapter 5.5](../05-services-endpoints-routing-connectivity/05-services-endpoints-routing-connectivity.md)  
**Default:** Production **[SAFE-READ]**. Optional deployment/route changes are **[LAB-MUTATE]** in a disposable nonproduction workspace.

## Objectives

Map a TrueFoundry endpoint to Kubernetes resources, verify Service-to-Pod readiness and port matching, test DNS/TLS from the correct client path, classify routing failures and produce a safe recovery plan.

## 0. Preflight

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services -o wide
```

Record workspace, deployment revision, intended exposure (internal/private/external), client network, namespace and authorized scope. Do not change production NetworkPolicy, ingress or DNS.

## 1. Service and endpoint inventory [SAFE-READ]

```bash
kubectl -n <namespace> describe service <service>
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get pods -l <label-selector> -o wide
kubectl -n <namespace> describe pod <pod>
```

Record selector, labels, Service port, targetPort, Pod readiness and EndpointSlice addresses/ports.

| Field | Observed |
|---|---|
| Service type | |
| Service selector | |
| Pod label match | |
| Service port → targetPort | |
| Ready Pod count | |
| EndpointSlice addresses | |
| Application listener | |

## 2. Discover routes and policies [SAFE-READ]

```bash
kubectl -n <namespace> get ingress
kubectl get gatewayclasses
kubectl -n <namespace> get gateways,httproutes
kubectl -n <namespace> get networkpolicies
```

Gateway commands require installed CRDs. Record **Not installed** where appropriate. Inspect actual controller and cloud/network policy ownership; do not assume absence of NetworkPolicy means unrestricted access.

## 3. Health and DNS tests [SAFE-READ with approved access]

From the affected client network:

```bash
nslookup <application-hostname>
curl --connect-timeout 5 --max-time 15 -i https://<application-hostname>/health
```

For internal Service isolation, where permitted:

```bash
kubectl -n <namespace> port-forward service/<service> 18080:<service-port>
```

In a second terminal:

```bash
curl -i http://127.0.0.1:18080/ready
curl -i http://127.0.0.1:18080/health
```

Port-forward is not a substitute for validating ingress, DNS, TLS or NetworkPolicy from a normal client. Redact authentication and internal details before sharing evidence.

## 4. Optional disposable service experiment [LAB-MUTATE]

In an approved nonproduction TrueFoundry workspace:
1. Deploy the known-good HTTP service from Chapter 5.1 using an immutable approved image.
2. Configure internal-only routing and verify Pod Ready.
3. Capture generated Service selector, ports and EndpointSlice addresses.
4. Verify health through Service port-forward and the supported TrueFoundry internal endpoint.
5. If permitted, deliberately misconfigure a **disposable** Service selector and observe no ready endpoints; immediately restore the authoritative TrueFoundry/GitOps configuration.
6. Confirm endpoint recovery and clean up only lab-owned resources.

Do not modify shared Services or create public routes.

## 5. Failure classification

| Case | Symptom | Evidence | First failed transition |
|---|---|---|---|
| N1 | DNS NXDOMAIN | DNS zone/resolver | Hostname → DNS |
| N2 | TLS certificate mismatch | SAN/SNI/cert metadata | TLS handshake |
| N3 | Gateway 404 | Route host/path match | Gateway → route |
| N4 | Gateway 502 | Backend port/protocol logs | Gateway → backend |
| N5 | Gateway 503 | Ready endpoints/route health | Route → backend availability |
| N6 | Service no endpoints | Selector/readiness | Service → Pod |
| N7 | Pod connection refused | Listener/port/logs | Pod network → process |
| N8 | NetworkPolicy blocks caller | CNI/policy/source identity | Source → destination |
| N9 | Internal success, external timeout | DNS/LB/firewall | External entrypoint |
| N10 | Intermittent errors on rollout | Endpoint churn/readiness | Endpoint lifecycle |

For each: identify lowest proven healthy layer, first failed transition, owner, safe remediation and rollback.

## 6. Recovery worksheet

```text
Date/time / cluster:
TrueFoundry workspace / deployment:
Client location and endpoint:
DNS result / TLS metadata:
Ingress/Gateway route:
Service selector / port / targetPort:
Ready EndpointSlice addresses:
Pod readiness and listener:
NetworkPolicy / cloud firewall:
Request status / latency / trace ID:
Lowest Proven Healthy Layer:
First Failed Transition:
Owner / approved change:
Rollback route/config revision:
Post-recovery end-to-end result:
Cleanup evidence:
```

## 7. Acceptance

- [ ] Correct Service selector, port and targetPort identified.
- [ ] Ready EndpointSlices and Pod readiness verified.
- [ ] DNS/TLS validated from the affected client path when accessible.
- [ ] Ingress/Gateway and NetworkPolicy scope identified.
- [ ] Ten routing failures classified without bypassing security.
- [ ] End-to-end health and rollback evidence recorded.
- [ ] No unapproved production changes; disposable resources cleaned up.

**Next:** Chapter 5.6 — Deployment Revisions, Rollouts, Rollbacks and Troubleshooting.
