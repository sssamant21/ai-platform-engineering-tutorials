# Chapter 022 — NCCL Fundamentals and Collective Operations

**Part 03 — CUDA, GPU Memory and Communication · Chapter 022 of 100**

**Publication:** Documentation published. Offline test execution, live NCCL/GPU validation, security review and production acceptance are independent pending milestones.

## Learning objectives
Understand NVIDIA Collective Communications Library (NCCL) communicators, ranks, collective semantics, CUDA stream integration, intra-node NVLink/NVSwitch versus inter-node network paths, and operational readiness. Build a reproducible collective smoke test; diagnose topology, rank mismatch, data correctness, GPU memory, timeout and bandwidth regressions; define safe recovery and production SLO gates.

## 1. Architecture

```text
PyTorch Distributed / TensorFlow / custom CUDA application
       |                |                 |
    Rank 0           Rank 1            Rank N-1
       |                |                 |
       +----- NCCL communicators / collective APIs ----+
                         |
            CUDA streams and device buffers
                         |
         NCCL topology and transport selection
           /                           \
     Intra-node                       Inter-node
 NVLink / NVSwitch / PCIe      NIC/HCA, RDMA, EFA, RoCE
           \                           /
             CUDA driver / GPU hardware
```

NCCL implements collective communication for supported NVIDIA GPUs. Its algorithm and transport choices depend on NCCL version, topology, message size, communicator size, network plugins, environment and runtime. A high-bandwidth NVLink path does **not** eliminate cross-node NIC bottlenecks. NCCL performance and correctness require every rank to follow compatible collective ordering and buffer/count/datatype semantics.

## 2. Collective operations

| Collective | Input/output behavior | Common AI use |
|---|---|---|
| AllReduce | combines all rank inputs; distributes result to every rank | gradient aggregation |
| Reduce | combines inputs; result on designated root | aggregation |
| Broadcast | copies root buffer to all ranks | parameter/config distribution |
| AllGather | gathers each rank's contribution to every rank | distributed tensor assembly |
| ReduceScatter | reduces across ranks and scatters result segments | sharded optimizer and tensor parallelism |
| AllToAll | each rank exchanges data segments with every rank (API support/version-dependent) | expert/token routing |
| Send/Recv | point-to-point communication | pipeline stages and custom topologies |

**Example AllReduce:** four ranks each provide a scalar 1, 2, 3 and 4. With SUM reduction, each rank receives **10**. Floating-point reductions may differ slightly by reduction order and hardware; use tolerances appropriate to datatype and application.

**ReduceScatter detail:** in the standard equal-count API, each rank supplies an input containing a segment for each destination; the reduced result is partitioned across ranks. AllGather is not interchangeable with Gather-to-root.

## 3. Communicator lifecycle and ordering

1. Determine the exact rank/world-size mapping and GPU UUID per process.
2. Initialize NCCL communicators through the application/framework-supported bootstrap.
3. Allocate correctly sized device buffers and select CUDA streams.
4. Issue compatible collective operations in matching order across ranks.
5. Synchronize or use events as required before consuming results.
6. Monitor asynchronous NCCL/CUDA errors, application progress and watchdog signals.
7. Abort/clean up communicators safely on a failed rank; recover the **whole distributed job** according to framework guarantees.
8. Restore from a consistent checkpoint if the collective state is uncertain.

CUDA/NCCL calls can be asynchronous. An apparently successful enqueue does not prove completion; monitor CUDA events and NCCL asynchronous error state where supported. Never retry one rank independently into a mismatched collective sequence.

## 4. Read-only preflight

On an authorized GPU node:

```bash
date -u
nvidia-smi -L
nvidia-smi topo -m
nvidia-smi --query-gpu=uuid,name,driver_version,memory.free --format=csv
lspci -nn | grep -Ei 'NVIDIA|Ethernet|InfiniBand'
ip -brief link
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|PCIe|AER|NCCL' | tail -n 100
```

For Kubernetes:

```bash
kubectl config current-context
kubectl get pods -A -o wide
kubectl describe pod APPROVED_POD -n APPROVED_NAMESPACE
kubectl describe node APPROVED_GPU_NODE
kubectl get events -A --sort-by=.lastTimestamp
```

Record driver/CUDA/NCCL version, GPU count, node SKU, NVLink topology, NIC mapping, container image digest, rank-to-GPU mapping, network policies, pod placement and workload SLO. NCCL environment variables and logs may contain sensitive network/cluster details; sanitize before sharing.

## 5. Hands-on lab A — single-node NCCL collective smoke

Use an **approved staging node**, with a CUDA toolkit, NCCL headers/libraries, at least two supported GPUs and sufficient GPU memory. Save as `nccl_allreduce.cu`:

