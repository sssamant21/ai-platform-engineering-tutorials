# 7.2 — vLLM Deployment with TrueFoundry and Kubernetes

**Part:** 07 — vLLM & LLM Serving · **Chapter:** 2 of 7  
**Audience:** Platform engineers, MLOps, Kubernetes/GPU SREs  
**Safety:** [SAFE-READ] for observation; [LAB-MUTATE] for approved isolated nonproduction; [CHANGE-CONTROL] for production changes.

## 1. Objectives

Design, deploy, validate and recover a version-pinned vLLM inference service using an installed-version-supported TrueFoundry workflow. Understand model provenance, GPU placement, Kubernetes resource reconciliation, network/authentication, cold starts, health probes, inference contracts, telemetry and rollback.

**Prerequisites:** Chapters 6.1–6.7 and 7.1; an approved TrueFoundry workspace, integrated Kubernetes cluster, approved model license, model artifact access, CPU/GPU capacity, a known-good rollback revision, and a synthetic test prompt.

## 2. Deployment ownership

```text
Git/CI release record → TrueFoundry deployment definition (source of truth)
                                      ↓
                              Kubernetes Deployment
                              ReplicaSet / Pod / Service
                                      ↓
                     GPU scheduler + device plugin + driver
                                      ↓
                Image pull → artifact download → vLLM engine
                                      ↓
                        warmup → readiness → endpoint
                                      ↓
               gateway TLS/auth → approved inference client
                                      ↓
                metrics / logs / traces / GPU dashboards
```

TrueFoundry capabilities and deployment fields change by platform version. Use the **supported TrueFoundry UI/SDK/GitOps workflow** for the installed release; the Kubernetes examples below illustrate expected resources, not a TrueFoundry configuration schema. Avoid direct patches to managed resources that the reconciler will overwrite.

## 3. Predeployment decision record

| Area | Required decision | Evidence |
|---|---|---|
| Model | Approved ID, immutable revision, tokenizer revision, license | Manifest and hash |
| Runtime | vLLM version, Python/PyTorch/CUDA stack | Image digest and compatibility |
| GPU | GPU SKU, count, memory, node labels/taints | Capacity and allocatable |
| Storage | Artifact URI, checksum, cache, scoped workload identity | Access and lifecycle |
| Serving | Chat/completions contract, context and output limits | Contract tests |
| Network | Internal/external endpoint, TLS, auth, quotas | Approved exposure |
| Reliability | Startup/readiness/liveness, draining, retry policy | Tested budgets |
| Rollout | Revision, spare GPU capacity, promotion gates | Change plan |
| Recovery | Known-good image/model/config, owner | Rollback procedure |

A model that fits as raw weights may still fail at runtime due to KV cache, CUDA context, temporary allocations, tokenizer overhead and concurrency. Measure headroom rather than sizing by parameter count alone.

## 4. Immutable release specification

This is an **evidence manifest**, not a TrueFoundry API definition:

```yaml
release:
  workspace: "<approved-workspace>"
  namespace: "<approved-namespace>"
  source_commit: "<git-sha>"
  container_image: "registry.example.com/vllm@sha256:<approved-digest>"
  vllm_version: "<validated-version>"
  model_id: "<approved-model-id>"
  model_revision: "<immutable-model-revision>"
  tokenizer_revision: "<immutable-tokenizer-revision>"
  artifact_manifest_sha256: "<verified-sha256>"
  execution_target: "gpu"
  gpu_type: "<supported-gpu>"
  gpu_count: 1
  tensor_parallel_size: 1
  max_model_len: "<validated-context-limit>"
  endpoint_visibility: "internal"
  auth_policy: "<approved-client-policy>"
  rollback_release: "<known-good-release>"
```

Never commit access tokens or cloud credentials. Avoid mutable image tags and model aliases. Validate model code trust policy; do not enable arbitrary remote model code to work around loading errors.

## 5. Kubernetes GPU readiness [SAFE-READ]

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe node <approved-gpu-node>
kubectl -n <namespace> get resourcequota,limitrange
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Confirm allocatable `nvidia.com/gpu`, node taints, workload tolerations, affinity and available GPU memory. The NVIDIA device plugin advertises GPU resources; a Pod cannot request a fractional `nvidia.com/gpu` using ordinary Kubernetes resource semantics. GPU sharing/MIG require explicit validated support.

