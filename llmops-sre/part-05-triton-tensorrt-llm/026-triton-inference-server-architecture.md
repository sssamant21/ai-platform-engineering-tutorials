# Chapter 026 — NVIDIA Triton Inference Server Architecture

**Part 05 — NVIDIA Triton and TensorRT-LLM | Chapter 026 of 100**

> **Publication status:** Tutorial published; technical/security review, live labs and operational acceptance pending. Pin Triton and backend versions before applying commands; features and endpoints differ across releases.

## Objectives

Understand Triton's client-to-backend request lifecycle, model repository, scheduling and batching, CPU/GPU instance placement, HTTP/gRPC, health/readiness, observability and safe Kubernetes operations. Differentiate Triton from vLLM and TensorRT-LLM, and build a version-aware production deployment design.

## 1. What Triton does

NVIDIA Triton Inference Server is a model-serving platform that accepts inference requests, routes them to loaded models, schedules execution and invokes compatible backends. It supports multiple model formats/backends depending on the installed build and configuration. **TensorRT-LLM** is a distinct LLM inference runtime/backend integration; **vLLM** is another serving/runtime stack. They are not interchangeable and do not share universal flags, tokenization behavior or streaming contracts.

```text
Client / SDK / Gateway
       │ TLS, identity, quota, routing
       ▼
Triton HTTP / gRPC front end
       │
       ├── Health and readiness
       ├── Model control and repository management
       ├── Metrics / tracing / logging
       └── Per-model scheduler
                 │
        ┌────────┼─────────┐
        ▼        ▼         ▼
    Backend A Backend B Backend C
    TensorRT  Python   TensorRT-LLM
        │        │         │
     Instance groups / CPU / GPU resources
        │
    Model artifacts and runtime dependencies
```

The diagram is conceptual; backend availability, scheduler capabilities and instance placement depend on the pinned image and model configuration.

## 2. Architecture components

| Component | Responsibility | Failure domain |
|---|---|---|
| Client/gateway | identity, TLS, request limits, retry and trace context | 401/403/429/504 |
| HTTP/gRPC endpoint | protocol, payload decoding and routing | connection, schema, timeout |
| Model repository | versioned model/config artifacts | missing or corrupt model |
| Model lifecycle manager | loading, unloading and readiness | load error, cold start |
| Scheduler | request queueing and supported batching | high queue/latency |
| Backend | model-specific inference execution | backend initialization/runtime |
| Instance group | concurrency and device placement | GPU OOM/contention |
| Metrics/tracing | health, queue, compute and device visibility | stale telemetry |
| GPU/device stack | CUDA, drivers, device plugin | GPU fault, incompatibility |

## 3. Request lifecycle

1. The client sends a request through an approved authenticated gateway.
2. Triton validates the protocol request, model name/version and tensor metadata.
3. The model scheduler queues work, optionally forming batches according to model/backend capabilities.
4. An available model instance invokes its backend.
5. The backend returns a response or, for supported decoupled/streaming models, one or more responses.
6. The gateway/client validates completion and accounts for latency, tokens/requests and errors.

Do not assume all backends implement streaming, cancellation, dynamic batching or decoupled responses identically. Standard inference tensor APIs are not automatically OpenAI-compatible chat APIs.

## 4. Protocols and API contracts

Triton commonly exposes HTTP/REST and gRPC inference APIs, plus health, model metadata/configuration and metrics interfaces. Typical ports in common deployments are **8000 HTTP**, **8001 gRPC** and **8002 metrics**, but they are configurable and must not be exposed directly to untrusted clients.

Illustrative read-only checks against an **authorized localhost port-forward**:

```bash
curl --fail --silent --show-error http://127.0.0.1:18000/v2/health/live
curl --fail --silent --show-error http://127.0.0.1:18000/v2/health/ready
```

These checks validate server health/readiness endpoints, not necessarily the readiness of a specific model or the correctness of an inference response. Confirm the pinned release and access policy.

## 5. Model repository layout

```text
model_repository/
  example_model/
    config.pbtxt
    1/
      model.onnx
    2/
      model.onnx
```

This is an illustrative ONNX model repository; required file names, config fields and directory layout differ by backend. Model versions and repository contents should be immutable or governed by explicit model-control changes. Validate checksum, provenance, license, storage permissions and model config before load.

