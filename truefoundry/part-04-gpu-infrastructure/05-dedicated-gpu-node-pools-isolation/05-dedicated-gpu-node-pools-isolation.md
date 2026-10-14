# 4.5 — Dedicated GPU Node Pools and Resource Isolation

**Part:** 04 — GPU Infrastructure  
**Audience:** TrueFoundry platform engineers, Kubernetes administrators, SREs and FinOps teams  
**Operational safety:** Read-only commands are **[SAFE-READ]**; node-pool, scheduling-policy and autoscaler modifications require **[CHANGE-CONTROL]**.

## 1. Learning outcomes

Design dedicated GPU node pools for predictable scheduling, tenancy and cost; distinguish labels, affinity, taints, tolerations and GPU resource accounting; assess GPU sharing options and their isolation limits; and troubleshoot workload placement without changing production nodes impulsively.

## 2. Reference architecture

```text
TrueFoundry workspace / environment / workload intent
                     │
             Kubernetes Pod template
             ├─ GPU resource limits
             ├─ nodeSelector / nodeAffinity
             ├─ tolerations
             ├─ priority / topology
             └─ CPU + memory requirements
                     │
               Kubernetes scheduler
                     │
      ┌──────────────┼──────────────┐
      │              │              │
 GPU A10G pool   GPU H100 pool   CPU-only pool
 dedicated taint dedicated taint no GPU resources
 GPU device      GPU device
 plugin          plugin
      │              │
   model Pods     model Pods
      └──────────────┴──────────────┘
              GPU observability
```

Labels such as `node-pool=gpu-reserved`, `gpu=true`, and `gpu_type=A10G` are **site-specific examples**, not built-in Kubernetes or TrueFoundry defaults. Confirm the labels and pool provisioning strategy in the actual cluster.

## 3. Why separate GPU pools?

Dedicated pools enable hardware selection, operational isolation, clearer quota/cost attribution, controlled driver rollout, and more predictable scheduling. They do **not** automatically guarantee security or performance isolation: tenant separation also depends on namespace RBAC, admission policy, workload identity, runtime hardening, network policy and GPU-sharing mode.

Choose pool boundaries based on GPU model and memory, driver/runtime compatibility, workload sensitivity, tenant boundaries, failure domains, utilization and expected demand. Avoid one pool per small workload when it creates unacceptable fragmentation or idle cost.

## 4. Placement primitives and their differences

| Mechanism | Function | Important limitation |
|---|---|---|
| Node label | Describes node characteristics | Does not itself reserve GPUs |
| `nodeSelector` | Requires matching labels | Incorrect labels can strand Pods |
| Required node affinity | Expresses mandatory placement logic | Too restrictive can cause Pending |
| Preferred node affinity | Biases scheduler selection | Not a hard isolation boundary |
| Taint `NoSchedule` | Repels Pods lacking toleration | Existing Pods generally unaffected |
| Toleration | Permits placement on tainted node | Does not attract Pod to that node |
| `nvidia.com/gpu` limit | Requests a device-plugin GPU resource | GPU utilization is not schedulable capacity |
| Namespace ResourceQuota | Limits aggregate resource consumption | Does not choose a GPU pool |
| PriorityClass / preemption | Expresses scheduling importance | Can disrupt tenants; does not create hardware |
| Topology spread | Distributes workloads across domains | Strict rules may block scheduling |

**Two-sided contract:** for a dedicated pool, use both a restrictive taint and matching workload placement rules. A toleration alone can allow a GPU workload to land on an unintended eligible pool; a selector alone does not prevent unrelated tolerated workloads from using the GPU pool.

## 5. Example: A10G dedicated GPU pool

### 5.1 Node policy (illustrative)

```text
Labels:
  node-pool=gpu-reserved
  gpu=true
  gpu_type=A10G

Taint:
  dedicated=gpu:NoSchedule

GPU resource:
  nvidia.com/gpu: <observed allocatable integer>
```

Node labels and taints must be configured through the authoritative node-pool/IaC mechanism; manual changes can be overwritten on replacement or scaling. GPU type labels should not be used as the sole hardware attestation mechanism.

### 5.2 Workload Pod template fragment

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
        - name: inference
          image: <approved-inference-image@sha256:digest>
          resources:
            requests:
              cpu: "2"
              memory: 8Gi
            limits:
              memory: 16Gi
              nvidia.com/gpu: 1
