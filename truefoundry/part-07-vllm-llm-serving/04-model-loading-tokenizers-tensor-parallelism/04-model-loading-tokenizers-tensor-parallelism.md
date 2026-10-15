# 7.4 — Model Loading, Tokenizers and Tensor Parallelism

**Part:** 07 — vLLM & LLM Serving · **Chapter:** 4 of 7  
**Audience:** AI platform engineers, TrueFoundry administrators, GPU SREs and model-serving engineers  
**Safety:** **[SAFE-READ]** observation, **[LAB-MUTATE]** approved isolated nonproduction, **[CHANGE-CONTROL]** production modification.

## 1. Learning outcomes

After this chapter, you can inventory immutable model and tokenizer artifacts; explain the complete vLLM load path; distinguish model, tokenizer and chat-template failures; size GPU memory and distributed parallelism; diagnose NCCL and multi-GPU initialization; and execute a controlled recovery using the authoritative TrueFoundry deployment definition.

**Prerequisites:** Chapters 7.1–7.3; read access to the approved workspace, Kubernetes cluster and model registry; GPU capacity where distributed experiments are planned. No production fault injection, untrusted model code or sensitive inference payloads.

## 2. Where model loading fits

```text
Git / CI release record
   ├── immutable vLLM image digest + runtime versions
   ├── model ID and pinned revision
   ├── tokenizer/config/chat template revision
   └── artifact checksum and license approval
          │
          ▼
TrueFoundry authoritative deployment
          │
          ▼
Kubernetes Pod scheduled → GPU allocated
          │
          ▼
Workload identity → artifact storage / cache
          │
          ▼
config/tokenizer/weight shard retrieval and validation
          │
          ▼
vLLM architecture and precision validation
          │
          ▼
CUDA/PyTorch init → tensor-parallel workers/NCCL (if enabled)
          │
          ▼
weights allocated → KV cache profiled/allocated → warmup
          │
          ▼
model-ready endpoint → authorized golden inference
```

A Running Pod does not establish model readiness. A successful GPU check does not establish tokenizer, weight, architecture or tensor-parallel compatibility.

## 3. Model artifact anatomy

Typical Hugging Face-style model repositories may contain:

| Artifact | Purpose | Failure mode |
|---|---|---|
| `config.json` | Model architecture, hidden dimensions, layer/head counts | Unsupported architecture or incorrect configuration |
| `model.safetensors` / sharded weights | Parameters, often indexed by `model.safetensors.index.json` | Missing shard, checksum mismatch, insufficient disk |
| `tokenizer.json`, tokenizer config | Tokenization rules, vocabulary and special tokens | Token IDs differ from model expectations |
| `special_tokens_map.json` | Special token definitions | Missing or inconsistent BOS/EOS/PAD behavior |
| `generation_config.json` | Optional default generation parameters | Unexpected stopping or generation defaults |
| Chat template | Converts structured messages into model-specific prompt format | Chat API errors or malformed model prompts |
| Custom code (if present) | Model-specific implementation | Supply-chain/security and runtime compatibility risk |

Not every model uses every file. Treat the model and tokenizer as a **versioned, compatible artifact set**, not independent mutable downloads. Prefer signed/verified sources, safetensors where supported, and a pinned commit/revision; avoid `trust_remote_code` unless an explicitly approved security review authorizes it.

## 4. Immutable manifest and provenance

**Evidence template only — not a TrueFoundry schema:**

```yaml
model_release:
  model_id: "<approved-model-id>"
  model_revision: "<immutable-commit-or-artifact-version>"
  tokenizer_revision: "<matching-immutable-revision>"
  model_manifest_sha256: "<verified-hash>"
  license_review: "<approval-reference>"
  vllm_image_digest: "sha256:<verified-digest>"
  vllm_version: "<tested-version>"
  pytorch_cuda_stack: "<tested-build>"
  dtype: "<validated-precision>"
  tensor_parallel_size: 1
  pipeline_parallel_size: 1
  max_model_len: "<measured-safe-limit>"
  rollback_release: "<known-good-release-id>"
```

A model registry alias such as `latest` is insufficient for deterministic rollback. Store provenance and checksums without exposing private artifact URLs or access credentials.

## 5. Tokenization and chat templates

Tokenizers map text to token IDs; the model processes tokens, not characters. Different tokenizers can produce different IDs and token counts for the same text. Chat templates encode roles, separators and generation prompts in model-specific formats. A model may support raw completions but not chat completions without a compatible chat template.

**Validation checklist:** matching model/tokenizer revision; known special token IDs; correct context accounting; prompt truncation policy; documented chat template; stop-token behavior; streaming completion; synthetic golden response. Do not assume the tokenizer's declared max length equals the engine's validated usable context.

