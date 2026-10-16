# Chapter 006 — Transformer Inference Request Lifecycle

**Part 02 — LLM Architecture and Inference Internals | Chapter 006 of 100**

> **Status:** Written tutorial published. Vendor/version validation, live lab execution, security review and operational acceptance are pending. Examples use synthetic data.

## Learning objectives

Explain the decoder-only transformer inference path, distinguish tokenization, prefill and decode, identify where KV cache and batching affect memory and latency, trace streaming completion, and diagnose customer-visible inference regressions using approved read-only evidence.

## 1. Production request lifecycle

```text
Client request (synthetic prompt)
   │ HTTPS, auth, rate limits, request deadline
   ▼
Gateway / load balancer
   │ model routing, admission, retry policy, request ID
   ▼
Serving API (vLLM / Triton / Ray Serve / KServe)
   │ validation, tokenizer, context limits, queue
   ▼
Prefill phase (process input tokens)
   │ attention/MLP layers, initial KV cache
   ▼
Decode phase (one or more next tokens per step)
   │ KV cache reused, continuous batching where supported
   ▼
Detokenize → stream token events → final completion
   │ finish reason, usage, safety/application checks
   ▼
Client receives usable answer

Side channels: GPU/VRAM/DCGM, metrics, tracing, logs, audit
Control plane: GitOps / registry / platform operator / Kubernetes API
```

The precise pipeline depends on architecture, runtime, sampling algorithm and speculative decoding. This chapter uses a conventional autoregressive decoder-only model.

## 2. Step-by-step internals

### Step A — Request admission and tokenization

The gateway validates authentication, tenant, quotas, request size, model ID and deadline. The tokenizer converts prompt text into token IDs, adding model-specific special tokens. Input characters are not equal to tokens. Chat templates can increase token counts, and tokenizer/model revisions must be compatible.

A context limit generally constrains the combined input and generated tokens, subject to runtime/model-specific semantics. Validate the actual engine limits before accepting a request.

### Step B — Prefill

The transformer processes prompt tokens through attention and feed-forward layers, generating intermediate activations and initializing key/value attention state. Prefill is typically compute-intensive for large prompts; actual performance depends on model architecture, sequence length, hardware, kernels and batching.

**Operational impact:** large prompts, cold model loads and queuing can increase time to first token (TTFT). Do not attribute all TTFT regression to GPU compute; split queue wait, tokenization, prefill and network overhead.

### Step C — KV cache

During autoregressive decoding, the runtime stores attention keys and values for previously processed tokens, avoiding recomputation of all earlier attention projections. KV cache scales with resident sequences and their token lengths; exact size depends on layers, KV heads, head dimension, bytes per element, implementation and cache sharing.

Illustrative *unreplicated* KV-cache sizing for a conventional architecture:

```text
bytes_per_token ≈ 2 × num_layers × num_kv_heads × head_dim × bytes_per_element
cache_bytes ≈ bytes_per_token × resident_cached_tokens
```

This excludes allocator fragmentation, block rounding, prefix sharing, sliding-window behavior, compression, offloading and distributed sharding. Tensor parallelism can change per-GPU allocation. Use runtime-reported values for capacity decisions.

### Step D — Decode

The model predicts the next-token distribution, applies sampling/decoding rules, appends the chosen token and updates KV cache. A single autoregressive sequence typically advances token by token, while the GPU can batch many sequences. Runtime support for speculative decoding or other techniques may alter the flow.

Decode can be memory-bandwidth or compute constrained depending on batch size, model and hardware. The request may wait between steps because of scheduling or contention.

### Step E — Streaming and termination

In streaming mode, generated output is delivered as protocol events (often SSE). Distinguish:
- First HTTP byte vs first generated token.
- Server token generation vs client receipt.
- Normal completion vs length limit, cancellation, error or transport interruption.
- HTTP 200 vs an actually complete valid stream.

Define a customer-path synthetic completion check; a stream that ends without the expected protocol terminator must not count as successful simply because headers returned 200.

## 3. Performance metrics and units

| Metric | Definition | Dominant fault domains |
|---|---|---|
| Queue wait | accepted to scheduled inference | admission, concurrency, saturation |
| Tokenization time | request parsing to token IDs | CPU, templates, tokenizer |
| TTFT | request start to first *output token* observed | queue, prefill, cold start, gateway |
| TPOT | defined average time per subsequent output token | decode, batching, GPU |
| ITL | gap between successive delivered tokens | scheduling, network, decode |
| E2E latency | request start to valid final completion | entire path |
| Input tokens/s | prompt-processing throughput | prefill |
| Output tokens/s | generated-token throughput | decode |
| KV cache occupancy | active cache use vs available blocks | context/concurrency |
| GPU VRAM | model weights, KV cache, workspace, fragmentation | memory capacity |

**TPOT definition must be explicit.** A common request-level approximation is `(E2E - TTFT)/(output_tokens - 1)` when at least two output tokens exist; it includes post-first-token non-decode overhead and network effects. Vendor implementations may report different definitions. Never mix units or aggregate per-Pod p95s by averaging.

