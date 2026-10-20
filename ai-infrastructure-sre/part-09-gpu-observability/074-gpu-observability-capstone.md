# Chapter 074 — GPU Observability Capstone

**Part 09 — GPU Observability | AI Infrastructure Engineering & SRE**

**Status:** Published curriculum content. **NOT** a validated production deployment: offline execution, live staging exercises, security review and independent acceptance remain pending.

## 1. Capstone mission

Design, deploy in isolated staging, test and defend an end-to-end GPU observability platform that joins **DCGM Exporter, Prometheus, Grafana, OpenTelemetry, Kubernetes Events, GPU health/Xid alerting, workload capacity SLOs and telemetry disaster recovery**. The deliverable is an evidence-backed operational readiness decision—not just a collection of dashboards.

**Prior chapters:** 067 DCGM Exporter; 068 Prometheus/Grafana; 069 OpenTelemetry; 070 GPU health/Xid; 071 capacity SLOs; 072 incident correlation; 073 pipeline reliability.

**Audience:** GPU platform SRE, Kubernetes SRE, ML platform engineers, on-call, security, observability owners and workload owners.

## 2. Scenario and scope

A hypothetical enterprise serves latency-sensitive inference and distributed training on Kubernetes GPU nodes, with optional MIG profiles and mixed GPU SKUs. Recent incidents include a GPU Xid event, disappearing exporter targets, high-priority training jobs queued despite low fleet-average utilization, and an observability backend outage.

The capstone must establish **which devices and workloads are expected**, whether telemetry is fresh and complete, whether SLOs are meaningful, and whether incident responders can recover without risking data integrity or tenant privacy.

### 2.1 Reference architecture

