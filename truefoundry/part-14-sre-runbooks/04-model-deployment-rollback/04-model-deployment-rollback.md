# Chapter 14.4 — Model Deployment and Rollback Runbook

**Part 14 — TrueFoundry SRE Runbooks | Production change and incident recovery**

## Mission

Safely deploy, canary, promote, diagnose and roll back TrueFoundry model-serving applications across Kubernetes, GPU nodes, vLLM and gateway routing. Preserve immutable image/model/config lineage, minimize blast radius, protect streaming sessions and prove customer-path recovery.

**[SAFE-READ]** approved revision, events, logs and SLO checks. **[LAB-MUTATE]** disposable staging models. **[CHANGE-CONTROL]** production promotion, rollback, traffic shifting, secrets, ingress, capacity and model changes. Use authoritative TrueFoundry deployment/GitOps workflow; do not patch platform-managed Kubernetes resources out of band.

## 1. Deployment dependency map

```text
Source / CI build / image digest / SBOM / provenance
    ↓ approved TrueFoundry deployment specification
Model artifact digest / tokenizer / runtime / env / secret references
    ↓ TrueFoundry reconciliation → Kubernetes Deployment/Pod
GPU SKU + CUDA/driver + vLLM model load + probes
    ↓ Service / EndpointSlice / gateway / traffic routing
Synthetic requests / TTFT / TPOT / stream completion / SLO
    ↓ promotion or rollback to known-good complete revision
```

A revision is not only an image tag: record model weights, tokenizer, vLLM/CUDA version, configuration, resource requests, routing, autoscaling and feature flags. A rollback can fail if dependent artifacts or secrets are no longer accessible.

## 2. Preflight and change approval

Before deployment record:
1. Change ticket, UTC window, owner, incident commander/escalation and risk class.
2. Source commit, immutable container image digest, model artifact checksum/version and tokenizer compatibility.
3. TrueFoundry specification revision, Kubernetes namespace, GPU SKU, driver/CUDA/vLLM compatibility.
4. Approved identity, secret references, egress and registry/model storage access (never secret values).
5. Current replicas, GPU requests, quota, autoscaler max and N-1 capacity.
6. Current baseline availability, p95 TTFT/TPOT, completion, 5xx, queue and synthetic workload mix.
7. Canary strategy, guardrails, abort thresholds, monitoring owner and rollback artifact availability.
8. Data/model compatibility, schema migration and irreversible-change review.

## 3. Read-only diagnostics

```bash
kubectl config current-context
kubectl -n MODEL_NAMESPACE get deployments,replicasets,pods -o wide
kubectl -n MODEL_NAMESPACE get events --sort-by=.lastTimestamp
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE get svc,endpointslices
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE logs MODEL_POD --since=30m --tail=200
kubectl -n MODEL_NAMESPACE logs MODEL_POD --previous --tail=200
```

`--previous` works only when a previous container instance exists. Sanitize logs, environment metadata and model names. Compare observed Kubernetes revision to TrueFoundry desired state; a Deployment Available condition does not prove model readiness or successful inference.

## 4. Deployment lifecycle and gates

**Prepare:** reproducible build, image/model provenance, approved security scan, least-privilege identity and compatible CUDA/vLLM runtime.

**Validate in staging:** image pull, model artifact access, GPU scheduling, startup/warmup, readiness, HTTP and streaming synthetic probes, bounded load and N-1 headroom.

**Canary:** route a bounded synthetic or explicitly approved percentage of traffic using platform-supported traffic control. Ensure the canary has its own metrics labels and does not accidentally receive regulated workloads. Avoid assuming TrueFoundry supports a particular traffic-split feature/version.

**Promote:** increase exposure in defined steps only when error budget, TTFT/TPOT, stream completion, queue and GPU cost remain within approved gates.

**Rollback:** revert the entire compatible release unit through authoritative TrueFoundry/GitOps state; verify routing and artifact availability. Do not treat `kubectl rollout undo` as authoritative for a reconciled platform.

## 5. Failure classification matrix

| Symptom | Evidence | Likely domain |
|---|---|---|
| No Pod created | TrueFoundry status, Events, RBAC, admission | reconciliation/admission |
| Pending | FailedScheduling, GPU quota, taints | scheduling/capacity |
| ImagePullBackOff | image digest, registry authorization, Events | supply chain/registry |
| Init failure | init logs, artifact storage, secrets | bootstrap/model access |
| CrashLoopBackOff | previous logs, startup/probes, memory | runtime |
| No CUDA device | driver/plugin/Pod GPU allocation | GPU runtime |
| CUDA OOM | vLLM error, GPU memory/KV cache | model sizing |
| Readiness false | warmup, probes, dependencies | startup readiness |
| HTTP 502/503 | Service, EndpointSlice, gateway | routing |
| p95 TTFT/TPOT regression | traffic mix, queue, GPU metrics | performance |
| Streaming truncated | gateway timeout, drain, client | data plane |
| Rollback stuck | desired/observed mismatch, old artifact | release management |

