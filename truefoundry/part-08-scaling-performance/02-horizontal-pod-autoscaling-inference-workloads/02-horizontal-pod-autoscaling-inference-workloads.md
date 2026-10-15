# 8.2 — Horizontal Pod Autoscaling for Inference Workloads

**Part:** 08 — Scaling & Performance · **Chapter:** 2 of 7  
**Audience:** TrueFoundry platform engineers, Kubernetes SREs, GPU inference operators  
**Safety:** **[SAFE-READ]** read-only diagnostics; **[LAB-MUTATE]** isolated approved staging; **[CHANGE-CONTROL]** production changes.

## 1. Learning objectives

Explain the Kubernetes Horizontal Pod Autoscaler (HPA) reconciliation loop, resource and custom metrics, desired-replica calculations, scaling behavior, GPU scheduling constraints, model-aware readiness and safe rollout/drain. Build a measurable inference scaling policy, identify metrics-adapter failures and recover from autoscaling incidents.

**Prerequisites:** Chapter 8.1, a pinned TrueFoundry/vLLM deployment, approved GPU quota and model-serving SLOs, and read access to Kubernetes metrics and node inventory. Do not run production load tests or mutate a TrueFoundry-managed Deployment outside its authoritative workflow.

## 2. HPA architecture and ownership

```text
Client → Gateway → Kubernetes Service → Ready inference Pods
                                      │
             vLLM queue, tokens/s, latency, KV utilization
                                      │
                              metrics exporter
                                      ▼
             Prometheus / metrics adapter (where installed)
                                      ▼
                       custom.metrics.k8s.io
                                      │
Metrics Server → metrics.k8s.io ───────┤
                                      ▼
                            HPA controller
                                      ▼
                         Deployment scale subresource
                                      ▼
                           ReplicaSet → Pending Pods
                                      ▼
                      Scheduler → GPU nodes/allocatable
                                      ▼
                     Model pull → CUDA init → warmup
                                      ▼
                      Readiness → serving capacity
```

TrueFoundry or GitOps may own replica and autoscaling settings. Confirm its documented integration and reconciliation behavior **before** creating an HPA: two independent writers to the replica count can fight each other. HPA does not provision GPU nodes; that requires a separate, supported node-scaling mechanism.

## 3. HPA control algorithm

For a single metric, Kubernetes conceptually computes:

```text
desiredReplicas = ceil(currentReplicas × currentMetricValue / desiredMetricValue)
```

The controller applies tolerance, readiness/missing-metric adjustments, behavior policies, stabilization and min/max limits. For multiple metrics, it generally selects the **largest** recommended replica count when metrics are valid; partial metric errors can prevent downscaling. Exact behavior depends on Kubernetes version and metric type.

**Example (illustrative):** 3 replicas, current average CPU utilization 80%, target 60% → ceil(3 × 80/60) = **4** replicas, before controller adjustments. CPU is a poor primary proxy for GPU decode workloads, so use a measured inference-specific signal where supported.

## 4. Metric types and when to use them

| HPA metric type | Example | Advantages | Limitations |
|---|---|---|---|
| Resource | CPU utilization | Built-in, well understood | GPU workload may be CPU-idle |
| ContainerResource | Sidecar-independent CPU | More targeted CPU scaling | Version/support dependent |
| Pods | Average queue depth per Pod | Reflects active demand | Needs custom metrics adapter |
| Object | Service or queue metric | Scales from one aggregate | Must match actual routing |
| External | Shared queue backlog | Event-driven demand | Adapter/security/freshness |

**Avoid scaling directly on GPU utilization alone.** A healthy GPU can be near 100% busy while maintaining SLOs; KV cache, queue age, TTFT and admitted tokens are more useful together. Validate that per-Pod metrics are scoped to the target Pods and that old Pods do not leave stale series.

## 5. CPU resource metrics example — lab-only

The following is a **generic Kubernetes HPA**, not a TrueFoundry API object. Use only where the authoritative platform permits HPA ownership.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: inference-cpu-lab
  namespace: inference-lab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: inference-cpu-lab
  minReplicas: 1
  maxReplicas: 3
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
```

**Important:** CPU utilization requires meaningful container CPU requests. This example is useful for CPU-bound inference/tokenization demonstrations, not a production GPU-vLLM scaling recommendation. Confirm the cluster API and autoscaling controller support the chosen behavior fields.

## 6. Inference-specific custom metric example — lab-only

Assume an adapter already exposes an **approved and validated** metric called `inference_waiting_requests` for each serving Pod:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: inference-queue-lab
  namespace: inference-lab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: inference-queue-lab
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Pods
      pods:
        metric:
          name: inference_waiting_requests
        target:
          type: AverageValue
          averageValue: "2"
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
```