```

This is a **Pod-template fragment**, not a complete deployable TrueFoundry specification. Replace the image placeholder and confirm actual labels/taints before use. CPU limits, memory sizing, security context and probes depend on the deployment. For standard GPU resources, a GPU limit alone supplies the matching GPU request.

## 6. Tenant isolation and policy

A production isolation design should include:
- **RBAC and workload identity:** restrict who can submit GPU workloads and access tenant data.
- **ResourceQuota:** constrain namespace GPU resource consumption; validate admission behavior for extended resources in the target cluster.
- **LimitRange/admission controls:** require approved GPU limits, registries, security contexts and pool selection as appropriate.
- **NetworkPolicy:** control east-west and egress paths, subject to CNI support.
- **Node-pool boundaries:** separate incompatible or sensitive workloads where operational risk justifies it.
- **Data handling:** avoid host mounts, privileged containers and shared scratch storage unless explicitly approved.
- **GPU sharing review:** validate isolation guarantees with the chosen hardware and sharing mechanism; do not assume shared GPU memory or execution is tenant-isolated.

Kubernetes namespaces and node taints are **not** hard security boundaries by themselves.

## 7. GPU sharing and fragmentation

### Whole-GPU allocation

The default device-plugin `nvidia.com/gpu` resource generally represents integer GPU allocations. Scheduling reserves a resource even when instantaneous GPU utilization is low. This is predictable but can produce idle capacity.

### MIG (Multi-Instance GPU)

On supported NVIDIA GPUs and configurations, MIG partitions hardware into GPU instances with defined compute/memory resources. Advertised resource names and scheduling semantics depend on the device-plugin strategy. **Do not assume A10G supports MIG** or that all GPU models expose the same profiles.

### Time-slicing / shared GPU

Device-plugin time-slicing can advertise shared resources, but it does not provide the same memory/fault isolation guarantees as hardware partitioning. Oversubscription can cause contention, unpredictable latency and out-of-memory failures. Review NVIDIA plugin configuration, monitoring semantics and tenant risk before enabling it.

### Choosing a mode

| Workload | Typical starting point | Validate |
|---|---|---|
| Latency-sensitive production inference | Dedicated whole GPU | P99 latency, VRAM, availability |
| Multi-tenant strict isolation | Dedicated pool / whole GPU; MIG only when supported and approved | Threat model, isolation, GPU SKU |
| Low-utilization dev/test | Approved sharing strategy | Contention, fairness, cost |
| Large model training | Dedicated multi-GPU nodes | Interconnect, topology, gang scheduling requirements |
| Bursty inference | Capacity-aware scaling and warm pools where needed | Cold-start, quotas, GPU provisioning lead time |

No sharing choice is universally optimal.

## 8. Capacity and autoscaling

Capacity planning must consider **allocatable GPUs minus allocated GPU requests**, not just `nvidia-smi` utilization. A Pod also needs CPU, memory, storage, topology and taint/affinity fit.

### Planning model

```text
Schedulable GPU slots per pool ≈
  sum(node allocatable GPUs) - sum(assigned GPU requests)
```

This is a simplified GPU-only inventory, **not** a complete scheduler simulation. Track unusable fragmentation from memory, CPU, GPU model, node topology, taints, quotas and Pod constraints.

### Autoscaling and cloud operations

- **EKS:** GPU managed node groups or Karpenter provisioners/NodePools, instance availability, AMI/driver strategy and GPU Pod provisioning.
- **AKS:** GPU VM node pools, cluster autoscaler, GPU SKU quotas and node image compatibility.
- **GKE:** GPU accelerator node pools, autoscaling, GPU availability and driver installation modes.

Autoscalers cannot resolve every Pending Pod: impossible selectors, unsupported GPU types, quota exhaustion, cloud capacity shortage, and incorrect plugin registration still require operator action. GPU node startup and driver readiness can create substantial cold-start latency.

## 9. Production-safe diagnostic workflow [SAFE-READ]

### 9.1 Confirm context and pool inventory

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -L node-pool,gpu,gpu_type
kubectl get nodes --show-labels
kubectl describe node <gpu-node>
kubectl get node <gpu-node> -o jsonpath='{.status.allocatable.nvidia\.com/gpu}{" GPU allocatable\n"}{.spec.taints}{" taints\n"}'
```

### 9.2 Check workload placement and quota

