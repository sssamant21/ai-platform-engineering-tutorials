# Chapter 070 — Model Precision, Quantization and Quality Gates

**Part 09 — Performance Engineering and Capacity · Chapter 070 of 100**

**Status:** Published documentation and companion lab; live execution and technical/security reviews pending.

## Learning outcomes
Explain FP32, TF32, FP16, BF16, FP8, INT8 and INT4 tradeoffs; distinguish weight-only, activation and KV-cache quantization; measure quality and performance under matched workloads; enforce rollback-safe release gates.

## Architecture and theory
```text
approved model digest -> calibration/evaluation corpus
          |                       |
     quantization profile     quality baseline
          |                       |
     serving runtime -----> benchmark cohorts
          |                       |
     TTFT/TPOT/goodput + accuracy/safety + cost
                       |
               gated deployment
```
Precision affects memory, bandwidth, numerical stability, supported kernels and sometimes throughput. FP16/BF16 generally reduce storage relative to FP32; BF16 offers a wider exponent range than FP16. FP8/INT8/INT4 performance depends on architecture, quantization scheme, group size, scaling, activation handling and runtime support. **Lower bit width does not guarantee lower end-to-end latency or equivalent quality.** Weight-only INT4 does not imply 4-bit KV cache.

### Quality dimensions
- Task accuracy: exact match, domain F1, tool-call correctness, structured JSON validity.
- Generation fidelity: groundedness, factual consistency, instruction following, long-context retrieval.
- Safety: refusal behavior, PII handling, jailbreak resistance, content-policy regressions.
- Serving: TTFT, TPOT, E2E p95, output tokens/sec, goodput, memory, OOM, cost/good response.
- Stability: deterministic seed where supported, repeated evaluations, subgroup/cohort breakdowns.

## Controlled benchmark procedure
1. Pin base model, tokenizer, quantizer, runtime, kernel, hardware, context and sampling configuration.
2. Select approved de-identified evaluation data; separate calibration from held-out evaluation.
3. Define quality thresholds **before** evaluating candidates. Example: task score drop <= 1 percentage point, no critical safety regression, JSON validity >= 99.5%; these are illustrative only.
4. Run FP16/BF16 baseline and each quantized candidate on identical prompt/output-length and concurrency cohorts.
5. Report paired deltas, confidence intervals or bootstrap uncertainty, and per-cohort failures; never hide long-context regressions in aggregates.
6. Validate peak memory, OOMs, latency, goodput and cost at matched SLOs.
7. Use canary rollout, observability and rollback when approved; fail closed on missing quality evidence.

## Common incidents and troubleshooting
| Symptom | Investigate |
|---|---|
| Nonsensical or repetitive output | calibration drift, quantization scale, tokenizer mismatch |
| Tool-call JSON failures | logits/precision changes, schema-constrained decoding |
| Slower INT4 serving | dequantization overhead, unsupported optimized kernel |
| KV-cache OOM | KV dtype and resident tokens independent of weight size |
| Quality loss on rare languages | stratified evaluation coverage |
| Safety regression | targeted red-team suite, policy gate |
| Unexpected cost increase | low goodput, retries, memory fragmentation |

## Production runbooks
**RB-070-A Quality regression:** freeze rollout, compare paired evals and model digest, inspect tokenizer/calibration, roll back via approved deployment control and verify affected cohorts.

**RB-070-B Numerical instability:** check NaNs, overflow, precision-dependent kernels and output anomalies; isolate candidate and revert if thresholds fail.

**RB-070-C Memory or performance regression:** inspect weight footprint versus KV, active sequences, TPOT, throughput and kernel support; compare identical traffic mix.

**RB-070-D Release acceptance:** record evaluation version, model lineage, reproducibility, subgroup results, performance evidence, reviewer sign-off and rollback readiness.

## Security and acceptance
No customer prompts or completions in metrics; secure evaluation datasets and model artifacts. Reject candidates with critical safety failures even if cost improves. Tests in the companion lab are **synthetic contract tests**, not model-quality certification.

- [ ] Model/tokenizer/runtime versions pinned.
- [ ] Calibration and evaluation sets separated.
- [ ] Quality and safety gates reviewed.
- [ ] Performance compared at matched SLO.
- [ ] Rollback and canary plan approved.
- [ ] Live tests and technical/security review recorded.

## References
- [PyTorch numerical accuracy](https://docs.pytorch.org/docs/stable/notes/numerical_accuracy.html)
- [NVIDIA TensorRT-LLM](https://nvidia.github.io/TensorRT-LLM/)
- [Hugging Face quantization](https://huggingface.co/docs/transformers/main/quantization/overview)
## Hands-on lab
[Executable Chapter 070 lab](../labs/070-model-precision-quantization-quality-gates-check.md). Local synthetic checks only; no production endpoint or real model required.

**Next:** Chapter 71 — GPU Throughput and Cost per Successful Token.
