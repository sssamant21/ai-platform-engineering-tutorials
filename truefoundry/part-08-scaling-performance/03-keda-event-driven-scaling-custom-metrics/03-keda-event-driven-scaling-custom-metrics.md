# 8.3 — KEDA Event-Driven Scaling and Custom Metrics

**Part 08 — Scaling & Performance | Chapter 3 of 7**  
**Audience:** TrueFoundry platform engineers, Kubernetes SREs, GPU inference operators  
**Execution labels:** [SAFE-READ] inventory; [LAB-MUTATE] approved isolated staging; [CHANGE-CONTROL] production changes.

## 1. Objectives

Explain KEDA architecture, ScaledObjects, trigger authentication, Prometheus/custom metric integration, HPA ownership, activation versus target thresholds, polling/cooldown/stabilization, GPU node capacity, and safe scaling of streaming inference. Implement a bounded staging exercise and diagnose failures using **Lowest Proven Healthy Layer (LPHL)** and **First Failed Transition (FFT)**.

**Prerequisites:** Chapters 8.1–8.2; approved TrueFoundry deployment and model; immutable vLLM image/model revision; Kubernetes read permissions; known GPU quota, model warmup duration, request/latency SLOs and rollback owner. KEDA may not be installed. Do not install cluster-scoped operators or mutate production without separate approval.

## 2. Where KEDA fits

```text
Authorized client → Gateway → Service → Ready vLLM Pods
                             │         │
                      requests/queue   ├─ running/waiting
                             │         ├─ tokens/s and KV
                             ▼         └─ TTFT / ITL
                 Prometheus / queue system
                             │
                  KEDA trigger scaler
                             │
                  ScaledObject controller
                      │               │
        activation (0 ↔ 1)     generated HPA (1 ↔ N)
                      │               │
                      └───── Deployment replicas
                                   │
                           GPU scheduler / nodes
                                   │
                        model load → warmup → Ready
```

KEDA does **not** supply GPU nodes or make model loading instantaneous. It uses an HPA for scaling between nonzero replica counts, while KEDA controls activation behavior where supported. Actual behavior depends on KEDA version, scaler type, and min/max replica configuration. TrueFoundry/GitOps remains the authoritative owner of application and scaling configuration; avoid competing controllers.

## 3. KEDA building blocks

| Component | Responsibility | Operator check |
|---|---|---|
| KEDA operator | Reconciles ScaledObjects | Healthy controller and CRDs |
| Metrics server/adapter | Exposes scaler metrics to HPA | APIService and metric freshness |
| ScaledObject | Defines scale target, min/max and triggers | Correct namespace and Deployment |
| TriggerAuthentication | References credentials for scalers | Least privilege, secret lifecycle |
| ClusterTriggerAuthentication | Cluster-wide reusable authentication | Security approval and tenancy |
| HPA | Calculates nonzero desired replicas | Ownership, conditions and policies |
| Event source | Prometheus, queue or external system | Reachability and data validity |
| GPU node provisioner | Adds schedulable GPU nodes | Cloud quota, SKU and node warmup |

A ScaledJob is for event-driven **Jobs**, not a substitute for continuously serving a stateful model endpoint.

## 4. Choosing a useful inference scaling signal

| Signal | Appropriate use | Caution |
|---|---|---|
| Queue backlog | Durable asynchronous inference queue | HTTP requests may not have a durable queue |
| Waiting requests | Admission saturation per endpoint | Pod churn and load-balancing skew |
| Queue age | User-impacting wait | Requires validated instrumentation |
| Accepted token demand | Token-weighted capacity | Prompt/output mix affects cost |
| Requests/s | Homogeneous short requests | Not a reliable proxy for long-context cost |
| TTFT/ITL | SLO feedback and alerting | Lagging/noisy; avoid as sole scaler |
| GPU utilization | Supporting capacity evidence | High utilization may be healthy |
| KV occupancy | Context/concurrency pressure | High usage alone not proof of overload |

**Prefer a validated, stable metric with known semantics and a measured relationship to safe throughput.** Prometheus series are not automatically available to KEDA; the Prometheus scaler queries an authorized endpoint.

## 5. Polling, activation, target and cooldown

- `pollingInterval`: how often KEDA evaluates event-source triggers; it does not eliminate HPA's independent sync period.
- `cooldownPeriod`: applies primarily to KEDA's transition back to zero, not as a universal HPA downscale delay.
- `minReplicaCount`: keep warm inference replicas; **0 is risky** for latency-sensitive synchronous inference without buffering and a supported activation path.
- `maxReplicaCount`: enforce GPU quota and cost ceiling.
- `activationThreshold`: determines whether a trigger is active for zero-to-one behavior, where supported.
- `threshold`: scaling target used for desired nonzero replicas; **not the same** as activation threshold.
- `advanced.horizontalPodAutoscalerConfig.behavior`: HPA scale-up/down stabilization and policies.
- `fallback`: scaler-specific protection when metrics fail; validate supported types, replica limits and semantics in the installed KEDA version.

## 6. Example ScaledObject — isolated staging only

