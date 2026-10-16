# Chapter 032 — TensorRT-LLM Quantization and Engine Compatibility

**Part 05 — NVIDIA Triton and TensorRT-LLM | Chapter 032 of 100**

> **Status:** Written tutorial published; technical/security review, live lab execution and operational acceptance pending. Precision formats, quantization recipes, build flags and engine portability are release-, model- and GPU-specific. Verify the pinned TensorRT-LLM documentation and hardware support matrix before use.

## Objectives

Understand TensorRT-LLM precision and quantization tradeoffs, weight-only versus activation-aware approaches, calibration and quality gates, model/engine/runtime compatibility, GPU compute capability, deterministic release manifests, memory and performance benchmarking, fallback and safe rollback.

## 1. Quantization in production inference

Quantization represents weights, activations or caches at reduced precision to improve memory footprint, bandwidth or throughput under supported kernels. It may reduce output quality, increase calibration complexity or require architecture-specific execution paths.

```text
Approved source weights + tokenizer
                │
      precision/quantization policy
       ├─ baseline BF16/FP16
       ├─ supported FP8
       ├─ supported INT8
       └─ supported INT4/weight-only
                │
       calibration (if required)
                │
      conversion/build/runtime prep
                │
        compatibility validation
                │
       golden quality + safety
                │
       TTFT/TPOT/VRAM/FinOps
                │
         approved promotion
```

**Never assume lower bit width always means faster or better.** Kernel support, dequantization overhead, batch shape, context length and GPU generation determine results.

## 2. Precision and quantization comparison

| Method | Typical benefit | Key limitation |
|---|---|---|
| FP32 | numeric baseline/reference | high memory/compute cost |
| FP16 | common accelerated inference | numerical overflow/underflow risk |
| BF16 | broad dynamic range | hardware/backend support |
| FP8 | potential speed/memory improvements | GPU/kernel/scaling support |
| INT8 | smaller weights/activations | calibration/accuracy tradeoffs |
| INT4 weight-only | reduced weight memory | kernel/model support, quality risk |
| mixed precision | preserve sensitive layers | more complex validation |

These are categories, not guarantees that any particular TensorRT-LLM release supports a method for a given model.

## 3. Quantization taxonomy

- **Weight-only:** weights are compressed; compute may use mixed or higher-precision activation paths.
- **Weight-and-activation:** both representations may be quantized; often needs additional scaling or calibration.
- **Post-training quantization:** uses a trained model and may require representative calibration.
- **Quantization-aware training:** accounts for quantization effects during training; not automatically available in serving workflows.
- **KV-cache precision:** can be tuned separately where supported and affects memory, numerical behavior and context capacity.

Record exactly which tensors and operations are quantized; a marketing label such as “INT4 model” is insufficient.

## 4. Quantization artifacts and provenance

A release is more than a weights file:

```text
source model revision
source weights digest
tokenizer and chat-template digest
quantization algorithm/version
calibration dataset identity and policy
calibration statistics/scales
runtime and backend versions
target GPU architecture
build flags/config
engine or runtime artifact digest
golden quality/performance evidence
rollback artifact
```

Do not place sensitive calibration examples in logs, Git history or public registries.

## 5. Calibration quality

If the chosen method requires calibration, the dataset must represent expected prompt lengths, languages, formats, tool-use patterns and safety-sensitive categories without leaking private data. Separate calibration from holdout evaluation to prevent optimistic quality estimates.

Suggested test cohorts:
1. short factual prompts;
2. long-context retrieval;
3. structured JSON/tool outputs;
4. multilingual inputs;
5. numerical reasoning;
6. stop/EOS/stream completion;
7. refusal and safety behavior;
8. concurrency under memory pressure.

All fixtures should use approved synthetic or appropriately governed data.

## 6. Quantization quality gates

Use model-appropriate evaluation rather than a single aggregate accuracy number. Record exact-match, format validity, semantic consistency, task metrics, regression count, latency and incomplete-response rate.

Example **policy template** (thresholds must be approved per use case):

```yaml
quantization_acceptance:
  golden_fixture_pass_rate: APPROVED_THRESHOLD
  structured_output_validity: APPROVED_THRESHOLD
  safety_regression_tolerance: APPROVED_THRESHOLD
  ttft_p95_ms: APPROVED_SLO
  tpot_p95_ms: APPROVED_SLO
  gpu_memory_headroom: APPROVED_MINIMUM
  cost_per_valid_completion: APPROVED_CEILING
```

