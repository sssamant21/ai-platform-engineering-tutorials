# Chapter 039 — AKS GPU Node Pools and Drivers

**AI Infrastructure Engineering & SRE | Part 05 — Cloud GPU Infrastructure | Chapter 039/100**

**Status:** Published tutorial and offline lab. Live AKS/GPU testing, security review and production acceptance remain separate activities.

## Learning objectives

Design, provision, secure, observe, upgrade and troubleshoot Azure Kubernetes Service (AKS) GPU node pools. Understand Azure VM SKU/quota/zone constraints, AKS system versus user pools, driver installation ownership, NVIDIA GPU Operator and device plugin, kubelet extended resources, taints/affinity, Azure CNI/IP allocation, workload identity, autoscaling, node image upgrades, disruption, checkpoint recovery and FinOps.

## 1. Enterprise reference architecture

```text
Azure subscription / regional GPU family + total vCPU quotas
             |
AKS control plane / Azure RBAC / managed identity
             |
AKS GPU user node pool -> VM Scale Set -> GPU Azure VM
             |                              |
      Azure CNI / NSG                 OS image / kernel
             |                              |
Kubernetes scheduler             NVIDIA driver ownership
             |                              |
GPU pod requests                 toolkit/CDI + device plugin
  nvidia.com/gpu                          |
             +---------- kubelet GPU allocatable
                          |
                    CUDA / NCCL / workload
                          |
             DCGM / Azure Monitor / SLO / checkpoint
                          |
                  drain / rollback / RCA
```

**Boundaries:** An AKS node being Ready is not proof that GPUs are allocatable or CUDA works. Azure VM quota is not physical capacity. AKS system pools and user GPU pools serve different purposes; keep critical system services on appropriately sized system node pools unless architecture explicitly requires otherwise.

## 2. GPU node pool architecture

| Decision | Enterprise guidance |
|---|---|
| Pool mode | use a dedicated **User** GPU node pool for application workloads |
| VM size | validate actual GPU SKU, vCPU/VRAM, region and zone |
| Scaling | set min/max/desired and choose supported AKS Cluster Autoscaler |
| Driver | choose exactly one supported driver owner |
| Runtime | containerd, NVIDIA toolkit/CDI and device plugin |
| Scheduling | GPU requests, taints, tolerations and node affinity |
| Identity | managed identity and Microsoft Entra Workload ID |
| Networking | Azure CNI mode, pod IP headroom, NSGs and private endpoints |
| Reliability | zone eligibility, N-1, PDB, checkpoints, upgrade surge |
| Observability | DCGM, node events, Azure Monitor, application SLO |

AKS GPU image behavior and default driver installation are version-, image- and SKU-sensitive. Validate the selected AKS release, node OS image, `--gpu-driver` options if supported, and NVIDIA GPU Operator integration in current Azure documentation. **Do not run two independent NVIDIA driver managers** on one node.

## 3. GPU driver and Operator ownership

Possible supported patterns include AKS-managed GPU driver installation, approved custom image/bootstrap, or NVIDIA GPU Operator-managed driver/runtime operands. Exact compatibility depends on AKS version, VM SKU, OS and Operator release. When choosing Operator-managed drivers, disable competing AKS driver installation using the supported AKS mechanism for that release; verify before deploying. Operator can also manage only selected operands while drivers remain image-managed.

Readiness chain: VM provisioned -> kubelet Ready -> NVIDIA driver loaded -> toolkit/CDI compatible -> device plugin registered -> GPU resource allocatable -> CUDA canary -> model SLO. Record host `nvidia-smi`, driver version, device-plugin logs and GPU UUID.

## 4. Read-only Azure and Kubernetes discovery

Use a least-privilege Azure role and approved Kubernetes context:

```bash
export AZ_RESOURCE_GROUP=approved-rg
export AKS_CLUSTER=approved-aks
az account show --query '{subscription:name,id:id}' -o json
az aks show --resource-group "$AZ_RESOURCE_GROUP" --name "$AKS_CLUSTER" \
  --query '{Name:name,Version:kubernetesVersion,NodeResourceGroup:nodeResourceGroup,Network:networkProfile}'
az aks nodepool list --resource-group "$AZ_RESOURCE_GROUP" --cluster-name "$AKS_CLUSTER" \
  --query '[].{Name:name,Mode:mode,VMSize:vmSize,Count:count,Min:minCount,Max:maxCount,Autoscaling:enableAutoScaling,State:provisioningState,OS:osSKU}' -o table
az vm list-usage --location eastus -o table
az vm list-skus --location eastus --resource-type virtualMachines \
  --all --query "[?contains(name, 'Standard_N')].{Name:name,Restrictions:restrictions}" -o json
kubectl config current-context
kubectl get nodes -L agentpool,topology.kubernetes.io/zone
kubectl get nodes -o json | jq -r '.items[] | [.metadata.name, (.status.allocatable["nvidia.com/gpu"] // "none")] | @tsv'
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

`--all` includes restricted SKUs. A SKU listed in a region is not proof of capacity or subscription eligibility. Check both family and regional vCPU quotas. Avoid exposing resource IDs, kubeconfig or sensitive logs in public tickets.

## 5. Review-only AKS GPU pool command

The following is an **illustrative staging command**. It would provision billable resources if executed; do not run without change approval, quota/cost review and verified CLI compatibility.

```bash
# REVIEW ONLY — DO NOT RUN WITHOUT APPROVAL
az aks nodepool add \
  --resource-group APPROVED_RG \
  --cluster-name APPROVED_AKS \
  --name gpustage \
  --mode User \
  --node-vm-size Standard_NC24ads_A100_v4 \
  --node-count 1 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 2 \
  --node-taints dedicated=gpu:NoSchedule
```

Choose driver management, zones, encryption, identity, image, subnet and network controls through a reviewed IaC module. Command flags may vary with AKS CLI versions; confirm against current Azure documentation. Do not assume a GPU VM SKU is offered in the chosen region/zone.

## 6. GPU workload scheduling and tenant isolation

GPU pods typically request and limit `nvidia.com/gpu` as an integer extended resource, with appropriate tolerations for the dedicated GPU taint and node affinity. MIG resource keys vary by profile and device plugin configuration. Namespace ResourceQuota, RBAC, Pod Security Admission, network policy and Workload Identity are independent controls. Time-slicing is not equivalent to hardware isolation. Avoid privileged GPU tenant containers or arbitrary hostPath mounts.

Example scheduling fragment:

```yaml
spec:
  nodeSelector:
    accelerator: approved-gpu-pool
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu
      effect: NoSchedule
  containers:
    - name: inference
      image: registry.example.invalid/inference@sha256:REPLACE_APPROVED_DIGEST
      resources:
        limits:
          nvidia.com/gpu: 1
