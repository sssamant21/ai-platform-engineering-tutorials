# Chapter 005 — Hands-On Capstone: Fundamentals Acceptance and Incident Walkthrough

**Part 01 final lab | Chapter 005 | Status: Published; not executed**

## Purpose

Combine Chapters 001–004 into one SRE acceptance exercise: customer-path inference mapping, SLO/error budget, GPU/Kubernetes triage, incident command, safe rollback design and evidence review.

## Lab rules

**[SAFE-READ]** approved cluster inspection and synthetic telemetry. **[LAB-MUTATE]** separately approved staging-only synthetic load. **[CHANGE-CONTROL]** production rollback, fault injection, GPU provisioning or gateway changes. No PHI, secrets, real prompts or private model outputs.

## Lab 0 — Entry and approval

Record UTC window, service owner, approved cluster/context, namespace, synthetic model, incident commander, reviewer, budget, time limit, abort conditions and evidence destination.

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get pods,deploy,sts,ds -A -o wide
kubectl get svc,endpointslices -A
kubectl get resourcequota,hpa,pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

Stop if the context is unverified or any command is unauthorized.

## Lab 1 — Architecture and responsibility map

Produce a diagram of synthetic client → gateway → Service → runtime → GPU/model store, plus separate GitOps/TrueFoundry/control-plane path. Annotate trust boundaries, dependencies, RACI, pager and rollback owner.

## Lab 2 — SLI and error budget

Define eligible requests, good complete SSE, 429/cancel/retry rules, TTFT/TPOT and telemetry freshness. For illustrative 20,000 eligible attempts at 99.9%, compute 20 allowed failures. For 100 failures, error budget consumed is 500%, and availability is 99.5%. Explain why the example is not a real SLO.

## Lab 3 — Triage simulated incident

Given the scenario of rising TTFT, incomplete SSE, a Pending GPU Pod and queue saturation, create a hypothesis table: signal, fault domain, read-only validation, owner, mitigation and recovery proof. Use approved read-only cluster and dashboard evidence where available. Never inject failures without authorization.

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE get pods -o wide
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Sanitize all logs.

## Lab 4 — Incident command

Write a UTC timeline, severity and customer impact, incident roles, 15-minute update, escalation and release freeze decision. Distinguish facts, hypotheses and simulated evidence.

## Lab 5 — Recovery plan

Describe safe rollback via the authoritative controller, compatibility checks, customer-path synthetic validation, SSE completion, TTFT/TPOT and GPU capacity. Record rollback abort conditions, change approval and communications. Do not mutate production.

## Lab 6 — Eight game days

| Game day | Default mode |
|---|---|
| Truncated SSE with HTTP 200 | Tabletop |
| 429 quota versus saturation | Tabletop |
| GPU Pod Pending | Tabletop |
| CUDA runtime mismatch | Tabletop |
| Model artifact unavailable | Tabletop |
| High TTFT from prefill | Tabletop |
| Metrics missing | Tabletop |
| GitOps rollback conflict | Tabletop |

Record executed/tabletop/blocked and expected versus observed evidence separately.

## Twenty-scenario checklist

F01 context; F02 TLS; F03 auth; F04 429; F05 endpoints; F06 Pending; F07 plugin; F08 CUDA; F09 OOM; F10 artifact; F11 tokenizer; F12 cold start; F13 queue; F14 TPOT; F15 SSE; F16 metrics; F17 duplicate counters; F18 low volume; F19 spend; F20 controller conflict.

## Assessment rubric

| Area | Max points | Awarded |
|---|---:|---|
| Authorization and safety | 10 | Pending |
| Architecture and RACI | 15 | Pending |
| SLI/SLO/error budget | 15 | Pending |
| Read-only investigation | 15 | Pending |
| Incident leadership | 10 | Pending |
| Containment and rollback | 15 | Pending |
| Recovery verification | 10 | Pending |
| RCA/COA and follow-up | 10 | Pending |
| **Total** | **100** | **Not assessed** |

Suggested acceptance: ≥85/100 with no critical safety failures and reviewer signoff.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Authorized scope | Pending |
| G02 | Verified context | Pending |
| G03 | Redacted evidence | Pending |
| G04 | Inference request path | Pending |
| G05 | RACI | Pending |
| G06 | Trust boundaries | Pending |
| G07 | Eligible SLI contract | Pending |
| G08 | Valid SSE completion | Pending |
| G09 | TTFT/TPOT units | Pending |
| G10 | Error budget calculation | Pending |
| G11 | Telemetry freshness | Pending |
| G12 | GPU scheduling triage | Pending |
| G13 | CUDA compatibility | Pending |
| G14 | Model/version provenance | Pending |
| G15 | Incident declaration | Pending |
| G16 | Mitigation authorization | Pending |
| G17 | Rollback plan | Pending |
| G18 | Customer-path recovery | Pending |
| G19 | RCA/COA | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
Lab/UTC/context/approval:
Service owner/incident commander/reviewer:
Synthetic request path:
Model/runtime/image digest/GPU:
SLI eligibility/TTFT/TPOT/SSE:
Error-budget calculation:
Timeline and fault hypotheses:
Read-only evidence and redactions:
Mitigation and rollback ownership:
Recovery verification:
Game days (tabletop vs executed):
Assessment points, blockers and signoff:
```

## Cleanup and next steps

The default read-only/tabletop exercise creates no resources. If separate staging load is authorized, stop traffic, verify cloud cost and clean only explicitly owned resources. Record incomplete gates as pending/blocked, not passed.

**Next: Chapter 006 — Transformer Inference Request Lifecycle.**
