# Chapter 14.5 — Hands-on Lab: vLLM Serving Health and Performance

**Part 14 — SRE Runbooks | Approved staging | Live execution pending**

## Goal and safety

Establish a measurable vLLM inference baseline, classify latency/queue/memory faults, test synthetic streaming, verify scaling and rollback. **[SAFE-READ]** authorized metrics and logs; **[LAB-MUTATE]** isolated staging with fixed GPU cost and synthetic traffic; **[CHANGE-CONTROL]** production traffic, concurrency, model context, rollout and node operations. No real prompts, PHI, customer responses or secrets.

## Lab 0 — Inventory and baseline

```bash
kubectl config current-context
kubectl -n MODEL_NAMESPACE get deployments,pods -o wide
kubectl -n MODEL_NAMESPACE get svc,endpointslices
kubectl -n MODEL_NAMESPACE get events --sort-by=.lastTimestamp
```

Record TrueFoundry revision, model/tokenizer digest, vLLM version, driver/CUDA, GPU SKU, max context, batch/concurrency configuration, replicas, autoscaler/quota, SLO and cost ceiling.

## Lab 1 — Synthetic inference SLO

Run approved low-rate short/long synthetic prompts and outputs. Record eligible requests, success, 429/5xx, p50/p95 TTFT, TPOT, queue, tokens/sec, end-to-end latency and SSE completion. Keep traffic mix stable when comparing changes.

## Lab 2 — Metrics and GPU correlation

Verify actual vLLM metrics names and units from approved internal `/metrics` endpoint, scrape freshness and DCGM GPU utilization/memory/XID/ECC. Separate model-serving signals from ingress and cloud GPU capacity.

```bash
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE logs MODEL_POD --since=30m --tail=200
```

## Lab 3 — Bounded load and memory classification

In staging, vary one supported concurrency/batch/context variable at a time under budget. Measure TTFT, TPOT, queue, throughput and GPU memory. Tabletop unsafe CUDA OOM, driver reset or multi-tenant memory exhaustion. Distinguish host OOMKilled from CUDA OOM and KV cache pressure.

## Lab 4 — Streaming and autoscaling

Validate complete SSE streams, proxy timeout, client cancellation and drain behavior using approved synthetic requests. Verify whether additional replicas are schedulable and warm before assuming autoscaling can restore the SLO.

## Lab 5 — Seven game days

1. Cold start and long-prompt TTFT.
2. Queue saturation and batching contention.
3. TPOT regression with GPU saturation.
4. KV cache pressure/CUDA OOM (tabletop if unsafe).
5. Gateway 504 and truncated SSE.
6. Autoscaler GPU quota or N-1 shortfall.
7. Bad model/runtime revision with supported rollback.

For each capture UTC timeline, expected/actual, SLO, workload token distribution, mitigation, rollback, GPU-hours, reviewer and executed/tabletop status.

## Forty-eight scenario checklist

01 image; 02 artifact; 03 tokenizer; 04 CUDA; 05 GPU missing; 06 warmup; 07 TTFT; 08 TPOT; 09 queue; 10 long prompt; 11 long output; 12 prefill; 13 waiting; 14 batch; 15 concurrency; 16 throughput; 17 workload mix; 18 priority; 19 CUDA OOM; 20 host OOM; 21 KV full; 22 preemption; 23 fragmentation; 24 context; 25 XID; 26 ECC; 27 thermal; 28 healthy high utilization; 29 low utilization slow; 30 multi-GPU; 31 502; 32 503; 33 504; 34 buffering; 35 truncation; 36 cancel; 37 quota; 38 autoscaler; 39 zone; 40 cold replica; 41 N-1; 42 spot; 43 metrics; 44 alert; 45 retries; 46 cost/token; 47 rollback; 48 UI-only verification.

## Fourteen acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Approved scope/GPU cost | Pending |
| G2 | Model/runtime immutable inventory | Pending |
| G3 | Synthetic inference/SSE | Pending |
| G4 | TTFT/TPOT baseline | Pending |
| G5 | Queue/throughput correlation | Pending |
| G6 | KV cache/memory | Pending |
| G7 | CUDA vs host OOM classification | Pending |
| G8 | GPU/DCGM telemetry | Pending |
| G9 | Gateway/429/5xx | Pending |
| G10 | Autoscaler/N-1 capacity | Pending |
| G11 | Compatible rollback | Pending |
| G12 | Metrics/alerts freshness | Pending |
| G13 | Cost/token and recovery SLO | Pending |
| G14 | Reviewer sign-off | Pending |

## Evidence template

```text
Exercise / change approval / UTC:
Cluster/model/revision/vLLM/CUDA/GPU:
Synthetic input/output token distribution:
Eligible requests, TTFT, TPOT, queue, tokens/sec:
GPU memory/utilization/KV cache/preemptions:
Gateway HTTP and SSE completion:
Autoscaler, quota, N-1:
Expected/actual, hypothesis, mitigation:
Rollback and post-recovery SLO:
GPU-hours and cost per successful token:
Executed/tabletop, owner, reviewer:
```

## Cleanup

Restore staging model/config/traffic, remove disposable synthetic load, confirm healthy streams, queue, GPU reserve, alerts and spend. Publication does not mean exercises ran.

**Next:** Chapter 14.6 — Incident Response, Escalation and Communications Runbook.
