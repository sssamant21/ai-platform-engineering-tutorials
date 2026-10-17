# Chapter 038 — Ray Serve Routing, Batching and Backpressure

**Part 06 — Ray Serve and KServe | Chapter 038 of 100**

> **Publication:** Written tutorial published. Live labs, independent technical/security review and operational acceptance pending. Ray Serve routing, batching, streaming and concurrency APIs change across releases; pin versions and verify documented signatures before deployment.

## Objectives

Understand ingress routing, deployment handles, request queues, async execution, dynamic batching, backpressure, per-tenant fairness, admission control, cancellation, streaming completion and production latency SLOs. Build executable CPU-only demonstrations and design safe GPU staging experiments.

## 1. Serving request path

```text
Client (auth + deadline)
      │
Gateway (TLS, rate limit, tenant quota)
      │
Ray Serve proxy (route and protocol)
      │
Deployment handle / router
      │
Pending requests and replica selection
      │
Ray Serve replica actor
      ├─ preprocessing
      ├─ model inference
      └─ postprocessing
      │
Terminal response / streaming completion
```

Each layer may queue, reject or retry. A successful HTTP status is not proof of a complete LLM response.

## 2. Routes, ingress and deployment handles

A Serve application exposes ingress routes; deployments can call other deployments through handles. Keep routing contracts explicit and versioned. Internal deployment handles are not substitutes for an authenticated external gateway.

Conceptual application graph:

```text
/v1/infer → GatewayDeployment
                ├─ TokenizerDeployment
                ├─ InferenceDeployment (GPU)
                └─ ValidatorDeployment
```

Separate lightweight CPU work from GPU model replicas where it improves isolation, but measure extra serialization and network overhead.

## 3. Routing modes and tradeoffs

| Strategy | Benefit | Risk |
|---|---|---|
| single deployment | simple | shared contention |
| composed deployments | modular | more hops/queues |
| versioned routes | canary/rollback | client drift |
| tenant-aware gateway | quotas/isolation | policy errors |
| streaming response | faster perceived output | cancellation and terminal semantics |

Verify Ray Serve's current ingress and route-prefix behavior for the installed release.

## 4. Asynchronous request handling

Async methods can overlap I/O and waiting, but do not magically parallelize GPU kernels. Blocking CPU code inside an event loop can stall unrelated requests. For CPU-bound work, use appropriate executors or actors; for GPU inference, bound in-flight work to actual VRAM and KV capacity.

```python
# Conceptual Serve handler; version-specific request signature.
async def __call__(self, request):
    payload = await request.json()
    validated = validate_synthetic_payload(payload)
    result = await self.model_handle.remote(validated)
    return result
```

This snippet illustrates the call chain, not a production-authenticated endpoint.

## 5. Dynamic batching

Batching can improve GPU throughput by combining requests, but increases wait time and may reduce fairness. Ray Serve supports batching patterns through version-dependent APIs, including `serve.batch` in compatible releases.

| Parameter | Operational effect |
|---|---|
| maximum batch size | throughput versus VRAM |
| batch wait timeout | latency versus occupancy |
| replica concurrency | queue pressure versus utilization |
| input shape compatibility | padding and kernel efficiency |
| output length mix | tail latency and head-of-line blocking |

A high average batch size is not itself a success metric; validate SLO-qualified goodput.

## 6. CPU-only batching example

**[LAB-MUTATE: local sandbox]** For a compatible Ray Serve release, this demonstrates batching without GPU inference:

```python
from ray import serve

@serve.deployment
class BatchEcho:
    @serve.batch(max_batch_size=4, batch_wait_timeout_s=0.05)
    async def batch_echo(self, items: list[str]) -> list[str]:
        return [f"synthetic:{item}" for item in items]

    async def __call__(self, request):
        item = request.query_params.get("item", "test")
        return await self.batch_echo(item)

app = BatchEcho.bind()
```

