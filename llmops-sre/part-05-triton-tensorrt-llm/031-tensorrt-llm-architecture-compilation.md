# Chapter 031 — TensorRT-LLM Architecture and Compilation

**Part 05 — NVIDIA Triton and TensorRT-LLM | Chapter 031 of 100**

> **Status:** Written tutorial published; live labs, independent technical/security review and operational acceptance pending. TensorRT-LLM CLI names, build workflows, engine portability and backend integration vary by release: pin the exact version and consult its bundled documentation before executing.

## Learning objectives

Explain TensorRT-LLM's optimized inference architecture; distinguish model conversion, checkpoint preparation, compilation/build and runtime execution; understand attention kernels, KV cache, in-flight batching, quantization and distributed parallelism; design a reproducible build pipeline, compatibility matrix, GPU memory budget, Triton integration, acceptance tests and rollback.

## 1. What TensorRT-LLM is

NVIDIA TensorRT-LLM is an optimized inference framework/runtime for supported large language models on NVIDIA GPUs. Depending on release and integration, it uses TensorRT engines or other supported execution paths, model-specific kernels and optimized scheduling. **Do not assume every release requires the same standalone engine build** or that an engine built for one GPU/software stack is portable to another.

Triton is a serving platform; TensorRT-LLM is an inference runtime/backend integration. vLLM is a separate LLM serving stack. Choose by validated model/GPU support, performance, feature compatibility and operations.

## 2. Architecture

```text
Approved model weights + tokenizer + config
                    │
        Model conversion / checkpoint
                    │
       Version-pinned build pipeline
          ├─ dtype / quantization
          ├─ max context / batch / tokens
          ├─ attention / KV cache
          └─ tensor/pipeline parallelism
                    │
       Compiled or supported runtime artifact
                    │
       TensorRT-LLM executor / runtime
          ├─ request admission / scheduling
          ├─ prefill and decode execution
          ├─ KV cache management
          └─ GPU kernels / NCCL if distributed
                    │
      Triton TensorRT-LLM backend (optional)
                    │
        Authenticated gateway / clients
                    │
         Metrics, tracing, SLO, FinOps
```

This is a conceptual lifecycle. Modern releases can offer multiple supported deployment and execution paths. Record the exact chosen path.

## 3. Compile/build versus serve

| Stage | Inputs | Outputs | Operational concern |
|---|---|---|---|
| Model qualification | architecture, license, tokenizer | approved support matrix | unsupported ops/features |
| Conversion/checkpoint | source weights and config | runtime-compatible checkpoint | tensor mapping/precision |
| Build/optimization | checkpoint, target GPU, runtime config | optimized artifact/engine where applicable | build memory, time, compatibility |
| Runtime initialization | artifact, plugins, CUDA/NCCL | ready workers | GPU VRAM and rank health |
| Serving | prompts, sampling, token budgets | complete generated responses | TTFT, TPOT, fairness |
| Release promotion | tests, provenance, rollback | immutable approved release | drift and reproducibility |

A successful build is not proof of semantic correctness, tokenizer compatibility, safe streaming or production SLOs.

## 4. Compilation and graph optimization

Where an engine-build workflow is used, compilation can select kernels, precision, tensor layouts, plugins and execution profiles for supported shapes and GPU architectures. Engine size, build time and peak build memory can be substantial. Build on an approved compatible environment and record architecture/driver/CUDA/TensorRT/TensorRT-LLM versions.

Treat compilation flags as a **versioned API**. Do not copy options from another release without checking its `--help`, installed examples and compatibility matrix.

## 5. Build manifest

```yaml
release_id: trtllm-model-staging-001
source_model_revision: PINNED
source_weights_sha256: VERIFIED
tokenizer_revision: PINNED
tokenizer_sha256: VERIFIED
model_architecture: APPROVED
model_license: APPROVED
tensorrt_llm_version: PINNED
tensorrt_version: PINNED
cuda_version: PINNED
driver_version: PINNED
gpu_sku_compute_capability: VERIFIED
precision: bf16_or_supported
quantization: none_or_approved
max_context_tokens: BOUNDED
max_batch_size: BOUNDED
max_num_tokens: BOUNDED
tensor_parallel_size: 1
pipeline_parallel_size: 1
build_artifact_digest: VERIFIED
serving_backend_version: PINNED
rollback_artifact: KNOWN_GOOD
```

This is an evidence template, not a runnable command.

