# Chapter 035 — Hands-On Lab: NVIDIA Inference Stack Acceptance

**Part 05 capstone | Chapter 035 | Written execution worksheet — no live tests executed**

## Mission

Produce a complete, auditable staging acceptance package for Kubernetes + Triton + TensorRT-LLM, from artifact provenance through SLO-qualified inference, security, recovery and FinOps. Publishing this lab does **not** certify production readiness.

## Safety and approval

**[SAFE-READ]** authorized Kubernetes/GPU inventory and sanitized logs. **[LAB-MUTATE]** separately approved isolated staging, synthetic workloads, bounded concurrency/time and GPU-hour budget. **[CHANGE-CONTROL]** production model/engine, GPU, network, routing, fault injection and rollback. Do not use PHI, customer prompts, Secrets or unauthorized tenant data.

## Lab 0 — Acceptance charter

Record change ID, scope, owner, reviewer, approval, time window, GPU quota, test budget, SLO, synthetic dataset, abort conditions, known-good rollback and evidence destination.

```text
Change ID:
UTC window:
Owner / reviewer:
Staging context / namespace:
GPU SKU / count / quota:
Triton / TensorRT-LLM / backend:
Model / tokenizer / engine digests:
Synthetic dataset:
Approved SLO / cost ceiling:
Abort and rollback conditions:
```

## Lab 1 — Read-only platform inventory

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

For approved namespace:

```bash
kubectl -n MODEL_NAMESPACE get pods -o wide
kubectl -n MODEL_NAMESPACE describe deployment APPROVED_DEPLOYMENT
kubectl -n MODEL_NAMESPACE get endpointslices
```

On authorized GPU host:

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

## Lab 2 — Immutable release and security review

Verify image, model, tokenizer, engine, quantization scales, runtime/backend, gateway policy and known-good rollback digests. Review TLS/auth, ServiceAccount, namespace, NetworkPolicy, artifact IAM, metrics/model-control isolation and privacy-safe telemetry. Record evidence links.

## Lab 3 — Readiness and correctness

Design/execute only when approved: process health, server readiness, model-version readiness, synthetic golden prompts, structured output, stop/EOS, quality/safety and terminal streaming completion. Distinguish HTTP success from valid completion.

## Lab 4 — Performance and KV/VRAM

Test short/short, long/short, short/long, long/long, mixed and burst synthetic cohorts. Capture TTFT, TPOT, E2E p95/p99, valid completion, throughput, queue/batch, per-GPU VRAM, KV allocator, SLO and GPU-hours. Abort on OOM, runaway retries, SLO burn or cost ceiling.

## Lab 5 — Resilience and rollback

Default to tabletop for GPU loss, engine incompatibility, KV exhaustion, rank failure, gateway policy and rollback drift. For separately approved staging injection, measure detection, mitigation, restoration and evidence. Verify **complete-release** rollback, not just Pod revision.

## Lab 6 — Eight tabletop game days

01 device plugin; 02 model repository outage; 03 engine incompatibility; 04 KV exhaustion; 05 batching p99 regression; 06 TP rank failure; 07 admin exposure; 08 incomplete rollback.

## Twenty-four failure checklist

F01 device plugin; F02 scheduling; F03 registry; F04 repository; F05 checksum; F06 engine; F07 CUDA; F08 quant quality; F09 tokenizer; F10 startup OOM; F11 KV; F12 batching; F13 queue; F14 retries; F15 streaming; F16 cancellation; F17 TP/PP; F18 NCCL; F19 GPU node; F20 CNI; F21 exposure; F22 telemetry; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G01 | Change approval and scope | Pending |
| G02 | Immutable release manifest | Pending |
| G03 | Model/license/provenance | Pending |
| G04 | GPU/runtime compatibility | Pending |
| G05 | Scheduling/storage | Pending |
| G06 | Server/model readiness | Pending |
| G07 | Golden correctness | Pending |
| G08 | Terminal streaming | Pending |
| G09 | TLS/auth/admin isolation | Pending |
| G10 | NetworkPolicy/privacy | Pending |
| G11 | TTFT/TPOT/E2E | Pending |
| G12 | Throughput/goodput | Pending |
| G13 | Queue/batch fairness | Pending |
| G14 | KV/VRAM headroom | Pending |
| G15 | TP/PP/NCCL | Pending |
| G16 | Failure response/HA | Pending |
| G17 | Full-stack rollback | Pending |
| G18 | Telemetry/SLO alerts | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Final sign-off worksheet

```text
Acceptance run ID:
UTC start/end:
Change and approval:
Model/runtime/GPU release digests:
Test cohort and client metrics:
Security/privacy review:
Failure/tabletop evidence:
Rollback verification:
SLO/error budget:
GPU-hours and cost:
G01–G20 reviewer/evidence:
Exceptions/remediation:
Decision: HOLD (default) / APPROVED
Authorized reviewer/date:
```

## Cleanup

Read-only/tabletop work creates no resources. For approved staging tests, stop load, restore GitOps source of truth, remove only owned test resources, close port-forwards, retain sanitized evidence and reconcile GPU-hours.

**Current outcome:** HOLD / not executed. All gates Pending until measured and independently reviewed.

**Part 05 written curriculum complete (026–035). Next: Chapter 036 — Ray Serve Architecture and Deployment.**
