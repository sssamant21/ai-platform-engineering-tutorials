# Chapter 024 — vLLM Slowness, CUDA OOM and Recovery Runbooks

**Part 04 — vLLM Administration and Operations | Chapter 024 of 100**

> **Status:** Written tutorial published. Independent technical/security review, live lab execution and operational acceptance pending. Commands, metrics and runtime behavior must be validated for the pinned vLLM version and environment.

## Learning objectives

Investigate slow or failing vLLM inference using customer-path SLIs, scheduler/queue telemetry, GPU memory, CUDA/NCCL errors and Kubernetes evidence; distinguish prefill, decode, KV cache, model loading, network and proxy bottlenecks; recover safely without data loss, uncontrolled retries or unnecessary GPU spending; produce RCA, COA and evidence-based acceptance.

## 1. Incident response principles

1. **Establish impact:** model, route, tenant class, geography, first failure time, error budget and number of incomplete streams.
2. **Preserve evidence:** immutable release identifiers, events, sanitized logs, GPU/rank health and dashboards.
3. **Separate symptoms from causes:** high TTFT may be queue or prefill; high TPOT may be decode or transport; OOM may be weights, KV or workspace.
4. **Stabilize before tuning:** stop unsafe rollouts, enforce admission, redirect only to known-good capacity and prevent retry storms.
5. **Change one thing at a time:** use approved staging/canary and explicit rollback.
6. **Verify customer-path recovery:** complete SSE, p95/p99 TTFT/TPOT, errors and cost—not Pod Ready alone.

## 2. Inference request lifecycle

```text
Client request
   │ TLS/auth/quota/gateway
   ▼
HTTP request accepted
   │ tokenization + chat template
   ▼
Scheduler queue
   │ prefill / KV allocation
   ▼
First generated token (TTFT)
   │ decode / KV growth / GPU communication
   ▼
Streaming chunks (TPOT / ITL)
   │ gateway flush / cancellation
   ▼
Finish reason + terminal SSE marker
   ▼
Valid completed request (E2E / FinOps)
```

A successful HTTP 200 is insufficient evidence of successful streaming completion. A client can receive partial content before backend failure, gateway timeout or cancellation.

## 3. Symptom-to-component mapping

| Symptom | Primary hypotheses | Evidence |
|---|---|---|
| TTFT rises | queue, CPU tokenization, long prefill, model cold start | queue wait, prompt buckets |
| TPOT rises | decode saturation, KV pressure, bandwidth, collectives | inter-token time, KV, GPU |
| 429 spikes | tenant quotas or overload admission | gateway/backend reason |
| 502/503 spikes | unready Pod, worker crash, service endpoints | Kubernetes events |
| 504 spikes | gateway idle/total timeout, backend stall | trace timeline |
| 200 but partial SSE | worker failure, timeout, cancellation | terminal-event validator |
| CUDA OOM at startup | weights, CUDA graphs, workspace, other processes | per-rank VRAM and logs |
| CUDA OOM on long prefill | temporary workspace, prompt length | input cohort |
| CUDA OOM during decode | resident KV, concurrency, cache blocks | active tokens/preemption |
| NCCL timeout | failed rank, network, incompatible stack | all-rank health |
| GPU idle while queue grows | scheduler, CPU, routing or model unready | queue and device utilization |
| GPU busy, throughput low | long contexts, inefficient batching, memory pressure | tokens/s and cohort mix |

## 4. Read-only first-response checklist

**[SAFE-READ]** in the approved cluster:

```bash
date -u
kubectl config current-context
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

For a verified deployment and namespace:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE get pods -l app=APPROVED_LABEL -o wide
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=150
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu,temperature.gpu --format=csv
nvidia-smi topo -m
```

The label, Service and Deployment names are placeholders. Do not run commands against an unverified production context, dump Secrets, or capture customer prompts. When an incident requires broader access, follow the incident authorization process.

## 5. Investigating slow TTFT

