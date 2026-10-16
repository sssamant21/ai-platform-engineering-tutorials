# Chapter 15.2 — Kubernetes and GPU Infrastructure Foundation

**Part 15 — End-to-End Production Project | Capstone 2 of 7**

## Objective

Build and validate the Kubernetes and NVIDIA GPU foundation required for TrueFoundry-managed vLLM inference. Translate Chapter 15.1 architecture decisions into repeatable infrastructure, secure cluster access, GPU node pools, scheduling controls, device-plugin/DCGM telemetry, quota, N-1 capacity and rollback. This chapter is **implementation guidance**; cloud provisioning and live acceptance have not been performed.

### Safety classifications

- **[SAFE-READ]** authorized discovery, `kubectl get/describe`, metrics, quota checks.
- **[LAB-MUTATE]** approved isolated staging cluster with GPU-hour budget and cleanup.
- **[CHANGE-CONTROL]** production IAM, GPU node pools, autoscalers, networking, admission policy, cluster upgrades, GPU driver changes or destructive operations.
- Never paste kubeconfigs, cloud credentials, model secrets, tenant prompts or private endpoints into Git or incident tickets.

## 1. Target architecture

```text
Approved IaC + policy checks + peer-reviewed PR
        │
        ▼
Cloud VPC/VNet (private subnets, DNS, egress, endpoints)
        │
        ▼
Managed Kubernetes (EKS / AKS / GKE)
  ├─ identity / RBAC / namespaces / admission / policies
  ├─ system node pool (CPU, separate from GPU)
  ├─ GPU node pool(s) with zone/topology strategy
  │    ├─ NVIDIA driver / container toolkit
  │    ├─ NVIDIA device plugin or supported GPU Operator
  │    ├─ GPU allocatable resource and scheduling
  │    └─ DCGM exporter / hardware telemetry
  ├─ metrics / logs / events / alerting
  └─ reserved capacity for vLLM / TrueFoundry integration
        │
        ▼
Chapter 15.3 — TrueFoundry agent and model deployment
```

Separate cloud foundation from TrueFoundry platform reconciliation. Do not install multiple competing device plugins, driver managers or GPU Operators.

## 2. Provider selection and prerequisites

| Domain | EKS | AKS | GKE |
|---|---|---|---|
| Cluster identity | IAM / IRSA or EKS Pod Identity as supported | Microsoft Entra / Workload Identity | Workload Identity Federation for GKE |
| GPU pools | GPU EC2 node groups or supported autoscaling | GPU VM-size node pools | GPU node pools or supported Autopilot capabilities |
| Network | VPC CNI, subnets, endpoints | Azure CNI variant, VNets | VPC-native networking |
| Autoscaling | Cluster Autoscaler/Karpenter per design | AKS cluster autoscaler | GKE cluster autoscaler |
| GPU software | AMI/driver/plugin compatibility | AKS GPU driver mode and plugin compatibility | GKE driver installation and GPU plugin model |

Confirm region/AZ GPU availability, VM/GPU SKUs, driver/CUDA compatibility, account quotas, private egress and support matrices from the deployed provider version. **Provider examples are architectural, not guaranteed defaults.** Use a single approved cluster provisioning approach and pin module/provider versions.

## 3. Infrastructure as Code repository layout

```text
infrastructure/
  environments/
    staging/
    production/
  modules/
    network/
    kubernetes/
    gpu-nodepool/
    identity/
    observability/
  policies/
  tests/
  README.md
kubernetes/
  base/
    namespaces/
    quota/
    limitrange/
    scheduling/
    monitoring/
  overlays/
    staging/
    production/
docs/
  adr/
  capacity/
  security/
  runbooks/
```

IaC plan output is reviewed before apply; protect state with encryption, access control, locking and recovery. Never use `terraform apply -auto-approve` in production. Pin images and modules and verify provenance. Cloud resources can incur charges even when idle.

## 4. Preflight checklist

