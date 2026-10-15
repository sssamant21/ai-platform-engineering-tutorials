# 6.5 — Model Loading, Warmup, Health Probes and Readiness

**Part:** 06 — Model Serving · **Chapter:** 5 of 7  
**Audience:** TrueFoundry platform engineers, MLOps, Kubernetes administrators and SREs  
**Safety:** Diagnostics are **[SAFE-READ]** unless specified. Probe edits, rollouts, replica changes, model downloads and fault injection are **[CHANGE-CONTROL]** in production and **[LAB-MUTATE]** only in an approved disposable environment.

## 1. Learning objectives

Trace a model-serving Pod from scheduling through artifact download, initialization, CPU/GPU allocation, warmup and readiness; choose startup, readiness and liveness probes; avoid premature traffic and destructive restart loops; measure cold-start latency; and recover a failed model rollout safely.

## 2. Model startup state machine

```text
Deployment reconciled
    ↓ Pod scheduled / image pulled
Container starts
    ↓ identity, storage, configuration
Artifact download / checksum validation
    ↓ deserialize model + tokenizer/config
Runtime initialization (CPU / CUDA / execution provider)
    ↓ weights and buffers allocated (host RAM / VRAM)
Warmup / golden smoke test
    ↓ application declares ready
Kubernetes readiness true
    ↓ Service EndpointSlice ready address
Gateway/Service sends inference requests
    ↓ runtime health + SLO monitoring
Drain → shutdown / restart / rollout
```

The state machine is conceptual; a runtime may load lazily or asynchronously. **Running**, **Started**, **Ready**, and **Available** are different conditions. A process may be alive but unable to serve a correct inference. An HTTP 200 from `/health` does not prove a model is loaded.

## 3. Cold-start budget

```text
T_ready = T_schedule + T_image_pull + T_container_start
        + T_model_fetch + T_integrity_check + T_deserialize
        + T_runtime_init + T_allocate + T_warmup + T_probe
```

Some stages overlap or are cached; this is a diagnostic decomposition, not a guaranteed additive wall-clock formula. Capture P50/P95 cold-start by model version, node type, cache state, artifact size and concurrency. Cold-start SLO must account for node provisioning and GPU availability, not only process initialization.

| Stage | Evidence | Common bottleneck |
|---|---|---|
| Scheduling | Pending duration, events | GPU scarcity, affinity, quota |
| Image pull | Container waiting reason | Registry, digest, bandwidth |
| Model fetch | Download timing, storage logs | IAM, private network, object size |
| Checksum | Manifest and verified hash | Corruption, partial cache |
| Deserialize | Loader logs, CPU/RAM | Model format, CPU bottleneck |
| Runtime init | Framework logs | CUDA/driver/provider mismatch |
| Allocation | Host/VRAM metrics | Memory fragmentation, OOM |
| Warmup | Synthetic golden request | Kernel compilation, first-use overhead |
| Readiness | Probe events, EndpointSlice | Premature/late readiness |

## 4. Startup, readiness and liveness probes

| Probe | Purpose | Failure behavior |
|---|---|---|
| Startup | Give slow initialization a bounded grace period | Repeated failure eventually restarts container; suppresses liveness/readiness until success |
| Readiness | Control whether Pod receives ordinary Service traffic | Failure removes Pod from ready endpoints; does not restart it |
| Liveness | Detect an unrecoverably unhealthy process | Repeated failure restarts container |

Do not set liveness to fail whenever an object store, identity provider or downstream dependency is briefly unavailable. That can trigger a restart storm, repeated model downloads and GPU reallocation. Keep liveness cheap and local; readiness should reflect ability to serve the intended model and essential dependencies.

**Readiness nuance:** A readiness probe returning success before model load/warmup is complete can cause 5xx, long tail latency or invalid results. A readiness probe that performs expensive full inference on every check can overload the runtime.

## 5. Illustrative Kubernetes Pod-template configuration

This is a **Kubernetes fragment, not a TrueFoundry API schema**:

```yaml
containers:
  - name: inference
    image: registry.example.com/ml/serving@sha256:<approved-digest>
    ports:
      - name: http
        containerPort: 8080
    resources:
      requests:
        cpu: "1000m"
        memory: 4Gi
      limits:
        memory: 8Gi
    startupProbe:
      httpGet:
        path: /startup
        port: http
      periodSeconds: 10
      timeoutSeconds: 3
      failureThreshold: 90
    readinessProbe:
      httpGet:
        path: /ready
        port: http
      periodSeconds: 5
      timeoutSeconds: 2
      failureThreshold: 3
    livenessProbe:
      httpGet:
        path: /health
        port: http
      periodSeconds: 15
      timeoutSeconds: 3
      failureThreshold: 3
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]
    terminationGracePeriodSeconds: 60
```

**Important correction for implementation:** `terminationGracePeriodSeconds` belongs to the **Pod spec**, not inside a container entry. Place it alongside `containers` under `spec.template.spec`. The `preStop` example is merely illustrative; a fixed sleep does not guarantee graceful draining and may not be appropriate for your runtime. Verify executable availability and use a measured shutdown protocol.

For GPU serving, include the appropriate integer `nvidia.com/gpu` allocation and validated driver/CUDA/runtime stack. Set startup budget from observed P99 initialization time plus headroom; excessive budgets can hide genuine failure.

## 6. Warmup strategies

- **Minimal warmup:** approved synthetic request after model load, before readiness.
- **Shape-aware warmup:** exercise common input dimensions, batch sizes and execution paths.
- **GPU kernel warmup:** account for first-use compilation and allocator initialization.
- **Cache warmup:** prefetch immutable model artifacts where policy permits.
- **Traffic shadowing:** only with explicit privacy, authorization and capacity review; do not send production data to an unapproved candidate.
- **Progressive traffic:** gradually expose a ready candidate after golden validation and SLO checks.

Warmup should be bounded, deterministic where possible and never leak real customer inputs. For stochastic models, validate schema, finite values and tolerated output ranges rather than expecting byte-identical responses.

## 7. Readiness contract for model-serving endpoints

A strong readiness implementation verifies:
1. Process HTTP server is responsive.
2. Expected **immutable model ID/version** is loaded.
3. Required runtime/execution provider is initialized.
4. Critical model resources are allocated and healthy.
5. The server is accepting work (not draining or overloaded beyond policy).

Readiness should **not** require every optional downstream service to be perfect; otherwise transient dependency issues can remove all endpoints simultaneously. Decide which dependencies are truly essential and monitor the others separately.

## 8. Graceful termination and draining

When a Pod terminates, Kubernetes updates endpoint conditions asynchronously and sends termination signals according to the Pod lifecycle. Existing connections and streaming requests may outlive the readiness transition. Design the serving process to stop admitting new work, complete or cancel in-flight requests within a bounded deadline, and release GPU memory and artifact locks.

A fixed `preStop` delay alone is not a complete drain strategy. Coordinate gateway connection draining, application SIGTERM handling, `terminationGracePeriodSeconds`, long-running streaming deadlines and rollout strategy. Test in nonproduction under realistic traffic.

## 9. Rolling deployment implications

Model-serving rollouts can temporarily double artifact download bandwidth, disk usage and GPU demand. `maxSurge: 1` may be unschedulable when all GPU slots are occupied; `maxUnavailable: 0` can stall indefinitely without headroom. A stalled Deployment progress deadline does **not** automatically trigger rollback.

For large models, plan warm capacity, staged downloads, compatible revisions and explicit promotion/rollback gates. A healthy old ReplicaSet is valuable until the new version proves ready and correct.

