# Chapter 033 — TensorRT-LLM Throughput, Latency and KV Cache

**Part 05 — NVIDIA Triton and TensorRT-LLM | Chapter 033 of 100**

> **Publication status:** Written tutorial published. Independent technical/security review, live lab execution and operational acceptance are pending. All feature names, CLI flags and metric availability must be validated against the pinned TensorRT-LLM release and serving integration.

## Learning objectives

Measure throughput, TTFT, inter-token latency, TPOT, end-to-end latency and goodput; model KV-cache capacity; understand prefill/decode contention, in-flight batching, scheduling and admission; tune safely under SLO and GPU-cost constraints; diagnose OOM, fragmentation, starvation and distributed execution.

## 1. Serving execution path

```text
Authenticated gateway → request validation / token limits
    → scheduler queue → prefill → first token
    → iterative decode / KV reuse → terminal response
    → client-side completion validation
                         │
                 TensorRT-LLM runtime
                   ├─ GPU kernels
                   ├─ KV cache allocator
                   ├─ batching / scheduling
                   └─ TP/PP ranks where configured
```

The exact executor, scheduling, KV-cache and streaming interfaces are release-dependent. Do not assume all TensorRT-LLM releases expose the same flags or that Triton metrics fully capture client-perceived streaming completion.

## 2. Performance vocabulary

| Metric | Definition | Pitfall |
|---|---|---|
| TTFT | client request start to first output token | includes gateway, queue, prefill |
| ITL | interval between consecutive emitted tokens | sensitive to streaming transport |
| TPOT | time per output token, using an explicitly documented denominator | implementations differ on first token |
| E2E latency | request start to validated terminal completion | HTTP 200 alone is insufficient |
| Output throughput | output tokens per second across valid completions | distinguish offered vs completed |
| Request throughput | completed eligible requests per second | ignores token-length mix if used alone |
| Goodput | completed work satisfying correctness and SLO | define SLO eligibility |
| Queue duration | time awaiting execution | not equivalent to TTFT |
| GPU utilization | device activity | high utilization can coexist with poor goodput |
| GPU-hours per valid completion | cost-efficiency proxy | include idle and failed work |

For a nonstreaming response, TTFT may not be observable in the same way. Define timestamp source, clocks and client instrumentation before comparing results.

## 3. TTFT versus decode

Prefill processes input context; decode iteratively generates tokens using attention and KV state. Long prompts can inflate TTFT; long outputs consume KV capacity and can increase decode contention. Queue wait and batching can dominate either phase.

```text
E2E ≈ gateway + queue + prefill + decode + response overhead
```

These are conceptual components, not an assertion that independently sampled metrics sum exactly.

## 4. KV-cache fundamentals

For a conventional transformer, approximate logical KV bytes per token:

```text
KV_bytes_per_token = 2 × num_layers × num_kv_heads × head_dim × bytes_per_element
```

Synthetic BF16 example: 32 layers, 8 KV heads, head dimension 128:

```text
2 × 32 × 8 × 128 × 2 = 131072 bytes = 128 KiB/token
32768 resident tokens ≈ 4 GiB logical KV
```

This is a simplified logical estimate. Actual memory differs with TP/PP partitioning, block/page allocation, alignment, reuse, quantized KV, speculative decoding, sliding-window attention, allocator overhead and model-specific architecture.

## 5. KV capacity worksheet

```text
usable_GPU_VRAM =
  physical_VRAM
  - weights_per_GPU
  - runtime_and_CUDA_workspace
  - activation_and_graph_buffers
  - communication_buffers
  - other_processes
  - safety_headroom

estimated_KV_token_capacity ≈ usable_KV_bytes / logical_KV_bytes_per_token
```

Do not equate estimated token capacity with an approved maximum context. The serving runtime's actual allocator, block fragmentation, per-request limits and admission policy determine effective capacity.

## 6. Paged KV and cache reuse

Supported runtimes may manage KV in blocks/pages to reduce allocation waste and support concurrent sequences. Prefix reuse can save prefill work for repeated contexts, but correctness depends on tokenizer, model revision, position encoding and cache key isolation. Cross-tenant cache reuse must be reviewed for privacy and data-leakage risks.

Monitor allocated/free blocks, evictions, cache hits, reuse correctness and request-level cache pressure **only where metrics are actually supported**.

