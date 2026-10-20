# Chapter 093 — Validate Drivers CUDA and NCCL

**Part 12 — Enterprise Operations and Final Capstones | AI Infrastructure Engineering & SRE**

**Status:** Documentation published. GPU lab execution, measured baselines, independent technical/security review and production acceptance **PENDING**.

## 1. Mission

Establish a reproducible, evidence-based validation process for NVIDIA GPU drivers, CUDA runtimes, container tooling, Kubernetes GPU device plugins and NCCL communication before accepting a GPU cluster for training or inference. Detect compatibility, topology, networking and hardware faults early; define safe rollback and on-call response.

**Prerequisites:** Chapters 091–092, approved GPU cluster staging context, GPU node inventory, NVIDIA GPU Operator/device plugin, Linux/Kubernetes administration and permission to run controlled GPU test workloads.

### Objectives

- Distinguish **host kernel driver**, **driver-advertised CUDA compatibility**, **CUDA toolkit/runtime**, **framework build** and **NCCL library** versions.
- Verify node discovery, container GPU access, GPU health and telemetry.
- Test single-GPU CUDA, intra-node and multi-node NCCL collectives.
- Diagnose PCIe/NVLink/NVSwitch, RDMA/InfiniBand, DNS, MTU, firewall and NCCL interface selection.
- Establish throughput/latency baselines, safe thresholds and fault-isolation runbooks.
- Publish evidence without asserting tests were executed.

## 2. Dependency and compatibility chain