```

The node label and image digest are placeholders; this fragment is not a deployable Pod. Verify GPU extended-resource accounting and quota behavior in disposable staging.

## 7. Autoscaling and capacity admission

AKS Cluster Autoscaler can adjust node pool capacity within configured bounds but remains constrained by GPU family/regional vCPU quotas, subscription SKU restrictions, zone capacity, subnet IPs, maximum pods per node and bootstrap latency. Scale-from-zero depends on the scheduler/autoscaler being able to infer node labels, taints and GPU resources. Do not use AWS Karpenter NodePool/EC2NodeClass procedures for AKS. Measure Pending-to-CUDA-ready latency, not only VM provisioning time.

## 8. Azure networking and distributed training

Inspect Azure CNI overlay versus traditional pod subnet design, VNet/subnet sizing, NSGs, DNS, private endpoints, outbound connectivity and service mesh overhead. GPU/HPC SKU RDMA/InfiniBand support is not universal; validate selected SKU and Azure-supported driver/network plugin integration. Multi-node NCCL requires compatible topology, drivers, rank coordination and storage; success of single-node `nvidia-smi` does not validate distributed collectives.

## 9. Safe node image and Kubernetes upgrades

Before a node pool upgrade: validate AKS version skew, driver/CUDA/Operator compatibility, PDB, upgrade surge, regional GPU quota, zone capacity, checkpoint RPO/RTO, N-1 headroom, SLO and rollback. A node image upgrade can change the kernel and invalidate a driver module. Prefer a parallel canary GPU pool and tested workload migration. Do not assume an AKS node image rollback is an in-place revert; recovery may require a replacement pool and workload rescheduling.

## 10. Observability

| Signal | Operational use |
|---|---|
| node pool provisioningState / health | VMSS and upgrade failure |
| kubelet Ready and GPU allocatable | GPU scheduling readiness |
| driver/toolkit/device plugin/Operator | runtime compatibility |
| DCGM GPU Xid/ECC, utilization and memory | GPU health |
| Azure CNI available IPs and pod Pending age | networking and scale |
| CUDA/NCCL canary | workload correctness |
| inference p95/p99, training step time | customer SLO |
| checkpoint age and restore RTO | resilience |
| GPU VM hours, idle GPUs and cost/job | FinOps |
| zone spread, N-1 and telemetry freshness | platform reliability |

## 11. Twelve troubleshooting scenarios

1. GPU VM family quota exhausted.
2. SKU unavailable in selected region or zone.
3. AKS GPU user node pool provisioning fails.
4. Node Ready but driver missing or incompatible.
5. AKS driver conflicts with NVIDIA GPU Operator.
6. Device plugin advertises zero `nvidia.com/gpu`.
7. CDI/containerd allocation fails.
8. GPU workload Pending due to taints/affinity/quota.
9. Autoscaler cannot scale GPU pool from zero.
10. Azure CNI IP exhaustion blocks pod creation.
11. Node image upgrade triggers CUDA regression.
12. NCCL training hangs or checkpoint restore fails.

## 12. Ten incident simulations

1. GPU quota prevents scale-out.
2. GPU SKU capacity unavailable in a zone.
3. VMSS provisioning stalls.
4. Driver management ownership conflict.
5. Device plugin crash after AKS upgrade.
6. CNI IP exhaustion.
7. GPU pool autoscaler permission/config regression.
8. Spot node eviction races checkpoint.
9. GPU node image update violates N-1.
10. Unused GPU pool breaches budget.

For each, record UTC timeline, cluster/pool, VM SKU, node image/driver/toolkit versions, GPU UUID, pod events, application SLO, checkpoint, mitigation, rollback and RCA/COA.

## 13. Eight runbooks

**RB-039-A — Pool discovery:** inspect AKS version, user pool, VMSS health, SKU and zones.

**RB-039-B — Quota/capacity:** verify regional/family vCPU quotas, SKU restrictions and physical availability.

**RB-039-C — Driver lifecycle:** establish AKS versus Operator ownership, module, toolkit/CDI and CUDA.

**RB-039-D — Device plugin:** trace GPU hardware -> driver -> plugin -> kubelet allocatable.

**RB-039-E — Scheduling/scale:** check taints, affinity, quota, autoscaler, subnet IP and GPU bootstrap.

**RB-039-F — Security:** validate Entra Workload ID, RBAC, PSA, network policy and secrets handling.

**RB-039-G — Safe upgrade/recovery:** checkpoint, N-1, PDB, canary pool, rollback and SLO.

**RB-039-H — RCA/FinOps:** confirm GPU health, cost attribution, telemetry and owner signoff.

## 14. Hands-on labs

1. Inventory approved AKS GPU pools, VM SKUs, quota and zones.
2. Review driver ownership and pinned AKS/Operator compatibility matrix.
3. Inspect device-plugin and node GPU allocatable resources.
4. Validate GPU pod scheduling, tenant quota and restricted security.
5. Exercise a synthetic driver conflict, quota shortage and CNI exhaustion offline.
6. Perform a disposable CUDA canary on approved staging GPU pool.
7. Simulate a controlled node pool upgrade and checkpoint/rollback plan.
8. Produce redacted evidence, SLO, cost and signoff.

[Lab 039 — AKS GPU Node Pools and Drivers Gate](../labs/039-aks-gpu-node-pools-drivers-check.md) includes an offline Python 3.11+ validator and pytest suite. It does not deploy Azure resources.

## 15. 100-point assessment

| Category | Points |
|---|---:|
| AKS GPU node pool architecture and quotas | 20 |
| Driver, toolkit, device plugin and CUDA | 20 |
| Scheduling, security, autoscaling and networking | 20 |
| Upgrades, incidents, checkpoints and FinOps | 20 |
| Automated tests and staging evidence | 20 |
| **Total** | **100** |

Pass >=90 with no critical GPU correctness, tenant security, quota/capacity, checkpoint, telemetry or rollback blocker.

## References

- [AKS GPU workloads](https://learn.microsoft.com/en-us/azure/aks/gpu-cluster)
- [AKS node pools](https://learn.microsoft.com/en-us/azure/aks/use-multiple-node-pools)
- [AKS Cluster Autoscaler](https://learn.microsoft.com/en-us/azure/aks/cluster-autoscaler)
- [AKS node image upgrades](https://learn.microsoft.com/en-us/azure/aks/node-image-upgrade)
- [AKS Workload Identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)
- [Azure VM quotas](https://learn.microsoft.com/en-us/azure/quotas/per-vm-quota-requests)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)

**Next: Chapter 040 — Cloud GPU Identity, Storage and Security.**
