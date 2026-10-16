# Chapter 017 — vLLM Model Loading and Runtime Configuration

**Part 04 — vLLM Administration and Operations | Chapter 017 of 100**

> **Publication status:** Written tutorial; live labs, technical/security review and acceptance pending. vLLM options, endpoint behavior and engine internals change across releases. Validate every flag against the exact pinned version before use.

## Learning objectives

Trace model loading from artifact authorization to healthy serving; configure tokenizer/chat template, dtype, quantization, context, GPU memory and parallelism; distinguish loading errors from readiness and admission failures; safely investigate cold starts, OOM, artifact drift and quality regressions; create a reproducible release manifest and controlled rollback.

## 1. Model loading lifecycle

```text
Release approval → model registry / revision / checksum / license
   → workload identity + artifact fetch / local cache
   → config + tokenizer + chat template resolution
   → GPU/driver/CUDA/framework compatibility checks
   → weights deserialization / dtype / quantization
   → tensor/pipeline parallel worker initialization
   → GPU allocation / KV-cache and workspace planning
   → optional kernel compilation / CUDA graph capture / warmup
   → health / readiness + synthetic end-to-end validation
   → bounded admission and production routing
```

A running container or responsive health endpoint is not proof that the intended model, tokenizer, tool-call format and full SSE response are correct.

## 2. Release contract and provenance

Pin the **container digest**, vLLM release, CUDA/driver stack, GPU SKU, model and tokenizer revision, chat template, quantization artifact, source checksum, serving alias and inference settings. Do not rely on mutable `latest` tags or unpinned remote model revisions.

```yaml
release_id: "STAGING_CHANGE_ID"
image_digest: "sha256:APPROVED_DIGEST"
vllm_version: "PINNED_RELEASE"
model_id: "APPROVED_MODEL"
model_revision: "IMMUTABLE_REVISION"
tokenizer_revision: "IMMUTABLE_REVISION"
chat_template_revision: "APPROVED_TEMPLATE"
dtype: "APPROVED_PRECISION"
quantization: "NONE_OR_APPROVED_FORMAT"
max_model_len: "REVIEWED_CONTEXT_LIMIT"
gpu_memory_utilization: "REVIEWED_FRACTION"
tensor_parallel_size: "REVIEWED_GPU_COUNT"
artifact_provenance: "CHECKSUM_AND_LICENSE_RECORD"
rollback_release: "PREVIOUS_KNOWN_GOOD"
```

This is a **documentation worksheet**, not a vLLM configuration file. Exact CLI arguments and accepted values are release-specific.

## 3. Model artifact security

Model loading is a supply-chain boundary. Require license approval, artifact provenance, checksum/digest validation, restricted registry access, egress controls, vulnerability scanning, signed image policy and redacted logs. Some model formats or model families require custom code. **Do not enable remote custom model code execution merely to fix a loader error**; require security review, provenance and sandboxing. Protect model weights, prompts and tokenizer artifacts according to organizational policy.

Prefer immutable, approved local model artifacts in staging when possible. Define cache location, disk capacity, eviction and restart behavior; a cache miss can extend cold start or fail under egress restrictions.

## 4. Model and tokenizer compatibility

A model's weights, config, tokenizer, special tokens, chat template and generation defaults must be evaluated together. Symptoms of mismatch include invalid role formatting, early EOS, unexpected token counts, malformed JSON/tool calls, silent truncation and hallucinated special tokens.

Test with synthetic system/user/assistant turns, structured-output fixtures, stop conditions and expected finish behavior. Keep tokenizer versions consistent between admission token counting and serving.

## 5. Runtime configuration decision matrix

| Config family | Operational impact | Validation |
|---|---|---|
| Model/revision | correctness and provenance | immutable artifact |
| Tokenizer/template | role handling, token count | synthetic fixtures |
| dtype | VRAM, numerical behavior | supported GPU kernels |
| Quantization | memory and quality | eval + benchmark |
| Max context | KV allocation and admission | token arithmetic |
| GPU memory target | KV reserve vs OOM | actual per-GPU memory |
| Batched token budget | prefill/decode latency | TTFT/TPOT cohorts |
| Concurrent sequences | throughput, queue, cache | load/admission test |
| Tensor/pipeline parallelism | model fit, NCCL | rank and topology |
| Prefix caching | latency, memory and tenant isolation | security + correctness |
| Model alias | API routing | request/model match |
| Compilation/graph capture | warmup and memory peaks | startup and steady state |

