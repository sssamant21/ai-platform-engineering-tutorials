# Chapter 073 — Load, Stress, Soak and Failure Testing

**Part 09 — Performance Engineering and Capacity · Chapter 073 of 100**  
**Publication status:** Chapter and lab published. Live execution and technical/security review pending.

## Learning objectives
Design and interpret baseline load, ramp, stress, spike, soak and failure-injection tests for LLM inference. Apply open-loop arrivals, workload cohorts, SLO goodput, stop conditions, N-1 failure tests and reproducible performance evidence. Distinguish an offline synthetic exercise from production-like GPU benchmarking.

## 1. Testing taxonomy
| Test | Question | Example (illustrative) |
|---|---|---|
| Baseline | What is normal performance? | 10 RPS for 15 minutes |
| Load | Can the system meet expected demand? | 20 RPS for 30 minutes |
| Ramp | Where does performance degrade? | +5 RPS every 5 minutes |
| Stress | Where is the saturation knee? | increase until SLO or safety stop |
| Spike | Can admission handle sudden bursts? | 4× normal load for 60 seconds |
| Soak | Are there leaks and cumulative effects? | 6 hours at safe load |
| Failure | Does N-1 remain within limits? | terminate one staging replica |
| Recovery | How quickly does goodput return? | restore replica and observe |

All values must be replaced with service-specific approved limits. Never run disruptive tests against production without explicit authorization and a coordinated change window.

## 2. Benchmark architecture
```text
approved synthetic cohort dataset
             |
open-loop load driver -----> offered arrivals + timestamps
             |
gateway quotas / admission ----> 429 / 503 / timeout
             |
scheduler queue ---> prefill / decode GPU workers
             |                    |
             +---- OTel traces ----+---- GPU and KV metrics
                                  |
                      TTFT / TPOT / E2E / goodput
                                  |
                     SLO, error and spend stop gates
```
A closed-loop driver waits for responses before sending the next request; this can conceal overload through **coordinated omission**. For capacity experiments, track offered load independently from successful completions and use open-loop arrivals with explicit caps. A benchmark client that saturates CPU or network invalidates conclusions about server capacity.

## 3. Workload cohort design
Reuse Chapters 066–072:
- Interactive: short input and output, strict TTFT.
- Retrieval: long input, moderate output, prefill pressure.
- Agentic: multiple tool rounds, bursty concurrency and structured output.
- Decode-heavy: short input, long output, TPOT sensitivity.
- Long context: KV-cache occupancy, eviction and OOM sensitivity.
- Mixed production-like: representative weighted cohorts with approved synthetic data.

Pin tokenizer, model digest, quantization, context limits, runtime, GPU SKU, worker count, scheduler knobs, traffic mix, region and network path. Keep prompt diversity realistic without exposing customer data.

## 4. Measurement contract
Capture attempted/offered, admitted, completed, timed-out, rejected and cancelled requests; distinguish completed from SLO-compliant **good** requests. Measure p50/p95/p99 TTFT, TPOT and E2E; queue depth/age, active sequences, KV occupancy, GPU utilization/memory, retries, output token rate and cost per million good tokens. Report per-cohort and aggregate results with denominators.

A successful HTTP status alone does not prove full stream completion. Partial streams, late completions and timeouts count according to a documented SLI contract. Never exclude failed or timed-out requests from the error denominator.

## 5. Reproducible procedure
1. **Safety approval:** staging namespace, budget cap, test owner, on-call contact, abort authority and rollback plan.
2. **Baseline:** pin deployment and collect idle, warmed and steady-state measurements.
3. **Load:** use open-loop arrivals; verify driver can sustain offered load and timestamps are monotonic.
4. **Ramp/stress:** increase load gradually; stop at predeclared p95 latency, error, OOM, queue age or cost thresholds.
5. **Spike:** inject bounded burst; observe admission/rejection and recovery, not just peak throughput.
6. **Soak:** run approved sustained load; inspect memory growth, GPU allocation, connection leaks, queue drift and telemetry cardinality.
7. **Failure:** remove one replica or inject approved network/storage impairment in staging; validate N-1 and fallback behavior.
8. **Recovery:** restore capacity, observe time-to-healthy, goodput and tail latency; preserve immutable artifacts and configs.
9. **Acceptance:** compare results to explicit SLO, cost and safety gates; obtain independent review.