## 6. Model control modes

Triton deployments may use model-control modes such as **NONE**, **POLL** and **EXPLICIT** depending on version and configuration:
- NONE: initial repository load with no automatic subsequent repository polling.
- POLL: monitor repository changes; avoid assuming atomic safety on shared or remote storage.
- EXPLICIT: load/unload models through authorized model-control operations.

Confirm precise behavior and security implications in the pinned version. Model load/unload APIs are administrative surfaces; do not expose them to arbitrary tenants.

## 7. Scheduling and batching

A scheduler can reduce per-request overhead by batching compatible requests, but added queue delay can harm TTFT and latency-sensitive SLOs. Supported scheduler options depend on backend and model. Dynamic batching is common for compatible stateless models; sequence batching is designed for models requiring sequence state; LLM decoupled/iterative execution requires backend-specific contracts.

Record queue duration, compute input/infer/output, batch sizes, instance utilization and tail latency. Do not claim a model benefits from batching without cohort-specific benchmarks.

## 8. Instance groups and GPU placement

Triton model configuration can assign model instances to CPU or GPU resources as supported. More instances may increase throughput but replicate weights, workspaces or KV allocations and cause GPU OOM. Validate per-GPU peak VRAM and device UUID, not only aggregate GPU capacity.

In Kubernetes, verify GPU extended resources, device plugin, node affinity, quotas, Pod security, readiness/startup and model artifact access. One Pod does not span multiple nodes; distributed TensorRT-LLM backends require their own supported orchestration and topology.

## 9. Backend architecture and isolation

Backends are runtime plugins/process integrations with varying execution and memory models. Python backend environments and custom backends introduce dependency and code-execution risk. Only install approved, pinned backends and model artifacts. Review backend-specific config, shared memory behavior and CUDA requirements. A backend error can make one model unavailable while server health still reports success.

## 10. Kubernetes deployment design

Production design requirements:
1. approved Triton container digest and model artifact provenance;
2. restricted Pod identity, permissions, volume mounts and egress;
3. GPU scheduling, model cold-start budget and sufficient memory headroom;
4. internal ClusterIP service and authenticated TLS gateway;
5. readiness reflecting intended model availability, not only process liveness;
6. separate metrics access and bounded cardinality;
7. rolling/canary plan accounting for GPU surge capacity;
8. immutable rollback and controlled model loading;
9. quotas, cost ceilings and on-call ownership.

**Never deploy a publicly reachable unauthenticated Triton API or administrative model-control interface.**

## 11. Health and readiness semantics

Distinguish:
- process liveness;
- server readiness;
- model readiness for a specific name/version;
- synthetic inference correctness;
- complete-stream success where applicable.

A ready server with an unloaded or failed model is not an accepted serving deployment. Gate traffic on the intended model contract and representative synthetic inference.

## 12. Metrics, tracing and logging

Triton provides version-dependent metrics and tracing options. Inventory actual metric names and labels for inference requests, queue time, compute input/infer/output, GPU memory/utilization and backend failures. GPU telemetry availability depends on device and exporter configuration. Trace spans can help identify scheduler versus backend latency but may not capture every GPU kernel.

Do not place raw inputs, model outputs, bearer tokens or sensitive tensors into logs or trace attributes. Verify scrape freshness and use low-cardinality model/version labels.

## 13. Triton versus vLLM versus TensorRT-LLM

| Dimension | Triton | vLLM | TensorRT-LLM |
|---|---|---|---|
| Primary role | multi-backend inference server | LLM serving/runtime | optimized LLM inference runtime |
| Model support | depends on installed backend | supported LLM architectures | supported LLM architectures/engines |
| API | KServe-style HTTP/gRPC and extensions | commonly OpenAI-compatible endpoints | integration-dependent |
| Scheduling | model/backend-specific | LLM token-aware scheduler | runtime/backend-specific |
| Model artifacts | versioned repository | weights/tokenizer/runtime config | engines/checkpoints/config |
| Production tradeoff | multi-model flexibility | LLM-focused serving | optimized engine lifecycle |

Evaluate compatibility, end-to-end latency, cost, security and operational burden using the exact model, GPU, version and request shape. Do not assume universal performance rankings.