\`\`\`text
GPU node: NVIDIA driver / DCGM / device plugin / kubelet
  ├─ DCGM Exporter ──> ServiceMonitor ──> Prometheus HA ──> metrics backend
  ├─ kernel logs ─────> approved log agent ───────────────> log backend
  └─ Kubernetes Events ──> event collector ───────────────> event backend
AI inference/training services
  ├─ app metrics ─────────────────────────────────────────> metrics backend
  └─ OTel traces ──> Collector gateways ──────────────────> trace backend
Scheduler / queue / cloud changes ────────────────────────> event timeline
                      |
                      v
           Correlation contract + inventory
                      |
           Grafana dashboards / SLO rules
                      |
        Alertmanager / on-call / incident record
                      |
      Runbooks / DR / change-control acceptance
\`\`\`

Keep signals' transport and ownership distinct. DCGM metrics do not automatically create application traces. Do not use trace IDs or unbounded job IDs as metric labels.

## 3. Deliverable inventory

| Deliverable | Minimum acceptance artifact |
|---|---|
| Architecture and ownership | diagram, dataflows, failure domains, escalation matrix |
| Source inventory | GPU/MIG/node inventory, expected exporter targets, versions |
| Collector and scrape configs | pinned versions, least privilege, TLS/auth, validated YAML |
| Dashboards | fleet health, node/device drilldown, workload latency/queue, pipeline health |
| Alerts | Xid/ECC triage, target disappearance, SLO burn, Collector failure |
| SLO specification | denominator, window, objective, exclusions, owner and budget |
| Correlation evidence | UTC incident timeline, pod UID → node → GPU/MIG mapping |
| Resilience evidence | staging failure drills, queue/retry/duplicate behavior |
| DR evidence | restore plan, RPO/RTO, synthetic alert delivery after recovery |
| Operational readiness | runbooks, risk register, scorecard, reviewer signatures |

## 4. Architecture and security gates

### 4.1 Scope and trust boundaries

- Separate staging and production namespaces, tenant boundaries, backends and alert destinations.
- Collector ingress: approved OTLP/gRPC 4317 and/or OTLP/HTTP 4318, with NetworkPolicies and TLS/mTLS as required.
- Prometheus scraping: explicitly authorized ServiceMonitors/PodMonitors and selectors.
- Collector export: authenticated encrypted destinations; avoid \`debug\` exporter for sensitive production payloads.
- Credentials: Kubernetes Secrets or approved secret manager; no plaintext credentials in manifests, dashboards or tickets.
- Runtime: reviewed pinned images, non-root where supported, restricted ServiceAccounts and Pod Security controls.
- Privacy: redact prompts, training data paths, customer identifiers and secrets before external export.
- Retention: bounded by incident response, audit requirements, cost and jurisdiction.

### 4.2 Capacity and failure-domain design

Document Prometheus replicas, Alertmanager redundancy, Collector gateway replicas, trace-affinity requirements for tail sampling, queue durability, remote-write deduplication and backend quotas. A second Prometheus replica improves scrape availability but is **not** a historical backup. A Collector in-memory queue does **not** survive pod termination.

## 5. SLI/SLO scorecard

These are **illustrative design objectives**, not established production guarantees:

| Objective | Numerator / denominator | Example target |
|---|---|---|
| GPU telemetry coverage | expected GPU-minutes with valid fresh samples / expected eligible GPU-minutes | ≥99.9% over 30 days |
| GPU availability | healthy schedulable compatible GPU-minutes / expected eligible GPU-minutes | ≥99.5% over 30 days |
| High-priority job admission | eligible jobs admitted within 5 min / eligible submitted jobs | ≥99% over 28 days |
| Inference request success | successful eligible requests / eligible requests | ≥99.9% over 30 days |
| Telemetry export | accepted eligible OTLP items exported / accepted eligible items | ≥99.5% over 30 days |
| Incident queryability | successful bounded queries / attempted diagnostic queries | ≥99.9% over 30 days |
| Alert delivery | successful synthetic notification checks / scheduled checks | team-defined |

Use **independent inventory** for expected GPUs/targets. Missing DCGM series means unknown health, not zero errors. Explicitly distinguish GPU device utilization, schedulable capacity, service throughput and job admission. Calculate budgets using the documented eligible denominator and time window.

### 5.1 Reproducible budget example

For 99.9% telemetry coverage over 30 days with a fixed expected population of 20 GPUs:

\`\`\`text
expected GPU-minutes = 20 × 30 × 24 × 60 = 864,000
allowed missing GPU-minutes = 0.001 × 864,000 = 864
\`\`\`

Real inventory changes require time-varying denominators and maintenance policy; do not copy the fixed example directly to production.

## 6. Operational validation commands

Start **read-only**:

\`\`\`bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get servicemonitors,podmonitors,prometheusrules -A
kubectl get pvc -A
kubectl get networkpolicy -A
kubectl get resourcequota -A
kubectl get pods -A | grep -Ei 'dcgm|prometheus|alertmanager|otel|nvidia'
# Approved GPU node access only:
nvidia-smi -L
nvidia-smi -q -d ECC,PCI,TEMPERATURE,POWER
journalctl -k --since '-2 hours' | grep -Ei 'NVRM|Xid|nvidia'
\`\`\`

Discover actual Prometheus target labels and DCGM exporter \`HELP\`/\`TYPE\` metadata before applying queries. \`DCGM_FI_DEV_XID_ERRORS\` may be a gauge with the latest Xid code; **do not** apply \`increase()\` or \`rate()\` without confirming semantics.

\`\`\`promql
up{job=~".*dcgm.*"}
DCGM_FI_DEV_GPU_UTIL
DCGM_FI_DEV_XID_ERRORS
scrape_samples_scraped{job=~".*dcgm.*"}
kube_node_status_condition{condition="Ready",status="true"}
\`\`\`

These queries depend on installed exporters and labels. \`up\` does not detect a target removed from service discovery; compare with expected inventory.

## 7. Sixteen capstone work packages

1. **Baseline inventory:** enumerate GPU SKUs, MIG profiles, driver versions, nodes, namespaces and workload owners.
2. **Source contract:** verify DCGM metric types, OTel SDK instrumentation, logs, events and scrape cadence.
3. **Threat model:** document tenant isolation, secrets, PII/prompt redaction, egress and RBAC.
4. **Architecture review:** design HA and ownership across all telemetry planes.
5. **DCGM target coverage:** reconcile expected GPU inventory against observed targets.
6. **Prometheus health:** validate scrape, WAL/TSDB retention, PVC and rule evaluation.
7. **OTel gateway:** validate pinned Collector configuration, transport, authentication and exporter health.
8. **Grafana dashboards:** build fleet, node, GPU/MIG, job and pipeline views.
9. **GPU fault alerts:** validate Xid/ECC semantics and safe on-call routing.
10. **Capacity SLOs:** calculate eligible capacity, fragmentation, queue admission and error budget.
11. **Incident timeline:** join kernel, Kubernetes, scheduler, metrics, traces and logs by verified identity/time.
12. **Exporter outage:** staging-only scrape failure plus separate disappeared-target test.
13. **Backend outage:** staging-only telemetry queue/backpressure/recovery test.
14. **Alerting DR:** restore a test notification path and measure end-to-end delivery.
15. **Incident tabletop:** multi-node training interruption during GPU/telemetry failure; preserve checkpoints.
16. **Readiness board:** evidence review, security sign-off, 100-point scoring and go/no-go.

## 8. Fault-injection and recovery matrix

**All disruptive scenarios must run only in approved isolated staging.** Do not trigger real Xid faults, ECC corruption, GPU resets, production drains or TSDB deletion.

| Scenario | Controlled injection | Expected detection | Recovery proof |
|---|---|---|---|
| Exporter scrape failure | disable synthetic staging target | \`up == 0\` while target remains discovered | alert resolves and samples resume |
| Target disappears | remove synthetic target from discovery | inventory-based missing-target alert | expected/observed reconciliation |
| Collector backend outage | block staging test backend only | retries, queue, refused/dropped metrics | backend receipt and measured gaps |
| Collector restart | restart disposable staging Collector | health and queue-persistence observations | trace receipt, duplicates/loss documented |
| Xid alert | synthetic gauge series / tabletop | code-specific triage and test routing | no unsafe auto-reset |
| GPU capacity shortage | synthetic scheduler dataset / tabletop | queue SLO, topology constraints | compatible headroom plan |
| Missing Kubernetes Events | synthetic retention-gap dataset | uncertainty flagged in timeline | alternate evidence or explicit gap |
| Alert delivery failure | disable test receiver | synthetic alert delivery check | restored test notification |

**Abort conditions:** unexpected production impact, data exposure, uncontrolled load, cross-tenant telemetry, unapproved paging, missing rollback or unsafe remediation.

## 9. Troubleshooting and failure patterns

| Failure | Root-cause candidates | Evidence needed |
|---|---|---|
| Dashboard shows green but GPU failed | missing exporter target, stale series | independent inventory, kernel logs |
| Training job queued with idle GPUs | topology, MIG fragmentation, quotas | scheduler events, node allocation |
| Xid alarm floods on-call | latched gauge, duplicate alert labels | exporter type, Alertmanager grouping |
| Traces disappear during peak | sampling, Collector OOM, backend throttling | SDK, queue and export metrics |
| Prometheus storage fills | cardinality, retention, compaction | TSDB stats, PVC headroom |
| DR restores queries but no pages | alert rules/routing not recovered | synthetic end-to-end notification |
| Timeline suggests wrong cause | skew, pod name reuse, missing events | UTC normalization and pod UID |

## 10. Production runbooks

### Runbook A — GPU hardware health and Xid

Preserve driver/kernel logs and GPU identity → verify code semantics and affected workloads → inspect DCGM and node state → classify severity using NVIDIA guidance → coordinate owner-approved remediation → validate GPU, node and workload recovery. **Never automatically reset a GPU from an Xid alert.**

### Runbook B — Telemetry blindness

Compare expected GPU inventory with exporter discovery → inspect target health, OTel pipelines and backend lag → declare unknown health for missing signals → restore source/ingest/query paths → verify fresh samples and synthetic alert delivery → record gaps.

### Runbook C — Capacity/SLO degradation

Check workload admission, queue, model-specific throughput and p99 latency → reconcile compatible healthy GPU capacity and failure reserves → test topology/MIG/quotas → scale or control admission under change policy → confirm error-budget recovery.

### Runbook D — Observability pipeline disaster recovery

Restore independent monitoring → recover ingestion and credentials → validate backend storage/query → restore alert evaluation and routing → verify synthetic notification → reconcile duplicates and data loss → close only after RPO/RTO evidence.

### Runbook E — Distributed training failure correlation

Capture NCCL and checkpoint evidence → map pod UID/node/GPU over time → correlate driver Xid, scheduler changes and network/storage symptoms → test competing hypotheses → coordinate safe restart from verified checkpoint.

## 11. Acceptance score (100 points)

| Gate | Points |
|---|---:|
| Architecture, ownership and threat model | 15 |
| DCGM/Prometheus/OTel source correctness | 15 |
| GPU health alerting and safe routing | 15 |
| SLOs, capacity and inventory denominators | 15 |
| Incident correlation and evidence quality | 15 |
| Failure drills, HA and DR | 15 |
| Runbooks, peer review and operational handoff | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 **plus mandatory gates**: no critical privacy/security issue; inventory-backed missing-target detection; no destructive auto-remediation; synthetic end-to-end alert delivery; and signed peer review. A score alone cannot override a mandatory failure.

### 11.1 Decision levels

- **GO (staging accepted):** all mandatory gates passed with reproducible evidence and no unresolved critical risk.
- **CONDITIONAL:** limited staging follow-up only; explicit risk owner and due date; not production approval.
- **NO-GO:** missing critical evidence, unsafe remediation, unresolved data exposure or failing recovery/alerting gates.

Publication, static lint, staging validation and production acceptance are **separate statuses**. The repository status tracker records publication only until real evidence exists.

## 12. Evidence and final handoff

Each work package must include: owner, execution UTC time, version/commit, environment, exact commands, expected versus observed results, sanitized screenshots/queries, rollback, risks, reviewer and evidence URL. Link results from the companion lab rather than claiming undocumented success.

## 13. References

- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)
- [NVIDIA Xid Errors](https://docs.nvidia.com/deploy/xid-errors/)
- [Prometheus Operator](https://prometheus-operator.dev/)
- [Prometheus Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)

**Part 09 — GPU Observability: all eight chapters documented. Live capstone execution and independent review are still required.**

**Next:** Chapter 075 — GPU Node NotReady and Device Plugin Failures (Part 10 — Incident Response and Troubleshooting).
