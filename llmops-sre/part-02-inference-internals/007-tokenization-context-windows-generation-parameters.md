# Chapter 007 — Tokenization, Context Windows and Generation Parameters

**Part 02 — LLM Architecture and Inference Internals | Chapter 007 of 100**

**Status:** Published learning material. Technical/security review, real lab execution and operational acceptance remain pending.

## Learning objectives

Explain token IDs, model-specific chat templates, context limits, truncation, sampling and stop conditions; design reliable inference request validation; diagnose token-budget and latency failures; and build safe, reproducible synthetic tokenization tests.

## 1. Why tokens matter to SRE

LLM inference is scheduled and billed in tokens, not words or characters. A tokenizer converts text into token IDs using a model-specific vocabulary and normalization rules. Token counts depend on language, punctuation, whitespace, Unicode, chat formatting, tools and special tokens. The same string may have different counts for different tokenizers or revisions.

```text
client messages + tools + generation settings
   → gateway validation / tenant token budget
   → model-specific chat template
   → tokenizer / token IDs
   → context-window admission
   → prefill (prompt tokens)
   → decode (generated tokens)
   → detokenization / SSE completion
   → usage accounting and telemetry
```

A tokenizer mismatch can cause invalid special tokens, inaccurate admission counts, output differences and degraded quality even when the serving Pod is healthy.

## 2. Chat templates and special tokens

A chat API typically transforms structured roles and content into a model-specific sequence with role delimiters, system instructions, optional tool schemas and generation markers. These additions consume context and may not be visible in a simple character count.

**Operational contract:** pin the model revision, tokenizer revision and chat template together. Record compatibility in the release manifest. Avoid mixing an independently upgraded tokenizer with a cached older model.

## 3. Context-window accounting

For a simple decoder-only model without special sliding-window handling, a useful admission approximation is:

```text
formatted_prompt_tokens + reserved_generation_tokens
    <= effective_runtime_context_limit
```

The effective limit can be lower than a model's advertised maximum because of runtime configuration, KV-cache capacity, GPU memory, batching or application policy. Some models support sliding-window attention, truncation or context extension with different semantics. Never assume automatic truncation is safe: it may remove critical system instructions or citations.

A reliable gateway should validate the **formatted** prompt with the **actual** tokenizer and explicitly decide how to handle over-limit requests. Return a clear, documented client error rather than silently discarding content unless the product has approved a truncation policy.

### Example budget (illustrative)

| Item | Tokens |
|---|---:|
| Effective context limit | 8,192 |
| Formatted messages and tool schema | 6,900 |
| Requested maximum generation | 1,024 |
| Total reservation | 7,924 |
| Remaining headroom | 268 |

This is a planning example; real token counts require the pinned tokenizer and template. Generation may stop before its cap.

## 4. Generation parameters and production effects

| Parameter | Function | Operational consideration |
|---|---|---|
| max output tokens | upper bound on generated tokens | deadline, cost, KV cache |
| temperature | modifies sampling distribution | reproducibility, quality |
| top_p | nucleus sampling threshold | output diversity |
| top_k | limits candidate tokens, if supported | runtime-specific |
| repetition/frequency/presence penalties | discourage certain repetition patterns | model/API-specific semantics |
| stop sequences / stop token IDs | terminate generation when matched | incomplete output risk |
| seed | requests repeatable randomness, if supported | not guaranteed across hardware/runtime |
| beam search | alternative decoding strategy, if supported | potentially higher compute |
| logprobs | expose token probabilities, if supported | overhead, data sensitivity |
| stream | deliver incremental output | SSE, timeouts, partial completion |

**Do not assume parameter compatibility** across OpenAI-compatible APIs, vLLM versions, Triton backends or managed providers. Validate unsupported parameter behavior in staging. Some values are constrained to particular ranges or are mutually incompatible; consult the deployed engine's API schema.

### Temperature and determinism

Temperature near zero often favors more deterministic choices, but exact reproducibility is not guaranteed because of floating-point kernels, distributed execution, batching, model revisions and sampling implementations. A seed is not a universal reproducibility guarantee. For evaluations, record model, tokenizer, template, parameters, runtime, hardware and dataset revision.

## 5. Token limits, latency and cost

Prompt tokens predominantly affect prefill work and TTFT; generated tokens affect decode duration, stream length and KV-cache residency. Larger max-output settings increase **reserved capacity and worst-case cost**, even when actual output is short.

```text
estimated_total_tokens = input_tokens + output_tokens
estimated_variable_cost = input_tokens × input_rate
                        + output_tokens × output_rate
```

