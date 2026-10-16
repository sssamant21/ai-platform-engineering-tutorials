# Chapter 011 — Hands-On Lab: GPU Architecture, SMs, HBM and Memory Bandwidth

**Part 03 | Chapter 011 | Published plan; execution pending**

## Goal

Inspect approved GPU inventory, map SM/HBM/VRAM and topology, calculate a synthetic roofline and KV-cache budget, and prepare an SRE diagnostic and FinOps evidence pack.

## Safety

**[SAFE-READ]** authorized Kubernetes/GPU inventory and dashboards. **[LAB-MUTATE]** separately approved staging synthetic benchmark only. **[CHANGE-CONTROL]** production GPU drivers, device plugins, node pools, MIG, runtime and scheduling changes. Never install privileged software or run stress tools on production as part of this lab.

## Lab 0 — Preflight

Record UTC, approved cluster/context, namespace, node, GPU SKU/UUID, NVIDIA driver, CUDA runtime compatibility, model/runtime image, monitoring exporter, owner, reviewer and cost ceiling.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

Stop if context or permissions are not verified.

## Lab 1 — GPU device inventory [SAFE-READ]

Run on an approved GPU node or authorized troubleshooting container:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu,temperature.gpu,power.draw,clocks.sm --format=csv
nvidia-smi topo -m
```

If commands are unavailable, record **blocked**; do not add privileges. Compare device inventory with Kubernetes allocatable extended resources and workload requests.

## Lab 2 — Architecture diagram [DESIGN]

Map SMs, warps, tensor cores, registers, L1/shared memory, L2, HBM, host RAM, PCIe/NVLink and model weights/KV/workspace. Annotate which resources constrain prefill versus decode for your assumed workload.

## Lab 3 — Roofline arithmetic [DESIGN]

Synthetic sustainable HBM bandwidth = 2.0 TB/s; arithmetic intensity = 20 operations/byte; relevant peak compute = 100 TOPS:

```text
bandwidth roof = 2.0e12 × 20 = 40e12 operations/s = 40 TOPS
predicted roof = min(100, 40) = 40 TOPS
```

State why this is not an observed benchmark or a GPU specification.

## Lab 4 — KV-cache and VRAM [DESIGN]

Assume 32 layers, 8 KV heads, dimension 128, BF16 (2 bytes/element):

```text
KV bytes/token = 2 × 32 × 8 × 128 × 2 = 131,072 bytes
8,192 resident tokens ≈ 1 GiB logical KV state
```

Build a worksheet for weights, KV, workspace, runtime buffers, fragmentation and safety headroom. Identify sharding, quantization, sliding window and allocator caveats.

## Lab 5 — Performance diagnosis [SAFE-READ]

Correlate approved runtime TTFT, TPOT, tokens/s, queue, KV-cache occupancy and preemptions with GPU utilization, HBM traffic, clocks, power and temperature. Do not infer bottlenecks from GPU utilization alone. Document missing signals and exporter freshness.

```bash
kubectl describe nodes
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/MODEL_DEPLOYMENT --since=15m --tail=100
```

Sanitize logs and do not collect prompts or credentials.

## Lab 6 — Capacity and cost worksheet [DESIGN]

Record GPU SKU, allocated GPU-hours, average and peak concurrency, valid completed requests, actual generated tokens, p95 TTFT/TPOT, idle capacity, retry waste, projected GPU-hours per million successful output tokens, and rollback capacity. Leave unknown values blank or marked unavailable.

## Lab 7 — Eight tabletop game days

01 GPU allocatable missing; 02 driver mismatch; 03 KV OOM; 04 decode bandwidth; 05 thermal throttle; 06 topology regression; 07 stale DCGM; 08 idle GPU cost. Default **tabletop**.

## Twenty-scenario checklist

F01 no GPU; F02 pending Pod; F03 CUDA; F04 wrong SKU; F05 weight OOM; F06 KV; F07 fragmentation; F08 bandwidth; F09 compute; F10 poor throughput; F11 idle GPU/queue; F12 thermal; F13 power; F14 ECC/XID; F15 PCIe; F16 NVLink; F17 collectives; F18 noisy neighbor; F19 telemetry; F20 FinOps.

## Sixteen acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | GPU architecture map | Pending |
| G02 | SM/tensor core explanation | Pending |
| G03 | HBM hierarchy | Pending |
| G04 | Roofline calculation | Pending |
| G05 | Prefill/decode diagnosis | Pending |
| G06 | KV-cache math | Pending |
| G07 | VRAM budget | Pending |
| G08 | GPU topology | Pending |
| G09 | Approved read-only inventory | Pending |
| G10 | Hardware health metrics | Pending |
| G11 | Kubernetes GPU allocation | Pending |
| G12 | Incident triage | Pending |
| G13 | Staging test plan | Pending |
| G14 | FinOps estimate | Pending |
| G15 | Rollback/recovery | Pending |
| G16 | Reviewer signoff | Pending |

## Evidence template

```text
UTC / approval / context / namespace:
Node / GPU SKU / UUID / driver:
CUDA runtime and serving image:
SM/HBM/topology:
Roofline assumptions and calculations:
Weights/KV/workspace/VRAM:
TTFT/TPOT/tokens/s and GPU telemetry:
Game-day mode and findings:
Cost, mitigation, reviewer and gate status:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop any separately approved staging benchmark, verify no GPU resources were left running and reconcile costs. All gates remain pending until independently reviewed.

**Next: Chapter 012 — CUDA Runtime, Drivers, Containers and Compatibility.**
