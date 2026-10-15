# 7.1 — vLLM Fundamentals and Serving Architecture

**Part:** 07 — vLLM & LLM Serving · **Chapter:** 1 of 7  
**Audience:** AI platform engineers, TrueFoundry administrators, GPU engineers, ML engineers and SREs  
**Safety:** `[SAFE-READ]` for observation; `[LAB-MUTATE]` only for approved isolated nonproduction changes; `[CHANGE-CONTROL]` for production changes.

## 1. Learning objectives

Explain the LLM inference lifecycle; identify the roles of vLLM, TrueFoundry, Kubernetes and GPU infrastructure; distinguish prefill from decode, throughput from time-to-first-token, and GPU memory from host RAM; trace an inference request; and use a layered method to diagnose startup, CUDA, routing and latency failures.

## 2. What vLLM is

**vLLM** is an open-source LLM inference and serving engine focused on efficient execution and request scheduling. Depending on release, model architecture and backend, it provides continuous batching, KV-cache management including PagedAttention, distributed inference options, and an OpenAI-compatible HTTP server. Feature and CLI availability are **version dependent**; always validate the exact image, engine version, model support matrix and flags before deployment.

**TrueFoundry** supplies the supported model deployment and platform workflow; **Kubernetes** schedules and operates Pods; the **NVIDIA driver/device plugin/container runtime** exposes GPUs; **vLLM** loads and executes the model. They are separate fault and ownership domains.

## 3. Reference architecture

```text
Authorized client / SDK
   ↓ DNS, TLS, authentication, quotas
Gateway / load balancer / ingress
   ↓ Service → Ready EndpointSlice
TrueFoundry-managed vLLM workload (Kubernetes Pod)
   ├─ API frontend / tokenizer
   ├─ request scheduler / admission / batching
   ├─ model executor / prefill + decode
   ├─ weights + activations + KV cache
   └─ CPU/GPU execution / distributed workers (when configured)
          ↑
Immutable model registry/object store + manifest + checksum
          ↑
Workload identity / approved cache or volume
          ↓
GPU node: driver → device plugin → container CUDA/framework
          ↓
Prometheus/OTel/logs + GPU telemetry → dashboards and alerts
```

The exact vLLM process architecture and supported deployment pattern depend on version and engine configuration. The diagram is conceptual, not a TrueFoundry resource schema.

## 4. LLM request lifecycle

1. Client submits an authenticated request with model name, prompt/messages, sampling parameters and limits.
2. Gateway validates TLS/auth and forwards to a Ready serving endpoint.
3. vLLM tokenizes or receives tokenized input, validates context and request limits.
4. Scheduler admits requests and forms work batches.
5. **Prefill** processes prompt tokens and populates attention KV cache.
6. **Decode** generates output tokens incrementally while using KV cache.
7. Runtime streams tokens or returns a complete response; request cancellation releases work.
8. Telemetry records TTFT, inter-token latency, total latency, token throughput, errors and queue wait.

For streaming, a successful initial HTTP status is not proof that the stream completed. Capture final completion and cancellation.

## 5. Prefill versus decode

| Property | Prefill | Decode |
|---|---|---|
| Work | Processes prompt/context tokens | Generates subsequent tokens |
| Main user signal | Time-to-first-token (TTFT) | Inter-token latency (ITL) |
| Compute profile | Often compute intensive | Often memory-bandwidth/KV-cache sensitive |
| Sensitivity | Prompt length, batch size, model | Active sequences, KV capacity, generation length |
| Failure pattern | Slow first token / prefill OOM | Slow token stream / KV exhaustion |

These are workload-dependent tendencies, not hard guarantees. Benchmark representative prompt/output lengths and concurrency.

## 6. PagedAttention and KV cache

Transformers reuse attention key/value tensors from earlier tokens. The KV cache consumes GPU (or supported tiered) memory and typically grows with active sequence length and concurrency. **PagedAttention** manages KV-cache storage in blocks to reduce fragmentation and improve allocation efficiency. It does not make VRAM unlimited.

Memory categories include model weights, runtime workspaces, activations, CUDA context, KV cache and allocator overhead. For a simplified decoder-only architecture, an approximate unquantized KV-cache requirement is:

```text
KV bytes ≈ 2 × layers × KV heads × head dimension
         × live tokens × bytes per element
```

This excludes allocator overhead, padding/block granularity, sharing, compression, sliding-window behavior and backend-specific details. Multi-query/grouped-query attention changes KV-head count. Verify architecture metadata and actual engine metrics rather than blindly applying a parameter-count formula.

