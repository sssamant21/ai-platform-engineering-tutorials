# Chapter 017 — Hands-On Lab: vLLM Model Loading and Runtime Configuration

**Part 04 | Chapter 017 | Published lab plan; live execution pending**

## Goal

Build a version-pinned model loading manifest, design a safe staging vLLM configuration, validate model/tokenizer/artifact provenance and prepare cold-start, CUDA OOM, readiness, correctness and rollback acceptance evidence.

## Safety

**[SAFE-READ]** approved metadata, redacted logs and GPU inventory. **[LAB-MUTATE]** approved isolated staging vLLM deployment and synthetic API checks. **[CHANGE-CONTROL]** production model/image/driver/context/parallelism/traffic changes. Never use customer prompts, PHI, secrets or unapproved remote model code.

## Lab 0 — Authorization and inventory

Record UTC, cluster/context, namespace, GPU SKU/count, driver/CUDA, vLLM version/image digest, model and tokenizer revisions, chat template, quantization, license, artifact checksum, staging budget, reviewer and rollback owner.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — Immutable release worksheet [DESIGN]

| Field | Expected evidence |
|---|---|
| vLLM image digest | immutable sha256 |
| vLLM/CUDA/driver | compatibility matrix |
| Model weights | approved ID/revision/checksum |
| Tokenizer/chat template | pinned and tested |
| dtype/quantization | hardware and quality validation |
| Context length | approved token budget |
| GPU memory utilization | measured headroom |
| Concurrency/batched tokens | staging SLO matrix |
| TP/PP | rank and GPU topology |
| Artifact access | workload identity and egress |
| Rollback | known-good release reference |

No actual values are supplied by this tutorial; do not invent them.

## Lab 2 — vLLM CLI verification [DESIGN; optional staging]

For a **pinned, installed** vLLM version:

```bash
vllm --help
vllm serve --help
```

Check exact flags and defaults for model, tokenizer, dtype, quantization, context, GPU memory, batching and parallelism. This may be unavailable in a read-only environment; mark blocked rather than installing software without approval.

## Lab 3 — Model loading failure tabletop

Trace the loading path for: registry 403, corrupt artifact, missing cache, wrong tokenizer, unsupported dtype, load-time OOM, graph-capture OOM, rank/NCCL failure and early readiness. For each, record first evidence, affected layer, owner, safe mitigation and rollback.

## Lab 4 — Authorized GPU diagnostics [SAFE-READ]

On an approved GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,driver_version,memory.total,memory.used --format=csv
```

For a verified serving workload:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=20m --tail=150
```

Redact sensitive logs. Distinguish weight allocation, workspace, KV cache and other GPU processes.

## Lab 5 — Synthetic staging API [DESIGN; optional approved execution]

```bash
curl --fail-with-body --no-buffer --max-time 120 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $STAGING_API_TOKEN" \
  -d '{"model":"APPROVED_MODEL_ALIAS","messages":[{"role":"user","content":"Return exactly: READY"}],"max_tokens":32,"stream":true}' \
  "$STAGING_BASE_URL/v1/chat/completions"
```

Check exact API support for the pinned release, auth, model alias, finish reason, terminal SSE, correctness, cancellation and complete connection close. Do not store credentials in logs.

## Lab 6 — Cold-start and configuration matrix [DESIGN]

| Test cohort | Expected measurements |
|---|---|
| cold cache startup | artifact download, load, warmup |
| warm cache startup | load and ready time |
| short prompt/output | TTFT, TPOT, success |
| long prompt/output | KV/cache, p95 latency |
| structured output | JSON/tool correctness |
| bounded concurrency | queue, preemption, OOM |
| rollback | model revision and complete SSE |

Define abort limits and GPU-hour budget before optional staging execution.

## Lab 7 — Eight tabletop game days

01 registry 403; 02 cache lost; 03 tokenizer mismatch; 04 BF16 OOM; 05 unsupported quantization; 06 early readiness; 07 wrong alias; 08 GPU cost and stream interruption. Default tabletop.

## Twenty-four failure checklist

F01 registry 403; F02 missing revision; F03 checksum; F04 cache disk; F05 mutable drift; F06 remote code; F07 tokenizer; F08 template; F09 alias; F10 dtype; F11 quant kernel; F12 weights OOM; F13 graph OOM; F14 context; F15 concurrency; F16 TP rank; F17 NCCL; F18 startup; F19 readiness; F20 warm cache; F21 quality; F22 JSON/tools; F23 SSE; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Immutable release provenance | Pending |
| G02 | License and security | Pending |
| G03 | Registry identity | Pending |
| G04 | Cache and checksum | Pending |
| G05 | Tokenizer/template | Pending |
| G06 | dtype compatibility | Pending |
| G07 | Quantization quality | Pending |
| G08 | GPU VRAM | Pending |
| G09 | Context/admission | Pending |
| G10 | TP/NCCL | Pending |
| G11 | Cold/warm startup | Pending |
| G12 | Readiness | Pending |
| G13 | Complete SSE | Pending |
| G14 | JSON/tool quality | Pending |
| G15 | TTFT/TPOT | Pending |
| G16 | Privacy/quotas | Pending |
| G17 | Observability | Pending |
| G18 | FinOps | Pending |
| G19 | Rollback | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/approval:
GPU/driver/CUDA/vLLM:
Image digest/model/tokenizer/template:
License/artifact checksum/access:
dtype/quantization/context/concurrency/TP:
Cold vs warm load timings:
Weights/KV/workspace/headroom:
Synthetic correctness and complete SSE:
TTFT/TPOT/queue/GPU cost:
Game-day mode and findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop any separately approved staging load, remove only resources owned by the change through the authoritative controller, and reconcile GPU-hours. Leave acceptance gates pending until independent review.

**Next: Chapter 018 — Continuous Batching, PagedAttention and KV Cache.**
