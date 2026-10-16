# Chapter 13.6 — NVIDIA Drivers, CUDA, GPU OOM and vLLM Performance

**Part 13 — Troubleshooting | TrueFoundry SRE canonical tutorial**

## Learning outcomes

Diagnose the full NVIDIA GPU inference stack: node hardware, driver, device plugin, container runtime, CUDA compatibility, GPU memory, vLLM model loading, KV cache, batching, TTFT/TPOT and request throughput. Differentiate hardware, scheduling, model configuration and traffic-shape regressions using objective evidence.

**[SAFE-READ]** approved Kubernetes, node and GPU telemetry. **[LAB-MUTATE]** bounded synthetic staging. **[CHANGE-CONTROL]** production driver/CUDA upgrades, GPU resets, node drains, model parameters, autoscaling or workload placement. Never run stress tests on shared production GPUs, reset an active device, or collect customer prompts/PHI in performance traces.

## 1. Layered failure model

```text
TrueFoundry model deployment + immutable revision
    → Kubernetes GPU Pod scheduled to eligible node
    → NVIDIA kernel driver / GPU firmware / hardware
    → NVIDIA device plugin / container runtime / GPU allocation
    → CUDA runtime + PyTorch + vLLM binary compatibility
    → model weights, tokenizer, tensor parallel, quantization
    → KV cache / prefill / decode / batching / queue
    → gateway / streaming client / SLO
```

A CUDA initialization error may stem from a driver mismatch, missing device injection, unsupported GPU architecture or application packaging. A high TTFT can stem from queueing, long input prompts, cold model loading or network latency. Investigate the failing layer before tuning.

## 2. Read-only triage commands

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl -n MODEL_NAMESPACE get pods -o wide
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE get events --sort-by=.lastTimestamp
kubectl describe node GPU_NODE
```

Confirm `nvidia.com/gpu` allocatable, GPU SKU, taints, node labels, Pod resource requests/limits and container termination reasons. `kubectl top` reports CPU and memory (when Metrics Server is installed), **not GPU utilization**.

If approved, run `nvidia-smi` in a GPU-enabled diagnostic environment where the binary and devices exist:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,driver_version,memory.total,memory.used,utilization.gpu,temperature.gpu,power.draw --format=csv
```

Some fields are unsupported on particular GPU/MIG configurations; record unsupported values rather than inventing measurements. Avoid sharing full GPU UUIDs where infrastructure identifiers are restricted.

## 3. NVIDIA driver and device plugin

Check whether the GPU appears on the host, whether the device plugin advertises allocatable resources and whether the Pod receives the requested devices. Use vendor/cluster-supported node diagnostics for driver errors, XID, ECC and PCIe events. `nvidia-smi` succeeding on the host does not prove the Pod sees CUDA. Conversely, a Pending Pod without GPU allocatable may be a device plugin/driver/bootstrap failure.

For MIG, match the exact exposed resource profile and configured isolation. Fractional sharing, time-slicing and MIG are not interchangeable and require platform-specific validation.

## 4. CUDA and runtime compatibility

Capture driver version, CUDA driver capability, container CUDA runtime, PyTorch build, vLLM version, Python version, GPU compute capability, NCCL (if tensor parallel) and image digest. Driver `nvidia-smi` CUDA Version represents supported driver API capability, **not necessarily** the installed toolkit version. Validate compatibility from release documentation and the deployed binary rather than assuming latest is best.

Common signatures: `no CUDA-capable device`, `CUDA driver version is insufficient`, `invalid device ordinal`, unsupported architecture, NCCL initialization errors and CUDA illegal memory access. Capture sanitized error and exact revision.

## 5. Distinguish GPU OOM, host OOM and KV cache pressure

| Failure | Evidence | Typical direction |
|---|---|---|
| GPU CUDA OOM | CUDA allocator/device memory exception | model fit, context, KV cache, batch/concurrency |
| Kubernetes OOMKilled | container lastState reason OOMKilled | host RAM request/limit, loading peak |
| KV cache exhaustion | vLLM engine logs, cache/queue metrics | context length, concurrency, cache configuration |
| GPU allocation absent | Pod device visibility/allocatable | plugin, runtime, requests, node |
| GPU memory fragmentation | allocator traces, workload pattern | allocator/config/version investigation |

Do not reflexively raise `gpu_memory_utilization` or `max_model_len`: both can increase pressure. Quantization, tensor parallelism and context-length changes have correctness, latency and capacity tradeoffs; validate in staging with real distribution-shaped synthetic traffic.

## 6. vLLM performance model

Break request time into admission/queue, prefill, first token, decode and network streaming. Define SLIs explicitly:

- **TTFT:** request accepted until first generated token, with consistent measurement boundary.
- **TPOT:** per-output-token generation time after first token, specifying calculation method.
- **End-to-end latency:** request start through final streamed response.
- **Goodput:** requests/tokens meeting SLO, not merely raw tokens/sec.
- **Completion ratio:** completed streams divided by eligible started streams.

Input tokens, output tokens, concurrency, prefix caching, batch scheduling and speculative decoding may affect results. Compare like-for-like workload mix, model revision, GPU SKU, quantization and SLO windows.

## 7. Metrics and dashboards

Use deployed-version vLLM exporter metrics, NVIDIA DCGM exporter, Kubernetes Events, container logs, ingress metrics and synthetic probes. Metric names and labels vary across versions; verify against the actual `/metrics` endpoint and exporter documentation. Correlate GPU utilization/memory with queue, running/waiting requests, TTFT/TPOT, model warmup, error rate and output token distribution. Missing metrics are an observability incident, not evidence of zero usage.

## 8. Performance investigation workflow

1. Scope impacted models/regions and UTC interval; establish synthetic SLI.
2. Compare last known good image/model/driver/TrueFoundry revision.
3. Identify node/GPU health and allocatable device visibility.
4. Classify GPU OOM, host OOM, startup, NCCL or saturation.
5. Compare traffic mix, concurrency, queue and model cold starts.
6. Verify GPU memory headroom, KV cache and batch configuration.
7. Test one change in approved staging with reproducible workload.
8. Canary through TrueFoundry desired state, monitor SLO/cost and retain rollback.
9. Validate streaming completion and N-1 capacity after recovery.

## 9. Sixteen operational runbooks

1. GPU node not reporting devices.
2. NVIDIA device plugin unhealthy.
3. Driver/runtime mismatch.
4. CUDA unavailable inside Pod.
5. MIG profile mismatch.
6. XID/ECC/hardware incident.
7. Host OOMKilled.
8. CUDA GPU OOM.
9. KV cache pressure.
10. Model load/cold-start failure.
11. Tensor parallel/NCCL failure.
12. High TTFT and prefill regression.
13. High TPOT and decode regression.
14. Queue/concurrency saturation.
15. Streaming completion regression.
16. Safe model tuning, rollback and cost/SLO validation.

## 10. Forty failure scenarios

| # | Scenario | # | Scenario |
|---|---|---|---|
| 01 | GPU missing from node | 21 | Model weights too large |
| 02 | Device plugin down | 22 | Context length too high |
| 03 | GPU allocatable zero | 23 | KV cache exhausted |
| 04 | Wrong node GPU SKU | 24 | GPU memory fragmentation |
| 05 | MIG profile mismatch | 25 | Excessive concurrency |
| 06 | Time-slicing mismatch | 26 | Batch configuration regression |
| 07 | NVIDIA driver mismatch | 27 | High input token distribution |
| 08 | Container CUDA mismatch | 28 | High output token distribution |
| 09 | GPU architecture unsupported | 29 | TTFT p95 regression |
| 10 | CUDA device ordinal invalid | 30 | TPOT p95 regression |
| 11 | GPU not injected in Pod | 31 | Queue wait spike |
| 12 | XID event | 32 | Cold start p95 spike |
| 13 | ECC error | 33 | Prefix cache effectiveness drops |
| 14 | Thermal throttling | 34 | Streaming completion falls |
| 15 | Power limit throttling | 35 | Client retry amplification |
| 16 | NCCL initialization failure | 36 | Tensor parallel imbalance |
| 17 | Host memory OOMKilled | 37 | Autoscaler cold capacity delay |
| 18 | CUDA OOM at startup | 38 | N-1 GPU headroom insufficient |
| 19 | CUDA OOM under load | 39 | GPU cost per token spike |
| 20 | Model artifact download failure | 40 | Metrics missing or stale |

## 11. Twelve acceptance gates

G1 approved scope/spend cap; G2 GPU node and plugin evidence; G3 driver/CUDA/runtime compatibility; G4 Pod resource allocation; G5 host vs CUDA OOM classification; G6 KV cache/model fit; G7 reproducible synthetic traffic mix; G8 TTFT/TPOT/stream completion SLO; G9 hardware and exporter health; G10 controlled mitigation/rollback; G11 N-1 capacity and FinOps validation; G12 sanitized evidence and reviewer sign-off.

Complete the [companion lab](../labs/06-nvidia-drivers-cuda-gpu-oom-vllm-performance-check.md). Publication does not establish live validation or vendor certification.

**Next:** Chapter 13.7 — Enterprise Troubleshooting and Incident Recovery Capstone.
