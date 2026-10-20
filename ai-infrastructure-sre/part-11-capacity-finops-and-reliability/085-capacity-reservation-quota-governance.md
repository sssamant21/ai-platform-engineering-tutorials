# Chapter 085 — Capacity Reservation and Quota Governance

**Part 11 — Capacity, FinOps and Reliability | AI Infrastructure Engineering & SRE**

**Status:** Documentation published. Offline exercises, cloud/Kubernetes staging validation, security review and independent acceptance remain pending.

## 1. Mission

Design auditable GPU capacity reservations and quota governance across cloud providers, Kubernetes clusters, AI training queues and production inference. Prevent noisy-neighbor starvation, quota exhaustion, reservation waste, unapproved overcommit and reliability headroom erosion while supporting fair, predictable access to scarce accelerators.

**Prerequisites:** Chapters 083–084, Kubernetes ResourceQuota and PriorityClass, node provisioning, Kueue/Volcano or equivalent batch queue, cloud account quota/capacity management, FinOps and incident management.

### Outcomes

- Distinguish **cloud quota**, **physical capacity**, **capacity reservation**, **billing commitment**, **Kubernetes admission quota** and **actual GPU scheduling feasibility**.
- Define tenant allocation, borrowing, preemption, expiration, audit and exception controls.
- Calculate headroom for high-priority inference and distributed training.
- Reconcile reservation coverage and cost without assuming that quota approval guarantees GPU supply.
- Validate changes through read-only audits, policy tests and controlled staging admission exercises.

## 2. Six independent capacity control planes

