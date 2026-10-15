# Chapter 11.3 — Hands-on Lab: Prometheus Metrics, Exporters and Recording Rules

**Part 11 | Staging only | Live execution pending**

## Mission and safeguards

Validate Prometheus target discovery, Kubernetes/node/GPU exporters, inference metrics, PromQL, recording rules, cardinality, storage and failure recovery. **[SAFE-READ]** inspect; **[LAB-MUTATE]** isolated staging; **[CHANGE-CONTROL]** production approval. Use synthetic traffic and no secrets, raw prompts or customer data.

## Lab 0 — Inventory [SAFE-READ]

```bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl get svc -A
kubectl get servicemonitors,podmonitors -A 2>/dev/null || true
kubectl get prometheusrules -A 2>/dev/null || true
kubectl get deployments,daemonsets -A
kubectl top nodes 2>/dev/null || true
```

Record Prometheus version, Operator/managed implementation, retention, target labels, namespace selectors, exporters, GPU runtime, approved metric names and rule owner. Missing CRDs mean use the installed scrape/rule mechanism or tabletop.

## Lab 1 — Target discovery and exporter validation

From an approved Prometheus UI/API, list UP/DOWN targets and scrape duration/sample count. Inspect kube-state-metrics, node exporter, kubelet/cAdvisor, gateway/app and optional DCGM/vLLM endpoints. Validate units, labels and version-dependent metric availability.

## Lab 2 — PromQL validation

Run `up`, `rate`, `increase`, `sum by`, histogram p95 and `absent` against verified metrics. Compare synthetic request counts with gateway logs. Explain missing series vs zero, counter reset and sparse-traffic edge cases. Do not aggregate summary quantiles across instances.

## Lab 3 — Recording rules [LAB-MUTATE]

Create staging-only rules for request rate, 5xx ratio and p95 based on existing metrics. Run:
```bash
promtool check rules staging-recording-rules.yml
```
Use a local file matching the installed Prometheus rule syntax; if Prometheus Operator is installed, apply a reviewed `PrometheusRule` in an approved staging namespace. Confirm selection, evaluation interval, series creation and query parity with the source expression. Revert lab rules afterward.

## Lab 4 — Cardinality and storage review

Estimate series count by metric and label dimensions; identify unbounded labels and propose source-side fixes. Inspect TSDB head series, ingestion, WAL, retention and remote-write queue from approved dashboards. Record storage/cost impact before any change.

## Lab 5 — Faults and recovery

Rehearse target DOWN, invalid scrape path, bad rule expression, high cardinality and remote-write outage on disposable resources or tabletop. Verify alert routing and rollback. Do not disable production scraping or delete TSDB data.

## Lab 6 — Evidence and cleanup

Restore lab-only rules and workloads, validate target health and no new series explosion. Capture sanitized UTC evidence, PromQL, expected/actual values, rule validation, reviewer and rollback commit.

## Thirty failure scenarios

| # | Scenario | Diagnostic focus |
|---|---|---|
| 01 | Prometheus Pod not Ready | StatefulSet/storage |
| 02 | Prometheus target DOWN | Discovery/network |
| 03 | ServiceMonitor CRD missing | Operator installation |
| 04 | ServiceMonitor selector mismatch | Labels |
| 05 | Namespace selector excludes target | Scope |
| 06 | Service port name wrong | Endpoint |
| 07 | Metrics path wrong | HTTP |
| 08 | Metrics endpoint TLS error | Trust |
| 09 | Metrics endpoint 401/403 | Auth |
| 10 | NetworkPolicy blocks scrape | Connectivity |
| 11 | Exporter reports UP but series absent | Exporter/filter |
| 12 | kube-state-metrics missing | Deployment |
| 13 | Node exporter not scheduled | DaemonSet |
| 14 | kubelet/cAdvisor series unavailable | Managed cluster |
| 15 | DCGM exporter absent | GPU metrics |
| 16 | MIG GPU labels incompatible | Exporter version |
| 17 | vLLM metric renamed | Runtime version |
| 18 | PromQL returns no data | Labels/staleness |
| 19 | Counter reset misinterpreted | `rate` |
| 20 | Sparse traffic inflates error ratio | Denominator |
| 21 | Histogram p95 wrong | Buckets/units |
| 22 | Summary quantiles averaged | Invalid aggregation |
| 23 | Recording rule YAML invalid | `promtool` |
| 24 | PrometheusRule not selected | Operator labels |
| 25 | Recording rule evaluation slow | Query cost |
| 26 | High-cardinality label explosion | Series budget |
| 27 | WAL/disk saturation | Storage |
| 28 | Remote-write queue backlog | Receiver/network |
| 29 | Alert missing or noisy | Rules/routing |
| 30 | Metrics endpoint leaks sensitive labels | Privacy |

For each: expected/actual, time, target/series, sanitized evidence, cause hypothesis, mitigation, rollback and reviewer. Mark unsafe tests tabletop.

## Ten acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Approved scope and exporter inventory | Pending |
| G2 | Scrape target discovery and health verified | Pending |
| G3 | Kubernetes/node/container metrics verified | Pending |
| G4 | GPU/vLLM metrics verified or gaps recorded | Pending |
| G5 | PromQL counters, gauges, histograms validated | Pending |
| G6 | Recording rules syntax/evaluation verified | Pending |
| G7 | Cardinality, privacy and storage budgets reviewed | Pending |
| G8 | Dashboards and alert ownership tested | Pending |
| G9 | Recovery, rollback and cleanup demonstrated | Pending |
| G10 | SRE review and sign-off recorded | Pending |

**Pass** requires timestamped evidence and reviewer approval; pending/blocked is not pass.

**Next:** Chapter 11.4 — OpenTelemetry Distributed Tracing and Correlation.