Rates are provider/contract-specific and may include caching, GPU-hours, batch discounts, network and storage. For self-hosted GPU serving, cost per successful output token should include idle capacity, failures and retries.

### Guardrails

- Bound prompt bytes, formatted tokens, output tokens, request duration and concurrency.
- Enforce per-tenant quotas and reject or queue excess requests predictably.
- Do not silently expand context to solve a single failure; model/VRAM limits apply.
- Treat user-provided stop strings and tool schemas as untrusted input.
- Keep prompts and generated text out of Prometheus labels and general-purpose logs.
- Validate Unicode, empty inputs, tool messages and multilingual inputs.

## 6. Tokenization and context failure tree

```text
Request rejected or response unexpected?
 ├─ 400/422 context overflow → actual formatted tokens vs limit
 ├─ model-specific special-token error → tokenizer/template revision
 ├─ output stops too early → stop sequence, EOS, max token cap
 ├─ response ignores earlier instructions → truncation/window policy
 ├─ high TTFT → long prompt, tokenization, queue, prefill
 ├─ high E2E/TPOT → long generation, decode pressure
 ├─ GPU OOM → KV cache, context, concurrency, model weights
 ├─ quality drift after release → model/template/parameter diff
 ├─ inconsistent output with same seed → runtime/hardware/sampling
 └─ cost spike → token budgets, retries, output caps, caching
```

## 7. Safe inspection and instrumentation

**[SAFE-READ]** Confirm the authorized context and inspect the serving revision:

```bash
kubectl config current-context
kubectl get deployments,pods -A -o wide
kubectl get resourcequota -A
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

For an approved deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Avoid logging raw request bodies. Instrument bounded histograms for input/output token counts, TTFT and E2E latency, plus counters for context rejects, truncation, stop reasons and quota enforcement. These are **proposed signal categories**, not guaranteed vendor metric names.

## 8. Eighteen production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | wrong tokenizer revision | release manifest |
| F02 | chat template mismatch | formatted-token test |
| F03 | special token collision | tokenizer IDs |
| F04 | prompt exceeds context | token count vs effective limit |
| F05 | output reservation overflow | input + max output |
| F06 | silent truncation | gateway policy, prompt hash |
| F07 | system instructions truncated | truncation order |
| F08 | tool schema inflates context | formatted tool-token count |
| F09 | Unicode token explosion | language cohort |
| F10 | premature stop sequence | finish reason and stop config |
| F11 | max-token truncation | finish reason |
| F12 | invalid sampling parameter | API validation |
| F13 | nondeterministic eval | version/seed/hardware |
| F14 | TTFT regression | tokenization/prefill |
| F15 | long decode / high E2E | output length, TPOT |
| F16 | KV-cache OOM | VRAM and concurrency |
| F17 | tenant token-budget bypass | gateway accounting |
| F18 | token-based cost spike | input/output counts and retries |

## 9. Twelve operational runbooks

1. Verify model/tokenizer/template provenance.
2. Reproduce formatted token counts with synthetic inputs.
3. Investigate context admission failures.
4. Audit truncation and preservation of system instructions.
5. Validate generation parameter compatibility.
6. Diagnose early EOS/stop sequence termination.
7. Diagnose max-output truncation.
8. Investigate long-prompt TTFT regressions.
9. Investigate output-heavy decode latency.
10. Triage KV cache/VRAM and concurrency.
11. Audit token quota and cost attribution.
12. Roll back incompatible tokenizer/template releases through the authoritative owner.

Each runbook requires ownership, safe evidence, mitigation, change control, rollback and customer-path verification.

## 10. Six game days

1. Context overflow from a large tool schema (tabletop).
2. Tokenizer mismatch after a canary (tabletop).
3. Silent truncation removes a system instruction (tabletop).
4. Stop sequence terminates valid output (tabletop).
5. Output token cap drives latency/cost (staging synthetic only if approved).
6. Token quota bypass triggers spend alert (tabletop).

## 11. Twelve acceptance gates

G01 tokenizer/model pinning; G02 chat-template overhead; G03 context admission math; G04 truncation policy; G05 sampling compatibility; G06 stop/finish semantics; G07 TTFT vs output-length behavior; G08 KV-cache and GPU impact; G09 token accounting; G10 privacy/security controls; G11 incident runbooks; G12 reviewer signoff.

All gates remain pending until evidence is collected and reviewed. See [companion lab](../labs/007-tokenization-context-windows-generation-parameters-check.md).

**Next: Chapter 008 — Prefill, Decode, KV Cache and Continuous Batching.**
