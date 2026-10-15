# 9.3 — Ingress, Gateway API and Cloud Load Balancers

**Part 09 — Networking | Chapter 3 of 7**  
**Audience:** TrueFoundry platform engineers, Kubernetes SREs, cloud networking teams, model-serving operators.  
**Execution classes:** [SAFE-READ] inspection; [LAB-MUTATE] approved isolated staging; [CHANGE-CONTROL] production.

## 1. Learning objectives and prerequisites
Trace public/private HTTP(S) traffic through DNS, cloud load balancer, controller, route, Service, EndpointSlice and inference Pod. Compare Kubernetes Ingress and Gateway API; identify cloud-specific controllers; validate TLS, health checks, auth boundaries and stream-safe shutdown. Prerequisites: Chapters 9.1–9.2, authorized Kubernetes/cloud read access, known TrueFoundry endpoint and rollback owner. **Discover the actual TrueFoundry ingress topology; do not assume a particular controller or managed gateway.**

## 2. Request path and failure domains
```text
Client / authorized API consumer
  -> DNS (public/private zone) -> cloud LB listener (public/internal)
  -> TLS boundary / WAF (placement varies)
  -> ingress controller OR Gateway listener
  -> Ingress rule OR HTTPRoute (host/path/method/header match)
  -> Kubernetes Service:port
  -> EndpointSlice -> Ready inference Pod:targetPort
  -> model runtime (vLLM/other) -> streamed response
```
The cloud LB, gateway proxy and application are distinct failure domains. A successful LB TCP health check does not prove an authenticated model inference works. Record whether TLS is terminated at the LB, re-encrypted to the gateway, or passed through; identify the component enforcing authentication and authorization.

## 3. Ingress vs Gateway API
| Property | Ingress | Gateway API |
|---|---|---|
| API objects | Ingress + IngressClass | GatewayClass, Gateway, HTTPRoute; optional ReferenceGrant |
| Controller | Ingress controller required | Compatible Gateway API controller required |
| Routing | HTTP host/path | Extended matches, filters, weighted backends where supported |
| Separation of duties | Often annotation-driven | Infrastructure owner Gateway, app owner Route |
| Cross-namespace backend | Controller-specific patterns | Requires explicit ReferenceGrant where applicable |
| TLS | Ingress TLS secret and controller-specific termination | Gateway listener certificateRefs, implementation-dependent |
| Portability | Annotation differences | Standardized API with support-level differences |

Gateway API CRDs alone do **not** implement routing. Verify installed controller, supported resource versions, GatewayClass acceptance and route conditions. Ingress and Gateway API may coexist, but avoid accidentally publishing the same hostname through two independent data planes.

## 4. Read-only inventory [SAFE-READ]
```bash
kubectl config current-context
kubectl get ingressclass,gatewayclass
kubectl -n <namespace> get ingress,svc,endpointslices -o wide
kubectl -n <namespace> get gateway,httproute -o wide
kubectl -n <namespace> describe ingress <ingress>
kubectl -n <namespace> describe gateway <gateway>
kubectl -n <namespace> describe httproute <route>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> get pods -o wide
```
Gateway API resource commands may fail if CRDs are absent; record as Not installed, not an outage. Use `kubectl api-resources | grep -E 'gateway|httproute'` first. Collect addresses, listeners, parentRefs, hostnames, backendRefs, route conditions and controller names; redact sensitive DNS/IP data.

## 5. Minimal Ingress example [LAB-MUTATE only]
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: inference-ingress
  namespace: tfy-network-lab
spec:
  ingressClassName: <approved-ingress-class>
  tls:
  - hosts: [inference.lab.example.com]
    secretName: inference-lab-tls
  rules:
  - host: inference.lab.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: llm-backend
            port:
              number: 80
```
Use an authorized DNS name, controller and certificate. Never apply placeholders. Ingress has no universal annotations for streaming timeouts or backend protocols; consult the installed controller's supported configuration.

## 6. Gateway API example [LAB-MUTATE only]
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: inference-gateway
  namespace: tfy-network-lab
spec:
  gatewayClassName: <approved-gateway-class>
  listeners:
  - name: https
    hostname: inference.lab.example.com
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: inference-lab-tls
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: inference-route
  namespace: tfy-network-lab
spec:
  parentRefs:
  - name: inference-gateway
    sectionName: https
  hostnames: [inference.lab.example.com]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: llm-backend
      port: 80
```
Confirm GatewayClass controller availability, `Accepted`, `Programmed` (Gateway), `ResolvedRefs` (HTTPRoute), listener attachment and TLS Secret ownership. A Gateway resource does not guarantee a cloud LB is provisioned. For cross-namespace Secret/backend references, apply the controller-supported ReferenceGrant and policy model.

## 7. AWS EKS implementation checklist
Identify whether AWS Load Balancer Controller, an ingress proxy with NLB, or another approved integration owns the endpoint. ALB generally supports L7 routing; NLB is L4 and commonly fronts a gateway/controller. Validate ingressClass/loadBalancerClass, target type (instance/IP), subnet discovery, public/internal scheme, security groups, listener certificates (ACM when used), target group health checks, deregistration delay, idle timeout, WAF placement and access logs. Never infer these from Kubernetes `Service.type` alone.
```bash
kubectl get ingressclass -o yaml
kubectl -n <namespace> describe ingress <ingress>
kubectl -n <namespace> get svc <gateway-service> -o yaml
aws elbv2 describe-load-balancers --region <region>
aws elbv2 describe-target-health --target-group-arn <arn> --region <region>
```
Use authorized read-only cloud credentials and filter results to the owned endpoint; do not paste account identifiers or certificate material into incident reports.

