# Chapter 019 — CUDA Processes, Streams and Contexts

**Part 03 — CUDA, GPU Memory and Communication · Chapter 019 of 100**

**Publication:** Documentation complete. Executable tests, staging GPU qualification, security review and production acceptance are separately tracked and must not be inferred from publication.

## Learning objectives
Understand CUDA process isolation, primary and user-created contexts, default/per-thread streams, asynchronous execution, events and synchronization; troubleshoot device visibility, context contention, multi-process service (MPS), GPU scheduling and cross-process memory pressure; design reproducible SRE experiments and safe incident recovery.

## 1. CUDA execution architecture
```text
Linux processes / container namespaces / Kubernetes pods
        |                 |                 |
     Process A         Process B         Process C
   CUDA runtime      Driver API       Framework API
        |                 |                 |
   Primary context   User context     Primary context
        |                 |                 |
   streams/events    streams/events   streams/events
        +-----------------+-----------------+
                       GPU
        scheduler / SMs / registers / L2 / VRAM
```
A **CUDA context** holds device execution state, allocations, modules and scheduling state. The runtime commonly uses a device's primary context; explicit Driver API contexts have distinct ownership/lifecycle semantics. A **stream** is an ordered sequence of GPU work, while independent streams may execute concurrently subject to dependencies and resources. A **process** may create multiple streams, and different processes may have separate contexts; context switching and memory overhead can reduce effective throughput. GPU process isolation does not by itself guarantee hard tenant security or deterministic performance.

## 2. Critical distinctions
| Concept | Meaning | SRE consequence |
|---|---|---|
| Host process | OS PID and namespace | PID shown on host may differ inside container |
| CUDA context | Per-device execution state | context startup, teardown and memory overhead |
| CUDA stream | Ordered command queue | operations in different streams may overlap |
| Default stream | Legacy or per-thread semantics | implicit synchronization varies with compilation/runtime |
| CUDA event | Device-side ordering/timing | measure work with explicit synchronization |
| Host synchronization | `cudaDeviceSynchronize`, stream sync | reveals deferred asynchronous errors |
| MPS | NVIDIA multi-process service | sharing and isolation constraints; vendor/version specific |
| MIG | Hardware GPU instance partitioning | distinct resources and memory isolation where supported |
| CUDA IPC | Inter-process memory/event sharing | lifetime, trust and container boundary implications |

A CUDA launch can return before the GPU has finished. Errors may surface later at a synchronization point. CUDA event timing excludes unrelated host-side queueing unless measured separately. Use explicit dependencies and validate behavior against the active CUDA runtime version.

## 3. Read-only GPU and process inspection
On an authorized host:
```bash
date -u
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version,memory.used,utilization.gpu --format=csv
nvidia-smi --query-compute-apps=gpu_uuid,pid,process_name,used_gpu_memory --format=csv
nvidia-smi pmon -c 1
ps -eo pid,ppid,user,etime,rss,args | head -n 40
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|CUDA|oom' | tail -n 100
```
Per-process accounting can be unavailable or restricted, particularly with MIG, containers and some driver versions. Do not treat `N/A` as zero. Process command lines may contain sensitive values; sanitize before sharing.

For Kubernetes:
```bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl describe node APPROVED_GPU_NODE
kubectl describe pod APPROVED_POD -n APPROVED_NAMESPACE
kubectl get events -A --sort-by=.lastTimestamp
```
Correlate GPU UUID, pod UID, namespace, node, container and process; PID alone is not a stable fleet identifier. Inspect CUDA_VISIBLE_DEVICES and NVIDIA_VISIBLE_DEVICES only in approved contexts, remembering logical device indices may be remapped.

## 4. Executable CUDA stream lab (staging GPU required)
Save as `stream_demo.cu`:
```cuda
#include <cstdio>
#include <cuda_runtime.h>
#define CHECK(call) do { cudaError_t e = (call); if (e != cudaSuccess) {   std::fprintf(stderr, "%s:%d %s\n", __FILE__, __LINE__, cudaGetErrorString(e)); return 1; } } while (0)
__global__ void add(int *p, int n) {
  int i = blockIdx.x * blockDim.x + threadIdx.x;
  if (i < n) p[i] += 1;
}
int main() {
  int count = 0;
  CHECK(cudaGetDeviceCount(&count));
  if (count == 0) return 2;
  constexpr int n = 4096;
  int *d = nullptr;
  cudaStream_t s1, s2;
  CHECK(cudaMalloc(&d, n * sizeof(int)));
  CHECK(cudaMemset(d, 0, n * sizeof(int)));
  CHECK(cudaStreamCreateWithFlags(&s1, cudaStreamNonBlocking));
  CHECK(cudaStreamCreateWithFlags(&s2, cudaStreamNonBlocking));
  add<<<8, 256, 0, s1>>>(d, n / 2);
  CHECK(cudaGetLastError());
  add<<<8, 256, 0, s2>>>(d + n / 2, n / 2);
  CHECK(cudaGetLastError());
  CHECK(cudaStreamSynchronize(s1));
  CHECK(cudaStreamSynchronize(s2));
  int result[n];
  CHECK(cudaMemcpy(result, d, sizeof(result), cudaMemcpyDeviceToHost));
  for (int v : result) if (v != 1) return 3;
  CHECK(cudaStreamDestroy(s1));
  CHECK(cudaStreamDestroy(s2));
  CHECK(cudaFree(d));
  std::puts("stream smoke passed");
  return 0;
}
```
Compile and run on an **approved staging GPU**:
```bash
nvcc -O2 stream_demo.cu -o stream_demo
./stream_demo
```
This demonstrates two nonblocking streams operating on **disjoint** device-memory ranges and explicit synchronization; it does not guarantee overlap. To test overlap, add CUDA events and compare against serialized execution with warm-up and multiple repetitions. Do not use concurrent writes to the same address without synchronization.

