# Chapter 11.1 — Hands-on Lab: Observability Architecture and Telemetry Foundations

**Part 11 | Approved staging | Live execution pending**

## Mission and safety

Map a TrueFoundry request from gateway to model server, and verify Kubernetes/agent/GPU metrics, logs, traces, dashboard health, privacy controls and alert ownership. Use synthetic traffic only. **[SAFE-READ]** inspect; **[LAB-MUTATE]** disposable staging; **[CHANGE-CONTROL]** production review. Do not store real prompts, tokens, API keys or PHI.

## Lab 0 — Preflight [SAFE-READ]

```bash
kubectl config current-context
kubectl get namespaces
kubectl get pods -A -o wide
kubectl get svc -A
kubectl get deployments,daemonsets -A
kubectl get events -A --sort-by=.lastTimestamp | tail -40
kubectl top nodes 2>/dev/null || true
kubectl top pods -A 2>/dev/null || true
```

Record TrueFoundry version, agent namespace, metrics-server, Prometheus/OTel/logging components, GPU exporter, gateway/model runtime and SLO owner. Mark absent integrations as gaps; do not install cluster-wide operators without approval.

## Lab 1 — Telemetry source map

For each platform, gateway, Pod, node, GPU and cloud dependency source record endpoint, transport, owner, label/attribute schema, sampling, retention, security, collection lag and test evidence. Trace one synthetic request using safe correlation IDs; never record the prompt body.

## Lab 2 — Metrics and dashboard checks [SAFE-READ]

Inspect targets and scrape health through your approved Prometheus UI/API. Validate request rate, errors, latency histograms, Pod readiness, node resource metrics, GPU signals and collector health. Compare units and label cardinality. If a metric does not exist, document instrumentation gap rather than inventing it.

## Lab 3 — Synthetic serving traffic [LAB-MUTATE]

Issue approved low-volume synthetic inference traffic with valid/invalid auth credentials from the lab harness. Compare gateway count, server count, request duration, TTFT and token throughput where available. Avoid flooding the system or disclosing response content.

## Lab 4 — Controlled alert and fault tabletop

Use a disposable test alert or approved low-impact staging workload failure to verify alert delivery, grouping, on-call routing, dashboard links and runbook. Rehearse missing scrape target, log collector outage and GPU metric gap on paper if active faults are unsafe.

## Lab 5 — Cleanup and acceptance

Remove test workloads/alerts and restore previous scrape/rule configuration. Validate collection stability and no unexpected high-cardinality series. Produce sanitized evidence with UTC timestamps, metric name, query, screenshot reference, expected/actual result and reviewer.

## Twenty-four failure scenarios

| # | Scenario | Investigation |
|---|---|---|
| 01 | Prometheus target down | Service discovery |
| 02 | ServiceMonitor CRD absent | Installation |
| 03 | Scrape path wrong | Endpoint |
| 04 | Scrape TLS rejected | Trust |
| 05 | Metrics endpoint denied | Auth/RBAC |
| 06 | Collector Pod CrashLoopBackOff | Pipeline health |
| 07 | Collector queue saturated | Backpressure |
| 08 | Telemetry dropped | Exporter errors |
| 09 | Metrics-server unavailable | `kubectl top` |
| 10 | Agent logs absent | Collection |
| 11 | Kubernetes Events expired | Retention |
| 12 | GPU exporter missing | DaemonSet |
| 13 | GPU metric incompatible with MIG | Labels/devices |
| 14 | High-cardinality request label | Cost/privacy |
| 15 | Counter reset misread | PromQL `rate` |
| 16 | Histogram buckets incompatible | Quantile query |
| 17 | Trace context lost | Propagation |
| 18 | Trace sampling too aggressive | Coverage |
| 19 | Inference 5xx rise | Error segmentation |
| 20 | TTFT spike | Prefill/queue |
| 21 | Token throughput drop | GPU/runtime |
| 22 | Alert never delivered | Routing |
| 23 | Alert storm | Grouping/burn rate |
| 24 | Telemetry contains sensitive payload | Redaction/incident |

For each: expected/actual, safe evidence, root-cause hypothesis, mitigation, rollback and reviewer. Execute only approved staging faults; otherwise mark tabletop.

## Ten acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Approved scope, telemetry source inventory | Pending |
| G2 | Agent/Kubernetes/node health signals verified | Pending |
| G3 | Gateway/inference RED metrics verified | Pending |
| G4 | GPU/runtime signals mapped or gap documented | Pending |
| G5 | Logs, traces and correlation validated | Pending |
| G6 | SLI definitions, labels and units reviewed | Pending |
| G7 | Privacy, retention and cardinality controls verified | Pending |
| G8 | Dashboard and alert routing tested | Pending |
| G9 | Rollback, cleanup and pipeline health validated | Pending |
| G10 | SRE review and sign-off recorded | Pending |

**Pass** requires timestamped evidence and reviewer approval. Pending/blocked is not pass.

**Next:** Chapter 11.2 — Logging Architecture, Structured Logs and Kubernetes Events.