## 6. Canary and rollback guardrails

Define model-specific objective gates before deployment: request success, 5xx/429, TTFT, TPOT, streaming completion, queue wait, GPU memory, cost per successful token, saturation and error-budget burn. Compare identical synthetic workload distributions and sample volumes; a small canary with no errors is not proof of safety.

**Abort immediately** on security isolation breach, incorrect model output contract, major inference unavailability, repeated OOM, excessive error-budget burn or unexpected cost. Freeze promotion, preserve evidence, restore known-good compatible revision and validate end-to-end serving. If rollback itself risks data loss or customer impact, escalate before acting.

## 7. Streaming and graceful drain

Long-lived SSE streams require correct readiness removal, endpoint propagation, termination grace, proxy timeout and model warmup. Verify clients can reconnect safely and retries are bounded. A new Pod becoming Ready does not mean old streams have completed. Measure stream completion during canary and rollback.

## 8. GPU and cost considerations

Rolling deployment can temporarily require both old and new GPU replicas. Check peak overlapping GPU requests, cluster quota, cloud GPU availability, topology, reserved capacity and cost ceiling before change. If no spare GPU slots exist, select an approved alternate strategy rather than silently dropping N-1 availability.

## 9. Fourteen operational runbooks

1. Preflight change and release provenance.
2. TrueFoundry desired/observed reconciliation.
3. GPU scheduling and quota preflight.
4. Image pull and registry failures.
5. Model artifact/secret initialization.
6. CrashLoopBackOff and probe failures.
7. CUDA driver/runtime/model fit.
8. vLLM startup and warmup.
9. Canary traffic and SLO monitoring.
10. HTTP 5xx, EndpointSlice and gateway routing.
11. Streaming drain and retry safety.
12. Full compatible rollback.
13. Failed rollback and escalation.
14. Post-release recovery, cost and audit evidence.

Each runbook records trigger, impact, safe evidence, mitigation, rollback, SLO verification, owner and escalation.

## 10. Forty failure scenarios

| # | Scenario | # | Scenario |
|---|---|---|---|
| 01 | Missing image digest | 21 | Model weights incompatible |
| 02 | Registry 401/403 | 22 | Tokenizer mismatch |
| 03 | ImagePullBackOff | 23 | Model load timeout |
| 04 | Agent disconnected | 24 | CUDA OOM |
| 05 | Reconciliation stale | 25 | KV cache exhaustion |
| 06 | Admission denied | 26 | TTFT regression |
| 07 | GPU Pod Pending | 27 | TPOT regression |
| 08 | Taint mismatch | 28 | Queue saturation |
| 09 | GPU quota exceeded | 29 | HTTP 502 |
| 10 | Autoscaler max | 30 | HTTP 503 |
| 11 | Cloud GPU unavailable | 31 | HTTP 504 |
| 12 | Init container failed | 32 | Empty EndpointSlice |
| 13 | Artifact IAM denied | 33 | TLS failure |
| 14 | Secret reference missing | 34 | SSE stream truncated |
| 15 | CrashLoopBackOff | 35 | Retry storm |
| 16 | Host OOMKilled | 36 | Canary sample too small |
| 17 | Startup probe too short | 37 | GPU cost exceeds cap |
| 18 | Readiness incorrectly green | 38 | Old artifact unavailable |
| 19 | Driver/CUDA mismatch | 39 | Manual drift reverted |
| 20 | GPU device missing | 40 | Rollback fails |

## 11. Twelve acceptance gates

G1 approved change/scope; G2 immutable image/model/config provenance; G3 security/identity/artifact access; G4 GPU quota/N-1/cost; G5 startup/readiness/streaming; G6 synthetic workload baseline; G7 canary SLO; G8 TTFT/TPOT/GPU; G9 abort/escalation; G10 compatible rollback; G11 end-to-end post-change SLO and alert health; G12 reviewer/audit sign-off.

See the [companion lab](../labs/04-model-deployment-rollback-check.md). Publication does not mean deployment was performed or reviewed.

**Next:** Chapter 14.5 — vLLM Serving Health and Performance Runbook.
