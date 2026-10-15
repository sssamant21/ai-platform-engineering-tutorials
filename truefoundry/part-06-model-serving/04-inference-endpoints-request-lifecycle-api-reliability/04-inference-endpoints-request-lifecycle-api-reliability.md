# 6.4 — Inference Endpoints, Request Lifecycle and API Reliability

**Part:** 06 — Model Serving · **Chapter:** 4 of 7  
**Audience:** TrueFoundry platform engineers, MLOps, API engineers, security engineers and SREs  
**Safety:** Default inspection is **[SAFE-READ]**. Endpoint exposure, auth policies, timeouts, traffic controls and deployment changes are **[CHANGE-CONTROL]**; experiment only in an approved isolated nonproduction workspace.

## 1. Learning objectives

Trace an inference request across DNS, TLS, gateway, authentication, Service, model runtime and response; design stable APIs; implement bounded concurrency, rate limits, timeouts and safe retries; distinguish infrastructure failures from invalid input and model errors; and validate endpoint SLOs and rollback.

## 2. Serving request architecture

```text
Authorized client / SDK
  → DNS resolution / TLS handshake
  → Load balancer / Gateway or Ingress
  → Authentication / authorization / quotas
  → Request size and schema validation
  → Kubernetes Service → Ready EndpointSlice → serving Pod
  → Admission queue / concurrency limit / batching
  → Preprocessing / tokenization
  → CPU or GPU model execution
  → Postprocessing / response serialization
  → HTTP response or streaming chunks
  → metrics / traces / structured, sanitized logs
```

**Control path:** TrueFoundry deployment specification, workspace, cluster integration and approved endpoint configuration reconcile desired infrastructure.  
**Data path:** clients send actual traffic through the installed ingress/gateway and workload. A healthy control plane does not guarantee request success. TrueFoundry endpoint configuration and supported inference API schemas vary by platform version; use the installed version's supported workflow rather than guessed CLI flags.

## 3. Endpoint exposure and trust boundaries

| Endpoint mode | Use case | Controls |
|---|---|---|
| Internal-only | Private service-to-service inference | Namespace/network policy, service identity, internal DNS |
| External authenticated | Approved application clients | TLS, authentication, authorization, quotas, abuse protection |
| Public unauthenticated | Generally inappropriate for sensitive models | Explicit exception, strict quotas and security review |
| Streaming | Token or event stream | Long-lived connection limits, cancellation, idle timeout |

Distinguish Kubernetes ClusterIP from an externally reachable route. The Service may be healthy while gateway host/path, certificate, firewall or client authorization is broken. Never expose a model endpoint publicly as a troubleshooting shortcut.

## 4. API contract and versioning

Document method, path, content type, auth, schema, max request size, response schema, model ID/version, timeout, errors and streaming semantics. A minimal illustrative synchronous contract:

```http
POST /v1/predict HTTP/1.1
Content-Type: application/json
Authorization: Bearer <redacted-token>
X-Request-ID: <client-generated-id>

{"model_version":"042","inputs":[{"features":[0.1,0.2]}]}
```

```json
{"request_id":"example-123","model_version":"042","predictions":[0.91]}
```

These are **illustrative application API examples**, not guaranteed TrueFoundry-native routes or schemas. Avoid logging raw payloads, access tokens or regulated data. Reject incompatible schema changes or use explicit API versioning and compatibility tests.

## 5. HTTP and streaming response semantics

| Status | Typical meaning | First diagnostic check |
|---|---|---|
| 200 | Request succeeded | Validate output correctness/version |
| 400/422 | Invalid schema/input | Client contract, validation |
| 401 | Authentication missing/invalid | Identity provider and token metadata |
| 403 | Authorized identity lacks permission | Role, policy, model access |
| 404 | Route/model path missing | Host/path and model registry |
| 408/504 | Request or gateway timeout | Deadline budget and backend latency |
| 413 | Payload too large | Gateway/app size limits |
| 429 | Quota/rate/concurrency limit | Retry-After, queue admission |
| 500 | Application/runtime failure | Sanitized app logs, trace |
| 502 | Bad gateway/upstream protocol failure | Gateway-to-backend protocol |
| 503 | No ready backend/overload | Ready EndpointSlices, capacity |

