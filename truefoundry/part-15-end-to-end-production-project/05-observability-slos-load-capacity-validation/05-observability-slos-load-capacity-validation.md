# Chapter 15.5 — Observability, SLOs and Load/Capacity Validation

**Part 15 — End-to-End Production Project | Capstone 5 of 7**

## Objective and safety

Establish production-grade observability for TrueFoundry-managed vLLM inference, define customer-centered service level objectives (SLOs), and prove performance, GPU capacity, resilience and cost under representative staging workloads. Integrates Chapters 15.1–15.4: GPU infrastructure, TrueFoundry reconciliation, secure networking, GitOps and rollback.

**[SAFE-READ]** authorized dashboards, metrics, traces, logs, configuration and read-only kubectl. **[LAB-MUTATE]** approved isolated staging load tests with concurrency ceilings, synthetic prompts, spend caps and teardown. **[CHANGE-CONTROL]** production load, autoscaling, alert routing, GPU provisioning and model configuration. No real customer prompts, PHI, tokens, credentials or unredacted outputs in telemetry.

**Publication does not mean SLOs were met or benchmarks executed.** Metrics and flags vary across TrueFoundry, vLLM, Kubernetes, DCGM and exporter versions; discover actual names before queries or alerts.

## 1. End-to-end telemetry architecture

```text
Synthetic client / gateway
   │ request id, tenant class, status, TTFT, TPOT, stream completion
   ▼
Ingress / Service / EndpointSlice
   │ latency, 401/403, 429, 5xx, TLS, connection/drain
   ▼
vLLM engine
   │ prefill, decode, queue, KV cache, tokens/s, OOM
   ▼
Kubernetes Pods / nodes / autoscaler
   │ restarts, Pending, quotas, placement, scaling
   ▼
GPU device plugin / DCGM
   │ GPU memory, utilization, temperature, XID/ECC
   ▲
TrueFoundry control plane / agent reconciliation
   │ desired vs observed, connectivity, rollout state
   └─ Metrics + logs + traces + events + audit
              │
        SLO recording rules / dashboards / pager
```

Separate control-plane deployment health from data-plane inference availability. A healthy agent or HTTP 200 is not proof that streaming completes or tokens arrive within SLO.

## 2. SLI/SLO specification

| SLI | Numerator / measure | Denominator / cohort | Failure nuance |
|---|---|---|---|
| Availability | eligible successful completed inference | eligible requests | exclude only policy-approved classes |
| Streaming completion | streams with valid protocol completion | eligible streaming requests | HTTP 200 then disconnect is failure |
| TTFT | first token latency | streaming cohorts by input size/model | include queue, gateway and prefill |
| TPOT / ITL | per-output-token time | output-token cohort | avoid averaging across dissimilar lengths |
| E2E latency | completed request duration | model/tenant/region/cohort | non-streaming separate |
| Errors | 429, 5xx, auth, timeout | classified requests | 401/403 may be expected negative tests |
| Queue wait | scheduler wait time | accepted requests | capture concurrency and arrival |
| GPU reliability | hardware incidents, availability | GPU pool and time | XID/ECC and telemetry freshness |
| Control-plane | reconciliation age and success | approved deployments | version-dependent metrics |

Set actual numeric SLOs only after service-owner approval and staging baselines. Example math for availability SLO 99.9% over 30 days: allowed errors = eligible requests × 0.001; do not assume this target is approved. For p95 TTFT/TPOT use representative traffic mix and enough samples for statistical confidence.

## 3. Error budgets and alerting

For a request-based availability SLO:
```text
error_ratio = bad_eligible_requests / all_eligible_requests
error_budget_fraction = 1 - SLO_target
burn_rate = observed_error_ratio / error_budget_fraction
```

Use multiwindow burn-rate alerts tailored to policy and traffic volume, plus fast paging for complete outages and GPU hardware faults. Avoid alerting on low sample counts without confidence. Confirm paging routes, silences, escalation, runbook URLs and synthetic checks. Telemetry absence is an alertable condition; missing data must never be interpreted as zero errors.