\`\`\`text
GPU hardware / firmware
         ↓
Host kernel + NVIDIA kernel modules
         ↓
NVIDIA driver / device nodes
         ↓
Container runtime + NVIDIA Container Toolkit
         ↓
Kubernetes GPU Operator / device plugin / NFD
         ↓
Container CUDA runtime + framework build
         ↓
NCCL / network transport / GPU topology
         ↓
Distributed training or inference workload
\`\`\`

**Critical distinction:** \`nvidia-smi\` displays a **CUDA Version** representing the maximum CUDA driver API compatibility supported by the installed driver; it does **not** prove the CUDA toolkit is installed or that \`nvcc\` matches it. A newer driver often supports older compatible CUDA runtimes, but verify vendor-supported combinations for the exact stack.

### Version matrix to record

| Component | Evidence | Why it matters |
|---|---|---|
| GPU model, HBM, MIG mode | \`nvidia-smi -L\`, \`nvidia-smi -q\` | memory, partitioning, supported features |
| OS/kernel | \`uname -r\`, image ID | driver module ABI |
| NVIDIA driver | \`nvidia-smi\`, module logs | device control |
| CUDA toolkit | \`nvcc --version\` if installed | compilation toolchain |
| Container CUDA | image digest and runtime files | application dependencies |
| PyTorch/TensorFlow | framework build/version | ABI and supported kernels |
| NCCL | library/build version and test output | collective communication |
| NIC/OFED/RDMA | \`ibv_devinfo\`, \`ip link\` if available | cross-node fabric |
| GPU Operator/device plugin | chart/images and pod health | Kubernetes GPU registration |

Never copy a compatibility table from a different cluster without checking OS, GPU architecture, driver branch, CUDA runtime and framework release.

## 3. Safety and change management

Run the exercises only in an approved **staging** namespace with disposable data and bounded GPU reservations. Production node inspection may be read-only. NCCL all-reduce tests can saturate GPU, memory, PCIe and RDMA fabric; schedule them away from customer workloads. Do not unload drivers, reboot nodes, modify firmware, change MIG mode or disable security controls outside approved maintenance.

Pin test images by digest in a real implementation; the examples below are diagnostic patterns, not guaranteed working deployments. Never set \`NCCL_IB_DISABLE=1\`, \`NCCL_P2P_DISABLE=1\` or \`NCCL_SOCKET_IFNAME\` as a universal fix; such changes alter transport behavior and require measured justification.

## 4. Layer 1 — Host GPU discovery

Run on an approved GPU node:

\`\`\`bash
uname -r
nvidia-smi
nvidia-smi -L
nvidia-smi --query-gpu=index,name,uuid,driver_version,memory.total,temperature.gpu --format=csv
nvidia-smi topo -m
lsmod | grep -E '^nvidia' || true
journalctl -k --no-pager | grep -Ei 'NVRM|Xid|nvidia' | tail -60
\`\`\`

**Pass evidence:** expected GPU count/model, no unexplained missing devices, supported driver and no active critical Xid/ECC issues. Some commands require host permissions or are unavailable in managed environments. Record \`NOT_ACCESSIBLE\`, not \`PASS\`.

### Common host failures

- \`NVIDIA-SMI has failed\`: driver missing, kernel mismatch, device inaccessible or GPU unhealthy.
- Missing GPU: PCIe enumeration, node image, firmware, device plugin or MIG configuration.
- Driver module not loading: secure boot/module signing, incompatible kernel, DKMS build or operator collision.
- Repeated Xid/ECC: hardware, power, memory or driver fault; triage by exact code and vendor guidance.

## 5. Layer 2 — Kubernetes registration and isolation

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.nvidia\.com/gpu}{"\t"}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'
kubectl get pods -A | grep -Ei 'gpu-operator|device-plugin|dcgm|nfd' || true
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

GPU allocatable count can differ from physical count because of MIG profiles, health or device-plugin strategy. Verify **requested resource names** and scheduling policies; a \`Ready\` node is not necessarily a healthy GPU node.

### Minimal staging smoke-pod pattern

\`\`\`yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-smoke
  namespace: gpu-validation
spec:
  restartPolicy: Never
  containers:
    - name: test
      image: REPLACE_WITH_APPROVED_CUDA_IMAGE_DIGEST
      command: ["sh", "-c", "nvidia-smi && sleep 5"]
      resources:
        limits:
          nvidia.com/gpu: 1
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
        seccompProfile:
          type: RuntimeDefault
\`\`\`

Replace the placeholder with an approved CUDA-capable image digest and create the namespace only in authorized staging. Validate admission/security policies and any image-specific user/runtime requirements. \`nvidia-smi\` inside a container proves device visibility, **not** successful CUDA kernel execution.

## 6. Layer 3 — CUDA runtime and kernel test

On an authorized environment with a CUDA-enabled PyTorch build and a visible GPU:

\`\`\`python
import torch
print("torch:", torch.__version__)
print("compiled_cuda:", torch.version.cuda)
print("cuda_available:", torch.cuda.is_available())
assert torch.cuda.is_available(), "CUDA not available"
print("device:", torch.cuda.get_device_name(0))
a = torch.randn((1024, 1024), device="cuda")
b = torch.randn((1024, 1024), device="cuda")
c = a @ b
torch.cuda.synchronize()
assert torch.isfinite(c).all().item()
print("CUDA_MATMUL_PASS", float(c.norm().item()))
\`\`\`

Capture image digest, framework build, GPU UUID and driver version. A failed assertion must remain **FAIL** until root cause is understood. The test uses GPU memory/compute and must not run on a shared production GPU without approval.

## 7. Layer 4 — GPU topology and interconnect

Use \`nvidia-smi topo -m\` to map GPU-to-GPU paths (e.g., NVLink/NVSwitch or PCIe) and GPU-to-NIC affinity. Compare with physical inventory and workload placement. The existence of an NVLink-capable GPU does not guarantee the expected links are active or that a job spans compatible devices.

Benchmark peer-to-peer paths with vendor-approved CUDA samples where available. Record payload size, warmup, iterations, GPU clocks/power state, contention and test image. A topology mismatch can severely reduce all-reduce throughput while single-GPU CUDA appears healthy.

## 8. Layer 5 — NCCL intra-node collectives

Use a reviewed \`nccl-tests\` build matching the installed NCCL/CUDA environment. Example **inside a prepared test image with \`all_reduce_perf\` installed**:

\`\`\`bash
./build/all_reduce_perf -b 8M -e 1G -f 2 -g 8 -w 5 -n 20
\`\`\`

The command assumes **eight accessible compatible GPUs**, sufficient HBM, a built binary at that path and an isolated node. Reduce \`-g\` to the actual allocated count. Inspect both \`algbw\` and \`busbw\` as reported by the test, plus validation errors and timeouts; they are not interchangeable. Compare against **same-SKU, same-topology** approved baselines rather than generic internet numbers.

**Pass gate:** no correctness errors/hangs; bandwidth and latency within approved measured tolerance, with known noise and contention documented.

## 9. Layer 6 — NCCL multi-node communication

A successful single-node all-reduce does not validate RDMA or inter-node fabric. For multi-node tests, validate:

- identical supported NCCL/CUDA/framework compatibility across ranks;
- MPI/launcher and rank/world-size correctness;
- NIC selection and routability, DNS, ports, firewall and MTU;
- InfiniBand/RoCE link state, PFC/ECN where applicable, GID and RDMA permissions;
- GPUDirect RDMA support and GPU-to-NIC topology;
- timeouts, retries, packet drops and switch congestion;
- consistent security policy without unrestricted privileged access.

Illustrative launcher pattern (not directly deployable):

\`\`\`bash
# In an approved MPI-enabled test environment with hostfile and SSH/MPI configured:
mpirun -np 16 -N 8 --hostfile hosts \
  ./build/all_reduce_perf -b 8M -e 1G -f 2 -g 1 -w 5 -n 20
\`\`\`

**Caution:** process-to-GPU mapping, launcher flags and multi-node \`nccl-tests\` options depend on the installed MPI/build and deployment method. In this example, each process requests one GPU; confirm each rank sees the correct distinct GPU. Do not interpret a successful TCP fallback as proof of RDMA.

### Diagnostic logging

\`\`\`bash
export NCCL_DEBUG=INFO
export NCCL_DEBUG_SUBSYS=INIT,NET
# Execute only the approved bounded test command; capture sanitized logs.
\`\`\`

Debug logs can expose hostnames and network topology; restrict access and disable verbose logging afterward.

## 10. Performance baseline methodology

| Test | Baseline dimensions | Acceptance |
|---|---|---|
| GPU discovery | GPU UUID/SKU/driver/health | exact inventory |
| CUDA compute | image/framework/shape/precision | correctness, measured throughput |
| Intra-node all-reduce | GPU count, payload, topology | correctness and same-SKU tolerance |
| Inter-node all-reduce | nodes, NIC, transport, payload | correctness and approved fabric baseline |
| Training microbenchmark | batch/sequence, model, checkpoint | steps/sec and quality |
| Serving smoke/load | model, concurrency, prompt/output mix | p99, TTFT/TPOT, error rate |
| Resilience | GPU/node fault in staging | recovery and no data loss |

Warm up GPU clocks and kernels; run multiple repetitions and report median, dispersion and worst-case. Avoid comparing runs with different GPU count, tensor precision, contention, clocks or network paths. Preserve raw logs and test harness revision.

## 11. Compatibility and upgrade strategy

1. Inventory all driver, CUDA, NCCL, GPU Operator, kernel and framework versions.
2. Consult exact NVIDIA/vendor release compatibility notes for the planned combination.
3. Verify support for GPU architecture, OS, MIG and RDMA stack.
4. Canary one **staging** node pool with pinned images and controlled workload.
5. Run single-GPU CUDA, intra-node and inter-node collectives.
6. Run training restart and inference SLO canaries.
7. Record rollback strategy: node image replacement/blue-green if in-place driver downgrade is unsafe.
8. Promote only after independent review, then monitor Xid, performance and scheduling.

Do not assume that a driver update is harmless because \`nvidia-smi\` still runs.

## 12. Six production runbooks

**R1 — Driver unavailable:** inspect node kernel, module signing, driver/container runtime and GPU Operator logs; compare known-good node; quarantine faulty node and use approved replacement.

**R2 — CUDA kernel fails:** check framework CUDA build, GPU architecture, driver compatibility, HBM/OOM and container device visibility; reproduce minimal matrix test.

**R3 — NCCL all-reduce hangs:** inspect rank/world size, failed GPU, interface, firewall, MTU, RDMA state and timeouts; isolate single-node versus cross-node; avoid disabling transports blindly.

**R4 — NCCL bandwidth regression:** compare same-SKU baseline, NVLink/NIC topology, throttling, congestion, GPUDirect settings, payload and contention; rollback recent change if verified.

**R5 — GPU Xid/ECC incident:** capture code/time/GPU UUID, inspect vendor guidance, isolate affected hardware and preserve workload checkpoint; escalate replacement.

**R6 — Driver/Operator rollout failure:** halt promotion, restore approved node image/chart configuration, validate device discovery and test matrix, document corrective action.

## 13. Sixteen hands-on exercises

1. Capture cluster context and GPU node inventory.
2. Record GPU/driver/kernel/operator compatibility matrix.
3. Inspect \`nvidia-smi\`, GPU UUIDs and health.
4. Review Xid/ECC and thermal/power signals.
5. Validate Kubernetes allocatable GPU resources.
6. Inspect GPU Operator/device plugin health.
7. Run staging container GPU visibility smoke test.
8. Run CUDA kernel/matmul correctness test.
9. Map NVLink/PCIe/GPU-to-NIC topology.
10. Execute intra-node NCCL all-reduce.
11. Inspect RDMA/NIC/MTU configuration.
12. Execute bounded multi-node NCCL all-reduce.
13. Measure repeatability and baseline variance.
14. Simulate one compatibility or transport failure in staging.
15. Rehearse canary upgrade and safe rollback.
16. Produce evidence and independent acceptance score.

## 14. Twelve troubleshooting scenarios

| Symptom | Primary investigation |
|---|---|
| \`nvidia-smi\` missing | container/host package and PATH |
| \`nvidia-smi\` cannot communicate | driver/module/device |
| CUDA unavailable in framework | build/runtime/driver |
| CUDA OOM | HBM, fragmentation, tensor shape |
| GPU count lower than expected | device plugin, MIG, health |
| Pod Pending with GPU request | quota, taints, node supply |
| NCCL timeout intra-node | rank mapping, topology, GPU health |
| NCCL timeout cross-node | NIC/firewall/MTU/RDMA |
| Bandwidth below baseline | PCIe/NVLink/fabric contention |
| TCP fallback masks RDMA issue | transport selection |
| Xid/ECC alarms after upgrade | driver/hardware regression |
| CUDA works but training fails | framework/NCCL/model state |

## 15. Acceptance gate (100 points)

| Domain | Points |
|---|---:|
| Inventory, versions and compatibility | 15 |
| Host driver and GPU health | 15 |
| Kubernetes/container GPU access | 15 |
| CUDA correctness and compute baseline | 15 |
| Intra-node NCCL correctness/performance | 15 |
| Multi-node NCCL and fabric verification | 15 |
| Upgrade, rollback, evidence and peer review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 and no critical CUDA correctness failure, unverified GPU isolation, NCCL collective corruption/hang, unresolved critical Xid/ECC or absent rollback. Runtime tests and independent review remain **NOT RUN**.

## 16. References

- [NVIDIA CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
- [NVIDIA CUDA Installation Guide](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/)
- [NVIDIA NCCL User Guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [NVIDIA nccl-tests](https://github.com/NVIDIA/nccl-tests)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [Kubernetes Schedule GPUs](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)

**Next:** Chapter 094 — Deploy Kubernetes GPU Workloads.
