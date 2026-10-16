# Chapter 027 — Hands-On Lab: Triton Model Repository and Model Lifecycle

**Part 05 | Chapter 027 | Published lab plan; execution pending**

## Goal

Design and inspect a versioned Triton repository, validate model configuration, readiness, version policy, explicit model control, GPU overlap, security and immutable rollback without modifying production.

## Safety

**[SAFE-READ]** approved repository metadata, Kubernetes inventory and redacted logs. **[LAB-MUTATE]** separately approved isolated staging with synthetic models and bounded GPU-hours. **[CHANGE-CONTROL]** production repository writes, model load/unload, version policy, credentials and routing. Do not use customer tensors, PHI, secrets or unapproved backend code.

## Lab 0 — Preflight

Record UTC, cluster/context, namespace, Triton image/backend version, model artifact checksum, repository storage, model-control mode, GPU SKU/UUID, owner, approval and rollback.

```bash
date -u
kubectl config current-context
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — Repository layout [DESIGN]

```text
model_repository/
  example_model/
    config.pbtxt
    1/
      model.onnx
    2/
      model.onnx
```

Create a design inventory linking model name, numeric Triton version, Git revision, artifact digest, backend, license and storage path. Do not claim the example is executable without a matching model.

## Lab 2 — Configuration contract [DESIGN]

For a synthetic ONNX classifier, specify exact tensor names, dtypes, dimensions, max_batch_size, backend and instance placement. Explain why config and artifact must agree. Review version-specific config auto-completion.

## Lab 3 — Model control and authorization [DESIGN]

Compare NONE, POLL and EXPLICIT. Record which actors can modify repository artifacts, trigger loads/unloads and change traffic routing. Identify the audit trail and rollback for an accidental unload. Do not call administrative APIs in production.

## Lab 4 — Readiness inspection [SAFE-READ; authorized localhost tunnel]

```bash
kubectl -n MODEL_NAMESPACE port-forward svc/APPROVED_TRITON_SERVICE 18000:8000
```

In another terminal:

```bash
curl --fail --silent --show-error http://127.0.0.1:18000/v2/health/ready
curl --fail --silent --show-error http://127.0.0.1:18000/v2/models/APPROVED_MODEL/ready
```

Use an approved Service and model name, bind localhost and close the tunnel. Distinguish server readiness from model/version readiness and correct synthetic inference.

## Lab 5 — GPU overlap worksheet [DESIGN]

For each GPU, estimate resident old/new model weights, workspaces, temporary loading allocations, backend caches and headroom. Plan canary/rollback when double capacity is unavailable. Do not infer memory fit from cluster aggregate.

## Lab 6 — Promotion and rollback [DESIGN]

Define build, checksum, security scan, stage, load, synthetic contract, performance baseline, route promotion, observation and rollback. Require an immutable known-good artifact and tested credentials.

## Lab 7 — Eight tabletop game days

01 expired artifact credentials; 02 invalid dtype; 03 wrong version policy; 04 partial POLL; 05 unauthorized unload; 06 GPU overlap OOM; 07 ready-but-wrong output; 08 missing rollback artifact.

## Twenty-four failure checklist

F01 path; F02 IAM; F03 checksum; F04 config; F05 tensor name; F06 shape/dtype; F07 backend; F08 CUDA; F09 directory; F10 policy; F11 stale revision; F12 control mode; F13 unauthorized action; F14 unload; F15 POLL; F16 storage listing; F17 GPU OOM; F18 overlap; F19 cold start; F20 model unready; F21 wrong output; F22 stale metrics; F23 privacy; F24 cost.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Repository layout | Pending |
| G02 | Immutable artifacts | Pending |
| G03 | Checksum/provenance | Pending |
| G04 | Storage IAM | Pending |
| G05 | Config schema | Pending |
| G06 | Backend compatibility | Pending |
| G07 | Version policy | Pending |
| G08 | Model-control mode | Pending |
| G09 | Admin authorization | Pending |
| G10 | Model readiness | Pending |
| G11 | Synthetic inference | Pending |
| G12 | GPU headroom | Pending |
| G13 | Cold-start | Pending |
| G14 | Rollout overlap | Pending |
| G15 | Privacy | Pending |
| G16 | Telemetry freshness | Pending |
| G17 | Rollback | Pending |
| G18 | GPU FinOps | Pending |
| G19 | Operational owner | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/approval:
Triton/backend/model versions:
Repository layout/storage/IAM:
Model artifact digest/license:
config.pbtxt and tensor schema:
Model-control mode/version policy:
Server/model readiness:
GPU overlap/cold-start:
Privacy/telemetry/FinOps:
Game-day mode/findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Close port-forwards; for approved staging remove only owned test artifacts through the authoritative controller, restore known-good model selection and reconcile GPU-hours. Gates remain pending until independently reviewed.

**Next: Chapter 028 — Triton Scheduling, Dynamic Batching and Ensembles.**
