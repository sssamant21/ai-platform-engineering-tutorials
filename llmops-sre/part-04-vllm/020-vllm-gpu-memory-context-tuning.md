# Chapter 020 — vLLM GPU Memory and Context Tuning

**Part 04 — vLLM Administration and Operations | Chapter 020 of 100**

> **Status:** Written tutorial published; technical/security review, live lab execution and operational acceptance pending. vLLM options and metrics vary by version. Confirm exact behavior in the pinned release and hardware.

## Learning objectives

Construct a GPU VRAM budget for model weights, KV cache, runtime workspace and CUDA graphs; understand context length versus concurrency; tune vLLM memory utilization, maximum model length, active sequences and batched token limits; diagnose CUDA OOM, fragmentation, preemption and p99 latency; define safe experiments, rollback and GPU FinOps gates.

## 1. GPU memory architecture

```text
GPU HBM total
 ├─ model weights (dtype / quantization / sharding)
 ├─ KV cache (resident prompt + generated tokens)
 ├─ prefill/decode workspace and activations
 ├─ CUDA graph / kernel / communication buffers
 ├─ allocator reserve and fragmentation
 ├─ other GPU processes / driver overhead
 └─ unallocated safety headroom
```

The allocation balance changes with model architecture, vLLM release, GPU SKU, quantization, TP/PP, concurrency and input/output distributions. GPU utilization and GPU memory utilization are different signals.

## 2. Context window is not capacity

**Maximum model context** limits tokens a single sequence can admit, subject to model and runtime constraints. **Concurrent resident tokens** are the sum of active KV states and are bounded by KV allocation, scheduler policy and latency SLOs. A model supporting 32K context does not imply that 32 simultaneous 32K sequences will fit.

Admission must consider prompt tokens, requested output cap, model limits and runtime token budget. Validate tokenizer/chat template overhead and reject impossible requests predictably instead of silently truncating.

## 3. KV cache arithmetic

For conventional decoder self-attention:

```text
KV bytes per token ≈ 2 × layers × KV_heads × head_dim × bytes_per_element
logical resident KV ≈ bytes_per_token × resident_token_count
```

Synthetic example: 32 layers, 8 KV heads, head dimension 128, BF16 (2 bytes):

```text
KV/token = 2 × 32 × 8 × 128 × 2 = 131,072 bytes = 128 KiB
8,192 tokens ≈ 1 GiB logical KV
32,768 tokens ≈ 4 GiB logical KV
65,536 tokens ≈ 8 GiB logical KV
```

This excludes allocator block rounding, metadata, model-specific attention, prefix sharing, KV dtype differences and TP/PP distribution. Always use the actual model configuration and per-rank allocation when sizing production.

## 4. Weight and KV tradeoffs

| Lever | Benefit | Risk |
|---|---|---|
| Lower weight precision | frees VRAM for KV | quality/kernel compatibility |
| Weight quantization | smaller weight footprint | accuracy and throughput variation |
| KV cache dtype | smaller KV footprint | numerical quality and support |
| Shorter max context | lower worst-case KV pressure | rejects legitimate workloads |
| Lower concurrency | less KV contention | higher queue/TTFT |
| Lower batched-token budget | smaller prefill bursts | lower throughput, longer queue |
| More GPUs/TP | weight fit | collectives and cost |
| Prefix caching | prefill reuse | eviction and tenant isolation |
| Additional replicas | throughput/isolation | duplicate weights and GPU-hours |

Changing one setting can improve one cohort while hurting another. Benchmark complete-stream success, p95/p99 TTFT, TPOT, throughput, quality and GPU cost together.

## 5. vLLM runtime knobs

Common configuration families include model context length, GPU memory utilization target, maximum active sequences, maximum batched tokens, KV cache dtype, quantization, tensor/pipeline parallelism and prefix caching.

A **conceptual** staging launch command (verify flags with the pinned version):

```bash
vllm serve "APPROVED_MODEL_ID_OR_LOCAL_PATH" \
  --served-model-name "APPROVED_MODEL_ALIAS" \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.85
```

These numeric values are **illustrative, not recommended defaults**. Do not run outside an approved staging GPU environment. Consult `vllm serve --help` for version-specific flags and support. Memory utilization targets are not hard OOM guarantees.

## 6. Synthetic memory envelope

For an illustrative GPU with 80 GiB physical HBM, assume:

```text
Weights                  28 GiB
Workspace/graphs/buffers  8 GiB
KV allocation            34 GiB
Safety headroom           6 GiB
Other processes/driver    4 GiB
Total                    80 GiB
```

This is a budgeting worksheet, not a measured allocation. The categories can overlap depending on runtime reporting. Use actual device accounting, per-rank measurements and release-specific memory profiling before approval.

At 128 KiB logical KV/token, 34 GiB could theoretically hold about 278,528 tokens, before block rounding, metadata and other runtime constraints. This is not a supported throughput or concurrency claim.

## 7. Scheduler pressure and preemption

Increasing concurrency can improve GPU occupancy until KV or compute saturates. Beyond that point, queueing, preemption, recomputation, tail latency and failed streams can grow sharply.

Capture active/waiting requests, preemption events, KV occupancy, GPU VRAM, TTFT, TPOT, context buckets and successful output tokens/s. When telemetry is unavailable, do not infer that zero preemptions occurred.

## 8. Diagnosing OOM by phase