TTFT comprises request routing, admission/queue, tokenization, prefill and first-token delivery. Confirm measurement boundaries first.

**Decision sequence:**
1. Verify customer TTFT and complete-stream success against baseline.
2. Check waiting/running requests, admission and per-tenant quotas.
3. Compare short and long prompt cohorts.
4. Check CPU saturation/tokenization, gateway delay and prefill GPU activity.
5. Inspect context limits, batched-token scheduling, preemption and recent model/template revisions.
6. Stabilize via bounded admission, workload isolation or known-good capacity before experimental tuning.

Do not automatically increase concurrency when TTFT is already queue-bound.

## 6. Investigating slow TPOT and decode

High TPOT often correlates with large resident KV, high decode concurrency, GPU memory bandwidth pressure, communication overhead or streaming proxy behavior.

Compare backend generated-token timing with client-observed chunk arrival. If backend tokens are timely but clients receive bursts, inspect proxy buffering and flush settings. If backend decode slows with high KV occupancy/preemption, examine resident tokens and scheduler limits. Avoid increasing GPU memory utilization without headroom analysis.

## 7. CUDA OOM classification

```text
CUDA OOM?
 ├─ startup/model load → weights, dtype, quantization, graphs
 ├─ first long prompt → prefill workspace, max context
 ├─ high concurrent decode → KV blocks, resident tokens
 ├─ one TP rank only → per-rank imbalance, topology
 ├─ after rollout → model/revision/config/driver drift
 └─ apparent free VRAM → allocator reserve, fragmentation,
                          other processes, reporting lag
```

Per-GPU peak usage matters more than cluster aggregate. OOM may terminate a worker or leave the runtime unhealthy; restart only through the approved controller after identifying the likely cause and preventing recurrence.

## 8. Memory capacity worksheet

For a conventional attention model:

```text
KV bytes/token ≈ 2 × layers × KV_heads × head_dim × bytes/element
```

Synthetic example: 32 layers, 8 KV heads, 128 head dimension, BF16 (2 bytes):

```text
KV/token = 2 × 32 × 8 × 128 × 2 = 131,072 bytes = 128 KiB
32,768 resident tokens ≈ 4 GiB logical KV
```

This excludes block rounding, CUDA graphs, workspace, allocator reserve, TP/PP distribution and runtime-specific caching. Model weights, KV, workspace, other GPU processes and headroom must fit together on each participating GPU.

## 9. NCCL and multi-GPU incident triage

A collective timeout can be a secondary symptom of a peer rank OOM, process exit, NIC failure or mismatched configuration.

Record:
- exact vLLM image, model/tokenizer revision, TP/PP layout;
- rank-to-GPU UUID and node mapping;
- driver/CUDA/NCCL versions;
- per-rank OOM/restart timestamps;
- network interface, GPU topology and approved fabric telemetry;
- whether failures began after a rollout.

Do not enable verbose NCCL logs or packet capture in production without approval; coordination endpoints must remain isolated.

## 10. Kubernetes and gateway failure modes

A healthy Kubernetes Pod does not guarantee model readiness, correct Service routing or complete SSE. Check:
- image pull, scheduling, GPU device plugin, readiness and startup probes;
- OOMKilled versus CUDA OOM (host/container memory and GPU memory differ);
- Service endpoints, ingress timeouts and buffering;
- rollout surge GPU requirements and coexistence of old/new replicas;
- model download/registry access and startup duration;
- rate-limit/retry behavior that can amplify load.

Distinguish Linux cgroup OOM from CUDA allocation failure before changing limits.

## 11. Error budget and incident severity

Illustrative complete-stream SLO:

```text
Target: 99.9%
Eligible requests: 100,000
Failed or incomplete: 400
Allowed failures: 100
Observed success: 99.6%
Burn rate: 4×
```

Use approved SLO windows and low-volume safeguards. Severity also depends on affected customers, data sensitivity, security and availability—not only burn rate.

## 12. Safe mitigation patterns

