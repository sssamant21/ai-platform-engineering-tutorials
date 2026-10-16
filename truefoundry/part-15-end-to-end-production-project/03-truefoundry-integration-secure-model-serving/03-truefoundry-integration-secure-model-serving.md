# Chapter 15.3 — TrueFoundry Integration and Secure Model Serving

**Part 15 — End-to-End Production Project | Capstone 3 of 7**

## Objective

Connect the approved Kubernetes GPU foundation from Chapter 15.2 to TrueFoundry and deploy an authenticated, observable, rollback-ready vLLM inference service. Validate control-plane registration, agent reconciliation, model provenance, GPU scheduling, runtime compatibility, gateway authorization and complete streaming responses. This is a production design and execution guide; **no deployment or lab execution is claimed**.

### Safety classification

- **[SAFE-READ]** cluster/agent health, namespace inventory, sanitized logs, service endpoints and telemetry.
- **[LAB-MUTATE]** disposable staging TrueFoundry model deployment with synthetic traffic and GPU-hour cap.
- **[CHANGE-CONTROL]** production cluster registration, IAM/RBAC, agent installation/upgrades, ingress, secrets, model releases, scaling and rollback.
- Never paste TrueFoundry tokens, cloud credentials, kubeconfigs, private model weights, real prompts or patient data into Git, tickets or telemetry.

## 1. End-to-end architecture

```text
Git/CI + immutable image digest + model artifact checksum
       │ reviewed TrueFoundry deployment spec
       ▼
TrueFoundry control plane (authenticated management API)
       │ approved connectivity and agent reconciliation
       ▼
Kubernetes cluster / platform agent / namespace
       ├─ workload identity + secret references
       ├─ GPU node pool + NVIDIA plugin + DCGM
       ├─ vLLM Pod(s) + pinned runtime/model/tokenizer
       ├─ Service / EndpointSlice / authenticated gateway
       └─ Prometheus/logs/traces + customer-path probes
       ▼
Authorized synthetic client → inference → complete SSE stream
```

**Critical boundary:** control-plane connectivity is not equivalent to inference data-plane availability. Validate both independently. Use the installed TrueFoundry version's official supported UI/API/SDK for cluster registration and deployment; do not invent universal `tfy` commands or API paths.

## 2. Prerequisites and compatibility contract

| Area | Required input | Verification |
|---|---|---|
| Cluster | Chapter 15.2 acceptance, namespace, Kubernetes version | nodes/Pods/RBAC |
| Connectivity | approved TrueFoundry endpoint, egress DNS/TLS/proxy | sanctioned connectivity check |
| Identity | cluster registration and least-privilege agent/workload roles | access review |
| GPU | allocatable resources, SKU, driver/CUDA and device plugin | node/diagnostic evidence |
| Image | immutable digest, provenance, vulnerability review | registry and CI evidence |
| Model | weights checksum, tokenizer, license and artifact ACL | artifact manifest |
| Runtime | pinned vLLM/CUDA/PyTorch versions, context/KV sizing | staging smoke test |
| Gateway | TLS, auth, tenant isolation, rate limits, private/public decision | negative auth tests |
| Operations | SLO, GPU budget, on-call, rollback artifact | change ticket |

Do not proceed if a GPU diagnostic Pod fails, secret handling is unclear, or the model cannot fit VRAM with the intended context and concurrency.

## 3. Control-plane and agent integration

1. Identify the approved TrueFoundry organization/workspace, deployment region and agent version.
2. Confirm outbound agent-to-control-plane networking, DNS, TLS trust and required proxy settings; verify exact endpoints/ports against the installed platform documentation.
3. Register the cluster using a supported workflow with reviewed privileges and scoped credentials.
4. Record desired/observed cluster state, agent namespace, service account, deployment/Helm release and reconciliation events.
5. Confirm the agent can reconcile an authorized staging workload without cluster-admin privileges beyond the supported installation minimum.
6. Test a benign desired-state change and rollback in staging; capture reconciliation duration and logs.
7. Confirm existing inference traffic remains healthy during an agent connectivity interruption tabletop.

