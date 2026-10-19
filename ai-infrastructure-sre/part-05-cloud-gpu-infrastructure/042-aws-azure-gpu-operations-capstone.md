# Chapter 042 — AWS and Azure GPU Operations Capstone

**AI Infrastructure Engineering & SRE · Part 05 — Cloud GPU Infrastructure · Chapter 042/100**

> **Scope:** Final enterprise acceptance capstone for Chapters 035–041. The chapter and offline lab are documentation artifacts; no AWS/Azure production access, GPU benchmark, quota approval or independent audit is implied.

## Outcomes and prerequisites

Demonstrate end-to-end GPU platform readiness on **AWS EKS/EC2** and **Azure AKS/GPU VMs**: capacity and quotas, secure GPU node bootstrap, driver ownership, CUDA/NCCL and networking, storage/checkpoint recovery, tenant identity and isolation, observability, FinOps, incident response and change governance. Use disposable, authorized staging subscriptions/accounts. Prerequisites: Chapters 035–041, AWS CLI, Azure CLI, kubectl, Python 3.11+, pytest, approved budgets and change windows.

## 1. Enterprise acceptance architecture

```text
AI service intake / SLO / security classification / data residency
                            |
              GPU demand, model VRAM and topology
                            |
           +----------------+----------------+
           |                                 |
    AWS account / region              Azure subscription / region
 EC2 quota, AZ offering,             family + regional quota,
 capacity reservation                SKU restriction, zone capacity
           |                                 |
       EKS GPU nodes                      AKS GPU user pool
  node IAM / Pod Identity              managed / workload identity
  driver / plugin / EFA                driver / plugin / RDMA
           |                                 |
           +---------------+-----------------+
                           |
         signed image, CUDA/NCCL, GPU workload canary
                           |
      encrypted datasets, durable checkpoint and restore
                           |
    DCGM, app SLO, logs, cost allocation and incident desk
                           |
      release gate: N-1, RPO/RTO, approval and rollback
```

**Acceptance rule:** Neither a green node nor an available quota proves usable physical GPU capacity. GPU allocatable does not prove CUDA/NCCL correctness. A saved checkpoint does not prove a recoverable restore. Discount commitments do not necessarily reserve GPU capacity.

## 2. Define the reference workload

Create two independently benchmarked reference workloads: **inference** (p95 latency, throughput, warm-up, concurrency, model VRAM, availability SLO) and **distributed training** (GPU-hours, dataset throughput, all-reduce performance, checkpoint RPO/RTO, job deadline). Record CUDA/NCCL versions, GPU generation, memory and topology. Never compare GPU counts across families without representative benchmarks.

Example *synthetic* acceptance targets (replace with approved service objectives): inference p95 <= 250 ms, CUDA-ready p95 <= 12 minutes, GPU telemetry freshness <= 120 seconds, checkpoint RPO <= 30 minutes, restore RTO <= 60 minutes, and budget <= approved forecast. These numbers are examples, **not** universal enterprise standards.

## 3. Capacity and quota acceptance

For AWS, determine the correct EC2 **On-Demand or Spot accelerated-instance vCPU quota bucket**, instance type offerings per AZ, current consumption, available network IPs and physical capacity evidence. For Azure, check **both total regional vCPU and VM-family vCPU quotas**, subscription SKU restrictions, zone offerings, current usage and physical capacity. For each cloud record GPU SKU, GPU count, VRAM, vCPUs, quota headroom, max fleet, AZ spread, N-1 replacement and expected provisioning time.

A valid failover plan must show the alternative GPU SKU can run the workload, the data can legally reside in the target region, and credentials, checkpoints and network paths have been tested. A fallback cloud is not ready merely because it has an account.

## 4. Read-only discovery

### AWS

```bash
export AWS_PROFILE=approved-readonly
export AWS_REGION=us-east-1
aws sts get-caller-identity --profile "$AWS_PROFILE"
aws ec2 describe-instance-type-offerings --location-type availability-zone \
  --filters Name=instance-type,Values=g5.12xlarge \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws service-quotas list-service-quotas --service-code ec2 \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'Quotas[].{Name:QuotaName,Limit:Value,Code:QuotaCode}'
aws eks describe-cluster --name APPROVED_CLUSTER --region "$AWS_REGION" \
  --profile "$AWS_PROFILE" --query 'cluster.{Name:name,Status:status,Version:version}'
```

### Azure

```bash
az account show --query '{Subscription:id,Tenant:tenantId}' -o json
az vm list-skus --location eastus --resource-type virtualMachines --all \
  --query "[?contains(name,'Standard_N')].{SKU:name,Restrictions:restrictions}" -o json
az vm list-usage --location eastus \
  --query '[].{Name:name.localizedValue,Used:currentValue,Limit:limit}' -o table
az aks nodepool list --resource-group APPROVED_RG --cluster-name APPROVED_AKS \
  --query '[].{Name:name,Mode:mode,Size:vmSize,Count:count,State:provisioningState}' -o table
```

