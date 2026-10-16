# Chapter 005 — Fundamentals Acceptance Lab and Incident Walkthrough

**Part 01 — AI/LLM SRE Fundamentals | Chapter 005 of 100 | Part 01 capstone**

> **Publication status:** Tutorial and companion lab are written for GitHub. No live lab, technical/security review, or production readiness signoff has occurred.

## Learning objectives

Demonstrate the Part 01 fundamentals through a complete SRE incident exercise: reconstruct a customer inference path, define reliable SLIs, calculate error-budget impact, triage Kubernetes/GPU and streaming failures, produce a severity-based incident timeline, and pass an evidence-driven readiness review.

## 1. Capstone scenario

A synthetic interactive chat service uses a gateway and Kubernetes Service to route traffic to a GPU-backed vLLM workload. After a deployment, customers report slow first tokens and streams ending prematurely. Gateway responses are often HTTP 200, but some streams lack valid completion. One GPU Pod is Pending, and the inference queue is growing.

**This is a simulated incident.** Do not infer it occurred in your environment.

```text
synthetic client
  → DNS/TLS/auth/rate-limit gateway
  → ingress/Service/EndpointSlice
  → inference Pod (vLLM / compatible runtime)
  → tokenizer → prefill → KV cache → decode → SSE
  → GPU / CUDA / driver / device plugin
  → immutable model registry/object store

control: GitOps / CI / TrueFoundry or operator / Kubernetes API
telemetry: customer-path synthetic + Prometheus/OTel + GPU/DCGM + paging
```

The control plane is not automatically on the request path. Confirm which controller owns each workload before proposing rollback.

## 2. Entry criteria and lab boundaries

Before beginning, confirm the approved cloud account, Kubernetes context, namespace, UTC window, synthetic endpoint, service owner, on-call contact, budget, allowed commands and evidence destination.

- **[SAFE-READ]**: approved inventory, logs and dashboards; redact sensitive values.
- **[LAB-MUTATE]**: isolated staging synthetic traffic, only with explicit approval and bounded tokens/requests.
- **[CHANGE-CONTROL]**: production changes, model rollouts, GPU provisioning, failure injection, gateway changes and cleanup of shared resources.

Stop if environment identity, ownership or authorization is unclear. Never use PHI, real customer prompts, credentials or private model outputs in exercises.

## 3. Establish the service and ownership map

| Component | Accountable owner | Required evidence |
|---|---|---|
| User journey and SLO | service owner | eligible request contract |
| Gateway/TLS/auth | API/security owner | request routing, quotas, SSE behavior |
| Kubernetes networking | platform SRE | Service, endpoints, readiness |
| GPU scheduling/CUDA | GPU platform owner | allocatable, plugin, compatibility |
| Model/runtime | inference owner | image digest, model and tokenizer revision |
| GitOps/TrueFoundry | release/platform owner | authoritative reconciler |
| Telemetry/on-call | SRE | alerts, traces, synthetic freshness |
| Cost/DR | FinOps and service owner | quotas, fallback, recovery goals |

Record primary, backup and escalation contacts. Avoid ambiguous ownership of canary, rollback and emergency capacity.

## 4. Customer-facing reliability contract

Define one **eligible request** as a valid, authorized synthetic request the service accepted responsibility to serve. Decide explicitly whether quota 429, client cancellation and retries are included. Count a streaming response as successful only after protocol-valid completion; HTTP 200 alone is insufficient.

```text
availability = good_eligible_requests / eligible_requests
error_ratio = failed_eligible_requests / eligible_requests
budget_fraction = 1 - SLO_target
burn_rate = observed_error_ratio / budget_fraction
```

For an illustrative 99.9% request SLO and 20,000 eligible requests, 20 failures exhaust the entire request-based budget. A 1% observed error ratio burns budget at 10×. These numbers are examples, not approved objectives.

