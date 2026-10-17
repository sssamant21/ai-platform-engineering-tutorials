# Chapter 048 — GPU Node Pools, Taints, Tolerations and Affinity

**Part 07 — Kubernetes GPU Scheduling and Autoscaling | Chapter 048/100**

> **Status:** Written tutorial published. Live labs, independent technical/security review and operational acceptance remain pending. Node-pool APIs, GPU labels and provisioning features vary across EKS, AKS, GKE and installed controllers. Do not change production node pools or taints without approved change control.

## Learning objectives

Design dedicated GPU node pools for inference and training; apply Kubernetes taints/tolerations, node selectors, required/preferred affinity and topology spread; protect latency-critical serving from batch jobs; troubleshoot scheduling deadlocks; validate GPU quota, node autoscaler and FinOps.

## 1. Reference architecture

```text
Ray Serve / KServe / vLLM GPU workload
            │
        Pod admission
        ├─ namespace quota
        ├─ Pod Security/RBAC
        └─ priority and policies
            │
    Kubernetes scheduler
        ├─ nodeSelector / required nodeAffinity
        ├─ taints / tolerations
        ├─ topology spread / pod affinity
        ├─ nvidia.com/gpu allocatable
        └─ resource requests / topology
            │
   GPU node pools (dedicated/spot/on-demand)
        ├─ NVIDIA GPU driver/device plugin
        ├─ model image/artifact cache
        └─ node provisioning (Karpenter/CA/cloud)
            │
      GPU telemetry, SLO, cost
```

**Toleration permits** scheduling onto a tainted node; **affinity/selector directs** scheduling toward a matching node. Neither creates GPU capacity.

## 2. Node-pool design matrix

| Pool | Typical workload | Isolation and operational concerns |
|---|---|---|
| latency-critical inference | user-facing streaming LLM | warm capacity, stable nodes, conservative disruption |
| batch inference | async jobs | bounded cost, interruptible where tolerated |
| training/fine-tuning | multi-GPU distributed jobs | same-node/rack topology, networking |
| MIG partitions | smaller isolated GPU profiles | supported SKU and resource profile |
| staging | synthetic validation | strict GPU-hour budget and quotas |

Choose GPU SKU, memory, region/AZ, on-demand/spot mix, instance quota and model storage bandwidth from measured workload needs.

## 3. Inventory [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl get pods -A -o wide
kubectl get priorityclasses
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl describe node APPROVED_GPU_NODE
```

Record verified labels, taints, GPU allocatable/allocated, zones, GPU node image, autoscaler owner and active model replicas.

## 4. Taints and tolerations

Kubernetes taints include `NoSchedule`, `PreferNoSchedule` and `NoExecute`. `NoSchedule` blocks new Pods without a matching toleration; `NoExecute` may evict existing Pods without one. Avoid using a broad `Exists` toleration that bypasses unrelated isolation.

```yaml
# DESIGN FRAGMENT — ONLY VALID IF NODE TAINT EXISTS
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu-inference"
    effect: "NoSchedule"
```

A toleration alone does not guarantee a Pod will land in the GPU pool.

## 5. Node selectors and required affinity

```yaml
# DESIGN FRAGMENT — VERIFY NODE LABELS FIRST
nodeSelector:
  workload-tier: "gpu-inference"
```

`requiredDuringSchedulingIgnoredDuringExecution` constrains placement at scheduling time. `preferredDuringSchedulingIgnoredDuringExecution` influences scoring but does not guarantee placement. A required selector that matches no GPU node yields Pending Pods.

## 6. Example complete Pod [LAB-MUTATE only if approved]

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-placement-smoke
  namespace: approved-staging
spec:
  restartPolicy: Never
  nodeSelector:
    workload-tier: gpu-inference
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu-inference
      effect: NoSchedule
  containers:
    - name: cuda
      image: registry.example.invalid/approved-cuda@sha256:REPLACE_WITH_REAL_DIGEST
      command: ["sh", "-c", "nvidia-smi && sleep 10"]
      resources:
        limits:
          nvidia.com/gpu: 1
```

The image/digest and node label are placeholders. Do not apply without verifying approved cluster, namespace, labels, taints, image and GPU-hour budget.

## 7. Pod affinity and anti-affinity

Use anti-affinity to avoid co-locating failure-domain-sensitive replicas where capacity permits. Hard anti-affinity can make GPU scheduling impossible when pools are small. Evaluate soft rules first, then validate disruption and SLO requirements.

## 8. Topology spread

`topologySpreadConstraints` can distribute replicas across nodes/zones using verified labels and `topologyKey`. `DoNotSchedule` is a hard constraint and can block rollout if zones lack matching GPU capacity; `ScheduleAnyway` is a preference. Verify eligible node domains and `minDomains` support in the installed Kubernetes version.

## 9. Multi-GPU placement

Tensor-parallel model serving may require multiple GPU devices in one Pod and possibly one node with NVLink/NVSwitch. Scheduler sees resource counts but not all interconnect properties. For distributed training or multi-node inference, coordinate workload operators, topology and network fabric.

## 10. EKS / AKS / GKE

EKS node groups and Karpenter NodePools, AKS node pools, and GKE node pools/auto-provisioning use different configuration APIs. Inspect provider-supported GPU families, quotas, autoscaler labels/taints propagation, node image drivers and disruption settings. A pending Pod cannot scale a pool if its hard constraints do not match any provisionable template.

