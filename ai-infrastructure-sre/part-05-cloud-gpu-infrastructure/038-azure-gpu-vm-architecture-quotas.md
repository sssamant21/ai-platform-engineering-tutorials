# Chapter 038 — Azure GPU VM Architecture and Quotas

**AI Infrastructure Engineering & SRE | Part 05 — Cloud GPU Infrastructure | Chapter 038/100**

**Publication:** Documentation published; executable offline tests, Azure GPU staging, independent security review and production acceptance must be verified separately.

## Learning outcomes

Architect, size, secure and operate Microsoft Azure GPU virtual machines for inference, fine-tuning and distributed training. Distinguish GPU SKU availability from quota and real allocatable capacity, model family-specific vCPU quotas, region/zones, reservations and Spot, GPU driver/CUDA compatibility, accelerated networking, RDMA/InfiniBand when supported, storage, identity, observability, cost and recovery. Perform read-only Azure discovery, synthetic admission tests, staging canaries and evidence-based go/no-go reviews.

## 1. Enterprise reference architecture

```text
AI workload requirements (GPU VRAM, CUDA, NCCL, RPO/RTO, SLO)
         |
subscription + region + GPU VM family quota + SKU restrictions
         |
VNet private subnet / NSG / private endpoints / managed identity
         |
GPU Azure VM / VM Scale Set / AKS GPU node pool
    |               |                  |
GPU + host CPU   NIC/RDMA         managed disk / local NVMe
    |               |                  |
NVIDIA driver -> CUDA/NCCL -> model or training service
         |
Azure Monitor / VM Insights / DCGM / app traces
         |
Blob / Azure Managed Lustre / approved storage checkpoint
         |
quota governance, scaling, incident response and FinOps
```

**Important distinctions:** A VM SKU listed in a region is not guaranteed to be purchasable for a specific subscription, zone, offer or current capacity. Approved quota is not a capacity reservation. VM Running does not establish driver health, CUDA compatibility, RDMA connectivity or model correctness.

## 2. Azure GPU VM families and workload fit

Azure GPU SKUs span NC (compute/training), ND (distributed AI/training), NV (visualization and inference in some generations) and other specialized offerings. Naming, GPU generation, CPU/VRAM ratios, local storage and network features vary materially. **Do not assume all NC/ND/NV SKUs support InfiniBand, NVLink, MIG or identical CUDA versions.** Verify current Azure VM sizes, region restrictions and the exact model's GPU count, VRAM, vCPUs, NIC/RDMA capabilities and OS image support.

| Workload | Sizing emphasis | Failure mode |
|---|---|---|
| Low-latency inference | VRAM, concurrency, p95/p99, scale-up time | GPU OOM or cold-start SLO |
| Fine-tuning | GPU memory, host RAM, dataset I/O | insufficient memory/checkpoint throughput |
| Multi-node training | GPU interconnect, RDMA/InfiniBand eligibility, topology | NCCL hangs/low bus bandwidth |
| Batch embeddings | useful GPU utilization and cost/job | overprovisioning |
| High-availability inference | multi-zone eligible SKUs, N-1 | regional or zone capacity unavailable |

## 3. Quota model: total regional vCPU versus family quota

Azure commonly enforces **Total Regional vCPUs** and **VM-family vCPU quotas**, with additional restrictions for Spot, subscription offer, VM SKU and deployment model. A deployment must satisfy all applicable limits. GPU device count is not a quota unit. Some GPU families have distinct quotas and may require support requests.

Example (illustrative only): a proposed fleet of 3 VMs × 96 vCPUs = 288 vCPUs. If family quota remaining is 192 and total regional quota remaining is 400, the family quota blocks the request. If both quotas are sufficient, SKU availability, zone placement, subscription restrictions and physical capacity can still block deployment.

Quota governance workflow:
1. Identify tenant/subscription, approved region, SKU, zone and purchase model.
2. Determine actual SKU vCPU count from Azure, not a hardcoded assumption.
3. Inventory **both** applicable regional and VM-family quota limits/current usage.
4. Calculate existing usage + requested + replacement/N-1 headroom.
5. Validate SKU restrictions and zone offerings for the subscription.
6. Request quota increase through approved Azure portal/Quota API workflow; record case ID and lead time.
7. Validate physical capacity separately via approved staging or supported reservation mechanisms.
8. Apply cost budget, owner tags, queue admission and rollback.

## 4. Read-only Azure CLI discovery

