# Chapter 052 — Cluster Autoscaler, Karpenter and GPU Warmup

**Part 07 — Kubernetes GPU Scheduling and Autoscaling | Chapter 052/100**

> **Status:** Written tutorial published. Live labs, independent technical/security review and operational acceptance pending. Autoscaler APIs, Karpenter CRDs, EKS/AKS/GKE support, node image drivers and GPU families are version/provider-specific. Do not alter production node pools or autoscaler configuration without approved change control.

## Learning objectives

Explain how GPU Pod demand triggers node provisioning, compare Cluster Autoscaler and Karpenter, model end-to-end GPU readiness and model warmup, size warm pools and disruption budgets, diagnose cloud capacity failures and GPU provisioning loops, and build SLO/FinOps-safe runbooks.

## 1. Architecture

```text
LLM traffic / queue / token demand
            │
        HPA or KEDA
            │
      desired GPU Pods
            │
     Pending / unschedulable
            │
 ┌──────────┴───────────┐
 │ Cluster Autoscaler   │  Karpenter (supported platforms)
 │ node-group templates │  NodePool / NodeClass / NodeClaim
 └──────────┬───────────┘
            │
  cloud quota / GPU SKU / zone / capacity
            │
        GPU node launch
            │
     OS, driver, toolkit, device plugin
            │
      kubelet Ready + GPU allocatable
            │
  Pod image pull → model artifact → CUDA warmup
            │
    readiness → traffic → TTFT/TPOT SLO
```

Autoscalers create node capacity; they do not directly make models ready. A node can be `Ready` before GPUs are allocatable or model servers can answer requests.

## 2. Cluster Autoscaler (CA)

CA generally scales configured node groups using their scheduling templates and Pod unschedulability. Correct GPU resources, labels, taints and group boundaries matter. CA cannot create a GPU SKU outside available node groups. Cloud provider integration and version compatibility must be verified.

## 3. Karpenter

Karpenter provisions nodes from workload constraints using NodePool/NodeClass/NodeClaim APIs on supported platforms. Current API versions, fields, interruption handling and consolidation policies change; inspect installed CRDs and version rather than copying outdated manifests. On EKS, EC2NodeClass and NodePool are common, but not portable to AKS/GKE.

## 4. Avoid competing ownership

Do not configure CA and Karpenter to independently manage the same capacity domain without an explicit design. Distinct node groups/pools and clear ownership prevent unexpected scale-in and race conditions. Verify actual controllers before changes.

## 5. GPU scheduling prerequisites

Node labels, taints/tolerations, GPU SKU, per-node device count, MIG profiles, topology, CPU/memory/ephemeral storage and zone constraints must all be satisfiable. If the Pod requires a label or GPU resource no provisionable node template advertises, autoscaling will not solve Pending.

## 6. Safe-read discovery

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get crd | grep -Ei 'nodepool|nodeclaim|ec2nodeclass|karpenter'
kubectl get daemonsets -A
```

Inspect discovered controller namespaces and logs only with authorized access. Do not assume Karpenter exists.

## 7. GPU warmup critical path

```text
T_ready =
 T_queue_observation + T_HPA/KEDA_decision
 + T_provisioner_reconciliation + T_cloud_launch
 + T_node_bootstrap + T_driver_toolkit_plugin
 + T_image_pull + T_model_download
 + T_CUDA_initialize + T_model_warmup
 + T_readiness_stabilization
```

These terms can overlap; the sum is a conservative critical-path model, not a guaranteed measured duration. Record p50/p95 for each stage from actual traces and events.

## 8. Worked example: cold-start budget

Illustrative sequential timings:

| Stage | Example |
|---|---:|
| queue observation and scaling decision | 30 s |
| cloud GPU node launch | 150 s |
| node bootstrap and GPU readiness | 90 s |
| image pull | 45 s |
| model download | 120 s |
| CUDA/model warmup | 75 s |
| readiness | 15 s |
| **Total illustrative** | **525 s (8 min 45 s)** |

If a request's TTFT objective is seconds, scaling from zero GPUs cannot satisfy it. Maintain warm replicas or queue asynchronously with an explicit user-facing SLA.

## 9. Warm pool capacity

```text
warm GPU demand ≈ ceil(peak admitted token-work rate /
                        measured sustainable token-work per GPU)
