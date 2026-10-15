# 6.6 — Model-Serving Observability, Performance and Troubleshooting

**Part:** 06 — Model Serving · **Chapter:** 6 of 7  
**Audience:** TrueFoundry platform engineers, MLOps, GPU engineers, incident commanders and SREs  
**Safety:** Observations are **[SAFE-READ]**. Load tests, alert/routing changes, resource adjustments, model changes and rollbacks require **[CHANGE-CONTROL]** in production or **[LAB-MUTATE]** in approved nonproduction.

## 1. Learning objectives

Build an end-to-end inference observability model across TrueFoundry, Kubernetes, gateways, model runtimes and accelerators; design actionable SLOs and alerts; interpret latency, queueing, throughput, error, CPU and GPU signals; distinguish saturation from cold starts and routing faults; and lead evidence-based incident recovery.

## 2. Telemetry architecture

```text
Authorized inference clients
    │ request ID / trace context
    ▼
DNS/TLS → Gateway/Auth → Service/EndpointSlice → Serving Pod
                                           │
                              Admission queue / batching
                                           │
                              Preprocess → Model → Response
                                           │
           ┌───────────────────────────────┼───────────────────────┐
           ▼                               ▼                       ▼
      Metrics exporter                OTel traces            Structured logs
           │                               │                       │
   Prometheus/compatible TSDB       Trace collector          Log pipeline
           │                               │                       │
           └────────────── Grafana / dashboards / alerts ─────────┘
                                           ▲
                    Kubernetes events / GPU DCGM telemetry
```

TrueFoundry dashboards and supported metrics integrations depend on installed platform version. The Kubernetes and application layers remain the sources of truth for actual Pod behavior and request results. Verify each metric's exporter, scrape interval, labels, units and cardinality before building alerts.

## 3. Golden signals for model serving

| Domain | Signals | Why they matter |
|---|---|---|
| Traffic | RPS, tokens/s, requests by model/revision | Workload mix and demand |
| Errors | 5xx, timeouts, model exceptions, invalid output, failed streams | Availability and correctness |
| Latency | P50/P95/P99 end-to-end, queue, execution, TTFT, inter-token latency | User experience |
| Saturation | In-flight requests, queue depth, CPU throttling, GPU busy, VRAM, rejection rate | Capacity limit |
| Startup | Model fetch/load/warmup, startup probe, Ready transition | Cold-start availability |
| Kubernetes | Pending, restarts, OOMKilled, unavailable replicas, endpoints | Workload reliability |
| Artifact | Download errors, checksum failures, cache misses | Model supply chain |
| Cost | GPU-hours, utilization, throughput per GPU-hour, storage/egress | Efficiency |

HTTP 200 does not guarantee model correctness. Include approved golden-input or quality validation signals with appropriate privacy protections. Streaming services should measure completion and midstream errors, not just initial HTTP status.

## 4. SLI and SLO design

Define eligible requests and exclusions before computing SLOs. Example:

- **Availability SLI:** successful eligible inference responses / eligible inference requests, where success includes contract-valid output and completed stream when applicable.
- **Latency SLI:** fraction of eligible successful requests completed within a defined end-to-end threshold.
- **Correctness gate:** golden test pass rate on approved non-sensitive test cases.
- **Freshness/version gate:** response reports expected immutable model version.
- **Cold-start objective:** P95/P99 time from requested new serving capacity to model-ready endpoint.

Illustrative SLOs, **not universal targets**: 99.9% availability over 30 days, 95% of eligible synchronous requests under 500 ms, and 99% successful golden checks. Tune to model size, input distribution, user needs and costs. For a 99.9% 30-day SLO, the error budget is 0.1% of eligible requests, not a fixed outage duration when traffic varies.

## 5. Prometheus instrumentation patterns

Use stable, low-cardinality labels such as `service`, `namespace`, `model_id`, `model_version`, `revision`, `status_class`. **Never label metrics with request IDs, user IDs, raw prompts or arbitrary model inputs.**

Recommended **illustrative application-owned metric names** (not guaranteed TrueFoundry built-ins):

```text
inference_requests_total{service,model_id,model_version,status_class}
inference_request_duration_seconds_bucket{service,model_id,model_version,le}
inference_queue_duration_seconds_bucket{service,model_id,le}
inference_execution_duration_seconds_bucket{service,model_id,le}
inference_inflight_requests{service,model_id}
inference_queue_depth{service,model_id}
model_load_duration_seconds_bucket{service,model_id,le}
model_ready{service,model_id,model_version}
inference_stream_completions_total{service,model_id,result}
```

Prometheus histograms should have buckets appropriate to actual model latency and traffic. Cardinality grows with models, versions, namespaces, paths and tenants; use bounded dimensions and retention controls.

### Example PromQL: request rate and 5xx fraction

