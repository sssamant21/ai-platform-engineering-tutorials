# Chapter 11.2 — Hands-on Lab: Logging Architecture, Structured Logs and Kubernetes Events

**Part 11 | Isolated staging | Live execution pending**

## Mission and safeguards

Validate TrueFoundry agent/application/inference logs, Kubernetes Events, structured JSON, collection, retention, redaction, correlation, alerting and incident recovery. **[SAFE-READ]** inspection; **[LAB-MUTATE]** synthetic staging; **[CHANGE-CONTROL]** production review. No real prompts, patient data, credentials, tokens or model weights in test events.

## Lab 0 — Preflight [SAFE-READ]

```bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -40
kubectl get deployments,daemonsets -A
kubectl get svc -A
```

Record collector type, backend (Loki/OpenSearch/cloud), agent namespace, gateway workload, Pod logging format, Event exporter availability, retention, RBAC, buffer size and reviewer. Do not assume any optional integration exists.

## Lab 1 — Kubernetes event timeline [SAFE-READ]

For an approved staging Pod, capture Pod UID, image digest, creation timestamp, container restarts, relevant Warning Events and bounded current/previous logs. Build a UTC timeline without exposing sensitive fields. Confirm event age/retention expectations.

## Lab 2 — Structured synthetic log [LAB-MUTATE]

Emit harmless JSON events with `timestamp`, `level`, `service`, `trace_id`, `request_id`, `event`, `duration_ms`, `status_code` from a disposable workload. Confirm collector parses them, adds namespace/Pod UID, preserves timestamp and exposes a searchable record. Do not include payload bodies.

## Lab 3 — Correlation and queries

Generate one approved synthetic inference request; correlate gateway, model server and platform agent where supported using safe request/trace IDs. Test narrow time-bounded queries, error counts and log-to-trace navigation. Record missing spans/logs as instrumentation gaps.

## Lab 4 — Collector recovery tabletop

Using a disposable collector or tabletop, rehearse backend unavailable, retry/backpressure, buffer overflow, log rotation, duplicate delivery and parser failure. Never disable production log collection. Confirm alert ownership and expected loss/duplication semantics.

## Lab 5 — Redaction, retention and cleanup

Review synthetic forbidden-field patterns without emitting actual secrets. Validate access controls and retention policy. Delete lab-owned workload and synthetic events according to policy; restore any lab-only configuration and verify collector health.

## Twenty-eight failure scenarios

| # | Scenario | Investigation focus |
|---|---|---|
| 01 | Pod name incorrect | Namespace/workload |
| 02 | Container name incorrect | Multi-container Pod |
| 03 | Previous container logs unavailable | Runtime retention |
| 04 | CrashLoopBackOff with empty logs | Exit reason/events |
| 05 | Kubernetes Event expired | Event TTL |
| 06 | Warning Events not exported | Event collector |
| 07 | TrueFoundry agent log source absent | Agent deployment |
| 08 | Gateway access log missing | Logging config |
| 09 | vLLM stderr not collected | Container stream |
| 10 | Collector DaemonSet unscheduled | Node selectors/taints |
| 11 | Collector file permission denied | Runtime/Pod security |
| 12 | Collector RBAC denied | API permissions |
| 13 | Collector TLS handshake fails | Certificates |
| 14 | Collector backend 401/403 | Identity |
| 15 | Collector backend 429 | Rate limit |
| 16 | Collector backend 5xx | Retry/buffer |
| 17 | Buffer fills and drops | Capacity |
| 18 | Log rotation loses records | Tail/checkpoints |
| 19 | Duplicate logs after retry | Delivery semantics |
| 20 | JSON parsing fails | Schema |
| 21 | Multiline stack trace split | Parser |
| 22 | Timestamp timezone wrong | UTC parsing |
| 23 | Missing trace ID | Propagation |
| 24 | Kubernetes metadata missing | Enrichment |
| 25 | High-cardinality labels | Cost |
| 26 | Sensitive field not redacted | Privacy incident |
| 27 | Unauthorized log query access | RBAC/tenant boundary |
| 28 | Log query latency excessive | Index/stream layout |

For each record expected/actual, UTC timestamp, environment, source, sanitized evidence, hypothesis, mitigation, rollback and reviewer. Perform active tests only in approved staging; otherwise mark tabletop.

## Ten acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Approved scope and logging source inventory | Pending |
| G2 | Pod logs and Kubernetes Events correlated | Pending |
| G3 | TrueFoundry agent/gateway/runtime sources verified | Pending |
| G4 | Structured JSON schema and metadata enrichment validated | Pending |
| G5 | Trace/request correlation demonstrated | Pending |
| G6 | Collector buffering/retry and loss semantics reviewed | Pending |
| G7 | Redaction, access control and retention validated | Pending |
| G8 | Log queries, dashboards and alerts tested | Pending |
| G9 | Collector recovery, rollback and cleanup documented | Pending |
| G10 | SRE/security review and sign-off | Pending |

**Pass** requires timestamped evidence and reviewer approval. Unexecuted or blocked tests remain pending.

## Evidence worksheet

UTC time, cluster, namespace, Pod UID, source/collector/backend, schema version, request/trace ID, query, observed lag, sanitized event reference, rollback commit, reviewer.

**Next:** Chapter 11.3 — Prometheus Metrics, Exporters and Recording Rules.