Validate `serve.batch` decorator options and Serve deployment APIs against your installed Ray version. Do not expose the local test endpoint to public networks.

## 7. Backpressure and bounded queues

Overload protection should exist at gateway, Serve routing and replica levels. Define:
- per-tenant token/request rate limits;
- maximum pending requests and queue timeout;
- maximum active requests/replica;
- input and output token caps;
- cancellation/deadline propagation;
- bounded retries with jitter;
- explicit 429/503 or documented rejection semantics.

Unbounded queueing can preserve nominal success rate while destroying TTFT p99 and exhausting memory.

## 8. Queueing theory for SRE

As utilization approaches effective service capacity, queue wait often increases sharply. Little's Law relates long-run average number in system, arrival rate and time in system:

```text
L = λ × W
```

It requires stable flow and consistent system boundaries. It does not provide p99 latency directly. Use measured load curves and tail latency to select admission thresholds.

## 9. Streaming and cancellation

Streaming reduces perceived latency but adds correctness obligations. Define:
- first token timestamp;
- ordered chunks;
- terminal marker/final flag;
- cancellation cleanup;
- timeout and client disconnect;
- token accounting;
- no duplicate final response on retries.

Ray Serve's streaming support and generator handling are version-dependent; validate HTTP and deployment-handle behavior separately.

## 10. Multi-tenant fairness

A noisy tenant can monopolize replicas with long prompts or long outputs. Apply gateway quotas and per-tenant admission where appropriate. Avoid placing sensitive tenant identifiers into metric labels. Test fairness using synthetic tenant cohorts with equal and unequal prompt lengths.

## 11. Request budgets

For a request deadline, account for gateway, queue, preprocessing, model prefill/decode, postprocessing and transport:

```text
deadline_budget >= gateway + queue + prefill + decode + response
```

These are conceptual components, not guaranteed additive independent metrics. Ensure downstream calls receive remaining deadline and cancel when the upstream request is gone.

## 12. GPU batching constraints

GPU memory may be limited by model weights, activations, workspace, CUDA graphs and KV cache. A larger batch can trigger OOM despite ample Ray logical GPU reservations. Track per-GPU VRAM and active tokens, not only batch count.

For LLM serving, continuous/in-flight batching at a specialized backend may differ from request-level `serve.batch`; avoid stacking batchers blindly without end-to-end measurement.

## 13. Autoscaling interaction

Autoscaling based on ongoing requests or queue pressure can relieve sustained load but may lag cold model startup. Admission and backpressure remain necessary. Test cooldown, min/max replicas, quota and cost impact. Do not assume autoscaling can instantly provision GPU nodes.

## 14. Metrics and SLO

Record:
- ingress request rate, 4xx/5xx, rejection and retry rates;
- pending/in-flight requests per deployment;
- batch size and wait time;
- replica queue/processing time;
- client TTFT/TPOT/E2E p50/p95/p99;
- terminal completion and valid output rate;
- GPU VRAM/utilization and GPU-hours per valid completion;
- telemetry freshness and error-budget burn.

Metric names and units depend on the pinned Ray version; inspect actual exporters before writing alerts.

## 15. Security and privacy

Authenticate at the gateway; isolate Ray dashboard, Serve management and Jobs API. Validate payload schema and token limits before GPU execution. Do not log raw prompts, outputs or tenant identifiers. Guard against untrusted model code, request amplification and cross-tenant cache sharing.

## 16. Safe-read triage

```bash
date -u
kubectl config current-context
kubectl get rayservices,rayclusters -A
kubectl get pods,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
```

For approved namespace:

```bash
kubectl -n RAY_NAMESPACE describe rayservice APPROVED_RAYSERVICE
kubectl -n RAY_NAMESPACE get pods -o wide
```

If Ray CRDs are unavailable, record the limitation; do not install an operator without approval.

## 17. Troubleshooting decision tree