HTTP semantics depend on gateway/runtime; confirm actual response source and avoid interpreting all 5xx as GPU failures. For streaming APIs, an HTTP 200 may precede a midstream failure; track stream completion and client cancellation separately.

## 6. End-to-end latency budget

```text
Client connect/TLS + gateway processing + auth
 + routing + admission queue + preprocessing
 + model execution + postprocessing + network transfer
```

Stages may overlap, especially for streaming/batching, so sum only mutually exclusive measured spans. Define SLOs for P95/P99 end-to-end latency and availability, plus time-to-first-token (TTFT), inter-token latency and stream completion where relevant. A useful budget identifies the owner and maximum acceptable duration at each boundary.

| Layer | Metric | Typical failure |
|---|---|---|
| Client/DNS/TLS | Connect/handshake time | DNS/certificate/network |
| Gateway | Upstream latency and 5xx | Routing, backend health |
| Auth | Authorization duration/rejections | Identity outage/policy |
| Admission | Queue wait/rejections | Overload |
| Runtime | Inference time | CPU/GPU saturation |
| Serialization | Response size/time | Oversized payload |
| Streaming | TTFT, tokens/s, completion | Buffering/disconnect |

## 7. Timeouts and cancellation

Set a **client deadline** and align downstream timeouts (gateway, application, model execution, dependencies) so no component continues expensive work after cancellation without a deliberate reason. Differentiate connect, request, read, idle and total deadlines. For streaming, idle timeouts and heartbeats may differ from total-generation limits.

When an upstream cancels, the serving process should release queued work, GPU memory and per-request resources when supported. A gateway 504 can mean the backend is still executing unless cancellation is propagated.

## 8. Retries and idempotency

- Retry only errors that are transient and safe for the operation.
- Use bounded attempts, exponential backoff with jitter and a total deadline.
- Honor `Retry-After` on 429/503 where supplied.
- Prevent synchronized retries that amplify GPU overload.
- Treat inference as potentially **non-idempotent** if it logs billable requests, changes state or uses stochastic generation.
- Use request IDs and server-side deduplication where required.
- Never retry malformed 400/422 requests without correcting the payload.

**Retry budget example:** maximum two retries, bounded by an overall client deadline and a retry-rate budget. Values must be tested against your actual latency SLO and request cost.

## 9. Concurrency, batching and backpressure

Concurrency can improve throughput until queueing, CPU preprocessing, GPU saturation or VRAM becomes limiting. Dynamic batching may improve GPU utilization but adds wait time; set maximum batch size and maximum queue delay. Separate queue capacity from worker capacity and reject excess requests quickly with a clear 429/503 rather than accepting unbounded work.

Use a **load-shedding policy**: per-tenant quotas, bounded admission queue, maximum in-flight requests, cancellation and request-size limits. Avoid unlimited gateway retries and scale-outs that all cold-load the same large model simultaneously.

## 10. Authentication and authorization

Prefer short-lived workload identities or supported gateway/OIDC authentication. Validate issuer, audience, expiry and authorized model/tenant scopes. Keep inference identities separate from model-artifact download identities and deployment administrators.

TLS should be validated end to end according to the chosen termination model; mTLS/mesh may add additional boundaries. Never use `curl -k` as an acceptance test or bypass auth to diagnose production failures.

## 11. Health and readiness

Use startup probes for initialization, readiness probes to gate serving after model load/warmup, and liveness for process health. A lightweight health endpoint must not execute an expensive model request on every probe.

Validate the **normal client path** separately from port-forward or Pod-local checks. A Ready Pod with zero ready Service endpoints may indicate label/port mismatch, readiness gating or controller propagation delay.