```promql
sum(rate(inference_requests_total[5m]))
```

```promql
sum(rate(inference_requests_total{status_class="5xx"}[5m]))
/
clamp_min(sum(rate(inference_requests_total[5m])), 1e-9)
```

If your metrics expose HTTP status differently, adjust labels. Use `increase()` and minimum traffic guards for low-volume services to avoid noisy ratios.

### Example PromQL: P95 end-to-end latency

```promql
histogram_quantile(
  0.95,
  sum by (le, service) (
    rate(inference_request_duration_seconds_bucket[5m])
  )
)
```

The query assumes classic histograms with `le` buckets; native histograms and summaries require different treatment. Do not average P95 values across replicas.

## 6. Kubernetes and GPU telemetry

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,replicasets,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> logs <pod> --tail=150
kubectl -n <namespace> logs <pod> --previous --tail=150
kubectl -n <namespace> top pods --containers
```

Metrics Server and previous logs may be unavailable. For GPU, use approved NVIDIA DCGM exporter or equivalent telemetry if deployed. Observe GPU utilization, framebuffer/VRAM memory, power, temperature, ECC errors where supported and device health. **GPU utilization is not a throughput or correctness metric.** High VRAM allocation does not necessarily mean a memory leak, and low GPU busy with high P99 can indicate CPU preprocessing, network, admission queue or tiny batches.

## 7. Distributed tracing

Propagate a trace context through gateway, inference server and downstream calls. Recommended spans:

```text
client request
  └─ gateway/auth
      └─ inference admission / queue
          ├─ model artifact lookup (only if request-time)
          ├─ preprocess / tokenize
          ├─ batch wait
          ├─ runtime execution
          └─ postprocess / serialize / stream
```

Sampling must balance incident usefulness and cost. Record model ID/version, revision, status, duration and sanitized error type; do not capture raw patient/customer inputs, prompts, model outputs, tokens or credentials. A trace without a request's queue span can falsely attribute high latency to the model itself.

## 8. Logs, correlation and privacy

Structured log fields: timestamp, severity, service, namespace, revision, model ID/version, request/trace ID, stage, duration, sanitized error code and retry count. Keep access tokens, secrets, input payloads and model outputs out of routine logs. Audit who can access model artifacts and sensitive inference telemetry.

Use consistent clocks and correlation IDs; node and application timestamps can differ. Preserve only approved diagnostic evidence during an incident.

## 9. Performance baselines and capacity

Create baselines for each model version, hardware SKU, precision, input size, batch policy, concurrency, cache state and runtime version. Measure:

- Throughput at acceptable P95/P99 latency, not maximum throughput alone.
- Queue wait and admission rejections as load rises.
- CPU preprocessing and cgroup throttling separately from GPU execution.
- VRAM peaks and model/KV-cache footprint for concurrent workloads.
- Model cold-start P95 and replica scale-up delay.
- GPU cost per successful inference and per approved workload unit.

Use controlled step-load tests in nonproduction with synthetic inputs. Record exact concurrency, duration, warmup, payload distribution and sample size. Averages hide queue spikes and latency tails.

## 10. Dashboard design

**Panel A — Client experience:** eligible RPS, 2xx/4xx/5xx, timeout and 429 rates, P50/P95/P99, TTFT/stream completion.  
**Panel B — Inference execution:** queue depth/wait, batch size, execution latency, in-flight requests, token throughput.  
**Panel C — Model lifecycle:** model version, load/warmup duration, Ready replicas, failed downloads and cache hit rate.  
**Panel D — Infrastructure:** Pod Pending/restarts/OOM, CPU/RAM, GPU/VRAM, node allocation, Service endpoints.  
**Panel E — Change correlation:** image/model revision, deployment timestamps, scaling, feature/config changes and alert annotations.

Always show time ranges, units and revision breakdown. A cluster-wide aggregate can hide one bad model or GPU node.

## 11. Alert design and burn rates

Alert on **user-impacting symptoms** first: sustained eligible error fraction, P99/latency SLO violations, zero ready endpoints, failed streams and excessive rejections. Pair with diagnostic alerts for model load failures, GPU device errors, repeated OOM, persistent queue growth and long Pending time.

For a 99.9% availability SLO, an illustrative 14.4× burn rate corresponds to a 1.44% error fraction over the evaluated window. Multiwindow alerts (e.g., 1h plus 5m) reduce noise; select windows and thresholds based on actual traffic, alert ownership and incident policy. Low-volume services need minimum-event guards or synthetic checks.

Avoid paging solely on a brief GPU utilization spike, a single failed startup probe or an isolated model cache miss.

## 12. Diagnostic decision tree

```text
User impact?
  ├─ No → inspect trends / prevent capacity risk
  └─ Yes
      ├─ No ready endpoints → Pod state, probes, model load, scheduler
      ├─ High 4xx → client schema, auth, quota policy
      ├─ High 5xx/504 → gateway, queue, runtime, dependencies
      ├─ High P99 + low GPU busy → network, auth, CPU, queue, small batches
      ├─ High P99 + high GPU busy → execution capacity, batching, VRAM
      ├─ 429 rising → admission limits and tenant load
      ├─ Only new revision → image/model/config regression
      └─ Correct status but bad predictions → model/schema/correctness
