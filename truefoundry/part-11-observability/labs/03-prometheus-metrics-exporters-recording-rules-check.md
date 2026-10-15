# Lab 11.3 — Prometheus Metrics, Exporters and Recording Rules Check

**Canonical:** [Chapter 11.3](../03-prometheus-metrics-exporters-recording-rules/03-prometheus-metrics-exporters-recording-rules.md)  
**Default:** [SAFE-READ]. Only approved isolated staging changes are [LAB-MUTATE].

## 1. Mission

Inventory the actual Prometheus pipeline, prove scrape health and metric semantics, validate a versioned recording rule, and diagnose 24 common observability failures. Do not scrape customer data, expose /metrics publicly, leak bearer tokens or mutate production monitoring.

## 2. Preflight

```text
Date/time, approval and owner:
TrueFoundry workspace/cluster/namespace:
Prometheus/Operator/collector and versions:
Monitoring namespace, target labels and endpoint:
vLLM/GPU exporter versions and metric names:
Known-good rule revision and rollback:
Series/cardinality, cost and query budgets:
```

## 3. Read-only inventory

```bash
kubectl config current-context
kubectl -n <monitoring-namespace> get pods,svc -o wide
kubectl -n <namespace> get svc,endpointslices,pods -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

If Operator CRDs exist:

```bash
kubectl -n <monitoring-namespace> get servicemonitors.monitoring.coreos.com
kubectl -n <monitoring-namespace> get podmonitors.monitoring.coreos.com
kubectl -n <monitoring-namespace> get prometheusrules.monitoring.coreos.com
```

If absent, mark Not installed and inspect the installed collector rather than installing components.

## 4. Target and exporter worksheet

| Source | Target discovered | up | Key series | Owner/evidence |
|---|---|---|---|---|
| kube-state-metrics | | | | |
| node exporter | | | | |
| kubelet/cAdvisor | | | | |
| DCGM GPU exporter | | | | |
| vLLM /metrics | | | | |
| TrueFoundry integration | | | | |
| Gateway/Ingress | | | | |
| Autoscaler | | | | |

PromQL:

```promql
up
count by (job) (up)
count by (job) (up == 0)
scrape_duration_seconds
scrape_samples_scraped
prometheus_rule_evaluation_failures_total
```

Remember: a missing target has **no** `up` series; do not count it as up=0.

## 5. Metric correctness checks

| Check | Query/observation | Result |
|---|---|---|
| Counter rate survives restart | | |
| Gauge units documented | | |
| Request throughput by service | | |
| 5xx ratio with low-traffic handling | | |
| Histogram P95 aggregated by le/service | | |
| GPU metric labels and units | | |
| vLLM TTFT/ITL metric type | | |
| Queue/KV/token metrics available | | |
| No sensitive/high-cardinality labels | | |

Example PromQL **only after verifying names**:

```promql
sum by (service) (rate(http_requests_total[5m]))
sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
/
clamp_min(sum by (service) (rate(http_requests_total[5m])), 1e-9)
histogram_quantile(0.95, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
```

## 6. Recording rule test [SAFE-READ local validation]

Extract a standard Prometheus rule file from the chapter's illustrative `PrometheusRule.spec.groups` (do not pass the CRD wrapper directly to promtool). Then run:

```bash
promtool check rules inference-rules.yaml
promtool test rules inference-rules-test.yaml
```

The second command requires a correctly authored test file with `rule_files`, `evaluation_interval`, `tests` and expected series. Cover counter resets, no traffic, missing targets, histogram bucket aggregation and rule label scope. Record test file commit and output.

## 7. Optional staging deployment [LAB-MUTATE]

1. Obtain change approval and confirm an isolated monitoring target.
2. Validate Prometheus Operator CRDs, selectors and Service named metrics port.
3. Deploy only a lab-owned ServiceMonitor/PrometheusRule through the authorized GitOps workflow.
4. Confirm target discovery, `up=1`, fresh samples and rule evaluation.
5. Verify dashboards and an approved non-paging test alert route.
6. Stop for cardinality spikes, scrape pressure, sensitive data or unexpected alerts.
7. Roll back via the authoritative revision; verify targets, rules and alert routing.

If mutation is not approved, mark live deployment Not run.

## 8. Failure tabletop

| ID | Scenario | LPHL | FFT | Evidence | Recovery |
|---|---|---|---|---|---|
| M1 | Exporter absent | | | | |
| M2 | Operator CRD absent | | | | |
| M3 | ServiceMonitor ignored | | | | |
| M4 | Metrics port mismatch | | | | |
| M5 | Target undiscovered | | | | |
| M6 | up=0 | | | | |
| M7 | Scrape TLS failure | | | | |
| M8 | Scrape auth denied | | | | |
| M9 | NetworkPolicy block | | | | |
| M10 | Scrape timeout | | | | |
| M11 | Renamed metric | | | | |
| M12 | DCGM missing GPU | | | | |
| M13 | vLLM metrics missing | | | | |
| M14 | Counter reset artifact | | | | |
| M15 | Histogram P95 wrong | | | | |
| M16 | Summary quantiles averaged | | | | |
| M17 | High cardinality | | | | |
| M18 | Relabel drop | | | | |
| M19 | Invalid recording rule | | | | |
| M20 | Rule not evaluated | | | | |
| M21 | Expensive query | | | | |
| M22 | Remote-write backlog | | | | |
| M23 | Missing target hidden | | | | |
| M24 | Alert route failure | | | | |

LPHL = Lowest Proven Healthy Layer; FFT = First Failed Transition.

## 9. Acceptance gates

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| A1 Architecture and ownership | | |
| A2 Target discovery and scrape health | | |
| A3 Kubernetes/GPU/vLLM signals | | |
| A4 PromQL and histogram semantics | | |
| A5 Rule validation/version control | | |
| A6 Cardinality/retention/security | | |
| A7 M1–M24 LPHL/FFT | | |
| A8 Staging, rollback and alert | | |

## 10. Final report

```text
Scope/owner/date/approval:
Prometheus/collector/exporter versions:
Target inventory and scrape health:
GPU/vLLM metric families and units:
PromQL and rule test output:
Cardinality, retention, data security:
M1–M24 LPHL/FFT:
A1–A8 Pass/Fail/Not run:
Rollback revision, reviewer, open risks:
```

**Next:** Chapter 11.4 — OpenTelemetry Distributed Tracing and Correlation.