## 12. Production-safe diagnostic commands [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services,ingresses -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> describe service <service>
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get networkpolicies
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> logs <pod> --tail=150
```

For Gateway API, inspect installed resources if supported:

```bash
kubectl -n <namespace> get gateways,httproutes
kubectl -n <namespace> describe httproute <route>
```

The Gateway API CRDs may not be installed; an absent resource type is not proof of a serving failure.

### Authorized request checks

```bash
curl --connect-timeout 5 --max-time 30 -i https://<approved-endpoint>/health
curl --connect-timeout 5 --max-time 30 -i -H 'Content-Type: application/json' -H 'Authorization: Bearer <redacted-token>' --data '{"inputs":[{"features":[0.1,0.2]}]}' https://<approved-endpoint>/v1/predict
```

Replace placeholders and sample payload with the **documented** API contract. Prefer an approved secret injection method instead of entering bearer tokens in shell history; do not share raw request transcripts containing tokens or sensitive outputs.

```bash
openssl s_client -connect <approved-host>:443 -servername <approved-host> -showcerts
```

Inspect certificate chain, SAN and expiry without weakening verification.

## 13. Failure classification matrix

| Symptom | Evidence | First failed transition |
|---|---|---|
| DNS lookup fails | Resolver, records, search domain | Client → DNS |
| TLS SAN/expiry failure | Certificate chain, SNI | Client → TLS edge |
| 401/403 | Gateway identity/policy | Auth → authorization |
| 404 on correct host | Route/path match | Gateway → route |
| 413 | Gateway/app body limit | Request → validation |
| 422 | Schema contract | Input → runtime |
| 429 | Quota or concurrency | Admission → queue |
| 502 | Upstream protocol/connection | Gateway → Pod |
| 503 | No ready endpoints | Service → backend |
| 504 | Deadline exceeded | Gateway → runtime |
| Slow TTFT | Queue/preprocess/model load | Admission → first output |
| Stream stalls after 200 | Buffering/cancellation | Runtime → stream |
| Retry storm | Client retries + overloaded backend | Failure → admission amplification |
| Wrong model output | Model version/schema drift | Model → response correctness |
| Cross-tenant access | Authz policy gap | Identity → model scope |
| High P99, low GPU usage | CPU/network/queue bottleneck | Preprocessing → execution |

## 14. Incident runbook

1. **Detect:** capture endpoint, model version, first failure, error code mix, P95/P99, TTFT/stream completion, traffic and affected tenants.
2. **Trace:** DNS/TLS → gateway/auth → Service/EndpointSlice → admission → preprocessing → execution → response. Identify lowest proven healthy layer and first failed transition.
3. **Contain:** stop risky promotion, reduce traffic with approved controls, disable runaway retries and preserve known-good routing/revision.
4. **Repair:** correct the narrowest confirmed fault; adjust measured concurrency or timeouts only with load evidence.
5. **Rollback:** restore authoritative TrueFoundry/GitOps revision, compatible image/model version, endpoint policy and configuration.
6. **Validate:** authenticated golden request, correct model ID, readiness, normal traffic SLO, no queue growth, acceptable rejection rate and stream completion.
7. **Close:** incident timeline, RCA/COA, test gaps, rate-limit policy and ownership.

## 15. Acceptance gates

| Gate | Required evidence |
|---|---|
| G1 | Endpoint exposure and auth boundary documented |
| G2 | API schema, version and error contract tested |
| G3 | TLS and authorized client path verified |
| G4 | Ready endpoints and model correctness verified |
| G5 | Timeout and cancellation policy documented |
| G6 | Retry/rate-limit/backpressure policy validated |
| G7 | P95/P99, queue, error codes and stream metrics captured |
| G8 | Authoritative rollback and post-recovery tests documented |

## 16. Evidence template

```text
Date/time / TrueFoundry workspace / deployment:
Endpoint hostname/path and exposure mode:
Client identity scope (no credentials):
Image digest / model version:
DNS/TLS/gateway/route validation:
Service selector / Ready EndpointSlices:
API request/response schema and golden result:
Connect/gateway/queue/model/response latency:
P95/P99, TTFT, completion, 4xx/5xx/429:
Timeout, retry, rate-limit and concurrency settings:
Lowest Proven Healthy Layer:
First Failed Transition:
Owner / approved repair / rollback revision:
Post-recovery acceptance and cleanup:
```

**Hands-on lab:** [6.4 — Inference Endpoint Reliability Check](../labs/04-inference-endpoints-request-lifecycle-api-reliability-check.md)  
**Next:** Chapter 6.5 — Model Loading, Warmup, Health Probes and Readiness.

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Gateway API](https://gateway-api.sigs.k8s.io/)
- [HTTP Semantics RFC 9110](https://www.rfc-editor.org/rfc/rfc9110)
