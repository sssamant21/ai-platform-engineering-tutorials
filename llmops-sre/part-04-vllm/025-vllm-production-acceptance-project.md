# Chapter 025 — vLLM Production Acceptance Project

**Part 04 — vLLM Administration and Operations | Chapter 025 of 100 | Part 04 capstone**

> **Status:** Written capstone published; technical/security review, live labs, evidence collection and operational sign-off pending. This document defines acceptance criteria, not a claim that production is certified.

## Objective and scope

Integrate Chapters 016–024 into a reproducible, evidence-driven **production readiness review** for an authenticated, observable, reliable and cost-controlled vLLM inference service. The capstone combines deployment, model loading, continuous batching, KV cache, streaming APIs, GPU/context tuning, TP/PP distributed serving, security, tracing and incident recovery.

**Pass condition:** Every mandatory gate has verified evidence, accountable owner, review and an approved rollback. Publication alone never satisfies a gate.

## 1. Reference architecture

```text
Synthetic client + contract/SSE validator
              │
      TLS/Auth/Quota Gateway
              │
   Kubernetes Service + routing
              │
       vLLM API + scheduler
          │          │
    model/tokenizer   queue + KV
          │          │
       GPU worker(s) / TP-PP ranks
              │
  NCCL/coordination (only if distributed)
              │
   Prometheus + DCGM + OTel + logs
              │
     SLO burn + incident response
              │
     GitOps rollback + FinOps
```

The serving backend must not be directly exposed to untrusted clients. Validate network isolation, secrets, artifact provenance and model-specific capability contracts.

## 2. Acceptance inventory

Record and freeze:
- cluster, namespace, environment, Kubernetes and GPU driver versions;
- GPU SKU, memory, topology, device plugin and node allocation;
- vLLM image digest, runtime version, model/tokenizer/template revision and license;
- model alias, endpoints, supported streaming/tool/JSON capabilities;
- max context, concurrency, batched-token and GPU memory configuration;
- TP/PP mapping, distributed executor, NCCL/CUDA compatibility if used;
- gateway TLS, authn/authz, quotas, NetworkPolicy and workload identity;
- Prometheus, OTel, DCGM exporter versions and retention;
- on-call owner, SLOs, cost ceiling, change window and rollback reference.

## 3. Evidence and scoring policy

A **100-point weighted readiness rubric** is used for prioritization, but a high numeric score **does not override mandatory safety gates**.

| Domain | Points | Evidence |
|---|---:|---|
| Architecture, deployment and provenance | 15 | immutable manifests, model checksum |
| GPU, KV and performance envelope | 20 | per-rank VRAM, TTFT/TPOT, load cohorts |
| API correctness and streaming | 15 | schema, complete SSE, cancellation |
| Security, privacy and tenant isolation | 20 | auth, RBAC, network policy, audit |
| Observability and SLOs | 10 | metrics/logs/traces, alert delivery |
| Reliability, recovery and DR | 15 | incident game days, rollback proof |
| FinOps and operational handoff | 5 | GPU-hours, runbooks, ownership |
| **Total** | **100** | **independently reviewed evidence** |

**Mandatory no-go conditions:** unauthorized exposure, credential leakage, cross-tenant access, missing rollback, unbounded GPU spend, unverified complete SSE, missing telemetry freshness, unapproved model provenance, untested critical recovery, or absent security sign-off. A 100/100 self-reported score without independent evidence is **not accepted**.

## 4. Staging environment and safety

**[SAFE-READ]** approved inventory, metrics, redacted logs and configuration. **[LAB-MUTATE]** only approved isolated staging with synthetic prompts, bounded token budgets, concurrency, duration and GPU-hours. **[CHANGE-CONTROL]** production model/runtime changes, gateway/network policies, GPU scheduling, fault injection and rollback.

Do not use PHI, customer prompts, production secrets or unapproved GPU stress. Capture sanitized artifacts in an access-controlled evidence location.

## 5. Workstream A — Deployment and model integrity

Acceptance tests:
1. immutable image digest and approved vLLM release;
2. pinned model/tokenizer/template revision and checksum;
3. verified model license, artifact access and no unapproved remote code;
4. GPU scheduling, readiness, model warmup and cold-start time;
5. deployment rollback and safe old/new replica coexistence.

Reject success based only on Pod Running. Verify model responds correctly to approved synthetic fixtures.

