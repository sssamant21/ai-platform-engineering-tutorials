# Chapter 067 — Prefill-Heavy Versus Decode-Heavy Workloads

**Part 09 — Performance Engineering and Capacity · Chapter 067 of 100**  
**Publication:** Tutorial and lab published; live execution, technical review and security review pending.

## Learning objectives
- Explain transformer prefill and autoregressive decode phases and their distinct bottlenecks.
- Profile time to first token (TTFT), time per output token (TPOT), inter-token latency (ITL), queue wait and end-to-end completion.
- Compare long-prompt/short-output and short-prompt/long-output traffic at equal request rate and token load.
- Diagnose KV-cache pressure, batching, scheduler contention and latency outliers.
- Construct reproducible synthetic experiments, tests, acceptance gates and production runbooks.

## 1. Inference request lifecycle
```text
client request
    |
gateway admission / auth / routing
    |
scheduler queue ------> wait / reject / retry
    |
PREFILL: process prompt tokens, initialize attention KV cache
    |
first generated token (TTFT)
    |
DECODE: repeatedly generate next token using prior KV cache
    |
last token / finish reason / stream completion
    |
client acknowledgment and SLI accounting
```

**Prefill** processes input tokens and builds KV state; it often exhibits high arithmetic intensity and is sensitive to prompt length, batch shape and memory. **Decode** generates tokens sequentially and often becomes memory-bandwidth/KV-cache limited, but actual bottlenecks depend on architecture, hardware, kernel, batch size and speculative decoding. Continuous batching interleaves phases; phase separation is a conceptual model, not necessarily separate physical workers.

## 2. Cohort taxonomy
| Cohort | Example input/output tokens | Dominant stress | Investigate |
|---|---|---|---|
| Prefill-heavy | 8,192 / 32 | prompt processing, first-token delay | TTFT, prefill tokens/sec, GPU compute |
| Decode-heavy | 128 / 1,024 | sustained generation, KV residency | TPOT, output tokens/sec, active sequences |
| Balanced | 1,024 / 256 | mixed scheduling | TTFT, TPOT, p95 end-to-end |
| Long-context + long-output | 16,384 / 1,024 | KV capacity, memory fragmentation | OOM, eviction, admission |
| Short interactive | 128 / 64 | per-request overhead and responsiveness | p95 TTFT, gateway latency |

These are synthetic examples; model context window, tokenizer and output caps may invalidate a cohort. Never test an unsupported context length.

## 3. Measurement contract
For request arrival `t0`, first token `t1`, last token `tn`, completion `tc` and output token count `n`:
- **TTFT** = `t1 - t0`; includes queue and gateway path if measured client-side.
- **TPOT** = `(tn - t1) / (n - 1)` for `n > 1`; otherwise undefined.
- **ITL** = consecutive token arrival gaps; report p50/p95 across gaps and cohorts.
- **End-to-end** = `tc - t0`.
- **Queue wait** = `admission_start - queue_enter`.
- **Prefill service time** = prefill_end − prefill_start when runtime spans expose these timestamps.

TTFT is **not** equivalent to pure prefill time. Network buffering, admission and queueing can dominate. Likewise TPOT can reflect transport backpressure rather than GPU decode alone. Report measurement boundaries and clocks. Use monotonic timers and avoid logging token text.

## 4. Controlled experimental design
1. Pin model weights/digest, tokenizer, quantization, runtime, scheduler configuration, accelerator SKU/count, KV-cache limits, batch sizes, region and driver.
2. Create at least three cohorts: prefill-heavy, decode-heavy, balanced. Use synthetic token-count fixtures or approved privacy-preserving datasets.
3. Warm up, measure steady-state and cool down between repetitions; report cold-start separately.
4. Run **open-loop** arrivals to reveal queue growth and prevent coordinated omission. Sweep offered load until error/latency stop criteria are reached.
5. Measure both per-cohort and weighted aggregate TTFT/TPOT, errors, output token rate, queue, KV occupancy and GPU utilization.
6. Repeat runs and compare distributions rather than single percentiles. Separate hardware saturation from driver bottlenecks.
7. Enforce concurrency, duration, token, cost and request-rate caps; never load-test production without approval.