## 4. Metrics discovery and label hygiene

Inventory available Prometheus targets and metric families from **deployed** vLLM, gateway, kube-state-metrics, node exporter, DCGM and TrueFoundry-supported integration. Document metric name, type, unit, labels, scrape interval, owner and version. Never copy metric names from a different vLLM release without verification.

Keep model revision, environment, cluster, zone and approved tenant class as bounded-cardinality labels. Do not label raw prompts, user IDs, request IDs, token text or secrets. Traces may use request IDs with controlled access and retention, not as Prometheus labels.

## 5. Read-only diagnostics

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get deployments,pods -A -o wide
kubectl get svc,endpointslices -A
kubectl get hpa,pdb,resourcequota -A
kubectl get events -A --sort-by=.lastTimestamp | tail -80
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=20m --tail=100
```

Use approved GPU dashboards/diagnostic Pods for `nvidia-smi`; don't exec into production model Pods without authorization. Review DCGM exporter freshness and XID/ECC, VRAM, GPU utilization, power/thermal signals where supported.

## 6. Workload model and baseline

Document model weights/tokenizer/revision, GPU SKU, vLLM/CUDA versions, prompt length p50/p95/p99, output token lengths, streaming ratio, tenant mix, request arrival distribution, concurrency, batch behavior, context limits and warm/cold startup. Benchmark prefill-heavy and decode-heavy cohorts separately. Measure TTFT, TPOT, E2E latency, output correctness, errors, queue wait, tokens/sec, GPU utilization, VRAM, KV cache and cost. Use only synthetic test vectors.

## 7. Safe load test stages

1. **Baseline:** single client and low concurrency; prove output contract and complete SSE.
2. **Ramp:** bounded step increases with observation and cooldown; monitor queue, VRAM and errors.
3. **Steady:** representative arrival rate and token distribution across approved stability window.
4. **Burst:** short approved burst to assess rate limits and queuing, not uncontrolled stress.
5. **N-1 tabletop or approved staging failover:** prove remaining capacity and recovery.
6. **Soak:** approved duration, watch leaks, restarts, thermal and cost.

**Abort immediately** on unauthorized traffic, data leakage, unexpected production route, sustained severe 5xx, GPU XID/ECC, repeated CUDA OOM, budget breach, uncontrolled scaling or alert/pager failure. Record test controller settings and cloud spend cap before starting.

## 8. Capacity planning

```text
offered_input_tokens_per_s  = request_rate × mean_input_tokens
offered_output_tokens_per_s = request_rate × mean_output_tokens
required_GPUs = ceil(peak_demand / measured_safe_per_GPU_capacity)
rollout_peak = steady_GPU + surge_GPU
spare_after_N1 = schedulable_GPU_after_failure - required_GPUs
```

These are estimates; measured safe throughput must respect p95 TTFT/TPOT and streaming completion. GPU capacity is constrained by VRAM, KV cache, scheduling, batch size, model context, tensor parallelism, CPU/network and cloud GPU availability. Verify actual node packing, topology, quota, GPU-hours, zone spread and warmup delay. HPA/KEDA/custom metrics and cluster autoscaling may be used where supported and approved; scaling cannot solve GPU quota shortages or cold-start latency automatically.

## 9. Dashboard specification

**Executive:** availability, error budget, TTFT/TPOT p95, complete streams, active incidents, cost/1k successful output tokens.

**SRE:** 429/5xx, gateway timeouts, queue, tokens/sec, KV cache, CUDA/host OOM, Pod readiness, restarts, HPA, Pending, agent reconciliation, GPU allocatable, DCGM XID/ECC, VRAM, thermal, telemetry freshness.

**Release:** canary versus stable revisions, rollout duration, model warmup, rollback history, Git/TrueFoundry desired/observed drift, synthetic SLO, GPU surge.

## 10. Troubleshooting decision tree

```text
SLO breach?
 ├─ 401/403 → auth/tenant/TLS/secret rotation
 ├─ 429 → gateway rate limit vs model queue saturation
 ├─ 5xx/timeout → ingress, EndpointSlice, vLLM crash/oom
 ├─ TTFT high → prefill, queue, cold start, gateway
 ├─ TPOT high → decode saturation, throttling, GPU
 ├─ SSE truncated → buffering, timeout, drain
 ├─ GPU low but queue high → CPU/network/scheduler bottleneck
 ├─ GPU high + KV pressure → concurrency/context/VRAM
 ├─ metrics absent → exporter, scrape, RBAC, cardinality
 └─ cost/token spike → idle warm GPUs, retries, inefficiency
