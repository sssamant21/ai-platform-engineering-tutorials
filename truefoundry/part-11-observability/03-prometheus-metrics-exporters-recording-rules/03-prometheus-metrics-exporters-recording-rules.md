# 11.3 — Prometheus Metrics, Exporters and Recording Rules

**Part 11 — Observability | Chapter 3 of 7**  
**Audience:** TrueFoundry platform/SRE engineers, Kubernetes and GPU operators, inference reliability teams.  
**Safety:** **[SAFE-READ]** read-only; **[LAB-MUTATE]** isolated approved staging; **[CHANGE-CONTROL]** production.

## 1. Learning objectives

Design an end-to-end Prometheus metrics pipeline; distinguish scrape targets, ServiceMonitor/PodMonitor, exporters, remote write and dashboards; collect Kubernetes, TrueFoundry integration, NVIDIA GPU and vLLM signals; write safe PromQL for rates, errors, latency and saturation; implement recording rules with correct aggregation; troubleshoot missing, stale, high-cardinality or misleading metrics; and validate operational acceptance.

**Prerequisites:** Chapters 11.1–11.2, read-only cluster/Prometheus access, approved namespace, inventory of Prometheus Operator or alternative deployment, GPU exporter status, model endpoint, data retention and monitoring owner. Metric names and TrueFoundry-managed components are **version dependent**: discover before assuming.

## 2. Reference metrics architecture

```text
Kubernetes API/kube-state-metrics/node exporter
               │
TrueFoundry integration / workload metrics
               │
vLLM /metrics   │    NVIDIA DCGM exporter
       └────────┼───────────┘
                ▼
Prometheus scrape / service discovery / relabel
                │
          TSDB and rules
                │
       ┌────────┴─────────┐
       ▼                  ▼
Grafana dashboards   Alertmanager / approved routing
       │
       └── optional remote_write → long-term metrics backend
```

The TrueFoundry UI is not necessarily the metrics store. Some platforms use managed Prometheus or OpenTelemetry collectors instead of an in-cluster Prometheus server. Inventory the real path.

## 3. Metrics types and interpretation

| Type | Example | Correct interpretation |
|---|---|---|
| Counter | requests_total, tokens_generated_total | Use `rate()` or `increase()`; reset on restart |
| Gauge | running requests, queue depth, GPU memory used | Instantaneous; use time windows appropriately |
| Classic histogram | request_duration_seconds_bucket | Aggregate `rate(bucket)` before `histogram_quantile` |
| Native histogram | implementation dependent | Use compatible PromQL/functions and storage |
| Summary | quantile exposed by a single instance | Quantiles generally cannot be aggregated across replicas |
| Info | model/runtime build information | Avoid unbounded revision/tenant labels |

**Unit contract:** distinguish seconds vs milliseconds, GPU bytes vs MiB, output vs input tokens, requests/s vs tokens/s, and TTFT vs ITL/TPOT vs end-to-end latency.

## 4. Exporter and source inventory

| Source | Key questions | Representative signals |
|---|---|---|
| kube-state-metrics | RBAC and namespace coverage? | deployment replicas, Pod readiness, HPA state |
| node exporter | Node scrape reachable? | CPU, memory, filesystem, network |
| kubelet/cAdvisor | Distribution and version? | container CPU/memory/restarts |
| NVIDIA DCGM exporter | GPU Operator or standalone? | GPU utilization, framebuffer memory, health |
| vLLM | Does installed version expose /metrics? | request queue, running requests, token counters, latency |
| TrueFoundry components | Which metrics are actually exposed? | reconciliation/deployment health if available |
| Gateway/Ingress | Which controller? | 4xx/5xx, upstream latency, active streams |
| Cloud autoscaler | Which provider? | node provisioning failures, GPU Pending duration |

Check actual exporter documentation and live `/metrics` metadata before using any example metric name. Never assume DCGM is installed merely because GPU workloads run.

## 5. Discovery and target health [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <monitoring-namespace> get pods,svc -o wide
kubectl -n <monitoring-namespace> get servicemonitors.monitoring.coreos.com
kubectl -n <monitoring-namespace> get podmonitors.monitoring.coreos.com
kubectl -n <monitoring-namespace> get prometheusrules.monitoring.coreos.com
kubectl -n <namespace> get svc,endpointslices,pods -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

The Prometheus Operator CRDs are optional. If absent, record **Not installed** and inspect the actual collector's configuration. Do not install CRDs for an observational lab.

In Prometheus, inspect **Status → Targets**, **Service Discovery**, and rule health. Queries:

```promql
up
count by (job) (up)
count by (job) (up == 0)
scrape_duration_seconds
scrape_samples_scraped
prometheus_rule_evaluation_failures_total
```

`up == 0` detects a discovered target that failed scraping; **a missing target produces no `up` series**. Detect absent expected targets using `absent()` with appropriate static labels or external inventory.

## 6. ServiceMonitor example (only if Prometheus Operator is installed)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: inference-staging
  namespace: monitoring
  labels:
    monitoring: approved
spec:
  namespaceSelector:
    matchNames: [tfy-network-lab]
  selector:
    matchLabels:
      metrics: inference
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

