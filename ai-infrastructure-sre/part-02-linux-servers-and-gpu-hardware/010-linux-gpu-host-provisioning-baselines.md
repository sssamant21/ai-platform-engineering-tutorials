# Chapter 010 — Linux GPU Host Provisioning and Baselines

**Part 02 — Linux, Servers and GPU Hardware · Chapter 010 of 100**  
**Status:** Published. Offline lab tests, real GPU validation, independent security review and production acceptance pending.

## Objectives
Provision a repeatable GPU Linux host; define immutable host configuration and compatibility contracts; validate kernel, firmware, driver, container runtime, GPU operator, NUMA, storage and networking; detect drift; plan staged rollout and rollback; troubleshoot production incidents safely.

## 1. Reference architecture
```text
Git + IaC + signed image catalog
             |
        change approval
             |
    cloud-init / image builder
             |
        Linux GPU node
  +----------+----------+
  | kernel / firmware   |
  | CPU NUMA / PCIe     |
  | NVIDIA driver       |
  | container runtime   |
  | network / storage   |
  | GPU telemetry       |
  +----------+----------+
             |
 Kubernetes kubelet + device plugin / GPU Operator
             |
        GPU workloads
             |
       SLO / security / cost
```
A host is not ready because SSH works or the Kubernetes Node condition is Ready. GPU readiness additionally requires correct PCIe enumeration, healthy driver, expected GPU count, compatible container runtime and device-plugin resources, network/storage access, tenant isolation and observed workload success.

## 2. Supported provisioning approaches
| Pattern | Advantages | Risks |
|---|---|---|
| Immutable golden image | fast consistent startup, controlled promotion | stale driver/kernel if not rebuilt |
| Base image + cloud-init | flexible cloud deployment | drift, boot latency, partial failure |
| Image + configuration management | auditable desired state | race with driver/operator ownership |
| Managed GPU node image | provider-supported integration | release cadence, version constraints |
| Bare metal PXE + config management | topology and firmware control | BMC, network boot and replacement complexity |

Choose **one owner for driver installation**: host image, node bootstrap or NVIDIA GPU Operator, according to the validated platform design. Installing drivers in multiple layers causes conflicts. Keep the host image, driver, kernel and CUDA user-space compatibility matrix versioned; never assume latest versions are mutually compatible.

## 3. Provisioning contract
| Control | Required evidence |
|---|---|
| OS and kernel | distro/version, supported kernel, security patch baseline |
| Hardware | CPU, NUMA, GPU model/count, PCIe, ECC/Xid health |
| Driver and CUDA | tested driver version and supported runtime compatibility |
| Container runtime | containerd/CRI configuration, GPU runtime integration |
| Kubernetes | kubelet version, GPU plugin/operator, GPU allocatable resources |
| Network | DNS, MTU, firewall/SG/NSG, NIC/RDMA if applicable |
| Storage | local NVMe, dataset path, checkpoint permissions, capacity |
| Identity | node role, workload identity, least privilege, no static credentials |
| Telemetry | node exporter, DCGM, logs, synthetic GPU job, alert route |
| Reliability | N-1 capacity, drain/rollback, change window, recovery owner |
| Governance | signed image, vulnerability scan, change ID, artifact retention |

Version pinning must include the actual tested matrix, not just a major version. EKS and AKS managed node pools have provider-specific supported images and driver behaviors; validate exact VM/instance SKU, OS and cluster release before rollout.

## 4. Hands-on Lab A — Read-only Linux baseline
On an **authorized** GPU host:
```bash
uname -a
cat /etc/os-release
lscpu -e=CPU,NODE,SOCKET,CORE
lsblk -o NAME,TYPE,SIZE,ROTA,MOUNTPOINTS
df -hT
free -h
ip -br addr
ip route
systemctl --failed
journalctl -p err -b --no-pager | tail -n 50
command -v nvidia-smi && nvidia-smi -L || true
command -v nvidia-smi && nvidia-smi --query-gpu=index,name,driver_version,pci.bus_id,temperature.gpu --format=csv || true
```
Collect a sanitized UTC evidence bundle and compare it to the approved image manifest. Logs and routes may contain sensitive identifiers; never publish them unredacted.

## 5. Hands-on Lab B — GPU/Kubernetes readiness
```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -A -o wide
kubectl get daemonsets -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe node APPROVED_GPU_NODE
```
Inspect allocatable GPU resources and the actual device-plugin namespace. A read-only inventory cannot verify device isolation or workload execution. A separately approved staging smoke test should run a pinned GPU container image, verify GPU enumeration and a small compute kernel, record logs, then delete the job via the authorized deployment workflow.

## 6. Hands-on Lab C — Baseline manifest and drift
Maintain an environment-specific JSON manifest containing: node ID, image digest, OS/kernel, driver, GPU count, GPU operator version, container runtime, security patch baseline, NTP/time sync, monitoring status, disk capacity, approved owner and rollback artifact. The companion Python lab performs fail-closed synthetic checks and produces deterministic JSON results.