Use TTFT, TPOT/ITL, end-to-end latency, stream completion, queue delay, GPU memory and request error classes as separate indicators. Segment by model revision, prompt length, output length, region and streaming mode; never average per-Pod p95s.

## 5. Read-only triage workflow

First confirm the customer-path symptom and independent synthetic probe. Check telemetry freshness before treating missing errors as success.

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get deploy,sts,ds,pods -A -o wide
kubectl get svc,endpointslices -A
kubectl get resourcequota,hpa,pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

For an approved namespace and deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE get pods -o wide
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Logs may contain sensitive data; sanitize before sharing. Confirm image/model/tokenizer/driver/CUDA compatibility and scheduling requests. Use approved DCGM dashboards rather than privileged GPU debug containers.

### Decision tree

```text
synthetic fails?
 ├─ DNS/TLS/401/403 → edge/auth/tenant boundary
 ├─ 429 → quota policy versus queue saturation
 ├─ 502/503/504 → gateway, endpoints, readiness, timeout
 ├─ 200 but incomplete SSE → proxy drain, stream timeout, runtime
 ├─ high TTFT → queue, prefill, model warmup, cold start
 ├─ high TPOT → decode, GPU contention, KV cache
 ├─ Pending Pod → quota, taints, allocatable, AZ stock
 ├─ CUDA OOM/no CUDA → memory, image/driver/device plugin
 ├─ quality regression → model/tokenizer/evaluation
 └─ missing telemetry → scrape, synthetic, paging
```

Do not scale blindly: additional GPU Pods can remain Pending or increase model-load pressure and costs. Never use direct `kubectl rollout undo` against controller-managed resources without confirming the authoritative rollback mechanism.

## 6. Incident command and communication

A production incident should have an incident commander, technical lead, communications owner, scribe, declared severity, customer impact, decision log and next update time.

Illustrative timeline:

| UTC | Observation | Decision | Evidence |
|---|---|---|---|
| 14:00 | new release promoted | note version and canary | release manifest |
| 14:05 | TTFT and incomplete SSE rise | declare incident | synthetic, gateway |
| 14:08 | GPU Pod Pending | check scheduler/quota | events |
| 14:12 | queue rises | protect traffic, halt rollout | queue and rate |
| 14:20 | authorized rollback initiated | owner confirms compatible version | GitOps history |
| 14:30 | synthetic and SLO recover | monitor sustained recovery | good/total events |
| 15:00 | handoff and RCA assigned | follow-up owners | incident record |

This is an example only, not evidence of an executed rollback.

### Status update template

```text
Incident ID / severity / UTC:
Customer impact and affected cohorts:
What is known / what is not known:
Current mitigation and owner:
SLO/error-budget impact:
Security and data-exposure assessment:
Next update:
```

## 7. Safe containment and recovery

Possible mitigations include freezing promotion, routing to a previously verified healthy model revision, reducing concurrency or output limits under approved policy, controlling client retries, or recovering a misconfigured gateway. Validate each action against security, tenant fairness, data integrity and customer impact.

Recovery requires **customer-path** success, valid complete SSE, stable TTFT/TPOT and queue, sufficient GPU capacity, no active data exposure, telemetry freshness and sustained SLO recovery. Healthy Pods alone are insufficient.

## 8. Twenty failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | Wrong cluster context | cluster/account identity |
| F02 | Gateway TLS failure | handshake probe |
| F03 | Auth policy regression | 401/403 cohort |
| F04 | Quota misclassification | 429 policy |
| F05 | Empty EndpointSlice | endpoints/readiness |
| F06 | Pod Pending GPU | scheduler/allocatable |
| F07 | Device plugin missing | GPU resource advertisement |
| F08 | Driver/CUDA mismatch | runtime logs |
| F09 | GPU OOM | VRAM/runtime logs |
| F10 | Model artifact denied | IAM and object-store logs |
| F11 | Model/tokenizer mismatch | immutable manifest |
| F12 | Cold-start latency | model load and TTFT |
| F13 | Queue saturation | queue, concurrency |
| F14 | Decode slowdown | TPOT/GPU |
| F15 | SSE truncated after HTTP 200 | client completion probe |
| F16 | Missing scrape | target freshness |
| F17 | Counter double count | edge/runtime instrumentation |
| F18 | Noisy low-traffic burn | traffic-floor guard |
| F19 | Unexpected GPU spend | budget and quotas |
| F20 | Rollback controller conflict | GitOps/owner reconciliation |

