# Chapter 031 — GPU Node Autoscaling and Karpenter

**AI Infrastructure Engineering & SRE · Part 04 — Kubernetes GPU Administration · Chapter 031/100**

**Status:** Published documentation. Offline tests, live AWS/EKS staging, cloud-cost validation, independent review and production acceptance remain pending.

## Learning objectives

- Explain GPU scale-out and scale-in across Kubernetes scheduler, Karpenter, EC2 capacity, EKS, GPU Operator and device-plugin readiness.
- Compare Karpenter with Cluster Autoscaler and managed node groups, and identify ownership boundaries.
- Design GPU NodePool/EC2NodeClass requirements, capacity type, GPU SKU, quotas, topology, N-1 resilience, warm capacity and disruption controls.
- Investigate Pending pods, failed provisioning, GPU node readiness, driver/plugin delays, interrupted training, cost overruns and scale-in hazards.
- Build an offline fail-closed autoscaling admission gate and perform approved staging game days.

## 1. Reference architecture

```text
GPU workload pending (nvidia.com/gpu request)
    -> scheduler cannot fit on existing nodes
    -> Karpenter controller observes unschedulable workload
    -> NodePool constraints / EC2NodeClass configuration
    -> EC2 fleet capacity, instance type, zone, subnet, security group, IAM
    -> EC2 node boot / EKS join
    -> driver + toolkit + GPU Operator/device plugin readiness
    -> kubelet advertises GPU extended resources
    -> scheduler binds pod / device-plugin Allocate
    -> CUDA/NCCL smoke / application SLO
    -> consolidation, drift or interruption handling
    -> checkpoint / drain / replacement / scale-in
```

**Important:** an EC2 node joining Kubernetes is not proof it can run CUDA. Karpenter can provision instances while the GPU Operator/device plugin is still initializing. Scale-in is an application-reliability event for stateful training.

## 2. Autoscaling control planes

| Capability | Karpenter | Cluster Autoscaler |
|---|---|---|
| Scale-out | provisions nodes from NodePool constraints and cloud offerings | adjusts configured node-group sizes |
| Instance flexibility | selects eligible EC2 instance offerings | constrained by node-group templates/configuration |
| Scale-in | consolidation, drift, disruption budgets | utilization-driven group scale-down |
| GPU integration | requires valid node class, GPU-compatible image/bootstrap and device discovery | requires valid GPU node-group templates and device discovery |
| AWS-specific | Karpenter AWS provider | works across multiple providers |

Avoid overlapping autoscaler ownership of the same GPU capacity. AWS Karpenter is not a generic AKS autoscaler; AKS node pools typically use supported AKS Cluster Autoscaler or other Azure-specific mechanisms. Verify Karpenter API version, AWS provider release, NodePool and EC2NodeClass schema against the installed version before applying examples.

## 3. GPU capacity planning

GPU provisioning depends on:
- GPU instance family/SKU and region/AZ offering.
- EC2 On-Demand or Spot capacity, service quotas, account permissions and pricing.
- vCPU/memory/ephemeral storage alongside GPU count.
- Subnet IP availability, ENI/network and security group reachability.
- Node image/AMI, EKS cluster endpoint access, driver/toolkit/operator compatibility.
- Taints, startup taints, node selectors, topology, ResourceQuota and priority.
- Multi-node training gang scheduling or queue coordination where needed.
- N-1 failure-domain resilience and checkpoint durability.

**Illustrative calculation:** 12 GPU requests with 4 GPUs per node require at least 3 nodes for raw GPU count; N-1 survival of a single node requires at least 4 eligible nodes **if all other CPU/memory, topology and scheduling constraints permit**. Real capacity plans include node/system overhead and reservation headroom.

## 4. Karpenter resource model and staged example

