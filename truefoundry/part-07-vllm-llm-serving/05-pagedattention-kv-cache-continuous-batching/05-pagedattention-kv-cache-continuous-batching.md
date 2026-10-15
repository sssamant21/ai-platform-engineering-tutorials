# 7.5 — PagedAttention, KV Cache and Continuous Batching

**Part:** 07 — vLLM & LLM Serving · **Chapter:** 5 of 7  
**Audience:** TrueFoundry platform engineers, GPU SREs, inference performance engineers  
**Safety:** **[SAFE-READ]** for observation, **[LAB-MUTATE]** only in approved isolated nonproduction, **[CHANGE-CONTROL]** for production changes.

## 1. Learning objectives

Explain why autoregressive LLM inference needs a KV cache; describe PagedAttention and block-based cache allocation; understand continuous batching, prefill and decode scheduling; estimate capacity and admission limits; distinguish throughput, TTFT and inter-token latency; identify memory pressure and preemption; and recover an overloaded vLLM deployment safely.

**Prerequisites:** Chapters 7.1–7.4; an approved TrueFoundry deployment; Kubernetes read access; access to documented GPU and vLLM metrics. Never use sensitive prompts, unbounded load tests or production fault injection.

## 2. Reference serving flow

```text
Authorized client → TLS/auth gateway → Kubernetes Service
                              ↓
                       vLLM API frontend
                              ↓
                   request validation/tokenizer
                              ↓
                 scheduler + admission / waiting queue
                              ↓
                  prefill and decode scheduling
                              ↓
                 model executor on allocated GPU(s)
                              ↓
            attention layers → block-managed KV cache
                              ↓
                  tokens / stream / completion
                              ↓
        request metrics + queue/KV/GPU telemetry + traces
```

TrueFoundry owns the supported deployment workflow; Kubernetes provides placement and service routing; vLLM owns model execution and internal scheduling. Do not patch managed Pods directly when a TrueFoundry/GitOps controller is the source of truth.

## 3. Why KV caching matters

For autoregressive transformers, each new output token attends to earlier context. Without caching, the engine would repeatedly compute previous attention keys and values. The **KV cache** stores key/value tensors for reusable tokens and reduces repeated computation.

KV cache is separate from model weights, activations, CUDA context and other runtime buffers. It grows with the amount of **live** context being served and therefore with prompt length, generated output and concurrent requests. Memory is freed or reused as requests complete or are cancelled, subject to the engine's allocator and cache policy.

### Simplified capacity formula

For a decoder-only model:

```text
KV bytes ≈ 2 × number_of_layers × KV_heads × head_dimension
         × live_token_count × bytes_per_KV_element
```

Example for **illustration only**: 32 layers, 8 KV heads, head dimension 128, BF16 KV values (2 bytes), and 8,192 live tokens:

```text
2 × 32 × 8 × 128 × 8192 × 2 = 1,073,741,824 bytes ≈ 1 GiB
```

This is a logical unsharded estimate, not a guaranteed per-GPU allocation. Actual capacity depends on model architecture, grouped-query attention, tensor parallelism, block rounding, prefix reuse, sliding-window attention, KV precision and engine version.

## 4. PagedAttention and block allocation

PagedAttention is an attention/memory-management approach that lets the runtime store KV tensors in **non-contiguous physical blocks** while maintaining a logical token-to-block mapping. It can reduce memory fragmentation and enable more flexible KV sharing/reuse compared with rigid contiguous allocations.

Conceptually:

```text
Request A logical blocks: A0 → A1 → A2
                             │    │    │
Physical KV pool:       [B7][B2][B9][free][B4]...
Request B logical blocks: B0 → B1
                             │    │
Physical KV pool:           [B4][B2] ... (illustrative)
```

Physical block sharing is possible only where the engine supports safe reuse and reference accounting; the sketch is **not** a statement that unrelated requests can arbitrarily share blocks. Cache block size, scheduler implementation and prefix-caching behavior vary by vLLM release.

**PagedAttention does not eliminate VRAM limits.** If model weights and runtime buffers leave insufficient space for KV blocks, the engine may reject requests, preempt work, recompute or fail according to its version and configuration.

## 5. Prefill and decode

| Phase | Work | Primary signal | Pressure |
|---|---|---|---|
| Prefill | Process input/prompt tokens and build KV | Time to first token (TTFT) | Prompt length, compute, queue |
| Decode | Generate next tokens using KV cache | Inter-token latency (ITL) | Active sequences, memory bandwidth, KV capacity |
| Queue/admission | Wait for scheduling capacity | Queue wait / waiting requests | Concurrency and limits |
| Completion | Deliver final response or end stream | End-to-end latency / stream completion | Timeouts, cancellation, network |

Long-context prefill can delay decode steps if scheduling is not tuned for the workload. Features such as chunked prefill, prefix caching and preemption differ across engine versions and must be verified against the installed release.