**Drift classifications:** security-critical (unsigned image, missing tenant isolation, leaked credentials), availability-critical (missing GPU, failed driver, lost telemetry), performance-critical (PCIe/NUMA or network regression), informational (approved metadata changes). Any security-critical drift is a hard stop even if performance SLOs pass.

## 7. Production rollout workflow
1. Open a change with owner, affected zones, expected GPU capacity, SLO and rollback.
2. Build signed image with pinned packages; scan for vulnerabilities and record SBOM/provenance.
3. Validate on an isolated staging GPU node, including reboot, GPU diagnostics, synthetic workload, storage and network.
4. Canary one node in a single zone; ensure N-1 capacity and bounded admission before drain.
5. Confirm kubelet Ready **and** GPU allocatable, DCGM health, inference/training smoke tests and paging.
6. Roll one failure domain at a time; stop on Xid/ECC, missing GPU, latency regression or security gate failure.
7. Record final version inventory, incident/change evidence and approval.

Never blindly restart, drain or replace production GPU nodes without authorization and capacity analysis. A driver update may require reboot and can interrupt long-running training.

## 8. Incident simulations
**A — Driver missing after kernel update:** `nvidia-smi` fails; check module load, Secure Boot signing, kernel headers and supported driver matrix. Roll back approved image.

**B — GPU node Ready, zero allocatable:** investigate device plugin, operator, driver and runtime configuration; isolate change and verify a GPU smoke job.

**C — Cloud-init partial failure:** node joins cluster before storage/network/telemetry is ready. Enforce bootstrap gates and quarantine incomplete nodes.

**D — GPU Xid/ECC regression:** preserve host diagnostics, isolate the node, escalate hardware/vendor and avoid repeated unsafe retries.

**E — Container runtime mismatch:** pod cannot initialize GPU libraries. Compare driver/runtime/toolkit and GPU operator ownership; test pinned known-good image.

**F — Rollout causes zone capacity shortage:** verify N-1 headroom, stop additional drains and apply admission policy rather than overcommitting.

**G — Telemetry absent:** stop canary promotion, use independent probes and restore DCGM/collector/pager delivery.

## 9. Troubleshooting matrix
| Signal | Candidate | Evidence | Owner |
|---|---|---|---|
| no NVIDIA devices | PCIe/driver | lspci, nvidia-smi, kernel logs | GPU host SRE |
| kernel module rejected | signing/version | dmesg, module metadata | Linux/GPU SRE |
| zero GPU allocatable | device plugin/operator | DaemonSet, node capacity | Kubernetes/GPU SRE |
| CUDA init fails | compatibility/runtime | driver/toolkit logs | GPU platform |
| GPU throttling | cooling/power | temperature/power telemetry | hardware/cloud |
| workload Pending | quota/taints | events, node affinity | Kubernetes |
| dataset unavailable | mount/identity/network | filesystem, IAM, DNS | storage/cloud |
| missing metrics | exporter/collector | scrape, synthetic probe | observability |

## 10. Operational runbooks
**RB-010-A — Host admission:** confirm signed image, version matrix, OS patching, time sync, GPU health, runtime, network, storage, identity and monitoring.

**RB-010-B — GPU driver failure:** preserve logs, inspect driver/kernel compatibility, quarantine affected host and restore approved version with change control.

**RB-010-C — Node readiness:** correlate kubelet, GPU plugin/operator, allocatable count, taints and synthetic GPU job; do not equate node Ready with usable GPU.

**RB-010-D — Safe maintenance:** verify workload drain semantics, checkpoint integrity, N-1 headroom, staged rollout, abort conditions and rollback artifact.

**RB-010-E — Security deviation:** block promotion, preserve evidence, involve security and validate signed artifacts/least privilege before reopening.

**RB-010-F — Post-change acceptance:** compare before/after SLO, GPU telemetry, performance, version drift, cost, incident history and recovery.

## 11. Final 100-point assessment
| Domain | Points |
|---|---:|
| Provisioning architecture and compatibility | 20 |
| Linux/GPU/Kubernetes discovery | 20 |
| Security, identity and baseline controls | 20 |
| Incidents, maintenance and rollback | 20 |
| Executable lab and evidence | 20 |
| **Total** | **100** |

Pass >=90 with no critical security, GPU readiness or recovery blocker. Publication is not evidence that a physical node has passed acceptance.

## 12. Companion executable lab
[Chapter 010 — Linux GPU Host Baseline Validator](../labs/010-linux-gpu-host-provisioning-baselines-check.md) contains a Python 3.11+ validator and pytest suite. Synthetic inputs cannot validate real hardware or cloud provisioning.

## References
- [NVIDIA Driver Installation Guide](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [Kubernetes Node Status](https://kubernetes.io/docs/reference/node/node-status/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [Linux kernel documentation](https://docs.kernel.org/)

**Next:** Chapter 011 — NVIDIA Driver Lifecycle and Compatibility.
