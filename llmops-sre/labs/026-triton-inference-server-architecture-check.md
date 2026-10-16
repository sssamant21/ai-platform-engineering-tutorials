# Chapter 026 — Hands-On Lab: Triton Inference Server Architecture

**Part 05 | Chapter 026 | Published lab plan; execution pending**

## Goal

Trace Triton's request lifecycle, distinguish server/model readiness, inspect model repository and backend architecture, design secure Kubernetes placement and validate batching, GPU memory, protocol contracts, telemetry and rollback.

## Safety

**[SAFE-READ]** authorized Kubernetes/GPU metadata and sanitized logs. **[LAB-MUTATE]** approved isolated staging with synthetic models and bounded GPU-hours. **[CHANGE-CONTROL]** production Triton/model-control, gateway, network policy and GPU deployment changes. Never use PHI, customer tensors, secrets or unapproved model code.

## Lab 0 — Preflight

Record UTC, cluster/context, namespace, Triton image digest/release, backend list, model artifact revisions, GPU SKU/UUID, CUDA/driver, model-control mode, gateway, owner, cost ceiling and rollback.

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — Architecture diagram [DESIGN]

Draw client/gateway → HTTP/gRPC → model scheduler → backend → model instance → GPU/CPU → response. Mark model repository, health, metrics, logs, tracing and administrative model-control boundaries.

## Lab 2 — Repository inventory [DESIGN]

Illustrative layout:

```text
model_repository/
  example_model/
    config.pbtxt
    1/
      model.onnx
```

Record actual backend-specific file conventions, checksums, permissions, license and version-control policy. Do not assume this example works for TensorRT-LLM.

## Lab 3 — Health versus model readiness [SAFE-READ; optional authorized port-forward]

For an approved Triton Service:

```bash
kubectl -n MODEL_NAMESPACE port-forward svc/APPROVED_TRITON_SERVICE 18000:8000
```

In another terminal:

```bash
curl --fail --silent --show-error http://127.0.0.1:18000/v2/health/live
curl --fail --silent --show-error http://127.0.0.1:18000/v2/health/ready
```

Bind localhost only, close promptly and verify model-specific readiness with the pinned API. Do not interpret server readiness as model acceptance.

## Lab 4 — Scheduler and batching [DESIGN]

Compare a latency-sensitive synthetic workload and a throughput-oriented workload. Define queue, compute input/infer/output, batch sizes, p95/p99 and cost. Identify backend/scheduler compatibility before enabling dynamic or sequence batching.

## Lab 5 — Instance group and GPU headroom [SAFE-READ]

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Document per-GPU weights, workspaces, instance replication, KV if applicable, other processes and headroom. Plan how rolling upgrades affect GPU surge demand.

## Lab 6 — Security, observability and rollback [DESIGN]

Map TLS/auth, internal Service, model-control authorization, network policy, workload identity, model provenance, privacy-safe telemetry, metrics freshness, alerting and immutable rollback. Ensure HTTP, gRPC and metrics ports have separate intended exposure policies.

## Lab 7 — Eight tabletop game days

01 invalid repository config; 02 model unready; 03 batching p99 regression; 04 instance GPU OOM; 05 model-control exposure; 06 protocol routing; 07 incomplete decoupled response; 08 rollout cost spike.

## Twenty-four failure checklist

F01 repository; F02 config; F03 version; F04 backend; F05 CUDA; F06 checksum; F07 control mode; F08 unauthorized load; F09 model readiness; F10 shape; F11 dtype; F12 queue; F13 p99 batching; F14 instance oversubscription; F15 GPU OOM; F16 scheduling; F17 egress; F18 HTTP/gRPC; F19 decoupled completion; F20 stale metrics; F21 tensor logs; F22 admin exposure; F23 GPU surge; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Pinned Triton/backend | Pending |
| G02 | Model provenance | Pending |
| G03 | Repository layout | Pending |
| G04 | Model config | Pending |
| G05 | Model-control mode | Pending |
| G06 | Model readiness | Pending |
| G07 | Synthetic inference | Pending |
| G08 | HTTP/gRPC contract | Pending |
| G09 | Batching | Pending |
| G10 | Instance placement | Pending |
| G11 | GPU headroom | Pending |
| G12 | Kubernetes scheduling | Pending |
| G13 | TLS/auth | Pending |
| G14 | Admin isolation | Pending |
| G15 | Privacy | Pending |
| G16 | Telemetry freshness | Pending |
| G17 | Complete response | Pending |
| G18 | Rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/authorization:
Triton image/backend/model revisions:
HTTP/gRPC and gateway:
Model repository/control mode:
Server and model readiness:
Scheduler/batching/instance groups:
GPU/CPU placement and headroom:
Security/model-control exposure:
Metrics/traces/privacy:
Synthetic response validation:
Game-day mode/results:
Rollback/GPU-hours/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Close port-forwards; stop separately approved staging load, remove only owned test models/resources through the authoritative controller and reconcile GPU-hours. Gates remain pending until independently reviewed.

**Next: Chapter 027 — Triton Model Repository and Model Lifecycle.**
