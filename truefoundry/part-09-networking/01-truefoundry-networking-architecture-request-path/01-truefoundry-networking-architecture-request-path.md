# 9.1 — TrueFoundry Networking Architecture and End-to-End Request Path

**Part 09 — Networking | Chapter 1 of 7**  
**Audience:** TrueFoundry platform engineers, Kubernetes SREs, cloud networking engineers, inference operators  
**Labels:** **[SAFE-READ]** observation; **[LAB-MUTATE]** approved isolated staging; **[CHANGE-CONTROL]** production changes.

## 1. Learning objectives

Trace both the **control-plane management path** and the **application inference request path**; identify Kubernetes Services, EndpointSlices, gateways/Ingress, DNS, TLS, CNI and cloud load balancers; understand internal versus external exposure, routing and streaming timeouts; diagnose network faults with Lowest Proven Healthy Layer (LPHL) and First Failed Transition (FFT); and validate rollback safely.

Prerequisites: Parts 01–08, read-only access to an authorized TrueFoundry workspace and Kubernetes cluster, a known-good model endpoint, approved networking inventory and a named network owner. **TrueFoundry deployment-specific components, gateway providers, CRDs and hostnames must be discovered rather than assumed.**

## 2. Two distinct network paths

### A. Management/control-plane path

```text
TrueFoundry control plane / API / UI
          ↕ authenticated outbound/inbound integration (deployment-dependent)
cluster integration / tfy-agent
          ↕ Kubernetes API and authorized resources
workloads / namespaces / reconciliation
```

A healthy TrueFoundry UI does not prove application traffic is healthy. A healthy model endpoint does not prove management reconciliation is healthy. The precise `tfy-agent` connection direction, ports and dependencies are **installation-specific**; verify the deployed architecture and vendor docs.

### B. Runtime/inference data path

```text
Client → DNS → external/internal LB → Gateway/Ingress
       → TLS termination / authentication / routing
       → Kubernetes Service → EndpointSlice (Ready Pod IP)
       → Pod network / CNI → application/vLLM listener
       → token streaming → gateway → client
```

Not every deployment has all layers; internal clients may access ClusterIP Services directly, and gateway/TLS termination may be elsewhere. Identify the **actual** path before interpreting metrics.

## 3. Layer-by-layer ownership and evidence

| Layer | Healthy evidence | Common owner |
|---|---|---|
| Client | Correct URL, trust chain, authenticated request | Application |
| DNS | Expected A/AAAA/CNAME resolution and TTL | DNS/network |
| Cloud LB | Listener, target health, security rules | Cloud/platform |
| Gateway/Ingress | Route match, policy, upstream health | Gateway/platform |
| TLS | Valid SAN, certificate chain, SNI, protocol | Security/network |
| Service | Correct port/targetPort, selector | Workload/platform |
| EndpointSlice | Ready addresses, correct namespace/port | Kubernetes/workload |
| CNI/network policy | Pod-to-Pod route and allowed flow | Cluster/network |
| Pod listener | Bound port, readiness, correct protocol | Application |
| vLLM | Model Ready, TTFT/ITL, successful stream | Inference/SRE |
| Control integration | Agent/API reconciliation, credentials | TrueFoundry/platform |

A missing EndpointSlice Ready address is not a DNS failure. A 503 from a gateway may mean no healthy upstreams; confirm the gateway's actual error reason before assigning ownership.

## 4. Kubernetes Service discovery and port mapping

Kubernetes Services provide stable virtual addresses and service discovery. `ClusterIP` is normally internal, `LoadBalancer` provisions cloud-facing or internal load balancing according to provider configuration, and `NodePort` exposes node ports. An Ingress or Gateway routes HTTP(S) according to installed controller and configuration; creating an Ingress without a controller does not create a working route.

Example **illustrative staging-only** Service (do not apply to existing workloads without approval):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: inference-staging
  namespace: inference-staging
spec:
  type: ClusterIP
  selector:
    app: inference-staging
  ports:
    - name: http
      port: 80
      targetPort: 8000
      protocol: TCP