Prerequisites: Azure CLI, approved least-privilege subscription access and selected context. Never paste credentials or raw access tokens into a public issue.

```bash
az account show --query '{subscription:name,id:id,tenant:tenantId}' -o json
az account set --subscription APPROVED_SUBSCRIPTION_ID
az vm list-skus --location eastus --resource-type virtualMachines \
  --all --query "[?contains(name, 'Standard_N')].{Name:name,Restrictions:restrictions,Capabilities:capabilities}" -o json
az vm list-usage --location eastus \
  --query "[].{Name:name.localizedValue,Current:currentValue,Limit:limit}" -o table
az vm list-sizes --location eastus -o table
az vm list --show-details --query "[].{Name:name,Location:location,Size:hardwareProfile.vmSize,State:powerState}" -o table
```

`az vm list-skus --all` can be large and may include restricted SKUs; inspect the returned `restrictions` field. Some CLI usage views omit or aggregate relevant quota families; cross-check with the Azure Quotas service/portal and subscription-specific SKU APIs. Do not treat a global list as proof of zone or subscription deployability.

## 5. Capacity procurement and availability zones

Compare pay-as-you-go, Azure Spot VMs, eligible VM reservations and Capacity Reservations (where supported for the specific GPU SKU and location). **Billing reservations do not necessarily reserve physical GPU capacity.** Spot has eviction risk and may have different quota and capacity behavior. Zone-redundant architecture requires that the desired GPU SKU is actually available in each target zone; a VM may need a different compatible GPU type as fallback, which requires driver/model retesting.

For training, preserve checkpoints before eviction; for inference, use availability, warm capacity, and workload SLOs. Document alternative regions, data residency, storage access and network cost before failover.

## 6. GPU driver and runtime lifecycle

Use an approved Azure GPU-compatible OS image and one accountable driver owner (VM extension, image pipeline or platform automation). For AKS, driver ownership may instead be managed by the AKS GPU node image or NVIDIA GPU Operator; do not double-manage. Pin tested OS/kernel, driver, CUDA, NCCL, container runtime and application versions. Verify:

```bash
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version,memory.total --format=csv
nvidia-smi topo -m
lsmod | grep nvidia
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|ECC' | tail -n 80
```

Host GPU discovery is not proof that a container can allocate GPUs or that CUDA and NCCL run correctly.

## 7. Azure GPU networking

Verify Accelerated Networking support for the selected SKU and host image. For distributed GPU training, confirm whether the SKU provides InfiniBand/RDMA, required drivers and network topology; do not assume all GPU VMs do. Validate NSG rules, VNet/subnet IP headroom, proximity placement where supported, DNS, private endpoints, and actual NCCL transport. EFA is AWS-specific and should not be presented as an Azure networking option.

Measure `nccl-tests` all-reduce on authorized staging and correlate network counters, GPU utilization and application training step time. Some Azure GPU/HPC images bundle RDMA drivers; ensure version compatibility rather than blindly installing conflicting drivers.

## 8. Storage and checkpointing

Choose managed disks for OS/persistent block storage, local ephemeral NVMe for disposable scratch, Blob Storage for durable objects and approved parallel filesystem services where supported. Measure real read/write throughput, latency, dataset cache hit rate, checkpoint completion and restore time. Local temporary disks do **not** satisfy durable checkpoint requirements. Use managed identity, RBAC, private endpoints and encryption.

## 9. Identity and security

Use Microsoft Entra identity, managed identities for workload storage access, least-privilege Azure RBAC, NSGs, private networking, disk encryption, Key Vault and Defender for Cloud where approved. Avoid VM-local service-principal secrets, public SSH/RDP and permissive inbound rules. Require patching, secure image provenance, vulnerability review, activity logs and break-glass procedures.

## 10. Observability and FinOps

| Signal | Acceptance purpose |
|---|---|
| Azure VM provisioning and quota failures | capacity readiness |
| GPU UUID, Xid/ECC, driver version | hardware/runtime health |
| DCGM GPU utilization, memory, temperature | utilization and faults |
| InfiniBand/RDMA or NIC errors | training communication |
| p95/p99 inference or step time | customer SLO |
| checkpoint age and restore RTO | recoverability |
| VM SKU, region/zone, quota headroom | scaling and resilience |
| idle GPU-hours, storage and egress | cost attribution |
| telemetry freshness, alert ownership | on-call readiness |

GPU cost should include VM hours, disks, network, storage, monitoring and capacity commitments. Low GPU utilization can be justified by SLO headroom; compare against agreed workload and reliability targets.

