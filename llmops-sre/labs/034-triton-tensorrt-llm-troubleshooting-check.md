# Chapter 034 — Hands-On Lab: Triton and TensorRT-LLM Troubleshooting

**Part 05 | Chapter 034 | Published lab plan; live execution pending**

## Goal

Perform structured incident triage across Kubernetes, Triton, TensorRT-LLM, model repository, GPU, KV cache, scheduler, distributed ranks, streaming, observability, security and FinOps. Produce evidence-based hypotheses, mitigation and RCA.

## Safety

**[SAFE-READ]** authorized Kubernetes/GPU state and sanitized logs. **[LAB-MUTATE]** separately approved isolated staging with synthetic workloads, bounded GPU-hours and explicit rollback. **[CHANGE-CONTROL]** production model control, runtime, driver, network, restart, load shedding and fault injection. No PHI, customer prompts, Secrets or unapproved tenant testing.

## Lab 0 — Preflight

Record incident ID, UTC, cluster/context, model/version, Triton/backend/runtime digests, engine/model/tokenizer hashes, GPU SKU/UUID, TP/PP, owners, SLO, approval and rollback.

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

## Lab 1 — Kubernetes and GPU [SAFE-READ]

```bash
kubectl -n MODEL_NAMESPACE describe deployment APPROVED_DEPLOYMENT
kubectl -n MODEL_NAMESPACE get pods -o wide
kubectl -n MODEL_NAMESPACE get endpointslices
kubectl -n MODEL_NAMESPACE logs deployment/APPROVED_DEPLOYMENT --since=20m --tail=200
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Classify Pending, ImagePullBackOff, CrashLoopBackOff, model unready, OOM or hardware fault. Sanitize logs.

## Lab 2 — Model and engine readiness [DESIGN]

Compare pinned model repository/config, engine digest, TensorRT-LLM/TensorRT/CUDA/driver, GPU architecture, tokenizer and quantization scales. Design a synthetic golden inference through the authenticated gateway.

## Lab 3 — Latency and queue triage [DESIGN]

Collect client TTFT/TPOT/E2E, terminal completion, queue and compute, input/output cohorts, concurrency, retries and batch efficiency. Avoid calling cumulative average duration p99.

## Lab 4 — KV and distributed triage [DESIGN]

Compute logical KV/token from model layers, KV heads, head dimension and precision. Compare with allocator block metrics and per-rank VRAM. Define rank/NCCL evidence and cancellation cleanup.

## Lab 5 — Rollback and RCA [DESIGN]

Specify known-good complete release bundle, approval, canary, model readiness, golden fixtures, terminal streaming, SLO, GPU memory, telemetry and cost verification. Record hypotheses, root cause and preventive controls.

## Lab 6 — Eight tabletop game days

01 device plugin failure; 02 model unready; 03 engine GPU mismatch; 04 KV exhaustion and retries; 05 batch p99 regression; 06 NCCL rank hang; 07 model-control exposure; 08 incomplete rollback.

## Twenty-four failure checklist

F01 device plugin; F02 Pod Pending; F03 image pull; F04 process crash; F05 model readiness; F06 repository drift; F07 engine; F08 quant scales; F09 tokenizer; F10 startup OOM; F11 KV exhaustion; F12 fragmentation; F13 queue; F14 batching; F15 priority; F16 retries; F17 streaming; F18 cancellation; F19 rank; F20 NCCL; F21 GPU fault; F22 telemetry; F23 admin exposure; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Incident severity/ownership | Pending |
| G02 | UTC evidence | Pending |
| G03 | GPU/Pod triage | Pending |
| G04 | Image/model provenance | Pending |
| G05 | Process/model readiness | Pending |
| G06 | Engine compatibility | Pending |
| G07 | Tokenizer/quality | Pending |
| G08 | Startup OOM | Pending |
| G09 | KV/allocator | Pending |
| G10 | Scheduler queue | Pending |
| G11 | TTFT/TPOT/E2E | Pending |
| G12 | Complete streaming | Pending |
| G13 | TP/PP/NCCL | Pending |
| G14 | GPU hardware | Pending |
| G15 | Telemetry freshness | Pending |
| G16 | Security/privacy | Pending |
| G17 | SLO/error budget | Pending |
| G18 | Full-stack rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
Incident/UTC/context/approval:
Affected model/engine/tokenizer digests:
Kubernetes events/Pod/Service:
Triton/backend readiness:
GPU/driver/CUDA/TP/PP/NCCL:
TTFT/TPOT/queue/compute:
KV/VRAM/batching/retries:
Streaming terminal completion:
Security/privacy/telemetry:
SLO/error budget/GPU-hours:
Hypotheses/mitigation/rollback:
Root cause/actions/owners/reviewer:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop separately approved staging tests, restore authoritative runtime/model configuration, remove only owned resources and reconcile GPU-hours. Acceptance gates remain pending until independently reviewed.

**Next: Chapter 035 — NVIDIA Inference Stack Acceptance Lab.**
