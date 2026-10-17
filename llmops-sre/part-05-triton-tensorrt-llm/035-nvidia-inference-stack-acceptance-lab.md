# Chapter 035 — NVIDIA Inference Stack Acceptance Lab

**Part 05 — NVIDIA Triton and TensorRT-LLM | Chapter 035 of 100 | Written capstone**

> **Publication status:** Curriculum and lab plan published; **no live acceptance tests have been executed**. All gates, reviews and sign-offs are pending. Execute only in an approved staging environment with bounded GPU-hours and synthetic/approved data.

## Mission

Validate a complete NVIDIA inference service—from authenticated client through gateway, Kubernetes, Triton, TensorRT-LLM backend, model artifacts, GPU and observability—and establish evidence that the stack is secure, reliable, performant, recoverable and cost-controlled.

This capstone integrates Chapters 026–034. Passing documentation checks is **not** production readiness. Production promotion requires independent technical/security approval and measured live evidence.

## 1. Reference topology

```text
Approved synthetic load generator
             │
      TLS/Auth/Quota gateway
             │
    Kubernetes ClusterIP Service
             │
    Triton replicas on GPU nodes
             │
   TensorRT-LLM executor/backend
      ├─ pinned model + tokenizer
      ├─ quantization/engine bundle
      ├─ prefill/decode + KV cache
      ├─ scheduling/batching
      └─ TP/PP + NCCL (if used)
             │
 NVIDIA driver/CUDA/GPU device plugin
             │
 Prometheus/DCGM/client histograms
             │
  SLO alerts / on-call / FinOps
```

Record exact product versions, GPU model, topology, model license, artifact digests and all deviations from the reference.

## 2. Acceptance scope and boundaries

In scope: model repository, image/backend compatibility, GPU scheduling, readiness, inference correctness, streaming, latency, throughput, KV/VRAM, security, rollback, node fault tolerance, observability, cost and operational response.

Out of scope unless separately approved: production fault injection, live tenant data, unauthorized model-control APIs, destructive GPU/node tests, external benchmark publication and claims of universal performance.

## 3. Immutable release bill of materials

```yaml
release_id: APPROVED_STAGING_RELEASE
triton_image_digest: PINNED
triton_version: PINNED
tensorrt_llm_version: PINNED
tensorrt_version: PINNED
cuda_driver_matrix: VERIFIED
gpu_sku_and_compute_capability: VERIFIED
model_revision_and_sha256: PINNED
tokenizer_revision_and_sha256: PINNED
quantization_scales_digest: PINNED_IF_APPLICABLE
engine_artifact_digest: PINNED_IF_APPLICABLE
model_repository_revision: PINNED
backend_config_digest: PINNED
gateway_policy_revision: PINNED
rollback_bundle: TESTED_KNOWN_GOOD
owner: NAMED
reviewer: NAMED
```

This is a worksheet, not a deployable configuration. Engine portability, precision support and model-control behavior vary by pinned release.

## 4. Environment readiness

Before live testing, verify:
- approved cluster context, namespace, change record and time window;
- GPU device plugin, node allocatable GPU and ResourceQuota;
- driver/CUDA/backend compatibility;
- model and tokenizer license/provenance;
- GPU memory and quota headroom;
- read-only model repository or controlled immutable artifact sync;
- least-privilege ServiceAccount, NetworkPolicy, TLS/auth gateway;
- SLO, alert routes, on-call and rollback owners;
- synthetic dataset, rate cap, duration and GPU-hour ceiling.

## 5. Safe-read preflight

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

For authorized namespace:

```bash
kubectl -n MODEL_NAMESPACE get pods -o wide
kubectl -n MODEL_NAMESPACE describe deployment APPROVED_DEPLOYMENT
kubectl -n MODEL_NAMESPACE get endpointslices
kubectl -n MODEL_NAMESPACE logs deployment/APPROVED_DEPLOYMENT --since=15m --tail=100
```

On an authorized GPU host:

```bash
nvidia-smi
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,uuid,memory.total,memory.used,utilization.gpu --format=csv
```

Redact logs and never print Secrets or customer prompts.

## 6. Acceptance test matrix

