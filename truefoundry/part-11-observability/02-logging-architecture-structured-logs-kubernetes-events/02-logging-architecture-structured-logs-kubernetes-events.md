# Chapter 11.2 — Logging Architecture, Structured Logs and Kubernetes Events

**Part 11 — Observability | Canonical production SRE tutorial**

## Objectives and operating contract

Design a secure, resilient logging pipeline for TrueFoundry control-plane integrations, cluster agents, application Pods, inference gateways, model servers and GPU nodes. Distinguish Kubernetes Events from container logs and cloud audit logs; correlate them with traces and metrics; build operational queries, retention policies and failure recovery.

**Execution:** [SAFE-READ] metadata/limited logs; [LAB-MUTATE] synthetic staging; [CHANGE-CONTROL] production changes. Confirm installed log agent (Fluent Bit, Vector, OpenTelemetry Collector or managed provider), actual TrueFoundry log capabilities, permissions and retention. Never assume logs exist in the platform UI. Avoid logging credentials, raw prompts/responses, PHI, model weights, authorization headers or environment dumps.

## 1. Logging reference architecture

```text
TrueFoundry cluster agent ────┐
Inference gateway/access logs ┤
App + vLLM stdout/stderr ─────┤
Kubernetes Events exporter ───┤→ node/cluster collectors → parse/redact/enrich
GPU device plugin/driver ─────┤                       ↓
Kubernetes API audit* ────────┤                 queue/buffer/retry
Cloud LB/IAM/registry logs ───┘                       ↓
                                              central log backend
                                         (Loki / OpenSearch / cloud)
                                                   ↓
                                       dashboards / alerts / incident
*Provider audit access/configuration varies by EKS, AKS and other clusters.
```

Container stdout/stderr is captured by kubelet/container runtime and made available via `kubectl logs` while retained locally; centralized collection needs an agent or supported integration. Kubernetes Events are API objects with limited retention, not durable application logs. Kubernetes audit events are a separate security source and may be managed by the cloud provider.

## 2. Structured logging contract

Prefer JSON lines, one event per record, with consistent fields:

```json
{"timestamp":"2026-10-15T12:00:00Z","level":"INFO","service":"inference-gateway","environment":"staging","namespace":"tfy-staging","trace_id":"00000000000000000000000000000001","request_id":"synthetic-req-001","route":"/v1/chat/completions","model_alias":"demo-model","status_code":200,"duration_ms":42,"event":"request_completed"}
```

Use UTC RFC3339 timestamps, stable severity levels, safe deployment/version identifiers, namespace, Pod UID, trace/span IDs, status, duration units, event name and error category. `trace_id` example is illustrative; real tracing requires correctly generated W3C IDs. Do not include high-cardinality request IDs as **metric labels**; request IDs are useful in logs with appropriate access and retention.

**Never log:** Authorization headers, bearer tokens, ServiceAccount tokens, database passwords, private keys, raw patient/user content, full inference prompts or unapproved response bodies. Redact at source and again at collector where appropriate. A redaction pipeline is not a substitute for preventing sensitive fields from being emitted.

## 3. Kubernetes logging and events [SAFE-READ]

```bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -50
kubectl -n YOUR_NAMESPACE get events --field-selector type=Warning
kubectl -n YOUR_NAMESPACE describe pod YOUR_POD
kubectl -n YOUR_NAMESPACE logs YOUR_POD --since=15m --tail=100
kubectl -n YOUR_NAMESPACE logs YOUR_POD -c YOUR_CONTAINER --previous --tail=100
kubectl -n YOUR_NAMESPACE get pod YOUR_POD -o jsonpath='{.metadata.uid}{"\n"}'
```

`--previous` is available only if a prior terminated container instance is still retained. `kubectl describe` and logs may contain sensitive content; review before sharing. Events such as `FailedScheduling`, `FailedMount`, `ErrImagePull`, `BackOff` and readiness probe failures are important deployment evidence, but may be rate-limited or expire.

## 4. TrueFoundry-specific boundaries

Locate actual agent deployment and namespace, then inspect bounded logs:

```bash
kubectl -n AGENT_NAMESPACE get deploy AGENT_DEPLOYMENT
kubectl -n AGENT_NAMESPACE logs deploy/AGENT_DEPLOYMENT --since=10m --tail=100
```

Differentiate **agent reconciliation**, **Kubernetes scheduler/controller**, **model-serving container**, **inference gateway** and **cloud identity/storage** errors. The TrueFoundry UI may expose some logs or links depending on edition/version and configuration. Verify timestamps, filtering, tenant access and source completeness before relying on it for incidents.

## 5. Collector design and durability

