# Chapter 037 — Ray Actors, Placement Groups and GPU Resources

**Part 06 — Ray Serve and KServe | Chapter 037 of 100**

> **Status:** Written tutorial published. Live labs, technical/security review and operational acceptance pending. Ray and KubeRay APIs, placement strategies and scheduling behavior are version-dependent; verify against a pinned release.

## Learning objectives

Explain Ray tasks and actors, logical CPU/GPU resources, placement groups, scheduling strategies, resource bundles, actor lifetime, GPU device visibility, KubeRay worker topology and failure recovery. Build capacity and isolation plans for inference replicas and diagnose scheduling failures.

## 1. Ray Core resource model

Ray executes tasks and stateful actors across nodes. A task is a remotely executed function; an actor is a stateful worker with methods invoked remotely. Ray's scheduler accounts for **logical resource reservations** such as CPU and GPU. These are scheduling controls, not hard CPU/VRAM isolation or a replacement for Kubernetes cgroups and device-plugin allocation.

```text
Kubernetes GPU worker Pod (device-plugin allocated GPUs)
          │
     Ray worker node advertises logical resources
          │
     Ray scheduler and placement strategy
          ├─ remote tasks
          ├─ actors
          └─ placement groups with resource bundles
                 │
       Ray Serve replica actors
                 │
        CUDA-visible GPU(s), model, KV cache
```

Never equate Ray logical `num_gpus=1` with guaranteed exclusive GPU memory in every deployment mode.

## 2. Tasks versus actors

| Dimension | Ray task | Ray actor |
|---|---|---|
| Execution | function invocation | persistent stateful process |
| State | usually stateless per call | process-local state |
| GPU reservation | task options | actor creation options |
| Failure | retry policy/version dependent | actor restart/reconstruction policy |
| Inference use | preprocessing/batch jobs | loaded model replica |
| Capacity risk | burst concurrency | persistent VRAM residency |

Ray Serve deployments generally use Ray actors for replicas; the Serve controller manages replica lifecycle. Direct Ray actors and Serve replicas are not interchangeable operationally.

## 3. Basic Ray resource discovery

**[SAFE-READ]** from an authorized Ray client/session with the correct cluster connection:

```python
import ray
ray.init(address="auto")
print(ray.cluster_resources())
print(ray.available_resources())
for node in ray.nodes():
    print(node.get("NodeID"), node.get("Alive"), node.get("Resources"))
```

Avoid printing private node metadata in shared logs. `ray.init(address="auto")` connects only when an authorized cluster is discoverable; use a local sandbox otherwise.

## 4. GPU actors

Illustrative Ray Core API for a pinned compatible version:

```python
import ray

ray.init()

@ray.remote(num_gpus=1)
class GpuProbe:
    def gpu_visibility(self):
        import os
        return {"cuda_visible_devices": os.environ.get("CUDA_VISIBLE_DEVICES", "")}

actor = GpuProbe.remote()
print(ray.get(actor.gpu_visibility.remote()))
ray.shutdown()
```

Run only in an approved GPU sandbox. Ray commonly sets `CUDA_VISIBLE_DEVICES` for GPU-assigned workers, but this environment variable alone is not proof of usable CUDA hardware or exclusive VRAM. Do not use fractional GPU reservations for large model replicas without strict capacity validation.

## 5. Placement groups

Placement groups reserve collections of logical resource bundles together for gang-style scheduling. This is useful for distributed inference needing multiple actors/GPU ranks. Common placement strategies include `PACK`, `SPREAD`, `STRICT_PACK` and `STRICT_SPREAD`; verify semantics in the pinned Ray release.

```text
Placement group:
  bundle 0: CPU 2 + GPU 1
  bundle 1: CPU 2 + GPU 1

PACK           → prefer co-location where possible
SPREAD         → prefer distributing bundles
STRICT_PACK    → require one node for all bundles
STRICT_SPREAD  → require separate nodes
```

Placement-group reservation does not guarantee NVLink topology, GPU locality within a host, available VRAM or NCCL health. Verify GPU UUID and physical topology.

## 6. Minimal placement-group lab

**[LAB-MUTATE]** approved isolated Ray sandbox:

```python
import ray
from ray.util.placement_group import placement_group, remove_placement_group

ray.init()
pg = placement_group([{"CPU": 1}, {"CPU": 1}], strategy="PACK")
try:
    ray.get(pg.ready(), timeout=30)
    print("placement group ready:", pg.id)
finally:
    remove_placement_group(pg)
    ray.shutdown()
```

