# Chapter 037 — AWS EFA and GPU Networking

**AI Infrastructure Engineering & SRE | Part 05 — Cloud GPU Infrastructure | Chapter 037/100**

**Status:** Published tutorial. Offline lab execution, live EFA/NCCL benchmarks, security review and production acceptance are tracked separately.

## Learning objectives

Design, validate and troubleshoot AWS Elastic Fabric Adapter (EFA) for GPU distributed training on EC2 and Amazon EKS. Explain EFA, libfabric, AWS OFI NCCL, topology, placement groups, ENA versus EFA, subnet and security-group requirements, host/kernel/container compatibility, multi-node collectives, latency and throughput SLOs, and recovery. Distinguish supported EFA network paths from generic TCP, and avoid inferring RDMA readiness from EC2 launch or Kubernetes node readiness.

## 1. Enterprise architecture

```text
Distributed training: PyTorch / torchrun / NCCL ranks
             |
NCCL collectives and communicator topology
             |
AWS OFI NCCL plugin -> libfabric (EFA provider)
             |
EFA device + kernel driver / userspace stack
             |
EC2 instance EFA-capable network interface
             |
VPC subnet / SG / cluster placement / AZ capacity
             |
Peer GPU training node (same supported topology)
             |
CUDA, NCCL, DCGM, CloudWatch, training-step SLO
```

EFA enables low-latency, high-throughput communication for supported EC2 workloads through the libfabric provider. It is not a replacement for VPC IP connectivity or security controls. EFA-capable EC2 does not imply that the EFA device was attached, the libfabric provider is active, or NCCL is actually using the desired transport.

## 2. ENA, EFA and GPU interconnect boundaries

| Layer | Purpose | Operational evidence |
|---|---|---|
| ENA / IP networking | standard VPC packet connectivity | interface/IP, route, SG, reachability |
| EFA device | supported low-latency HPC communication | EC2 interface type, OS EFA device |
| libfabric EFA provider | user-space communication path | `fi_info -p efa` and provider version |
| AWS OFI NCCL | NCCL integration with EFA | plugin loading, NCCL transport logs |
| NVLink/NVSwitch | intra-node GPU peer links where supported | `nvidia-smi topo -m` |
| CUDA/NCCL | GPU computation and collectives | validated smoke and `nccl-tests` |
| DCGM/CloudWatch | hardware/network/workload telemetry | GPU Xid/ECC, training throughput |

EFA uses its own provider stack; do not describe every EFA path as generic InfiniBand verbs or assume GPU Direct RDMA availability across every instance type and release. Validate hardware, driver, CUDA/NCCL and AWS OFI NCCL compatibility for the selected instance.

## 3. Capacity and placement prerequisites

Confirm exact GPU instance type, EFA support, number of EFA interfaces, supported AZ offering, placement-group strategy, subnet IP availability, EC2 vCPU quota, actual physical capacity and any Capacity Block/reservation constraints. EFA network interfaces are AZ-bound. For tightly coupled training, avoid arbitrary cross-AZ rank placement; measure performance and failure-domain tradeoffs. A cluster placement group can improve proximity but is not a guarantee of instance capacity or application throughput.

For EKS, distinguish GPU managed node group, Karpenter NodePool and underlying EC2 launch template. EFA attachment and EFA device exposure require an explicitly supported instance/network configuration. Karpenter/managed node group cannot make an unsupported instance EFA-capable.

## 4. Security group and network policy

EFA traffic needs a security-group configuration compatible with AWS EFA requirements, commonly permitting required traffic between participating EFA interfaces within a dedicated security group. Review the current AWS documentation and apply the narrowest approved rule scope; do **not** indiscriminately open all traffic to the internet. VPC endpoints, DNS, routing, host firewall, IAM and EKS network policies serve different boundaries. NetworkPolicy alone does not replace EC2 security-group configuration for EFA.

Do not publish private IP addresses, security-group IDs, IAM roles, checkpoint locations or complete unredacted training logs.

## 5. Read-only AWS discovery