For an authorized existing container where tools are installed:

```bash
nvidia-smi
python -c "import torch; print(torch.__version__, torch.version.cuda, torch.cuda.is_available())"
python -c "import vllm; print(vllm.__version__)"
```

A missing diagnostic utility is not proof that the GPU is broken. Never install packages into production serving containers for troubleshooting.

## 6. Illustrative Kubernetes deployment intent

**Not a ready-to-apply manifest.** Replace placeholders, verify vLLM version-specific invocation and supply a supported container entrypoint through the TrueFoundry deployment workflow.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-example
  namespace: <approved-namespace>
spec:
  replicas: 1
  selector:
    matchLabels: {app: vllm-example}
  template:
    metadata:
      labels: {app: vllm-example}
    spec:
      serviceAccountName: <scoped-model-reader>
      terminationGracePeriodSeconds: 90
      containers:
        - name: vllm
          image: registry.example.com/vllm@sha256:<approved-digest>
          ports:
            - name: http
              containerPort: 8000
          resources:
            requests:
              cpu: "2000m"
              memory: 8Gi
              nvidia.com/gpu: "1"
            limits:
              memory: 16Gi
              nvidia.com/gpu: "1"
          startupProbe:
            httpGet: {path: /health, port: http}
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 90
          readinessProbe:
            httpGet: {path: /health, port: http}
            periodSeconds: 5
            timeoutSeconds: 3
          livenessProbe:
            tcpSocket: {port: http}
            periodSeconds: 20
            timeoutSeconds: 3
---
apiVersion: v1
kind: Service
metadata:
  name: vllm-example
  namespace: <approved-namespace>
spec:
  selector: {app: vllm-example}
  ports:
    - name: http
      port: 80
      targetPort: http
```

The `/health` route is **version/runtime dependent**; verify it actually gates engine readiness. TCP liveness only detects a listening socket and does not prove model correctness. Startup grace must be derived from measured image pull, artifact fetch, CUDA init and model load time. On GPU-constrained pools, `replicas: 2` or surge may be unschedulable; capacity planning is required before scaling.

## 7. Artifact download and model startup

```text
Pod scheduled → image pulled → identity resolved
 → immutable weights/config/tokenizer fetched → checksum validated
 → framework/CUDA/device selected → weights allocated
 → KV cache and execution capacity established
 → bounded warmup → engine health and readiness
 → Service EndpointSlice ready → authorized golden inference
```

Separate image pull failure from object-storage access failure. Track artifact download duration, cache hit/miss, model load time, GPU VRAM after initialization and readiness transition. Do not mark Ready merely because a web server process started.

## 8. Endpoint security and request contract

Choose private service exposure by default. External endpoints require explicit TLS, client authentication, authorization, request-size limits and quotas. Confirm the installed vLLM release's OpenAI-compatible route and feature support.

Illustrative **authorized synthetic** request for a deployment whose documented API supports chat completions:

```bash
curl --connect-timeout 5 --max-time 60 \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <redacted-token>' \
  -d '{"model":"<served-model-id>","messages":[{"role":"user","content":"Reply with the word ready."}],"max_tokens":16}' \
  https://<approved-host>/v1/chat/completions
