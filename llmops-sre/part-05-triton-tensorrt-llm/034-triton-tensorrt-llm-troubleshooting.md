# Chapter 034 — Triton and TensorRT-LLM Troubleshooting

**Part 05 — NVIDIA Triton and TensorRT-LLM | Chapter 034 of 100**

> **Status:** Written tutorial published. Live execution, technical/security review and operational acceptance remain pending. All commands require an authorized environment; backend flags, metrics and APIs depend on pinned versions.

## Objectives

Develop a production incident workflow for Triton and TensorRT-LLM spanning Kubernetes, NVIDIA GPUs, CUDA, TensorRT engines, model repositories, scheduler queues, dynamic batching, KV cache, distributed ranks, streaming, SLOs, security and GPU FinOps. Collect minimally invasive evidence, identify root causes, recover safely and document RCA.

## 1. Failure-domain map

```text
Client / SDK
   │ auth, deadlines, request/response contract
Gateway / Load Balancer
   │ TLS, rate limits, retries, routing
Kubernetes Service / EndpointSlice
   │ ready endpoints, CNI, DNS, NetworkPolicy
Triton Server
   │ process health, model lifecycle, scheduler
TensorRT-LLM Backend / Executor
   │ engine compatibility, tokens, streaming, KV
CUDA / NCCL / GPU
   │ VRAM, driver, kernels, topology, hardware
Node / Storage / Registry
   │ device plugin, artifact provenance, I/O
Observability / Change Control
   │ metrics, traces, SLO, rollback, audit
```

Classify the incident by the first failing layer; do not assume a GPU is at fault because inference runs on a GPU.

## 2. Incident severity and initial response

For high-severity incidents:
1. establish incident commander, technical lead and communications owner;
2. freeze nonessential model/runtime changes;
3. identify affected model/version, tenant cohorts, regions and request protocols;
4. capture UTC onset, deploy events, error rates, valid completion, TTFT/TPOT and GPU faults;
5. decide whether to shed load, disable a broken route or roll back an immutable release;
6. preserve sanitized evidence and timestamps;
7. monitor recovery against client-perceived SLOs.

Do not restart all replicas simultaneously or force-unload production models without change control.

## 3. Read-only triage bundle

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

For approved namespace/deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment APPROVED_DEPLOYMENT
kubectl -n MODEL_NAMESPACE get pods -o wide
kubectl -n MODEL_NAMESPACE get endpointslices
kubectl -n MODEL_NAMESPACE logs deployment/APPROVED_DEPLOYMENT --since=20m --tail=200
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu,temperature.gpu --format=csv
```

Collect only approved sanitized logs. Do not print Kubernetes Secrets, customer prompts, completions, raw tensors or access tokens.

## 4. Symptom-first decision tree

```text
Inference incident?
 ├─ Pod Pending → GPU allocatable, taints, quota, PVC
 ├─ ImagePullBackOff → registry auth/digest
 ├─ CrashLoopBackOff → CUDA/driver, security, model load
 ├─ server healthy but model unready → repository/config/engine
 ├─ 4xx → auth, schema, admission, limits
 ├─ 5xx → backend, engine, resource pressure
 ├─ gateway timeout → queue, prefill, decode, retries
 ├─ TTFT high → gateway/queue/prefill
 ├─ TPOT high → decode/KV/NCCL
 ├─ partial stream → terminal response/cancel path
 ├─ GPU OOM → weights/workspace/KV/instances
 ├─ wrong outputs → tokenizer/quantization/version
 ├─ distributed hang → rank/NCCL/topology
 ├─ metrics missing → exporter/scrape/labels
 └─ cost spike → GPU-hours per valid completion