Set the approved AWS identity, region and instance IDs. Commands below inspect configuration only:

```bash
export AWS_PROFILE=approved-readonly
export AWS_REGION=us-east-1
aws sts get-caller-identity --profile "$AWS_PROFILE"
aws ec2 describe-instance-types --instance-types p4d.24xlarge \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'InstanceTypes[].{Type:InstanceType,Efa:NetworkInfo.EfaSupported,EfaMax:NetworkInfo.EfaInfo.MaximumEfaInterfaces,Network:NetworkInfo.NetworkPerformance}'
aws ec2 describe-instance-type-offerings --location-type availability-zone \
  --filters Name=instance-type,Values=p4d.24xlarge \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws ec2 describe-instances --instance-ids APPROVED_INSTANCE_ID \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'Reservations[].Instances[].{Type:InstanceType,AZ:Placement.AvailabilityZone,Interfaces:NetworkInterfaces[].InterfaceType,State:State.Name}'
```

Verify the AWS CLI version and returned fields; API shapes and instance offerings vary. A listed offering is not a physical capacity guarantee.

## 6. Host and container readiness

On an approved GPU/EFA node, collect:

```bash
uname -r
nvidia-smi -L
nvidia-smi topo -m
fi_info -p efa
lsmod | grep -E 'efa|nvidia'
ip -brief link
```

The `fi_info` binary must be installed from the supported EFA/libfabric stack. Verify `libfabric`, EFA kernel driver, EFA installer, AWS OFI NCCL plugin, CUDA/NCCL and container image versions. Container workloads must have the required EFA device access via the supported Kubernetes device-plugin/runtime design, not unrestricted host device mounts.

## 7. EKS EFA topology and scheduling

A distributed GPU training job should explicitly constrain GPU SKU, AZ, EFA capability, CPU/memory, `nvidia.com/gpu`, topology, device-plugin resource keys and scheduling policy. Validate any EFA resource key against the installed EFA device plugin; do not invent a universal resource name. Kubernetes may schedule each rank independently unless a training operator or queue implements coordinated/gang admission. Use supported job controllers and a compatible NCCL communication plan.

**Preflight:** all ranks have compatible GPUs/drivers, EFA interfaces, network/security-group reachability, NCCL/libfabric plugin versions and adequate checkpoint storage.

## 8. NCCL diagnostics and performance baseline

Use an approved training image and `nccl-tests` release. In staging, collect rank count, GPUs per rank, message sizes, `all_reduce_perf` throughput, bus bandwidth, p50/p95 step time and error rate. Use `NCCL_DEBUG=INFO` only for controlled troubleshooting, and review logs for sensitive data. Confirm that the AWS OFI NCCL plugin and EFA provider were actually selected; successful NCCL collectives may otherwise use TCP or a different transport.

Do not force undocumented NCCL environment settings into production. Tune transport, NUMA affinity and EFA interface selection only after measuring baseline behavior.

## 9. Reliability and observability

| Signal | Why it matters |
|---|---|
| EFA interface attachment/provider discovery | network readiness |
| NCCL transport/plugin initialization | expected collective path |
| all-reduce bus bandwidth and p95 latency | distributed communication SLO |
| training step p95 and stalled-rank duration | user-facing training progress |
| network retransmission/error signals where supported | transport diagnosis |
| GPU utilization, Xid/ECC, NVLink topology | distinguish compute from network |
| EFA/libfabric error logs | host/provider issues |
| checkpoint age and restore success | interruption RPO/RTO |
| AZ/rank placement and N-1 capacity | failure-domain risk |
| GPU-hour and inter-AZ transfer costs | FinOps |

Set performance thresholds from actual workload baseline, not arbitrary universal NCCL bandwidth targets. Collect UTC timestamps and versions for correlation.

## 10. Twelve troubleshooting scenarios

