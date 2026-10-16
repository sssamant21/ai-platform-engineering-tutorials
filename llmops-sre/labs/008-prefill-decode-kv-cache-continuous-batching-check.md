# Chapter 008 — Hands-On Lab: Prefill, Decode, KV Cache and Continuous Batching

**Part 02 | Chapter 008 | Published plan; not executed**

## Goal

Quantify a synthetic KV-cache model, distinguish queue/prefill/decode bottlenecks and create a safe staging tuning plan with explicit SLO, cost and rollback gates.

## Guardrails

**[SAFE-READ]** approved Kubernetes and telemetry only. **[LAB-MUTATE]** staging-only synthetic requests and approved runtime tuning. **[CHANGE-CONTROL]** production, GPU node and serving changes. Never capture real prompts, PHI, secrets or proprietary model artifacts.

## Lab 0 — Preconditions

Record UTC, approved cluster/context/namespace, model/tokenizer/runtime/image, GPU SKU, driver/CUDA, current concurrency and batching configuration, synthetic workload shape, owner, cost ceiling and abort thresholds.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc,endpointslices -A -o wide
kubectl get resourcequota,hpa,pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — KV-cache sizing exercise [DESIGN]

Assume 32 layers, 8 KV heads, head dimension 128, 2 bytes/element:

```text
KV bytes/token = 2 × 32 × 8 × 128 × 2 = 131,072
8,192 resident tokens = 1,073,741,824 bytes ≈ 1 GiB
16,384 resident tokens ≈ 2 GiB
```

State exclusions: block rounding, cache sharing, fragmentation, sharding, offload and workspace. Compare this logical estimate to **approved runtime metrics** if available; do not equate them blindly.

## Lab 2 — Latency attribution [DESIGN]

For an illustrative request:

```text
gateway 80 ms + queue 120 ms + tokenization 30 ms
+ prefill 370 ms + decode remainder 1,400 ms = E2E 2,000 ms
simplified TTFT = 600 ms
for 71 generated tokens, post-first-token TPOT ≈ 1,400 / 70 = 20 ms/token
```

Record assumptions and explain why real traces may overlap phases or include network jitter.

## Lab 3 — Runtime and GPU evidence [SAFE-READ]

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Use approved dashboards for queue wait, prefill/decode, TTFT/TPOT, output tokens/s, KV-cache occupancy/preemption and GPU VRAM. Verify version-specific metric names. Redact logs.

## Lab 4 — Continuous-batching comparison [DESIGN]

Build a worksheet comparing two hypothetical workload cohorts: short prompt/short output and long prompt/long output. Record concurrency, max batched tokens, queue time, TTFT p95, TPOT p95, completion, GPU-hours and tenant fairness. Do not claim a measured improvement without actual staging data.

## Lab 5 — Safe tuning plan [DESIGN]

Propose one version-supported change (for example concurrency cap or batch-token limit) with baseline, hypothesis, synthetic workload, rollback owner, abort criteria, expected benefit, cost ceiling and evidence. **Do not apply** unless staging authorization is granted.

## Lab 6 — Eight game days

01 prefill head-of-line blocking; 02 decode slowdown; 03 KV preemption; 04 queue growth; 05 OOM from batch size; 06 prefix-cache invalidation; 07 tenant unfairness; 08 telemetry blind spot. Default **tabletop**.

## Eighteen-scenario checklist

F01 long prompt; F02 queue; F03 prefill/decode conflict; F04 TPOT; F05 cache exhaustion; F06 preemption; F07 OOM; F08 context reservation; F09 fairness; F10 concurrency; F11 batch tokens; F12 cold cache; F13 invalidation; F14 GPU throttle; F15 429; F16 retries; F17 SSE; F18 stale metrics.

## Fourteen acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Prefill explanation | Pending |
| G02 | Decode explanation | Pending |
| G03 | TTFT/TPOT units | Pending |
| G04 | KV-cache math | Pending |
| G05 | GPU memory envelope | Pending |
| G06 | Continuous batching | Pending |
| G07 | Fairness analysis | Pending |
| G08 | Token cohorts | Pending |
| G09 | Authorized diagnostics | Pending |
| G10 | Staging tuning plan | Pending |
| G11 | Complete SSE criterion | Pending |
| G12 | Cost/security guardrails | Pending |
| G13 | Rollback and runbook | Pending |
| G14 | Reviewer signoff | Pending |

## Evidence and cleanup

```text
UTC/context/approval:
Model/runtime/driver/GPU:
KV assumptions and logical bytes:
Queue/prefill/decode/TTFT/TPOT:
Token cohorts and cache pressure:
Staging change hypothesis and abort threshold:
Game-day mode (tabletop/executed):
Findings, cost, reviewer, acceptance:
```

Read-only/tabletop work creates no resources. Stop separately approved synthetic load, reconcile GPU costs and leave acceptance pending until independently reviewed.

**Next: Chapter 009 — Streaming, Backpressure and API Gateway Behavior.**
