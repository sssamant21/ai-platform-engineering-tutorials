# Lab 7.4 — Model Loading, Tokenizers and Tensor Parallelism Check

**Canonical:** [Chapter 7.4](../04-model-loading-tokenizers-tensor-parallelism/04-model-loading-tokenizers-tensor-parallelism.md)  
**Default:** [SAFE-READ]. Artifact changes, distributed execution tests and rollout/rollback require [LAB-MUTATE] in an approved isolated nonproduction workspace.

## 1. Lab outcomes

Inventory model and tokenizer revisions, trace loading stages, estimate GPU/KV memory, verify TP rank allocation, diagnose NCCL and tokenizer errors, classify 18 failure scenarios, and document authoritative rollback.

## 2. Preflight

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Record approved TrueFoundry workspace, deployment owner, image digest, model/tokenizer revision, license approval and rollback release. Use only synthetic inputs. No credential dumps.

## 3. Model provenance inventory

| Artifact | Revision/hash | Verified? | Evidence |
|---|---|---|---|
| vLLM image digest | | | |
| Model config/architecture | | | |
| Weight manifest and shards | | | |
| Tokenizer/config | | | |
| Special tokens/chat template | | | |
| Model license/security review | | | |
| Known-good release | | | |

## 4. Loading-stage investigation [SAFE-READ]

```bash
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> logs <pod> --tail=200
kubectl -n <namespace> logs <pod> --previous --tail=200
kubectl describe node <gpu-node>
```

| Stage | Start/end or duration | Evidence | Result |
|---|---|---|---|
| Image pull | | | |
| Model artifact fetch | | | |
| Integrity validation | | | |
| Tokenizer/config init | | | |
| CUDA/worker init | | | |
| Weight allocation | | | |
| KV cache planning | | | |
| Warmup/readiness | | | |

Missing timing instrumentation is a documented gap, not a successful check.

## 5. Tokenizer and chat-template exercise

In an approved local lab with preinstalled Transformers and vetted local artifacts:

```python
from transformers import AutoTokenizer
tok = AutoTokenizer.from_pretrained(
    "/approved/local/model",
    local_files_only=True,
    trust_remote_code=False,
)
text = "Return a short explanation of readiness."
print("tokens:", len(tok(text)["input_ids"]))
print("special tokens:", tok.special_tokens_map)
```

Check model/tokenizer revision alignment and supported chat template. Do not run arbitrary remote code or use sensitive prompts.

## 6. Memory and TP sizing

Record layers, KV heads, head dimension, bytes per KV element, estimated live tokens, weight precision, GPU count, VRAM, TP/PP size and estimated headroom. Compare approximate formula to actual engine-reported cache capacity.

| Quantity | Planned | Observed |
|---|---|---|
| GPU SKU/count | | |
| Tensor parallel size | | |
| Pipeline parallel size | | |
| Weight footprint | | |
| KV-cache footprint | | |
| Startup VRAM peak | | |
| Steady-state VRAM | | |
| Context/concurrency limit | | |

## 7. Optional isolated multi-GPU exercise [LAB-MUTATE]

Only if two approved GPUs are available:

1. Verify both GPUs are allocatable and assigned to the **lab-owned** Pod.
2. Deploy a vetted model using a supported vLLM version and a tested TP configuration.
3. Capture rank-to-device mapping and NCCL initialization logs with sensitive fields redacted.
4. Confirm warmup, readiness and a synthetic golden inference request.
5. Compare TP=1 and TP=2 only when the model fits and both configurations are supported; record throughput, TTFT/ITL and VRAM.
6. Restore the known-good release and clean up lab-owned resources.

Do not change NCCL network settings or device plugins on shared nodes. If GPU capacity is unavailable, mark this exercise **Not run** and complete a tabletop.

## 8. Failure classification

| ID | Scenario | LPHL | FFT | Evidence | Owner |
|---|---|---|---|---|---|
| M1 | Artifact 403 | | | | |
| M2 | Artifact 404 | | | | |
| M3 | Missing weight shard | | | | |
| M4 | Checksum mismatch | | | | |
| M5 | Unsupported architecture | | | | |
| M6 | Tokenizer mismatch | | | | |
| M7 | Missing chat template | | | | |
| M8 | Invalid context length | | | | |
| M9 | Host RAM OOM | | | | |
| M10 | Weight VRAM OOM | | | | |
| M11 | KV-cache allocation failure | | | | |
| M12 | TP exceeds GPU count | | | | |
| M13 | NCCL timeout | | | | |
| M14 | Rank crash | | | | |
| M15 | Slow cold start | | | | |
| M16 | Never Ready | | | | |
| M17 | Wrong model after rollback | | | | |
| M18 | TP latency regression | | | | |

Classify by lowest proven healthy layer and first failed transition; never claim a tabletop as a live execution.

## 9. Acceptance gates

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| G1 Immutable provenance | | |
| G2 Tokenizer/chat compatibility | | |
| G3 Storage/identity/load stages | | |
| G4 CPU/GPU/KV budget | | |
| G5 TP/PP rank mapping | | |
| G6 NCCL diagnostics | | |
| G7 Readiness/golden request | | |
| G8 Rollback/cleanup | | |

## 10. Evidence report

```text
Date/time and approval:
TrueFoundry workspace/cluster/namespace:
Image digest/vLLM/CUDA/PyTorch:
Model and tokenizer revisions:
Artifact hash, license, storage identity:
Loading-stage timings and warmup:
GPU/VRAM, TP/PP, KV cache:
Rank/NCCL evidence:
Synthetic golden inference:
M1–M18 classifications:
Lowest Proven Healthy Layer:
First Failed Transition:
Known-good rollback and SLO:
Unexecuted gates and cleanup:
```

**Next:** Chapter 7.5 — PagedAttention, KV Cache and Continuous Batching.
