# Lab 8.5 — Inference Concurrency, Latency and Throughput Optimization Check

**Canonical:** [Chapter 8.5](../05-inference-concurrency-latency-throughput-optimization/05-inference-concurrency-latency-throughput-optimization.md)  
**Default:** [SAFE-READ]. Benchmarking and tuning require [LAB-MUTATE] in isolated approved staging.

## 1. Mission and safety

Establish a reproducible inference baseline, distinguish prefill/decode/KV bottlenecks, calculate SLO-constrained goodput, test one supported tuning variable and document safe rollback. No customer prompts, production stress tests or unbounded request generation.

## 2. Preflight evidence

```text
Approval, owner, workspace/namespace:
Immutable model/tokenizer/image digest and vLLM version:
GPU SKU/count, tensor parallelism, context and dtype:
Runtime settings, KV cache and memory headroom:
TTFT/ITL/E2E P95/P99 SLO and error budget:
Synthetic workload seed and token distributions:
Hard request/token/duration/GPU-cost ceilings:
Stop thresholds and known-good rollback:
```

## 3. Read-only inventory

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,hpa -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> describe pod <inference-pod>
kubectl get nodes -o wide
```

Use approved metrics dashboards to capture queue, KV occupancy, preemption, TTFT/ITL, throughput and GPU telemetry. Verify metric names and labels from installed versions.

## 4. Workload matrix

| Test | Input tokens | Output tokens | Concurrency | SLO/notes |
|---|---:|---:|---:|---|
| B1 | 128 | 64 | 1 | Warm baseline |
| B2 | 128 | 256 | 8 | Decode |
| B3 | 2,048 | 64 | 8 | Prefill |
| B4 | 2,048 | 512 | 16 | Mixed |
| B5 | 8,192 | 128 | 4 | Long context |
| B6 | P50/P95 production-shaped synthetic | Mixed | Ramped | Goodput |
| B7 | Shared-prefix synthetic | Mixed | Ramped | Prefix cache |
| B8 | Interactive + batch | Mixed | Ramped | Fairness |

These are illustrative; reduce lengths/concurrency to approved limits and the actual model context.

## 5. Benchmark result worksheet

| Scenario | RPS | Input tok/s | Output tok/s | TTFT P95 | ITL P95 | E2E P95 | Errors | SLO goodput |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| B1 | | | | | | | | |
| B2 | | | | | | | | |
| B3 | | | | | | | | |
| B4 | | | | | | | | |
| B5 | | | | | | | | |
| B6 | | | | | | | | |
| B7 | | | | | | | | |
| B8 | | | | | | | | |

Capture sample counts, warmup duration and repeated-run variance. Do not report percentiles without adequate samples.

## 6. Optional bounded staging run [LAB-MUTATE]

1. Verify isolated endpoint, synthetic inputs, quota, GPU budget and stop conditions.
2. Inspect installed benchmark tool's supported CLI and choose a reproducible test harness.
3. Run warm baseline for B1–B8 within limits; record request/response/token distribution.
4. Record GPU/CPU, KV usage, preemption, queue, TTFT/ITL and SLO goodput.
5. Change **one** supported variable (e.g., a modest concurrency limit or scheduler token budget) through TrueFoundry/GitOps.
6. Repeat identical matrix, compare SLO-qualified completions, tails, quality and cost.
7. Abort and roll back for OOM, SLO breach, rising errors/queue, stream failures or cost ceiling.
8. Restore immutable known-good revision and verify correct model, endpoints, latency and resource cleanup.

If benchmark execution is not approved, complete the tabletop and mark live results Not run.

## 7. Tuning decision worksheet

| Candidate lever | Hypothesis | Expected benefit | Risk | Validation |
|---|---|---|---|---|
| max_model_len | | | | |
| max_num_seqs | | | | |
| max_num_batched_tokens | | | | |
| gpu_memory_utilization | | | | |
| KV dtype | | | | |
| Prefix caching | | | | |
| Chunked prefill | | | | |
| Tensor parallelism | | | | |
| Quantization | | | | |
| Replica count | | | | |

Validate flag support for the pinned vLLM version; never change all levers together.

## 8. Failure tabletop

| ID | Scenario | LPHL | FFT | Evidence | Safe response |
|---|---|---|---|---|---|
| P1 | Queue-driven TTFT | | | | |
| P2 | Long-prompt TTFT | | | | |
| P3 | Decode ITL | | | | |
| P4 | KV exhaustion | | | | |
| P5 | Preemption | | | | |
| P6 | Startup OOM | | | | |
| P7 | Long-context OOM | | | | |
| P8 | Low GPU/high latency | | | | |
| P9 | High GPU/healthy SLO | | | | |
| P10 | Quantization quality regression | | | | |
| P11 | Tensor-parallel overhead | | | | |
| P12 | Prefix-cache miss | | | | |
| P13 | Batch hurts TTFT | | | | |
| P14 | Cold-start delay | | | | |
| P15 | Scale-down stream stall | | | | |
| P16 | Gateway 429 | | | | |
| P17 | High throughput/low goodput | | | | |
| P18 | Benchmark variance | | | | |
| P19 | GPU throttling | | | | |
| P20 | GPU ECC/XID | | | | |
| P21 | CPU tokenizer bottleneck | | | | |
| P22 | Transport backpressure | | | | |
| P23 | Retry storm | | | | |
| P24 | Release regression | | | | |

LPHL = Lowest Proven Healthy Layer; FFT = First Failed Transition. P9 is not a failure when SLOs are met.

## 9. Acceptance gates

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| A1 Baseline/revision/SLO | | |
| A2 Latency and goodput definitions | | |
| A3 Bottleneck classification | | |
| A4 Tuning tradeoffs | | |
| A5 Representative benchmark matrix | | |
| A6 Controlled staging test | | |
| A7 P1–P24 LPHL/FFT | | |
| A8 Canary/rollback/recovery | | |

## 10. Final evidence report

```text
Date/time, approval and owner:
Model/tokenizer/image/vLLM/GPU:
Runtime flags and KV budget:
Prompt/output distributions, concurrency and arrival rate:
B1–B8 runs, sample counts and variance:
TTFT/ITL/E2E percentiles, errors, SLO goodput:
GPU VRAM/utilization, KV/preemption, CPU/network:
One-variable diff, quality check and cost:
P1–P24 LPHL/FFT and A1–A8:
Rollback, cleanup, unexecuted gates:
```

**Next:** Chapter 8.6 — Scaling Observability, Cost and Troubleshooting.
