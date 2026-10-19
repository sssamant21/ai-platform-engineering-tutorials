# Chapter 033 — GPU Operator and Device Plugin Troubleshooting

**Part 04 — Kubernetes GPU Administration | Chapter 033 of 100**

**Publication state:** Documentation published. Lab runtime, live GPU staging, security review and production acceptance require separate verification.

## Learning outcomes

Diagnose NVIDIA GPU Operator reconciliation, ClusterPolicy, driver and toolkit DaemonSets, containerd/CDI integration, device-plugin registration, MIG Manager, DCGM exporter and CUDA workloads on EKS/AKS. Distinguish API admission, scheduling, node, GPU driver, runtime and application failures. Execute safe, evidence-driven rollback and write production RCA/COA.

## 1. End-to-end architecture

```text
GPU Pod nvidia.com/gpu: 1
    | API admission / quota / PSA
    v
Kubernetes scheduler -> node allocatable GPU resources
    v
kubelet DeviceManager -> NVIDIA device plugin Allocate
    v
CRI (containerd/CRI-O) -> NVIDIA Container Toolkit / CDI
    v
container -> libcuda -> host NVIDIA kernel driver -> GPU
                                    |
                       Xid/ECC / NVML / DCGM

GPU Operator -> ClusterPolicy reconciliation
    | driver / toolkit / device-plugin / GPU Feature Discovery
    | MIG Manager / DCGM exporter / validator
    + readiness, status, DaemonSet and logs
```

**Critical distinction:** `kubectl get nodes` showing Ready does not establish driver health or GPU allocatable resources. A GPU allocated to a pod does not establish successful CUDA initialization or application correctness. NVIDIA GPU Operator components may be intentionally disabled when a cloud-managed image already supplies a compatible driver or toolkit; inspect the effective ClusterPolicy rather than assuming missing DaemonSets are failures.

## 2. Symptom-to-layer mapping

| Symptom | First boundary | Evidence |
|---|---|---|
| Pod rejected by API | RBAC/PSA/quota | API error and admission audit |
| Pod Pending | scheduler, taints, quota, GPU count | FailedScheduling events |
| Node Ready, zero GPUs | driver, plugin registration, MIG config | allocatable, device-plugin logs |
| Operator Degraded | ClusterPolicy/operand reconciliation | ClusterPolicy conditions, operator logs |
| Driver pod CrashLoopBackOff | kernel modules, secure boot, incompatible image | driver logs, kernel journal |
| Toolkit pod unhealthy | container runtime config | toolkit logs, CRI config |
| CreateContainerError | CDI, device injection, runtime | kubelet/containerd logs |
| CUDA initialization failure | libcuda/driver/runtime | application logs, version inventory |
| Wrong MIG resource key | MIG Manager and plugin strategy | resource names, MIG UUID/profile |
| Xid/ECC | GPU hardware/driver | kernel journal, NVML, DCGM |
| DCGM metrics missing | exporter/scrape/permissions | exporter logs, Prometheus targets |
| GPU present, NCCL hangs | ranks/network/topology | NCCL diagnostics, GPU/NIC inventory |

## 3. Read-only triage: first 15 minutes

