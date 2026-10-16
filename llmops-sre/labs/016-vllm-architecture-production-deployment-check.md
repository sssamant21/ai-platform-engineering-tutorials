# Chapter 016 — Hands-On Lab: vLLM Architecture and Production Deployment

**Part 04 | Chapter 016 | Published lab plan; not executed**

## Goal

Create a reviewer-ready vLLM deployment design, validate GPU prerequisites, pin model/runtime provenance, test synthetic API and complete-stream correctness where staging is approved, and prepare rollout/rollback and incident response.

## Safety

**[SAFE-READ]** approved cluster metadata and redacted logs. **[LAB-MUTATE]** separately approved isolated staging deployment and synthetic requests with cost, token and time limits. **[CHANGE-CONTROL]** production model, GPU, ingress, routing or rollout. Never use customer prompts, PHI or live credentials in evidence.

## Lab 0 — Preflight

Record UTC, context/namespace, GPU SKU/count, device plugin, driver/CUDA, vLLM image digest/version, approved model/tokenizer/template revision, license, model source and auth, owner, cost ceiling and rollback authority.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — vLLM architecture [DESIGN]

Diagram client → auth/gateway → Kubernetes Service → vLLM API → engine/scheduler → prefill/KV/decode → SSE. Add model artifact source, GPU workers, observability, workload identity and GitOps. Annotate failure boundaries and owners.

## Lab 2 — Runtime and GPU inventory [SAFE-READ]

On an authorized GPU node:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,driver_version,memory.total,memory.used --format=csv
```

Record model weights, estimated KV, workspace, context, concurrency and headroom. A successful nvidia-smi result is not proof of CUDA or model readiness.

## Lab 3 — Manifest review [DESIGN]

Use the illustrative Deployment in the tutorial as a **review template**, not a ready-to-apply manifest. Validate namespace, immutable image, approved model path, device resources, model-load startup budget, readiness semantics, authentication, secrets, egress, GPU scheduling, graceful termination and GitOps ownership. Do not run kubectl apply without approval.

## Lab 4 — Synthetic API validation [DESIGN; optional approved staging]

For an approved endpoint only:

```bash
curl --fail-with-body --no-buffer --max-time 120 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $STAGING_API_TOKEN" \
  -d '{"model":"APPROVED_MODEL_ALIAS","messages":[{"role":"user","content":"Return exactly: READY"}],"max_tokens":32,"stream":true}' \
  "$STAGING_BASE_URL/v1/chat/completions"
```

Confirm auth and exact endpoint contract for the pinned vLLM version. Do not echo tokens or save raw authorization headers. Validate SSE framing, first output token, finish reason, terminal completion and clean close; classify HTTP 200 partial streams as failures.

## Lab 5 — Performance and observability [DESIGN]

Create a bounded workload matrix for short/long prompts and short/long outputs. Measure TTFT, TPOT/ITL, E2E, complete-stream success, queue wait, KV occupancy/preemption, GPU VRAM, errors, retries and GPU-hours. Metric names are release-specific; verify exporter schema.

## Lab 6 — Rollout and rollback [TABLETOP]

Describe image/model pinning, staging warmup, canary routing, health checks, disruption budgets, GPU spare capacity, rollback via GitOps and sustained customer-path recovery. Set abort thresholds before any execution.

## Lab 7 — Eight game days

01 model OOM; 02 tokenizer mismatch; 03 early readiness; 04 TTFT queue saturation; 05 SSE buffering; 06 NCCL rank crash; 07 rollout stream loss; 08 GPU cost spike. Default tabletop.

## Twenty-four failure checklist

F01 GPU; F02 CUDA; F03 image; F04 artifact; F05 tokenizer; F06 context; F07 load OOM; F08 KV OOM; F09 cold start; F10 readiness; F11 probes; F12 auth; F13 quota; F14 queue; F15 prefill; F16 decode; F17 buffering; F18 partial SSE; F19 cancellation; F20 rollout; F21 NCCL; F22 metrics; F23 privacy; F24 FinOps.

## Eighteen acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Architecture and ownership | Pending |
| G02 | Cluster/GPU preflight | Pending |
| G03 | Pinned image/model/tokenizer | Pending |
| G04 | CUDA compatibility | Pending |
| G05 | Model artifact security | Pending |
| G06 | Startup/readiness contract | Pending |
| G07 | Manifest security review | Pending |
| G08 | Context/KV budget | Pending |
| G09 | Synthetic API validation | Pending |
| G10 | Complete SSE | Pending |
| G11 | TTFT/TPOT SLO | Pending |
| G12 | Queue/fairness | Pending |
| G13 | GPU/DCGM telemetry | Pending |
| G14 | Auth/quota/privacy | Pending |
| G15 | TP/NCCL | Pending |
| G16 | Rollout/rollback | Pending |
| G17 | FinOps | Pending |
| G18 | Reviewer signoff | Pending |

## Evidence template

```text
UTC/context/approval:
GPU SKU/driver/CUDA:
vLLM image digest/version:
Model/tokenizer/template/license:
Artifact access and identity:
Manifest/startup/readiness:
Synthetic API and complete SSE:
TTFT/TPOT/queue/KV/GPU:
Game-day mode and findings:
Cost/rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. For approved staging, stop synthetic load, remove only owned deployment resources through the authoritative controller and reconcile GPU-hours. Leave all gates pending until independently reviewed.

**Next: Chapter 017 — vLLM Model Loading and Runtime Configuration.**
