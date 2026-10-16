# Chapter 020 — Hands-On Lab: vLLM GPU Memory and Context Tuning

**Part 04 | Chapter 020 | Published lab plan; not executed**

## Goal

Build a GPU VRAM envelope, calculate logical KV demand, distinguish context limits from resident-token capacity, design safe scheduler tuning and diagnose OOM, preemption, p99 and GPU cost.

## Safety

**[SAFE-READ]** approved metrics, inventory and sanitized logs. **[LAB-MUTATE]** approved isolated staging synthetic load with explicit token/concurrency/time/GPU-hour limits. **[CHANGE-CONTROL]** production GPU memory, context, dtype, quantization, scheduling or routing changes. No PHI, customer prompts or unapproved stress tests.

## Lab 0 — Preflight

Record UTC, cluster/context, namespace, GPU SKU/VRAM, driver/CUDA, pinned vLLM image/version, model/tokenizer revision, dtype, KV architecture, owner, cost ceiling and rollback.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — Per-GPU memory budget [DESIGN]

Synthetic 80 GiB GPU worksheet:

| Category | Illustrative allocation |
|---|---:|
| Weights | 28 GiB |
| Workspace/graphs | 8 GiB |
| KV allocation | 34 GiB |
| Headroom | 6 GiB |
| Driver/other | 4 GiB |
| **Total** | **80 GiB** |

These are not measured allocations and can overlap in runtime reporting. Replace with real device/rank measurements before any acceptance.

## Lab 2 — KV arithmetic [DESIGN]

```text
layers=32, KV_heads=8, head_dim=128, BF16=2 bytes
KV/token = 2×32×8×128×2 = 131,072 bytes = 128 KiB
8,192 tokens ≈ 1 GiB logical KV
32,768 tokens ≈ 4 GiB logical KV
65,536 tokens ≈ 8 GiB logical KV
```

Explain block rounding, metadata, cache sharing, TP/PP and architecture caveats.

## Lab 3 — Context versus concurrency [DESIGN]

Construct three synthetic cohorts: 512-token prompts, 8K-token prompts and 32K-token prompts with bounded outputs. Show why a 32K model context does not mean unlimited simultaneous 32K requests. Calculate resident-token demand for each cohort and define admission limits.

## Lab 4 — Read-only GPU and vLLM evidence [SAFE-READ]

On an approved GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

For a verified deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Record actual KV occupancy/preemption, queue, TTFT/TPOT, memory and exporter freshness only if exposed by the pinned runtime. Redact logs.

## Lab 5 — Safe tuning matrix [DESIGN; optional staging]

| Variant | One changed dimension | Observe |
|---|---|---|
| Baseline | none | VRAM, TTFT/TPOT, SSE |
| A | context cap | admission, KV |
| B | max sequences | queue, preemption |
| C | batched token budget | prefill/TTFT |
| D | memory target | headroom/OOM |
| E | dtype/quantization | quality, memory, speed |

Define numeric thresholds with an owner before any run. Change one dimension at a time, pin workload fixtures and stop on OOM, partial SSE, SLO burn, stale metrics or cost ceiling.

## Lab 6 — Rollback [TABLETOP]

Record the authoritative GitOps release/config, GPU scheduling impact, old/new replica coexistence, abort conditions and post-rollback complete-stream SLO. Do not run production changes.

## Lab 7 — Eight tabletop game days

01 model startup OOM; 02 prefill OOM; 03 KV decode OOM; 04 context over-admission; 05 memory target instability; 06 quantization quality; 07 TP rank imbalance; 08 cost/p99 regression.

## Twenty-four failure checklist

F01 weight OOM; F02 graphs; F03 prefill; F04 decode; F05 context low; F06 context high; F07 concurrency high; F08 concurrency low; F09 batched tokens high; F10 batched tokens low; F11 utilization high; F12 utilization low; F13 KV dtype; F14 quality; F15 block overhead; F16 fragmentation; F17 other process; F18 rank imbalance; F19 prefix churn; F20 fairness; F21 partial SSE; F22 telemetry; F23 rollout GPU; F24 cost.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Pinned vLLM release | Pending |
| G02 | GPU inventory | Pending |
| G03 | Weight budget | Pending |
| G04 | KV bytes/token | Pending |
| G05 | Workspace/headroom | Pending |
| G06 | Context admission | Pending |
| G07 | Concurrency envelope | Pending |
| G08 | Batched-token tradeoff | Pending |
| G09 | Preemption | Pending |
| G10 | Quantization quality | Pending |
| G11 | Prefix isolation | Pending |
| G12 | OOM diagnosis | Pending |
| G13 | TP rank balance | Pending |
| G14 | TTFT/TPOT | Pending |
| G15 | Complete SSE | Pending |
| G16 | Metric freshness | Pending |
| G17 | Safe tuning plan | Pending |
| G18 | Rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/approval:
GPU/model/vLLM/dtype:
Weight/KV/workspace/headroom:
KV architecture and arithmetic:
Context/concurrency/batched tokens:
OOM/preemption/queue evidence:
TTFT/TPOT/SSE and quality:
GPU cost and test bounds:
Game-day mode/findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop any approved staging load, restore known-good config through the authoritative controller, remove only owned resources and reconcile GPU-hours. All gates remain pending until independently reviewed.

**Next: Chapter 021 — vLLM Tensor/Pipeline Parallelism and Distributed Serving.**
