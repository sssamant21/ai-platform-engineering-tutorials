# Chapter 015 — Hands-On Lab: GPU Diagnostics, DCGM and Failure Acceptance

**Part 03 final capstone | Chapter 015 | Written lab published; not executed**

## Goal

Integrate GPU architecture, CUDA compatibility, VRAM/quantization, multi-GPU topology and DCGM incident diagnostics. Produce a reviewer-ready GPU SRE evidence pack and 100-point acceptance assessment.

## Safety

**[SAFE-READ]** authorized GPU/Kubernetes metrics and redacted logs. **[LAB-MUTATE]** separately approved staging synthetic inference or fault simulation. **[CHANGE-CONTROL]** production driver, node, GPU reset, MIG, power, network, model or scheduler changes. No PHI, customer prompts, secrets or privileged debug installation.

## Lab 0 — Preflight

Record UTC, cluster/context, namespace, GPU SKU/UUID, driver, CUDA/framework, device plugin, model/runtime/image digest, DCGM exporter version, approved window, cost ceiling, incident commander, rollback owner and reviewer.

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,daemonsets -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

Stop if scope or permissions are unverified.

## Lab 1 — GPU inventory and topology [SAFE-READ]

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi -q
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,driver_version,memory.total,memory.used,utilization.gpu,temperature.gpu,power.draw,clocks.sm --format=csv
```

Capture GPU UUID mapping, topology, driver, temperature/power, clocks and memory. Missing commands are **blocked**, not a reason to escalate privileges.

## Lab 2 — DCGM freshness [SAFE-READ]

Locate the actual DCGM exporter through approved cluster inventory. Record exporter version, scrape target, last sample timestamp, GPU UUID labels and alert ownership. Do not assume metric names; inspect configured exporter fields. Demonstrate why missing series must not be counted as healthy zero errors.

## Lab 3 — CUDA/driver compatibility [DESIGN]

Build a matrix of host kernel, NVIDIA driver, CUDA driver API, container CUDA runtime, cuDNN/cuBLAS/NCCL, GPU compute capability, serving framework, device plugin, toolkit and immutable image digest. Cite the applicable vendor compatibility matrix in the evidence record.

## Lab 4 — VRAM and quantization [DESIGN]

For synthetic 32 layers, 8 KV heads, 128 head dimension and BF16:

```text
KV/token = 2 × 32 × 8 × 128 × 2 = 131,072 bytes
8,192 resident tokens ≈ 1 GiB logical KV
```

For 7 billion ideal dense weights: BF16 14 GB; INT8 7 GB; 4-bit 3.5 GB before metadata and runtime overhead. Document accuracy/JSON/tool-calling evaluation requirements, fragmentation, workspace and headroom.

## Lab 5 — Multi-GPU and NCCL [DESIGN]

Draw TP/PP/DP/EP mapping, GPU/NIC topology, per-rank memory and failure domains. Explain how one rank OOM or a slow collective can stall the group. Record approved read-only rank/transport evidence if available.

## Lab 6 — SLO burn and customer path [DESIGN]

Synthetic 99.9% success SLO, 50,000 eligible requests and 250 failures:

```text
allowed failures = 50
observed failure rate = 250/50,000 = 0.5%
success = 99.5%
burn = 0.5%/0.1% = 5×
```

Classify HTTP 200 with incomplete SSE as failed customer requests per the agreed contract. Keep synthetic and actual metrics separate.

## Lab 7 — Ten tabletop game days

01 stale DCGM; 02 missing GPU allocatable; 03 XID after rollout; 04 ECC and incomplete SSE; 05 thermal throttle; 06 KV OOM; 07 fragmentation; 08 NCCL rank loss; 09 utilization/throughput mismatch; 10 partial HTTP 200 recovery. Default **tabletop**.

For each: impact, evidence, hypothesis, owner, escalation, authorized mitigation, rollback and sustained customer-path recovery.

## Twenty-four failure checklist

F01 exporter; F02 freshness; F03 labels; F04 counter reset; F05 device plugin; F06 allocatable; F07 CUDA init; F08 driver; F09 XID; F10 uncorrected ECC; F11 corrected ECC; F12 thermal; F13 power; F14 load OOM; F15 prefill OOM; F16 KV OOM; F17 fragmentation; F18 preemption; F19 utilization; F20 queue; F21 NCCL; F22 rank memory; F23 GPU reset; F24 partial SSE.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope approval | Pending |
| G02 | Correct context | Pending |
| G03 | Sanitized evidence | Pending |
| G04 | GPU architecture | Pending |
| G05 | Memory hierarchy | Pending |
| G06 | Driver/CUDA compatibility | Pending |
| G07 | Device plugin/toolkit | Pending |
| G08 | VRAM budget | Pending |
| G09 | Quantization quality | Pending |
| G10 | TP/PP/DP/EP | Pending |
| G11 | Topology/NCCL | Pending |
| G12 | DCGM freshness | Pending |
| G13 | XID/ECC triage | Pending |
| G14 | Thermal/power | Pending |
| G15 | CUDA OOM | Pending |
| G16 | TTFT/TPOT/SSE | Pending |
| G17 | SLO burn | Pending |
| G18 | GPU FinOps | Pending |
| G19 | Rollback/RCA | Pending |
| G20 | Independent signoff | Pending |

## 100-point assessment

| Category | Max | Awarded |
|---|---:|---|
| Safety/privacy | 10 | Pending |
| GPU architecture/memory | 10 | Pending |
| CUDA compatibility | 15 | Pending |
| VRAM/quantization | 15 | Pending |
| Multi-GPU/NCCL | 15 | Pending |
| DCGM/diagnostics | 15 | Pending |
| Reliability/recovery | 10 | Pending |
| FinOps/evidence/review | 10 | Pending |
| **Total** | **100** | **Not assessed** |

Suggested passing threshold: ≥85, no critical safety failure, correct diagnostic evidence, approved rollback and reviewer signoff.

## Evidence record

```text
UTC/context/authorization:
GPU SKU/UUID, topology and owner:
Kernel/driver/CUDA/runtime:
Device plugin/DCGM/exporter:
VRAM/quantization/KV assumptions:
NCCL/rank and fabric:
TTFT/TPOT/complete SSE and error budget:
XID/ECC/thermal/power findings:
Game days tabletop/executed/blocked:
GPU cost, incident RCA/COA, rollback:
Reviewer, score and gate status:
```

## Cleanup and closeout

Read-only/tabletop work creates no resources. Stop separately approved staging tests, reconcile GPU-hours and remove only owned resources. Keep all acceptance gates pending until actual evidence and independent review exist.

**Next: Chapter 016 — vLLM Architecture and Production Deployment.**
