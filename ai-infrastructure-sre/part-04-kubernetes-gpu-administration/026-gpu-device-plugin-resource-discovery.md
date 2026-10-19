# Chapter 026 — GPU Device Plugin and Resource Discovery

**AI Infrastructure Engineering & SRE · Part 04 — Kubernetes GPU Administration · Chapter 026 of 100**

**Status:** Published documentation. Offline tests, live Kubernetes/GPU staging validation, independent security review and production acceptance remain separate pending milestones.

## Learning objectives

By completing this chapter, an SRE/DBRE should be able to explain Kubernetes extended GPU resources, the NVIDIA device-plugin lifecycle, kubelet device registration and allocation, CDI/container-runtime integration, MIG discovery, and operational failure domains. Learners will build a reproducible, fail-closed GPU resource-discovery admission gate, investigate production incidents, and demonstrate a safe staging recovery procedure.

## 1. Architecture and control flow

```text
              Kubernetes API / scheduler
                node allocatable GPUs
                          |
                    kubelet node
                          |
           device-plugin registration socket
                          |
             NVIDIA device-plugin DaemonSet
               | discovery / health
               | Allocate() / device metadata
               v
            NVIDIA NVML / driver
                          |
             GPU / MIG device inventory
                          |
       OCI runtime / CDI / NVIDIA Container Toolkit
                          |
              container visible GPU(s)
                          |
                 CUDA application
```

**Key distinction:** a node advertising `nvidia.com/gpu: 4` proves only that Kubernetes currently exposes an allocatable resource count. It does **not** prove CUDA initialization, device health, GPU UUID mapping, MIG profile suitability, topology, or workload SLOs. GPU resources are vendor-defined extended resources. Kubernetes schedules integer quantities; the NVIDIA plugin supplies device-specific allocation details.

### Control-plane contracts

1. Device plugin discovers supported GPU/MIG devices, registers with kubelet, and reports health through the device-plugin API.
2. Kubelet updates node capacity/allocatable and uses the plugin's Allocate response to configure eligible containers.
3. The scheduler considers the pod's GPU resource requests against node allocatable and existing allocations.
4. NVIDIA Container Toolkit and the configured runtime/CDI strategy make the allocated device visible inside the container.
5. CUDA or another supported workload library initializes the device.
6. Monitoring independently verifies application correctness and progress.

An `Allocatable` resource count is **not** a real-time GPU utilization measurement, nor a guarantee of placement on a specific GPU UUID. Resource allocation, health and workload readiness must be observed independently.

## 2. Extended resource semantics

A standard GPU request uses the vendor resource name:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-discovery-canary
  namespace: gpu-staging
spec:
  restartPolicy: Never
  nodeSelector:
    accelerator-pool: approved-staging
  containers:
    - name: gpu-check
      image: nvcr.io/nvidia/cuda:12.4.1-base-ubuntu22.04
      command: ["sh", "-c"]
      args: ["nvidia-smi -L && nvidia-smi --query-gpu=uuid,name --format=csv"]
      resources:
        limits:
          nvidia.com/gpu: 1
