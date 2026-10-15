# 6.7 — Production Model-Serving Deployment and Recovery Capstone

**Part:** 06 — Model Serving · **Chapter:** 7 of 7 · **Type:** Production-oriented capstone  
**Audience:** TrueFoundry platform engineers, MLOps engineers, Kubernetes/GPU SREs, security and incident response  
**Safety:** [SAFE-READ] = observational. [LAB-MUTATE] = approved disposable nonproduction changes. [CHANGE-CONTROL] = production modification requiring owner authorization and rollback plan.

## 1. Mission and outcomes

Deliver a production-like inference service through the supported TrueFoundry workflow, prove its artifact integrity and runtime compatibility, validate its authenticated endpoint and SLO, simulate recoverable failures in isolation, and restore the last-known-good image **and** model version. By completion, produce a deployment design record, immutable release manifest, observability baseline, incident evidence, recovery log and acceptance report.

**Prerequisites:** Chapters 6.1–6.6; approved workspace/namespace; access to a vetted synthetic or non-sensitive model; Kubernetes read access; authorized model storage and inference endpoint; available CPU capacity (GPU optional and only with approval).

**Never use real patient/customer data, production fault injection, untrusted model code or privileged diagnostics to complete this capstone.**

## 2. Reference architecture

```text
Approved source / CI ─→ image scan + digest ─→ container registry
         │
         └──────────→ model validation + manifest + checksum
                                         │
                                   immutable artifact
                                  S3 / Blob / GCS / registry
                                         │
                                  workload identity
                                         ▼
TrueFoundry workspace / deployment / authoritative release
                    │ reconcile
                    ▼
Kubernetes Deployment → ReplicaSet → scheduled Pod(s)
      │                  │
      │                  ├─ CPU or GPU + compatible runtime
      │                  ├─ artifact fetch / integrity check / cache
      │                  ├─ model deserialize / allocate / warmup
      │                  └─ startup → readiness → liveness
      ▼
Service / Ready EndpointSlice → Gateway / TLS / Auth / quota
                                         ▲
                               authorized synthetic client
                                         │
                              request → queue → execution
                                         │
                                response / stream
                                         │
               metrics + traces + sanitized logs + GPU telemetry
                                         ▼
                               SLO / alert / incident
```

TrueFoundry platform API/SDK and managed endpoint features vary by installed version. **Do not invent CLI syntax**: use the supported installed-version UI/SDK/GitOps process and inspect the rendered Kubernetes resources. A Kubernetes-only hotfix may be overwritten by reconciliation. Record the authoritative source of truth and release owner.

## 3. Capstone service requirements

| Category | Minimum requirement |
|---|---|
| Model | Approved model ID, immutable version, manifest, verified checksums |
| Image | Approved immutable image digest, vulnerability/license review |
| Identity | Separate scoped image pull, model read and client inference identities |
| Runtime | Validated CPU or GPU/framework compatibility, model schema |
| Capacity | Measured CPU/RAM; GPU/VRAM and quota if applicable |
| Health | Startup budget, model-aware readiness, cheap local liveness |
| Network | Internal or explicitly approved external TLS/auth endpoint |
| Reliability | Bounded admission, timeout, retries, cancellation, graceful drain |
| Telemetry | Request errors, P95/P99, queue, model load, Ready endpoints, memory/GPU |
| Recovery | Known-good compatible image + model + config revision and owner |

**RPO/RTO:** For a stateless inference API, request loss/retry semantics and service restoration are central; traditional data RPO may be **not applicable** to the serving Pod. Any stateful audit, feature store or artifact registry dependency has its own RPO/RTO. Set an explicit capstone recovery-time objective based on environment and cold-start measurement.

## 4. Release record and change gates

Record:

```yaml
release:
  truefoundry_workspace: "<approved-workspace>"
  cluster: "<approved-cluster>"
  namespace: "<approved-namespace>"
  service: "<inference-service>"
  image_digest: "sha256:<verified-digest>"
  model_id: "<model-id>"
  model_version: "<immutable-version>"
  model_manifest_sha256: "<verified-manifest-hash>"
  runtime: "<framework-and-version>"
  execution_target: "cpu-or-approved-gpu"
  source_commit: "<git-commit>"
  rollout_owner: "<owner>"
  rollback_release: "<known-good-release-id>"
```

This is an **evidence template**, not a TrueFoundry deployment schema. Validate image provenance, artifact integrity, least-privilege identity, network policy, model schema, rollout capacity, baseline SLO and rollback availability before any promotion.

## 5. Workload design (illustrative Kubernetes resource)

