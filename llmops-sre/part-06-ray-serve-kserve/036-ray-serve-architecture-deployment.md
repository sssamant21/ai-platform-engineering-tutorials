# Chapter 036 — Ray Serve Architecture and Deployment

**Part 06 — Ray Serve and KServe | Chapter 036 of 100**

> **Status:** Written tutorial published. Live labs, technical/security review and operational acceptance remain pending. Ray Serve, Ray Core and KubeRay APIs and YAML fields vary by version; pin the exact release and validate schemas before use.

## Learning objectives

Explain Ray Core and Ray Serve architecture, deployments, replicas, actors, application routing, request handling, autoscaling, GPU resource reservations, Kubernetes/KubeRay operations, health checks, observability, security, change management and rollback. Build a safe staging deployment plan and diagnose common production failures.

## 1. What Ray Serve provides

Ray Serve is a scalable model/application serving layer built on Ray. It runs user-defined deployments as Ray actors and supports composition of model inference, preprocessing and postprocessing. Ray Core manages distributed processes and resources; Serve manages serving replicas and request routing. **KubeRay** is a Kubernetes operator for managing Ray clusters and related resources. Do not conflate Ray Serve with KServe: they have different reconciliation and routing models.

## 2. Reference architecture

```text
Authenticated client
        │
TLS/Auth/API gateway and quotas
        │
Kubernetes Service / ingress
        │
Ray Serve HTTP proxy / ingress
        │
Serve application route
        │
Deployment handle / request router
        │
Ray Serve replicas (Ray actors)
  ├─ preprocessing
  ├─ model inference on CPU/GPU
  └─ postprocessing
        │
Ray Core scheduler and object store
        │
Ray head + worker nodes / Kubernetes Pods
        │
GPU device plugin, storage, observability
```

Actual proxy placement, controller behavior, routing semantics and APIs depend on Ray version and deployment mode.

## 3. Control plane and data plane

| Component | Role | Operational failure |
|---|---|---|
| Ray head | cluster coordination and services | head unreachable |
| Ray workers | execute Ray actors/tasks | scheduling/capacity loss |
| Serve controller | application/deployment reconciliation | stalled rollout |
| Serve proxy | HTTP/gRPC request ingress where supported | 502/timeouts |
| Serve deployment | versioned replica specification | bad model/config |
| Serve replica | actor handling requests | crash/OOM/slowdown |
| Ray object store | shared objects and transfers | memory pressure |
| KubeRay operator | Kubernetes CRD reconciliation | cluster drift |

A running Ray head does not prove Serve application readiness or valid LLM output.

## 4. Deployment modes

Ray Serve can run on a Ray cluster outside Kubernetes, on a manually managed Ray cluster, or on Kubernetes via KubeRay. For Kubernetes, a RayService custom resource is commonly used to manage a Ray cluster and Serve application lifecycle. Its exact CRD schema and embedded Serve configuration vary across KubeRay releases.

Use:
- local Ray Serve for development;
- dedicated staging RayService for end-to-end validation;
- GitOps-controlled, version-pinned production RayService for managed operations.

Never apply an example manifest directly to production without validating installed CRDs and operator versions.

## 5. Serve deployment model

A Serve application can be built from deployments that encapsulate actors. Replica count, per-replica CPU/GPU resources, concurrency, autoscaling, health checks and startup behavior are version-specific settings.

Conceptual pseudocode (not an executable version-independent example):

```python
# Illustrative only: inspect installed Ray Serve API for supported options.
# @serve.deployment(num_replicas=2, ray_actor_options={"num_gpus": 1})
# class Inference:
#     async def __call__(self, request):
#         return await approved_model_inference(request)
# app = Inference.bind()
```

Pin and test actual import paths and deployment arguments against the chosen Ray release.

## 6. GPU resource scheduling

Ray resource requests describe logical scheduling reservations; they do not replace Kubernetes GPU device-plugin allocation or guarantee VRAM isolation. Align:
- Kubernetes Pod GPU limits and Ray node advertised resources;
- per-replica `num_gpus` and actual CUDA device visibility;
- actor concurrency and model VRAM;
- placement group/topology where required;
- autoscaling max replicas and GPU quotas.

