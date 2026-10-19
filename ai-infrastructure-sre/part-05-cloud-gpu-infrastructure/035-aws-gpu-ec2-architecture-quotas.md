# Chapter 035 — AWS GPU EC2 Architecture and Quotas

**AI Infrastructure Engineering & SRE | Part 05 — Cloud GPU Infrastructure | Chapter 035/100**

**Publication status:** Documentation published; offline executable tests, live AWS validation, security review and production acceptance are tracked separately.

## Objectives

Design, size, secure and troubleshoot GPU-backed Amazon EC2 capacity for inference, fine-tuning and distributed training. Understand instance architecture, GPU/NVLink/network/storage tradeoffs, regional capacity, EC2 service quotas, On-Demand/Spot/Capacity Blocks/reservations, AMI and NVIDIA driver lifecycle, IAM, observability, N-1 and FinOps. Deliver executable read-only discovery labs, quota-aware admission checks and an incident-response evidence pack.

## 1. AWS GPU reference architecture

```text
AI workload SLO / GPU count / memory / NCCL topology
         |
capacity design + region/AZ + quota + availability
         |
VPC private subnets + endpoints + SG + IAM + KMS
         |
EC2 GPU fleet (G/P families, generation-specific)
  |           |             |             |
GPU/VRAM   vCPU/RAM    EBS/instance    EFA/ENA/network
  |                         storage
NVIDIA driver + CUDA + NCCL + toolkit
         |
bare EC2 service / Slurm / EKS node group / Karpenter
         |
CloudWatch + DCGM + NVML + SLO + cost allocation
         |
checkpoint S3/FSx, recover, scale and audit
```

**Boundary:** EC2 quota approval does not reserve physical GPU capacity; instance type availability is region/AZ-specific. A running GPU EC2 instance is not evidence that the NVIDIA driver, CUDA or workload is healthy.

## 2. EC2 GPU families and workload fit

| Family / examples | Primary fit | Architecture consideration |
|---|---|---|
| G4dn/G5/G6 and successors | inference, rendering, lighter training | GPU memory and cost per useful inference |
| P4/P5 and successors | high-end training and distributed workloads | interconnect, high bandwidth, large accelerator memory |
| GPU-enabled alternatives | specific workload compatibility | software stack and framework validation |

AWS adds instance families and sizes over time; **never hard-code a claim that a given GPU family is currently offered in every region**. Verify actual `describe-instance-type-offerings`, `describe-instance-types`, official instance pages, pricing and driver support for your chosen region. A GPU instance with multiple devices does not guarantee NVLink, EFA or equivalent GPU peer-to-peer topology.

### Sizing dimensions

- **GPU:** device count, VRAM, compute capability, supported precision, ECC and MIG capabilities where applicable.
- **Host:** vCPU/RAM, NUMA, CPU-to-GPU ratio, PCIe topology, system overhead.
- **Network:** ENA/EFA eligibility, placement groups, security groups, latency, throughput and NCCL.
- **Storage:** EBS throughput/IOPS, instance store lifecycle, FSx for Lustre, S3 staging and checkpoint durability.
- **Software:** AMI/kernel/driver/CUDA/NCCL compatibility, NVIDIA Container Toolkit, DCGM.
- **Reliability:** AZ capacity, quota, replacement lead time, checkpoint RPO/RTO, N-1 and capacity reservations.
- **Cost:** hourly instance rate, idle GPU hours, data transfer, EBS, FSx, CloudWatch, reservations and interruptions.

## 3. Quota model and capacity availability

EC2 GPU instance families can consume **vCPU-based EC2 service quotas** whose names and buckets depend on purchase model and family. Spot and On-Demand have separate quota policies. An increase request is not instant and is not a capacity reservation. A fleet request can still fail with `InsufficientInstanceCapacity` even when quota is sufficient.

Distinguish:

1. **Quota ceiling:** account/region purchase-model vCPU entitlement.
2. **Current quota consumption:** already running/requested eligible instances, including other workloads in the same quota bucket.
3. **Offering:** the instance type is supported in selected AZs.
4. **Physical capacity:** AWS can actually fulfill the launch.
5. **Capacity guarantee/commitment:** the selected reservation product and its terms, region, AZ, start/end, instance type and ownership.
6. **Workload fit:** the allocated instance satisfies GPU, memory, EFA, storage, CUDA and application requirements.

**Example (illustrative, not a current AWS quota):** 4 instances × 96 vCPUs each = 384 vCPUs of demand in the applicable quota bucket. If the regional quota is 320 and 64 are already consumed, remaining headroom is 256; the proposed request exceeds headroom by 128 vCPUs. The correct comparison is quota-bucket vCPUs, **not GPU device count**.

