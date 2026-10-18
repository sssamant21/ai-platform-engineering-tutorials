# Chapter 003 — GPU Infrastructure Reference Architectures

**Part 01 — Foundations and Architecture · Chapter 003 of 100**  
**Publication:** Complete. **Lab execution, real GPU validation, security review and production acceptance:** Pending.

## Learning objectives
Design reliable AI infrastructure for online inference, batch inference, fine-tuning and multi-node training. Map failure domains, GPU topology, network, storage, scheduling, identity, telemetry and cost. Compare EKS/AKS and bare-metal approaches. Evaluate N-1 capacity, placement, recovery and release gates using a reproducible offline lab.

## 1. Architecture requirements and constraints
Start with a versioned workload contract, not GPU model selection. Capture tenant and data sensitivity, model size and precision, peak input/output tokens, context length, request arrival distribution, concurrency, training parallelism, checkpoint interval, throughput, p95 TTFT/TPOT, availability SLO, RPO/RTO, region/residency, budget and quota.

| Dimension | Online inference | Distributed training |
|---|---|---|
| Primary objective | latency, stream completion, availability | useful tokens/s, successful jobs |
| GPU pressure | weights, KV cache, prefill/decode | weights, gradients, optimizer, activations |
| Networking | ingress, model routing, RAG | NCCL collectives, RDMA, topology |
| Storage | artifact cache, logs, RAG data | datasets, parallel FS, checkpoints |
| Scaling | warm replicas, admission, autoscaler | gang scheduling, elastic workers where supported |
| Recovery | N-1 failover, rollback, bounded retries | restore consistent model/optimizer/data cursor |
| Unit economics | cost per delivered token | cost per successful training objective |

GPU architecture, CUDA version, driver, GPU Operator, device plugin, Kubernetes version, NCCL and framework compatibility must be pinned and tested. Avoid assuming every GPU supports MIG or NVLink.

## 2. Reference architecture A — Multi-zone online inference
```text
   clients / synthetic probes
              |
       identity + WAF
              |
    gateway / per-tenant quotas
              |
      admission + routing
         /      |      \
      Zone A  Zone B  Zone C
      GPU     GPU     GPU
      pods    pods    pods
         \      |      /
        signed model cache
              |
       object/artifact store
              |
   OTel + DCGM + Prometheus
              |
      SLO + cost governance
```
Design each zone with independent failure boundaries where supported. Use topology spread constraints, disruption budgets, minimum warm replicas, compatible device plugins and enough N-1 throughput. Check whether the regional load balancer, quota, registry, KMS, control plane or object store remains a shared dependency. Availability across zones does not imply cross-region disaster recovery.

**Streaming correctness:** Count a response as successful only when the full stream terminates correctly. Measure first-token time and inter-token cadence at the client. Limit retries after output begins; repeated requests can duplicate side effects or spend.

## 3. Reference architecture B — Multi-node distributed training
```text
 job submission + identity + quota
                 |
        Slurm / K8s / Ray
                 |
       gang / topology scheduler
        /        |        \
    GPU host  GPU host  GPU host
     ranks      ranks      ranks
        \        |        /
        NCCL / RDMA fabric
                 |
     dataset cache / parallel FS
                 |
    versioned checkpoint store
                 |
       metrics + job ledger
```
Account for PCIe/NUMA, intra-host NVLink/NVSwitch where available, inter-host RDMA/EFA/InfiniBand/RoCE and dataset bandwidth. A fast GPU can idle when the slowest rank waits for data or a collective. Do not promise elastic training unless the framework and checkpoint design support safe membership changes.

**Checkpoint contract:** Record model, optimizer, scheduler, RNG, data cursor, topology assumptions and signed artifact metadata. Measure actual restore time and recovery point. A checkpoint file existing is not proof of recoverability.

## 4. Reference architecture C — Shared enterprise AI platform
Separate sensitive tenant classes and workload priorities using namespace, identity, network policy, quota, node-pool and storage boundaries. Isolate low-latency inference from opportunistic batch/training, even if the scheduler permits sharing. For hard multitenancy, verify GPU isolation, memory erasure, MIG availability, device exposure and vendor-specific security requirements.

Control plane: GitOps/IaC, policy admission, workload scheduler, artifact registry and inventory. Data plane: GPU execution, network, storage and serving. Management plane: secrets, IAM, audit, monitoring and incident control. Preserve read-only diagnostics when the management plane is degraded.

## 5. AWS/EKS and Azure/AKS patterns
| Area | AWS | Azure | SRE check |
|---|---|---|---|
| Compute | GPU EC2 families and EKS nodes | GPU VM SKUs and AKS pools | regional SKU availability, quota |
| Networking | VPC, ENI, EFA where supported | VNet, accelerated networking / RDMA where supported | topology, driver and bandwidth |
| Storage | S3, EBS, FSx for Lustre | Blob, managed disks, Azure HPC storage options | sustained throughput and checkpoint restore |
| Identity | IAM roles for service accounts / Pod Identity | workload identity / managed identity | least privilege, token lifetime |
| Observability | CloudWatch + Prometheus/DCGM | Azure Monitor + Prometheus/DCGM | client SLO and GPU telemetry |
| Recovery | multi-AZ plus DR design | availability zones plus DR design | independent dependencies, RPO/RTO |

