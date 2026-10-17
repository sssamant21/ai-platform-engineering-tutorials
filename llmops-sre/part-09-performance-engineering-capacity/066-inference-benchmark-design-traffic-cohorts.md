# Chapter 066 — Inference Benchmark Design and Traffic Cohorts

**Part 09 — Performance Engineering and Capacity · Chapter 066 of 100**  
**Status:** Published tutorial and lab. Technical/security review and live execution pending.

## Learning objectives
- Define a reproducible, user-centered LLM inference benchmark.
- Separate prefill, decode, queueing, gateway and network contributions to latency.
- Design traffic cohorts for prompt length, output length, concurrency, arrival pattern, streaming and model configuration.
- Calculate throughput, TTFT, TPOT, end-to-end latency, goodput and SLO attainment.
- Prevent benchmark bias, data leakage, unrealistic closed-loop load and accidental production saturation.
- Implement and test a local synthetic workload generator and analysis pipeline.

## 1. Performance architecture
```text
                 load driver / cohort planner
                       |
              bounded arrival schedule
                       |
                 gateway / router
                       |
           admission / queue / batching
                       |
          model prefill -> token decode
                       |
            stream completion / errors
                       |
           OTEL + metrics + safe logs
                       |
          per-cohort benchmark analyzer
                       |
        capacity curve / SLO / cost report
```

A valid benchmark measures **user-visible** behavior at a defined boundary. Device utilization alone is not throughput; tokens per second alone is not acceptable latency; aggregate p95 can hide a failing cohort.

## 2. Core terminology
| Metric | Definition | Caveat |
|---|---|---|
| TTFT | request arrival to first generated token | includes queue, network and prefill at the chosen boundary |
| TPOT | average interval per generated token after first token | use `(last_token_time-first_token_time)/(output_tokens-1)` when output_tokens > 1 |
| End-to-end latency | arrival to completed response | includes admission and streaming completion |
| Output throughput | generated output tokens / measurement seconds | distinguish completed-only and attempted requests |
| Request throughput | completed requests / measurement seconds | also report offered request rate |
| Goodput | completed requests satisfying all defined SLO thresholds per second | thresholds must be fixed before test |
| Error ratio | bad eligible requests / eligible requests | streaming partial failures count by policy |
| Concurrency | simultaneous in-flight requests | not identical to requests/sec |

**Time boundaries matter.** Use monotonic clocks for local duration measurement, synchronized timestamps for distributed tracing, and a consistent definition of token event timing. TPOT is undefined for 0–1 output tokens. TTFT may be unavailable for failures before the first token.

## 3. Cohort design
| Dimension | Example cohorts | Why it matters |
|---|---|---|
| Input tokens | 128, 1,024, 8,192 | prefill and KV-cache pressure |
| Output tokens | 32, 256, 1,024 | decode duration and GPU occupancy |
| Arrival model | constant, Poisson, burst | queueing and tail latency |
| Concurrency | 1, 8, 32, 128 | saturation and batching |
| Traffic shape | steady, ramp, spike, soak | stability and autoscaling |
| Model/runtime | pinned model digest, tokenizer, quantization, runtime build | reproducibility |
| Routing | single replica, multi-replica, regional | scheduling and network impact |
| Streaming | streaming vs buffered | first-token experience |

Cohorts are examples, **not** recommended production settings. Choose based on measured real-world distributions using privacy-preserving aggregate token counts, not customer prompt text. Include rare but expensive long-context requests. Stratify or weight cohorts by intended traffic share; disclose weights.

## 4. Experimental controls
- Fix model artifact digest, tokenizer revision, serving runtime, driver version, GPU model/count, MIG configuration, batch/sequence limits, KV-cache policy, autoscaling, region and network path.
- Warm up deliberately, then use a measured steady-state window; report cold-start separately.
- Record offered load, achieved load, errors, timeouts, retries and queue growth. **Coordinated omission** occurs when a closed-loop client waits for completion before sending more work, under-sampling the worst periods. Use independently scheduled open-loop arrivals where appropriate.
- Isolate load driver bottlenecks from serving bottlenecks. Monitor client CPU, socket limits and event-loop lag.
- Run repetitions and report distributions/confidence intervals; avoid comparing single p95 samples from different traffic mixes.
- Set hard request, concurrency, duration, token and spend caps before running against any real endpoint.