```

## 11. Eighteen operational runbooks

1. SLO definition and eligibility review.
2. Customer-path synthetic probe.
3. Streaming completion and timeout.
4. TTFT/TPOT cohort analysis.
5. Gateway 429/5xx investigation.
6. vLLM queue and KV cache.
7. GPU/DCGM hardware telemetry.
8. Kubernetes scheduling and autoscaler.
9. TrueFoundry reconciliation health.
10. Prometheus target/metric discovery.
11. Logs/traces correlation and redaction.
12. Burn-rate alerts and pager drill.
13. Load-test approval and guardrails.
14. Capacity N-1 and rollout overlap.
15. Model cold start and warmup.
16. Canary SLO and rollback.
17. GPU cost/token regression.
18. Acceptance evidence and handoff.

## 12. Forty-eight failure scenarios

| Domain | Scenarios |
|---|---|
| SLO | 01 eligibility wrong; 02 low samples; 03 burn-rate wrong; 04 synthetic missing; 05 p95 misleading; 06 missing stream completion |
| Gateway | 07 401; 08 403; 09 429; 10 502; 11 503; 12 504 |
| vLLM | 13 TTFT high; 14 TPOT high; 15 queue spike; 16 KV pressure; 17 CUDA OOM; 18 output contract drift |
| GPU | 19 allocatable zero; 20 plugin stale; 21 DCGM missing; 22 XID; 23 ECC; 24 thermal throttle |
| Kubernetes | 25 Pending; 26 CrashLoop; 27 OOMKilled; 28 HPA stale; 29 autoscaler quota; 30 rollout surge shortfall |
| Telemetry | 31 scrape down; 32 cardinality blowup; 33 logs missing; 34 trace sampling wrong; 35 alerts silenced; 36 paging misrouted |
| Capacity | 37 N-1 fail; 38 cold start; 39 burst saturation; 40 soak leak; 41 noisy neighbor; 42 zone GPU shortage |
| Governance | 43 unapproved load; 44 PHI in logs; 45 budget exceeded; 46 drift; 47 rollback absent; 48 no sign-off |

## 13. Six staging game days

1. TTFT regression with prefill-heavy load.
2. TPOT regression with decode-heavy load.
3. SSE truncation and gateway timeout.
4. GPU XID/ECC telemetry and failover tabletop.
5. Missing Prometheus target plus alert/pager drill.
6. N-1 capacity and failed canary rollback under cost cap.

Record UTC timeline, synthetic traffic profile, approved load, expected/actual SLO, GPU-hours, rollback, alerts and reviewer.

## 14. Fourteen acceptance gates

G1 SLO/eligibility approved; G2 synthetic customer path; G3 complete SSE; G4 TTFT/TPOT cohorts; G5 gateway error/queue; G6 vLLM/KV telemetry; G7 GPU/DCGM freshness; G8 Kubernetes/agent visibility; G9 Prometheus/logs/traces; G10 burn-rate/paging; G11 baseline/ramp/steady/burst; G12 N-1/surge/rollback; G13 cost/token and budget; G14 reviewed evidence and sign-off.

See [companion lab](../labs/05-observability-slos-load-capacity-validation-check.md). All gates pending until executed and independently reviewed.

**Next:** Chapter 15.6 — Failure Injection, Incident Response and Recovery.