The YAML below demonstrates the **rendered Kubernetes intent**, not a TrueFoundry API definition. The image digest and probe paths are placeholders; do not apply it as-is.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: model-serving-example
  namespace: <approved-namespace>
spec:
  replicas: 2
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: model-serving-example
  template:
    metadata:
      labels:
        app: model-serving-example
    spec:
      serviceAccountName: <scoped-workload-service-account>
      terminationGracePeriodSeconds: 60
      containers:
        - name: inference
          image: registry.example.com/inference@sha256:<verified-digest>
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
            httpGet: {path: /startup, port: http}
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 60
          readinessProbe:
            httpGet: {path: /ready, port: http}
            periodSeconds: 5
            timeoutSeconds: 2
          livenessProbe:
            httpGet: {path: /health, port: http}
            periodSeconds: 15
            timeoutSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: model-serving-example
  namespace: <approved-namespace>
spec:
  selector:
    app: model-serving-example
  ports:
    - name: http
      port: 80
      targetPort: http
```

For GPU inference, validate and add the appropriate integer `nvidia.com/gpu` request/limit, compatible node placement and CUDA runtime. `replicas: 2` and `maxSurge: 1` require capacity for an extra warm replica; on scarce GPU pools, explicitly plan surge/unavailability and test service continuity. Add namespace policies, image pull identity, secret references, topology spread and shutdown hooks according to the real environment. Never hardcode credentials or mutable model aliases.

## 6. Preflight and observability [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,replicasets,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get resourcequota,limitrange,networkpolicies
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> logs <pod> --tail=150
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=180s
```

Use `kubectl -n <namespace> get gateways,httproutes` only if Gateway API CRDs exist. Use approved metrics/dashboards for GPU utilization, VRAM, queue depth and model load time. Do not dump Secrets, entire Pod environment or sensitive model input/output.

## 7. Staged execution

### Phase A — Architecture and preflight

1. Identify installed TrueFoundry version, workspace, cluster, namespace, source-of-truth and owner.
2. Approve immutable image/model references and artifact checksum.
3. Verify workload identity, storage networking and model permissions without printing secrets.
4. Confirm CPU/GPU capacity, Pod placement, node quotas, model cache and storage headroom.
5. Capture baseline request/error/latency, warmup and alert state.

**Gate A:** all artifacts and identities are traceable; capacity and rollback are documented.

### Phase B — Release A (known-good)

1. Deploy a vetted small model via the installed TrueFoundry workflow in a disposable nonproduction workspace.
2. Observe image pull, model download, checksum verification, runtime initialization and warmup.
3. Verify startup probe completes before readiness, and Ready EndpointSlice appears.
4. Run a documented **authorized synthetic golden inference request** through normal TLS/auth routing.
5. Record model version, schema/correctness, P95/P99, queue depth, CPU/RAM and GPU/VRAM if used.

**Gate B:** model A serves correct requests and meets the approved nonproduction baseline.

### Phase C — Candidate B rollout

1. Select a **compatible** model or configuration change with a new immutable release record.
2. Check surge capacity and simultaneous artifact downloads.
3. Roll out via TrueFoundry/GitOps, monitor old/new replicas, model versions and Ready endpoints.
4. Compare golden requests, errors, P95/P99, queue and resource use.
5. Halt promotion if the new revision violates an acceptance gate.

**Gate C:** new revision demonstrates correctness, capacity and endpoint continuity; otherwise initiate rollback.

### Phase D — Nonproduction failure exercises

Choose **two or three** safe exercises in disposable workloads: (a) an intentionally incorrect readiness path on a lab-only revision, (b) an incorrect model artifact reference to a **synthetic missing object**, (c) a mismatched Service selector on a **disposable Service**, or (d) tabletop-only GPU OOM or TLS failures. Capture first failed transition. **Never** disrupt shared routes, nodes, GPU device plugins, Secrets, PVCs or production services.

### Phase E — Authoritative recovery

1. Declare rollback trigger and incident owner.
2. Restore **image digest + model version + runtime/resources + probes + endpoint policy** using the authoritative TrueFoundry/GitOps source.
3. Verify the old model actually loaded (not a mutable alias or stale cache).
4. Check Ready endpoints, golden inference, request SLO, queue stability and no repeated OOM/restarts.
5. Record recovery time and compare to the agreed objective.

### Phase F — Cleanup and handoff

Remove only lab-owned resources through the approved workflow; retain required evidence and known-good artifacts. Confirm no unexpected Service, volume, artifact or cloud-cost residue. Hand off findings, unresolved gates, review owners and corrective actions.

## 8. Incident simulations

