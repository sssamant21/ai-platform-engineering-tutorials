# Chapter 11.5 — GPU, vLLM and Inference Performance Observability

**Part 11 — Observability | Canonical production SRE tutorial**

## Objectives and safeguards

Observe the full request path for GPU-backed model serving on TrueFoundry: gateway, application, vLLM scheduler, CUDA runtime, NVIDIA device plugin, GPU node, storage and cloud dependencies. Distinguish throughput from latency, prefill from decode, GPU capacity from actual saturation, and model runtime failure from Kubernetes scheduling failure.

**[SAFE-READ]** observe and query; **[LAB-MUTATE]** synthetic load on approved staging GPUs; **[CHANGE-CONTROL]** production alert or configuration changes. Verify installed TrueFoundry, vLLM, CUDA, driver, DCGM exporter, Prometheus and NVIDIA device plugin versions. Metric names and capabilities vary by version. Never expose prompts, model weights, customer data, tokens or secrets in logs, metrics or traces.

## 1. Request and telemetry architecture

```text
Client → TLS/Ingress/Gateway → Model service → vLLM scheduler
                │                 │             ├─ waiting/running requests
                │                 │             ├─ prefill → first token
                │                 │             ├─ decode → output tokens
                │                 │             └─ KV cache / batching
                │                 └─ GPU runtime (CUDA/NCCL)
                │                        └─ NVIDIA driver / GPU / MIG
                └─ network, auth, rate limiting, 4xx/5xx
             metrics + structured logs + traces → collector → Prometheus/Grafana
```

Trace the **same synthetic request** across gateway, application and runtime when instrumentation supports it. A successful HTTP 200 may still hide poor TTFT, partial streaming, low output throughput or an incomplete response.

## 2. Key performance definitions

| Signal | Definition | Why it matters |
|---|---|---|
| Request rate | Eligible inference requests per second | Offered load |
| Success/error ratio | Successful vs eligible requests | Reliability |
| TTFT | Request start to first generated token | Interactive responsiveness |
| Time per output token (TPOT) | Decode time normalized by output tokens, using runtime definition | Streaming responsiveness |
| Inter-token latency | Time between emitted tokens | User-perceived pacing |
| End-to-end latency | Request start to completion | Total experience |
| Input tokens | Prompt length/tokenized input | Prefill pressure |
| Output tokens | Generated tokens | Decode workload |
| Tokens/sec | Input/output/aggregate as separately defined | Capacity |
| Queue wait | Time awaiting execution | Saturation |
| Running/waiting requests | Runtime scheduler state | Concurrency |
| KV cache usage | Runtime cache pressure/availability | Memory-bound scheduling |
| GPU utilization | Device activity over sampling interval | Resource use |
| GPU framebuffer memory | Allocated/used device memory | Model/KV fit |
| GPU power/thermal/ECC/XID | Hardware health | Fault detection |

**Important:** Different vLLM releases expose different metric families and definitions. Build a metric dictionary from the actual `/metrics` endpoint and release documentation. Do not infer TTFT or TPOT from a generic HTTP latency histogram. Streaming duration and TTFT require different measurements.