Deploy node-level log agents as approved DaemonSets or use provider-managed collection. Configure:
- Kubernetes metadata enrichment with bounded labels (namespace, Pod UID, workload, container, node).
- Multi-line parsing for stack traces without unlimited buffers.
- Parser failure handling, log severity mapping and schema versioning.
- TLS/mTLS and workload identity for collector-to-backend transport.
- Disk/memory buffers, retries with jitter, backpressure and overflow behavior.
- Separate security/audit logs from verbose application logs.
- Explicit collection of short-lived Jobs and crash-looping Pods.
- Drop/transform rules for noisy health checks and sensitive fields.
- Per-team retention, access controls, index/stream partitioning and cost budgets.

**Delivery semantics:** do not promise exactly-once logging. Retries can duplicate records; backend and queries should tolerate duplicates. Under outage, bounded buffers can drop logs; monitor drop counters and blind windows.

## 6. Query and correlation patterns

Start with cluster, namespace, workload and time window; narrow by Pod UID, deployment revision, request ID, trace ID and error category. Example Loki LogQL (adapt labels to your pipeline):

```logql
{namespace="tfy-staging",app="inference-gateway"} | json | level="ERROR"
```

```logql
sum by (app) (count_over_time({namespace="tfy-staging"} |= "timeout" [5m]))
```

Avoid relying on string matching alone for error rate SLIs; use instrumented metrics and structured error categories. In OpenSearch/Elasticsearch, use structured fields and time filters; avoid uncontrolled wildcard searches across all production indices. Correlate logs to traces through trace ID and to deployments through image digest/revision.

## 7. Logging security, compliance and cost

Implement tenant/workspace isolation, least-privilege read access, retention and deletion policy, encryption in transit/at rest, audit of log access and legal hold where applicable. Set storage budgets by log rate × average event size × retention × replication/compression assumptions; validate against measured ingestion. Do not copy incident logs into public issues. Treat log contents as potentially sensitive even after redaction.

## 8. Production dashboards and alerting

Monitor collector health, tailing lag, dropped/retried records, parser failures, backend ingestion errors, index/stream growth, query latency and missing-source heartbeat. Alert on sudden logging silence for critical workloads **only when traffic/heartbeat expectations are known**. Correlate alert with Kubernetes Events, metrics, traces and cloud provider logs to avoid false root causes.

## 9. Twelve operational troubleshooting runbooks

### RB-01 — Pod logs unavailable
Check Pod/container name, termination history, node and retention. Use centralized backend if local logs were rotated.

### RB-02 — CrashLoopBackOff without useful logs
Check `--previous`, exit reason/OOMKilled, events, startup probes, config and dependencies. Do not increase log verbosity blindly.

### RB-03 — Events missing after incident
Confirm event retention, event exporter/collection and cloud audit. Reconstruct from deployment controller and node logs if possible.

### RB-04 — TrueFoundry agent logs missing
Verify actual deployment, Pod health, namespace, collector inclusion and permissions.

### RB-05 — Collector cannot read node files
Check DaemonSet scheduling, host mounts, runtime log paths, Pod security policy and RBAC; avoid broad privileged exceptions.

### RB-06 — Collector backpressure/drops
Inspect buffer capacity, output latency, retry policy and backend health; throttle verbose logs before risking node disk pressure.

### RB-07 — Parser corrupts JSON or stack traces
Check schema, escaping, multiline rules, time parsing and dead-letter behavior.

### RB-08 — Duplicate logs after restart
Review offsets/checkpoints and retry semantics; deduplicate at query or ingestion layer as appropriate.

### RB-09 — Missing correlation IDs
Verify gateway/app instrumentation, propagation and collector field transformations.

### RB-10 — Sensitive data found in logs
Restrict access, stop emission, follow incident process, purge/rotate as approved and verify redaction. Assume copies may persist in downstream systems.

### RB-11 — Log query slow/cost spike
Inspect high-cardinality fields, time range, partitioning, retention, index/stream layout and expensive regex queries.

### RB-12 — Central logging outage
Use bounded local/Kubernetes/cloud telemetry, restore collector/backend, assess data loss and document blind window.

## 10. Change management and acceptance

Baseline log volume, schema, collector CPU/memory/disk, backend latency and retention. Test synthetic events and controlled collector interruption in staging, verify redaction, role-based access, correlation and recovery. Document rollback of configuration; never disable mandatory security audit to save cost.

Complete the [companion lab](../labs/02-logging-architecture-structured-logs-kubernetes-events-check.md) with **28 failure scenarios and 10 acceptance gates**. Publication does not imply execution or vendor review.

**Next:** Chapter 11.3 — Prometheus Metrics, Exporters and Recording Rules.
