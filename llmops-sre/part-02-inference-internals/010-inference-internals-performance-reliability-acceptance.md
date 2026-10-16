# Chapter 010 — Inference Internals Performance Lab and Reliability Acceptance Project

**Part 02 — LLM Architecture and Inference Internals | Chapter 010 of 100 | Part 02 capstone**

> **Publication:** Written tutorial. **Not yet executed, independently reviewed, security approved or operationally accepted.** All evidence and incident data below are synthetic examples.

## 1. Outcomes and prerequisites

Integrate Chapters 006–009 into one production-grade acceptance exercise: trace the request lifecycle, validate tokenizer/context contracts, quantify prefill/decode and KV-cache behavior, measure complete SSE streams, triage gateway/backpressure failures, calculate SLO burn, plan mitigation and produce a reproducible reliability report.

Prerequisites: completed reading of Chapters 006–009; an approved nonproduction inference endpoint (optional for tabletop); read-only Kubernetes and observability permissions; model/tokenizer/runtime manifests; synthetic data; incident commander and independent reviewer.

**No production change is authorized by this chapter.**

## 2. System under test and trust boundaries

```text
synthetic client / completion validator
   │ HTTPS, auth, tenant and quota
   ▼
API gateway → ingress / service mesh → Kubernetes Service / EndpointSlice
   │                 └─ timeout, buffering, retry and cancellation policy
   ▼
runtime admission / tokenizer / chat template / scheduler
   │ queue wait
   ├─ prefill: prompt processing, initial KV
   ├─ decode: iterative generation, KV reuse, continuous batching
   └─ SSE: token events, finish reason, terminal marker, clean close
   ▼
client-side complete-stream SLI

GPU: HBM/VRAM, compute, bandwidth, cache, CUDA, device plugin
Control: GitOps / platform operator / immutable model registry
Telemetry: gateway, runtime, DCGM, OTel, customer-path synthetic
```

Create a RACI identifying the service owner, API gateway owner, inference runtime owner, GPU platform owner, security reviewer, FinOps reviewer, incident commander and rollback authority. The controller owning deployment reconciliation must approve and execute changes.

## 3. Mandatory safety classification

- **[SAFE-READ]** Approved cluster metadata, release manifests, dashboards, redacted logs, synthetic traces and evidence worksheets.
- **[LAB-MUTATE]** Explicitly approved staging synthetic inference or performance load with a request/token limit, time window, cost ceiling, owner, rollback and stop criteria.
- **[CHANGE-CONTROL]** Production traffic shifting, runtime tuning, model rollout, autoscaling/GPU provisioning, fault injection, quota or gateway changes.

Stop on unverified context, missing authorization, real PHI, leaked credentials, customer prompt capture, uncontrolled GPU spending or unexpected external effects.

## 4. Acceptance workload specification

Use *synthetic*, fixed and versioned prompt templates with token counts measured by the **pinned tokenizer and chat template**. Do not infer tokens from word counts.

| Cohort | Example input tokens | Example output cap | Purpose |
|---|---:|---:|---|
| A — short interactive | 128 | 64 | TTFT, short-user journey |
| B — balanced | 1,024 | 256 | representative latency |
| C — long prompt | 4,096 | 128 | prefill and context pressure |
| D — long generation | 512 | 1,024 | decode and streaming |
| E — boundary | effective limit minus reserved output | 64 | admission/context validation |
| F — slow reader | 256 | 128 | cancellation and backpressure |

These counts are **design targets**, not measured tokens. The context-boundary case must be computed from actual formatted input and the configured limit. Use one workload revision, deterministic fixtures where possible, and record supported sampling parameters.

The load envelope is **not specified here**: service owner must approve concurrency, requests, maximum generated tokens, duration and GPU budget. Start with tabletop if approval is missing.

## 5. Measurement contract

| Indicator | Measurement and acceptance intent |
|---|---|
| Eligible request | Valid, authorized request included per agreed contract |
| Good streaming request | Valid final protocol completion, expected finish semantics, no corruption |
| TTFT | client request start to first *generated output token* |
| TPOT | documented per-request post-first-token average |
| ITL | client-observed successive token gap |
| E2E | request start to valid final completion |
| Queue time | admission to runtime scheduling |
| Prefill time | prompt-processing phase, where instrumented |
| Decode rate | actual generated output tokens per second |
| KV-cache pressure | runtime block utilization, preemptions, evictions |
| GPU envelope | weights + KV + workspace + fragmentation/headroom |
| Fairness | p95/p99 and completion by bounded cohort/tenant |
| FinOps | GPU-hours, actual input/output tokens, retry waste |

