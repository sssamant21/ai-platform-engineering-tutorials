# Chapter 067 — DCGM Exporter and GPU Metrics

**AI Infrastructure Engineering & SRE · Part 09 — GPU Observability · Chapter 067/100**

> **Evidence boundary:** Configuration and exercises below are reference material. No GPU cluster, scrape endpoint, Prometheus server or alert has been exercised by publication alone. Test on approved staging before operational use.

## Mission and learning outcomes

Deploy and validate NVIDIA DCGM Exporter telemetry on Kubernetes GPU nodes; map GPU, node, pod and workload identities; distinguish health from utilization; diagnose scrape gaps, ECC and Xid incidents; and build auditable production monitoring. Prerequisites: Chapters 012–013, 026–027, 059–066; a Kubernetes cluster with GPU nodes, a compatible NVIDIA driver/device plugin or GPU Operator, and authorized read-only cluster access.

## 1. Architecture and ownership

```text
GPU / NVIDIA driver / DCGM
       | per-node DCGM exporter (DaemonSet, GPU Operator managed or standalone)
       v
Kubernetes Service / ServiceMonitor / Prometheus scrape
       |
       +--> recording rules and alert evaluation --> Alertmanager / on-call
       +--> Grafana fleet and workload dashboards
       +--> historical retention / capacity / incident evidence
Kube-state-metrics + node-exporter + Kubernetes events
       | correlate node, namespace, pod, job and GPU UUID
```

DCGM exposes device health and activity counters; the exporter converts configured DCGM fields into Prometheus exposition format. A scrape failure is *not* proof that a GPU is healthy. Metrics names, available fields, labels and Kubernetes pod mapping vary by DCGM/exporter version, GPU model, MIG configuration and deployment mode. Record versions and verify actual `/metrics` output rather than assuming a particular field exists.

## 2. Preflight and safe discovery

```bash
kubectl get nodes -o wide
kubectl get nodes -o json | jq -r '.items[] | [.metadata.name, (.status.allocatable["nvidia.com/gpu"] // "0")] | @tsv'
kubectl get pods -A -o wide | grep -Ei 'gpu-operator|dcgm|device-plugin' || true
kubectl get crd servicemonitors.monitoring.coreos.com
kubectl get pods -A -l app=nvidia-dcgm-exporter -o wide
```

On an approved GPU host, `nvidia-smi -L` and `nvidia-smi --query-gpu=uuid,name,temperature.gpu,utilization.gpu,memory.used --format=csv` provide an independent snapshot. The latter is not a replacement for time-series metrics. Collect driver, CUDA, GPU Operator, DCGM Exporter, Prometheus Operator, GPU model, MIG mode and namespace versions. Avoid assuming the GPU Operator uses a specific label or release name.

## 3. Installation choices and change control

**Preferred when GPU Operator is already installed:** configure its DCGM Exporter component through the supported, version-pinned ClusterPolicy or Helm values. Do not deploy a second exporter on the same nodes without a deliberate design: duplicated targets and conflicting host-engine modes cause confusing data. Review NVIDIA's chart/ClusterPolicy schema for the installed version before changing it.

**Standalone option:** deploy the vendor's version-pinned DCGM Exporter chart to approved GPU nodes only, with explicit security, tolerations, Service, resource limits, and a validated exporter-to-DCGM host-engine topology. Pin image digests and chart versions after testing; avoid unreviewed `latest` images. The exporter typically serves Prometheus metrics on TCP 9400, but confirm the deployed configuration.

Example read-only endpoint discovery:

```bash
kubectl get svc -A | grep -i dcgm
kubectl get endpointslice -A | grep -i dcgm
kubectl -n gpu-operator get pods -o wide | grep -i dcgm
# Adjust namespace/service to observed deployment:
kubectl -n gpu-operator port-forward svc/nvidia-dcgm-exporter 9400:9400
# In a second terminal:
curl -fsS http://127.0.0.1:9400/metrics | head -40
```

Port forwarding is for a local, authorized test only. If the Service name or namespace differs, use discovered values; do not create or patch objects simply to make these commands work.

## 4. Prometheus discovery and scraping

The following is an **illustrative** ServiceMonitor. Verify Service labels, namespace, port *name*, Prometheus selector, RBAC and network policies in your cluster.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: dcgm-exporter
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  namespaceSelector:
    matchNames: [gpu-operator]
  selector:
    matchLabels:
      app: nvidia-dcgm-exporter
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
      scrapeTimeout: 10s
