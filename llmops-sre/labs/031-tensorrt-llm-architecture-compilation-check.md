# Chapter 031 — Hands-On Lab: TensorRT-LLM Architecture and Compilation

**Part 05 | Chapter 031 | Published lab plan; live execution pending**

## Goal

Design a reproducible TensorRT-LLM model conversion/build/runtime workflow, verify GPU and software compatibility, calculate KV/VRAM capacity, define golden output and performance gates, and plan Triton integration and rollback.

## Safety

**[SAFE-READ]** approved model/GPU metadata and sanitized build logs. **[LAB-MUTATE]** isolated staging with licensed synthetic/approved models, bounded GPU-hours and approved build resources. **[CHANGE-CONTROL]** production engine, runtime, backend, GPU node, model routing and rollback. No PHI, customer prompts, secrets or untrusted model code.

## Lab 0 — Preflight

Record UTC, model/license, source revision/checksum, tokenizer/template, TensorRT-LLM/TensorRT/CUDA/driver, GPU SKU/compute capability, build/serve environments, owner, approval, cost ceiling and rollback.

```bash
date -u
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used --format=csv
```

On an approved Kubernetes cluster:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
```

## Lab 1 — Architecture [DESIGN]

Draw source model/tokenizer → conversion/checkpoint → versioned build or supported runtime preparation → artifact registry → TensorRT-LLM executor → optional Triton backend → authenticated gateway. Label ownership, GPU resources, failure domains and metrics.

## Lab 2 — Compatibility matrix [DESIGN]

| Component | Required evidence | Status |
|---|---|---|
| Model architecture | supported by pinned release | Pending |
| Tokenizer/chat template | pinned digest | Pending |
| GPU SKU/compute capability | target support | Pending |
| Driver/CUDA/TensorRT | compatible matrix | Pending |
| TensorRT-LLM | pinned build/runtime | Pending |
| Precision/quantization | kernel/model support | Pending |
| TP/PP topology | ranks and NCCL | Pending |
| Triton backend | matching integration | Pending |

## Lab 3 — Reproducible build plan [DESIGN]

Create an immutable build manifest with model/tokenizer digests, precision, context/batch/token limits, GPU architecture, runtime versions, build artifact digest, provenance, golden fixtures and rollback. Use CLI `--help` from the pinned release; do not copy flags from a different version.

## Lab 4 — KV and GPU memory [DESIGN]

For a conventional synthetic 32-layer model with 8 KV heads, 128 head dimension and BF16:

```text
KV/token = 2 × 32 × 8 × 128 × 2 = 131,072 bytes
32,768 resident tokens ≈ 4 GiB logical KV
```

Add weights, activations, build/runtime workspace, CUDA graphs, buffers, fragmentation and headroom. Recalculate for actual model and TP/PP topology.

## Lab 5 — Golden fixtures and streaming [DESIGN]

Define approved synthetic prompts and expected invariants: tokenizer/template, stop/EOS, output shape/format, refusal/security behavior where applicable, full streaming terminal marker, cancellation and token accounting. Quality must be verified after precision changes.

## Lab 6 — Performance and FinOps [DESIGN]

Plan short/medium/long prompt cohorts and bounded concurrency. Record TTFT, TPOT, E2E, output tokens/s, per-rank VRAM, build GPU-hours, serving GPU-hours and cost per valid completion. Mark unmeasured results Pending.

## Lab 7 — Eight tabletop game days

01 conversion mismatch; 02 GPU engine incompatibility; 03 quantization quality regression; 04 build OOM; 05 KV exhaustion; 06 rank failure; 07 incomplete streaming; 08 rollback incompatibility.

## Twenty-four failure checklist

F01 architecture; F02 weights digest; F03 tokenizer; F04 precision; F05 conversion; F06 plugin; F07 host RAM; F08 build GPU OOM; F09 engine portability; F10 software matrix; F11 artifact corruption; F12 wrong output; F13 stop/EOS; F14 context; F15 startup OOM; F16 KV; F17 TP ranks; F18 NCCL; F19 TTFT; F20 TPOT; F21 partial stream; F22 telemetry; F23 rollback; F24 GPU cost.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Approved model/license | Pending |
| G02 | Weights digest | Pending |
| G03 | Tokenizer/template | Pending |
| G04 | Pinned runtime | Pending |
| G05 | GPU compatibility | Pending |
| G06 | Precision support | Pending |
| G07 | Conversion validation | Pending |
| G08 | Build reproducibility | Pending |
| G09 | Artifact digest | Pending |
| G10 | Runtime readiness | Pending |
| G11 | Golden fixtures | Pending |
| G12 | Context/KV budget | Pending |
| G13 | GPU memory | Pending |
| G14 | TP/PP health | Pending |
| G15 | TTFT/TPOT | Pending |
| G16 | Complete streaming | Pending |
| G17 | Security/privacy | Pending |
| G18 | Telemetry/SLO | Pending |
| G19 | Rollback/FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/approval:
Model/license/weights/tokenizer digests:
GPU SKU/compute capability:
TensorRT-LLM/TensorRT/CUDA/driver:
Conversion/build/runtime path:
Precision/context/batch/TP/PP:
Artifact digest/registry:
Golden fixtures/streaming:
KV/VRAM/TTFT/TPOT:
Security/telemetry/SLO:
Build and serving GPU-hours:
Game-day mode/findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop separately approved staging builds/load, remove only owned temporary artifacts, preserve immutable evidence and reconcile GPU-hours. Gates remain pending until independently reviewed.

**Next: Chapter 032 — TensorRT-LLM Quantization and Engine Compatibility.**