Never equate HTTP 200 with success; never count an SSE heartbeat as the first token; never average per-Pod p95s; treat missing telemetry as unknown rather than zero failures. A client disconnect may require separate eligibility classification.

## 6. Worked arithmetic (synthetic)

### Context and KV cache

With effective context 8,192, formatted input 6,900 and reserved output 1,024:

```text
reserved = 6,900 + 1,024 = 7,924 tokens
headroom = 8,192 - 7,924 = 268 tokens
```

For an illustrative conventional attention architecture (32 layers, 8 KV heads, head dimension 128, BF16):

```text
KV bytes/token ≈ 2 × 32 × 8 × 128 × 2 = 131,072 bytes
8,192 resident tokens ≈ 1,073,741,824 bytes = 1 GiB logical KV
```

The estimate excludes allocator blocks, sharing, offload, sliding windows, sharding and runtime overhead; it is **not** a per-GPU capacity guarantee.

### Latency

```text
gateway 80ms + queue 120ms + tokenizer 30ms
+ prefill 370ms + remaining decode/stream 1400ms = E2E 2000ms
simplified TTFT = 600ms
if 71 output tokens: TPOT ≈ 1400ms / 70 = 20ms/token
```

These sequential assumptions are illustrative; real prefill/first-decode/network phases may overlap.

### Error budget

At a hypothetical 99.9% request-success SLO and 20,000 eligible requests, 20 failures consume the budget. If 100 eligible requests fail, observed success is 99.5%, failure rate is 0.5%, burn is 5×, and the window consumes 500% of its allowed failure count. This is **not** a live SLO measurement.

## 7. Execution plan

**Phase 0 — preflight:** verify context, model/tokenizer/template/runtime/image digest, gateway revision, API streaming contract, GPU type, budget and owners. Collect baseline only after permissions are confirmed.

**Phase 1 — correctness:** exercise chat-template and context boundary cases; validate parameter compatibility and output stop/finish semantics; verify complete SSE with client-side parser fixtures.

**Phase 2 — performance:** separate gateway, queue, tokenization, prefill, decode and stream delivery. Compare TTFT/TPOT/ITL and complete-stream rate across A–D. Inspect runtime KV occupancy and GPU memory.

**Phase 3 — contention:** design (or separately approve in staging) increasing concurrency and slow-reader tests. Compare throughput, fairness, p95/p99, cache preemptions and cancellation propagation. Apply a hard abort threshold for error spikes, GPU memory, quota and spend.

**Phase 4 — fault response:** run tabletop incident game days, establish severity and ownership, choose a safe containment action, describe controller-authorized rollback and specify sustained customer-path recovery.

**Phase 5 — acceptance:** compile sanitized evidence, unresolved gaps, security/FinOps assessment and reviewer signoff. Mark each gate passed, failed, blocked or pending based on actual evidence.

## 8. Read-only command checklist

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get deployments,pods,svc,endpointslices -A -o wide
kubectl get resourcequota,hpa,pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

After validating namespace, workload and permission:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Do not copy raw customer prompts, auth headers or secrets from logs. Kubernetes and runtime metric names are version-specific; verify actual exporter schema before writing PromQL or alerts.

## 9. Integrated troubleshooting decision tree

```text
Customer synthetic unhealthy?
 ├─ connection / 401 / 403 → DNS, TLS, identity, routing
 ├─ 429 → contractual quota vs overload policy
 ├─ 502/503/504 → endpoints, readiness, deadlines, queue
 ├─ no first token → buffering, queue, tokenizer, prefill, cold start
 ├─ high TTFT only → long prompt cohort, queue, prefill
 ├─ high TPOT/ITL → decode contention, scheduling, GPU/network
 ├─ CUDA OOM → weights, KV cache, context, concurrency
 ├─ context rejection → formatted tokens + output reservation
 ├─ 200 but incomplete SSE → proxy timeout, crash, cancellation
 ├─ slow consumer → proxy buffer and cancellation propagation
 ├─ high spend → retries, token limits, GPU idle time
 └─ no telemetry → independent synthetic and scrape freshness
```

## 10. Twenty-four failure scenarios

