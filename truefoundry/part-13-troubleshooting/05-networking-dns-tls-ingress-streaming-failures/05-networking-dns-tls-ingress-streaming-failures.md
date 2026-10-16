# Chapter 13.5 — Networking, DNS, TLS, Ingress and Streaming Failures

**Part 13 — Troubleshooting | TrueFoundry SRE canonical tutorial**

## Objectives and safety

Trace an inference request from client to TrueFoundry gateway/ingress, Kubernetes Service and EndpointSlice, vLLM Pod, and streaming response. Diagnose DNS, TLS, ingress routing, network policies, connection reuse, HTTP timeouts, SSE streaming, retries and deployment drains while preserving customer SLOs and security.

**[SAFE-READ]** inspect authorized routes, Service, EndpointSlice, Events, certificate metadata and sanitized metrics. **[LAB-MUTATE]** disposable staging with synthetic traffic. **[CHANGE-CONTROL]** production DNS, certificates, ingress, gateways, network policies, service routing and drain settings. Never use `curl -k` to conceal a TLS failure, dump private keys, or log customer prompts/completions.

## 1. End-to-end request path

```text
Client / SDK / proxy
   │ DNS + TLS + HTTP keepalive
   ▼
External LB / WAF / ingress
   │ gateway route, auth, rate limit, timeout
   ▼
TrueFoundry deployment endpoint
   │ Kubernetes Service → EndpointSlice → ready Pod IP
   ▼
vLLM HTTP server / model runtime
   │ request queue / token generation
   ▼
Streaming SSE chunks → gateway → client
```

Distinguish **control-plane connectivity** (agent/platform reconciliation) from **data-plane inference**. A healthy control plane does not guarantee customer traffic works, and an inaccessible dashboard does not prove serving failure.

## 2. Incident triage and blast radius

Capture UTC start, affected model/revision, region/zone, client SDK, HTTP status, connection failures, p95 TTFT/TPOT, streaming completion, gateway 4xx/5xx, backend 5xx, retries and recent releases. Correlate with DNS, certificate, ingress and network policy changes. Record request IDs only if permitted and sanitized.

## 3. Read-only Kubernetes network inspection

```bash
kubectl config current-context
kubectl -n MODEL_NAMESPACE get svc -o wide
kubectl -n MODEL_NAMESPACE get endpointslices -o wide
kubectl -n MODEL_NAMESPACE get pods -o wide
kubectl -n MODEL_NAMESPACE get ingress
kubectl -n MODEL_NAMESPACE get networkpolicy
kubectl -n MODEL_NAMESPACE get events --sort-by=.lastTimestamp
```

Not every deployment uses Kubernetes Ingress; discover whether it uses Gateway API, service mesh, cloud load balancer or TrueFoundry-managed gateway. Check actual controller ownership before changing anything.

```bash
kubectl -n MODEL_NAMESPACE describe svc MODEL_SERVICE
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE get endpointslices -l kubernetes.io/service-name=MODEL_SERVICE -o wide
```

An empty EndpointSlice can reflect selector mismatch, failed readiness or endpoints not yet propagated. An endpoint being present does not guarantee application-layer inference success.

## 4. DNS troubleshooting

Separate client public DNS, cluster DNS, upstream registry/model storage DNS and TrueFoundry control-plane DNS. Compare expected FQDN, resolved IP, TTL, split-horizon configuration and resolver errors. Use approved `nslookup`/`dig` from the relevant network vantage point; do not assume an external laptop reproduces Pod DNS behavior. Check CoreDNS health and network policies if resolution fails inside cluster.

## 5. TLS and certificate failures

Identify termination point(s): client→LB, LB→gateway, gateway→backend and optional service-mesh mTLS. Check hostname/SAN, certificate chain, validity dates, issuer, trust store, SNI and clock skew. Certificate expiry can manifest as connection failures, gateway 502/503 or agent connectivity errors. Rotate through approved certificate management; avoid disabling verification.

Example metadata-only test from an approved environment:

```bash
openssl s_client -connect INFERENCE_HOST:443 -servername INFERENCE_HOST -brief </dev/null
```

Never put authentication headers or tokens into shell history or ticket attachments.

## 6. HTTP errors and timeout ownership

| Symptom | Likely boundary | Verification |
|---|---|---|
| 401/403 | gateway authentication/authorization | approved identity and policy |
| 404 | hostname/path/model route | gateway route and deployed model |
| 408/499 | client cancellation or timeout | client and gateway timestamps |
| 429 | gateway/model rate limiting | quotas, concurrency and retry policy |
| 502 | upstream failure/reset | gateway logs, endpoint health |
| 503 | no healthy upstream/capacity | EndpointSlice, readiness, queue |
| 504 | upstream response timeout | TTFT, model load, gateway timeout |
| TCP reset | drain, LB, proxy or Pod termination | connection lifetime and events |

