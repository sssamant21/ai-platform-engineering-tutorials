# 4.2 — GPU Nodes and Kubernetes GPU Scheduling

**Part:** 04 — GPU Infrastructure  
**Audience:** TrueFoundry platform engineers, Kubernetes administrators, SREs and ML engineers  
**Safety:** All production investigation commands are **[SAFE-READ]**. The companion lab contains an optional **[LAB-MUTATE]** exercise restricted to a disposable, approved environment.

## 1. Objectives

Explain the path from GPU node registration through device-plugin resource advertising to Kubernetes scheduling and runtime allocation. Diagnose a Pending TrueFoundry-managed GPU workload without assuming that a GPU instance or healthy node implies schedulable GPU capacity.

## 2. GPU scheduling architecture

```text
GPU-equipped cloud/physical node
   ├── host driver + runtime integration
   ├── NVIDIA device plugin registers nvidia.com/gpu
   └── kubelet reports capacity and allocatable
                    │
TrueFoundry deployment intent → rendered Pod spec
   ├── resources.limits.nvidia.com/gpu
   ├── nodeSelector / required nodeAffinity
   ├── tolerations
   ├── topology and storage constraints
   └── CPU/memory/other scheduling requirements
                    │
             Kubernetes scheduler
                    │
          eligible nodes with free GPU
                    │
             Pod bound to node
                    │
        kubelet + device plugin allocation
                    │
      GPU devices exposed to container
                    │
         CUDA/framework/model readiness
```

**Different transitions require different evidence.** The scheduler places a Pod using reported extended-resource availability; it does not run CUDA or prove the model can initialize the GPU. A Pod can be bound and then fail at device allocation, container creation, CUDA initialization or model loading.

## 3. Prerequisites and ownership

- Kubernetes context and namespace known; confirm that `dev`, `staging` and `prod` may be namespaces within one cluster.
- Read-only access to nodes, Pods, events and (if permitted) device-plugin DaemonSets/logs.
- Identify the TrueFoundry deployment, owning Kubernetes controller and authoritative desired-state source.
- An optional GPU lab requires a disposable cluster/namespace with a working NVIDIA device plugin and an organization-approved container image.

**Ownership:** Cloud/node-pool team owns instance provisioning; GPU platform team owns driver/plugin/runtime integration; workload/platform team owns requested resources and placement constraints; model team owns framework and model initialization. Assign based on evidence, not symptoms.

## 4. How Kubernetes advertises GPU resources

The NVIDIA device plugin registers GPU devices with kubelet, which exposes resources such as `nvidia.com/gpu` on the Node's **capacity** and **allocatable** fields.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -L node-pool,gpu,gpu_type
kubectl describe node <gpu-node>
kubectl get node <gpu-node> -o jsonpath='{.status.capacity.nvidia\.com/gpu}{" capacity\n"}{.status.allocatable.nvidia\.com/gpu}{" allocatable\n"}'
```

**Capacity** is the advertised total. **Allocatable** is the portion exposed for scheduling, not a real-time free-GPU counter. To understand remaining scheduling capacity, examine resource allocations and existing GPU Pod requests on candidate nodes.

A blank `nvidia.com/gpu` field means Kubernetes is not advertising that resource; it does not establish whether the physical GPU is present. Check device-plugin registration and driver state in later chapters.

### GPU resource request semantics

GPU resources are extended resources and are normally integer counts. A typical workload requests one whole GPU:

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

For standard device-plugin GPU resources, specifying the GPU limit alone is supported; Kubernetes defaults the corresponding request to the same value. A GPU request without an equal limit is not the supported configuration pattern. GPU resource sharing and MIG can change advertised resource names and accounting, so do not assume a universal `nvidia.com/gpu` count across clusters.

**Avoid confusing these metrics:**
- `nvidia.com/gpu: 1` = scheduling/allocation of a device resource.
- `nvidia-smi` utilization = measured device activity.
- VRAM usage = memory allocated on a GPU.
- `kubectl top` CPU/memory = different resource metrics; it does not by itself expose GPU utilization.

## 5. GPU node eligibility: complete placement contract

A GPU Pod is schedulable only if a candidate node satisfies **all** relevant constraints:

1. Node is Ready and schedulable.
2. Node labels match `nodeSelector` and required affinity.
3. Taints are tolerated.
4. Requested GPU resources fit the node's remaining allocatable extended resources.
5. CPU/memory, pod count, topology and storage requirements also fit.
6. Other scheduling gates (for example admission/scheduling policies) do not block placement.

Example **Pod template fragment** for an environment-specific dedicated pool:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        node-pool: gpu-reserved
        gpu: "true"
        gpu_type: A10G
      tolerations:
        - key: dedicated
          operator: Equal
          value: gpu
          effect: NoSchedule
      containers:
        - name: model
          image: <approved-model-image@sha256:digest>
          resources:
            requests:
              cpu: "2"
              memory: 8Gi
            limits:
              cpu: "4"
              memory: 16Gi
              nvidia.com/gpu: 1
```