## 6. Model and tokenizer compatibility

Verify model architecture, attention layout, rotary embeddings/position encoding, vocabulary, special token IDs, chat template, EOS/stop semantics, max context, multimodal components if used and tool/structured-output contracts. A model can load yet produce incorrect results if tokenizer/template revisions drift.

Use synthetic golden fixtures and quality/regression checks in addition to numeric performance.

## 7. Precision and quantization

FP16/BF16, FP8, INT8 and INT4 or other methods are not interchangeable. Support depends on model, GPU architecture, calibration/quantization workflow, kernel path and release. Quantization can lower weight memory or increase throughput but may change output quality and memory/workspace needs. **Chapter 032** covers detailed quantization and compatibility.

Do not promise memory savings or speedups without per-GPU measurement and quality gates.

## 8. KV cache and context budget

Approximate attention KV memory for a conventional transformer:

```text
KV bytes/token ≈ 2 × layers × KV_heads × head_dim × bytes/element
```

Synthetic example: 32 layers × 8 KV heads × 128 head dimension × BF16(2 bytes) × 2:

```text
KV per token = 131,072 bytes = 128 KiB
32,768 resident tokens ≈ 4 GiB logical KV
```

Actual allocations differ with tensor/pipeline parallelism, paged KV, cache precision, allocator fragmentation, speculative decoding, context reuse and backend configuration. Include weights, activations, CUDA graphs, workspaces, NCCL buffers and safety headroom.

## 9. Prefill and decode execution

Prefill processes prompt tokens and often stresses compute and memory bandwidth differently from iterative decode. Measure TTFT and TPOT/ITL separately. Optimization for throughput can degrade TTFT under high concurrency. Record prompt/output length distributions, request admission, batch occupancy and cache preemption.

## 10. Parallelism and topology

Tensor parallelism partitions supported operations across GPU ranks and requires communication. Pipeline parallelism divides layers/stages, potentially increasing bubbles and inter-stage transfers. Multi-node setups add network, NCCL, rank health, topology and fault domains.

Before choosing TP/PP:
- verify model and release support;
- inventory NVLink/PCIe and network topology;
- measure per-rank VRAM and communication cost;
- test rank failure and recovery;
- account for GPUs consumed per serving replica.

## 11. TensorRT-LLM and Triton integration

A supported Triton TensorRT-LLM backend typically needs a model repository/configuration consistent with the selected runtime path. The exact repository layout, preprocessing/postprocessing components, executor settings, decoupled responses and engine locations are release-specific.

Do not assume standard Triton tensor inference is equivalent to an OpenAI-compatible chat API. Validate gateway translation, tokenization, streaming completion, cancellation and error mapping.

## 12. Reproducible build pipeline

```text
Model approval
   → license/security/provenance
   → pin source/tokenizer/runtime/GPU
   → conversion/checkpoint validation
   → build or runtime preparation
   → artifact checksum and metadata
   → staging load and golden fixtures
   → TTFT/TPOT/E2E, VRAM, quality
   → security/observability checks
   → controlled canary
   → promotion or immutable rollback
```

Keep build artifacts in an access-controlled registry. Record build logs without secrets or proprietary tensors. Avoid untrusted custom kernels or remote model code.

## 13. Safe environment discovery

**[SAFE-READ]** approved GPU host:

```bash
date -u
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,compute_cap,memory.total,memory.used --format=csv
```

If the `compute_cap` query field is unsupported by the installed driver/tool, use `nvidia-smi -q` and vendor documentation; do not infer GPU capability.

For approved Kubernetes cluster:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

On an authorized pinned build image, inspect installed CLI help and package versions before generating any build command:

```bash
python -c "import importlib.metadata as m; print(m.version('tensorrt_llm'))"
```

Package naming and CLI availability vary; record an unsupported lookup as a compatibility finding rather than guessing.

## 14. Build acceptance checklist

- source model and tokenizer digests verified;
- license, security and custom-code policy approved;
- target GPU/compute capability and software matrix recorded;
- supported precision and build configuration selected;
- conversion/build logs archived and artifact checksummed;
- runtime loads and model/version readiness verified;
- golden prompt fixtures and stop/template behavior validated;
- short/long context, concurrency and GPU VRAM benchmarked;
- distributed rank/NCCL checks if applicable;
- streaming completion and cancellation tested;
- SLO, privacy, on-call, rollback and cost ceiling approved.

## 15. Performance and FinOps