## 3. Safe Kubernetes/GPU inventory [SAFE-READ]

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -o custom-columns='NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu'
kubectl get pods -A -o wide
kubectl get daemonsets -A
kubectl get events -A --sort-by=.lastTimestamp | tail -50
kubectl top nodes 2>/dev/null || true
```

Identify the approved GPU node and monitoring Pod before running:
```bash
kubectl -n GPU_NAMESPACE logs daemonset/DCGM_EXPORTER --since=10m --tail=80
```

For a permitted GPU-enabled Pod with NVIDIA tools:
```bash
kubectl -n MODEL_NAMESPACE exec MODEL_POD -c MODEL_CONTAINER -- nvidia-smi
```

`nvidia-smi` is a read-only query, but `kubectl exec` is privileged interactive access and may be disallowed; use approved node/exporter metrics if so. A GPU resource allocation does not prove usable CUDA or healthy driver compatibility.

## 4. GPU observability and device topology

Monitor utilization, memory, temperature, power, throttling, ECC/XID faults, driver health, GPU reset, device plugin readiness and allocatable resources. For MIG, distinguish physical GPU, GPU instance and compute instance; avoid double-counting slices. Correlate GPU identity with node, Pod, workload and model version using approved labels.

DCGM exporter metric names such as `DCGM_FI_DEV_GPU_UTIL` or `DCGM_FI_DEV_FB_USED` are **examples only** and depend on exporter configuration/field set. Check actual units, labels, MIG support and scrape intervals. A high utilization sample does not alone mean the system is saturated; compare queues, token throughput and TTFT.

## 5. vLLM scheduler and KV cache

Investigate queue growth, prefill/decode competition, max concurrent sequences, token budgets, batching, prefix caching and KV cache pressure. vLLM scheduling and memory behavior depend on version and engine mode; do not apply an older metric name or flag to a newer engine without verification.

Typical bottleneck patterns:
- **High queue + rising TTFT + stable GPU throughput:** serving concurrency/capacity saturation.
- **High input tokens + TTFT regression:** prefill-heavy traffic or longer context windows.
- **KV cache near exhaustion + preemptions:** memory scheduling pressure; inspect max context and concurrency.
- **Low GPU utilization + high latency:** gateway, CPU tokenization, network, storage or request serialization.
- **High GPU memory + CUDA OOM:** model/KV memory fit; distinguish process OOM from Kubernetes container OOMKilled.
- **GPU XID/ECC + intermittent failures:** device/driver health; follow hardware incident policy.

## 6. Metrics and PromQL

Use exact names discovered from your installed gateway and vLLM exporter. Illustrative HTTP queries:

```promql
sum by (model) (rate(inference_requests_total[5m]))
```

```promql
sum(rate(inference_requests_total{outcome="error"}[5m]))
/
clamp_min(sum(rate(inference_requests_total[5m])), 1e-9)
```

```promql
histogram_quantile(
  0.95,
  sum by (le,model) (rate(inference_ttft_seconds_bucket[5m]))
)
```

These `inference_*` names are **illustrative custom instrumentation**, not asserted vLLM metric names. Verify `model`, `outcome` labels and histogram bucket units. For GPU utilization, use exporter-provided gauge and inspect its documented scale (percent vs fraction). Never average p95 across Pods; aggregate compatible histogram buckets first.

## 7. Load-test design and capacity curves

Use approved synthetic prompts with documented input/output token distributions. Ramp concurrency in bounded steps and record offered RPS, accepted RPS, error rate, TTFT p50/p95/p99, TPOT, output tokens/sec, queue depth, KV cache, GPU utilization/memory, CPU, network and cost. Keep model weights, quantization, context length, batching flags, replicas and GPU type constant during comparisons.

Identify the **knee** where additional offered load increases queues/TTFT faster than throughput. Stop on error budget, OOM, GPU faults or customer impact. Distinguish warmup, model loading and steady-state samples; compare identical time windows.

## 8. Dashboards and alerts

Create panels for gateway RED, request tokens and concurrency, TTFT/TPOT distributions, output token throughput, running/waiting requests, KV cache, preemption, GPU utilization/memory/thermal/XID, Pod readiness/restarts and model load duration. Include deployment revision and GPU type. Alert on sustained user-impacting latency/errors with queue or resource evidence, and separately on hard GPU faults and telemetry silence. Use environment-specific baselines rather than arbitrary universal thresholds.

## 9. Twelve operational troubleshooting runbooks

1. **GPU metrics absent:** inspect exporter DaemonSet, scrape targets, driver and MIG field support.
2. **GPU allocated but CUDA unavailable:** check device plugin, container runtime, driver/toolkit compatibility and Pod resources.
3. **TTFT p95 high:** compare input token distribution, prefill time, queue depth, CPU tokenization and network.
4. **TPOT high:** inspect decode throughput, batching, GPU saturation and competing workloads.
5. **Queue grows without throughput:** check scheduler limits, KV cache, stalled requests and readiness.
6. **GPU memory near full:** distinguish expected model allocation from dynamic KV cache pressure.
7. **CUDA OOM:** inspect model size, max sequence length, quantization, concurrency and device memory.
8. **Pod OOMKilled:** inspect cgroup memory and Kubernetes limits; separate from GPU OOM.
9. **Low GPU utilization:** inspect CPU/network/storage bottlenecks, small batches and lack of demand.
10. **GPU XID/ECC or thermal throttling:** correlate DCGM/node events and escalate hardware issue.
11. **Streaming stalls or disconnects:** inspect gateway timeouts, client cancellation, backpressure and runtime logs.
12. **Throughput regression after deploy:** compare digests, vLLM flags, tokenizer, CUDA driver, traffic mix and warmup.

## 10. Production acceptance and rollback

Require verified metrics inventory, GPU-to-Pod attribution, synthetic baseline, p95/p99 TTFT and TPOT, queue and KV cache dashboards, alert owners, versioned runtime configuration, canary plan, rollback commit and evidence. A performance rollback must not reintroduce insecure images, exposed endpoints or compromised credentials.

Complete the [companion lab](../labs/05-gpu-vllm-inference-performance-observability-check.md) with **32 failure scenarios and 10 acceptance gates**. Formal review and live execution remain pending.

**Next:** Chapter 11.6 — Dashboards, SLIs, SLOs, Alerting and Error Budgets.