Do not invent universal quality thresholds.

## 7. Engine compatibility matrix

| Dimension | Required check | Failure symptom |
|---|---|---|
| GPU architecture/SM | target support | engine fails to deserialize/load |
| driver/CUDA | supported combination | initialization error |
| TensorRT version | build/runtime compatibility | deserialization/plugin error |
| TensorRT-LLM version | executor/backend compatibility | missing field/kernel |
| model architecture | supported ops | conversion/build failure |
| precision/quant method | kernel/GPU support | build/load failure |
| tensor/pipeline parallelism | rank topology | startup/NCCL failure |
| max context/batch/tokens | supported profiles | shape/admission error |
| tokenizer/template | same semantics | wrong output |
| Triton backend | matching integration | model load or streaming failure |

Engine portability can be constrained by target GPU and software versions. Even where compatibility features exist, validate their documented limits. Never assume cross-GPU or cross-release portability by default.

## 8. Build versus runtime compatibility

A successful engine build does not imply it will run in production. Validate the **exact serving image and GPU type** before promotion. Treat model weights, quantization artifacts, engine, runtime plugins and tokenizer as one immutable release unit.

```text
Build GPU/software matrix
           │
           ▼
    artifact compatibility
           │
           ▼
Target serving GPU/software matrix
           │
           ▼
   load + readiness + inference
           │
           ▼
 quality + SLO + cost acceptance
```

## 9. GPU memory budget

Approximate model weight storage before overhead:

```text
weight bytes ≈ parameter_count × bits_per_weight / 8
```

For a synthetic 7B-parameter model:
- BF16: approximately 14 GB decimal weight bytes.
- INT8: approximately 7 GB decimal weight bytes.
- INT4: approximately 3.5 GB decimal weight bytes.

These are **idealized weight-only figures**, not total serving VRAM. Add quantization scales/metadata, unquantized layers, activations, workspace, CUDA graphs, KV cache, fragmentation, communication buffers and headroom. Report GiB versus GB correctly.

## 10. KV cache precision and context

For conventional attention:

```text
KV bytes/token ≈ 2 × layers × KV_heads × head_dim × bytes/element
```

Example: 32 layers × 8 KV heads × 128 head dimension × 2 bytes (BF16) × 2 = 131,072 bytes/token, or 128 KiB/token. At 32,768 resident tokens, logical KV is about 4 GiB.

KV-cache quantization may reduce that portion where supported, but can change output quality and require specific scaling/attention kernels. Do not extrapolate total GPU memory savings from weight precision alone.

## 11. Performance experiments

| Cohort | Measure | Why |
|---|---|---|
| short prompt / short output | TTFT, TPOT | interactive latency |
| long prompt / short output | prefill, VRAM | context stress |
| short prompt / long output | decode, KV growth | generation stress |
| long prompt / long output | tail latency, OOM | worst case |
| mixed concurrency | throughput, fairness | production mix |
| structured/safety fixtures | valid output/quality | regression |
| target GPU matrix | load/readiness | compatibility |

Compare against the same unquantized baseline, model/tokenizer, GPU, runtime, concurrency and test duration. Report **cost per valid completion**, not cost per attempted request.

## 12. Quantization and distributed inference

Tensor parallelism may shard weights, activations or quantization metadata. Verify rank-consistent formats, scale handling, NCCL topology and per-rank memory. Pipeline parallelism may require compatible engine partitioning. A single-rank pass is not proof that multi-rank serving works.

## 13. Safe-read environment inspection

```bash
date -u
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used --format=csv
```

For authorized Kubernetes:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

Record pinned build/runtime image digests, TensorRT-LLM/TensorRT/CUDA/driver versions and actual target GPU. Do not run an arbitrary engine build from an unverified repository.

## 14. Release and rollback

```text
Pin source/tokenizer
 → approve quant method + calibration
 → generate artifact + checksum
 → verify target GPU/software matrix
 → load in isolated staging
 → golden quality + safety fixtures
 → TTFT/TPOT/VRAM/throughput
 → canary and error budget
 → promote or rollback complete release
```

Rollback must restore the matching engine, quantization scales, runtime/backend, tokenizer and model config. Reverting only the engine file may leave an incompatible stack.

## 15. Security and governance

