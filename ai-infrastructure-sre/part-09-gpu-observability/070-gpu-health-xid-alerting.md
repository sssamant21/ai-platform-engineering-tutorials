# Chapter 070 — GPU Health and Xid Alerting

**Part 09 — GPU Observability | AI Infrastructure Engineering & SRE**

**Publication:** Documentation published. Offline lab execution, live staging tests, security review and independent technical acceptance remain pending.

## 1. Learning objectives

Build production-minded GPU hardware alerting for Kubernetes AI clusters using DCGM Exporter, NVIDIA driver evidence, Prometheus and Alertmanager. Distinguish physical GPU faults from telemetry loss, classify Xid and ECC signals without assuming every event is fatal, implement safe routing, and document incident recovery and evidence-based return-to-service.

**Prerequisites:** Chapters 013 (GPU health), 067 (DCGM exporter), 068 (Prometheus/Grafana) and 069 (OpenTelemetry). Access: read-only Kubernetes and monitoring access; approved staging GPU node for live drills. No destructive fault injection on production GPUs.

## 2. Failure domains and signal semantics

\`\`\`text
GPU silicon / HBM / PCIe / NVLink
   → NVIDIA kernel driver / NVML / DCGM
      → DCGM Exporter → ServiceMonitor → Prometheus
         → recording + alert rules → Alertmanager → on-call
         → Grafana GPU health / incident timeline
   └─ kernel logs / node events / NVIDIA diagnostics → incident evidence
\`\`\`

| Signal | What it tells you | Critical caveat |
|---|---|---|
| Xid | NVIDIA driver reports a GPU-related event/error | **Code-specific**; not all Xids are hardware failures or grounds for replacement |
| ECC corrected | Correctable memory error activity | Trend and vendor thresholds matter; do not page on a single unqualified event |
| ECC uncorrected | Uncorrectable error activity | Requires urgent triage, but validate counter semantics and device context |
| GPU temperature | Current thermal reading | Thresholds vary by GPU model and thermal policy |
| Power / clocks | Throttling or power envelope context | Power draw alone is not health status |
| PCIe replay / NVLink errors | Interconnect health indicators | Counter type and GPU support vary |
| DCGM health watches | DCGM diagnostic/watch state | Health coverage depends on enabled watches and privileges |
| GPU reset / unavailable | GPU/driver not usable | Confirm with node, driver and workload state |
| Missing metrics | Telemetry pipeline or target failure | Missing data is **unknown health**, not healthy |

The exporter metric catalog depends on exporter version, configured counter CSV, GPU model and MIG mode. Check \`# HELP\`, \`# TYPE\` and labels from the actual endpoint. Do not assume \`DCGM_FI_DEV_XID_ERRORS\` is a monotonically increasing counter; in some configurations it represents the most recent Xid code as a gauge. Therefore **never** apply \`rate()\` or \`increase()\` to Xid values without proving counter semantics. Likewise, ECC series can be gauge-like accumulated totals; a reset can create false deltas.

## 3. Triage: Xid is a symptom, not a diagnosis

Example categories for investigation (not an exhaustive vendor codebook):

| Example | Operational interpretation | Initial response |
|---|---|---|
| Xid 13 | Graphics engine exception; can involve application or hardware | Correlate process, driver, workload and recurrence |
| Xid 31 | GPU memory page fault / MMU-related event | Check process behavior and device state |
| Xid 43 | GPU stopped processing for an application | Determine whether process termination isolated impact |
| Xid 48 | Uncorrectable ECC error reported | Urgent containment and vendor-guided recovery |
| Xid 79 | GPU has fallen off the bus | Treat as serious GPU/PCIe/host incident; inspect hardware path |
| Xid 94 / 95 | Contained / uncontained ECC-related conditions on supported architectures | Follow GPU-generation-specific NVIDIA recovery guidance |

**Important:** these examples are starting points, not universal severity assignments. Interpret each event using the current NVIDIA Xid Errors documentation, GPU generation, driver release and surrounding evidence. Some codes require a reset, others an application restart or node maintenance; the correct action is not inferable from the code alone.

### 3.1 Evidence collection: read-only first

\`\`\`bash
# Replace with discovered GPU node and namespace
kubectl get nodes -o wide
kubectl describe node <gpu-node>
kubectl get pods -A -o wide --field-selector spec.nodeName=<gpu-node>
kubectl get events -A --sort-by=.lastTimestamp | tail -80
kubectl get pods -A | grep -Ei 'dcgm|gpu-operator|nvidia'
# On an approved GPU host, if access is granted:
nvidia-smi -L
nvidia-smi -q -d ECC,TEMPERATURE,POWER,PCI,PERFORMANCE
journalctl -k --since '-2 hours' | grep -Ei 'NVRM|Xid|nvidia'
\`\`\`

Never assume a GPU reset is safe while training jobs, inference requests or other MIG instances share the physical GPU. Preserve logs, timestamps, driver and firmware versions, node identifiers, GPU UUID, workload owners and customer impact in access-controlled incident records.

## 4. Alerting design: four distinct layers

1. **Telemetry availability:** independently expected GPU targets versus observed scrape targets; \`up == 0\` catches scrape failures for still-discovered targets but not vanished targets.
2. **Device events:** validated Xid event semantics, ECC deltas, GPU reset and health check results.
3. **Degradation:** temperature/throttling/interconnect counters correlated with service latency and training step times.
4. **User impact:** job failures, checkpoint loss, inference error rate, queue delay, SLO burn. Page for sustained impact, not every isolated device fluctuation.

Use routing labels \`team\`, \`service\`, \`cluster\`, \`severity\` and \`runbook_url\` only when they are stable and authorized. Do not expose private node names or GPU UUIDs to public incident destinations. Maintain a maintenance-window inhibition policy and a separate path for missing telemetry.

## 5. Prometheus rules: conservative, version-aware examples

The following Kubernetes \`PrometheusRule\` is **a staging template**. It assumes the installed exporter exposes \`DCGM_FI_DEV_XID_ERRORS\` as a **gauge containing the latest Xid code**, with \`0\` representing no current Xid; confirm the behavior on the installed version before enabling. It does not infer an event rate.

\`\`\`yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gpu-health-staging
  namespace: monitoring
  labels:
    release: kube-prometheus-stack # replace with actual ruleSelector match
spec:
  groups:
    - name: gpu.health.staging
      interval: 30s
      rules:
        - alert: GPUExporterScrapeFailed
          expr: up{job=~".*dcgm.*"} == 0
          for: 5m
          labels:
            severity: warning
            team: gpu-platform
          annotations:
            summary: "GPU exporter target cannot be scraped"
            description: "Check target discovery, exporter pod and GPU driver"
        - alert: GPUXidNonzeroObserved
          expr: DCGM_FI_DEV_XID_ERRORS != 0
          for: 1m
          labels:
            severity: warning
            team: gpu-platform
          annotations:
            summary: "Nonzero GPU Xid code observed"
            description: "Triage code against NVIDIA documentation; do not reset automatically"
        - alert: GPUMetricSeriesStale
          expr: time() - timestamp(DCGM_FI_DEV_GPU_UTIL) > 180
          for: 2m
          labels:
            severity: warning
            team: gpu-platform
          annotations:
            summary: "Present GPU utilization series has stale sample"
\`\`\`

**Limitations:** \`timestamp()\` cannot alert for a series that has already disappeared; stale series may vanish before this rule fires. Use independent expected inventory or a controlled inventory exporter to detect disappeared GPUs. A nonzero Xid gauge can remain latched after an event; deduplicate and resolve using current hardware evidence, not the gauge alone. Adjust the \`job\` selector to actual discovered labels. Alert on specific ECC metrics only after confirming names, units, gauge/counter types and reset behavior; avoid generic \`increase()\` examples that can silently misclassify gauges.

### 5.1 PromQL verification

\`\`\`promql
# Actual exporter scrape targets (adapt selector to the discovered job label)
up{job=~".*dcgm.*"}

# Current Xid gauge, if present and verified
DCGM_FI_DEV_XID_ERRORS

# Physical GPU utilization; preserve device labels
DCGM_FI_DEV_GPU_UTIL

# Count present utilization series, NOT independently expected inventory
count(DCGM_FI_DEV_GPU_UTIL)

# Present series with a timestamp older than three minutes
time() - timestamp(DCGM_FI_DEV_GPU_UTIL) > 180
\`\`\`

Inspect \`/api/v1/targets\`, \`/api/v1/rules\`, \`/api/v1/alerts\` and Alertmanager routing through authenticated access. PrometheusRule CRDs are not directly compatible with \`promtool check rules\`; extract \`spec.groups\` to a conventional Prometheus rules YAML document first.

## 6. Alert severity and safe automation

| Severity | Criteria | Operational action |
|---|---|---|
| Info | Single transient event, no service impact | Record and correlate; do not page |
| Warning | Repeated device signal, noncritical scrape outage | Ticket and on-call review |
| Critical | Verified GPU unavailability, uncorrectable error with impact, multi-node service degradation | Page GPU platform SRE; coordinate workload owner |
| Emergency | Fleet-wide outage, active data loss or widespread inference failure | Incident command and approved recovery plan |

**Never** automatically reboot a node, reset a GPU, cordon/drain, evict jobs or replace hardware solely from an Xid metric. Automation may collect read-only diagnostics, attach dashboards, create an incident ticket, and notify owners. Any disruptive remediation requires blast-radius analysis, workload checkpoint/replica assessment, maintenance authorization and a rollback/abort plan.

## 7. Hands-on lab: 14 exercises

1. Identify GPUs, drivers, MIG partitions, exporter versions and ownership.
2. Inventory exporter metric names, HELP/TYPE, units and sample label sets.
3. Build a GPU health dashboard with Xid, ECC, thermal and target health context.
4. Correlate a historical Xid log line to node, GPU UUID and workload timeline.
5. Classify three vendor-documented Xid examples by impact and required evidence.
6. Compare corrected and uncorrected ECC behavior, including resets and unsupported GPUs.
7. Confirm \`up\` targets against independently expected GPU node inventory.
8. Validate alert rule YAML and expression queries in a staging Prometheus.
9. Route warning alerts to a test Alertmanager receiver without production paging.
10. Verify inhibition/deduplication and maintenance-window behavior.
11. Simulate a **staging-only** exporter scrape failure and confirm alert lifecycle.
12. Simulate an expected-target disappearance in a controlled test inventory and verify detection.
13. Run an incident tabletop for GPU lost-on-bus during multi-node training.
14. Capture evidence, review runbooks and score the production-readiness gate.

## 8. Troubleshooting matrix

| Symptom | Read-only triage | Avoid |
|---|---|---|
| Xid warning but workloads healthy | check code, timestamp, recurrence, driver logs | unconditional reset |
| Repeated Xid 79 | PCIe link, node logs, power, GPU inventory, vendor guidance | assuming application-only fault |
| ECC count rises | type, baseline, reset, workload and GPU model | treating gauge as monotonic counter |
| GPU disappears from metrics | target discovery, node readiness, driver, expected inventory | declaring zero failures |
| Hot GPU | cooling, fan, power, throttling and ambient context | universal temperature threshold |
| Alert flood | grouping, Xid latch semantics, missing inhibition | suppressing all GPU alerts |
| GPU unavailable after maintenance | driver compatibility, device plugin, host state | force-remediation without rollback |

## 9. Production runbooks

### A — Xid event triage

**Trigger:** nonzero Xid or kernel log event. **Action:** preserve event and driver version; map GPU UUID and affected pods; inspect vendor codebook; determine impact and recurrence. **Escalate:** critical hardware indicators or repeated failures. **Exit:** documented cause/mitigation, workload health restored, alert cleared by verified signal.

### B — ECC escalation

**Trigger:** uncorrectable ECC or abnormal corrected error pattern. **Action:** validate counter semantics and GPU generation, assess data integrity and active workloads, preserve evidence, consult NVIDIA/vendor support. **Remediation:** only under approved drain/reset/replacement workflow. **Exit:** post-recovery health and workload acceptance checks pass.

### C — GPU fallen off bus

**Trigger:** Xid 79 or GPU missing from \`nvidia-smi\`. **Action:** correlate PCIe/host logs, power and node events; engage platform/hardware support; plan workload migration/checkpoint recovery. **Exit:** verified device detection, driver/health checks and successful staging workload probe.

### D — GPU telemetry blind spot

**Trigger:** expected GPU missing from Prometheus. **Action:** compare inventory to discovered targets; inspect exporter, ServiceMonitor, GPU Operator and node state; annotate unknown health. **Exit:** full expected-versus-observed reconciliation and alert coverage.

## 10. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Inventory and signal semantics | 20 |
| Xid/ECC triage accuracy | 20 |
| Rule validity and target coverage | 20 |
| Alert routing and safe automation | 15 |
| Staging failure drill and recovery | 15 |
| Runbooks, evidence and peer review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 with no critical safety/security failure. A published rule is not a validated rule; offline linting, staging execution, independent review and production acceptance are separate milestones.

## 11. References

- [NVIDIA Xid Errors](https://docs.nvidia.com/deploy/xid-errors/)
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
- [NVIDIA DCGM documentation](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [Prometheus alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Alertmanager configuration](https://prometheus.io/docs/alerting/latest/configuration/)
- [Prometheus Operator API](https://prometheus-operator.dev/docs/api-reference/api/)

**Next:** Chapter 071 — GPU Utilization and Capacity SLOs.