This is illustrative Kubernetes YAML, **not** a complete TrueFoundry deployment specification. The label names and taint are site-specific examples. Inspect actual nodes before applying any placement contract. A toleration permits a Pod to be considered for a tainted node; it does not select the node or create capacity.

## 6. Production-safe inspection workflow [SAFE-READ]

### 6.1 Verify workload identity and scheduler evidence

```bash
kubectl -n <namespace> get deployments,statefulsets,pods -o wide
kubectl -n <namespace> get pod <pod> -o yaml
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Inspect `PodScheduled` condition, `nodeName`, resource requests/limits for **all containers**, init containers, selectors, affinity, tolerations, topology spread and PVC bindings. Pod phase `Pending` alone is insufficient to conclude a scheduler problem.

### 6.2 Compare candidate nodes

```bash
kubectl get nodes --show-labels
kubectl describe node <candidate-gpu-node>
kubectl get node <candidate-gpu-node> -o jsonpath='{.spec.unschedulable}{" unschedulable\n"}{.spec.taints}{" taints\n"}'
kubectl -n <namespace> get pods --field-selector spec.nodeName=<candidate-gpu-node> -o wide
```

In `kubectl describe node`, inspect the **Allocated resources** section, not just total capacity. The GPU resource can be fully allocated even if `nvidia-smi` shows low current utilization.

### 6.3 Inspect device-plugin health where authorized

```bash
kubectl get daemonsets -A
kubectl get pods -A -o wide
kubectl -n <plugin-namespace> describe daemonset <nvidia-device-plugin-daemonset>
kubectl -n <plugin-namespace> logs <plugin-pod> --tail=200
```

Plugin names, namespaces and installation methods vary (standalone plugin versus GPU Operator). Discover actual names first. A missing resource advertisement is a different fault from insufficient free GPU capacity.

### 6.4 Record an exclusion ledger

| Candidate node | Ready / schedulable | Label match | Taints tolerated | GPU available | Other constraints | Eligible? |
|---|---|---|---|---|---|---|
| gpu-node-a | Yes | Yes | Yes | No | Fit | No — GPU allocated |
| gpu-node-b | Yes | No | Yes | Yes | Fit | No — selector mismatch |
| gpu-node-c | Yes | Yes | No | Yes | Fit | No — untolerated taint |

These are hypothetical examples. Build the actual ledger from current cluster evidence, not guessed node state.

## 7. Troubleshooting decision tree

```text
GPU Pod Pending?
  ├─ No → inspect container/runtime/framework/model readiness
  └─ Yes
      ├─ PodScheduled=True? → inspect kubelet/container creation, not scheduler
      └─ PodScheduled=False
          ├─ No GPU nodes? → cloud/node-pool provisioning
          ├─ GPU nodes, no nvidia.com/gpu? → driver/plugin registration
          ├─ Insufficient nvidia.com/gpu? → allocations and capacity
          ├─ node affinity/selector mismatch? → desired-state placement
          ├─ untolerated taint? → pool policy / Pod tolerations
          ├─ CPU/memory/pod limit? → broader resource fit
          └─ volume/topology constraints? → storage/topology owner
