# Chapter 002 — AI Workload Taxonomy: Training, Fine-Tuning and Inference

**Part 01 — Foundations and Architecture · Chapter 002 of 100**  
**Status:** Published documentation and offline executable lab. Runtime tests, cloud/GPU experiments and independent technical/security review pending.

## Learning outcomes
By the end of this chapter, an AI Infrastructure SRE should be able to:
1. Classify pretraining, full fine-tuning, PEFT/LoRA, batch inference, online inference and RAG/agent workloads by resource profile and reliability requirements.
2. Translate workload SLOs into GPU, CPU, memory, network, storage, scheduler and quota requirements.
3. Explain prefill versus decode, model memory versus KV-cache pressure, optimizer states, gradient synchronization and checkpoint recovery.
4. Design safe scheduling, monitoring, incident response and FinOps controls for each workload.
5. Run a deterministic Python lab that rejects unsupported or unsafe workload placement decisions.

## 1. Why workload classification matters
AI jobs are not interchangeable GPU consumers. A low-latency streaming inference service is sensitive to **TTFT**, per-token decode speed, queueing and uninterrupted streams. Distributed training is sensitive to all-reduce performance, checkpoint consistency, dataset I/O and time lost after worker failure. Fine-tuning may need less total compute but still experience optimizer-memory pressure and artifact provenance risks.

```text
                 AI Platform Workload Intake
                            |
              classify SLA + data + model
               /            |           \
         Training       Fine-tuning      Inference
         /      \        /       \       /        \
    single GPU multi GPU  full    LoRA  online     batch
        |         |         |       |      |         |
    data/checkpoint     memory/adapter    TTFT/TPOT  throughput
        |         |         |       |      |         |
       storage + network + GPU + scheduler + identity
                            |
              telemetry / quota / recovery
```

## 2. Taxonomy and production characteristics
| Workload | Main success metric | GPU bottleneck | Other dependencies | Recovery strategy |
|---|---|---|---|---|
| Pretraining | useful tokens/s and training stability | compute + interconnect | dataset, RDMA, checkpoints | restart from consistent checkpoint |
| Full fine-tuning | step time, convergence, validation quality | model + gradient + optimizer VRAM | dataset, artifact registry | checkpoint and optimizer restore |
| LoRA/PEFT | training throughput and adapter quality | activations and adapter compute | base-model version, adapter storage | pinned base and adapter checkpoint |
| Online streaming inference | p95 TTFT, TPOT, completion, availability | prefill/decode, KV cache | gateway, model router, tokenizer | N-1, warm replica, safe admission |
| Batch inference | records/hour, unit cost, deadline | sustained throughput | input/output object storage | resumable idempotent batches |
| Embedding/reranking | p95 latency or documents/s | batching and tensor throughput | vector index, source data | replay with deduplication |
| RAG and agent workflow | correct end-to-end outcome | mixed GPU + external I/O | vector DB, tools, APIs | authorization, tool idempotency |

**Operational distinction:** A successful HTTP response is not enough for streaming inference; a stream must reach a valid terminal event. A restarted training process is not a successful recovery unless the optimizer state and data cursor are consistent.

## 3. Resource models
### 3.1 Training
A simplified mixed-precision Adam-style training estimate may require roughly **12–20+ bytes per parameter** across parameters, gradients, optimizer states and implementation-specific copies; activations, temporary buffers and fragmentation add more. This is an illustrative sizing heuristic, **not a guaranteed formula**: sharding, offload, quantization, precision, optimizer and framework materially change the footprint.

```text
training_GPU_memory ≈ model + gradients + optimizer + activations
                      + communication_buffers + safety_margin
```
Measure real peak allocation at the target sequence length, microbatch, parallelism strategy and checkpointing configuration.

### 3.2 Inference
```text
inference_GPU_memory ≈ weights + KV_cache + activations
                       + runtime_buffers + safety_margin
```
Long context, concurrency and output lengths can cause KV-cache pressure even when model weights fit. Prefill is often compute-heavy; autoregressive decode is often memory-bandwidth sensitive, but actual behavior depends on batching, model architecture and accelerator.

### 3.3 Distributed training
Effective step time includes forward/backward compute, gradient communication, synchronization and data wait. NCCL hangs can arise from network issues, crashed ranks, topology mismatch or application divergence. Check interconnect and job-wide logs, not only the busiest GPU.

## 4. Scheduling and isolation decisions
| Dimension | Training | Online inference |
|---|---|---|
| Priority | queue and checkpoint-aware | low-latency protected service |
| Preemption | possible only with tested checkpoint/restart | avoid user-visible interruption |
| Autoscaling | elastic where framework supports it | warm capacity and queue-driven scale |
| Placement | gang scheduling and locality | spread replicas across zones |
| Storage | high-throughput datasets/checkpoints | fast artifact warmup and model cache |
| Network | RDMA/NCCL bandwidth critical | ingress, token streaming and RAG latency |
| SLO | job success, throughput, restart loss | availability, TTFT, TPOT, completion |
| Cost | GPU-hours per successful training objective | cost per million delivered tokens |

Never assume MIG partitioning is supported on every GPU or appropriate for every workload. Validate device, driver, GPU Operator and framework support. Protect inference SLOs from batch starvation using explicit quotas and workload isolation.