Measure build GPU-hours separately from serving GPU-hours. Include build cache/storage, engine size, GPU occupancy, TTFT/TPOT, complete valid outputs, energy/power if available and failed/retried requests. A faster engine can still be economically worse if it requires more expensive GPUs or reduces useful capacity.

## 16. Incident decision tree

```text
TensorRT-LLM build or serve fails?
 ├─ model unsupported → support matrix/ops
 ├─ conversion mismatch → weights/tokenizer/config
 ├─ build OOM → build workspace/host+GPU memory
 ├─ engine load fails → target GPU/software mismatch
 ├─ incorrect output → tokenizer/precision/quantization
 ├─ startup CUDA OOM → weights/workspace/KV/headroom
 ├─ decode OOM → KV/context/concurrency
 ├─ TP/PP hangs → rank/NCCL/topology
 ├─ TTFT high → prefill/queue/admission
 ├─ TPOT high → decode/bandwidth/communication
 ├─ streaming partial → backend/gateway completion
 ├─ metrics missing → exporter/schema/version
 └─ cost high → GPU count, build/rebuild, throughput
```

## 17. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | unsupported model architecture | support matrix |
| F02 | source weights checksum mismatch | provenance |
| F03 | tokenizer revision drift | fixture |
| F04 | unsupported precision | GPU/runtime matrix |
| F05 | conversion tensor mapping wrong | conversion log |
| F06 | missing plugin/kernel | build log |
| F07 | build host RAM exhaustion | build telemetry |
| F08 | build GPU OOM | GPU memory |
| F09 | engine incompatible with target GPU | load log |
| F10 | driver/CUDA/TensorRT mismatch | versions |
| F11 | artifact corrupted in registry | digest |
| F12 | model loads with wrong outputs | golden fixtures |
| F13 | EOS/stop semantics regress | stream validator |
| F14 | context admission exceeds budget | token limits |
| F15 | startup CUDA OOM | per-rank VRAM |
| F16 | decode KV exhaustion | KV metrics |
| F17 | TP rank mismatch | rank inventory |
| F18 | NCCL timeout | rank/network |
| F19 | TTFT regression | prefill/queue |
| F20 | TPOT regression | decode/comm |
| F21 | incomplete stream | terminal validator |
| F22 | missing telemetry | scrape freshness |
| F23 | rollback engine not portable | compatibility |
| F24 | build/serving GPU cost exceeds ceiling | FinOps |

## 18. Sixteen operational runbooks

1. Verify model architecture support.
2. Verify source weights and tokenizer provenance.
3. Validate GPU/runtime compatibility matrix.
4. Diagnose precision and quantization support.
5. Diagnose checkpoint conversion.
6. Diagnose missing plugins and kernels.
7. Recover build host/GPU OOM.
8. Validate engine artifact digest and portability.
9. Diagnose engine/runtime initialization.
10. Validate golden prompt/output fixtures.
11. Diagnose context/KV budget.
12. Recover startup/decode CUDA OOM.
13. Recover TP/PP rank and NCCL failures.
14. Diagnose TTFT/TPOT regressions.
15. Restore streaming, metrics and GPU FinOps.
16. Execute immutable rollback and RCA.

## 19. Eight tabletop game days

1. Model conversion produces a valid but incorrect tensor mapping.
2. Engine built on one GPU fails on target GPU.
3. Quantization improves throughput but breaks golden fixtures.
4. Build GPU OOM occurs near artifact publication.
5. KV cache exhausts during long-context decode.
6. One tensor-parallel rank fails during serving.
7. Gateway returns HTTP 200 but incomplete output.
8. Rollback artifact cannot load after driver upgrade.

Default tabletop. Live fault injection requires separately approved isolated staging.

## 20. Twenty acceptance gates

G01 approved model/license; G02 weights digest; G03 tokenizer/template; G04 pinned runtime; G05 GPU compatibility; G06 precision support; G07 conversion validation; G08 build reproducibility; G09 artifact digest; G10 runtime readiness; G11 golden fixtures; G12 context/KV budget; G13 GPU memory; G14 TP/PP health; G15 TTFT/TPOT; G16 complete streaming; G17 security/privacy; G18 telemetry/SLO; G19 rollback/FinOps; G20 independent review.

See [companion lab](../labs/031-tensorrt-llm-architecture-compilation-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 032 — TensorRT-LLM Quantization and Engine Compatibility.**
