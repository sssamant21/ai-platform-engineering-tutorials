# Chapter 027 — Triton Model Repository and Model Lifecycle

**Part 05 — NVIDIA Triton and TensorRT-LLM | Chapter 027 of 100**

> **Status:** Written tutorial published. Live labs, technical/security review and operational acceptance pending. Validate all model-control behavior and repository conventions against the pinned Triton and backend release.

## Objectives

Design a secure, versioned Triton model repository; distinguish model artifact layout, configuration and backend requirements; operate load, unload, version policies and readiness; plan immutable releases, rollback, multi-model GPU capacity, repository storage and production recovery.

## 1. Model repository as a deployment contract

A Triton model repository is not merely a directory of weights. It is a versioned serving contract tying together artifact format, backend, model configuration, input/output tensor schema, instance placement, scheduling and readiness.

```text
Approved model registry / artifact store
                 │
        immutable revision + checksum
                 ▼
       Triton model repository
         ├─ model_A/
         │   ├─ config.pbtxt
         │   ├─ 1/model.onnx
         │   └─ 2/model.onnx
         └─ model_B/
             ├─ config.pbtxt
             └─ 1/model.plan
                 │
           Model lifecycle
       discover → validate → load
                 │
         backend initialization
                 │
        model/version readiness
                 │
          serving traffic
                 │
      unload / replace / rollback
```

This layout is illustrative; ONNX, TensorRT, Python, PyTorch and TensorRT-LLM backends have different artifact and configuration requirements. Some backends support additional files or layouts. Validate actual conventions before publishing.

## 2. Canonical repository layout

```text
model_repository/
  fraud_classifier/
    config.pbtxt
    1/
      model.onnx
    2/
      model.onnx
  ranking_engine/
    config.pbtxt
    1/
      model.plan
```

A numeric version directory is part of Triton's conventional versioning scheme. Do not equate a version directory number with a Git commit, model registry digest or semantic version. Maintain a manifest linking all three.

## 3. config.pbtxt fundamentals

Illustrative configuration for a compatible ONNX model:

```protobuf
name: "fraud_classifier"
platform: "onnxruntime_onnx"
max_batch_size: 0
input [
  {
    name: "INPUT"
    data_type: TYPE_FP32
    dims: [ 8 ]
  }
]
output [
  {
    name: "OUTPUT"
    data_type: TYPE_FP32
    dims: [ 2 ]
  }
]
instance_group [
  { kind: KIND_CPU count: 1 }
]
```

**Not executable without a matching model**. Input/output tensor names, shapes, dtype and backend support must exactly match the artifact. `max_batch_size: 0` means the model is configured without Triton batching; do not assume a leading batch dimension. GPU placement and backend capability must be validated separately.

## 4. Model configuration and validation

Before loading, validate:
- model name, platform/backend and compatible artifact;
- exact tensor names, dtypes, dimensions and shape tensors where applicable;
- max batch size, dynamic/sequence batching compatibility;
- version policy and model dependencies;
- CPU/GPU instance groups, memory and execution environment;
- artifact checksum, tokenizer/preprocessor compatibility and license;
- pinned Triton image, backend versions and CUDA/driver requirements.

Configuration auto-completion is version/backend-dependent. Explicit configuration is preferable when production reproducibility and auditability matter.

## 5. Model control modes

Triton commonly supports the following model-control modes, subject to release and deployment configuration:

| Mode | Behavior | Operational risk |
|---|---|---|
| NONE | load configured repository at startup; no routine runtime repository control | changes may require restart |
| POLL | periodically detect repository changes | partial updates or inconsistent remote listings |
| EXPLICIT | load/unload via authorized model-control API | admin exposure, accidental unload |

Choose a mode deliberately. Do not combine mutable repository updates with an uncoordinated production rollout. For object storage, ensure complete, immutable artifact publication and consistent manifest visibility before any load.

## 6. Version policies

