# 7.7 — Production vLLM Deployment and Recovery Capstone

**Part 07 — vLLM & LLM Serving | Final capstone (7 of 7)**  
**Audience:** TrueFoundry platform engineers, SRE/DBRE, GPU infrastructure operators, AI serving owners  
**Execution:** Isolated approved staging; production is observation-only unless governed by a separate change.  
**Labels:** [SAFE-READ] read-only; [LAB-MUTATE] approved staging changes; [CHANGE-CONTROL] production approval.

## 1. Mission and success criteria

Build, deploy, measure, operate, break safely and recover a **version-pinned vLLM inference service on TrueFoundry and Kubernetes**. This capstone integrates Chapters 7.1–7.6: architecture, deployment, GPU/CUDA/PyTorch, artifact and tokenizer provenance, tensor parallelism, PagedAttention/KV memory, continuous batching, observability, performance testing and recovery.

**Success means** the approved model serves authorized synthetic inference, can be observed from client through GPU, survives a controlled staging failure, rolls back to an immutable known-good release, and meets **team-approved** reliability and latency SLOs. Publication of this document is not proof of lab execution or production certification.

### Objectives

1. Validate source-of-truth deployment and immutable release identity.
2. Prove GPU driver → device plugin → Pod allocation → CUDA → PyTorch → vLLM chain.
3. Validate model weights, tokenizer, chat template, storage identity and checksums.
4. Establish a measured VRAM/KV budget, bounded context, output and concurrency.
5. Measure TTFT, inter-token latency (ITL/TPOT), end-to-end latency and token throughput.
6. Exercise version-supported single-GPU and optional tensor-parallel topology.
7. Build release-aware dashboards and actionable SLO alerts.
8. Simulate controlled incidents **only in isolated staging**, then recover and verify.
9. Produce an audit-ready acceptance report with owners, evidence and open gaps.

## 2. Prerequisites and guardrails

- Chapters 7.1–7.6 completed or reviewed.
- An approved TrueFoundry staging workspace, cluster/namespace, service account and GPU allocation.
- A **vetted, license-approved model**, pinned model and tokenizer revisions, immutable vLLM image digest and verified artifact hashes.
- GPU quota, model registry/storage permissions, supported NVIDIA driver/container runtime, Prometheus/Grafana and a known-good rollback release.
- Synthetic requests only; no patient data, secrets, customer prompts or personal identifiers in logs.
- A rollback owner, maximum test duration, request-rate ceiling, token budget, VRAM stop threshold and incident communications channel.
- **Never** reboot shared GPU nodes, restart cluster-wide device plugins, change production networking, disable authentication, bypass workload identity or use `trust_remote_code` without explicit review.

TrueFoundry deployment specifications and vLLM flags vary by version. Use the installed version's supported deployment schema and CLI; examples below are read-only Kubernetes commands or conceptual release manifests, **not** claims about a universal TrueFoundry CLI.

## 3. Capstone reference architecture

```text
Authorized synthetic client
      │ HTTPS / bounded retry and timeout
      ▼
Auth / gateway / rate limits / request tracing
      ▼
TrueFoundry managed endpoint / Kubernetes Service
      ▼
Ready EndpointSlice → vLLM API Pod
                           │
            tokenizer + request validation
                           ▼
              admission / waiting queue
                           ▼
              prefill / decode scheduler
                           ▼
          GPU worker(s) / TP / NCCL if used
                           ▼
         weights + PagedAttention KV blocks
                           ▼
                streamed response
   ┌───────────────────────┴───────────────────────┐
   ▼                                               ▼
Prometheus/Grafana, vLLM metrics             Logs / traces
DCGM GPU telemetry, Kubernetes health        Release audit
```

**Ownership boundaries:** TrueFoundry/GitOps reconciles desired deployment; Kubernetes schedules and routes; NVIDIA stack exposes accelerators; PyTorch/vLLM initialize and execute; model registry/storage supplies artifacts; gateway governs external traffic. Every failed handoff must be isolated before remediation.

## 4. Immutable release contract

Use a release record like this **illustrative inventory** (not a TrueFoundry CRD):

```yaml
release:
  id: "<approved-staging-release>"
  workspace: "<workspace>"
  cluster: "<cluster>"
  namespace: "<namespace>"
  image_digest: "sha256:<approved-image-digest>"
  vllm_version: "<tested-version>"
  pytorch_cuda_stack: "<tested-build>"
  model_id: "<approved-model-id>"
  model_revision: "<immutable-commit>"
  tokenizer_revision: "<matching-immutable-commit>"
  model_manifest_sha256: "<verified-hash>"
  gpu_sku: "<validated-gpu>"
  gpu_count: 1
  tensor_parallel_size: 1
  pipeline_parallel_size: 1
  max_model_len: "<approved-bounded-value>"
  max_output_tokens: "<approved-bounded-value>"
  max_concurrency: "<tested-safe-value>"
  rollback_release: "<known-good-release>"
  owner: "<team>"
```

