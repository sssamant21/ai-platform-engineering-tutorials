# Lab 070 — GPU Health and Xid Alerting Acceptance Gate

**Chapter:** [070 — GPU Health and Xid Alerting](../part-09-gpu-observability/070-gpu-health-xid-alerting.md)

**Status:** NOT EXECUTED. Sample rules and expected observations are not evidence of successful tests.

## Safety preflight

- Start read-only; changes require an approved **isolated staging** environment and change ticket.
- No real GPU fault injection, ECC corruption, driver unloading, node draining or GPU resets.
- Confirm on-call test receiver so synthetic alerts never page production.
- Record exact GPU model, driver, DCGM Exporter, Prometheus, GPU Operator and Kubernetes versions.
- Protect GPU UUIDs, private hostnames, logs and customer/job metadata.

## Exercises and evidence

| ID | Exercise | Expected evidence | State |
|---|---|---|---|
| E01 | GPU/driver/MIG inventory | version and device mapping | Pending |
| E02 | Exporter HELP/TYPE | Xid/ECC/temperature metadata | Pending |
| E03 | Dashboard health panels | sanitized screenshots and queries | Pending |
| E04 | Historical Xid correlation | redacted driver log + pod timeline | Pending |
| E05 | Vendor code classification | documented decision matrix | Pending |
| E06 | ECC semantics | counter/gauge and reset assessment | Pending |
| E07 | Expected target coverage | independent inventory comparison | Pending |
| E08 | Prometheus rule checks | promtool output and active rules | Pending |
| E09 | Alert routing | test receiver proof | Pending |
| E10 | Grouping and inhibition | controlled alert examples | Pending |
| E11 | Scrape failure | test target alert lifecycle and recovery | Pending |
| E12 | Disappeared target | inventory-based missing alert evidence | Pending |
| E13 | Incident tabletop | roles, decisions, checkpoint safety | Pending |
| E14 | Acceptance | peer review and score | Pending |

## Read-only command pack

\`\`\`bash
kubectl get nodes -o wide
kubectl get pods -A -o wide | grep -Ei 'dcgm|nvidia|prometheus'
kubectl get servicemonitors,prometheusrules -A
kubectl get events -A --sort-by=.lastTimestamp | tail -80
# GPU host access only if approved:
nvidia-smi -L
nvidia-smi -q -d ECC,TEMPERATURE,POWER,PCI
journalctl -k --since '-2 hours' | grep -Ei 'NVRM|Xid|nvidia'
# After forwarding the discovered exporter Service:
curl -fsS http://127.0.0.1:9400/metrics | grep -E '^(# (HELP|TYPE) DCGM_|DCGM_FI_DEV_)' | head -100
\`\`\`

## Rule validation

1. Inspect actual metric type, labels and current series; verify Xid semantics for the installed exporter.
2. Adapt the chapter's \`PrometheusRule\` to the actual Prometheus \`ruleSelector\` and \`ruleNamespaceSelector\`.
3. Extract \`spec.groups\` into a plain Prometheus rules file and run \`promtool check rules rules.yaml\`.
4. Use \`kubectl apply --dry-run=server\` on the Kubernetes CRD manifest.
5. With explicit staging approval, deploy and inspect Prometheus \`/api/v1/rules\` and \`/api/v1/alerts\`.
6. Route controlled alerts only to the isolated test receiver. Record pending → firing → resolved and actual timestamps.

## Safe failure drill

Use a dedicated **synthetic staging exporter target** to simulate scrape failure; do not disconnect a production GPU. Test separately that removal of the target from service discovery is detected by an **independent expected-inventory signal**. If no inventory signal exists, record this as an unresolved coverage gap rather than claiming success.

The test must show a warning, the alert destination, time-to-detection, restoration, and no impact outside staging. A nonzero Xid test may be performed with a dedicated synthetic Prometheus test series, but never by attempting to cause a real GPU Xid.

## Incident tabletop: training node failure

Scenario: one GPU node in a multi-node training job reports Xid 79 while the job has a checkpoint in progress. Document: incident commander, GPU platform owner, workload owner, checkpoint integrity decision, recovery-point objective, impact scope, escalation to hardware support, criteria for safe rescheduling, and evidence required before node return to service.

## Evidence template

\`\`\`yaml
chapter: "070"
environment: "staging"
date_utc: null
reviewer: null
gpu_model: null
driver_version: null
dcgm_exporter_version: null
prometheus_version: null
expected_gpu_count: null
observed_gpu_count: null
xid_metric_type_verified: "PENDING"
ecc_metric_semantics_verified: "PENDING"
rule_lint: "NOT_RUN"
rule_loaded: "NOT_RUN"
alert_routing: "NOT_RUN"
scrape_failure_drill: "NOT_RUN"
disappeared_target_drill: "NOT_RUN"
incident_tabletop: "NOT_RUN"
security_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## Gate

Score the chapter's gates **20 + 20 + 20 + 15 + 15 + 10 = 100**. Pass at **≥85** with no critical safety/security failure, validated alert routing and verified telemetry coverage. Stop immediately for unauthorized production disruption, unexpected on-call paging, exposed credentials, or unsafe GPU reset/eviction proposals.

**Next:** Chapter 071 — GPU Utilization and Capacity SLOs.
