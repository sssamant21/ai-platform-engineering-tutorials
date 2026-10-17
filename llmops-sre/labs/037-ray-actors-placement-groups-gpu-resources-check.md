# Chapter 037 — Hands-On Lab: Ray Actors, Placement Groups and GPU Resources

**Part 06 | Chapter 037 | Published lab plan; live execution pending**

## Goal

Inventory Ray/Kubernetes GPU resources, demonstrate actors and CPU-only placement groups safely, design GPU placement for inference, validate replica VRAM headroom, actor recovery, cleanup and SLO/FinOps evidence.

## Safety

**[SAFE-READ]** authorized Ray/Kubernetes resource inventory. **[LAB-MUTATE]** isolated local CPU sandbox or approved GPU staging with bounded quota. **[CHANGE-CONTROL]** production placement groups, Ray worker scaling, GPU quotas and actor restarts. No PHI, customer prompts, Secrets or public Ray dashboard.

## Lab 0 — Preflight

Record UTC, Ray/KubeRay/Serve versions, cluster/context, owner, approval, GPU quota, model/engine digests, TP/PP, cost ceiling and rollback.

```bash
date -u
kubectl config current-context
kubectl get rayclusters,rayservices -A
kubectl get nodes -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

## Lab 1 — Ray resource inventory [SAFE-READ]

In an authorized Ray client session:

```python
import ray
ray.init(address="auto")
print(ray.cluster_resources())
print(ray.available_resources())
ray.shutdown()
```

If the authorized Ray cluster is not reachable, record the limitation; do not start a cluster on shared infrastructure without approval.

## Lab 2 — CPU actor [OPTIONAL LOCAL LAB]

```python
import ray
ray.init()

@ray.remote(num_cpus=1)
class Counter:
    def __init__(self):
        self.value = 0
    def inc(self):
        self.value += 1
        return self.value

a = Counter.remote()
assert ray.get(a.inc.remote()) == 1
assert ray.get(a.inc.remote()) == 2
ray.shutdown()
```

Record that actor state is process-local and is not a durable database.

## Lab 3 — CPU-only placement group [OPTIONAL LOCAL LAB]

```python
import ray
from ray.util.placement_group import placement_group, remove_placement_group

ray.init()
pg = placement_group([{"CPU": 1}, {"CPU": 1}], strategy="PACK")
try:
    ray.get(pg.ready(), timeout=30)
    print("ready:", pg.id)
finally:
    remove_placement_group(pg)
    ray.shutdown()
```

Requires two schedulable logical CPUs; use an approved sandbox with sufficient capacity. Confirm reservation cleanup.

## Lab 4 — GPU placement design [DESIGN]

Compare PACK, SPREAD, STRICT_PACK and STRICT_SPREAD for a two-rank synthetic inference workload. Record node GPU count, NVLink/PCIe topology, GPU UUID, Ray logical GPU, Kubernetes device allocation, TP/PP and NCCL requirements. Do not assume placement group implies correct CUDA rank initialization.

## Lab 5 — GPU capacity and SLO [DESIGN]

Estimate model weights + activations + CUDA graphs + KV cache + workspace + communication + headroom per GPU. Measure/plan actor startup, pending placement duration, TTFT/TPOT, queue, valid terminal completion, per-GPU VRAM and GPU-hours per valid completion.

## Lab 6 — Eight tabletop game days

01 GPU allocatable loss; 02 Ray actor pending; 03 STRICT_PACK infeasible; 04 orphaned placement group; 05 fractional GPU OOM; 06 TP topology slowdown; 07 worker loss during streaming; 08 dashboard exposure.

## Twenty-four failure checklist

F01 allocatable; F02 Pod Pending; F03 Ray GPU count; F04 CPU reservation; F05 bundle fit; F06 STRICT_PACK; F07 STRICT_SPREAD; F08 leaked group; F09 CUDA; F10 UUID; F11 fractional GPU; F12 cold start; F13 state loss; F14 duplicate retry; F15 TP placement; F16 NCCL; F17 object store; F18 backlog; F19 contention; F20 drain; F21 worker loss; F22 dashboard; F23 metrics; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions | Pending |
| G02 | GPU device plugin | Pending |
| G03 | Pod GPU allocation | Pending |
| G04 | Ray resource inventory | Pending |
| G05 | Actor scheduling | Pending |
| G06 | Placement feasibility | Pending |
| G07 | Strategy/topology | Pending |
| G08 | Reservation cleanup | Pending |
| G09 | CUDA visibility | Pending |
| G10 | VRAM headroom | Pending |
| G11 | Actor startup | Pending |
| G12 | Actor recovery | Pending |
| G13 | Idempotency | Pending |
| G14 | TP/PP/NCCL | Pending |
| G15 | Object-store/backlog | Pending |
| G16 | Fairness/streaming | Pending |
| G17 | Security/privacy | Pending |
| G18 | Telemetry/SLO | Pending |
| G19 | FinOps/rollback | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/approval:
Ray/KubeRay/Serve versions:
Node GPU and Ray logical resources:
Actor scheduling and placement groups:
PACK/SPREAD/topology:
CUDA visibility/VRAM:
Model startup/recovery:
TP/PP/NCCL:
Queue/TTFT/TPOT/terminal completion:
Security/telemetry:
GPU-hours/cost:
Game-day mode/findings:
Cleanup/rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Optional CPU-only examples call `ray.shutdown()` and remove placement groups. For approved staging, stop owned actors/load, restore GitOps configuration and reconcile GPU-hours. All gates remain pending until independently reviewed.

**Next: Chapter 038 — Ray Serve Routing, Batching and Backpressure.**
