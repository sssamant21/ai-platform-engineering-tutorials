# Chapter 12.4 — Production Releases, Upgrades and Rollback

**Part 12 — Production Operations | TrueFoundry SRE canonical tutorial**

## Objectives and change boundaries

Build a production-safe delivery lifecycle for TrueFoundry inference workloads: immutable releases, dependency compatibility, staged promotion, model and vLLM upgrades, GPU node maintenance, streaming-safe rollout, rollback and post-change acceptance.

**[SAFE-READ]** inspect releases, cluster state, logs and SLOs. **[LAB-MUTATE]** isolated staging deployments and synthetic inference. **[CHANGE-CONTROL]** production routing, upgrades, GPU node drains, IAM/secrets and model versions. Verify actual TrueFoundry deployment and traffic-splitting capabilities before implementing any procedure. Never run an unapproved production drain, mutate live secrets or log prompts, completions, PHI or credentials.

## 1. Release unit and dependency graph

```text
Git commit / signed release manifest
  ├─ container image digest + SBOM + provenance
  ├─ model artifact digest / tokenizer / revision
  ├─ vLLM engine + CUDA/PyTorch/NVIDIA driver compatibility
  ├─ TrueFoundry deployment/agent version and config
  ├─ Kubernetes manifests, GPU resources and runtime policy
  ├─ ingress/gateway routing, TLS, timeout and drain policy
  └─ SLO/alerts, canary tests and rollback reference
                   │
          staging → canary → production
                   │
         observe → promote / abort
```

Pin immutable digests where possible; mutable image tags or model names are insufficient rollback references. Treat config, model, runtime and GPU driver as one compatibility matrix. A rollback of the container image may fail if model artifacts, schemas, secrets or traffic policy changed independently.

## 2. Change classification and ownership

| Class | Examples | Approval |
|---|---|---|
| Routine | dashboard/runbook revision | Peer review; telemetry privacy checks |
| Application | inference code, prompt template, tokenizer, model revision | Service owner + SRE |
| Runtime | vLLM, CUDA/PyTorch, base image | ML platform + GPU/SRE |
| Platform | TrueFoundry agent, Kubernetes, CNI, ingress | Platform change control |
| Infrastructure | NVIDIA driver, GPU node image, GPU pool, autoscaler | Cloud/GPU/SRE and maintenance window |
| Emergency | rollback for user-impacting incident | Incident commander; expedited audit trail |

Define blast radius, maintenance window, change freeze, security review, rollback owner, incident escalation and explicit stop conditions. Use separate staging and production identities and credentials.

## 3. Pre-release checklist [SAFE-READ]

```bash
kubectl config current-context
kubectl get deployments -A
kubectl get pods -A -o wide
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get pdb -A
kubectl get hpa -A
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

For an approved namespace/deployment:
```bash
kubectl -n MODEL_NAMESPACE rollout history deployment/MODEL_DEPLOYMENT
kubectl -n MODEL_NAMESPACE get deployment MODEL_DEPLOYMENT -o yaml
kubectl -n MODEL_NAMESPACE describe deployment MODEL_DEPLOYMENT
```

**Important:** exporting complete workload YAML may reveal secret references or sensitive annotations; sanitize evidence. Do not assume the model workload is a Kubernetes Deployment or that a direct `kubectl rollout undo` is supported by the TrueFoundry reconciliation model. Identify the authoritative control plane and use its supported rollback workflow.

## 4. Upgrade compatibility matrix

Record before/after versions for Kubernetes API, TrueFoundry agent/SDK, container base OS, Python, PyTorch, CUDA toolkit/runtime, NVIDIA host driver, GPU device plugin, DCGM exporter, vLLM, model weights/tokenizer, quantization, serving API and gateway. Verify supported combinations against release notes and vendor matrices.

**Upgrade order is dependency-specific.** Do not prescribe a universal driver-first or vLLM-first sequence. Validate a compatibility plan in staging, especially for driver/CUDA, GPU operator, model serialization, engine CLI flags, attention kernels and tokenizer behavior.

## 5. Progressive delivery patterns

**Rolling update:** appropriate when sufficient GPU headroom and safe termination exist. `maxSurge` can temporarily double scarce GPU requirements; `maxUnavailable` can violate latency SLOs.

**Blue/green:** maintains independent old/new capacity and enables rapid traffic reversal, but requires extra GPU allocation and verified model/artifact parity.

**Canary:** shifts a bounded traffic share to a new revision using an actually supported ingress/service-mesh/platform mechanism; do not assume native weighted traffic shifting. Evaluate functional quality, streaming completion, TTFT/TPOT, errors, GPU memory and queue.

**Shadow:** only with privacy, model-output and cost approvals; shadow inference can double GPU cost and may leak sensitive inputs.

Never equate a ready Pod with an SLO-compliant inference endpoint.

## 6. Streaming-safe drain

Sequence: stop new requests → propagate readiness/EndpointSlice/gateway routing change → allow active streams to finish → enforce approved grace/timeout → terminate → verify no stranded connections. Validate ingress/LB idle timeouts, Pod `terminationGracePeriodSeconds`, `preStop` behavior, vLLM shutdown handling and client retry semantics. Long-running streams can outlast rollout deadlines.

## 7. Rollback and forward-fix decision

Rollback if customer-facing SLO burn, error ratio, stream completion, TTFT/TPOT, model quality, GPU OOM or security policy crosses preapproved stop criteria. Preserve the prior known-good image, model digest, config, gateway route and capacity reservation.

```text
Can prior revision run against current config/artifacts?
  ├─ yes → shift traffic to known-good revision, verify SLO recovery
  └─ no  → activate documented forward-fix or disaster recovery plan