## 5. Capacity curve and knee point
Sweep offered load while holding cohort mix constant. Plot throughput, p95 TTFT, p95 end-to-end latency, error ratio and queue depth. The **knee** is the region where throughput gains flatten while latency and errors accelerate. Select an operating target below this point with headroom for failures, autoscaling delay and traffic bursts. Benchmark both typical and degraded capacity (e.g. one replica unavailable).

### Illustrative capacity acceptance gates
- TTFT p95 ≤ 2 seconds for short prompts; long-context cohorts may have different approved objectives.
- End-to-end p95 ≤ approved cohort budget.
- Error ratio ≤ approved SLO.
- Queue length stable during steady state; no unbounded growth.
- Goodput meets required peak forecast with resilience margin.

These are sample gates only; product owners must approve production thresholds.

## 6. Executable lab
The [Chapter 066 companion lab](../labs/066-inference-benchmark-design-traffic-cohorts-check.md) generates deterministic synthetic request cohorts, simulates arrival/completion and computes per-cohort throughput, p95 TTFT, p95 latency, error ratio and goodput. No GPU, external model, customer prompt or cloud key is required.

## 7. Troubleshooting and incident scenarios
1. **High TTFT, normal TPOT:** inspect queue wait, admission, prefill, long prompts, network buffering.
2. **Normal TTFT, poor TPOT:** inspect decode scheduling, KV-cache pressure, GPU utilization and competing sequences.
3. **High throughput but low goodput:** check timeouts, streaming failures and cohort-specific latency SLOs.
4. **Benchmark unexpectedly improves under load:** investigate warm cache, prompt duplication, load-generator bottleneck and coordinated omission.
5. **Long-context cohort collapses:** inspect memory, KV cache, batching limits and tokenization drift.
6. **Tail latency spikes after scaling:** examine replica cold starts, model load time and uneven routing.
7. **Cost per good request increases:** compare GPU-hours, token mix, retries and utilization at matched SLO targets.

## 8. Production runbooks

### RB-066-A — Benchmark preparation
**[SAFE-READ]** capture approved environment inventory and baseline, verify pinned versions, select synthetic/approved dataset, define load caps and stop conditions. **[CHANGE-CONTROL]** request a staging window and notify service owners before generating load.

### RB-066-B — Saturation during benchmark
Immediately stop load at preapproved queue, error, latency or spend thresholds; inspect worker health and ongoing customer impact. Do not expand capacity or bypass quotas without approval. Preserve sanitized evidence.

### RB-066-C — Regression investigation
Compare identical cohorts, hardware, model digest and runtime; inspect TTFT/TPOT split, queue and error metrics. Bisect deployment changes and repeat tests before concluding regression.

### RB-066-D — Capacity sign-off
Record forecast peak, safe operating throughput, replica loss margin, cost/good-request, autoscaling lag, limitations and approval owner. Treat synthetic lab output as an educational baseline, not production capacity proof.

## 9. Security, FinOps and acceptance
- No real prompt/response content in benchmark logs; synthetic inputs or approved de-identified corpora only.
- Protect test identities and model endpoints; enforce quotas and termination conditions.
- Attribute benchmark GPU-hours and token cost; disclose retries and wasted work.
- [ ] Cohorts, weights and arrival models documented.
- [ ] TTFT, TPOT, latency, throughput and goodput definitions verified.
- [ ] Deterministic tests pass and cohort-level reporting is reproducible.
- [ ] Load caps, stop criteria and data privacy reviewed.
- [ ] Technical/security review and live execution tracked separately.

## 10. References
- [vLLM benchmarking documentation](https://docs.vllm.ai/en/latest/benchmarking/)
- [NVIDIA Triton performance analyzer](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/perf_analyzer/README.html)
- [OpenTelemetry semantic conventions](https://opentelemetry.io/docs/specs/semconv/)
- [Google SRE Workbook — Load Balancing and Capacity Planning](https://sre.google/workbook/load-balancing/)

**Next:** Chapter 067 — Prefill, Decode and Token-Level Latency Profiling.