## 10. Diagnostic commands [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,replicasets,pods -o wide
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=180s
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> logs <pod> --tail=200
kubectl -n <namespace> logs <pod> --previous --tail=200
kubectl -n <namespace> get endpointslices -o wide
```

`rollout status` is observational but waits up to the timeout. Previous logs may not exist. For a multi-container Pod, specify `-c <container>` when needed. Use approved dashboards for model load, cache hit, host memory and GPU VRAM trends.

### Inspect readiness and termination state

```bash
kubectl -n <namespace> get pod <pod> -o jsonpath='{range .status.conditions[*]}{.type}{"="}{.status}{" "}{.reason}{"\n"}{end}'
kubectl -n <namespace> get pod <pod> -o jsonpath='{range .status.containerStatuses[*]}{.name}{" ready="}{.ready}{" restarts="}{.restartCount}{" lastReason="}{.lastState.terminated.reason}{"\n"}{end}'
```

Avoid dumping Secret values, full environment variables or sensitive model payloads.

## 11. Failure matrix

| Symptom | Evidence | First failed transition |
|---|---|---|
| Pod Pending | Scheduler events, quota, GPU allocatable | Scheduling → node |
| ImagePullBackOff | Image digest, pull identity | Registry → container |
| Model 403/404 | Object version, scoped IAM, sanitized logs | Model reference → storage |
| Checksum mismatch | Manifest, hash evidence | Download → integrity |
| Init container blocked | Init status and events | Initialization → main container |
| Framework import fails | Previous logs, runtime versions | Image → framework |
| CUDA initialization fails | Device allocation, driver/runtime | GPU device → framework |
| Host OOMKilled | Last termination, host memory | Process → cgroup |
| CUDA OOM | VRAM, batch/concurrency | Model → GPU allocation |
| Startup probe restarts slow model | Probe events and load timing | Initialization → startup gate |
| Running but not Ready | Model state, readiness response | Model → readiness |
| Ready before warmup | First-request latency/5xx | Readiness → traffic |
| Probe endpoint overloaded | Probe latency and runtime CPU | Probe → process |
| No ready EndpointSlices | Pod readiness, selector/ports | Pod → Service |
| Deployment stalled | ReplicaSets, surge capacity | Rollout → scheduling |
| SIGTERM kills active stream | Termination and gateway logs | Drain → in-flight request |

## 12. Incident recovery runbook

**A. Establish baseline:** record TrueFoundry deployment revision, image digest, model version, model size, node/GPU, first failure, request SLO and recent changes.

**B. Isolate:** scheduler → image → artifact → runtime → memory → warmup → probes → Ready EndpointSlice → gateway. Identify the **Lowest Proven Healthy Layer** and **First Failed Transition**.

**C. Contain:** stop promotion, preserve healthy old replicas, avoid indiscriminate restarts and protect shared model caches. Do not disable probes as a shortcut.

**D. Repair:** correct the confirmed resource, artifact, runtime or probe issue; adjust budgets only using measured load-time and SLO evidence.

**E. Recover:** use the authoritative TrueFoundry/GitOps workflow to restore a compatible **image + model + resources + probes** revision. Kubernetes-only rollout undo may be overwritten by reconciliation.

**F. Validate:** correct model version, golden inference, Ready EndpointSlices, stable error rate, P95/P99, restarts, CPU/GPU memory and stream completion over an approved observation window.

## 13. Acceptance gates

| Gate | Evidence |
|---|---|
| G1 | Startup state machine and version ownership documented |
| G2 | Cold-start stage timing and P95/P99 captured |
| G3 | Probe purposes and measured thresholds validated |
| G4 | Model warmup and golden request complete before readiness |
| G5 | Host RAM/GPU VRAM and cache capacity checked |
| G6 | Service endpoint continuity and draining tested |
| G7 | Stalled rollout and failure recovery classified |
| G8 | Authoritative rollback and post-recovery SLO documented |

## 14. Evidence template

```text
Date/time / TrueFoundry workspace / deployment:
Image digest / model immutable version:
Pod/node/GPU SKU and resources:
Scheduling/image pull/model fetch timings:
Checksum/load/runtime initialization/warmup:
Startup/readiness/liveness probe settings:
Ready conditions / EndpointSlice state:
Host memory / GPU VRAM / restart reason:
P95/P99, 5xx, first-request latency:
Termination/drain behavior:
Lowest Proven Healthy Layer:
First Failed Transition:
Approved fix and owner:
Known-good revision and rollback:
Post-recovery validation and cleanup:
```

**Hands-on lab:** [6.5 — Model Loading and Probe Reliability Check](../labs/05-model-loading-warmup-health-readiness-check.md)  
**Next:** Chapter 6.6 — Model-Serving Observability, Performance and Troubleshooting.

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Kubernetes Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