```

For database or metadata migrations, use backward-compatible expand/contract and independent backups; a binary rollback does not reverse irreversible data changes.

## 8. GPU node, Kubernetes and TrueFoundry upgrades

Plan spare GPU capacity and failure-domain placement before node upgrades. Inspect PDBs and active streaming sessions. Upgrade a canary node pool and run device plugin/CUDA/model smoke tests before widening. Keep Kubernetes and TrueFoundry agent compatibility under vendor guidance; a healthy Kubernetes node does not guarantee a working GPU runtime.

After any upgrade, verify GPU allocatable, CUDA availability, vLLM model load, request success, streaming completion, TTFT/TPOT, queue, autoscaler behavior, traces and alert routing.

## 9. Deployment evidence and observability

Capture approved Git commit, artifact digests, compatibility matrix, deployment owner, change ID, canary traffic share, synthetic sample size, SLI before/after, GPU memory, cold-start time, alerts, UTC timestamps, decision log and rollback verification. Set alert thresholds from approved SLOs rather than arbitrary universal numbers. Treat missing telemetry as an acceptance blocker.

## 10. Twelve production runbooks

1. **Canary fails readiness:** inspect image/model loading, probes, GPU placement and runtime logs.
2. **ImagePullBackOff:** verify digest, registry auth, node network and image architecture.
3. **CUDA compatibility regression:** compare driver, runtime, PyTorch and device plugin.
4. **vLLM API/flag regression:** compare supported CLI/config and model format.
5. **GPU memory regression:** inspect model revision, context length, KV cache and allocator.
6. **TTFT/TPOT degradation:** correlate canary vs baseline, queue and token distribution.
7. **Streaming connections cut:** inspect routing, grace, EndpointSlice and gateway timeout.
8. **Rolling update deadlocks:** inspect GPU headroom, surge, PDB and scheduler Events.
9. **TrueFoundry reconciliation overwrites change:** restore desired state via authoritative platform.
10. **Rollback fails:** verify prior artifact availability, config compatibility and routing.
11. **Node upgrade causes GPU unavailable:** inspect driver/device plugin/runtime and cordon status.
12. **Telemetry missing during promotion:** halt promotion, repair metrics and validate SLOs.

## 11. Release acceptance

Required: signed change, compatibility matrix, artifact integrity, staging and canary results, rollback rehearsal, streaming-drain proof, failure-domain capacity, alert ownership, cost estimate and post-change observation. Separate publication from actual execution and approval.

Complete the [companion lab](../labs/04-production-releases-upgrades-rollback-check.md): **32 failure scenarios, five game days and 12 acceptance gates**. Formal review and live execution pending.

**Next:** Chapter 12.5 — FinOps, GPU Cost Governance and Rightsizing.
