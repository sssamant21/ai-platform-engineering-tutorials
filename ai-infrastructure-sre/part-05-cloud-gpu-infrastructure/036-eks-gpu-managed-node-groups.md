# Chapter 036 — EKS GPU Managed Node Groups

**AI Infrastructure Engineering & SRE · Part 05 — Cloud GPU Infrastructure · Chapter 036/100**

**Status:** Published documentation. Offline code execution, EKS staging validation, security review and production acceptance are separate pending activities.

## Learning objectives

Design, provision, secure, operate and recover Amazon EKS GPU managed node groups (MNGs). Understand the EKS control plane, EC2 Auto Scaling groups, GPU-optimized AMIs, node bootstrap, IAM, launch templates, NVIDIA driver and device-plugin ownership, scheduling, quota/capacity constraints, rolling updates, disruption, telemetry and cost. Build an offline admission policy with negative tests and an approved staging acceptance plan.

## 1. Reference architecture and ownership

```text
AWS account / region / VPC / private subnets / endpoints
  |
EKS control plane -> Managed Node Group -> EC2 Auto Scaling group
  |                                           |
Kubernetes API + scheduler                    EC2 launch template / AMI
  |                                           |
GPU pod requests nvidia.com/gpu                kubelet registration
  |                                           |
taints / tolerations / topology                NVIDIA driver + toolkit
  |                                           |
  +----> GPU device plugin -> allocatable GPU resources
                       |
                   CUDA / NCCL / model runtime
                       |
       DCGM / CloudWatch / Prometheus / application SLO
                       |
            checkpoints, drain, rollback, RCA
```

EKS manages parts of node-group provisioning and updates, **not the entire GPU application runtime**. EC2 instances joining EKS does not guarantee NVIDIA driver readiness, device-plugin registration, CDI allocation or CUDA compatibility. Managed node groups typically use an underlying EC2 Auto Scaling group; avoid manually changing ASG properties that EKS owns.

## 2. Node group lifecycle and design

Plan cluster version, node AMI OS family, instance type, architecture, AZ/subnet placement, root EBS encryption, launch template, scaling limits, labels, taints, node IAM role, cloud networking, update configuration and release/rollback.

| Dimension | Production decision |
|---|---|
| GPU family | choose exact EC2 GPU SKU from regional offerings |
| AMI | reviewed EKS-optimized accelerated AMI or compatible custom AMI |
| Driver ownership | AMI/host automation **or** GPU Operator, not competing owners |
| Runtime | containerd, NVIDIA toolkit/CDI, device-plugin strategy |
| Placement | AZs, subnet IP capacity, topology, optional EFA |
| Isolation | dedicated node group, taints, admission, tenant quotas |
| Scaling | desired/min/max with Cluster Autoscaler where supported |
| Disruption | update maxUnavailable, PDB, checkpoint, N-1 |
| Identity | EKS node role and separate workload identity |
| Cost | instance-hours, EBS, cross-AZ transfer, unused GPUs |

**Version sensitivity:** EKS AMI choices, AL2/AL2023 bootstrap methods, launch-template support, managed node group update settings and GPU driver availability change with EKS/Kubernetes releases. Consult the currently supported EKS documentation and validate the selected AMI release. For AL2023, custom launch-template user data typically uses `nodeadm` NodeConfig; do not copy AL2 `bootstrap.sh` procedures into AL2023 blindly.

## 3. IAM, network and security boundaries

The node IAM role needs only permissions required for node registration, image pulling and platform-specific add-ons. Separate application S3/FSx access via EKS Pod Identity or IRSA where configured. Use private subnets, VPC endpoints/NAT as appropriate, restricted security groups, KMS-encrypted EBS, IMDSv2, patching, signed/pinned images, least-privilege RBAC and Pod Security Admission. Nodes with public IPs or broad administrative instance profiles require explicit security review.

The VPC CNI must have enough pod IP capacity. ENI and prefix-delegation behavior varies by instance type and CNI settings. A node may be Ready while pods remain Pending or ContainerCreating due to IP exhaustion. EFA requires supported instance type, network interface, security-group and NCCL/libfabric configuration; do not assume all GPU MNGs have EFA.

## 4. GPU device and driver readiness

The operational chain is EC2 running -> kubelet registered -> node Ready -> NVIDIA driver healthy -> container toolkit/CDI compatible -> NVIDIA device plugin registered -> expected `nvidia.com/gpu` or MIG resource key allocatable -> CUDA smoke -> application SLO. A managed node group does not automatically guarantee all these stages.

