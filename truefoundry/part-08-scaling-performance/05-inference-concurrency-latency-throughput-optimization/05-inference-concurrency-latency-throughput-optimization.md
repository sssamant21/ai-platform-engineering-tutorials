# 8.5 — Inference Concurrency, Latency and Throughput Optimization

**Part 08 — Scaling & Performance | Chapter 5 of 7**  
**Audience:** TrueFoundry platform engineers, GPU SREs, vLLM inference operators  
**Execution labels:** **[SAFE-READ]** observation; **[LAB-MUTATE]** approved isolated staging; **[CHANGE-CONTROL]** production changes.

## 1. Learning objectives and prerequisites

Profile an LLM inference request from admission through prefill, first token, decode and streaming completion; distinguish TTFT, ITL/TPOT, end-to-end latency and goodput; select safe concurrency and batching settings; understand KV-cache constraints and preemption; design repeatable load tests; and recover from performance regressions.

Prerequisites: Parts 06–07 and Chapters 8.1–8.4; pinned TrueFoundry deployment, vLLM release, model/tokenizer and image digest; approved GPU capacity and read-only metrics; documented TTFT/ITL/error SLO and rollback owner.

**Golden rule:** Optimize **completed useful work within the agreed latency/error SLO**, not peak tokens/sec in isolation.

## 2. Inference critical path

```text
client request
  → DNS/TLS/auth/gateway/rate limits
  → routing and admission queue
  → tokenizer / prompt validation
  → scheduler waiting → prefill (prompt processing)
  → first token (TTFT)
  → decode iterations / streaming chunks (ITL)
  → final token / transport completion (E2E)
```

A high TTFT with normal ITL often points toward queueing, long prompt prefill, cold starts or gateway delay; high ITL with normal TTFT points more toward decode saturation, KV contention, GPU stalls or transport backpressure. These are diagnostic hypotheses, not proofs.

## 3. Define measurable service objectives

| Metric | Definition | Operational meaning |
|---|---|---|
| TTFT | Time from accepted request to first emitted token | Responsiveness |
| ITL | Time between generated tokens | Streaming smoothness |
| TPOT | Time per output token; definitions vary | Decode speed |
| E2E latency | Request acceptance to completed response | User experience |
| Throughput | Input/output tokens/s and requests/s | Raw work rate |
| Goodput | Requests/tokens completed within SLO | Useful capacity |
| Queue wait/age | Admission to active execution | Saturation |
| Error/abort rate | 429/5xx, timeout, cancellation | Reliability |
| GPU-hours / useful tokens | Cost per SLO-compliant work | Efficiency |

Specify where timestamps are taken (client, gateway or engine). Compare like-for-like. Use P50/P95/P99 by prompt/output length and streaming mode, not only averages. Treat low sample counts and coordinated omission carefully.

## 4. Workload shapes drive bottlenecks

| Workload | Likely pressure | Test dimensions |
|---|---|---|
| Short prompt / short output | Request overhead, queue, tokenizer | RPS, concurrency |
| Long prompt / short output | Prefill, context, VRAM | Input tokens, TTFT |
| Short prompt / long output | Decode and KV residency | Output tokens, ITL |
| Long prompt / long output | Both prefill and decode | Total live tokens |
| Shared-prefix requests | Prefix-cache reuse | Hit/miss mixes |
| Mixed interactive and batch | Fairness and tail latency | Priority/admission |
| Streaming slow clients | Transport/backpressure | Disconnects and drains |

A benchmark that uses one tiny prompt cannot establish production capacity for long-context RAG or mixed workloads.

## 5. Concurrency, Little's Law and safe admission

For a stable system, **L ≈ λW**: average in-flight requests ≈ arrival rate × mean response time. This is a planning relationship, not a guarantee under overload. If arrival rate exceeds sustainable completion rate, queue length and tail latency rise; adding concurrency alone can make throughput and SLOs worse.

```text
offered_load = requests_per_second × average_request_cost
admitted_load ≤ sustainable_SLO_capacity
```

