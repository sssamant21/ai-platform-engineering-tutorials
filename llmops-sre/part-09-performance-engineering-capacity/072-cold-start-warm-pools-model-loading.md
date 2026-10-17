# Chapter 072 — Cold Start, Warm Pools and Model Loading

**Part 09 — Performance Engineering and Capacity · Chapter 072 of 100**  
**Publication:** Chapter and companion lab published. **Live execution and technical/security review:** pending.

## Learning objectives
- Explain cold-start stages for LLM inference: scheduling, image pulls, GPU provisioning, model artifact transfer, deserialization, weight placement, compilation, KV reservation and warmup.
- Distinguish model loading latency from readiness and first-token latency.
- Size warm pools against traffic variability, scale-up delay, failover margin and GPU cost.
- Establish readiness, admission, rollout and SLO acceptance gates for new replicas.
- Diagnose slow starts, failed warmups, GPU OOMs and cold-start incident cascades.

## 1. Startup architecture
```text
autoscaler decision
      |
GPU capacity scheduling
      |
container image / driver / runtime initialization
      |
model artifact fetch (object store / registry / cache)
      |
checksum/signature and tokenizer compatibility verification
      |
weights load / quantization / GPU placement
      |
kernel compilation / CUDA graph capture / KV reservation
      |
synthetic warmup and health verification
      |
readiness TRUE -> controlled admission -> user traffic
```

A process listening on a port is **not necessarily ready** for inference. Define separate startup, liveness and readiness checks. Liveness should not kill healthy-but-slow model loading; readiness should fail until the selected model revision can complete a bounded synthetic request. A warm pool is spare, already initialized capacity that can absorb traffic while new replicas load.

## 2. Latency decomposition
```text
cold_start_total =
  scheduling + image + artifact_transfer + verification
  + load_to_memory + compile + warmup + readiness_delay

time_to_usable_capacity =
  autoscaler_detection + cold_start_total + routing_propagation
```
These stages can overlap in real systems, so wall-clock total should be measured from timestamps rather than blindly summing overlapping spans. Record p50/p95 and tail behavior by model size, artifact cache state, GPU SKU, region, storage path and deployment revision.

| Stage | Common bottleneck | Observability |
|---|---|---|
| Scheduling | GPU scarcity, taints, quotas | pending pods, unschedulable reasons |
| Image pull | registry latency, large image | pull duration, errors |
| Artifact transfer | object-store throughput, cross-region egress | bytes/sec, retries, checksum failures |
| Weight placement | GPU memory bandwidth, topology | load phase duration, GPU memory |
| Compilation/warmup | kernel compile, CUDA graphs | warmup duration, cache hit |
| Readiness | probe threshold, route propagation | readiness transition and 5xx |
| First request | lazy initialization | first-request TTFT vs warmed TTFT |

## 3. Warm-pool sizing
For a simplified fixed-capacity service:
```text
extra_replicas_needed ≈ ceil(max(0, peak_RPS − existing_safe_RPS) / safe_RPS_per_replica)
```
If scale-up requires T seconds, warm capacity must absorb the **expected demand during that interval**, with an additional failure-domain reserve. A static warm-pool count is not a universal answer: arrival bursts, model footprint, routing, multi-tenant demand and N-1 resilience alter requirements. GPU idle cost must be compared against cold-start SLO violations and incident risk.

Example: safe capacity 12 RPS per replica, forecast peak 55 RPS, and 3 warm serving replicas (36 RPS) imply at least `ceil((55−36)/12)=2` more ready replicas for peak, before N-1 and headroom. This is illustrative, not a production recommendation.

## 4. Deployment patterns
- **Pre-pulled artifacts:** reduce network transfer but require digest pinning, signature validation and storage eviction policy.
- **Immutable model digests:** prevent revision mismatch during rollout and rollback.
- **Warm pools:** keep standby capacity loaded; monitor stale versions, readiness, cost and eviction.
- **Progressive rollout:** initialize candidate, pass synthetic warmup, then route a small approved canary fraction.
- **Scale-down safeguards:** avoid oscillation and repeated cold starts through cooldowns, minimum replicas and workload-aware thresholds.
- **Regional failover:** verify model artifact locality, quotas and warm capacity in the secondary region.
- **Disaggregated serving:** evaluate separate prefill/decode worker readiness and KV transfer dependencies.