This is an illustrative KEDA Prometheus scaler manifest, **not** a TrueFoundry resource. The Prometheus query and metric are hypothetical: map them to real, validated series before applying.

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: inference-queue-lab
  namespace: inference-lab
spec:
  scaleTargetRef:
    name: inference-queue-lab
  pollingInterval: 30
  cooldownPeriod: 300
  minReplicaCount: 2
  maxReplicaCount: 5
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleUp:
          stabilizationWindowSeconds: 0
          policies:
            - type: Pods
              value: 1
              periodSeconds: 60
        scaleDown:
          stabilizationWindowSeconds: 600
          policies:
            - type: Pods
              value: 1
              periodSeconds: 60
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-operated.monitoring.svc:9090
        metricName: inference_waiting_requests_lab
        query: sum(inference_waiting_requests_lab{namespace="inference-lab"})
        threshold: "4"
        activationThreshold: "0"
```

The server URL is a **placeholder** that must be replaced with the actual authorized Prometheus endpoint; `inference_waiting_requests_lab` is **not** a claimed vLLM metric. The query must return a valid single numeric value according to the installed scaler's expectations. For synchronous HTTP traffic, retain sufficient warm capacity because scale-from-zero can strand requests before a metrics signal exists.

## 7. Secure trigger authentication

Use workload identity or appropriately scoped secret references supported by the selected scaler. Store tokens in a secret manager/Kubernetes Secret only as required; do not embed credentials in ScaledObjects, query strings, dashboards or logs.

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: prometheus-auth-lab
  namespace: inference-lab
spec:
  secretTargetRef:
    - parameter: bearerToken
      name: prometheus-access-lab
      key: token
```

**This is schematic.** Authentication parameters differ by scaler/version. Validate exact keys in the installed KEDA documentation; a TriggerAuthentication must be referenced by the trigger's `authenticationRef` to take effect. Do not deploy this example without an approved secret and verified supported authentication mode.

## 8. How KEDA interacts with HPA and TrueFoundry

Before enabling KEDA, answer:

1. Does TrueFoundry own the Deployment's `spec.replicas` or native autoscaling settings?
2. Is an HPA already attached to the target? KEDA typically creates/manages an HPA for its ScaledObject.
3. Is the target Deployment allowed to be externally scaled?
4. Does GitOps reconcile ScaledObject and HPA desired state without drift?
5. Can GPU nodes provision before queued requests time out?
6. Are request draining and streaming completion protected during scale-down?

**Never attach two independent HPAs to the same scale target.** When in doubt, test on a lab-owned Deployment and record integration as pending vendor validation.

## 9. GPU-aware capacity and cold-start budget

```text
trigger observed → KEDA poll → HPA decision → Deployment scale
   → GPU Pod Pending → node provision (if needed)
   → image pull → model artifact fetch → CUDA/vLLM init
   → warmup → Ready EndpointSlice → successful inference
```

Measure time-to-usable-capacity, not time-to-Pod-created. A Pending GPU Pod cannot serve traffic. GPU extended resources, taints, affinity, quotas, instance availability and model storage bandwidth remain independent constraints.

A rough **illustrative** capacity calculation:

```text
required_replicas = ceil(peak_output_tokens_per_second
                         / sustainable_output_tokens_per_second_per_replica)
```

Then add explicit N-1 failure-domain reserve, rolling-update surge and cold-start headroom. Use measured TTFT/ITL at representative prompt/output lengths to establish sustainable throughput; do not size from GPU utilization alone.

## 10. Read-only inspection [SAFE-READ]

```bash
kubectl config current-context
kubectl get crd scaledobjects.keda.sh
kubectl -n <namespace> get scaledobjects.keda.sh
kubectl -n <namespace> describe scaledobject <name>
kubectl -n <namespace> get hpa,deployments,pods -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> get resourcequota,pdb
kubectl get nodes -o wide
kubectl describe node <gpu-node>
```

Only if installed and authorized:

```bash
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
```

If the CRD does not exist, report **KEDA not installed**; do not interpret it as a vLLM failure. Do not inspect Secret values during routine diagnosis.

## 11. Observability dashboard

| Layer | Signal | Why |
|---|---|---|
| Event source | Query result, timestamp, errors | Trigger validity |
| KEDA | ScaledObject Ready/Active/Fallback conditions | Controller health |
| HPA | Current/desired replicas and metric target | Scale decision |
| Deployment | Current/available/Ready replicas | Actual progress |
| Scheduler | Pending reason, GPU allocatable | Placement |
| Model runtime | Load/warmup duration, Ready endpoints | Usable capacity |
| Inference | TTFT/ITL, tokens/s, queue, KV/preemptions | SLO |
| Gateway | 429/5xx, timeouts, stream completion | Client impact |
| Cost | GPU-hours, quota, idle replicas | Guardrails |

Discover actual metric names and labels from the running versions. Keep labels low cardinality and exclude prompt text and user identifiers.

## 12. Controlled staging validation [LAB-MUTATE]