## 5. Modeling phase cost
An intentionally simplified latency decomposition is:
```text
E2E ≈ gateway + queue + prefill(input_tokens, batch)
    + decode(output_tokens, active_sequences, KV_pressure)
    + stream_flush
```
The actual runtime may overlap computation and streaming. An increased prompt length can increase prefill and KV occupancy, while output length increases decode duration and KV residency. Throughput may improve with batching even as interactive latency worsens.

### Practical hypotheses
- If **TTFT rises but TPOT remains stable**, investigate prefill, queueing and gateway buffering.
- If **TPOT rises with active sequences**, investigate decode contention, memory bandwidth and KV-cache pressure.
- If **both rise**, check shared GPU saturation, admission, memory pressure, network and scheduling.
- If **only end-to-end rises**, check output length distribution, retries and stream termination.

Avoid concluding root cause from a single metric.

## 6. Scheduler and KV-cache considerations
Continuous batching admits new requests while older requests decode; it can improve utilization but create interference between long prefills and latency-sensitive decode. Chunked prefill may reduce decode starvation, with overhead and implementation-specific behavior. KV-cache paging/eviction, prefix caching and speculative decoding change performance; benchmark each feature under representative mixes and disclose whether cache hit rates are artificial. Separate prefill/decode deployments only after measuring transfer overhead, placement and fault-domain complexity.

## 7. Hands-on lab
The [Chapter 067 companion lab](../labs/067-prefill-heavy-versus-decode-heavy-workloads-check.md) provides an offline deterministic simulator with synthetic prefill/decode phase timing, per-cohort metrics, queueing effects and pytest regression checks. This lab is **not** a GPU simulator or an empirical capacity benchmark.

## 8. Failure scenarios
| Symptom | Likely hypotheses | First checks |
|---|---|---|
| High TTFT on long prompts | prefill contention, queue | input distribution, queue span, prefill spans |
| High TPOT on long generations | decode contention, KV residency | active sequences, memory and token rate |
| OOM at moderate RPS | long context and concurrent KV usage | reserved KV pages, max sequences |
| Goodput falls after batching change | latency SLO breached | per-cohort TTFT/TPOT and scheduler |
| Synthetic benchmark looks unusually fast | prefix cache or repeated prompts | cache hit rate, dataset diversity |
| 200 responses but missing final tokens | stream disconnect | finish reason, completion SLI |
| GPU idle while queue grows | driver, admission or router bottleneck | worker queues and routing |

## 9. Production runbooks

### RB-067-A — Prefill latency regression
**[SAFE-READ]** inspect TTFT by input-length cohort, queue delay, GPU compute, prefill spans and recent model/runtime changes. Reproduce in staging at fixed offered load. **[CHANGE-CONTROL]** roll back scheduler/config changes or adjust admission after approval.

### RB-067-B — Decode slowdown
Correlate TPOT, output length, active sequences, KV occupancy, GPU memory bandwidth and streaming transport. Reduce workload only through approved rate limits/traffic shaping. Validate user impact and error budget after mitigation.

### RB-067-C — KV-cache exhaustion
Inspect max context, reserved KV blocks, active sequences, evictions, OOMs and retries. Stop unsafe benchmark traffic immediately; do not increase limits without memory headroom and rollout plan.

### RB-067-D — Capacity acceptance
Document cohort weights, p95 TTFT/TPOT, achieved token throughput, error ratio, goodput, safety margins, GPU-hours and reproducible configuration. Sign off only on actual staging evidence, not synthetic lab values.

## 10. Acceptance and security
- [ ] Model/runtime/hardware/tokenizer versions pinned.
- [ ] Input/output cohorts and traffic weights documented.
- [ ] TTFT and TPOT measured at stated boundaries; single-token outputs excluded from TPOT.
- [ ] Queue, prefill, decode and network effects distinguished.
- [ ] Fault and saturation drills reviewed; load caps enforced.
- [ ] No prompt text, token output, secret or user identifier in telemetry.
- [ ] Technical/security reviews and live execution independently recorded.

## References
- [vLLM benchmarking documentation](https://docs.vllm.ai/en/latest/benchmarking/)
- [NVIDIA Triton performance analyzer](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/perf_analyzer/README.html)
- [OpenTelemetry documentation](https://opentelemetry.io/docs/)

**Next:** Chapter 068 — Concurrency, Queueing and Backpressure.
