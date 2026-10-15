# Chapter 11.1 — Observability Architecture and Telemetry Foundations

**Part 11 — Observability | Canonical TrueFoundry SRE tutorial**

## Objectives and boundaries

Build a production-ready observability architecture for TrueFoundry workloads on Kubernetes, including cluster agent health, application/model-serving metrics, GPU signals, logs, traces, SLOs and actionable alerts. Distinguish **platform control plane**, **Kubernetes cluster agent**, **data-plane workloads**, **inference gateway**, **vLLM/model server** and **cloud dependencies**. Verify available TrueFoundry integrations and exact metrics against your deployed version; do not assume the platform emits any particular metric.

**Safety:** [SAFE-READ] observation; [LAB-MUTATE] disposable staging; [CHANGE-CONTROL] production instrumentation and alert changes. Do not collect raw prompts, responses, PHI, API keys, tokens or model weights in shared telemetry.

## 1. Architecture and telemetry flow

```text
Users → Gateway / Load Balancer → Inference Service → vLLM / Model Runtime
             │                         │               │
             └─ request/error/latency ─┴─ runtime ──────┘
Kubernetes API / Events ─┐
kube-state-metrics ──────┤
Node exporter / cAdvisor┤
GPU DCGM exporter* ──────┤→ Prometheus / OTel Collector → TSDB / logs / traces
TrueFoundry agent logs ──┤                                ↓
App structured logs ─────┤                           dashboards / SLOs
OTel traces ─────────────┘                                ↓
                                                alerts → on-call / runbooks
*Install/exporter support depends on GPU stack and policy.
```

Logs explain discrete events, metrics measure aggregate behavior, traces connect request stages, and Kubernetes Events capture short-lived orchestration symptoms. Events are not durable log storage; export them if long-term incident correlation is required.

## 2. Golden signals and RED/USE

**Gateway and inference RED:** request rate, error rate, duration (p50/p95/p99), separated by route/model/version and safe tenant identifiers. Define 4xx treatment deliberately: authentication failures and client errors may be excluded from availability SLI while still tracked as security/product signals. Distinguish streaming time-to-first-token (TTFT) from full completion duration.

**Resource USE:** utilization, saturation and errors for CPU, memory, disk, network and GPU. For vLLM/LLMs, add queue depth, running/waiting requests, tokens/sec, prefill/decode latency, KV-cache pressure and GPU memory, but use only version-supported exporter metric names.

**Platform:** agent availability/restarts, deployment reconciliation lag, Kubernetes API error/429 rate, Pod readiness, rollout duration and image/model download errors. Cloud dependencies: IAM denied actions, object store failures, secret sync, load balancer 5xx and registry availability.

## 3. Safe discovery [SAFE-READ]

```bash
kubectl config current-context
kubectl get namespaces
kubectl get pods -A -o wide
kubectl get deployments,daemonsets -A
kubectl get svc -A
kubectl get events -A --sort-by=.lastTimestamp | tail -50
kubectl get --raw /metrics >/dev/null 2>&1 && echo 'API metrics endpoint reachable' || true
kubectl top nodes 2>/dev/null || true
kubectl top pods -A 2>/dev/null || true
```

`kubectl top` requires metrics-server and appropriate permissions; absence is not proof of zero utilization. Query a specific approved agent deployment only after confirming namespace and name:
```bash
kubectl -n AGENT_NAMESPACE logs deploy/AGENT_DEPLOYMENT --since=15m --tail=100
kubectl -n AGENT_NAMESPACE get deploy AGENT_DEPLOYMENT -o wide
```
Review log content for sensitive data before exporting.

## 4. OpenTelemetry and Prometheus instrumentation

Choose a consistent telemetry SDK/collector path. Use W3C trace context across gateway, app and model server where supported. Propagate correlation IDs without carrying user prompts in baggage. Configure OTLP transport, mTLS/auth, resource attributes (cluster, namespace, service, deployment version), sampling and exporter retry/backpressure.

Prometheus scraping: confirm ServiceMonitor/PodMonitor CRDs or use native scrape configuration. Scope scrape endpoints, apply network policy and authenticate sensitive endpoints. Recording rules should aggregate before dashboards where practical.