```

## 5. Triton readiness and model lifecycle

Triton process liveness, server readiness and model/version readiness are different signals. Confirm the exact model and version via approved management or health APIs, then run a synthetic golden inference through the same gateway used by clients. A successful health probe does not prove tokenization, streaming or quality.

Review model repository revision, `config.pbtxt`, backend availability, model-control mode, load/unload events and artifact hashes. Do not assume a mutable model repository rolls back with a Kubernetes Deployment.

## 6. Engine and backend initialization

Engine deserialization and initialization failures can arise from incompatible GPU architecture, TensorRT/TensorRT-LLM/CUDA/driver versions, plugins, quantization artifacts, max context/batch profiles or corrupted artifacts. Capture build/runtime version matrix and engine checksum before attempting a rebuild.

Rebuilds can be expensive and should occur in approved isolated staging, not ad hoc on production GPU nodes.

## 7. GPU OOM: classify the phase

| Phase | Likely pressure | Evidence |
|---|---|---|
| build/compilation | engine build workspace | build log and GPU VRAM |
| startup | weights, plugins, graphs, instances | initialization log |
| prefill | prompt activations and attention | long-prompt cohort |
| decode | KV growth, concurrency, allocator | active token/KV metrics |
| rollout | overlapping replicas/models | Pod placement and VRAM |

Check per-GPU memory, not total cluster memory. Recover by reverting the triggering release or reducing approved concurrency/context/instance settings through change control. Preserve headroom and validate output correctness.

## 8. Latency and throughput investigation

Define and compare:
- gateway E2E and terminal completion;
- TTFT, TPOT/ITL with explicit client timestamp definitions;
- Triton queue and compute duration;
- request rate, backend execution rate and batch efficiency;
- GPU VRAM/utilization/power and throttling;
- prompt/output length cohorts and concurrency;
- valid completion rate and GPU-hours.

Cumulative microsecond duration counters cannot directly provide p99. Use verified histograms, traces or client instrumentation.

## 9. Scheduler and dynamic batching

Queue p99 regression may result from max queue delay, priority starvation, insufficient instances, shape incompatibility or overload. Compare baseline and current model config, input distribution and request rate. A batch efficiency increase can coincide with a tail-latency regression.

Check queue timeouts and load shedding; uncontrolled retries can worsen saturation.

## 10. KV-cache and decode failures

Estimate logical KV per token:

```text
KV bytes/token ≈ 2 × layers × KV_heads × head_dim × bytes/element
```

Compare runtime block allocator metrics, resident token count, context caps, cancellation cleanup, prefix reuse and per-rank GPU memory. An apparently free GPU may still suffer allocator fragmentation or reserved workspace constraints.

Prefix-cache reuse across tenants requires explicit privacy review.

## 11. Distributed TP/PP and NCCL

Investigate rank count, Pod placement, GPU UUID/topology, network, driver/runtime versions and rank-local logs. A single failed or slow rank can stall an entire request. Do not indiscriminately restart a multi-rank job without preserving evidence and understanding quorum/coordination semantics.

## 12. Streaming correctness

A successful HTTP/gRPC status is not necessarily a complete LLM response. Verify:
- first token and subsequent token timings;
- terminal completion marker/final flag according to protocol;
- cancellation propagation;
- output token count and stop/EOS;
- timeout classification;
- no duplicated or out-of-order chunks under retry.

Use approved synthetic fixtures; avoid raw production prompt capture.

## 13. Security incident crossover

Treat unexpectedly exposed model-control endpoints, metrics, artifact registries, calibration data or cross-tenant cache reuse as security incidents. Restrict access through approved controls, preserve audit logs and escalate to security owners. Do not attempt unauthorized tenant-crossing tests.

## 14. Rollback decision framework

```text
Regression detected
 → isolate blast radius and changed components
 → compare immutable release manifest
 → select tested known-good bundle
 → roll back gateway + Triton + backend + engine
    + model/config + tokenizer/quantization as needed
 → verify model readiness and synthetic golden fixtures
 → validate complete streaming, TTFT/TPOT, error/SLO
 → confirm GPU VRAM, cost and telemetry
 → close incident only after stable observation
