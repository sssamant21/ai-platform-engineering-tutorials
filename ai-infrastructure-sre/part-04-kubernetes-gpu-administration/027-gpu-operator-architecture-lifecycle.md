# Chapter 027 — GPU Operator Architecture and Lifecycle

**AI Infrastructure Engineering & SRE · Part 04 — Kubernetes GPU Administration · Chapter 027 of 100**

**Status:** Published documentation; offline tests, live GPU staging, security review, and production acceptance remain pending.

## Learning objectives

- Explain NVIDIA GPU Operator's reconciliation model, operands, dependencies, and lifecycle.
- Distinguish GPU Operator management from independently installed device plugins, host drivers and toolkit components.
- Plan safe installation, upgrades, driver management, node replacement and rollback across EKS, AKS and on-premises clusters.
- Troubleshoot operator reconciliation, operand DaemonSets, NVML, MIG, DCGM, CDI/runtime and CUDA initialization.
- Implement a fail-closed offline change gate and conduct controlled staging game days.

## 1. Architecture

```text
Cluster administrator / GitOps / Helm release
                    |
           NVIDIA GPU Operator
          ClusterPolicy reconciliation
                    |
       +------------+------------+
       |            |            |
    Driver DS   Toolkit DS   Device-plugin DS
       |            |            |
    host kernel  containerd   kubelet device API
       |            |            |
       +------- GPU devices -----+
                    |
           GPU Feature Discovery
           DCGM / DCGM Exporter
           MIG Manager (when used)
           Node Status Exporter / validators
                    |
         Kubernetes node resources
                    |
            CUDA workload pods
```

Actual operands and their namespaces vary by GPU Operator version, operating system, runtime, and `ClusterPolicy`. Some platforms use preinstalled host drivers, virtualized GPUs, managed driver extensions, or cloud-managed device-plugin integration. **Do not assume the Operator must install drivers everywhere.** Fabric Manager and additional networking components may be managed separately or by supported operator configurations.

## 2. Ownership and reconciliation

GPU Operator is an operator/controller, not the NVIDIA device plugin itself. It manages desired configuration and operand lifecycles through Kubernetes resources and platform-specific controls. Inspect its Helm release, operator deployment, `ClusterPolicy`, DaemonSets, ConfigMaps, validation pods, node labels, and events before making changes.

### Management decision table

| Component | Operator-managed option | Alternative |
|---|---|---|
| NVIDIA driver | driver operand where supported | preinstalled/managed host driver |
| NVIDIA Container Toolkit | toolkit operand | managed host runtime integration |
| device plugin | operator operand | existing managed plugin |
| GPU Feature Discovery | operator operand | external discovery controller |
| DCGM exporter | operator operand | existing observability stack |
| MIG Manager | optional operand | platform-controlled MIG lifecycle |

**Critical:** installing an additional independent device-plugin DaemonSet alongside an operator-managed one can create conflicts. Use one accountable owner for each operand.

## 3. Prerequisites and compatibility

Record before deployment:

1. Kubernetes version, distribution, node OS and kernel, containerd/CRI-O version.
2. GPU SKU, PCIe/NVLink/MIG capabilities and supported driver branch.
3. GPU Operator chart and application version, device plugin, NVIDIA Container Toolkit, CUDA/framework versions.
4. Whether host driver installation is enabled or disabled.
5. RuntimeClass/CDI strategy, Pod Security/privileged-operand approval and admission controls.
6. Network/image-registry reachability, proxy, DNS, certificates and image digests.
7. Cluster autoscaler/Karpenter/AKS node-pool behavior and GPU node taints.
8. GPU capacity, N-1 recovery, tenant scheduling and rollback strategy.

Compatibility is **version-specific**; use the NVIDIA GPU Operator platform support matrix and vendor cloud guidance for the deployed release. Never copy an unverified driver/operator version combination into production.

