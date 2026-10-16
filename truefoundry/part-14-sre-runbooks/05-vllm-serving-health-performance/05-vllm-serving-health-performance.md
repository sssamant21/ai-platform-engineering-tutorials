# Chapter 14.5 — vLLM Serving Health and Performance Runbook

**Part 14 — TrueFoundry SRE Runbooks | Production inference operations**

## Mission

Operate vLLM-backed TrueFoundry model serving with reliable request admission, queueing, batching, KV cache management, GPU memory, TTFT/TPOT latency, streaming, autoscaling and incident recovery. Diagnose customer-facing performance regressions without assuming every slowdown is a GPU fault.

**[SAFE-READ]** authorized metrics, Kubernetes Events and sanitized logs. **[LAB-MUTATE]** disposable staging inference with bounded synthetic load. **[CHANGE-CONTROL]** production concurrency, GPU memory utilization, model/context settings, autoscaling, rollout, traffic routing and GPU/node changes. Never log real prompts, completions, PHI, API keys or customer model artifacts.

## 1. Inference serving dependency map

```text
Client / gateway / authentication / rate limits
   ↓ admission / request queue / streaming proxy
TrueFoundry desired deployment → Kubernetes Pod/Service
   ↓ vLLM API server / scheduler / continuous batching
Model weights + tokenizer + context limits + KV cache
   ↓ CUDA runtime / NVIDIA GPU / device plugin
   ↓ token prefill (TTFT) → decode (TPOT) → stream completion
Prometheus / OpenTelemetry / DCGM / alerting / SLO
```

Distinguish startup/warmup, queue wait, prefill, decode and gateway overhead. TTFT is time to first token; TPOT is per-output-token latency and must be calculated consistently across implementations. Total latency is influenced by prompt length, output length, batch shape, network and queueing.

## 2. SLI and SLO model

| Signal | Interpretation | Investigation |
|---|---|---|
| Eligible request success | end-to-end availability | HTTP/SDK errors, stream completion |
| p95 TTFT | queue + prefill + network | prompt tokens, queue, cold start |
| p95 TPOT | decode speed | batch size, KV cache, GPU, output mix |
| Queue time/length | admission pressure | arrival rate vs service rate |
| Running/waiting requests | scheduler utilization | max concurrency and capacity |
| KV cache utilization | memory pressure | model fit, context, eviction/preemption |
| GPU memory/utilization | resource load | DCGM, driver, model allocation |
| Tokens/sec | throughput | input vs output token accounting |
| Streaming completion | client-visible success | proxy buffering, timeout, drain |
| 429/5xx | rate limits/upstream failure | gateway and model logs |
| Metrics freshness | observability health | exporter scrape age |

Thresholds must come from approved model-specific baselines and workload profiles. High GPU utilization can be healthy. A successful HTTP 200 does not prove SSE completion.

## 3. First five minutes: on-call triage

1. Record UTC, model revision, cluster/region, GPU SKU, traffic volume, prompt/output token distributions and affected tenants.
2. Run approved synthetic request and streaming completion probe through the customer path.
3. Compare current vs baseline TTFT, TPOT, queue wait, throughput, 429/5xx, error budget and stream truncation.
4. Check recent TrueFoundry image/model/config releases, routing and autoscaler changes.
5. Inspect Kubernetes Pod state, restarts, probes, Events, node/GPU allocation.
6. Inspect vLLM startup/logs and **version-verified** metrics for running/waiting requests, KV cache and request timing.
7. Inspect DCGM GPU memory/utilization, driver and XID/ECC; check ingress timeouts and retries.
8. Choose reversible mitigation with owner, rollback, cost ceiling and SLO verification.

## 4. Read-only command pack

```bash
kubectl config current-context
kubectl -n MODEL_NAMESPACE get deployments,pods -o wide
kubectl -n MODEL_NAMESPACE get events --sort-by=.lastTimestamp
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE logs MODEL_POD --since=30m --tail=200
kubectl -n MODEL_NAMESPACE logs MODEL_POD --previous --tail=200
kubectl -n MODEL_NAMESPACE get svc,endpointslices
```

`--previous` is available only if a previous container instance exists. Logs can include sensitive inference data: redact before sharing. Do not scrape a publicly exposed `/metrics` endpoint without authorization.

On an approved GPU-enabled diagnostic host/Pod:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,memory.total,memory.used,utilization.gpu,temperature.gpu --format=csv
```

## 5. Version-verified vLLM telemetry

Metric names and labels vary across vLLM versions. Discover `/metrics` through the approved internal endpoint and inspect series before constructing alerts. Common metric families may include request success/failure, time-to-first-token, time-per-output-token, request queue/waiting/running, KV cache and token counts; do not assume exact names.

Illustrative PromQL shape **only after mapping actual histogram name and labels**:

```promql
histogram_quantile(
  0.95,
  sum by (le, model) (
    rate(VLLM_TTFT_BUCKET_METRIC[5m])
  )
)
```

`VLLM_TTFT_BUCKET_METRIC` is a placeholder, not a valid metric. Check whether metric is classic histogram or native histogram; use the appropriate query, confirm units and avoid mixing model/region populations. Alert on sustained SLO breaches with request-volume gates and missing-metrics detection.

## 6. Performance root-cause decision tree

```text
Inference degraded?
 ├─ 401/403/429 → auth, policy, quotas, retries
 ├─ 502/503/504 → gateway, Service, readiness, timeout
 ├─ Pod unavailable → scheduling, image, probes, model startup
 └─ Pod serving
     ├─ TTFT high, queue high → admission/capacity/concurrency
     ├─ TTFT high, queue normal → prompt length/prefill/cold start
     ├─ TPOT high → decode, batch mix, GPU throttling/KV pressure
     ├─ CUDA OOM → model/KV/context/memory/concurrency
     ├─ Host OOMKilled → container memory/model load
     ├─ Stream truncated → proxy timeout, buffering, drain, client
     └─ Metrics absent → telemetry freshness before conclusions