This CPU-only demonstration avoids requiring a GPU. It may remain pending if the local cluster lacks enough schedulable logical CPU capacity. Ensure cleanup on exceptions.

## 7. GPU placement-group design

For a two-rank tensor-parallel workload, a candidate design reserves two GPU bundles. Decide whether ranks require the same host, different hosts or specific interconnect topology. A `STRICT_PACK` group cannot schedule across multiple nodes; `STRICT_SPREAD` requires enough distinct eligible nodes.

Do not claim a two-GPU placement group automatically starts a correct NCCL process group. Runtime orchestration, rank mapping, port/network access and health checks remain necessary.

## 8. Ray Serve replica resources

Serve deployments can request resources through supported actor options. Example (API version dependent):

```python
from ray import serve

@serve.deployment(
    ray_actor_options={"num_cpus": 2, "num_gpus": 1}
)
class ModelReplica:
    def __init__(self):
        self.model_loaded = True  # replace only with approved model initialization

    def __call__(self, request):
        return {"ready": self.model_loaded}

app = ModelReplica.bind()
```

This is a teaching snippet, not a production application. Verify installed Serve decorator signatures and model lifecycle; avoid exposing unauthenticated routes.

## 9. Kubernetes GPU resource alignment

| Layer | Verify |
|---|---|
| Kubernetes node | `nvidia.com/gpu` allocatable and healthy |
| GPU worker Pod | GPU limits/requests and device-plugin assignment |
| Ray node | advertised logical GPU count |
| Serve replica | actor GPU reservation and concurrency |
| CUDA runtime | visible GPU UUID and driver support |
| Model | weights + KV + workspace + safety headroom |
| Distributed inference | placement, ranks, NCCL, topology |

Misalignment can create Ray-level pending actors even when Kubernetes Pods are Running, or CUDA OOM even when Ray reports adequate logical GPU resources.

## 10. Capacity worksheet

```text
GPU replicas per node <= min(
    Kubernetes allocated GPU devices / GPU devices per replica,
    Ray logical GPU capacity / Ray GPUs per replica,
    VRAM-safe model replicas,
    topology and quota constraints
)
```

GPU memory is the limiting factor for many inference workloads. Account for model weights, activations, CUDA graphs, KV cache, communication buffers, allocator overhead and headroom. Record model cold-start times.

## 11. Actor lifecycle and fault tolerance

Actor startup includes worker process initialization and model loading. Ray can restart actors or retry tasks under configured policies, but state restoration, GPU memory cleanup and idempotency must be validated. A restarted actor may lose in-memory KV state and outstanding requests.

Plan:
- startup timeout and readiness;
- bounded retries/backoff;
- health probes and dead actor detection;
- request deduplication and terminal completion;
- draining before scale-down;
- recovery when the Ray head or GPU worker disappears.

## 12. Resource contention and fairness

A deployment can be resource-starved by other Ray workloads sharing a cluster. Logical reservations do not prevent contention for memory bandwidth, PCIe/NVLink, disk I/O, network, object store or host memory. Use dedicated worker pools and namespaces for sensitive GPU serving where appropriate.

Avoid unbounded actor/task creation: pending placement groups and task backlogs can amplify incidents.

## 13. KubeRay operational inspection

**[SAFE-READ]** approved Kubernetes context:

```bash
date -u
kubectl config current-context
kubectl get rayclusters,rayservices -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

For authorized namespace:

```bash
kubectl -n RAY_NAMESPACE describe rayservice APPROVED_RAYSERVICE
kubectl -n RAY_NAMESPACE get pods -o wide
```

If CRDs are absent, stop and record that fact; do not install or modify the operator without approval.

## 14. Metrics, SLO and FinOps

Track pending actors, placement-group scheduling delay, resource reservations, Ray node availability, worker restarts, replica cold starts, per-GPU VRAM, request queue, TTFT/TPOT, terminal completion and GPU-hours per valid completion. Metric names and endpoints differ by release; inventory actual labels/units before alerting.

GPU allocation efficiency is not the same as useful model throughput. Measure SLO-qualified goodput and wasted reservation time.

## 15. Security

Protect Ray dashboard/Jobs API and control endpoints. Validate actor code and model artifact provenance. Never run arbitrary user-supplied Python in a shared inference cluster. Use least-privilege Kubernetes ServiceAccounts, private Ray networking, namespace isolation, GPU quotas and approved registry IAM.

## 16. Troubleshooting decision tree

```text
Actor or GPU scheduling failure?
 ├─ Pod Pending → Kubernetes quota/taint/GPU
 ├─ Ray actor Pending → logical resources/placement group
 ├─ group never ready → infeasible bundles/strategy
 ├─ actor starts but CUDA unavailable → device mapping/driver
 ├─ actor CUDA OOM → VRAM oversubscription
 ├─ TP ranks mislocated → topology/strategy
 ├─ actor repeatedly restarts → startup/model/health
 ├─ object store pressure → object lifecycle
 ├─ latency high → resource contention/queue
 ├─ scale-down interrupts requests → draining
 ├─ metrics missing → exporter/schema
 └─ cost high → idle reservations/valid goodput