1. EC2 instance type does not support EFA.
2. Instance EFA capable, but no EFA interface attached.
3. EFA driver absent or incompatible with kernel.
4. `fi_info -p efa` finds no provider.
5. Container lacks EFA device access.
6. Security group blocks required peer EFA communication.
7. EKS scheduler places ranks across incompatible AZs.
8. AWS OFI NCCL plugin fails to load.
9. NCCL silently falls back to slower transport.
10. GPU/NUMA placement causes poor all-reduce throughput.
11. EFA instance launch fails due to AZ capacity or EC2 quota.
12. Spot interruption causes NCCL rank hang and stale checkpoint.

## 11. Ten production incident simulations

1. EFA device omitted from a new launch template.
2. libfabric upgrade breaks the EFA provider.
3. Security-group change blocks collective communication.
4. EKS EFA device plugin registration disappears.
5. NCCL image update breaks AWS OFI NCCL loading.
6. Training rank lands on an unsupported GPU instance.
7. Cross-AZ placement causes training throughput regression.
8. EFA host network errors cause intermittent rank stalls.
9. Spot interruption kills a rank mid-checkpoint.
10. Regional EFA-capable GPU capacity becomes unavailable.

Record UTC timeline, instance SKU/AZ, interface type, host driver/libfabric/NCCL versions, GPU UUID, rank mapping, SLO, checkpoint, containment, rollback and RCA/COA. Inject faults only in approved disposable staging environments or offline simulations.

## 12. Eight operational runbooks

**RB-037-A — EFA inventory:** confirm EC2 instance EFA eligibility, interface attachment and AZ.

**RB-037-B — Host provider:** inspect kernel, EFA module, `fi_info`, libfabric and permissions.

**RB-037-C — EKS device exposure:** reconcile EFA device plugin, kubelet resources and container allocation.

**RB-037-D — Network security:** validate approved security-group peer rules and route/DNS boundaries.

**RB-037-E — NCCL transport:** confirm AWS OFI NCCL load, selected transport, rank mapping and benchmark.

**RB-037-F — Performance regression:** compare message sizes, GPU/NUMA placement, baseline and network errors.

**RB-037-G — Rank recovery:** preserve checkpoints, restart communicator/ranks, validate RPO/RTO and SLO.

**RB-037-H — Capacity and RCA:** evaluate quotas, offerings, reservations, cost and owner signoff.

## 13. Executable hands-on labs

1. Inventory EFA support and AZ offerings with AWS CLI read-only queries.
2. Verify EFA interface attachment and security-group architecture.
3. Inspect host EFA kernel module and libfabric provider.
4. Validate EKS GPU/EFA device plugin ownership and scheduling constraints.
5. Run a reviewed multi-node `nccl-tests` baseline on approved staging GPU capacity.
6. Simulate missing EFA, blocked peer connectivity and plugin fallback **offline**.
7. Exercise checkpoint recovery and measure p95 training-step performance.
8. Produce redacted evidence, rollback and cost/reliability acceptance.

[Lab 037 — AWS EFA and GPU Networking Gate](../labs/037-aws-efa-gpu-networking-check.md) provides a fail-closed Python 3.11+ policy evaluator and pytest cases. It does not access AWS or GPUs.

## 14. 100-point assessment

| Domain | Points |
|---|---:|
| EFA architecture, placement and capacity | 20 |
| Host, libfabric, device plugin and security | 20 |
| NCCL performance and workload SLO | 20 |
| Incidents, checkpoint and recovery | 20 |
| Executable tests and staging evidence | 20 |
| **Total** | **100** |

Pass >=90 and no critical network isolation, EFA readiness, GPU correctness, checkpoint or rollback blocker.

## References

- [AWS Elastic Fabric Adapter](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)
- [AWS EFA getting started](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa-start.html)
- [EFA security groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa-start.html)
- [AWS OFI NCCL plugin](https://github.com/aws/aws-ofi-nccl)
- [libfabric](https://github.com/ofiwg/libfabric)
- [NVIDIA NCCL](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [Amazon EKS GPU workloads](https://docs.aws.amazon.com/eks/latest/userguide/ml-eks-optimized-ami.html)

**Next: Chapter 038 — Azure GPU VM Architecture and Quotas.**
