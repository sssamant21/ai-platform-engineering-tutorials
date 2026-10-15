# 7.6 — vLLM Performance Tuning, Observability and Troubleshooting

**Part:** 07 — vLLM & LLM Serving · **Chapter:** 6 of 7  
**Audience:** GPU SREs, TrueFoundry platform engineers, inference performance engineers  
**Safety:** **[SAFE-READ]** observation; **[LAB-MUTATE]** isolated approved nonproduction; **[CHANGE-CONTROL]** production changes.

## 1. Learning objectives

Establish a repeatable vLLM performance baseline; separate time-to-first-token (TTFT), inter-token latency (ITL), end-to-end latency, throughput and availability; correlate request queue, prefill/decode, KV cache, GPU and gateway signals; design actionable Prometheus/Grafana observability; tune model-serving capacity without exceeding SLO or error budgets; and recover regressions through the authoritative TrueFoundry/GitOps release.

**Prerequisites:** Chapters 7.1–7.5, an approved TrueFoundry workspace and cluster, a pinned vLLM/model release, access to dashboards and a synthetic workload. Never run uncontrolled load against production or log real prompts and secrets.

## 2. Performance is a multi-dimensional objective

```text
Client / SDK → TLS/auth gateway → Service / Ready endpoints
                                       ↓
                                 vLLM frontend
                                       ↓
                         admission + request queue
                                       ↓
                  tokenizer → prefill → first output token
                                       ↓
                     decode / KV cache / GPU executor
                                       ↓
                      stream completion / response
                                       ↓
             metrics, logs, traces, GPU/DCGM, alerts
```

An increase in tokens per second can coincide with worse P99 TTFT or ITL. Optimize against an **explicit workload and SLO**, not a single maximum-throughput number.

## 3. Define a representative workload

Capture these dimensions without storing sensitive content:

| Dimension | Example bands | Why |
|---|---|---|
| Input tokens | short / medium / long | Prefill cost |
| Output tokens | short / medium / long | Decode cost |
| Concurrency | low / moderate / high | Queue and KV occupancy |
| Arrival pattern | steady / bursty | Admission behavior |
| API | chat / completions / streaming | Response and timeout contract |
| Model | pinned architecture, revision, dtype | Reproducibility |
| Runtime | pinned vLLM image, GPU SKU, TP/PP | Comparability |
| Cache | cold / warm, prefix reuse enabled? | Latency differences |
| Network | internal gateway path | Client-experienced latency |

Keep prompt/output distributions stable across A/B comparisons. Record whether samples are measured at client, gateway or engine; do not combine incomparable histograms.

## 4. Service level indicators

| SLI | Definition | Why it matters |
|---|---|---|
| TTFT | Request accepted to first generated token (define instrumentation boundary) | Perceived responsiveness |
| ITL / TPOT | Time between output tokens; exact aggregation varies | Streaming smoothness |
| E2E latency | Client send to complete response | Total user experience |
| Prompt tokens/s | Processed input tokens over time | Prefill capacity |
| Output tokens/s | Generated tokens over time | Decode capacity |
| Running/waiting | Executing versus queued requests | Saturation |
| KV-cache occupancy | Active KV allocation relative to capacity | Memory pressure |
| Preemption/recompute | Scheduler displacement or repeated work | Cache pressure |
| Error ratio | 429, 5xx, model failures and stream failures separately | Availability |
| GPU utilization/VRAM | Device pressure, allocation and headroom | Hardware context |

**Set numerical SLO targets with product owners from measured baselines.** Illustrative objectives such as “P95 TTFT below X seconds” are placeholders until approved. A completed HTTP 200 header does not prove streaming completion.

## 5. Instrumentation architecture

```text
vLLM metrics endpoint (if enabled/supported) ──┐
TrueFoundry deployment/service signals ────────┤
Kubernetes kube-state-metrics/events ──────────┤→ Prometheus → Grafana / alert rules
GPU DCGM exporter / node metrics ──────────────┤
Gateway metrics and tracing ────────────────────┘
                logs / traces → approved central pipeline
```

Actual vLLM metric names, histogram types and labels differ by engine release. **Discover them from the running, authorized deployment**; do not paste unverified metric names into production alert rules. Keep model revision and endpoint labels low cardinality. Exclude prompt text, authorization headers, personal data and request IDs from metric labels.