Use bounded queues, token-aware quotas, per-tenant fairness, backpressure and explicit 429/retry-after semantics where supported. Retries must use exponential backoff with jitter to avoid amplifying an incident.

## 6. Prefill versus decode

**Prefill:** processes prompt tokens, often compute intensive; large prompts increase TTFT and activation pressure. **Decode:** generates tokens sequentially, often constrained by weight/KV memory bandwidth and the number of active sequences. Continuous batching interleaves work from multiple requests. Chunked prefill can improve fairness/TTFT tradeoffs for mixed workloads but must be validated against the installed engine version and model.

Tune both phases against a realistic prompt/output token distribution.

## 7. KV cache and concurrency budget

Approximate logical decoder-only KV memory:

```text
KV_bytes ≈ 2 × num_layers × num_kv_heads × head_dim
          × live_tokens × bytes_per_element
```

The real per-GPU budget depends on tensor parallel partitioning, block allocation, cache dtype, prefix reuse, fragmentation and runtime overhead. Observe engine-reported KV capacity, preemption, active/waiting sequences and VRAM; do not infer safe concurrency from total GPU VRAM alone.

Symptoms of KV saturation include increasing waiting requests, preemption/recompute, TTFT/ITL degradation and possibly OOM. Lowering max context or concurrency can protect stability but may reject legitimate requests—validate API contracts and tenant needs.

## 8. vLLM tuning levers and tradeoffs

| Lever | Potential benefit | Tradeoff / validation |
|---|---|---|
| `max_model_len` | Bound context and KV demand | May reject long inputs |
| `max_num_seqs` | Control active sequence count | Too high increases KV contention |
| `max_num_batched_tokens` | Tune token scheduling | Prefill/decode fairness tradeoff |
| `gpu_memory_utilization` | Adjust cache reservation | Too aggressive risks OOM |
| KV-cache dtype | Lower memory footprint | Accuracy and hardware compatibility |
| Prefix caching | Reuse shared prefixes | Workload dependent, isolation considerations |
| Chunked prefill | Bound long prefill monopolization | Version/model dependent |
| Tensor parallelism | Split model across GPUs | Communication and topology overhead |
| Quantization | Reduce weights/memory | Quality and kernel support |
| CUDA graphs / eager execution | Reduce overhead or aid debugging | Memory and compatibility tradeoff |
| Replica count | Add parallel serving capacity | GPU quota, warmup and cost |

**Flags and defaults are version-sensitive.** Verify `vllm serve --help`, the deployed TrueFoundry serving interface, official release documentation and model support before applying. Change one factor at a time; record model quality and safety acceptance when changing quantization, dtype or context.

## 9. Scaling decision tree

```text
SLO breach?
 ├─ High 429/5xx → gateway/admission/quotas, capacity, retries
 ├─ High TTFT
 │   ├─ waiting queue high → admission / replica capacity
 │   ├─ prompt tokens high → prefill / chunking / prompt limits
 │   └─ cold replicas → warm minimum / readiness / artifact path
 ├─ High ITL
 │   ├─ KV/preemption high → lower concurrency / KV budget
 │   ├─ GPU bandwidth/compute saturation → profile kernels/model
 │   └─ slow client streams → gateway/transport
 └─ High E2E only → output length / network / postprocessing
```

Correlate all signals; a high GPU utilization alone is not an incident.

## 10. Observability by layer

| Layer | Signals | Tools |
|---|---|---|
| Client/gateway | TTFT/E2E, 429/5xx, disconnects | Gateway telemetry, OTel |
| vLLM scheduler | Running/waiting, preemption, queue age | Engine metrics |
| vLLM execution | Prefill/decode throughput, ITL, KV usage | Engine metrics |
| GPU | SM utilization, memory, ECC/XID, power | NVIDIA DCGM |
| Pod/node | CPU, RAM, restarts, throttling, GPU placement | Kubernetes/Prometheus |
| TrueFoundry | Deployment revision, scaling events, routing | Platform telemetry |
| Cost | GPU-hours, useful tokens, idle warm capacity | Cloud billing + metrics |