```cuda
#include <cstdio>
#include <cstdlib>
#include <vector>
#include <cuda_runtime.h>
#include <nccl.h>

#define CUDA_OK(call) do { cudaError_t e = (call); if (e != cudaSuccess) { \
  std::fprintf(stderr, "CUDA %s:%d %s\n", __FILE__, __LINE__, cudaGetErrorString(e)); \
  std::exit(1); } } while (0)
#define NCCL_OK(call) do { ncclResult_t e = (call); if (e != ncclSuccess) { \
  std::fprintf(stderr, "NCCL %s:%d %s\n", __FILE__, __LINE__, ncclGetErrorString(e)); \
  std::exit(2); } } while (0)

int main() {
  int available = 0;
  CUDA_OK(cudaGetDeviceCount(&available));
  if (available < 2) {
    std::fprintf(stderr, "requires >=2 visible GPUs\n");
    return 3;
  }
  constexpr int n = 2;
  int devices[n] = {0, 1};
  ncclComm_t comms[n];
  cudaStream_t streams[n];
  float *send[n], *recv[n];
  NCCL_OK(ncclCommInitAll(comms, n, devices));
  for (int i = 0; i < n; ++i) {
    CUDA_OK(cudaSetDevice(devices[i]));
    CUDA_OK(cudaStreamCreateWithFlags(&streams[i], cudaStreamNonBlocking));
    CUDA_OK(cudaMalloc(&send[i], sizeof(float)));
    CUDA_OK(cudaMalloc(&recv[i], sizeof(float)));
    float value = static_cast<float>(i + 1);
    CUDA_OK(cudaMemcpy(send[i], &value, sizeof(float), cudaMemcpyHostToDevice));
  }
  NCCL_OK(ncclGroupStart());
  for (int i = 0; i < n; ++i) {
    NCCL_OK(ncclAllReduce(send[i], recv[i], 1, ncclFloat, ncclSum, comms[i], streams[i]));
  }
  NCCL_OK(ncclGroupEnd());
  for (int i = 0; i < n; ++i) {
    CUDA_OK(cudaSetDevice(devices[i]));
    CUDA_OK(cudaStreamSynchronize(streams[i]));
    float result = 0;
    CUDA_OK(cudaMemcpy(&result, recv[i], sizeof(float), cudaMemcpyDeviceToHost));
    std::printf("rank=%d result=%.1f\n", i, result);
    if (result != 3.0f) return 4;
  }
  for (int i = 0; i < n; ++i) {
    CUDA_OK(cudaSetDevice(devices[i]));
    NCCL_OK(ncclCommDestroy(comms[i]));
    CUDA_OK(cudaFree(send[i]));
    CUDA_OK(cudaFree(recv[i]));
    CUDA_OK(cudaStreamDestroy(streams[i]));
  }
  std::puts("NCCL AllReduce smoke PASS");
  return 0;
}
```

Example compilation (include/library paths depend on installation):

```bash
nvcc -O2 nccl_allreduce.cu -lnccl -o nccl_allreduce
./nccl_allreduce
```

Expected for two ranks: each result is `3.0`. This example uses **one host process controlling two devices**; it is not a multi-node benchmark. Validate GPU visibility, NCCL version and library paths first. Avoid using this smoke test on production GPUs.

## 6. Hands-on lab B — nccl-tests (approved staging)

Use the upstream `nccl-tests` version matched to your NCCL/CUDA stack and documented build steps. For a single-node two-GPU canary, a representative command is:

```bash
./build/all_reduce_perf -b 8 -e 64M -f 2 -g 2
```

Record actual command-line options from `./build/all_reduce_perf -h`, test version, GPU UUIDs, message sizes, `algbw`, `busbw`, error checks, driver/NCCL/CUDA versions and repeatability. `busbw` is a benchmark-derived metric, not a physical link counter. **Do not compare it directly to raw NVLink advertised throughput** or infer topology from one throughput value. For multi-node tests, use the approved MPI/launcher and transport configuration and a network-capacity window.

## 7. Algorithm, protocol and transport selection

NCCL may select ring/tree and other algorithms and protocols according to workload and version. Do not hardcode a universal “best” setting. Environment variables such as `NCCL_DEBUG`, `NCCL_SOCKET_IFNAME`, `NCCL_IB_DISABLE`, `NCCL_ALGO`, and `NCCL_PROTO` can influence diagnostics or transport; support and semantics are version-dependent. Use diagnostic overrides only under change control, and remove them after the experiment.

A missing or incorrect NIC interface, RDMA capability, security group/network policy, GPU-to-NIC affinity, or network plugin can cause slow fallback or connection failures. Always separate **single-node** NVLink/NVSwitch evidence from **multi-node** transport evidence.

