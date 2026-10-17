# Chapter 069 — Batching, KV Cache and Memory Tuning

**Part 09 — Performance Engineering and Capacity · Chapter 069 of 100**  
**Status:** Published tutorial and lab; live execution and technical/security review pending.

## Objectives
Explain static and continuous batching; estimate KV-cache capacity and fragmentation; tune scheduler admission and memory utilization against TTFT/TPOT, throughput, goodput and OOM risk; validate performance changes with controlled cohorts, executable tests and production runbooks.

## 1. Serving architecture
```text
incoming prompts -> bounded admission -> scheduler
                                          |
                               prefill / chunked prefill
                                          |
                          active sequence KV-cache blocks
                                          |
                                decode / continuous batching
                                          |
                            streaming output + completion
                                          |
                 metrics: TTFT TPOT queue KV utilization OOM
```
A transformer decoder stores attention keys and values for past tokens to avoid recomputing the full prefix on each generated token. KV memory grows with resident tokens, model architecture and precision. Real implementations may page blocks, share prefixes, offload to CPU, quantize KV, use sliding windows, or employ tensor/pipeline parallelism; these change capacity and failure modes.

## 2. Batching strategies
| Strategy | Benefit | Tradeoff |
|---|---|---|
| Static batching | straightforward scheduling, high batch efficiency | stragglers and head-of-line blocking |
| Continuous batching | admit/retire sequences each decode step | scheduling complexity and KV pressure |
| Chunked prefill | interleave long prefills with decode | chunk overhead and TTFT tradeoffs |
| Prefix caching | reuse repeated prompt prefix computation/KV | memory residency and cache isolation |
| Disaggregated prefill/decode | specialize phase resources | transfer overhead, networking and placement |

**Batch size is not a single knob:** concurrent requests, max batched tokens, active sequences, prefill chunk size and KV blocks interact. Raising concurrency can increase throughput until cache saturation, then cause evictions, re-computation, preemption, OOM or latency collapse.

## 3. KV-cache capacity estimation
For a conventional decoder-only attention implementation without sharing/offload, approximate bytes per resident token:
```text
bytes_per_token ≈ 2 × n_layers × n_kv_heads × head_dim × bytes_per_element
KV_bytes ≈ resident_tokens × bytes_per_token
```
The factor 2 accounts for K and V. Grouped-query attention uses `n_kv_heads` rather than query heads. Example: 32 layers, 8 KV heads, head dimension 128, FP16 (2 bytes) → 131,072 bytes/token (128 KiB/token). At 100,000 resident tokens, theoretical KV payload is ~12.2 GiB. This **excludes** model weights, runtime overhead, fragmentation, graph capture, activations, temporary buffers and allocator reserve.

When paged attention uses fixed blocks, allocated capacity is rounded to block boundaries:
```text
allocated_tokens = ceil(sequence_tokens / block_size) × block_size
fragmentation_tokens = allocated_tokens − sequence_tokens
```
Prefix sharing and block reuse make aggregate accounting more complex; never treat this estimate as exact GPU memory usage.

## 4. Memory budget and admission
```text
GPU memory
  ├─ model weights and runtime
  ├─ activation / workspace / CUDA graphs
  ├─ KV cache reservation
  ├─ allocator fragmentation
  └─ safety headroom
```
Reserve memory for worst-case batch and long-context cohorts. Admission should consider **estimated in-flight tokens**, not just request count. Reject early when projected KV blocks exceed a tested safe budget. Avoid allowing a burst of long-context prompts to evict latency-sensitive sessions without fairness policy.

