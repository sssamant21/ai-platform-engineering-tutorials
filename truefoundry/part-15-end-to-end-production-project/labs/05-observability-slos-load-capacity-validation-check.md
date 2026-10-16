# Chapter 15.5 — Hands-on Lab: Observability, SLOs and Load/Capacity Validation

**Part 15 — End-to-End Production Project | Approved staging | Live execution pending**

## Goal

Define approved inference SLOs, discover actual vLLM/DCGM/TrueFoundry metrics, validate synthetic TTFT/TPOT and complete SSE, conduct bounded staging load tests, model GPU N-1 capacity, verify alerts and calculate cost per successful output token.

**Safety:** [SAFE-READ] inventory/dashboards; [LAB-MUTATE] isolated staging synthetic load with budget, max concurrency, duration and rollback; [CHANGE-CONTROL] production scaling, monitoring changes and traffic. Never test customer data or log PHI.

## Lab 0 — Preflight

Record service owner, SLO target, eligible requests, model/tokenizer/image/vLLM/CUDA, GPU SKU, staging endpoint, approved load, rate/concurrency ceilings, budget, pager contact and rollback. Verify network, security and GitOps acceptance from Chapter 15.4.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get deployments,pods -A -o wide
kubectl get hpa,pdb,resourcequota -A
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — Metric and alert inventory

Discover live Prometheus targets and available gateway/vLLM/Kubernetes/DCGM/agent metric families. Record names, units, labels, scrape freshness and owners. Validate logs/traces redaction and cardinality. Do not fabricate metric names.

## Lab 2 — Customer-path synthetic probe

Using approved authenticated synthetic client, measure HTTP success, complete SSE termination, TTFT, TPOT, output tokens, gateway latency, 429/5xx and correctness. Separate prompt/output cohorts and model revisions. Never print bearer tokens or private model outputs.

## Lab 3 — Controlled load

Execute baseline → bounded ramp → steady → short burst → cooldown with hard stop on 5xx, XID/ECC, repeated OOM, unauthorized route, cost cap or SLO breach. Capture request rate, concurrency, prompt/output distribution, TTFT/TPOT p50/p95/p99, tokens/sec, queue, GPU memory/utilization, KV cache and Pod restarts. Do not claim test passed before evidence review.

## Lab 4 — N-1, autoscaling and cost

Use measured safe capacity (not theoretical peak) to calculate GPU demand, rolling surge, failure-domain N-1, cloud quota, warmup and spend. Compare cost per successful 1,000 output tokens, including idle and retries. Tabletop destructive node failure unless specifically approved in staging.

## Lab 5 — Six game days

1. Prefill-heavy TTFT.
2. Decode-heavy TPOT.
3. Gateway SSE truncation.
4. GPU XID/ECC tabletop.
5. Prometheus scrape/paging gap.
6. N-1 plus canary rollback.

For each record UTC, approval, synthetic profile, expected/actual, SLO, abort/rollback, GPU-hours and reviewer.

## Forty-eight scenario checklist

01 eligibility; 02 samples; 03 burn; 04 probe; 05 p95; 06 stream; 07 401; 08 403; 09 429; 10 502; 11 503; 12 504; 13 TTFT; 14 TPOT; 15 queue; 16 KV; 17 CUDA OOM; 18 contract; 19 allocatable; 20 plugin; 21 DCGM; 22 XID; 23 ECC; 24 thermal; 25 Pending; 26 CrashLoop; 27 OOMKilled; 28 HPA; 29 autoscaler; 30 surge; 31 scrape; 32 cardinality; 33 logs; 34 traces; 35 silences; 36 pager; 37 N-1; 38 cold start; 39 burst; 40 soak; 41 noisy neighbor; 42 zone; 43 approval; 44 PHI; 45 budget; 46 drift; 47 rollback; 48 sign-off.

## Fourteen acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | SLO/eligible requests approved | Pending |
| G2 | Synthetic customer path | Pending |
| G3 | Complete SSE streams | Pending |
| G4 | TTFT/TPOT by cohort | Pending |
| G5 | Gateway error and queue | Pending |
| G6 | vLLM/KV cache metrics | Pending |
| G7 | GPU/DCGM freshness | Pending |
| G8 | Kubernetes/TrueFoundry visibility | Pending |
| G9 | Prometheus/logs/traces | Pending |
| G10 | Burn-rate alerts/paging | Pending |
| G11 | Baseline/ramp/steady/burst | Pending |
| G12 | N-1/surge/rollback | Pending |
| G13 | GPU cost/token/budget | Pending |
| G14 | Reviewer evidence/sign-off | Pending |

## Evidence template

```text
Lab ID / UTC / approval:
Cluster/model/image/tokenizer/vLLM/GPU:
Synthetic request cohorts and token mix:
SLO eligibility/targets/observation window:
Metric names, units, scrape freshness:
TTFT/TPOT p50/p95/p99, SSE completion:
429/5xx, queue, tokens/sec, KV cache:
GPU VRAM/utilization/XID/ECC:
HPA/autoscaler/N-1/zone/rollout overlap:
Alerts/pager/rollback:
GPU-hours and cost per 1k successful output tokens:
Executed/tabletop, blockers, reviewer:
```

## Cleanup

Stop load generator, verify no unintended scaling or orphaned GPUs, restore approved staging revision, re-enable authorized alerts, retain only sanitized evidence and confirm budget. Leave gates pending until execution and independent review.

**Next:** Chapter 15.6 — Failure Injection, Incident Response and Recovery.
