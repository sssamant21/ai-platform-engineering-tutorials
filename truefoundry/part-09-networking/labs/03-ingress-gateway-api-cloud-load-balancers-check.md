# Lab 9.3 — Ingress, Gateway API and Cloud Load Balancers Check

**Canonical:** [Chapter 9.3](../03-ingress-gateway-api-cloud-load-balancers/03-ingress-gateway-api-cloud-load-balancers.md)  
**Default:** [SAFE-READ]. [LAB-MUTATE] requires isolated staging approval. Production changes require [CHANGE-CONTROL].

## 1. Mission
Prove the authorized inference endpoint routes securely through DNS, cloud LB, ingress/Gateway, Service, EndpointSlice and model-ready Pod; localize the first failing boundary, verify long-lived streaming and controlled rollback.

## 2. Preflight worksheet
```text
Owner / approval / UTC time:
Kubernetes context / namespace / TrueFoundry workspace:
Cloud account/subscription / region / approved resource group:
IngressClass or GatewayClass and controller:
Endpoint DNS / expected private-public classification:
TLS/SNI and auth enforcement:
Backend Service / port / model revision:
Baseline SLO / stream duration / health path:
GitOps source / rollback revision / evidence storage:
```

## 3. Read-only discovery [SAFE-READ]
```bash
kubectl config current-context
kubectl api-resources | grep -E 'gateway|httproute|ingress'
kubectl get ingressclass,gatewayclass
kubectl -n <namespace> get ingress,svc,pods,endpointslices -o wide
kubectl -n <namespace> get gateway,httproute -o yaml
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> describe svc <backend-service>
kubectl -n <namespace> get endpointslices -l kubernetes.io/service-name=<backend-service> -o yaml
```
Skip Gateway commands when CRDs are not installed. Inspect controller logs only with authorized access and redact credentials.

## 4. Evidence worksheet
| Layer | Expected | Observed | LPHL / FFT evidence |
|---|---|---|---|
| DNS and IP scope | | | |
| Cloud LB listener/targets | | | |
| TLS certificate and SNI | | | |
| Auth/WAF boundary | | | |
| Ingress or Gateway/HTTPRoute | | | |
| Backend Service/port | | | |
| EndpointSlice readiness | | | |
| Model readiness and revision | | | |
| SSE stream completion/drain | | | |

## 5. Authorized synthetic probes
```bash
# Do not paste tokens into terminal history or shared evidence.
curl -sS -D - -o /dev/null https://<approved-host>/health
curl -v --connect-timeout 5 --max-time 20 https://<approved-host>/<approved-test-path>
openssl s_client -connect <approved-host>:443 -servername <approved-host> </dev/null
```
Only use approved synthetic inference inputs, approved authentication handling and test accounts. HTTP health may be intentionally protected; a 401 does not by itself imply failure. For SSE, use `curl -N` with a bounded approved synthetic request, measure first token, inter-token gaps and completion; do not store sensitive output.

## 6. Controller and cloud checks
For EKS, inspect the authorized ALB/NLB listener, target groups, target health, SG, subnet and idle/deregistration settings. For AKS, inspect the deployed Application Gateway/NGINX/other controller, frontend exposure, NSG, backend health and timeout settings. Record which controller owns each resource; never guess from a Kubernetes object alone.

## 7. Isolated staging mutation [LAB-MUTATE]
1. Obtain written scope and namespace ownership; ensure test DNS, certificates and private exposure.
2. Verify the approved Ingress or Gateway API controller supports required APIs.
3. Apply **one** of the Chapter 9.3 illustrative manifests only after replacing placeholders and reviewing the generated diff.
4. Confirm Accepted/Programmed/ResolvedRefs, TLS, route matching and target health.
5. Test authorized healthy and unauthorized requests; verify no direct-backend bypass.
6. Execute a bounded long-running SSE request and, if separately approved, a rolling termination/drain exercise.
7. Restore the GitOps/TrueFoundry authoritative revision and delete only lab-owned resources.
8. Stop immediately on public exposure, failed auth, wrong model routing, SLO regression or unexpected cloud charges.

## 8. Troubleshooting tabletop — 24 scenarios
| ID | Scenario | LPHL | FFT | Evidence | Recovery |
|---|---|---|---|---|---|
| S1 | DNS NXDOMAIN | | | | |
| S2 | Stale DNS target | | | | |
| S3 | LB listener absent | | | | |
| S4 | Public instead of private LB | | | | |
| S5 | Wrong ingressClass | | | | |
| S6 | Ingress controller missing | | | | |
| S7 | GatewayClass unsupported | | | | |
| S8 | Gateway not Programmed | | | | |
| S9 | HTTPRoute not Accepted | | | | |
| S10 | ResolvedRefs false | | | | |
| S11 | Cross-namespace reference denied | | | | |
| S12 | Host mismatch 404 | | | | |
| S13 | Path rewrite error | | | | |
| S14 | Backend port mismatch | | | | |
| S15 | No Ready EndpointSlices | | | | |
| S16 | LB health probe mismatch | | | | |
| S17 | Target group unhealthy | | | | |
| S18 | AWS security group deny | | | | |
| S19 | Azure NSG/UDR deny | | | | |
| S20 | TLS expired or SNI mismatch | | | | |
| S21 | Missing client authentication | | | | |
| S22 | WAF 403 | | | | |
| S23 | Cold-start 504 | | | | |
| S24 | LB idle timeout resets SSE | | | | |

For each scenario, state whether it was executed, simulated or discussed. LPHL = Lowest Proven Healthy Layer; FFT = First Failed Transition.

## 9. Acceptance gates
| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| A1 Authorized DNS and LB exposure | | |
| A2 Listener and TLS/SNI validation | | |
| A3 Auth, WAF and no bypass | | |
| A4 Ingress/Gateway route acceptance | | |
| A5 Backend Service/EndpointSlice readiness | | |
| A6 Cloud target health EKS/AKS as applicable | | |
| A7 SSE/stream drain and timeout behavior | | |
| A8 S1–S24 LPHL/FFT completed | | |
| A9 GitOps rollback and SLO recovery | | |

## 10. Final report
```text
Scope / approver / date:
Architecture/controller/cloud:
LPHL and FFT by failure:
TLS/auth/security:
SSE and target health:
S1-S24 tabletop completion:
A1-A9 pass/fail/not run:
Rollback revision and residual risks:
Vendor review: Pending / Complete
SRE review: Pending / Complete
Live execution: Not run / Completed with evidence
```

**Next:** Chapter 9.4 — DNS, TLS, Certificates and Secure Endpoint Exposure.