1. Approved Chapter 15.1 ADRs, service ownership and risk classification.
2. Cloud subscription/account, region, zones and approved budget.
3. GPU SKU/VRAM, quota, capacity reservations and provisioning lead time.
4. Private networking, DNS, egress to required endpoints and IP exhaustion headroom.
5. Kubernetes supported version, node OS, GPU driver and CUDA/runtime matrix.
6. IAM/RBAC and workload identity boundaries; no static long-lived keys.
7. IaC plan, policy validation, rollback, cluster/node backup strategy.
8. GPU pool taints/labels, topology and scheduler constraints.
9. GPU plugin and DCGM deployment ownership; no overlapping controllers.
10. Synthetic GPU smoke workload and teardown approval.

## 5. Read-only cluster inventory

```bash
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl get nodes -L topology.kubernetes.io/zone,node.kubernetes.io/instance-type
kubectl get namespaces
kubectl get resourcequota,limitrange -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
```

Inspect node allocatable GPU values without assuming that the driver or plugin is installed:

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" allocatable="}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'
```

If the custom resource path is missing, validate the actual advertised GPU resource name and device-plugin installation. MIG exposes different resource names.

## 6. GPU node pool and scheduling design

Use dedicated GPU pools with labels, taints and topology constraints. **Labels are not security boundaries**: protect node-pool access through admission policy and namespace authorization. Match tolerations to actual taints and ensure resource requests align with device plugin advertisements.

Example **illustrative staging-only** Pod specification (review image, runtime and registry before applying):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-smoke
  namespace: gpu-lab
spec:
  restartPolicy: Never
  nodeSelector:
    workload-tier: gpu-staging
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu-staging
      effect: NoSchedule
  containers:
    - name: gpu-check
      image: APPROVED_GPU_DIAGNOSTIC_IMAGE_BY_DIGEST
      command: ["/bin/sh", "-c"]
      args: ["nvidia-smi && sleep 10"]
      resources:
        limits:
          nvidia.com/gpu: "1"
```

**Do not apply the placeholder image.** Replace with an approved immutable GPU-capable diagnostic image and verify tolerations/labels. Kubernetes GPU extended resources are commonly declared as limits; confirm admission behavior and device-plugin policy. Clean up promptly to avoid GPU charges.

## 7. Device plugin, GPU Operator and driver verification

Determine whether cloud provider or node image installs NVIDIA driver and plugin. If using NVIDIA GPU Operator, explicitly configure driver and toolkit management to avoid collisions with managed node images. Check plugin DaemonSet status, node allocatable GPU, device visibility and DCGM exporter health. A node showing GPU allocatable is not sufficient to prove CUDA runtime or vLLM model compatibility.

Read-only commands:

```bash
kubectl get daemonsets -A
kubectl get pods -A -o wide
kubectl describe node GPU_NODE_NAME
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

On an approved GPU host or diagnostic Pod, run `nvidia-smi` and record GPU model, memory, driver, XID/ECC, thermal state and device count.

## 8. Capacity and high availability

```text
steady_GPU = replicas × GPU_per_replica
rollout_peak_GPU = (old_replicas + surge_replicas) × GPU_per_replica
spare_GPU = schedulable_GPU - committed_GPU
```

These are first-pass arithmetic checks, not proof of schedulability. Validate node packing, per-zone allocation, topology, Pod anti-affinity, PDB, instance availability, IP addresses and GPU quota. Document whether N-1 means one GPU, one node or one AZ failure and the allowed degradation under that event. Account for model download/warmup when planning recovery time.

## 9. Quotas and guardrails

Define namespace resource quotas for CPU, memory and GPU extended resources where supported, limit ranges, admission policy, GPU pool isolation and cluster autoscaler limits. Apply cost allocation labels and cloud budget alerts. Avoid setting resource quotas that silently block system/agent Pods. Stage quotas with synthetic workloads before production.

## 10. Observability and SLO preflight

Require Kubernetes Events, node readiness, scheduling latency, device plugin availability, DCGM metrics and GPU XID/ECC alerting. Verify scrape freshness, namespace/cluster labels, dashboard ownership and paging routes. Do not invent vLLM or DCGM metric names: discover version-specific exported series before writing alerts.

## 11. Failure triage decision tree

```text
GPU workload unavailable?
 ├─ Node missing → cloud quota / autoscaler / AZ / IAM / subnet IPs
 ├─ Node NotReady → kubelet / CNI / disk / driver / node pressure
 ├─ GPU allocatable absent → driver / plugin / Operator collision
 ├─ Pod Pending → requests / taints / affinity / quota / topology
 ├─ Pod running, CUDA unavailable → toolkit / device visibility / image
 ├─ CUDA OOM → VRAM/model/context/KV cache fit
 └─ Works, slow → GPU throttling, queue, batching, network
