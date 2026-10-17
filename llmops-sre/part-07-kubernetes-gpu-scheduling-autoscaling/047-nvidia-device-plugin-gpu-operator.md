# Chapter 047 — NVIDIA Device Plugin and GPU Operator

**Part 07 — Kubernetes GPU Scheduling and Autoscaling | Chapter 047 of 100**

> **Status:** Written tutorial published. Live execution, independent technical/security review and operational acceptance pending. NVIDIA GPU Operator components, chart values, driver management and MIG strategy vary by version, GPU SKU, cloud provider and cluster policy. Never install or upgrade drivers/operators in production without change approval.

## Learning outcomes

Explain GPU device-plugin registration, NVIDIA GPU Operator reconciliation, driver/toolkit/DCGM dependencies, runtime and CDI integration, MIG resource strategies, cloud-managed GPU nodes, security, upgrade and recovery. Build an auditable diagnosis and acceptance plan.

## 1. Architecture

```text
Kubernetes API / GPU Pod requesting nvidia.com/gpu
                  │
          Kubernetes scheduler
                  │
         kubelet device-plugin API
                  │
         NVIDIA device plugin DaemonSet
                  │
       NVIDIA Container Toolkit / CDI
                  │
           NVIDIA driver / NVML
                  │
       Physical GPU / MIG partitions
                  │
GPU Operator (optional management control plane)
   ├─ driver management (optional)
   ├─ toolkit management (optional)
   ├─ device plugin
   ├─ GPU Feature Discovery / Node Feature Discovery
   ├─ DCGM exporter and monitoring components
   └─ MIG Manager and validation components (when enabled)
```

GPU Operator is **not required** on every cluster. Some EKS/AKS/GKE GPU node images manage drivers and toolkits independently. Do not let both a cloud image and an Operator driver DaemonSet independently overwrite the same driver stack.

## 2. Responsibility boundaries

| Layer | Function | SRE validation |
|---|---|---|
| physical GPU | compute/VRAM/ECC | NVML, DCGM |
| NVIDIA driver | kernel-to-device compatibility | driver version, kernel logs |
| Container Toolkit/CDI | inject GPU devices/libraries | runtime configuration |
| device plugin | advertise extended resources | node allocatable, plugin logs |
| GPU Operator | reconcile GPU software components | ClusterPolicy/DaemonSets |
| scheduler | assign Pod to suitable node | events, allocatable |
| workload runtime | CUDA/NCCL/model execution | synthetic inference |

Driver healthy on host does not prove a container can use CUDA; node allocatable GPU count does not prove sufficient VRAM.

## 3. Safe-read discovery

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl get daemonsets -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get crd | grep -Ei 'nvidia|clusterpolic'
```

Discover operator namespace and names instead of assuming `gpu-operator` or `nvidia-device-plugin` is installed.

## 4. Device-plugin registration

The plugin registers with kubelet and advertises resources such as `nvidia.com/gpu` or MIG-specific names, depending on configuration. Scheduler decisions use Capacity/Allocatable and allocated requests, not observed GPU utilization.

```bash
kubectl describe node APPROVED_GPU_NODE
kubectl -n DISCOVERED_NS get daemonset
kubectl -n DISCOVERED_NS logs DEVICE_PLUGIN_POD --tail=100 --since=15m
```

Inspect sanitized logs for registration failures, NVML errors, missing libraries, incompatible runtime and permission issues.

## 5. GPU Operator reconciliation

GPU Operator uses operator-managed resources and ClusterPolicy-style configuration in supported versions. Its components may include driver, toolkit, device plugin, feature discovery, DCGM exporter, validator and MIG Manager. Discover the actual installed CRD and controller status before reading configuration.

```bash
kubectl api-resources | grep -Ei 'clusterpolicy|nvidia'
kubectl get crd | grep -Ei 'clusterpolic|nvidia'
```

Do not assume any fixed ClusterPolicy name, namespace or chart value. Inspect installed operator version and chart release before planning changes.

## 6. Driver lifecycle

Validate GPU SKU, node OS/kernel, NVIDIA driver version, CUDA compatibility, container runtime, security modules and cloud-managed image ownership. Kernel upgrades can break driver modules; changing drivers can affect every GPU Pod on a node. Require drain/capacity and rollback plans.

## 7. NVIDIA Container Toolkit and CDI

The NVIDIA Container Toolkit configures container runtimes for GPU access. Container Device Interface (CDI) is another supported device injection path on compatible stacks. Determine actual configured mode and avoid simultaneously applying conflicting runtime changes.

## 8. GPU Feature Discovery and labels

GPU Feature Discovery/Node Feature Discovery may label nodes with GPU product, architecture, MIG and other capabilities. Label keys and values are version-dependent. Verify discovered labels before writing affinity or node selectors.

## 9. MIG strategies

On supported NVIDIA hardware, MIG creates GPU instances with distinct resource profiles. Device-plugin `single` and `mixed` strategies affect advertised resource names. Changing MIG configuration can disrupt existing workloads and often requires node maintenance. Confirm exact GPU SKU and operator/device-plugin configuration.

## 10. Time-slicing and sharing

Time-slicing may oversubscribe GPU allocation but does not provide hard VRAM isolation. Do not treat a time-sliced GPU replica as a dedicated full GPU. Apply admission, per-tenant quotas and runtime memory controls; document performance and noisy-neighbor risks.

## 11. DCGM and telemetry

DCGM exporter can provide GPU utilization, memory, power, temperature and error metrics if installed and configured. Inspect actual exporter endpoints and metric labels before creating alerts. Watch ECC/Xid errors and device disappearance; avoid high-cardinality Pod or tenant labels where unnecessary.

## 12. EKS, AKS and GKE considerations

Managed cloud GPU images may already contain NVIDIA drivers/toolkits. Determine provider support and node image ownership first. Document GPU instance families, kernel upgrade policy, daemonset scheduling, GPU quotas, autoscaler limits and node replacement procedures. Do not assume identical behavior across EKS/AKS/GKE.

## 13. Security

Restrict GPU Operator/driver management RBAC, DaemonSet privileges, hostPath access, driver container images and Helm values. Pin approved images, use supply-chain verification, isolate node pools, protect telemetry and enforce tenant quotas. Driver management is a privileged node operation.

## 14. Safe upgrade sequence

```text
Compatibility matrix and maintenance approval
 → inventory GPU nodes/active workloads
 → capacity for failover and draining
 → immutable operator/chart/driver bundle
 → test isolated staging GPU node
 → validate driver, plugin, CDI and DCGM
 → synthetic CUDA and model inference
 → one-node canary and SLO observation
 → progressive rollout or restore known-good image/config
