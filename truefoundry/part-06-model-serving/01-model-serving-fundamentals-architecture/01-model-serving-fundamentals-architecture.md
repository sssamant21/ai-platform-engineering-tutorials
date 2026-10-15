# 6.1 — Model-Serving Fundamentals and Architecture

**Part:** 06 — Model Serving · **Chapter:** 1 of 7  
**Audience:** TrueFoundry platform engineers, MLOps engineers, Kubernetes SREs and inference service owners  
**Safety:** Production commands are **[SAFE-READ]** unless explicitly marked **[CHANGE-CONTROL]**. Use approved, non-sensitive sample models and nonproduction namespaces for experiments.

## 1. Learning objectives

Explain model serving as an online inference system; trace the TrueFoundry-to-Kubernetes control and data paths; distinguish CPU and GPU execution, model storage/loading and request handling; identify readiness, throughput, latency and reliability boundaries; and diagnose a failed inference request without assuming the platform is the cause.

## 2. What model serving means

**Training** produces model artifacts. **Serving** loads a model into a runtime and exposes an interface that transforms an input into a prediction or generated output. The serving lifecycle includes artifact provenance, model download, process initialization, runtime allocation, endpoint readiness, inference, telemetry and rollback.

A healthy Kubernetes Pod does **not** prove a model is loaded, warmed up, able to allocate accelerator memory or producing valid responses.

### Online versus batch inference

| Dimension | Online serving | Batch inference |
|---|---|---|
| Trigger | Request/response | Scheduled or queued dataset |
| Primary SLO | Availability, tail latency, correctness | Completion time, throughput, correctness |
| Resource pattern | Burst and concurrency sensitive | Sustained utilization |
| Recovery | Endpoint failover/retry and revision rollback | Checkpoint, retry and idempotent reprocessing |
| Typical failure | Timeout, 5xx, model not ready | Failed partition, backlog, partial output |

This chapter focuses on **online model serving**. Streaming token generation and vLLM internals are covered later in Part 07.

## 3. Reference architecture and failure domains

```text
                 MANAGEMENT / CONTROL PATH
Git / CI / model registry / approved artifact metadata
                  │
                  ▼
TrueFoundry workspace + deployment specification
                  │  deployment / identity / resources / endpoint
                  ▼
Cluster integration / Kubernetes API
                  │
                  ▼
Deployment or supported workload controller → Pod(s)
                  │
                  ├── CPU or GPU scheduling
                  ├── Secret / ServiceAccount / workload identity
                  ├── Model artifact fetch / mounted storage
                  ├── Runtime initialization / model load / warmup
                  └── Readiness and serving endpoint
                  
                    RUNTIME / DATA PATH
Authorized client → DNS/TLS/auth → route/Service → Ready Pod
                  → request validation → preprocessing
                  → model execution → postprocessing → response
                  → metrics/traces/logs
```

**Control-plane failure versus runtime failure:** an existing healthy serving Pod may continue processing traffic during a transient TrueFoundry control-plane outage, depending on cluster/network dependencies. Conversely, a successful deployment reconciliation does not imply inference success. Verify the actual deployed workload and platform version.

## 4. Model-serving building blocks

| Component | Responsibility | Failure evidence |
|---|---|---|
| Model registry/artifact store | Versioned weights/config/tokenizer | Missing version, checksum mismatch, permission denied |
| Container image | Runtime, libraries, application server | ImagePullBackOff, incompatible runtime |
| TrueFoundry deployment | Desired deployment configuration and ownership | Reconciliation/validation status |
| Kubernetes scheduler | CPU/GPU placement | Pending, insufficient resource/taint mismatch |
| GPU driver/device plugin | Accelerator discovery and allocation | Missing `nvidia.com/gpu`, driver/device errors |
| Model loader | Download, deserialize, allocate weights | Download errors, OOM, initialization timeout |
| Inference runtime | Execute forward pass or generation | Runtime exceptions, latency, OOM |
| Service/route | Expose authorized inference API | No endpoints, TLS/DNS/route mismatch |
| Telemetry | Measure reliability, saturation, correctness | Missing traces/metrics, unexplained regression |

Model artifacts may live in S3, Azure Blob Storage, GCS, PVCs or an approved registry. Avoid downloading untrusted models or arbitrary code during load; verify provenance and follow runtime-specific safe-loading controls.

## 5. CPU and GPU serving

**CPU inference** suits smaller models, lower throughput, portable deployments and some optimized quantized runtimes. **GPU inference** suits compute-intensive models and larger throughput needs but adds driver, CUDA/runtime, accelerator-memory, scheduling and cost dependencies.