## 8. Reliability, telemetry and SLOs

Track job world size, healthy rank count, last completed collective/step, checkpoint age, GPU UUID, NCCL/CUDA/driver version, topology, network errors/retransmits, Xid/ECC, job throughput, p95/p99 collective duration, training step time and restart count. A collective may hang without an obvious GPU error; use application progress watchdogs and bounded timeouts appropriate to the workload.

Never assume a timeout proves which rank failed first. Correlate monotonic progress and UTC timestamps from all ranks. Avoid storing model input tensors or secrets in diagnostic logs.

## 9. Twelve troubleshooting scenarios

| Symptom | Evidence | First hypothesis |
|---|---|---|
| collective never completes | per-rank sequence, watchdog | rank/order mismatch |
| wrong AllReduce result | count, datatype, buffer, root | semantic mismatch |
| NCCL unhandled system error | network/driver logs | transport or environment |
| invalid usage | API contract, group order | application misuse |
| OOM during collective | device free, buffers, workspace | insufficient VRAM |
| single-node fast, multi-node slow | NIC/RDMA counters | inter-node bottleneck |
| only one GPU visible | device plugin, CUDA visibility | resource mapping |
| intermittent hang | Xid, rank death, node events | failed participant |
| NCCL version mismatch | image/linked library | incompatible deployment |
| throughput regression | topology, clocks, message sizes | changed benchmark conditions |
| wrong NIC selected | interface/route logs | transport configuration |
| repeated restart with no progress | checkpoint and launcher state | incomplete distributed recovery |

## 10. Ten production incident simulations

1. Rank 3 never enters AllReduce: detect collective sequence mismatch and stop/recover whole job.
2. One rank supplies a different element count: reject before NCCL call through an application contract gate.
3. One GPU fails with Xid: preserve evidence, quarantine node if needed and recover from checkpoint.
4. NIC misconfiguration causes TCP fallback: compare single-node and multi-node performance and restore approved transport.
5. Container update changes NCCL version: canary, compare compatibility and roll back.
6. Model rollout exceeds GPU memory: inspect buffer/activation/optimizer budgets and reduce load safely.
7. NVLink path changes after node replacement: revalidate topology, P2P and collectives.
8. MPI/launcher reports success while one rank stalled: use application-level progress and per-rank completion checks.
9. Telemetry exporter loses NCCL metrics: stop promotion until independent evidence is restored.
10. Network congestion increases p99 collective time: correlate NIC queue/drop metrics and application SLO.

## 11. Eight operational runbooks

**RB-022-A — Preflight:** capture GPU UUIDs, rank map, CUDA/NCCL versions, topology, NIC, container digest and approved benchmark scope.

**RB-022-B — Collective correctness:** verify world size, count/datatype, collective order, buffers and explicit completion checks.

**RB-022-C — Hang triage:** collect all-rank progress, launcher logs, Xid, network state and last completed checkpoint before restart.

**RB-022-D — Transport diagnostics:** distinguish intra-node NVLink/PCIe from inter-node NIC/RDMA; compare exact network configuration.

**RB-022-E — Memory/driver faults:** classify OOM, Xid, CUDA initialization and NCCL errors; avoid assuming a single root cause.

**RB-022-F — Performance regression:** run version-pinned, approved canaries with identical message sizes, topology and load.

**RB-022-G — Distributed recovery:** coordinate communicator teardown, checkpoint restore, rank reformation and data-integrity validation.

**RB-022-H — RCA/acceptance:** record timeline, blast radius, SLO impact, fix/rollback evidence, owner approval and follow-up tests.

## 12. Companion executable lab

[Lab 022 — NCCL Collective Contract and Reliability Gate](../labs/022-nccl-fundamentals-collectives-check.md) includes a Python 3.11+ synthetic collective-contract evaluator and pytest tests. It does **not** execute NCCL or validate live network performance.

## 13. 100-point assessment

| Domain | Points |
|---|---:|
| Collective semantics and NCCL architecture | 20 |
| Communicator lifecycle and correctness | 20 |
| Topology, transport and performance | 20 |
| Incidents and recovery runbooks | 20 |
| Executable lab and staging evidence | 20 |
| **Total** | **100** |

Pass >=90 and no critical correctness, checkpoint, security, telemetry or SLO blocker.

## References

- [NVIDIA NCCL User Guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [NVIDIA NCCL API](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/api.html)
- [NVIDIA NCCL Environment Variables](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html)
- [NVIDIA nccl-tests](https://github.com/NVIDIA/nccl-tests)
- [NVIDIA CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
- [PyTorch Distributed](https://docs.pytorch.org/docs/stable/distributed.html)

**Next:** Chapter 023 — NCCL Diagnostics and Hang Recovery.
