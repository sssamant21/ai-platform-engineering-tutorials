# Chapter 041 — Multi-Cloud GPU Capacity and Cost

**AI Infrastructure Engineering & SRE | Part 05 — Cloud GPU Infrastructure | Chapter 041/100**

**Status:** Documentation published. Synthetic lab execution, live AWS/Azure quota verification, pricing validation and operational signoff are tracked separately.

## Learning objectives

Build a defensible multi-cloud GPU capacity plan across AWS EC2/EKS and Azure VM/AKS, incorporating GPU and memory fit, quota headroom, physical availability, availability-zone resilience, lead time, utilization, Spot interruption, reservations, network and storage costs, and service-level objectives. Implement admission control, forecast scenarios, FinOps showback and incident runbooks.

## 1. Architecture and decision flow

```text
Forecast: inference QPS / model VRAM / training GPU-hours / SLA
                 |
      normalize workload requirements
                 |
AWS EC2/EKS                 Azure VM/AKS
quota bucket + AZ           regional + family vCPU quotas
offering + capacity         SKU restriction + zone capacity
                 \         /
            candidate comparison
             |      |      |
        reliability cost security/data residency
                 |
       admission / reservation / queue
                 |
       provision / observe / allocate cost
                 |
     forecast error, RCA, re-plan, rollback
```

**Critical distinction:** Quota is an administrative limit; an instance-type offering is an eligibility signal; neither guarantees physical capacity. Reserved pricing, capacity reservations and scheduled GPU capacity are different products with different commitments. Compare only verified eligible SKUs and workload-equivalent performance.

## 2. Normalize capacity and demand

A cross-cloud comparison must begin with workload-level requirements, not hourly VM price alone:

- Inference: model VRAM footprint, quantization, batch size, p95/p99 latency, tokens/sec, peak concurrency, availability SLO, cold-start and warm capacity.
- Training: GPU-memory requirement, CUDA/NCCL support, ranks, all-reduce topology, dataset locality, checkpoint interval, deadline and restart penalty.
- Platform: CPU/memory, container image and driver support, EFA versus Azure RDMA/InfiniBand, persistent throughput, network isolation and compliance.
- Reliability: availability zones, N-1 spare GPU units, replacement lead time, RPO/RTO, interruption tolerance and tested rollback.

GPU models with different generations, memory bandwidth, interconnects and software stacks are **not interchangeable merely because GPU count matches**. Benchmark representative throughput and correctness before substituting SKUs.

## 3. Quota and capacity matrix

| Dimension | AWS | Azure |
|---|---|---|
| Administrative limit | relevant EC2 On-Demand or Spot vCPU quota bucket | total regional and VM-family vCPU quotas |
| Instance offering | instance-type offerings by region/AZ | subscription SKU restrictions and zone offerings |
| Physical availability | independent EC2 capacity fulfillment | independent Azure allocation capacity |
| Procurement | On-Demand, Spot, eligible Capacity Reservations/Blocks | PAYG, Spot, eligible Capacity Reservations |
| GPU runtime | AMI, NVIDIA driver, CUDA, EFA as supported | GPU VM image, NVIDIA driver, RDMA as supported |
| Cluster layer | EKS MNG or Karpenter | AKS GPU user node pools |
| Identity | EKS Pod Identity/IRSA, IAM, KMS | Entra Workload ID, managed identity, Key Vault |
| Durable checkpoint | S3/FSx/EBS as appropriate | Blob/parallel filesystem/managed disks |

Calculate quota demand in **vCPUs** for each applicable bucket. Track quota consumed by other teams, requested new instances, surge/replacement and existing fleet. For Azure check **both** regional and family headroom. For AWS check the **correct purchase-model quota bucket**.

## 4. Capacity forecasting

Let:

- `G_peak` = workload-required GPU units at peak, from measured workload sizing.
- `G_reserve` = separately validated replacement/availability reserve.
- `G_total = G_peak + G_reserve`.
- `N = ceil(G_total / GPUs_per_instance)` for a homogeneous fleet.
- `VCPU_demand = N × vCPUs_per_instance`.
- `Quota_headroom = quota_limit - current_usage - VCPU_demand`.

