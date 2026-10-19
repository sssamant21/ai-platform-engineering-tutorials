# Chapter 045 — AWS EFA for Distributed AI

**AI Infrastructure Engineering & SRE · Part 06 — High Performance Networking · Chapter 045/100**

**Publication status:** Documentation and offline synthetic lab only. Live EFA throughput, GPU correctness, AWS quota/capacity, security review and production acceptance remain pending.

## Learning objectives

Design, deploy, secure, benchmark, observe and troubleshoot AWS Elastic Fabric Adapter (EFA) for distributed AI on supported EC2/EKS GPU instances. Explain EFA versus ENA/InfiniBand/RoCE, libfabric and the AWS OFI NCCL plugin; validate GPU/NIC/NUMA placement, EKS device plugins, peer security groups, cluster placement groups, same-AZ constraints, NCCL collectives, failure recovery, SLOs and FinOps.

## 1. Production reference architecture

```text
   training scheduler / EKS / workload identity / IAM
                        |
         Kubernetes Job / distributed ranks
                        |
    +-------------------+-------------------+
    |                                       |
 EC2 GPU node A                         EC2 GPU node B
 GPU + PCIe/NUMA                        GPU + PCIe/NUMA
 NVIDIA driver/CUDA/NCCL                NVIDIA driver/CUDA/NCCL
 AWS OFI NCCL -> libfabric              AWS OFI NCCL -> libfabric
 EFA interface + kernel driver          EFA interface + kernel driver
    |                                       |
    +---------- AWS EFA fabric -------------+
                        |
        same-AZ eligible instance topology
                        |
 S3/FSx checkpoint + DCGM/CloudWatch/Prometheus
                        |
     SLO, incident response, cost and rollback
```

EFA provides AWS-specific low-latency communication for supported EC2 instance families. **EFA is not native InfiniBand and does not require an InfiniBand subnet manager.** Its transport/provider semantics are not identical to RoCE. EFA-capable instance type, enabled interface, host kernel driver, libfabric provider, AWS OFI NCCL integration and a working NCCL collective are separate readiness gates.

## 2. Transport and dependency chain

| Layer | Responsibility | Validation |
|---|---|---|
| EC2 instance type | EFA capability and GPU hardware | `describe-instance-types` |
| VPC subnet/AZ | placement and routing boundary | subnet/AZ inventory |
| EFA network interface | enabled interface and attachment | EC2 ENI description |
| Kernel/host | EFA driver and supported OS | `modinfo efa`, `dmesg` |
| libfabric | EFA provider | `fi_info -p efa` |
| AWS OFI NCCL | NCCL transport plugin | plugin logs and linkage |
| NVIDIA stack | GPU driver, CUDA and NCCL | `nvidia-smi`, versions |
| EKS | resource allocation/device plugin | node allocatable and pod events |
| Application | multi-rank correctness/performance | `nccl-tests`, training SLO |

An ENA-only instance can communicate over TCP but cannot satisfy an EFA transport requirement. EFA support and maximum interface counts vary by instance type; do not hardcode interface-per-GPU ratios. Cross-AZ EFA collective transport is not a valid assumption: plan communicating ranks in supported same-AZ placements and test failure-domain recovery separately.

## 3. Quota, availability and placement

Check EC2 accelerated-instance vCPU quotas for the applicable On-Demand/Spot bucket, SKU offerings, current usage, subnet IP headroom, instance EFA capability and physical capacity. For tightly coupled training, evaluate a cluster placement group when supported and operationally appropriate. A placement group can improve proximity but may constrain available capacity and is not an availability guarantee. Reserve N-1 capacity and verify checkpoint recovery if an AZ or rack-related failure removes ranks.

**Read-only discovery:**

```bash
export AWS_PROFILE=approved-readonly
export AWS_REGION=us-east-1
aws sts get-caller-identity --profile "$AWS_PROFILE"
aws ec2 describe-instance-types --instance-types p4d.24xlarge \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'InstanceTypes[].{Name:InstanceType,EFA:NetworkInfo.EfaSupported,MaxEFA:NetworkInfo.EfaInfo.MaximumEfaInterfaces,GPUs:GpuInfo.Gpus}'
aws ec2 describe-instance-type-offerings --location-type availability-zone \
  --filters Name=instance-type,Values=p4d.24xlarge \
  --region "$AWS_REGION" --profile "$AWS_PROFILE"
aws service-quotas list-service-quotas --service-code ec2 \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --query 'Quotas[].{Name:QuotaName,Value:Value,Code:QuotaCode}'
aws ec2 describe-placement-groups --region "$AWS_REGION" \
  --profile "$AWS_PROFILE" --query 'PlacementGroups[].{Name:GroupName,Strategy:Strategy,State:State}'
```

