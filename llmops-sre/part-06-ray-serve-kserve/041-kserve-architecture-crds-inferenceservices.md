# Chapter 041 — KServe Architecture, CRDs and InferenceServices

**Part 06 — Ray Serve and KServe | Chapter 041 of 100**

> **Publication:** Written tutorial published. Live labs, independent technical/security review and operational acceptance pending. KServe APIs, CRD versions, deployment modes, runtimes and networking dependencies vary by installed release; validate the exact environment before use.

## Learning objectives

Explain KServe control and data planes, InferenceService and ServingRuntime CRDs, predictor/transformer/explainer concepts, model storage and runtime selection, Kubernetes reconciliation, autoscaling, GPU allocation, networking, security, monitoring, canary/rollback and production incident response.

## 1. Why KServe?

KServe is a Kubernetes-native model-serving framework. It provides declarative resources to deploy and manage inference workloads on Kubernetes, using serving runtimes and controller reconciliation. KServe is not Ray Serve: Ray Serve orchestrates Ray actors/applications, while KServe exposes Kubernetes CRDs and integrates with different serving backends.

## 2. Reference architecture

```text
Client (TLS, auth, request deadline)
        │
API gateway / ingress / mesh (deployment-mode dependent)
        │
KServe InferenceService endpoint
        │
Optional transformer / request adaptation
        │
Predictor runtime container(s)
        ├─ model server (e.g. supported vLLM/Triton/custom)
        ├─ model artifact storage and credentials
        └─ CPU/GPU allocation
        │
Optional postprocessing / response contract
        │
Valid terminal inference response

Kubernetes API + KServe controller
        │
InferenceService, ServingRuntime, ClusterServingRuntime
        │
Deployment/Pod/Service and networking resources
        │
Kubernetes scheduler, GPU device plugin, autoscaler
        │
Prometheus/OTel, logs, DCGM, SLO, FinOps
```

The exact created Kubernetes objects depend on whether KServe uses Standard or Knative-based serving mode and the installed version. Do not assume every deployment uses Knative, Istio or scale-to-zero.

## 3. Control plane and data plane

| Component | Responsibility | SRE evidence |
|---|---|---|
| KServe controller | CRD reconciliation | controller logs, status conditions |
| InferenceService | desired model-serving endpoint | generation, conditions, URL |
| ServingRuntime | namespaced runtime definition | supported model formats, image |
| ClusterServingRuntime | cluster-scoped runtime definition | cluster policy |
| predictor | executes model inference | Pod logs, readiness, latency |
| transformer | optional request/response processing | errors, traces |
| networking | exposes internal/external service | route, TLS, endpoints |
| GPU device plugin | device allocation | node allocatable, Pod resources |

Some capabilities and field names vary by KServe release; consult the installed CRD OpenAPI schema rather than treating examples as universal.

## 4. InferenceService lifecycle

```text
GitOps manifest reviewed
    → Kubernetes API admission
    → KServe controller observes desired spec
    → selects/validates runtime and storage
    → reconciles underlying workload
    → schedules Pods / GPU resources
    → loads model artifacts and warms runtime
    → readiness and routing
    → synthetic golden inference
    → SLO/FinOps monitoring
```

An accepted InferenceService object is not evidence that the predictor is ready or returns correct results.

## 5. CRD discovery: safe-read

```bash
date -u
kubectl config current-context
kubectl get crd | grep -i kserve
kubectl api-resources | grep -Ei 'inferenceservice|servingruntime'
kubectl get inferenceservices -A
kubectl get servingruntimes -A
kubectl get clusterservingruntimes
```

If CRDs are missing, record the finding. Do not install or upgrade KServe without change approval.

Inspect installed schemas:

```bash
kubectl explain inferenceservice --recursive | head -100
kubectl explain servingruntime --recursive | head -100
```

The `kubectl explain` command uses API discovery and may require the precise resource kind/group on some clusters.

