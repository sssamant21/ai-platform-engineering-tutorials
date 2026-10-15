# Lab 7.5 — PagedAttention, KV Cache and Continuous Batching Check

**Canonical:** [Chapter 7.5](../05-pagedattention-kv-cache-continuous-batching/05-pagedattention-kv-cache-continuous-batching.md)  
**Default:** [SAFE-READ]. Benchmarks and configuration changes are [LAB-MUTATE] in an approved isolated nonproduction workspace only.

## 1. Goal

Explain vLLM's block-managed KV cache, estimate capacity, observe continuous batching, measure TTFT/ITL and queue pressure, classify 18 failure modes and demonstrate safe recovery.

## 2. Preflight [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> logs <pod> --tail=150
```

Record approved TrueFoundry workspace, model and image digest, GPU SKU, owner, context/concurrency limits and known-good rollback release. No production load injection or sensitive prompts.

## 3. KV cache estimation

Fill in the actual model metadata and calculate the logical unsharded estimate:

```text
KV bytes = 2 × layers × KV_heads × head_dim
           × live_tokens × bytes_per_KV_element
```

| Input | Value | Source |
|---|---|---|
| Number of layers | | |
| KV heads | | |
| Head dimension | | |
| Bytes per KV element | | |
| Live tokens | | |
| Logical KV bytes | | |
| GPUs / tensor parallelism | | |
| Observed cache capacity | | |
| Observed GPU VRAM | | |

Explain deviations from real cache allocation due to sharding, block rounding, prefix reuse and runtime overhead.

## 4. Metric discovery

| Signal | Available? | Observed value | Source |
|---|---|---|---|
| TTFT P95/P99 | | | |
| Inter-token latency | | | |
| Prompt tokens/s | | | |
| Output tokens/s | | | |
| Running requests | | | |
| Waiting requests | | | |
| KV cache utilization | | | |
| Preemption/recomputation | | | |
| GPU utilization/VRAM | | | |
| 429/5xx/stream failures | | | |

Do not invent vLLM metric names. If a series is absent, record an instrumentation gap.

## 5. Optional bounded synthetic benchmark [LAB-MUTATE]

Only in an approved isolated nonproduction workspace:

1. Verify the golden synthetic request and model revision.
2. Record idle GPU/VRAM, queue, KV occupancy and error rate.
3. Run a short-prompt/short-output baseline at concurrency 1.
4. Repeat at approved concurrency 2 and 4 **only if capacity permits**.
5. Repeat with a bounded longer prompt; keep output limit fixed.
6. Compare TTFT, ITL, throughput, queue, KV usage and VRAM.
7. Stop if errors, tail latency, queue or memory exceed agreed limits.
8. Restore the known-good release and confirm readiness and baseline health.

Do not treat the example concurrency values as production recommendations.

| Workload | Concurrency | TTFT P95 | ITL P95 | Output tokens/s | Waiting | KV use | Errors |
|---|---:|---|---|---|---|---|---|
| Short prompt baseline | 1 | | | | | | |
| Short prompt bounded | 2 | | | | | | |
| Short prompt bounded | 4 | | | | | | |
| Longer prompt bounded | approved | | | | | | |

## 6. Failure tabletop

| ID | Scenario | LPHL | FFT | Evidence | Mitigation |
|---|---|---|---|---|---|
| K1 | High TTFT long prompt | | | | |
| K2 | High TTFT low GPU | | | | |
| K3 | High ITL many sequences | | | | |
| K4 | Waiting queue grows | | | | |
| K5 | KV cache near full | | | | |
| K6 | Preemption spikes | | | | |
| K7 | Load-time GPU OOM | | | | |
| K8 | Concurrent KV OOM | | | | |
| K9 | Throughput drops with batching | | | | |
| K10 | Busy GPU low throughput | | | | |
| K11 | Prefix cache ineffective | | | | |
| K12 | Long request starvation | | | | |
| K13 | Burst 429/503 | | | | |
| K14 | Gateway 504 | | | | |
| K15 | Stream interruption | | | | |
| K16 | Upgrade latency regression | | | | |
| K17 | KV metric absent | | | | |
| K18 | Wrong model after rollback | | | | |

**LPHL:** Lowest Proven Healthy Layer. **FFT:** First Failed Transition. Mark tabletop-only exercises as such; do not claim live validation.

## 7. Recovery check

Record rollback trigger, owner, image/model/config revision and timestamp. Restore via the authoritative TrueFoundry/GitOps workflow. Verify actual loaded model, Ready endpoints, golden inference, TTFT/ITL, queue, KV headroom, GPU VRAM and stream completion. Clean up only lab-owned resources.

## 8. Acceptance gates

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| A1 Block-managed KV cache explained | | |
| A2 KV estimate and assumptions | | |
| A3 Prefill/decode and batching | | |
| A4 Safe capacity/admission budget | | |
| A5 Metrics and telemetry | | |
| A6 Bounded benchmark/tabletop distinction | | |
| A7 K1–K18 classification | | |
| A8 Recovery/cleanup | | |

## 9. Evidence report

```text
Date/time, approval and workspace:
Image/model version, GPU SKU:
KV-cache formula inputs and estimate:
Observed engine KV capacity and VRAM:
Context/concurrency and request mix:
TTFT/ITL/tokens-per-second/queue:
Preemptions, errors and stream completion:
K1–K18 classification:
Lowest Proven Healthy Layer:
First Failed Transition:
Rollback release and post-recovery SLO:
Not run checks and cleanup:
```

**Next:** Chapter 7.6 — vLLM Performance Tuning, Observability and Troubleshooting.