| Topic | CPU | GPU |
|---|---|---|
| Scheduling | CPU/memory requests | CPU/memory plus extended GPU resource |
| Runtime | CPU framework libraries | Compatible CUDA/driver/framework stack |
| Memory | Host RAM | Host RAM plus accelerator VRAM |
| Scaling | Replica/CPU capacity | GPU availability, model footprint, utilization |
| Bottlenecks | CPU contention, threads, memory | VRAM, compute, transfer, batching, GPU contention |

A Pod requesting `nvidia.com/gpu` needs compatible nodes and device-plugin registration. Do not assume `nvidia-smi` working on a node proves a container can access CUDA; inspect runtime and device allocation.

## 6. Model artifact lifecycle

1. **Register:** record model ID, version, source, license and validation results.
2. **Verify:** check checksum/signature, compatible framework/runtime and tokenizer/config files.
3. **Authorize:** grant only the model version/prefix and storage operations needed.
4. **Fetch:** download or mount model into controlled cache/storage.
5. **Load:** allocate host/GPU memory and initialize execution engine.
6. **Warm:** run approved representative input when required.
7. **Ready:** publish readiness only after critical model initialization and dependencies succeed.
8. **Serve:** validate inputs, enforce size/concurrency/timeouts and return bounded responses.
9. **Observe:** track errors, tail latency, queue time, throughput, resource utilization and model correctness signals.
10. **Retire:** drain traffic, revoke obsolete credentials and retain rollback artifacts per policy.

**Version contract:** pin image digest and model artifact version independently. A container rollback may still load the latest model if the model reference is mutable; avoid mutable model aliases in production release evidence.

## 7. Request lifecycle and latency budget

```text
Client DNS/TLS/auth
    → Gateway/Service queue
    → Pod ingress and validation
    → Preprocessing/tokenization
    → Runtime queue/batching
    → Model execution
    → Postprocessing/serialization
    → Network response
```

End-to-end latency is the sum of these stages, but asynchronous queues, overlapping work and streaming responses complicate simple attribution. Measure time to first byte/token separately from total request time for streaming systems.

**Key metrics:**

- Request rate, success/error rate, HTTP 4xx versus 5xx and timeout rate.
- P50/P95/P99 end-to-end latency, runtime execution latency and queue wait.
- Concurrency, queue depth, saturation, retries and rejected requests.
- Model load time, readiness duration, cache hit/miss and artifact download time.
- CPU/memory and, where applicable, GPU utilization, VRAM, OOM and device errors.
- Model-level validation metrics and drift indicators appropriate to the use case.

Do not expose sensitive prompts, model inputs, outputs or credentials in logs by default.

## 8. Kubernetes serving resources and readiness

**Illustrative Pod-template fragment** (not a TrueFoundry SDK schema):

```yaml
containers:
  - name: inference
    image: registry.example.com/ml/inference@sha256:<approved-digest>
    ports:
      - name: http
        containerPort: 8080
    resources:
      requests:
        cpu: "500m"
        memory: 2Gi
      limits:
        memory: 4Gi
    startupProbe:
      httpGet:
        path: /startup
        port: http
      periodSeconds: 10
      failureThreshold: 60
    readinessProbe:
      httpGet:
        path: /ready
        port: http
      periodSeconds: 5
      timeoutSeconds: 2
    livenessProbe:
      httpGet:
        path: /health
        port: http
      periodSeconds: 15
      timeoutSeconds: 3
```

This assumes the application implements the paths. Configure a realistic startup budget based on model load; do not make liveness dependent on slow remote model storage. Readiness should remain false until the model is genuinely able to serve requests. GPU workloads also need a verified GPU resource request/limit, appropriate scheduling and runtime compatibility.

## 9. TrueFoundry integration workflow

1. Confirm installed TrueFoundry version, workspace, cluster and authorized namespace.
2. Identify supported model-serving deployment workflow and inference runtime.
3. Record image digest, model version/URI, artifact checksum and approved model-access identity.
4. Configure CPU/GPU, memory, placement, storage/cache, environment variables and secret references.
5. Configure health probes, replica strategy, endpoint exposure and authentication.
6. Deploy through the supported platform workflow; inspect rendered Kubernetes resources.
7. Verify model load, warmup, readiness, endpoint and an authorized sample prediction.
8. Baseline request latency, errors, queueing, resource use and cost.
9. Record authoritative rollback for both image and model version.

