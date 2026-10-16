# Chapter 13.6 — Hands-on Lab: NVIDIA Drivers, CUDA, GPU OOM and vLLM Performance

**Part 13 | Approved staging | Live execution pending**

## Goal and safety

Diagnose GPU device allocation, driver/CUDA compatibility, host vs device OOM, KV cache, model loading and vLLM TTFT/TPOT performance using synthetic inference. **[SAFE-READ]** approved diagnostics. **[LAB-MUTATE]** bounded staging only. **[CHANGE-CONTROL]** production GPU resets, driver upgrades, model parameters and traffic changes. Do not run uncontrolled stress tests, log real prompts or drain shared GPU nodes.

## Lab 0 — Inventory and baseline

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl -n MODEL_NAMESPACE get pods -o wide
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE get events --sort-by=.lastTimestamp
```

Record TrueFoundry/model/image digests, GPU SKU/MIG, driver/CUDA/PyTorch/vLLM versions, Pod GPU requests, node allocatable, cold-start p95, synthetic workload token distributions, concurrency, TTFT/TPOT, streaming completion and approved GPU-hour budget.

## Lab 1 — GPU device and driver checks

From an authorized GPU diagnostic environment (if present):

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,driver_version,memory.total,memory.used,utilization.gpu --format=csv
```

Confirm device plugin allocatable resources, Pod device visibility and version compatibility. Record unsupported fields and absence of `nvidia-smi` rather than assuming GPU failure.

## Lab 2 — OOM classification

Using sanitized pre-existing evidence or a tightly bounded disposable staging workload, distinguish cgroup OOMKilled from CUDA OOM and KV cache exhaustion. Capture Pod last termination reason, model engine exception, GPU memory, concurrency and context length. Do not deliberately exhaust a shared GPU.

## Lab 3 — vLLM latency breakdown

Run a bounded synthetic request matrix with short/long input tokens, short/long output tokens and approved concurrency levels. Record queue, TTFT, TPOT, completion, throughput, GPU memory, cost and model revision. Compare same traffic mix before/after any tuning.

## Lab 4 — Six game days

1. Device plugin absent or GPU not injected (tabletop if shared).
2. Driver/CUDA version incompatibility (isolated image test).
3. CUDA OOM vs host OOM evidence classification.
4. KV cache pressure and long-context synthetic workload.
5. TTFT/TPOT saturation and queue mitigation.
6. Bad model tuning change with rollback and N-1 capacity check.

Record UTC timeline, actual/tabletop status, blast radius, expected/observed, SLO, cost, mitigation and reviewer.

## Forty scenarios

01 missing GPU; 02 device plugin; 03 allocatable zero; 04 wrong SKU; 05 MIG; 06 time-slicing; 07 driver; 08 container CUDA; 09 architecture; 10 device ordinal; 11 injection; 12 XID; 13 ECC; 14 thermal; 15 power; 16 NCCL; 17 host OOM; 18 startup CUDA OOM; 19 load CUDA OOM; 20 model artifact; 21 weights fit; 22 context length; 23 KV cache; 24 fragmentation; 25 concurrency; 26 batching; 27 input mix; 28 output mix; 29 TTFT; 30 TPOT; 31 queue; 32 cold start; 33 prefix cache; 34 streaming; 35 retries; 36 tensor parallel; 37 cold autoscale; 38 N-1; 39 cost/token; 40 missing metrics.

## Twelve acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G1 | Approved scope and spend cap | Pending |
| G2 | GPU node/device plugin verified | Pending |
| G3 | Driver/CUDA/PyTorch/vLLM compatibility checked | Pending |
| G4 | Pod GPU resource allocation reconciled | Pending |
| G5 | Host OOM vs CUDA OOM distinguished | Pending |
| G6 | KV cache and model fit assessed | Pending |
| G7 | Reproducible synthetic traffic matrix | Pending |
| G8 | TTFT/TPOT and streaming completion measured | Pending |
| G9 | Hardware and DCGM/exporter signals checked | Pending |
| G10 | Safe tuning/rollback documented | Pending |
| G11 | N-1 capacity and cost verified | Pending |
| G12 | Sanitized evidence and reviewer sign-off | Pending |

## Evidence record

```text
Exercise / approval / UTC start-end:
TrueFoundry revision / image / model digest:
GPU SKU, MIG, driver, CUDA, PyTorch, vLLM:
Pod GPU request/limit and node allocatable:
Host vs CUDA OOM / device plugin findings:
Synthetic input/output token mix and concurrency:
Queue, TTFT, TPOT, completion, GPU memory:
Baseline vs changed revision / cost:
Hypothesis, expected/actual, mitigation, rollback:
Executed or tabletop:
Owner, reviewer, open risks:
```

## Cleanup

Restore disposable staging model parameters, delete only approved workloads, confirm normal GPU allocation, inference SLO and budget, and close test alerts. Live execution and acceptance are pending until independently verified.

**Next:** Chapter 13.7 — Enterprise Troubleshooting and Incident Recovery Capstone.
