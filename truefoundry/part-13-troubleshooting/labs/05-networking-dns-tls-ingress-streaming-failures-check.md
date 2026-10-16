# Chapter 13.5 — Hands-on Lab: Networking, DNS, TLS, Ingress and Streaming Failures

**Part 13 | Approved staging | Live execution pending**

## Mission and safety

Diagnose customer-facing TrueFoundry inference failures across DNS, TLS, gateway, Service/EndpointSlice, NetworkPolicy and SSE streaming. [SAFE-READ] inspect; [LAB-MUTATE] isolated staging; [CHANGE-CONTROL] production DNS, cert, gateway, routing or policy changes. Use synthetic prompts and short-lived staging credentials. Never disable TLS verification or leak tokens.

## Lab 0 — Inventory and baseline

```bash
kubectl config current-context
kubectl -n MODEL_NAMESPACE get svc,endpointslices,pods -o wide
kubectl -n MODEL_NAMESPACE get ingress,networkpolicy
kubectl -n MODEL_NAMESPACE get events --sort-by=.lastTimestamp
```

Record ingress/Gateway API controller, hostnames, TLS termination, model route, TrueFoundry revision, vLLM version, namespace, synthetic streaming baseline, p95 TTFT/TPOT, completion ratio and change approval.

## Lab 1 — DNS and TLS

From an approved network vantage point, resolve client-facing and in-cluster FQDNs, inspect TTL and DNS failure mode, then inspect TLS chain, SNI and validity. Compare expected with actual; do not assume the same DNS path inside and outside Kubernetes.

```bash
openssl s_client -connect INFERENCE_HOST:443 -servername INFERENCE_HOST -brief </dev/null
```

## Lab 2 — Service and gateway

Inspect Service selectors, EndpointSlice ready addresses, Pod readiness, ingress/gateway routes and network policies. Correlate synthetic request HTTP status with gateway logs and model logs. Use approved RBAC and sanitize evidence.

## Lab 3 — Synthetic streaming

```bash
curl --no-buffer --connect-timeout 5 --max-time 120 \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer ${TEST_TOKEN}" \
  --data '{"model":"SYNTHETIC_MODEL","messages":[{"role":"user","content":"Say hello"}],"stream":true}' \
  "https://INFERENCE_HOST/v1/chat/completions"
```

The API path is illustrative; verify actual TrueFoundry route. Measure time to first event, stream completion, idle gaps, client cancellation and final termination marker if protocol supplies one. Never attach token-bearing terminal output.

## Lab 4 — Five game days

1. DNS resolution mismatch in disposable staging.
2. TLS certificate trust/hostname issue using test certificates only.
3. Service selector/EndpointSlice mismatch in disposable namespace.
4. Gateway timeout or SSE buffering using synthetic traffic.
5. Rolling update and streaming drain validation with rollback.

Tabletop unsafe cases. Record UTC timeline, blast radius, request outcome, expected/actual, mitigation, rollback, SLO and reviewer.

## Thirty-six scenarios

01 public DNS NXDOMAIN; 02 stale DNS; 03 split-horizon; 04 CoreDNS; 05 DNS egress; 06 expired TLS; 07 SAN mismatch; 08 CA trust; 09 SNI; 10 mTLS; 11 ingress unhealthy; 12 Gateway route rejected; 13 LB target; 14 selector mismatch; 15 empty EndpointSlice; 16 readiness; 17 ingress policy; 18 egress policy; 19 502; 20 503; 21 504; 22 429; 23 401; 24 403; 25 404; 26 client timeout; 27 proxy idle timeout; 28 SSE buffering; 29 stream truncation; 30 cancellation; 31 retries; 32 drain; 33 grace period; 34 mesh retry; 35 zone LB; 36 missing trace.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Authorized staging scope | Pending |
| G2 | Customer-path SLO baseline | Pending |
| G3 | DNS vantage points verified | Pending |
| G4 | TLS chain, SNI and expiry checked | Pending |
| G5 | Gateway route traced | Pending |
| G6 | Service/EndpointSlice/readiness validated | Pending |
| G7 | NetworkPolicy/mTLS reviewed | Pending |
| G8 | SSE stream completion measured | Pending |
| G9 | Timeout and retry budgets reviewed | Pending |
| G10 | Drain and rollback rehearsed | Pending |
| G11 | Post-recovery SLO validated | Pending |
| G12 | Sanitized evidence and reviewer sign-off | Pending |

## Evidence record

```text
Exercise ID / UTC start-end:
Environment, model, TrueFoundry revision:
Client and Pod DNS observations:
TLS chain, hostname, SNI and expiry:
Gateway/controller and route:
Service, EndpointSlice and readiness:
NetworkPolicy/mTLS:
Synthetic status, TTFT/TPOT, stream completion:
Timeouts, retries and client behavior:
Expected/actual, mitigation and rollback:
Executed or tabletop:
Owner/reviewer:
```

## Cleanup

Restore disposable staging routes, policies and test certs, close test alerts, confirm DNS/TLS and streaming baseline, and verify no unexpected costs. Publication does not establish lab completion.

**Next:** Chapter 13.6 — NVIDIA Drivers, CUDA, GPU OOM and vLLM Performance.