For an **approved local lab with already-installed libraries and locally available artifacts**:

```python
from transformers import AutoTokenizer

model_path = "/approved/local/model"
tokenizer = AutoTokenizer.from_pretrained(
    model_path,
    local_files_only=True,
    trust_remote_code=False,
)
sample = "Explain the word readiness in one sentence."
encoded = tokenizer(sample, add_special_tokens=True)
print("token_count:", len(encoded["input_ids"]))
print("special_tokens:", tokenizer.special_tokens_map)
```

Use synthetic inputs; avoid printing production prompts, access tokens or full model manifests containing private endpoints.

## 6. Loading stages and observability

| Stage | Useful signals | Common bottleneck |
|---|---|---|
| Image pull | Pull duration, registry errors | Image size/identity |
| Artifact access | 401/403/404, cache hit, download time | IAM/KMS, network, storage |
| Artifact validation | Hash, shard inventory, architecture metadata | Corrupt/incomplete artifacts |
| Tokenizer init | Parser/template errors, load time | Revision mismatch |
| CUDA initialization | Driver/device visibility, CUDA errors | Runtime compatibility |
| Parallel worker init | Rank mapping, NCCL timeout | GPU topology/interconnect |
| Weight load | GPU/host RAM, VRAM, duration | Insufficient memory |
| KV cache planning | Blocks, capacity, context/concurrency | VRAM exhaustion |
| Warmup/readiness | Startup time, ready endpoints | Probe budget/model errors |

Instrument each stage where the installed vLLM/TrueFoundry version supports it. Never treat a single startup log message as proof of full readiness.

## 7. GPU memory budget

An approximate unquantized weight footprint is `parameter_count × bytes_per_weight`, but actual allocation depends on precision, quantization metadata, temporary workspaces, activation peaks, CUDA context, parallelism and runtime implementation.

For decoder-only attention, a rough KV-cache estimate:

```text
KV bytes ≈ 2 × layers × KV heads × head_dim × live_tokens × bytes_per_element
```

This is an approximation: tensor parallelism, grouped-query attention, sliding-window attention, cache block rounding, reuse and backend features change real allocation. Reserve headroom for model load peaks and long prompts. A successful warmup at concurrency 1 does not prove production capacity.

## 8. Tensor parallelism (TP)

TP distributes supported model operations across multiple GPU workers, typically reducing per-device weight footprint while introducing collective communication. It may be needed when one GPU cannot fit the model; it is not automatically faster for small models or low concurrency.

Key requirements:

- A vLLM release and model architecture that support the intended TP size.
- Sufficient **allocated** GPUs, with compatible topology and device visibility.
- A correct rank-to-device mapping and compatible NCCL/PyTorch/CUDA stack.
- Interconnect/network capacity, especially across hosts.
- KV-cache and batch limits sized for per-rank VRAM, not merely total VRAM.
- Clear failure ownership across scheduler, worker processes and collectives.

`tensor_parallel_size` should match the intended GPU worker topology. For example, TP=2 generally requires two usable GPUs assigned to that instance. Actual support and multi-node configuration are version dependent.

## 9. Tensor vs pipeline parallelism

| Topic | Tensor parallelism | Pipeline parallelism |
|---|---|---|
| Partitioning | Splits supported layer computations/tensors | Splits model layers into stages |
| Communication | Frequent collectives during execution | Activations between pipeline stages |
| Strength | Per-device weight distribution | Layer-stage memory distribution |
| Risk | Interconnect/NCCL overhead | Pipeline bubbles/imbalanced stages |
| Operational check | Rank/device mapping, collective health | Stage placement, inter-stage health |

Multi-node serving adds DNS, network policies, port ranges, rendezvous, clock/timeout behavior and host failure domains. Do not expose worker control ports publicly. Validate the supported deployment architecture before enabling cross-node parallelism.

## 10. NCCL troubleshooting

NCCL supports GPU collective communication for distributed execution. Common symptoms include initialization timeout, worker rank crash, topology mismatch, collective hang and performance regression.

**[SAFE-READ] investigation:**

```bash
kubectl config current-context
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <vllm-pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> logs <vllm-pod> --tail=200
kubectl -n <namespace> logs <vllm-pod> --previous --tail=200
kubectl describe node <gpu-node>
```

Where already available in an authorized container:

```bash
nvidia-smi
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.device_count())"
python -c "import vllm; print(vllm.__version__)"
```

Inspect sanitized worker/rank errors and GPU topology from approved observability. Do not enable verbose NCCL debug logging indefinitely in production; it may increase log volume and expose infrastructure details. Do not disable transport/security settings blindly to bypass a hang.

## 11. Readiness and golden inference