Discover actual metrics and labels from the running versions; do not assume specific metric names. Exclude prompt text, tokens and sensitive identifiers from telemetry labels. Use latency histograms where available and calculate percentiles from buckets with adequate sample volume.

## 11. Benchmark design and regression gates

Use a **reproducible matrix**, not a single throughput number:

| Scenario | Input tokens | Output tokens | Concurrency | Purpose |
|---|---:|---:|---:|---|
| B1 | 128 | 64 | 1 | Warm single request |
| B2 | 128 | 256 | 8 | Decode contention |
| B3 | 2,048 | 64 | 8 | Prefill pressure |
| B4 | 2,048 | 512 | 16 | Mixed pressure |
| B5 | 8,192 | 128 | 4 | Long-context boundary |
| B6 | Production P50/P95 mix | Production mix | Ramped | Representative goodput |
| B7 | Shared-prefix mix | Mixed | Ramped | Cache value |
| B8 | Mixed interactive/batch | Mixed | Ramped | Fairness |

These are **illustrative**; respect actual model context limits and approved capacity. Include warm/cold comparisons, streaming/non-streaming, repeat runs, controlled seeds when meaningful, versioned prompts, arrival rate and confidence intervals. Keep concurrency, RPS and request length distinct.

**Safety:** synthetic inputs only; isolated staging; hard ceilings for request count, tokens, duration and GPU spend; abort on OOM, rising 5xx/429, persistent queue, SLO breach or node pressure.

## 12. Benchmark execution approach [LAB-MUTATE]

1. Pin model, tokenizer, image, vLLM version, GPU SKU, TP, context, dtype and runtime flags.
2. Verify readiness and synthetic baseline. Record GPU temperature, power and co-tenancy.
3. Use a vetted benchmark tool supported by the deployed version; inspect its CLI help before running.
4. Warm the engine; capture steady-state baseline for each workload shape.
5. Ramp arrival rate and concurrency separately. Observe **goodput at SLO**, not only aggregate tokens/s.
6. Change one supported tuning parameter; rerun identical matrix and compare P95/P99 TTFT/ITL, errors and cost.
7. Restore baseline config and verify model quality/latency; attach evidence.

For example, benchmark tooling may be available through the installed vLLM package; CLI subcommands differ across releases. Do not copy unverified flags into production automation.

## 13. Production tuning safety

- Use immutable deployment revisions and explicit rollout/rollback owners.
- Canary a small authorized traffic share; compare like-for-like workload buckets.
- Verify GPU memory headroom, startup, readiness and N-1 reserve before changing concurrency.
- Avoid changing batching, quantization, context and parallelism simultaneously.
- Set stop criteria and rollback on tail-latency, error, quality or cost regressions.
- Coordinate graceful termination and completion of active streams.

## 14. Failure scenarios and diagnostics

