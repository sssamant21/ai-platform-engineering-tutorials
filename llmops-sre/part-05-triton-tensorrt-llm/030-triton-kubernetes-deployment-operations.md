# Chapter 030 — Triton Kubernetes Deployment and Operations

**Part 05 — NVIDIA Triton and TensorRT-LLM | Chapter 030 of 100**

> **Publication:** Written tutorial published; live lab execution, technical/security review and operational acceptance pending. The manifests are illustrative staging examples, not production-approved deployments.

## Learning objectives

Design, deploy and operate Triton Inference Server on Kubernetes with GPU scheduling, pinned images/model repositories, readiness/startup/liveness probes, internal Services, TLS gateways, RBAC, NetworkPolicy, observability, resource governance, rollout/rollback, node failure handling, capacity planning and FinOps.

## 1. Production reference architecture

```text
Approved inference clients
        │
   TLS/Auth/Quota gateway
        │
  Kubernetes internal Service
        │
  Triton Deployment (1+ replicas)
        │
   ┌────┴─────────────────────┐
   │ HTTP/gRPC + scheduler     │
   │ model repo / backends     │
   │ CPU/GPU model instances   │
   └────┬─────────────────────┘
        │
 GPU device plugin + node pool
        │
  NVIDIA driver / CUDA / GPU
        │
 Prometheus + DCGM + OTel
        │
 SLO alerts + on-call + GitOps
```

Model artifacts come from an approved immutable registry, object store or mounted repository. Expose only authorized gateway endpoints, not direct Triton administrative or metrics interfaces.

## 2. Kubernetes prerequisites

- Supported Kubernetes and container runtime versions.
- NVIDIA GPU drivers, compatible Triton image/backend and CUDA stack.
- GPU device plugin exposing `nvidia.com/gpu` (or supported partitioned resource names).
- GPU nodes with verified topology, taints/labels and sufficient allocatable GPU/memory.
- Namespace quotas, RBAC, admission controls, Pod Security and approved egress.
- Model artifact provenance, license, immutable revision and workload identity.
- Prometheus/DCGM exporter integration, on-call, SLO and rollback owner.

A GPU device visible to the node does not guarantee the Pod can request it; check `kubectl describe node` allocatable resources and scheduling events.

## 3. Release manifest

```yaml
release:
  triton_image_digest: "sha256:PINNED"
  triton_version: "PINNED"
  backend_versions: ["PINNED"]
  model_repository_revision: "PINNED"
  model_checksums: ["PINNED"]
  gpu_sku: "APPROVED"
  driver_cuda_compatibility: "VALIDATED"
  gpu_request_per_pod: 1
  model_readiness_contract: "APPROVED_MODEL"
  rollback_release: "LAST_KNOWN_GOOD"
```

This is an evidence template, not a deployable manifest. Never substitute mutable `latest` tags for approved immutable releases.

## 4. Example staging namespace and workload

The following is a **non-production design example**. Replace placeholders and validate admission policy, Triton image entrypoint, model repository mount and startup flags for the pinned release. It intentionally omits an externally reachable Ingress.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: triton-staging
  labels:
    pod-security.kubernetes.io/enforce: restricted
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: triton-serving
  namespace: triton-staging
automountServiceAccountToken: false
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: triton
  namespace: triton-staging
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: triton
  template:
    metadata:
      labels:
        app: triton
    spec:
      serviceAccountName: triton-serving
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: triton
          image: nvcr.io/nvidia/tritonserver:PINNED_TAG@sha256:PINNED_DIGEST
          command: ["/opt/tritonserver/bin/tritonserver"]
          args: ["--model-repository=/models", "--strict-readiness=true"]
          ports:
            - {name: http, containerPort: 8000}
            - {name: grpc, containerPort: 8001}
            - {name: metrics, containerPort: 8002}
          resources:
            requests:
              cpu: "2"
              memory: "8Gi"
              nvidia.com/gpu: "1"
            limits:
              cpu: "4"
              memory: "16Gi"
              nvidia.com/gpu: "1"
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          startupProbe:
            httpGet: {path: /v2/health/ready, port: http}
            periodSeconds: 10
            failureThreshold: 60
          readinessProbe:
            httpGet: {path: /v2/health/ready, port: http}
            periodSeconds: 10
          livenessProbe:
            httpGet: {path: /v2/health/live, port: http}
            periodSeconds: 20
          volumeMounts:
            - {name: models, mountPath: /models, readOnly: true}
      volumes:
        - name: models
          persistentVolumeClaim:
            claimName: approved-model-repository