## 5. Intake contract and ownership
Every request to onboard an AI workload should include:
- Owner, escalation, environment, data classification, residency and tenant boundary.
- Workload class, framework/runtime, model and container digests, precision, tokenizer, checkpoint or adapter dependencies.
- GPU type/count, VRAM estimate, CPU/RAM, local scratch, dataset and object-store throughput, expected network traffic.
- Arrival rate or training job schedule; prompt/output token distributions or batch/sequence lengths.
- SLO, RPO/RTO, checkpoint interval, deadline, budget and scaling policy.
- Security controls: workload identity, egress allowlist, encrypted artifacts, signed provenance and approval.

Reject incomplete critical security or recovery metadata instead of silently selecting a GPU pool.

## 6. Read-only lab: discover actual capabilities
**[SAFE-READ]** Execute only with approved context. Commands may reveal internal metadata; keep output private.
```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get nodes -o custom-columns=NAME:.metadata.name,ALLOC_GPU:.status.allocatable.nvidia\.com/gpu
kubectl get resourcequota -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
nvidia-smi -L
nvidia-smi --query-gpu=index,name,memory.total,memory.used,utilization.gpu,driver_version --format=csv
```
The custom column path and vendor resource key may vary with Kubernetes CLI/version. `nvidia-smi` requires a supported NVIDIA GPU and host driver; on CPU-only machines record it as unavailable rather than a platform failure. GPU memory/utilization snapshots do not establish sustained safe throughput.

## 7. Scenario labs and failure injection plan
**Lab A — Online inference:** synthetic prompt mixes (short/long), request arrival and completion tokens. Capture TTFT, TPOT, stream termination, KV-cache occupancy, 429 rate, GPU utilization and budget. Define admission and rollback gates.

**Lab B — Fine-tuning:** test model/adapter provenance, peak VRAM, optimizer memory, sequence-length sensitivity, checkpoint restore and loss continuity using a small disposable dataset.

**Lab C — Distributed training:** simulate worker failure in an isolated cluster, inspect NCCL rank diagnostics and prove checkpoint replay preserves optimizer state and dataset cursor.

**Lab D — Shared cluster contention:** compare a synthetic batch job and inference replicas under approved quotas. Confirm inference isolation and explain any preemption or scheduling delay.

**Lab E — Storage failure:** introduce controlled checkpoint storage latency in a disposable test and compare recovery point, job failure rate and write throughput. Never inject storage faults against production data.

## 8. Troubleshooting matrix
| Symptom | Likely bottleneck | Triage |
|---|---|---|
| Inference TTFT high, TPOT normal | prefill or queue saturation | trace admission, batching, prompt length |
| TPOT high, TTFT normal | decode/KV/memory bandwidth | decode throughput, KV pressure |
| Training OOM after longer sequences | activation growth | peak VRAM and checkpoint strategy |
| GPU idle during training | dataset/CPU/network bottleneck | input queue, I/O, rank skew |
| NCCL timeout | failed rank or fabric | collective logs, topology, network |
| Fine-tuned adapter loads wrong | base revision mismatch | signed base/adapter digests |
| Batch inference duplicates output | unsafe retry/checkpoint | idempotency keys and manifests |
| Cost spikes with low output | warm idle GPUs or retry storm | GPU-hours, output tokens, quota |

## 9. Production runbooks
**RB-002-A — Workload onboarding:** validate intake contract, owner, classification, SLO, security and capacity; deploy to staging only after policy checks.

**RB-002-B — GPU OOM:** preserve traces and model revision, distinguish weight/optimizer/activation/KV pressure, reproduce with synthetic load, change limits only under approval.

**RB-002-C — Training stall:** inspect ranks, GPU telemetry, dataset throughput, NCCL and checkpoint age; fence/restart only after state-integrity review.

**RB-002-D — Inference overload:** measure TTFT/TPOT, queue, token mix and stream completion; enforce bounded admission and avoid uncontrolled retries.

**RB-002-E — Cost escalation:** compare allocated versus productive GPU-hours, checkpoint waste, retry overhead and cost per successful output; route corrective changes through governance.

## 10. Assessment — 100 points
| Category | Points |
|---|---:|
| Workload taxonomy and architecture | 20 |
| GPU memory and resource sizing | 20 |
| Scheduling, security and ownership | 20 |
| SLOs, telemetry and FinOps | 20 |
| Troubleshooting, runbooks and lab evidence | 20 |
| **Total** | **100** |

Educational pass >=90 with no critical security or data-integrity violation. The companion lab uses **synthetic** inputs; it cannot verify real hardware, SLOs or security controls.

## 11. Companion executable lab
[Chapter 002 — Workload Classification and Placement Acceptance Lab](../labs/002-ai-workload-taxonomy-check.md) contains a Python 3.11+ workload classifier, fail-closed placement evaluator and pytest suite.

## References
- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/)
- [NVIDIA CUDA Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
- [PyTorch Distributed](https://docs.pytorch.org/docs/stable/distributed.html)
- [Google SRE Workbook](https://sre.google/workbook/)

**Next:** Chapter 003 — GPU Infrastructure Reference Architectures.