## 5. Measurement and tuning experiment
1. Pin model weights/digest, tokenizer, GPU SKU, runtime version, quantization, KV dtype, context limits and scheduler config.
2. Record baseline with short, long, prefill-heavy, decode-heavy and mixed cohorts from Chapters 066–068.
3. Sweep `max_num_seqs`, max batched tokens, prefill chunking and cache budget **one variable at a time**, then test interactions.
4. Monitor TTFT/TPOT p50/p95/p99, achieved tokens/sec, goodput, queue age, GPU utilization, KV occupancy, cache hit rate, preemption, OOM and error budget burn.
5. Use open-loop arrivals, warm-up, repeated steady-state windows and degraded-capacity tests. Detect coordinated omission and driver bottlenecks.
6. Reject any tuning improvement that increases privacy risk, instability, tail latency or cost per good request beyond approved gates.

### Sample acceptance gates
- No OOM/preemption during steady-state approved load.
- p95 TTFT and TPOT remain within service-specific objectives.
- Error budget burn stays within release policy.
- Goodput and cost/good-request improve at **matched cohort mix**.
- Safe recovery from replica loss and cold cache demonstrated.

## 6. Failure scenarios
| Symptom | Likely causes | Checks |
|---|---|---|
| OOM after raising concurrency | KV reservation exceeds headroom | resident tokens, memory budget, preemption |
| TPOT degrades under long context | decode contention, cache pressure | KV occupancy, active sequences |
| TTFT improves, stream stalls | aggressive prefill or scheduler unfairness | chunk scheduling, token cadence |
| Throughput improves but goodput drops | tail latency and errors | cohort SLOs and queue |
| High cache hit ratio, poor real performance | synthetic prompt repetition | workload diversity and cache scope |
| Memory fragmentation | block size, variable sequence lengths | allocated vs useful tokens |
| Tenant cache isolation concern | unsafe prefix reuse | access controls and cache namespace |

## 7. Production runbooks

### RB-069-A — KV-cache exhaustion
**[SAFE-READ]** inspect active sequences, resident tokens, KV block occupancy, preemptions, OOMs and recent tuning changes. **[CHANGE-CONTROL]** reduce approved concurrency, route load or roll back scheduler changes. Verify completion SLI and GPU memory headroom.

### RB-069-B — Batching regression
Compare pinned cohort mix, prefill/decode split, TTFT/TPOT, token throughput and goodput. Reproduce with one variable changed; restore known-good scheduler settings through normal release controls.

### RB-069-C — Prefix-cache privacy risk
Stop unsafe cache sharing under incident authority; review isolation boundaries and synthetic cross-tenant tests, restrict evidence access, notify security and document remediation. Do not store customer prompts in logs.

### RB-069-D — Capacity sign-off
Record approved maximum in-flight tokens, cache budget, safe operating concurrency, p95 TTFT/TPOT, OOM/preemption rates, failover margin, cost and rollback plan. Sign-off requires actual staging evidence.

## 8. Executable lab
The [Chapter 069 companion lab](../labs/069-batching-kv-cache-memory-tuning-check.md) implements a deterministic KV-memory estimator, block-fragmentation analysis, bounded admission simulator and pytest tests. **It does not emulate real GPU allocation or vLLM scheduling.**

## 9. Security, FinOps and acceptance
- No prompts, completions, credentials or tenant IDs in metrics labels or test reports.
- Prefix-cache sharing requires strict authorization and isolation; never assume a shared cache is safe across tenants.
- GPU memory tuning is a production change requiring rollback and stop conditions.
- [ ] KV math validated against actual model config.
- [ ] Weights, overhead, cache and headroom accounted for.
- [ ] Admission rejects oversized requests and respects capacity.
- [ ] Cohort goodput, p95 latency and OOM/preemption assessed.
- [ ] Technical/security review and live lab execution independently recorded.

## References
- [vLLM documentation](https://docs.vllm.ai/)
- [PagedAttention paper](https://arxiv.org/abs/2309.06180)
- [NVIDIA CUDA memory management](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
- [Google SRE Workbook — Handling Overload](https://sre.google/workbook/handling-overload/)

**Next:** Chapter 070 — Model Precision, Quantization and Quality Gates.
