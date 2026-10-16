# Chapter 033 — Hands-On Lab: TensorRT-LLM Throughput, Latency and KV Cache

**Part 05 | Chapter 033 | Published lab plan; live execution pending**

## Goal

Measure TTFT/TPOT/E2E and goodput, estimate KV cache capacity, design admission and batching experiments, validate cancellation/cache isolation and establish SLO, rollback and FinOps evidence.

## Safety

**[SAFE-READ]** authorized GPU/Kubernetes inventory, approved metrics and sanitized logs. **[LAB-MUTATE]** separately approved isolated staging with synthetic prompts, bounded duration/concurrency and GPU-hour ceiling. **[CHANGE-CONTROL]** production scheduler, KV cache, runtime, GPU, gateway and fault injection. No customer prompts, PHI, secrets or unapproved model code.

## Lab 0 — Preflight

Record UTC, context, model/tokenizer digest, TensorRT-LLM/Triton versions, GPU SKU/UUID, TP/PP, KV precision, max context/output tokens, concurrency, owners, approval, cost ceiling and rollback.

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

## Lab 1 — Latency contract [DESIGN]

Define timestamps for request start, first output token, subsequent tokens and terminal completion. Record whether TPOT includes first token and whether E2E includes gateway retries. Treat HTTP 200 without a terminal marker as incomplete when streaming requires one.

## Lab 2 — KV memory worksheet [DESIGN]

Synthetic conventional model:

```text
layers=32, KV_heads=8, head_dim=128, BF16=2 bytes
KV/token = 2 × 32 × 8 × 128 × 2 = 131072 bytes = 128 KiB
32768 tokens ≈ 4 GiB logical KV
```

Subtract weights, runtime, activations, graph/workspace, communication, other processes and safety headroom from physical per-GPU VRAM. Compare with allocator measurements if supported; never set limits from arithmetic alone.

## Lab 3 — Synthetic performance matrix [DESIGN]

| Cohort | Prompt | Output | Key metrics |
|---|---|---|---|
| A | short | short | TTFT, E2E |
| B | long | short | prefill, TTFT |
| C | short | long | TPOT, decode |
| D | long | long | KV, p99, OOM |
| E | mixed | mixed | fairness/goodput |
| F | repeated prefix | mixed | reuse/correctness |
| G | burst | mixed | queue/admission |
| H | canceled | partial | cache cleanup |

Use pinned model/GPU and repeatable inputs. Record offered and completed load, client p95/p99, per-GPU peak VRAM, valid output tokens/s and GPU-hours.

## Lab 4 — Admission and batching [DESIGN]

Specify bounded context/output limits, max active requests, queue timeout, per-tenant quota, retry backoff and overload responses. Compare two scheduling configurations while changing only one variable. Abort on SLO burn, OOM or cost ceiling.

## Lab 5 — Prefix cache and privacy [DESIGN]

Review cache-key model/tokenizer/version isolation, tenant boundary and eviction semantics. Validate synthetic fixtures for stale-prefix correctness. Do not attempt cross-tenant probing on production.

## Lab 6 — Rollback and cost [DESIGN]

Define SLO targets per workload cohort and the exact immutable rollback unit. Calculate GPU-hours per **valid terminal completion**, including idle, rejected and failed work according to an explicit accounting policy.

## Lab 7 — Eight tabletop game days

01 long-prompt TTFT; 02 tenant starvation; 03 KV fragmentation; 04 cancellation leak; 05 cross-tenant prefix reuse; 06 TP rank slowdown; 07 partial stream; 08 rollback cost spike.

## Twenty-four failure checklist

F01 TTFT semantics; F02 TPOT denominator; F03 partial completion; F04 queue; F05 prefill; F06 decode; F07 batching; F08 fairness; F09 context cap; F10 output cap; F11 fragmentation; F12 KV exhaustion; F13 stale prefix; F14 cache isolation; F15 startup OOM; F16 decode OOM; F17 cancellation leak; F18 TP imbalance; F19 NCCL; F20 streaming; F21 units; F22 freshness; F23 retries; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Pinned runtime/model | Pending |
| G02 | Tokenizer/contract | Pending |
| G03 | TTFT definition | Pending |
| G04 | TPOT/ITL definition | Pending |
| G05 | E2E/terminal completion | Pending |
| G06 | Workload cohorts | Pending |
| G07 | Queue/admission | Pending |
| G08 | Prefill | Pending |
| G09 | Decode | Pending |
| G10 | Batch fairness | Pending |
| G11 | KV sizing | Pending |
| G12 | KV allocator health | Pending |
| G13 | Cache isolation | Pending |
| G14 | GPU headroom | Pending |
| G15 | TP/PP health | Pending |
| G16 | Telemetry freshness | Pending |
| G17 | SLO burn | Pending |
| G18 | Rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/approval:
Runtime/model/tokenizer/GPU/TP/PP:
TTFT/TPOT/E2E timestamp definitions:
Prompt/output/concurrency cohorts:
Queue/prefill/decode:
KV formula/allocator/per-GPU VRAM:
Cache reuse/isolation/cancellation:
Valid completions/throughput:
SLO/error budget/telemetry:
GPU-hours/cost:
Game-day mode/findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop separately approved synthetic load, remove only owned staging artifacts, restore authoritative scheduler/runtime settings and reconcile GPU-hours. All gates remain pending until independently reviewed.

**Next: Chapter 034 — Triton and TensorRT-LLM Troubleshooting.**