**[CHANGE-CONTROL]** for production:
- pause the suspect rollout or route traffic to a known-good version;
- enforce per-tenant token/concurrency limits and bounded retry;
- reduce expensive long-context cohorts through approved admission policy;
- scale replicas only if GPU quota, model load time and cost allow;
- rollback model/runtime/scheduler changes via authoritative GitOps;
- cordon or replace a demonstrably unhealthy GPU node under platform procedures;
- keep evidence and track impact throughout mitigation.

Never terminate all replicas simultaneously or clear caches indiscriminately during an incident.

## 13. Incident command and communications

Use a single incident commander, clear severity, timestamps, mitigation owner, SLO/impact tracker and rollback authority. Updates should distinguish **confirmed facts**, **working hypotheses**, **actions taken** and **next validation**. Capture UTC timelines and sanitized links; do not put customer prompts, tokens or credentials in incident channels.

## 14. Troubleshooting decision tree

```text
Customer-path vLLM incident?
 ├─ no response → DNS/TLS/ingress/service/readiness
 ├─ 401/403 → identity and policy
 ├─ 429 → quota vs overload
 ├─ 502/503/504 → upstream crash/timeout
 ├─ 200 partial SSE → backend rank/proxy/cancel
 ├─ high TTFT → queue/tokenization/prefill
 ├─ high TPOT → decode/KV/collectives/flush
 ├─ startup OOM → weights/graphs/workspace
 ├─ prefill OOM → long prompt/temp memory
 ├─ decode OOM → KV/concurrency
 ├─ NCCL timeout → rank/network/versions
 ├─ metrics flatline → telemetry freshness
 └─ cost spikes → retries/idle GPUs/long outputs
```

## 15. Twenty-four production failure scenarios

| ID | Failure | Primary evidence |
|---|---|---|
| F01 | queue saturation | waiting requests |
| F02 | tokenization CPU saturation | CPU and TTFT |
| F03 | long prefill starvation | prompt cohorts |
| F04 | decode bandwidth saturation | TPOT and GPU |
| F05 | KV block exhaustion | cache/preemption |
| F06 | preemption thrash | scheduler counters |
| F07 | startup weight OOM | model load logs |
| F08 | CUDA graph workspace OOM | startup logs |
| F09 | long-prompt prefill OOM | prompt cohort |
| F10 | concurrent decode OOM | resident tokens |
| F11 | TP rank-specific OOM | per-rank VRAM |
| F12 | cgroup OOMKilled | Pod termination reason |
| F13 | NCCL initialization failure | rank startup |
| F14 | NCCL collective timeout | peer/fabric health |
| F15 | model artifact download failure | registry/network |
| F16 | incompatible tokenizer/template | synthetic fixture |
| F17 | gateway SSE buffering | chunk timing |
| F18 | gateway idle timeout | 504/partial SSE |
| F19 | retry amplification | gateway request rate |
| F20 | cancellation leak | active work after disconnect |
| F21 | rollout GPU surge shortage | scheduling events |
| F22 | stale/renamed metrics | exporter schema |
| F23 | GPU XID/ECC hardware issue | device telemetry |
| F24 | GPU cost per valid completion rises | FinOps |

## 16. Sixteen operational recovery runbooks

### RB01 — Declare and scope the incident
**Trigger:** SLO burn, failed/incomplete streams or material latency regression. **Action:** record UTC, model/tenant scope, incident commander, impact and baseline. **Recovery gate:** impact is quantified and tracked.

### RB02 — Validate telemetry freshness
**Trigger:** flatline/missing metrics. **Action:** inspect scrape targets, exporter schema, logs and client evidence. **Gate:** measurements are fresh or explicitly marked unknown.

### RB03 — Diagnose queue saturation
**Trigger:** rising waiting requests/TTFT. **Action:** classify tenant and prompt cohorts; review quotas and capacity. **Gate:** queue trend stabilizes without retry amplification.