## 6. Continuous batching

Traditional static batching waits for a fixed group of requests to complete. **Continuous batching** allows the scheduler to admit eligible new requests between execution iterations while active requests continue generating tokens.

Potential benefits:
- Higher accelerator utilization for variable-length requests.
- Better aggregate token throughput.
- Less idle time as individual requests finish.

Tradeoffs:
- Higher concurrency can increase queueing, per-token latency and KV occupancy.
- Large prefill batches can increase TTFT for competing requests.
- Saturated memory may cause preemption/recomputation or request failure.
- Client retry storms can amplify load even when the GPU is healthy.

**Do not equate maximum throughput with best user experience.** Set explicit latency, throughput, fairness and admission objectives.

## 7. Request scheduling and capacity boundaries

Capacity is governed by multiple independent limits:

1. **Model and runtime fit:** weights, CUDA context and temporary buffers.
2. **KV cache:** number of active tokens and allocated blocks.
3. **Sequence length:** approved model context and engine `max_model_len` equivalent.
4. **Scheduler budget:** running sequences, batched tokens and waiting queue.
5. **Client/API limits:** prompt length, output tokens, rate quotas, timeouts and cancellation.
6. **GPU topology:** tensor parallelism, memory per rank and collective overhead.

Do not blindly increase concurrency or GPU-memory utilization settings. A larger scheduler budget can improve throughput until memory or tail latency becomes the bottleneck.

## 8. Example capacity worksheet

These are **hypothetical numbers**, not recommended vLLM defaults.

| Variable | Example | Source in a real deployment |
|---|---:|---|
| GPU VRAM | 48 GiB | Device telemetry |
| Model weights and runtime buffers | 30 GiB | Measured steady state |
| Reserved operational headroom | 4 GiB | Peak load / policy |
| Remaining theoretical KV budget | 14 GiB | 48 − 30 − 4 |
| Approximate logical KV per live token | 128 KiB | Model metadata/formula |
| Theoretical live-token ceiling | 114,688 | 14 GiB ÷ 128 KiB |
| Safe admission limit | **Not derivable from formula alone** | Load test and engine metrics |

This example omits allocator granularity, per-rank sharding, cache reuse, transient peaks and engine-specific memory reservation. The **safe** token/concurrency budget must be measured in a controlled environment.

## 9. SLO and performance metrics

| Metric | Why it matters | Diagnostic use |
|---|---|---|
| TTFT P50/P95/P99 | Responsiveness to first token | Queue + prefill |
| ITL / time per output token | Streaming smoothness | Decode saturation |
| End-to-end P95/P99 | Client experience | Gateway + runtime |
| Prompt tokens/s | Prefill throughput | Prompt workload |
| Output tokens/s | Decode throughput | Generation workload |
| Running requests | Active concurrency | Scheduler load |
| Waiting requests | Admission pressure | Queue buildup |
| KV-cache utilization | Memory occupancy | Preemption risk |
| Preemptions/recomputations | Cache pressure, if exposed | Efficiency loss |
| GPU VRAM and utilization | Hardware capacity | Correlate with queue |
| 429/5xx/timeout/stream failures | Reliability | Saturation and overload |

Metric names/labels change across vLLM releases. Discover actual available series in the authorized `/metrics` endpoint or platform observability before building alerts. Do not store raw prompts, tokens or user identifiers in metric labels.