## 11. Twelve troubleshooting scenarios

1. VM-family vCPU quota exhausted.
2. Total regional vCPU quota exhausted.
3. SKU restricted for subscription or region.
4. VM SKU available in region but unavailable in requested zone.
5. Quota sufficient but Azure physical capacity unavailable.
6. VM boots without a working NVIDIA driver.
7. CUDA runtime incompatible with driver.
8. InfiniBand/RDMA unsupported or unconfigured.
9. NCCL collective stalls after image update.
10. Managed disk/Blob throughput blocks checkpoints.
11. Spot eviction occurs before checkpoint completion.
12. GPU VM idle hours exceed budget or observability misses Xid.

## 12. Ten incident simulations

1. Regional GPU-family quota breach.
2. Cross-team regional quota contention.
3. AZ capacity shortage during inference burst.
4. Driver extension regression.
5. CUDA image mismatch on newly provisioned VM.
6. RDMA driver/NSG regression breaks NCCL.
7. Dataset storage throttling stalls training.
8. Spot eviction without durable checkpoint.
9. GPU ECC/Xid triggers model failures.
10. Cost overrun caused by orphaned GPU VMs.

Capture UTC timeline, subscription alias, region/zone, VM SKU, driver/CUDA versions, GPU UUID, workload SLO, checkpoint, owner, containment, rollback and RCA/COA.

## 13. Eight operational runbooks

**RB-038-A — GPU VM sizing:** validate VRAM, GPU count, vCPUs, networking, storage and SLO.

**RB-038-B — Azure quota:** identify family/regional limits, consumption, request ID and fallback.

**RB-038-C — Capacity failure:** inspect SKU restrictions, zone alternatives and reservation eligibility.

**RB-038-D — Driver/CUDA:** inspect image, extension, kernel, NVIDIA driver and canary.

**RB-038-E — RDMA/NCCL:** verify SKU support, drivers, NSG, rank mapping and benchmark.

**RB-038-F — Storage/checkpoint:** validate durable storage, throughput, RPO/RTO and restore.

**RB-038-G — Spot recovery:** respond to eviction signals, restart training and verify consistency.

**RB-038-H — Security/FinOps/RCA:** verify identity, private access, budget, SLO and signoff.

## 14. Hands-on labs

1. Inventory Azure subscription, region, GPU SKU restrictions and applicable quota families.
2. Calculate fleet vCPU demand, family/regional headroom and N-1 replacement.
3. Verify SKU GPU/VRAM, zone and RDMA capabilities against workload requirements.
4. Review an approved secure GPU VM IaC design with managed identity, encryption and private networking.
5. On disposable staging capacity, validate driver/CUDA, GPU UUID and NCCL as applicable.
6. Benchmark durable checkpoint storage and recovery.
7. Simulate quota, Spot, driver and RDMA failures offline.
8. Produce redacted evidence, cost model, SLO and approval signoff.

[Lab 038 — Azure GPU VM Architecture and Quotas Gate](../labs/038-azure-gpu-vm-architecture-quotas-check.md) contains a fail-closed offline Python 3.11+ policy evaluator and pytest suite. It does not query or modify Azure.

## 15. 100-point production acceptance

| Category | Points |
|---|---:|
| Azure GPU VM sizing and architecture | 20 |
| Quotas, regional/zone capacity and reservations | 20 |
| Driver/CUDA, security, RDMA and storage | 20 |
| Reliability, incidents, checkpoints and FinOps | 20 |
| Automated tests, staging evidence and signoff | 20 |
| **Total** | **100** |

Pass >=90 with no critical GPU correctness, identity, quota/capacity, checkpoint, observability or rollback blocker.

## References

- [Azure GPU optimized virtual machine sizes](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes-gpu)
- [Azure VM quotas](https://learn.microsoft.com/en-us/azure/quotas/per-vm-quota-requests)
- [Azure VM SKU availability](https://learn.microsoft.com/en-us/azure/virtual-machines/quotas)
- [Azure Spot Virtual Machines](https://learn.microsoft.com/en-us/azure/virtual-machines/spot-vms)
- [Azure Capacity Reservation](https://learn.microsoft.com/en-us/azure/virtual-machines/capacity-reservation-overview)
- [Azure GPU driver setup](https://learn.microsoft.com/en-us/azure/virtual-machines/extensions/hpccompute-gpu-windows)
- [Azure HPC networking](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes-hpc)

**Next: Chapter 039 — AKS GPU Node Pools and Drivers.**
