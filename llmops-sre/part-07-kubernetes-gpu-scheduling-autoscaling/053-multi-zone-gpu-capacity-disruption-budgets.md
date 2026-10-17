# Chapter 053 — Multi-Zone GPU Capacity and Disruption Budgets

**Part 07 — Kubernetes GPU Scheduling and Autoscaling | Chapter 053/100**

> **Status:** Written tutorial published. Live execution, independent technical/security review and operational acceptance pending. GPU SKU availability, Kubernetes disruption semantics, autoscaler behavior and cloud zone capabilities depend on provider and version. Never drain production GPU nodes or change node pools/PDBs without approved change control.

## Learning outcomes

Design zone-resilient GPU inference capacity, size N-1 failure headroom, use topology spread and PodDisruptionBudgets appropriately, coordinate Cluster Autoscaler/Karpenter with draining and streaming, distinguish voluntary versus involuntary disruption, and implement runbooks, game days and acceptance evidence.

## 1. Multi-zone GPU architecture

```text
Client → regional ingress / LLM gateway
              │
     inference admission and rate limits
              │
      zone-aware traffic routing
        ┌─────┼─────┐
        ▼     ▼     ▼
       AZ-a  AZ-b  AZ-c
       GPU   GPU   GPU
       pool  pool  pool
        │     │     │
   model replicas + caches + telemetry
              │
   node autoscaler / quota / capacity
              │
  PDB + topology spread + stream drain
              │
  zone failure recovery + RTO/RPO evidence
```

GPU inventory across three AZs does not guarantee failover if the required GPU SKU is unavailable, hard affinity is unsatisfiable, model artifacts are inaccessible or the surviving zones lack spare headroom.

## 2. Capacity and N-1 sizing

Define `D` as measured peak concurrent GPU demand, `C_z` as usable healthy GPU capacity in zone `z`, and `R` as reserved maintenance/headroom. To tolerate losing the largest zone, require:

```text
sum(C_z) - max(C_z) >= D + R
```

Example: three zones each have 4 usable GPUs, peak demand is 7 GPU-equivalents and maintenance headroom is 1: surviving 8 GPUs meets the 8-GPU target **only if workloads can be rebalanced and model warmup is within RTO**. MIG and time-sliced advertised units must be normalized to actual capacity rather than blindly counted as physical GPUs.

## 3. Failure domains

A zone failure can impact compute, networking, volumes, object storage access, IAM/KMS, private endpoints, registry/model caches and control-plane dependencies. Validate independent failure domains and regional dependencies rather than assuming all replicas are independent.

## 4. Topology spread constraints

```yaml
# DESIGN FRAGMENT: labels/topology verified before use
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: llm-inference
```

Hard `DoNotSchedule` may block rollout if a zone has no matching GPU capacity. `ScheduleAnyway` is a preference, not a guarantee. Check node affinity/taints and topology domains before deployment.

## 5. Pod anti-affinity

Anti-affinity can spread model replicas across nodes, but hard anti-affinity may deadlock when GPU pools are small or zones fail. Compare `preferredDuringSchedulingIgnoredDuringExecution` with required rules and validate actual eligible nodes.

## 6. PodDisruptionBudgets

PDBs constrain certain **voluntary evictions** through the Eviction API, such as many drain operations. They do not prevent node crashes, zone outages or all controller-driven terminations. A PDB is not an availability guarantee. Configure `minAvailable` or `maxUnavailable` against the actual replica count and expected maintenance capacity.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: llm-inference
  namespace: approved-staging
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: llm-inference
```

This is a design example; the `minAvailable` value must be derived from real serving capacity and replicas.

## 7. PDB arithmetic

For 3 healthy Pods with `minAvailable: 2`, at most one healthy Pod can ordinarily be voluntarily evicted while two remain available. However, readiness changes, in-flight disruptions, unavailable replicas and `unhealthyPodEvictionPolicy` affect behavior. Validate installed Kubernetes semantics and use `kubectl describe pdb` to inspect `DisruptionsAllowed`.

## 8. GPU readiness and disruption

GPU Pods can take minutes to recover due to node provisioning, driver/plugin initialization, image pulls, model downloads, CUDA graph capture and warmup. A Pod becoming Ready does not necessarily prove it can meet steady-state TTFT/TPOT; use synthetic inference probes and SLO telemetry.

## 9. Stream-safe draining

Stop routing new requests, allow active streams to complete or hand off where supported, set realistic `terminationGracePeriodSeconds`, and coordinate gateway endpoints, model server readiness, preStop hooks and scale-down. A PDB cannot protect an in-flight stream from all failures.

## 10. Cluster Autoscaler and Karpenter

CA and Karpenter must understand zone labels, node-pool templates, taints, GPU families, disruption budgets and cloud capacity. Consolidation and node termination should respect SLO and cost guardrails. Do not assume a Karpenter disruption budget is identical to a Kubernetes PDB; both must be validated separately.

## 11. EKS, AKS and GKE

Verify per-zone GPU SKU availability, cloud quotas, autoscaler/node pool limits, zone-affinity rules, managed node image driver support, regional model artifact replication and failover paths. GPU inventory and reservation options differ by cloud and region.

## 12. RTO and model cache

Recovery time includes failure detection, scheduling/provisioning, GPU initialization, image/model download, CUDA warmup and traffic restoration. RPO for stateless inference may be zero for model weights if artifacts are durable, but in-flight requests and conversation state can be lost; define business-appropriate recovery semantics.

## 13. Observability

Track GPU capacity by AZ, healthy/allocatable devices, Pod readiness and zone skew, `DisruptionsAllowed`, drain latency, pending duration, autoscaler failures, model warmup, stream termination rate, TTFT/TPOT and GPU-hours.

## 14. Safe-read diagnostics

```bash
date -u
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get nodes --show-labels
kubectl get pods -A -o wide
kubectl get pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl -n APPROVED_NS describe pdb APPROVED_PDB
kubectl describe node APPROVED_GPU_NODE
```

No `kubectl drain`, node deletion or zone fault injection without approval.

## 15. Troubleshooting decision tree

```text
Zone-resilience incident?
 ├─ GPU zone unavailable → capacity/quota/family
 ├─ Pods all in one zone → topology/affinity
 ├─ Pending after zone loss → surviving GPU headroom
 ├─ drain blocked → PDB DisruptionsAllowed
 ├─ drain succeeds but streams fail → graceful shutdown
 ├─ node ready but model cold → GPU/model warmup
 ├─ autoscaler cannot expand → zone template/quota
 ├─ artifact download fails → regional storage/IAM
 └─ rollback incomplete → PDB/topology/node-pool drift
