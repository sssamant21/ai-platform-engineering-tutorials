# Chapter 071 — GPU Throughput and Cost per Successful Token

**Part 09 — Performance Engineering and Capacity · Chapter 071 of 100**

**Status:** Published documentation and companion lab; live execution and technical/security reviews pending.

## Learning outcomes
Measure achieved request and output-token throughput; distinguish raw token throughput from successful, SLO-compliant output; calculate GPU-hour cost per successful token; account for idle capacity, retries, replica redundancy and autoscaling; define defensible capacity/cost gates.

## Cost and inference architecture
```text
offered arrivals -> admission -> GPU workers -> stream completion
                          |             |
                   errors / latency   utilization
                          |             |
                  SLO-compliant tokens  GPU-hours
                          \             /
                         cost per good token
```
Output tokens per second, request throughput, TTFT, TPOT and p95 completion latency answer different questions. A higher raw tokens/sec figure can conceal slow first-token response, errors or low goodput. Compute costs over a consistent observation interval and matched workload mix.

## Financial definitions
```text
GPU_cost = sum(replica_GPU_count × billed_hours × price_per_GPU_hour)
successful_tokens = sum(output_tokens for fully completed successful requests)
good_tokens = sum(output_tokens for successful requests meeting all SLOs)
cost_per_million_good_tokens = GPU_cost / good_tokens × 1,000,000
```
Report **undefined**, not zero, if no good tokens. Distinguish **GPU-only** cost from fully loaded cost (CPU, memory, storage, networking, orchestration, observability and licenses). Include billed idle time, reserved standby capacity and retries. Define whether prompt tokens are part of the unit economics; for most user-facing generation goodput measures, output tokens are the denominator, but disclose both input and output volumes.

## Benchmark dimensions
| Dimension | Required segmentation |
|---|---|
| Workload | input/output token lengths, streaming, long context |
| Service | model digest, quantization, runtime, region |
| Load | offered RPS, concurrency, burstiness |
| Hardware | accelerator SKU, count, utilization, power if measured |
| Reliability | error ratio, timeouts, partial streams, SLO compliance |
| Cost | on-demand/reserved price, utilization window, idle/standby |

## Capacity and FinOps experiment
1. Pin workload cohorts, model, quantization, GPU type and pricing assumption.
2. Warm up and measure sustained traffic at multiple offered loads.
3. Collect eligible, completed, good requests and generated output tokens; exclude partial responses from good-token denominator unless an explicit contract states otherwise.
4. Measure billed GPU-hours for **all** replicas, including idle and failover headroom.
5. Compare cost per million good tokens alongside p95 TTFT, TPOT, errors and queue age.
6. Evaluate N-1 capacity and cold-start effects; select operating point below saturation knee.
7. Perform sensitivity analysis for GPU price, workload mix, utilization and failure rate.

## Failure scenarios
- High GPU utilization, low goodput: queueing, long outputs, retries or quality/latency failures.
- Cheap GPU-hour, expensive good token: insufficient throughput, bad kernel or frequent failures.
- Cost improvement after removing replicas, degraded resilience: restore approved failover margin.
- Output token counts mismatch billing: tokenizer/version mismatch or stream completion errors.
- Sudden cost jump: idle replicas, failed autoscaling, retry storm, price changes.
- Incomplete telemetry: mark cost per good token unknown and investigate missing counters.

## Production runbooks
**RB-071-A Unit-cost regression:** verify billing interval, price source, replica count, token taxonomy and SLO-compliant denominator before attributing cost change.

**RB-071-B Throughput collapse:** correlate goodput with TTFT/TPOT, queue, KV occupancy, GPU metrics and recent changes; use approved rollback or traffic shift.

**RB-071-C Capacity and budget guardrail:** stop benchmarks on spend/latency/error limits; never sacrifice N-1 availability solely for lower measured cost.

**RB-071-D FinOps sign-off:** publish normalized cost curves, SLO gates, pricing assumptions, standby overhead and evidence with owner approval.

## Security and acceptance
Use only aggregate token counts and bounded cohort labels; do not export prompts, completions, tenant identifiers or credentials. Synthetic cost examples are **not** live billing estimates.

- [ ] Price, time window and GPU count documented.
- [ ] Complete vs partial streams classified.
- [ ] Good-token denominator tied to explicit SLO.
- [ ] Retry/idle/standby costs included.
- [ ] N-1 and saturation margin validated.
- [ ] Independent live execution and reviews pending or completed explicitly.

## References
- [Google SRE Workbook — Capacity Planning](https://sre.google/workbook/non-abstract-design/)
- [Prometheus metric types](https://prometheus.io/docs/concepts/metric_types/)
- [NVIDIA DCGM exporter](https://github.com/NVIDIA/dcgm-exporter)
- [OpenTelemetry](https://opentelemetry.io/docs/)
## Hands-on lab
[Executable Chapter 071 lab](../labs/071-gpu-throughput-cost-per-successful-token-check.md). Local synthetic checks only; no production endpoint or real model required.

**Next:** Chapter 72 — Cold Start, Warm Pools and Model Loading.
