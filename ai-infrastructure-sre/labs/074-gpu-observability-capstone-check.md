# Lab 074 — GPU Observability Capstone Acceptance Gate

**Chapter:** [074 — GPU Observability Capstone](../part-09-gpu-observability/074-gpu-observability-capstone.md)

**Status:** NOT EXECUTED. This is a staged acceptance protocol, not evidence of a passed production readiness review.

## 1. Preconditions

- Approved disposable GPU staging environment or clearly labeled offline tabletop for unavailable hardware.
- Inventory of GPU SKUs/MIG, drivers, DCGM, Prometheus, Alertmanager, Grafana, Collector, SDKs and Kubernetes versions.
- Read-only access first; scoped change ticket and rollback for every write/failure drill.
- Test-only alert destination, separate tenant-safe telemetry backends and approved retention/redaction.
- No real Xid/ECC fault injection, destructive GPU reset, production node drain, WAL deletion or unauthorized cloud capacity changes.

## 2. Sixteen-work-package checklist

| ID | Work package | Required evidence | State |
|---|---|---|---|
| WP01 | GPU/node/workload inventory | SKU, MIG, node and owner mapping | Pending |
| WP02 | Source contract | HELP/TYPE, SDKs, scrape/trace source matrix | Pending |
| WP03 | Threat model | tenant, RBAC, TLS/auth, redaction | Pending |
| WP04 | HA architecture | replicas, queues, storage and failure domains | Pending |
| WP05 | GPU target coverage | independent expected vs observed inventory | Pending |
| WP06 | Prometheus health | scrape, WAL, PVC, rules and retention | Pending |
| WP07 | OTel gateway | pinned config validation, accepted/exported | Pending |
| WP08 | Dashboards | fleet, node, workload and pipeline panels | Pending |
| WP09 | Xid/ECC alerts | metric semantics and test routing | Pending |
| WP10 | Capacity SLO | denominator, queue, error budget, headroom | Pending |
| WP11 | Timeline | UTC, pod UID, GPU/MIG, competing hypotheses | Pending |
| WP12 | Exporter outage | staging alert lifecycle and recovery | Pending |
| WP13 | Backend outage | retries, queue, loss, restore evidence | Pending |
| WP14 | Alert DR | synthetic end-to-end notification after restore | Pending |
| WP15 | Training incident tabletop | NCCL, checkpoint, safe recovery decisions | Pending |
| WP16 | Readiness review | score, risk owners and reviewer sign-off | Pending |

## 3. Read-only discovery

\`\`\`bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get servicemonitors,podmonitors,prometheusrules -A
kubectl get networkpolicy -A
kubectl get pvc -A
kubectl get resourcequota -A
kubectl get pods -A | grep -Ei 'dcgm|otel|prometheus|alertmanager|nvidia'
# Approved GPU host only:
nvidia-smi -L
nvidia-smi -q -d ECC,PCI,TEMPERATURE,POWER
\`\`\`

For the actual exporter and Prometheus, inspect metadata and target discovery before using \`DCGM_FI_DEV_XID_ERRORS\`, \`DCGM_FI_DEV_GPU_UTIL\` or \`up\`. Missing targets require an independent inventory denominator.

## 4. Failure-drill record

For each controlled staging scenario capture:

\`\`\`yaml
scenario: "REPLACE_WITH_SCENARIO"
environment: "isolated-staging"
approved_change_id: null
owner: null
start_utc: null
end_utc: null
baseline_metrics: {}
injection_method: null
detection_signal: null
alert_destination: "test-only"
time_to_detect_seconds: null
data_lost_or_duplicated: null
recovery_action: null
time_to_recover_seconds: null
rollback_proof: null
production_impact: "MUST_BE_NONE"
evidence_links: []
reviewer: null
result: "NOT_RUN"
\`\`\`

Run synthetic exporter outage, discovery disappearance, backend outage, Collector restart and alert receiver outage only if the staging blast radius is contained. Record **not run** for unavailable hardware or backend capability; a tabletop cannot count as a passed live recovery drill.

## 5. SLO arithmetic sanity test

Assume **synthetic** fixed eligible inventory of 20 GPUs over 30 days, 99.9% coverage target:

\`\`\`python
expected_gpu_minutes = 20 * 30 * 24 * 60
allowed_missing_minutes = expected_gpu_minutes * (1 - 0.999)
assert expected_gpu_minutes == 864_000
assert round(allowed_missing_minutes) == 864
print(expected_gpu_minutes, allowed_missing_minutes)
\`\`\`

For real production, calculate time-varying eligible GPU inventory and explicit maintenance exclusions. Verify source-to-query freshness and error budgets independently.

## 6. Readiness score and decision

| Gate | Points | Awarded |
|---|---:|---|
| Architecture, ownership and threat model | 15 | Pending |
| DCGM/Prometheus/OTel source correctness | 15 | Pending |
| GPU health alerting and safe routing | 15 | Pending |
| SLOs, capacity and inventory denominators | 15 | Pending |
| Incident correlation and evidence quality | 15 | Pending |
| Failure drills, HA and DR | 15 | Pending |
| Runbooks, peer review and operational handoff | 10 | Pending |
| **Total** | **100** | **Not evaluated** |

**Mandatory:** ≥85/100, privacy/security gate passed, inventory-backed target coverage, no destructive alert automation, test notification delivered end to end, peer review signed. Any critical gate failure yields **NO-GO** regardless of numeric score.

## 7. Final evidence record

\`\`\`yaml
chapter: "074"
part: "09-gpu-observability"
environment: "STAGING_OR_OFFLINE"
execution_date_utc: null
reviewer: null
versions_and_digests: {}
expected_gpu_inventory_source: null
target_coverage_result: "NOT_RUN"
otel_pipeline_result: "NOT_RUN"
gpu_health_alert_result: "NOT_RUN"
capacity_slo_result: "NOT_RUN"
incident_timeline_result: "NOT_RUN"
staging_exporter_failure: "NOT_RUN"
staging_backend_failure: "NOT_RUN"
alert_delivery_dr: "NOT_RUN"
training_tabletop: "NOT_RUN"
security_privacy_gate: "PENDING"
independent_review: "PENDING"
score: null
decision: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 8. Closeout

Publish the evidence pack in an approved restricted location, obtain independent sign-off, and update the [Status Tracker](../STATUS-TRACKER.md) **only** for milestones actually completed. Completion of eight written chapters is **not** completion of eight runtime validations.

**Next:** Chapter 075 — GPU Node NotReady and Device Plugin Failures.
