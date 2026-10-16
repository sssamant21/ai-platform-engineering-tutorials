# Chapter 008 — Prefill, Decode, KV Cache and Continuous Batching

**Part 02 — LLM Architecture and Inference Internals | Chapter 008 of 100**

**Status:** Written tutorial; technical/security review, actual lab execution and production acceptance pending.

## Learning objectives

Distinguish prefill from decode; estimate KV-cache capacity; explain continuous batching and scheduling fairness; isolate TTFT and TPOT regressions; and produce an evidence-based tuning and incident response plan for GPU inference.

## 1. End-to-end inference stages

```text
Synthetic client → TLS/auth gateway → admission/queue
  → tokenize/chat template
  → PREFILL: process prompt tokens, initialize KV cache
  → DECODE: select next token, update KV cache
  → continuous scheduler interleaves active sequences
  → detokenize and stream events
  → valid finish reason / final protocol completion
```

**Prefill** processes prompt context and is often compute intensive; **decode** iteratively generates tokens and can be memory-bandwidth-sensitive at low batch sizes. Hardware, architecture, kernel, context length and batch size alter the bottleneck. Queue time and networking contribute to customer TTFT independently of either GPU phase.

## 2. Prefill: operational tradeoffs

Prefill transforms input token embeddings through transformer layers and creates the initial attention key/value state. Its performance depends on prompt length, batch composition, kernel selection, GPU compute and memory, prefix caching and scheduling.

Key controls: maximum admitted prompt tokens, chunked prefill (when supported), prefix caching, max batched tokens, concurrency and prompt length cohorts. Large prompts may monopolize compute and worsen short-request TTFT without admission fairness.

Measure **queue wait**, **tokenization**, **prefill** and **first output token** separately where supported. Do not treat time to first HTTP byte as TTFT.

## 3. Decode: operational tradeoffs

During autoregressive generation, each sequence typically advances one token at a time. The runtime reuses stored KV state and schedules active sequences together. Decode performance depends on model weight reads, cache access, GPU bandwidth, batch occupancy and scheduling.

Measure output tokens/s, inter-token latency (ITL), time per output token (TPOT), per-cohort end-to-end latency and completion. Long-output requests can occupy KV cache and scheduling slots long after first token.

## 4. KV-cache capacity model

For a conventional attention implementation:

```text
KV_bytes_per_token ≈
  2 × layers × KV_heads × head_dim × bytes_per_element

KV_total_bytes ≈ KV_bytes_per_token × resident_cached_tokens
```

This simple model excludes allocation blocks, fragmentation, padding, prefix sharing, sliding-window attention, quantized KV, offload, speculative decoding and per-GPU sharding.

### Worked example — illustrative

Assume 32 layers, 8 KV heads, 128 head dimension, BF16 (2 bytes/element):

```text
2 × 32 × 8 × 128 × 2 = 131,072 bytes/token
                       = 128 KiB/token
8,192 resident tokens × 128 KiB ≈ 1 GiB
```

This is **total logical KV state for the assumed model** before implementation overhead, not a promise of GPU VRAM consumption. With tensor parallelism or cache replication, per-device allocation differs. Do not use this estimate alone to size GPUs.

### GPU memory envelope

```text
GPU VRAM ≈ model weights + KV cache + activation/workspace
         + runtime buffers + fragmentation + safety headroom
```

A model that fits at startup can still OOM at high concurrency or long context. Runtime cache block availability is often a better admission signal than raw GPU utilization.

## 5. Continuous batching

Traditional static batching waits for a fixed set of requests to complete together. Continuous (iteration-level) batching can add requests and retire finished sequences between decode iterations, improving occupancy and throughput. It also introduces scheduling and fairness decisions.

Important controls, **version-dependent**:
- Maximum concurrent sequences.
- Maximum batched tokens.
- Prefill/decode scheduling priority.
- Chunked prefill and prefix caching.
- KV-cache block allocation and preemption.
- Per-tenant admission, rate/token budgets and queue timeout.

A throughput improvement can increase tail latency. Optimize against approved customer SLIs, not GPU utilization alone.

## 6. Head-of-line blocking, preemption and fairness

A long prompt can delay short prompts if prefill work is scheduled without fairness. Cache exhaustion can cause preemption/recomputation or request rejection, depending on runtime and version. Priorities may starve low-priority traffic.

Record model, runtime version, prompt/output token cohorts, concurrency, cache occupancy, queue time, TTFT, TPOT, p95/p99 and complete-stream rate for each tuning experiment.

## 7. Performance worksheet