```

Only apply this after verifying that the exporter Service has `app=nvidia-dcgm-exporter` and a port **named** `metrics`, and that the Prometheus instance selects this ServiceMonitor. Prefer network-restricted scrapes; avoid exposing GPU inventory metrics publicly.

Validate from Prometheus:

```promql
up{job=~".*dcgm.*"}
count by (job, instance) (up{job=~".*dcgm.*"})
count by (instance) (DCGM_FI_DEV_GPU_UTIL)
max by (instance) (time() - timestamp(DCGM_FI_DEV_GPU_UTIL))
```

If a metric is missing, inspect `/metrics`, exporter logs, CSV counter configuration, DCGM support, scrape target labels and Prometheus relabeling. `up=1` means a scrape succeeded; it does **not** prove all expected GPUs or fields are represented.

## 5. GPU metrics: meaning and caveats

| Metric family (verify on target) | Operational meaning | Interpretation |
|---|---|---|
| `DCGM_FI_DEV_GPU_UTIL` | GPU activity percent | Low activity can be idle reservation, input starvation, or workload behavior |
| `DCGM_FI_DEV_FB_USED` / `FB_FREE` | Framebuffer memory MiB | Capacity pressure is not equivalent to CUDA OOM |
| `DCGM_FI_DEV_GPU_TEMP` | GPU temperature °C | Compare with device-specific thermal policy |
| `DCGM_FI_DEV_POWER_USAGE` | Power W | Use model-specific power cap context |
| `DCGM_FI_DEV_SM_CLOCK` / `MEM_CLOCK` | GPU clock MHz | Throttling diagnosis requires corroboration |
| `DCGM_FI_DEV_ECC_DBE_AGG_TOTAL` | Aggregate double-bit ECC errors | Availability depends on device/ECC support; counter semantics matter |
| `DCGM_FI_DEV_XID_ERRORS` | Driver-reported Xid telemetry | Export/field semantics and presence depend on version; corroborate with kernel logs |
| `DCGM_FI_DEV_PCIE_REPLAY_COUNTER` | PCIe replay counter | Investigate deltas, not arbitrary absolute values |

MIG instances may produce per-instance series with different labels and visibility. Do not sum MIG utilization percentages as if they were whole-device utilization. Avoid joining on GPU index alone across node replacements; prefer GPU UUID plus node identity, with controlled cardinality.

## 6. PromQL operational queries

```promql
# GPU activity by observed node/GPU identity (adjust labels to your exporter)
avg by (Hostname, UUID) (DCGM_FI_DEV_GPU_UTIL)

# Used framebuffer memory (MiB)
max by (Hostname, UUID) (DCGM_FI_DEV_FB_USED)

# Aggregate ECC double-bit counter increase over one hour, if exposed
sum by (Hostname, UUID) (increase(DCGM_FI_DEV_ECC_DBE_AGG_TOTAL[1h]))

# Prometheus scrape freshness: absent series require separate missing-target alerting
time() - max by (instance) (timestamp(DCGM_FI_DEV_GPU_UTIL))
```

For Xid, determine whether the exposed field is an event code, last observed value or counter before writing `increase()` alerts. Correlate Xid with `journalctl -k`, DCGM diagnostics, GPU health and job failures. Treat Xid as a triage signal, not an automatic node-replacement order.

## 7. Reliability alert design

**TelemetryMissing:** expected GPU node has no successful exporter scrape for more than 5 minutes; correlate with Kubernetes GPU allocatable and node readiness. A basic `up == 0` expression alone cannot detect a target disappearing entirely; compare against expected node inventory or use `absent_over_time` for known scoped targets.

**GPUHealthInvestigation:** new double-bit ECC errors, repeated confirmed Xid events, or sustained thermal throttling should page or ticket based on device/vendor policy and workload impact. Define separate severities, runbook URLs and inhibition during approved maintenance.

**GPUUtilizationLow:** a 30-minute low-activity signal should usually create a capacity/efficiency ticket, not a page, unless it is tied to a stalled critical job. Evaluate per allocated GPU and workload phase.

Illustrative rule for observed series (requires compatible labels and counters):

```yaml
groups:
  - name: gpu-health
    rules:
      - alert: GPUUncorrectableECCIncrease
        expr: increase(DCGM_FI_DEV_ECC_DBE_AGG_TOTAL[10m]) > 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "New uncorrectable ECC telemetry on GPU"
          description: "Investigate GPU, job impact and driver logs before remediation."