## 4. Read-only AWS discovery lab

Prerequisites: AWS CLI v2, approved SSO/assumed role, `ec2:Describe*` and relevant Service Quotas read permissions. Select the intended account/region explicitly.

```bash
export AWS_PROFILE=approved-readonly-profile
export AWS_REGION=us-east-1
aws sts get-caller-identity --profile "$AWS_PROFILE"
aws ec2 describe-instance-type-offerings \
  --location-type region \
  --filters Name=instance-type,Values=g5.12xlarge \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws ec2 describe-instance-types \
  --instance-types g5.12xlarge \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'InstanceTypes[].{Type:InstanceType,VCpus:VCpuInfo.DefaultVCpus,Gpus:GpuInfo.Gpus,Network:NetworkInfo.NetworkPerformance}'
aws ec2 describe-vpcs --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws service-quotas list-service-quotas \
  --service-code ec2 --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'Quotas[?contains(QuotaName, `Running On-Demand`) || contains(QuotaName, `Spot`)].{Name:QuotaName,Code:QuotaCode,Value:Value}'
```

CLI queries above are discovery examples. Confirm the actual quota bucket and `QuotaCode` from returned data; do not guess a fixed quota code for every GPU generation. `list-service-quotas` may paginate and may not return all default quotas; use `list-aws-default-service-quotas` or `get-service-quota` as appropriate. Verify instance type and offering before estimating fleet costs.

## 5. Quota calculation and request workflow

1. Inventory account, region, GPU instance family, exact size, vCPU count and purchase model.
2. Identify the correct EC2 quota bucket and current utilization across all teams.
3. Calculate planned peak vCPUs, baseline consumption, N-1 replacement capacity and headroom.
4. Review AZ offerings and procurement alternatives.
5. Submit a quota increase through approved Service Quotas workflows when necessary; capture request ID and lead time.
6. Validate physical capacity separately through supported reservation or controlled staging launch processes.
7. Set workload queue/admission controls and cost guardrails before scaling.
8. Record escalation, rollback and owner signoff.

## 6. GPU host provisioning and security

Use reviewed GPU-capable AMIs with pinned image IDs or approved image pipelines. Decide whether drivers are managed by the AMI, NVIDIA GPU Operator or your host automation; avoid conflicting owners. Prefer IMDSv2, encrypted EBS/KMS, least-privilege instance profiles, SSM Session Manager over broad SSH, restricted security groups, patching, private networking, audit trails and signed artifacts.

Do not embed AWS access keys in AMIs, user data or model containers. Restrict S3/FSx checkpoint access through workload identity/instance profile and bucket policies. Enable termination protection only where compatible with lifecycle and orchestration requirements; it is not a replacement for checkpointing.

## 7. Networking, EFA and storage

For distributed training, verify EFA-supported instance type, subnet/AZ, EFA network interface, security group configuration, libfabric/NCCL plugin compatibility, placement group strategy and actual collective performance. EFA availability differs by instance type and region. EBS and FSx throughput must be sized for dataset and checkpoint peaks; instance store is ephemeral. Cross-AZ training may introduce cost/latency and complicate topology; validate before deploying.

## 8. Capacity procurement and recovery

| Model | Advantage | Reliability risk |
|---|---|---|
| On-Demand | flexible usage | quota and physical capacity may still constrain launch |
| Spot | lower unit price when available | interruption and availability volatility |
| On-Demand Capacity Reservation | targeted capacity reservation | AZ/type specificity and financial commitment |
| Capacity Blocks for ML | scheduled GPU capacity for eligible types | eligibility, timing and reservation constraints |
| Savings Plans/RI | billing commitment benefits | do not inherently guarantee all GPU capacity |

Check current AWS product terms. Use Spot for checkpointable workloads with validated interruption recovery; use reservations when business continuity demands predictable capacity. Maintain a fallback strategy and communicate that cross-AZ alternatives may not meet NCCL or data-locality constraints.

## 9. Observability and SLOs

| Metric | Interpretation |
|---|---|
| GPU UUID, Xid/ECC, temperature, memory | hardware correctness |
| DCGM GPU utilization, SM occupancy | workload efficiency |
| EC2 launch failure reason and latency | quota/capacity issue |
| GPU-ready time after instance launch | driver/toolkit/bootstrap bottleneck |
| EFA throughput, packet drops, NCCL step time | distributed communication |
| EBS/FSx IOPS, throughput and latency | data bottleneck |
| training checkpoint age and restore time | RPO/RTO |
| idle GPU-hours and dollars/job | FinOps |
| regional/AZ eligible N-1 spare capacity | resilience |
| p95/p99 inference latency, training progress | customer SLO |