```

Confirm that the selector matches Pods and the application actually listens on port 8000. For HTTP routing, `Host` and path matching may determine backend selection.

## 5. EndpointSlices and readiness

EndpointSlices contain network endpoints associated with a Service. An endpoint's ready/serving/terminating conditions affect traffic routing depending on controller behavior. During GPU model startup, Pods can be Running yet not Ready because model download/CUDA initialization is unfinished. During scale-down, endpoint removal and connection draining must preserve long-lived streams.

```bash
kubectl -n <namespace> get svc -o wide
kubectl -n <namespace> describe svc <service>
kubectl -n <namespace> get endpointslices -l kubernetes.io/service-name=<service> -o yaml
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <pod>
```

**[SAFE-READ]** Be aware that YAML output may include internal addresses; keep evidence within approved channels.

## 6. DNS resolution and failure domains

Trace client resolver → authoritative record → cloud LB/gateway → Service DNS where relevant. Kubernetes cluster-local names typically resemble `service.namespace.svc.cluster.local`, but cluster domain can differ. A DNS record resolving to an address does **not** prove the listener, route or backend is healthy.

Validate split-horizon/private DNS, record TTL, search domains, CNAME chains and negative caching. Avoid publishing private endpoint addresses in incident tickets accessible outside the authorized team.

## 7. TLS and identity boundaries

Inventory where TLS terminates (cloud LB, ingress/gateway, service mesh, Pod) and whether TLS is re-established upstream. Check SNI, SAN, certificate expiry, intermediate chain, minimum TLS version and mTLS where configured. Avoid `curl -k` as a default diagnostic; bypassing verification can hide the real failure.

A trusted TLS connection does not imply application authorization. Authentication/authorization may occur at the gateway, service or both; trace 401/403 separately from handshake errors.

```bash
# From an authorized workstation with approved endpoint:
curl --head --connect-timeout 5 --max-time 15 https://<approved-host>/
openssl s_client -connect <approved-host>:443 -servername <approved-host> -showcerts
```

The `openssl` command exposes public certificate chain data; verify output carefully and do not capture credentials.

## 8. Gateway, load balancer and streaming

LLM streaming responses can last much longer than ordinary HTTP requests. Check the complete timeout chain: client, cloud LB idle timeout, gateway upstream timeout, reverse proxy, application timeout and termination grace. Ensure HTTP/1.1 chunking or HTTP/2 behavior is supported end to end, including server-sent events if used.

Validate:

- Host/path route, gateway class and controller ownership.
- LB target health, security group/NSG, firewall and subnet routing.
- Keep-alive, idle/read/write timeouts and buffering behavior.
- Streaming completion on rolling updates, HPA scale-down and node drains.
- Retries: never replay a partially streamed non-idempotent request blindly.
- Rate limits and admission responses (429) versus upstream 503/504.

## 9. Kubernetes CNI, network policy and cloud boundaries

CNI provides Pod networking; implementation varies (EKS VPC CNI, Azure CNI modes, other CNIs). NetworkPolicy only takes effect when the installed dataplane supports and enforces it. A policy that allows traffic to a Service address may need validation against actual Pod-level policy semantics.

Review namespace isolation, egress to model registries/artifact storage, DNS egress, control-plane connectivity, private endpoint routing, NAT and cloud security rules. An image/model download timeout may be an egress, identity or storage problem rather than an inference-engine issue.

## 10. Safe diagnostic sequence

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods -o wide
kubectl -n <namespace> get svc,ingress -o wide
kubectl -n <namespace> get gateways.gateway.networking.k8s.io,httproutes.gateway.networking.k8s.io
kubectl -n <namespace> get endpointslices
kubectl -n <namespace> get networkpolicy
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> describe svc <service>
```

The Gateway API command is **optional only if CRDs exist**; absence means not installed. Use an existing approved diagnostic Pod to run `nslookup` or `curl` if present; do not create a privileged debugging Pod in production. `kubectl port-forward` changes local connectivity and may bypass ingress controls—use only when approved.

## 11. Troubleshooting decision tree

```text
Request failed
 ├─ DNS lookup fails → resolver/record/private DNS
 ├─ TCP connect fails → LB/listener/firewall/routing
 ├─ TLS handshake fails → SNI/certificate/protocol
 ├─ 401/403 → gateway/application authorization
 ├─ 404 → Host/path route or application route
 ├─ 502/503 → upstream/EndpointSlice/readiness/port
 ├─ 504/timeout → gateway timeout, model cold start, backend stall
 ├─ first token slow → gateway queue / vLLM prefill / model warmup
 └─ stream truncates → idle timeout / buffering / Pod termination
```