## 6. InferenceService manifest anatomy

The following is a **design skeleton, not a deployable version-independent manifest**:

```yaml
apiVersion: serving.kserve.io/v1beta1  # VERIFY installed CRD
kind: InferenceService
metadata:
  name: approved-model
  namespace: approved-staging
spec:
  predictor:
    # Exact model/runtime fields depend on KServe version.
    # Select an approved ServingRuntime and immutable model artifact.
    # Configure container resources, serviceAccount and networking.
    # Do not apply this incomplete skeleton.
```

Verify the installed CRD schema, admission webhooks and runtime support before authoring a deployable manifest.

## 7. Runtime selection

ServingRuntime and ClusterServingRuntime describe serving implementations and supported model formats. Select a runtime compatible with model format, protocol, container image, GPU requirements, artifact loading and lifecycle behavior. For LLM workloads, evaluate vLLM/Triton/custom runtimes as supported by the chosen KServe release.

Never assume a runtime name exists in every cluster. Inventory runtime CRDs and policy first.

## 8. Model storage and provenance

Use immutable model and tokenizer revisions, approved object-store paths, least-privilege workload identity, encrypted transport and validated checksums. Storage initializer behavior and credential injection vary by deployment mode/runtime. Verify that large model downloads fit startup budgets and ephemeral storage quotas.

## 9. Deployment modes and networking

Standard deployment mode and Knative-based deployment mode have different controllers, autoscaling, routing and dependencies. Document which mode is installed and how ingress, mesh, TLS, hostname, DNS and authorization are configured. Knative-based scale-to-zero must not be assumed for Standard mode.

## 10. GPU scheduling and VRAM

Align Kubernetes GPU Pod limits, device-plugin health, node selectors/taints, runtime logical GPU use, tensor-parallel ranks and actual VRAM. KServe reconciliation does not guarantee enough GPU memory for model weights, KV cache and runtime workspace.

## 11. Autoscaling and readiness

Set min/max replicas, warm capacity, cold-start budgets, per-replica concurrency and backpressure according to the installed serving mode. Autoscaling a Pod is not the same as warming a model. Use golden synthetic inference and terminal response checks beyond Kubernetes readiness.

## 12. Inference protocols and request contracts

Runtime endpoints can differ: KServe V1/V2 inference protocols, OpenAI-compatible LLM APIs or custom REST/gRPC depending on backend. Validate protocol version, route path, headers, auth, streaming and terminal completion for the actual runtime. Never assume an LLM endpoint supports every KServe V2 operation.

## 13. Observability

Collect InferenceService status conditions, reconciliation errors, Pod readiness, model load duration, request rates, errors, TTFT/TPOT/E2E, terminal validity, queue/admission, GPU VRAM and cost per valid completion. Metric names and endpoints are runtime-specific; inspect exporters and labels before alerting.

## 14. Security

Enforce namespace RBAC, workload identity, private model storage, immutable images, NetworkPolicies, TLS, authenticated gateways and restricted admission policies. Avoid exposing KServe control endpoints, model storage credentials or management dashboards. Apply tenant quotas and request/token limits at the appropriate ingress layer.

## 15. Safe rollout and rollback

```text
Approved GitOps PR
 → CRD/schema and runtime validation
 → model/tokenizer/image digests
 → isolated staging InferenceService
 → predictor readiness and route
 → synthetic golden/streaming validation
 → TTFT/TPOT/E2E + GPU memory + cost
 → approved canary/traffic shift if supported
 → promote or restore immutable release bundle
```

Canary support and traffic splitting depend on deployment mode, runtime and networking. Validate installed capabilities rather than relying on universal behavior.

## 16. Troubleshooting decision tree

