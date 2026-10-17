# Chapter 046 — Kubernetes GPU Scheduling Fundamentals

**Part 07 — Kubernetes GPU Scheduling and Autoscaling | Chapter 046/100**

> **Status:** Written tutorial published. Live lab execution, independent technical/security review and operational acceptance are pending. GPU support depends on Kubernetes version, installed device plugins, drivers, runtime and cloud node configuration.

## Learning outcomes

Explain how Kubernetes schedules NVIDIA GPU Pods; inspect extended resources, node labels, taints, affinity and topology; distinguish device allocation from VRAM capacity; plan MIG and multi-GPU workloads; diagnose Pending Pods, driver and plugin issues; define production runbooks, SLOs, FinOps and safe rollback.

## 1. Architecture

```text
LLM request → KServe / Ray Serve / vLLM / Triton
                         │
                  Pod GPU request
                         │
            Kubernetes scheduler
             ├─ node allocatable GPU
             ├─ taints / tolerations
             ├─ affinity / topology
             └─ quotas / priority
                         │
                kubelet / device plugin
                         │
              container runtime / CDI
                         │
                NVIDIA driver / CUDA
                         │
           physical GPU / MIG instances
                         │
        DCGM metrics, health, VRAM and cost
```

The scheduler uses advertised extended-resource quantities; it does not directly solve model memory placement, tensor-parallel communication or runtime CUDA compatibility.

## 2. GPU extended resources

NVIDIA device plugins typically advertise `nvidia.com/gpu` as an extended resource. GPU requests are integer device counts in common configurations. GPU limits/requests have special Kubernetes rules; the resource is not equivalent to CPU millicores. Some MIG strategies advertise distinct resource names. Always inspect installed device plugin configuration.

**[SAFE-READ]**

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl describe nodes | grep -E 'Name:|nvidia.com/|Allocatable:|Capacity:' 
kubectl get pods -A -o wide
```

Use `kubectl describe node NODE_NAME` for a reliable per-node Capacity/Allocatable view; text filtering may hide context.

## 3. Pod GPU request example

This is an **illustrative, not production-ready** synthetic CUDA test Pod. It requires an approved namespace, GPU runtime and image digest; do not apply it without staging authorization.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-smoke
  namespace: approved-staging
spec:
  restartPolicy: Never
  containers:
    - name: cuda
      image: registry.example.invalid/approved-cuda@sha256:REPLACE_WITH_VALID_DIGEST
      command: ["sh", "-c", "nvidia-smi && sleep 5"]
      resources:
        limits:
          nvidia.com/gpu: 1
```

The image and digest are placeholders and will not run as-is. GPU allocation does not guarantee enough VRAM for an LLM.

## 4. Node labeling and selection

Node selectors are hard constraints. Node affinity supports required and preferred placement. Use verified GPU SKU labels, node pools and cloud labels rather than assuming a universal `gpu=true` label. Never relabel production nodes without approval.

```bash
kubectl get nodes --show-labels
kubectl describe node APPROVED_GPU_NODE
```

## 5. Taints and tolerations

Dedicated GPU pools commonly use taints to repel non-GPU workloads. A toleration permits but does not force scheduling onto a node. Pair tolerations with selectors/affinity and quotas to avoid unintended GPU placement.

## 6. ResourceQuota and LimitRange

Enforce per-namespace GPU resource ceilings and fair allocation. Check namespace quotas and Pod admission events when scheduling fails. Quota exhaustion may reject Pod creation before a Pending Pod ever appears.

```bash
kubectl -n APPROVED_NS get resourcequota,limitrange
kubectl -n APPROVED_NS get events --sort-by=.lastTimestamp
```

## 7. Device plugin lifecycle

Device plugin registration advertises resources to kubelet. Drivers, container runtime, CDI/NVIDIA Container Toolkit and plugin must be compatible. A healthy GPU host does not imply the device plugin advertises GPUs to Kubernetes; conversely allocatable GPU resources do not prove CUDA works inside the container.

## 8. GPU memory versus scheduling

```text
VRAM budget = model weights + KV cache + activations
            + CUDA graphs + runtime workspace
            + NCCL buffers + headroom
```

Kubernetes usually schedules devices, not per-Pod VRAM headroom. Validate model size, context length, concurrent requests, batch behavior and GPU telemetry. Shared GPU strategies require explicit isolation and memory safety policy.

## 9. MIG and GPU sharing

NVIDIA Multi-Instance GPU partitions supported hardware into isolated GPU instances; the resource names and profile availability depend on device plugin strategy and GPU SKU. Time-slicing provides a different sharing model and does not automatically give memory isolation. Never treat MIG and time-slicing as interchangeable.

## 10. Multi-GPU topology

Tensor parallelism, pipeline parallelism and NCCL collective communication depend on GPU placement, PCIe/NVLink, node boundaries and network topology. Four allocatable GPUs spread across nodes are not equivalent to four local GPUs in one Pod. Confirm model-server and scheduling requirements before selecting topology.

## 11. Priority and preemption

PriorityClasses and preemption may displace lower-priority Pods but do not create physical GPU capacity. Protect production SLOs with admission policies and avoid aggressive preemption of long-lived inference streams without drain planning.

