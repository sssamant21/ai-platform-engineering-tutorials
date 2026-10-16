# Chapter 001 — Introduction to LLMOps and AI Inference Site Reliability Engineering

**Part 01 — AI/LLM SRE Fundamentals | Chapter 001 of 100**

## Learning objectives

By the end, an engineer should be able to trace a production LLM inference request, identify reliability ownership boundaries, select customer-centered service level indicators, inspect Kubernetes/GPU health safely, distinguish a platform outage from a model-quality incident, and write an evidence-backed first-response incident note.

**Status:** Published tutorial; companion lab is a runnable *plan*. No live execution, vendor validation, technical review or production acceptance is implied.

## 1. What is LLMOps?

LLMOps is the set of engineering practices used to build, package, deploy, evaluate, observe, secure, scale and maintain large-language-model applications and their inference infrastructure. AI inference SRE applies reliability engineering: service objectives, failure budgets, operational automation, capacity, on-call, incident command, recovery and continuous improvement.

Unlike conventional stateless web APIs, LLM inference reliability includes token-level latency, output correctness, streaming completion, model/adapter compatibility, KV cache pressure, GPU memory and model warmup.

### Responsibilities and ownership

| Area | LLMOps / ML platform | SRE / infrastructure | Application / security |
|---|---|---|---|
| Model artifacts | version, evaluation, licensing, tokenizer | immutable delivery, availability | output contract, compliance |
| Inference runtime | vLLM/Triton/Ray/KServe config | scheduling, rollout, recovery | request semantics |
| GPU | precision/parallelism choices | drivers, device plugin, quotas, DCGM | budget approval |
| API | model routes, versioning | gateway, TLS, retries, rate limits | authentication, tenant isolation |
| Quality | eval datasets, regressions | alert integration, rollback | business acceptance |
| Operations | release playbooks | SLOs, incidents, DR, paging | data governance |

The same organization may combine these roles, but explicit decision ownership is required.

## 2. Reference architecture

```text
Approved client (synthetic for labs)
  │ TLS, auth, tenant policy, quota, rate limit
  ▼
API Gateway / Ingress / WAF
  │ request id, 401/403/429/5xx, SSE timeout
  ▼
Kubernetes Service / EndpointSlice
  │ readiness, network policy, routing
  ▼
vLLM / Triton / Ray Serve / KServe inference workload
  │ tokenizer → prefill → KV cache → decode → SSE tokens
  ▼
GPU node / NVIDIA driver / container runtime / device plugin
  │ VRAM, compute, XID/ECC, CUDA, GPU scheduling
  ▼
model registry/object store + approved workload identity
  │
  └──── telemetry: Prometheus, OpenTelemetry, logs, traces, DCGM
                        │
                SLO alerts / on-call / runbooks
```

A TrueFoundry control plane may reconcile deployments, but the serving path and control path are separate. A healthy dashboard or agent does not prove customer inference works.

## 3. The inference request lifecycle

1. Client sends authenticated request with model, messages, parameters and streaming preference.
2. Gateway enforces TLS, identity, tenant, token/rate budget and payload constraints.
3. Request reaches a ready inference endpoint, or queues if the server is saturated.
4. Tokenization transforms input text to token IDs; validate max context and tokenizer version.
5. **Prefill** processes prompt tokens and constructs attention KV cache.
6. **Decode** generates tokens iteratively; batching and memory determine throughput.
7. Streaming returns tokens as server-sent events (SSE), where applicable.
8. Client validates protocol completion, correctness contract and final usage.
9. Telemetry records sanitized request-level, runtime, GPU and infrastructure signals.

HTTP 200 alone is not a success signal: a stream can terminate early or produce invalid output.

## 4. Reliability metrics: define units and cohorts

| Metric | Definition | Typical investigation |
|---|---|---|
| TTFT | request start to first output token | queue, prefill, gateway |
| TPOT | time per generated output token; define calculation | decode, GPU, batching |
| ITL | inter-token latency between output tokens | jitter, streaming |
| E2E latency | start to valid completion | all stages |
| Complete-stream rate | streams with valid termination / eligible streams | proxy, disconnect |
| Throughput | requests/s, input/output tokens/s | capacity |
| Error rate | eligible failed requests / eligible requests | 429, 5xx, timeouts |
| GPU utilization/VRAM | hardware activity and memory pressure | scheduling, cache |
| KV cache pressure | allocated/cache blocks and evictions, version-specific | context/concurrency |
| Cost per 1k successful output tokens | attributable cost / successful output tokens × 1000 | FinOps |

Break out streaming/non-streaming, model revision, region, prompt length, output length, tenant class and concurrency. Never publish raw prompts, credentials or PHI in metric labels.

## 5. SLI, SLO and error-budget foundations

**SLI:** a measured indicator of service behavior, such as valid complete inference streams. **SLO:** a service-owner-approved target for that SLI over a defined window. **Error budget:** allowed failure fraction (1 − SLO target) over that window.

Example **only**, not an approved target: 99.9% eligible request availability over 30 days allows 0.1% failures. The budget is request-weighted; this is not automatically 43.2 minutes of downtime unless request arrival is uniform and the objective is time-based.