## 4. Read-only inventory lab

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get deployments,daemonsets,pods -A | grep -Ei 'gpu-operator|nvidia|dcgm|mig'
kubectl get clusterpolicies.nvidia.com -o yaml
kubectl get events -A --sort-by=.lastTimestamp
helm list -A | grep -Ei 'gpu-operator|nvidia'
```

Some installations do not expose the same CRD names or may not have `helm` available. An absent resource is a discovery finding, not proof of a broken cluster.

After identifying the actual namespace and resource names:

```bash
kubectl describe deployment APPROVED_OPERATOR_DEPLOYMENT -n APPROVED_NAMESPACE
kubectl logs deployment/APPROVED_OPERATOR_DEPLOYMENT -n APPROVED_NAMESPACE --since=30m
kubectl describe daemonset APPROVED_OPERAND_DAEMONSET -n APPROVED_NAMESPACE
kubectl get pods -n APPROVED_NAMESPACE -o wide
kubectl describe node APPROVED_GPU_NODE
```

On the authorized host:

```bash
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version --format=csv
journalctl -u kubelet -b --no-pager | tail -n 100
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|ECC|AER' | tail -n 100
```

Correlate the node name, physical GPU UUID, MIG UUID/profile, driver, container runtime, plugin-advertised resource keys, pod identity and UTC timestamp.

## 5. Controlled installation plan

Do **not** run these commands against production without a reviewed change. The following is a template for a dedicated staging cluster:

```bash
# Staging only; review the exact chart version and values before use.
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
helm show values nvidia/gpu-operator --version APPROVED_CHART_VERSION > reviewed-values.yaml
helm template gpu-operator nvidia/gpu-operator \
  --version APPROVED_CHART_VERSION \
  --namespace gpu-operator \
  -f reviewed-values.yaml > rendered-gpu-operator.yaml
