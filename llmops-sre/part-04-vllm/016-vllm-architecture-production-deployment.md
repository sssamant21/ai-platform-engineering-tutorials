# Chapter 016 — vLLM Architecture and Production Deployment

**Part 04 — vLLM Administration and Operations | Chapter 016 of 100**

> **Publication:** Written tutorial. **Not** technically/security reviewed, executed in a live cluster, or operationally accepted. vLLM APIs and flags change by release: pin a version and validate its official documentation before deployment.

## 1. Learning objectives

Explain vLLM's API server, engine, request scheduler, continuous batching, KV-cache management and GPU workers; design a secure Kubernetes deployment; distinguish model readiness from Pod readiness; define SLOs and observability; plan safe canary, rollback, scaling, and incident response.

## 2. Production architecture

```text
Synthetic client / application
    │ TLS, auth, tenant quota, trace ID
    ▼
API Gateway / Ingress / WAF
    │ timeout, retry, cancellation, stream flush
    ▼
Kubernetes Service → vLLM serving Pods
                       ├─ HTTP/OpenAI-compatible API server
                       ├─ tokenizer/chat template
                       ├─ engine + admission/scheduler
                       ├─ prefill/decode continuous batching
                       ├─ paged KV-cache blocks
                       ├─ GPU worker(s), TP/PP where supported
                       └─ SSE output + terminal completion
    │
    ├─ model registry / approved model artifact storage
    ├─ workload identity / restricted egress
    ├─ Prometheus / OTel / GPU DCGM telemetry
    └─ GitOps / immutable image / controlled rollout
```

The architecture and exact internal process boundaries depend on the vLLM release. The API server may be separate from engine/workers. GPU node health does not prove model readiness or complete-stream success.

## 3. vLLM runtime concepts

- **Continuous batching:** schedules work as requests arrive rather than waiting for a static batch; improves utilization but introduces fairness, queue and KV-capacity tradeoffs.
- **PagedAttention / paged KV management:** manages KV-cache blocks to reduce allocation waste; actual implementation and prefix-sharing behavior depend on version/configuration.
- **Prefill versus decode:** long prompts stress prefill and initial KV; long outputs stress iterative decode, memory bandwidth and resident cache.
- **Tensor/pipeline parallelism:** can partition model across GPUs; requires compatible topology, rank health and runtime flags.
- **OpenAI-compatible API:** compatibility is endpoint/model/version-specific. Validate supported request fields, streaming event semantics, errors and finish behavior rather than assuming full parity.
- **Model loading:** readiness requires tokenizer, weights, GPU allocation, warmup and API health; a running process is not enough.

## 4. Deployment prerequisites and approval

Before creating a staging deployment, record:
1. Kubernetes context, namespace, GPU SKU/count and device plugin status.
2. Approved model ID, exact revision, license, tokenizer and chat template.
3. Signed/scanned serving image digest, pinned vLLM version and supported CUDA/driver stack.
4. Model source, immutable artifact checksum, restricted access and caching policy.
5. GPU VRAM budget: weights, KV, workspace, CUDA graphs, fragmentation and headroom.
6. Runtime config for context, maximum active sequences, token budget and parallelism.
7. Ingress authentication, TLS, quotas, rate limits and safe logging.
8. Health probes, startup budget, disruption strategy, cost ceiling, owner and rollback authority.

**[SAFE-READ]** inspection; **[LAB-MUTATE]** approved staging deployment; **[CHANGE-CONTROL]** production model, GPU, routing, ingress, runtime or node changes.

## 5. Read-only preflight

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods,deploy,svc -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -80
kubectl describe nodes
```

On an authorized GPU host or approved troubleshooting Pod:

```bash
nvidia-smi
nvidia-smi --query-gpu=index,name,uuid,driver_version,memory.total,memory.used --format=csv
```

Never install GPU drivers, NVIDIA toolkit, operators or privileged debug Pods as a read-only step.

## 6. Illustrative Kubernetes deployment design

This is a **review template, not a directly deployable production manifest**. Substitute approved image/model/namespace, verify runtime flags and health endpoints for your pinned vLLM release, and ensure the cluster supports the GPU resource name.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-staging
  namespace: APPROVED_NAMESPACE
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: vllm-staging
  template:
    metadata:
      labels:
        app: vllm-staging
    spec:
      terminationGracePeriodSeconds: 180
      containers:
        - name: vllm
          image: APPROVED_IMAGE@sha256:APPROVED_DIGEST
          args:
            - "--model"
            - "APPROVED_MODEL_ID_OR_LOCAL_PATH"
          ports:
            - name: http
              containerPort: 8000
          resources:
            requests:
              nvidia.com/gpu: "1"
              cpu: "4"
              memory: 16Gi
            limits:
              nvidia.com/gpu: "1"
              memory: 32Gi
          startupProbe:
            httpGet:
              path: /health
              port: http
            periodSeconds: 10
            failureThreshold: 90
          readinessProbe:
            httpGet:
              path: /health
              port: http
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: http
            periodSeconds: 30
            failureThreshold: 3
```

