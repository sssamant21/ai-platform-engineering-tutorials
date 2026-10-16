# Chapter 013 — Hands-On Lab: GPU VRAM Sizing, Precision and Quantization

**Part 03 | Chapter 013 | Published lab plan; execution pending**

## Goal

Build a reproducible VRAM worksheet, compare precision/quantization footprints, estimate KV-cache concurrency, design quality/performance gates and prepare an OOM incident response plan.

## Safety

**[SAFE-READ]** approved Kubernetes/GPU inventory, redacted runtime metrics and release manifests. **[LAB-MUTATE]** approved staging synthetic inference and quantized-model benchmark only. **[CHANGE-CONTROL]** production model/quantization/GPU/concurrency changes. Never collect customer prompts, PHI, credentials or proprietary model artifacts.

## Lab 0 — Preflight

Record UTC, context, namespace, GPU SKU, model parameter count, model/tokenizer/runtime versions, quantization artifact provenance, driver/CUDA compatibility, evaluator, cost ceiling and rollback owner.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — Weight footprint worksheet [DESIGN]

For synthetic 7-billion-parameter dense model:

```text
FP32: 7e9 × 4 bytes = 28 GB
BF16/FP16: 7e9 × 2 bytes = 14 GB
FP8/INT8: 7e9 × 1 byte = 7 GB
4-bit: 7e9 × 0.5 byte = 3.5 GB
```

Note decimal GB, metadata overhead, higher-precision exceptions and runtime allocations. Do not call these measured VRAM.

## Lab 2 — KV-cache worksheet [DESIGN]

For 32 layers, 8 KV heads, head dimension 128 and 2 bytes/element:

```text
KV bytes/token = 131,072 bytes = 128 KiB
8,192 resident tokens ≈ 1 GiB logical KV
32,768 resident tokens ≈ 4 GiB logical KV
```

Repeat for 1-byte KV representation **only if supported** and explain why accuracy/kernel support must be tested. Document cache block rounding, sharding, sharing and sliding-window caveats.

## Lab 3 — Full VRAM budget [DESIGN]

Synthetic 24 GiB budget:

| Component | GiB |
|---|---:|
| Loaded weights | 13 |
| KV allocation | 5 |
| Workspace/runtime | 2 |
| Fragmentation/loading reserve | 1 |
| Safety headroom | 3 |
| **Total** | **24** |

Identify what to measure on the actual runtime and how to handle missing metrics. Do not size a production GPU from this example alone.

## Lab 4 — Approved read-only evidence

On an authorized GPU node:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,memory.free,utilization.gpu --format=csv
```

For a verified deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Collect redacted cache occupancy, preemption, OOM, TTFT/TPOT and GPU utilization if available. Never assume `nvidia-smi` alone explains allocator fragmentation.

## Lab 5 — Quantization evaluation plan [DESIGN]

Compare BF16 baseline with one runtime-supported lower-precision candidate. Pin model/tokenizer/template, calibration provenance, evaluation fixtures, sampling, GPU SKU and image digest. Measure correctness, safety, JSON validity, tool calls, multilingual/long-context performance, TTFT, TPOT, complete SSE, VRAM and GPU-hours. Define approval thresholds before tests.

## Lab 6 — Concurrency and cost [DESIGN]

Estimate KV consumption from resident-token distributions, not just request count. Build a matrix for short/long prompts, short/long outputs, active sequences, queue, KV blocks, preemptions, p95 latency and GPU-hours per valid request. Optional synthetic staging execution requires separate authorization and abort thresholds.

## Lab 7 — Eight tabletop game days

01 load-time OOM; 02 JSON regression; 03 KV exhaustion; 04 preemption thrash; 05 unsupported FP8; 06 fragmentation; 07 tensor-parallel imbalance; 08 worse cost despite smaller weights.

## Twenty-scenario checklist

F01 weight OOM; F02 FP8; F03 INT8 quality; F04 4-bit loader; F05 FP16 fallback; F06 dequantization; F07 JSON; F08 tools; F09 context; F10 concurrency; F11 prefill; F12 CUDA graphs; F13 fragmentation; F14 contention; F15 rank imbalance; F16 KV quality; F17 truncation; F18 preemption; F19 throughput; F20 FinOps.

## Eighteen acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | GPU memory inventory | Pending |
| G02 | Weight arithmetic | Pending |
| G03 | Precision differences | Pending |
| G04 | Quantization formats | Pending |
| G05 | Calibration provenance | Pending |
| G06 | Accuracy/safety evaluation | Pending |
| G07 | JSON/tool evaluation | Pending |
| G08 | KV-cache sizing | Pending |
| G09 | Concurrency/admission | Pending |
| G10 | Workspace/headroom | Pending |
| G11 | CUDA OOM triage | Pending |
| G12 | Fragmentation diagnosis | Pending |
| G13 | Multi-GPU memory | Pending |
| G14 | Authorized diagnostics | Pending |
| G15 | Benchmark design | Pending |
| G16 | FinOps comparison | Pending |
| G17 | Rollback/recovery | Pending |
| G18 | Reviewer signoff | Pending |

## Evidence template

```text
UTC/context/approval:
GPU SKU and available VRAM:
Model/tokenizer/runtime/quantizer versions:
Raw weights versus measured allocation:
KV assumptions and resident-token cohorts:
Workspace, fragmentation and headroom:
Quality/safety/JSON/tool evaluation:
TTFT/TPOT/complete SSE and cost:
Game-day mode, incident findings:
Reviewer, rollback, acceptance:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop separately approved staging load, remove only owned resources, reconcile GPU-hours and preserve sanitized evidence. All gates remain pending until reviewed.

**Next: Chapter 014 — Multi-GPU Parallelism and Topology.**