```

## 17. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | Kubernetes GPU allocatable missing | node status |
| F02 | GPU Pod Pending due to quota | events |
| F03 | Ray advertises incorrect GPU count | cluster resources |
| F04 | actor CPU reservation infeasible | pending actor |
| F05 | placement bundles cannot fit | group state |
| F06 | STRICT_PACK cannot fit one node | node resources |
| F07 | STRICT_SPREAD lacks distinct nodes | topology |
| F08 | group reservation leaked | active groups |
| F09 | actor GPU assigned but CUDA fails | driver/log |
| F10 | GPU UUID mismatch | device mapping |
| F11 | fractional GPU causes VRAM OOM | per-GPU memory |
| F12 | model cold start exceeds deadline | startup timing |
| F13 | actor restarts lose state | request traces |
| F14 | retries duplicate inference | completion IDs |
| F15 | TP rank placement wrong | rank topology |
| F16 | NCCL communication stalls | distributed logs |
| F17 | object store memory exhausted | memory stats |
| F18 | task backlog grows unbounded | pending tasks |
| F19 | noisy neighbor reduces throughput | contention |
| F20 | scale-down interrupts streaming | terminal validator |
| F21 | worker node disappears | Ray node state |
| F22 | Ray dashboard exposed | security audit |
| F23 | metrics stale | scrape freshness |
| F24 | GPU-hours per valid completion rises | FinOps |

## 18. Sixteen operational runbooks

1. Verify Kubernetes GPU allocatable and device plugin.
2. Diagnose GPU Pod scheduling and quotas.
3. Reconcile Ray logical resource advertisements.
4. Diagnose pending tasks and actors.
5. Diagnose placement-group feasibility.
6. Correct PACK/SPREAD strategy under change control.
7. Detect and clean up orphaned reservations.
8. Diagnose CUDA visibility and GPU UUID.
9. Recover GPU VRAM oversubscription.
10. Diagnose actor cold start and readiness.
11. Recover actor crash/retry and state loss.
12. Validate TP/PP rank topology and NCCL.
13. Recover object-store/task backlog pressure.
14. Diagnose noisy neighbors and streaming drains.
15. Contain management exposure and restore telemetry/FinOps.
16. Execute worker/Serve release rollback and RCA.

## 19. Eight tabletop game days

1. Kubernetes GPU allocatable drops to zero.
2. Ray reports GPU resources but actors remain pending.
3. STRICT_PACK cannot fit two GPU bundles on a node.
4. A placement group remains reserved after cancellation.
5. Fractional GPU oversubscription triggers CUDA OOM.
6. One TP rank lands on a slow interconnect.
7. GPU worker disappears during streaming.
8. Ray dashboard is exposed during an autoscaling event.

Default tabletop; live fault injection requires separately approved isolated staging.

## 20. Twenty acceptance gates

G01 approved scope/versions; G02 GPU device plugin; G03 Pod GPU allocation; G04 Ray resource inventory; G05 actor scheduling; G06 placement-group feasibility; G07 strategy/topology; G08 cleanup of reservations; G09 CUDA device visibility; G10 VRAM headroom; G11 actor startup/readiness; G12 actor recovery; G13 request idempotency; G14 TP/PP/NCCL; G15 task/object-store pressure; G16 fairness/streaming; G17 security/privacy; G18 telemetry/SLO; G19 GPU FinOps/rollback; G20 independent review.

See [companion lab](../labs/037-ray-actors-placement-groups-gpu-resources-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 038 — Ray Serve Routing, Batching and Backpressure.**