Avoid scheduling multiple large model replicas on one GPU merely because Ray's logical resource configuration appears to permit it.

## 7. Model lifecycle and initialization

A serving replica often loads model weights at actor startup. Account for artifact download, image pull, tokenizer/config, CUDA initialization, model warmup and readiness. Store immutable model digests and license provenance. Use bounded initialization timeouts and capacity headroom.

Distinguish RayService health, Ray cluster health, Serve application status, replica readiness and synthetic inference success.

## 8. Request routing and concurrency

Client request → proxy → deployment handle/router → replica. Queueing can occur at the gateway, proxy, router and replica. Set per-request deadlines, concurrency and backpressure, with explicit cancellation and streaming completion semantics. Autoscaling on queued requests or concurrency requires validation against the installed Serve release.

## 9. RayService lifecycle and safe rollout

```text
GitOps PR + pinned image/model
      → schema validation
      → security and quota review
      → isolated staging RayService
      → controller reconciliation
      → Serve app and replica readiness
      → synthetic golden inference
      → TTFT/TPOT/E2E, GPU and cost tests
      → approved canary
      → promote or rollback immutable bundle
```

Do not assume updating a RayService guarantees zero downtime; verify cluster update strategy, readiness, traffic switching and rollback semantics for the installed operator.

## 10. Kubernetes inspection

**[SAFE-READ]** authorized context:

```bash
date -u
kubectl config current-context
kubectl get crd | grep -i ray
kubectl get rayclusters,rayservices -A
kubectl get pods,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

If the Ray CRDs are not installed, the resource query can fail; record that finding rather than installing an operator without approval.

For approved namespace:

```bash
kubectl -n RAY_NAMESPACE describe rayservice APPROVED_RAYSERVICE
kubectl -n RAY_NAMESPACE get pods -o wide
kubectl -n RAY_NAMESPACE get svc,endpointslices
```

Use the exact CRD group/version from the cluster when required.

## 11. Application example: minimal local Serve

**[LAB-MUTATE, local sandbox only]** The following demonstrates the conceptual Serve API for a version with compatible `serve.run` and `@serve.deployment`; validate against installed documentation:

```python
from ray import serve

@serve.deployment
class HealthEcho:
    def __call__(self, request):
        return {"status": "ok", "message": "synthetic-only"}

app = HealthEcho.bind()

if __name__ == "__main__":
    serve.run(app, route_prefix="/")
```

This is a minimal learning example, not a secure production service. Do not expose it to the public internet. Ray versions differ in lifecycle and ingress configuration.

## 12. Health and readiness

Monitor:
- KubeRay operator reconciliation and RayService status;
- head and worker Pod readiness;
- Serve application/deployment status;
- replica start/stop/error counters;
- gateway and proxy availability;
- valid synthetic request completion;
- GPU VRAM and scheduler capacity.

Use client-observed health alongside platform health. A green dashboard is not a guarantee of semantic correctness.

## 13. Autoscaling and capacity

Define min/max replicas, GPU quota, actor startup time, queue/backlog signal and cooldown. Slow model loading can make reactive scaling insufficient for bursts. Preserve warm capacity for latency-sensitive workloads. Avoid scale-to-zero claims unless the exact deployment stack supports and has tested it.

## 14. Observability and SLOs

Collect request rates, errors, latency histograms, queue/backpressure, replica restarts, actor startup, object-store pressure, Ray node resources, GPU utilization/VRAM, TTFT/TPOT and terminal streaming completion where applicable. Ray metrics and endpoint names vary by release.

Define SLO-qualified goodput and cost per valid completion. Keep prompts, outputs and tenant identifiers out of metric labels and unredacted traces.

## 15. Security and governance

Protect Ray dashboard, Jobs API, Serve management surfaces and metrics behind private networking, RBAC and authenticated gateways. Ray's internal control interfaces are not a substitute for an internet-facing authentication layer. Use least-privilege ServiceAccounts, artifact IAM, image digests, NetworkPolicies, TLS and approved model licenses.

## 16. Incident decision tree

```text
Ray Serve inference failing?
 ├─ RayService CRD absent → operator/install governance
 ├─ reconciliation stalled → operator/CRD/schema
 ├─ head unavailable → head Pod/network/storage
 ├─ worker Pending → quota/GPU/taint
 ├─ replica startup fails → model artifact/CUDA/OOM
 ├─ Serve route 404 → route/application binding
 ├─ 502/503 → proxy/replica readiness
 ├─ latency high → queue/concurrency/model
 ├─ GPU OOM → VRAM/actor oversubscription
 ├─ object store pressure → object lifecycle
 ├─ autoscaling stuck → quota/backlog/cooldown
 ├─ incomplete stream → cancellation/terminal contract
 ├─ metrics missing → exporter/scrape
 └─ cost spike → idle GPU/failed completions