Model version policy can determine which versions are available, such as a latest-N selection, all available versions or specific versions, depending on the pinned Triton release.

Production guidance:
1. publish immutable version directories;
2. verify each artifact digest;
3. stage model loading;
4. validate model/version readiness and synthetic inference;
5. update routing only after acceptance;
6. preserve a known-good version for rollback.

Do not assume version-policy selection is a traffic-splitting mechanism. Canary routing generally belongs in an external routing layer or an explicitly supported serving design.

## 7. Explicit load/unload lifecycle

Illustrative **administrative** API patterns:

```text
POST /v2/repository/models/{model_name}/load
POST /v2/repository/models/{model_name}/unload
GET  /v2/models/{model_name}/ready
GET  /v2/models/{model_name}/versions/{version}/ready
```

Validate endpoint support and request body semantics for the pinned release. Model-control operations mutate serving state and must be limited to authorized operators. Never issue load/unload calls against production from a tutorial without approved change control.

## 8. Readiness is layered

```text
Triton process live
   └─ server ready
        └─ target model loaded
             └─ target version ready
                  └─ synthetic inference correct
                       └─ customer-path SLO healthy
```

A server may be live while one model fails to load. A model can be ready but produce wrong tensor shapes or violate latency SLOs. Readiness is necessary but not sufficient for production acceptance.

## 9. Repository storage and consistency

| Storage | Benefit | Tradeoff |
|---|---|---|
| immutable image/volume | deterministic release | larger images and rollout time |
| object storage | centralized artifacts | download latency, permissions, listing consistency |
| persistent volume | shared/local cache | stale state, lifecycle coupling |
| local SSD cache | faster repeated load | eviction and warmup |

Use workload identity and least-privilege read access. Treat remote storage credentials as secrets. Model-control interfaces and writable repository mounts are high-risk administrative surfaces.

## 10. Multi-model capacity planning

A server hosting multiple models must account for the **sum of resident model instances**, weights, backend workspaces, GPU caches and peak allocations. Loading a new model before unloading the old can require temporary double capacity.

```text
per-GPU peak VRAM =
  resident model instances
  + backend workspace / activations
  + cache or KV (if applicable)
  + CUDA/runtime overhead
  + temporary load/unload overlap
  + safety headroom
```

A successful load is not proof of adequate headroom for production concurrency. Record cold-start and reload time, eviction/availability strategy and rollback feasibility.

## 11. Immutable promotion workflow

```text
Build → scan → checksum → stage
  → config validation → model load
  → readiness + synthetic contract
  → performance + security checks
  → approved route promotion
  → observe SLO/FinOps
  → preserve rollback artifact
```

Model, backend and runtime versions form one tested release unit. Updating only the weights while leaving preprocessing or config unchanged can silently corrupt predictions.

## 12. Read-only inspection

**[SAFE-READ]** in an authorized cluster:

```bash
date -u
kubectl config current-context
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

For an approved Triton deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment TRITON_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/TRITON_DEPLOYMENT --since=15m --tail=100
```

For an authorized localhost tunnel to the HTTP endpoint:

```bash
kubectl -n MODEL_NAMESPACE port-forward svc/APPROVED_TRITON_SERVICE 18000:8000
```

In another terminal:

```bash
curl --fail --silent --show-error http://127.0.0.1:18000/v2/health/ready
curl --fail --silent --show-error http://127.0.0.1:18000/v2/models/APPROVED_MODEL/ready
```

Confirm model name and endpoint support; bind localhost, close the tunnel and avoid exposing administrative operations. These are health checks, not inference acceptance.

## 13. Incident decision tree