## 11. Priority, preemption and PDBs

PriorityClass can protect critical inference from lower-priority workloads, but preemption is disruptive and does not create physical GPUs. PodDisruptionBudgets constrain some voluntary evictions, not all failure modes; do not rely on PDBs for hardware failures or all autoscaler actions.

## 12. Warm pools and startup budgets

GPU node provisioning + driver/plugin readiness + image pull + model download + CUDA/model warmup can exceed user request deadlines. Maintain warm replicas and minimum GPU nodes where justified; track GPU-hours and idle cost. Do not use aggressive scale-to-zero for strict TTFT SLOs without measured cold-start acceptance.

## 13. Security and tenant isolation

Apply RBAC, Pod Security, workload identity, namespace quotas, NetworkPolicies and admission rules. Taints are placement controls, not a strong security boundary. Prevent untrusted tenants from self-labeling Pods into privileged GPU pools without policy.

## 14. Observability

Track Pending reasons, scheduling latency, node allocatable GPU, GPU requests, GPU VRAM, node pool desired/current/ready, provisioning time, zone skew, eviction/preemption, TTFT/TPOT/E2E, terminal completion and GPU-hours per valid completion.

## 15. Troubleshooting decision tree

```text
GPU Pod Pending?
 ├─ admission denied → quota/policy
 ├─ insufficient GPU → allocatable vs allocated
 ├─ untolerated taint → tolerations
 ├─ selector mismatch → actual labels
 ├─ affinity mismatch → required expressions
 ├─ topology spread conflict → zones/eligible domains
 ├─ autoscaler cannot provision → template/quota/cloud
 ├─ multi-GPU fit impossible → per-node capacity
 ├─ node not Ready → driver/device plugin
 └─ image/model startup → registry/storage/runtime
```

Inspect scheduler events first; do not infer root cause from status alone.

## 16. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | GPU pool absent | node inventory |
| F02 | wrong GPU SKU | model VRAM |
| F03 | GPU node pool quota exhausted | cloud events |
| F04 | GPU node image incompatible | driver logs |
| F05 | node taint unexpected | node describe |
| F06 | toleration too broad | Pod spec |
| F07 | toleration missing | scheduler events |
| F08 | nodeSelector matches zero nodes | labels |
| F09 | required affinity impossible | scheduling |
| F10 | soft affinity sends workload to wrong pool | placement |
| F11 | anti-affinity blocks replica | events |
| F12 | topology spread blocks rollout | zone inventory |
| F13 | multi-GPU Pod cannot fit | per-node GPUs |
| F14 | device plugin not registered | allocatable |
| F15 | cloud autoscaler template mismatch | provisioner |
| F16 | spot GPU interruption | node events |
| F17 | PriorityClass preempts inference | Pod events |
| F18 | PDB assumption incorrect | disruption audit |
| F19 | image pull slow | Pod events |
| F20 | model download cold start | initializer |
| F21 | VRAM OOM despite GPU assignment | DCGM |
| F22 | tenant crosses intended pool boundary | admission audit |
| F23 | rollback leaves affinity/taint drift | manifest diff |
| F24 | idle GPU node cost spikes | FinOps |

## 17. Sixteen operational runbooks

1. Inventory GPU node pools and cloud ownership.
2. Validate GPU SKU, VRAM and model fit.
3. Recover cloud GPU quota/capacity.
4. Diagnose node image/driver compatibility.
5. Recover taint and toleration mismatch.
6. Audit broad tolerations and admission.
7. Restore selectors and required affinity.
8. Diagnose anti-affinity deadlocks.
9. Recover topology spread and zone skew.
10. Diagnose multi-GPU per-node fit.
11. Restore device-plugin and GPU allocatable.
12. Recover node autoscaler template mismatch.
13. Handle spot interruption and disruption.
14. Recover model startup and VRAM failures.
15. Restore tenant isolation, telemetry and FinOps.
16. Execute full node-pool/placement rollback and RCA.

## 18. Eight tabletop game days

1. Dedicated GPU node pool loses a required label.
2. Taint changes without Pod toleration update.
3. One AZ runs out of GPU instances.
4. Hard topology spread blocks a KServe rollout.
5. Multi-GPU Ray workload cannot fit on any node.
6. Spot GPU node interruption disrupts streaming.
7. Broad toleration places a tenant on restricted nodes.
8. Rollback restores app but not node-pool constraints.

Default tabletop; live fault injection requires isolated-staging approval.

## 19. Twenty acceptance gates

G01 scope/versions; G02 node-pool inventory; G03 GPU SKU/VRAM; G04 quota/capacity; G05 taints; G06 tolerations; G07 node selectors; G08 affinity; G09 anti-affinity; G10 topology spread; G11 multi-GPU fit; G12 autoscaler templates; G13 driver/device plugin; G14 startup/warm pool; G15 priority/disruption; G16 tenant/security; G17 telemetry/SLO; G18 rollback; G19 GPU FinOps; G20 independent review.

See [companion lab](../labs/048-gpu-node-pools-taints-tolerations-affinity-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 049 — GPU Resource Quotas and Namespace Isolation.**