```

Validate rendered resources for privileged DaemonSets, hostPath mounts, service accounts, RBAC, images, node selectors, tolerations and driver/toolkit configuration. Do not apply the chart until security, platform, application and SRE owners approve. Use a pinned chart and reviewed image digests where supported.

A later approved staging deployment might use `helm upgrade --install` with the reviewed values. **Never blindly reuse `driver.enabled=true` on nodes with cloud-managed drivers.**

## 6. Reconciliation and upgrade lifecycle

**Desired state:** Helm/ClusterPolicy configuration declares operands and settings. **Observed state:** pods, DaemonSets, node labels, driver, runtime and GPU inventory report reality. Reconciliation can fail when an operand cannot start, required host permissions are missing, driver/OS compatibility is broken, or a node is draining.

Safe upgrade sequence:

1. Inventory current chart/operator, operands, node OS, driver, toolkit and GPU resources.
2. Read version-specific upgrade/release notes and known issues.
3. Freeze unrelated GPU platform changes and preserve workload checkpoints.
4. Confirm N-1 capacity, PDB constraints, node drain and rollback.
5. Canary one isolated GPU node pool or staging cluster.
6. Upgrade chart/operator using approved immutable artifacts.
7. Verify operator reconciliation and operand DaemonSets.
8. Verify `nvidia.com/gpu`/MIG resources, CDI/runtime, CUDA smoke and NCCL where relevant.
9. Compare GPU health, SLO, training progress and observability.
10. Expand rollout only after explicit acceptance; otherwise roll back configuration and recover affected nodes by supported procedures.

**Rollback caveat:** rolling back a Helm chart does not necessarily roll back a host driver, kernel module, node image, MIG partition or firmware. Define a component-specific rollback plan.

## 7. Driver management and host reboots

GPU driver operands may build/load kernel modules or install drivers through supported mechanisms. Kernel version mismatch, Secure Boot, driver/container-toolkit drift, container-runtime reload, incompatible node images and cloud-provider managed drivers are common failure domains. Reboots and driver changes can interrupt GPU workloads. Coordinate checkpointing, drain and node replacement; do not restart shared GPU hosts as a first diagnostic action.

## 8. MIG and GPU sharing

GPU Operator may deploy MIG Manager where supported. MIG strategy affects the NVIDIA device plugin's resource keys and available capacity. Time-slicing and MPS are **not** equivalent to MIG isolation. Resource requests may represent shared logical access rather than an entire physical GPU under supported sharing configurations. Require tenant isolation, fairness, GPU UUID mapping, profile inventory and explicit change approval.

## 9. Observability and SLOs

| Signal | Evidence | Interpretation |
|---|---|---|
| operator reconciliation status | ClusterPolicy/operator logs | desired-state convergence |
| operand DaemonSet ready/desired | Kubernetes status | component rollout progress |
| plugin registration | kubelet/plugin logs | scheduler resource exposure |
| GPU allocatable drift | node status | supply regression |
| CUDA smoke and UUID | staging workload | end-to-end runtime proof |
| Xid/ECC/temperature | DCGM/driver | hardware health |
| driver/toolkit version | host inventory | compatibility |
| telemetry freshness | monitoring | trustworthy alerting |
| MIG strategy/profile | node/Operator inventory | tenant resource correctness |
| checkpoint age / job SLO | application | reliability impact |

A Running operator pod does not guarantee a healthy GPU. Establish platform-specific alert thresholds and correlate to application progress.

## 10. Twelve troubleshooting scenarios

1. Operator deployment unavailable — inspect namespace, image pull, RBAC and admission.
2. ClusterPolicy reconciliation stalled — inspect status, events and operand logs.
3. Driver DaemonSet CrashLoopBackOff — compare kernel/driver compatibility and host logs.
4. Toolkit DaemonSet failure — verify runtime configuration and security permissions.
5. Device plugin Running but no GPU resources — inspect NVML, registration and node status.
6. CUDA unavailable in workload — trace device allocation, CDI/runtime and driver mounts.
7. Duplicate plugin installations — reconcile ownership and remove conflict only via change.
8. MIG Manager changes resources unexpectedly — verify profile, strategy and tenant impact.
9. GPU node NotReady after upgrade — correlate kubelet, runtime, driver and cloud node lifecycle.
10. GPU telemetry missing — inspect DCGM/exporter, permissions and scraping.
11. Operator upgrade succeeds but application regresses — compare CUDA/NCCL canary and SLO.
12. Helm rollback does not restore driver — execute host-level rollback or node replacement.

## 11. Ten production incident simulations

1. GPU Operator chart incompatible with cluster version.
2. Driver operand fails after kernel upgrade.
3. Secure Boot prevents driver module loading.
4. Toolkit/CDI change hides devices from containers.
5. Device-plugin operand fails registration.
6. MIG Manager changes advertised resources during workload placement.
7. Duplicate manual plugin and Operator plugin conflict.
8. GPU node autoscaling races operand readiness.
9. DCGM exporter becomes unavailable during GPU failure.
10. Chart rollback leaves an incompatible host driver.

For each drill record UTC timeline, GPU UUID, affected node/pod, desired/observed state, checkpoint, rollback, SLO impact and RCA/COA.

## 12. Eight operational runbooks

**RB-027-A — Operator inventory:** identify release, ClusterPolicy, operand ownership and versions.

**RB-027-B — Reconciliation stall:** inspect controller status, events, permissions, image pulls and DaemonSets.

**RB-027-C — Driver failure:** compare OS/kernel, Secure Boot, driver logs and managed-driver policy.

**RB-027-D — Toolkit/CDI failure:** validate container runtime, toolkit, Allocate strategy and canary.

**RB-027-E — Plugin/MIG drift:** reconcile device-plugin resources, MIG profiles and tenant scheduling.

**RB-027-F — Safe upgrade:** canary, checkpoint, N-1 capacity, drain, compatibility and post-change validation.

**RB-027-G — Rollback and node replacement:** distinguish chart rollback from host driver and node-image recovery.

**RB-027-H — RCA and acceptance:** verify reconciliation, resource supply, CUDA smoke, telemetry, SLO and signoff.

## 13. Executable companion lab

[Lab 027 — GPU Operator Lifecycle and Upgrade Gate](../labs/027-gpu-operator-lifecycle-check.md) implements an offline Python 3.11+ validator with pytest cases for component ownership, reconciliation, compatibility, operand health, driver/toolkit/plugin readiness, MIG, canary, SLO and rollback. The lab is **synthetic** and does not change a cluster.

## 14. 100-point assessment

| Domain | Points |
|---|---:|
| Operator architecture and component ownership | 20 |
| Compatibility, deployment and upgrades | 20 |
| Runtime, MIG and resource discovery | 20 |
| Troubleshooting, incidents and runbooks | 20 |
| Executable lab and staging evidence | 20 |
| **Total** | **100** |

Pass >=90 with no critical security, correctness, driver/runtime, rollback or SLO blocker.

## References

- [NVIDIA GPU Operator documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [NVIDIA GPU Operator platform support](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html)
- [NVIDIA GPU Operator GitHub](https://github.com/NVIDIA/gpu-operator)
- [NVIDIA Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)

**Next:** Chapter 028 — GPU Scheduling, Taints and Affinity.