| Phase | Objective | Evidence |
|---|---|---|
| A | pinned build and artifact provenance | digests and approvals |
| B | GPU scheduling and storage | node/Pod events |
| C | server/model readiness | health and golden inference |
| D | auth/TLS/admin isolation | security test report |
| E | short/long context correctness | fixture report |
| F | complete streaming/cancellation | terminal validator |
| G | TTFT/TPOT/E2E | client histograms |
| H | queue/batching/KV/VRAM | backend + GPU telemetry |
| I | failure response and HA | tabletop or approved staging evidence |
| J | rollback, observability and FinOps | recovery and cost worksheet |

Every test needs owner, approved pass threshold, time window, actual result, evidence link, reviewer and pass/fail decision. No invented results.

## 7. Model correctness and output contract

Validate synthetic golden prompts across short/long context, stop/EOS, JSON/structured output, numerical behavior, multilingual cases where applicable, safety behavior, timeout/cancellation and streaming terminal completion. Confirm model/tokenizer/template versions. A healthy process or HTTP 200 alone is insufficient.

## 8. Latency and throughput methodology

Collect client-side TTFT, TPOT/ITL (with explicit denominator), E2E p50/p95/p99, valid completion rate, requests/s, output tokens/s, queue and compute durations, GPU VRAM/utilization and GPU-hours per valid completion.

Run reproducible short/short, long/short, short/long, long/long, mixed-concurrency and burst cohorts. Keep baseline and candidate model/GPU/runtime consistent. Include warmup and cooldown. Triton cumulative microsecond duration counters do not directly yield p99.

## 9. KV and GPU memory

Approximate conventional logical KV bytes/token:

```text
2 × layers × KV_heads × head_dim × bytes_per_element
```

Account for weights, activation/workspace, CUDA graphs, allocator blocks, fragmentation, TP/PP buffers and safety headroom. Verify actual per-GPU peak VRAM during approved load; theoretical capacity is not an admission limit.

## 10. Failure and recovery objectives

Document target RTO/RPO where meaningful, but inference serving is typically evaluated by time to restore valid completions, availability and model artifact recovery rather than database-style RPO alone.

Exercise:
- Pod crash and restart behavior;
- model unready after rollout;
- GPU node loss/failover (tabletop unless approved);
- engine incompatibility;
- KV exhaustion;
- streaming interruption;
- metrics/alerting failure;
- complete-release rollback.

Record time-to-detect, time-to-mitigate, time-to-restore and error-budget impact.

## 11. Security and privacy

Test only authorized controls: gateway TLS/auth, role separation, internal-only metrics/model control, namespace policy, model registry access, artifact provenance and privacy-safe telemetry. No cross-tenant probing or customer data. Security incidents require separate escalation and evidence handling.

## 12. GPU FinOps

Record build GPU-hours, serving GPU-hours, reserved versus used capacity, peak VRAM, model load time, failed/retried work, valid completion count and cost per valid completion. Evaluate whether higher throughput improves SLO-qualified goodput and actual cost.

## 13. Promotion and rollback

```text
Evidence complete?
   ├─ No → HOLD; assign remediation
   └─ Yes → independent technical/security review
             ├─ Fail → HOLD / remediate
             └─ Pass → approved canary
                       ├─ SLO or quality regression → full rollback
                       └─ stable observation → promotion approval
```

Restore the complete tested unit: Triton image, TensorRT-LLM/backend, engine, quantization artifacts, model/tokenizer, config and gateway routing. Kubernetes rollout undo alone may not restore external artifacts.

## 14. Twenty-four failure scenarios

