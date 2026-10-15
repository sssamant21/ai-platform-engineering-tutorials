# Lab 7.7 — Production vLLM Deployment and Recovery Capstone Check

**Canonical:** [Chapter 7.7](../07-production-vllm-deployment-recovery-capstone/07-production-vllm-deployment-recovery-capstone.md)  
**Scope:** Approved isolated TrueFoundry staging; [SAFE-READ] observation and [LAB-MUTATE] authorized lab changes. Production mutation requires [CHANGE-CONTROL].

## 1. Mission

Prove the entire model-serving chain: GPU scheduling → CUDA/PyTorch/vLLM → pinned model/tokenizer → model-aware readiness → bounded synthetic inference → SLO telemetry → controlled staging failure → authoritative rollback.

**No lab execution has been performed by publishing this document.** Mark each gate Pass, Fail or Not run with evidence.

## 2. Safety and approval record

```text
Change ticket / owner / reviewer:
Workspace, cluster, namespace:
GPU quota and isolated node/pool:
Vetted model ID, license, revision:
Tokenizer revision and checksum:
Image digest, vLLM/PyTorch/CUDA:
Known-good rollback release:
Maximum test duration, request rate and tokens:
Error, TTFT/ITL, queue, KV, VRAM stop thresholds:
Incident contact and cleanup owner:
```

Do not use sensitive prompts, disable auth, restart shared GPU nodes or alter cluster-wide device plugins.