Do not deploy `latest` for image/model/tokenizer or use mutable model aliases as rollback evidence. Keep credentials out of the manifest.

## 5. Phase A — Read-only platform and GPU preflight [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get resourcequota,limitrange
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl describe node <approved-gpu-node>
kubectl -n <namespace> describe pod <existing-vllm-pod>
```

Record node GPU SKU, `nvidia.com/gpu` capacity/allocatable, Pod GPU request, quota, taints/tolerations, affinity and scheduling events. Check device-plugin/operator inventory using authorized cluster metadata, not an assumed namespace. A Running Pod does not prove GPU access.

If permitted on an **existing authorized** container:

```bash
kubectl -n <namespace> exec <pod> -c <container> -- nvidia-smi
kubectl -n <namespace> exec <pod> -c <container> -- python -c 'import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available(), torch.cuda.device_count())'
kubectl -n <namespace> exec <pod> -c <container> -- python -c 'import vllm; print(vllm.__version__)'
```

Missing tools in a minimal image are **Unknown**, not automatic GPU failure. Capture the complete sanitized error before concluding driver, CUDA or device inference is broken.

## 6. Phase B — Artifact integrity and model load [SAFE-READ / LAB-MUTATE]

Verify workload identity, storage/KMS authorization, model license and architecture, complete weight-shard inventory, immutable model and tokenizer revisions, special tokens and chat template. Prefer safe artifact formats and reject hash mismatches. Track cold-start stages:

| Stage | Evidence | Acceptance |
|---|---|---|
| Image pull | Digest, duration | Expected digest, no registry errors |
| Artifact fetch | Storage response, bytes/time | Authorized and complete |
| Integrity check | Manifest/hash | Verified immutable artifacts |
| Tokenizer/config | Revision and template | Model/API contract compatible |
| CUDA and workers | PyTorch/vLLM logs | GPU usable, supported backend |
| Weights | Load duration and VRAM | No host/GPU OOM |
| KV planning | Engine cache capacity | Bounded safe workload |
| Warmup | Model-aware readiness | Actual loaded revision correct |

For chat endpoints, verify the installed model supports the chosen chat template. Do not infer readiness solely from an HTTP port opening.

## 7. Phase C — GPU memory, KV and parallelism

Estimate unsharded logical KV memory for a decoder-only transformer:

```text
KV bytes ≈ 2 × layers × KV_heads × head_dim
         × active_tokens × bytes_per_KV_element