## 9. Fourteen operational runbooks

1. Preflight context, approval and incident safety.
2. Declare severity and establish incident command.
3. Validate customer-path synthetic and stream completion.
4. Inspect gateway DNS/TLS/auth/429/5xx.
5. Inspect Kubernetes Services, endpoints and readiness.
6. Triage GPU scheduling, taints, quota and autoscaler.
7. Triage CUDA, driver, device plugin and VRAM.
8. Triage model artifact and tokenizer revision.
9. Triage TTFT, prefill, queue and cold starts.
10. Triage TPOT, decode, KV cache and contention.
11. Validate SLO/error-budget and missing-data handling.
12. Freeze/canary/rollback through release authority.
13. Validate recovery, cost, security and capacity.
14. Complete incident handoff, RCA/COA and corrective actions.

Each runbook needs a named owner, read-only diagnostics, escalation, approved mutation, rollback and evidence of recovery.

## 10. Eight capstone game days

1. Streaming HTTP 200 but truncated completion.
2. Gateway 429 from internal saturation versus contractual quota.
3. GPU Pending with no available capacity.
4. CUDA mismatch after image upgrade.
5. Model artifact unavailable during scale-out.
6. TTFT regression during prefill-heavy traffic.
7. Missing metrics while synthetic probes fail.
8. Controller conflict during attempted rollback.

Use tabletop mode by default; fault injection is restricted to separately approved staging with abort conditions and a cost ceiling.

## 11. Twenty-point acceptance rubric

| Category | Points | Evidence |
|---|---:|---|
| Authorization and safety | 10 | scope, context, approvals, redaction |
| Architecture and RACI | 15 | dependency map, ownership |
| SLI/SLO/error budget | 15 | eligibility, streaming, arithmetic |
| Read-only investigation | 15 | gateway, Kubernetes, GPU evidence |
| Incident leadership | 10 | timeline, severity, communications |
| Containment and rollback design | 15 | safe action and authoritative owner |
| Recovery verification | 10 | synthetic, SLO, capacity, telemetry |
| RCA/COA and follow-up | 10 | corrective actions and reviewer |
| **Total** | **100** | |

**Proposed passing threshold:** at least 85/100, no critical safety violations, and all required evidence reviewed. This threshold is a curriculum design choice, not an actual assessment result.

### Twenty acceptance gates

G01 approved scope; G02 verified context; G03 sanitized evidence; G04 request-path diagram; G05 RACI; G06 trust boundaries; G07 SLI eligibility; G08 complete SSE success; G09 TTFT/TPOT units; G10 error-budget math; G11 telemetry freshness; G12 GPU scheduling triage; G13 CUDA/runtime compatibility; G14 model artifact and versioning; G15 incident declaration; G16 mitigation approval; G17 rollback design; G18 customer-path recovery; G19 RCA/COA; G20 independent reviewer.

All gates remain **pending** until executed and signed off. See the [Chapter 005 companion lab](../labs/005-fundamentals-acceptance-lab-incident-walkthrough-check.md).

## Part 01 completion criteria

Part 01 **written curriculum** is complete when Chapters 001–005 and companion labs are published and indexed. **Operational acceptance** is separate: technical/security review, lab execution and reviewer signoff remain required.

**Next: Chapter 006 — Transformer Inference Request Lifecycle**, beginning Part 02 — LLM Architecture and Inference Internals.