```

Never perform cluster-wide driver changes as a troubleshooting shortcut.

## 15. Troubleshooting decision tree

```text
GPU unavailable?
 ├─ node lacks GPU hardware → cloud instance inventory
 ├─ driver not loaded → kernel/NVML
 ├─ plugin Pod absent → DaemonSet scheduling
 ├─ plugin fails NVML → driver/runtime compatibility
 ├─ GPU allocatable zero → kubelet registration
 ├─ GPU allocatable positive but Pod Pending → quota/taints
 ├─ container CUDA unavailable → toolkit/CDI
 ├─ MIG resource missing → MIG profile/strategy
 ├─ device unhealthy/Xid → DCGM/hardware
 ├─ exporter missing → monitoring components
 └─ upgrade regression → immutable rollback
```

## 16. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | GPU Operator absent by design | inventory |
| F02 | conflicting driver managers | node image/chart |
| F03 | driver/kernel mismatch | dmesg |
| F04 | NVIDIA driver not loaded | nvidia-smi |
| F05 | toolkit runtime misconfigured | container startup |
| F06 | CDI devices unavailable | runtime diagnostics |
| F07 | device-plugin DaemonSet missing | DaemonSet |
| F08 | device-plugin crash loop | Pod logs |
| F09 | NVML initialization failure | plugin logs |
| F10 | kubelet plugin registration fails | node allocatable |
| F11 | node allocatable GPU zero | node status |
| F12 | GPU Pod Pending due to quota | events |
| F13 | GPU node taint mismatch | scheduler |
| F14 | GPU Feature Discovery labels stale | node labels |
| F15 | MIG profile unavailable | GPU/MIG inventory |
| F16 | MIG Manager disrupts workloads | node events |
| F17 | time-slicing noisy neighbor | latency/VRAM |
| F18 | DCGM exporter absent | metrics discovery |
| F19 | GPU Xid/ECC fault | health telemetry |
| F20 | cloud node image upgrade breaks CUDA | version matrix |
| F21 | operator RBAC denied | controller logs |
| F22 | privileged DaemonSet security violation | admission |
| F23 | rollback leaves mixed driver versions | fleet audit |
| F24 | idle GPU nodes increase cost | FinOps |

## 17. Sixteen operational runbooks

1. Inventory GPU Operator and managed-node ownership.
2. Validate driver/kernel/CUDA compatibility.
3. Recover driver module and NVML initialization.
4. Diagnose Container Toolkit and CDI.
5. Restore device-plugin DaemonSet scheduling.
6. Recover device-plugin crash loops and registration.
7. Diagnose zero GPU allocatable resources.
8. Recover GPU quota/taint scheduling failures.
9. Validate GPU Feature Discovery and labels.
10. Recover MIG configuration/profile issues.
11. Contain time-slicing/noisy-neighbor incidents.
12. Restore DCGM exporter and telemetry.
13. Recover Xid/ECC unhealthy devices.
14. Resolve operator RBAC and privileged admission.
15. Execute staged driver/operator upgrade and rollback.
16. Complete fleet audit, RCA and GPU FinOps reconciliation.

## 18. Eight tabletop game days

1. Driver fails after node kernel upgrade.
2. Device plugin stops registering GPUs.
3. Toolkit/CDI configuration breaks CUDA containers.
4. GPU Operator conflicts with managed cloud image.
5. MIG profile change makes workloads Pending.
6. Xid/ECC errors render a GPU unhealthy.
7. DCGM exporter fails during an inference incident.
8. Rollback leaves half the GPU nodes on old driver.

Tabletop by default; live fault injection requires approved isolated staging.

## 19. Twenty acceptance gates

G01 scope/versions/ownership; G02 GPU node inventory; G03 driver/kernel compatibility; G04 toolkit/CDI; G05 device-plugin DaemonSet; G06 kubelet registration; G07 allocatable GPU; G08 quota/taints; G09 feature labels; G10 MIG/sharing; G11 CUDA smoke test; G12 model inference; G13 GPU health; G14 DCGM telemetry; G15 cloud image compatibility; G16 security/RBAC; G17 upgrade/canary; G18 rollback; G19 FinOps; G20 independent review.

See [companion lab](../labs/047-nvidia-device-plugin-gpu-operator-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 048 — GPU Node Pools, Taints, Tolerations and Affinity.**