Status interpretation varies by gateway/controller. Confirm actual logs and traces rather than inferring root cause from one code.

## 7. Streaming SSE and long-running inference

For OpenAI-compatible streaming, response delivery often uses Server-Sent Events; verify the actual deployed protocol. Inspect HTTP version, proxy buffering, idle/read timeouts, keepalive, max connection duration, client cancellation, backpressure and stream completion semantics. A `200` response with incomplete token stream is still a user-visible failure.

Use an approved synthetic request and redact payloads:

```bash
curl --fail-with-body --no-buffer --connect-timeout 5 --max-time 120 \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer ${TEST_TOKEN}" \
  --data '{"model":"SYNTHETIC_MODEL","messages":[{"role":"user","content":"Say hello"}],"stream":true}' \
  "https://INFERENCE_HOST/v1/chat/completions"
```

Use a short-lived staging token supplied via a secure environment; do not paste commands with expanded tokens into shared logs. `--max-time` caps the entire request; choose a bound consistent with approved synthetic workload. The path is an **example** for compatible APIs, not a guaranteed TrueFoundry route.

## 8. NetworkPolicy and service mesh

Inspect ingress/egress policies, namespace labels, Pod labels, DNS egress, gateway-to-model reachability, model storage and platform agent paths. Kubernetes NetworkPolicy enforcement depends on the installed CNI. Service mesh may add mTLS, sidecars, retries and independent timeout budgets. Avoid blanket allow-all as a production fix; use scoped least-privilege changes with review.

## 9. Rollouts, drains and streaming safety

Correlate stream drops with Pod termination, EndpointSlice removal, preStop hook, termination grace, PDB, gateway connection draining and in-flight requests. A Pod that passes readiness at startup may be removed before long-running streams finish. Test with synthetic streams under approved staging conditions and observe graceful termination before promoting changes.

## 10. Twelve operational runbooks

1. External DNS resolution failure.
2. In-cluster CoreDNS failure.
3. TLS expiry, hostname or trust-chain failure.
4. Ingress/Gateway route mismatch.
5. Service selector/EndpointSlice empty.
6. Pod readiness and backend reachability.
7. NetworkPolicy/CNI isolation failure.
8. Gateway 401/403/429 authentication or rate limit.
9. Gateway 502/503/504 upstream errors.
10. SSE buffering, idle timeout and client disconnect.
11. Deployment drain and long-running stream interruption.
12. Regional LB/zone outage and customer-path recovery.

Each runbook records symptoms, UTC evidence, owner, read-only diagnostics, change-controlled mitigation, rollback, independent synthetic recovery and escalation.

## 11. Thirty-six troubleshooting scenarios

| # | Scenario | # | Scenario |
|---|---|---|---|
| 01 | Public DNS NXDOMAIN | 19 | Gateway 502 |
| 02 | Stale DNS cache | 20 | Gateway 503 |
| 03 | Split-horizon DNS | 21 | Gateway 504 |
| 04 | CoreDNS unavailable | 22 | Rate-limit 429 |
| 05 | DNS egress blocked | 23 | Unauthorized 401 |
| 06 | TLS certificate expired | 24 | Forbidden 403 |
| 07 | Hostname/SAN mismatch | 25 | Incorrect route 404 |
| 08 | Untrusted CA | 26 | Client timeout |
| 09 | SNI mismatch | 27 | Proxy idle timeout |
| 10 | mTLS identity rejected | 28 | SSE response buffered |
| 11 | Ingress controller unhealthy | 29 | Stream truncation |
| 12 | Gateway API route rejected | 30 | Client cancellation |
| 13 | LB target unhealthy | 31 | Retry storm |
| 14 | Service selector mismatch | 32 | Pod drain interrupts stream |
| 15 | EndpointSlice empty | 33 | Termination grace too short |
| 16 | Readiness fails | 34 | Service mesh retry amplification |
| 17 | NetworkPolicy ingress denied | 35 | Zone LB failure |
| 18 | NetworkPolicy egress denied | 36 | Missing request trace |

## 12. Twelve acceptance gates

G1 approved scope; G2 customer-path impact measured; G3 DNS vantage points checked; G4 TLS chain/SNI validated; G5 gateway/ingress routing traced; G6 Service/EndpointSlice/readiness verified; G7 network policy/mTLS checked; G8 streaming completion measured; G9 timeout/retry budgets reviewed; G10 drain/rollback tested; G11 recovery SLO verified; G12 sanitized evidence and reviewer sign-off.

Complete the [companion hands-on lab](../labs/05-networking-dns-tls-ingress-streaming-failures-check.md). Formal review and live execution pending.

**Next:** Chapter 13.6 — NVIDIA Drivers, CUDA, GPU OOM and vLLM Performance.