```

A Kubernetes rollout undo alone may leave incompatible external engine/model artifacts. Roll back the entire dependency unit.

## 15. Evidence and RCA template

```text
Incident ID / severity / UTC timeline:
Affected model/version/region/tenant cohorts:
Symptoms and client SLO impact:
Last known good release and change timeline:
Gateway/Triton/backend/GPU evidence:
TTFT/TPOT/queue/compute:
Engine/model/tokenizer checksums:
GPU memory/NCCL/rank topology:
Hypotheses and eliminated causes:
Mitigation/rollback and verification:
Root cause and contributing factors:
Corrective actions / owners / due dates:
Privacy/security review:
GPU-hours and cost impact:
```

## 16. Twenty-four production failure scenarios

| ID | Failure | Primary evidence |
|---|---|---|
| F01 | GPU device plugin unavailable | node allocatable |
| F02 | Pod Pending due to quota/taint | events |
| F03 | image digest pull failure | pull events |
| F04 | Triton process crash | container logs |
| F05 | server ready but model unready | model readiness |
| F06 | repository revision drift | artifact digest |
| F07 | TensorRT engine incompatible | runtime matrix |
| F08 | quantization scale mismatch | quality fixtures |
| F09 | tokenizer/chat-template drift | golden tests |
| F10 | startup GPU OOM | initialization VRAM |
| F11 | decode KV exhaustion | token/KV metrics |
| F12 | allocator fragmentation | cache blocks |
| F13 | scheduler queue saturation | queue p99 |
| F14 | dynamic batching tail regression | cohort p99 |
| F15 | priority starvation | per-class SLO |
| F16 | gateway retry storm | retry rate |
| F17 | incomplete decoupled response | terminal validator |
| F18 | canceled request leaks KV | active slots |
| F19 | TP/PP rank mismatch | rank inventory |
| F20 | NCCL timeout | rank/network logs |
| F21 | GPU hardware/XID fault | DCGM/device |
| F22 | telemetry outage | scrape freshness |
| F23 | admin API exposed | security audit |
| F24 | GPU-hours per valid completion spikes | FinOps |

## 17. Sixteen operational runbooks

1. Establish incident command and evidence timeline.
2. Diagnose GPU scheduling/device-plugin failures.
3. Diagnose image pull, storage and model repository.
4. Recover Triton process startup and readiness.
5. Diagnose engine/GPU/software compatibility.
6. Diagnose tokenizer/quantization output regressions.
7. Recover startup GPU OOM.
8. Recover decode KV exhaustion/fragmentation.
9. Diagnose queue saturation and dynamic batching.
10. Diagnose gateway timeout/retry storms.
11. Recover incomplete streaming/cancellation.
12. Diagnose TP/PP and NCCL rank failures.
13. Investigate GPU hardware faults.
14. Restore observability and SLO alerting.
15. Contain administrative exposure and FinOps spikes.
16. Execute full-stack rollback and RCA.

## 18. Eight tabletop game days

1. GPU device plugin fails during a model rollout.
2. Triton reports healthy while a model remains unready.
3. New engine fails on target GPU architecture.
4. Long-context traffic exhausts KV and causes retry storms.
5. Batch optimization improves throughput but breaks p99.
6. One tensor-parallel rank hangs and streaming stalls.
7. Model-control API becomes accessible from untrusted network.
8. Rollback restores Pods but not tokenizer/engine compatibility.

Tabletop by default. Live fault injection requires separately approved isolated staging.

## 19. Twenty acceptance gates

G01 incident severity/ownership; G02 UTC evidence; G03 GPU/Pod triage; G04 image/model provenance; G05 process/model readiness; G06 engine compatibility; G07 tokenizer/quality; G08 startup OOM; G09 KV/allocator; G10 scheduler queue; G11 TTFT/TPOT/E2E; G12 complete streaming; G13 TP/PP/NCCL; G14 GPU hardware; G15 telemetry freshness; G16 security/privacy; G17 SLO/error budget; G18 full-stack rollback; G19 GPU FinOps; G20 independent review.

See [companion lab](../labs/034-triton-tensorrt-llm-troubleshooting-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 035 — NVIDIA Inference Stack Acceptance Lab.**