```

Adjust for redundancy, burst buffer, KV-cache headroom, zone loss, MIG profile, tenant isolation and maintenance. Avoid sizing solely from average GPU utilization.

## 10. Model artifact and image caching

Pre-pulled images and regional model artifact caches can reduce startup, but cache invalidation, model digests, IAM/KMS, EBS/PVC/object-store throughput and disk pressure create operational risks. Pin image and model versions; do not preload confidential weights onto unapproved shared nodes.

## 11. GPU node startup validation

A node becomes useful for inference only after node Ready, `nvidia.com/gpu` allocatable, driver/toolkit compatibility, Pod admission/scheduling, CUDA runtime success, model load and readiness. Track each stage separately; do not use node Ready as the sole SLO.

## 12. Spot/on-demand and disruption

Spot GPU capacity may lower cost but introduces interruptions. Protect user-facing inference with on-demand baseline, zone diversity, termination handling and stream-aware drain. PDBs only constrain certain voluntary disruptions; they do not prevent spot reclaim or hardware failure.

## 13. Cloud provider specifics

EKS supports managed/self-managed node groups and Karpenter with AWS GPU instance families, quotas, EC2 capacity and EBS/image choices. AKS uses GPU node pools and supported autoscaling integrations. GKE offers GPU node pools and provider-specific autoscaling. Always verify regional GPU SKU availability, driver management and provisioning policy.

## 14. Observability

Measure Pending duration, autoscaler decision time, node launch latency, kubelet Ready, GPU allocatable delay, model pull/warmup, readiness, GPU utilization, TTFT/TPOT, streaming terminal completion, capacity shortfall, consolidation churn, interruptions and GPU-hours per valid completion.

## 15. Cost and reliability controls

Set node count limits, GPU quota, instance-family allowlists, minimum warm capacity, idle TTL/consolidation policy, budget alerts and emergency scaling stop. Consolidation can save money but may cause repeated model downloads, cold starts and interrupted streams.

## 16. Troubleshooting decision tree

```text
GPU Pods Pending or cold?
 ├─ HPA not scaling → metrics/thresholds
 ├─ provisioner absent → CA/Karpenter ownership
 ├─ no matching node template → selectors/taints/GPU SKU
 ├─ cloud quota/capacity → region/AZ/family
 ├─ node launched but not Ready → bootstrap/CNI
 ├─ node Ready but GPU zero → driver/plugin
 ├─ Pod scheduled but not ready → image/model/KMS
 ├─ model ready but SLO poor → warmup/KV/traffic
 ├─ nodes churn → consolidation/disruption
 └─ idle GPU cost → warm pool/limits/FinOps
```

## 17. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | CA controller absent | deployment |
| F02 | Karpenter CRD/controller absent | discovery |
| F03 | CA/Karpenter overlapping ownership | pool audit |
| F04 | node template missing GPU resource | Pending |
| F05 | node selector cannot match any pool | scheduler |
| F06 | GPU node taint mismatch | events |
| F07 | unsupported GPU SKU in region | cloud capacity |
| F08 | cloud GPU quota exhausted | provisioner |
| F09 | AZ GPU capacity unavailable | cloud |
| F10 | EC2NodeClass/NodeClass invalid | controller |
| F11 | NodeClaim provisioning stuck | status |
| F12 | node bootstrap fails | kubelet |
| F13 | GPU driver incompatible | driver logs |
| F14 | device plugin advertises zero GPU | allocatable |
| F15 | image pull throttled | events |
| F16 | model artifact IAM/KMS denied | startup |
| F17 | model download saturates network | telemetry |
| F18 | CUDA/model warmup exceeds SLO | readiness |
| F19 | scale-from-zero breaks TTFT | request traces |
| F20 | consolidation churns GPU nodes | events |
| F21 | spot interruption drops streams | termination |
| F22 | PDB assumed to protect all evictions | audit |
| F23 | rollback leaves mixed provisioners | GitOps diff |
| F24 | warm pool GPU-hours exceed budget | FinOps |

## 18. Sixteen operational runbooks

1. Inventory CA/Karpenter ownership and versions.
2. Diagnose unschedulable GPU Pods.
3. Validate node templates and GPU resources.
4. Recover labels/taints/affinity mismatches.
5. Resolve cloud GPU quota/capacity.
6. Recover Karpenter NodeClass/NodeClaim failures.
7. Diagnose node bootstrap/CNI failures.
8. Restore driver/device-plugin GPU readiness.
9. Recover image pull and registry failures.
10. Restore model artifact IAM/KMS and download.
11. Optimize CUDA/model warmup and readiness.
12. Protect TTFT with minimum warm replicas.
13. Contain consolidation/scale-down churn.
14. Handle spot interruption and stream drain.
15. Reconcile GPU-hours and idle node costs.
16. Execute provisioner rollback and incident RCA.

## 19. Eight tabletop game days

1. Cloud GPU quota prevents scale-out.
2. GPU instance family unavailable in one AZ.
3. Karpenter NodeClaim fails to bootstrap.
4. Driver/plugin delay leaves nodes Ready but unusable.
5. Model download takes ten minutes.
6. Scale-to-zero breaks interactive TTFT.
7. Consolidation terminates an active streaming replica.
8. Rollback leaves CA and Karpenter competing.

Default tabletop; live injection requires approved isolated staging.

## 20. Twenty acceptance gates

G01 scope/versions; G02 provisioner ownership; G03 node templates; G04 GPU SKU/quota; G05 labels/taints; G06 Pending detection; G07 node launch; G08 bootstrap/CNI; G09 driver/plugin; G10 image pull; G11 model download; G12 CUDA warmup; G13 readiness; G14 warm pool sizing; G15 spot/disruption; G16 TTFT/TPOT SLO; G17 telemetry; G18 rollback; G19 GPU FinOps; G20 independent review.

See [companion lab](../labs/052-cluster-autoscaler-karpenter-gpu-warmup-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 053 — Multi-Zone GPU Capacity and Disruption Budgets.**