```text
error_ratio = failed_eligible_requests / all_eligible_requests
error_budget_fraction = 1 - SLO_target
burn_rate = error_ratio / error_budget_fraction
```

Define treatment of client cancellation, intentional 401/403, rate-limit 429, invalid prompts and synthetic tests. Track telemetry freshness; missing measurements must not count as zero failures.

## 6. First response to inference slowness

```text
Customer SLO degraded?
 ├─ 401/403 → gateway auth, tenant policy, token expiry
 ├─ 429 → client quota or model saturation? distinguish
 ├─ 502/503/504 → gateway, Service, endpoints, readiness, timeout
 ├─ high TTFT → queue, prefill, cold start, model download
 ├─ high TPOT → decode, GPU throttling, batching, KV cache
 ├─ truncated SSE → proxy buffering, read timeout, draining
 ├─ Pending GPU Pod → allocatable, taints, quota, autoscaler
 ├─ CUDA OOM → VRAM, model precision, context, concurrency
 ├─ quality regression → tokenizer/model/config drift, evaluation
 └─ no telemetry → scrape, exporter, logging and synthetic probe
```

Do not blindly increase GPU replicas or client retries: quota limits, cold starts and retry storms can worsen incidents.

## 7. Safe baseline inspection

**[SAFE-READ]** Only on an approved context:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get deployments,svc,endpointslices -A
kubectl get resourcequota,hpa,pdb -A
kubectl get events -A --sort-by=.lastTimestamp
```

For a selected approved namespace:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Redact logs before sharing; these commands are not guaranteed safe for every access policy. Use approved GPU/DCGM dashboards for hardware inspection. GPU memory can be fully occupied even if utilization is low; correlate queue, tokens/s, VRAM, model warmup and error rates.

## 8. Security and cost are reliability requirements

Apply least-privilege workload identity, namespace RBAC, TLS, gateway auth, tenant isolation, model artifact ACLs, supply-chain scanning, immutable image/model versions, secrets redaction and audit. Enforce input/output limits and rate/token budgets to reduce denial-of-wallet. Treat prompt injection and data exfiltration as application security risks requiring separate controls.

GPU capacity planning must include warmup, rollouts, N-1, AZ quota, model VRAM, KV cache and idle cost. Track cost per successful output token rather than GPU utilization alone.

## 9. Twelve production failure scenarios

| ID | Symptom | First evidence | Response |
|---|---|---|---|
| F01 | GPU Pod Pending | scheduling events, quota, taints | confirm allocatable and placement |
| F02 | CUDA runtime missing | Pod logs, image/driver compatibility | validate toolkit/device plugin |
| F03 | CUDA OOM | container and runtime logs, VRAM | cap concurrency/context; rollback |
| F04 | Model download denied | object-store IAM, egress, events | restore approved access |
| F05 | Gateway 429 | rate limit, queue, client load | separate quota vs saturation |
| F06 | Gateway 503 | EndpointSlice, readiness | restore healthy endpoints |
| F07 | High TTFT | prompt cohort, queue, warmup | isolate prefill bottleneck |
| F08 | High TPOT | decode metrics, GPU throttle | inspect batch and GPU |
| F09 | SSE truncation | complete-stream probe, gateway | correct timeout/drain |
| F10 | Quality regression | model/tokenizer diff, eval | stop promotion and revert |
| F11 | Missing metrics | scrape/target freshness | restore observability |
| F12 | GPU cost spike | GPU-hours, tokens, retries | cap spend and rightsizing |

## 10. Eight operational runbooks

1. Validate customer-path synthetic inference.
2. Identify release/model/tokenizer/driver revision.
3. Triage gateway errors and SSE termination.
4. Triage Kubernetes scheduling and GPU allocatable.
5. Triage CUDA/runtime/OOM.
6. Triage TTFT/TPOT and KV cache.
7. Verify metrics, alerts and incident escalation.
8. Roll back through authoritative deployment owner and document RCA/COA.

## 11. Six introductory game days

1. Synthetic 429 versus queue saturation.
2. Pending GPU Pod due to quota (tabletop).
3. CUDA mismatch and vLLM startup (tabletop).
4. High TTFT under prefill-heavy synthetic load (approved staging only).
5. Truncated SSE from gateway timeout (tabletop).
6. Missing Prometheus/DCGM data and alert routing (tabletop).

All exercises require UTC timeline, approved scope, expected/actual behavior, abort thresholds, cost ceiling, cleanup and reviewer. Do not inject faults into production.

## 12. Ten acceptance gates

G01 explain LLMOps/SRE boundaries; G02 trace full request path; G03 classify TTFT/TPOT/E2E; G04 define eligible request SLI; G05 identify gateway and SSE failure; G06 inspect Kubernetes/GPU safely; G07 distinguish CUDA/driver/model issues; G08 document security and cost guardrails; G09 complete incident triage worksheet; G10 reviewer signs off staging evidence.

See [companion lab](../labs/001-introduction-to-llmops-ai-inference-sre-check.md). All gates are **pending** until reviewed.

## Next

**Chapter 002 — Production LLM Service Architecture and SRE Responsibilities.**