| ID | Inject/tabletop symptom | First failed transition | Recovery direction |
|---|---|---|---|
| C1 | ImagePullBackOff | Registry → image | Verify digest and pull identity |
| C2 | Pod Pending | Scheduler → node | Check quota, affinity and CPU/GPU capacity |
| C3 | Model storage 403 | Workload identity → artifact | Restore scoped permissions |
| C4 | Missing model version | Reference → object | Restore approved immutable version |
| C5 | Checksum mismatch | Download → integrity | Reject corrupt artifact |
| C6 | Runtime/CUDA mismatch | Device → framework | Restore compatible image/runtime |
| C7 | Host OOMKilled | Process → RAM limit | Investigate footprint/worker count |
| C8 | GPU VRAM OOM | Model/batch → VRAM | Bound concurrency/model footprint |
| C9 | Startup probe restarts | Model load → startup budget | Measure cold-start and tune |
| C10 | Premature readiness | Warmup → Service traffic | Fix model-aware readiness |
| C11 | Service has no endpoints | Selector/readiness → EndpointSlice | Correct disposable selector |
| C12 | TLS/auth failure | Client → gateway | Restore certificate/policy |
| C13 | 429/503 and queue growth | Admission → execution | Bound load and retries |
| C14 | P99 regression with low GPU busy | Preprocess/network → runtime | Trace stages |
| C15 | Wrong prediction on HTTP 200 | Model/schema → output | Golden test and model rollback |
| C16 | Reconciler reverts manual patch | Kubernetes change → desired state | Fix authoritative configuration |
| C17 | Rollback reloads new model | Mutable model alias → runtime | Pin model artifact version |
| C18 | Active streams fail during rollout | Drain → client completion | Fix cancellation/drain protocol |

For each record **Lowest Proven Healthy Layer**, **First Failed Transition**, evidence, affected SLO, smallest approved fix, rollback trigger and recovery validation.

## 9. Production runbook

**Declare:** model ID/version, endpoint, severity, affected tenants, request volume, error/429/5xx, P95/P99, stream completion, first failure and recent deployment.

**Localize:** desired state → scheduling → image → artifact identity/storage → runtime/GPU → warmup/readiness → Service/gateway/auth → queue/execution → response. Do not assume a 503 means a model bug.

**Contain:** freeze promotion, preserve healthy replicas, limit retry amplification and use approved traffic controls.

**Recover:** choose narrow confirmed repair or authoritative rollback. Do not bypass authentication, disable probes, restart shared GPU nodes or delete shared caches to hide symptoms.

**Validate:** immutable loaded model, golden correctness, healthy endpoint, stable SLO and resources for an agreed observation window.

**Communicate:** incident commander owns status cadence; deployment owner executes authorized change; security/cloud owners approve identity or network changes; SRE validates service health; document timeline and follow-ups.

## 10. Acceptance rubric (100 points)

| Area | Points |
|---|---:|
| Design, immutable provenance and artifact integrity | 15 |
| Identity, networking and security | 15 |
| CPU/GPU scheduling, memory and startup capacity | 15 |
| Health probes, readiness, golden correctness | 15 |
| Endpoint SLO, telemetry and alerts | 15 |
| Controlled rollout, fault isolation and rollback | 20 |
| Cleanup, evidence and handoff | 5 |
| **Total** | **100** |

**Pass:** at least **85/100** plus **all mandatory safety gates**: approved nonproduction scope, no sensitive inputs/credentials, verified model correctness, secure authorized endpoint and a documented recoverable known-good revision. Mark any unexecuted test **Not run**; do not award execution credit for a tabletop.

## 11. Evidence package

```text
Workspace/cluster/namespace, approval and owner:
Installed TrueFoundry version and source-of-truth:
Image digest, model ID/version, manifest/checksum:
Identity, artifact storage and network policy:
CPU/GPU requests, host RAM/VRAM and capacity:
Model fetch/load/warmup and probe timings:
Gateway/auth, Service/EndpointSlice, golden result:
RPS, error ratio, P95/P99, queue, TTFT/stream:
Release A/B revision, promotion/rollback timestamps:
Incident C1–C18 observations:
Lowest Proven Healthy Layer / First Failed Transition:
Recovery time versus objective:
Post-recovery SLO and model version:
Cleanup, open risks and reviewer sign-off:
```

**Hands-on capstone lab:** [6.7 — Production Model-Serving Deployment and Recovery Acceptance](../labs/07-production-model-serving-deployment-recovery-capstone-check.md)

**Next:** Part 07 — vLLM and LLM Serving (verify detailed curriculum titles in the Part 07 plan before publishing).

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [Kubernetes EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Prometheus Histograms](https://prometheus.io/docs/practices/histograms/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