1. Obtain explicit approval, GPU budget, model revision, stop thresholds and rollback.
2. Validate Prometheus query freshness, aggregation, units and access.
3. Ensure no conflicting HPA or TrueFoundry autoscaler controls the same Deployment.
4. Deploy a lab-owned ScaledObject through the approved source of truth.
5. Generate bounded synthetic demand; capture trigger → HPA → Pod → GPU → Ready timeline.
6. Compare TTFT/ITL, queue, KV occupancy, GPU VRAM and errors against SLO.
7. Remove synthetic demand; observe stabilization, stream draining and controlled scale-down.
8. Simulate a metrics outage only against a disposable lab scaler/endpoint; verify failure behavior and restore.
9. Restore known-good configuration and verify actual model revision, Ready endpoints and SLO.

Do not force scale-to-zero for live synchronous inference without a verified request buffer and warmup contract.

## 13. Failure scenarios

| ID | Symptom | First failed transition | Evidence |
|---|---|---|---|
| K1 | KEDA CRD missing | Platform → operator | CRD inventory |
| K2 | ScaledObject not Ready | ScaledObject → scaler | Conditions |
| K3 | Wrong scaleTargetRef | ScaledObject → Deployment | Target UID/name |
| K4 | Prometheus unreachable | Scaler → event source | Network/HTTP |
| K5 | Trigger auth rejected | Identity → event source | 401/403 |
| K6 | Query returns multiple series | Query → scaler | Query shape |
| K7 | Query returns no samples | Scrape → scaler | Series freshness |
| K8 | Wrong metric units | Metric → threshold | Unit contract |
| K9 | HPA not created | KEDA → HPA | Operator events |
| K10 | Competing HPA controllers | Ownership → replicas | Reconciliation |
| K11 | Max replicas too low | HPA → capacity ceiling | Limits |
| K12 | Pods Pending GPU | Deployment → scheduler | Events |
| K13 | Cloud GPU quota exhausted | Provisioner → cloud | Quota errors |
| K14 | Model cold start too slow | Pod → Ready | Startup timeline |
| K15 | Queue remains high after scale | Ready capacity → demand | Throughput |
| K16 | Replica oscillation | Metrics → HPA behavior | Stabilization |
| K17 | Scale-down drops streams | Termination → client | Drain logs |
| K18 | Scale-to-zero breaks requests | Gateway → warm backend | Buffer/activation |
| K19 | Metrics outage causes wrong fallback | Scaler → policy | Fallback conditions |
| K20 | GPU spend increases without SLO gain | Scale → useful capacity | Cost and TTFT |

For each, record **LPHL, FFT, blast radius, safe mitigation, authoritative rollback and recovery validation**. Avoid shared-cluster fault injection.

## 14. Incident runbook

1. **Declare impact:** endpoint, TTFT/ITL, 429/5xx, queue, active/Ready replicas and cost.
2. **Trace control path:** event source → KEDA → HPA → Deployment → scheduler → GPU node → vLLM Ready endpoint.
3. **Correlate:** trigger freshness, authentication, ScaledObject conditions, HPA target, cloud GPU quota, model load and release changes.
4. **Contain:** freeze unsafe rollout, retain warm replicas, bound admission and retries with jitter.
5. **Recover:** repair the confirmed first failed transition or roll back ScaledObject/TrueFoundry release through the authoritative workflow.
6. **Validate:** actual loaded model, Ready EndpointSlices, queue drain, TTFT/ITL, stream completion, 429/5xx and GPU cost ceiling.
7. **Document:** LPHL/FFT, detection and recovery times, metric contract and follow-up owners.

## 15. Acceptance gates

| Gate | Required evidence |
|---|---|
| A1 | KEDA/TrueFoundry/HPA ownership verified |
| A2 | Trigger source, units, freshness and auth validated |
| A3 | Activation/threshold, min/max, polling/cooldown understood |
| A4 | GPU quota, cold-start and failure headroom evaluated |
| A5 | Scale-down stream draining and safety reviewed |
| A6 | Bounded staging exercise or marked Not run |
| A7 | K1–K20 classified with LPHL/FFT |
| A8 | Authoritative rollback and post-recovery SLO documented |

## 16. Evidence template

```text
Date/time / approval / owner:
TrueFoundry workspace, namespace, cluster:
Image digest, model revision, GPU SKU:
KEDA version/CRD, ScaledObject, HPA owner:
Event source, metric/query, units, sample freshness:
TriggerAuthentication mode (no secret values):
Activation, threshold, min/max, polling/cooldown:
Desired/current/Ready replicas, GPU Pending:
Node provision/model load/time to usable capacity:
TTFT/ITL, queue, KV, 429/5xx, stream completion:
LPHL / FFT and K1–K20 findings:
Rollback release, recovery, cost ceiling, cleanup:
```

**Hands-on lab:** [8.3 — KEDA Event-Driven Scaling and Custom Metrics Check](../labs/03-keda-event-driven-scaling-custom-metrics-check.md)  
**Next:** Chapter 8.4 — GPU Node Autoscaling and Cluster Capacity.

## References

- [KEDA Documentation](https://keda.sh/docs/)
- [KEDA Prometheus Scaler](https://keda.sh/docs/latest/scalers/prometheus/)
- [Kubernetes Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [vLLM Documentation](https://docs.vllm.ai/)
