# Chapter 9.6 — Inference Streaming, Timeouts and Networking Troubleshooting

**Part 09 — Networking | Canonical production tutorial**

## Objectives and operating rules

Diagnose and optimize streaming inference over HTTP chunked responses, Server-Sent Events (SSE), and WebSockets where supported; distinguish **time to first token (TTFT)** from **first byte**, total response duration, and end-to-end latency; identify timeouts across client, DNS, TLS, load balancer, ingress/gateway, Service, proxy, model server, and upstream dependencies; safely recover from 499/502/503/504, resets, stalls, and premature disconnects.

**Safety:** `[SAFE-READ]` observational; `[LAB-MUTATE]` isolated staging only; `[CHANGE-CONTROL]` production approval with rollback. Confirm the actual TrueFoundry deployment topology, inference API and controller/version before following configuration examples. Product features and timeout knobs are not assumed.

## 1. Inference request path and streaming phases

```text
Client
  → DNS lookup → TCP connection → TLS handshake
  → external LB / WAF / CDN (if deployed)
  → ingress / Gateway API / proxy
  → Kubernetes Service → ready EndpointSlice → serving Pod
  → inference queue → prefill → first token → subsequent tokens
  → reverse stream path → client
```

Separate:
- **DNS/TCP/TLS setup:** resolution, connect, handshake.
- **Gateway admission:** auth, rate limiting, request-body read, backend routing.
- **Queue/prefill:** GPU/model readiness, queue time, prefill compute; may delay first token without any network fault.
- **First byte:** earliest HTTP payload byte; headers or heartbeat may precede first model token.
- **TTFT:** time until first model token reaches the client, measured with the application's streaming protocol.
- **Inter-token latency (ITL):** time between generated tokens/chunks; **time per output token (TPOT)** is workload-dependent and should be defined explicitly.
- **Completion/stream close:** last token, usage metadata if emitted, clean end marker, or client cancellation.

`LPHL` (last progress/healthy log) and `FFT` (first failure timestamp) can help bound incidents; define them in incident notes rather than treating them as standardized vendor metrics.

## 2. Streaming protocols and failure behavior

**SSE:** usually `Content-Type: text/event-stream`, events separated by blank lines, client reconnect semantics depend on application. Reverse-proxy buffering may batch events and destroy token cadence. Test with `curl -N` and application-level event parsing; headers alone do not prove live streaming.

**HTTP chunked:** HTTP/1.1 may use `Transfer-Encoding: chunked`; HTTP/2 uses data frames rather than that header. Never diagnose absence of a chunked header on HTTP/2 as a failure.

**WebSockets:** require a successful HTTP upgrade for HTTP/1.1 or supported extended CONNECT for HTTP/2; load balancers and gateways differ. Ping/pong and idle limits must be validated. Do not assume TrueFoundry inference supports WebSockets for every deployment.

**gRPC streaming:** requires compatible HTTP/2 path, trailers, flow control and proxy support. Distinguish gRPC status from HTTP status.

**Cancellation:** a client disconnect can terminate generation; it should release compute and avoid expensive orphaned GPU work. Check actual server behavior and logs. A proxy 499 is a common NGINX-specific client-closed-request status, not a universal HTTP status.

## 3. Timeout budget: use the smallest effective deadline

A stream can be terminated by any component with a shorter timeout than its expected quiet interval or total duration. Inventory these distinct controls:

| Layer | Typical failure class | What to inspect |
|---|---|---|
| Client SDK | overall request deadline/read idle | client timeout, retry policy, cancellation |
| DNS/TCP/TLS | connect timeout/handshake | resolver, firewall, certificates, SYN retries |
| CDN/WAF | hard origin response/idle timeout | provider product limits and buffering |
| Cloud LB | idle connection timeout | AWS ALB/NLB or Azure LB/App Gateway specifics |
| Ingress/gateway | read/send/stream idle timeout | controller-specific annotations/policies |
| Service mesh | route timeout, per-try timeout, idle | Envoy/Istio/mesh settings if deployed |
| Inference proxy | upstream timeout, buffer limits | proxy deployment config |
| Model server | request/queue/decode limits | worker saturation, token limits |
| Client network | NAT/firewall idle session | connection resets, mobile networks |