```

Measure model weights, CUDA context, workspaces, peak activation/loader memory, KV blocks and reserved headroom **per GPU**. Account for tensor parallel sharding, cache block rounding, prefix reuse and architecture-specific attention. Actual safe concurrency comes from measured engine capacity, not formula alone.

For optional TP>1: require allocated compatible GPUs, a supported model/vLLM release, rank mapping, NCCL connectivity, per-rank VRAM and an approved topology. Compare TP=1 versus TP>1 only when both are valid and fit. Never assume adding GPUs improves P99 latency.

## 8. Phase D — Controlled staging deployment [LAB-MUTATE]

1. Confirm workspace/namespace and change ticket, known-good rollback and GPU availability.
2. Publish/approve the immutable image/model/tokenizer release record.
3. Configure GPU resource requests, workload identity, secret references, network policy and supported vLLM settings through the **documented TrueFoundry interface or GitOps repository**.
4. Configure startup/readiness probes with realistic model-loading time and **model-aware** readiness semantics; avoid sending traffic to partially loaded engines.
5. Deploy to isolated staging; inspect reconciliation, Pod events and sanitized startup logs.
6. Verify assigned GPU devices, PyTorch CUDA and vLLM engine initialization.
7. Verify Ready EndpointSlices and the expected immutable model revision.
8. Run an authorized bounded synthetic golden inference and verify complete stream (if streaming).
9. Record actual image digest, model/tokenizer revision, GPU topology and release timestamp.

**Stop and roll back** for unsupported architecture, CUDA failure, missing artifact, repeated OOM, incorrect loaded model or unrecoverable readiness failure. Do not bypass auth or trust unreviewed model code.

## 9. Phase E — SLO baseline and synthetic load [LAB-MUTATE]

Define approved numerical targets **before** testing:

| SLI | Baseline | Target | Stop threshold | Measured |
|---|---|---|---|---|
| TTFT P95/P99 | | | | |
| ITL P95/P99 | | | | |
| End-to-end P95/P99 | | | | |
| Output tokens/s | | | | |
| 429 / 5xx / stream errors | | | | |
| Running/waiting requests | | | | |
| KV occupancy/preemptions | | | | |
| GPU VRAM/headroom | | | | |

Use a fixed synthetic workload with short/medium/long prompt bands, bounded output, low-to-moderate concurrency and an agreed test window. Warm the engine before steady-state comparison and separately measure cold-start behavior. Increase load only in approved increments; stop on error-budget breach, sustained queue growth, high KV pressure, VRAM exhaustion or shared-cluster impact. **Never run an unbounded production stress test.**

## 10. Phase F — Observability and alerts

Discover actual vLLM metric series, types, units and labels from the installed release. Dashboard panels:

1. TTFT, ITL, E2E P50/P95/P99 and release annotations.
2. Requests/s, 429/5xx, streaming failures and retry rates.
3. Prompt/output token throughput and request length bands.
4. Running/waiting requests, KV utilization and preemptions.
5. GPU VRAM, utilization and device faults via approved DCGM telemetry.
6. Pod Pending/restarts, Ready endpoints, model loading and gateway latency.

PromQL patterns are only valid **after** verifying counter/histogram names and buckets:

```promql
sum(rate(<validated_request_counter>[5m]))
histogram_quantile(0.95, sum by (le, model) (rate(<validated_ttft_histogram_bucket>[5m])))
```

Do not paste placeholder expressions into live dashboards. Exclude prompt text, authorization headers and request IDs from metric labels. Alert on sustained user-impacting conditions, with traffic floors, owner and runbook, rather than GPU utilization alone.

## 11. Phase G — Safe failure simulation and recovery [LAB-MUTATE]

Execute **only** on lab-owned isolated staging resources, with approved fault type and rollback ready. Choose at least three controlled scenarios, such as:

- **Artifact access failure:** revoke access only on a disposable lab-specific artifact path/identity, never shared IAM; observe 403 and restore identity.
- **Invalid tokenizer revision:** use an intentionally incompatible lab release, observe load/chat contract failure and restore pinned matching revision.
- **Resource pressure:** apply a *bounded* request mix or an intentionally undersized lab-only resource limit; stop at agreed thresholds.
- **Bad release:** deploy an approved lab-only image/config mismatch, capture startup/readiness failure and roll back.
- **Traffic/gateway failure tabletop:** analyze pre-captured sanitized events instead of disrupting shared routing.

Capture before/during/after timestamps, LPHL (Lowest Proven Healthy Layer), FFT (First Failed Transition), detection time, recovery time, actual model revision, TTFT/ITL, queue/KV and ready endpoint count. Do not claim tabletop scenarios were live-tested.

## 12. End-to-end failure matrix

| ID | Scenario | First failed transition | First evidence |
|---|---|---|---|
| C1 | GPU node absent | Infrastructure → node | Node inventory |
| C2 | No allocatable GPU | Driver/plugin → Kubernetes | Node describe |
| C3 | GPU Pod Pending | Scheduler → placement | Events, quota, taints |
| C4 | No CUDA runtime | Device/runtime → PyTorch | CUDA check |
| C5 | Failed to infer device type | PyTorch/device → vLLM | Engine traceback |
| C6 | Model artifact 403 | Identity → storage | Sanitized storage logs |
| C7 | Missing weight shard | Manifest → artifact | Shard inventory |
| C8 | Tokenizer/chat template mismatch | Tokenizer → prompt | Golden chat request |
| C9 | Unsupported model architecture | Config → engine | vLLM load logs |
| C10 | GPU OOM at model load | Weights → VRAM | Load logs, VRAM |
| C11 | KV OOM under load | Active tokens → cache | KV, preemptions |
| C12 | TP rank/NCCL timeout | Workers → collectives | Per-rank logs |
| C13 | Running Pod, not Ready | Engine → readiness | Probes, EndpointSlice |
| C14 | High TTFT | Queue/prefill → first token | Queue and TTFT |
| C15 | High ITL | Decode → token stream | ITL, VRAM |
| C16 | 429 retry storm | Client → admission | Gateway/queue |
| C17 | 503 no ready endpoints | Service → backend | EndpointSlice |
| C18 | Stream failure after 200 | Decode → client | Stream completion |
| C19 | Metrics/alert blind spot | Runtime → telemetry | Scrape/series |
| C20 | Upgrade regression | Release → engine | A/B baseline |
| C21 | Wrong model after rollback | Alias → artifact | Loaded revision |
| C22 | GPU fault/Xid | Device → executor | Approved DCGM/driver telemetry |
| C23 | Cold-start timeout | Load → probe budget | Startup timeline |
| C24 | Capacity shortfall during failover | Replica loss → admission | GPU quota, queue |

For each scenario record **symptom, LPHL, FFT, blast radius, owner, immediate containment, corrective action, rollback and post-recovery proof**. Not all scenarios require live injection.

## 13. Production recovery runbook [CHANGE-CONTROL]

1. **Declare and scope:** endpoint, affected tenants, onset, TTFT/ITL, 429/5xx, stream failures, error budget and current release.
2. **Preserve evidence:** immutable image/model/tokenizer, GPU SKU, CUDA/PyTorch/vLLM, scheduler settings, queue/KV, Pod events and sanitized logs.
3. **Localize:** client → gateway → Service → Pod → GPU/CUDA → artifact/tokenizer → engine → KV/scheduler → stream.
4. **Contain:** freeze rollout, retain healthy replicas, cap retries with backoff/jitter, apply approved admission limits; coordinate with GPU/node owners.
5. **Rollback:** use authoritative TrueFoundry/GitOps to restore **image digest + model/tokenizer revisions + GPU resources + TP/PP + scheduler/cache limits + probe contract**.
6. **Validate:** actual loaded model revision, Ready EndpointSlices, synthetic golden inference, full stream completion, GPU/KV headroom, queue drain, TTFT/ITL and error recovery.
7. **Close:** incident timeline, MTTR, detection gaps, RCA, preventive tests, owner signoff and residual risks.

Do not restart shared nodes, change driver/plugin fleet-wide, delete shared model caches or disable authentication as a shortcut.

## 14. Final acceptance gates

| Gate | Evidence | Owner |
|---|---|---|
| A1 | Approved staging scope, owners, stop conditions | Platform/SRE |
| A2 | Immutable model/image/tokenizer provenance and license | ML/security |
| A3 | Kubernetes GPU allocation, CUDA/PyTorch/vLLM compatibility | GPU/SRE |
| A4 | Artifact loading, model-aware readiness, golden inference | Serving |
| A5 | VRAM/KV budget and optional TP/NCCL topology | GPU/serving |
| A6 | TTFT/ITL/E2E/throughput baseline and SLO | Product/SRE |
| A7 | Dashboard, validated metric types, actionable alerts | Observability |
| A8 | At least 3 approved staging failure drills or explicitly noted not run | Incident lead |
| A9 | Immutable rollback and post-recovery correctness/SLO | Release owner |
| A10 | Evidence archive, risk register and owner signoffs | Platform lead |

**Completion rules:** A1–A10 must be marked Pass, Fail or Not run with links to evidence. Publication alone does **not** mark any live gate as passed. Record formal vendor compatibility review and SRE production approval separately.

## 15. Final evidence and handoff

```text
Date/time / change ticket / execution owner:
TrueFoundry workspace, cluster, namespace:
Image digest / vLLM / PyTorch / CUDA / driver:
Model ID and immutable revision / tokenizer revision:
Artifact checksums, license and storage identity:
GPU SKU/count / TP/PP / Pod allocation:
Weights, peak VRAM, KV capacity and safe concurrency:
Readiness, EndpointSlices, synthetic golden response:
TTFT/ITL/E2E P95/P99, prompt/output tokens/s:
Running/waiting, preemptions, 429/5xx/stream failures:
Dashboard/alert URLs and metric validation:
Failure drill IDs, LPHL/FFT, detection and recovery times:
Rollback release, actual loaded revision, post-recovery SLO:
A1–A10 results, unexecuted checks, risks and signoffs:
```

**Companion lab:** [7.7 — Production vLLM Deployment and Recovery Capstone Check](../labs/07-production-vllm-deployment-recovery-capstone-check.md)

**Part 07 publication complete:** Chapters 7.1–7.7 and seven companion labs. Formal vendor/SRE review and live lab execution remain pending until separately evidenced.

## References

- [vLLM Documentation](https://docs.vllm.ai/)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [PyTorch CUDA Semantics](https://docs.pytorch.org/docs/stable/notes/cuda.html)
- [NVIDIA NCCL Documentation](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [Prometheus Histograms](https://prometheus.io/docs/practices/histograms/)