```

Check actual Events and logs; avoid blindly restarting nodes or modifying driver installations.

## 12. Sixteen operational runbooks

1. Cloud account/region and GPU quota preflight.
2. IaC state, plan and policy review.
3. Private network and egress readiness.
4. Cluster version and access controls.
5. System/GPU node pool separation.
6. GPU SKU/driver/CUDA matrix.
7. Device plugin/Operator ownership.
8. Node allocatable GPU inventory.
9. Taints, tolerations and affinity.
10. Namespace GPU quota/admission.
11. Autoscaler and AZ capacity.
12. DCGM and hardware telemetry.
13. GPU diagnostic smoke workload.
14. N-1 and rolling surge capacity.
15. Cost/cleanup and rollback.
16. Infrastructure readiness handoff to TrueFoundry.

## 13. Forty-eight failure scenarios

| Domain | Scenarios |
|---|---|
| Cloud | 01 quota denied; 02 SKU unavailable; 03 AZ shortage; 04 subnet IP exhaustion; 05 private endpoint blocked; 06 IAM denied |
| Cluster | 07 unsupported version; 08 RBAC denied; 09 CNI failure; 10 CoreDNS failure; 11 node NotReady; 12 certificate expiry |
| GPU | 13 driver missing; 14 CUDA mismatch; 15 plugin absent; 16 plugin collision; 17 GPU allocatable zero; 18 MIG resource mismatch |
| Scheduling | 19 Pod Pending; 20 taint mismatch; 21 affinity conflict; 22 topology spread unsatisfiable; 23 namespace quota; 24 PDB blocks drain |
| Runtime | 25 image pull denied; 26 device invisible; 27 CUDA init fails; 28 GPU OOM; 29 host OOM; 30 XID/ECC |
| Autoscaling | 31 scale-out delayed; 32 autoscaler max; 33 GPU node never registers; 34 wrong node template; 35 spot interruption; 36 rollout surge exceeds quota |
| Observability | 37 DCGM absent; 38 stale metrics; 39 alerts muted; 40 node log gap; 41 GPU utilization misleading; 42 untagged cost |
| Operations | 43 IaC drift; 44 state lock conflict; 45 unreviewed change; 46 no N-1; 47 budget overrun; 48 orphaned GPU nodes |

## 14. Six staging game days

1. GPU quota shortage and Pending Pod.
2. Driver/device-plugin mismatch (tabletop if unsafe).
3. GPU node NotReady and failover capacity.
4. Autoscaler max/zone shortage.
5. GPU XID/ECC alert and drain decision (tabletop).
6. IaC drift, rollback and cost cleanup.

Record expected/actual, UTC evidence, affected SLO, owner, mitigation, rollback, GPU-hours and review.

## 15. Fourteen acceptance gates

G1 approved IaC/ADRs; G2 private networking; G3 cluster/RBAC; G4 GPU SKU/driver matrix; G5 GPU allocatable; G6 plugin/DCGM; G7 scheduling/taints; G8 namespace quotas; G9 autoscaling/AZ; G10 N-1/surge; G11 synthetic GPU smoke; G12 security/audit; G13 budget/cleanup; G14 reviewer handoff.

See [Chapter 15.2 companion lab](../labs/02-kubernetes-gpu-infrastructure-foundation-check.md). All gates remain pending until executed and reviewed.

**Next:** Chapter 15.3 — TrueFoundry Integration and Secure Model Serving.