---
apiVersion: v1
kind: Service
metadata:
  name: triton-internal
  namespace: triton-staging
spec:
  type: ClusterIP
  selector:
    app: triton
  ports:
    - {name: http, port: 8000, targetPort: http}
    - {name: grpc, port: 8001, targetPort: grpc}
```

**Critical caveats:** This sample requires a pre-created approved PVC and compatible runtime filesystem permissions. The `restricted` security profile may conflict with vendor images/backends or GPU integration; test and obtain an approved exception rather than silently weakening policy. The digest/tag are placeholders, not deployable values. `strict-readiness` is version-sensitive and server readiness alone does not validate a specific model's inference contract. A `Recreate` strategy avoids GPU surge for the sample but causes downtime; production HA requires multiple replicas, capacity and a tested strategy.

## 5. GPU scheduling and resource requests

For a whole GPU allocation, request and limit the same `nvidia.com/gpu` quantity. CPU/memory requests influence scheduling; limits constrain runtime. Verify GPU device plugin, node labels, tolerations and GPU capacity.

```bash
kubectl get nodes -o wide
kubectl describe node APPROVED_GPU_NODE
kubectl get pods -n triton-staging -o wide
kubectl get events -n triton-staging --sort-by=.lastTimestamp
```

Do not assume fractional GPUs are available; MIG and time-slicing require explicit cluster configuration and may change isolation/performance properties.

## 6. Probe design and model readiness

- **Startup probe:** allow approved model download and backend initialization without premature restarts.
- **Readiness probe:** gate Service endpoints on server readiness; additionally validate target model/version and synthetic inference in deployment acceptance.
- **Liveness probe:** detect dead process without triggering restart loops for transient model-load pressure.

Set thresholds based on measured cold-start distributions. Avoid probes that trigger expensive inference on every Kubernetes health check.

## 7. Model repository and storage

Use read-only mounted artifacts or an approved artifact synchronization workflow. Pin model weights, config.pbtxt, backend and tokenizer/preprocessing versions. A PVC is not inherently immutable; enforce release-level digest checks and change control. Verify storage latency, egress/IAM, node locality and cache warmup.

## 8. Network and security

Triton HTTP/gRPC ports should be internal. Metrics should be accessible only to approved collectors. Model-control APIs may share inference listeners, so **port-level isolation alone does not secure administrative paths**. Use gateway authorization and/or disable/limit model-control operations as appropriate to the pinned mode.

Apply least-privilege ServiceAccounts, no unnecessary token mount, NetworkPolicy enforced by CNI, TLS gateway, audit logs and approved egress. NetworkPolicy does not by itself implement user authentication or TLS.

## 9. NetworkPolicy design

A default-deny policy is a strong starting point but must explicitly allow intended gateway traffic, monitoring and required DNS/artifact egress. Do not apply a broad deny policy to production without testing. Namespaces and Pod selectors must match the real cluster and CNI behavior.

## 10. Availability, disruption and GPU capacity

One replica is not HA. For multiple replicas:
- confirm at least two independent failure domains and enough GPU inventory;
- validate PodDisruptionBudget, topology spread and node drain behavior;
- account for model loading time and surge GPU capacity;
- ensure gateway routing avoids unready models;
- measure recovery after GPU/node loss and prevent retry storms.

A PDB does not prevent involuntary failures and cannot create missing GPU capacity.

## 11. Deployment rollout and rollback

```text
Approve pinned release
   → preflight GPU + storage + security
   → deploy to isolated staging
   → model readiness + synthetic inference
   → SLO/latency/VRAM/cost baseline
   → approved canary / routing
   → monitor complete responses
   → promote or rollback atomically