The simple N-1 rule must be evaluated **per failure domain**: a fleet with N GPUs spread across zones is not necessarily resilient to the loss of its largest zone. Reserve capacity is only credible if the replacement SKU, AZ, quota and launch path are validated. GPU count alone cannot certify topology or application SLO.

### Worked example (synthetic)

A training job requires 16 GPU units and 8 reserve units. A candidate has 8 GPUs and 96 vCPUs per instance. `ceil(24/8) = 3` instances and `3×96 = 288` vCPUs. With 64 vCPUs already consumed and a relevant quota of 320, quota headroom is `320−64−288 = −32` vCPUs. Reject admission or request quota/choose a validated alternative. **These are synthetic inputs, not actual cloud instance specifications.**

## 5. Total cost of ownership

Compare useful work rather than GPU list price:

```text
effective_cost_per_successful_job =
 (compute + idle/warm + storage + checkpoint + data transfer
  + observability + support + reservation commitments
  + expected interruption/restart waste)
 / successful_jobs
```

For inference use cost per 1M successful tokens or cost per SLO-compliant request, with the same prompt/output distribution. For training use cost per successful training run or per achieved validated training objective. Include storage/egress and failure recovery. Avoid comparing spot price directly to a guaranteed-capacity service without interruption modeling.

### Example FinOps allocation labels

```yaml
metadata:
  labels:
    cost-center: ai-platform
    workload: approved-training
    environment: staging
    owner-team: ml-infra
    gpu-cost-pool: shared
```

Enforce tag/label policies through approved IaC and Kubernetes admission. Cost allocation may require cloud-specific billing tags, resource groups and shared-cost distribution beyond Kubernetes labels.

## 6. Purchase model tradeoffs

| Option | Benefit | Risk |
|---|---|---|
| On-Demand / PAYG | flexible start/stop | quota and capacity shortages |
| Spot | lower price when available | eviction, requeue and checkpoint cost |
| Capacity reservation | eligible targeted capacity | cost/terms, region/zone/SKU restrictions |
| Committed discount | potentially lower unit cost | may not reserve GPU physical capacity |
| Cross-cloud fallback | broader supply options | data residency, egress, framework and IAM complexity |

Reservations should be evaluated using actual provider terms and service support. A cross-cloud fallback is **not** operationally ready until the image, model artifacts, networking, checkpoint restore and identity controls have passed staging tests.

## 7. Read-only discovery labs

### AWS

```bash
export AWS_PROFILE=approved-readonly
export AWS_REGION=us-east-1
aws sts get-caller-identity --profile "$AWS_PROFILE"
aws ec2 describe-instance-types --instance-types g5.12xlarge \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'InstanceTypes[].{Type:InstanceType,VCpu:VCpuInfo.DefaultVCpus,GPU:GpuInfo.Gpus}'
aws ec2 describe-instance-type-offerings --location-type availability-zone \
  --filters Name=instance-type,Values=g5.12xlarge \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws service-quotas list-service-quotas --service-code ec2 \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'Quotas[].{Name:QuotaName,Value:Value,Code:QuotaCode}'
```

Confirm the exact EC2 quota bucket; a generic quota list is not a complete availability check.

### Azure

```bash
az account show --query '{Name:name,Id:id}' -o json
az vm list-skus --location eastus --resource-type virtualMachines --all \
  --query "[?contains(name, 'Standard_N')].{Name:name,Restrictions:restrictions}" -o json
az vm list-usage --location eastus \
  --query '[].{Name:name.localizedValue,Current:currentValue,Limit:limit}' -o table
```

Check both total regional and family quotas; SKU restrictions are subscription-sensitive. Neither discovery flow performs a launch or guarantees physical capacity.

## 8. SLOs and telemetry

Track GPU-ready time, Pending-to-Ready p95, GPU utilization and memory, Xid/ECC, NCCL throughput, p95/p99 inference, successful job completion, queue time, checkpoint RPO/RTO, cost per successful job, idle GPU-hours, forecast error and quota headroom. Separate measured values from estimates. Do not auto-provision expensive GPUs solely on a synthetic signal without policy-approved spending limits.

## 9. Twelve troubleshooting scenarios

