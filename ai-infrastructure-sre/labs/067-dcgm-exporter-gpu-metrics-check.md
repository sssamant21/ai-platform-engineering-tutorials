# Lab 067 — DCGM Exporter and GPU Metrics Acceptance Gate

**Chapter:** [067 — DCGM Exporter and GPU Metrics](../part-09-gpu-observability/067-dcgm-exporter-gpu-metrics.md)

**Execution state:** Not run. Reference lab and evidence checklist only. Use approved staging, never disrupt a production exporter to simulate failure.

## Required evidence

- [ ] Record cluster, GPU node inventory, GPU UUIDs, MIG mode, driver/exporter/Operator versions and UTC timestamp.
- [ ] Identify exporter ownership (GPU Operator or standalone) and capture DaemonSet desired/ready.
- [ ] Verify Service labels, port name, EndpointSlices and a local authorized `/metrics` response.
- [ ] Confirm Prometheus ServiceMonitor selection, target health, scrape errors and scrape freshness.
- [ ] Reconcile observed UUIDs with expected GPUs and explain MIG representation.
- [ ] Capture GPU utilization, memory, temperature and power queries for a documented window.
- [ ] Inspect ECC/Xid availability and independently corroborate any observed error.
- [ ] Exercise missing-series, failed-scrape and ECC alert expressions against synthetic fixtures.
- [ ] Check RBAC, network exposure, scrape credentials (if used), retention and metric cardinality.
- [ ] Submit four runbook walkthroughs and rollback/escalation decisions.

## Read-only command pack

```bash
date -u +%FT%TZ
kubectl get nodes -o wide
kubectl get pods -A -o wide | grep -i dcgm || true
kubectl get svc -A | grep -i dcgm || true
kubectl get servicemonitors -A 2>/dev/null || true
kubectl get endpointslice -A | grep -i dcgm || true
```

Adjust namespaces and selectors from discovery, not guesses. In a second terminal after authorized `kubectl port-forward`, inspect `curl -fsS http://127.0.0.1:9400/metrics`. Do not upload secrets, hostnames or tenant IDs into public issues.

## Negative test vectors (synthetic, not live disruption)

| Fixture | Expected outcome |
|---|---|
| Scrape target present, `up=0` for >5m | Telemetry degradation alert |
| GPU target disappears from discovery | Inventory-based missing-target alert |
| Scrape succeeds but one expected GPU UUID absent | Coverage failure, gate blocked |
| ECC aggregate double-bit counter increases | Investigate health with corroboration |
| Xid field absent or gauge-like | Do not apply counter-rate rule |
| MIG enabled and GPU index labels repeat | Join using verified UUID/instance identities |

## Evidence record

```yaml
chapter: 067
environment: staging
executed: false
reviewed: false
utc_timestamp: null
operator_version: null
dcgm_exporter_version: null
gpu_count_expected: null
gpu_count_observed: null
scrape_target_count: null
metric_families_verified: []
synthetic_alert_cases_passed: []
open_findings: []
score: null
reviewer: null
```

## Scoring and stop conditions

Use Chapter 067's 100-point rubric. **No pass claim** until all required evidence is captured and reviewed. Stop if exporter coverage is incomplete, alerts rely on unverified metric semantics, or the change risks disrupting tenant GPU workloads. Preserve logs and roll back via the approved deployment owner.

**Next:** Chapter 068 — Prometheus and Grafana GPU Dashboards.