## 5. Process and context lifecycle
1. Discover visible devices and map logical ordinal to stable GPU UUID.
2. Initialize CUDA in the correct process; avoid forking a process after CUDA initialization without framework-supported handling.
3. Select device, create streams, allocate memory and load modules.
4. Submit work with explicit stream/event dependencies; check launch and asynchronous errors.
5. Synchronize where correctness requires it; avoid unnecessary global synchronization.
6. Release streams, events and allocations; terminate workers cleanly and verify resources are reclaimed.
7. For Python multiprocessing, use framework-supported `spawn` or `forkserver` approaches where documented; test worker lifecycle and restart behavior.

## 6. MPS, MIG and tenancy
**MPS** can reduce overhead for certain multi-process workloads and improve concurrency, but its scheduling, resource sharing and isolation characteristics depend on GPU architecture and MPS version. It is not a substitute for security boundaries or admission control. **MIG** creates supported hardware-partitioned GPU instances with isolated resources; its relationship to MPS is SKU- and configuration-specific. Review NVIDIA guidance and your security policy before combining them.

Kubernetes `nvidia.com/gpu` requests reserve GPU resources through the device plugin; Kubernetes does not automatically cap CUDA context count or per-process VRAM. Workload admission should consider concurrency, GPU memory budget, tenant isolation, node affinity and capacity.

## 7. Observability and performance
Collect context initialization latency, process restarts, GPU memory used, allocator reserved/allocated, kernel duration, stream wait time, GPU active time, SM occupancy, Xid, workload queue time, throughput, p95/p99 inference latency and training step time. Use Nsight Systems/Compute only with approved access and bounded profiling overhead. A high GPU utilization percentage alone cannot prove stream-level efficiency.

## 8. Ten troubleshooting scenarios
| Symptom | Evidence | Diagnosis |
|---|---|---|
| device ordinal invalid | visible-device mapping | wrong index/MIG remap |
| CUDA init fails in child | multiprocessing start method | inherited unsafe CUDA state |
| GPU memory not reclaimed | process/context lifecycle | still-live process or allocation |
| work appears serialized | default-stream semantics, dependencies | implicit synchronization |
| nondeterministic output | concurrent overlapping writes | missing dependency |
| asynchronous error reported late | launch vs synchronize | earlier kernel failure |
| high latency with many processes | context switching, memory | oversubscription |
| MPS server unavailable | MPS daemon/logs | service/permission mismatch |
| host sees GPU, pod does not | device plugin/toolkit | runtime visibility |
| Xid during workload | NVRM logs | hardware/driver/kernel fault |

## 9. Eight incident simulations
1. Worker fork after CUDA init causes initialization failure; reproduce safely and adopt supported process start method.
2. Stream race corrupts synthetic results; add event ordering and deterministic tests.
3. New release creates excessive contexts; measure startup, memory and latency and roll back.
4. MPS daemon failure impacts colocated jobs; disable unsafe admission and restore approved configuration.
5. CUDA error surfaces only at synchronization; correlate prior kernel launch and log the original failure.
6. Pod reports invalid ordinal after MIG layout change; reconcile UUID/profile and deployment resource.
7. GPU memory stays occupied after workload exit; inspect remaining processes before approved intervention.
8. Multi-tenant workload causes p99 latency regression; adjust isolation and scheduling after capacity review.

## 10. Production runbooks
**RB-019-A — Device/process correlation:** capture UTC, node, GPU UUID, pod UID, host/container PID and CUDA visibility; redact process data.

**RB-019-B — Context initialization:** verify driver/runtime, process start method, device selection and kernel logs.

**RB-019-C — Stream correctness:** reproduce on staging, inspect default-stream mode, events, dependencies and synchronization.

**RB-019-D — Memory/context cleanup:** identify owning processes and live allocations; never kill shared processes without approval.

**RB-019-E — MPS incident:** identify server/client scope, isolation policy, restart safety and workload checkpoint recovery.

**RB-019-F — Kubernetes GPU visibility:** verify device-plugin resources, toolkit, container image and MIG mappings.

**RB-019-G — Performance regression:** compare context count, kernel timing, CPU launch overhead, workload SLO and rollback.

**RB-019-H — RCA/acceptance:** preserve evidence, code/driver versions, tested corrective action, SLO and independent review.

## 11. Executable companion lab
[Chapter 019 — CUDA Context and Stream Operations Lab](../labs/019-cuda-processes-streams-contexts-check.md) includes a synthetic Python dependency-graph validator with pytest and staging CUDA smoke. Synthetic tests do **not** prove GPU concurrency, stream safety or MPS isolation.

## 12. 100-point final assessment
| Area | Points |
|---|---:|
| Process/context/stream architecture | 20 |
| Asynchronous correctness and lifecycle | 20 |
| MPS/MIG/Kubernetes operations | 20 |
| Incident response and runbooks | 20 |
| Lab, tests and evidence | 20 |
| **Total** | **100** |

Pass >=90 and no critical correctness, data-loss, tenant isolation, security or SLO failure.

## References
- [CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
- [CUDA Runtime Stream Management](https://docs.nvidia.com/cuda/cuda-runtime-api/group__CUDART__STREAM.html)
- [CUDA Driver API](https://docs.nvidia.com/cuda/cuda-driver-api/)
- [NVIDIA MPS Documentation](https://docs.nvidia.com/deploy/mps/)
- [NVIDIA MIG User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

**Next:** Chapter 020 — MIG Partitioning and Isolation.
