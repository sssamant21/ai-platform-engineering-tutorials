# Chapter 13.4 — Deployment Failures, CrashLoopBackOff and ImagePullBackOff

**Part 13 — Troubleshooting | TrueFoundry SRE canonical tutorial**

## Objectives and operating rules

Diagnose TrueFoundry-managed model deployments that fail admission, image retrieval, initialization, startup, readiness or rollout. Distinguish **CrashLoopBackOff**, **ImagePullBackOff**, **ErrImagePull**, **CreateContainerConfigError**, **OOMKilled**, probe failures, model download failures and application errors. Recover GPU inference safely, preserving customer SLOs and immutable release history.

**[SAFE-READ]** Kubernetes/TrueFoundry inspection and sanitized logs. **[LAB-MUTATE]** disposable staging only. **[CHANGE-CONTROL]** production rollouts, image/registry credentials, resource limits, probe configuration, routing and rollback. Never expose registry secrets, patient data, prompts, completions, proprietary weights or access tokens.

## 1. Where deployment can fail

```text
TrueFoundry desired revision
  → Kubernetes admission / quotas / policy
  → Deployment / ReplicaSet / Pod creation
  → scheduler / GPU node placement
  → kubelet image pull / registry auth / architecture
  → init containers / secret/config mounts / volumes
  → CUDA + model artifacts + vLLM startup
  → startup probe → readiness → Service EndpointSlice
  → gateway route → customer synthetic inference
```

Each stage has different symptoms. A Deployment may be `Available=False` even when some replicas are healthy; a Pod may be `Running` but not `Ready`; a successful Pod startup does not prove inference or streaming health.

## 2. First five minutes: capture the evidence

```bash
kubectl config current-context
kubectl -n MODEL_NAMESPACE get deploy,rs,pods -o wide
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE get events --sort-by=.lastTimestamp
kubectl -n MODEL_NAMESPACE get pods -l app=MODEL_LABEL -o wide
kubectl -n MODEL_NAMESPACE get svc,endpointslices
```

Labels vary; use the actual TrueFoundry deployment inventory, not the example `app` selector. Record UTC start, desired image digest/model revision, Pod conditions, restart count, last termination reason, recent releases and affected customers. Do not assume a platform dashboard reflects the Kubernetes observed state instantly.

## 3. CrashLoopBackOff diagnostic flow

```bash
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE logs MODEL_POD --all-containers=true --since=30m --tail=200
kubectl -n MODEL_NAMESPACE logs MODEL_POD --all-containers=true --previous --tail=200
kubectl -n MODEL_NAMESPACE get pod MODEL_POD -o jsonpath='{range .status.containerStatuses[*]}{.name}{" restarts="}{.restartCount}{" waiting="}{.state.waiting.reason}{" last="}{.lastState.terminated.reason}{" exit="}{.lastState.terminated.exitCode}{"\n"}{end}'
```

`--previous` is available only if the kubelet retains the previous terminated container. `CrashLoopBackOff` is a backoff state, not a root cause. Common causes: process exits, missing config, invalid startup arguments, model load errors, CUDA incompatibility, liveness probe killing slow model initialization, OOMKilled or dependencies unavailable.

**Decision:** Did the process exit before the probe? If yes, inspect exception/exit code and runtime. If probe-induced, compare startup time with startup/liveness configuration. If OOMKilled, determine **container memory** vs **CUDA device memory**: the remedies differ.

## 4. ImagePullBackOff and registry triage

```bash
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE get pod MODEL_POD -o jsonpath='{range .spec.containers[*]}{.name}{" "}{.image}{"\n"}{end}'
kubectl -n MODEL_NAMESPACE get serviceaccount MODEL_SA
```

Interpret Events: `manifest unknown` / digest absent; `unauthorized` / registry auth; `toomanyrequests` / rate limit; DNS or TLS failures; unsupported architecture; private network egress. Verify image digest and registry availability without printing imagePullSecrets. **Do not** copy production registry credentials into a debugging Pod or disable TLS checks. `ErrImagePull` may precede `ImagePullBackOff`.

## 5. Configuration, admission and init containers

A missing Secret key or ConfigMap can cause `CreateContainerConfigError`; failed mounts can cause `ContainerCreating` delays; failing init containers produce `Init:CrashLoopBackOff`. Inspect Pod Events and `kubectl get pod -o yaml` with care: manifests can include environment values. Prefer selective JSONPath and sanitized output. Admission denials may prevent Pod creation entirely; inspect ReplicaSet/Deployment Events, policy controller messages, ResourceQuota and service account privileges.

## 6. GPU model startup specifics