## 10. Read-only production diagnostics [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> logs <pod> --tail=200
```

Where installed and authorized:

```bash
kubectl -n <namespace> exec <pod> -c <container> -- nvidia-smi
```

Use approved dashboards for KV-cache occupancy, queue, token throughput, preemption, TTFT/ITL and GPU VRAM. Missing metrics are an observability gap, **not** evidence of healthy capacity. Do not modify production concurrency or run stress tests as part of diagnosis.

## 11. Bounded nonproduction load-testing method [LAB-MUTATE]

Use an approved **synthetic** model and request corpus. Obtain authorization and a resource/time budget before starting.

1. Establish idle memory, startup time and golden inference correctness.
2. Define prompt-length bands (short, medium, long) and bounded output limits.
3. Run a low-concurrency baseline and capture TTFT, ITL, errors, queue and KV.
4. Increase concurrency in small approved steps with explicit stop thresholds.
5. Keep prompt/output distribution constant when comparing configurations.
6. Compare throughput gain against P95/P99, preemptions and stream completion.
7. Stop immediately for sustained error growth, VRAM exhaustion, uncontrolled queueing or shared-resource impact.
8. Restore the known-good release and verify post-test health.

Do not benchmark production endpoints without a separately approved change plan.

## 12. Troubleshooting scenarios

| ID | Symptom | First failed transition | Direction |
|---|---|---|---|
| K1 | High TTFT with long prompts | Prefill → first token | Prompt bands and prefill time |
| K2 | High TTFT, low GPU use | Admission/CPU → executor | Queue and tokenizer |
| K3 | High ITL with many sequences | Decode → GPU | Running count, VRAM |
| K4 | Waiting queue grows | Admission → scheduler | Bounded concurrency |
| K5 | KV cache near full | Live tokens → cache | Context/output budgets |
| K6 | Preemption spikes | Scheduler → cache capacity | Cache sizing and load |
| K7 | Load-time GPU OOM | Weights → VRAM | Model/runtime footprint |
| K8 | OOM only under concurrency | KV/activations → VRAM | Active tokens and peaks |
| K9 | Throughput drops after higher batching | Batch → execution | Scheduling overhead |
| K10 | GPU busy but throughput low | Executor → useful tokens | Kernel/model efficiency |
| K11 | Prefix cache ineffective | Prompt reuse → cache | Version/feature and keying |
| K12 | One long request starves short ones | Scheduler → fairness | Workload mix/chunking |
| K13 | 429/503 during bursts | Client → admission | Rate limits and queue |
| K14 | Gateway 504 | Runtime → deadline | Queue, TTFT and timeouts |
| K15 | Stream drops after HTTP 200 | Decode → client | Drain/cancellation |
| K16 | Model latency changes after upgrade | Version → scheduler | Immutable A/B benchmark |
| K17 | KV metric absent | Engine → telemetry | Instrumentation discovery |
| K18 | Rollback loads wrong model | Alias → artifact | Pin immutable revision |

For each scenario, record the **Lowest Proven Healthy Layer**, **First Failed Transition**, SLO impact, evidence, approved mitigation and post-recovery validation.

## 13. Safe mitigation and recovery

- **Queue saturation:** reduce admission rate or restore approved request limits; coordinate client retry backoff with jitter.
- **KV pressure:** bound prompt/output lengths and concurrency; evaluate supported cache or parallelism settings in staging.
- **High TTFT:** investigate tokenizer CPU, queue wait, prefill cost, prompt mix and supported chunking.
- **High ITL:** inspect decode concurrency, VRAM pressure, GPU memory bandwidth and worker communication.
- **OOM:** do not repeatedly restart Pods without correcting the memory budget; restore known-good model/runtime and capacity.
- **Regression after change:** roll back image digest, model/tokenizer revision and scheduler settings through TrueFoundry/GitOps.

Do not disable authentication, bypass integrity checks, evict shared model caches or restart shared GPU nodes to mask saturation.

## 14. Production incident runbook

1. **Declare:** affected model/endpoint, request volume, errors, TTFT/ITL, queue, KV occupancy, GPU/VRAM and recent releases.
2. **Localize:** client → gateway → admission → tokenizer → prefill → decode → KV cache → GPU → stream.
3. **Contain:** freeze promotion, preserve healthy replicas, bound retries and reject excess load predictably.
4. **Recover:** narrow confirmed repair or authoritative rollback with compatible image/model/config.
5. **Validate:** correct model revision, golden synthetic inference, Ready EndpointSlices, TTFT/ITL, stable queue, KV headroom and stream completion.
6. **Document:** incident timeline, LPHL/FFT, peak active tokens, mitigation, rollback owner and follow-ups.

## 15. Acceptance gates

| Gate | Required evidence |
|---|---|
| A1 | KV-cache architecture and block allocation explained |
| A2 | Correct model-specific KV estimate with assumptions |
| A3 | Prefill/decode and continuous batching understood |
| A4 | Safe concurrency/context/output budgets documented |
| A5 | TTFT, ITL, throughput, queue, KV and GPU observable |
| A6 | Bounded synthetic benchmark or clearly marked tabletop |
| A7 | K1–K18 scenarios classified by LPHL/FFT |
| A8 | Authoritative rollback and post-recovery SLO verified or marked Not run |

## 16. Evidence record

```text
Date/time / approval / workspace:
vLLM image and model revision:
GPU SKU, VRAM and tensor parallelism:
KV model layers/heads/dim/precision:
Engine KV capacity and observed utilization:
Context and concurrency limits:
Prompt/output distribution:
TTFT P95/P99, ITL, tokens/s:
Running/waiting requests, preemptions:
GPU utilization/VRAM, 429/5xx:
Lowest Proven Healthy Layer:
First Failed Transition:
Known-good rollback, SLO and cleanup:
```

**Hands-on lab:** [7.5 — PagedAttention, KV Cache and Continuous Batching Check](../labs/05-pagedattention-kv-cache-continuous-batching-check.md)  
**Next:** Chapter 7.6 — vLLM Performance Tuning, Observability and Troubleshooting.

## References

- [vLLM Documentation](https://docs.vllm.ai/)
- [vLLM PagedAttention Paper](https://arxiv.org/abs/2309.06180)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [NVIDIA GPU Monitoring](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [Prometheus Histograms](https://prometheus.io/docs/practices/histograms/)