### RB04 — Diagnose tokenization/prefill
**Trigger:** long-prompt TTFT regression. **Action:** compare CPU, tokenization, prefill and batched-token policy. **Gate:** short and long cohorts meet agreed latency.

### RB05 — Diagnose decode/KV
**Trigger:** TPOT and cache pressure. **Action:** inspect active tokens, KV occupancy and preemption. **Gate:** TPOT and complete SSE recover.

### RB06 — Recover startup weight OOM
**Trigger:** model fails to become ready. **Action:** check weights, dtype, graphs, available GPU memory and recent revision. **Gate:** pinned model loads with headroom.

### RB07 — Recover prefill OOM
**Trigger:** long prompt causes allocation failure. **Action:** confirm input cohort, temporary memory and admission caps. **Gate:** bounded synthetic long prompt completes.

### RB08 — Recover decode OOM
**Trigger:** high concurrency exhausts KV. **Action:** apply approved admission and rollback unsafe concurrency tuning. **Gate:** no recurrence under bounded load.

### RB09 — Distinguish cgroup versus CUDA OOM
**Trigger:** Pod restart or allocation failure. **Action:** inspect termination reason, container memory and GPU logs. **Gate:** correct resource domain identified.

### RB10 — Diagnose NCCL/rank failure
**Trigger:** rank loss or collective timeout. **Action:** inspect all ranks, driver/CUDA/NCCL, GPU UUIDs and fabric. **Gate:** distributed group healthy and complete streams validated.

### RB11 — Restore model artifact access
**Trigger:** model startup/download fails. **Action:** verify registry permissions, checksum, DNS/egress and pinned revision. **Gate:** approved model loads reproducibly.

### RB12 — Diagnose ingress/SSE failures
**Trigger:** HTTP 200 partial streams, 502/504 or delayed chunks. **Action:** inspect terminal validator, gateway buffering/timeouts and backend timeline. **Gate:** complete SSE success recovers.

### RB13 — Stop retry/cancellation amplification
**Trigger:** repeated retries, zombie generations or GPU burn. **Action:** enforce bounded backoff and cancellation propagation. **Gate:** retries and GPU work decline safely.

### RB14 — Recover GPU/node faults
**Trigger:** XID/ECC, device loss or node instability. **Action:** coordinate with platform team, preserve device evidence, follow approved node replacement. **Gate:** healthy GPU allocation and SLO.

### RB15 — Roll back release/config
**Trigger:** regression linked to deployment. **Action:** pause rollout, restore last known-good immutable revision through GitOps. **Gate:** release consistency and customer-path recovery.

### RB16 — Close incident and RCA/COA
**Trigger:** stabilized service. **Action:** validate error budget, p95/p99, complete SSE, GPU cost and follow-up owners. **Gate:** evidence-based review and preventive actions.

## 17. Eight tabletop game days

1. Long prompts saturate prefill and short requests starve.
2. KV exhaustion causes decode OOM and partial SSE.
3. New model revision OOMs during rollout.
4. One TP rank crashes and NCCL peers hang.
5. Gateway buffers SSE and then times out.
6. Retry storm doubles GPU-hours while success drops.
7. GPU XID error appears with stale exporter metrics.
8. Rollback restores Pods but customer TTFT remains degraded.

Default **tabletop**. Live fault injection requires approved isolated staging and explicit GPU cost ceilings.

## 18. Twenty acceptance gates

G01 incident scope; G02 UTC timeline; G03 pinned release; G04 telemetry freshness; G05 complete SSE; G06 TTFT; G07 TPOT; G08 queue; G09 KV/preemption; G10 GPU memory; G11 OOM classification; G12 NCCL/rank; G13 gateway timeout; G14 retry/cancel; G15 security/privacy; G16 rollback; G17 SLO recovery; G18 GPU FinOps; G19 RCA/COA; G20 independent review.

See [companion lab](../labs/024-vllm-slowness-cuda-oom-recovery-runbooks-check.md). All gates remain pending until executed and independently reviewed.

**Next: Chapter 025 — vLLM Production Acceptance Project.**