```bash
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <gpu-pod>
kubectl -n <namespace> get pod <gpu-pod> -o yaml
kubectl -n <namespace> get resourcequota
kubectl -n <namespace> describe resourcequota
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

### 9.3 Examine node resource consumers

```bash
kubectl -n <namespace> get pods --field-selector spec.nodeName=<gpu-node> -o wide
kubectl describe node <gpu-node>
```

`kubectl describe node` summarizes allocated resource requests across namespaces. The namespaced Pod query is useful for a particular tenant but **not** a complete cluster-wide GPU allocation inventory.

### 9.4 Exclusion ledger

| Node | Ready | Selector/affinity | Tolerated taints | GPU allocatable/allocated | CPU/memory/topology | Eligible? |
|---|---|---|---|---|---|---|
| gpu-a | | | | | | |
| gpu-b | | | | | | |

Populate with actual evidence. Multiple exclusion reasons can apply to the same node.

## 10. Failure and remediation matrix

| Evidence | Likely cause | Safe next action |
|---|---|---|
| `Insufficient nvidia.com/gpu` | No free schedulable GPU resources | Check allocations, scale-up feasibility, pool quota |
| `untolerated taint` | Missing permitted pool access | Compare approved pool policy and Pod tolerations |
| `didn't match node affinity/selector` | Placement mismatch | Compare desired and actual labels |
| GPU nodes exist, no GPU advertised | Device-plugin/driver issue | Inspect GPU stack health; see Chapter 4.3 |
| Pending after autoscaler attempts | Quota, cloud capacity, impossible constraints | Inspect autoscaler/provisioning evidence |
| Pods Running, P99 latency spikes | Contention, GPU sharing, model or workload pressure | Correlate DCGM, VRAM, inference metrics |
| Pool heavily underutilized | Fragmentation, overprovisioning, demand variability | Review scheduling inventory and FinOps baselines |
| Unexpected tenant on GPU pool | Incomplete taint/admission/RBAC isolation | Audit policy and owning controllers |

Avoid removing taints, deleting other tenants' Pods, changing node labels or relaxing admission policy as an unapproved workaround.

## 11. FinOps and reliability controls

Track GPU node-hours, reserved versus allocated GPUs, active utilization, VRAM, idle time, inference throughput, latency, provisioning lead time and failed placements. Attribute costs by pool/workspace/team using trusted labels and billing data. Node-hour cost and allocated-GPU cost are not equivalent; account for warm capacity, fragmentation, storage, networking and interconnect.

For production inference, optimize **cost per successful request under SLO**, not GPU utilization alone. Aggressive scale-to-zero may violate cold-start and availability targets.

## 12. Change management and rollback [CHANGE-CONTROL]

**Prechange:** export approved node-pool configuration, current taints/labels, quota and admission rules; record affected deployments, PodDisruptionBudgets, autoscaler settings and GPU availability baseline.

**Canary:** create or modify a small nonproduction pool through IaC; validate label/taint registration, device plugin, Pod scheduling, model readiness and inference.

**Rollout:** stage tenant migrations; preserve enough capacity for disruption; confirm cloud quota and GPU supply. GPU node drain may interrupt stateful inference and should have explicit outage approval.

**Rollback:** revert IaC/GitOps changes to the prior pool configuration and deployment placement contract; restore known-good node images and capacity as necessary; verify no orphaned workloads or unexpected tenant placement.

**Acceptance:** correct tenants scheduled to correct pools, no unauthorized placement, expected GPU resources advertised, inference healthy, P99/error rates stable and cost impact recorded.

## 13. Acceptance checklist

- [ ] Can explain why taints, tolerations and selectors must be considered together.
- [ ] Can inventory allocated GPUs and distinguish capacity from utilization.
- [ ] Can identify namespace and pool isolation limits.
- [ ] Can compare whole-GPU, MIG and time-slicing tradeoffs without assuming hardware support.
- [ ] Can diagnose Pending GPU Pods and autoscaler blockers with evidence.
- [ ] Can design canary, rollback, SLO and FinOps checks.

**Lab:** [4.5 — GPU Pool Isolation and Capacity Check](../labs/05-dedicated-gpu-node-pools-isolation-check.md)  
**Next:** Chapter 4.6 — GPU Validation, Monitoring and Troubleshooting.

## References

- [Kubernetes — Assign Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Kubernetes — Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Kubernetes — Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes — Schedule GPUs](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [NVIDIA Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
