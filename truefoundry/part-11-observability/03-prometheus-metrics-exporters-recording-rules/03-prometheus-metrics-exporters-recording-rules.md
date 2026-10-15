# Chapter 11.3 — Prometheus Metrics, Exporters and Recording Rules

**Part 11 — Observability | Canonical production SRE tutorial**

## 1. Learning objectives and safety

Build an operational Prometheus monitoring stack for TrueFoundry-managed Kubernetes and inference workloads. Discover metrics sources, configure scrape targets, choose exporters, write safe PromQL and recording rules, manage cardinality and retention, and troubleshoot failures with evidence.

**Safety:** [SAFE-READ] inspection; [LAB-MUTATE] disposable staging; [CHANGE-CONTROL] production scrape/rule/retention changes. Verify installed Prometheus, Prometheus Operator, kube-state-metrics, node-exporter, GPU/DCGM exporter, and the deployed TrueFoundry/vLLM versions. Do not assume any product-specific metric exists without querying its endpoint. Never put prompts, user IDs, tokens or request IDs in metric labels.

## 2. Metrics architecture and ownership

```text
Kubernetes API / kube-state-metrics ──┐
Node exporter / kubelet-cAdvisor ────┤
GPU DCGM exporter (optional) ────────┤
TrueFoundry agent metrics (if any) ──┤ → Prometheus scrape → TSDB
Gateway / application / vLLM ────────┤                    ├→ recording rules
Blackbox / cloud exporters ──────────┘                    ├→ Alertmanager
                                                         └→ Grafana
```

Prometheus pulls metrics from targets. Kubernetes metrics-server serves `kubectl top` and is not a replacement for a long-term Prometheus TSDB. OpenTelemetry metrics can reach Prometheus through supported exporters or remote write architectures, but translation, temporality and histogram handling must be validated. Cloud-managed Prometheus can differ in service discovery, auth, quotas and supported features.

## 3. Safe discovery [SAFE-READ]

```bash
kubectl config current-context
kubectl get ns
kubectl get pods -A -o wide
kubectl get svc -A
kubectl get servicemonitors,podmonitors -A 2>/dev/null || true
kubectl get prometheusrules -A 2>/dev/null || true
kubectl get deployments,daemonsets -A
kubectl top nodes 2>/dev/null || true
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

Optional custom resources require the Prometheus Operator CRDs. Verify target and service labels, ports, endpoints, TLS/auth and network policy. For a specific metric endpoint, use an approved authenticated port-forward or read-only Prometheus UI; avoid exposing `/metrics` publicly.

## 4. Instrumentation and metric types

- **Counter:** monotonic event count except process resets. Apply `rate` or `increase` before aggregating across instances.
- **Gauge:** instantaneous measurement such as queue depth or memory usage. Avoid treating it as a counter.
- **Histogram:** buckets, count and sum for request duration/size; aggregate buckets by `le` before `histogram_quantile` for classic histograms.
- **Summary:** client-computed quantiles generally cannot be aggregated meaningfully across instances. Use histograms for fleet p95/p99.
- **Native histogram:** version/feature support and PromQL functions differ; validate compatibility.

Use base units in metric names (e.g. `_seconds`, `_bytes`, `_total`) and document counter labels, reset behavior, ownership and cardinality budgets. Avoid embedding status/error message text as labels.

## 5. Exporters and Kubernetes metric sources

**kube-state-metrics:** desired/actual Kubernetes object state, workload readiness, restarts, resource requests and deployment rollout metadata. It is not a CPU usage source.

**node-exporter:** host CPU, memory, filesystem and network signals; node filesystems and host mounts may require security review.

**kubelet/cAdvisor:** container resource utilization; available series and labels vary by Kubernetes/runtime and managed provider.

**DCGM exporter:** GPU utilization, memory, thermal/power and errors where supported; MIG labels and driver compatibility require validation.

**Blackbox exporter:** synthetic availability/TLS/HTTP checks. Avoid probing private endpoints with unaudited credentials.

**App/model server:** request rate, 5xx, duration, queueing, TTFT, throughput and model loading. vLLM metric names and dimensions vary by version.

## 6. PromQL production examples

**Target health:**
```promql
up{job="inference"}
```

**Per-second request rate (illustrative metric):**
```promql
sum by (service) (rate(http_requests_total{job="inference"}[5m]))
```

**5xx error ratio:**
```promql
sum(rate(http_requests_total{job="inference",status=~"5.."}[5m]))
/
clamp_min(sum(rate(http_requests_total{job="inference"}[5m])), 1e-9)
```

**Classic histogram p95:**
```promql
histogram_quantile(
  0.95,
  sum by (le,service) (rate(http_request_duration_seconds_bucket{job="inference"}[5m]))
)
```

**CPU usage per Pod (example; metric/labels depend on scrape):**
```promql
sum by (namespace,pod) (
  rate(container_cpu_usage_seconds_total{container!="",image!=""}[5m])
)
```

**Memory working set:**
```promql
sum by (namespace,pod) (
  container_memory_working_set_bytes{container!="",image!=""}
)
```

**Container restarts:**
```promql
sum by (namespace,pod) (
  increase(kube_pod_container_status_restarts_total[15m])
)
```

Interpret no data separately from zero. For sparse traffic, error ratios can be unstable; gate alerts on sufficient request volume. Use `absent`/target health for missing-series detection. Do not average p95 values from multiple instances.

## 7. Recording rules

Recording rules precompute frequently used or expensive expressions. Example for a known HTTP counter; adapt names/labels to verified instrumentation:

```yaml
groups:
  - name: inference-recording
    interval: 30s
    rules:
      - record: service:http_requests:rate5m
        expr: sum by (service) (rate(http_requests_total{job="inference"}[5m]))
      - record: service:http_5xx:ratio5m
        expr: |
          sum by (service) (rate(http_requests_total{job="inference",status=~"5.."}[5m]))
          /
          clamp_min(sum by (service) (rate(http_requests_total{job="inference"}[5m])), 1e-9)
      - record: service:http_request_duration_seconds:p95_5m
        expr: |
          histogram_quantile(0.95,
            sum by (le,service) (rate(http_request_duration_seconds_bucket{job="inference"}[5m])))
