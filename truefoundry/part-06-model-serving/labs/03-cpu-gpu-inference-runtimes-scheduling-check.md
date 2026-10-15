# Lab 6.3 — CPU and GPU Inference Runtimes and Scheduling Check

**Canonical:** [Chapter 6.3](../03-cpu-gpu-inference-runtimes-scheduling/03-cpu-gpu-inference-runtimes-scheduling.md)  
**Default:** **[SAFE-READ]**. Optional CPU/GPU deployments or resource changes are **[LAB-MUTATE]** only in an approved nonproduction TrueFoundry workspace.

## 1. Objectives

Trace the GPU dependency chain, compare CPU and GPU runtime behavior, verify framework device visibility, distinguish host RAM from VRAM pressure, classify 15 failures and document safe rollback.

## 2. Preflight and inventory [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl get nodes -o wide
```

Record TrueFoundry workspace, deployment revision, image digest, model version, node type, GPU model and authorized scope. Do not run new load tests on production.

## 3. Scheduler and GPU allocation [SAFE-READ]

```bash
kubectl describe node <gpu-node>
kubectl -n <namespace> describe pod <gpu-pod>
kubectl -n <namespace> get pod <gpu-pod> -o jsonpath='{.spec.containers[*].resources}{"\n"}'
```

Record node allocatable GPUs, Pod GPU request/limit, CPU/memory fit, node selectors, affinity, taints/tolerations and scheduling events.

| Field | Observed | Interpretation |
|---|---|---|
| Node GPU SKU | | |
| Allocatable GPUs | | |
| Requested GPU resources | | |
| CPU/memory request | | |
| Node selector/affinity | | |
| Tolerations | | |
| Pod phase and scheduling event | | |

## 4. Runtime inspection [SAFE-READ]

Only in an existing authorized container that already includes the tools:

```bash
nvidia-smi
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available(), torch.cuda.device_count())"
python -c "import onnxruntime as ort; print(ort.__version__, ort.get_available_providers())"
```

If a library or `nvidia-smi` is absent, mark **Not installed**; do not install packages in a production container. Check actual ONNX Runtime session providers where the application exposes that evidence.

## 5. Host RAM and VRAM analysis

```bash
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get pod <pod> -o jsonpath='{range .status.containerStatuses[*]}{.name}{" last="}{.lastState.terminated.reason}{" exit="}{.lastState.terminated.exitCode}{"\n"}{end}'
kubectl -n <namespace> top pod <pod> --containers
```

Metrics Server is optional; use approved monitoring for trends and DCGM GPU metrics where deployed. Record host memory peak, VRAM used/free, batch size, concurrency, model weight estimate, queue depth and latency. Exit code 137 alone does not prove OOMKilled.

## 6. Optional controlled inference comparison [LAB-MUTATE]

1. Pin the same vetted model version and golden input dataset.
2. Deploy an approved CPU runtime in a disposable workspace; record P50/P95/P99 latency, throughput, CPU and RAM.
3. If GPU access is authorized, deploy a compatible GPU runtime and validate device allocation and golden output.
4. Measure the same workload under comparable concurrency and warmup conditions.
5. Compare throughput, tail latency, utilization, energy/cost assumptions and correctness.
6. Restore the known-good TrueFoundry revision and clean up only lab-owned resources.

Do not infer that one small benchmark generalizes to production traffic.

## 7. Failure tabletop

| ID | Symptom | First failed transition |
|---|---|---|
| R1 | GPU Pod Pending | Scheduler → node |
| R2 | GPU resource absent | Device plugin → node allocatable |
| R3 | GPU not visible inside container | Kubelet → runtime |
| R4 | PyTorch CUDA unavailable | Framework → CUDA |
| R5 | Device-type inference fails | Runtime → device selection |
| R6 | Unsupported GPU architecture | Kernels → GPU |
| R7 | VRAM OOM on model load | Weights → VRAM |
| R8 | VRAM OOM under concurrency | Batching/KV cache → VRAM |
| R9 | Container OOMKilled | Process → host memory |
| R10 | Low GPU utilization, high latency | Queue/preprocessing → execution |
| R11 | GPU saturation and queue growth | Execution capacity → response |
| R12 | CPU throttling | CPU limit → process |
| R13 | Wrong prediction on GPU | Precision/runtime → correctness |
| R14 | Model load never Ready | Initialization → readiness |
| R15 | Rollback still fails | Image/model/runtime compatibility |

For each, record lowest proven healthy layer, evidence, current actionable owner, minimal approved remediation, rollback and acceptance.

## 8. Acceptance scorecard

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| G1 Pinned runtime and model | | |
| G2 CPU/GPU benchmark rationale | | |
| G3 GPU allocation and scheduling | | |
| G4 CUDA/driver/framework compatibility | | |
| G5 Host RAM vs VRAM distinguished | | |
| G6 Ready and golden inference | | |
| G7 Latency, queue and utilization | | |
| G8 Rollback and cleanup | | |

## 9. Evidence report

```text
Date/time / approval:
TrueFoundry workspace / deployment:
Cluster, node, GPU model:
Image digest / model version:
Runtime and framework versions:
NVIDIA driver and CUDA compatibility:
Pod CPU/RAM/GPU resources:
Scheduler and device plugin evidence:
Host memory / VRAM peaks:
CPU/GPU utilization and queue:
Golden inference and latency:
R1–R15 classifications:
Lowest Proven Healthy Layer:
First Failed Transition:
Rollback owner / known-good artifact:
Post-recovery health:
Cleanup and unexecuted steps:
```

**Next:** Chapter 6.4 — Inference Endpoints, Request Lifecycle and API Reliability.