Prove the last healthy layer before changing a component. Do not assume a network fault solely because the client reports a timeout.

## 12. Failure matrix

| ID | Symptom | FFT hypothesis | Evidence |
|---|---|---|---|
| N1 | NXDOMAIN | DNS record → resolver | DNS response |
| N2 | Wrong private/public address | Split DNS → client | Resolver path |
| N3 | TCP timeout | Client → LB | Listener/security |
| N4 | TLS certificate mismatch | TLS SNI → cert | SAN/SNI |
| N5 | Expired TLS chain | TLS trust → gateway | Certificate dates |
| N6 | 401/403 | Identity → route | Auth policy |
| N7 | 404 at ingress | Route match → backend | Host/path |
| N8 | 503 no endpoints | Service → Ready Pods | EndpointSlice |
| N9 | 502 wrong targetPort | Service → listener | Port mapping |
| N10 | 504 cold model | Gateway → Ready backend | Warmup/timeout |
| N11 | NetworkPolicy denies traffic | CNI policy → Pod | Policy/flow |
| N12 | Security group/NSG blocks | LB → node/Pod | Flow logs |
| N13 | Model registry egress fails | Pod → artifact | Egress path |
| N14 | DNS egress blocked | Pod → resolver | DNS policy |
| N15 | Streaming truncated | Proxy → client | Idle timeout |
| N16 | Gateway buffers stream | Engine → gateway | Chunk timing |
| N17 | Scale-down interrupts stream | Endpoint drain → client | Termination |
| N18 | LB target unhealthy | LB → endpoint | Health check |
| N19 | Service selector mismatch | Service → EndpointSlice | Labels |
| N20 | Gateway API controller absent | Route → controller | CRD/controller |
| N21 | CNI IP exhaustion | Node → Pod IP | CNI events |
| N22 | MTU/fragmentation issue | Pod network → transport | Packet evidence |
| N23 | TLS re-encryption mismatch | Gateway → upstream | Upstream SNI |
| N24 | Control-plane integration unavailable | tfy-agent → API | Agent/API logs |

Each scenario requires LPHL, FFT, affected traffic, evidence confidence, Current Actionable Owner, mitigation, rollback and end-to-end validation.

## 13. Incident runbook

1. **Declare** affected hostname/route, status codes, TTFT/ITL, stream completion and onset.
2. **Separate** management-path symptoms from runtime request-path symptoms.
3. **Trace** DNS → TCP → TLS → gateway/LB → Service → EndpointSlice → Pod → vLLM.
4. **Preserve** route configuration, controller events, target health, cert chain, flow logs and known-good revision.
5. **Contain** using approved routing, rate limits or rollback; avoid exposing private services or disabling TLS verification.
6. **Recover** the first proven failed transition via authoritative TrueFoundry/GitOps or cloud change control.
7. **Validate** client TLS, correct model response, Ready endpoints, streaming completion, latency/error SLO and clean rollback.

## 14. Acceptance gates

| Gate | Required evidence |
|---|---|
| A1 | Management and runtime paths documented separately |
| A2 | DNS, LB, TLS and gateway ownership verified |
| A3 | Service/EndpointSlice/Pod port and readiness mapping |
| A4 | CNI, NetworkPolicy, cloud firewall and egress reviewed |
| A5 | Streaming timeout and graceful drain contract |
| A6 | N1–N24 failure classifications with LPHL/FFT |
| A7 | Approved staging path validation or Not run |
| A8 | Rollback, model correctness and end-to-end SLO evidence |

## 15. Evidence template

```text
Date/time / owner / change approval:
TrueFoundry workspace, cluster, namespace:
Management path vs runtime path:
Client host / DNS resolver / LB / gateway controller:
TLS termination, SNI, SAN, cert validity, mTLS:
Service type/port/targetPort/selector:
EndpointSlice Ready addresses / Pod listener:
CNI/NetworkPolicy/cloud firewall/egress:
Gateway idle/upstream timeouts, streaming behavior:
Status codes, TTFT/ITL, stream completion:
LPHL/FFT, N1–N24, mitigation, rollback:
A1–A8 Pass/Fail/Not run, owner and evidence:
```

**Hands-on lab:** [9.1 — Networking Architecture and Request Path Check](../labs/01-truefoundry-networking-architecture-request-path-check.md)

**Next:** Chapter 9.2 — Kubernetes Services, EndpointSlices and Service Discovery.

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