## 6. Workstream B — GPU and KV envelope

For conventional attention:

```text
KV bytes/token ≈ 2 × layers × KV_heads × head_dim × bytes/element
```

Synthetic example: 32 layers, 8 KV heads, head dimension 128, BF16:

```text
KV/token = 131,072 bytes = 128 KiB
32,768 resident tokens ≈ 4 GiB logical KV
```

Measure actual per-GPU weights, KV, workspace, CUDA graphs, communication buffers, allocator reserve, other processes and headroom. Include short/long prompt and output cohorts. Reject a capacity plan based on aggregate VRAM alone.

## 7. Workstream C — API and streaming contract

Test:
- model alias and supported endpoint/schema;
- unauthorized/invalid request rejection;
- non-streaming JSON and streaming SSE;
- UTF-8 framing across arbitrary network chunks;
- valid finish reason and terminal [DONE] for the pinned contract;
- HTTP 200 partial stream classified as failure;
- proxy flush, idle/total timeouts, cancellation propagation;
- bounded retries that do not duplicate partial output;
- usage/token accounting and tool/structured output if supported.

Client-observed complete-stream success is the acceptance SLI.

## 8. Workstream D — Performance and capacity

Record p50/p95/p99 TTFT, TPOT/ITL and E2E, output tokens/s, queue time, active/waiting requests, KV occupancy, preemptions, per-rank GPU memory, power and GPU-hours/valid completion.

Define approved SLO thresholds **before** testing; do not invent universal latency thresholds. Benchmark warm/cold starts and short/long prompts under bounded concurrency. Quality/JSON correctness must not regress when dtype, quantization or KV precision changes.

## 9. Workstream E — Security and privacy

Verify:
- TLS and authenticated gateway;
- per-tenant model/route authorization;
- token, request, concurrency and GPU cost quotas;
- no public backend, metrics/debug or distributed control ports;
- Kubernetes least-privilege RBAC and scoped ServiceAccounts;
- CNI-enforced NetworkPolicies, DNS and approved egress;
- immutable model/image provenance and safe remote-code policy;
- no prompts, outputs, PHI or bearer tokens in telemetry;
- documented key rotation, audit access and retention.

A passing health probe does not prove security isolation.

## 10. Workstream F — Observability and SLOs

Validate fresh Prometheus metrics, bounded cardinality, correct histogram units, DCGM GPU telemetry where supported, structured sanitized logs, trace propagation, complete SSE SLI, TTFT/TPOT definitions, 429/5xx classification, multi-window burn alerts and missing-data alerts.

Example **synthetic** success SLO:

```text
Target = 99.9%; eligible = 100,000; failures = 200
Allowed failures = 100; observed success = 99.8%; burn = 2×
```

Alert on incomplete SSE even when HTTP status is 200.

## 11. Workstream G — Reliability and recovery

Run tabletop and, only with separate approval, bounded staging exercises for:
- model load failure and startup CUDA OOM;
- long-prompt prefill OOM and decode KV exhaustion;
- rank/NCCL failure if distributed;
- gateway SSE buffering and timeout;
- client cancellation and retry amplification;
- metrics exporter loss and stale SLOs;
- bad model/runtime rollout and GitOps rollback;
- GPU hardware/node failure and capacity fallback.

Measure time to detect, mitigate, roll back and validate recovery. Do not claim RTO/RPO without measured evidence and scope.

## 12. Read-only preflight

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc,ingress -A -o wide
kubectl get networkpolicy -A
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Never dump Secrets, environment credentials, raw request bodies or customer model output. Verify all placeholders and context before execution.

## 13. Production go/no-go decision

| Decision | Meaning |
|---|---|
| **GO** | mandatory gates passed, weighted evidence reviewed, rollback/on-call/cost approved |
| **CONDITIONAL NO-GO** | noncritical findings have explicit owner/date but mandatory gates not waived |
| **NO-GO** | any mandatory security, privacy, recovery, SLO or cost requirement fails |
| **RETEST** | evidence stale, missing or invalid after model/runtime change |

The security reviewer, platform/SRE owner and service owner should independently sign off. A conditional label is **not** permission to bypass mandatory controls.