Correlate CloudWatch, DCGM, application traces, EC2 state transitions and AWS Health. Avoid interpreting low GPU utilization as waste without considering reservations, burst protection or N-1.

## 10. Twelve troubleshooting scenarios

1. EC2 quota exceeded despite sufficient budget.
2. Instance type not offered in selected AZ.
3. InsufficientInstanceCapacity with unused quota.
4. GPU AMI boots but driver module fails.
5. CUDA version incompatible with installed driver.
6. EC2 launches but EFA is unavailable or misconfigured.
7. NCCL performance collapses across network topology.
8. Checkpoint writes saturate EBS/FSx.
9. Spot interruption exceeds checkpoint RPO.
10. IAM/IMDSv2 regression blocks S3 checkpoint access.
11. GPU Xid/ECC causes training instability.
12. Idle GPU fleet creates unbounded spend.

## 11. Ten incident simulations

1. Regional GPU quota exhausted during burst.
2. AZ-specific physical capacity shortage.
3. Failed quota increase before planned launch.
4. GPU driver regression after AMI rollout.
5. EFA misconfiguration produces NCCL hangs.
6. S3/FSx checkpoint access denied.
7. Spot reclaim during distributed checkpoint.
8. GPU hardware ECC error on a training node.
9. Multi-AZ replacement cannot satisfy placement constraints.
10. Cost guardrail exceeded by orphaned GPU instances.

For each, capture UTC timeline, account/region/AZ, instance type, quota bucket, instance ID (redacted where needed), GPU UUID, owner, SLO, checkpoint, containment, recovery, rollback and RCA/COA.

## 12. Eight operational runbooks

**RB-035-A — GPU capacity intake:** requirements, instance fit, vCPU quota bucket and cost.

**RB-035-B — Quota failure:** inventory consumption, request increase, communicate lead time and fallback.

**RB-035-C — Capacity shortage:** evaluate offerings, AZ alternatives and reservation strategy.

**RB-035-D — Driver/CUDA:** compare AMI, kernel, driver, toolkit, framework and smoke test.

**RB-035-E — EFA/NCCL:** verify topology, EFA eligibility, security groups, libfabric and collective benchmark.

**RB-035-F — Storage/checkpoint:** verify EBS/FSx/S3 throughput, permissions, durability and restore.

**RB-035-G — Spot recovery:** checkpoint, interruption signal, replacement and rank coordination.

**RB-035-H — FinOps and RCA:** attribute GPU-hours, enforce budgets, validate SLO and signoff.

## 13. Hands-on lab sequence

1. Identify workload GPU/VRAM/vCPU/EFA/storage requirements and SLO.
2. Inventory actual instance type offerings, vCPU sizes, quotas and AZ availability with read-only AWS CLI.
3. Calculate quota headroom, N-1 capacity, projected GPU-hours and budget.
4. Validate an approved GPU AMI, IMDSv2, IAM, KMS and private-network design.
5. On disposable staging capacity, perform driver/CUDA/NVML and EFA smoke tests if authorized.
6. Benchmark checkpoint throughput and restore; simulate Spot interruption offline.
7. Exercise a quota-exhaustion scenario and propose an approved fallback.
8. Record evidence, recovery runbooks, cost model and owner signoff.

[Lab 035 — AWS GPU EC2 Architecture and Quota Gate](../labs/035-aws-gpu-ec2-architecture-quotas-check.md) provides a Python 3.11+ fail-closed offline evaluator and pytest tests. It does not query or change AWS.

## 14. 100-point assessment

| Area | Points |
|---|---:|
| GPU EC2 architecture and instance sizing | 20 |
| Quotas, offerings and capacity procurement | 20 |
| Security, drivers, EFA and storage | 20 |
| Incident response, checkpoint and FinOps | 20 |
| Tests, staging evidence and signoff | 20 |
| **Total** | **100** |

Pass >=90 with no unresolved quota, physical-capacity, security, checkpoint, GPU correctness or cost blocker.

## References

- [Amazon EC2 accelerated computing instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/accelerated-computing-instances.html)
- [Amazon EC2 service quotas](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-resource-limits.html)
- [EC2 instance type offerings](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instance-type-offerings.html)
- [AWS Service Quotas](https://docs.aws.amazon.com/servicequotas/latest/userguide/intro.html)
- [Elastic Fabric Adapter](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)
- [Capacity Blocks for ML](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-blocks.html)
- [NVIDIA Data Center GPU Driver](https://docs.nvidia.com/datacenter/tesla/)

**Next: Chapter 036 — EKS GPU Managed Node Groups.**
