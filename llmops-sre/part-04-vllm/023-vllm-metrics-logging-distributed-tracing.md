# Chapter 023 — vLLM Metrics, Logging and Distributed Tracing

**Part 04 — vLLM Administration and Operations | Chapter 023 of 100**

> **Status:** Written tutorial published. Live labs, independent technical/security review and operational acceptance pending. vLLM metric names, trace hooks and labels are version-dependent; verify the pinned runtime.

## Learning objectives

Design customer-path observability for vLLM using metrics, structured logs and distributed traces; measure TTFT, TPOT/ITL, E2E latency, complete SSE success, queue wait, KV pressure and GPU health; integrate Prometheus, OpenTelemetry and DCGM; distinguish stale telemetry from healthy zeros; build SLO burn alerts, dashboards and incident runbooks without leaking prompts or secrets.

## 1. Observability architecture

```text
Client SDK → TLS/API gateway → vLLM API → scheduler → GPU workers
    │             │             │          │          │
    ├── trace context propagation ──────────────────────┤
    ├── request/stream-completion SLIs                   │
    ├── structured metadata-only logs                   │
    ├── Prometheus runtime metrics                      │
    └── DCGM exporter / node / K8s metrics ─────────────┘
                    │
          OTel Collector / Prometheus
                    │
       dashboards, alerts, SLO burn, RCA
```

A healthy Pod, a successful HTTP 200 and nonzero GPU utilization do not prove that the customer received a complete response. Correlate gateway, runtime, GPU and client evidence using bounded, privacy-safe identifiers.

## 2. Golden signals for inference

| Layer | Signal | Why it matters |
|---|---|---|
| Customer path | complete-stream success | detects partial HTTP 200 responses |
| Customer path | TTFT | time until first generated token |
| Customer path | TPOT/ITL | decode responsiveness |
| Customer path | E2E | total user-visible latency |
| Gateway | 401/403/429/5xx, timeout, retries | auth/overload/upstream |
| Scheduler | waiting/running requests, queue time | saturation/fairness |
| Engine | prefill/decode, token throughput | workload and bottleneck |
| KV cache | occupancy, preemptions, evictions | memory pressure |
| GPU | HBM, SM, temperature, power, XID/ECC | hardware/runtime health |
| Platform | Pod restarts, readiness, scheduling | availability |
| FinOps | GPU-hours per completed valid request | efficiency |

Track cohorts by approved model alias, route, prompt/output token buckets and workload class. Avoid raw prompt, user ID, request ID or generated text as metric labels.

## 3. Correct inference latency definitions

```text
TTFT = first meaningful output token time − request start time
TPOT ≈ time between generated output tokens
E2E = validated completion time − request start time
```

The exact start point must be documented: client, gateway or vLLM ingress. Client-side TTFT includes network and gateway delay; backend TTFT may not. Metadata-only SSE chunks do not count as generated tokens. For TPOT, specify whether you use average per request, inter-token samples or a percentile of request-level values; these are not interchangeable.

For long output streams, an HTTP response may be successful at the transport layer but fail before the terminal SSE event. Count such streams as incomplete in the customer-path SLI.

## 4. Metrics architecture and scrape contracts

vLLM commonly exposes Prometheus-style metrics in supported serving configurations, but **specific names, histogram buckets, counters and labels differ across versions**. Inspect the actual deployed endpoint and document the schema. Do not copy PromQL from another version without verification.

An approved read-only endpoint check may use a local port-forward in a safe context:

```bash
kubectl -n MODEL_NAMESPACE port-forward svc/APPROVED_VLLM_SERVICE 18000:8000
```

In another terminal, if the pinned runtime exposes `/metrics`:

```bash
curl --fail --silent --show-error http://127.0.0.1:18000/metrics | head -80
```

Port-forward establishes a local tunnel and can expose service data to the operator; obtain authorization, bind localhost only, avoid sensitive output and close it promptly. The command is illustrative; verify Service name, port and endpoint.

## 5. Prometheus recording rules: schema-neutral examples

The following is **pseudocode**. Replace metric names, labels and histogram bucket conventions with the actual pinned vLLM/gateway exporter schema:

