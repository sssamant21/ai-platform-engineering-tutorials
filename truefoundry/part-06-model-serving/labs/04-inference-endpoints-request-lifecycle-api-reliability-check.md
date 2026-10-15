# Lab 6.4 — Inference Endpoints, Request Lifecycle and API Reliability Check

**Canonical:** [Chapter 6.4](../04-inference-endpoints-request-lifecycle-api-reliability/04-inference-endpoints-request-lifecycle-api-reliability.md)  
**Default:** **[SAFE-READ]**. Optional load, route and failure experiments are **[LAB-MUTATE]** only in an isolated approved nonproduction TrueFoundry workspace.

## 1. Objectives

Trace a model request from authorized client through gateway to runtime; validate API contract, authentication, TLS, readiness and golden output; characterize timeouts, retries, concurrency and rate limits; diagnose 16 failure scenarios and document rollback.

## 2. Preflight [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Record TrueFoundry workspace/version, cluster, namespace, endpoint hostname, model version, owner, approved dummy input and change scope. Do not expose a private endpoint publicly.

## 3. Map endpoint layers

| Layer | Observed | Evidence |
|---|---|---|
| Client DNS | | |
| TLS and SNI | | |
| Gateway/Ingress/HTTPRoute | | |
| Authentication and authorization | | |
| Kubernetes Service/targetPort | | |
| Ready EndpointSlices | | |
| Pod and runtime readiness | | |
| Model version and schema | | |

```bash
kubectl -n <namespace> describe service <service>
kubectl -n <namespace> get networkpolicies
kubectl -n <namespace> describe pod <pod>
```

Where Gateway API is installed:

```bash
kubectl -n <namespace> get gateways,httproutes
```

## 4. TLS and authorized smoke test [SAFE-READ]

```bash
openssl s_client -connect <approved-host>:443 -servername <approved-host> -showcerts
curl --connect-timeout 5 --max-time 30 -i https://<approved-host>/health
```

A health route may be intentionally protected or absent; use the documented application contract. Use an approved client/secret-injection mechanism to send a **non-sensitive golden inference request**, verifying status, schema, request ID, model version and expected result. Do not paste real bearer tokens into shared transcripts or shell history.

## 5. Request-path timing worksheet

| Metric | Baseline | Candidate | Notes |
|---|---|---|---|
| DNS/connect/TLS | | | |
| Gateway/auth | | | |
| Queue wait | | | |
| Preprocessing | | | |
| Model execution | | | |
| Response transfer | | | |
| P95/P99 end-to-end | | | |
| TTFT/stream completion (if used) | | | |
| Error/429/timeout rate | | | |

Correlate traces and logs using sanitized request IDs. Do not sum overlapping spans as independent latency.

## 6. Optional controlled API reliability test [LAB-MUTATE]

1. In an isolated nonproduction workspace, pin a known-good image and model version.
2. Set bounded client concurrency, request size, timeout and retry budget.
3. Send approved synthetic requests at gradually increasing, explicitly approved rates.
4. Measure throughput, P95/P99, queue depth, rejections and resource utilization.
5. Verify 429/503 behavior and whether clients honor backoff without retry storms.
6. For streaming endpoints, verify client cancellation and completion metrics.
7. Restore known-good TrueFoundry configuration and clean up lab-owned resources.

Do not run denial-of-service tests or uncontrolled traffic against shared gateways.

## 7. Failure tabletop

| ID | Scenario | First failed transition |
|---|---|---|
| E1 | DNS failure | Client → DNS |
| E2 | Expired/SAN-mismatched certificate | Client → TLS |
| E3 | 401 missing auth | Gateway → identity |
| E4 | 403 insufficient model scope | Identity → authorization |
| E5 | 404 route/path mismatch | Gateway → backend route |
| E6 | 413 oversized input | Request → body limit |
| E7 | 422 schema mismatch | Input → validation |
| E8 | 429 rate limit | Admission → quota |
| E9 | 502 upstream protocol error | Gateway → backend |
| E10 | 503 no ready endpoints | Service → Ready Pods |
| E11 | 504 deadline exceeded | Gateway → execution |
| E12 | Slow time-to-first-token | Queue → first output |
| E13 | Streaming disconnect after 200 | Runtime → client stream |
| E14 | Retry amplification | Client retries → admission |
| E15 | Wrong model version | Model reference → result |
| E16 | Cross-tenant inference access | Identity → model scope |

For each record lowest proven healthy layer, evidence, owner, minimal safe fix, rollback trigger and acceptance result.

## 8. Acceptance scorecard

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| G1 Exposure/auth boundaries | | |
| G2 API version/schema/error contract | | |
| G3 TLS and authorized path | | |
| G4 Readiness and golden inference | | |
| G5 Timeouts/cancellation | | |
| G6 Retries/limits/backpressure | | |
| G7 Tail latency/queue/errors/streaming | | |
| G8 Rollback and cleanup | | |

## 9. Evidence and handoff

```text
Date/time, owner, approval:
TrueFoundry version/workspace/deployment:
Endpoint host/path/exposure:
Image digest / model version:
DNS/TLS/auth results (no tokens):
Gateway/Service/EndpointSlice:
Golden inference and schema:
Latency budget, P95/P99 and TTFT:
Error code mix, retries, 429, queue depth:
Concurrency/rate-limit policy:
E1–E16 scenario classifications:
Lowest Proven Healthy Layer:
First Failed Transition:
Remediation and rollback owner:
Post-recovery authorized request:
Cleanup and Not run gates:
```

**Next:** Chapter 6.5 — Model Loading, Warmup, Health Probes and Readiness.