## 7. Continuous batching

Traditional static batching waits for a batch to complete before new work enters. Continuous batching allows eligible new requests to join scheduling cycles as other requests finish or progress, improving GPU utilization under variable-length generation.

**Tradeoff:** increased throughput may raise queue wait or per-token latency. Track P95/P99, TTFT, ITL, batch occupancy, in-flight requests, waiting requests and rejection rate. Configure bounded concurrency and request limits; avoid unbounded queues and client retry storms.

## 8. Model loading and startup

```text
Pod scheduled
 → image pulled
 → model artifact reference resolved
 → identity/storage access
 → weights/config/tokenizer downloaded or cache hit
 → integrity and compatibility validated
 → CUDA/framework initialized
 → model weights allocated / distributed workers initialized
 → KV cache budget established
 → warmup / engine ready
 → Kubernetes readiness true
 → authorized golden request succeeds
```

A Running Pod is not necessarily model-ready. A GPU visible to `nvidia-smi` is not proof that vLLM can load the model. Readiness should remain false until engine initialization and required warmup complete.

## 9. Single-GPU and distributed serving

- **Single GPU:** simplest operational model when weights, KV cache and runtime overhead fit.
- **Tensor parallelism:** partitions supported model computation across multiple GPUs; requires compatible interconnect, topology and engine configuration.
- **Pipeline parallelism:** splits model layers across stages when supported; may trade throughput and latency for memory distribution.
- **Multi-node execution:** adds network, collective communication, worker lifecycle and failure domains.

Do not assume multiple GPUs automatically improve latency. Benchmark interconnect bandwidth, parallelism overhead, memory headroom and request distribution. Kubernetes GPU resources are integer-valued for conventional `nvidia.com/gpu` requests; sharing/MIG require explicit supported configuration.

## 10. Model and API compatibility

Validate model architecture, tokenizer, context length, weight format, precision/quantization, attention backend, CUDA/framework versions, GPU compute capability and license. An OpenAI-compatible endpoint may support only a subset of parameters and API features; use the installed engine's documentation and a contract test.

Pin the **image digest**, **vLLM version**, **model revision**, **tokenizer revision** and approved configuration. Avoid mutable `latest` references and untrusted remote model code.

## 11. Example deployment design record

This is an **evidence template**, not TrueFoundry YAML or a guaranteed vLLM command:

```yaml
service:
  workspace: "<approved-workspace>"
  cluster: "<approved-cluster>"
  namespace: "<approved-namespace>"
  image_digest: "sha256:<verified-digest>"
  vllm_version: "<tested-version>"
  model_id: "<approved-model>"
  model_revision: "<immutable-revision>"
  tokenizer_revision: "<immutable-revision>"
  gpu_type: "<supported-gpu>"
  gpu_count: 1
  tensor_parallel_size: 1
  context_limit: "<measured-safe-limit>"
  concurrency_limit: "<measured-safe-limit>"
  auth_policy: "<approved-policy>"
  rollback_release: "<known-good-release>"
```

Values are intentionally placeholders. In TrueFoundry, map them to **supported version-specific deployment fields** and inspect rendered Kubernetes resources.