```

## 17. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | RayService CRD missing | API discovery |
| F02 | operator version/schema mismatch | reconcile logs |
| F03 | RayService status stalled | conditions |
| F04 | Ray head crash | head logs |
| F05 | worker Pod Pending | quota/taint |
| F06 | GPU device plugin missing | allocatable |
| F07 | logical Ray GPU oversubscription | actor placement |
| F08 | model artifact unavailable | startup log |
| F09 | tokenizer/model revision drift | golden fixture |
| F10 | replica startup CUDA OOM | VRAM |
| F11 | actor crash loop | replica events |
| F12 | proxy unavailable | service/endpoints |
| F13 | Serve route mismatch | route config |
| F14 | queue saturation | backlog |
| F15 | concurrency overload | in-flight |
| F16 | autoscaling fails to add GPUs | quota/scale |
| F17 | scale-down interrupts requests | drain/cancel |
| F18 | Ray object-store memory pressure | memory |
| F19 | cross-node transfer bottleneck | network |
| F20 | incomplete streaming | terminal validator |
| F21 | Ray dashboard exposed | security audit |
| F22 | telemetry scrape gap | freshness |
| F23 | rollback cluster mismatch | release manifest |
| F24 | GPU-hours per valid completion rises | FinOps |

## 18. Sixteen operational runbooks

1. Validate Ray/KubeRay version and CRD schema.
2. Diagnose RayService reconciliation.
3. Recover Ray head availability.
4. Diagnose worker Pod/GPU scheduling.
5. Align Ray logical GPU and Kubernetes allocation.
6. Recover model artifact/tokenizer drift.
7. Recover replica CUDA OOM and actor crashes.
8. Diagnose proxy/service routing.
9. Diagnose Serve route/application binding.
10. Diagnose queue and concurrency saturation.
11. Recover autoscaling and quota failures.
12. Diagnose object-store/network pressure.
13. Restore streaming and cancellation.
14. Contain dashboard/management exposure.
15. Restore telemetry/SLO and GPU FinOps.
16. Execute RayService release rollback and RCA.

## 19. Eight tabletop game days

1. KubeRay operator cannot reconcile RayService.
2. Ray head fails during a model rollout.
3. GPU workers remain Pending after autoscaling.
4. Replica startup OOM after a model upgrade.
5. Serve proxy is healthy but route returns 404.
6. Burst traffic saturates the request queue.
7. Ray dashboard is accidentally exposed.
8. Rollback restores Ray Pods but not model/tokenizer compatibility.

Default tabletop. Live injection requires separately approved isolated staging.

## 20. Twenty acceptance gates

G01 approved scope/owner; G02 pinned Ray/KubeRay versions; G03 CRD validation; G04 head/worker health; G05 GPU scheduling; G06 Ray resource alignment; G07 model provenance; G08 replica startup; G09 Serve route/ingress; G10 golden correctness; G11 streaming completion; G12 queue/backpressure; G13 autoscaling/quota; G14 object-store health; G15 TTFT/TPOT/E2E; G16 security/privacy; G17 telemetry/SLO; G18 rollback; G19 GPU FinOps; G20 independent review.

See [companion lab](../labs/036-ray-serve-architecture-deployment-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 037 — Ray Actors, Placement Groups and GPU Resources.**
