# Chapter 005 — Linux, Kubernetes and Cloud Dependency Mapping

**Part 01 — Foundations and Architecture · Chapter 005 of 100**  
**Status:** Published. Offline test execution, authorized staging discovery, independent security review and production acceptance pending.

## Learning outcomes
- Build a versioned, directed dependency graph spanning Linux GPU hosts, Kubernetes, cloud services, network, storage, identity and AI applications.
- Distinguish **hard runtime dependencies**, **deployment dependencies**, **control-plane dependencies** and **observability dependencies**.
- Identify single points of failure, hidden shared dependencies, failure-domain concentration and blast radius.
- Use safe discovery commands and a deterministic Python graph analyzer with pytest.
- Convert dependency findings into SLO, recovery, ownership and change-management actions.

## 1. Why dependency mapping matters
A GPU node can report Ready while its NVIDIA device plugin fails, a training job can hang because its storage data loader is slow, and an inference service can return 503 because its model registry or KMS is unavailable during cold start. Cloud infrastructure failures are rarely isolated to one layer. An SRE needs a dependency graph that explains **what breaks when a component fails** and **which team owns recovery**.

```text
           client / batch job / training scheduler
                             |
                     API / job gateway
                             |
                Kubernetes / Slurm / Ray
                  /          |          \
             GPU host      identity      model registry
             /     \          |              |
        driver     device   IAM/KMS       object storage
        CUDA/NCCL  plugin
             \      /
           GPU workload
              |
       network / RDMA / DNS
              |
      checkpoint + datasets
              |
       logs / metrics / traces
```

The arrows in this conceptual view show relationships; a real graph must explicitly state direction. This chapter uses **service → dependency** edges. For example, `inference-api → model-cache` means the inference API depends on the model cache.

## 2. Dependency classes
| Class | Example | Impact | Recovery control |
|---|---|---|---|
| Hard runtime | serving pod → GPU driver | immediate workload failure | healthy GPU host and compatible runtime |
| Startup | serving pod → model registry | cold-start failure; warm pods may survive | pinned artifact cache and verified digests |
| Control plane | running pod → Kubernetes API | existing pod may continue, replacements fail | reconciler recovery and N-1 warm replicas |
| Identity | workload → IAM/KMS | access denied, token refresh failure | scoped identity, expiry monitoring |
| Network | training ranks → RDMA fabric | NCCL hang, slow all-reduce | topology, link health and rank diagnostics |
| Storage | trainer → checkpoint store | failed checkpoint, RPO violation | durable store and restore tests |
| Observability | inference → telemetry pipeline | monitoring blind spot, not necessarily data-plane outage | independent health probe and alert fallback |
| Shared external | multiple regions → central registry | correlated regional startup failures | regional mirrors and artifact pinning |

A dependency's severity depends on whether it is needed for startup, steady-state execution or recovery. Do not automatically treat all dependencies as hard runtime blockers.

## 3. Architecture and failure domains
```text
                          enterprise tenant
                                 |
                           global identity
                                 |
                     region / network boundary
                        /                \
                   zone A                zone B
                GPU pool A             GPU pool B
                  /    \                 /    \
               driver  CNI            driver  CNI
                  \    /                 \    /
               inference                inference
                    \                     /
                      shared model registry
                               |
                          object storage
```
A multi-zone architecture is not automatically highly available: global IAM, KMS, DNS, load balancer, registry, quota, monitoring and artifact storage can be shared failure points. Build an **explicit failure-domain inventory** by node, rack, zone, region, account/subscription and third-party dependency. Verify each component's actual scope with current provider documentation.

## 4. Linux host dependency discovery — [SAFE-READ]
Run on an authorized GPU host. These commands inspect state but can expose internal details.
```bash
uname -r
cat /etc/os-release
lscpu
lsblk
df -h
ip -br addr
ip route
systemctl --failed
command -v nvidia-smi && nvidia-smi -L || true
command -v nvidia-smi && nvidia-smi topo -m || true
command -v nvidia-smi && nvidia-smi --query-gpu=index,name,driver_version,memory.total --format=csv || true
```
On CPU-only systems, missing `nvidia-smi` is expected. Use approved tooling to inspect host logs for Xid/ECC and driver issues; avoid disclosing hostnames, private routes, secrets or tenant data in public documentation.

## 5. Kubernetes dependency discovery — [SAFE-READ]
```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -A -o wide
kubectl get daemonsets -A
kubectl get deployments,statefulsets,services,endpointslices -A
kubectl get networkpolicy -A
kubectl get resourcequota -A
kubectl get events -A --sort-by=.lastTimestamp
```
Inspect `kubectl describe node APPROVED_GPU_NODE` for GPU allocatable resources, taints, capacity and events. Device-plugin logs and GPU Operator resources vary by installation; locate actual namespace first. Do not assume all GPU workloads use the resource name `nvidia.com/gpu`.

## 6. Cloud dependencies
**AWS/EKS:** map EC2 GPU instances, managed node groups/Karpenter, VPC subnets, security groups, IAM/Pod Identity, ECR, S3, EBS, FSx for Lustre, EFA where supported, Route 53, load balancers, CloudWatch and quotas. Separate zone and regional dependencies.