## 14. Safe-read commands

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

For a verified namespace:

```bash
kubectl -n MODEL_NAMESPACE describe deployment TRITON_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/TRITON_DEPLOYMENT --since=15m --tail=100
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Redact sensitive logs and do not dump Secrets.

## 15. Operational decision tree

```text
Triton inference degraded?
 ├─ endpoint unreachable → gateway, Service, Pod, DNS
 ├─ live but not ready → startup/backend/repository
 ├─ model not ready → config, artifact, version, load
 ├─ 4xx schema error → tensor shape/type, model contract
 ├─ high queue → scheduler, instances, admission
 ├─ high compute → backend/GPU/model
 ├─ GPU OOM → instances, weights, workspace, KV
 ├─ gRPC/HTTP mismatch → client/protocol
 ├─ streaming incomplete → backend decoupled contract
 ├─ metrics stale → exporter/scrape/label drift
 ├─ admin API exposed → security containment
 └─ GPU cost high → idle replicas/inefficient batching
```

## 16. Twenty-four production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | model repository missing | artifact inventory |
| F02 | invalid config.pbtxt | model load logs |
| F03 | wrong model version | metadata |
| F04 | backend unavailable | backend inventory |
| F05 | incompatible CUDA/driver | initialization |
| F06 | model artifact checksum drift | provenance |
| F07 | model-control mode mismatch | startup flags |
| F08 | unauthorized model load/unload | audit |
| F09 | server live but model unready | model readiness |
| F10 | tensor shape mismatch | request schema |
| F11 | dtype mismatch | request schema |
| F12 | queue saturation | queue metric |
| F13 | batching worsens p99 | cohort benchmark |
| F14 | instance group oversubscribes GPU | per-GPU VRAM |
| F15 | startup GPU OOM | device logs |
| F16 | Kubernetes GPU scheduling failure | events |
| F17 | model download/egress blocked | network |
| F18 | HTTP/gRPC routing mismatch | gateway |
| F19 | decoupled response incomplete | terminal validator |
| F20 | stale metrics | scrape freshness |
| F21 | sensitive tensors in logs | privacy audit |
| F22 | public admin/model-control endpoint | exposure audit |
| F23 | rollout lacks GPU surge headroom | scheduling |
| F24 | GPU-hours per valid completion high | FinOps |

## 17. Sixteen operational runbooks

1. Verify pinned Triton/backend/model versions.
2. Diagnose server liveness/readiness.
3. Diagnose per-model readiness.
4. Validate repository paths and permissions.
5. Validate config.pbtxt and model versions.
6. Investigate backend/CUDA initialization.
7. Audit model-control mode and access.
8. Diagnose request tensor schema errors.
9. Investigate queue saturation.
10. Tune batching only after baseline.
11. Diagnose instance-group GPU OOM.
12. Diagnose Kubernetes GPU scheduling.
13. Diagnose gateway HTTP/gRPC routing.
14. Validate streaming/decoupled completion.
15. Restore metrics, privacy and security.
16. Execute rollback, verify SLO and document RCA.

## 18. Eight tabletop game days

1. Repository update contains an invalid model config.
2. Server ready but required model not ready.
3. Dynamic batching increases p99 latency.
4. Extra model instance causes GPU OOM.
5. Model-control API is exposed to an unauthorized tenant.
6. gRPC gateway routes to the HTTP listener.
7. Decoupled backend emits an incomplete response.
8. Rollout doubles GPU usage and exceeds cost ceiling.

Default tabletop; live fault injection requires separately approved isolated staging.

## 19. Twenty acceptance gates

G01 pinned Triton/backend; G02 model provenance; G03 repository layout; G04 model config; G05 control mode; G06 model readiness; G07 synthetic inference; G08 HTTP/gRPC contract; G09 batching; G10 instance placement; G11 GPU headroom; G12 Kubernetes scheduling; G13 TLS/auth; G14 admin isolation; G15 privacy; G16 telemetry freshness; G17 complete response; G18 rollback; G19 GPU FinOps; G20 independent review.

See [companion lab](../labs/026-triton-inference-server-architecture-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 027 — Triton Model Repository and Model Lifecycle.**
