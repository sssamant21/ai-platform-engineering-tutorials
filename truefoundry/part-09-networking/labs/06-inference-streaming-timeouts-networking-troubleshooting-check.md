# Chapter 9.6 — Hands-on Lab: Inference Streaming, Timeouts and Networking Troubleshooting

**Part 09 | Isolated staging | No production fault injection**

## Learning goals and safeguards

Trace a streaming request through DNS, TLS, LB, gateway, Service and serving Pod; distinguish first byte, first token and total duration; isolate fixed idle/absolute deadlines; detect proxy buffering, connection resets and client cancellations; rehearse safe rollback.

**[SAFE-READ]** inspect; **[LAB-MUTATE]** only authorized disposable staging; **[CHANGE-CONTROL]** production requires approval. Use synthetic prompts and non-sensitive endpoints. Never capture access tokens or prompt contents in shared evidence.

## Lab 0 — Inventory and baseline [SAFE-READ]

```bash
kubectl config current-context
kubectl get ingressclass,ingress -A
kubectl get gatewayclass,gateway,httproute -A 2>/dev/null || true
kubectl get svc,endpointslice -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -30
```

Document actual cloud LB/controller, ingress/gateway timeout configuration, backend readiness, inference protocol, streaming route, model version, client SDK and allowed test traffic. Stop if the topology is unknown.

## Lab 1 — Measure the full stream [SAFE-READ]

Using an approved authenticated staging request, record DNS, TCP, TLS, first byte, first model token, each subsequent token, last token and clean completion marker. Use an SSE-aware parser where applicable. A first-byte timing alone is **not** TTFT.

```bash
export URL='https://inference.lab.example.com/stream'
curl --http1.1 -N -v --connect-timeout 5 --max-time 180 "$URL" -o /tmp/lab-stream.out
curl -sS -o /dev/null -w 'dns=%{time_namelookup} tcp=%{time_connect} tls=%{time_appconnect} first_byte=%{time_starttransfer} total=%{time_total} http=%{http_code}\n' "$URL"
```

Replace the GET examples with approved POST payload/auth handling when the inference API requires it. Redact URLs, headers and output if sensitive. Compare HTTP/1.1 and HTTP/2 only where both are supported.

## Lab 2 — Build a timeout inventory [SAFE-READ]

Create a table for client connect/read/total deadlines; CDN/WAF; LB idle; gateway read/send/stream idle; service mesh; inference proxy; server queue/decode limits; NAT/firewall idle. Record values and evidence source. Identify the **minimum effective timeout** for each silent interval and total request duration.

## Lab 3 — Reproduce safe streaming behavior [LAB-MUTATE]

Deploy a synthetic staging stream generator that emits timestamps and numbered chunks/events with configurable inter-event delay. Use only a disposable namespace and test hostname. Run short, medium and long quiet-interval streams and a slow-reading client. Verify expected event cadence, clean completion, and bounded resource consumption.

## Lab 4 — Buffering and timeout fault injection [LAB-MUTATE]

1. Establish baseline SSE event spacing.
2. In a disposable gateway route, enable/disable response buffering **only if supported**; record observed batching.
3. Configure a lab-only idle timeout shorter than an intentional silent interval; capture disconnect timestamp and layer.
4. Restore the original timeout and confirm completion.
5. Test client cancellation and check server-side compute cleanup.
6. Run a staged rollout with active synthetic streams and observe graceful drain.
7. Roll back all lab changes, including proxy settings and workload versions.

Do not set cluster-wide defaults or impact shared ingress controllers.

## Lab 5 — Failure localization [SAFE-READ]

Correlate sanitized request IDs across client, LB/gateway and model server. Examine EndpointSlices, Pod restarts, readiness, queue depth, GPU/CPU utilization, gateway connection counters and CNI verdicts if relevant. Distinguish an HTTP 200 with a truncated stream from a completed inference response.

## Lab 6 — Capacity and safe recovery [LAB-MUTATE]