## 8. Azure AKS implementation checklist
Discover whether Application Gateway for Containers, Application Gateway Ingress Controller, NGINX or another approved Gateway API/Ingress controller owns routing. Validate public/private frontend, ingress/GatewayClass, Azure load balancer integration, subnet/NSG/UDR, private DNS, health probes, backend health, certificate integration (Key Vault where applicable), WAF, idle timeout and diagnostic logs. Product feature support varies by controller/version; never assume every Gateway API feature is available.
```bash
kubectl get ingressclass,gatewayclass
kubectl -n <namespace> get ingress,svc -o wide
az network application-gateway list -o table
az network lb list -o table
```
Scope Azure CLI queries to the approved resource group/subscription before operational use.

## 9. Health checks and readiness
Distinguish (1) LB target health, (2) gateway listener/proxy health, (3) Service EndpointSlice readiness, and (4) model-ready application health. A 200 from a proxy status endpoint is insufficient to establish that a GPU model has loaded. Verify target port, probe path, protocol, success codes, host header, security rules, interval/threshold and warmup. Prevent unhealthy targets from receiving new requests; test failover without bypassing the application authorization layer.

## 10. TLS, identity and security boundaries
Document client-to-LB TLS, LB-to-gateway TLS and gateway-to-backend TLS independently. Verify hostname/SNI, certificate chain/expiry, secret rotation, trusted proxy headers and origin IP forwarding. Apply authN/authZ at the designated enforcement point and ensure no alternate NodePort, public LB or direct backend path bypasses it. Restrict network ingress and egress by least privilege; verify private endpoints, security groups/NSGs, WAF and rate limits. Do not log bearer tokens, patient data, full prompts or model outputs.

## 11. Streaming and connection draining
For SSE/token streaming, compare **idle timeout** (gap between bytes), request/response timeout, overall duration, upstream read timeout, proxy buffering, LB deregistration and Pod termination grace. A 60-second idle timeout can terminate a stream even when the model remains healthy. Disable buffering only where supported and justified; validate client disconnect propagation and cancellation. On rollout: stop new traffic, mark endpoint draining, allow existing streams to finish within bounded grace, then terminate. Test long responses, client cancellation and model cold starts in isolated staging. WebSocket upgrades require separate validation where used.

## 12. Safe change and rollback runbook
1. Record baseline: 2xx/4xx/5xx, 502/503/504, p50/p95/p99, active streams, resets, backend health, route conditions, target counts and TLS errors.
2. Identify GitOps/TrueFoundry authoritative config, owner, approved change window and immutable version.
3. Validate manifests using server-side dry run in staging; check host conflicts and security exposure.
4. Apply to isolated staging; confirm route accepted/programmed, DNS, TLS, auth, health, synthetic inference and stream completion.
5. Roll out in approved increments with explicit stop thresholds and monitoring.
6. On regression, restore authoritative previous config and confirm backend/target recovery, stream integrity and SLO.
Never mutate production controllers, security groups or DNS as an ad-hoc diagnostic shortcut.

## 13. Troubleshooting matrix
| Symptom | First checks | Likely boundary |
|---|---|---|
| NXDOMAIN | DNS zone, record, resolver | Client → DNS |
| Connection refused | listener, LB address/port | Client → LB |
| TLS mismatch | SNI, cert, listener | TLS termination |
| 404 | hostname/path, route match | Gateway routing |
| 401/403 | auth, WAF, policy | Security boundary |
| 502 | gateway-to-backend connect, protocol | Gateway → Service |
| 503 | backend health, endpoints, route refs | Service → Pod |
| 504 | upstream timeout, cold start | Gateway → runtime |
| Midstream reset | idle timeout, draining, buffering | Streaming path |
| One AZ fails | targets, subnets, SG/NSG, routing | Cloud LB/network |

For every incident capture **LPHL** (Lowest Proven Healthy Layer) and **FFT** (First Failed Transition), correlate request IDs across LB/gateway/app and avoid attributing errors to TrueFoundry without evidence.

## 14. Operational runbooks
**R1 502/503:** confirm gateway route and backend references; check EndpointSlices and target health; rollback incorrect mapping.  
**R2 TLS outage:** check SNI/chain/expiry/rotation; restore approved certificate reference.  
**R3 504:** compare cold-start metrics and gateway/LB timeouts; tune only with evidence and change control.  
**R4 Streaming resets:** inspect active streams, idle timers and rollout termination; test bounded draining.  
**R5 Public exposure:** stop change, engage security owner, restore private listener/security rules through controlled rollback.  
**R6 Cloud LB unhealthy:** inspect target registration, security groups/NSGs and probe configuration; correct source-of-truth.  
**R7 Gateway API rejected:** inspect conditions, controller support, parentRefs and ReferenceGrant; revert invalid routes.  
**R8 AZ impairment:** validate healthy-zone capacity, topology and cloud LB target selection; follow incident command.

## 15. Production acceptance criteria
All approved hostnames route to the correct immutable model revision; no auth bypass; TLS and health checks pass; routing conditions healthy; long-running inference streams finish or fail predictably under controlled drain; dashboards and alert ownership documented; rollback rehearsed; evidence archived. Unexecuted tests must be recorded as **Not run**, never Pass.

**Next:** Chapter 9.4 — DNS, TLS, Certificates and Secure Endpoint Exposure.