This discovers **Services** carrying `metrics: inference` and a Service port named `metrics`. It is illustrative; the Prometheus custom resource may select ServiceMonitors by namespace and labels, so creation alone does not ensure scraping. Verify NetworkPolicy, TLS/auth, relabeling and RBAC. Avoid exposing metrics publicly.

## 7. PromQL cookbook

Use discovered metric names and labels in real deployments.

```promql
# Scrape health by job
sum by (job) (up)

# Restart counter trend
sum by (namespace, pod) (increase(kube_pod_container_status_restarts_total[30m]))

# Container CPU cores (representative cAdvisor metric)
sum by (namespace, pod) (rate(container_cpu_usage_seconds_total{container!="",image!=""}[5m]))

# Pod memory working set
sum by (namespace, pod) (container_memory_working_set_bytes{container!="",image!=""})

# HTTP request throughput — only if the exporter actually uses these names/labels
sum by (service) (rate(http_requests_total[5m]))

# HTTP 5xx ratio, preserve matching numerator and denominator scope
sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
/
clamp_min(sum by (service) (rate(http_requests_total[5m])), 1e-9)

# P95 from a classic histogram; aggregate bucket rates BEFORE quantile
histogram_quantile(
  0.95,
  sum by (le, service) (rate(http_request_duration_seconds_bucket[5m]))
)
```

**Important:** `clamp_min` prevents divide-by-zero display errors but does not make a zero-traffic ratio meaningful. Alert on minimum traffic and/or use `unless` or `and` to gate low-volume services.

For request success SLIs, define whether 429, client cancellations and 4xx count as service failures. Document the contract before building a dashboard.

## 8. GPU and inference metrics

For NVIDIA DCGM, discover series with a query such as `{__name__=~"DCGM_.*"}`; metric availability varies by exporter and collector configuration. Representative commonly exposed names include `DCGM_FI_DEV_GPU_UTIL` and `DCGM_FI_DEV_FB_USED`, but verify their units and labels from live metadata.

For vLLM, metric naming and histogram types have changed between versions. Locate `vllm_` families in the actual exporter, then map:

- requests waiting/running, queue wait and preemptions;
- KV cache usage and memory pressure;
- prompt tokens/s versus generation tokens/s;
- TTFT, inter-token latency and end-to-end latency distributions;
- finished/failed/cancelled requests;
- model revision, GPU assignment and replica health.

Do **not** create labels containing prompt text, user IDs, API keys, request IDs or arbitrary tenant identifiers. Prefer controlled service/model-class/workspace labels, with an explicit cardinality budget.

## 9. Recording rules

Recording rules precompute frequently used, consistently defined aggregates and reduce dashboard query cost. Choose rule groups and intervals appropriate to scrape cadence, and maintain clear naming and ownership.

Illustrative PrometheusRule (Prometheus Operator required; metric names must be validated):

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: inference-staging-sli
  namespace: monitoring
  labels:
    monitoring: approved