**Cautions:** image and digest are placeholders; the /health endpoint must be validated for the selected release; probes alone may not validate model readiness. A production design also needs model volume/auth, pod/node security, scheduling, egress, secrets, topology, service routing, autoscaling, shutdown/drain and resource limits appropriate to the actual model. `Recreate` can cause downtime; use an approved blue/green strategy and sufficient GPU capacity for availability.

## 7. Example synthetic API validation

Use only an approved staging endpoint and a non-sensitive synthetic prompt. Exact endpoint, model alias, authentication and streaming contract depend on the deployment. This is an **illustrative API shape**, not an assertion that an endpoint exists:

```bash
curl --fail-with-body --no-buffer --max-time 120 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $STAGING_API_TOKEN" \
  -d '{"model":"APPROVED_MODEL_ALIAS","messages":[{"role":"user","content":"Return exactly: READY"}],"max_tokens":32,"stream":true}' \
  "$STAGING_BASE_URL/v1/chat/completions"
```

Do not print or commit tokens. A successful HTTP 200 is insufficient: validate SSE framing, generated tokens, finish reason, final event and clean connection close. The `--fail-with-body` flag requires a compatible curl version; use an equivalent supported option if unavailable.

## 8. Runtime configuration decision matrix

| Setting family | Why it matters | Required evidence |
|---|---|---|
| Model and tokenizer revision | correct prompts and weights | immutable provenance |
| Context length | admission and KV size | token arithmetic |
| GPU memory utilization target | KV/cache headroom | measured VRAM |
| Max concurrent sequences | throughput vs p99 and OOM | queue/cache/load test |
| Batched token budget | prefill/decode scheduling | TTFT/TPOT cohorts |
| Quantization | model fit and performance | quality and latency evaluation |
| TP/PP | multi-GPU model fit | topology, rank and NCCL |
| Prefix caching | KV reuse and tenant isolation | privacy/efficiency review |
| Tool/chat template support | structured output correctness | synthetic protocol tests |
| API concurrency/limits | overload safety | 429 and fairness |
| Graceful shutdown | streaming drain | rollout and cancellation tests |

Use the exact CLI flags documented for the pinned release; do not cargo-cult settings across vLLM versions.

## 9. Observability and SLOs

**Customer path:** eligible requests, valid complete SSE success, TTFT, TPOT/ITL, E2E latency, 5xx/429 categories, cancellations, tenant fairness.

**Runtime:** queue depth/wait, running/waiting requests, prefill/decode timing, KV-cache usage/preemption, output tokens/s, model loading, runtime errors.

**GPU/platform:** memory, SM/HBM activity, XID/ECC, thermal/power, CUDA errors, GPU allocatable, Pod restarts and DCGM freshness.

Metric names and labels vary by vLLM release. Inspect the deployed metrics endpoint and exporter version before authoring PromQL. Missing telemetry is unknown, not a healthy zero.

## 10. Deployment rollout and rollback

1. Freeze model, tokenizer, serving image digest and configuration.
2. Approve change window, GPU headroom, failure budget and owner.
3. Validate GPU scheduling, model artifact access and CUDA compatibility.
4. Deploy in isolated staging, allow model load and bounded warmup.
5. Run synthetic non-streaming and full-stream protocol checks.
6. Measure TTFT/TPOT, cache, GPU VRAM, 429/5xx and cost.
7. Verify authentication, quotas, network policy and log redaction.
8. Canary with explicit stop thresholds only after change approval.
9. Roll back via the authoritative GitOps/deployment controller if gates fail.
10. Verify sustained customer-path recovery and no orphan GPU capacity.

Never use a blind rolling update if spare GPUs cannot run old and new replicas concurrently; account for scheduling and model warmup.

## 11. Troubleshooting decision tree