| ID | Failure | Evidence to capture |
|---|---|---|
| F01 | wrong model route | gateway/release manifest |
| F02 | tokenizer revision mismatch | tokenizer and model IDs |
| F03 | chat template overhead ignored | formatted token count |
| F04 | context overflow | input + reserved output |
| F05 | unsafe truncation | policy and instruction preservation |
| F06 | invalid generation parameter | API validation |
| F07 | premature stop | stop/finish reason |
| F08 | output cap exceeded/early stop | token accounting |
| F09 | long prompt TTFT | prompt cohort, prefill |
| F10 | queue saturation | queued/running requests |
| F11 | decode contention | TPOT, output throughput |
| F12 | KV cache exhaustion | cache blocks, preemption |
| F13 | CUDA OOM | runtime logs, VRAM |
| F14 | GPU throttling | DCGM clocks and power |
| F15 | unfair continuous batching | cohort tail latency |
| F16 | proxy buffers tokens | upstream/client first-token times |
| F17 | SSE framing bug | client parser fixture |
| F18 | HTTP 200 partial stream | terminal event check |
| F19 | gateway idle timeout | timeout hierarchy |
| F20 | client disconnect leaks GPU work | cancellation propagation |
| F21 | retry amplification | attempts, token waste |
| F22 | rollout terminates streams | drain and controller history |
| F23 | prompt leakage in logs | privacy review |
| F24 | stale telemetry hides outage | probe freshness |

## 11. Sixteen operational runbooks

1. Preflight environment and authority.
2. Validate customer-path synthetic and complete SSE.
3. Check gateway DNS/TLS/auth and quota.
4. Diagnose 502/503/504 and endpoints.
5. Validate tokenizer/chat template/context.
6. Investigate stop/finish and sampling parameters.
7. Isolate queue and prefill TTFT.
8. Isolate decode TPOT and ITL.
9. Inspect KV-cache pressure/preemption.
10. Triage CUDA OOM and GPU hardware.
11. Investigate continuous-batching fairness.
12. Diagnose buffering, timeout and backpressure.
13. Verify cancellation and retry safety.
14. Execute bounded staging benchmark and FinOps review.
15. Plan authorized rollback and recovery.
16. Conduct incident review, RCA/COA and signoff.

For each: accountable owner, safe checks, impact, mitigation, change approval, rollback, customer-path verification and evidence.

## 12. Ten game days

1. Wrong tokenizer after canary.
2. Context overflow from tool schema.
3. Long prompt starves short requests.
4. KV-cache preemption causes TTFT/TPOT regression.
5. Decode slows while prefill appears healthy.
6. Gateway buffers SSE and hides first token.
7. HTTP 200 stream ends without completion.
8. Client cancellation does not stop GPU generation.
9. Retries amplify 429 overload and spend.
10. Telemetry disappears during customer-path failures.

Default **tabletop**. Live fault injection requires a separate staging change record.

## 13. 100-point acceptance rubric

| Category | Points |
|---|---:|
| Safety, approvals and data privacy | 10 |
| Request path, ownership and architecture | 10 |
| Tokenizer, template, context and parameters | 15 |
| Prefill/decode/KV/cache arithmetic | 15 |
| Streaming, gateway and cancellation correctness | 15 |
| SLI/SLO, latency and cohort analysis | 15 |
| Incident response, rollback and recovery | 10 |
| FinOps, evidence quality and independent review | 10 |
| **Total** | **100** |

**Proposed threshold:** ≥85/100 **and** no critical safety failure, complete-stream success contract, correct token-budget math, controller-authorized rollback plan and independent signoff. A published rubric is not a passing result.

## 14. Twenty acceptance gates

G01 approved scope; G02 verified context; G03 sanitized evidence; G04 request/ownership map; G05 model/tokenizer/template provenance; G06 context budget; G07 generation settings; G08 prefill/decode; G09 KV sizing assumptions; G10 continuous-batching fairness; G11 TTFT/TPOT/ITL units; G12 valid SSE completion; G13 proxy/backpressure/cancellation; G14 quota/retry safety; G15 SLO burn math; G16 telemetry freshness; G17 GPU/cost envelope; G18 incident command; G19 rollback/recovery; G20 independent review.

All gates **pending** until independently evidenced. See [Chapter 010 companion lab](../labs/010-inference-internals-performance-reliability-acceptance-check.md).

## 15. Part 02 closure

**Publication completion** means Chapters 006–010 and their labs are indexed and verified. **Operational completion** requires live lab execution where authorized, technical/security review, signed evidence and resolved critical findings.

**Next: Chapter 011 — GPU Architecture, SMs, HBM and Memory Bandwidth**, beginning Part 03 — GPU Architecture, CUDA and VRAM.
