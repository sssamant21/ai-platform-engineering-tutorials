# Chapter 045 — Ray Serve and KServe Operations Project

**Part 06 — Ray Serve and KServe | Final Capstone | Chapter 045 of 100**

> **Status:** Written capstone published. Live lab execution, independent technical/security review, and operational acceptance are pending. No production or staging workloads were deployed by publishing this chapter.

## Mission and learning outcomes

Design, deploy in approved isolated staging, observe, secure, load-test and recover **two inference-serving approaches**: Ray Serve with KubeRay and KServe with a verified installed ServingRuntime. Compare their control planes, GPU placement, routing, autoscaling, resilience, SLOs, security and GPU FinOps. Produce an evidence-backed go/no-go review rather than declaring a winner by architecture alone.

## 1. Scope and prerequisites

Required prior chapters: 036–044. Prerequisites: authorized Kubernetes cluster, isolated namespace(s), compatible pinned Ray/KubeRay/KServe versions, GPU device plugin and quotas, synthetic model/test fixtures, workload identity, approved ingress, Prometheus/OTel/DCGM where available, immutable artifact digests, staging budget and rollback approvals. CPU-only Ray Serve control-plane exercises may substitute when GPU capacity is unavailable; **do not claim GPU acceptance from CPU-only tests**.

## 2. Comparative architecture

```text
                  Synthetic client + gateway
                    /                 \
             Ray Serve               KServe
           Serve app/route        InferenceService
          replicas as actors       predictor runtime
                 |                      |
         Ray Core/KubeRay       KServe controller/CRDs
                 |                      |
            GPU Pods              GPU Pods
                 \                    /
                Kubernetes scheduler
                   GPU device plugin
               node provisioning/quotas
                       |
             metrics/logs/traces/DCGM
                       |
             SLO + incident + FinOps
```

Ray Serve scaling, actor placement and fault tolerance differ from KServe reconciliation and mode-specific scaling. Evaluate both using the same client-level request contract, model revision, traffic distribution and hardware assumptions wherever possible.

## 3. Capstone deliverables

1. Architecture decision record (ADR) with version matrix and explicit assumptions.
2. GitOps-ready immutable release manifests for each platform, verified against installed schemas.
3. Least-privilege workload identity and model storage policy.
4. GPU scheduling, topology, VRAM and quota capacity plan.
5. Authentication, TLS, tenant isolation, request admission and privacy controls.
6. Synthetic golden/streaming fixtures and bounded load-test harness.
7. Telemetry dashboard and alert specification for both serving paths.
8. Failure injection **tabletop by default**, approved staging execution optional.
9. Full-stack rollback and recovery evidence.
10. FinOps comparison and signed operational acceptance report.

## 4. Release identity and provenance

Pin application image digest, Ray/KubeRay or KServe controller version, serving runtime, model and tokenizer artifact manifest digests, gateway policy, GPU SKU, tensor-parallel configuration and runtime parameters. A mutable model alias or container tag invalidates a reproducible rollback.

```yaml
# DESIGN CONTRACT ONLY — NOT A DEPLOYABLE KUBERNETES MANIFEST
release_id: approved-immutable-release
ray:
  ray_version: PINNED
  kuberay_version: PINNED
  serve_app_digest: PINNED
kserve:
  controller_version: PINNED
  serving_mode: VERIFIED_STANDARD_OR_KNATIVE
  runtime_name: VERIFIED_INSTALLED_RUNTIME
model_sha256_manifest: VERIFIED
tokenizer_sha256_manifest: VERIFIED
image_digest: VERIFIED
gpu_sku_and_count: APPROVED
rollback_release: KNOWN_GOOD
```

## 5. Deployment gates

**Gate A — environment:** cluster/context and namespaces verified; quotas and GPU availability approved.

**Gate B — compatibility:** pinned Ray/KubeRay/KServe versions, CRD schemas, runtime protocols and model format validated.

**Gate C — security:** workload identity, signed images, TLS/auth, NetworkPolicies, prompt redaction and private management interfaces.

**Gate D — readiness:** both stacks expose healthy endpoints and complete synthetic golden inference; model warmup and VRAM captured.

**Gate E — SLO:** terminal-validity availability, TTFT, TPOT, E2E p95/p99, queue wait and fairness within approved thresholds.

