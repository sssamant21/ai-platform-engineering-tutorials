# Chapter 094 — Deploy Kubernetes GPU Workloads

**Part 12 — Enterprise Operations and Final Capstones | AI Infrastructure Engineering & SRE**

**Status:** Documentation published. Kubernetes/GPU staging deployment, performance measurements, security validation and independent acceptance **PENDING**.

## 1. Mission

Deploy, schedule, observe and safely operate GPU workloads on Kubernetes for **inference and distributed training**. Cover GPU extended resources, node placement, topology, tenant isolation, device plugins, job lifecycle, rollout safety, checkpoints, queues, observability, and incident recovery.

**Prerequisites:** Chapter 092 secure GPU cluster, Chapter 093 driver/CUDA/NCCL validation, approved staging cluster, GPU Operator/device plugin, container registry, workload identity and GPU quota.

### Outcomes

1. Validate GPU allocatable resources and version-pinned runtime images.
2. Deploy a least-privilege single-GPU smoke Pod.
3. Schedule inference Deployments and training Jobs with explicit resource requirements.
4. Apply affinity, taints, topology spread and PriorityClass policy safely.
5. Integrate gang/queue admission for multi-GPU training.
6. Test scaling, checkpoint restart, rollback, and end-to-end SLOs.
7. Produce evidence-backed acceptance rather than relying on healthy Pod status alone.

## 2. Workload architecture