The metric name is **hypothetical**: it will not work without a correctly configured adapter and real series mapping. A target queue average of 2 is an example, **not** an SLO-approved value. Queue metrics are sensitive to sampling, burst length, uneven routing and model cold-start time.

## 7. Scaling behavior and stability

- **minReplicas:** maintain warm capacity for long model load times, N-1 resilience and traffic bursts.
- **maxReplicas:** cap GPU quota, cloud cost and blast radius; check node-pool max size.
- **scaleUp:** allow controlled growth without overwhelming model storage, GPU supply or cluster.
- **scaleDown:** use longer stabilization and a gentle policy for streaming workloads.
- **missing/stale metrics:** diagnose adapter and target labels before interpreting desired replicas.
- **zero replicas:** native HPA is not a general scale-to-zero solution; investigate KEDA or supported platform features in Chapter 8.3.

Behavior settings alone cannot guarantee safe request draining. Coordinate gateway termination, readiness removal, in-flight stream completion, grace period and PDB.

## 8. GPU scheduling and readiness constraints

A serving Pod may request:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

GPU extended resources are integer and typically require equal requests and limits (when only limits are supplied, Kubernetes derives requests). Confirm installed device-plugin behavior, MIG or time-slicing policy if used, and that GPU allocatable exists. GPU availability is not inferred from VRAM graphs alone.

Before raising maxReplicas, verify:
1. GPU allocatable after subtracting other workloads.
2. Node affinity, taints/tolerations and topology-spread feasibility.
3. Namespace ResourceQuota and cloud GPU quotas.
4. Model storage throughput and concurrent cold starts.
5. Model-aware startup/readiness probes and warmup.
6. N-1 reserve, rollouts and surge GPU requirements.

**HPA desired=6, ready=2** is a capacity incident, not successful scaling. Track desired, current, available and **Ready endpoints** separately.