Do not invent TrueFoundry CLI commands or assume a platform feature exists across versions; follow the installed version's documentation and inspect actual resources.

## 10. Production-safe investigation [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> logs <pod> --tail=150
kubectl -n <namespace> logs <pod> --previous --tail=150
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> top pod <pod> --containers
```

Metrics Server may not be installed, and previous logs may be unavailable. Redact model URIs, input data and authentication information before sharing evidence.

### GPU-specific discovery

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu
kubectl -n <namespace> describe pod <gpu-pod>
```

A node can advertise a GPU resource yet fail CUDA initialization because of device-plugin, runtime, driver or image compatibility. `nvidia-smi` inside a container is useful only when the image contains the tool and device access is configured.

## 11. Failure classification matrix

| Symptom | First likely failed transition | Evidence |
|---|---|---|
| Deployment absent | TrueFoundry desired state → Kubernetes | Platform reconciliation status |
| Pod Pending | Scheduler → node allocation | Events, GPU/CPU requests, taints |
| ImagePullBackOff | Registry → node | Pull errors, digest, identity |
| Model download 403/404 | Identity/artifact reference → storage | Sanitized loader logs, cloud audit |
| Checksum/model mismatch | Artifact → loader | Version and checksum metadata |
| CrashLoopBackOff during load | Model loader → process | Previous logs, exit reason |
| CUDA device detection failure | GPU allocation/runtime → framework | Driver/runtime evidence |
| GPU OOM | Model allocation/inference → VRAM | Runtime logs and VRAM metrics |
| Container OOMKilled | Process → cgroup memory limit | Last termination, host memory |
| Pod Running, not Ready | Model load/warmup → readiness | Probe events, load state |
| Endpoint 503 | Ready Pods → route | EndpointSlice, gateway logs |
| High P99, low GPU utilization | Queue/preprocess/network → execution | Stage-level latency metrics |
| High GPU utilization and queue growth | Execution capacity → admission | Queue depth, utilization |
| Prediction invalid despite 200 | Model version/schema → application contract | Golden test comparison |
| New release regresses | Revision/model → SLO | Before/after request telemetry |

**Never** resolve a failed inference by disabling readiness, bypassing identity or increasing GPU memory allocations blindly.

## 12. Incident recovery runbook

**A. Capture impact:** affected model/version, endpoint, traffic, request IDs, first failure time, TrueFoundry revision, error rate and P95/P99.

**B. Trace the path:** desired deployment → scheduler → image → model storage → load → readiness → route → runtime execution → response. Identify the **Lowest Proven Healthy Layer** and **First Failed Transition**.

**C. Select smallest approved fix:** repair access or artifact reference, resolve placement, restore compatible runtime, adjust measured capacity or roll back the authoritative image **and** model versions. Avoid manual Pod patches that reconciliation will overwrite.

**D. Validate:** correct model version, successful golden request, Ready endpoints, normal authentication, stable error/latency, memory/GPU health and no new restarts.

**E. Record:** incident timeline, contributing factors, corrective and preventive actions, owner, monitoring gap and recovery duration.

## 13. Acceptance gates

| Gate | Required evidence |
|---|---|
| G1 | TrueFoundry workspace, cluster and deployment identified |
| G2 | Image digest and model version pinned and traceable |
| G3 | Model storage access uses least privilege |
| G4 | CPU/GPU and memory resources scheduled correctly |
| G5 | Model loading and warmup complete before readiness |
| G6 | Authorized inference request passes schema/golden check |
| G7 | Latency, error, queue and resource baselines captured |
| G8 | Recovery and compatible rollback documented |

## 14. Evidence template

```text
Date/time / workspace / cluster / namespace:
Deployment and TrueFoundry revision:
Git commit / image digest:
Model ID / immutable version / checksum:
Artifact storage and identity (no secrets):
CPU/GPU requests and node allocation:
Runtime/framework/driver compatibility:
Load and warmup time:
Probe state / Ready endpoints:
Inference request ID and expected schema:
Error rate / P95/P99 / queue time:
CPU, memory, GPU, VRAM metrics:
Lowest Proven Healthy Layer:
First Failed Transition:
Owner / approved remediation:
Image+model rollback and post-recovery validation:
Lab status and cleanup:
```

**Hands-on lab:** [6.1 — Model-Serving Architecture and Request-Path Check](../labs/01-model-serving-fundamentals-architecture-check.md)  
**Next:** Chapter 6.2 — Model Artifacts, Registries and Storage Integration.

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