**Example only:** if a stream has a 75-second period with no bytes, a 60-second idle timeout anywhere on its path can close it, regardless of a 10-minute client total deadline. Conversely, increasing every timeout can conceal queue overload and exhaust connections. Budget queue/prefill and token streaming separately; prefer bounded application heartbeats when supported and semantically safe, without treating heartbeats as successful model progress.

## 4. Establish baseline and collect evidence [SAFE-READ]

```bash
kubectl config current-context
kubectl get ingressclass,ingress -A
kubectl get gatewayclass,gateway,httproute -A 2>/dev/null || true
kubectl get svc,endpointslice -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

For an approved **non-sensitive** inference test endpoint:

```bash
export URL='https://inference.lab.example.com/stream'
curl --http1.1 -N -v --connect-timeout 5 --max-time 180 "$URL" -o /tmp/stream-test.out
curl -sS -o /dev/null -w 'dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} first_byte=%{time_starttransfer} total=%{time_total} status=%{http_code}\n' "$URL"
```

These commands require the endpoint's approved auth and request method/body; do not send tokens in command-line URLs, terminal history or captured logs. A GET may not be supported for an inference POST API. `time_starttransfer` is first byte, **not** TTFT. Avoid interpreting a single curl sample as representative p95/p99.

For an authenticated POST, use an approved test payload file and protected header mechanism; do not publish credentials. Record per-event timestamps with an SSE-aware client to measure actual token arrival.

## 5. Structured fault isolation

1. **Identify scope:** all users or one region, model, route, gateway, pod, tenant or client SDK?
2. **Classify symptom:** DNS/TLS/connect failure; HTTP error before stream; stream stalls; truncated stream; 499/client cancel; backend 5xx; auth error.
3. **Correlate request IDs:** client → LB → ingress → inference server, without logging sensitive prompts.
4. **Compare first byte vs first token:** first byte fast + token slow suggests queue/prefill/model path; both slow may implicate routing, proxy, auth, or model.
5. **Check readiness and saturation:** EndpointSlices, restarts, CPU/GPU saturation, KV-cache pressure, batch/queue depth, OOM and cold model loads.
6. **Check idle vs absolute deadlines:** proxy, LB, mesh, WAF, SDK, and upstream server.
7. **Check stream mechanics:** response buffering, compression, HTTP version, proxy flush, heartbeats, flow control, slow clients.
8. **Verify egress dependencies:** DNS, object storage/model fetch, telemetry, private endpoints, network policy.
9. **Recover safely:** restore last known-good config, drain active streams, validate synthetic and real workload metrics.

## 6. Controller/cloud-specific validation

**NGINX Ingress (if installed):** inspect proxy read/send timeouts, request/response buffering, annotations supported by installed controller and ConfigMap defaults. Disabling buffering globally can increase resource use; scope changes to verified streaming routes. Check upstream keepalive and client abort logs.

**Gateway API / Envoy / service mesh:** inspect listener, HTTPRoute, BackendRef conditions, route timeout policies where supported, per-try vs global timeout, stream idle timeout and retry behavior. Gateway API support varies by controller/version; never assume a timeout field is portable.

**AWS EKS:** identify ALB vs NLB, idle timeout, target group health, deregistration delay, security group/NAT behavior and whether a WAF/CDN sits in front. Do not conflate ALB idle timeout with application deadline.

**Azure AKS:** identify Application Gateway or other controller, backend request timeout, probes, NSGs, Azure Firewall and private endpoint routing. Check controller-specific backend timeouts rather than assuming a Kubernetes Ingress annotation is honored.

**Client retries:** retrying a non-idempotent generation can duplicate work/cost. Do not automatically retry after tokens were emitted; define cancellation, request IDs, retryable status codes and idempotency support explicitly.

## 7. Reliability metrics and capacity

Capture request count and status, streaming success/completion rate, client cancellation, TTFT p50/p95/p99, ITL/TPOT, output tokens/sec, queue delay, concurrent streams, active connections, upstream resets, gateway buffer usage, model worker utilization, GPU memory, and backend readiness. Define **stream completion** at the application protocol level; HTTP 200 with an interrupted body is not success.

Estimate concurrency from arrival rate × mean duration (Little's Law, under stable conditions). Long-lived streams consume connections even with low bandwidth. Budget file descriptors, proxy worker connections, backend concurrency, NAT ports and graceful drain duration. Load tests must use representative token counts and realistic client read rates.

## 8. Twelve production runbooks

### RB-01 — High TTFT, fast first byte
Check queue, prefill, cold model load, GPU saturation and application heartbeats; avoid blaming DNS without evidence. Scale or restore healthy serving capacity under change control.

### RB-02 — First byte slow and elevated 504
Compare ingress upstream timings, backend response timeout, pod readiness and queue depth. Identify which component generated 504; reduce load or revert the failed rollout before blindly increasing timeouts.

### RB-03 — SSE events arrive in batches
Inspect proxy response buffering, compression, content type and application flush behavior. Test a scoped staging change, then promote with monitoring.

### RB-04 — Stream ends at fixed elapsed time
Compare exact termination interval with LB/gateway/mesh/client idle and absolute timeouts; identify whether bytes were flowing. Adjust the actual limiting layer under change control.

### RB-05 — Client 499 spikes
Confirm NGINX status semantics, client timeout/cancellation, slow first token, mobile network churn and user navigation. Correlate client traces and gateway access logs.

### RB-06 — 502 or upstream reset
Inspect Pod restarts, server crashes, connection resets, backend TLS, proxy upstream logs and EndpointSlice churn. Roll back bad release or restore capacity.

### RB-07 — 503 after rollout
Check ready endpoints, readiness probes, gateway route attachment, NetworkPolicy and HPA scale-up lag. Restore previous healthy backend configuration.

### RB-08 — Streams break during deployment
Inspect terminationGracePeriodSeconds, preStop behavior, readiness withdrawal, gateway deregistration delay, PodDisruptionBudget and active-stream drain. Use rolling rollout and verify completion.

### RB-09 — WebSocket upgrade fails
Check Upgrade/Connection headers on HTTP/1.1, proxy compatibility, route policies and auth. Do not assume an HTTP/2 path automatically supports WebSockets.

### RB-10 — HTTP/2 or gRPC stream reset
Inspect GOAWAY/RST_STREAM, flow control, proxy idle policies, trailers, stream limits and upstream logs. Validate the exact protocol across every hop.

### RB-11 — Slow-reading client causes memory pressure
Inspect gateway response buffering, backpressure, connection count and output queue size. Set bounded concurrency and resource controls; avoid unbounded buffering.

### RB-12 — Retries amplify GPU load during outage
Check client/proxy retry policy, 429/503 responses, duplicate generation and cost. Disable unsafe retries, use backoff/jitter where appropriate, and restore stable admission.

## 9. Production change, rollback and incident record

Document: timestamp and region; user impact; LPHL/FFT; endpoint/model/version; client and proxy protocol; exact timeout values per layer; request IDs; baseline TTFT/ITL/error rate; concurrency and GPU saturation; chosen change; approval; staged validation; rollback command or Git commit; post-change SLO evidence. Prefer **reverting the specific bad configuration** over disabling auth/TLS or opening network policy. Preserve active streams where possible; never claim zero disruption without testing.

## 10. Completion

Execute the [companion hands-on lab](../labs/06-inference-streaming-timeouts-networking-troubleshooting-check.md) with **28 failure scenarios and 10 acceptance gates**. Publication alone does not constitute execution, vendor review or production certification.

**Next:** Chapter 9.7 — Production Networking and Recovery Capstone.