Check for driver conflicts between the GPU-optimized AMI and GPU Operator. If using NVIDIA GPU Operator with preinstalled drivers, select the appropriate driver-management configuration and validate the Operator version. Keep privileged Operator operands in governed platform namespaces; ordinary GPU workload pods should remain least privilege.

## 5. Example node-group configuration (review-only)

This is an illustrative `eksctl` configuration for a disposable EKS environment; it is **not deployable unchanged**. Replace placeholders, validate installed `eksctl` schema, EKS/Kubernetes version, IAM, subnet IDs, instance availability, quotas, AMI/driver ownership and budget. No infrastructure should be created merely to run this tutorial.

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: REPLACE_APPROVED_CLUSTER
  region: us-east-1
managedNodeGroups:
  - name: gpu-staging
    instanceType: g5.xlarge
    desiredCapacity: 1
    minSize: 0
    maxSize: 2
    privateNetworking: true
    volumeSize: 100
    labels:
      workload-tier: gpu-staging
    taints:
      - key: dedicated
        value: gpu
        effect: NoSchedule
```

Do not treat this as a complete production cluster definition. Review supported GPU AMI type, bootstrap, encrypted volume and launch-template controls. Prefer approved IaC/Terraform or GitOps with code review and drift detection.

## 6. Read-only discovery lab

Requires approved AWS CLI v2 role and Kubernetes read-only access. Set cluster and region explicitly.

```bash
export AWS_PROFILE=approved-readonly
export AWS_REGION=us-east-1
export EKS_CLUSTER=approved-staging-cluster
aws sts get-caller-identity --profile "$AWS_PROFILE"
aws eks describe-cluster --name "$EKS_CLUSTER" --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'cluster.{Name:name,Version:version,Status:status,EndpointPublicAccess:resourcesVpcConfig.endpointPublicAccess}'
aws eks list-nodegroups --cluster-name "$EKS_CLUSTER" --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws eks describe-nodegroup --cluster-name "$EKS_CLUSTER" --nodegroup-name APPROVED_NODEGROUP \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'nodegroup.{Status:status,AMI:amiType,InstanceTypes:instanceTypes,Scaling:scalingConfig,Update:updateConfig,Health:health.issues}'
kubectl config current-context
kubectl get nodes -L eks.amazonaws.com/nodegroup,topology.kubernetes.io/zone
kubectl get nodes -o json | jq -r '.items[] | [.metadata.name, (.status.allocatable["nvidia.com/gpu"] // "none")] | @tsv'
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

Verify AWS and Kubernetes contexts point to the same approved environment. Avoid writing credentials or unredacted VPC identifiers to public logs.

## 7. GPU scheduling and quotas

A workload needs the appropriate GPU extended resource, tolerations for dedicated node taints and approved node affinity. A namespace GPU quota is an admission limit, not a capacity guarantee. Node-group max size, regional vCPU quotas, instance-type offerings, AZ capacity, pod IPs and GPU Operator readiness all influence Pending-to-GPU-ready latency. Use workload queues or admission control to prevent unbounded burst provisioning.

## 8. Cluster Autoscaler and managed node groups

Cluster Autoscaler can scale supported EKS managed node groups when configured with appropriate IAM, tags/discovery and node-group templates. Karpenter is an alternative node provisioning model and uses NodePool/EC2NodeClass rather than scaling an EKS managed node group in the same manner. Do not configure overlapping autoscaler ownership for the same GPU capacity. Validate autoscaler version compatibility with EKS, taints and GPU resources. Scale-from-zero needs correct node-group template information and GPU resource awareness.

## 9. Updates, PDBs and rollback

MNG update strategies can replace nodes, but GPU workload continuity depends on PDBs, checkpoint durability, N-1 headroom, device-plugin readiness and application SLO. A PDB governs voluntary eviction API operations, not all failures. Managed node group update failure codes and behavior should be inspected via `describe-nodegroup` and `describe-update`. An AMI update rollback may require a new approved node-group/launch-template release rather than a simple Kubernetes `rollout undo`.

Safe procedure: freeze changes -> inventory -> confirm checkpoint and spare capacity -> update one staging node group -> CUDA/NCCL canary -> workload SLO -> advance in controlled batches -> rollback on breach -> RCA/COA. Do not use `--force` node updates casually.

## 10. Observability and reliability metrics

