# Lab 068 — Prometheus and Grafana GPU Dashboards: Evidence and Acceptance

**Chapter:** [068 — Prometheus and Grafana GPU Dashboards](../part-09-gpu-observability/068-prometheus-grafana-gpu-dashboards.md)

**Execution status:** Not executed. All sample values are examples, not observed results.

## Safety and prerequisites

- Use a Kubernetes staging cluster and Grafana staging folder for writes; begin with read-only discovery.
- Obtain owner approval for changes to ServiceMonitor, PrometheusRule or Grafana provisioning.
- Do not disable production scraping, restart production GPU workloads, or store credentials in evidence.
- Required tools: `kubectl`, `curl`, `jq`, Grafana Explore; `promtool` for rule validation.
- Record exporter, Prometheus, Grafana, GPU Operator, driver and Kubernetes versions.

## Evidence worksheet

| ID | Exercise | Evidence to capture | Result |
|---|---|---|---|
| E01 | GPU and exporter inventory | node list, pods, exporter deployment mode | Pending |
| E02 | Metric contract | sanitized HELP/TYPE and label examples | Pending |
| E03 | Identity map | GPU UUID / node / MIG mapping | Pending |
| E04 | Target discovery | expected and observed target counts | Pending |
| E05 | Coverage reconciliation | independently expected GPUs versus seen series | Pending |
| E06 | Utilization | 1m/5m/1h query screenshots and interpretation | Pending |
| E07 | Memory | used/total/percent, denominator edge case | Pending |
| E08 | Health | temperature and power with confirmed units | Pending |
| E09 | Missing telemetry | No data, zero and stale sample distinction | Pending |
| E10 | Dashboard | sanitized exported JSON and version | Pending |
| E11 | Access boundaries | cluster filters, folder/data source permissions | Pending |
| E12 | Rules | promtool output and loaded rule evidence | Pending |
| E13 | Staging failure drill | approved change, start/detect/restore times | Pending |
| E14 | Handoff | peer-reviewed runbook and scored acceptance | Pending |

## Read-only command pack

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide | grep -Ei 'dcgm|prometheus|grafana'
kubectl get servicemonitors,podmonitors,prometheusrules -A
kubectl -n monitoring get prometheus -o yaml
# Substitute real namespace/service after discovery:
kubectl -n gpu-operator port-forward svc/nvidia-dcgm-exporter 9400:9400
# Separate terminal:
curl -fsS http://127.0.0.1:9400/metrics | grep -E '^(# (HELP|TYPE) DCGM_|DCGM_FI_DEV_)' | head -100
```

For Prometheus, forward its actual Service to localhost and query:

```bash
curl -fsSG 'http://127.0.0.1:9090/api/v1/query' --data-urlencode 'query=up' | jq .
curl -fsSG 'http://127.0.0.1:9090/api/v1/query' --data-urlencode 'query=DCGM_FI_DEV_GPU_UTIL' | jq .
curl -fsS 'http://127.0.0.1:9090/api/v1/targets' | jq '.data.activeTargets[] | {scrapeUrl,health,lastError}'
```

These commands require an approved local port-forward and authorized access. Do not publish private endpoint addresses or bearer tokens.

## Validation tests

1. **Metric semantics:** confirm every query metric exists; inspect `# TYPE` and units; record unsupported series.
2. **Identity:** compare independent GPU inventory with observed time series; flag physical/MIG double counting.
3. **No data:** verify dashboards distinguish absent series from real zero utilization.
4. **Datasource isolation:** a staging viewer must not access another tenant's GPU inventory.
5. **Performance:** record dashboard range, resolution, query duration, active series and expected cost.
6. **Failure injection:** in an isolated staging target only, temporarily make a test exporter unreachable, measure time to detection, restore, and document actual observed timestamps. Do not perform without approval.
7. **Rule check:** extract PrometheusRule `spec.groups` to a conventional Prometheus rules YAML file and run `promtool check rules rules.yaml`. A Kubernetes CRD is not directly the expected promtool rules format.

## Evidence record template

```yaml
chapter: "068"
environment: "STAGING-ONLY"
date_utc: null
reviewer: null
kubernetes_version: null
gpu_driver_version: null
dcgm_exporter_version: null
prometheus_version: null
grafana_version: null
expected_gpu_devices: null
observed_gpu_series: null
mig_mode: null
scrape_targets_expected: null
scrape_targets_healthy: null
missing_series_test: "NOT_RUN"
staging_failure_drill: "NOT_RUN"
dashboard_json_review: "PENDING"
security_review: "PENDING"
score: null
result: "NOT_EVALUATED"
evidence_links: []
open_risks: []
```

## Acceptance and stop conditions

Score against Chapter 068's six gates (15 + 20 + 20 + 15 + 15 + 15 = 100). Pass at **85/100 or higher** only if there is no critical safety/security failure. Stop and escalate on production-target impact, unapproved write operations, credential exposure, cross-tenant visibility, or GPU/MIG double counting that could drive an unsafe capacity decision.

**Publication does not imply execution.** Update the tracker only after real evidence has been reviewed.