## 5. Benchmark procedure
1. Pin model digest, tokenizer, container image, GPU SKU, runtime, region, storage source and cache state.
2. Test **cold cache**, **warm local artifact cache**, **warm model in GPU**, and **rolling replacement** independently.
3. Instrument each stage with trace spans and monotonic timestamps; record failures and retry costs.
4. Inject safe staging faults: slow artifact download, checksum failure, unavailable GPU, failed warmup, insufficient KV headroom and readiness flapping.
5. Run synthetic inference probes and verify readiness transitions before traffic.
6. Measure p95 cold-start, time to usable capacity, queue/rejection during scale-out, cost of warm pool and recovery time after N-1 loss.
7. Stop tests on approved error budget, spend, timeout and capacity thresholds.

## 6. Example Kubernetes controls
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-example
spec:
  replicas: 2
  selector:
    matchLabels: {app: inference-example}
  template:
    metadata:
      labels: {app: inference-example}
    spec:
      containers:
        - name: inference
          image: registry.example.invalid/inference@sha256:REPLACE_WITH_REAL_DIGEST
          ports:
            - containerPort: 8000
          startupProbe:
            httpGet: {path: /startup, port: 8000}
            periodSeconds: 10
            failureThreshold: 90
          readinessProbe:
            httpGet: {path: /ready, port: 8000}
            periodSeconds: 5
            failureThreshold: 2
          livenessProbe:
            httpGet: {path: /live, port: 8000}
            periodSeconds: 20
            failureThreshold: 3
          resources:
            requests:
              nvidia.com/gpu: "1"
            limits:
              nvidia.com/gpu: "1"
```
**Illustrative only:** placeholder image digest, endpoints and GPU resources must be adapted to the actual inference runtime and cluster. A startup probe suppresses liveness/readiness until startup succeeds, so tune its budget against measured cold starts. Protect readiness from accidental public exposure.

## 7. Troubleshooting
| Symptom | First investigation |
|---|---|
| Pending GPU pods | capacity, quotas, device plugin, node selectors |
| Repeated image pulls | digest/tag drift, registry authentication, cache eviction |
| Model load stalls | object-store throttling, disk IO, network and checksums |
| GPU OOM during startup | weights, temporary buffers, CUDA graph capture, KV reserve |
| Ready before model loaded | probe contract and lazy initialization |
| Warm pool consumes too much budget | replica hours, standby ratio, scale-down policy |
| Cold-start latency increases after deploy | larger artifact, compile cache miss, runtime changes |
| Failover cold-start exceeds RTO | regional artifact/cache locality and warm capacity |

## 8. Production runbooks

### RB-072-A — Cold-start regression
**[SAFE-READ]** compare stage durations, model digest, image size, cache state, GPU SKU and rollout timestamps. **[CHANGE-CONTROL]** revert deployment or prefetch strategy after approval. Confirm readiness and first-token SLO recovery.

### RB-072-B — Model load failure
Check artifact integrity, registry/object-store access, tokenizer compatibility, available disk/GPU memory and runtime logs with secrets redacted. Do not bypass signature or checksum verification to restore service.

### RB-072-C — Warm-pool exhaustion
Measure offered load, ready replicas, per-replica safe throughput, queue age and rejection rate. Apply approved admission and traffic routing; do not declare newly scheduled pods as usable capacity until warmup passes.

### RB-072-D — N-1 readiness acceptance
Remove one staging replica under change control, measure queue/TTFT/recovery and verify warm standby availability, rollback and cost budget. Record evidence and owner approval.

## 9. Security and acceptance
- Use signed immutable artifacts; verify model digest and tokenizer compatibility.
- Never expose credentials or customer prompts in warmup probes.
- Protect artifact caches and enforce tenant isolation.
- [ ] Cold/warm start definitions documented.
- [ ] Startup, liveness and readiness semantics validated.
- [ ] Stage latency and failure metrics observable.
- [ ] Warm capacity and N-1 resilience justified.
- [ ] Safe rollback, budget caps and admission policy approved.
- [ ] Technical/security review and live execution tracked separately.

## References
- [Kubernetes configure liveness, readiness and startup probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Google SRE Workbook — Handling Overload](https://sre.google/workbook/handling-overload/)
- [OpenTelemetry tracing](https://opentelemetry.io/docs/concepts/signals/traces/)

## Hands-on lab
[Chapter 072 — Cold Start, Warm Pools and Model Loading Lab](../labs/072-cold-start-warm-pools-model-loading-check.md).

**Next:** Chapter 073 — Load, Stress, Soak and Failure Testing.
