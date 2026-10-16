# Chapter 028 — Triton Scheduling, Dynamic Batching and Ensembles

**Part 05 — NVIDIA Triton and TensorRT-LLM | Chapter 028 of 100**

> **Status:** Written tutorial published. Live lab execution, technical/security review and operational acceptance pending. Scheduling behavior, config syntax and metrics depend on the pinned Triton release and backend.

## Objectives

Understand Triton scheduling, dynamic and sequence batching, model instance groups, queue policies and ensembles; distinguish throughput optimization from tail-latency and fairness; design safe GPU/CPU placement and model pipelines; benchmark with synthetic cohorts; troubleshoot batching, queue, memory and ensemble failures.

## 1. Request scheduling architecture

```text
Client / Authenticated Gateway
           │
     Triton HTTP/gRPC
           │
      Model scheduler
       ├─ queue / priority / timeout policy
       ├─ compatible request grouping
       └─ instance dispatch
             │
      Model instance group
       ├─ CPU instance(s)
       └─ GPU instance(s)
             │
       Backend execution
             │
       Response / tracing
```

Triton supports multiple scheduling modes depending on model/backend and configuration. Do not assume dynamic batching is suitable for every LLM, stateful model or decoupled backend. Batch composition can reduce compute overhead while increasing queue wait and p99 latency.

## 2. Scheduling types and tradeoffs

| Scheduling pattern | Typical use | Key risk |
|---|---|---|
| Default scheduling | simple compatible model requests | low throughput if underutilized |
| Dynamic batching | compatible stateless inference | queue delay and tail latency |
| Sequence batching | stateful sequences with correlation | state affinity, timeout, slot pressure |
| Ensemble scheduling | multi-stage inference DAG | dependency failure and amplified latency |
| Backend-specific iterative/decoupled | supported autoregressive/streaming backends | completion and cancellation semantics |

Scheduler choice must follow the model contract. A batching policy is not a substitute for tenant admission, per-request token limits or GPU memory headroom.

## 3. Dynamic batching fundamentals

Dynamic batching collects compatible requests within an optional bounded queue delay and dispatches batches to available model instances. The scheduler may use preferred batch sizes when configured and supported.

Illustrative **non-production** model configuration fragment:

```protobuf
max_batch_size: 16
dynamic_batching {
  preferred_batch_size: [ 4, 8 ]
  max_queue_delay_microseconds: 1000
}
```

This fragment is valid only when the model's tensor dimensions, backend and configuration support batching. It is not a recommendation to enable preferred batch sizes universally. `max_queue_delay_microseconds: 1000` means up to roughly 1 ms of intentional batching wait, not a guarantee of 1 ms total queue time.

## 4. Queueing and latency budget

```text
E2E latency ≈ gateway + Triton queue + compute input
            + compute inference + compute output
            + response transport
```

Components can overlap depending on instrumentation. Measure p50/p95/p99 by input shape, batch size, request class and model version.

Synthetic example:

```text
Without batching:
queue 1 ms + compute 18 ms = 19 ms

With batching:
queue 7 ms + compute 9 ms = 16 ms
```

This shows possible improvement for one cohort, not universal behavior. Under low load, waiting to fill batches can worsen latency; under saturation, queue time can dominate even when compute efficiency improves.

## 5. Batch size, GPU memory and instance groups

Larger batches can improve GPU utilization but increase activation/workspace memory. Multiple model instances can improve concurrency yet replicate weights and workspaces.

```text
peak GPU memory =
  model instances / weights
  + activation and batching workspace
  + backend runtime buffers
  + KV or state cache if applicable
  + other GPU processes
  + headroom
```

Measure **per GPU** peak usage under the actual workload. Do not scale batch size and instance count simultaneously without an isolated experiment.

## 6. Sequence batching

Sequence batching is designed for compatible stateful models. It uses sequence identity and lifecycle controls so requests from a logical sequence are handled consistently according to the configured strategy. Validate correlation identifiers, start/end markers, idle timeouts, slot limits and failure cleanup in the pinned backend.