**Azure/AKS:** map GPU VM SKUs/node pools, VNet/NSGs, managed/workload identity, Azure Container Registry, Blob, managed disks, load balancers, Azure Monitor, Key Vault and GPU SKU quotas. Verify regional availability and support for GPU features and RDMA.

**Cross-cloud:** do not assume shared cloud identity, image registries or object stores are independent; record common third-party providers, network peering, DNS and secrets systems.

## 7. Build the dependency register
| Field | Example |
|---|---|
| service ID | ai-inference-prod |
| dependency ID | regional-model-registry |
| direction | ai-inference-prod → regional-model-registry |
| type | startup |
| owner | ML platform |
| failure domain | region |
| SLO impact | cold-start failure |
| detection | failed artifact pull |
| fallback | pre-warmed signed artifact cache |
| recovery runbook | RB-005-C |
| verification | synthetic cold-start canary |
| last reviewed | date and change ticket |

Record edge criticality, observed behavior under failure and confidence. Keep a separate **owner of the dependency** and **owner of the consumer**. Maintain graph versions alongside infrastructure changes.

## 8. Failure-mode analysis
**Scenario A — GPU device plugin failure:** Node Ready but GPU allocatable count is zero. Follow workload → scheduler → device plugin → driver → host. GPU platform SRE owns remediation; Kubernetes SRE owns scheduling validation.

**Scenario B — Cold-start registry outage:** Existing pods serve but replacement pods cannot load model weights. Dependency is startup-critical, not necessarily runtime-critical. Test pinned digest and warm cache without bypassing signatures.

**Scenario C — RDMA network degradation:** Training ranks block on NCCL while GPU utilization falls. Follow job → rank → fabric → interface → network ownership. Correlate per-rank logs and interconnect metrics.

**Scenario D — Checkpoint storage outage:** Training continues temporarily but cannot meet RPO. Halt unsafe checkpoint claims; verify restored checkpoint including optimizer state and data cursor.

**Scenario E — IAM/KMS expiry:** Model download fails after token refresh. Preserve least privilege and investigate token lifetime, KMS policy and service identity; never grant broad permissions as a workaround.

**Scenario F — Observability blackout:** Production data plane may remain healthy but paging fails. Use independent synthetic health probes and declare reduced observability; abort planned chaos experiments.

## 9. Troubleshooting matrix
| Symptom | Dependency candidate | Evidence | Escalation |
|---|---|---|---|
| GPU invisible | driver/device plugin | allocatable GPU, driver health | GPU/Kubernetes SRE |
| pod Pending | quota/scheduler/zone | events, taints, quota | Kubernetes/cloud SRE |
| model pull fails | registry/KMS/network | pull logs, auth decisions | ML platform/security |
| NCCL timeout | fabric/rank failure | rank logs, link metrics | HPC/network SRE |
| checkpoint restore fails | object store/manifest | checksum, restore logs | storage/DBRE |
| 503 after zone outage | N-1 capacity, gateway | queue, endpoints, SLO | inference/platform SRE |
| metrics missing | exporter/collector | telemetry health, probe | observability SRE |

## 10. Runbooks
**RB-005-A — Dependency inventory:** export sanitized component list, owners, versions, zone/region and service-to-dependency edges; review with service owners.

**RB-005-B — Blast-radius triage:** identify failing component and all reverse-dependent consumers; distinguish observed impact from theoretical exposure; prioritize critical tenants.

**RB-005-C — Registry or KMS outage:** preserve approved signed artifacts, test warm-cache behavior, verify scoped identity and pause unsafe rollouts.

**RB-005-D — GPU or NCCL fault:** inspect driver/device-plugin and network/rank diagnostics; isolate the failure domain only with approved change.

**RB-005-E — Dependency recovery:** verify consumer-level SLO, data integrity, signed model revision, checkpoint consistency and tenant isolation before closure.

**RB-005-F — Change governance:** compare dependency graph before and after Terraform/GitOps changes, flag new single points of failure, require owner signoff and update runbooks.

## 11. 100-point acceptance assessment
| Area | Points |
|---|---:|
| Dependency inventory and ownership | 20 |
| Linux, Kubernetes and cloud discovery | 20 |
| Failure-domain and blast-radius analysis | 20 |
| Recovery, security and SLO controls | 20 |
| Executable lab, evidence and runbooks | 20 |
| **Total** | **100** |

Educational pass >=90/100 with no critical unowned dependency, tenant/security violation or invalid recovery assertion. Offline graph analysis cannot prove live failure behavior.

## 12. Executable lab
[Chapter 005 — Dependency Graph and Blast-Radius Lab](../labs/005-linux-kubernetes-cloud-dependency-mapping-check.md) implements cycle detection, reverse-dependency impact, critical-owner checks and failure-domain analysis in Python 3.11+ with pytest.

## References
- [Kubernetes Nodes](https://kubernetes.io/docs/concepts/architecture/nodes/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [NVIDIA Data Center Documentation](https://docs.nvidia.com/datacenter/)
- [Google SRE Workbook — Incident Response](https://sre.google/workbook/incident-response/)

**Next:** Chapter 006 — AI Infrastructure Reliability, SLOs and Error Budgets.
