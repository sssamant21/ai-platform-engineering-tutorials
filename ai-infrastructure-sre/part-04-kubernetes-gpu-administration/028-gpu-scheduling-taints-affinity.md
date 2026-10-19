# Chapter 028 — GPU Scheduling, Taints and Affinity

**Part 04 — Kubernetes GPU Administration · Chapter 028/100**

**Publication:** Documentation published; offline execution, live GPU staging, security review, and production acceptance remain separate milestones.

## Outcomes

Diagnose GPU pod Pending/Unschedulable, design taints/tolerations, node affinity, pod affinity/anti-affinity, topology spread, priorities, quotas and admission gates, and safely recover a GPU scheduling incident on EKS/AKS/Kubernetes. Integrate NVIDIA GPU Operator/device plugin resource discovery, MIG, topology, N-1 capacity and tenant boundaries.

## Architecture

```text
GPU workload (resource request, selector, affinity, toleration)
  -> API admission (quota / policy)
  -> scheduler: Filter (resources, taints, affinity, volume/topology)
  -> Score (preferred affinity, spread, plugins)
  -> bind to node
  -> kubelet -> NVIDIA device plugin Allocate -> runtime/CDI
  -> CUDA smoke -> application SLO
```

**Crucial:** scheduler placement is not proof of GPU health, device UUID affinity, CUDA readiness or NCCL topology. `nvidia.com/gpu` is an integer extended resource; capacity and allocatable differ from free device memory. GPU scheduling under MIG/time-slicing may expose different resource names/semantics. A node's GPU label is an administrative assertion, not proof of actual hardware.

## 1. Resource and scheduling semantics

- **requests/limits:** GPU extended resources generally require limits, with an omitted request defaulting to the limit; if both specified they must match. Fractional `nvidia.com/gpu` requests are not supported as ordinary integer extended resources.
- **taints:** `NoSchedule` blocks new placement without matching toleration; `NoExecute` can evict existing pods lacking toleration; `PreferNoSchedule` is soft. Toleration permits but does not force placement.
- **nodeSelector:** all labels must match; useful for a dedicated, verified GPU pool.
- **required node affinity:** hard placement requirement; preferred affinity is a scoring preference, not a guarantee.
- **pod anti-affinity:** spread failure domains, but overly strict rules can strand scarce GPU capacity.
- **topologySpreadConstraints:** `maxSkew`, `topologyKey`, `whenUnsatisfiable` and label selectors govern distribution. Ensure eligible nodes carry consistent topology labels.
- **priority/preemption:** can displace lower-priority pods if permitted; does not create GPU capacity and may conflict with checkpoint/recovery objectives.
- **node readiness:** allocatable GPU count, node Ready, plugin registration, taints and other resources must all satisfy placement.

## 2. Example staging GPU workload

Review labels and image digest before use. The example is **not** a production deployment manifest.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-scheduler-canary
  namespace: gpu-staging
  labels:
    app: gpu-scheduler-canary
spec:
  restartPolicy: Never
  nodeSelector:
    accelerator-pool: staging-gpu
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: [approved-zone]
  containers:
    - name: gpu-smoke
      image: nvcr.io/nvidia/cuda:12.4.1-base-ubuntu22.04
      command: ["sh", "-c"]
      args: ["nvidia-smi -L"]
      resources:
        requests:
          cpu: "500m"
          memory: 1Gi
          nvidia.com/gpu: 1
        limits:
          cpu: "1"
          memory: 2Gi
          nvidia.com/gpu: 1