Karpenter versions and CRD schemas change. The following example is a **design illustration only** for a reviewed Karpenter release; validate the exact installed CRD before deployment.

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: gpu-staging
spec:
  template:
    metadata:
      labels:
        accelerator-pool: gpu-staging
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: gpu-staging
      taints:
        - key: dedicated
          value: gpu
          effect: NoSchedule
      requirements:
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["g5"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
  limits:
    cpu: "128"
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 10m
```

Do not apply this YAML without an approved matching EC2NodeClass, security-reviewed AMI, subnet/security-group selectors, IAM role, bootstrap, cluster endpoint, Karpenter CRD/version check, and node disruption policy. Exact `disruption` fields and allowed values are release-dependent. A CPU-only NodePool limit is not a complete GPU budget; also govern node count, GPU-hours, cloud spend, GPU resource quotas and cloud quotas.

GPU workloads must request the correct extended resource and tolerate the protected taint. Do not permit tenants to bypass protected node pools by freely adding tolerations or selectors.

## 5. Read-only operational discovery

```bash
kubectl config current-context
kubectl get nodepools
kubectl get ec2nodeclasses
kubectl get nodes -L karpenter.sh/nodepool,topology.kubernetes.io/zone
kubectl get pods -A --field-selector=status.phase=Pending -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl get daemonsets -A | grep -Ei 'nvidia|gpu-operator'
```

These Karpenter CRDs are absent when Karpenter is not installed; treat that as an architecture finding. On the approved Karpenter controller deployment:

```bash
kubectl get deployments -A | grep -i karpenter
kubectl logs deployment/APPROVED_KARPENTER_DEPLOYMENT -n APPROVED_NAMESPACE --since=30m
kubectl describe pod APPROVED_GPU_POD -n APPROVED_NAMESPACE
kubectl describe node APPROVED_GPU_NODE
```

Use AWS read-only permissions for EC2 capacity, quota, subnet IP and launch failures; avoid exposing IAM or private network details in public tickets.

## 6. GPU node readiness pipeline

A newly launched GPU node must pass distinct stages: EC2 instance running; EKS node registration; kubelet Ready; NVIDIA driver initialized; GPU Operator operands healthy; device-plugin registered; expected `nvidia.com/gpu`/MIG resources allocatable; CUDA canary passed; DCGM telemetry fresh; application correctness and SLO verified.

Use approved startup taints or workload admission to prevent GPU pods being scheduled before critical components are ready. Startup taints must be removed by the responsible bootstrap process; a stuck taint can cause prolonged Pending.

## 7. Scale-in, consolidation and disruption

Consolidation or drift replacement can terminate GPU nodes and interrupt training. Karpenter disruption budgets and `karpenter.sh/do-not-disrupt` behavior are version-specific and do not replace checkpointing or cluster-wide reliability planning. Coordinate PDBs, graceful termination, terminationGracePeriodSeconds, workload queues, training checkpoints, Spot interruption signals and storage durability.

**Do not assume** a PDB alone protects all GPU workloads or all node termination paths. N-1 capacity must be demonstrated, not inferred from aggregate GPU counts.

## 8. Spot versus On-Demand

Spot reduces cost but has interruption risk and may be unavailable for scarce GPU families. Use Spot only when workloads can checkpoint, resume, tolerate restart and satisfy SLO. On-Demand can also encounter capacity or quota failures. Maintain diversified instance offerings and zones when workload compatibility allows; avoid mixing incompatible GPU generations within tightly coupled distributed jobs without validation.

## 9. Metrics, SLOs and FinOps

| Metric | Why it matters |
|---|---|
| pending GPU pod age p95/p99 | queue/scale-out SLO |
| EC2 launch failure rate | capacity/permissions issue |
| node launch-to-GPU-ready time | bootstrap/operator bottleneck |
| allocatable GPU units vs requested | scheduler supply |
| GPU UUID / driver / plugin readiness | end-to-end correctness |
| Spot interruption and failed drain | training reliability |
| checkpoint age and resume time | data-loss/recovery objective |
| idle GPU node-hours | consolidation/cost |
| GPU-hours per successful training step or inference token | unit economics |
| N-1 capacity and AZ distribution | failure tolerance |

Alert on correlated signals, not utilization alone. A GPU node at low utilization may be intentionally reserved for burst or N-1 recovery.

## 10. Twelve troubleshooting scenarios

1. GPU pod Pending due to quota or unavailable GPU instance family.
2. Karpenter controller lacks EC2/IAM permission.
3. EC2NodeClass subnet/security-group selectors match no resources.
4. Insufficient subnet IPs prevent node startup.
5. EC2 Spot or On-Demand capacity unavailable in requested AZ.
6. Node joins but GPU driver fails to initialize.
7. Device plugin does not advertise GPU resources.
8. Startup taint remains after Operator becomes ready.
9. GPU Pod affinity/taints conflict with NodePool constraints.
10. Consolidation interrupts an uncheckpointed training job.
11. Karpenter drift rollout exceeds N-1 capacity.
12. GPU spend spikes due to idle nodes or overbroad instance eligibility.

## 11. Ten incident simulations

1. AWS quota exhausted during training burst.
2. Spot interruption kills an uncheckpointed worker.
3. GPU AMI upgrade breaks driver compatibility.
4. Missing subnet IPs block provisioning.
5. GPU Operator readiness lags EC2 node launch.
6. Wrong startup taint blocks all GPU pods.
7. Karpenter consolidation races training checkpoint.
8. Single-zone GPU capacity outage.
9. IAM regression prevents instance launch.
10. Cost runaway due to stale GPU workloads.

Each drill records UTC timeline, NodePool/EC2NodeClass, instance SKU, node/GPU UUID, affected pod/job, checkpoint, SLO, cost, rollback and RCA/COA.

## 12. Eight operational runbooks

**RB-031-A — Pending GPU jobs:** separate admission/scheduler failures from Karpenter provisioning failures.

**RB-031-B — EC2 provisioning:** inspect cloud quotas, instance offerings, IAM, subnet IPs and security groups.

**RB-031-C — GPU readiness:** trace EC2 -> kubelet -> driver -> toolkit -> plugin -> CUDA canary.

**RB-031-D — NodePool drift:** verify immutable node images, disruption budget, PDBs, checkpoints and rollback.

**RB-031-E — Spot recovery:** preserve checkpoint and restart distributed ranks consistently.

**RB-031-F — Topology/capacity:** reconcile AZ, GPU SKU, NCCL needs and N-1 headroom.

**RB-031-G — Cost containment:** identify idle GPU-hours, capacity reservations, right-sizing and SLO guardrails.

**RB-031-H — RCA and acceptance:** verify recovery, GPU correctness, telemetry, cost, owner and signoff.

## 13. Hands-on staging labs

1. Inventory actual EKS version, Karpenter version, NodePool, EC2NodeClass, GPU SKU, node AMI and quotas.
2. Render and validate a reviewed staging NodePool/EC2NodeClass against installed CRDs; do not apply unreviewed templates.
3. Submit an approved GPU canary and record Pending-to-GPU-ready latency.
4. Confirm driver, toolkit, device plugin, CUDA UUID and telemetry.
5. Simulate a blocked NodePool requirement in **offline** policy tests, not by breaking production.
6. Perform a controlled staging drain/scale-in with checkpoint and N-1 headroom.
7. Compare On-Demand/Spot cost and interruption risk using actual billing inputs.
8. Document rollback, evidence and change acceptance.

[Lab 031 — GPU Node Autoscaling and Karpenter Gate](../labs/031-gpu-node-autoscaling-karpenter-check.md) implements offline Python 3.11+ validation. It does not provision EC2 or alter Kubernetes.

## 14. 100-point assessment

| Domain | Points |
|---|---:|
| Autoscaling architecture | 20 |
| Karpenter/EC2NodeClass and capacity | 20 |
| GPU readiness, security and disruption | 20 |
| Incidents, recovery and FinOps | 20 |
| Executable tests and staging evidence | 20 |
| **Total** | **100** |

Pass >=90 with no critical checkpoint, GPU correctness, security, N-1 or cost blocker.

## References

- [Karpenter documentation](https://karpenter.sh/docs/)
- [Karpenter NodePools](https://karpenter.sh/docs/concepts/nodepools/)
- [Karpenter EC2NodeClasses](https://karpenter.sh/docs/concepts/nodeclasses/)
- [Karpenter Disruption](https://karpenter.sh/docs/concepts/disruption/)
- [Amazon EKS Auto Scaling](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [Kubernetes Node Autoscaling](https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/)

**Next: Chapter 032 — GPU Workload Rollouts and Disruptions.**
