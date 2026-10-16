# Chapter 032 — Hands-On Lab: TensorRT-LLM Quantization and Engine Compatibility

**Part 05 | Chapter 032 | Published lab plan; live execution pending**

## Goal

Compare precision and quantization approaches, verify engine/runtime/GPU compatibility, estimate weight/KV/VRAM budgets, design representative calibration and holdout tests, evaluate TTFT/TPOT and quality, and plan safe promotion/rollback.

## Safety

**[SAFE-READ]** approved model metadata, GPU inventory and sanitized logs. **[LAB-MUTATE]** isolated staging with licensed synthetic/approved models and bounded GPU-hours. **[CHANGE-CONTROL]** production quantization, engine, backend, GPU driver and routing changes. No PHI, customer prompts, secrets or untrusted code.

## Lab 0 — Preflight

Record UTC, model/license, weights/tokenizer hashes, calibration/holdout dataset governance, TensorRT-LLM/TensorRT/CUDA/driver versions, GPU SKU, quant method, owner, approval, cost ceiling and rollback.

```bash
date -u
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used --format=csv
```

For an approved Kubernetes cluster:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
```

## Lab 1 — Precision compatibility [DESIGN]

| Method | Model support | GPU/kernel support | Calibration | Status |
|---|---|---|---|---|
| BF16 baseline | Verify | Verify | Usually not PTQ | Pending |
| FP8 | Verify | Verify | Method-dependent | Pending |
| INT8 | Verify | Verify | Method-dependent | Pending |
| INT4 weight-only | Verify | Verify | Method-dependent | Pending |
| KV-cache quant | Verify | Verify | Method-dependent | Pending |

Do not infer support solely from GPU marketing materials; validate the pinned runtime and model architecture.

## Lab 2 — Calibration and holdout [DESIGN]

Define synthetic short/long, multilingual, structured output, numerical and safety-sensitive cohorts. Separate calibration from holdout. Record provenance and retention without exposing prompt content.

## Lab 3 — Weight and KV memory worksheet [DESIGN]

For an illustrative 7B-parameter model:

```text
BF16 idealized weights ≈ 14 GB decimal
INT8 idealized weights ≈ 7 GB decimal
INT4 idealized weights ≈ 3.5 GB decimal
```

For a synthetic 32-layer, 8 KV-head, 128-head-dim BF16 model:

```text
KV/token = 2 × 32 × 8 × 128 × 2 = 131,072 bytes
32,768 resident tokens ≈ 4 GiB logical KV
```

Add scales, unquantized tensors, workspace, CUDA graphs, communication buffers, fragmentation and headroom.

## Lab 4 — Engine compatibility [DESIGN]

Record build GPU, serving GPU, SM capability, driver, CUDA, TensorRT, TensorRT-LLM, backend, plugins, max batch/context/token profiles, TP/PP and artifact digests. Identify nonportable combinations and required rebuilds.

## Lab 5 — Quality and performance matrix [DESIGN]

| Cohort | Quality | TTFT | TPOT | VRAM | Status |
|---|---|---|---|---|---|
| short/short | golden | p95 | p95 | peak | Pending |
| long/short | retrieval | p95 | p95 | peak | Pending |
| short/long | stop/EOS | p95 | p95 | peak | Pending |
| long/long | correctness | p99 | p99 | peak | Pending |
| structured/safety | schema/refusal | p95 | p95 | peak | Pending |
| mixed concurrency | completion | p99 | p99 | peak | Pending |

Compare with the same baseline hardware/model/tokenizer and record GPU-hours per valid completion.

## Lab 6 — Promotion and rollback [DESIGN]

Design a release bundle including weights, tokenizer, quantization scales, engine/runtime, config, checksums, evaluation results and known-good rollback. Do not rely on restoring the engine alone.

## Lab 7 — Eight tabletop game days

01 unsupported INT4; 02 long-context calibration gap; 03 JSON regression; 04 missing scales; 05 GPU mismatch; 06 rank mismatch; 07 TTFT/TPOT cost tradeoff; 08 rollback incompatibility.

## Twenty-four failure checklist

F01 method; F02 GPU kernel; F03 conversion; F04 scales; F05 calibration; F06 privacy; F07 tokenizer; F08 JSON; F09 safety; F10 long context; F11 build OOM; F12 target GPU; F13 plugin; F14 CUDA; F15 batch profile; F16 KV quality; F17 VRAM; F18 TP rank; F19 TTFT; F20 TPOT; F21 streaming; F22 telemetry; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Approved model/license | Pending |
| G02 | Source/tokenizer digests | Pending |
| G03 | Quant method support | Pending |
| G04 | Calibration provenance | Pending |
| G05 | Holdout quality | Pending |
| G06 | Structured output | Pending |
| G07 | Safety behavior | Pending |
| G08 | GPU precision support | Pending |
| G09 | Runtime matrix | Pending |
| G10 | Engine artifact digest | Pending |
| G11 | Target load/readiness | Pending |
| G12 | Context/batch profile | Pending |
| G13 | KV memory | Pending |
| G14 | Per-GPU VRAM | Pending |
| G15 | TP/PP health | Pending |
| G16 | TTFT/TPOT | Pending |
| G17 | Streaming completion | Pending |
| G18 | Privacy/telemetry | Pending |
| G19 | Rollback/FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/approval:
Model/license/weights/tokenizer digests:
Quant method/scales/calibration:
Holdout quality/safety/JSON:
Build and target GPU/runtime matrix:
Engine artifact digest:
Context/batch/KV/VRAM:
TP/PP and streaming:
TTFT/TPOT/throughput:
Privacy/telemetry:
GPU-hours/cost:
Game-day mode/findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop approved staging builds/load, remove only owned temporary artifacts, preserve immutable evidence and reconcile GPU-hours. Gates remain pending until independently reviewed.

**Next: Chapter 033 — TensorRT-LLM Throughput, Latency and KV Cache.**