Confirm context and blast radius before touching operator-managed resources:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -o json | jq -r '.items[] | [.metadata.name, (.status.capacity["nvidia.com/gpu"] // "none"), (.status.allocatable["nvidia.com/gpu"] // "none")] | @tsv'
kubectl get pods -A -o wide | grep -Ei 'nvidia|gpu|Pending|CrashLoop'
kubectl get events -A --sort-by=.lastTimestamp
kubectl get daemonsets -A
kubectl get clusterpolicies.nvidia.com -o yaml
```

If the ClusterPolicy CRD does not exist, first establish whether GPU Operator is installed. Avoid running `kubectl get ... -o yaml` on sensitive resources in public tickets. Select the actual namespaces and workload names discovered from the cluster:

```bash
kubectl describe pod APPROVED_GPU_POD -n APPROVED_NAMESPACE
kubectl logs -n OPERATOR_NAMESPACE deployment/OPERATOR_DEPLOYMENT --since=30m
kubectl logs -n OPERATOR_NAMESPACE daemonset/DEVICE_PLUGIN_DAEMONSET --since=30m --all-containers=true
kubectl describe node APPROVED_GPU_NODE
```

Not all GPU Operator releases use identical DaemonSet names or expose the same ClusterPolicy status schema. Read the installed version and effective CRDs first.

## 4. Host-side checks

On an authorized GPU node:

```bash
nvidia-smi -L
nvidia-smi --query-gpu=index,uuid,name,driver_version,memory.total --format=csv
nvidia-smi -q -d ECC
nvidia-smi topo -m
lsmod | grep nvidia
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|ECC|AER' | tail -n 100
journalctl -u kubelet -b --no-pager | tail -n 100
```

Check containerd/CRI and toolkit versions through approved host procedures. Where supported, inspect CDI inventory with `nvidia-ctk cdi list`; verify the command against the installed toolkit. **Never** treat `nvidia-smi` on the host as proof that a container sees the same devices.

## 5. GPU Operator lifecycle and ownership

The operator reconciles desired configuration into privileged platform operands. Investigate:
- Driver management enabled/disabled and cloud-managed driver ownership.
- Host OS/kernel/driver/CUDA compatibility, secure boot and kernel module loading.
- NVIDIA Container Toolkit and CDI integration with the actual CRI.
- Device plugin registration with kubelet and integer extended-resource advertisement.
- GPU Feature Discovery labels, MIG Manager profile changes and sharing strategy.
- DCGM exporter/validator status, image provenance, pull failures, node taints and Pod Security exceptions.
- ClusterPolicy reconciliation loops, resource conflicts, upgrade sequencing and stuck finalizers.

Avoid two competing driver managers, device plugins or toolkits on one node. A mixed strategy may be intentional only with an explicitly reviewed design.

## 6. Safe recovery decision tree

1. **API rejection?** Resolve admission, namespace quota, RBAC or Pod Security. Do not weaken global policy.
2. **Pending?** Verify resource key, node allocatable, taints, affinity, CPU/memory, quota and topology.
3. **GPU not advertised?** Inspect driver health, plugin logs, kubelet registration and MIG configuration.
4. **Container fails?** Inspect CDI/toolkit/CRI and driver/library compatibility; do not mount GPU devices manually as a shortcut.
5. **CUDA works but application fails?** Check GPU UUID, ECC/Xid, memory, NCCL transport, rank coordination and model SLO.
6. **Operator upgrade regression?** Halt rollout, preserve evidence, validate N-1 and checkpoints, and use a documented version-compatible rollback. Helm rollback alone may not undo host driver changes.

No destructive driver reset, host reboot, node drain, MIG repartition, plugin deletion or ClusterPolicy edit without authorized maintenance and impact analysis.

## 7. EKS, AKS and GPU platform specifics

**EKS:** distinguish GPU-ready AMI drivers from Operator-managed drivers, Karpenter node initialization, EC2 GPU SKU/quotas, node IAM and EFA/NCCL dependencies. **AKS:** identify AKS-managed GPU driver choices, node pool image, taints and GPU Operator integration. For either cloud, node Ready and GPU driver readiness are separate milestones. Driver/toolkit upgrades require tested compatibility with Kubernetes, container runtime and CUDA workloads.

## 8. Twelve troubleshooting scenarios

1. ClusterPolicy reports Degraded after a driver image update.
2. NVIDIA driver DaemonSet cannot load a kernel module.
3. Secure Boot prevents unsigned driver module loading.
4. Container Toolkit config diverges from containerd.
5. Device plugin is healthy as a pod but fails kubelet registration.
6. Node reports zero `nvidia.com/gpu` despite visible host GPUs.
7. Duplicate device plugins race to register.
8. MIG Manager reconfiguration changes resource keys unexpectedly.
9. GPU time-slicing advertises misleading logical capacity.
10. CDI allocation fails with CreateContainerError.
11. DCGM exporter stops publishing Xid/ECC metrics.
12. CUDA canary works but multi-node NCCL training hangs.

For each scenario: capture UTC timestamps, node name, GPU UUID, ClusterPolicy/Operator version, kernel/driver/toolkit/runtime versions, pod UID, logs, SLO, owner, rollback and RCA.

## 9. Ten production incident simulations

1. GPU Operator upgrade stalls half the nodes.
2. Managed driver conflicts with Operator driver operand.
3. Device plugin registration disappears after kubelet restart.
4. Containerd upgrade breaks CDI device injection.
5. MIG partition change strands queued inference pods.
6. GPU Xid causes CUDA failures during model rollout.
7. Missing DCGM telemetry hides unhealthy GPUs.
8. Operator namespace security hardening blocks privileged operands.
9. A GPU node drains before distributed training checkpoints.
10. Driver rollback restores GPU count but CUDA/NCCL correctness remains broken.

Simulations must be isolated to approved staging or conducted synthetically; do not inject hardware faults into production.

## 10. Eight operational runbooks

**RB-033-A — GPU discovery:** reconcile physical GPU inventory, node allocatable, device plugin and resource keys.

**RB-033-B — ClusterPolicy degraded:** inspect reconciliation conditions, operator events and operand readiness.

**RB-033-C — Driver failure:** verify kernel, secure boot, driver versions, Xid/ECC and vendor guidance.

**RB-033-D — Toolkit/CDI:** compare CRI/toolkit/plugin modes and test device injection on a staging canary.

**RB-033-E — MIG/sharing:** inventory profiles, workload resource requests, partition-change approvals and rollback.

**RB-033-F — Observability:** restore DCGM metrics, freshness, alert coverage and SLO correlation.

**RB-033-G — Safe maintenance:** preserve checkpoints, validate N-1, cordon/drain only with approval, canary and rollback.

**RB-033-H — RCA/COA:** verify GPU UUID, CUDA/NCCL, workload correctness, capacity, telemetry and signoff.

## 11. Hands-on labs

1. Capture read-only Kubernetes GPU inventory and ClusterPolicy.
2. Map driver/toolkit/plugin/validator/DCGM operand ownership.
3. Compare host NVML and Kubernetes allocatable GPU inventory.
4. Run a reviewed immutable-image CUDA canary on an approved staging node.
5. Exercise a synthetic missing device-plugin registration case.
6. Exercise a synthetic CDI mismatch and MIG resource-key drift.
7. Simulate a degraded operator upgrade and evaluate rollback readiness.
8. Verify N-1, checkpoint, workload SLO, metrics freshness and incident evidence.

[Companion Lab 033](../labs/033-gpu-operator-device-plugin-troubleshooting-check.md) provides an offline Python 3.11+ policy evaluator and pytest suite. It **does not** access a Kubernetes cluster or certify real GPU hardware.

## 12. Production acceptance (100 points)

| Category | Points |
|---|---:|
| Operator and device-plugin architecture | 20 |
| Driver/toolkit/CDI/MIG diagnostics | 20 |
| GPU hardware and application validation | 20 |
| Incident response, runbooks and rollback | 20 |
| Automated tests and staging evidence | 20 |
| **Total** | **100** |

Pass >=90 with no critical GPU correctness, tenant isolation, checkpoint, telemetry or rollback blocker.

## References

- [NVIDIA GPU Operator documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [NVIDIA Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [Kubernetes Debug Running Pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [Kubernetes Debug Nodes](https://kubernetes.io/docs/tasks/debug/debug-cluster/)

**Next: Chapter 034 — Kubernetes GPU Platform Capstone.**