## 7. In-flight batching and scheduling

Continuous/in-flight batching can combine active decode sequences with arriving work, improving GPU occupancy. It can also increase per-request latency under overload. Scheduling policies may balance prefill and decode differently by release.

Tune:
- maximum concurrent sequences;
- token budgets and admission;
- maximum batch/request size;
- KV-cache budget;
- scheduling and preemption;
- per-tenant fairness and queue timeout;
- output-token caps and cancellation.

Change one variable at a time and preserve rollback.

## 8. Queueing and overload

As offered work approaches effective capacity, queueing delay can grow nonlinearly. A healthy throughput plateau is not evidence of healthy p99 latency. Define admission limits and backpressure before the GPU is saturated.

Classify:
1. rejected requests (429 or documented equivalent);
2. queue timeout;
3. backend execution failure;
4. canceled requests;
5. partial streaming completion;
6. successful complete responses within SLO.

Never retry unlimited long-context requests; retries can amplify saturation.

## 9. Workload matrix

| Cohort | Prompt | Output | Primary concern |
|---|---|---|---|
| A | short | short | overhead/TTFT |
| B | long | short | prefill/TTFT |
| C | short | long | decode/TPOT |
| D | long | long | KV/OOM |
| E | mixed | mixed | fairness/p99 |
| F | repeated prefix | mixed | cache correctness |
| G | burst | mixed | queue/admission |
| H | cancel/timeout | partial | cleanup and billing |

Use synthetic or approved datasets and reproducible seeds where meaningful. Record model/tokenizer, runtime, GPU SKU, TP/PP, concurrency, input/output distributions and warmup.

## 10. Benchmark methodology

Use a version-compatible benchmarking client and verify flags via its own help output. Keep warmup, measurement window, prompt lengths, output caps, concurrency and error classification consistent. Record both offered load and **completed valid** load.

A benchmark must include:
- client TTFT p50/p95/p99;
- TPOT/ITL distribution with explicit denominator;
- E2E p95/p99 and valid completion rate;
- output tokens/s and requests/s;
- queue time and backend execution;
- peak per-GPU VRAM, GPU power and utilization;
- KV allocator/cache metrics if supported;
- GPU-hours and cost per valid completion.

## 11. Synthetic capacity example

Assume 1 GPU with 80 GiB VRAM, 44 GiB weights/runtime, 8 GiB additional workspace and 8 GiB safety headroom:

```text
KV budget = 80 - 44 - 8 - 8 = 20 GiB
At 128 KiB logical KV/token:
20 GiB / 128 KiB ≈ 163840 logical resident tokens
```

**Illustrative arithmetic only.** It excludes allocator inefficiency and other architecture-dependent overhead; it is not a safe configuration recommendation. Validate actual allocated KV blocks and concurrent request shapes in staging.

## 12. Multi-GPU TP/PP effects

Tensor parallelism may partition KV and weights across ranks while adding communication cost. Pipeline parallelism changes layer placement and can create pipeline bubbles. Evaluate:
- per-rank memory, not cluster aggregate;
- NVLink/PCIe topology;
- NCCL timeouts and rank readiness;
- batch-size sensitivity;
- degraded behavior on a single rank failure;
- GPU-hours per valid completion.

A larger GPU count can lower single-request latency yet worsen cost efficiency.

## 13. Prometheus and tracing

Metric families differ by runtime/backend and integration. Inventory actual names, types, labels and units. Triton cumulative duration counters are not p99 histograms. Prefer client histograms for streaming TTFT/E2E and validated token-level instrumentation for TPOT.

Never place raw prompts, completions, tenant identifiers or tokens into metric labels or unredacted traces. Alert on missing telemetry separately from a true zero-traffic interval.

## 14. SLO and error budget

Define eligibility and completion:

```text
Valid completion = authorized eligible request
                   + correct output contract
                   + terminal stream completion
                   + within latency SLO
```

Example **policy illustration**:
- TTFT p95 below an approved target for interactive cohort;
- TPOT p95 below an approved target for decode cohort;
- valid completion at or above an approved percentage;
- no GPU OOM, unsafe cross-tenant cache reuse or unbounded retries;
- cost per valid completion below an approved ceiling.

No universal SLO threshold fits all models or use cases.

