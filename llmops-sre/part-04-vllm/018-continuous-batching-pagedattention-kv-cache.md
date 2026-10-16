# Chapter 018 — Continuous Batching, PagedAttention and KV Cache

**Part 04 — vLLM Administration and Operations | Chapter 018 of 100**

> **Status:** Written tutorial published. Technical/security review, real lab execution and operational acceptance pending. vLLM scheduling algorithms, configuration flags and metric names vary by release; validate the pinned version.

## 1. Learning objectives

Explain continuous batching, prefill and decode scheduling, PagedAttention, KV block allocation, prefix caching, admission control and preemption; size KV capacity using model architecture and token distributions; diagnose TTFT/TPOT regressions, cache exhaustion and fairness; design safe load tests, SLOs and recovery runbooks.

## 2. Inference scheduling architecture

```text
Requests (tenant / prompt / max output / deadline)
      │
      ▼
Gateway auth + quota + admission
      │
      ▼
vLLM API → tokenizer → waiting queue
                           │
                    engine scheduler
                    ├─ prefill work
                    ├─ decode work
                    ├─ continuous batching
                    ├─ KV block allocation
                    ├─ prefix cache (if enabled)
                    └─ preemption / recomputation or swap (version-specific)
                           │
                       GPU workers
                           │
                     streamed tokens
                           ▼
                    SSE completion
```

A scheduler must balance throughput, latency, memory and fairness. Continuous batching can admit and retire sequences dynamically, unlike fixed static batches. The precise prioritization policy and preemption mechanism are implementation- and release-specific.

## 3. Prefill versus decode

**Prefill** processes input tokens and builds KV state; it is often compute-intensive for large prompts. **Decode** generates tokens iteratively using cached keys and values; for many workloads it is memory-bandwidth sensitive.

| Stage | Typical pressure | Customer SLI |
|---|---|---|
| Queue/admission | overload, quotas, scheduler capacity | queue wait |
| Tokenization | CPU, template, long prompts | pre-processing time |
| Prefill | prompt length, compute, temporary workspace | TTFT |
| Decode | resident KV, memory bandwidth, concurrency | TPOT/ITL |
| Stream/proxy | flush, timeout, client cancellation | complete SSE |

TTFT includes queueing, network and prefill; it is not a pure GPU kernel metric. TPOT can deteriorate while overall GPU utilization remains high.

## 4. PagedAttention and KV block management

PagedAttention treats KV memory as blocks mapped to logical token sequences, reducing the need for a single contiguous allocation for each sequence. Block sizes, sharing, reclamation and allocator details depend on runtime version.

```text
Sequence A logical tokens: [A0][A1][A2][A3]
                            │   │   │   │
Physical KV blocks:       [B7][B2][B9][B1]

Sequence B logical tokens: [B0][B1][B2]
                            │   │   │
Physical KV blocks:       [B3][B5][B8]
```

Logical allocation differs from actual device reservation; block rounding, metadata, CUDA graphs, model weights and workspaces consume additional memory. Paged KV reduces fragmentation in some patterns but does not eliminate OOM.

## 5. KV cache sizing

For a conventional attention model:

```text
KV bytes/token ≈ 2 × layers × KV_heads × head_dim × bytes_per_element
logical_KV_bytes ≈ KV_bytes/token × resident_tokens
```

Synthetic example: 32 layers, 8 KV heads, 128 head dimension, BF16 at 2 bytes/element:

```text
KV/token = 2 × 32 × 8 × 128 × 2 = 131,072 bytes = 128 KiB
8,192 resident tokens ≈ 1 GiB logical KV
32,768 resident tokens ≈ 4 GiB logical KV
```

This excludes block rounding, replicated/sharded layers, metadata, cross-attention and runtime-specific allocation. Grouped-query attention reduces KV heads compared with full multi-head attention. Sliding-window, prefix reuse, cache quantization, TP/PP and offload change real capacity.

## 6. Concurrency and memory headroom

```text
GPU VRAM =
  weights + allocated KV blocks + workspace + runtime buffers
  + CUDA graphs + allocator reserve + safety headroom
```

A rough homogeneous upper bound is:

```text
max_active_sequences ≈ floor(
  allocatable_KV_bytes /
  (mean_resident_tokens_per_sequence × KV_bytes_per_token)
)
```

Do **not** treat the formula as a guaranteed concurrency setting: real token lengths vary, block rounding matters, and scheduler token budgets, compute and p99 latency can constrain concurrency before VRAM does.

Synthetic 4 GiB logical KV budget at 128 KiB/token provides 32,768 logical resident tokens. If each active sequence consumes 2,048 tokens on average, the arithmetic suggests 16 sequences, before block rounding and reserves. For a long-tail workload, enforce a conservative admission envelope based on distributions and SLOs.

## 7. Continuous batching, fairness and starvation

Throughput optimization can hurt interactive latency if long-prefill jobs monopolize compute or large decode cohorts consume KV. Analyze per-tenant queue wait, prompt/output length buckets, prefill/decode interleaving, cancellation and preemption.

Possible policies include separate latency-sensitive and batch pools, per-tenant quotas, token budgets, queue deadlines, maximum context/output limits and controlled overload rejection. Never silently truncate prompts or bypass safety instructions to fit cache.

## 8. Prefix caching and tenant isolation

Prefix caching can reuse KV blocks for identical or compatible prefixes, reducing redundant prefill. Its effectiveness depends on exact tokenization, template, model revision, cache lifetime and workload repetition.

**Security:** shared caches are a multi-tenant isolation concern. Confirm the runtime's isolation guarantees, cache keying and access controls before enabling cross-tenant reuse. Do not infer safe isolation solely from a performance gain. Record cache hit/miss, eviction and memory pressure, where exposed.

## 9. Preemption, recomputation and eviction