\`\`\`text
Business demand and service priority
          ↓
Cloud service quota / project limits
          ↓
Cloud regional/AZ physical supply
          ↓
Capacity reservations / procurement contracts
          ↓
Kubernetes allocatable GPUs / node topology
          ↓
Namespace quotas + batch queue admission + priority
          ↓
Actual workload placement / SLO
\`\`\`

| Mechanism | What it controls | What it does NOT guarantee |
|---|---|---|
| Cloud quota | permitted resource count/request limit | actual GPU stock |
| Physical availability | GPU inventory in region/AZ | tenant allocation |
| Capacity reservation | specific provider-defined reserved capacity | compatibility with every workload |
| Savings commitment | billing discount or spend commitment | physical capacity unless explicitly bundled |
| Kubernetes ResourceQuota | namespace resource consumption/admission | fairness across queues or GPU topology |
| Kueue/Volcano quota | queue/cohort admission and sharing | node/GPU physical placement by itself |

Cloud provider products and guarantees vary by region, instance family and contract. Confirm current reservation, payment, cancelation and failover terms from provider documentation and your actual agreement.

## 3. Governance principles

1. **Explicit ownership:** every reservation and quota has tenant, service, cost center, environment and technical owner.
2. **SLO first:** preserve minimum serving capacity and documented training deadlines.
3. **Least privilege:** only authorized teams change cloud quotas, reservations, PriorityClasses or queue policies.
4. **Finite borrowing:** borrowed capacity must have an expiry, reclaim policy and visibility.
5. **No hidden overcommit:** account for physical SKU, memory, MIG profile, node count, fabric and failure domains.
6. **Auditable changes:** record requester, approver, reason, effective time, rollback and budget impact.
7. **Cost transparency:** distinguish committed spend, reserved capacity, consumed GPU-hours and idle exposure.
8. **Policy as code:** version control admission, quota and priority configuration with tests.

## 4. Workload priority and tenant tiers

| Tier | Example workload | Policy posture |
|---|---|---|
| P0 | critical customer inference | protected baseline, multi-zone redundancy |
| P1 | time-critical training / batch serving | deadline-aware allocation |
| P2 | normal experimentation | guaranteed floor plus fair-share borrowing |
| P3 | interruptible research | opportunistic, checkpoint-aware preemption |

These are illustrative. PriorityClass alone does not implement business fairness; scheduler configuration, preemption behavior, PodDisruptionBudgets and workload semantics matter. Avoid granting arbitrary teams the ability to self-assign P0.

### 4.1 Policy contract

\`\`\`yaml
tenant: team-a
gpu_sku: H100
region: region-a
guaranteed_gpus: 16
burst_ceiling_gpus: 32
priority_tier: P1
borrowing_allowed: true
borrow_expiry_hours: 24
preemption_policy: "CHECKPOINT_AWARE"
reservation_owner: "platform-capacity"
budget_owner: "finops"
review_due_utc: null
\`\`\`

This is a governance example, **not** a native Kubernetes manifest.

## 5. Kubernetes ResourceQuota

GPU extended resources can be quota-managed by their resource names. Confirm installed vendor plugin and cluster support before applying policy.

### 5.1 Example namespace quota (staging only)

\`\`\`yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: gpu-team-a
  namespace: gpu-team-a
spec:
  hard:
    requests.nvidia.com/gpu: "8"
    requests.cpu: "64"
    requests.memory: 256Gi
    pods: "50"
\`\`\`

\`ResourceQuota\` is namespace-scoped and counts admission requests. It does not guarantee eight physical GPUs exist, that nodes are healthy, or that the requested topology can be placed. Use an explicit namespace and verify \`kubectl describe resourcequota\` after a staging apply.

### 5.2 Read-only audit

\`\`\`bash
kubectl get resourcequota -A
kubectl get limitrange -A
kubectl get priorityclass
kubectl get nodes -o wide
kubectl describe nodes
kubectl get pods -A --field-selector=status.phase=Pending
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

A pod can remain Pending due to quota, unschedulable GPU SKU, missing MIG profile, taints, topology, CPU/memory or provisioning delay. Diagnose the exact scheduler/admission event rather than assuming quota.

## 6. Kueue and gang admission

Kueue can manage ClusterQueues, LocalQueues, ResourceFlavors and cohort borrowing. Configuration and semantics depend on the installed version; validate API schema with the cluster's CRDs.

**Example conceptual resource design:**

\`\`\`text
ClusterQueue: production-serving
  guaranteed: 32 compatible GPUs
  borrowing: disabled
ClusterQueue: scheduled-training
  guaranteed: 32 compatible GPUs
  borrowing: up to 16 from research cohort
ClusterQueue: research
  guaranteed: 8 compatible GPUs
  borrowing: opportunistic, reclaimable
\`\`\`

Queue admission does not replace Kubernetes placement. For multi-node training, require gang-aware admission and enough compatible nodes/NIC topology; otherwise reserved GPU counts may still strand jobs.

## 7. Cloud capacity and commercial commitments

| Provider family | Capacity mechanisms to evaluate | Governance concern |
|---|---|---|
| AWS | EC2 quotas, On-Demand Capacity Reservations, Capacity Blocks where eligible, Savings Plans/Reserved Instances | reservation scope, instance matching, pricing and cancellation |
| Azure | regional VM quotas, capacity reservations, savings plans/reservations | SKU/zone availability and distinct billing terms |
| Google Cloud | GPU quotas, reservations, committed-use discounts | zone/machine compatibility and commitment terms |
| On-prem | rack/GPU inventory, procurement, power/network reservations | lead time, hardware sparing and depreciation |

Do not equate a Savings Plan, reserved-instance discount or committed-use discount with guaranteed physical GPU capacity. Some offerings have separate reservation and billing mechanics.

## 8. Capacity math and failure headroom

### 8.1 Baseline + burst + resilience

\[
C_{\mathrm{target}}=C_{\mathrm{protected}}+C_{\mathrm{guaranteed}}+C_{\mathrm{approved\,burst}}+C_{\mathrm{failure\,buffer}}
\]

Avoid double-counting overlapping commitments. Partition by GPU SKU and region/AZ, then round to valid node bundles.

### 8.2 Synthetic example

A platform has 64 installed GPUs (8 nodes × 8 GPUs):
- P0 serving protected: 16 GPUs.
- P1 training guaranteed: 24 GPUs.
- P2 research guaranteed: 8 GPUs.
- Unallocated physical pool: 16 GPUs.

If one 8-GPU node fails, installed usable GPUs become **56**, but placement and tenancy can still block an 8-GPU job. If the platform must tolerate an entire AZ failure, the reserve must be calculated from **zone-specific** supply rather than an aggregate fleet count.

A research burst of 16 GPUs can be allowed only if borrowing does not violate protected capacity and documented failure reserve. Define a reclaim process that respects checkpointing and customer impact.

## 9. Reservation lifecycle and audit

\`\`\`text
Request → forecast justification → quota/capacity feasibility
   → FinOps approval → platform/security approval
   → provision/reserve → attach workload ownership
   → monitor utilization and coverage → renewal/expiry review
   → release/resize with rollback → verify billing and capacity
\`\`\`

Every reservation record should include provider resource ID, SKU, region/AZ, start/end, owner, billing basis, capacity guarantee scope, utilization, expiration and cancellation constraints. Use automation to flag orphaned, underused or expiring reservations.

### 9.1 Reservation register template

\`\`\`yaml
reservation_id: RES-EXAMPLE
provider: AWS_OR_AZURE_OR_GCP
region: null
zone: null
gpu_sku: null
gpu_count: null
capacity_guarantee: "UNVERIFIED"
billing_commitment: "UNVERIFIED"
tenant: null
service: null
cost_center: null
owner: null
approver: null
start_utc: null
expiry_utc: null
renewal_review_utc: null
utilization_percent: null
rollback: null
audit_links: []
\`\`\`

## 10. Metrics and alerts

Monitor:
- quota usage and denial events per namespace/project/tenant;
- reserved versus consumed GPU-hours and stranded capacity;
- pending jobs by **reason**, queue age p95 and deadline miss rate;
- compatible available GPUs by SKU/MIG, node and AZ;
- borrowing, reclaim and preemption events;
- reservation expiration/renewal lead time;
- protected SLO headroom and AZ/node failure coverage;
- billing commitment utilization and actual avoidable spend.

Avoid labeling telemetry with unrestricted customer identifiers. Define alert ownership and an escalation chain.

## 11. Failure and policy scenarios

| Incident | Investigation |
|---|---|
| Cloud quota approved, provisioning fails | physical regional stock |
| Reservation exists, job Pending | instance/GPU/MIG/topology mismatch |
| P0 serving degraded after P2 burst | protection and borrowing enforcement |
| Queue quota free, no node capacity | scheduling vs admission |
| GPU pool underutilized, tenant denied | fairness, partitioning and constraints |
| AZ failure consumes protected reserve | zone-specific resilience |
| Reservation expiring before launch | renewal lead time |
| Commitment bill persists after downsizing | billing vs capacity separation |
| Priority escalation by untrusted tenant | RBAC and admission control |
| Preemption kills training without checkpoint | workload-safe reclaim |
| New GPU SKU not in policy | ResourceFlavor and quota review |
| Orphaned reservation after project closure | ownership and expiry audit |

## 12. Five production runbooks

**A — GPU quota exhaustion:** identify exact cloud/Kubernetes/queue denial; verify demand and available supply; request approved increase; protect P0 service; validate admission.

**B — Reservation supply mismatch:** compare SKU, region/AZ, node shape, MIG, fabric and instance matching; engage provider; use approved alternative placement.

**C — Noisy-neighbor and borrowing incident:** inspect tenant floors/ceilings, priorities and reclaim policy; preserve checkpoints; restore protected capacity without unapproved evictions.

**D — Expiring or idle commitment:** verify contract, utilization and upcoming demand; obtain FinOps approval for renewal, transfer or release; reconcile invoice and capacity.

**E — Zone failure headroom:** inspect compatible per-zone inventory and protected service redundancy; stop unsafe maintenance, coordinate scaling/placement and validate SLO.

## 13. Sixteen hands-on exercises

1. Inventory GPU capacity by SKU, AZ and node bundle.
2. Distinguish quota, reservation, commitment and placement.
3. Audit namespace ResourceQuota and LimitRange.
4. Audit PriorityClass/RBAC escalation paths.
5. Model Kueue ClusterQueue/LocalQueue and ResourceFlavor.
6. Simulate gang scheduling admission failure.
7. Define P0–P3 tenant allocation policies.
8. Calculate guaranteed, burst and protected headroom.
9. Simulate safe capacity borrowing and reclaim.
10. Model node and AZ failure under reservations.
11. Draft AWS/Azure/GCP quota/reservation comparison.
12. Build an auditable reservation register.
13. Detect orphaned and expiring reservations.
14. Calculate commitment utilization and cost exposure.
15. Test policy-as-code and alert-routing tabletop.
16. Produce governance runbooks and acceptance evidence.

## 14. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Capacity control-plane distinctions | 20 |
| Kubernetes/queue quota policy and tests | 20 |
| Tenant fairness, borrowing and isolation | 20 |
| Cloud reservation/commitment governance | 15 |
| Failure-domain headroom and FinOps | 15 |
| Runbooks, audit evidence and review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 with no critical SLO protection, privilege escalation, checkpoint safety, quota bypass, unapproved commitment or data-security failure. Documentation publication does not mean policies were applied or tested.

## 15. References

- [Kubernetes Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [Kueue Documentation](https://kueue.sigs.k8s.io/)
- [AWS On-Demand Capacity Reservations](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html)
- [Azure Capacity Reservations](https://learn.microsoft.com/en-us/azure/virtual-machines/capacity-reservation-overview)
- [Google Cloud Reservations](https://cloud.google.com/compute/docs/instances/reservations-overview)
- [FinOps Foundation Framework](https://www.finops.org/framework/)

**Next:** Chapter 086 — N-1 and Multi-Zone GPU Resilience.