A memory utilization target is not a hard guarantee against OOM. Context length is not equivalent to concurrent resident-token capacity. A model that loads does not necessarily meet SLOs.

## 6. Version-aware launch pattern

Use the pinned vLLM release's CLI reference. A **minimal conceptual example** for an approved, isolated staging environment is:

```bash
vllm serve "APPROVED_MODEL_ID_OR_LOCAL_PATH" \
  --served-model-name "APPROVED_MODEL_ALIAS" \
  --dtype auto
```

Confirm `vllm serve` and option support for the installed version before execution. **Do not execute this command on production or an unapproved GPU host.** Model identity, image, GPU allocation, cache, authentication, ingress, context and memory policy must be resolved separately. Avoid enabling `--trust-remote-code` without security approval.

To inspect supported options for a pinned installed release in a sandbox:

```bash
vllm --help
vllm serve --help
```

Some releases have different command entry points. The official CLI help for the pinned image is authoritative.

## 7. Kubernetes deployment considerations

A production design must account for GPU scheduling, node affinity, device plugin, CUDA compatibility, immutable image digest, approved model cache, identity, restricted egress, startup probes and readiness semantics, grace periods, PodDisruptionBudget, rollout surge capacity and GitOps ownership.

Do not assume readiness should become green merely because the API server bound its socket. Define a version-specific readiness check and an independent synthetic model completion probe. Use a startup probe budget that accommodates expected artifact download, deserialization, compilation and warmup without hiding a permanent failure.

If a deployment needs multiple GPUs for tensor parallelism, ensure its requested GPU count, node topology and runtime parallelism agree. Avoid rolling an old replica out before new capacity is genuinely ready unless downtime is explicitly approved.

## 8. Read-only diagnostics

**[SAFE-READ]** in an approved Kubernetes context:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

For a verified namespace/deployment (replace placeholders):

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=20m --tail=150
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,driver_version,memory.total,memory.used --format=csv
```

Never export secrets, authorization headers, customer prompts or model artifacts into evidence.

## 9. Synthetic inference acceptance

On an approved staging endpoint only, using a short non-sensitive prompt:

```bash
curl --fail-with-body --no-buffer --max-time 120 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $STAGING_API_TOKEN" \
  -d '{"model":"APPROVED_MODEL_ALIAS","messages":[{"role":"user","content":"Return exactly: READY"}],"max_tokens":32,"stream":true}' \
  "$STAGING_BASE_URL/v1/chat/completions"
```

Verify HTTP status, correct model alias, SSE event framing, nonempty content, finish reason, terminal event and clean connection close. An HTTP 200 with truncated SSE is a failure. A model may produce a semantically different answer despite technically correct streaming; evaluate correctness separately.

## 10. Cold-start and readiness SLOs

Break down startup into artifact access/download, cache hit/miss, model deserialization, GPU allocation, rank initialization, kernel compilation/graph capture and warmup. Capture p50/p95 load time across controlled restarts, cold versus warm cache and per-rank failure evidence. Do not extrapolate from one successful restart.

Define deployment readiness budgets and capacity buffers separately from inference TTFT/TPOT. Cold starts can exhaust available GPU capacity during rollouts; document cost of temporarily running old and new replicas.

## 11. Failure decision tree

```text
Model unavailable or incorrect?
 ├─ image pull fails → registry identity/digest
 ├─ artifact download fails → permissions, egress, cache, disk
 ├─ config/tokenizer load fails → revision, files, template
 ├─ CUDA init fails → host driver, runtime, toolkit
 ├─ model load OOM → weights, dtype, workspace, TP
 ├─ post-load OOM → KV/context/concurrency
 ├─ TP rank fails → mapping, NCCL, GPU topology
 ├─ health green, synthetic fails → readiness contract
 ├─ invalid chat/tool output → template/quantization
 ├─ long cold start → cache, network, compile, warmup
 ├─ wrong model response → alias/revision/routing
 ├─ TTFT high → queue, prefill, startup spillover
 ├─ 200 partial SSE → proxy, timeout, cancellation
 └─ telemetry absent → metric schema/freshness
