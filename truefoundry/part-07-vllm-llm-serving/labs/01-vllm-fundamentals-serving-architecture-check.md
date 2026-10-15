# Lab 7.1 — vLLM Architecture and GPU Readiness Check

**Canonical:** [Chapter 7.1](../01-vllm-fundamentals-serving-architecture/01-vllm-fundamentals-serving-architecture.md)  
**Default:** [SAFE-READ]. Any deployment, runtime change or load test is [LAB-MUTATE] only in approved isolated nonproduction TrueFoundry resources.

## 1. Objectives

Map vLLM request and model-loading stages, inspect GPU scheduling and CUDA visibility, identify prefill/decode and KV-cache telemetry, classify 16 failure modes, and document a safe rollback.

## 2. Preflight

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Record TrueFoundry version, workspace, namespace, model, immutable revision, vLLM image digest, GPU SKU and approved access scope. Do not disclose secrets or real prompts.

## 3. Architecture worksheet

| Layer | Observed component | Evidence | Owner |
|---|---|---|---|
| TrueFoundry desired state | | | |
| Kubernetes Deployment/Pod | | | |
| GPU node/device plugin | | | |
| CUDA/PyTorch/vLLM | | | |
| Model storage/tokenizer | | | |
| API gateway/TLS/auth | | | |
| Service/EndpointSlice | | | |
| Scheduler/prefill/decode | | | |
| Metrics/traces/logs | | | |

## 4. Kubernetes/GPU validation [SAFE-READ]

```bash
kubectl -n <namespace> describe pod <pod>
kubectl describe node <gpu-node>
kubectl -n <namespace> logs <pod> --tail=200
kubectl -n <namespace> logs <pod> --previous --tail=200
```

In an **already provisioned, authorized** runtime container if installed:

```bash
nvidia-smi
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available())"
python -c "import vllm; print(vllm.__version__)"
```

If tools are absent, record Not installed; do not modify production images.

## 5. Model and KV-cache budget

Record weight format/precision, model parameter count, layers, KV heads, head dimension, context limit, concurrency, VRAM total/free and runtime KV-cache metrics. Use the chapter's approximate formula as a planning check only; compare with measured memory and block allocation.

## 6. Request lifecycle and metrics

Use one approved synthetic request through the authorized endpoint. Record API contract, response model version, correctness and streaming completion.

| Signal | Value | Source |
|---|---|---|
| TTFT P95/P99 | | |
| Inter-token latency | | |
| Prompt tokens/s | | |
| Output tokens/s | | |
| Running requests | | |
| Waiting requests | | |
| KV cache use | | |
| GPU utilization/VRAM | | |
| Error/429/5xx rate | | |

Do not assume metrics exist or invent results. Mark missing series as an instrumentation gap.

## 7. Optional controlled nonproduction experiment [LAB-MUTATE]

1. Deploy a small approved model with a pinned image and model revision via supported TrueFoundry workflow.
2. Confirm GPU allocation and CUDA initialization.
3. Observe model load, startup probe, readiness and authorized golden response.
4. Send bounded synthetic short- and long-prompt requests at approved concurrency.
5. Compare TTFT, ITL, token throughput, queue and KV cache use.
6. Stop on instability, VRAM pressure or exceeding agreed limits.
7. Restore known-good release and remove only lab-owned resources.

## 8. Failure tabletop

| ID | Scenario | First failed transition |
|---|---|---|
| V1 | Pod Pending | Request → node |
| V2 | GPU resource missing | Plugin → allocatable |
| V3 | CUDA unavailable | Container → framework |
| V4 | Device inference fails | Runtime → GPU |
| V5 | Model storage 403 | Identity → object |
| V6 | Unsupported architecture | Model → engine |
| V7 | Tokenizer mismatch | Artifact → parser |
| V8 | Load-time VRAM OOM | Weights → VRAM |
| V9 | KV-cache OOM | Concurrency → VRAM |
| V10 | Readiness never succeeds | Engine → probe |
| V11 | TTFT high, GPU idle | Queue → execution |
| V12 | ITL high, GPU saturated | Decode → GPU |
| V13 | No ready endpoints | Pod → Service |
| V14 | Stream stops after 200 | Runtime → client |
| V15 | Wrong model version | Release → runtime |
| V16 | Reconciler reverts hotfix | Desired state → Pod |

For each record lowest proven healthy layer, first failed transition, evidence, owner, safe remediation, rollback and acceptance.

## 9. Acceptance scorecard

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| G1 Ownership/architecture | | |
| G2 Immutable release | | |
| G3 GPU/CUDA compatibility | | |
| G4 Model lifecycle/readiness | | |
| G5 KV cache/concurrency | | |
| G6 Authorized golden API | | |
| G7 TTFT/ITL/queue/VRAM | | |
| G8 Rollback/cleanup | | |

## 10. Evidence report

```text
Date/time / approval:
TrueFoundry workspace/cluster:
Image digest, vLLM version, model/tokenizer revision:
GPU SKU/driver/CUDA/runtime:
GPU allocation and VRAM:
Model load/warmup and readiness:
Context/concurrency/KV cache:
Golden synthetic request and endpoint:
TTFT/ITL/throughput/queue/errors:
V1–V16 classification:
Lowest Proven Healthy Layer:
First Failed Transition:
Known-good rollback and owner:
Post-recovery checks and cleanup:
Unexecuted gates:
```

**Next:** Chapter 7.2 — vLLM Deployment with TrueFoundry and Kubernetes.