### Kubernetes

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone,agentpool
kubectl get nodes -o json | jq -r '.items[] | [.metadata.name, (.status.allocatable["nvidia.com/gpu"] // "none")] | @tsv'
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

Commands are read-only; protect cluster identifiers and output. Listing an offering or SKU does not prove deployability.

## 5. GPU driver and runtime acceptance

For each node pool, record **one accountable NVIDIA driver owner**: approved AMI/node image/managed installer **or** NVIDIA GPU Operator driver operand, not competing managers. Pin tested OS/kernel, driver, CUDA, NCCL, device plugin and container runtime versions. Validate host GPU UUID, GPU memory, Xid/ECC, device-plugin registration, `nvidia.com/gpu` allocatable and an immutable-image CUDA canary. Where applicable, run `nccl-tests` with the intended rank count and topology. AWS EFA and Azure InfiniBand/RDMA have different drivers and support matrices; never assume cross-cloud interchangeability.

On authorized disposable GPU nodes:

```bash
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version,memory.total --format=csv
nvidia-smi topo -m
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|ECC' | tail -n 80
```

Do not run disruptive benchmarks or driver upgrades on production without an approved change.

## 6. Identity, isolation and data protection

AWS: prefer EKS Pod Identity/IRSA and least-privilege S3/KMS access; Azure: Entra Workload ID or managed identity with appropriate Blob data-plane RBAC and Key Vault permissions. Node identity must not grant every tenant broad dataset access. Require signed/pinned images, protected CI, GPU pod admission, restricted host access, namespace RBAC, ResourceQuota, network egress policy, private storage endpoints, encryption and audit retention. GPU time-slicing is not equivalent to strong hardware tenant isolation.

## 7. Storage and checkpoint acceptance

Choose durable S3/Blob or approved parallel filesystem for checkpoint artifacts; local NVMe is disposable scratch. For each checkpoint capture job ID, model revision, step, object version/hash, key reference, UTC time, storage location and retention. Verify integrity and perform a real restore in staging. Measure both RPO and RTO. Check key availability, data residency and cross-cloud transfer cost before declaring DR readiness.

## 8. Scaling, reliability and FinOps

Validate queue admission, autoscaler bounds, cold-start GPU-ready latency, pod IP headroom, topology constraints, zone loss and N-1. Spot requires interruption handling and tested checkpoint recovery. Track GPU idle-hours, utilization, queue age, GPU-hour per completed job, cost per successful job, storage and egress, reservation commitments and forecast error. Set budget limits and owner tags before scaling.

## 9. Fifteen acceptance scenarios

| # | Scenario | Required evidence |
|---|---|---|
| 01 | AWS correct accelerated vCPU quota | limit, usage, fleet demand |
| 02 | Azure family and regional quotas | both limits, usage, demand |
| 03 | AWS/Azure SKU eligibility | region, zone, restrictions |
| 04 | Physical capacity readiness | staging launch or eligible reservation evidence |
| 05 | N-1 GPU resilience | largest failure-domain loss and recovery |
| 06 | Node bootstrap | Ready, driver, GPU UUID |
| 07 | GPU Operator/device plugin | allocatable and runtime |
| 08 | CUDA/NCCL canary | versions, correctness, throughput |
| 09 | EFA/RDMA where required | supported topology and tests |
| 10 | Tenant isolation | workload identity, RBAC, admission |
| 11 | Storage encryption | key access, private network |
| 12 | Checkpoint restore | checksum, RPO and RTO |
| 13 | Autoscaler and upgrade | cold-start, PDB, rollback |
| 14 | GPU SLO and telemetry | DCGM, errors, alert owner |
| 15 | FinOps and change governance | unit cost, budget, approval |

A failed **critical** scenario blocks production regardless of overall score.

## 10. Twelve troubleshooting cases

1. AWS quota sufficient but EC2 GPU launch returns insufficient capacity.
2. Azure family quota exhausted while regional quota remains.
3. Azure region quota exhausted while family quota remains.
4. GPU VM SKU not eligible in target AZ.
5. GPU node Ready but NVIDIA device plugin reports zero resources.
6. GPU Operator conflicts with image-managed driver.
7. CUDA initialization fails after node image upgrade.
8. NCCL collective hangs due to incompatible EFA/RDMA topology.
9. CNI/subnet IP exhaustion prevents GPU pods.
10. Checkpoint appears complete but cannot decrypt or restore.
11. Spot eviction exceeds recovery SLO.
12. GPU idle-hour spend breaches the approved budget.

