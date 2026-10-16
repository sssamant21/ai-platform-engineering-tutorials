# Chapter 007 — Hands-On Lab: Tokenization, Context Windows and Generation Parameters

**Part 02 | Chapter 007 | Published lab plan; execution pending**

## Goal

Create a synthetic tokenizer/context test matrix, verify output controls, quantify token budgets and classify failures without using real customer data.

## Safety and prerequisites

**[SAFE-READ]** approved model/tokenizer metadata and Kubernetes inventory. **[LAB-MUTATE]** staging-only synthetic inference with explicit authorization and request/token ceiling. **[CHANGE-CONTROL]** production model/parameter changes. No PHI, private prompts, credentials or proprietary model artifacts in evidence.

Record UTC, cluster/context, model revision, tokenizer revision, chat template, runtime version, effective context limit, owner, cost ceiling and reviewer.

## Lab 1 — Version and environment baseline

```bash
kubectl config current-context
kubectl get deployments,pods -A -o wide
kubectl get resourcequota -A
```

Use approved release manifests to record image digest, model, tokenizer and template revisions. Do not assume the tokenizer matches the model without verification.

## Lab 2 — Context-budget worksheet

For a synthetic 8,192-token effective limit, 6,900 formatted prompt tokens and 1,024 reserved output tokens:

```text
Total = 6,900 + 1,024 = 7,924
Headroom = 8,192 - 7,924 = 268
```

Repeat with 7,500 prompt tokens and 1,024 reserved output tokens. The resulting 8,524 total exceeds the example limit by 332 tokens; define the expected rejection or approved truncation policy.

## Lab 3 — Synthetic tokenization matrix

| Test | Input class | Expected check |
|---|---|---|
| T01 | plain English | tokenizer count |
| T02 | punctuation/whitespace | count change |
| T03 | Unicode/multilingual | correct encoding/count |
| T04 | long system instruction | truncation preservation |
| T05 | tool schema | formatting overhead |
| T06 | empty/minimal message | API validation |
| T07 | context boundary | admission rejection |
| T08 | max-output reservation | combined limit |

Actual token counts require the pinned tokenizer. Mark unexecuted cases **pending**.

## Lab 4 — Generation parameter matrix

Design synthetic requests varying one parameter at a time: max output tokens, temperature, top_p, stop sequence and seed **only if supported**. Record supported ranges, validation errors, finish reasons, output tokens, TTFT and E2E. Do not claim deterministic results from seed alone.

## Lab 5 — Observability and failure triage

Review approved dashboards for input/output token count distributions, context rejection, finish reasons, TTFT, TPOT, KV cache and token budget. Do not put raw prompts in metric labels.

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Sanitize logs. If metrics are absent, document a gap rather than fabricating data.

## Lab 6 — Six game days

01 tool schema context overflow; 02 tokenizer mismatch; 03 unsafe truncation; 04 premature stop; 05 output cap and cost; 06 quota bypass. Default **tabletop**. Staging execution requires separate approval, bounded traffic and cleanup.

## Eighteen-scenario checklist

F01 tokenizer; F02 template; F03 special tokens; F04 prompt overflow; F05 reservation; F06 silent truncation; F07 system prompt loss; F08 tool schema; F09 Unicode; F10 stop; F11 output cap; F12 parameter invalid; F13 nondeterminism; F14 TTFT; F15 E2E; F16 OOM; F17 quota bypass; F18 cost spike.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Model/tokenizer version mapping | Pending |
| G02 | Template overhead | Pending |
| G03 | Context arithmetic | Pending |
| G04 | Truncation policy | Pending |
| G05 | Sampling compatibility | Pending |
| G06 | Finish/stop semantics | Pending |
| G07 | TTFT/output length | Pending |
| G08 | KV-cache impact | Pending |
| G09 | Token accounting | Pending |
| G10 | Privacy/security | Pending |
| G11 | Incident triage | Pending |
| G12 | Independent review | Pending |

## Evidence template and cleanup

```text
UTC / approval / model:
Tokenizer/template/runtime:
Context limit and prompt/output token counts:
Parameter matrix and finish reasons:
TTFT/TPOT/E2E, token budget:
Rejection/truncation behavior:
Security/privacy review:
Game days (tabletop/executed):
Findings, reviewer, gate status:
```

Read-only/tabletop work creates no resources. Stop any separately authorized synthetic requests and reconcile costs. All gates remain pending until verified.

**Next: Chapter 008 — Prefill, Decode, KV Cache and Continuous Batching.**