## 15. Safe operational checks

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

For an approved deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment APPROVED_INFERENCE_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/APPROVED_INFERENCE_DEPLOYMENT --since=15m --tail=100
```

Redact logs; do not dump Secrets or customer prompts.

## 16. Troubleshooting decision tree

```text
Latency or throughput regression?
 ├─ TTFT high, queue high → admission/scheduler
 ├─ TTFT high, queue low → prefill/context/GPU
 ├─ TPOT high → decode/kernel/communication
 ├─ E2E high, TTFT normal → output length/transport
 ├─ KV exhausted → concurrency/context/cache
 ├─ GPU OOM at startup → weights/workspace
 ├─ GPU OOM during decode → KV growth/fragmentation
 ├─ repeated-prefix wrong → cache isolation/key
 ├─ one TP rank slow → topology/NCCL
 ├─ cancellation leaks memory → lifecycle cleanup
 ├─ incomplete stream → backend/gateway
 ├─ metrics stale → exporter/scrape
 └─ cost high → goodput/overprovisioning
```

## 17. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | TTFT definition inconsistent | instrumentation |
| F02 | TPOT denominator wrong | query review |
| F03 | HTTP 200 counted as completion | terminal validator |
| F04 | queue saturation | queue metric |
| F05 | prefill stalls | prompt cohort |
| F06 | decode throughput collapse | token timings |
| F07 | low-load batching regression | p99 |
| F08 | tenant starvation | fairness metrics |
| F09 | context limit bypass | admission |
| F10 | output cap bypass | token accounting |
| F11 | KV allocator fragmentation | block stats |
| F12 | KV cache exhausted | per-GPU VRAM |
| F13 | stale prefix cache | fixture |
| F14 | cross-tenant cache leak | security audit |
| F15 | startup GPU OOM | initialization |
| F16 | decode GPU OOM | active tokens |
| F17 | cancellation fails to free KV | active slots |
| F18 | TP rank imbalance | per-rank timings |
| F19 | NCCL timeout | network/rank |
| F20 | gRPC stream incomplete | client completion |
| F21 | metric units mislabeled | HELP/TYPE |
| F22 | telemetry flatline | scrape freshness |
| F23 | retries amplify overload | gateway retries |
| F24 | cost per valid completion spikes | FinOps |

## 18. Sixteen operational runbooks

1. Define and validate TTFT/TPOT/E2E instrumentation.
2. Verify terminal response completion.
3. Diagnose scheduler queue saturation.
4. Diagnose prefill/long-context latency.
5. Diagnose decode throughput regression.
6. Tune batching and fairness safely.
7. Enforce context/output/admission limits.
8. Diagnose KV capacity and fragmentation.
9. Validate prefix-cache correctness and isolation.
10. Recover startup GPU OOM.
11. Recover decode GPU OOM.
12. Recover canceled-request KV leakage.
13. Diagnose TP/PP rank imbalance and NCCL.
14. Restore streaming and gateway deadlines.
15. Restore metrics/SLO and GPU FinOps.
16. Execute immutable rollback and RCA.

## 19. Eight tabletop game days

1. Long prompts cause TTFT p99 SLO burn.
2. Batch optimization increases throughput but starves small requests.
3. KV fragmentation triggers OOM despite apparent free memory.
4. Cancellation leaks cache blocks.
5. Prefix reuse returns stale/cross-tenant state.
6. One tensor-parallel rank slows decode.
7. Gateway returns HTTP 200 but no terminal completion.
8. Rollback lowers latency but doubles GPU-hours per valid completion.

Tabletop by default. Live injection requires separately approved isolated staging.

## 20. Twenty acceptance gates

G01 pinned runtime/model; G02 tokenizer/contract; G03 TTFT definition; G04 TPOT/ITL definition; G05 E2E/terminal completion; G06 workload cohorts; G07 queue/admission; G08 prefill; G09 decode; G10 batch fairness; G11 KV sizing; G12 KV allocator health; G13 cache isolation; G14 GPU headroom; G15 TP/PP health; G16 telemetry freshness; G17 SLO burn; G18 rollback; G19 GPU FinOps; G20 independent review.

See [companion lab](../labs/033-tensorrt-llm-throughput-latency-kv-cache-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 034 — Triton and TensorRT-LLM Troubleshooting.**