## 6. Metric discovery [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> logs <pod> --tail=200
```

If an **authorized metrics endpoint** is exposed by the installed release, inspect it through the approved monitoring path; do not publicly expose an internal metrics port or bypass authentication. Identify metric type (counter, gauge, classic/native histogram), units, reset behavior and labels before writing PromQL.

Where an authorized GPU runtime includes the tool:

```bash
kubectl -n <namespace> exec <pod> -c <container> -- nvidia-smi
```

Missing metrics are an observability gap, not proof of a healthy engine.

## 7. PromQL patterns — adapt to discovered metric names

The following are **templates**. Replace `<...>` with real validated series/labels; do not run literally.

**Request rate** (counter):

```promql
sum(rate(<completed_request_counter>[5m]))
```

**Error ratio** (counter with validated status labels):

```promql
sum(rate(<request_counter>{<error_selector>}[5m]))
/
clamp_min(sum(rate(<request_counter>[5m])), 1e-9)
```

**TTFT P95** (classic histogram only, if available):

```promql
histogram_quantile(
  0.95,
  sum by (le, model) (rate(<ttft_histogram_bucket>[5m]))
)
```

**Queue depth** (gauge):

```promql
sum by (model) (<waiting_request_gauge>)
```

Never calculate histogram quantiles from an arbitrary gauge or mix incompatible buckets/units. For native histograms, use version-appropriate PromQL. Use separate panels for engine TTFT, gateway latency and client E2E.

## 8. Grafana dashboard blueprint

| Panel | Visual | Correlate with |
|---|---|---|
| SLO overview | TTFT/ITL/E2E P50/P95/P99 | Release annotations |
| Requests/errors | RPS, 429, 5xx, stream failures | Queue |
| Prompt/output throughput | Tokens/s | GPU use |
| Admission pressure | Running/waiting requests | TTFT |
| KV-cache pressure | Cache utilization/preemptions | ITL, OOM |
| GPU capacity | VRAM, utilization, ECC/Xid where available | Device errors |
| Kubernetes health | Pending, restarts, Ready endpoints | Rollouts |
| Model lifecycle | Artifact fetch/load/warmup time | Cold starts |
| Gateway/network | 401/403/502/503/504, retries, timeouts | Client impact |

Annotate image digest, vLLM version, model/tokenizer revision, scheduler settings and deployment timestamp. Do not put raw prompt payloads in dashboards.

## 9. Alerting and escalation

Prefer actionable, sustained signals and defined owners:

| Alert | Condition concept | First action |
|---|---|---|
| No Ready serving endpoints | Zero ready backends for agreed duration | Check startup, Pods and Service |
| High error ratio | Sustained 5xx or stream failure above SLO | Check release/gateway/runtime |
| TTFT SLO breach | Sustained P95/P99 violation | Queue, prefill, prompt mix |
| ITL SLO breach | Sustained streaming degradation | Decode, KV and GPU |
| Queue saturation | Waiting requests rise with no drain | Admission limits and capacity |
| KV pressure | High occupancy + preemption/error | Concurrency/context budgets |
| GPU health fault | Device telemetry errors | Node/GPU owner |
| Cold-start regression | Load/warmup duration exceeds baseline | Artifact storage and startup |

Use rate windows, minimum traffic floors and burn-rate/error-budget policy. Avoid paging on isolated GPU utilization or a single request outlier.

## 10. Performance tuning levers

| Lever | Potential benefit | Risk / validation |
|---|---|---|
| Model precision/quantization | Lower weight footprint | Accuracy, kernel support |
| Context limit | Bound KV capacity | Truncation/rejection contract |
| Output token limit | Bound decode work | Client functionality |
| Concurrent sequences | Increase throughput | KV OOM, TTFT/ITL regression |
| Batched-token budget | Better prefill scheduling | Peak VRAM and fairness |
| Chunked prefill (if supported) | Reduce long-prefill blocking | Version-dependent behavior |
| Prefix caching (if supported) | Reuse common prefixes | Cache correctness/security |
| Tensor parallelism | Fit large models | NCCL overhead and topology |
| GPU memory utilization setting | Adjust engine allocation budget | Starvation of other buffers |
| Autoscaling/replicas | Absorb demand | GPU availability and cold starts |
| Client retry/backoff | Protect saturated endpoint | Retry amplification |

Every change requires a hypothesis, pinned A/B releases, representative workload, rollback trigger and measured SLO comparison. Never assume a vLLM CLI flag exists in every release; consult installed-version documentation.

## 11. Controlled A/B tuning methodology [LAB-MUTATE]

1. **Baseline A:** record model, image, GPU, TP, context, concurrency, prompt/output mix and SLO.
2. **Hypothesis:** change **one** supported tuning dimension, with expected impact and risk.
3. **Capacity check:** verify spare GPUs, VRAM headroom, quotas and rollback readiness.
4. **Candidate B:** deploy through the supported TrueFoundry workflow in isolated nonproduction.
5. **Warmup:** wait for model-aware readiness and golden synthetic inference.
6. **Bounded test:** fixed duration, concurrency ceiling, synthetic inputs and stop conditions.
7. **Compare:** P95/P99 TTFT, ITL, E2E, output tokens/s, queue, KV, preemptions, errors, GPU.
8. **Decision:** accept only if required SLOs and correctness remain satisfied.
9. **Rollback/cleanup:** restore approved A and verify actual loaded revision and baseline health.

Do not compare a cold A to a warm B, or different prompt distributions, without explicitly accounting for those differences.

## 12. Troubleshooting matrix

| ID | Symptom | First failed transition | Check |
|---|---|---|---|
| P1 | TTFT high, queue high | Admission → prefill | Arrival/concurrency |
| P2 | TTFT high, GPU idle | CPU/tokenizer → executor | CPU and tokenizer |
| P3 | TTFT high, long prompts | Prefill → first token | Prompt distribution |
| P4 | ITL high, GPU saturated | Decode → GPU | Active sequences |
| P5 | ITL high, KV near full | Cache → decode | Preemption/cache |
| P6 | Tokens/s low, GPU busy | Executor → useful output | Kernels/model |
| P7 | Tokens/s high, P99 poor | Scheduler → tail latency | Fairness and batching |
| P8 | Queue never drains | Admission → capacity | Limits/autoscaling |
| P9 | 429 burst | Client → admission | Quotas/backoff |
| P10 | 503/no ready endpoints | Engine → Service | Probes/EndpointSlice |
| P11 | 504 under load | Queue/runtime → gateway deadline | Timeouts |
| P12 | Stream stops after 200 | Runtime → client | Drain/cancel |
| P13 | GPU OOM at startup | Weights → VRAM | Model footprint |
| P14 | GPU OOM only at concurrency | KV → VRAM | Live tokens |
| P15 | Latency regression after upgrade | Release → engine | Version diff |
| P16 | NCCL throughput regression | Worker → interconnect | TP topology |
| P17 | Missing TTFT metric | Engine → telemetry | Exporter/series |
| P18 | Wrong histogram percentile | Metrics → query | Buckets/labels |
| P19 | Dashboard cardinality explosion | Labels → TSDB | High-cardinality fields |
| P20 | Wrong model after rollback | Alias → runtime | Immutable revision |

For each, document **Lowest Proven Healthy Layer (LPHL)**, **First Failed Transition (FFT)**, evidence, owner, safe mitigation, rollback and validation.

## 13. Production incident runbook

1. **Declare impact:** endpoint/model, affected clients, onset, TTFT/ITL/E2E, 429/5xx, stream failures and error budget.
2. **Correlate:** image digest, model revision, vLLM version, GPU node, scheduler settings, rollout time and traffic mix.
3. **Localize:** gateway → Ready endpoints → admission → tokenizer → prefill → decode → KV cache → GPU/TP → stream.
4. **Contain:** freeze promotion, protect known-good replicas, enforce approved request limits and client backoff.
5. **Repair:** address the first confirmed failed transition; do not disable auth or restart shared GPU nodes to hide saturation.
6. **Rollback:** restore compatible image/model/tokenizer/runtime/scheduler configuration through TrueFoundry/GitOps.
7. **Validate:** actual model revision, Ready endpoints, synthetic golden request, TTFT/ITL, queue, KV headroom, errors and stream completion.
8. **Review:** incident timeline, LPHL/FFT, detection gap, action owners and regression test.

## 14. Acceptance gates

| Gate | Required evidence |
|---|---|
| A1 | Representative workload and immutable baseline |
| A2 | TTFT/ITL/E2E/throughput definitions and SLO |
| A3 | Metrics types, units and PromQL validated |
| A4 | Dashboard and release annotations designed |
| A5 | Actionable alerts and owners defined |
| A6 | Controlled A/B tuning or clearly marked tabletop |
| A7 | P1–P20 failures classified by LPHL/FFT |
| A8 | Authoritative rollback and post-recovery checks |

## 15. Evidence record

```text
Date/time, owner, change approval:
TrueFoundry workspace / cluster / endpoint:
Image digest, vLLM version, model/tokenizer revision:
GPU SKU/count, TP, context, concurrency:
Synthetic workload and arrival distribution:
TTFT P50/P95/P99, ITL, E2E:
Prompt/output tokens/s:
Running/waiting, KV occupancy, preemptions:
GPU utilization/VRAM, 429/5xx/stream failures:
Prometheus series and histogram validation:
Dashboard and alert owners:
Lowest Proven Healthy Layer:
First Failed Transition:
A/B decision, rollback release and post-recovery SLO:
```

**Hands-on lab:** [7.6 — vLLM Performance Tuning and Observability Check](../labs/06-vllm-performance-tuning-observability-troubleshooting-check.md)  
**Next:** Chapter 7.7 — Production vLLM Deployment and Recovery Capstone.

## References

- [vLLM Documentation](https://docs.vllm.ai/)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Prometheus Histograms](https://prometheus.io/docs/practices/histograms/)
- [Prometheus Alerting Practices](https://prometheus.io/docs/practices/alerting/)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)
