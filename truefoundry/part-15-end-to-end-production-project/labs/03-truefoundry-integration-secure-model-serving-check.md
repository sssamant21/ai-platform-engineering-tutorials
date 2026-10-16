# Chapter 15.3 — Hands-on Lab: TrueFoundry Integration and Secure Model Serving

**Part 15 — End-to-End Production Project | Approved staging | Live execution pending**

## Goal

Register an approved Kubernetes GPU cluster with TrueFoundry, validate agent reconciliation and identity, deploy a pinned vLLM staging model, verify authenticated synthetic inference and complete streaming, and rehearse safe rollback.

**Safety:** read-only by default. Cluster registration, agent install, GPU model deployment, traffic changes and secrets require explicit staging change approval, spend cap and cleanup. No real prompts, PHI, credentials or customer model artifacts in lab logs.

## Lab 0 — Inventory and readiness

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get namespaces
kubectl get deployments,daemonsets -A
kubectl get events -A --sort-by=.lastTimestamp
```

Record Chapter 15.2 GPU acceptance, cloud/cluster/namespace, agent version and installation method, approved TrueFoundry endpoint, IAM/RBAC, registry/model artifact digests, vLLM/CUDA and GPU SKU.

## Lab 1 — Agent and connectivity

Use the approved installed-version TrueFoundry registration workflow. Discover agent namespace and deployment, then inspect reconciliation, DNS/TLS egress, service account permissions and sanitized logs.

```bash
kubectl -n AGENT_NAMESPACE get deployments,pods,serviceaccounts
kubectl -n AGENT_NAMESPACE describe deployment AGENT_DEPLOYMENT
kubectl -n AGENT_NAMESPACE logs deployment/AGENT_DEPLOYMENT --since=20m --tail=150
```

Never assume these placeholder names or attempt registration with an unreviewed token.

## Lab 2 — Immutable vLLM staging model

Through supported TrueFoundry UI/API/SDK and approved CI, deploy one model pinned to image digest, weights checksum, tokenizer, vLLM/CUDA version and GPU requests. Verify model storage access, startup, GPU visibility, readiness and EndpointSlice routing.

## Lab 3 — Security and synthetic inference

Test authorized vs unauthorized synthetic requests, tenant isolation, TLS, rate limiting and complete SSE streaming. Record TTFT, TPOT, queue, token mix, HTTP errors, GPU memory and telemetry freshness. Never log model inputs/outputs beyond approved synthetic test vectors.

## Lab 4 — Canary and rollback

Perform a safe staging config/model revision canary using supported platform controls, then restore previously validated compatible bundle. Verify TrueFoundry desired/observed agreement, customer-path inference, SSE completion, GPU headroom and alert re-arming.

## Lab 5 — Six game days

1. Agent disconnected (tabletop where unsafe).
2. Artifact/registry IAM denied.
3. GPU quota/taint Pending.
4. vLLM CUDA/model mismatch.
5. Gateway auth or SSE timeout.
6. Failed canary and rollback.

For each record UTC, change approval, expected/actual, customer SLO, mitigation, rollback, reviewer and GPU-hours.

## Forty-eight scenario checklist

01 agent; 02 DNS; 03 TLS; 04 token; 05 reconciliation; 06 RBAC; 07 image; 08 tag; 09 weights; 10 tokenizer; 11 license; 12 secret; 13 Pending; 14 quota; 15 taint; 16 plugin; 17 CUDA; 18 VRAM; 19 startup; 20 probe; 21 CrashLoop; 22 host OOM; 23 CUDA OOM; 24 KV; 25 TLS ingress; 26 401; 27 unauthorized; 28 502; 29 503; 30 504; 31 SSE; 32 buffering; 33 drain; 34 retries; 35 timeout; 36 contract; 37 TTFT; 38 TPOT; 39 queue; 40 cold start; 41 throttle; 42 cost; 43 canary; 44 rollback; 45 drift; 46 telemetry; 47 paging; 48 review.

## Fourteen acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Chapter 15.2 GPU readiness | Pending |
| G2 | Approved cluster registration | Pending |
| G3 | Agent connectivity/reconciliation | Pending |
| G4 | Least-privilege identity and secrets | Pending |
| G5 | Immutable image/model/tokenizer | Pending |
| G6 | GPU scheduling/CUDA/vLLM | Pending |
| G7 | Model warmup and readiness | Pending |
| G8 | TLS/auth/tenant isolation | Pending |
| G9 | Synthetic inference and SSE completion | Pending |
| G10 | TTFT/TPOT/queue | Pending |
| G11 | Canary promotion | Pending |
| G12 | Compatible rollback | Pending |
| G13 | Telemetry/paging/GPU cost | Pending |
| G14 | Reviewer sign-off | Pending |

## Evidence template

```text
Lab ID / UTC / approval:
TrueFoundry workspace/agent version:
Cluster/namespace/GPU SKU/driver:
Image/model/tokenizer/vLLM/CUDA digests:
Agent desired/observed and connectivity:
IAM/RBAC/secret references:
GPU scheduling and model warmup:
Synthetic authorized/denied requests:
TTFT/TPOT/queue/SSE/HTTP errors:
Canary, rollback and post-change SLO:
GPU-hours, cleanup and reviewer:
```

## Cleanup

Restore approved staging desired state, delete only lab-owned model resources, verify no orphaned GPU nodes/routes or secret permissions, confirm alerts and cost budgets remain active. Leave acceptance gates pending until independently performed and signed off.

**Next:** Chapter 15.4 — Networking, Security and GitOps Delivery.