```

This rule does not fire when the series is absent; pair it with telemetry completeness monitoring. Tune reset handling, label aggregation and severity using real baseline data.

## 8. Hands-on lab sequence (staging)

1. Inventory GPU nodes and exporter/driver versions; save redacted UTC evidence.
2. Identify whether GPU Operator or a standalone deployment owns exporter lifecycle.
3. Discover exporter Service, endpoint and configured counter set.
4. Read `/metrics`; verify help/type, device UUID, node and MIG labels.
5. Check ServiceMonitor selector and Prometheus discovered targets.
6. Compare target count with expected GPU nodes and physical/MIG device inventory.
7. Query GPU utilization, framebuffer, power and temperature over 30 minutes.
8. Correlate a GPU allocation with pod/node and training job, noting missing mappings.
9. Compare `nvidia-smi` snapshot with exporter metrics within sampling limitations.
10. Inspect ECC and Xid fields and corroborate with approved host logs.
11. Simulate missing telemetry by using a synthetic Prometheus fixture, **not** by disrupting production exporter.
12. Validate missing-target and ECC alert rules against recorded/synthetic test vectors.
13. Document metric cardinality and retention/cost risks.
14. Produce an evidence dossier with queries, screenshots, versions and rollback plan.

## 9. Troubleshooting matrix

| Symptom | First safe check | Likely branch | Recovery boundary |
|---|---|---|---|
| No exporter pods | Operator/DaemonSet status and events | Scheduling, toleration, image, RBAC | Review operator reconciliation |
| `up=0` | Target error, Service endpoints, network policy | TLS, connectivity, port mismatch | Fix staging config through change control |
| `up=1` but no GPU series | `/metrics`, exporter logs, DCGM host engine | Unsupported field, initialization, no GPU | Validate host and field support |
| Fewer GPUs than expected | UUID/MIG inventory and pod mapping | Missing devices, MIG representation | Escalate device-plugin/driver mismatch |
| Xid with no job failure | Kernel logs and DCGM diagnostics | Transient vs persistent device issue | Use vendor-guided severity |
| Unexpected utilization zero | Job phase, input throughput, power, clock | Idle, stalled, data starvation | Correlate before paging |
| ECC counter jumps | Reset/restart timestamps and device history | Genuine new error vs reset/label change | Confirm before isolation |
| Cardinality explosion | Prometheus series by labels | Pod UID, job ID churn | Relabel carefully; retain traceability |

## 10. Incident runbooks

**RB-067-A — Exporter absent:** confirm expected GPU nodes, DaemonSet desired/ready, target discovery, scrape errors, network policy, then restore only through reviewed staging change. Verify series freshness and target completeness.

**RB-067-B — ECC/Xid investigation:** identify node/UUID, affected tenants/jobs, UTC first occurrence, driver/kernel evidence and repeat rate; consult hardware vendor guidance, cordon/drain only with approval and checkpoint safety. Preserve device state for RCA.

**RB-067-C — Metrics mismatch:** compare DCGM field CSV, MIG mode, UUID labels, exporter version and `nvidia-smi`; correct dashboards/queries before altering devices.

**RB-067-D — Alert noise:** validate rule type, reset semantics, maintenance windows, routing, thresholds and evidence; test with synthetic data before promoting.

## 11. Acceptance criteria and scoring (100 points)

| Gate | Points | Evidence |
|---|---:|---|
| Inventory and ownership | 15 | Version/host/GPU/MIG inventory |
| Exporter availability | 15 | Pod, endpoint and /metrics proof |
| Prometheus discovery | 15 | Target health and selector audit |
| Device completeness | 15 | UUID/expected node reconciliation |
| Query correctness | 10 | GPU, memory, power, temp queries |
| Health triage | 10 | ECC/Xid evidence and correlation |
| Alert safety | 10 | Synthetic tests and missing-target design |
| Security/runbooks | 10 | Access, change/rollback, escalation |

**Pass recommendation:** >= 85/100, with no unresolved exporter coverage gap or unreviewed critical alert. Scores remain **unassessed** until evidence is independently reviewed.

## 12. Review questions

1. Why is `up=1` insufficient to establish complete GPU observability?
2. How can a disappearing scrape target evade `up == 0`?
3. What changes when MIG is enabled?
4. Why must Xid semantics be verified before using a PromQL counter function?
5. Which dimensions support durable incident correlation without unbounded cardinality?

## 13. References

- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
- [NVIDIA Data Center GPU Manager](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [Prometheus configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [Prometheus alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)

**Next:** Chapter 068 — Prometheus and Grafana GPU Dashboards.