**Gate F — resilience:** restart/worker loss, model download failure, route failure and rollback evaluated under approved scope.

**Gate G — cost:** GPU-hours and cost per valid completion reconciled, including warm capacity and cold-start waste.

**Gate H — independent sign-off:** platform, security and application reviewers approve evidence. None of these gates are marked complete by writing this chapter.

## 6. Safe-read discovery

```bash
date -u
kubectl config current-context
kubectl get crd | grep -Ei 'ray|kserve|knative'
kubectl get rayservices,rayclusters -A
kubectl get inferenceservices -A
kubectl get servingruntimes -A
kubectl get clusterservingruntimes
kubectl get pods -A -o wide
kubectl get nodes -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

If a CRD family is absent, its `kubectl get` will fail; record this as a prerequisite gap and do not install operators without change control.

## 7. Model and request parity

Use an approved synthetic fixture with documented context length, output length, tokenizer revision, decoding parameters and terminal-response contract. Do not compare unlike model versions or precision modes without disclosing differences. Include streaming and cancellation behavior.

## 8. Ray Serve path

Validate Serve deployment config, actor resources, placement groups, proxy route, replica health, pending requests, worker/head state, model warmup and autoscaler bounds. Inspect actual Ray metrics and dashboard access policies; do not assume metric names or transparent head failover.

## 9. KServe path

Validate installed InferenceService/ServingRuntime CRD schema, controller reconciliation, serving mode, predictor Pod readiness, model initializer/storage, gateway path, inference protocol and autoscaling. Standard and Knative-based mode do not share universal routing/scale-to-zero semantics.

## 10. GPU scheduling and VRAM

```text
per-replica VRAM = model weights + KV cache + activations
                 + runtime buffers + CUDA graphs
                 + distributed communication + headroom
```

Match GPU allocation to topology, MIG profile, taints/affinity, node provisioning and model parallelism. Record per-GPU utilization, memory, throttling, errors and startup time. Never infer model safety from Pod Ready alone.

## 11. Load testing methodology

Run only approved synthetic traffic with explicit RPS/concurrency/token caps, duration and abort thresholds. Include baseline, step-load, burst, long-context, streaming disconnect and steady-state tests. Measure terminal-validity goodput and per-cohort p95/p99, not just HTTP 200 or aggregate GPU utilization.

## 12. Autoscaling and backpressure

Compare Ray Serve replica autoscaling/placement with installed KServe mode-specific scaling, plus Kubernetes node provisioning. Track desired/ready replicas, queue depth, request rejection, cold-start stages and GPU quota. Require bounded retries and queues; load shedding is preferable to an unbounded retry storm.

## 13. Observability

Define client SLIs, gateway 4xx/5xx/429, queue wait, TTFT/TPOT/E2E, terminal completion, actor/Pod restarts, Ray head and KServe reconciliation, GPU VRAM and GPU-hours. Metric names must be inspected on pinned exporters. Correlation IDs must not reveal PHI, prompts or tenant identity.

## 14. Security acceptance

Verify namespace RBAC, service account/workload identity, private object storage, TLS, model authorization, per-tenant quotas, CNI NetworkPolicies, management endpoint isolation, admission controls, signed images and artifact integrity. A shadow canary must not mirror sensitive production prompts to an unapproved runtime.

## 15. Incident command and recovery

```text
Detect client SLO burn
 → freeze rollout and assign incident commander
 → isolate gateway vs Serve/KServe vs GPU vs model
 → capture sanitized evidence and immutable release IDs
 → contain via approved traffic admission/routing
 → restore full image + model + tokenizer + runtime + gateway
 → verify golden inference and terminal streaming
 → observe latency, VRAM, queue, error budget and GPU-hours
 → record RCA, reviewer and corrective actions