## 3. Stage A — Platform preflight [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get resourcequota,limitrange
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl describe node <gpu-node>
kubectl -n <namespace> describe pod <pod>
```

Record GPU SKU, `nvidia.com/gpu` allocatable, Pod GPU request, quota, affinity/taints and runtime compatibility.

## 4. Stage B — CUDA and engine evidence [SAFE-READ]

Only in an existing authorized container:

```bash
kubectl -n <namespace> exec <pod> -c <container> -- nvidia-smi
kubectl -n <namespace> exec <pod> -c <container> -- python -c 'import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available(), torch.cuda.device_count())'
kubectl -n <namespace> exec <pod> -c <container> -- python -c 'import vllm; print(vllm.__version__)'
kubectl -n <namespace> logs <pod> --tail=200
```

Missing utility = Unknown; crashing Pod = use sanitized startup logs. Never print credentials or sensitive environment variables.

## 5. Stage C — Model provenance and memory

| Item | Expected | Observed | Pass/Fail |
|---|---|---|---|
| Image digest | | | |
| Model revision and hash | | | |
| Tokenizer/chat template | | | |
| Workload identity/storage access | | | |
| Model load and warmup | | | |
| GPU count and TP/PP | | | |
| Peak VRAM/headroom | | | |
| KV capacity and live tokens | | | |
| Actual loaded model ID | | | |

KV estimate: `2 × layers × KV_heads × head_dim × live_tokens × bytes_per_element` with assumptions recorded; compare to actual engine metrics.

## 6. Stage D — Deploy and verify [LAB-MUTATE]

Through the supported TrueFoundry/GitOps workflow, release only the approved immutable image/model/config into isolated staging. Verify Pod GPU allocation, sanitized engine logs, readiness, Ready EndpointSlice and authorized synthetic golden inference. Verify full stream completion, not merely HTTP 200.

## 7. Stage E — Bounded performance baseline [LAB-MUTATE]

| Workload | TTFT P95 | ITL P95 | E2E P95 | Tokens/s | Queue | KV | Errors |
|---|---|---|---|---|---|---|---|
| Short prompt, low concurrency | | | | | | | |
| Medium prompt, bounded concurrency | | | | | | | |
| Long prompt, approved limit | | | | | | | |
| After rollback | | | | | | | |

Use fixed synthetic prompt/output distributions, duration and stop thresholds. Record GPU VRAM and release annotations. Stop for sustained errors, queue growth, memory pressure or shared-resource impact.

## 8. Stage F — Observability and alerting

| Signal | Real series/type/units | Dashboard | Alert owner |
|---|---|---|---|
| TTFT/ITL/E2E | | | |
| Request errors/stream failures | | | |
| Prompt/output throughput | | | |
| Running/waiting | | | |
| KV/preemptions | | | |
| GPU VRAM/health | | | |
| Ready endpoints/restarts | | | |

Discover actual metric names from the installed vLLM release. Validate histograms and labels before PromQL. No prompt text or request IDs in metric labels.

## 9. Stage G — Approved failure drills [LAB-MUTATE]

Choose at least three **isolated staging** drills with explicit owner and rollback. Suggested: disposable model artifact access denied, incompatible tokenizer release, bounded KV pressure, bad lab-only image/config release. Do not alter shared identity, device plugins, nodes or production routing.

| Drill | Injection scope | Expected detection | Actual detection | LPHL/FFT | Recovery | Outcome |
|---|---|---|---|---|---|---|
| 1 | | | | | | |
| 2 | | | | | | |
| 3 | | | | | | |

Tabletop-only scenarios must be marked Not run for live execution.

## 10. Comprehensive incident tabletop

| ID | Failure | LPHL | FFT | Evidence | Mitigation |
|---|---|---|---|---|---|
| C1 | GPU node absent | | | | |
| C2 | GPU not allocatable | | | | |
| C3 | GPU Pod Pending | | | | |
| C4 | No CUDA runtime | | | | |
| C5 | Device inference failed | | | | |
| C6 | Artifact 403 | | | | |
| C7 | Weight shard missing | | | | |
| C8 | Tokenizer mismatch | | | | |
| C9 | Unsupported architecture | | | | |
| C10 | Weight VRAM OOM | | | | |
| C11 | KV-cache OOM | | | | |
| C12 | TP/NCCL timeout | | | | |
| C13 | Not Ready | | | | |
| C14 | TTFT regression | | | | |
| C15 | ITL regression | | | | |
| C16 | 429 retry storm | | | | |
| C17 | 503 no endpoints | | | | |
| C18 | Stream failure | | | | |
| C19 | Missing telemetry | | | | |
| C20 | Upgrade regression | | | | |
| C21 | Wrong model rollback | | | | |
| C22 | GPU device fault | | | | |
| C23 | Cold-start timeout | | | | |
| C24 | Failover capacity shortage | | | | |

LPHL = Lowest Proven Healthy Layer; FFT = First Failed Transition. Do not present tabletop classifications as actual observed incidents.

## 11. Stage H — Recovery and cleanup

1. Freeze the lab release and preserve sanitized evidence.
2. Restore the known-good image/model/tokenizer/runtime/GPU/scheduler configuration through TrueFoundry/GitOps.
3. Verify actual loaded model revision, Ready EndpointSlices, golden synthetic response, stream completion, TTFT/ITL, queue, KV headroom and GPU VRAM.
4. Confirm alert recovery and no unintended resource changes.
5. Delete **only** lab-owned disposable resources after approval; retain required audit evidence.

## 12. Final acceptance gates

| Gate | Pass / Fail / Not run | Evidence link / owner |
|---|---|---|
| A1 Approved scope and stop conditions | | |
| A2 Immutable artifacts and license | | |
| A3 GPU/CUDA/PyTorch/vLLM compatibility | | |
| A4 Model load, readiness, golden inference | | |
| A5 VRAM/KV/TP sizing | | |
| A6 TTFT/ITL/throughput SLO | | |
| A7 Dashboards/alerts validated | | |
| A8 Three isolated staging drills | | |
| A9 Authoritative rollback and recovery | | |
| A10 Evidence archive and signoff | | |

**Publication status:** Chapter and lab published. **Execution status:** Not run unless populated with actual evidence. **Formal vendor/SRE signoff:** Pending until explicitly completed.

## 13. Final report template

```text
Capstone date/time and owner:
Approval and isolated staging scope:
Immutable release/model/tokenizer and hashes:
GPU/node/runtime and TP topology:
Model load, readiness and golden response:
VRAM/KV estimate vs observed:
TTFT/ITL/E2E, throughput, errors:
Dashboard and alert validation:
Three drill records with LPHL/FFT:
Rollback identity and recovery duration:
A1–A10 status and evidence links:
Outstanding risks, cleanup, reviewers:
```

**Part 07 complete for publication:** seven canonical chapters and seven labs. Live execution and production-readiness approval remain independent milestones.
