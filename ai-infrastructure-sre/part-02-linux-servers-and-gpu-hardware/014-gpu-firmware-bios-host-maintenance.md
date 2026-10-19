# Chapter 014 — GPU Firmware, BIOS and Host Maintenance

**Part 02 — Linux, Servers and GPU Hardware · Chapter 014 of 100**  
**Publication:** Complete. **Executable tests, hardware maintenance drills, security review and production acceptance:** Pending.

## Objectives and scope
Design an auditable GPU host maintenance lifecycle covering server BIOS/UEFI, BMC, GPU firmware, NIC firmware, NVMe firmware, CPU microcode, Secure Boot, kernel and NVIDIA driver compatibility. Collect read-only baselines, define change gates, perform staged maintenance and recovery, and distinguish firmware-induced regressions from application, driver, PCIe and Kubernetes failures.

Firmware changes can permanently alter device behavior or make a host unbootable. **All flashing, BIOS setting changes, reboots, BMC actions, GPU resets, cordons and drains require an approved change, vendor-supported procedure, capacity analysis and rollback/recovery plan.** The default labs below are read-only or synthetic.

## 1. Hardware and management architecture
```text
Git/IaC + signed artifacts + CMDB + change ticket
                       |
             maintenance controller
                       |
  +--------------------+--------------------+
  | BMC / Redfish / out-of-band management  |
  | UEFI/BIOS: PCIe, NUMA, IOMMU, SecureBoot|
  | CPU microcode, DIMMs, NIC, NVMe, GPU FW |
  | OS kernel + NVIDIA driver + containerd  |
  +--------------------+--------------------+
                       |
           GPU device plugin / Operator
                       |
         Kubernetes GPU workloads and SLO
                       |
           metrics, logs, rollback proof
```
A BMC can be reachable when the OS is down; BMC health does **not** imply host readiness. GPU firmware and board firmware are model-specific; cloud GPU VMs often expose neither BMC nor BIOS flash capability. For EKS/AKS managed nodes, use supported node-image/instance lifecycle mechanisms rather than assuming bare-metal controls.

## 2. Version and configuration dependencies
| Layer | Inventory | Change risk | Validation |
|---|---|---|---|
| BIOS/UEFI | vendor/model/version, PCIe, NUMA, IOMMU, Secure Boot | changed enumeration/topology | boot + topology diff |
| BMC | version, management interface, network ACL | remote management loss | out-of-band access |
| GPU firmware | GPU SKU, VBIOS/GSP versions where exposed | GPU initialization | vendor matrix + GPU smoke |
| NIC firmware | adapter model/version, RDMA mode | NCCL/RDMA degradation | fabric counters + bandwidth |
| NVMe firmware | model/version, namespace | checkpoint corruption or outage | I/O integrity + restore |
| CPU microcode | CPU family/revision | stability/performance change | OS and stress baseline |
| Kernel/driver | kernel, module signature, NVIDIA version | device not ready | nvidia-smi + CUDA |
| GPU Operator/runtime | operator/toolkit/device plugin | allocatable mismatch | staging pod |

Maintain a signed and approved compatibility matrix for each **exact server SKU**, GPU, NIC, firmware, OS, kernel, driver and operator combination. Do not invent universal supported version pairings or assume the latest version is compatible.

## 3. Read-only discovery
On an **authorized Linux host**:
```bash
date -u
hostnamectl
uname -r
cat /etc/os-release
lscpu | grep -Ei 'model name|socket|numa'
lspci -tv
lspci -nn | grep -Ei 'nvidia|ethernet|infiniband|nvme'
command -v dmidecode && sudo -n dmidecode -t bios -t system || true
command -v fwupdmgr && fwupdmgr get-devices || true
command -v mokutil && mokutil --sb-state || true
command -v nvidia-smi && nvidia-smi -q | grep -Ei 'VBIOS Version|GSP Firmware Version|Driver Version' || true
lsblk -o NAME,TYPE,SIZE,MODEL
journalctl -k -b --no-pager | grep -Ei 'AER|pcie|nvrm|xid|firmware' | tail -n 100
```
Some commands require approved privileged read access; `sudo -n` intentionally fails instead of prompting. Not every GPU exposes VBIOS/GSP data. `fwupdmgr get-devices` enumerates supported devices; it does **not** install updates. For BMC inventory, use a vendor-approved **read-only** Redfish interface with scoped credentials, TLS validation and audit logging; never embed credentials in examples or logs.

### Kubernetes correlation
```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl describe node APPROVED_GPU_NODE
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl get daemonsets -A
```
Compare GPU allocatable resources, Xid/ECC, driver health, device-plugin status and application SLOs before and after any authorized change.

## 4. Maintenance design: preflight → canary → waves → acceptance
**Preflight gates:** approved ticket, vendor advisory and signed artifact; validated version matrix; current firmware and settings backup; independent BMC/console recovery path; verified checkpoints; N-1 GPU capacity and quota; node and zone impact; no conflicting driver owner; rollback or documented forward-recovery path; alerting and on-call.

**Staging:** reproduce target SKU, apply vendor-supported change in a maintenance window, reboot when required, inspect POST/BMC logs, enumerate GPUs and NICs, validate PCIe gen/width under load, NUMA mapping, ECC/Xid, CUDA and NCCL, RDMA, NVMe/checkpoint recovery, Secure Boot and app SLO.

