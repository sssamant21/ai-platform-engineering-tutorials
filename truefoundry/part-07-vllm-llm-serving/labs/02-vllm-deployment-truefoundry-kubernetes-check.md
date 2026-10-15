# Lab 7.2 — vLLM Deployment with TrueFoundry and Kubernetes Check

**Canonical:** [Chapter 7.2](../02-vllm-deployment-truefoundry-kubernetes/02-vllm-deployment-truefoundry-kubernetes.md)  
**Default:** [SAFE-READ]. Deployment, load tests and rollback are [LAB-MUTATE] in an approved isolated nonproduction TrueFoundry workspace.

## 1. Goal

Validate an immutable vLLM release end to end: TrueFoundry desired state, GPU scheduling, artifact access, CUDA, model load, health probes, authorized inference, telemetry and authoritative rollback.

## 2. Preflight [SAFE-READ]

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,replicasets,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get resourcequota,limitrange
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Record workspace, cluster, namespace, owner, approved model and license, image digest, model/tokenizer immutable revisions, GPU SKU, approved endpoint and rollback release. Never print credentials or sensitive prompts.

## 3. Artifact, identity and runtime inventory

| Requirement | Observed | Evidence | Pass/Fail |
|---|---|---|---|
| Image digest | | | |
| Model revision | | | |
| Tokenizer revision | | | |
| Model manifest/checksum | | | |
| Workload identity/storage scope | | | |
| GPU SKU and allocatable | | | |
| CUDA/framework/vLLM versions | | | |
| Context/concurrency/KV budget | | | |
| Known-good rollback release | | | |

## 4. GPU scheduling and startup

```bash
kubectl describe node <approved-gpu-node>
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> logs <pod> --tail=200
kubectl -n <namespace> logs <pod> --previous --tail=200
```

Use installed, authorized `nvidia-smi`, PyTorch and vLLM diagnostics only where available. Capture scheduling, image pull, artifact fetch, CUDA initialization, weights allocation, KV budget, warmup and readiness times.

## 5. Optional release A deployment [LAB-MUTATE]

1. In an approved disposable TrueFoundry workspace, deploy a vetted small model and pinned vLLM image using the **installed-version-supported** workflow.
2. Confirm Pod GPU allocation, model download and compatibility.
3. Observe startup probe, model-aware readiness and Ready EndpointSlice.
4. Send one approved synthetic inference request through TLS/auth and validate schema and served model.
5. Record TTFT, inter-token latency, tokens/s, queue, errors, GPU/VRAM and load time.
6. Preserve release A as a known-good rollback reference.

## 6. Optional release B and rollback [LAB-MUTATE]

Promote an approved compatible candidate with a distinct immutable revision. Check GPU surge headroom first. Compare A/B model version, golden correctness, Ready endpoints, TTFT/ITL, queue and memory. On failed gate, restore A through TrueFoundry/GitOps, not a temporary unmanaged Pod patch.

| Metric | Release A | Release B | After rollback |
|---|---|---|---|
| Model/tokenizer revision | | | |
| Ready endpoints | | | |
| Golden request | | | |
| TTFT / ITL | | | |
| Running/waiting requests | | | |
| GPU busy/VRAM | | | |
| 429/5xx/stream errors | | | |

## 7. Failure tabletop

| ID | Scenario | First failed transition |
|---|---|---|
| D1 | Pending GPU Pod | Scheduler → node |
| D2 | ImagePullBackOff | Registry → image |
| D3 | Model 403 | Identity → storage |
| D4 | Missing immutable artifact | Reference → object |
| D5 | Tokenizer mismatch | Artifact → runtime |
| D6 | Unsupported architecture | Model → vLLM |
| D7 | CUDA unavailable | Device → framework |
| D8 | Device type detection fails | Runtime → accelerator |
| D9 | Weights VRAM OOM | Load → VRAM |
| D10 | KV-cache exhaustion | Concurrency → VRAM |
| D11 | Startup restart loop | Load → probe |
| D12 | Running but not Ready | Engine → readiness |
| D13 | No ready endpoints | Pod → Service |
| D14 | 401/403 API | Client → gateway |
| D15 | 504/slow TTFT | Queue → deadline |
| D16 | Reconciler reverts hotfix | Desired state → Pod |

For each record the lowest proven healthy layer, evidence, owner, safe repair, rollback trigger and acceptance. Only inject failures into disposable lab-owned resources; otherwise use tabletop analysis.

## 8. Acceptance scorecard

| Gate | Pass / Fail / Not run | Evidence |
|---|---|---|
| G1 Workspace/ownership | | |
| G2 Immutable release/compatibility | | |
| G3 GPU/CUDA readiness | | |
| G4 Model loading and identity | | |
| G5 Probes/ready endpoints | | |
| G6 Authorized inference | | |
| G7 TTFT/ITL/queue/VRAM | | |
| G8 Rollback/cleanup | | |

## 9. Final evidence

```text
Date/time and change approval:
TrueFoundry workspace/cluster/namespace:
Image digest, vLLM/model/tokenizer versions:
GPU node/device plugin/driver/CUDA:
Storage identity and artifact integrity:
Model load/warmup and probes:
Service/EndpointSlice and gateway/TLS/auth:
Synthetic inference result and TTFT/ITL:
Queue, token throughput, GPU/VRAM:
D1–D16 failure classifications:
Lowest Proven Healthy Layer:
First Failed Transition:
Rollback revision, validation and cleanup:
Not run steps and formal review owners:
```

**Next:** Chapter 7.3 — GPU CUDA PyTorch and Device Detection.