Illustrative PromQL (replace with verified metric names):
```promql
sum(rate(http_requests_total{job="inference",status=~"5.."}[5m]))
/
clamp_min(sum(rate(http_requests_total{job="inference"}[5m])), 1e-9)
```
```promql
histogram_quantile(0.95,
  sum by (le, service) (rate(http_request_duration_seconds_bucket{job="inference"}[5m]))
)
```
Histograms require compatible bucket boundaries and labels; never average percentile series. For native histograms, use appropriate histogram functions. Verify the exact `status`/duration label and units.

## 5. GPU and LLM-specific observability

Where approved, collect NVIDIA DCGM exporter signals: GPU utilization, framebuffer memory, power/thermal throttling, ECC/XID errors and device health. Metric availability depends on driver, GPU, MIG, exporter configuration and node permissions. Correlate GPU saturation with token throughput, TTFT, queue length, batch scheduling and model version. High GPU utilization alone is not a failure; investigate throughput, errors and user latency together.

## 6. Label cardinality, privacy and cost

**Never** label metrics with request IDs, user IDs, prompts, session IDs or unbounded tenant values. Use trace attributes only with approved cardinality and redaction. Control retention, scrape interval, histogram buckets, sampling, log levels and high-cardinality dimensions. Monitor collector queue/drops and TSDB ingestion cost. Treat telemetry endpoints and stored traces as sensitive production data.

## 7. Dashboards and SLO foundations

Minimum dashboards:
- **Platform:** agent availability, reconcile/rollout failures, API throttling, Pod health.
- **Serving:** traffic, error ratio, p95/p99, TTFT, token throughput, queues, saturation.
- **GPU:** allocated devices, utilization, memory, throttling, XID/ECC, node scheduling.
- **Dependencies:** storage/model pulls, cloud IAM, DNS/TLS, secret sync, gateway and registry.
- **Telemetry pipeline:** scrape failures, collector queue/drop rate, log lag, trace sampling and ingestion health.

Define SLIs with numerator/denominator, measurement window and exclusions. Example availability: successful eligible inference requests divided by eligible inference requests. Use multi-window burn-rate alerts after SLO/error budget approval; avoid arbitrary global thresholds.

## 8. Twelve troubleshooting runbooks

1. **No metrics:** confirm endpoint, Service/Pod labels, scrape config, RBAC, TLS and network policy.
2. **Missing GPU metrics:** confirm exporter/driver/MIG compatibility, DaemonSet placement and permission.
3. **Agent telemetry silent:** check agent health, log shipping, cluster connectivity and collector pipeline.
4. **High p99 with low CPU:** inspect queueing, GPU saturation, model loading and dependency latency.
5. **TTFT spike:** inspect prefill load, queue depth, input tokens, batching and network.
6. **Low tokens/sec:** compare GPU memory, model config, decode throughput and request concurrency.
7. **High 5xx:** segment gateway, application, runtime, upstream and rollout changes.
8. **Prometheus high cardinality:** identify unbounded labels, drop at source, estimate cost and validate queries.
9. **Trace gaps:** verify propagation, sampling, exporter retry and missing spans.
10. **Logs missing after restart:** verify stdout collection, retention, collector DaemonSet and buffering.
11. **Alert storm:** examine grouping, inhibition, recording rules, thresholds and SLO relevance.
12. **Observability outage:** use independent Kubernetes/cloud telemetry, restore collector/TSDB and record blind window.

## 9. Production acceptance and rollback

Before enabling in production: inventory actual exporters, verify access and redaction, load-test collection overhead, check metric labels/units, define dashboards and SLO owners, test one actionable alert and runbook, verify telemetry retention/cost, and document rollback of collectors/configuration. Do not roll back security-critical audit collection merely to reduce monitoring load.

Complete the [hands-on lab](../labs/01-observability-architecture-telemetry-foundations-check.md), including **24 failure scenarios and 10 acceptance gates**. Formal technical review and live execution remain pending.

**Next:** Chapter 11.2 — Logging Architecture, Structured Logs and Kubernetes Events.
