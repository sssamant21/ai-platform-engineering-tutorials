# Chapter 024 — Hands-On Lab: vLLM Slowness, CUDA OOM and Recovery Runbooks

**Part 04 | Chapter 024 | Published lab plan; live execution pending**

## Goal

Run a safe vLLM incident tabletop, classify TTFT/TPOT, queue, KV, CUDA OOM, NCCL and gateway failures, create a timeline, choose a mitigation and produce RCA/COA and customer-path recovery evidence.

## Safety

**[SAFE-READ]** approved cluster/GPU evidence and redacted logs. **[LAB-MUTATE]** only separately approved isolated staging with synthetic prompts and bounded GPU-hours. **[CHANGE-CONTROL]** production traffic routing, replicas, admission, GPU nodes, model/runtime revisions and rollback. No customer prompts, PHI, secrets or unapproved fault injection.

## Lab 0 — Incident preflight

Record UTC, cluster/context, namespace, model and tokenizer revision, vLLM image, GPU SKU/UUID, TP/PP layout, baseline SLO, incident commander, cost ceiling, approval and rollback.

```bash
date -u
kubectl config current-context
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — Customer impact worksheet [DESIGN]

| Metric | Baseline | Incident | Source |
|---|---|---|---|
| complete SSE | TBD | TBD | client/gateway |
| p95/p99 TTFT | TBD | TBD | defined boundary |
| p95/p99 TPOT | TBD | TBD | generated tokens |
| 429/5xx/504 | TBD | TBD | gateway |
| waiting/running | TBD | TBD | runtime |
| KV/preemption | TBD | TBD | pinned exporter |
| GPU VRAM/XID | TBD | TBD | GPU telemetry |
| GPU-hours/valid request | TBD | TBD | FinOps |

Never fabricate incident measurements; mark missing evidence unknown.

## Lab 2 — Read-only Kubernetes and GPU checks [SAFE-READ]

For an approved deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=150
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu,temperature.gpu --format=csv
nvidia-smi topo -m
```

Redact logs; never export Secrets or request bodies.

## Lab 3 — OOM classifier [DESIGN]

Classify five synthetic cases:
1. model fails before ready → weights/graphs/workspace;
2. 32K prompt fails during prefill → temporary memory/context;
3. many active decode requests fail → KV/concurrency;
4. Pod OOMKilled with no CUDA error → container/cgroup memory;
5. only TP rank 2 fails → rank-specific memory/topology.

For each, list confirming and falsifying evidence, containment, rollback and validation.

## Lab 4 — Slowness decision tree [DESIGN]

Build branches for high TTFT, high TPOT, HTTP 200 partial SSE, 429, 504, GPU idle with queue and high GPU utilization with low tokens/s. Record evidence before remediation.

## Lab 5 — Synthetic error-budget worksheet [DESIGN]

```text
Target complete-stream success = 99.9%
Eligible streams = 100,000
Failed or incomplete = 400
Allowed failures = 100
Observed success = 99.6%
Burn rate = 4×
```

Use approved SLO windows, cancellation policy and low-volume safeguards.

## Lab 6 — Recovery plan and rollback [TABLETOP]

Define incident commander, mitigation owner, stop conditions, known-good image/model/config, routing strategy, rollback controller, synthetic verification, post-recovery observation window and GPU cost accounting.

## Lab 7 — Eight tabletop game days

01 long prefill starvation; 02 KV decode OOM; 03 model load OOM; 04 NCCL rank crash; 05 SSE gateway timeout; 06 retry amplification; 07 XID plus stale telemetry; 08 incomplete recovery after rollback.

## Twenty-four failure checklist

F01 queue; F02 CPU; F03 prefill; F04 decode; F05 KV; F06 preemption; F07 weights; F08 graphs; F09 prefill OOM; F10 decode OOM; F11 rank OOM; F12 cgroup; F13 NCCL init; F14 collective; F15 registry; F16 tokenizer; F17 buffering; F18 timeout; F19 retries; F20 cancellation; F21 rollout GPU; F22 telemetry; F23 XID/ECC; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Incident scope | Pending |
| G02 | UTC timeline | Pending |
| G03 | Pinned release | Pending |
| G04 | Telemetry freshness | Pending |
| G05 | Complete SSE | Pending |
| G06 | TTFT | Pending |
| G07 | TPOT | Pending |
| G08 | Queue | Pending |
| G09 | KV/preemption | Pending |
| G10 | GPU memory | Pending |
| G11 | OOM classification | Pending |
| G12 | NCCL/rank | Pending |
| G13 | Gateway timeout | Pending |
| G14 | Retry/cancel | Pending |
| G15 | Security/privacy | Pending |
| G16 | Rollback | Pending |
| G17 | SLO recovery | Pending |
| G18 | GPU FinOps | Pending |
| G19 | RCA/COA | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
Incident ID/UTC/commander:
Context/model/vLLM/GPU:
Customer impact and eligible streams:
TTFT/TPOT/complete SSE:
Queue/KV/preemption:
CUDA vs cgroup OOM evidence:
NCCL/rank/gateway:
Hypotheses and ruled-out causes:
Mitigation and rollback:
SLO/FinOps recovery:
RCA/COA owner and due date:
Reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop any separately approved staging load, restore known-good config through the authoritative controller, remove only owned artifacts and reconcile GPU-hours. Acceptance remains pending until independently reviewed.

**Next: Chapter 025 — vLLM Production Acceptance Project.**