When KV capacity is insufficient, a scheduler may delay, preempt or recompute sequences depending on version and configuration. Preemption can preserve availability but amplify latency, GPU work and cost. A growing preemption rate often signals an unsafe concurrency/context envelope.

Investigate active/waiting requests, allocated/free KV blocks, preemptions, resident tokens, queue wait, GPU VRAM and prompt/output cohorts. Blindly raising a GPU memory target or max sequences can increase OOM and SLO burn.

## 10. Read-only investigation

**[SAFE-READ]** on approved systems:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

For a verified vLLM deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Metric names vary by release. Inspect the actual metrics endpoint/schema and ensure scrape freshness before writing PromQL. Never collect customer prompts or secrets.

## 11. Safe performance experiment

**[LAB-MUTATE]** only in isolated, approved staging with synthetic requests:

1. Pin model, tokenizer, vLLM version, image digest, GPU SKU and driver.
2. Establish low-load baseline with complete SSE verification.
3. Sweep bounded prompt lengths, output lengths and concurrent sequences.
4. Record p50/p95/p99 TTFT, TPOT/ITL, E2E, queue wait, throughput, 429/5xx, complete SSE, KV occupancy, preemptions and GPU memory.
5. Compare a single reviewed scheduler/admission setting at a time.
6. Stop on OOM, SLO breach, runaway queue, GPU cost ceiling or missing telemetry.
7. Revert to approved baseline through the authoritative controller.
8. Retain sanitized reproducible evidence.

Never run saturation tests against customer production endpoints.

## 12. Troubleshooting decision tree

```text
vLLM slow or failing?
 ├─ high queue wait → admission, quotas, concurrency
 ├─ high TTFT with long prompts → prefill/token budget
 ├─ high TPOT → decode, KV pressure, memory bandwidth
 ├─ KV free blocks low → resident tokens, context, cache
 ├─ frequent preemption → oversubscription / scheduling
 ├─ CUDA OOM → weights + KV + workspace + graphs
 ├─ one tenant starves → fairness / per-tenant limits
 ├─ cache hit ratio drops → template/revision/eviction
 ├─ partial SSE → cancellation, proxy timeout, overload
 ├─ CPU bottleneck → tokenization, API process, logging
 ├─ GPU utilization low, queue high → scheduler/routing
 ├─ TP rank stalled → NCCL, per-rank cache, topology
 └─ missing metrics → exporter schema / freshness
```

## 13. Twenty-four production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | queue saturation | waiting requests |
| F02 | long prefill blocks short requests | prompt cohorts |
| F03 | decode TPOT regression | output-token timing |
| F04 | scheduler unfairness | per-tenant wait |
| F05 | active sequence oversubscription | KV occupancy |
| F06 | max context misconfigured | token admission |
| F07 | batched token budget too small | prefill throughput |
| F08 | batched token budget too large | TTFT/OOM |
| F09 | KV block exhaustion | free block metrics |
| F10 | block rounding waste | block occupancy |
| F11 | preemption thrash | preemption counters |
| F12 | recomputation cost spike | GPU-hours |
| F13 | CUDA OOM under decode | VRAM/cache |
| F14 | prefill workspace OOM | prompt length |
| F15 | prefix cache miss regression | cache hit metrics |
| F16 | prefix reuse security concern | isolation review |
| F17 | cache eviction churn | cache turnover |
| F18 | tokenization CPU bottleneck | CPU and queue |
| F19 | cancelled request retains resources | active requests |
| F20 | SSE incomplete under overload | terminal validator |
| F21 | TP rank cache imbalance | per-rank memory |
| F22 | GPU memory high, throughput low | tokens/s |
| F23 | exporter metric drift | scrape schema |
| F24 | GPU cost per good request rises | FinOps/SLO |

## 14. Sixteen operational runbooks

1. Validate workload and release baseline.
2. Diagnose queue saturation.
3. Diagnose long-prefill TTFT regression.
4. Diagnose decode TPOT regression.
5. Audit admission and tenant fairness.
6. Calculate KV bytes/token and capacity.
7. Investigate KV block exhaustion.
8. Investigate preemption/recomputation.
9. Investigate CUDA OOM and workspace.
10. Validate prefix cache correctness/isolation.
11. Diagnose cache hit/eviction regression.
12. Investigate cancellation and incomplete SSE.
13. Diagnose CPU tokenization bottleneck.
14. Diagnose TP rank KV imbalance.
15. Execute safe tuning/rollback.
16. Verify sustained SLO and FinOps recovery.

## 15. Eight game days

1. Long prompts cause TTFT burn.
2. High decode concurrency exhausts KV.
3. Scheduler preemption causes retry amplification.
4. Prefix cache invalidates after tokenizer revision.
5. One tenant starves behind bulk traffic.
6. CUDA OOM occurs despite apparent free memory.
7. Partial SSE appears under saturation.
8. Higher throughput lowers successful-request efficiency.

Default **tabletop**; live experiments require approved staging and bounded costs.

## 16. Twenty acceptance gates

G01 pinned runtime; G02 prefill/decode distinction; G03 continuous batching; G04 PagedAttention mapping; G05 KV bytes/token; G06 context/concurrency math; G07 GPU headroom; G08 admission policy; G09 tenant fairness; G10 prefix isolation; G11 preemption behavior; G12 OOM triage; G13 TTFT/TPOT/SSE; G14 queue metrics; G15 cache metrics freshness; G16 staging test safety; G17 rollback; G18 FinOps; G19 RCA/runbook; G20 independent review.

See [companion lab](../labs/018-continuous-batching-pagedattention-kv-cache-check.md). All gates remain pending until evidenced and reviewed.

**Next: Chapter 019 — vLLM Streaming and OpenAI-Compatible APIs.**