```

Correlate all branches with recent changes and synthetic traffic. Distinguish CUDA OOM, host OOM and scheduler preemption/eviction.

## 7. Queueing, batching and concurrency

Arrival rate, token-length distribution, prefill/decode ratio and effective concurrency govern latency. Higher batching can improve throughput but worsen tail TTFT/TPOT or KV pressure. Tune one variable at a time in staging: request concurrency, max model length, max batched tokens, GPU memory utilization, prefill scheduling and replicas, subject to version support. Preserve output quality and context contract; never silently truncate customer requests.

## 8. KV cache and GPU memory

GPU memory includes model weights, runtime overhead, activations and KV cache. A model may load but fail under long-context bursts. Monitor cache occupancy, preemptions/recomputations, CUDA allocation errors and prompt-length percentiles. Lower concurrency/context or add approved replicas only after validating SLO and GPU capacity. Avoid arbitrary memory-utilization settings that jeopardize stability.

## 9. Streaming and graceful termination

Test end-to-end SSE events, first token, termination/completion marker where applicable, gateway idle/read timeout, buffering, load-balancer drain, client cancellation and bounded retries. Use synthetic requests of short/long lengths. A response that starts streaming and then fails is an availability defect even if the initial HTTP status is 200.

## 10. Scaling, resilience and FinOps

Scale on customer SLO, queue/waiting requests, token throughput and GPU constraints rather than GPU utilization alone. Verify model warmup time, provisioning lead time, N-1 GPU headroom, zone placement and max node pool/cloud quota. Track cost per successfully served 1,000 output tokens, warm capacity cost and retries. Confirm workload quality/latency before choosing lower-cost GPU SKUs.

## 11. Safe mitigation hierarchy

1. Freeze unrelated releases and establish model-specific SLO impact.
2. Stop retry amplification; use approved rate limits/circuit breakers.
3. Route to healthy compatible model replicas and preserve tenant isolation.
4. Roll back known-bad model/image/config through TrueFoundry authoritative desired state.
5. Add replicas only if schedulable GPU slots and cloud capacity exist.
6. Tune bounded concurrency/context/batching only with staging evidence.
7. Escalate NVIDIA/driver/hardware faults for change-controlled node recovery.
8. Validate HTTP success, TTFT, TPOT, queue, SSE completion, GPU health and cost before closure.

## 12. Sixteen operational runbooks

1. Synthetic inference and streaming health.
2. vLLM startup and model warmup.
3. TTFT regression.
4. TPOT regression.
5. Request queue saturation.
6. Continuous batching and prefill/decode contention.
7. KV cache pressure and preemption.
8. CUDA OOM.
9. Host OOMKilled.
10. GPU utilization, XID/ECC and thermal.
11. 429/5xx and retry storm.
12. Gateway SSE timeout/truncation.
13. Autoscaling and GPU capacity shortage.
14. Model revision rollback.
15. Metrics/alert freshness and telemetry gaps.
16. Recovery, error budget and FinOps verification.

Every runbook records trigger, impact, evidence, hypothesis, approved mitigation, rollback, synthetic SLO and owner.

## 13. Forty-eight failure scenarios

| Domain | Scenarios |
|---|---|
| Startup | 01 image pull; 02 model artifact access; 03 tokenizer mismatch; 04 CUDA mismatch; 05 GPU missing; 06 warmup timeout |
| Latency | 07 high TTFT; 08 high TPOT; 09 high queue wait; 10 long-prompt surge; 11 long-output surge; 12 prefill contention |
| Scheduling | 13 waiting requests spike; 14 batch tokens too high; 15 max concurrency too high; 16 throughput collapse; 17 unfair workload mix; 18 priority inversion |
| Memory | 19 CUDA OOM; 20 host OOMKilled; 21 KV cache full; 22 preemption/recompute; 23 memory fragmentation; 24 context length mismatch |
| GPU | 25 driver XID; 26 ECC fault; 27 thermal throttle; 28 high GPU healthy; 29 low GPU slow; 30 multi-GPU communication |
| Networking | 31 gateway 502; 32 gateway 503; 33 gateway 504; 34 SSE buffering; 35 stream truncation; 36 client cancellation |
| Scaling | 37 GPU quota; 38 autoscaler max; 39 zone shortage; 40 cold replica; 41 N-1 shortfall; 42 spot interruption |
| Operations | 43 telemetry stale; 44 alert missing; 45 retry storm; 46 cost/token spike; 47 rollback artifact missing; 48 recovery checked only in UI |

## 14. Fourteen acceptance gates

G1 approved scope/budget; G2 immutable model/runtime inventory; G3 synthetic inference/stream; G4 TTFT/TPOT baseline; G5 queue and throughput; G6 KV cache/memory; G7 CUDA vs host OOM; G8 GPU/DCGM; G9 gateway/429/5xx; G10 autoscaling/N-1; G11 rollback; G12 alert freshness; G13 cost/token and customer SLO recovery; G14 reviewer sign-off.

See the [hands-on lab](../labs/05-vllm-serving-health-performance-check.md). Formal technical/security review and live execution pending.

**Next:** Chapter 14.6 — Incident Response, Escalation and Communications Runbook.
