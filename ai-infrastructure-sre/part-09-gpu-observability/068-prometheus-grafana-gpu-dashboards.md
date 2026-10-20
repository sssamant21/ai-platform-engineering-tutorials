# Chapter 068 — Prometheus and Grafana GPU Dashboards

**Part 09 — GPU Observability | AI Infrastructure Engineering & SRE**

**Status:** Published documentation; live execution, independent technical review and security acceptance pending.

## 1. Learning outcomes

By the end, an AI infrastructure SRE can design a Kubernetes GPU observability pipeline, verify DCGM target discovery, build accurate PromQL, deploy version-controlled Grafana dashboards, diagnose missing and stale data, and prove telemetry coverage without mistaking dashboard appearance for reliability.

**Prerequisites:** Chapter 067, Kubernetes read access, Prometheus Operator/ServiceMonitor familiarity, and Grafana viewer or staging editor access. No GPU is required for the offline design exercises; a GPU-backed staging cluster is required for live validation.

## 2. Reference architecture and ownership

```text
GPU node / NVIDIA driver
  └─ DCGM + dcgm-exporter (one approved deployment per GPU scope)
       └─ Kubernetes Service :metrics
            └─ ServiceMonitor / PodMonitor
                 └─ Prometheus scrape + TSDB
                      ├─ recording rules / alert rules
                      ├─ Grafana datasource → version-controlled dashboards
                      └─ Alertmanager → on-call
Kubernetes API / kube-state-metrics / node-exporter ───────┘
```

Ownership: GPU platform team owns exporter/driver compatibility and metric semantics; observability team owns scrape configuration, TSDB retention, dashboards and alert routing; workload owners provide job identity and expected GPU demand; SRE owns incident procedures and acceptance evidence.

**Production principles:** pin and review component versions; avoid duplicate DCGM exporter instances; scope scrape discovery to intended namespaces; use TLS/authentication and least privilege; label with bounded-cardinality identifiers; never add dataset names, prompts, customer IDs or unbounded job IDs to GPU metrics. Kubernetes node names can be operationally sensitive.

## 3. Dashboard design: golden signals and GPU-specific context

| Dashboard panel | Suggested query / input | Interpretation and caveat |
|---|---|---|
| Exporter scrape health | `up{job=~".*dcgm.*"}` | `up=1` means scrape succeeded, **not** GPU health |
| Scrape sample age | `time() - timestamp(DCGM_FI_DEV_GPU_UTIL)` | Evaluate only for series that exist; absence needs separate detection |
| GPU busy percentage | `DCGM_FI_DEV_GPU_UTIL` | Often 0–100; confirm exporter/version units |
| Framebuffer used | `DCGM_FI_DEV_FB_USED` | Typically MiB; verify metric HELP/TYPE before labeling |
| Framebuffer utilization | `100 * DCGM_FI_DEV_FB_USED / clamp_min(DCGM_FI_DEV_FB_TOTAL, 1)` | Show percent only when both series share matching labels |
| Temperature | `DCGM_FI_DEV_GPU_TEMP` | Degrees Celsius if confirmed by exporter metadata |
| Power | `DCGM_FI_DEV_POWER_USAGE` | Usually watts; compare to model-specific policy |
| ECC / Xid | exporter-supported ECC and Xid series | Some are counters, others event gauges; inspect TYPE and semantics first |
| GPU inventory | exporter series grouped by node, GPU UUID, MIG identity | Do not sum physical GPU and MIG child utilization as equivalent GPUs |
| Scheduling context | Kubernetes allocatable and requested `nvidia.com/gpu` | Requires kube-state-metrics and validated label joins |

**Critical:** exporter metric names and labels differ by DCGM Exporter release, counter configuration, MIG mode and GPU generation. Run `curl -fsS localhost:9400/metrics | grep -E '^(# (HELP|TYPE) DCGM_|DCGM_FI_DEV_)' | head -80` against a locally forwarded endpoint; treat queries in this chapter as templates to adapt, not universally guaranteed contracts.

### 3.1 PromQL cookbook

```promql
# Per-GPU utilization; keep the original identity labels
DCGM_FI_DEV_GPU_UTIL

# Average by Kubernetes node only when the exporter emits a stable node label
avg by (Hostname) (DCGM_FI_DEV_GPU_UTIL)

# Fraction of framebuffer occupied, same-series vector matching
100 * DCGM_FI_DEV_FB_USED / clamp_min(DCGM_FI_DEV_FB_TOTAL, 1)

# Prometheus scrape failures for exporter jobs; adjust selector after discovery
up{job=~".*dcgm.*"} == 0

# Scrape freshness for present utilization series
time() - timestamp(DCGM_FI_DEV_GPU_UTIL)

# Number of GPU series reporting in the last five minutes
count(count_over_time(DCGM_FI_DEV_GPU_UTIL[5m]) > 0)

# Per-GPU 95th percentile over a window; this is time-series quantile,
# not fleet-level percentile across all GPUs
quantile_over_time(0.95, DCGM_FI_DEV_GPU_UTIL[1h])
```