```

**Example only:** pin a security-reviewed immutable image digest and verify its driver/CUDA compatibility before deployment. Do not assume the sample tag is approved or available. For ordinary GPU extended resources, a limit may imply an equivalent request; explicit matching request and limit is also valid. Requests greater than limits are invalid. Fractional GPU quantities are not natively supported for `nvidia.com/gpu`; supported sharing/MIG strategies have their own resource model.

The canary uses a staging namespace and requires an existing trusted node label. It does not create that namespace or label. In a real environment, enforce least-privilege service accounts, Pod Security admission, restricted filesystem/mounts and egress as appropriate.

## 3. GPU discovery and health inventory

Read-only discovery:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -o json | jq -r '.items[] | [.metadata.name, (.status.capacity["nvidia.com/gpu"] // "none"), (.status.allocatable["nvidia.com/gpu"] // "none")] | @tsv'
kubectl describe node APPROVED_GPU_NODE
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

On the approved host:

```bash
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,pci.bus_id,driver_version --format=csv
nvidia-smi -q -d ECC
nvidia-smi topo -m
journalctl -u kubelet -b --no-pager | tail -n 100
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|ECC|AER' | tail -n 100
```

For a known deployment, inspect the plugin DaemonSet and logs using its **actual namespace and resource name**; do not assume a universal `kube-system` installation:

```bash
kubectl get daemonsets -A | grep -Ei 'nvidia|device-plugin'
kubectl get pods -A | grep -Ei 'nvidia|gpu-operator'
kubectl logs -n APPROVED_NAMESPACE APPROVED_PLUGIN_POD --since=30m
```

Check plugin readiness, registration messages, NVML initialization, node taints, resource updates, GPU health, node-level allocatable count, container runtime configuration and workload errors. Device plugin pods are privileged infrastructure components; review their security context and permissions rather than copying them into application pods.

## 4. MIG and resource naming

MIG-capable devices can be partitioned into GPU Instances/Compute Instances. NVIDIA device-plugin **MIG strategy** determines resource exposure; depending on configuration, resources may be exposed under `nvidia.com/gpu` or profile-specific resource names such as `nvidia.com/mig-1g.10gb` on supported hardware. Exact profile names and memory capacities vary by GPU generation and profile.

**Do not** assume a physical GPU equals one schedulable `nvidia.com/gpu` unit under all strategies. Mixing full GPU and MIG resources requires platform-specific policies, and MIG changes can disrupt workloads. Record physical GPU UUID, MIG UUID/profile, node SKU, strategy, advertised resource key and tenant constraints.

## 5. Device-plugin versus GPU Operator ownership

The **NVIDIA device plugin** integrates GPU resources with kubelet. The **NVIDIA GPU Operator** can orchestrate components such as drivers, container toolkit, device plugin, GPU feature discovery, DCGM exporter and MIG Manager, depending on configuration. Do not install a second independently managed plugin over an existing GPU Operator deployment.

Ownership checklist: identify Helm/Operator release, namespace, chart/operator version, driver management mode, node OS, toolkit/CDI strategy, MIG strategy, configuration ConfigMaps, upgrade policy and change owner.

## 6. Runtime and CDI boundaries

The device-plugin Allocate response may rely on environment variables, mounts, device specs or CDI device names, depending on configured strategies and versions. Kubernetes support for CDI and NVIDIA toolkit integration is version-sensitive. Verify the **actual kubelet, containerd/CRI-O, NVIDIA toolkit and device-plugin compatibility matrix**.

A pod may be scheduled correctly but fail with `no NVIDIA devices found`, `libcuda.so` errors or `CUDA driver version is insufficient`. Investigate plugin registration, OCI runtime/CDI, driver mount and CUDA compatibility separately.

## 7. Scheduling, health and lifecycle

GPU device plugins generally run as DaemonSets on eligible GPU nodes. Nodes may be tainted for GPU workloads; workloads need appropriate tolerations, node affinity and integer resource requests. A DaemonSet pod being Running does not establish correct resource registration.

Typical lifecycle:

- GPU node boots with validated driver and runtime.
- Device plugin initializes NVML, discovers GPUs and registers with kubelet.
- Node capacity/allocatable reflects the plugin's advertised resources.
- Scheduler places an eligible GPU pod.
- Kubelet allocates devices and the runtime exposes them.
- Workload verifies CUDA/device identity and application progress.
- On device health degradation, plugin/kubelet updates may reduce available devices; existing workload behavior requires separate investigation.
- During maintenance, workloads are drained via approved policy and the node is requalified.

## 8. Production acceptance metrics

| Signal | Source | Meaning |
|---|---|---|
| expected physical GPU count | approved inventory | hardware baseline |
| plugin-advertised resource keys/count | Kubernetes node status | scheduler-visible supply |
| device plugin ready/registered | DaemonSet logs/kubelet | resource control-plane health |
| CUDA device count/UUID | staging canary | container visibility |
| driver/CUDA/toolkit compatibility | version inventory | runtime support |
| Xid/ECC and GPU health | driver/DCGM | device fault evidence |
| MIG profile mapping | NVIDIA inventory + plugin config | resource correctness |
| GPU pod Pending reasons | scheduler events | admission/placement evidence |
| canary completion | Kubernetes job/pod | functional smoke |
| telemetry age and node inventory drift | monitoring | evidence freshness |

**Hard rule:** stale metrics, missing GPU UUIDs or unknown resource strategy must not be silently treated as healthy.

## 9. Twelve troubleshooting scenarios

| Symptom | Evidence | Safe response |
|---|---|---|
| node shows no `nvidia.com/gpu` | plugin logs, kubelet, NVML | verify registration and host driver |
| plugin CrashLoopBackOff | image, NVML init, permissions | revert incompatible rollout |
| GPU pod Pending | request, node allocatable, taints | check scheduling constraints |
| GPU visible on host, not pod | Allocate/CDI/runtime | inspect runtime wiring |
| `libcuda.so` missing | toolkit and mounts | restore approved runtime config |
| CUDA init failure | driver/runtime versions | compare compatibility matrix |
| GPU count lower after Xid | health reports, kernel logs | quarantine failed device/node |
| wrong MIG resource key | plugin MIG strategy, profiles | correct approved configuration |
| unexpected oversubscription | time-slicing/MPS policy | enforce tenant admission |
| duplicate device-plugin deployment | DaemonSets, ownership | reconcile operator control |
| GPU pod stuck Terminating | finalizers/runtime state | follow controlled workload recovery |
| plugin telemetry absent | DCGM/exporter and API | fail closed on canary promotion |

## 10. Ten incident game days

1. Driver update breaks NVML registration.
2. Plugin DaemonSet rollout advertises zero GPUs.
3. GPU pod is Pending due to missing taint toleration.
4. CDI/runtime mismatch prevents device injection.
5. GPU Xid causes an unexpected allocatable decrease.
6. MIG profile changes invalidate workload requests.
7. GPU Operator and manually installed plugin conflict.
8. Node image rollback restores driver but not toolkit.
9. Resource labels drift after node replacement.
10. GPU telemetry becomes stale while scheduling appears healthy.

For each drill, record detection, GPU UUID, node, resource key, blast radius, workload correctness, rollback and incident owner.

## 11. Eight operational runbooks

**RB-026-A — Inventory:** capture node SKU, physical/MIG UUIDs, driver, plugin/operator versions and expected resources.

**RB-026-B — Plugin registration:** inspect DaemonSet, kubelet registration, NVML and node capacity/allocatable.

**RB-026-C — GPU pod Pending:** check extended resource requests, quotas, taints, affinity, node readiness and scheduler events.

**RB-026-D — Runtime/CDI failure:** reconcile Allocate strategy, toolkit, container runtime, driver mounts and security policy.

**RB-026-E — MIG discovery:** validate MIG mode, profiles, resource names, tenant isolation and owner approval.

**RB-026-F — Health degradation:** correlate Xid/ECC, plugin health, allocatable count, pod failures and replacement policy.

**RB-026-G — Safe upgrade/rollback:** freeze rollout, preserve checkpoint, validate N-1 capacity, drain and canary.

**RB-026-H — RCA and acceptance:** verify end-to-end device visibility, workload correctness, SLO, monitoring and signoff.

## 12. Executable companion lab

[Lab 026 — GPU Device Plugin Resource Discovery Gate](../labs/026-gpu-device-plugin-resource-discovery-check.md) includes an offline Python 3.11+ inventory validator, pytest suite, synthetic incidents and a staged acceptance checklist. It does **not** contact Kubernetes or modify device-plugin settings.

## 13. 100-point assessment

| Domain | Points |
|---|---:|
| Device plugin / kubelet architecture | 20 |
| Extended resources and MIG discovery | 20 |
| Runtime, scheduling and security | 20 |
| Troubleshooting and runbooks | 20 |
| Executable tests and staging acceptance | 20 |
| **Total** | **100** |

Pass >=90 and no unresolved correctness, security, resource-inventory, health or rollback blocker.

## References

- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [Kubernetes Extended Resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#extended-resources)
- [NVIDIA Kubernetes Device Plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/)
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)

**Next:** Chapter 027 — GPU Operator Architecture and Lifecycle.