Model initialization can include image pull, model artifact download, tokenizer/config validation, weight loading, tensor-parallel initialization, CUDA graph capture and warmup. Differentiate:
- image pull from container registry vs **model weight pull** from artifact storage;
- GPU allocation failure vs CUDA runtime initialization;
- GPU OOM from host cgroup OOM;
- slow legitimate warmup from deadlock or unreachable storage;
- vLLM readiness from customer-path inference success.

Check GPU node driver/device plugin, requested resources, model digest, context length, quantization, tensor parallel size, shared memory and memory limits. Version-specific vLLM flags and metrics must be verified against the deployed version.

## 7. Probe design and rollout safety

A **startup probe** can protect long model initialization from premature liveness failure. Readiness should reflect ability to serve requests; liveness should detect unrecoverable hangs without triggering restart storms. Set thresholds from observed cold-start distributions, not guesses. During rollout, verify `maxUnavailable`, `maxSurge`, GPU capacity, PDB, model warmup, draining, streaming termination and topology. GPU nodes may lack room for surge replicas; an unavailable old Pod can reduce customer capacity.

## 8. Safe rollback through authoritative control

Inspect current TrueFoundry desired revision and Kubernetes observed revision. Prefer platform-supported version rollback to an immutable image/model/config combination. Avoid untracked `kubectl set image` or `kubectl rollout undo` on TrueFoundry-managed Deployments unless explicitly approved and reconciliation behavior understood. Verify rollback with synthetic TTFT/TPOT, errors, streaming completion, readiness, GPU memory and remaining capacity. Record cost of extra warm replicas.

## 9. Twelve runbooks

1. Deployment ProgressDeadlineExceeded and unavailable replicas.
2. Pod admission rejected / ResourceQuota / policy.
3. Pending vs ContainerCreating classification.
4. ErrImagePull / ImagePullBackOff: digest and registry.
5. Registry auth and TLS/egress.
6. CreateContainerConfigError: missing configuration.
7. Init container failure and mount problems.
8. CrashLoopBackOff: exit codes and previous logs.
9. Startup/liveness/readiness probe failure.
10. CPU cgroup OOMKilled vs CUDA OOM.
11. vLLM model load, tokenizer, weights and CUDA compatibility.
12. Safe rollback, stream drain and post-recovery validation.

Each runbook captures trigger, scope, UTC evidence, hypothesis, read-only diagnosis, approved mitigation, rollback, SLO verification, escalation and owner.

## 10. Thirty-six troubleshooting scenarios

| # | Scenario | # | Scenario |
|---|---|---|---|
| 01 | ProgressDeadlineExceeded | 19 | Startup probe too short |
| 02 | ReplicaSet admission denied | 20 | Liveness restart loop |
| 03 | ResourceQuota exceeded | 21 | Readiness never passes |
| 04 | PodSecurity admission rejected | 22 | Host OOMKilled |
| 05 | Pod Pending GPU shortage | 23 | CUDA OOM |
| 06 | ContainerCreating mount delay | 24 | CUDA driver mismatch |
| 07 | ErrImagePull digest absent | 25 | Model weight IAM denied |
| 08 | ImagePullBackOff auth denied | 26 | Model artifact timeout |
| 09 | Registry rate limit | 27 | Tokenizer mismatch |
| 10 | Registry DNS failure | 28 | Tensor parallel misfit |
| 11 | Registry TLS trust failure | 29 | Slow model warmup |
| 12 | Architecture mismatch | 30 | GPU surge capacity absent |
| 13 | Missing imagePullSecret | 31 | maxUnavailable too high |
| 14 | CreateContainerConfigError | 32 | Pod termination interrupts stream |
| 15 | Secret key missing | 33 | Reconciliation reverts hotfix |
| 16 | ConfigMap key missing | 34 | Rollback artifact unavailable |
| 17 | Init:CrashLoopBackOff | 35 | Metrics missing during rollout |
| 18 | Application exit code nonzero | 36 | Pods Ready but inference fails |

## 11. Twelve acceptance gates

G1 authorized scope; G2 deployment/ReplicaSet/Pod state; G3 Events and termination evidence; G4 image digest/registry validation; G5 configuration/admission review; G6 probes and cold-start measurement; G7 GPU/CUDA/model diagnosis; G8 desired-vs-observed revision; G9 streaming-safe rollout plan; G10 approved rollback; G11 independent customer-path SLO recovery; G12 reviewed sanitized incident report.

See the [companion hands-on lab](../labs/04-deployment-failures-crashloopbackoff-imagepullbackoff-check.md). Formal review and live lab execution remain pending.

**Next:** Chapter 13.5 — Networking, DNS, TLS, Ingress and Streaming Failures.