```text
vLLM unavailable or slow?
 ├─ Pod Pending → GPU allocatable, quota, affinity, taints
 ├─ ImagePullBackOff → registry/identity/image digest
 ├─ model load failure → artifact auth, revision, tokenizer, disk
 ├─ CUDA init failure → driver/runtime/toolkit
 ├─ CUDA OOM → weights, KV, context, concurrency
 ├─ not ready → startup/model load/health contract
 ├─ HTTP 401/403 → gateway auth and identity
 ├─ HTTP 429 → tenant quota vs overloaded admission
 ├─ HTTP 502/503/504 → endpoints, readiness, gateway deadlines
 ├─ high TTFT → queue, prefill, long prompts
 ├─ high TPOT → decode, HBM, KV/preemption
 ├─ 200 partial SSE → proxy timeout, cancellation, process exit
 ├─ NCCL timeout → TP/PP rank, fabric, topology
 ├─ high cost → idle GPUs, retries, long output caps
 └─ metrics missing → exporter freshness, independent synthetic
```

## 12. Twenty-four production failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | insufficient GPU capacity | scheduler events |
| F02 | incompatible driver/CUDA | runtime error |
| F03 | image pull failure | registry/Pod events |
| F04 | model artifact access denied | loader logs |
| F05 | tokenizer/chat template mismatch | model metadata |
| F06 | context limit misconfigured | token validation |
| F07 | model load OOM | GPU VRAM |
| F08 | KV-cache OOM under load | cache/preemption |
| F09 | long cold start | startup timing |
| F10 | readiness green before model ready | synthetic probe |
| F11 | probe restarts healthy slow loader | startup config |
| F12 | gateway auth failure | 401/403 |
| F13 | quota vs overload 429 confusion | admission counters |
| F14 | queue saturation | waiting requests |
| F15 | prefill TTFT regression | prompt cohorts |
| F16 | decode TPOT regression | tokens/s, HBM |
| F17 | proxy buffers SSE | first-token timings |
| F18 | HTTP 200 incomplete SSE | completion validator |
| F19 | client cancellation leak | runtime active requests |
| F20 | rollout kills active streams | termination/drain |
| F21 | TP rank/NCCL failure | rank logs |
| F22 | metric name drift | exporter inventory |
| F23 | sensitive prompt logging | privacy audit |
| F24 | GPU cost spike | GPU-hours/good requests |

## 13. Sixteen operational runbooks

1. Validate context, ownership and model release.
2. Triage pending GPU Pod and node scheduling.
3. Diagnose image and model artifact access.
4. Validate CUDA/runtime compatibility.
5. Diagnose model load and cold start.
6. Validate tokenizer/chat template/context.
7. Investigate CUDA OOM and KV pressure.
8. Investigate queue and TTFT.
9. Investigate decode/TPOT.
10. Validate API authentication and quota.
11. Diagnose SSE buffering/partial streams.
12. Validate cancellation and graceful shutdown.
13. Investigate TP/NCCL failures.
14. Restore observability and GPU cost controls.
15. Execute approved rollout rollback.
16. Validate SLO recovery and complete RCA/COA.

Each runbook requires owner, safe checks, impact, mitigation approval, rollback and customer-path validation.

## 14. Eight game days

1. Model load fails because of GPU VRAM.
2. Wrong tokenizer revision changes admission.
3. Readiness becomes green before model loads.
4. Queue saturation causes TTFT SLO burn.
5. Proxy buffers SSE and breaks streaming.
6. GPU rank crashes during TP decode.
7. Deployment rollout interrupts active streams.
8. GPU costs rise despite healthy HTTP status.

Default **tabletop**; staging execution requires separate approval and cost limits.

## 15. Eighteen acceptance gates

G01 approved architecture; G02 context and GPU preflight; G03 pinned image/model/tokenizer; G04 CUDA compatibility; G05 model artifact security; G06 startup/readiness; G07 safe manifest review; G08 context/KV sizing; G09 bounded synthetic API; G10 full SSE completion; G11 TTFT/TPOT SLO; G12 queue/fairness; G13 GPU/DCGM; G14 auth/quota/privacy; G15 multi-GPU/NCCL; G16 rollout/rollback; G17 FinOps; G18 independent reviewer.

See [companion lab](../labs/016-vllm-architecture-production-deployment-check.md). All gates remain pending until actual evidence and review.

**Next: Chapter 017 — vLLM Model Loading and Runtime Configuration.**