| Signal | Interpretation | Common confounder |
|---|---|---|
| TTFT rises, TPOT stable | queue or prefill | gateway overhead |
| TPOT rises, TTFT stable | decode contention | network jitter |
| Both rise | GPU/cache saturation | rollout/cold start |
| Queue grows, GPU idle | scheduler/admission issue | metrics stale |
| VRAM high, utilization low | weights/KV reserved | healthy idle allocation |
| Frequent preemption | KV blocks insufficient | long-context mix |
| High tokens/s, poor p99 | throughput-tail tradeoff | tenant fairness |
| HTTP 200 but incomplete stream | SSE/timeout failure | client cancellation |

Always validate telemetry freshness and customer-path synthetic results.

## 8. Read-only investigation

**[SAFE-READ]** after confirming authorization:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc,endpointslices -A -o wide
kubectl get resourcequota,hpa,pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

For an approved serving deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Inspect approved GPU/DCGM and runtime dashboards. Metric names and scheduling knobs differ across vLLM/Triton/Ray versions; verify upstream release documentation before changes. Redact logs and avoid collecting customer prompts.

## 9. Safe tuning sequence

1. Freeze model, tokenizer, runtime, driver and image revisions.
2. Establish customer SLO, workload token distributions and synthetic baseline.
3. Measure queue/prefill/decode/KV cache and GPU VRAM.
4. Change **one** version-supported knob at a time in approved staging.
5. Keep fixed load shape, warmup, GPU SKU, model and sampling parameters.
6. Compare throughput, TTFT/TPOT percentiles, complete-stream rate, OOM, fairness and cost.
7. Apply abort thresholds for errors, latency, GPU memory, quota and spend.
8. Document rollback through the authoritative controller.
9. Never promote a faster-but-less-reliable setting without service-owner acceptance.

## 10. Eighteen production failure scenarios

| ID | Symptom | First check |
|---|---|---|
| F01 | high TTFT from long prompts | input-token cohort |
| F02 | high TTFT from queue | admitted/running/queued |
| F03 | prefill stalls decode | scheduler traces |
| F04 | high TPOT | output throughput/GPU |
| F05 | KV-cache exhaustion | block occupancy |
| F06 | cache preemption thrash | preemption counters |
| F07 | CUDA OOM | VRAM/runtime logs |
| F08 | excessive context reservation | admission limits |
| F09 | unfair tenant latency | cohort fairness |
| F10 | too many concurrent sequences | queue/cache |
| F11 | oversized batched tokens | memory/TTFT |
| F12 | cold-start cache miss | warmup/prefix metrics |
| F13 | prefix-cache invalidation | model/template revisions |
| F14 | GPU thermal/power throttle | DCGM clocks/temperature |
| F15 | 429 overload | admission vs contractual quota |
| F16 | retry amplification | gateway/client attempts |
| F17 | truncated SSE | protocol completion |
| F18 | metrics missing or stale | scrape and synthetic |

## 11. Twelve operational runbooks

1. Validate synthetic complete-stream request.
2. Establish token-length and concurrency cohorts.
3. Separate queue, tokenization, prefill and decode latency.
4. Diagnose TTFT regression.
5. Diagnose TPOT/ITL regression.
6. Inspect KV cache occupancy and preemption.
7. Triage CUDA OOM and GPU memory envelope.
8. Investigate scheduler fairness and head-of-line blocking.
9. Audit prefix caching and model revision compatibility.
10. Control retry storms and admission pressure.
11. Execute staging tuning with cost and abort guardrails.
12. Validate rollback and customer-path recovery.

## 12. Eight game days

1. Long-prompt prefill dominates short-request TTFT (tabletop).
2. Output-heavy decode increases ITL (tabletop).
3. KV-cache pressure triggers preemption (tabletop).
4. Concurrency limit causes queue growth (tabletop).
5. Aggressive batch tokens trigger OOM (tabletop).
6. Prefix cache becomes invalid after model change (tabletop).
7. Tenant fairness regresses after tuning (tabletop).
8. Missing metrics conceal incomplete SSE (tabletop).

Fault injection or synthetic load requires separately approved staging and bounded GPU cost.

## 13. Fourteen acceptance gates

G01 explain prefill; G02 explain decode; G03 TTFT/TPOT units; G04 logical KV sizing; G05 GPU memory envelope; G06 continuous batching; G07 scheduler fairness; G08 token-length cohorts; G09 safe read-only diagnostics; G10 bounded staging tuning plan; G11 SSE completion; G12 FinOps/security controls; G13 rollback and runbook; G14 independent reviewer.

See [companion lab](../labs/008-prefill-decode-kv-cache-continuous-batching-check.md). All gates are pending until executed and reviewed.

**Next: Chapter 009 — Streaming, Backpressure and API Gateway Behavior.**