Read-only commands (discover actual names first):

```bash
kubectl config current-context
kubectl get namespaces
kubectl get deployments,daemonsets -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl -n AGENT_NAMESPACE get deployments,pods,serviceaccounts
kubectl -n AGENT_NAMESPACE describe deployment AGENT_DEPLOYMENT
kubectl -n AGENT_NAMESPACE logs deployment/AGENT_DEPLOYMENT --since=20m --tail=150
```

`AGENT_NAMESPACE` and `AGENT_DEPLOYMENT` are discovered placeholders, not assumed vendor resource names. Sanitize logs.

## 4. Identity, secrets and authorization

Prefer cloud workload identity and approved external secret manager references. Separate platform agent permissions, model artifact read access, deployment operator permissions and inference client identity. Restrict namespace, image registry, model bucket, ingress, service accounts and audit trails. Validate deny-by-default gateway authorization and tenant routing. Secret rotation must not require exposing plaintext values in logs or model configuration.

### Trust-boundary tests

- Unauthenticated inference is rejected.
- Authorized synthetic client succeeds only in intended tenant/route.
- Model artifact identity has read-only access to approved prefix.
- Agent/service account cannot access unrelated namespaces except explicitly required.
- Token expiry, revocation and secret rotation follow an approved rollback plan.
- TLS verification is enabled end-to-end where specified; no insecure bypasses.

## 5. Immutable model-serving release bundle

Track as one compatible unit:
```text
source_commit
container_image_digest
model_weights_version_and_checksum
tokenizer_version
vllm_pytorch_cuda_versions
model_license_and_access_policy
prompt_template_and_adapter_versions
context_length_batching_and_kv_settings
gpu_sku_and_parallelism
startup_readiness_and_streaming_configuration
gateway_routing_and_autoscaling_policy
```

Model/image tags alone are insufficient for reproducible rollback. Preserve previously validated artifacts and verify they remain accessible. Do not roll back to a revision incompatible with external API or schema contracts.

## 6. Deploy and warm the staging model

Use the installed-version-supported TrueFoundry deployment specification and approved Git/CI workflow to select GPU node pool, resource requests, replicas, startup/readiness probes, model artifact access and vLLM command/config. Avoid copying unverified platform-specific YAML or CLI flags. Confirm:

1. TrueFoundry desired revision accepted.
2. Kubernetes Pod scheduled on approved GPU node and correct namespace.
3. Image and model artifacts downloaded using authorized identity.
4. vLLM initializes CUDA and loads weights/tokenizer.
5. Model warmup completes before readiness is exposed.
6. Service and EndpointSlice route only to ready Pods.
7. Authenticated synthetic inference succeeds and full SSE stream terminates correctly.
8. GPU memory, KV cache, queue, TTFT/TPOT, HTTP 429/5xx and spend are observable.

## 7. Customer-path synthetic verification

Define synthetic workload cohorts with short/long prompts and output lengths, expected response contract and stream completion marker where supported. Use a preapproved client and route; do not assume a specific public vLLM endpoint. Record UTC, model revision, eligible success, p95 TTFT/TPOT, queue, 5xx, 429, token counts, GPU memory and gateway errors. A successful health endpoint is not sufficient.

## 8. Canary, rollback and streaming-safe releases

Perform staged promotion using platform-supported controls only; do not invent traffic-split APIs. Confirm old/new GPU overlap, quota and warmup. Define canary abort conditions for correctness, security, latency, errors, incomplete streams, CUDA OOM and cost. Revert the full compatible release bundle through authoritative TrueFoundry desired state, not `kubectl rollout undo` when the platform reconciles it. Revalidate customer-path SLO and alert freshness after rollback.

## 9. Troubleshooting decision tree