spec:
  groups:
    - name: inference.sli
      interval: 30s
      rules:
        - record: service:http_requests:rate5m
          expr: sum by (service) (rate(http_requests_total[5m]))
        - record: service:http_5xx:rate5m
          expr: sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
        - record: service:http_5xx:ratio5m
          expr: service:http_5xx:rate5m / clamp_min(service:http_requests:rate5m, 1e-9)
        - record: service:http_request_duration_seconds:p95_5m
          expr: histogram_quantile(0.95, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
```

A rule must be loaded by the correct Prometheus instance and pass `promtool check rules`. Validate rule label sets and time windows before production rollout. Keep source counters/histograms available for audit; do not record per-Pod percentiles and then average them.

## 10. Rule test and deployment workflow

```bash
# Local/CI validation of an approved rules file:
promtool check rules inference-rules.yaml
# If using the PrometheusRule CRD, extract the spec.groups into a
# standard Prometheus rule-file structure for promtool or use an
# operator-aware validator; promtool does not validate CRD wrappers.
```

Prometheus supports unit tests with `promtool test rules <testfile>`. Test missing series, counter resets, low traffic, stale targets and histograms. Follow GitOps/TrueFoundry source of truth and peer review. Never edit production ConfigMaps or PrometheusRules directly if a controller owns them.

## 11. Cardinality, retention and cost

A time series is unique by metric name plus full label set. High-cardinality labels can multiply TSDB memory, CPU, WAL, storage and remote-write spend. Before onboarding an exporter:

1. Inventory series per job/namespace and label churn.
2. Reject unbounded labels such as request ID, prompt, token or arbitrary URL path.
3. Estimate scrape samples/sec, retention and remote-write bandwidth.
4. Bound histogram buckets, GPU/Pod/model labels and replica count.
5. Use recording rules for high-use aggregations, not to hide bad raw-label design.
6. Validate Prometheus head series, scrape duration, WAL/TSDB health and query latency.

Representative introspection:

```promql
prometheus_tsdb_head_series
sum by (job) (scrape_samples_post_metric_relabeling)
topk(10, count by (__name__) ({__name__!=""}))
```

The last query can be expensive; run it only within an approved monitoring query budget.

## 12. Troubleshooting decision tree

```text
Dashboard blank or alert missing
 ├─ exporter not running → Pod/runtime/metrics endpoint
 ├─ target not discovered → Service/Pod labels, selector, CRD selection
 ├─ target discovered but up=0 → DNS, TLS, auth, NetworkPolicy, port
 ├─ up=1 but metric absent → exporter version/config, relabel drop
 ├─ metric exists but wrong values → units, counter reset, query labels
 ├─ recording rule absent → rule selector, validation, evaluation error
 ├─ dashboard inconsistent → histogram aggregation, time window, staleness
 └─ alert silent → low traffic, pending period, Alertmanager routing
```

Always capture **LPHL** (Lowest Proven Healthy Layer) and **FFT** (First Failed Transition), with query, timestamp, label scope, exporter version and rule revision.

## 13. Failure scenarios

| ID | Symptom | FFT hypothesis | First check |
|---|---|---|---|
| M1 | No exporter Pod | Deployment → exporter | Pod/events |
| M2 | ServiceMonitor CRD absent | API → discovery | Installed CRDs |
| M3 | ServiceMonitor ignored | Selector → scrape | Prometheus selectors |
| M4 | Wrong metrics port | Service → exporter | Named port |
| M5 | Target missing | Discovery → target | Service Discovery |
| M6 | Target up=0 | Target → scrape | Target error |
| M7 | Metrics endpoint TLS failure | Scrape → TLS | CA/SNI |
| M8 | Scrape auth denied | Scrape → auth | Credentials/RBAC |
| M9 | NetworkPolicy blocks scrape | Prometheus → Pod | Policy/flow |
| M10 | Scrape timeout | Collector → endpoint | Duration/timeouts |
| M11 | Exporter metrics renamed | Version → query | /metrics metadata |
| M12 | GPU exporter lacks GPU | DCGM → device | Device visibility |
| M13 | vLLM metrics missing | Engine → exporter | Version/config |
| M14 | Counter reset spikes chart | Counter → rate | Restart/rate |
| M15 | Histogram P95 wrong | Buckets → query | le aggregation |
| M16 | Summary quantiles averaged | Replica → aggregation | Query semantics |
| M17 | Labels cause series explosion | Label → TSDB | Cardinality |
| M18 | Scrape relabel drops signal | Discovery → storage | Relabel rules |
| M19 | Recording rule syntax invalid | Rule → loader | promtool |
| M20 | Rule never evaluated | Loader → scheduler | Rule status |
| M21 | Query too expensive | TSDB → query | Query latency |
| M22 | Remote write backlog | TSDB → remote | Queue metrics |
| M23 | Missing target misread as healthy | Inventory → alert | absent()/inventory |
| M24 | Alert fails to fire | Rule → Alertmanager | State/routing |

## 14. Incident runbook

1. Identify the missing or misleading SLI, affected model/workspace and onset.
2. Check target inventory and `up` separately; missing series are not `up=0`.
3. Trace exporter → ServiceMonitor/PodMonitor or scrape config → target → TSDB → rule → dashboard/alert.
4. Inspect metric types, units, labels, scrape cadence, rule evaluation and recent version changes.
5. Contain false pages or blind spots using approved alert ownership and an alternate trusted signal.
6. Recover via the authoritative GitOps configuration; do not silently discard telemetry or disable TLS.
7. Validate fresh samples, recording rule outputs, correct SLI math, alert delivery and rollback.
8. Record LPHL/FFT, impact duration, data gaps and preventive action.

## 15. Acceptance gates

| Gate | Evidence |
|---|---|
| A1 | Actual metrics architecture and owners documented |
| A2 | Exporters/targets discovered with scrape health |
| A3 | Kubernetes, GPU and vLLM signal inventory |
| A4 | PromQL units, rate, error ratio and histogram math validated |
| A5 | Recording rules validated and version-controlled |
| A6 | Cardinality, retention and security budget reviewed |
| A7 | M1–M24 classified with LPHL/FFT |
| A8 | Safe staging test, rollback and alert verification or Not run |

## 16. Evidence template

```text
Date/time, owner and approval:
TrueFoundry workspace/cluster/namespace:
Prometheus version/Operator/remote backend:
Scrape targets/exporters/versions/labels:
GPU/vLLM metric families and units:
SLI PromQL and histogram semantics:
Recording rule file/commit, promtool test results:
TSDB head series, scrape/remote-write health:
M1–M24 LPHL/FFT and remediation:
A1–A8 Pass/Fail/Not run, rollback and reviewer:
```

**Hands-on lab:** [11.3 — Prometheus Metrics, Exporters and Recording Rules Check](../labs/03-prometheus-metrics-exporters-recording-rules-check.md)

**Next:** Chapter 11.4 — OpenTelemetry Distributed Tracing and Correlation.

## References

- [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
- [Prometheus Query Functions](https://prometheus.io/docs/prometheus/latest/querying/functions/)
- [Prometheus Recording Rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)
- [Prometheus Operator](https://prometheus-operator.dev/)
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
- [vLLM Documentation](https://docs.vllm.ai/)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