Do not use `rate()` on utilization gauges. Do not calculate fleet availability solely from observed exporter series: missing nodes disappear from the denominator. Compare observed inventory to an independently maintained expected GPU/node inventory. Use `absent_over_time()` only with known, stable selectors; it does not enumerate every missing label combination.

### 3.2 Recording rules for expensive queries

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gpu-dashboard-recording
  namespace: monitoring
  labels:
    release: kube-prometheus-stack  # replace with the installed Prometheus selector
spec:
  groups:
    - name: gpu.dashboard.rules
      interval: 30s
      rules:
        - record: gpu:utilization:avg_by_host
          expr: avg by (Hostname) (DCGM_FI_DEV_GPU_UTIL)
        - record: gpu:fb_used_percent
          expr: 100 * DCGM_FI_DEV_FB_USED / clamp_min(DCGM_FI_DEV_FB_TOTAL, 1)
```

**Before applying:** inspect `kubectl -n monitoring get prometheus -o yaml` for `ruleSelector` and `ruleNamespaceSelector`, confirm metric names/labels, and validate with `promtool check rules rules.yaml` after extracting the `groups` content into a standalone rules file. Apply only in approved staging with change control. Grafana panels can use recording rules only after Prometheus has loaded them.

## 4. Grafana dashboard-as-code

Provision a read-only Grafana Prometheus datasource and a Git-managed dashboard. Example dashboard JSON below is intentionally minimal and portable; the datasource UID must be replaced by a real provisioned UID.

```json
{
  "uid": "gpu-sre-overview",
  "title": "GPU Fleet — SRE Overview",
  "schemaVersion": 39,
  "version": 1,
  "refresh": "30s",
  "time": {"from": "now-1h", "to": "now"},
  "tags": ["gpu", "sre", "dcgm"],
  "templating": {"list": []},
  "panels": [
    {
      "id": 1,
      "type": "timeseries",
      "title": "GPU utilization by device (%)",
      "datasource": {"type": "prometheus", "uid": "REPLACE_PROMETHEUS_UID"},
      "gridPos": {"h": 9, "w": 24, "x": 0, "y": 0},
      "fieldConfig": {"defaults": {"unit": "percent", "min": 0, "max": 100}, "overrides": []},
      "targets": [{"refId": "A", "expr": "DCGM_FI_DEV_GPU_UTIL", "legendFormat": "{{Hostname}} / {{gpu}}"}]
    },
    {
      "id": 2,
      "type": "timeseries",
      "title": "Framebuffer used (MiB; verify exporter units)",
      "datasource": {"type": "prometheus", "uid": "REPLACE_PROMETHEUS_UID"},
      "gridPos": {"h": 9, "w": 24, "x": 0, "y": 9},
      "fieldConfig": {"defaults": {"unit": "mebibytes"}, "overrides": []},
      "targets": [{"refId": "A", "expr": "DCGM_FI_DEV_FB_USED", "legendFormat": "{{Hostname}} / {{gpu}}"}]
    }
  ]
}
```

Validate the installed Grafana version's dashboard JSON schema, legend labels, and provisioning conventions before deployment. For production, add scoped cluster/namespace/node filters, an explicit **No data** panel state, datasource health, expected-versus-observed GPU count, workload context, runbook links, and an annotation for maintenance windows. A dashboard should distinguish **0% utilization**, **missing data**, **not scheduled**, and **GPU unavailable**.

## 5. Kubernetes staging discovery and validation

Run these read-only commands first; replace namespace and resource names with discovered values.

```bash
kubectl get nodes -L nvidia.com/gpu.present
kubectl get pods -A -o wide | grep -Ei 'dcgm|prometheus|grafana'
kubectl get servicemonitors,podmonitors -A
kubectl -n monitoring get prometheus -o yaml
kubectl -n monitoring get svc | grep -Ei 'prometheus|grafana'
kubectl -n gpu-operator get svc | grep dcgm
# In a separate terminal, use the discovered Service and port:
kubectl -n gpu-operator port-forward svc/nvidia-dcgm-exporter 9400:9400
curl -fsS http://127.0.0.1:9400/metrics | head -60
```

For Prometheus, forward the actual Prometheus service, then inspect `/api/v1/targets`, `/api/v1/query?query=up`, `/api/v1/rules` with authenticated/local access. Use Grafana Explore to validate each query before importing dashboards. Prefer a staging datasource; never paste tokens into shell history, repository files or screenshots.

## 6. Lab sequence — 14 hands-on exercises

1. Inventory GPU nodes, MIG configuration, exporters, ServiceMonitors and Prometheus targets.
2. Record exporter build/version, `# HELP`, `# TYPE` and sample labels for six DCGM metrics.
3. Build an identity map of cluster, node, GPU UUID, GPU index and MIG slice; document absent labels.
4. Verify exporter target health and identify any undiscovered exporter targets.
5. Compare GPU inventory from Kubernetes/host tooling to observed Prometheus series.
6. Query utilization at 1m, 5m and 1h resolution; explain sampling/averaging trade-offs.
7. Build framebuffer used/total/percent panels; test missing denominator behavior.
8. Add temperature and power panels with verified units and safe thresholds.
9. Build a target-health and last-sample-age panel; verify No data versus zero.
10. Provision or import the dashboard in a disposable Grafana staging folder.
11. Test dashboard variable scoping and prevent cross-cluster data leakage.
12. Validate a recording rule with `promtool` and confirm it loads in staging.
13. Perform a **staging-only** telemetry outage simulation by changing an isolated test target; restore it and measure detection time. Never interrupt production exporter.
14. Export dashboard JSON, collect evidence and perform peer acceptance scoring.