| ID | Scenario | Acceptance evidence |
|---|---|---|
| F01 | GPU device plugin absent | scheduling events |
| F02 | Pod Pending due to quota/taint | node/quota |
| F03 | image registry failure | pull events |
| F04 | model PVC/artifact unavailable | mount/load |
| F05 | model revision checksum drift | digest |
| F06 | engine/GPU incompatibility | matrix/load |
| F07 | driver/CUDA/backend mismatch | startup |
| F08 | quantization quality regression | golden fixtures |
| F09 | tokenizer/chat template mismatch | output contract |
| F10 | startup GPU OOM | VRAM/logs |
| F11 | KV cache exhaustion | allocator |
| F12 | dynamic batching p99 regression | cohort latency |
| F13 | scheduler queue overload | queue/admission |
| F14 | request retry storm | gateway |
| F15 | partial streaming response | terminal validator |
| F16 | cancellation fails to release resources | active slots |
| F17 | TP/PP rank mismatch | rank topology |
| F18 | NCCL timeout | distributed logs |
| F19 | GPU node/hardware failure | device/Pod |
| F20 | NetworkPolicy blocks gateway | endpoints/CNI |
| F21 | model-control/metrics exposed | security review |
| F22 | metrics/alerts stale | scrape/alert |
| F23 | incomplete rollback artifact bundle | digest/fixtures |
| F24 | GPU cost per valid completion spikes | FinOps |

## 15. Sixteen operational runbooks

1. Acceptance preflight and approvals.
2. GPU device-plugin and scheduling triage.
3. Registry, PVC and model provenance recovery.
4. Triton startup/server/model readiness.
5. Engine/CUDA/GPU compatibility.
6. Tokenizer, quantization and output quality.
7. Startup GPU OOM and model placement.
8. KV cache and decode OOM.
9. Dynamic batching and queue saturation.
10. Gateway retry/admission and timeout.
11. Streaming terminal completion/cancellation.
12. TP/PP and NCCL recovery.
13. GPU node fault and HA recovery.
14. Network/security exposure containment.
15. Telemetry, SLO and GPU FinOps recovery.
16. Full-stack rollback, RCA and sign-off.

## 16. Eight game days

1. GPU device plugin disappears.
2. Model repository is inaccessible at startup.
3. Candidate engine cannot load on serving GPU.
4. Long-context load exhausts KV.
5. Batch optimization violates TTFT p99.
6. One TP rank fails during streaming.
7. Model-control endpoint is inadvertently exposed.
8. Rollback restores Pods but not model/engine bundle.

Default tabletop. Live injection only in separately approved staging.

## 17. Twenty measurable acceptance gates

| Gate | Requirement | Status |
|---|---|---|
| G01 | Change approval and scope | Pending |
| G02 | Immutable release bill of materials | Pending |
| G03 | Model/license/provenance | Pending |
| G04 | GPU/runtime compatibility | Pending |
| G05 | Kubernetes scheduling/storage | Pending |
| G06 | Triton process and model readiness | Pending |
| G07 | Golden correctness/quality | Pending |
| G08 | Streaming terminal completion | Pending |
| G09 | TLS/auth/admin isolation | Pending |
| G10 | NetworkPolicy/privacy | Pending |
| G11 | TTFT/TPOT/E2E targets | Pending |
| G12 | Throughput/goodput | Pending |
| G13 | Queue/batching fairness | Pending |
| G14 | KV/VRAM headroom | Pending |
| G15 | TP/PP/NCCL health | Pending |
| G16 | Failure response and HA | Pending |
| G17 | Full-stack rollback drill | Pending |
| G18 | Telemetry/SLO alerting | Pending |
| G19 | GPU FinOps/cost ceiling | Pending |
| G20 | Independent technical/security acceptance | Pending |

**Acceptance decision:** HOLD until all mandatory gates have evidence and authorized reviewers sign off.

## 18. Final evidence package

```text
Project/change ID, owner and UTC:
Cluster/namespace/GPU and cost ceiling:
Triton/backend/engine/model/tokenizer digests:
Security and privacy approval:
Synthetic fixture manifest:
Latency/throughput/streaming results:
Queue/KV/VRAM/GPU metrics:
Failure and rollback results:
SLO/error budget and alert validation:
GPU-hours/cost per valid completion:
G01–G20 evidence URLs and reviewer:
Exceptions, remediation owners and due dates:
Final decision: HOLD / APPROVED
```

See [companion execution worksheet](../labs/035-nvidia-inference-stack-acceptance-lab-check.md). No gates have been executed or passed by publishing this chapter.

**Part 05 written curriculum complete: Chapters 026–035. Next: Chapter 036 — Ray Serve Architecture and Deployment.**
