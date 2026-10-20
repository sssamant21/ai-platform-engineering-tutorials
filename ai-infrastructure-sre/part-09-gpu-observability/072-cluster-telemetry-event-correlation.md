# Chapter 072 — Cluster Telemetry and Event Correlation

**Part 09 — GPU Observability | AI Infrastructure Engineering & SRE**

**Publication status:** Published documentation. Offline tests, staging execution, independent technical review and security acceptance are pending.

## 1. Learning objectives

Reconstruct reliable incident timelines across Kubernetes events, GPU/driver logs, DCGM metrics, OpenTelemetry traces, workload scheduler state and cloud infrastructure. Learn evidence provenance, timestamp normalization, correlation keys, uncertainty management and safe incident automation.

**Prerequisites:** Chapters 067–071, read-only Kubernetes access, Prometheus/Grafana access, and a logging/tracing backend where available. All write operations and failure drills are restricted to approved staging.

## 2. Multi-layer telemetry architecture

\`\`\`text
Cloud instance / node / PCIe / GPU
   ├── kernel journal / NVRM Xid logs ─────────────┐
   ├── DCGM exporter → Prometheus ──────────────────┤
   ├── kubelet / Kubernetes Events ─────────────────┤
   ├── device plugin / GPU Operator logs ───────────┤
   ├── scheduler / queue / job controller ──────────┤
   ├── workload logs + OTel trace/span context ─────┤
   └── deployment / GitOps / cloud audit events ────┤
                                                    v
                                    Evidence collection and normalization
                                                    |
                                  Timeline → hypothesis → verification
                                                    |
                                  Incident record / runbook / postmortem
\`\`\`

No single backend is an authoritative record for every layer. Prometheus answers numeric state over time; Kubernetes Events describe object-related transitions but are ephemeral and may be aggregated; traces describe instrumented request paths; node logs reveal kernel/driver events; cloud audit logs identify control-plane changes.

### 2.1 Signal sources and limitations

| Source | Example | Strength | Limitation |
|---|---|---|---|
| Kubernetes Events | FailedScheduling, NodeNotReady, Killing | Object context | Short retention, deduplication, delayed delivery |
| Node journal | NVRM Xid 79, PCIe/AER messages | Hardware/driver evidence | Host access, retention, time sync |
| DCGM metrics | utilization, memory, temperature, Xid | Time-series trend | Scrape lag, gauge semantics, missing series |
| GPU Operator logs | driver/device plugin restart | Component lifecycle | Version-specific log formats |
| Scheduler/queue | Pending pod, gang wait, preemption | Admission causality | Scheduler plugins and policy differences |
| Application logs | NCCL timeout, checkpoint error | Workload impact | Missing structured fields or clock skew |
| OTel traces | inference span latency | Request-level critical path | Sampling and partial instrumentation |
| Cloud events | node termination, quota, autoscaling | Infrastructure changes | API delay and permissions |
| GitOps/change audit | rollout or policy update | Change attribution | Correlation alone does not prove causation |

## 3. Correlation contract and identity

Use a documented mapping among \`cluster\`, \`namespace\`, \`node\`, \`pod UID\`, \`container ID\`, \`workload owner\`, \`GPU UUID\`, \`MIG instance\`, and \`trace_id\`. IDs serve different purposes:

- **Pod UID**, not only pod name, distinguishes recreated pods.
- **Node UID/provider ID** helps distinguish replaced nodes with reused names.
- **GPU UUID** identifies a physical GPU, while MIG device identity is a distinct scope.
- **Trace ID** connects instrumented spans/logs but **not** raw hardware metrics.
- **Change ID / deployment revision** anchors approved rollout events.
- **Incident ID** links evidence and decisions without adding incident IDs to high-cardinality metric labels.

Never assume an exact join exists between GPU metrics and an application trace. Correlate via verified pod-to-node-to-GPU allocation at the event time, respecting rescheduling and MIG changes. Store high-cardinality identifiers in access-controlled logs/traces, not as unbounded Prometheus labels.

### 3.1 Time and clock discipline

Record event timestamp, ingestion timestamp, timezone, source clock, precision, scrape interval and estimated skew. Normalize to UTC while preserving original timestamps. For Prometheus, a sample time indicates scrape/ingest time, not necessarily when a GPU fault occurred. Kubernetes Event \`eventTime\`, deprecated \`firstTimestamp\`/\`lastTimestamp\`, and object metadata can differ; inspect the actual API representation. Use time windows rather than exact millisecond equality across systems.

Example evidence record:

\`\`\`json
{
  "incident_id": "STAGING-EXAMPLE-072",
  "source": "kubernetes-event",
  "observed_at_utc": "2026-01-01T12:00:05Z",
  "ingested_at_utc": "2026-01-01T12:00:09Z",
  "cluster": "gpu-staging",
  "node": "gpu-node-redacted",
  "pod_uid": "REDACTED",
  "gpu_uuid": null,
  "event_type": "NodeNotReady",
  "evidence_uri": "REPLACE_WITH_PRIVATE_EVIDENCE_LINK",
  "confidence": "source-reported",
  "clock_skew_estimate_ms": null
}
\`\`\`

This is a **synthetic example**, not a real incident.

## 4. Read-only Kubernetes evidence collection

\`\`\`bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get events.events.k8s.io -A -o json
kubectl describe node <gpu-node>
kubectl get pods -A --field-selector spec.nodeName=<gpu-node> -o json
kubectl -n <namespace> get pod <pod-name> -o json
kubectl -n <namespace> logs <pod-name> --all-containers=true --timestamps=true --since=2h
kubectl -n <namespace> logs <pod-name> --previous --timestamps=true
\`\`\`

Some Kubernetes versions/clusters expose Event APIs differently; use \`kubectl api-resources | grep -i event\` to discover. \`--previous\` is available only for a previous terminated container instance. Collect only approved log scopes and redact secrets, request payloads and customer data.

For approved GPU host access:

\`\`\`bash
date -u +%Y-%m-%dT%H:%M:%SZ
nvidia-smi -L
nvidia-smi -q -d ECC,PCI,PERFORMANCE
journalctl -k --since '-2 hours' --output=short-iso-precise | grep -Ei 'NVRM|Xid|nvidia|AER'
\`\`\`

## 5. Prometheus and OpenTelemetry correlation

\`\`\`promql
# Observed GPU utilization for the investigation window
DCGM_FI_DEV_GPU_UTIL

# Latest observed Xid gauge if this metric is supported and semantics verified
DCGM_FI_DEV_XID_ERRORS

# DCGM target health; adapt job selector
up{job=~".*dcgm.*"}

# Node readiness via kube-state-metrics, if installed
kube_node_status_condition{condition="Ready",status="true"}
\`\`\`

For each series, record query start/end, step, labels and source metric type. A missing series is unknown telemetry, not a healthy device. Xid gauge values must not be treated as event counters without verification.

Trace search should use bounded service and time filters. Confirm span context propagation and sampling; missing spans may indicate instrumentation gaps, not absence of a request. Where logs include \`trace_id\`/\`span_id\`, link them to spans. Correlate GPU symptoms to a workload using a time-bounded allocation record, not simply matching hostnames.

## 6. Incident timeline reconstruction workflow

1. **Declare incident scope:** impacted service, cluster, start time and user symptoms.
2. **Freeze evidence:** capture current API objects, relevant logs and dashboard query links before retention expires.
3. **Normalize timestamps:** UTC, event versus ingestion time, clock skew and scrape lag.
4. **Build identity map:** pod UID → node → GPU/MIG device → workload owner at each relevant time.
5. **Layer events:** hardware/driver → node → scheduler → application → customer impact.
6. **Identify hypotheses:** e.g., GPU failure, driver restart, storage stall, rollout, network partition.
7. **Test hypotheses:** look for independent corroboration and contradicting evidence.
8. **Record confidence:** confirmed, likely, possible or unknown, with source references.
9. **Document actions:** who authorized, what changed, timestamps and rollback.
10. **Close with review:** incident report, gap analysis, retention and alert improvements.

### 6.1 Example timeline — synthetic, not observed

| UTC time | Signal | Observation | Interpretation |
|---|---|---|---|
| 12:00:00 | Cloud audit | Node maintenance request | Potential change trigger |
| 12:00:04 | Kernel | NVRM Xid reported | GPU symptom, code-specific |
| 12:00:10 | Kubernetes | NodeNotReady | Node control-plane effect |
| 12:00:25 | Scheduler | Training pods Pending | Scheduling impact |
| 12:01:05 | Workload | NCCL timeout | Training failure |
| 12:02:00 | SLO | Job progress drops | Customer-facing consequence |

These times illustrate how to order evidence; they do **not** establish that maintenance caused the Xid or that the Xid caused the NCCL timeout. Independent verification is required.

## 7. Event collection and retention design

Kubernetes Events are not durable audit logs. For production, use a supported event exporter or an approved OTel Collector receiver where available, and send to a secure, searchable backend with retention policy. Avoid duplicate event collection by multiple agents. Enforce least-privilege read-only Kubernetes RBAC, TLS, authentication, tenant isolation, payload redaction and bounded cardinality.

Define retention tiers based on incident response needs: high-resolution metrics for immediate diagnosis, logs/events long enough for on-call investigation, traces with intentional sampling, and durable audit/change records according to policy. Quantify cost and privacy risk before increasing retention.

### 7.1 Automation boundaries

Safe automation: capture read-only diagnostics, create incident ticket, attach time-bounded queries, identify probable owners, preserve redacted evidence. Unsafe without approval: GPU reset, node drain, workload deletion, modifying alert rules in production, or exporting raw customer payloads.

## 8. Hands-on exercises — 14 labs

1. Inventory available Kubernetes, GPU, Prometheus, logs, traces and cloud event sources.
2. Define a correlation-key schema with stable IDs and privacy classifications.
3. Check NTP/time synchronization and document estimated skew.
4. Collect Kubernetes Events in both human-readable and structured formats.
5. Capture GPU node and driver logs with approved access.
6. Query DCGM GPU metrics and verify Xid metric semantics.
7. Trace a pod UID through scheduling, restart and node allocation history.
8. Link a synthetic application trace to structured logs using trace context.
9. Correlate a workload timeline with GPU telemetry using a verified mapping.
10. Reconstruct a **synthetic** multi-layer GPU failure timeline.
11. Introduce a staging-only workload restart and record the event sequence.
12. Test missing telemetry, sampling gaps and expired Kubernetes Events.
13. Write a structured incident narrative with alternative hypotheses and confidence.
14. Complete peer review, runbooks, retention/security assessment and scoring.

## 9. Troubleshooting matrix

| Symptom | First checks | Avoid |
|---|---|---|
| Events missing | API version, TTL, permissions, exporter | claiming no incident occurred |
| Conflicting timestamps | UTC conversion, skew, scrape lag | exact joins on different clocks |
| GPU event without workload match | pod UID, GPU UUID, MIG mapping, time | hostname-only correlation |
| Partial trace | sampling, propagation, service instrumentation | assuming absent spans imply success |
| Duplicate events | event aggregation, overlapping exporters | counting duplicates as separate incidents |
| No driver logs | journal permissions, rotation, host reboot | reconstructing from Prometheus alone |
| Sensitive logs | redaction and backend permissions | copying raw logs into tickets |

## 10. Production runbooks

**A — GPU incident timeline:** freeze evidence, identify GPU/node/pod mapping, normalize UTC, correlate Xid, Kubernetes Events and workload failures; escalate hardware if verified. Exit with reproducible timeline and evidence references.

**B — Missing event history:** verify Kubernetes Event TTL and collector health; recover from durable backend or audit records if available; record unknown intervals rather than fabricate events. Exit with documented evidence gaps.

**C — Training failure correlation:** collect NCCL and checkpoint logs, scheduler decisions, GPU health and network telemetry; test competing hypotheses. Exit with verified cause or clearly bounded uncertainty and recovery plan.

**D — Telemetry privacy incident:** stop unauthorized export, secure evidence, notify security owner, assess retention and affected systems, apply validated redaction. Exit after review and approved resumption.

## 11. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Source inventory and evidence provenance | 20 |
| Identity mapping and time normalization | 20 |
| Multi-signal timeline correctness | 20 |
| Missing-data and competing-hypothesis analysis | 15 |
| Staging exercise and incident runbooks | 15 |
| Security, retention and peer review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 with no critical security/privacy or unsupported-causality failure. Documentation publication does not mean staging drills or independent review were completed.

## 12. References

- [Kubernetes Events](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/event-v1/)
- [Kubernetes Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [OpenTelemetry Concepts](https://opentelemetry.io/docs/concepts/)
- [OpenTelemetry Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/)
- [NVIDIA Xid Errors](https://docs.nvidia.com/deploy/xid-errors/)
- [Prometheus Querying Basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)

**Next:** Chapter 073 — Observability Pipeline Reliability.