Do not use sequence batching simply because an API streams tokens. A decoupled streaming model and a sequence-batched stateful model have distinct contracts.

## 7. Ensemble scheduling

An ensemble can connect preprocessing, inference and postprocessing models by mapping tensors between steps.

```text
request
   │
   ▼
preprocess_model
   │ normalized_tensor
   ▼
classifier_model
   │ scores
   ▼
postprocess_model
   │
   ▼
response
```

Each stage has its own readiness, schema, resource usage and latency. The ensemble's critical path and failure domain include all required components. Version pinning and rollout must account for dependencies.

## 8. Illustrative ensemble configuration

```protobuf
name: "example_ensemble"
platform: "ensemble"
max_batch_size: 0
input [
  { name: "RAW" data_type: TYPE_FP32 dims: [ 8 ] }
]
output [
  { name: "RESULT" data_type: TYPE_FP32 dims: [ 2 ] }
]
ensemble_scheduling {
  step [
    {
      model_name: "preprocess_model"
      model_version: -1
      input_map { key: "INPUT" value: "RAW" }
      output_map { key: "OUTPUT" value: "PREPROCESSED" }
    },
    {
      model_name: "classifier_model"
      model_version: -1
      input_map { key: "INPUT" value: "PREPROCESSED" }
      output_map { key: "OUTPUT" value: "RESULT" }
    }
  ]
}
```

**Design illustration only.** Both component models must exist with compatible tensor contracts. `model_version: -1` may select a policy-dependent version; use explicit version governance for reproducibility. Verify exact protobuf schema for the pinned release. A postprocessing stage is omitted here for brevity.

## 9. Ensembles versus business workflows

An ensemble is a model execution graph, not a general-purpose durable workflow engine. It does not inherently provide business retries, human approvals, exactly-once processing, transactional state or cross-system compensation. Keep durable orchestration outside Triton.

## 10. Performance experiments

Design a controlled matrix:

| Test | Batch policy | Instance count | Cohort |
|---|---|---:|---|
| A | no dynamic batching | 1 | small/large inputs |
| B | dynamic batching, low delay | 1 | same input distribution |
| C | dynamic batching, higher delay | 1 | same distribution |
| D | chosen batching | 2 | same distribution |
| E | ensemble path | pinned components | end-to-end |

Hold model, backend, GPU, driver, request mix, concurrency and warmup consistent. Record throughput, p95/p99 queue, compute, E2E, GPU memory, errors and GPU-hours per valid completion. Abort on SLO burn, OOM or cost ceiling.

## 11. Triton metrics and tracing

Triton exposes version-dependent metrics for request counts, queue duration, compute input/infer/output and model execution. Inventory actual metric names, types, units and labels from the pinned exporter before writing PromQL. Use tracing to identify stage and queue bottlenecks; do not record sensitive input/output tensors in spans.

Monitor scrape freshness and model-version cardinality. GPU utilization alone cannot distinguish a healthy high-throughput batch from queue starvation.

## 12. Kubernetes scheduling and admission

Check:
- GPU device plugin and extended resources;
- node labels, taints/tolerations and quotas;
- per-Pod GPU allocation and NUMA/topology where relevant;
- startup/readiness of each ensemble component;
- model repository access and dependency versions;
- gateway rate limits and backpressure;
- rollout surge GPU capacity and rollback.

Kubernetes Pod scheduling and Triton model request scheduling are separate layers. Fix the correct scheduler.

## 13. Read-only inspection

**[SAFE-READ]** in an approved cluster:

```bash
date -u
kubectl config current-context
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

For a verified deployment:

```bash
kubectl -n MODEL_NAMESPACE describe deployment TRITON_DEPLOYMENT
kubectl -n MODEL_NAMESPACE logs deployment/TRITON_DEPLOYMENT --since=15m --tail=100
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Never dump Secrets or customer tensors.