```

For `PrometheusRule` CRDs, wrap `groups` under `spec` and add required `apiVersion`, `kind`, `metadata` and Operator label selectors. Check `promtool check rules` and rule discovery before promotion. Recording rules do not automatically become alerts; alert rules must define `alert`, `expr`, optional `for`, labels and annotations.

## 8. Cardinality, storage and federation

Approximate active series as the product of unique label combinations, not merely metric count. Avoid `request_id`, raw `user_id`, prompt text, URLs with IDs, unbounded `model_version` or arbitrary tenant labels. Review `/api/v1/status/tsdb` access, top label cardinality, sample ingestion, head memory, WAL pressure, retention and remote-write queue.

HA Prometheus replicas need deduplication at the query layer when remote writing. Federation and remote write have different semantics; use them intentionally. Keep long-term retention, backup/restore and remote-write failure plans distinct.

## 9. Operational dashboards and alert principles

Dashboard panels: scrape health, sample ingestion, rule evaluation duration, WAL/storage, remote-write failures, node/Pod health, GPU health, gateway errors, p95/p99, TTFT and queue depth. Alerts must be actionable, routed to owners and linked to runbooks. Define SLO burn-rate alerts in Chapter 11.6; a raw threshold alone is not a reliability objective.

## 10. Twelve production troubleshooting runbooks

1. **Target DOWN:** inspect Service endpoints, port, scrape path, TLS/auth, network policy and target labels.
2. **No ServiceMonitor discovery:** verify CRDs, Prometheus selector/namespace selectors and labels.
3. **No metrics despite UP:** check exporter output, endpoint path and relabeling/metric drop rules.
4. **PromQL empty:** validate exact metric/labels, scrape interval and staleness before assuming zero.
5. **Counter rate spike:** inspect restarts, resets, sparse samples and rate window.
6. **Incorrect p95:** check classic bucket `le`, units, histogram schema and grouping.
7. **Rule not evaluated:** verify `promtool`, PrometheusRule selection, rule group and evaluation errors.
8. **Rule evaluation slow:** simplify expression, preaggregate and inspect TSDB cardinality.
9. **High cardinality/OOM:** identify label explosion, fix instrumentation and apply approved retention/series controls.
10. **Remote write lag:** inspect queue, network/auth, receiver throttling, backpressure and data-loss window.
11. **Missing GPU metrics:** inspect DCGM exporter, driver/MIG compatibility, node placement and device plugin.
12. **Prometheus storage full:** assess WAL/retention, disk, ingest, safe capacity expansion and incident fallback; do not delete TSDB files ad hoc.

## 11. Acceptance and change management

Deliver exporter inventory, target discovery proof, verified metric dictionary, recording rules, `promtool` validation, dashboard links, cardinality/storage budget, alert owner, backup/rollback and incident evidence. Test staged changes before production. Security-sensitive metrics endpoints must remain private.

Complete the [hands-on lab](../labs/03-prometheus-metrics-exporters-recording-rules-check.md) with **30 failure scenarios and 10 acceptance gates**. Live execution and formal review remain pending.

**Next:** Chapter 11.4 — OpenTelemetry Distributed Tracing and Correlation.