```

Rollback must restore the tested unit: Triton image, backend, model/config revision, gateway policy and relevant scheduler settings. Kubernetes rollout undo alone may not revert mutable external model repositories.

## 12. HPA, KEDA and scaling cautions

CPU-based HPA can be a poor proxy for GPU inference saturation. Use verified model queue, latency, request rate or supported external/custom metrics with admission and cooldown controls. Scaling requires physical GPU capacity, model warmup and budget. HPA cannot schedule Pods onto GPUs that do not exist.

## 13. Observability

Capture:
- server and model/version readiness;
- valid request rate, error taxonomy and complete response;
- queue, compute input/infer/output, p95/p99 client E2E;
- model instance execution, batch efficiency;
- GPU VRAM/utilization, XID/ECC, power and throttling where supported;
- Pod scheduling, restart, node/device-plugin health;
- metrics freshness, trace/log privacy and GPU-hours per valid completion.

Validate metric names and units from the pinned Triton build.

## 14. Operational read-only commands

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

For approved namespace/deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment TRITON_DEPLOYMENT
kubectl -n MODEL_NAMESPACE get pods -o wide
kubectl -n MODEL_NAMESPACE logs deployment/TRITON_DEPLOYMENT --since=15m --tail=100
kubectl -n MODEL_NAMESPACE get endpointslices
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Redact sensitive output and never dump Secrets.

## 15. Incident decision tree

```text
Triton on Kubernetes degraded?
 ├─ Pending → GPU resource, quota, taints, PVC
 ├─ ImagePullBackOff → registry, digest, auth
 ├─ CrashLoopBackOff → runtime, model, security context
 ├─ startup probe fails → cold start, storage, backend
 ├─ server ready, model unready → config/model load
 ├─ Service unreachable → selector, EndpointSlice, CNI
 ├─ HTTP/gRPC 4xx/5xx → gateway/schema/backend
 ├─ GPU OOM → model instances, workspace, overlap
 ├─ node/GPU failure → device plugin, failover capacity
 ├─ model load slow → storage/cache/egress
 ├─ metrics stale → exporter/NetworkPolicy
 ├─ auth/admin exposure → containment/change control
 ├─ rollout stuck → unavailable GPU surge
 └─ cost high → idle replicas, overprovisioning
```

## 16. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | GPU resource not advertised | allocatable |
| F02 | Pod Pending on taint/affinity | scheduler events |
| F03 | namespace GPU quota exceeded | ResourceQuota |
| F04 | image digest unavailable | pull events |
| F05 | registry authentication failure | pull events |
| F06 | driver/CUDA/backend mismatch | startup log |
| F07 | model PVC missing | mount events |
| F08 | model repository permission denied | loader |
| F09 | model artifact checksum drift | manifest |
| F10 | non-root filesystem incompatibility | security log |
| F11 | startup probe too aggressive | restarts |
| F12 | server ready, model unready | model endpoint |
| F13 | Service selector mismatch | EndpointSlice |
| F14 | NetworkPolicy blocks gateway | CNI |
| F15 | DNS/egress blocks artifact fetch | connectivity |
| F16 | metrics port exposed | exposure audit |
| F17 | model-control API exposed | auth audit |
| F18 | GPU memory exhausted | per-GPU VRAM |
| F19 | rolling update lacks surge GPU | Pending Pods |
| F20 | node drain violates capacity | PDB/events |
| F21 | GPU hardware/device-plugin fault | DCGM/events |
| F22 | stale metrics during outage | scrape |
| F23 | rollback leaves mutable model version | digest |
| F24 | GPU-hours exceed ceiling | FinOps |

## 17. Sixteen operational runbooks

1. Validate cluster/GPU device-plugin prerequisites.
2. Diagnose Pending GPU Pods and quotas.
3. Diagnose image pull and registry identity.
4. Diagnose driver/CUDA/backend mismatch.
5. Diagnose PVC and model repository access.
6. Recover security-context startup failures.
7. Tune startup/readiness/liveness probes.
8. Verify model/version readiness and fixtures.
9. Diagnose Service/EndpointSlice routing.
10. Recover DNS, CNI and NetworkPolicy.
11. Audit TLS, model-control and metrics exposure.
12. Diagnose GPU OOM and instance placement.
13. Recover GPU node/device-plugin failures.
14. Plan PDB, drain and GPU surge capacity.
15. Restore metrics/SLO and FinOps visibility.
16. Execute immutable rollback and RCA.

## 18. Eight tabletop game days

1. GPU device plugin stops advertising resources.
2. A new Triton image fails under restricted security context.
3. Model PVC becomes unavailable during cold start.
4. NetworkPolicy blocks the authorized gateway.
5. Rolling update needs an unavailable surge GPU.
6. GPU node fails while another replica is loading.
7. Model-control API is accidentally exposed.
8. Rollback restores Pod but not external model revision.

Default tabletop; live injection requires approved isolated staging.

## 19. Twenty acceptance gates

G01 pinned image/backend; G02 model provenance; G03 GPU resources; G04 node scheduling; G05 storage; G06 security context; G07 startup probe; G08 model readiness; G09 synthetic inference; G10 internal Service; G11 TLS/auth; G12 admin isolation; G13 NetworkPolicy; G14 GPU memory; G15 HA/failure domain; G16 rollout/rollback; G17 observability freshness; G18 SLO; G19 FinOps/on-call; G20 independent review.

See [companion lab](../labs/030-triton-kubernetes-deployment-operations-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 031 — TensorRT-LLM Architecture and Compilation.**