## 7. Troubleshooting decision matrix

| Symptom | First checks | Likely root causes | Safe action |
|---|---|---|---|
| No GPU series | Prometheus targets, exporter `/metrics` | selector mismatch, exporter crash, GPU/driver issue | Correct discovery only after staging verification |
| `up=1`, blank GPU panel | query labels, metric HELP/TYPE | metric renamed, unsupported device/MIG mode | Adapt query; document unsupported series |
| Duplicate GPU lines | exporter instances, `instance`, UUID labels | duplicate scraping, duplicate exporter | Remove redundant target in approved change |
| Implausible GPU percent | units, MIG identity, aggregations | summed gauges, physical/MIG mix | Use device-scoped labels; avoid invalid sums |
| Gaps / stale panels | scrape interval, TSDB, network | timeout, overload, ingestion gaps | inspect target duration and Prometheus health |
| High Prometheus memory | active series, scrape sample count | unbounded labels, duplicate targets | bound cardinality and retention |
| Dashboard loads slowly | query inspector, range/step | expensive joins, high-cardinality query | record bounded aggregations and downsample |
| Grafana shows wrong cluster | datasource UID, variables, permissions | shared datasource, missing tenant filter | enforce folder/data source authorization |

## 8. Production runbooks

### Runbook A — GPU dashboards blank
**Trigger:** fleet dashboard shows No data. **Read-only triage:** check Grafana datasource connectivity → Prometheus `up` and target discovery → exporter endpoint → node GPU health. **Mitigation:** route to owning layer, annotate telemetry gap, preserve evidence. **Exit:** independent inventory and series coverage reconciled, panels restored, no hidden data gap.

### Runbook B — Utilization inconsistent with job behavior
**Trigger:** GPU looks idle during known training workload. **Triage:** check metric timestamp, GPU/MIG mapping, query step, process activity via approved host tooling, scheduling state. **Mitigation:** fix aggregation/identity mapping rather than changing workload placement based on unverified panels. **Exit:** representative workload and measured utilization align within documented tolerance.

### Runbook C — Cardinality explosion
**Trigger:** active series and Prometheus resource usage surge. **Triage:** inspect `/api/v1/status/tsdb` under authorized access, scrape targets and label dimensions. **Mitigation:** remove offending high-cardinality labels in staging first, then controlled rollout; protect retention and availability. **Exit:** series budget and scrape latency back within agreed limits.

### Runbook D — Scrape target missing during node outage
**Trigger:** expected GPU node vanishes without a failing `up` series. **Triage:** compare expected node inventory to exporter target set, node readiness, service discovery and Prometheus relabeling. **Mitigation:** restore discovery or node service as appropriate; add inventory-based missing-target alert. **Exit:** target coverage measured against expected inventory, not merely `up`.

## 9. Acceptance gates (100 points)

| Gate | Points | Required evidence |
|---|---:|---|
| Discovery and source authenticity | 15 | node/exporter/target map and versions |
| Metric semantics and identity | 20 | HELP/TYPE, unit verification, MIG handling |
| PromQL correctness | 20 | saved query outputs, missing-series behavior |
| Dashboard as code and security | 15 | exported JSON, datasource and RBAC review |
| Telemetry failure exercise | 15 | staging outage timestamps and recovery |
| Runbooks and operational handoff | 15 | owner, escalation, peer review |
| **Total** | **100** | **Pass: ≥85 with no critical safety/security failure** |

A published chapter is not a passed acceptance gate. Offline/static validation, live staging execution and independent review must be recorded separately in the status tracker.

## 10. Further reading

- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
- [Prometheus query functions](https://prometheus.io/docs/prometheus/latest/querying/functions/)
- [Prometheus Operator API](https://prometheus-operator.dev/docs/api-reference/api/)
- [Grafana dashboard provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards)
- [Grafana Prometheus datasource](https://grafana.com/docs/grafana/latest/datasources/prometheus/)

**Next:** Chapter 069 — OpenTelemetry for AI Infrastructure.