- Approve model license, origin and artifact hashes.
- Review custom kernels, plugins and remote model code.
- Keep calibration/holdout datasets governed and private.
- Audit build and artifact registry permissions.
- Do not log prompts, tokens, outputs or calibration samples.
- Restrict model-control and deployment actions.
- Track evaluation bias, quality regressions and safety behavior.
- Preserve a tested known-good baseline.

## 16. Incident decision tree

```text
Quantized engine degraded?
 ├─ build fails → unsupported method/model/kernel
 ├─ load fails → GPU/software/engine mismatch
 ├─ output wrong → scales/calibration/tokenizer
 ├─ JSON invalid → quantization quality regression
 ├─ TTFT high → prefill/queue/kernel path
 ├─ TPOT high → decode/dequant overhead
 ├─ VRAM higher than expected → metadata/KV/workspace
 ├─ GPU OOM → batch/context/concurrency
 ├─ TP rank fails → quant artifact/rank mismatch
 ├─ streaming incomplete → backend/gateway
 ├─ telemetry missing → metric schema/exporter
 └─ cost rises → valid throughput/GPU-hours
```

## 17. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | quant method unsupported | support matrix |
| F02 | GPU lacks precision kernel | GPU architecture |
| F03 | conversion version mismatch | tool versions |
| F04 | missing quant scales | artifact manifest |
| F05 | calibration data unrepresentative | cohort analysis |
| F06 | calibration data leakage | privacy audit |
| F07 | tokenizer drift | golden fixture |
| F08 | structured output regression | schema test |
| F09 | safety/refusal regression | safety fixtures |
| F10 | long-context quality regression | long cohort |
| F11 | build GPU OOM | build telemetry |
| F12 | engine target GPU mismatch | load log |
| F13 | TensorRT plugin mismatch | runtime log |
| F14 | CUDA/driver incompatibility | versions |
| F15 | batch profile unsupported | admission |
| F16 | quantized KV numerical regression | fixture |
| F17 | VRAM savings lower than forecast | per-GPU metrics |
| F18 | TP rank quant artifact mismatch | rank log |
| F19 | TTFT worsens | benchmark |
| F20 | TPOT worsens | benchmark |
| F21 | incomplete streaming | terminal validator |
| F22 | metric/trace gap | scrape |
| F23 | rollback artifact incompatible | rollback drill |
| F24 | cost per valid completion rises | FinOps |

## 18. Sixteen operational runbooks

1. Verify quantization support matrix.
2. Verify source/tokenizer and calibration provenance.
3. Validate precision/kernel compatibility.
4. Diagnose missing scales and conversion errors.
5. Investigate calibration distribution drift.
6. Investigate structured-output/quality regression.
7. Investigate safety/refusal regression.
8. Recover build OOM and artifact corruption.
9. Diagnose target GPU engine incompatibility.
10. Diagnose TensorRT/CUDA/plugin mismatch.
11. Diagnose context/batch profile rejection.
12. Investigate quantized KV and GPU memory.
13. Recover TP/PP rank mismatch.
14. Investigate TTFT/TPOT regressions.
15. Restore streaming/telemetry and cost visibility.
16. Execute complete-release rollback and RCA.

## 19. Eight tabletop game days

1. INT4 method unsupported on target GPU.
2. Calibration passes short prompts but fails long context.
3. Quantized model breaks JSON schema responses.
4. Quantization scales missing from artifact registry.
5. Engine built on one GPU fails on production GPU.
6. TP rank uses inconsistent quantization metadata.
7. TTFT improves but TPOT and cost worsen.
8. Rollback engine cannot load after runtime upgrade.

Default tabletop; live fault injection requires separately approved isolated staging.

## 20. Twenty acceptance gates

G01 approved model/license; G02 source/tokenizer digests; G03 quant method support; G04 calibration provenance; G05 holdout quality; G06 structured output; G07 safety behavior; G08 GPU precision support; G09 runtime matrix; G10 engine artifact digest; G11 target load/readiness; G12 context/batch profile; G13 KV memory; G14 per-GPU VRAM; G15 TP/PP health; G16 TTFT/TPOT; G17 streaming completion; G18 privacy/telemetry; G19 rollback/FinOps; G20 independent review.

See [companion lab](../labs/032-tensorrt-llm-quantization-engine-compatibility-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 033 — TensorRT-LLM Throughput, Latency and KV Cache.**