## 12. Cluster autoscaling boundary

Kubernetes scheduler places Pods on existing nodes; Cluster Autoscaler, Karpenter or cloud node provisioning can add nodes when configured and capacity exists. Cloud GPU quotas, instance availability, image startup, driver readiness and model warmup can dominate scale-out time.

## 13. GPU health checks

On an authorized GPU node or approved diagnostics Pod:

```bash
nvidia-smi
nvidia-smi -L
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu,temperature.gpu --format=csv
```

Host access may not be available. Prefer approved monitoring/diagnostic workflows; do not launch privileged Pods as a shortcut.

## 14. Security controls

Require least-privilege RBAC, restricted Pod Security, signed images, approved RuntimeClasses, workload identity, namespace quotas, NetworkPolicies and controlled GPU node access. GPU diagnostics must not leak prompts, model secrets or other tenants' workloads.

## 15. Observability and SLO

Measure Pod Pending time, FailedScheduling reasons, GPU allocatable versus allocated, per-device VRAM, ECC/Xid, temperature, utilization, startup and model readiness, TTFT/TPOT/E2E, terminal-valid completion and GPU-hours per valid completion. Use DCGM exporter or equivalent only if installed; validate metric names.

## 16. GPU Pending decision tree

```text
GPU Pod Pending?
 ├─ GPU quota rejected admission → ResourceQuota
 ├─ insufficient nvidia.com/gpu → allocatable/allocated
 ├─ taint mismatch → tolerations
 ├─ node selector mismatch → labels
 ├─ affinity/topology conflict → placement
 ├─ device plugin absent → kubelet/plugin
 ├─ cloud GPU nodes absent → provisioner/quota
 ├─ image pull → registry and digest
 ├─ PVC unavailable → storage scheduling
 └─ runtime crash after scheduling → driver/CUDA/VRAM
```

Inspect `kubectl describe pod` events first; do not guess from the Pending status alone.

## 17. Twenty-four failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | no GPU nodes registered | node inventory |
| F02 | device plugin missing | allocatable |
| F03 | driver incompatible | CUDA logs |
| F04 | runtime toolkit/CDI broken | container startup |
| F05 | GPU resource quota exceeded | admission events |
| F06 | insufficient GPU devices | scheduler |
| F07 | taint not tolerated | FailedScheduling |
| F08 | node selector mismatch | labels |
| F09 | required affinity impossible | scheduler |
| F10 | topology spread unsatisfiable | events |
| F11 | MIG profile unavailable | resource inventory |
| F12 | time-slicing isolation assumption | plugin config |
| F13 | multi-GPU Pod cannot fit | node allocatable |
| F14 | GPU hardware unhealthy | DCGM/Xid |
| F15 | ECC error | GPU health |
| F16 | CUDA OOM despite scheduling | VRAM |
| F17 | KV-cache OOM under long context | runtime |
| F18 | GPU Pod preempted | events |
| F19 | cloud GPU quota exhausted | provisioner |
| F20 | image pull blocked | registry events |
| F21 | model storage inaccessible | startup logs |
| F22 | GPU telemetry stale | scrape freshness |
| F23 | rollback leaves wrong node affinity | manifest diff |
| F24 | idle GPU node cost spikes | FinOps |

## 18. Sixteen operational runbooks

1. Inventory GPU nodes and allocatable resources.
2. Recover device plugin registration.
3. Diagnose NVIDIA driver and runtime compatibility.
4. Restore GPU namespace quota and admission.
5. Recover insufficient-GPU scheduling.
6. Diagnose taints and tolerations.
7. Recover node labels and affinity.
8. Diagnose topology spread and multi-GPU fit.
9. Validate MIG resource profiles.
10. Audit time-slicing and sharing isolation.
11. Recover unhealthy GPU/ECC/Xid.
12. Diagnose CUDA and KV memory exhaustion.
13. Recover cloud GPU node provisioning.
14. Diagnose image/model startup dependencies.
15. Restore telemetry, SLO and GPU FinOps.
16. Execute full GPU scheduling rollback and RCA.

## 19. Eight tabletop game days

1. Device plugin stops advertising GPUs.
2. GPU node pool gets an unexpected taint.
3. Namespace quota blocks an inference rollout.
4. MIG resource profile changes.
5. Four-GPU model cannot find same-node capacity.
6. GPU driver upgrade breaks CUDA startup.
7. GPU node disappears during streaming.
8. GPU autoscaler fails and idle GPU-hours increase.

Tabletop by default; live injection requires approved isolated staging.

## 20. Twenty acceptance gates

G01 approved scope/versions; G02 node inventory; G03 device plugin; G04 driver/runtime; G05 namespace GPU quota; G06 GPU scheduling; G07 taints/tolerations; G08 node affinity; G09 topology; G10 MIG/sharing; G11 multi-GPU placement; G12 VRAM/KV headroom; G13 GPU health; G14 node autoscaler/quota; G15 image/model readiness; G16 security/tenant isolation; G17 telemetry/SLO; G18 rollback; G19 GPU FinOps; G20 independent review.

See [companion lab](../labs/046-kubernetes-gpu-scheduling-fundamentals-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 047 — NVIDIA Device Plugin and GPU Operator.**