```

### Common failures

| Evidence | Meaning | Safe next action |
|---|---|---|
| `0/N nodes available: Insufficient nvidia.com/gpu` | Candidate nodes cannot satisfy GPU request | Review allocatable GPUs, assigned Pod requests, provisioning |
| `didn't match Pod's node affinity/selector` | Placement constraints exclude nodes | Compare rendered Pod rules with actual node labels |
| `had untolerated taint` | Tainted nodes excluded | Review dedicated-pool policy and approved tolerations |
| No `nvidia.com/gpu` on a GPU host | Kubelet not advertising GPU resource | Review device-plugin/driver integration |
| Pod scheduled, `ContainerCreating` | Scheduling succeeded; runtime setup pending/failing | Review Pod events, kubelet/runtime/device allocation |
| `No CUDA runtime is found` | Application GPU initialization failed | Prove device visibility and framework compatibility layer by layer |
| GPU node autoscaler active, Pod still Pending | Provisioning may be delayed or blocked | Check autoscaler/node registration and actual scheduling events |

**Important:** preemption is not a guaranteed GPU-capacity solution; it depends on priority, eligible victims and scheduling feasibility. Avoid deleting other tenants' Pods or relaxing production isolation rules to force scheduling.

## 8. TrueFoundry operational workflow

1. Record TrueFoundry deployment/workspace/environment identity.
2. Locate the corresponding Kubernetes controller and rendered Pod template.
3. Compare desired GPU count and placement intent to actual `nvidia.com/gpu`, labels and tolerations.
4. Determine whether the first failure is provisioning, registration, scheduling, runtime allocation or framework initialization.
5. Assign the current actionable owner and proposed change in the authoritative configuration source.
6. Validate both workload health and a successful inference request after any approved remediation.

**Avoid untracked hot patches:** TrueFoundry/GitOps reconciliation may revert direct Kubernetes edits.

## 9. Cloud provider considerations

- **EKS:** Verify GPU instance types, node-group or Karpenter provisioning, node labels/taints, compatible GPU AMI/driver strategy, device-plugin installation and capacity.
- **AKS:** Verify GPU VM SKU, node-pool labels/taints, supported GPU driver provisioning and plugin strategy for the AKS version.
- **GKE:** Verify GPU node-pool accelerator settings, driver installation mode, supported GPU resource advertisement and scheduling policy.

Cloud integrations vary by release and configuration. Do not treat a provider-specific label, driver installation mode or autoscaler feature as universal.

## 10. Change, rollback and recovery

**Production:** no speculative node label changes, taint removal, resource-limit edits or scale-out without an approved plan. Preserve incident evidence first. Correct the authoritative TrueFoundry/GitOps/IaC source; stage rollout, assess cost and tenant impact, and define a revert.

**Rollback acceptance:** previous placement policy restored, no unintended tenant movement, no orphaned GPU Pods and baseline inference availability maintained.

**Recovery acceptance:** eligible GPU node exists; resource advertised; Pod scheduled; GPU allocated; container can access device; framework detects CUDA; model Ready; inference succeeds; latency/errors within expected baseline.

## 11. Acceptance checklist

- [ ] Can distinguish GPU capacity, allocatable GPU and remaining schedulable GPU.
- [ ] Can explain GPU limit/request semantics and why GPU utilization does not equal free scheduling capacity.
- [ ] Can inspect rendered placement requirements and scheduler events.
- [ ] Can classify `Pending`, `ContainerCreating` and CUDA initialization failures separately.
- [ ] Can build a candidate-node exclusion ledger and identify the first failed transition.
- [ ] Can propose a safe owner-approved fix and end-to-end recovery checks.

**Hands-on:** [Lab 4.2 — GPU Nodes and Scheduling Check](../labs/02-gpu-nodes-kubernetes-scheduling-check.md)  
**Next:** Chapter 4.3 — NVIDIA Drivers, Device Plugin and GPU Operator.

## References

- [Kubernetes — Schedule GPUs](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [Kubernetes — Assign Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Kubernetes — Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Kubernetes — Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [NVIDIA — Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