```text
Requests slow or failing?
 ├─ 404 → ingress route or app binding
 ├─ 401/403 → auth/tenant policy
 ├─ 429 → rate limit/admission
 ├─ 502/503 → proxy/replica readiness
 ├─ TTFT high + queue high → backpressure/saturation
 ├─ TTFT high + queue low → prefill/CPU blocking
 ├─ TPOT high → decode/VRAM contention
 ├─ p99 high after batching → batch wait/head-of-line
 ├─ GPU OOM → batch/concurrency/KV
 ├─ incomplete stream → terminal/cancellation
 ├─ tenant starvation → fairness
 ├─ autoscaling lag → cold start/quota
 ├─ metrics stale → exporter/scrape
 └─ cost high → goodput/idle GPU
```

## 18. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | wrong route prefix | HTTP 404 |
| F02 | deployment handle stale | app revision |
| F03 | gateway auth bypass | policy audit |
| F04 | quota denies valid tenant | 403/429 |
| F05 | proxy unavailable | readiness |
| F06 | replica crash | actor logs |
| F07 | blocking async handler | event-loop lag |
| F08 | batch API version mismatch | startup log |
| F09 | incompatible request shapes | batch error |
| F10 | batch wait increases TTFT | p99 |
| F11 | large batch GPU OOM | VRAM |
| F12 | unbounded pending requests | queue depth |
| F13 | retry storm | retry metrics |
| F14 | missing deadline propagation | stale work |
| F15 | cancellation leaks resources | in-flight slots |
| F16 | partial streaming completion | terminal validator |
| F17 | duplicate response after retry | request IDs |
| F18 | long prompt starves short requests | cohort fairness |
| F19 | tenant quota bypass | admission audit |
| F20 | autoscaling cold-start lag | replica startup |
| F21 | GPU logical reservation mismatch | resource inventory |
| F22 | telemetry units mislabeled | HELP/TYPE |
| F23 | rollback leaves route drift | manifest |
| F24 | cost per valid completion rises | FinOps |

## 19. Sixteen operational runbooks

1. Verify ingress route and deployment binding.
2. Diagnose gateway authentication and quotas.
3. Recover proxy/replica readiness.
4. Diagnose blocking async handlers.
5. Validate Serve batching API/version.
6. Diagnose batch shape and serialization errors.
7. Tune batch size/wait against p99.
8. Recover GPU batch OOM.
9. Enforce pending queue and admission bounds.
10. Stop retry storms and propagate deadlines.
11. Recover cancellation and terminal streaming.
12. Investigate tenant starvation/fairness.
13. Recover autoscaling cold-start and GPU quota.
14. Reconcile Ray/Kubernetes GPU reservations.
15. Restore telemetry, SLO and FinOps.
16. Execute route/application rollback and RCA.

## 20. Eight tabletop game days

1. Route-prefix drift produces 404 during rollout.
2. Async handler blocks and stalls all requests.
3. Batch wait improves throughput but breaks TTFT p99.
4. GPU OOM after raising batch size.
5. Gateway retries amplify a saturated queue.
6. Client disconnect leaves GPU inference running.
7. One tenant starves other tenants.
8. Autoscaling lag and route rollback collide.

Tabletop by default; live injection requires separately approved isolated staging.

## 21. Twenty acceptance gates

G01 approved scope/versions; G02 route binding; G03 gateway auth; G04 tenant quota; G05 proxy/replica readiness; G06 async handler health; G07 batching API; G08 batch shape/correctness; G09 TTFT/E2E; G10 TPOT/decode; G11 bounded queue; G12 deadline/retry; G13 cancellation; G14 streaming completion; G15 tenant fairness; G16 autoscaling/quota; G17 GPU VRAM; G18 telemetry/SLO; G19 rollback/FinOps; G20 independent review.

See [companion lab](../labs/038-ray-serve-routing-batching-backpressure-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 039 — Ray Serve Autoscaling and Fault Tolerance.**
