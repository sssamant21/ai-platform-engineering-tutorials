# Chapter 006 — Hands-On Lab: Transformer Inference Request Lifecycle

**Part 02 | Chapter 006 | Published plan; not executed**

## Goal

Trace a synthetic LLM inference request from gateway admission through tokenization, prefill, KV cache, decode and stream completion. Explain TTFT/TPOT/ITL and classify performance failures.

## Guardrails

**[SAFE-READ]** approved Kubernetes, gateway and runtime observations. **[LAB-MUTATE]** bounded synthetic traffic in authorized staging only. **[CHANGE-CONTROL]** production rollouts, GPU provisioning, model changes and fault injection. Never capture PHI, secrets or real customer prompts.

## Lab 0 — Preflight

Record approved environment, context, namespace, model/runtime version, tokenizer revision, GPU SKU, gateway endpoint, synthetic token budget, UTC window, owner, cost ceiling, abort conditions and evidence destination.

```bash
kubectl config current-context
kubectl get pods,deploy,svc,endpointslices -A -o wide
kubectl get nodes -o wide
kubectl get resourcequota,hpa,pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

Stop if context or authorization cannot be verified.

## Lab 1 — Request trace [DESIGN / SAFE-READ]

Draw client → gateway → serving runtime → tokenizer → prefill → KV cache → decode → SSE completion. Mark data plane vs control plane and telemetry collection points. Record gateway request ID only in secured evidence.

## Lab 2 — Tokenization and context worksheet [DESIGN]

```text
Model and tokenizer version:
Chat template:
Synthetic prompt:
Measured input tokens (if approved):
Requested max output tokens:
Model/runtime context limit:
Expected admission behavior:
```

Do not assume words equal tokens. If tokenizer is unavailable, mark the measurement blocked.

## Lab 3 — Latency math [DESIGN]

Use synthetic values: gateway 80 ms, queue 120 ms, tokenization 30 ms, prefill 370 ms, decode remainder 1,400 ms. Calculate E2E = 2,000 ms and simplified TTFT = 600 ms. For 71 output tokens, the approximate post-first-token TPOT is `1,400 / 70 = 20 ms/token`. State simplifying assumptions.

## Lab 4 — Runtime and GPU observations [SAFE-READ]

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Use approved dashboards to compare queue time, input/output token throughput, KV-cache pressure, GPU VRAM, TTFT, TPOT and streaming completion. Metric availability and names are version-specific. Sanitize logs.

## Lab 5 — Six game days

| Game day | Default mode |
|---|---|
| Long-prompt TTFT | Tabletop |
| Output-heavy decode | Tabletop |
| KV cache saturation | Tabletop |
| Tokenizer mismatch | Tabletop |
| Truncated SSE | Tabletop |
| Missing metrics | Tabletop |

If a staging experiment is separately approved, record traffic limit, abort threshold, cost and cleanup. Otherwise do not generate load.

## Sixteen-scenario checklist

F01 wrong model; F02 tokenizer; F03 context; F04 cold start; F05 queue; F06 prefill; F07 decode; F08 KV cache; F09 OOM; F10 throttling; F11 fairness; F12 retries; F13 SSE; F14 504; F15 false success; F16 missing telemetry.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Request lifecycle diagram | Pending |
| G02 | Tokenizer/template contract | Pending |
| G03 | Prefill/decode distinction | Pending |
| G04 | KV-cache sizing assumptions | Pending |
| G05 | TTFT/TPOT/ITL units | Pending |
| G06 | Valid SSE completion definition | Pending |
| G07 | Cohort segmentation | Pending |
| G08 | Retry/concurrency safeguards | Pending |
| G09 | Authorized read-only inspection | Pending |
| G10 | Failure-domain classification | Pending |
| G11 | Runbook and rollback owner | Pending |
| G12 | Independent reviewer | Pending |

## Evidence template

```text
UTC / approval / context:
Model / tokenizer / runtime / image:
Synthetic request and token counts:
Gateway / queue / prefill / decode:
TTFT / TPOT / ITL / E2E:
KV cache and GPU metrics:
SSE completion and error classification:
Game-day mode (tabletop/executed):
Findings / mitigations / owner:
Reviewer and acceptance status:
```

## Cleanup

The default read-only/tabletop lab creates no resources. Stop any separately approved synthetic traffic, confirm no cloud resources were left running, and retain sanitized evidence. Gates stay pending until independently reviewed.

**Next: Chapter 007 — Tokenization, Context Windows and Generation Parameters.**
