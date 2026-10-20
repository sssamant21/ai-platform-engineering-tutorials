# Chapter 069 — OpenTelemetry for AI Infrastructure

**Part 09 — GPU Observability | AI Infrastructure Engineering & SRE**

**Publication:** Documentation published. Offline execution, live staging validation, security review and independent technical acceptance remain pending.

## 1. Outcomes and prerequisites

This chapter teaches SREs to design and operate OpenTelemetry (OTel) pipelines for Kubernetes-hosted AI workloads, correlate training/inference telemetry with GPU and node health, deploy a secure Collector, control cost and cardinality, and troubleshoot telemetry loss without mistaking missing telemetry for a healthy service.

Prerequisites: Chapters 067–068; Kubernetes access; Prometheus/Grafana basics; OTLP, PromQL and distributed tracing fundamentals. Examples use a **disposable staging namespace** and require adapting endpoints, versions, labels and RBAC to the target environment.

### 1.1 Telemetry types

| Signal | AI infrastructure examples | Source | SRE use |
|---|---|---|---|
| Metrics | GPU busy %, framebuffer, training step time, queue depth, inference latency | DCGM exporter, applications, kube-state-metrics | Trends, capacity, SLOs |
| Traces | inference request, retrieval, model serving, checkpoint write | instrumented applications | Critical-path latency and dependencies |
| Logs | Kubernetes events, model server failures, NCCL errors, node agent output | workload stdout/stderr, approved agents | Root cause and incident timeline |
| Profiles | CPU/heap profiling when supported | profiling tools | Expensive code paths; separate governance |

**Important boundary:** DCGM Exporter ordinarily exposes Prometheus metrics; it does **not** automatically generate distributed traces. A trace ID cannot be invented for GPU hardware counters. Correlation uses shared low-cardinality resource attributes and time windows, plus application instrumentation where traces exist.

## 2. Reference architecture