```promql
# Request rate
sum(rate(<verified_completed_request_counter>[5m]))

# 5xx ratio
sum(rate(<verified_5xx_counter>[5m]))
/
clamp_min(sum(rate(<verified_eligible_request_counter>[5m])), 1)

# p95 TTFT histogram, if exposed as classic buckets
histogram_quantile(
  0.95,
  sum by (le, model) (rate(<verified_ttft_bucket_metric>[5m]))
)
```

Do not deploy these placeholders. For native histograms, use the supported query form for your Prometheus version. Verify counter resets, scrape gaps, labels, units and low-traffic behavior.

## 6. Structured logging policy

Recommended fields:
- UTC timestamp and severity;
- service, environment, model alias, image/model revision;
- pseudonymized tenant class;
- route, HTTP status, error category;
- request/trace correlation ID in **logs**, not metric labels;
- prompt/output token counts, latency buckets, stream completion status;
- cancellation, queue and runtime error classification.

Exclude authorization headers, API keys, cookies, raw prompts, generated output, model weights, PHI and arbitrary exception payloads. Control log retention, access and audit. A security incident is not permission to turn on unredacted body logging.

## 7. Distributed tracing design

```text
Trace: inference.request
 ├─ gateway.auth
 ├─ gateway.admission
 ├─ serving.tokenization
 ├─ scheduler.queue
 ├─ engine.prefill
 ├─ engine.decode
 ├─ stream.first_token
 └─ stream.terminal_completion
```

These are **proposed span names**, not a claim that vLLM emits them. Instrument gateway/client/sidecars or runtime hooks only where supported. GPU kernel time may not be represented by a distinct OTel span. Preserve W3C trace context where appropriate and avoid adding raw prompt content to spans. Sampling must retain meaningful failures while bounding telemetry cost.

## 8. GPU/DCGM correlation

Compare GPU memory, utilization, HBM activity, temperature/power, XID/ECC and exporter freshness with vLLM queue, KV and token timing. High GPU utilization may be healthy or saturated; low GPU utilization with a long queue can indicate scheduling, tokenization, routing or network trouble.

DCGM exporter requires a compatible GPU monitoring stack and approved privileges. Verify actual metric availability; not every GPU supports every DCGM field. Do not interpret a missing XID metric as proof of zero errors.

## 9. SLOs and burn-rate alerts

Example **synthetic** SLO:

```text
complete-stream success target = 99.9%
eligible streams = 100,000
observed incomplete/failed = 250
allowed failures = 100
observed success = 99.75%
burn rate = (250/100,000) / 0.001 = 2.5×
```

Use multi-window burn alerts with a minimum request volume and missing-data detection. Alert separately on telemetry freshness; missing scrapes should not silently make the success ratio look healthy. Maintain distinct SLOs for latency-sensitive versus batch workloads where appropriate.

## 10. Dashboard design

Recommended panels:
1. complete SSE success and error-budget burn;
2. TTFT p50/p95/p99 by prompt cohort;
3. TPOT/ITL p50/p95/p99 by output cohort;
4. E2E and token throughput;
5. queue/running/waiting requests;
6. KV cache utilization and preemptions;
7. 401/403/429/5xx and retry amplification;
8. GPU VRAM/utilization/temperature/XID/ECC;
9. Pod readiness, restarts and model revision;
10. GPU-hours per valid completed request;
11. scrape freshness and trace/log ingestion health.

Prefer bounded-cardinality labels and retention-aware histogram buckets.

## 11. Read-only operational checks

**[SAFE-READ]** in an authorized cluster:

```bash
kubectl config current-context
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

For a verified serving deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

On an approved GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Use redacted log views; do not export Secrets, request bodies or sensitive model output.

## 12. Incident correlation decision tree

```text
Customer inference degraded?
 ├─ complete SSE fails → gateway, cancellation, rank loss
 ├─ TTFT high → queue, tokenization, prefill
 ├─ TPOT high → decode, KV, GPU memory bandwidth
 ├─ queue high + GPU low → scheduler/routing/CPU
 ├─ preemption high → KV oversubscription
 ├─ 429 high → quota vs overload
 ├─ 502/503/504 high → ingress/backend/timeout
 ├─ GPU XID/ECC → hardware/driver
 ├─ logs absent → collection/retention/pipeline
 ├─ traces disconnected → context propagation/sampling
 ├─ metrics flatlined → scrape freshness/label drift
 └─ GPU cost high → retries, idle replicas, long outputs