```

## 12. Rollout and rollback runbook

1. Freeze approved model, tokenizer/template, image digest, GPU configuration and artifact checksums.
2. Obtain deployment authorization and record owner, SLO guardrails, GPU-hour ceiling and spare capacity.
3. Validate compatibility, license, registry identity and model cache in isolated staging.
4. Start a bounded staging instance; collect cold/warm load evidence and VRAM.
5. Validate health and full-stream synthetic correctness.
6. Benchmark TTFT/TPOT, long-context handling, JSON/tool calling, GPU cost and 429/5xx behavior.
7. Verify model alias routing, log redaction, quotas and cancellation.
8. Canary only through approved routing; define abort thresholds in advance.
9. Revert model/image/config using the authoritative GitOps controller on any gate failure.
10. Verify complete-stream success, model revision and SLO recovery after rollback.

## 13. Twenty-four production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | model artifact 403 | registry identity |
| F02 | model artifact 404 | revision/path |
| F03 | corrupted/incomplete weight file | checksum |
| F04 | insufficient model-cache disk | filesystem quota |
| F05 | mutable model revision drift | artifact manifest |
| F06 | unapproved remote code dependency | security review |
| F07 | tokenizer mismatch | synthetic tokenization |
| F08 | wrong chat template | chat/tool fixture |
| F09 | invalid model alias | API routing |
| F10 | unsupported dtype | GPU/runtime matrix |
| F11 | unsupported quantization kernel | CUDA/framework |
| F12 | weight load CUDA OOM | per-GPU VRAM |
| F13 | workspace/graph capture OOM | startup allocation |
| F14 | max context too large | KV budget |
| F15 | excessive concurrency | cache/preemption |
| F16 | wrong TP rank count | GPUs vs config |
| F17 | NCCL init timeout | rank logs |
| F18 | model startup probe too short | startup timings |
| F19 | readiness green before usable model | synthetic probe |
| F20 | warm cache masks cold-start regression | cache cohorts |
| F21 | model output quality regression | evaluation baseline |
| F22 | tool-call/JSON regression | structured fixtures |
| F23 | HTTP 200 incomplete SSE | terminal validator |
| F24 | rollout cost/availability regression | GPU-hours and SLO |

## 14. Sixteen operational runbooks

1. Validate release provenance and context.
2. Triage artifact authorization and availability.
3. Investigate cache capacity and cold downloads.
4. Validate checksum and immutable revision.
5. Diagnose tokenizer/chat-template mismatch.
6. Triage unsupported dtype/quantization.
7. Investigate model load OOM.
8. Investigate workspace/CUDA graph OOM.
9. Investigate context/KV pressure.
10. Diagnose TP rank and NCCL initialization.
11. Review startup/readiness and synthetic checks.
12. Diagnose wrong model alias or routing.
13. Investigate quality/JSON/tool regressions.
14. Compare cold/warm startup and GPU cost.
15. Execute approved rollback.
16. Verify sustained recovery and RCA/COA.

## 15. Eight game days

1. Model registry returns 403 during cold start.
2. Cached model is missing after node replacement.
3. New tokenizer changes chat-template behavior.
4. BF16 weights exceed available VRAM.
5. Quantized kernel is unsupported on one GPU SKU.
6. Readiness passes while model still fails synthetic requests.
7. Model alias routes to the wrong revision.
8. Rollout increases GPU-hours and interrupts active streams.

Default **tabletop**; fault injection requires separately approved staging.

## 16. Twenty acceptance gates

G01 release provenance; G02 license/security; G03 registry identity; G04 checksum/cache; G05 tokenizer/template; G06 dtype support; G07 quantization evaluation; G08 GPU/VRAM budget; G09 context/admission; G10 TP/NCCL; G11 startup timing; G12 readiness correctness; G13 synthetic full SSE; G14 structured output quality; G15 TTFT/TPOT; G16 privacy and quotas; G17 observability; G18 FinOps; G19 rollback; G20 reviewer signoff.

See [companion lab](../labs/017-vllm-model-loading-runtime-configuration-check.md). All gates remain pending until evidenced and reviewed.

**Next: Chapter 018 — Continuous Batching, PagedAttention and KV Cache.**
