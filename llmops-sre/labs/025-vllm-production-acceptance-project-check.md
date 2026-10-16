# Chapter 025 — Hands-On Lab: vLLM Production Acceptance Project

**Part 04 | Chapter 025 | Capstone lab published; execution and sign-off pending**

## Goal

Combine Chapters 016–024 into an independently reviewed, 100-point vLLM production readiness assessment covering deployment, GPU/KV, streaming, security, observability, recovery and FinOps.

## Safety and authorization

**[SAFE-READ]** approved cluster, GPU, model, metrics and redacted logs. **[LAB-MUTATE]** synthetic isolated staging with approved GPU-hour, token, duration and concurrency caps. **[CHANGE-CONTROL]** production serving, gateway, secrets, network policy, fault injection and rollback. No PHI, customer prompts, credentials or unauthorized security tests.

## Lab 0 — Evidence preflight

Record UTC, context/namespace, model/tokenizer/template revision, vLLM image digest, GPU SKU/topology, driver/CUDA/NCCL, gateway, ServiceAccount, network policy, metrics/OTel/DCGM, on-call, approvers and rollback.

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc,ingress -A -o wide
kubectl get networkpolicy -A
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

## Lab 1 — Architecture and provenance [DESIGN]

Draw client → gateway → Kubernetes Service → vLLM → GPU/ranks → telemetry. Pin model weights, tokenizer/template, image digest and artifact checksum. Review license, remote-code policy, credentials and workload identity.

## Lab 2 — GPU/KV envelope [DESIGN]

Calculate KV bytes/token for actual model architecture and per-rank allocations. Inventory weights, KV, workspace, graphs, communication buffers, other processes and headroom. Define bounded short/long prompt and output cohorts, context admission and preemption thresholds.

## Lab 3 — API and SSE contract [DESIGN; optional staging]

Validate approved model alias, non-streaming JSON, streaming SSE, finish reason, terminal marker, partial HTTP 200 failure classification, gateway flushing, cancellation and bounded retries. Do not treat a successful HTTP status alone as acceptance.

## Lab 4 — Performance and FinOps [DESIGN; optional staging]

Measure or plan p50/p95/p99 TTFT, TPOT and E2E, queue wait, token throughput, KV/preemption, per-rank GPU VRAM, valid completed requests and GPU-hours/valid completion. Define SLO and cost thresholds **before** testing. Mark unmeasured metrics Pending.

## Lab 5 — Security and privacy [DESIGN]

Check TLS, authentication, tenant authorization, rate/token quotas, RBAC, NetworkPolicy, DNS/approved egress, secrets, model provenance, distributed coordination port isolation and prompt-safe telemetry. Security sign-off is mandatory.

## Lab 6 — Observability and recovery [DESIGN]

Verify metric schema/freshness, structured redacted logs, trace propagation, GPU/DCGM metrics, complete SSE SLI, multi-window burn and missing-data alerts. Plan rollback through authoritative GitOps and define customer-path post-rollback checks.

## Lab 7 — Eight tabletop game days

01 model load OOM; 02 decode KV exhaustion; 03 partial SSE; 04 gateway timeout; 05 auth/tenant isolation; 06 NCCL/GPU failure; 07 telemetry outage; 08 bad rollout/rollback.

## Twenty-four failure checklist

F01 provenance; F02 remote code; F03 GPU capacity; F04 startup OOM; F05 prefill OOM; F06 KV; F07 context; F08 TTFT; F09 TPOT; F10 partial SSE; F11 buffering; F12 cancel; F13 retry; F14 auth; F15 tenant; F16 secrets; F17 coordinator; F18 DNS; F19 NCCL; F20 metric drift; F21 stale telemetry; F22 alert; F23 orphan worker; F24 GPU cost.

## 100-point scorecard

| Domain | Max | Verified points |
|---|---:|---:|
| Architecture/provenance | 15 | Pending |
| GPU/KV/performance | 20 | Pending |
| API/streaming | 15 | Pending |
| Security/privacy/tenancy | 20 | Pending |
| Observability/SLO | 10 | Pending |
| Recovery/DR | 15 | Pending |
| FinOps/operational handoff | 5 | Pending |
| **Total** | **100** | **Not scored** |

A score does not waive mandatory no-go conditions.

## Twenty mandatory gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Pinned runtime/model provenance | Pending |
| G02 | GPU topology/scheduling | Pending |
| G03 | Model readiness | Pending |
| G04 | GPU memory/KV budget | Pending |
| G05 | Context/concurrency admission | Pending |
| G06 | TTFT/TPOT/E2E | Pending |
| G07 | Complete SSE | Pending |
| G08 | Gateway timeout/cancel/retry | Pending |
| G09 | Authentication | Pending |
| G10 | Tenant authorization | Pending |
| G11 | Network/workload isolation | Pending |
| G12 | Secrets/model security | Pending |
| G13 | Privacy-safe telemetry | Pending |
| G14 | Metrics/GPU freshness | Pending |
| G15 | SLO/missing-data alerts | Pending |
| G16 | OOM/NCCL recovery | Pending |
| G17 | Proven rollback | Pending |
| G18 | GPU cost ceiling | Pending |
| G19 | On-call/runbooks | Pending |
| G20 | Independent sign-off | Pending |

## Evidence package template

```text
UTC/context/approval:
Image/model/tokenizer/driver:
Architecture and tenant boundaries:
GPU/KV/TP-PP:
API/SSE and quality:
TTFT/TPOT/E2E/queue:
Auth/RBAC/network/secrets:
Metrics/logs/traces/alerts:
Game-day results:
Rollback and GPU FinOps:
Score, mandatory gate status:
Open risks/owners/dates:
Platform/SRE reviewer:
Security reviewer:
Service owner:
GO / NO-GO / RETEST:
```

## Final decision

**GO** requires every mandatory gate passed with independent evidence, approved on-call/rollback and acceptable cost. **NO-GO** for any mandatory security, privacy, streaming, SLO, recovery or cost failure. **RETEST** when evidence is missing or stale. This lab is a plan; no real-world gate is yet verified.

## Cleanup

Read-only/tabletop work creates no resources. Stop separately approved staging load, remove only owned artifacts through the authoritative controller, verify no orphan GPU workers and reconcile GPU-hours.

**Part 04 written curriculum complete (10/10); live execution and review pending.**

**Next: Chapter 026 — Triton Inference Server Architecture.**