## 11. Ten production-style game days

**GD-01:** AWS quota exhaustion during inference surge. **GD-02:** Azure GPU family quota denial. **GD-03:** AZ capacity loss and N-1 failure. **GD-04:** driver regression during node upgrade. **GD-05:** device plugin crash and pending workloads. **GD-06:** NCCL/EFA/RDMA degradation. **GD-07:** dataset throughput collapse. **GD-08:** checkpoint/key recovery failure. **GD-09:** workload identity or egress policy breach. **GD-10:** Spot eviction and runaway GPU costs.

For every game day, collect incident commander, UTC timeline, blast radius, workload SLO, containment, rollback, checkpoint state, owner and RCA/COA. Execute disruptive scenarios only in authorized staging.

## 12. Eight production runbooks

**RB-042-A — Quota and capacity:** identify exact quota bucket, SKU/AZ offering, physical capacity and approved fallback.

**RB-042-B — GPU node recovery:** inspect VM/VMSS, kernel, NVIDIA driver, GPU UUID, Xid and node replacement.

**RB-042-C — Operator and device plugin:** trace driver -> toolkit/CDI -> plugin -> kubelet allocatable -> CUDA canary.

**RB-042-D — Network and NCCL:** verify EFA/RDMA SKU support, NIC drivers, policy, topology and collective test.

**RB-042-E — Storage/DR:** validate durable checkpoint, key, hash, restore RPO/RTO and consistency.

**RB-042-F — Security:** contain compromised pod, revoke scoped identity, audit dataset access and preserve evidence.

**RB-042-G — Scaling and cost:** cap autoscaler, assess idle GPUs, owner tags, budget and SLO impact.

**RB-042-H — Change acceptance:** review evidence, rollback, approvers, unresolved risks and operational handoff.

## 13. Eight hands-on labs

1. Capture AWS and Azure GPU capacity, quota and SKU inventories.
2. Build a workload-equivalent GPU performance and cost comparison.
3. Validate node images, driver ownership, device plugins and CUDA canaries.
4. Verify GPU scheduling, autoscaler, tenant identity and network policy.
5. Test approved EFA or RDMA/NCCL where the GPU SKU supports it.
6. Restore encrypted checkpoint from durable storage and measure RPO/RTO.
7. Run all ten synthetic incident injections using the companion offline gate.
8. Complete the 100-point scorecard and publish redacted readiness evidence.

**Companion:** [Lab 042 — AWS and Azure GPU Operations Capstone Gate](../labs/042-aws-azure-gpu-operations-capstone-check.md). The offline evaluator checks attested evidence; it is not a substitute for live acceptance.

## 14. Final 100-point acceptance rubric

| Category | Points |
|---|---:|
| AWS/Azure GPU architecture, quotas and capacity | 20 |
| Driver, CUDA/NCCL, network and workload correctness | 20 |
| Identity, tenant security, storage and checkpoint | 20 |
| SLO, scaling, resilience, incident response and FinOps | 20 |
| Automated tests, live staging evidence and signoff | 20 |
| **Total** | **100** |

**Pass threshold:** >=90 points, zero critical blockers, evidence no older than the agreed change window, named owner and independently approved rollback. A synthetic passing gate alone does not establish production readiness.

## 15. Evidence template

```yaml
assessment:
  chapter: 042
  environment: approved-staging
  aws_account_alias: redacted
  azure_subscription_alias: redacted
  workload: approved-inference-and-training
  timestamp_utc: REPLACE
  quota_evidence: REPLACE
  capacity_evidence: REPLACE
  driver_cuda_nccl_matrix: REPLACE
  gpu_canary_evidence: REPLACE
  network_evidence: REPLACE
  identity_and_storage_evidence: REPLACE
  checkpoint_restore_evidence: REPLACE
  slo_and_finops_evidence: REPLACE
  incident_drills: REPLACE
  score: REPLACE
  critical_blockers: REPLACE
  change_owner: REPLACE
  security_reviewer: REPLACE
  sre_reviewer: REPLACE
  rollback_plan: REPLACE
```

## References

- [AWS EC2 Service Quotas](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-resource-limits.html)
- [AWS EKS GPU workloads](https://docs.aws.amazon.com/eks/latest/userguide/ml-eks-k8s-device-plugin.html)
- [AWS EFA](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)
- [Azure GPU VM sizes](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes-gpu)
- [Azure VM quotas](https://learn.microsoft.com/en-us/azure/quotas/per-vm-quota-requests)
- [AKS GPU workloads](https://learn.microsoft.com/en-us/azure/aks/gpu-cluster)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [Kubernetes resource management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

**Part 05 complete (8/8 chapters). Next: Chapter 043 — AI Cluster Network Architecture, beginning Part 06 — High Performance Networking.**
