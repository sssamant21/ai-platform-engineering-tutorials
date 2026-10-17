# Chapter 038 — Hands-On Lab: Ray Serve Routing, Batching and Backpressure

**Part 06 | Chapter 038 | Published lab plan; live execution pending**

## Goal

Map Ray Serve routing, demonstrate CPU-only batching in an approved sandbox, design queue/admission limits, validate deadlines, streaming completion, tenant fairness, autoscaling and GPU cost.

## Safety

**[SAFE-READ]** approved Kubernetes/Ray state and sanitized metrics. **[LAB-MUTATE]** isolated local CPU sandbox or separately approved staging, synthetic inputs and bounded load. **[CHANGE-CONTROL]** production routing, gateway quotas, GPU batching and fault injection. No PHI, customer prompts, Secrets or public Ray management exposure.

## Lab 0 — Preflight

Record UTC, context, Ray/Serve/KubeRay versions, route prefix, deployment handles, replica count, GPU quota, SLO, owner, approval, cost ceiling and rollback.

```bash
date -u
kubectl config current-context
kubectl get rayservices,rayclusters -A
kubectl get pods,svc -A -o wide
```

If CRDs are missing, record the finding; do not install KubeRay without approval.

## Lab 1 — Routing map [DESIGN]

Draw client → gateway → proxy → Serve route → deployment handle → replica → terminal response. List auth, queue, retry, timeout and cancellation ownership at each layer.

## Lab 2 — CPU-only batching [OPTIONAL LOCAL LAB]

For a pinned compatible Ray Serve release:

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

Validate the decorator and deployment APIs against installed Ray. Serve only on an approved local/private interface; stop the sandbox after testing.

## Lab 3 — Batch experiment [DESIGN]

| Variant | Max batch | Wait | Concurrency | Evidence |
|---|---|---|---|---|
| Baseline | approved | approved | approved | p95/p99, goodput |
| Candidate A | changed | same | same | p95/p99, goodput |
| Candidate B | same | changed | same | p95/p99, goodput |

Change one variable at a time. Compare short/short, long/short, short/long, mixed and burst synthetic cohorts. Abort on OOM, runaway retries, SLO burn or cost ceiling.

## Lab 4 — Backpressure and deadlines [DESIGN]

Specify per-tenant request/token caps, max pending queue, queue timeout, max active requests, deadline propagation, bounded retry/jitter, cancellation cleanup and overload response. Test only under approved synthetic staging.

## Lab 5 — Streaming and fairness [DESIGN]

Define first token, ordered chunks, terminal marker, cancellation and complete response. Design two synthetic tenants with different request lengths to detect starvation. Do not include tenant identifiers in metric labels.

## Lab 6 — Eight tabletop game days

01 route drift; 02 blocked async handler; 03 batch p99 regression; 04 GPU batch OOM; 05 retry storm; 06 disconnect leak; 07 tenant starvation; 08 autoscaling/rollback collision.

## Twenty-four failure checklist

F01 route; F02 handle; F03 auth; F04 quota; F05 proxy; F06 replica; F07 async; F08 batching API; F09 shapes; F10 wait; F11 OOM; F12 queue; F13 retry; F14 deadline; F15 cancel; F16 streaming; F17 duplicate; F18 fairness; F19 quota bypass; F20 autoscale; F21 GPU reservation; F22 metrics; F23 rollback; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Scope/versions | Pending |
| G02 | Route binding | Pending |
| G03 | Gateway auth | Pending |
| G04 | Tenant quota | Pending |
| G05 | Proxy/replica readiness | Pending |
| G06 | Async health | Pending |
| G07 | Batching API | Pending |
| G08 | Batch correctness | Pending |
| G09 | TTFT/E2E | Pending |
| G10 | TPOT/decode | Pending |
| G11 | Bounded queue | Pending |
| G12 | Deadline/retry | Pending |
| G13 | Cancellation | Pending |
| G14 | Terminal streaming | Pending |
| G15 | Tenant fairness | Pending |
| G16 | Autoscaling/quota | Pending |
| G17 | GPU VRAM | Pending |
| G18 | Telemetry/SLO | Pending |
| G19 | Rollback/FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/approval:
Ray/Serve/KubeRay versions:
Route and deployment handle:
Gateway auth/tenant quotas:
Batch size/wait/concurrency:
Queue/deadline/retry:
TTFT/TPOT/E2E and valid completion:
Streaming/cancellation:
Fairness/autoscaling:
GPU VRAM/cost:
Security/telemetry:
Game-day findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop optional local Serve process and approved synthetic load, remove only owned staging resources, restore GitOps source of truth and reconcile GPU-hours. All gates remain pending until independently reviewed.

**Next: Chapter 039 — Ray Serve Autoscaling and Fault Tolerance.**