```text
Model deployment degraded?
 ├─ repository missing → path, mount, object storage
 ├─ permission denied → workload identity/ACL
 ├─ config invalid → tensor schema/backend
 ├─ model version missing → version policy/directory
 ├─ backend unavailable → pinned image/plugins
 ├─ model load OOM → per-GPU capacity/overlap
 ├─ server ready, model not ready → load logs
 ├─ model ready, wrong output → preprocessing/schema
 ├─ explicit unload accidental → admin audit/rollback
 ├─ POLL partial artifact → immutable publication
 ├─ cold start slow → storage/cache/backend init
 ├─ stale metrics → exporter freshness
 └─ cost rises → idle/resident instances
```

## 14. Security and change control

- Restrict model-control API and repository write access to approved identities.
- Verify checksums and model provenance before promotion.
- Avoid untrusted custom Python/backend code.
- Pin runtime/backend and model artifacts.
- Protect model metadata, input tensors and output data in logs/traces.
- Audit administrative load/unload actions.
- Apply changes through authoritative GitOps or model-release orchestration.
- Keep rollback version available and test restoration before production.

## 15. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | repository path missing | mount/config |
| F02 | artifact download denied | IAM/ACL |
| F03 | artifact checksum mismatch | manifest |
| F04 | invalid config.pbtxt | loader log |
| F05 | tensor name mismatch | schema |
| F06 | dtype/shape mismatch | synthetic test |
| F07 | backend not installed | image inventory |
| F08 | CUDA/driver incompatibility | init log |
| F09 | missing version directory | listing |
| F10 | wrong version policy | config |
| F11 | stale model selected | metadata |
| F12 | model-control mode mismatch | startup |
| F13 | unauthorized load/unload | audit |
| F14 | accidental model unload | admin events |
| F15 | partial POLL update | artifact timestamps |
| F16 | object storage listing drift | manifest |
| F17 | model load GPU OOM | per-GPU VRAM |
| F18 | old/new overlap exhausts VRAM | rollout |
| F19 | cold start exceeds SLO | load timing |
| F20 | server ready but model unready | model readiness |
| F21 | model ready but wrong predictions | fixture |
| F22 | stale model metrics | scrape freshness |
| F23 | sensitive artifact/data in logs | privacy |
| F24 | resident model cost spikes | GPU-hours |

## 16. Sixteen operational runbooks

1. Verify repository root and mount.
2. Diagnose artifact store IAM and egress.
3. Validate checksum, license and provenance.
4. Validate config.pbtxt and tensor schema.
5. Diagnose backend/runtime compatibility.
6. Diagnose model version directory/policy.
7. Audit model-control mode.
8. Investigate unauthorized administrative action.
9. Recover accidental model unload.
10. Diagnose POLL and partial publication.
11. Diagnose model load GPU OOM.
12. Plan old/new model overlap capacity.
13. Diagnose cold-start and cache performance.
14. Validate model readiness and inference fixture.
15. Restore observability/privacy and FinOps.
16. Execute immutable rollback and RCA.

## 17. Eight tabletop game days

1. Object-store credentials expire during model load.
2. New config.pbtxt contains a tensor dtype mismatch.
3. Model version policy selects an unintended revision.
4. Partial repository update is observed by POLL.
5. Unauthorized actor calls model unload.
6. New model load OOMs while old version is resident.
7. Server is ready but model fails a synthetic fixture.
8. Rollback artifact was deleted and cold start exceeds SLO.

Default tabletop. Live fault injection requires approved isolated staging.

## 18. Twenty acceptance gates

G01 repository layout; G02 immutable artifacts; G03 checksum/provenance; G04 storage IAM; G05 config schema; G06 backend compatibility; G07 version policy; G08 model-control mode; G09 admin authorization; G10 per-model readiness; G11 synthetic inference; G12 GPU headroom; G13 cold-start; G14 rollout overlap; G15 privacy; G16 telemetry freshness; G17 rollback; G18 GPU FinOps; G19 operational ownership; G20 independent review.

See [companion lab](../labs/027-triton-model-repository-model-lifecycle-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 028 — Triton Scheduling, Dynamic Batching and Ensembles.**
