# Lab 072 — Cluster Telemetry and Event Correlation Acceptance Gate

**Chapter:** [072 — Cluster Telemetry and Event Correlation](../part-09-gpu-observability/072-cluster-telemetry-event-correlation.md)

**Execution status:** NOT RUN. All timeline values are synthetic examples until explicitly marked as observed.

## 1. Preconditions and safety

Use read-only Kubernetes, monitoring and log access initially. Obtain approval for staging restart tests; never restart production GPU workloads or induce actual GPU hardware failures. Use redacted evidence and access-controlled incident records. Do not publish secrets, prompts, customer payloads, GPU UUIDs or private hostnames.

## 2. Exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | Source inventory | systems, versions, retention | Pending |
| E02 | Identity contract | pod UID/node/GPU/MIG mapping | Pending |
| E03 | Time synchronization | UTC, skew and precision | Pending |
| E04 | Kubernetes Events | event API JSON and timestamps | Pending |
| E05 | Driver evidence | redacted NVRM/kernel logs | Pending |
| E06 | GPU metrics | query and Xid semantics | Pending |
| E07 | Pod lifecycle | UID, restart and allocation | Pending |
| E08 | Trace/log link | synthetic trace_id and log event | Pending |
| E09 | GPU/workload link | verified allocation mapping | Pending |
| E10 | Synthetic timeline | six-source incident narrative | Pending |
| E11 | Staging restart | approved timestamps and rollback | Pending |
| E12 | Missing-data test | retention/sampling/telemetry gap | Pending |
| E13 | Incident report | evidence, hypotheses, confidence | Pending |
| E14 | Review | runbooks, security, score | Pending |

## 3. Read-only command pack

\`\`\`bash
kubectl api-resources | grep -i event
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get events.events.k8s.io -A -o json
kubectl get pods -A --field-selector spec.nodeName=<gpu-node> -o json
kubectl -n <namespace> get pod <pod-name> -o json
kubectl -n <namespace> logs <pod-name> --timestamps=true --since=2h
kubectl -n <namespace> logs <pod-name> --previous --timestamps=true
\`\`\`

Use \`--previous\` only if a previous terminated container exists. For host journal commands, require separately approved access and collect minimal time windows.

## 4. Synthetic timeline exercise

Create a six-row incident timeline using these **invented** inputs: cloud node maintenance at 12:00:00Z, kernel Xid at 12:00:04Z, NodeNotReady at 12:00:10Z, pod Pending at 12:00:25Z, NCCL timeout at 12:01:05Z and training progress stall at 12:02:00Z. Add a source, original timestamp, UTC timestamp, ingestion delay, identity mapping, confidence and evidence reference for each.

Then answer:
- Does temporal order prove causality? **No.**
- What evidence would distinguish a hardware issue from a driver regression?
- How could a missing exporter target create false confidence?
- How would a recreated pod with the same name break a naïve join?
- Which data may be missing due to retention or sampling?

## 5. Staging-only lifecycle test

With approved change control, restart a **disposable non-GPU staging workload**, not a production GPU job. Capture Event and pod UID changes, container restart counts, logs and trace continuity. Record expected versus observed timestamps and any clock skew. Stop if activity affects shared workloads.

## 6. Evidence template

\`\`\`yaml
chapter: "072"
environment: "STAGING_OR_OFFLINE"
date_utc: null
reviewer: null
kubernetes_version: null
gpu_exporter_version: null
telemetry_sources: []
clock_skew_estimate_ms: null
pod_uid_mapping_verified: "PENDING"
gpu_mig_mapping_verified: "PENDING"
xid_semantics_verified: "PENDING"
synthetic_timeline_review: "NOT_RUN"
staging_restart: "NOT_RUN"
missing_data_test: "NOT_RUN"
privacy_review: "PENDING"
retention_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 7. Acceptance and stop conditions

Apply Chapter 072's gates: **20 + 20 + 20 + 15 + 15 + 10 = 100**. Pass at **≥85** only with evidence and no critical privacy/security failure or unsupported causal conclusion. Stop immediately for production disruption, unauthorized access, sensitive-data export, or unsafe correlation that could trigger destructive remediation.

**Next:** Chapter 073 — Observability Pipeline Reliability.