```text
InferenceService unhealthy?
 ├─ CRD absent → operator/install governance
 ├─ admission rejects spec → schema/webhook
 ├─ controller not reconciling → controller logs/RBAC
 ├─ runtime not found → ServingRuntime inventory
 ├─ model download fails → storage/IAM/network
 ├─ Pod Pending → GPU quota/taints
 ├─ image pull failure → registry/provenance
 ├─ startup OOM → VRAM/ephemeral storage
 ├─ ready but 404 → ingress/protocol route
 ├─ 502/503 → endpoints/predictor readiness
 ├─ invalid output → model/tokenizer drift
 ├─ streaming incomplete → backend/timeout
 ├─ telemetry missing → exporter/scrape
 └─ GPU cost spike → idle capacity/goodput
```

## 17. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | InferenceService CRD missing | discovery |
| F02 | incompatible CRD version | API schema |
| F03 | admission webhook rejects manifest | events |
| F04 | controller RBAC denied | logs |
| F05 | runtime name absent | runtime inventory |
| F06 | model format unsupported | runtime status |
| F07 | storage initializer IAM denied | startup log |
| F08 | model artifact checksum drift | digests |
| F09 | tokenizer/model mismatch | golden test |
| F10 | registry image pull denied | Pod events |
| F11 | GPU worker Pod Pending | quota/taints |
| F12 | device plugin missing | allocatable |
| F13 | CUDA runtime incompatible | driver logs |
| F14 | predictor GPU OOM | VRAM |
| F15 | model cold start exceeds timeout | startup timing |
| F16 | ingress DNS/TLS misconfigured | gateway |
| F17 | inference protocol mismatch | HTTP/gRPC |
| F18 | predictor returns invalid output | golden fixtures |
| F19 | incomplete streaming | terminal validator |
| F20 | autoscaling lags GPU demand | desired/ready |
| F21 | management endpoint exposed | security audit |
| F22 | telemetry missing/stale | scrape freshness |
| F23 | rollback restores image not model | release bundle |
| F24 | GPU-hours per valid completion spikes | FinOps |

## 18. Sixteen operational runbooks

1. Discover KServe CRDs and serving mode.
2. Diagnose admission/schema/webhook errors.
3. Recover KServe controller reconciliation and RBAC.
4. Validate ServingRuntime and model format.
5. Recover model artifact IAM and integrity.
6. Recover image pull and supply-chain errors.
7. Diagnose GPU scheduling and device plugin.
8. Recover CUDA/VRAM startup failures.
9. Diagnose model cold start and readiness.
10. Recover ingress, DNS and TLS.
11. Diagnose V1/V2/custom protocol mismatch.
12. Recover invalid model/tokenizer output.
13. Restore streaming/cancellation contract.
14. Diagnose autoscaling and quota delays.
15. Contain security exposure and restore telemetry/FinOps.
16. Execute full-stack InferenceService rollback and RCA.

## 19. Eight tabletop game days

1. CRD schema changes during an upgrade.
2. ServingRuntime removed before a rollout.
3. Model storage IAM fails at cold start.
4. GPU Pods remain Pending under quota pressure.
5. Predictor becomes ready but inference protocol returns 404.
6. Model/tokenizer drift breaks golden responses.
7. Gateway/mesh change breaks TLS or routing.
8. Rollback restores Pods but not immutable model revision.

Default tabletop. Live injection requires separately approved isolated staging.

## 20. Twenty acceptance gates

G01 approved scope/versions; G02 CRD discovery; G03 schema/admission; G04 controller reconciliation; G05 runtime selection; G06 model provenance; G07 storage IAM; G08 image provenance; G09 GPU scheduling; G10 VRAM headroom; G11 model startup/readiness; G12 ingress/TLS; G13 inference protocol; G14 golden output/streaming; G15 autoscaling/backpressure; G16 security/privacy; G17 telemetry/SLO; G18 rollback; G19 GPU FinOps; G20 independent review.

See [companion lab](../labs/041-kserve-architecture-crds-inferenceservices-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 042 — KServe Runtime Selection and Model Storage.**