1. AWS accelerated-instance quota exceeded.
2. Azure GPU-family quota exceeded while regional quota remains.
3. Azure total regional vCPU quota exhausted.
4. GPU SKU eligible but no physical capacity.
5. Region/AZ offers insufficient resilient placement.
6. GPU memory insufficient after model revision.
7. Driver/CUDA/NCCL mismatch on fallback GPU.
8. Spot eviction exceeds checkpoint RPO.
9. GPU node autoscaler overprovisions idle nodes.
10. Egress/parallel filesystem costs exceed compute savings.
11. Cost attribution labels absent or inconsistent.
12. Forecast underestimates peak inference demand.

## 10. Ten incident simulations

1. AWS quota saturation during traffic spike.
2. Azure GPU family quota denied before release.
3. Single-zone GPU capacity loss.
4. Cross-cloud failover blocked by data residency.
5. Spot reclaim interrupts multi-node training.
6. Reservation does not match deployed SKU/AZ.
7. Driver compatibility prevents fallback launch.
8. Idle GPU pool breaches monthly budget.
9. Checkpoint egress delays RTO.
10. Billing attribution drift hides a runaway GPU job.

Capture UTC timeline, provider/account/subscription, SKU, region/zone, quotas, capacity evidence, owner, SLO, checkpoint, cost impact, rollback and RCA/COA.

## 11. Eight runbooks

**RB-041-A — Demand intake:** capture VRAM, GPUs, SLO, topology and forecast confidence.

**RB-041-B — Quota preflight:** verify applicable AWS or Azure buckets and headroom.

**RB-041-C — Capacity shortage:** review offerings, zone placement, eligible reservations and approved alternatives.

**RB-041-D — SKU substitution:** rerun driver/CUDA/NCCL, model performance and checkpoint tests.

**RB-041-E — Spot interruption:** checkpoint, requeue, restore and validate RPO/RTO.

**RB-041-F — Cost anomaly:** isolate owner, GPU idle hours, egress and commitments; contain safely.

**RB-041-G — Multi-cloud failover:** validate data residency, identity, model artifacts, network and rollback.

**RB-041-H — Monthly FinOps review:** reconcile billing, showback, SLO, utilization, forecast error and approval.

## 12. Hands-on lab plan

1. Inventory cloud GPU SKU offerings, region/zone and relevant quotas.
2. Normalize a training and inference workload to a comparable benchmark.
3. Calculate vCPU demand, GPU reserve and zone-specific N-1.
4. Compare total cost per successful job under PAYG/Spot/reservation scenarios.
5. Validate cross-cloud artifact, checkpoint and driver compatibility on disposable staging.
6. Simulate quota shortage, Spot interruption and budget breach offline.
7. Build dashboards for queue, GPU utilization, spend and forecast error.
8. Produce a 100-point readiness assessment and signed capacity decision.

[Lab 041 — Multi-Cloud GPU Capacity and Cost Gate](../labs/041-multi-cloud-gpu-capacity-cost-check.md) provides a Python 3.11+ offline admission evaluator and pytest cases. It does not contact cloud APIs or spend money.

## 13. 100-point assessment

| Domain | Points |
|---|---:|
| GPU sizing and workload normalization | 20 |
| Quotas, offerings, zone capacity and resilience | 20 |
| Cost, procurement and FinOps | 20 |
| Incident recovery, security and SLOs | 20 |
| Tests, staging evidence and approval | 20 |
| **Total** | **100** |

Pass >=90 with no unresolved quota, capacity, security, RPO/RTO, cost or rollback blocker.

## References

- [Amazon EC2 service quotas](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-resource-limits.html)
- [EC2 instance-type offerings](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instance-type-offerings.html)
- [AWS Capacity Reservations](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html)
- [AWS Cost Explorer](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html)
- [Azure VM quotas](https://learn.microsoft.com/en-us/azure/quotas/per-vm-quota-requests)
- [Azure GPU VM sizes](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes-gpu)
- [Azure Cost Management](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/overview-cost-management)
- [Azure Spot VMs](https://learn.microsoft.com/en-us/azure/virtual-machines/spot-vms)

**Next: Chapter 042 — AWS and Azure GPU Operations Capstone.**