\`\`\`text
GitOps / CI security gates
          |
      Kubernetes API
          |
 RBAC + Pod Security + quota + queue admission
          |
      kube-scheduler
       /         \
Serving pool    Training pool
GPU nodes       topology-aware GPU nodes
  |                  |
Inference pods   Jobs / distributed ranks
  |                  |
Router/HPA       Checkpoint/object store
  \                  /
  DCGM + OTel + SLO + FinOps
\`\`\`

Inference optimizes latency, throughput, warm capacity and controlled rollouts. Training optimizes GPU topology, collective communication, throughput and restartable progress. Their scheduling priorities, eviction behavior and SLOs should not be conflated.

## 3. Preflight

\`\`\`bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'
kubectl get resourcequota -A
kubectl get priorityclass
kubectl get pods -A | grep -Ei 'gpu-operator|device-plugin|dcgm' || true
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

Confirm node SKU, HBM, CUDA/driver compatibility, storage throughput, GPU resource naming, quotas, tenancy policy, available physical GPU stock and staging cost limits. A node reporting \`Ready\` is not proof that GPU allocation or CUDA works.

## 4. GPU extended resource semantics

NVIDIA device plugins typically advertise \`nvidia.com/gpu\` as a Kubernetes extended resource. GPU resource requests are integer counts for whole-device allocation in the common configuration; fractional scheduling requires an explicitly supported partitioning/sharing strategy such as MIG or time-slicing, with different isolation guarantees.

For standard GPU limits:

\`\`\`yaml
resources:
  limits:
    nvidia.com/gpu: 1
\`\`\`

Kubernetes treats the limit as the GPU request for this extended resource when only the limit is specified. If both request and limit are specified, they must be equal for extended GPU resources. GPU memory capacity and physical topology are **not** encoded by this integer; validate SKU, MIG profile, HBM and node placement separately.

## 5. Safe single-GPU smoke workload

This **template is not deployable until the image placeholder and node policy are resolved**.

\`\`\`yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-smoke
  namespace: gpu-validation
  labels:
    app: gpu-smoke
spec:
  restartPolicy: Never
  terminationGracePeriodSeconds: 30
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: cuda
      image: REPLACE_WITH_APPROVED_CUDA_IMAGE_DIGEST
      command: ["sh", "-c", "nvidia-smi && sleep 10"]
      resources:
        limits:
          nvidia.com/gpu: 1
          memory: "2Gi"
          cpu: "1"
        requests:
          memory: "512Mi"
          cpu: "250m"
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
\`\`\`

Ensure the approved image can run as non-root and includes \`nvidia-smi\`. Device visibility is not sufficient for CUDA correctness: execute Chapter 093 kernel tests separately.

**Staging-only deployment sequence:**

\`\`\`bash
kubectl create namespace gpu-validation
kubectl apply --dry-run=server -f gpu-smoke.yaml
kubectl apply -f gpu-smoke.yaml
kubectl get pod gpu-smoke -n gpu-validation -o wide
kubectl logs -n gpu-validation gpu-smoke
kubectl describe pod -n gpu-validation gpu-smoke
\`\`\`

The namespace command is a mutation; use only in an approved disposable staging environment. Existing namespaces, quotas, policy and image signatures must be reviewed before apply. Record actual outputs rather than treating example commands as executed.

## 6. GPU placement and topology

Use node labels for verified hardware class; do not trust unprotected tenant-editable labels for security boundaries. Example:

\`\`\`yaml
nodeSelector:
  accelerator.example.com/class: approved-training
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu-training"
    effect: "NoSchedule"
\`\`\`

A toleration **allows** scheduling onto a tainted node; it does not force placement. Pair with node affinity/selector and admission policies. For trusted node isolation, use labels protected by appropriate node authorizer/admission controls.

**Topology considerations:**
- multi-GPU jobs should fit within compatible GPU/NVLink/NVSwitch/NIC domains;
- distribute serving replicas across nodes/AZs where compatible GPU capacity exists;
- avoid overfragmenting scarce GPUs with excessive anti-affinity;
- account for NUMA/CPU, memory, hugepages, RDMA and local storage locality;
- do not assume cross-node RDMA performance without NCCL validation.

## 7. Inference Deployment pattern

\`\`\`yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-staging
  namespace: gpu-validation
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  selector:
    matchLabels:
      app: inference-staging
  template:
    metadata:
      labels:
        app: inference-staging
    spec:
      terminationGracePeriodSeconds: 90
      containers:
        - name: server
          image: REPLACE_WITH_APPROVED_GPU_SERVER_IMAGE_DIGEST
          ports:
            - containerPort: 8080
          resources:
            limits:
              nvidia.com/gpu: 1
              cpu: "4"
              memory: "16Gi"
            requests:
              cpu: "2"
              memory: "8Gi"
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          startupProbe:
            httpGet:
              path: /ready
              port: 8080
            failureThreshold: 60
            periodSeconds: 5
\`\`\`

**Important:** \`maxSurge: 0\` avoids allocating extra GPUs during rollout, but \`maxUnavailable: 1\` reduces ready capacity temporarily. This example is **not automatically zero-downtime** and is unsuitable if a single unavailable replica violates the SLO. If headroom exists, consider \`maxSurge: 1\` and \`maxUnavailable: 0\`, but first verify spare compatible GPU capacity. The server image must implement \`/ready\`, and model quality, load time, traffic draining and authorization remain application-specific.

## 8. Training Job pattern

\`\`\`yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: gpu-training-staging
  namespace: gpu-validation
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: trainer
          image: REPLACE_WITH_APPROVED_TRAINING_IMAGE_DIGEST
          command: ["python", "train.py"]
          resources:
            limits:
              nvidia.com/gpu: 1
              cpu: "8"
              memory: "32Gi"
            requests:
              cpu: "4"
              memory: "16Gi"
\`\`\`

This is a **single-Pod Job**. Distributed multi-node training needs an approved job launcher/operator or indexed Jobs with correct rank/world-size discovery, topology, queue admission, NCCL networking, synchronized failure behavior and checkpoint semantics. A plain Job with parallelism alone does not guarantee gang scheduling.

## 9. Queue admission and training priority

Use Kueue or another validated scheduler/admission controller for ClusterQueues, LocalQueues, ResourceFlavors, quotas, borrowing and job admission. Confirm the installed CRD/API version before applying manifests. Protected inference capacity must not be borrowed by untrusted batch jobs without explicit policy. Restrict PriorityClass use and preemption permissions.

### Diagnostic sequence

\`\`\`bash
kubectl get jobs -A
kubectl get pods -A --field-selector=status.phase=Pending
kubectl describe pod -n gpu-validation <PENDING_POD_NAME>
kubectl get events -n gpu-validation --sort-by=.lastTimestamp
kubectl api-resources | grep -Ei 'kueue|workload|clusterqueue|localqueue' || true
\`\`\`

Do not assume Kueue is installed because the cluster has GPUs. Interpret scheduling events alongside quota, taints, resource fragmentation, topology and cloud supply.

## 10. Serving autoscaling and GPU cold starts

GPU serving HPA/KEDA requires meaningful metrics (requests, queue age, tokens/sec, p99-related indicators) and a functioning metric adapter. GPU utilization alone may not predict latency or HBM saturation. Node provisioning + image pull + model download + KV-cache warmup can exceed the reaction window; maintain protected warm replicas.

Scale-down should honor graceful shutdown, draining and active request completion. Test overload admission/backpressure before allowing aggressive autoscaling.

## 11. Storage and checkpoint lifecycle

Training Jobs should use versioned datasets and durable checkpoint storage, with a completion marker and hashes for weights, optimizer, RNG and sampler state. Validate write consistency, restore integrity, storage bandwidth and data residency. Avoid treating a writable PVC as automatically durable across AZ or region failures.

For inference, prefetch or cache model artifacts only after verifying integrity and access controls; avoid exposing proprietary weights through node-local shared caches.

## 12. Observability and operations

| Signal | Workload relevance | Operational response |
|---|---|---|
| Pod Pending reason | serving and training | quota/taint/stock diagnosis |
| GPU allocatable and assigned | placement | fragmentation/health |
| GPU Xid/ECC and memory | hardware | isolate/recover |
| CUDA/NCCL errors | training | topology/fabric/driver triage |
| Checkpoint age/last step | training | recovery/RPO |
| TTFT/TPOT/p99/errors | serving | rollback/scale/backpressure |
| Queue wait and admitted jobs | training | fairness/capacity |
| Billed GPU-hours and successful work | FinOps | unit-cost analysis |

Alert on user-visible SLOs and loss of monitoring itself. Correlate GPU metrics with Pod UID, namespace, node, model version and tenant while respecting privacy.

## 13. Safe rollout and rollback

1. Validate YAML, image digest, policy and security context.
2. Confirm compatible GPU headroom and rollout strategy.
3. Canary one staging inference replica or training Job.
4. Validate CUDA, model quality, p99, checkpoint and telemetry.
5. Promote through GitOps with a change record.
6. Watch scheduling, GPU health and end-to-end SLO.
7. Roll back to the last known-good image/config if gates fail.
8. Verify no orphaned training Jobs, leaked GPU allocations or stale model traffic.

A Kubernetes rollout rollback cannot reverse incompatible dataset/checkpoint changes or driver upgrades; define separate recovery for those dependencies.

## 14. Six production runbooks

**R1 — GPU Pod Pending:** inspect events, quota, extended resource allocatable, node affinity, taints, topology and physical stock; escalate if capacity absent.

**R2 — Container sees no GPU:** verify device plugin, runtime injection, resource limits, operator/driver and node health; compare with known-good smoke test.

**R3 — Inference rollout degrades p99:** halt rollout, check warmup, readiness, GPU HBM, spare headroom and model revision; rollback and verify traffic.

**R4 — Training Job repeatedly fails:** inspect OOM, CUDA/NCCL, node placement, checkpoint integrity and data access; resume only from committed checkpoint.

**R5 — Queue starvation/preemption:** inspect Kueue admission, quota borrowing, priority restrictions and tenant fairness; protect P0 serving.

**R6 — GPU hardware incident:** capture Xid/ECC, node/GPU UUID, affected Pods, checkpoints and replacement plan; cordon/drain only under approved procedure.

## 15. Sixteen hands-on exercises

1. Verify approved staging context and GPU inventory.
2. Review namespace security and quota.
3. Validate GPU device plugin and allocatable resources.
4. Build a pinned, least-privilege CUDA smoke Pod.
5. Run single-GPU device and kernel validation.
6. Apply and test GPU node placement policies.
7. Inspect topology and multi-GPU scheduling feasibility.
8. Deploy inference with safe readiness/startup probes.
9. Load-test inference and verify p99/TTFT/TPOT.
10. Run a single-Pod training Job.
11. Design a distributed job and queue admission policy.
12. Validate checkpoint durability and resume.
13. Simulate Pending Pod and quota failure in staging.
14. Test rollout/canary and rollback.
15. Measure GPU-hours, queue wait and utilization.
16. Assemble evidence and independent acceptance review.

## 16. Twelve troubleshooting scenarios

| Symptom | Investigation |
|---|---|
| GPU Pod Pending | quota, taint, allocatable, SKU |
| Node Ready but no GPUs | driver/device plugin |
| Smoke Pod sees no device | runtime/toolkit/injection |
| Wrong GPU type assigned | node selector/admission |
| MIG slice mismatch | resource name/profile |
| Inference readiness never passes | model warmup/endpoint |
| Rollout deadlock | insufficient spare GPU capacity |
| Inference p99 regresses | HBM, batching, warmup |
| Training fails on rank startup | launcher/NCCL/topology |
| Checkpoint cannot restore | partial state/KMS |
| Queue starvation | admission/priority/borrowing |
| GPU cost spikes | idle pods, failed retries, overprovision |

## 17. Acceptance gate (100 points)

| Domain | Points |
|---|---:|
| GPU discovery, security and resource governance | 20 |
| Correct scheduling and topology placement | 20 |
| Inference deployment, SLO and rollback | 20 |
| Training Job, queue and checkpoint recovery | 20 |
| Observability, cost and incident runbooks | 10 |
| Reproducible evidence and independent review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 and no critical tenant isolation breach, unsupported GPU placement, failed inference SLO, corrupt training checkpoint, unverified rollback or missing independent approval. **Current: documentation published; runtime acceptance not evaluated.**

## 18. References

- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [Kubernetes Assign Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kubernetes Pod Topology Spread](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Kueue](https://kueue.sigs.k8s.io/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)

**Next:** Chapter 095 — Implement Storage and Network Performance Tests.