Service availability and GPU SKU capabilities vary by region, instance generation and account quota. Validate against current provider documentation before implementing.

## 6. Capacity and cost governance
For a homogeneous workload mix, define measured safe per-zone throughput at passing SLOs:
```text
N-1 safe capacity = sum(zone safe throughput) - max(zone safe throughput)
admitted peak <= N-1 safe capacity * (1 - reserve_fraction)
cost_per_1M_tokens = total allocated cost / delivered_tokens * 1,000,000
```
Example: zones A/B/C each sustain 100 requests/s at passing SLO. N-1=200 requests/s; with 20% reserve, admitted peak must be <=160 requests/s. These numbers are illustrative, not GPU benchmarks. Mixed models, prompt lengths and prefill/decode profiles require separate safe-capacity measurements; do not sum incompatible token/s and request/s units.

For training, budget GPU-hours lost to retries, communication stalls, data waits and checkpoint overhead. For inference, budget warm idle replicas, autoscaling lead time, KV cache, queueing and region failover.

## 7. [SAFE-READ] Discovery lab
Only run with approved kubeconfig and monitoring permissions:
```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get nodes -o wide
kubectl get daemonsets -A
kubectl get resourcequota -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
nvidia-smi -L
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,memory.total,memory.used,driver_version,temperature.gpu --format=csv
```
`nvidia-smi` requires an NVIDIA GPU and working driver. The topology command is host-local; it does not prove cross-host RDMA health. Treat GPU metrics as point-in-time observations, not SLO validation. Do not publish internal cluster output.

## 8. Architecture acceptance labs
**Lab A — Inference:** synthetic workload with long and short prompts, stream interruption and token counts. Measure TTFT/TPOT, throughput, admission, 429, GPU memory, KV cache and N-1 zone failure.

**Lab B — Training:** synthetic distributed job, collect per-rank step time, network, GPU, dataset throughput, checkpoint integrity and recovery time. Stop on unexpected tenant impact.

**Lab C — Failure domains:** identify shared KMS, registry, DNS, quota and monitoring dependencies. Test control-plane outage and verify the existing data plane behaves as designed.

**Lab D — Security:** verify service identity, signed artifacts, network egress, storage encryption, tenant isolation and audit events. Never bypass security controls to pass a performance test.

**Lab E — Cost:** compare GPU allocation, productive GPU-hours, reservations, retry overhead and cost per successful output.

## 9. Troubleshooting and incident scenarios
| Symptom | Potential cause | First evidence |
|---|---|---|
| GPU pods Pending | quota, taints, placement, scarce SKUs | scheduler events, allocatable |
| GPU invisible | driver, device plugin, GPU Operator | node resources, plugin logs |
| TTFT spikes after zone loss | insufficient N-1, cold replicas | queue age, warm capacity |
| NCCL hangs | rank failure, RDMA/fabric, topology | rank logs, network telemetry |
| GPUs idle in training | dataset/CPU I/O bottleneck | data pipeline, GPU occupancy |
| checkpoint restore fails | incomplete optimizer/data cursor | restore logs and manifests |
| model loads wrong revision | unsigned or stale registry/cache | image/model digests |
| spend spikes | idle GPUs, retry storm, overprovisioning | cost ledger, GPU-hours |

## 10. Operational runbooks
**RB-003-A — Design review:** inventory dependencies, tenant classes, model/runtime versions, failure domains, SLOs, budgets, owners and recovery targets.

**RB-003-B — GPU node loss:** confirm node health, Xid/ECC, device plugin, pod placement and zone capacity; enforce bounded admission; drain/restart only with approved change and rollback.

**RB-003-C — NCCL/training outage:** correlate rank health, interconnect and storage; preserve checkpoint integrity and do not restart until restore preconditions are satisfied.

**RB-003-D — Regional failover:** fence writers, verify residency, identity, signed artifacts, quota, data freshness and synthetic probes before shifting traffic.

**RB-003-E — Architecture rollback:** restore last approved deployment, routing and driver/operator matrix; verify SLOs and tenant isolation, document RCA/COA.

## 11. 100-point assessment
| Domain | Points |
|---|---:|
| Requirements, workload mapping and architecture | 20 |
| GPU topology, storage and networking | 20 |
| N-1 capacity, SLOs and cost | 20 |
| Security, recovery and operational ownership | 20 |
| Executable lab, evidence and runbooks | 20 |
| **Total** | **100** |

Pass >=90/100 with no critical security, data integrity or budget violation. Offline evaluation does not certify real GPU infrastructure.

## 12. Executable lab
[Chapter 003 — Reference Architecture Acceptance Lab](../labs/003-gpu-infrastructure-reference-architectures-check.md) provides a standard-library Python architecture evaluator and pytest regression tests.

## References
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [Kubernetes Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [NVIDIA Data Center Documentation](https://docs.nvidia.com/datacenter/)
- [Google SRE Workbook](https://sre.google/workbook/)

**Next:** Chapter 004 — AI Infrastructure Service Ownership and Support Tiers.