## 14. Troubleshooting decision tree

```text
Triton latency/throughput degraded?
 ├─ high queue + low GPU → scheduler, instance availability
 ├─ high queue + high GPU → compute saturation/admission
 ├─ p99 worsens after batching → queue delay/shape mix
 ├─ GPU OOM after batching → batch workspace/instances
 ├─ sequence state wrong → correlation/slot lifecycle
 ├─ ensemble unready → missing component/dependency
 ├─ wrong ensemble output → tensor map/schema/version
 ├─ ensemble latency high → critical path per stage
 ├─ request timeout → queue policy/gateway/backend
 ├─ GPU idle after rollout → model not ready/routing
 ├─ telemetry flatline → scrape freshness/schema drift
 └─ GPU cost high → overprovisioned instances/bad batching
```

## 15. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | incompatible batch dimensions | tensor schema |
| F02 | dynamic batching unsupported by backend | backend config |
| F03 | max_batch_size mismatch | config/load |
| F04 | preferred batch sizes ineffective | batch distribution |
| F05 | excessive queue delay | queue p99 |
| F06 | queue policy rejects unexpectedly | reject counters |
| F07 | priority starvation | per-class latency |
| F08 | low-load latency regression | arrival rate |
| F09 | high-load queue saturation | queue depth |
| F10 | batch workspace GPU OOM | VRAM |
| F11 | too many model instances | instance memory |
| F12 | sequence correlation mismatch | sequence logs |
| F13 | sequence slot leak | active slots |
| F14 | sequence timeout too short | expiry |
| F15 | ensemble component missing | readiness |
| F16 | ensemble tensor map mismatch | schema |
| F17 | ensemble version drift | repository |
| F18 | ensemble critical path slow | stage traces |
| F19 | model backend crash | logs |
| F20 | gateway timeout before batch completes | gateway |
| F21 | GPU scheduling shortfall | Pod events |
| F22 | metrics schema drift | scrape |
| F23 | sensitive tensors in traces | privacy audit |
| F24 | GPU-hours per valid completion regress | FinOps |

## 16. Sixteen operational runbooks

1. Validate model/backend batching support.
2. Validate max_batch_size and tensor dimensions.
3. Diagnose queue delay and batch formation.
4. Diagnose priority and queue policy.
5. Investigate low-load p99 regression.
6. Investigate high-load queue saturation.
7. Recover batch workspace GPU OOM.
8. Tune instance groups with headroom.
9. Diagnose sequence identity and lifecycle.
10. Recover sequence slot exhaustion.
11. Diagnose ensemble component readiness.
12. Validate ensemble tensor mappings.
13. Diagnose ensemble version drift.
14. Trace ensemble critical path.
15. Recover gateway/telemetry/security regressions.
16. Execute rollback and verify SLO/FinOps.

## 17. Eight tabletop game days

1. Batching increases p99 during low traffic.
2. Queue delay causes gateway 504 errors.
3. Batch size change causes GPU OOM.
4. Sequence slots remain allocated after client failure.
5. Ensemble preprocessing version changes tensor shape.
6. A required ensemble component fails to load.
7. Priority policy starves low-priority tenants.
8. Rollback restores throughput but doubles GPU cost.

Default tabletop. Live fault injection requires separately approved isolated staging.

## 18. Twenty acceptance gates

G01 pinned backend; G02 batching support; G03 tensor dimensions; G04 max batch size; G05 queue policy; G06 batch distribution; G07 p95/p99 queue; G08 throughput; G09 GPU memory; G10 instance placement; G11 sequence lifecycle; G12 ensemble dependency readiness; G13 tensor mapping; G14 version pinning; G15 trace/privacy; G16 telemetry freshness; G17 rollback; G18 SLO; G19 GPU FinOps; G20 independent review.

See [companion lab](../labs/028-triton-scheduling-dynamic-batching-ensembles-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 029 — Triton GPU Metrics and Model Performance.**