\`\`\`text
Instrumented AI app (SDK / auto-instrumentation)
   ├─ OTLP/gRPC :4317 or OTLP/HTTP :4318
   └─ trace context propagated between services
             |
             v
   OTel Collector gateway (Kubernetes)
     receivers: OTLP
     processors: memory_limiter → resource / attributes → batch
     exporters: approved metrics / traces / logs backends
             |
             v
     Grafana / Prometheus-compatible backend / tracing backend
             ^
             |
GPU nodes → DCGM Exporter → Prometheus scrape (Chapter 067)
Kubernetes → kube-state-metrics / node-exporter / events
\`\`\`

Use an **agent DaemonSet** for node-local collection only when necessary; a **gateway Deployment** for OTLP ingestion, buffering and centralized policy. Decide deliberately whether logs are read by an agent, Fluent Bit or another approved component: running overlapping collectors causes duplication. A Collector is not a durable message queue; for prolonged backend outages use a reviewed persistent queue or message broker architecture.

### 2.1 Control plane and data plane

- Control plane: Kubernetes namespaces, NetworkPolicies, ServiceAccounts, RBAC, GitOps and approved Collector configuration.
- Data plane: workload SDK → OTLP receiver → processors → exporter → backend.
- Security: internal TLS/mTLS where required, authenticated backend export, restricted ingress, namespace isolation, Secret-backed credentials, egress allowlists and redaction before export.
- Reliability: collector CPU/memory limits, autoscaling from measured load, receiver/exporter queue metrics, dropped spans, backpressure and backend availability.

## 3. Resource attributes and correlation contract

Use standard OpenTelemetry semantic conventions when available, and document custom keys separately. Attribute availability depends on instrumentation and enrichment configuration.

| Attribute | Example | Cardinality / privacy |
|---|---|---|
| \`service.name\` | \`model-serving\` | Stable service identifier |
| \`service.namespace\` | \`ai-platform\` | Stable bounded grouping |
| \`deployment.environment.name\` | \`staging\` | Low cardinality |
| \`k8s.cluster.name\` | \`gpu-staging\` | Bounded cluster identity |
| \`k8s.namespace.name\` | \`ai-workloads\` | Scope authorization |
| \`k8s.node.name\` | \`gpu-node-1\` | Operational metadata; restrict access |
| \`k8s.pod.name\` | pod name | Ephemeral; **do not** blindly promote to metric labels |
| \`gpu.uuid\` | hardware UUID | Custom attribute; access-controlled and potentially high cardinality |
| \`ai.workload.class\` | \`training\` / \`inference\` | Custom bounded enumeration |

**Avoid** prompt contents, training data paths, user IDs, API keys, full request bodies, model inputs, raw job IDs or trace IDs as metric labels. Trace/span IDs belong in tracing/log correlation fields, not Prometheus series labels. Keep \`service.name\` and environment consistent across SDK, Collector and backend.

For a GPU incident, join signals by a documented timeline and node/pod/workload mapping; do not assert causal linkage from timestamp overlap alone.

## 4. Minimal secure staging Collector gateway

Pin a reviewed OpenTelemetry Collector **contrib** image by immutable digest in the deployment workflow. The configuration below uses only standard Collector components and a **debug exporter**, deliberately restricted to a disposable staging environment; never log production payloads with it.

**ConfigMap (staging example):**

\`\`\`yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-gateway-config
  namespace: ai-observability-lab
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      memory_limiter:
        check_interval: 1s
        limit_mib: 256
        spike_limit_mib: 64
      batch:
        timeout: 5s
        send_batch_size: 512
    exporters:
      debug:
        verbosity: basic
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [debug]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [debug]
      telemetry:
        metrics:
          level: normal
\`\`\`

**Deployment and Service (staging only):**

\`\`\`yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-gateway
  namespace: ai-observability-lab
spec:
  replicas: 1
  selector:
    matchLabels: {app: otel-gateway}
  template:
    metadata:
      labels: {app: otel-gateway}
    spec:
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        seccompProfile: {type: RuntimeDefault}
      containers:
        - name: collector
          image: otel/opentelemetry-collector-contrib:REPLACE_WITH_REVIEWED_VERSION
          args: ["--config=/conf/config.yaml"]
          ports:
            - {name: otlp-grpc, containerPort: 4317}
            - {name: otlp-http, containerPort: 4318}
          resources:
            requests: {cpu: 100m, memory: 128Mi}
            limits: {cpu: "1", memory: 384Mi}
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: {drop: ["ALL"]}
          volumeMounts:
            - {name: config, mountPath: /conf, readOnly: true}
      volumes:
        - name: config
          configMap: {name: otel-gateway-config}
---
apiVersion: v1
kind: Service
metadata:
  name: otel-gateway
  namespace: ai-observability-lab
spec:
  selector: {app: otel-gateway}
  ports:
    - {name: otlp-grpc, port: 4317, targetPort: otlp-grpc}
    - {name: otlp-http, port: 4318, targetPort: otlp-http}
\`\`\`

The image tag is a **non-runnable placeholder by design**: replace it with a tested, pinned digest after validating architecture and image provenance. Apply a default-deny NetworkPolicy plus an explicit namespace-scoped allow rule before opening the OTLP ports. Some managed clusters enforce additional Pod Security or image restrictions; adapt in staging. The memory limiter is not a substitute for capacity planning. Production pipelines should replace \`debug\` with authenticated, encrypted exporters and measured queue/retry behavior.

### 4.1 Validate before rollout

\`\`\`bash
kubectl get ns ai-observability-lab
kubectl -n ai-observability-lab get networkpolicy
kubectl -n ai-observability-lab get deployments,services,configmaps
# Use a reviewed, pinned collector binary matching the image version:
otelcol-contrib validate --config=config.yaml
kubectl -n ai-observability-lab apply --dry-run=server -f collector.yaml
# After explicit staging approval:
kubectl -n ai-observability-lab apply -f collector.yaml
kubectl -n ai-observability-lab rollout status deploy/otel-gateway --timeout=180s
kubectl -n ai-observability-lab logs deploy/otel-gateway --tail=100
\`\`\`

**Version warning:** Collector component names and telemetry settings evolve. Verify the chosen release's supported configuration using its actual binary. A \`kubectl --dry-run=server\` does not validate internal Collector component configuration.

## 5. Instrument an AI service

Set SDK exporter endpoint to the in-cluster gateway Service, and propagate W3C \`traceparent\` and \`tracestate\` across HTTP/gRPC boundaries. In Python, a minimal explicit trace is:

\`\`\`python
# pip install opentelemetry-api opentelemetry-sdk
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(ConsoleSpanExporter()))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("ai-sre-staging")

with tracer.start_as_current_span("inference.request") as span:
    span.set_attribute("ai.workload.class", "inference")
    span.set_attribute("model.family", "approved-demo")
    # No prompt, PII, token, or payload contents in span attributes.
    pass
\`\`\`

This sample writes traces to the console **only** and does not prove Collector ingestion. To send OTLP, install a compatible OTLP exporter package, configure the endpoint and transport deliberately, then verify the gateway received spans. Prefer framework auto-instrumentation where reviewed, and explicitly instrument expensive AI stages: request admission, preprocessing, retrieval, inference, postprocessing and checkpoint I/O. Never use traces to infer GPU power or ECC status.

## 6. Collector health, pipeline SLOs and backpressure

The Collector exposes internal telemetry; endpoint availability and exact metric names depend on version and configuration. Inspect the Collector's configured telemetry endpoint and emitted series rather than assuming a port or name.

| Reliability signal | Question | Failure interpretation |
|---|---|---|
| Accepted spans/points | Is input arriving? | Sudden zero may mean upstream outage or discovery loss |
| Exported spans/points | Is the backend receiving? | Falling output with stable input suggests bottleneck |
| Refused / dropped telemetry | Are we losing data? | Memory pressure, queue full, sampling or exporter errors |
| Export queue utilization | Can the gateway absorb bursts? | Sustained growth predicts drops |
| Export failures / retries | Is the backend healthy? | Network, auth, TLS or rate limiting |
| Collector CPU/memory | Is the Collector resource-bound? | OOM/restarts and ingestion gaps |
| End-to-end lag | Is telemetry timely enough for on-call? | Backend delays may hide active incidents |

Illustrative **design** objectives, not established production SLOs: ≥99.9% Collector pod availability, ≥99.5% of accepted telemetry successfully exported over 30 days, and p95 ingestion lag <60s. Specify how dropped/sampled traffic is accounted for and validate that backend receipt can actually be measured. Define service-specific targets with stakeholders before treating these as policy.

### 6.1 Sampling and cost

- Use head sampling for predictable cost, recognizing that it can miss rare failures.
- Tail sampling requires a supported processor and trace affinity so all spans of a trace reach the same decision point; it adds memory and latency.
- Never sample away security/audit events without approval; distinguish tracing samples from audit logs.
- Estimate data volume as \`events_per_second × bytes_per_event × seconds × replication_factor\`; add indexing/retention overhead and bursts.
- Keep high-cardinality span attributes out of metric dimensions. Verify histogram bucket strategy and exemplar support in the chosen backend.

## 7. Hands-on lab — 14 exercises

1. Inventory existing OTLP receivers, SDKs, Collector releases and tracing/logging backends.
2. Draw a per-signal dataflow with ownership, trust boundaries and failure modes.
3. Review telemetry attribute contracts for privacy and cardinality.
4. Create a disposable staging namespace with namespace-scoped NetworkPolicies.
5. Validate Collector YAML and reviewed image pinning; run server-side Kubernetes dry-run.
6. Deploy the isolated gateway after change approval and inspect startup diagnostics.
7. Emit a synthetic, non-sensitive trace and verify actual receiver/exporter observations.
8. Emit a bounded synthetic metric and confirm backend metric semantics.
9. Confirm a known GPU metric remains available through the existing DCGM/Prometheus path.
10. Correlate a synthetic inference trace with Kubernetes pod/node context and GPU telemetry by time, without asserting false causation.
11. Exercise a staging-only exporter/backend outage and measure queue growth, loss and recovery.
12. Test cardinality and attribute redaction with intentionally synthetic high-cardinality values; do not stress shared production.
13. Estimate throughput, storage, retention and Collector sizing under peak workload.
14. Export evidence, execute runbooks and complete a peer-reviewed acceptance score.

## 8. Troubleshooting matrix

| Symptom | First read-only checks | Likely cause | Recovery direction |
|---|---|---|---|
| No traces | SDK exporter config, Service DNS, receiver logs | wrong OTLP protocol/port, TLS, sampling | correct client/gateway contract |
| Partial traces | context propagation, sampling, service mesh | missing instrumentation or split trace | validate headers and trace affinity |
| Metrics but no GPU context | DCGM scrape, label map, time range | distinct pipelines or missing node mapping | reconcile by inventory and time |
| Duplicate spans | multiple agents, retry behavior, instrumentation | duplicate export/instrumentation | eliminate overlapping collection |
| Collector OOM | pod events, queue/memory stats | burst or oversized batches | size limits and reduce load safely |
| Backend rejects telemetry | exporter status, TLS, auth, quotas | invalid credentials, throttling | restore credentials and backoff policy |
| Cross-tenant visibility | RBAC, backend tenant isolation | unscoped attributes/datasource | revoke access, incident escalation |
| Traces contain sensitive content | span/log sample and processors | unsafe SDK attributes | stop export, rotate secrets if exposed, follow incident policy |

## 9. Runbooks

**A — Traces missing:** verify instrumentation and propagation → OTLP endpoint/protocol → gateway receiver → exporter/backend → sampling. Preserve redacted evidence; exit when a known synthetic trace is queryable end to end.

**B — Collector saturation:** check pod OOM/restarts, accepted/refused/exported counters, queue pressure and backend throttling. Scale only after identifying the bottleneck; exit when sustained input is exported without unacceptable loss.

**C — GPU-to-trace correlation mismatch:** verify node/pod scheduling history, GPU/MIG identity, clock synchronization, sampling and query time windows. Do not attribute an Xid event to a specific trace without corroboration.

**D — Sensitive data in telemetry:** stop or quarantine the offending export, engage security, identify affected destinations/retention, rotate exposed credentials and apply tested attribute filters before resuming.

## 10. Acceptance assessment (100 points)

| Gate | Points |
|---|---:|
| Architecture, ownership and threat model | 15 |
| Validated staging Collector and configuration | 20 |
| Trace/metric propagation and semantic correctness | 20 |
| GPU/Kubernetes correlation without false joins | 15 |
| Failure drill, backpressure and recovery evidence | 15 |
| Privacy, cardinality, runbooks and peer review | 15 |
| **Total** | **100** |

**Pass:** at least 85/100 **and** no critical security/privacy failure. Publication, offline validation, staging runtime results and independent review must be tracked separately. Do not mark any test passed without actual evidence.

## 11. References

- [OpenTelemetry Collector documentation](https://opentelemetry.io/docs/collector/)
- [Collector configuration](https://opentelemetry.io/docs/collector/configuration/)
- [OpenTelemetry semantic conventions](https://opentelemetry.io/docs/specs/semconv/)
- [OTLP specification](https://opentelemetry.io/docs/specs/otlp/)
- [Kubernetes OpenTelemetry guidance](https://opentelemetry.io/docs/platforms/kubernetes/)
- [NVIDIA DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)

**Next:** Chapter 070 — GPU Health and Xid Alerting.