```

Find the **Lowest Proven Healthy Layer** and **First Failed Transition** before modifying capacity or rolling back.

## 13. Troubleshooting matrix

| Symptom | Correlation | First investigation |
|---|---|---|
| Sudden 503 | Ready endpoints zero | Probe/model load/selector |
| 504 with healthy Pods | Queue or execution exceeds deadline | Gateway and runtime spans |
| P99 rises, CPU throttles | High preprocess time | CPU limits/threads |
| P99 rises, GPU idle | Queue/network/batching | Request stage latency |
| P99 rises, GPU saturated | Throughput plateau, queue growth | Capacity/batch/concurrency |
| GPU OOM | VRAM peak and runtime exception | Model footprint/KV cache |
| Host OOMKilled | Pod termination reason | RAM/cache/worker count |
| Cold-start spikes | Scale-up and model downloads | Artifact/storage/init |
| Model download 403 | Identity audit and loader errors | Scoped workload identity |
| CUDA initialization error | Runtime/device plugin | Driver/CUDA/framework |
| High 429 | Tenant concurrency/quota | Admission policy |
| Retry storm | Gateway/client retries | Retry budgets/backoff |
| Correct HTTP, wrong result | Golden test failure | Model version/schema |
| Stream ends early | Completion and cancellation spans | Gateway idle/stream handling |
| Metrics missing | Scrape/exporter configuration | Target and service monitor |
| Cardinality explosion | Unbounded labels | Instrumentation labels |

## 14. Production incident runbook

1. **Declare impact:** affected models/tenants, onset, eligible request error fraction, P95/P99, throughput, 429/5xx and stream completion.
2. **Correlate changes:** TrueFoundry revision, image digest, immutable model version, runtime, GPU node pool and rollout timestamp.
3. **Localize:** client → gateway → auth → ready endpoints → queue → CPU/GPU execution → response.
4. **Contain:** pause promotion, limit retry amplification, preserve known-good replicas and enforce approved admission controls.
5. **Repair:** use measured evidence for CPU/GPU sizing, model load, routing, queue, probe or storage fixes. Avoid disabling security, probes or telemetry.
6. **Rollback:** restore the authoritative compatible image + model + resource + endpoint configuration.
7. **Validate:** golden inference, model version, Ready endpoints, error/latency SLO, queue stability, memory/GPU health and no recurrent restarts.
8. **Close:** RCA, corrective/preventive actions, dashboard gaps, alert tuning, owners and follow-up.

## 15. Acceptance gates

| Gate | Required evidence |
|---|---|
| G1 | Client SLI/SLO definitions and eligibility documented |
| G2 | Metrics, labels, units and histogram semantics verified |
| G3 | Gateway-to-runtime traces and sanitized logs correlated |
| G4 | CPU/RAM and GPU/VRAM observability validated |
| G5 | Cold-start, queue, latency and error baselines captured |
| G6 | Actionable symptom-first alerts and burn-rate policy reviewed |
| G7 | Failure isolation uses lowest healthy layer / first failed transition |
| G8 | Authoritative rollback and post-recovery SLO verified |

## 16. Evidence template

```text
Date/time / incident owner / severity:
TrueFoundry workspace, cluster, deployment:
Image digest / immutable model version / revision:
Eligible request count, errors, 429/5xx, timeouts:
P50/P95/P99, TTFT and stream completion:
Queue wait/depth, batch, execution latency:
CPU/RAM/throttling, GPU/VRAM/device health:
Model load, warmup, cache and readiness:
Gateway/Service/EndpointSlice:
Trace IDs and sanitized log references:
Lowest Proven Healthy Layer:
First Failed Transition:
Change correlation / approved remediation:
Known-good rollback / observation window:
Post-recovery SLO / follow-up owner:
```

**Hands-on lab:** [6.6 — Observability, Performance and Incident Diagnosis](../labs/06-model-serving-observability-performance-troubleshooting-check.md)  
**Next:** Chapter 6.7 — Production Model-Serving Deployment and Recovery Capstone.

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Prometheus Histograms and Summaries](https://prometheus.io/docs/practices/histograms/)
- [Prometheus Alerting Best Practices](https://prometheus.io/docs/practices/alerting/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Kubernetes Resource Metrics Pipeline](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