| ID | Symptom | First failed transition | Evidence |
|---|---|---|---|
| P1 | High TTFT, high queue | Admission → scheduler | Waiting requests |
| P2 | High TTFT, long prompts | Prefill → first token | Input length |
| P3 | High ITL at high concurrency | Decode → token | Per-token latency |
| P4 | KV cache exhausted | Scheduler → cache | KV occupancy |
| P5 | Frequent preemption | KV budget → scheduler | Preemption |
| P6 | GPU OOM on startup | Model load → VRAM | CUDA OOM |
| P7 | OOM during long contexts | Live tokens → VRAM | Context distribution |
| P8 | Low GPU utilization, high latency | Gateway/CPU → GPU | CPU/queue |
| P9 | High GPU utilization, healthy SLO | No failed transition | Goodput/SLO |
| P10 | Model quality degrades after quantization | Config → output quality | Evaluation |
| P11 | Tensor parallel overhead high | GPU communication → decode | NCCL/profile |
| P12 | Prefix caching no benefit | Prompt mix → cache | Hit/miss |
| P13 | Batch tuning hurts TTFT | Scheduler → prefill | TTFT by bucket |
| P14 | Slow model cold starts | Artifact → readiness | Load timing |
| P15 | Stream stalls under scale-down | Pod drain → client | Disconnects |
| P16 | High 429 despite idle GPU | Gateway quota → admission | Rate limits |
| P17 | Throughput high, goodput low | Capacity → SLO | SLO-qualified completions |
| P18 | Benchmark results inconsistent | Test design → evidence | Repeats/variance |
| P19 | GPU thermal/power throttling | Device → kernels | DCGM clocks/power |
| P20 | GPU ECC/XID faults | Device health → runtime | DCGM/driver |
| P21 | CPU tokenizer bottleneck | Tokenizer → scheduler | CPU saturation |
| P22 | Network/stream backpressure | Engine → client | Gateway timing |
| P23 | Retry storm after timeout | Client → gateway | Amplification |
| P24 | New revision regression | Deployment → engine | Immutable diff |

Record **Lowest Proven Healthy Layer (LPHL)**, **First Failed Transition (FFT)**, impact, safe mitigation, rollback and post-recovery SLO. P9 is intentionally a non-incident: do not “fix” healthy high GPU utilization.

## 15. Production incident runbook

1. **Declare:** endpoint/model, start time, affected tenants, TTFT/ITL/E2E P95/P99, errors, queue and GPU utilization.
2. **Classify:** prefill-bound, decode-bound, KV-bound, admission-bound, cold-start, infrastructure or transport.
3. **Correlate:** traffic/token mix, config/release diff, active/waiting requests, KV/preemption, GPU telemetry and Ready replicas.
4. **Contain:** preserve known-good warm capacity, enforce approved admission, rate limits and retry jitter; freeze unsafe rollout.
5. **Recover:** correct first proven failure or roll back immutable TrueFoundry/GitOps revision. Avoid unreviewed simultaneous tuning changes.
6. **Validate:** correct model, GPU health, TTFT/ITL by workload bucket, 429/5xx, stream completion, goodput and GPU-hours.
7. **Document:** LPHL/FFT, change correlation, measured sustainable capacity, owner and regression gate.

## 16. Acceptance gates

| Gate | Required evidence |
|---|---|
| A1 | Immutable model/runtime/GPU baseline and SLO |
| A2 | TTFT, ITL, E2E, goodput and queue definitions |
| A3 | Prefill/decode/KV bottleneck classified |
| A4 | Concurrency, batching and memory tradeoffs reviewed |
| A5 | Representative reproducible workload matrix |
| A6 | Safe staging tuning run or Not run |
| A7 | P1–P24 classified with LPHL/FFT |
| A8 | Canary, rollback and post-recovery SLO documented |

## 17. Evidence template

```text
Date/time / owner / approval:
TrueFoundry workspace, cluster, namespace:
Model/tokenizer/image digest, vLLM version, GPU SKU:
Runtime flags, TP, context, KV dtype, memory budget:
Prompt/output length buckets and streaming ratio:
Arrival rate, concurrency, warm/cold, test duration:
TTFT/ITL/E2E P50/P95/P99, queue and errors:
Input/output tokens/s, SLO goodput, GPU-hours:
GPU VRAM, KV usage, preemption, CPU/network:
One-variable tuning diff and benchmark variance:
LPHL/FFT, P1–P24, stop/rollback and recovery:
Acceptance gates and reviewer:
```

**Hands-on lab:** [8.5 — Inference Concurrency, Latency and Throughput Optimization Check](../labs/05-inference-concurrency-latency-throughput-optimization-check.md)  
**Next:** Chapter 8.6 — Scaling Observability, Cost and Troubleshooting.

## References

- [vLLM Documentation](https://docs.vllm.ai/)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
- [Prometheus Histograms](https://prometheus.io/docs/practices/histograms/)
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