## 14. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | model artifact checksum mismatch | provenance |
| F02 | unapproved remote code | loader policy |
| F03 | insufficient GPU capacity | scheduling |
| F04 | startup CUDA OOM | per-rank logs |
| F05 | prefill OOM | long prompt |
| F06 | decode KV exhaustion | preemption |
| F07 | context over-admission | token budget |
| F08 | TTFT p99 regression | queue/prefill |
| F09 | TPOT p99 regression | decode/GPU |
| F10 | HTTP 200 partial SSE | terminal validator |
| F11 | proxy buffering | chunk timing |
| F12 | cancellation leak | active requests |
| F13 | retry amplification | GPU cost |
| F14 | tenant auth bypass | gateway test |
| F15 | cross-tenant access | authorization |
| F16 | secret leakage | privacy audit |
| F17 | public coordinator port | network scan |
| F18 | NetworkPolicy blocks DNS | egress |
| F19 | NCCL rank failure | rank health |
| F20 | metric schema drift | exporter inventory |
| F21 | stale SLO telemetry | scrape freshness |
| F22 | alert delivery failure | alert test |
| F23 | rollback leaves orphan workers | GPU inventory |
| F24 | GPU-hours per valid completion exceed ceiling | FinOps |

## 15. Sixteen production runbooks

1. Freeze runtime/model/architecture inventory.
2. Validate image/model provenance and licenses.
3. Verify GPU scheduling and readiness.
4. Validate per-rank VRAM and KV capacity.
5. Benchmark TTFT/TPOT/complete SSE.
6. Validate API schema and model alias.
7. Diagnose streaming cancellation/retries.
8. Audit TLS/authn/authz and tenant isolation.
9. Audit NetworkPolicy, secrets and distributed ports.
10. Verify Prometheus/OTel/DCGM freshness.
11. Validate SLO burn and alert delivery.
12. Recover startup/prefill/decode OOM.
13. Recover rank/NCCL and GPU node failures.
14. Execute gateway/runtime GitOps rollback.
15. Verify GPU FinOps and capacity headroom.
16. Conduct final independent go/no-go review.

## 16. Eight acceptance game days

1. Startup model revision OOM.
2. KV exhaustion during long-context decode.
3. HTTP 200 partial SSE after backend crash.
4. Gateway timeout under slow streaming.
5. Authentication bypass or cross-tenant access (tabletop only without security authorization).
6. NCCL rank failure and GPU node loss.
7. Telemetry pipeline outage during SLO burn.
8. Bad rollout requiring rollback and customer-path revalidation.

Default tabletop. Live injection only with separately approved staging.

## 17. Twenty mandatory acceptance gates

| Gate | Required evidence |
|---|---|
| G01 | Pinned runtime/model/image provenance |
| G02 | Approved GPU topology and scheduling |
| G03 | Model readiness and cold-start |
| G04 | Per-GPU memory/KV budget |
| G05 | Context and concurrency admission |
| G06 | TTFT/TPOT/E2E thresholds |
| G07 | Complete SSE contract |
| G08 | Gateway timeout/cancellation/retry |
| G09 | Authentication |
| G10 | Tenant authorization |
| G11 | Network/workload isolation |
| G12 | Secrets and artifact security |
| G13 | Privacy-safe logging/tracing |
| G14 | Fresh metrics and GPU telemetry |
| G15 | SLO alerts and missing-data detection |
| G16 | OOM/NCCL recovery |
| G17 | Proven GitOps rollback |
| G18 | GPU FinOps ceiling |
| G19 | Runbook/on-call ownership |
| G20 | Independent technical/security sign-off |

All gates start **Pending**; none are auto-passed by writing or publishing this chapter.

## 18. Evidence package

```text
capstone/
  00-authorization-and-scope.md
  01-pinned-release-and-model-provenance.md
  02-gpu-topology-memory-and-kv.md
  03-api-and-sse-contract.md
  04-performance-and-capacity.md
  05-security-privacy-and-tenancy.md
  06-observability-and-slo.md
  07-game-days-and-recovery.md
  08-rollback-and-finops.md
  09-risks-owners-and-go-no-go.md
```

Capture UTC, environment, artifact link, result, reviewer and expiration for every gate. Revalidate when model, driver, runtime, gateway or security policy changes.

See [companion lab](../labs/025-vllm-production-acceptance-project-check.md).

**Part 04 written curriculum complete (10/10). Live execution and independent acceptance pending.**

**Next: Chapter 026 — Triton Inference Server Architecture, beginning Part 05 — NVIDIA Triton and TensorRT-LLM.**