```text
Model deployment unhealthy?
 ├─ Cluster disconnected → agent, egress, TLS, credentials
 ├─ Desired != observed → RBAC, reconciliation, admission
 ├─ Pod missing/Pending → quota, GPU scheduling, taints, topology
 ├─ ImagePull/Init → registry, artifact IAM, secret reference
 ├─ CrashLoop → vLLM startup, probes, CUDA, host memory
 ├─ CUDA OOM → model/context/KV/concurrency/VRAM
 ├─ Pod Ready but 502/503 → Service, EndpointSlice, gateway
 ├─ HTTP 200 but SSE incomplete → proxy timeout/drain/client
 ├─ TTFT/TPOT high → queue, prefill/decode, GPU saturation
 └─ Metrics absent → exporter/scrape before claiming SLO
```

## 10. Sixteen operational runbooks

1. Cluster registration preflight.
2. Agent network/TLS connectivity.
3. Agent RBAC and reconciliation.
4. Workload identity and model storage access.
5. Immutable image/model supply-chain verification.
6. GPU resource and scheduling.
7. vLLM/CUDA startup.
8. Model warmup and readiness.
9. Gateway TLS/auth/rate limits.
10. Synthetic inference and SSE completion.
11. Queue, KV cache and TTFT/TPOT.
12. GPU telemetry and capacity.
13. Canary and release promotion.
14. Compatible rollback and drift recovery.
15. Incident escalation and vendor evidence.
16. SLO, cost and operational handoff.

## 11. Forty-eight failure scenarios

| Domain | Scenarios |
|---|---|
| Control plane | 01 agent disconnected; 02 DNS blocked; 03 TLS failure; 04 token expired; 05 reconciliation stale; 06 RBAC denied |
| Artifacts | 07 image denied; 08 mutable tag; 09 weights unavailable; 10 tokenizer mismatch; 11 license denied; 12 secret missing |
| GPU | 13 Pod Pending; 14 quota exceeded; 15 taint mismatch; 16 plugin absent; 17 CUDA mismatch; 18 insufficient VRAM |
| Runtime | 19 startup timeout; 20 probe premature; 21 CrashLoop; 22 host OOM; 23 CUDA OOM; 24 KV cache exhausted |
| Networking | 25 TLS ingress; 26 gateway 401; 27 unauthorized route; 28 502; 29 503; 30 504 |
| Streaming | 31 SSE truncated; 32 proxy buffering; 33 drain interrupted; 34 client retry storm; 35 timeout mismatch; 36 output contract drift |
| Performance | 37 TTFT regression; 38 TPOT regression; 39 queue saturation; 40 model cold start; 41 GPU throttling; 42 cost/token spike |
| Operations | 43 no canary; 44 rollback artifact missing; 45 manual drift reverted; 46 telemetry stale; 47 paging disabled; 48 no reviewer approval |

## 12. Six staging game days

1. Agent disconnect and reconciliation recovery (tabletop if disruptive).
2. Registry/model artifact permission denial.
3. GPU Pending/taint/quota failure.
4. CUDA OOM or model compatibility regression (tabletop if unsafe).
5. Gateway auth and SSE timeout fault.
6. Failed canary with complete release rollback and cost audit.

Capture UTC timeline, affected SLO, approved mutation, expected/actual, owner, rollback, cost and review.

## 13. Fourteen acceptance gates

G1 Chapter 15.2 infrastructure accepted; G2 approved cluster registration; G3 agent connectivity/reconciliation; G4 least-privilege identity; G5 immutable image/model/tokenizer; G6 GPU scheduling/runtime; G7 warmup/readiness; G8 TLS/auth/tenant isolation; G9 synthetic inference and SSE; G10 TTFT/TPOT/queue; G11 canary; G12 compatible rollback; G13 telemetry/paging/cost; G14 reviewer sign-off.

See the [hands-on integration lab](../labs/03-truefoundry-integration-secure-model-serving-check.md). All gates pending until live execution and formal review.

**Next:** Chapter 15.4 — Networking, Security and GitOps Delivery.