```

Do not assume Ray actor state or in-flight streams survive restarts. KServe rollback may require restoring gateway weights, not just predictor Pods.

## 16. Architecture decision record (ADR)

| Dimension | Ray Serve evidence | KServe evidence |
|---|---|---|
| desired deployment model | pending | pending |
| model/runtime compatibility | pending | pending |
| GPU placement/topology | pending | pending |
| routing and protocol | pending | pending |
| autoscaling lead time | pending | pending |
| streaming completion | pending | pending |
| failure recovery/RTO | pending | pending |
| security/tenant isolation | pending | pending |
| observability/SLO | pending | pending |
| GPU-hours/valid completion | pending | pending |
| operational ownership | pending | pending |

Choose only after measured evidence, explicit tradeoffs and independent review. Hybrid use is valid if operational complexity is justified.

## 17. Twenty-four failure scenarios

| ID | Failure | First evidence |
|---|---|---|
| F01 | Ray CRDs absent | API discovery |
| F02 | KServe CRDs absent | API discovery |
| F03 | version/API mismatch | schema |
| F04 | runtime/model incompatibility | startup |
| F05 | mutable model revision drift | digest |
| F06 | object storage IAM denied | audit |
| F07 | image pull denied | Pod events |
| F08 | GPU Pod Pending | scheduler |
| F09 | GPU device plugin missing | allocatable |
| F10 | CUDA/driver mismatch | logs |
| F11 | GPU VRAM OOM | DCGM |
| F12 | KV-cache exhaustion | backend |
| F13 | Ray actor restart loop | Ray state |
| F14 | Ray head failure | control plane |
| F15 | KServe reconciliation stalled | conditions |
| F16 | KServe ServingRuntime missing | runtime list |
| F17 | ingress/TLS/Host route error | gateway |
| F18 | protocol/stream mismatch | synthetic |
| F19 | autoscaler lag and queue growth | desired/ready |
| F20 | retries amplify overload | gateway |
| F21 | canary sends traffic to wrong model | route |
| F22 | telemetry leaks sensitive data | privacy audit |
| F23 | incomplete full-stack rollback | release diff |
| F24 | GPU cost per valid completion spikes | FinOps |

## 18. Sixteen operational runbooks

1. Validate prerequisites, CRDs and serving mode.
2. Recover version/API and runtime compatibility.
3. Restore immutable model/tokenizer provenance.
4. Recover storage identity, KMS and model download.
5. Resolve image pull and supply-chain failures.
6. Diagnose GPU Pending, quota and topology.
7. Recover device plugin, CUDA and VRAM failures.
8. Diagnose KV pressure and token budgets.
9. Recover Ray Serve actor/replica failures.
10. Recover Ray head/worker control plane.
11. Restore KServe reconciliation/runtime.
12. Recover gateway, TLS, protocol and streaming.
13. Contain queue saturation, retries and autoscaling lag.
14. Validate security, privacy and tenant isolation.
15. Restore telemetry, SLO alerting and FinOps.
16. Execute complete cross-stack rollback and RCA.

## 19. Eight game days

1. Ray actor crash under synthetic load.
2. Ray head outage and recovery tabletop.
3. KServe runtime missing after upgrade.
4. GPU device plugin failure.
5. Model storage IAM/KMS failure.
6. Canary gateway route sends traffic to wrong revision.
7. Long-context load triggers KV OOM and queue saturation.
8. Rollback restores Pods but not model or gateway config.

Tabletop is the default; live fault injection requires separate isolated-staging approval.

## 20. Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Charter, versions and owners | Pending |
| G02 | Ray/KServe CRD discovery | Pending |
| G03 | Runtime/protocol compatibility | Pending |
| G04 | Immutable image/model/tokenizer | Pending |
| G05 | Storage IAM/KMS | Pending |
| G06 | GPU scheduling/topology | Pending |
| G07 | VRAM/KV capacity | Pending |
| G08 | Ray Serve readiness | Pending |
| G09 | KServe readiness | Pending |
| G10 | TLS/auth/tenant isolation | Pending |
| G11 | Golden output and streaming | Pending |
| G12 | TTFT/TPOT/E2E SLO | Pending |
| G13 | Bounded queues/retries | Pending |
| G14 | Autoscaler lead time | Pending |
| G15 | Failure recovery/RTO | Pending |
| G16 | Security/privacy review | Pending |
| G17 | Telemetry/alerts | Pending |
| G18 | Full-stack rollback | Pending |
| G19 | GPU FinOps/ADR | Pending |
| G20 | Independent sign-off | Pending |

**Written Part 06 curriculum concludes here; live acceptance is not implied.** See [companion lab](../labs/045-ray-serve-kserve-operations-project-check.md).

**Next: Chapter 046 — Kubernetes GPU Scheduling Fundamentals**, beginning Part 07.
