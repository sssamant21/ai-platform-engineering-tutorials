# Chapter 028 — Hands-On Lab: Triton Scheduling, Dynamic Batching and Ensembles

**Part 05 | Chapter 028 | Published lab plan; live execution pending**

## Goal

Design and evaluate Triton dynamic batching, sequence scheduling, model instance groups and ensemble dependencies; measure queue, throughput, tail latency, GPU memory and cost; define safe rollback and acceptance.

## Safety

**[SAFE-READ]** authorized configuration, Kubernetes/GPU metadata and sanitized metrics. **[LAB-MUTATE]** isolated staging with synthetic tensors, approved GPU-hours, concurrency and duration. **[CHANGE-CONTROL]** production scheduler, model config, instance groups, ensemble dependencies and gateway policies. No PHI, customer tensors or secret dumps.

## Lab 0 — Preflight

Record UTC, cluster/context, Triton image/backend versions, model revision/config, GPU SKU/UUID, model repository, queue policy, owners, cost ceiling, approvals and rollback.

```bash
date -u
kubectl config current-context
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -60
```

## Lab 1 — Dynamic batching contract [DESIGN]

Compare default scheduling with:

```protobuf
max_batch_size: 16
dynamic_batching {
  preferred_batch_size: [ 4, 8 ]
  max_queue_delay_microseconds: 1000
}
```

This is an illustrative fragment, not a production setting. Verify model batch dimension, backend support, input compatibility and version-specific config.

## Lab 2 — Synthetic experiment matrix [DESIGN]

| Variant | Batching | Instances | Measure |
|---|---|---:|---|
| A | disabled | 1 | baseline |
| B | low-delay | 1 | p95/p99 queue |
| C | higher-delay | 1 | throughput/p99 |
| D | selected | 2 | VRAM/throughput |
| E | ensemble | pinned | critical path |

Keep hardware, model, warmup, input shapes, concurrency and duration constant. Record GPU-hours per valid completion. Mark unexecuted results Pending.

## Lab 3 — Sequence scheduling [DESIGN]

Define correlation ID, start/end controls, idle timeout, state affinity and slot reclamation for a compatible stateful model. Explain why this is not the same as an LLM streaming API.

## Lab 4 — Ensemble mapping [DESIGN]

Draw preprocess → classifier → postprocess with named input/output tensors. Record exact tensor dtypes/dims, component version, model readiness and failure handling. Do not use implicit version selection for a reproducible production release without governance.

## Lab 5 — GPU and queue evidence [SAFE-READ]

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Compare per-GPU peak VRAM, queue time, batch size distribution, compute duration and model instance count. Verify actual metric schema and freshness before querying Prometheus.

## Lab 6 — Rollback and SLO [DESIGN]

Define abort criteria for p99, queue, errors, GPU OOM, starvation, model readiness and GPU cost. Roll back one variable at a time through authoritative GitOps/model release controls. Validate full response correctness after rollback.

## Lab 7 — Eight tabletop game days

01 low-traffic p99; 02 queue timeout; 03 batch OOM; 04 sequence slot leak; 05 tensor shape drift; 06 ensemble component missing; 07 priority starvation; 08 cost regression after rollback.

## Twenty-four failure checklist

F01 batch dims; F02 backend; F03 max batch; F04 preferred sizes; F05 queue delay; F06 rejects; F07 priority; F08 low-load p99; F09 high-load queue; F10 batch OOM; F11 instances; F12 sequence ID; F13 slot leak; F14 timeout; F15 component missing; F16 tensor map; F17 version drift; F18 critical path; F19 backend crash; F20 gateway timeout; F21 GPU scheduling; F22 metrics; F23 privacy; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Pinned backend | Pending |
| G02 | Batching support | Pending |
| G03 | Tensor dimensions | Pending |
| G04 | Max batch size | Pending |
| G05 | Queue policy | Pending |
| G06 | Batch distribution | Pending |
| G07 | Queue p95/p99 | Pending |
| G08 | Throughput | Pending |
| G09 | GPU memory | Pending |
| G10 | Instance placement | Pending |
| G11 | Sequence lifecycle | Pending |
| G12 | Ensemble readiness | Pending |
| G13 | Tensor mapping | Pending |
| G14 | Version pinning | Pending |
| G15 | Trace/privacy | Pending |
| G16 | Telemetry freshness | Pending |
| G17 | Rollback | Pending |
| G18 | SLO | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence template

```text
UTC/context/approval:
Triton/backend/model/config:
Scheduler/batch policy:
Sequence lifecycle:
Ensemble dependency map:
Queue/compute/throughput:
p95/p99/E2E:
GPU memory/instance count:
Privacy/telemetry freshness:
Game-day mode/findings:
Rollback/GPU-hours/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Stop approved synthetic load, restore model config via authoritative controller, remove only owned staging resources and reconcile GPU-hours. Acceptance remains pending until independently reviewed.

**Next: Chapter 029 — Triton GPU Metrics and Model Performance.**
