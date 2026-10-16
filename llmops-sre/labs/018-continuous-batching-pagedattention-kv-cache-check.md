# Chapter 018 — Hands-On Lab: Continuous Batching, PagedAttention and KV Cache

**Part 04 | Chapter 018 | Published lab plan; live execution pending**

## Goal

Explain scheduler behavior, calculate logical KV capacity, design safe concurrency/admission limits, investigate queue/preemption/OOM, and evaluate TTFT/TPOT, complete SSE and GPU cost.

## Safety

**[SAFE-READ]** approved metrics and redacted logs. **[LAB-MUTATE]** isolated staging synthetic inference with explicit token, concurrency, duration and GPU-hour ceilings. **[CHANGE-CONTROL]** production scheduler/context/concurrency/cache/model changes. No customer prompts, PHI, secrets or unauthorized stress tests.

## Lab 0 — Preflight

Record UTC, cluster/context, namespace, GPU SKU, vLLM version/image digest, model/tokenizer revision, KV architecture, driver/CUDA, current scheduler/admission settings, owner, reviewer and rollback.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — Continuous batching [DESIGN]

Draw a timeline with three synthetic requests: short prefill/short decode, long prefill/short decode and short prefill/long decode. Show arrival, waiting, prefill, decode, token streaming and completion. Explain why static batching differs and where fairness can fail.

## Lab 2 — PagedAttention [DESIGN]

Sketch logical KV blocks for two sequences mapped to non-contiguous physical blocks. Explain allocation, block rounding, freeing on completion and possible prefix sharing. Note that actual block sizes and caching semantics are version-specific.

## Lab 3 — KV-cache arithmetic [DESIGN]

For a synthetic model:

```text
layers = 32
KV heads = 8
head dimension = 128
bytes/element = 2 (BF16)
KV/token = 2 × 32 × 8 × 128 × 2 = 131,072 bytes = 128 KiB
8,192 resident tokens ≈ 1 GiB logical KV
32,768 resident tokens ≈ 4 GiB logical KV
```

For a 4 GiB **logical** KV envelope and 2,048 mean resident tokens/sequence, the arithmetic suggests 16 active sequences **before** block rounding, workspace, runtime reserves and latency limits. Explain why this is not an approved production setting.

## Lab 4 — Read-only diagnostics [SAFE-READ]

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

If available, record actual runtime queue, running/waiting requests, KV cache, preemption, TTFT/TPOT and scrape freshness. Do not assume metric names without checking the pinned release.

## Lab 5 — Bounded staging load matrix [DESIGN; optional approved execution]

| Cohort | Prompt | Output | Concurrency | Observe |
|---|---|---|---|---|
| A | short | short | low | baseline TTFT/TPOT |
| B | long | short | low | prefill |
| C | short | long | medium | decode/KV |
| D | mixed | mixed | bounded | fairness/preemption |

Define exact numeric limits and abort thresholds with the owner before execution. Use only synthetic prompts. Record complete SSE, errors, 429s, queue, cache blocks, preemption and GPU-hours.

## Lab 6 — Prefix caching and security [DESIGN]

Document cache-key assumptions, tokenizer/template version, tenant boundaries, eviction, hit/miss counters and correctness tests. Require explicit security approval before enabling shared prefix caching.

## Lab 7 — Eight tabletop game days

01 long prefill; 02 decode KV exhaustion; 03 preemption storm; 04 tokenizer/cache invalidation; 05 tenant starvation; 06 OOM despite free memory; 07 partial SSE; 08 cost regression.

## Twenty-four failure checklist

F01 queue; F02 long prefill; F03 TPOT; F04 fairness; F05 concurrency; F06 context; F07 low token budget; F08 high token budget; F09 KV blocks; F10 block waste; F11 preemption; F12 recompute; F13 decode OOM; F14 prefill OOM; F15 cache miss; F16 isolation; F17 eviction; F18 tokenization CPU; F19 cancellation; F20 SSE; F21 TP imbalance; F22 utilization; F23 metric drift; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Pinned vLLM release | Pending |
| G02 | Prefill/decode analysis | Pending |
| G03 | Continuous batching timeline | Pending |
| G04 | Paged KV mapping | Pending |
| G05 | KV arithmetic | Pending |
| G06 | Context/concurrency assumptions | Pending |
| G07 | GPU headroom | Pending |
| G08 | Admission controls | Pending |
| G09 | Tenant fairness | Pending |
| G10 | Prefix isolation | Pending |
| G11 | Preemption behavior | Pending |
| G12 | OOM diagnosis | Pending |
| G13 | TTFT/TPOT/complete SSE | Pending |
| G14 | Queue evidence | Pending |
| G15 | Cache metrics freshness | Pending |
| G16 | Safe load test design | Pending |
| G17 | Rollback | Pending |
| G18 | GPU FinOps | Pending |
| G19 | Incident runbook | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/authorization:
GPU/vLLM/model/tokenizer:
Scheduler settings and limits:
KV model math/block assumptions:
Prompt/output cohorts:
Queue/running/waiting/preemption:
TTFT/TPOT/SSE/429/5xx:
GPU memory and GPU-hours:
Game-day mode/findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. For approved staging, stop synthetic load, restore baseline settings via the authoritative controller, remove only owned resources and reconcile GPU-hours. All gates remain pending until reviewed.

**Next: Chapter 019 — vLLM Streaming and OpenAI-Compatible APIs.**
