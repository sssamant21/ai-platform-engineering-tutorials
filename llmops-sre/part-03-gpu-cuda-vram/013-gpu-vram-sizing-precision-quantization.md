# Chapter 013 — GPU VRAM Sizing, Precision and Quantization

**Part 03 — GPU Architecture, CUDA and VRAM | Chapter 013 of 100**

> **Status:** Written tutorial published. Technical/security review, live lab execution and acceptance pending. All calculations are planning examples, not measurements from a deployed GPU.

## Learning objectives

Build a defensible GPU memory budget; distinguish weights, KV cache, activations, temporary workspace and allocator overhead; compare BF16, FP16, FP8, INT8 and 4-bit quantization; explain calibration and accuracy tradeoffs; size concurrency with headroom; investigate CUDA OOM and fragmentation; and evaluate performance, reliability and GPU cost.

## 1. GPU memory as a production constraint

```text
GPU VRAM
 ├─ model weights and embeddings
 ├─ KV cache (prompt + generated tokens for active sequences)
 ├─ prefill/decode activations and temporary workspaces
 ├─ CUDA graphs, kernels, runtime buffers and communication
 ├─ allocator blocks / fragmentation / loading peaks
 └─ safety headroom for bursts and maintenance
```

Model parameter count is only the starting point. Inference serving requires memory beyond weights, and real usable capacity depends on runtime implementation, GPU architecture, quantization scheme, tensor parallelism and context/concurrency distribution.

## 2. Weight-memory baseline

For a dense model with `P` stored parameters and an ideal fixed precision of `b` bits per parameter:

```text
raw_weight_bytes ≈ P × b / 8
```

Illustrative 7-billion-parameter model (decimal GB):

| Storage | Ideal raw weights | Important caveat |
|---|---:|---|
| FP32 (32-bit) | 28 GB | often not used for serving |
| BF16/FP16 (16-bit) | 14 GB | common weight baseline |
| FP8/INT8 (8-bit) | 7 GB | scaling metadata, kernel support |
| 4-bit | 3.5 GB | group scales, zero points, packing |

These are **lower-bound raw payload estimates**, not actual loaded VRAM. Quantized formats add metadata and sometimes leave layers in higher precision; frameworks may maintain additional copies or buffers.

## 3. Precision versus quantization

**Precision** describes numeric representation and dynamic range; **quantization** maps values to lower-bit representations, often with scales and sometimes zero points. Quantization may be weight-only, weight-and-activation, per-channel, per-group, static or dynamic.

| Format/approach | Potential benefit | Production risk |
|---|---|---|
| BF16 | broad exponent range, common modern training/inference | device/kernel support |
| FP16 | smaller memory than FP32 | overflow/underflow sensitivity |
| FP8 | lower memory and faster supported kernels | scaling/calibration, hardware support |
| INT8 | lower weight/activation footprint | calibration and quality |
| 4-bit weight-only | significant weight reduction | metadata, dequantization overhead, quality drift |
| Quantized KV cache | lower cache footprint | attention-kernel support and accuracy |

Do not assume FP8 and INT8 have identical performance, or that 4-bit always increases tokens/s. Dequantization, batch size, kernel fusion and bandwidth can reverse the expected advantage.

## 4. Quantization quality gates

Before promotion, compare candidate and approved baseline on a representative, **non-sensitive**, versioned evaluation set:
- Task success and answer correctness.
- Structured-output/JSON schema validity.
- Tool-calling and function argument correctness.
- Multilingual and long-context behavior.
- Safety and refusal behavior.
- TTFT, TPOT, tokens/s and complete-stream success.
- Memory footprint, GPU cost and error rate.

Use statistical confidence intervals where appropriate; a few prompts are insufficient to prove equivalence. Pin model, tokenizer, chat template, sampling, quantizer, calibration dataset, framework, kernels and GPU SKU. If model quality falls outside approved tolerance, reject the optimization.

## 5. KV-cache sizing

For conventional attention with `L` layers, `Hkv` KV heads, head dimension `D` and `s` bytes/element:

```text
KV_bytes_per_token ≈ 2 × L × Hkv × D × s
KV_logical_bytes ≈ KV_bytes_per_token × resident_tokens
```

Example with 32 layers, 8 KV heads, 128 head dimension and BF16 (2 bytes):

```text
2 × 32 × 8 × 128 × 2 = 131,072 bytes/token
8,192 tokens ≈ 1 GiB logical KV
32,768 tokens ≈ 4 GiB logical KV
```

Resident tokens include concurrent active sequences, not merely the maximum context length of one request. Real runtimes use paged allocation, prefix sharing, sliding windows, tensor parallelism, cache quantization or offload. Validate runtime-specific block metrics and device allocation.

## 6. Practical VRAM budget

```text
required_vram =
  weights_loaded
  + kv_cache_allocated
  + activations_workspace
  + runtime_cuda_graphs
  + allocator_fragmentation
  + safety_headroom
```

Illustrative **planning** budget for a 24 GiB GPU:

| Component | GiB |
|---|---:|
| BF16 weight allocation (hypothetical) | 13.0 |
| KV-cache allocation | 5.0 |
| workspace/runtime buffers | 2.0 |
| fragmentation/loading reserve | 1.0 |
| safety headroom | 3.0 |
| **Total** | **24.0** |

This is a synthetic budget, not a validated configuration. Actual weight allocation can differ from raw model size, and the runtime may reserve most available memory for KV blocks. GPU OOM risk can persist even when a coarse spreadsheet balances exactly.

## 7. Concurrency and admission

A useful approximation for homogeneous requests is:

```text
active_sequences × average_resident_tokens × KV_bytes_per_token
    ≤ allocatable_KV_cache_bytes
```

For heterogeneous workloads, use token-length distributions and scheduler behavior. Maximum context length × maximum concurrency is often too conservative for typical load yet necessary for worst-case admission guarantees. Explicitly decide whether to queue, reject or preempt; never silently truncate system instructions.

A model fitting on a GPU does **not** prove that the intended concurrency, TTFT/TPOT or p99 SLO is achievable.

## 8. OOM and fragmentation diagnosis

Differentiate:
- **Load-time OOM:** weights and initialization/workspace do not fit.
- **Prefill OOM:** large prompts, temporary attention workspace or long contexts.
- **Decode OOM:** growing KV cache and active sequences.
- **Allocator failure:** fragmentation, incompatible graph reservations or stale allocations.
- **Distributed imbalance:** one tensor-parallel rank exceeds its device budget.
- **Driver/hardware fault:** CUDA errors unrelated to capacity.

Observe allocation failures, runtime cache blocks, GPU memory used/free, preemption, request cohorts, scheduler concurrency, kernel/workspace and recent deployments. A restart may recover memory but does not resolve the underlying capacity model.

## 9. Read-only investigation

**[SAFE-READ]** on an approved cluster:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

On an authorized GPU host or approved debug environment:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,memory.free,utilization.gpu --format=csv
```

For a verified workload:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Sanitize logs. Device-wide memory use may include other processes; do not assume all VRAM belongs to one model. Exporter metric names vary by runtime/version.

## 10. Quantization rollout playbook

1. Freeze baseline model, tokenizer, prompts and GPU environment.
2. Choose a hardware- and runtime-supported quantization format.
3. Produce immutable quantized artifacts with provenance and calibration details.
4. Verify loader/kernel support and CUDA/driver compatibility.
5. Compare representative accuracy, safety, JSON and tool-calling evaluations.
6. Benchmark TTFT, TPOT, throughput, GPU VRAM and complete-stream success.
7. Apply a bounded staging concurrency and context matrix.
8. Document rollback artifact and controller owner.
9. Canary only under approved change control and monitor quality + SLOs.
10. Revert if correctness, safety, tail latency or memory thresholds fail.

## 11. FinOps tradeoffs

Lower VRAM can allow a smaller GPU SKU or higher density, but quantization may increase latency, reduce quality, require more retries or raise operational complexity. Model **cost per successful customer request** and GPU-hours per million valid output tokens, including idle capacity, retries, model-loading downtime and failure recovery.

## 12. Twenty production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | BF16 model load OOM | weight allocation |
| F02 | FP8 kernel unsupported | GPU/framework matrix |
| F03 | INT8 calibration regression | eval baseline |
| F04 | 4-bit loader incompatible | artifact metadata |
| F05 | quantized layer falls back to FP16 | actual memory footprint |
| F06 | dequantization hurts TPOT | kernel profile |
| F07 | quantization breaks JSON | structured-output eval |
| F08 | quantization harms tool calls | tool-call eval |
| F09 | long context exceeds KV | cache occupancy |
| F10 | concurrent decode OOM | resident tokens |
| F11 | prefill workspace OOM | prompt cohort |
| F12 | CUDA graph reserve OOM | graph/workspace |
| F13 | allocator fragmentation | runtime allocator logs |
| F14 | GPU process memory contention | device process inventory |
| F15 | TP rank memory imbalance | per-device usage |
| F16 | KV quantization quality drift | eval and cache format |
| F17 | silent context truncation | admission policy |
| F18 | preemption thrash | runtime preemption counters |
| F19 | high VRAM but low tokens/s | scheduler/bandwidth |
| F20 | GPU cost rises after quantization | good-request cost |

## 13. Fourteen operational runbooks

1. Inventory GPU VRAM and serving artifacts.
2. Calculate ideal and actual weight footprints.
3. Validate quantization format/hardware support.
4. Audit calibration and quality gates.
5. Investigate load-time OOM.
6. Investigate prefill workspace OOM.
7. Investigate decode/KV-cache OOM.
8. Investigate allocator fragmentation.
9. Diagnose tensor-parallel memory imbalance.
10. Investigate quantized kernel performance.
11. Triage structured-output/tool-call regressions.
12. Review admission, concurrency and preemption.
13. Compare cost per successful request.
14. Execute approved rollback and customer-path verification.

## 14. Eight game days

1. Model upgrade causes load-time OOM.
2. Quantized artifact loads but JSON validity falls.
3. Longer contexts exhaust KV cache.
4. Concurrent decode causes preemption thrash.
5. FP8 kernel unsupported on one GPU SKU.
6. Allocator fragmentation after prolonged load.
7. Tensor-parallel rank OOM.
8. Quantization saves VRAM but worsens cost per good request.

Default **tabletop**; real fault injection or benchmarking requires approved staging.

## 15. Eighteen acceptance gates

G01 GPU memory inventory; G02 weight math; G03 precision distinctions; G04 quantization formats; G05 calibration provenance; G06 accuracy/safety eval; G07 structured output/tool eval; G08 KV sizing; G09 concurrency/admission; G10 workspace/headroom; G11 CUDA OOM triage; G12 fragmentation; G13 multi-GPU memory; G14 read-only diagnostics; G15 performance benchmark design; G16 FinOps; G17 rollback/recovery; G18 reviewer signoff.

See [companion lab](../labs/013-gpu-vram-sizing-precision-quantization-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 014 — Multi-GPU Parallelism and Topology.**