## 12. Kubernetes and runtime inspection [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> logs <pod> --tail=200
kubectl -n <namespace> logs <pod> --previous --tail=200
kubectl describe node <gpu-node>
```

Where an **existing approved container** includes the utilities:

```bash
nvidia-smi
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available())"
python -c "import vllm; print(vllm.__version__)"
```

A missing utility is not definitive evidence of GPU failure. Do not install packages into production containers to diagnose. Never print environment secrets or inference payloads.

## 13. vLLM performance metrics

| Metric | What it reveals | Interpretation |
|---|---|---|
| TTFT P95/P99 | Prefill + queue + first output | Long prompts and waiting requests matter |
| ITL / time per output token | Decode experience | Sensitive to active sequences and KV use |
| End-to-end P95/P99 | Client latency | Includes network and gateway |
| Prompt/output tokens per second | Useful throughput | Keep workload mix comparable |
| Waiting/running requests | Admission pressure | Rising waiting queue predicts tail latency |
| KV cache usage | Available generation capacity | Near-capacity can trigger preemption/recompute |
| GPU utilization/VRAM | Hardware pressure | Not a direct correctness metric |
| Model load duration | Cold-start | Includes storage and initialization |
| Error, 429, 5xx, stream failure | Availability | Separate client versus server failures |

vLLM's exact metric names and labels change between releases. Discover the live `/metrics` endpoint **only where supported and authorized**; do not assume specific series exist. Avoid high-cardinality request IDs or prompt contents in metrics.

## 14. Failure matrix

| Symptom | First failed transition | Investigation |
|---|---|---|
| Pod Pending | GPU request → node | Quota, taints, capacity, affinity |
| GPU resource missing | Device plugin → allocatable | Driver/plugin/operator |
| No CUDA runtime | Container → framework | Image build and device allocation |
| Failed to infer device type | Framework → accelerator | Runtime/device detection |
| Model artifact 403 | Workload identity → storage | Scoped permissions/KMS |
| Model architecture unsupported | Model metadata → engine | Support matrix/version |
| Tokenizer mismatch | Artifact set → request parser | Revision and special tokens |
| Model OOM during load | Weights → VRAM | Weight footprint/parallelism |
| OOM under concurrency | KV cache → VRAM | Context, sequences, batch |
| Readiness never true | Engine init → probe | Startup time and logs |
| TTFT high, GPU idle | Queue/preprocessing → execution | Admission and CPU |
| ITL high, GPU saturated | Decode → device | Concurrency and memory bandwidth |
| 503/no ready endpoints | Pod readiness → Service | Probes and selectors |
| Stream terminates early | Runtime → gateway/client | Timeouts and cancellation |
| Wrong model response | Release → model revision | Immutable version check |
| Rollback fails after manual patch | Desired state → Pod | TrueFoundry/GitOps ownership |

## 15. Production incident runbook

1. **Detect:** affected endpoint, model revision, request count, 429/5xx, TTFT, ITL, P95/P99 and stream completion.
2. **Correlate:** TrueFoundry release, image digest, vLLM version, model/tokenizer revision, GPU node and last changes.
3. **Localize:** gateway → Service → Pod scheduling → GPU device → CUDA/framework → artifact → engine initialization → queue → prefill/decode → response.
4. **Contain:** freeze promotion, preserve healthy replicas, bound retries/concurrency and protect known-good artifacts.
5. **Repair:** address the first confirmed failed transition. Do not disable auth, bypass checksum validation or restart shared GPU nodes as a shortcut.
6. **Rollback:** restore approved image + model/tokenizer + vLLM config + resource/placement policy through the authoritative TrueFoundry/GitOps workflow.
7. **Validate:** actual loaded revision, golden synthetic request, readiness, TTFT/ITL, error rate, queue stability, GPU memory and streaming completion.
8. **Handoff:** incident timeline, lowest proven healthy layer, first failed transition, corrective actions and owners.

## 16. Acceptance gates

| Gate | Required evidence |
|---|---|
| G1 | TrueFoundry, Kubernetes, vLLM and GPU ownership mapped |
| G2 | Immutable image/model/tokenizer/runtime pinned |
| G3 | GPU and CUDA compatibility validated |
| G4 | Model lifecycle and readiness traced |
| G5 | KV cache and concurrency budget documented |
| G6 | Authorized API contract and golden request verified |
| G7 | TTFT/ITL, queue, GPU and errors observable |
| G8 | Authoritative rollback and post-recovery validation documented |

## 17. Evidence template

```text
Date/time / TrueFoundry workspace / deployment:
Image digest / vLLM version / model + tokenizer revision:
Node/GPU SKU / driver / CUDA / framework:
GPU allocation, CPU/RAM, VRAM:
Model fetch/load and readiness timing:
Context limit / parallelism / concurrency:
Endpoint/TLS/auth and golden request:
TTFT P95/P99 / ITL / token throughput:
Running/waiting requests and KV cache:
Lowest Proven Healthy Layer:
First Failed Transition:
Approved remediation / owner:
Known-good rollback and post-recovery SLO:
```

**Hands-on lab:** [7.1 — vLLM Architecture and GPU Readiness Check](../labs/01-vllm-fundamentals-serving-architecture-check.md)  
**Next:** Chapter 7.2 — vLLM Deployment with TrueFoundry and Kubernetes.

## References

- [vLLM Documentation](https://docs.vllm.ai/)
- [vLLM GitHub](https://github.com/vllm-project/vllm)
- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [PyTorch CUDA Semantics](https://docs.pytorch.org/docs/stable/notes/cuda.html)
- [NVIDIA CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