## 9. Read-only operational commands [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get hpa
kubectl -n <namespace> describe hpa <hpa-name>
kubectl -n <namespace> get deployment <deployment> -o wide
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> get resourcequota,pdb
kubectl get nodes -o wide
kubectl describe node <gpu-node>
```

Where permitted and installed:

```bash
kubectl top pods -n <namespace>
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
kubectl get --raw /apis/external.metrics.k8s.io/v1beta1
```

A missing API endpoint can indicate no adapter installed; it does not mean vLLM is broken. Avoid dumping secret-bearing Pod environment variables into incident tickets.

## 10. Metrics validation and control-loop verification

| Layer | Required evidence |
|---|---|
| vLLM | Metric exists for pinned release, units and labels verified |
| Scrape | Target healthy, recent samples, bounded cardinality |
| Adapter | Correct mapping, namespace, selector and API registration |
| HPA | Current metric, target, conditions and scaling events |
| Deployment | Desired vs available replicas |
| Scheduler | GPU allocatable and Pod events |
| Runtime | Model load, warmup and readiness duration |
| Service | Ready EndpointSlices and actual request distribution |
| Client | TTFT/ITL, 429/5xx and stream completion |

Prometheus metrics are **not automatically available to HPA**. A metrics adapter must expose them through a supported Kubernetes metrics API. Do not deploy adapters or cluster-wide APIService changes during a routine app-level lab without platform approval.

## 11. Bounded nonproduction test [LAB-MUTATE]

1. Record immutable model/image, TrueFoundry owner, existing HPA configuration, GPU quota and rollback.
2. Establish baseline TTFT/ITL, queue, GPU VRAM, desired/Ready replicas and model cold-start time.
3. Use a synthetic fixed prompt/output distribution and approved arrival-rate ceiling.
4. Increase load in small steps; observe metrics scrape → adapter → HPA → Deployment → GPU scheduling → Ready endpoints.
5. Stop for sustained queue growth, error/SLO breach, GPU Pending Pods, VRAM pressure or cost ceiling.
6. Reduce synthetic demand; observe controlled scale-down and completed streaming requests.
7. Restore known-good autoscaling policy via authoritative workflow; verify correct model revision, SLO and resource cleanup.

No production stress tests. If spare GPU quota or an approved adapter is unavailable, perform a tabletop and mark live gates Not run.

## 12. Failure scenarios and diagnosis

| ID | Symptom | First failed transition | First check |
|---|---|---|---|
| H1 | HPA target missing | HPA → Deployment | scaleTargetRef |
| H2 | Unknown CPU utilization | Metrics Server → HPA | CPU requests and metrics |
| H3 | Custom metric unavailable | Adapter → HPA | APIService and adapter |
| H4 | Metric exists but HPA sees zero | Labels → adapter | Selector/namespace |
| H5 | Stale queue metric | Scrape → adapter | Timestamp |
| H6 | Desired replicas stuck at max | HPA policy → ceiling | maxReplicas/quota |
| H7 | Desired rises, Pods Pending | Deployment → scheduler | GPU events |
| H8 | New GPU nodes absent | Provisioner → cloud | Quota/capacity |
| H9 | New Pods never Ready | Engine → readiness | Model load/CUDA |
| H10 | Scaling too late | Cold start → demand | Time to usable capacity |
| H11 | Oscillating replicas | Metrics → HPA behavior | Stabilization |
| H12 | Replica traffic skew | Gateway → Pod | Per-Pod queue |
| H13 | Scale-down drops streams | Pod termination → client | Drain/grace |
| H14 | HPA fights TrueFoundry | Source of truth → replicas | Reconciliation history |
| H15 | 503 during rollout | Service → endpoints | EndpointSlice |
| H16 | TTFT breach despite replicas | Queue → prefill | Prompt mix |
| H17 | ITL breach despite replicas | Decode → GPU | KV/VRAM |
| H18 | PDB blocks maintenance | Eviction → policy | PDB allowed disruptions |
| H19 | Unexpected GPU spend | Replicas → cost | Desired/idle utilization |
| H20 | Wrong model on new replicas | Artifact → runtime | Immutable revisions |

For each, record **Lowest Proven Healthy Layer (LPHL)**, **First Failed Transition (FFT)**, scope, evidence, owner, safe mitigation, rollback and acceptance check.

## 13. Production incident runbook

1. **Declare:** endpoint/model, onset, TTFT/ITL, 429/5xx, stream failures, desired/Ready replicas and queue.
2. **Inspect:** HPA conditions and metrics, adapter freshness, TrueFoundry reconciliation, GPU Pending events and node provisioner.
3. **Localize:** source metrics → adapter → HPA → Deployment → scheduler → GPU node → model readiness → Service.
4. **Contain:** freeze unsafe rollout, preserve known-good warm replicas, apply approved admission limits and client backoff/jitter.
5. **Recover:** fix first confirmed transition; if needed roll back authoritative HPA/model configuration.
6. **Validate:** correct model revision, desired/Ready convergence, queue drain, TTFT/ITL, full stream completion, errors and cost ceiling.
7. **Review:** LPHL/FFT, SLO impact, control-loop delay, capacity shortfall, owner and prevention.

Do not delete HPA or scale a production deployment manually when TrueFoundry/GitOps owns it.

## 14. Acceptance gates

| Gate | Required evidence |
|---|---|
| A1 | HPA/TrueFoundry ownership and safe scope verified |
| A2 | HPA formula, resource and custom metric types explained |
| A3 | Valid metric source/adapter/API verified or gap documented |
| A4 | GPU quota, node capacity, cold-start and Ready endpoints evaluated |
| A5 | Scale-up/down policies, stream drain and failure reserve reviewed |
| A6 | Bounded synthetic scaling test or explicitly marked tabletop |
| A7 | H1–H20 classified by LPHL/FFT |
| A8 | Authoritative rollback and post-recovery SLO documented |

## 15. Evidence template

```text
Date/time, workspace, cluster and namespace:
Approval, TrueFoundry/GitOps owner, rollback release:
Model/image digest, GPU SKU, per-Pod GPU count:
HPA API version, target, min/max, behavior:
Metric name/type/unit/labels/adapter/freshness:
Desired/current/Ready replicas and EndpointSlices:
GPU allocatable, quota, Pending events, cold-start:
Queue, TTFT/ITL, throughput, KV, 429/5xx:
Stream drain and scale-down observations:
Lowest Proven Healthy Layer / First Failed Transition:
H1–H20 classifications and A1–A8 results:
```

**Hands-on lab:** [8.2 — Horizontal Pod Autoscaling Check](../labs/02-horizontal-pod-autoscaling-inference-workloads-check.md)  
**Next:** Chapter 8.3 — KEDA Event-Driven Scaling and Custom Metrics.

## References

- [Kubernetes Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Kubernetes API Autoscaling v2](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#horizontalpodautoscaler-v2-autoscaling)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [vLLM Documentation](https://docs.vllm.ai/)
