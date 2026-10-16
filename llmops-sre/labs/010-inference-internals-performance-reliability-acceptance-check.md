# Chapter 010 — Hands-On Lab: Inference Internals Performance and Reliability Acceptance

**Part 02 final capstone | Chapter 010 | Published lab plan; not executed**

## Goal

Validate Chapters 006–009 together: tokenizer/context admission, prefill/decode/KV cache, batching, streaming completion, gateway backpressure, error budget, incident command and recovery. Produce a reviewer-ready 100-point acceptance worksheet.

## Safety and stop conditions

**[SAFE-READ]** approved Kubernetes, release manifests and sanitized metrics. **[LAB-MUTATE]** explicitly approved staging synthetic inference with request/token/time/cost caps. **[CHANGE-CONTROL]** production changes, GPU provisioning, failure injection, gateway/routing/timeout updates. No PHI, real customer prompts, credentials or private model outputs.

Abort on wrong context, missing permission, unexpected tenant impact, exposed data, rising error rate beyond the approved threshold, GPU memory danger or spending ceiling. If not approved, perform tabletop only.

## Lab 0 — Preflight

Record UTC window, context/namespace, service/model owner, runtime version, model/tokenizer/template/image digest, gateway API contract, GPU SKU, synthetic data revision, budget, incident commander, reviewer and rollback owner.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get deployments,pods,svc,endpointslices -A -o wide
kubectl get resourcequota,hpa,pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

## Lab 1 — Architecture and RACI [DESIGN]

Draw client → gateway → ingress → Service → runtime → prefill → KV cache → decode → SSE. Annotate security boundaries, telemetry, GitOps reconciliation and named owners. Identify rollback authority.

## Lab 2 — Token and context math [DESIGN]

For synthetic context limit 8,192, formatted prompt 6,900 and output cap 1,024:

```text
reserved = 7,924; headroom = 268 tokens
```

State why actual counts require pinned tokenizer/chat template. Document rejection/truncation policy and supported generation parameters.

## Lab 3 — KV cache and latency math [DESIGN]

For 32 layers, 8 KV heads, dimension 128, 2 bytes/element:

```text
KV/token = 131,072 bytes
8,192 resident tokens ≈ 1 GiB logical cache
```

For synthetic gateway 80 ms, queue 120 ms, tokenization 30 ms, prefill 370 ms and remaining decode/stream 1,400 ms:

```text
E2E = 2,000 ms; simplified TTFT = 600 ms
71 output tokens → approximate TPOT = 1,400/70 = 20 ms/token
```

List exclusions and overlapping phases; these are not live measurements.

## Lab 4 — SSE parser and customer success [DESIGN]

Create fixtures for: split event across chunks; multiple events per chunk; multiline data; UTF-8 boundary; heartbeat before token; HTTP 200 with missing terminal marker; valid finish and close; error after headers. Classify valid completion using the deployed API contract, not status alone.

## Lab 5 — Read-only evidence [SAFE-READ]

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Use approved dashboards to collect queue, TTFT, TPOT, ITL, E2E, prefill/decode, KV cache, GPU memory, SSE completion, quota/retries and telemetry freshness. Redact logs and record gaps rather than fabricating metrics.

## Lab 6 — Workload matrix [DESIGN; optional approved staging]

| Cohort | Synthetic target input tokens | Output cap | Question |
|---|---:|---:|---|
| A | 128 | 64 | interactive TTFT |
| B | 1,024 | 256 | balanced latency |
| C | 4,096 | 128 | prefill pressure |
| D | 512 | 1,024 | decode/streaming |
| E | near context limit | 64 | admission |
| F | 256 | 128 | slow-reader backpressure |

Token counts must be verified with the pinned tokenizer. No traffic without approval and a bounded workload specification.

## Lab 7 — SLO arithmetic [DESIGN]

Hypothetical SLO 99.9%, 20,000 eligible requests, 100 failures:

```text
allowed failures = 20
observed failure rate = 100/20,000 = 0.5%
observed success = 99.5%
burn rate = 0.5% / 0.1% = 5×
window budget consumed = 100/20 = 500%
```

Explicitly classify partial HTTP 200 streams as failures per approved SLI.

## Lab 8 — Ten tabletop game days

01 tokenizer mismatch; 02 tool-schema overflow; 03 prefill starvation; 04 KV preemption; 05 decode slowdown; 06 buffered SSE; 07 incomplete 200; 08 cancellation leak; 09 retry amplification; 10 missing telemetry.

For each: symptom, impact, evidence, fault hypothesis, owner, authorized mitigation, rollback, recovery signal and follow-up. Mark **tabletop**, **executed**, **blocked** or **pending** truthfully.

## Twenty-four failure checklist

F01 route; F02 tokenizer; F03 template; F04 context; F05 truncation; F06 parameters; F07 stop; F08 output cap; F09 prefill; F10 queue; F11 decode; F12 KV; F13 OOM; F14 throttling; F15 fairness; F16 buffering; F17 SSE parser; F18 partial 200; F19 idle timeout; F20 cancellation; F21 retries; F22 rollout; F23 privacy; F24 stale telemetry.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope approval | Pending |
| G02 | Correct context | Pending |
| G03 | Sanitized evidence | Pending |
| G04 | Architecture and RACI | Pending |
| G05 | Model/tokenizer/template provenance | Pending |
| G06 | Context math | Pending |
| G07 | Parameter compatibility | Pending |
| G08 | Prefill/decode distinction | Pending |
| G09 | KV sizing assumptions | Pending |
| G10 | Batching fairness | Pending |
| G11 | TTFT/TPOT/ITL | Pending |
| G12 | Complete SSE contract | Pending |
| G13 | Backpressure/cancellation | Pending |
| G14 | Quota/retry policy | Pending |
| G15 | SLO/burn math | Pending |
| G16 | Telemetry freshness | Pending |
| G17 | GPU and cost envelope | Pending |
| G18 | Incident command | Pending |
| G19 | Rollback and recovery | Pending |
| G20 | Independent signoff | Pending |

## 100-point assessment

| Category | Maximum | Awarded |
|---|---:|---|
| Safety and privacy | 10 | Pending |
| Architecture and ownership | 10 | Pending |
| Tokenizer/context/parameters | 15 | Pending |
| Prefill/decode/KV | 15 | Pending |
| Streaming/gateway/cancellation | 15 | Pending |
| SLO and performance | 15 | Pending |
| Incident/rollback/recovery | 10 | Pending |
| FinOps/evidence/review | 10 | Pending |
| **Total** | **100** | **Not assessed** |

Suggested passing threshold: ≥85 with no critical safety failure, correct stream completion and context accounting, authorized rollback plan and reviewer signoff.

## Evidence record

```text
UTC, account, context, authorization:
Owner, incident commander, reviewer:
Model/tokenizer/template/runtime/GPU:
Synthetic fixtures and version:
Context and KV math:
Queue/prefill/decode/TTFT/TPOT/ITL:
Complete SSE, quota and retries:
GPU/FinOps and telemetry freshness:
Game days (tabletop/executed):
SLO/burn calculation:
Findings, rollback, reviewer and score:
```

## Cleanup and closeout

Read-only/tabletop work creates no resources. For separately authorized staging, stop synthetic requests, validate cancellation, reconcile GPU costs and remove only owned resources. Unexecuted gates stay pending; written publication does not equal acceptance.

**Next: Chapter 011 — GPU Architecture, SMs, HBM and Memory Bandwidth.**