```

## 13. Rollout and rollback

1. Inventory pinned vLLM/exporter/Prometheus/OTel versions.
2. Verify security and privacy of proposed labels, logs and spans.
3. Record baseline metric schema, units, cardinality and scrape interval.
4. Deploy telemetry changes to approved staging only.
5. Run synthetic short/long prompts and complete-stream tests.
6. Confirm metrics, logs, traces and DCGM correlate without leaking data.
7. Test missing scrape, collector outage, partial SSE and error-budget burn in tabletop.
8. Canary dashboard/alert changes with owner and noise thresholds.
9. Revert through authoritative GitOps/observability controller on regression.
10. Verify alert delivery, SLO recovery and audit trail.

## 14. Twenty-four production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | metric renamed after upgrade | schema diff |
| F02 | histogram unit mismatch | bucket audit |
| F03 | counter reset misinterpreted | restart history |
| F04 | stale scrape treated as zero | target freshness |
| F05 | missing exporter target | service discovery |
| F06 | excessive label cardinality | TSDB growth |
| F07 | raw prompt used as metric label | privacy audit |
| F08 | bearer token in logs | log scan |
| F09 | PHI in trace attributes | trace audit |
| F10 | trace context not propagated | span linkage |
| F11 | trace sampling hides failures | sampling policy |
| F12 | OTel collector backpressure | queue/drop metrics |
| F13 | log pipeline outage | ingestion freshness |
| F14 | DCGM exporter incompatible | exporter logs |
| F15 | XID/ECC metric missing | GPU support |
| F16 | TTFT measured at wrong boundary | timing definition |
| F17 | TPOT includes metadata chunks | client instrumentation |
| F18 | HTTP 200 partial SSE counted success | terminal validator |
| F19 | 429 quota and overload conflated | error taxonomy |
| F20 | queue and GPU signals disagree | correlation |
| F21 | model revision absent from telemetry | release inventory |
| F22 | alert fires on low sample count | burn configuration |
| F23 | missing-data alert absent | alert audit |
| F24 | observability cost spikes | retention/cardinality |

## 15. Sixteen operational runbooks

1. Validate SLI definitions and eligible requests.
2. Audit pinned exporter schema and units.
3. Diagnose missing/stale Prometheus scrapes.
4. Diagnose counter reset and histogram errors.
5. Investigate metric cardinality explosion.
6. Triage sensitive logging/tracing leakage.
7. Diagnose trace propagation and sampling.
8. Diagnose OTel collector drops/backpressure.
9. Restore log ingestion and retention.
10. Investigate DCGM exporter and GPU telemetry.
11. Correlate TTFT with queue/prefill.
12. Correlate TPOT with decode/KV/GPU.
13. Investigate partial SSE and 429/5xx.
14. Validate SLO burn alerts and low traffic.
15. Execute observability rollback.
16. Verify end-to-end telemetry and RCA.

## 16. Eight tabletop game days

1. vLLM upgrade renames key metrics.
2. Prometheus target goes stale during outage.
3. A prompt appears in a trace attribute.
4. OTel collector drops spans under load.
5. HTTP 200 partial SSE is incorrectly counted successful.
6. DCGM exporter disappears while GPU faults occur.
7. Burn alert fires repeatedly on one low-traffic request.
8. High-cardinality labels exhaust telemetry budget.

Default **tabletop**; live failure injection requires approved isolated staging.

## 17. Twenty acceptance gates

G01 pinned schema; G02 SLI definitions; G03 complete SSE; G04 TTFT boundary; G05 TPOT definition; G06 queue metrics; G07 KV/preemption; G08 GPU/DCGM; G09 scrape freshness; G10 histogram units; G11 cardinality; G12 structured logs; G13 trace propagation; G14 privacy; G15 SLO burn; G16 missing-data alert; G17 safe rollout; G18 rollback; G19 telemetry FinOps; G20 independent review.

See [companion lab](../labs/023-vllm-metrics-logging-distributed-tracing-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 024 — vLLM Slowness, CUDA OOM and Recovery Runbooks.**