These commands reveal configuration and eligibility, not guaranteed physical capacity.

## 4. EFA security-group and network design

EFA traffic requires the AWS-documented EFA-compatible peer security-group rules, typically permitting appropriate traffic between EFA-enabled instances in the same tightly controlled security group. Review both ingress and egress; restrict membership to approved training nodes and do **not** expose unrestricted internet ingress. Verify SG/NACL, subnet, routing and EFA ENI configuration. Do not conflate security-group rules with EKS NetworkPolicy, IAM or dataset permissions.

## 5. Host validation and version compatibility

On an **approved disposable** EFA GPU host:

```bash
uname -r
lspci -nn | grep -Ei 'Amazon|EFA|NVIDIA'
modinfo efa | head -n 25
fi_info -p efa
nvidia-smi -L
nvidia-smi topo -m
ldconfig -p | grep -Ei 'fabric|nccl'
ip -s link
dmesg --color=never | grep -Ei 'efa|nvrm|xid' | tail -n 80
```

Confirm OS/kernel, EFA installer or managed AMI, libfabric, AWS OFI NCCL plugin, NVIDIA driver, CUDA and NCCL compatibility against current release documentation. Merely finding a library on disk does not prove it is selected at runtime. Check NCCL debug output in staging; redact secrets and internal topology from published logs.

## 6. EKS GPU and EFA scheduling

An EKS GPU training pod needs compatible GPU resource allocation plus EFA device resources when using the EFA device plugin. Exact advertised extended-resource names and counts depend on the installed plugin and instance configuration. Inspect:

```bash
kubectl get nodes -o json | jq -r '.items[] | [.metadata.name, (.status.allocatable | tojson)] | @tsv'
kubectl get daemonsets -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

Use node affinity for GPU/EFA-capable nodes, taints/tolerations for dedicated pools, gang-aware or coordinated rank scheduling where required, namespace quotas and immutable image digests. Validate the exact AWS EFA device-plugin resource key before placing it in a manifest. Avoid universally prescribing privileged containers, host networking or unbounded hostPath access; document exceptions and compensating controls.

## 7. NCCL collective validation

Run `nccl-tests` only in authorized staging with measured, reproducible rank count, GPU topology, message sizes, warm-up and iteration count. Validate correctness, `algbw`, `busbw`, p95 step duration and timeout behavior. Use supported AWS OFI NCCL instrumentation to verify EFA rather than silent TCP fallback. Correlate GPU health, EFA/libfabric counters, pod events and checkpoint status. Do not infer GPUDirect RDMA solely from GPU and EFA presence.

## 8. Scaling and failure-domain engineering

Autoscaling GPU nodes is limited by EC2 quota, physical capacity, AZ, placement group, subnet IPs, bootstrap time and EFA attachment. Scaling training ranks independently can cause collective hangs; design scheduler admission and restart boundaries. Validate N-1, Spot interruption handling where appropriate, checkpoint age and restore time. Same-AZ EFA topology improves communication locality but can increase AZ failure exposure; disaster recovery needs a separately tested restart/failover plan.

## 9. Observability and SLOs

| Signal | SRE interpretation |
|---|---|
| EFA ENI attachment/provider readiness | transport available |
| libfabric/AWS OFI NCCL selection | actual intended transport |
| GPU Xid/ECC and utilization | compute health |
| NCCL all-reduce bandwidth and step p95 | collective SLO |
| peer reachability and connection errors | SG, topology, fabric |
| pod Pending age and node bootstrap | capacity/cold start |
| checkpoint age and restore RTO | recoverability |
| EFA/GPU node cost per completed job | FinOps |
| audit freshness, owner and rollback | operational readiness |

Use baselines tied to instance family, GPU count, rank count, NCCL/plugin versions and message sizes. A link-up status alone is insufficient.

## 10. Twelve troubleshooting scenarios

1. EFA-unsupported EC2 instance selected.
2. EFA-capable instance launched without EFA ENI.
3. EFA kernel driver missing after AMI upgrade.
4. libfabric `efa` provider not discovered.
5. AWS OFI NCCL plugin absent or incompatible.
6. NCCL silently falls back to TCP.
7. Security-group self-reference or peer egress rule missing.
8. Distributed ranks scheduled across unsupported AZ boundaries.
9. EKS EFA device-plugin allocatable resources missing.
10. GPU–NIC PCIe/NUMA locality regression.
11. Quota or placement group capacity prevents scale-out.
12. Spot eviction or node loss stalls all-reduce and violates checkpoint RPO.

## 11. Ten incident simulations

1. EFA ENI attachment regression.
2. Kernel/driver incompatibility.
3. libfabric provider upgrade regression.
4. AWS OFI NCCL plugin mismatch.
5. Security-group rule drift blocks peers.
6. Incorrect AZ placement.
7. GPU/EFA device-plugin crash.
8. NCCL hang under rank loss.
9. EFA capacity shortfall during scale-out.
10. Telemetry outage combined with checkpoint RPO breach.

Run disruptive exercises only on approved staging infrastructure. Preserve UTC timeline, affected ranks/nodes, EFA and GPU versions, collective logs, security evidence, rollback and RCA/COA.

## 12. Eight operational runbooks

**RB-045-A — EFA eligibility:** validate SKU, quota, AZ, interface limits and placement group.

**RB-045-B — ENI/driver:** inspect EFA attachment, kernel driver, OS image and host errors.

**RB-045-C — libfabric/plugin:** verify provider and AWS OFI NCCL version/selection.

**RB-045-D — EKS device resources:** inspect GPU and EFA allocatable, DaemonSets and scheduling.

**RB-045-E — Security/reachability:** verify least-privilege peer SG/NACL and same-AZ layout.

**RB-045-F — NCCL hang:** preserve checkpoint, isolate rank/transport failure, compare baseline.

**RB-045-G — Capacity recovery:** review quota, physical capacity, placement, N-1 and fallback.

**RB-045-H — RCA/FinOps:** reconcile GPU cost, SLO, telemetry, approval and rollback.

## 13. Eight hands-on labs

1. Discover EFA-supported GPU instance families, interface limits and AZ offerings.
2. Calculate quota demand, placement capacity and N-1 resilience.
3. Inspect EFA ENI, driver, libfabric and AWS OFI NCCL readiness.
4. Validate EKS GPU/EFA resource advertisement and rank placement.
5. Review EFA peer SG policy and tenant isolation.
6. Benchmark NCCL all-reduce with transport verification on approved staging.
7. Simulate plugin, SG, AZ and checkpoint incidents **offline**.
8. Produce redacted readiness evidence and a signed 100-point scorecard.

[Lab 045 — AWS EFA for Distributed AI Gate](../labs/045-aws-efa-distributed-ai-check.md) supplies a Python 3.11+ synthetic readiness evaluator and pytest tests.

## 14. 100-point assessment

| Domain | Points |
|---|---:|
| AWS EFA architecture, quota and AZ capacity | 20 |
| Driver, libfabric, AWS OFI NCCL and GPU integration | 20 |
| EKS scheduling, identity and network security | 20 |
| Collective SLO, failure recovery and FinOps | 20 |
| Tests, staging evidence and reviewer approval | 20 |
| **Total** | **100** |

Pass >=90 and no critical quota/capacity, transport correctness, tenant security, checkpoint, SLO or rollback blocker.

## References

- [AWS Elastic Fabric Adapter](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)
- [AWS EFA with EKS](https://docs.aws.amazon.com/eks/latest/userguide/node-efa.html)
- [AWS OFI NCCL](https://github.com/aws/aws-ofi-nccl)
- [AWS EFA installer](https://github.com/aws/aws-efa-installer)
- [NVIDIA NCCL](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [EC2 placement groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html)
- [EKS GPU workloads](https://docs.aws.amazon.com/eks/latest/userguide/ml-eks-k8s-device-plugin.html)

**Next: Chapter 046 — RoCE and Ethernet Fabrics.**
