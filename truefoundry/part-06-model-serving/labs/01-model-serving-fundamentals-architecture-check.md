# Lab 6.1 — Model-Serving Fundamentals and Architecture Check

**Canonical:** [Chapter 6.1](../01-model-serving-fundamentals-architecture/01-model-serving-fundamentals-architecture.md)  
**Default:** **[SAFE-READ]** on existing approved deployments. Optional sample inference deployment is **[LAB-MUTATE]** in a disposable nonproduction TrueFoundry workspace.

## Objectives

Map the model-serving control and data paths; verify artifact identity, runtime scheduling, model loading, readiness, endpoint and inference response; classify failures; and document image+model rollback.

## 0. Safety and environment preflight

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Record TrueFoundry workspace, installed version, cluster, namespace, model owner and approved sample input. No real customer records, secrets or untrusted model artifacts.

## 1. Deployment and model provenance [SAFE-READ]

Record:
- TrueFoundry deployment/revision and authoritative source.
- Image digest and model version/checksum.
- Model artifact URI **metadata only** (redact sensitive bucket/path details where necessary).
- Model download identity, runtime identity and image-pull identity.
- CPU/GPU requests, memory requests/limits, node selection and quota.

```bash
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get pod <pod> -o jsonpath='{.spec.containers[*].image}{"\n"}'
```

Do not retrieve or print credential values.

## 2. Loading, readiness and runtime [SAFE-READ]

```bash
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> logs <pod> --tail=150
kubectl -n <namespace> logs <pod> --previous --tail=150
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> top pod <pod> --containers
```

`--previous` and `top` may be unavailable. Find sanitized evidence for model fetch, load completion, warmup, readiness, restart count and endpoint availability. Distinguish a Running container from a ready model.

## 3. Optional CPU/GPU scheduling checks [SAFE-READ]

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu
kubectl -n <namespace> describe pod <pod>
```

For GPU workloads, verify device-plugin allocation, scheduling constraints, compatible runtime and VRAM headroom. Do not assume GPU access just because a node has an NVIDIA device.

## 4. Authorized inference smoke test

Use the application's documented request schema and approved dummy payload. Replace placeholders with a real authorized endpoint and sample request; never paste authentication tokens into a shared terminal transcript.

```bash
curl --connect-timeout 5 --max-time 30 -i https://<approved-inference-endpoint>/health
```

Then submit a documented **non-sensitive** golden inference request through the supported client or approved test harness. Capture status, request ID, response schema, expected prediction class/shape and latency; avoid recording sensitive input/output.

Port-forward can isolate the Service-to-Pod path but does not prove normal gateway/TLS/auth behavior.

## 5. Optional disposable deployment [LAB-MUTATE]

1. Use a vetted small sample model and compatible CPU runtime first.
2. Pin image digest and model artifact version.
3. Deploy via the supported installed TrueFoundry workflow.
4. Confirm artifact download, warmup, readiness and golden request.
5. Measure P95/P99 latency, request errors, memory and queueing with a small authorized load.
6. If GPU is required, repeat only after GPU device/runtime checks pass.
7. Restore previous known-good image **and model version**, then clean up lab resources.

Do not use guessed TrueFoundry CLI/SDK syntax or stress shared clusters.

## 6. Failure tabletop

| ID | Symptom | First failed transition |
|---|---|---|
| M1 | Pod Pending | Scheduler → node |
| M2 | ImagePullBackOff | Registry → image |
| M3 | Model download denied | Identity → artifact store |
| M4 | Artifact checksum mismatch | Artifact → loader |
| M5 | CrashLoopBackOff during load | Loader → process |
| M6 | CUDA not detected | Device allocation → runtime |
| M7 | GPU OOM | Model execution → VRAM |
| M8 | Container OOMKilled | Process → memory cgroup |
| M9 | Running but not Ready | Load/warmup → readiness |
| M10 | Endpoint 503 | Ready endpoints → route |
| M11 | High P99 and queue depth | Admission → execution |
| M12 | HTTP 200 but incorrect output | Model/schema → application contract |

For each: collect evidence, lowest proven healthy layer, owner, approved repair and rollback validation.

## 7. Acceptance worksheet

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| G1 Deployment mapped | | |
| G2 Image/model versions pinned | | |
| G3 Storage identity scoped | | |
| G4 CPU/GPU scheduling validated | | |
| G5 Model ready after load | | |
| G6 Golden inference request passed | | |
| G7 Latency/errors/resources captured | | |
| G8 Recovery/rollback plan verified | | |

## 8. Evidence and cleanup

```text
Date/time, owner, approval:
TrueFoundry version/workspace:
Cluster/namespace/deployment:
Image digest and model version:
Model storage identity (no secrets):
CPU/GPU and memory configuration:
Model load/warmup/readiness evidence:
Service/EndpointSlice and authorized request:
Golden output schema / result:
Error rate / P95/P99 / queue time:
GPU/VRAM/CPU/memory observations:
M1–M12 classifications:
Image+model rollback plan:
Lab-owned resources removed:
Unexecuted checks and follow-ups:
```

Remove only disposable deployments/routes/artifacts created by the lab using the supported TrueFoundry workflow. Publication of this lab does not mean it has been executed.

**Next:** Chapter 6.2 — Model Artifacts, Registries and Storage Integration.