Run a bounded, rate-limited synthetic concurrency test approved for the staging cluster. Record p50/p95/p99 TTFT, ITL, error and cancellation rate, concurrent connections and GPU saturation. Stop on predefined thresholds. Revert test load and configuration; verify healthy baseline.

## Troubleshooting matrix — 28 scenarios

| # | Failure scenario | Primary evidence |
|---|---|---|
| 01 | DNS lookup slow | Resolver latency |
| 02 | TCP connect timeout | Route, firewall, SYN retries |
| 03 | TLS handshake failure | SNI, chain, listener |
| 04 | 401/403 before stream | Auth and policy |
| 05 | 404 from wrong host/path | Route matching |
| 06 | 429 from admission control | Rate limits and retries |
| 07 | 499 client cancel | Client deadline and access logs |
| 08 | 502 upstream reset | Pod/proxy connection logs |
| 09 | 503 no healthy backend | EndpointSlice readiness |
| 10 | 504 gateway timeout | Origin and configured deadline |
| 11 | Fast first byte, high TTFT | Queue/prefill |
| 12 | Cold model initialization | Model load and artifact fetch |
| 13 | GPU saturation | Utilization, queue and tokens/sec |
| 14 | KV-cache/memory pressure | Server metrics and OOM |
| 15 | SSE events buffered | Proxy buffering and flush |
| 16 | Compression delays flush | Compression config |
| 17 | HTTP/2 framing misdiagnosed | Protocol negotiation |
| 18 | WebSocket upgrade rejected | Upgrade support |
| 19 | gRPC stream reset | HTTP/2 GOAWAY/RST_STREAM |
| 20 | LB idle timeout | Fixed elapsed disconnect |
| 21 | Gateway read timeout | Upstream timing |
| 22 | Client total deadline | Client trace |
| 23 | NAT/firewall idle reset | Network flow and timestamps |
| 24 | Slow consumer backpressure | Proxy queue/memory |
| 25 | Pod rollout drops streams | Drain/termination |
| 26 | Readiness flapping | Pod/EndpointSlice events |
| 27 | Retry storm duplicates inference | Request IDs and GPU load |
| 28 | HTTP 200 but truncated body | Missing completion marker |

For each scenario: expected symptom, scope, first-failure time, source of evidence, likely layer, safe remediation, rollback, and proof of recovery. Simulate only in isolated staging; otherwise complete a tabletop exercise.

## Twelve production runbook rehearsals

Walk through RB-01 to RB-12 from the canonical chapter. Assign incident commander, SRE and application owners. Identify rollback commit/config, client communication, service SLO, abort criteria and follow-up action for each.

## Ten acceptance gates

| Gate | Required proof | Status |
|---|---|---|
| G1 | Approved staging context and architecture inventory | Pending |
| G2 | DNS/TCP/TLS timing baseline captured | Pending |
| G3 | First byte vs first token distinguished with event timestamps | Pending |
| G4 | Streaming protocol and clean completion validated | Pending |
| G5 | Client/LB/gateway/server timeout inventory complete | Pending |
| G6 | Buffering/idle timeout behavior reproduced safely | Pending |
| G7 | 499/502/503/504 diagnostic paths documented | Pending |
| G8 | Readiness, GPU queue and backpressure correlated | Pending |
| G9 | Active-stream rollout, cancellation and rollback validated | Pending |
| G10 | Evidence reviewed and all lab changes cleaned up | Pending |

**Pass:** each applicable gate has timestamped evidence and reviewer sign-off. Unexecuted or blocked gates cannot be marked passed.

## Evidence record

Capture UTC timestamp, cluster, namespace, controller/version, model/version, sanitized request ID, protocol, DNS/TCP/TLS/first-byte/TTFT/ITL/total timings, error class, stream completion marker, gateway/LB logs, readiness and saturation, policy change, rollback and reviewer.

**Next:** Chapter 9.7 — Production Networking and Recovery Capstone.