**Canary:** one node per approved failure domain, with bounded drain and checkpoint verification. Stop if the host fails POST, loses GPU/NIC count, changes expected topology, loses telemetry, raises critical Xid/ECC, fails workload smoke, or burns SLO/error budget.

**Rollout:** use serial waves and explicit acceptance between zones. A firmware downgrade is not always supported; a cold spare, node replacement or vendor recovery image may be the only viable recovery. Confirm this *before* flashing.

**Close:** preserve before/after manifests, UTC timeline, hashes, approvals, logs, benchmark comparison, incident references and asset updates.

## 5. Performance and reliability acceptance
Compare measured, repeatable baselines rather than relying on a successful boot:
- GPU UUID/count, PCIe link generation and width under controlled load, NUMA and GPU/NIC affinity.
- Host-to-device and peer transfer bandwidth; NCCL collective bandwidth where relevant.
- GPU temperature/power, ECC/Xid, PCIe AER, NIC/RDMA errors and NVMe latency.
- Inference TTFT/TPOT and errors or training step time/checkpoint restore; use workload-specific SLOs.
- Kubernetes capacity/allocatable GPU, operator/device plugin, time sync, storage/network and telemetry freshness.
- Firmware/BIOS security posture, BMC access isolation, Secure Boot and measured audit evidence.

## 6. Failure simulations
1. **BIOS update changes NUMA/PCIe topology:** compare `lspci`, `nvidia-smi topo -m` and workload throughput; stop promotion.
2. **GPU missing after firmware update:** inspect POST, PCIe enumeration, NVRM and vendor release notes; escalate.
3. **BMC unavailable during maintenance:** stop disruptive actions; restore approved out-of-band recovery before proceeding.
4. **Secure Boot blocks driver module:** compare signing state and module metadata; restore approved image.
5. **NIC firmware causes RDMA regression:** correlate link counters, rank logs and topology; isolate canary.
6. **NVMe firmware causes checkpoint errors:** halt rollout, validate backups and restore in staging.
7. **Thermal/power regression:** compare fan/power policy, temperatures and clocks; involve hardware owner.
8. **Rollout exhausts N-1 capacity:** halt additional drains; restore capacity and admission controls.

## 7. Troubleshooting matrix
| Symptom | Evidence | Suspected boundary | Owner |
|---|---|---|---|
| host fails POST | BMC event log/console | BIOS/firmware | hardware/vendor |
| GPU absent | PCIe + NVRM logs | GPU firmware/PCIe | GPU host SRE |
| CUDA fails | kernel module + driver | Secure Boot/driver | Linux/GPU |
| NCCL slows | NIC/RDMA + topology | NIC firmware/fabric | network/HPC |
| checkpoint errors | NVMe/FS logs | storage firmware | storage |
| unexpected NUMA | lscpu/lspci/topo | BIOS settings | hardware |
| BMC access fails | management ACL/TLS | OOB network/BMC | security/hardware |
| node Ready, GPU absent | device plugin/allocatable | runtime integration | Kubernetes |

## 8. Production runbooks
**RB-014-A — Baseline inventory:** collect signed artifact versions, server SKU, BMC/BIOS settings, GPU/NIC/NVMe, driver and workload owners.

**RB-014-B — Preflight approval:** validate compatibility evidence, security review, N-1 capacity, checkpoint, out-of-band access, maintenance window and recovery.

**RB-014-C — Staging qualification:** execute vendor-supported firmware procedure on staging; verify POST, device enumeration, topology, GPU health and workload smoke.

**RB-014-D — Canary rollout:** approve single-node isolation/drain, monitor error budget, stop on critical gate and preserve logs.

**RB-014-E — Failed boot recovery:** use approved console/vendor recovery, protect data and escalate; never attempt ad-hoc flash commands.

**RB-014-F — Regression rollback:** restore previous approved host image/settings when supported, or use vendor-assisted recovery/replacement.

**RB-014-G — Post-change acceptance:** confirm metrics, GPU/NIC counts, SLO, checkpoints, security controls and owner signoff.

**RB-014-H — RCA/COA:** record incident timeline, root-cause confidence, firmware versions, fleet blast radius, remediation and prevention.

## 9. Executable companion lab
[Chapter 014 — Firmware, BIOS and Maintenance Change Gate](../labs/014-gpu-firmware-bios-host-maintenance-check.md) contains a **synthetic, fail-closed** Python 3.11+ maintenance evaluator and pytest suite. It never flashes firmware or modifies a host.

## 10. Final assessment
| Domain | Points |
|---|---:|
| Firmware/BIOS architecture and compatibility | 20 |
| Inventory and security controls | 20 |
| Maintenance/change/rollback design | 20 |
| Incidents and operational runbooks | 20 |
| Executable lab and acceptance evidence | 20 |
| **Total** | **100** |

Pass >=90 **and** no critical security, capacity, recoverability or data-integrity gate failure.

## References
- [NVIDIA System Management Interface](https://docs.nvidia.com/deploy/nvidia-smi/)
- [NVIDIA Driver Installation](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/)
- [DMTF Redfish](https://www.dmtf.org/standards/redfish)
- [fwupd](https://fwupd.org/)
- [Kubernetes Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

**Next:** Chapter 015 — GPU Node Replacement and Recovery.