| Phase | Symptom | Hypothesis |
|---|---|---|
| startup | fails before ready | weight, workspace, CUDA graphs |
| prefill | large prompt fails | temporary workspace, context |
| decode | fails at high concurrency | resident KV exhaustion |
| multi-GPU | one rank fails | per-rank imbalance |
| rollout | new Pod cannot fit | duplicate model GPU capacity |
| after model update | VRAM rises | revision/dtype/config drift |

An OOM may originate from another GPU process or allocator fragmentation, even when a dashboard shows apparent free memory. Do not blindly increase GPU utilization target.

## 9. Read-only diagnostics

**[SAFE-READ]** in an authorized context:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

For a verified workload:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

On an approved GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Inspect runtime KV/preemption metrics from the actual pinned exporter schema; labels and names vary by version. Sanitize all evidence.

## 10. Safe tuning methodology

1. Pin model/tokenizer, image digest, driver/CUDA, GPU SKU and workload fixtures.
2. Baseline short/long prompt and short/long output cohorts with complete SSE validation.
3. Record current VRAM, KV occupancy, queue, TTFT/TPOT, preemption and cost.
4. Change **one approved configuration dimension at a time** in isolated staging.
5. Sweep bounded concurrency and context cohorts with abort thresholds.
6. Require quality/structured-output regression tests when dtype, KV precision or quantization changes.
7. Compare p50/p95/p99 and throughput, not only averages.
8. Stop on OOM, incomplete streams, runaway queue, SLO burn, stale metrics or GPU-hour ceiling.
9. Restore known-good config via authoritative GitOps controller.
10. Verify sustained full-stream recovery and document decision.

## 11. Failure decision tree

```text
GPU memory / context issue?
 ├─ load OOM → weights, dtype, workspace, graphs
 ├─ prefill OOM → prompt length, temporary buffers
 ├─ decode OOM → KV resident tokens, concurrency
 ├─ context rejection → max length/tokenizer overhead
 ├─ high TTFT → queue, prefill, token budget
 ├─ high TPOT → decode, HBM, cache pressure
 ├─ preemption spikes → oversubscription
 ├─ cache hit falls → revision/prefix/eviction
 ├─ rank-specific OOM → TP shard imbalance
 ├─ OOM after rollout → model/config/driver drift
 ├─ memory high, throughput low → batching/cost
 └─ missing KV metrics → schema/freshness
```

## 12. Twenty-four production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | weight load OOM | startup GPU logs |
| F02 | CUDA graph workspace OOM | initialization |
| F03 | long-prompt prefill OOM | prompt cohort |
| F04 | decode KV OOM | active resident tokens |
| F05 | context limit too low | admission errors |
| F06 | context limit too high | KV pressure |
| F07 | max sequences too high | preemption |
| F08 | max sequences too low | queue/throughput |
| F09 | batched tokens too high | TTFT/OOM |
| F10 | batched tokens too low | throughput |
| F11 | GPU utilization target too high | no headroom |
| F12 | GPU utilization target too low | unused VRAM |
| F13 | unsupported KV dtype | kernel/runtime |
| F14 | quantization quality regression | evaluation |
| F15 | block rounding overhead | cache allocation |
| F16 | fragmentation/allocator OOM | runtime logs |
| F17 | another GPU process consumes HBM | process inventory |
| F18 | TP rank memory imbalance | per-device metrics |
| F19 | prefix cache eviction churn | cache hit ratio |
| F20 | tenant starvation | per-tenant queue |
| F21 | HTTP 200 partial SSE under load | completion SLI |
| F22 | telemetry stale | scrape timestamp |
| F23 | rollout cannot reserve GPUs | scheduler events |
| F24 | GPU cost per valid request increases | FinOps |

## 13. Sixteen operational runbooks

1. Verify release, model and GPU memory baseline.
2. Calculate KV bytes/token from model architecture.
3. Build per-GPU weight/KV/workspace budget.
4. Diagnose load-time OOM.
5. Diagnose prefill OOM.
6. Diagnose decode/KV OOM.
7. Investigate context limit errors.
8. Investigate sequence concurrency and queue.
9. Investigate batched-token scheduling.
10. Investigate preemption and recomputation.
11. Validate dtype/quantization quality.
12. Investigate fragmentation/other GPU processes.
13. Diagnose TP rank imbalance.
14. Diagnose incomplete SSE and SLO burn.
15. Execute controlled tuning and rollback.
16. Verify FinOps and sustained recovery.

## 14. Eight tabletop game days

1. New model revision exceeds VRAM at startup.
2. Long prompt causes prefill OOM.
3. Decode concurrency exhausts KV blocks.
4. Context limit admits more tokens than cache can sustain.
5. Increased GPU memory target causes instability.
6. Quantization fits model but breaks JSON quality.
7. One TP rank OOMs while others remain healthy.
8. Tuning improves tokens/s but worsens p99 and GPU cost.

Default **tabletop**; live stress/fault injection requires approved staging.

## 15. Twenty acceptance gates

G01 pinned release; G02 GPU inventory; G03 weight budget; G04 KV bytes/token; G05 runtime buffers/headroom; G06 context admission; G07 concurrency envelope; G08 batched-token tradeoff; G09 preemption evidence; G10 dtype/quantization evaluation; G11 prefix cache isolation; G12 OOM triage; G13 TP rank balance; G14 TTFT/TPOT; G15 complete SSE; G16 metric freshness; G17 safe tuning experiment; G18 rollback; G19 GPU FinOps; G20 independent review.

See [companion lab](../labs/020-vllm-gpu-memory-context-tuning-check.md). All gates remain pending until executed and independently reviewed.

**Next: Chapter 021 — vLLM Tensor/Pipeline Parallelism and Distributed Serving.**