```

**Do not paste a real bearer token into shell history**. Use approved secret injection and a non-sensitive payload. If the selected model lacks a chat template or the endpoint does not support chat completions, use the documented compatible API instead. Validate status, response schema, model ID/revision, bounded token output and request latency; do not rely on an exact stochastic string.

## 9. Production signals

| Signal | Interpretation |
|---|---|
| Ready endpoints | Can normal traffic route to engine? |
| Startup/model load time | Cold-start and scaling delay |
| TTFT P95/P99 | Queue + prefill + first output |
| Inter-token latency | Decode experience |
| Prompt/output tokens/s | Effective throughput |
| Running/waiting requests | Scheduling pressure |
| KV-cache occupancy | Sequence capacity and memory |
| GPU busy / VRAM | Device use and memory headroom |
| 429/5xx/timeouts | Admission and availability |
| Stream completion | Success beyond initial HTTP status |

vLLM metric names and exporter availability vary by release. Discover actual series before writing PromQL or alerts. Never attach raw prompts, user identifiers or tokens to high-cardinality metrics.

## 10. Troubleshooting scenarios

| Symptom | First failed transition | Checks |
|---|---|---|
| Pending Pod | Scheduler → GPU node | Quota, allocatable, taints, affinity |
| ImagePullBackOff | Registry → image | Digest, pull identity |
| Model 403 | Workload identity → object | IAM/KMS scope |
| Missing model revision | Manifest → artifact | Immutable version |
| Tokenizer mismatch | Artifact → engine | Tokenizer revision |
| Unsupported architecture | Model → vLLM | Version support |
| CUDA unavailable | Device → framework | Driver/plugin/container |
| Failed to infer device type | Runtime → device | GPU allocation and vLLM stack |
| Weights exceed VRAM | Load → memory | Model size, precision, parallelism |
| KV-cache OOM | Concurrency → VRAM | Context and active sequences |
| Startup restart loop | Model load → probe | Observed load budget |
| Running but not Ready | Engine → probe | Readiness path and logs |
| No Service endpoints | Pod → Service | Selector, ports, readiness |
| 401/403 inference | Client → gateway | Token scope and policy |
| 504 slow request | Gateway → engine | Queue, TTFT, deadline |
| Rollback overwritten | Manual patch → reconciler | TrueFoundry source of truth |

## 11. Incident recovery runbook

1. **Identify impact:** endpoint, model revision, first failed time, 429/5xx, TTFT, inter-token latency and affected tenants.
2. **Correlate:** deployment revision, model/tokenizer version, vLLM image, GPU SKU, artifact permissions and configuration changes.
3. **Trace:** gateway → Service → Pod scheduling → GPU/device plugin → CUDA/framework → artifact → engine load → readiness → inference.
4. **Contain:** stop promotion, preserve known-good replicas, bound retries and request concurrency.
5. **Repair:** fix the first confirmed failed transition. Do not bypass auth, checksums or model validation.
6. **Rollback:** restore compatible image digest + model/tokenizer revision + runtime parameters + GPU resources + endpoint policy through authoritative TrueFoundry/GitOps.
7. **Validate:** golden request, actual loaded revision, Ready EndpointSlice, TTFT/ITL, queue, VRAM and stream completion.
8. **Document:** incident timeline, lowest proven healthy layer, first failed transition, owner and preventive actions.

## 12. Acceptance gates

| Gate | Evidence |
|---|---|
| G1 | Approved workspace, owner and deployment source of truth |
| G2 | Immutable image/model/tokenizer and compatibility |
| G3 | GPU node, device plugin and CUDA readiness |
| G4 | Storage identity and model load/warmup verified |
| G5 | Correct readiness and ready Service endpoints |
| G6 | Authorized synthetic inference contract validated |
| G7 | TTFT/ITL, queue, VRAM, errors and alerts baselined |
| G8 | Authoritative rollback and post-recovery checks documented |

## 13. Evidence record

```text
Date/time, approval, owner:
TrueFoundry version/workspace/cluster:
Image digest / vLLM version / model and tokenizer revisions:
GPU SKU/allocatable/driver/CUDA/PyTorch:
CPU/RAM/VRAM and model KV budget:
Model fetch/load/warmup timings:
Startup/readiness/liveness and EndpointSlice:
Gateway/TLS/auth and golden response:
TTFT/ITL/token throughput/queue/errors:
Lowest Proven Healthy Layer:
First Failed Transition:
Known-good release, rollback owner and recovery validation:
```

**Lab:** [7.2 — vLLM Deployment and Recovery Check](../labs/02-vllm-deployment-truefoundry-kubernetes-check.md)  
**Next:** Chapter 7.3 — GPU CUDA PyTorch and Device Detection.

## References

- [TrueFoundry Documentation](https://docs.truefoundry.com/)
- [vLLM Documentation](https://docs.vllm.ai/)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [Kubernetes Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [NVIDIA CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