## 4. Example latency budget

A hypothetical synthetic request has:

| Component | Duration |
|---|---:|
| Gateway and admission | 80 ms |
| Queue wait | 120 ms |
| Tokenization | 30 ms |
| Prefill | 370 ms |
| Decode/stream remainder | 1,400 ms |
| **End-to-end** | **2,000 ms** |

Assuming these are sequential and first token arrives at end of prefill, illustrative TTFT = 600 ms. This simplification omits overlap, first decode step and network jitter; do not treat it as a measured trace.

## 5. Concurrency and continuous batching

Continuous batching can insert newly admitted requests and remove completed sequences between decode iterations, increasing GPU utilization. It does not eliminate finite KV cache, queue limits or model-loading overhead. Long prompts and output-heavy requests compete differently for compute and memory.

Consider **head-of-line blocking**, priority fairness, maximum concurrent sequences, maximum batched tokens, preemption, admission limits and per-tenant quotas. Retry storms can worsen queue pressure; exponential backoff and bounded retries belong in the client contract.

## 6. Safe diagnostic workflow

**[SAFE-READ]** Only on an approved Kubernetes context:

```bash
kubectl config current-context
kubectl get pods,deploy,svc,endpointslices -A -o wide
kubectl get nodes -o wide
kubectl get resourcequota,hpa,pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

For a verified namespace and deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Correlate with **approved** gateway traces, runtime metrics, synthetic TTFT/ITL and GPU dashboards. Metric names vary by engine and release; do not assume custom names are native to vLLM. Redact prompt bodies and secrets from logs.

## 7. Triage decision tree

```text
Inference degraded?
 ├─ No first token → auth/routing, queue, tokenization, prefill, model load
 ├─ High TTFT only → queue/prefill/long-prompt cohort/cold start
 ├─ High TPOT/ITL → decode contention, batching, KV cache, network
 ├─ CUDA OOM → weights, KV, context, batch/concurrency
 ├─ Context rejection → tokenizer/chat template/limit mismatch
 ├─ 200 but incomplete SSE → proxy timeout, cancellation, server crash
 ├─ Throughput falls as concurrency rises → saturation or preemption
 ├─ One model revision regresses → image/model/tokenizer/runtime diff
 └─ Metrics missing → exporter/scrape, independent synthetic
```

## 8. Sixteen production failure scenarios

| ID | Symptom | First check |
|---|---|---|
| F01 | wrong model routed | gateway route and revision |
| F02 | tokenizer mismatch | model/tokenizer provenance |
| F03 | context window exceeded | input/output token budget |
| F04 | cold model startup | artifact load and readiness |
| F05 | high queue time | admitted vs running requests |
| F06 | high prefill latency | input length cohorts |
| F07 | high decode latency | output tokens/s and GPU |
| F08 | KV cache exhaustion | cache occupancy/preemption |
| F09 | CUDA OOM | VRAM and concurrency |
| F10 | GPU throttling | DCGM, clocks, thermal |
| F11 | unfair batching | priority/tenant cohorts |
| F12 | retry amplification | client/gateway retry counts |
| F13 | truncated SSE | completion protocol probe |
| F14 | gateway 504 | deadline alignment |
| F15 | false success HTTP 200 | good-event contract |
| F16 | missing traces/metrics | telemetry freshness |

## 9. Ten operational runbooks

1. Validate customer-path synthetic request and complete stream.
2. Confirm model, tokenizer, runtime and image versions.
3. Triage admission/auth and context validation.
4. Isolate queue wait from prefill latency.
5. Investigate TTFT by prompt-length cohort.
6. Investigate TPOT/ITL and decode contention.
7. Check KV-cache occupancy and preemption.
8. Triage CUDA OOM and GPU hardware metrics.
9. Investigate streaming timeout and completion.
10. Contain rollout, validate rollback ownership and verify recovery.

Each runbook must include owner, read-only evidence, safe containment, authorized change, rollback and customer-path recovery.

## 10. Six game days

1. Long-prompt synthetic TTFT regression (staging if approved).
2. Output-heavy decode slowdown (tabletop).
3. KV cache saturation under concurrency (tabletop).
4. Tokenizer/model mismatch after rollout (tabletop).
5. SSE stream terminates after HTTP 200 (tabletop).
6. Missing runtime metrics with synthetic failure (tabletop).

## 11. Twelve acceptance gates

G01 trace request path; G02 tokenization/template contract; G03 prefill vs decode; G04 KV-cache sizing assumptions; G05 TTFT/TPOT units; G06 complete-stream success; G07 cohort segmentation; G08 bounded concurrency and retries; G09 safe Kubernetes inspection; G10 failure triage; G11 runbook and rollback owner; G12 reviewer signoff.

All gates remain pending until evidenced. See [companion lab](../labs/006-transformer-inference-request-lifecycle-check.md).

## Next

**Chapter 007 — Tokenization, Context Windows and Generation Parameters.**