## 6. Performance interpretation
- Rising TTFT with stable TPOT suggests queue/prefill pressure.
- Rising TPOT with stable prompt cohorts suggests decode contention, KV residency or transport.
- Throughput flattening while offered load grows suggests saturation or admission cap.
- Rising queue age with stable GPU utilization may indicate routing or worker bottlenecks.
- Soak-only failures suggest leaks, cache fragmentation, resource churn or telemetry overhead.
- Fast recovery of replica readiness without restored goodput suggests hidden queue backlog or lazy warmup.

These are hypotheses, not root-cause proof; corroborate with traces and component metrics.

## 7. Troubleshooting matrix
| Failure | Inspect | Safe first response |
|---|---|---|
| Load driver under-delivers | driver CPU, event loop, sockets | reduce offered rate and fix driver |
| High 429/503 | admission, quota, replicas | stop ramp and check policy |
| GPU OOM | KV blocks, context mix, active sequences | abort test; preserve evidence |
| Soak memory creep | RSS, GPU memory, allocator, logs | end run at stop threshold |
| Retry storm | client attempts, jitter, backoff | disable unapproved retries |
| TTFT p99 cliff | queue age, long prefill, scheduling | inspect cohorts and revert |
| Failure test breaks availability | N-1 headroom, warm pool | restore replica, rollback |
| Invalid goodput metric | partial streams, missing tokens | reject acceptance evidence |

## 8. Production runbooks

### RB-073-A — Load-test abort
**[SAFE-READ]** identify breached guardrail, offered load, SLO burn, GPU OOM, queue age and cost. **[LAB-MUTATE]** stop synthetic driver. **[CHANGE-CONTROL]** restore staging capacity and route state. Preserve sanitized traces and incident timeline.

### RB-073-B — Soak degradation
Compare hourly trends in memory, queue age, active sequences, cache occupancy, p95 TPOT and restart counts. Isolate leak hypotheses with fixed cohort mix and a shorter reproduction. Avoid blindly restarting production workers.

### RB-073-C — N-1 failure drill
Preflight minimum replicas, failover readiness, rollback and abort authority. Remove one approved staging replica; record goodput, recovery, rejected requests and failover time. Restore and verify before sign-off.

### RB-073-D — Capacity acceptance
Publish raw offered-load schedule, observed completion/goodput curves, SLO gates, failure conditions, resource graphs, costs and confidence limits. Do not infer production capacity from synthetic simulations.

## 9. Security and release gates
- Use synthetic, non-sensitive prompts and signed immutable model artifacts.
- Enforce rate, token, spend, duration and GPU-resource caps.
- Disable unsafe automatic retries and test only authorized endpoints.
- Keep metric labels low cardinality; no prompts, completions, API keys or tenant identifiers.
- [ ] Workload cohorts and arrival schedule documented.
- [ ] Offered versus completed accounting verified.
- [ ] SLO and abort thresholds fixed before testing.
- [ ] Spike, soak, failure and recovery evidence recorded.
- [ ] Rollback and cost headroom reviewed.
- [ ] Technical/security review and live execution independently tracked.

## 10. Hands-on lab
The [Chapter 073 executable lab](../labs/073-load-stress-soak-failure-testing-check.md) provides a deterministic **offline** benchmark harness for load schedules, queue overload, replica loss, SLO goodput and abort gates. It is not a GPU or inference-engine simulator.

## References
- [Google SRE Workbook — Handling Overload](https://sre.google/workbook/handling-overload/)
- [Google SRE Book — Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)
- [vLLM benchmarking documentation](https://docs.vllm.ai/en/latest/benchmarking/)
- [OpenTelemetry documentation](https://opentelemetry.io/docs/)

**Next:** Chapter 074 — Capacity Forecasting, N-1 and Rollout Headroom.