```

## 16. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | only one GPU AZ provisioned | inventory |
| F02 | GPU SKU unavailable in backup AZ | cloud capacity |
| F03 | regional GPU quota exhausted | cloud quota |
| F04 | N-1 headroom miscalculated | capacity sheet |
| F05 | MIG shares counted as full GPUs | accounting |
| F06 | hard topology spread blocks rollout | scheduler |
| F07 | affinity pins all Pods to one zone | Pod spec |
| F08 | anti-affinity impossible after outage | Pending |
| F09 | node taints differ by zone | node describe |
| F10 | PDB selector matches wrong Pods | PDB status |
| F11 | PDB minAvailable exceeds viable replicas | drain |
| F12 | PDB has zero DisruptionsAllowed | describe |
| F13 | involuntary outage bypasses PDB | events |
| F14 | drain interrupts active streams | gateway |
| F15 | termination grace too short | termination |
| F16 | GPU node provisioning delayed | autoscaler |
| F17 | device plugin not ready after failover | allocatable |
| F18 | model download cache absent in AZ | readiness |
| F19 | KMS/storage unavailable in survivor zone | IAM/storage |
| F20 | Karpenter consolidation triggers churn | node events |
| F21 | spot reclaim hits all replicas | interruption |
| F22 | TTFT spikes during zone recovery | SLO |
| F23 | rollback leaves PDB/topology drift | GitOps diff |
| F24 | idle N-1 warm GPUs exceed budget | FinOps |

## 17. Sixteen operational runbooks

1. Inventory GPU capacity per zone.
2. Calculate N-1 GPU and model-serving headroom.
3. Recover GPU SKU/quota in surviving zones.
4. Validate topology spread constraints.
5. Recover affinity/anti-affinity deadlocks.
6. Resolve cross-zone taint/label drift.
7. Audit PDB selector and replica assumptions.
8. Recover zero DisruptionsAllowed during maintenance.
9. Handle involuntary zone outage beyond PDB.
10. Drain inference safely with active streams.
11. Tune termination grace and readiness.
12. Restore node autoscaler GPU provisioning.
13. Recover driver/plugin and model warmup.
14. Restore regional artifact/IAM dependencies.
15. Reconcile zone redundancy GPU-hours.
16. Execute zone failover rollback and RCA.

## 18. Eight tabletop game days

1. Entire AZ loses GPU capacity during peak demand.
2. Backup AZ cannot launch required GPU SKU.
3. Hard topology spread blocks recovery.
4. PDB prevents planned GPU node maintenance.
5. Node drain terminates active streams.
6. Model artifact store unavailable during failover.
7. Karpenter consolidation creates a second outage.
8. Rollback restores Pods but not zone topology.

Default tabletop; live fault injection requires isolated-staging approval.

## 19. Twenty acceptance gates

G01 scope/versions; G02 per-zone GPU inventory; G03 N-1 capacity; G04 cloud quota; G05 GPU SKU compatibility; G06 topology spread; G07 affinity; G08 taints; G09 PDB selection; G10 DisruptionsAllowed; G11 stream-safe drain; G12 termination grace; G13 autoscaler capacity; G14 driver/plugin; G15 artifact/model warmup; G16 TTFT/TPOT SLO; G17 zone failover RTO; G18 rollback; G19 GPU FinOps; G20 independent review.

See [companion lab](../labs/053-multi-zone-gpu-capacity-disruption-budgets-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 054 — GPU Scheduling, Quota and Pending Pod Troubleshooting.**