The readiness contract must ensure the **expected immutable model revision** has loaded and the engine can accept work. Kubernetes Service EndpointSlices should include only ready backends. Validate with an authorized synthetic request to the version-supported API, checking HTTP status, response schema, served model ID, bounded output and completion of any stream.

A valid 200 response with the wrong model revision is a failed release acceptance. Keep model revision in release evidence and low-cardinality telemetry.

## 12. Troubleshooting scenarios

| ID | Symptom | First failed transition | Investigation |
|---|---|---|---|
| M1 | Model artifact 403 | Workload identity → object | Scoped IAM/KMS |
| M2 | Model artifact 404 | Revision → object | Immutable reference |
| M3 | Weight shard missing | Manifest → shard | File inventory |
| M4 | Hash mismatch | Download → integrity | Reject and re-fetch approved artifact |
| M5 | Unsupported model architecture | Config → vLLM | Support matrix |
| M6 | Tokenizer revision mismatch | Tokenizer → model | Pin compatible revision |
| M7 | Chat template absent | Chat request → prompt | Model/API contract |
| M8 | Context length invalid | Input → engine | Config and token counts |
| M9 | CPU RAM OOM during loading | Download/load → host memory | Loader footprint |
| M10 | GPU VRAM OOM during weights | Weights → VRAM | Precision/model size |
| M11 | KV-cache allocation failure | Engine → cache | Context/concurrency budget |
| M12 | TP size exceeds GPU allocation | Worker map → devices | Resource requests |
| M13 | NCCL init timeout | Workers → collectives | Topology/network |
| M14 | Rank exits during loading | Parallel worker → engine | Per-rank logs |
| M15 | Slow cold start | Storage/cache → model load | Timings and IOPS |
| M16 | Running but not Ready | Engine → probe | Load/warmup status |
| M17 | Wrong model after rollback | Mutable alias → runtime | Loaded revision |
| M18 | Latency worse after TP | Collectives → execution | Interconnect/benchmark |

Use the **Lowest Proven Healthy Layer** and **First Failed Transition** to avoid speculative remediation.

## 13. Production recovery runbook

1. **Declare impact:** affected deployment, model ID/revision, startup failures, ready endpoints, error rate and TTFT/ITL.
2. **Freeze promotion:** preserve healthy revisions and artifact caches; do not delete shared storage.
3. **Localize:** image → identity → artifact → tokenizer/config → CUDA → worker/rank → weights → KV cache → readiness → request.
4. **Repair:** choose the narrowest confirmed fix with owner approval. For TP/NCCL issues, involve GPU/node/network owners before changing cluster settings.
5. **Rollback:** restore the known-good **image digest + model/tokenizer revisions + TP/PP settings + runtime/GPU resource policy** via authoritative TrueFoundry/GitOps.
6. **Verify:** actual loaded revision, ready endpoints, golden request, queue, TTFT/ITL, VRAM and stream completion.
7. **Document:** timeline, artifact checksums, compatibility matrix, LPHL/FFT, recovery time and follow-up actions.

## 14. Acceptance gates

| Gate | Required evidence |
|---|---|
| G1 | Approved artifact provenance and immutable revisions |
| G2 | Model/config/tokenizer compatibility and chat contract |
| G3 | Identity/storage integrity and loading-stage evidence |
| G4 | CPU/GPU/VRAM and KV-cache budget |
| G5 | TP/PP GPU topology and worker/rank mapping |
| G6 | NCCL/parallel-worker diagnostic approach |
| G7 | Model-aware readiness and authorized golden inference |
| G8 | Authoritative rollback and post-recovery validation |

## 15. Evidence record

```text
Date/time / owner / approval:
TrueFoundry workspace, cluster, namespace:
Image digest / vLLM and CUDA/PyTorch versions:
Model ID/revision / tokenizer revision / artifact hash:
Artifact source/cache, workload identity:
GPU SKU/count, TP/PP, rank placement:
Weight footprint, VRAM, KV-cache capacity:
Model load/warmup and Ready transition:
Golden inference and chat-template result:
NCCL/worker diagnostics (sanitized):
Lowest Proven Healthy Layer:
First Failed Transition:
Known-good rollback and post-recovery SLO:
```

**Hands-on lab:** [7.4 — Model Loading, Tokenizers and Tensor Parallelism Check](../labs/04-model-loading-tokenizers-tensor-parallelism-check.md)  
**Next:** Chapter 7.5 — PagedAttention, KV Cache and Continuous Batching.

## References

- [vLLM Documentation](https://docs.vllm.ai/)
- [Hugging Face Transformers Documentation](https://huggingface.co/docs/transformers/)
- [NVIDIA NCCL Documentation](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [PyTorch Distributed Documentation](https://docs.pytorch.org/docs/stable/distributed.html)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