```

Replace `approved-zone` with a real verified zone and pin an approved immutable image digest. A Kubernetes toleration for a taint named `nvidia.com/gpu` is meaningful only if the node is actually tainted with that key; taint keys vary by cluster. Never invent tolerations that bypass tenant isolation.

## 3. Read-only diagnostic lab

```bash
kubectl config current-context
kubectl get nodes -L accelerator-pool,topology.kubernetes.io/zone
kubectl get nodes -o json | jq -r '.items[] | [.metadata.name, (.status.allocatable["nvidia.com/gpu"] // "none")] | @tsv'
kubectl get pods -A --field-selector=status.phase=Pending -o wide
kubectl describe pod APPROVED_GPU_POD -n APPROVED_NAMESPACE
kubectl describe node APPROVED_GPU_NODE
kubectl get events -A --sort-by=.lastTimestamp
kubectl get daemonsets -A | grep -Ei 'nvidia|device-plugin'
```

Investigate `FailedScheduling` messages such as `Insufficient nvidia.com/gpu`, untolerated taints, unmatched node affinity, insufficient CPU/memory, unbound PVCs, quota violations, topology skew, and missing plugin registration. Pod scheduling failures may be admission errors (no Pod created), not just Pending pods.

On the approved node:

```bash
nvidia-smi -L
nvidia-smi topo -m
nvidia-smi --query-gpu=uuid,name,driver_version --format=csv
```

Do not equate scheduler node placement with NVLink peer topology; GPU-to-GPU and GPU-to-NIC placement require workload-specific and device-aware verification.

## 4. Safe placement design

**Training:** require a compatible GPU SKU and capacity; use topology/zone-aware scheduling and distributed-job coordination, then verify NCCL/rank mapping. Strict anti-affinity can reduce schedulability. **Inference:** favor availability-zone resilience, autoscaler capacity and measured p99 latency. **MIG:** request the precise plugin-exposed resource key/profile and isolate tenants according to platform policy. **Cloud GPU pools:** account for EC2/Azure SKU quotas, zone availability, Karpenter/node autoscaler provisioning latency, startup taints and driver/plugin readiness.

**N-1:** calculate usable capacity after loss of the largest relevant failure domain, not just total GPU count. Include CPU, memory, topology, network, tenant quota, and minimum checkpoint recovery capacity.

## 5. Twelve troubleshooting cases

| Symptom | Evidence | Response |
|---|---|---|
| Pending Insufficient GPU | allocatable, requests | validate true available capacity |
| untolerated taint | Pod events, node taints | approve precise toleration |
| required affinity mismatch | labels, expressions | correct verified node labels |
| wrong zone restriction | topology labels | review zone and quota |
| GPU plugin missing | plugin logs, allocatable | restore plugin/driver |
| MIG profile not found | node resource keys | request actual profile |
| CPU/memory blocks GPU pod | Pod requests and node resources | right-size safely |
| quota rejection | ResourceQuota/events | seek tenant-approved quota |
| PDB/drain stalls replacement | PDB and workload status | coordinate checkpoint/drain |
| priority preempts training | PriorityClass/events | revise priority policy |
| topology spread blocks placement | eligible domains, maxSkew | adjust only after resilience review |
| scheduled but CUDA fails | runtime, CDI, driver, UUID | troubleshoot post-scheduling path |

## 6. Ten incident simulations

1. GPU pool taint added without workload toleration.
2. Wrong `nodeSelector` prevents training startup.
3. GPU device plugin loses registration after driver upgrade.
4. MIG resource name changes during rollout.
5. Zone capacity exhausted while another zone is healthy.
6. GPU pod consumes CPU/memory headroom and cannot schedule.
7. High-priority job evicts an uncheckpointed training pod.
8. Strict anti-affinity prevents N-1 recovery.
9. Autoscaler launches nodes but GPU operands are not ready.
10. Placement succeeds while CUDA/NCCL canary fails.

For every drill capture UTC timeline, pod UID, node/GPU UUID, scheduler events, resource request, expected capacity, blast radius, checkpoint, SLO, rollback and owner.

## 7. Eight runbooks

**RB-028-A — Admission and quota:** distinguish API rejection from scheduler Pending; inspect ResourceQuota and LimitRange.

**RB-028-B — Taint/toleration:** compare exact keys, effects and operators; never remove protective taints without approval.

**RB-028-C — Node affinity:** validate node labels, SKU, zones, required versus preferred constraints.

**RB-028-D — Extended resources:** reconcile NVIDIA device-plugin registration, MIG resources and allocatable supply.

**RB-028-E — Topology/spread:** assess failure domains, NCCL needs and capacity tradeoffs.

**RB-028-F — Preemption/autoscaling:** protect checkpoint and SLO; verify cloud quota and node readiness.

**RB-028-G — CUDA canary:** validate container GPU UUID, driver/runtime and actual workload correctness.

**RB-028-H — RCA/acceptance:** record fix, rollback, N-1 capacity, tenant security, observability and signoff.

## 8. Companion lab and 100-point assessment

[Lab 028 — GPU Scheduling Admission Gate](../labs/028-gpu-scheduling-taints-affinity-check.md) implements an offline Python 3.11+ fail-closed evaluator and pytest scenarios. It cannot execute Kubernetes scheduling or validate real GPU hardware.

| Category | Points |
|---|---:|
| Scheduler/resource architecture | 20 |
| Taints, affinity and topology | 20 |
| MIG, autoscaling, quota and security | 20 |
| Incident response and runbooks | 20 |
| Lab tests and staging evidence | 20 |
| **Total** | **100** |

Pass >=90 and no critical isolation, correctness, capacity, checkpoint or SLO blocker.

## References

- [Kubernetes Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Kubernetes Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Kubernetes Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [NVIDIA Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)

**Next:** Chapter 029 — GPU Resource Quotas and Multitenancy.