| Signal | Meaning |
|---|---|
| EC2 launch and node-group health issues | capacity/IAM/bootstrap failures |
| EC2-to-node-Ready and GPU-ready duration | infrastructure provisioning SLO |
| GPU allocatable versus requested | usable scheduler capacity |
| NVIDIA driver and device-plugin status | runtime readiness |
| DCGM Xid/ECC, utilization, memory | hardware health and efficiency |
| pod Pending p95/p99 | queue and scaling experience |
| CNI available IPs | pod network headroom |
| GPU UUID and CUDA/NCCL smoke | correctness |
| checkpoint age, RPO/RTO | recoverability |
| GPU instance-hours and cost per job | FinOps |
| AZ spread and N-1 GPU headroom | resilience |

## 11. Twelve troubleshooting scenarios

1. Managed node group fails EC2 launch due to quota.
2. GPU instance type unavailable in requested AZ.
3. Node-group IAM role cannot join EKS.
4. AL2023 custom AMI bootstrap misconfigured.
5. Node Ready but NVIDIA driver absent or incompatible.
6. GPU device plugin registers zero resources.
7. GPU Operator conflicts with preinstalled AMI driver.
8. Containerd/CDI mismatch causes CreateContainerError.
9. Cluster Autoscaler cannot scale from zero.
10. VPC CNI pod IP exhaustion blocks workload startup.
11. Managed node group update stalls on PDB/checkpoint.
12. Inference p99 regresses after node AMI rollout.

## 12. Ten production incident simulations

1. Regional GPU vCPU quota exhausted.
2. Single-AZ GPU capacity shortage.
3. Node role or bootstrap regression.
4. GPU AMI driver incompatibility.
5. Device plugin fails after node update.
6. CNI IP exhaustion during scaling.
7. Autoscaler permissions regression.
8. Spot node group interruption before checkpoint.
9. Managed node group update exceeds N-1 headroom.
10. Orphaned GPU nodes breach cost guardrail.

Capture UTC timeline, account alias, cluster/node-group, EC2 instance type, AMI, GPU UUID, affected pods, SLO, checkpoint, rollback and owner.

## 13. Eight runbooks

**RB-036-A — Node-group discovery:** capture EKS version, node-group health, scaling, launch template and AMI.

**RB-036-B — EC2 capacity:** investigate quota, AZ offerings, ASG activity and subnet IPs.

**RB-036-C — Node registration:** inspect IAM, endpoint connectivity, bootstrap, kubelet and security groups.

**RB-036-D — GPU readiness:** driver -> toolkit -> plugin -> allocatable -> CUDA.

**RB-036-E — Autoscaler:** inspect scale-from-zero templates, IAM, quotas, scheduling and ownership.

**RB-036-F — Safe upgrade:** PDB, checkpoint, N-1, canary, version compatibility and rollback.

**RB-036-G — CNI/network:** IP allocation, ENI, DNS, EFA and pod connectivity.

**RB-036-H — RCA/FinOps:** restore SLO, validate GPU-hours, alerting, owner and change signoff.

## 14. Hands-on staging labs

1. Inventory EKS cluster, GPU managed node groups, AMI, launch template and driver ownership.
2. Review an approved IaC node-group definition; confirm instance offerings and quotas.
3. Measure node launch -> kubelet Ready -> GPU allocatable -> CUDA canary.
4. Validate GPU pod taints, affinity, quota, least privilege and workload identity.
5. Exercise a synthetic zero-GPU plugin or CNI-IP exhaustion incident offline.
6. Run an approved canary node-group update with checkpoint and N-1 capacity.
7. Measure recovery, application SLO and rollback.
8. Record evidence, cloud cost and security signoff.

[Lab 036 — EKS GPU Managed Node Groups Gate](../labs/036-eks-gpu-managed-node-groups-check.md) contains a Python 3.11+ fail-closed evaluator with pytest cases. It is not an EKS deployment tool.

## 15. 100-point assessment

| Domain | Points |
|---|---:|
| EKS MNG architecture, IAM, AMI and bootstrap | 20 |
| GPU runtime, scheduling and quotas | 20 |
| Scaling, AZ, networking and N-1 | 20 |
| Incidents, upgrades, rollback and FinOps | 20 |
| Automated tests and staging evidence | 20 |
| **Total** | **100** |

Pass >=90 with no critical GPU correctness, tenant security, capacity, checkpoint, networking or rollback blocker.

## References

- [Amazon EKS managed node groups](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)
- [EKS optimized AMIs](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html)
- [EKS managed node group updates](https://docs.aws.amazon.com/eks/latest/userguide/update-managed-node-group.html)
- [EKS Cluster Autoscaler](https://docs.aws.amazon.com/eks/latest/best-practices/cas.html)
- [EKS GPU support](https://docs.aws.amazon.com/eks/latest/userguide/ml-eks-optimized-ami.html)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)

**Next: Chapter 037 — AWS EFA and GPU Networking.**
