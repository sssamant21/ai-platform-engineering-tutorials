# Chapter 036 — Hands-On Lab: Ray Serve Architecture and Deployment

**Part 06 | Chapter 036 | Published lab plan; live execution pending**

## Goal

Inventory an approved KubeRay cluster, map Ray Serve control/data planes, validate GPU scheduling, model/replica readiness, request routing, autoscaling, observability, security and rollback. Use an isolated local synthetic Serve application only when authorized.

## Safety

**[SAFE-READ]** Kubernetes/Ray inventory and sanitized logs. **[LAB-MUTATE]** approved isolated staging or local sandbox, synthetic-only input, bounded GPU-hours. **[CHANGE-CONTROL]** operator, RayService, GPU worker pools, production routing and faults. No PHI, Secrets, customer prompts or public dashboard exposure.

## Lab 0 — Preflight

Record UTC, cluster/context, Ray/KubeRay/Serve versions, CRD group/version, model/tokenizer digests, GPU SKU/quota, owner, approval, SLO, cost ceiling and rollback.

```bash
date -u
kubectl config current-context
kubectl get crd | grep -i ray
kubectl get rayclusters,rayservices -A
kubectl get pods,svc -A -o wide
```

If CRDs are absent, stop and record the finding; do not install the operator without approval.

## Lab 1 — Architecture [DESIGN]

Draw gateway → Serve proxy → application → deployment handle/router → actor replicas → Ray head/workers → GPU. Identify reconciliation, artifact and telemetry ownership.

## Lab 2 — GPU resources [SAFE-READ]

```bash
kubectl get nodes -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

On authorized GPU host:

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used --format=csv
```

Compare Kubernetes allocatable GPU and Ray logical `num_gpus` per replica; record VRAM headroom and quota.

## Lab 3 — Synthetic Serve application [OPTIONAL LOCAL LAB]

For a pinned compatible Ray Serve release:

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

Do not bind an unauthenticated endpoint to a public interface. Validate actual version API and shut down the local sandbox after testing.

## Lab 4 — RayService staging plan [DESIGN]

Prepare an immutable RayService release manifest with approved Ray/KubeRay image, resource requests/limits, GPU worker group, Serve import path/config, internal-only ingress, artifact digests, startup/readiness, quota and rollback. Validate against installed CRD schema before any apply.

## Lab 5 — SLO and security [DESIGN]

Define client TTFT/TPOT/E2E, valid terminal completion, queue time, replica health, actor restarts, GPU VRAM, object-store pressure, autoscaling lag and GPU-hours per valid completion. Review dashboard/Jobs API access and model registry IAM.

## Lab 6 — Eight tabletop game days

01 operator reconciliation; 02 head failure; 03 GPU worker Pending; 04 replica CUDA OOM; 05 route 404; 06 queue overload; 07 dashboard exposure; 08 incomplete rollback.

## Twenty-four failure checklist

F01 CRD; F02 schema; F03 reconcile; F04 head; F05 Pending; F06 device plugin; F07 logical GPU; F08 artifact; F09 tokenizer; F10 CUDA OOM; F11 actor; F12 proxy; F13 route; F14 queue; F15 concurrency; F16 autoscaling; F17 drain; F18 object store; F19 network; F20 stream; F21 dashboard; F22 telemetry; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/owner | Pending |
| G02 | Pinned versions | Pending |
| G03 | CRD validation | Pending |
| G04 | Head/worker health | Pending |
| G05 | GPU scheduling | Pending |
| G06 | Ray GPU alignment | Pending |
| G07 | Model provenance | Pending |
| G08 | Replica startup | Pending |
| G09 | Route/ingress | Pending |
| G10 | Golden correctness | Pending |
| G11 | Terminal streaming | Pending |
| G12 | Queue/backpressure | Pending |
| G13 | Autoscaling/quota | Pending |
| G14 | Object-store health | Pending |
| G15 | TTFT/TPOT/E2E | Pending |
| G16 | Security/privacy | Pending |
| G17 | Telemetry/SLO | Pending |
| G18 | Rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/change approval:
Ray/KubeRay/Serve versions and CRDs:
Head/worker/Serve status:
Model/tokenizer/image digests:
GPU allocatable/logical/VRAM:
Route/replica readiness:
Golden fixture/streaming:
Queue/TTFT/TPOT/E2E:
Object store/autoscaling:
Security/privacy/telemetry:
GPU-hours and cost:
Game-day mode/findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop optional local Serve process and approved synthetic load, remove only owned staging resources, restore GitOps source of truth and reconcile GPU-hours. Acceptance gates remain pending until independently reviewed.

**Next: Chapter 037 — Ray Actors, Placement Groups and GPU Resources.**
