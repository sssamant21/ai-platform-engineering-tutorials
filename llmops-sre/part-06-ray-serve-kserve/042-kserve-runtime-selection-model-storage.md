# Chapter 042 — KServe Runtime Selection and Model Storage

**Part 06 — Ray Serve and KServe | Chapter 042/100**

> **Publication status:** Written tutorial. Live labs, independent technical/security review, and production acceptance are pending. KServe schemas, built-in runtimes, model URI schemes and protocol support vary by version and deployment mode. Inspect the installed CRDs and runtime specifications before applying examples.

## Outcomes

Select a production-compatible ServingRuntime or ClusterServingRuntime; design immutable, least-privilege model storage; validate cold-start, GPU memory, inference protocol and rollback; diagnose runtime/storage failures through executable safe-read checks and an isolated lab.

## 1. Architecture and responsibility boundaries

```text
GitOps / CI model release bundle
   ├─ image digest, model digest, tokenizer digest
   ├─ ServingRuntime or ClusterServingRuntime
   └─ InferenceService predictor and storage reference
               │
      Kubernetes API admission
               │
       KServe reconciliation
               │
   model storage / initializer (when applicable)
     ├─ object store or OCI artifacts
     ├─ workload identity / TLS
     └─ local cache, ephemeral/PV capacity
               │
   model server (vLLM/Triton/custom runtime)
     ├─ GPU device assignment
     ├─ weights and tokenizer loading
     ├─ CUDA/KV cache allocation
     └─ protocol and readiness
               │
   authenticated gateway → valid terminal response
```

KServe manages declarative serving resources. The chosen runtime owns model-loading details, inference APIs and model-server telemetry. A model URI is not proof that bytes are immutable or accessible.

## 2. Runtime types and scope

A **ServingRuntime** is namespaced. A **ClusterServingRuntime** is cluster-scoped and can provide shared runtime definitions subject to governance. Both describe supported model formats and runtime container specifications. The actual selection mechanism, runtime names and predictor fields depend on the installed KServe release.

**[SAFE-READ]**:

```bash
kubectl config current-context
kubectl api-resources | grep -Ei 'servingruntime|inferenceservice'
kubectl get servingruntimes -A
kubectl get clusterservingruntimes
kubectl get inferenceservices -A
kubectl explain servingruntime.spec --recursive | head -120
```

If a resource type is absent, stop and document it. Never install CRDs or mutate cluster-scoped runtimes as part of discovery.

## 3. Runtime selection decision matrix

| Criterion | Questions | Required evidence |
|---|---|---|
| model family/format | Does the runtime support the artifact format? | supported model formats and fixture |
| protocol | KServe V1/V2, OpenAI-compatible, gRPC, custom? | real endpoint contract |
| GPU | Does the backend support SKU, CUDA, TP/PP? | runtime/device compatibility |
| model loading | storage initializer, direct mount, OCI? | startup logs and digests |
| performance | TTFT/TPOT, batch behavior, context limit? | load test |
| isolation | namespace, identity, tenant quotas? | security review |
| operations | health probes, metrics, graceful drain? | failure drills |
| supply chain | image/model signatures, SBOM? | attestation |
| cost | warm replicas, startup GPU-hours? | FinOps worksheet |

For LLMs, compare vLLM, Triton/TensorRT-LLM and approved custom serving runtimes only when available and compatible. Do not infer a particular backend is installed merely because KServe supports the integration in some releases.

## 4. Model storage patterns

| Pattern | Benefits | Risks |
|---|---|---|
| object storage URI | central artifact management | network, IAM, cold downloads |
| OCI model artifact | digest pinning and distribution | registry permissions, image size |
| PVC/shared volume | predictable local mount | stale contents, multi-writer semantics |
| node-local cache | faster restarts | cache invalidation and node affinity |
| baked image | self-contained revision | enormous image pulls |

Document exact URI semantics, fetcher, checksum verification, local path, capacity and credential source for each pattern. An object-store version ID or content digest is preferable to mutable aliases.

## 5. Immutable release manifest

```yaml
# RELEASE CONTRACT / DOCUMENTATION — NOT A DEPLOYABLE KServe OBJECT
release:
  serving_image: registry.example.invalid/model-server@sha256:REPLACE
  model_artifact: object://approved-bucket/path/model?version=REPLACE
  model_sha256: REPLACE_WITH_REAL_DIGEST
  tokenizer_sha256: REPLACE_WITH_REAL_DIGEST
  runtime_kind: ServingRuntime
  runtime_name: VERIFIED_INSTALLED_RUNTIME
  inference_protocol: VERIFIED_RUNTIME_PROTOCOL
  gpu_sku: APPROVED_SKU
  tensor_parallel_size: APPROVED_VALUE
  storage_identity: WORKLOAD_IDENTITY_REFERENCE
  rollback_release: KNOWN_GOOD_IMMUTABLE_BUNDLE
```

Treat this as a checklist, not Kubernetes YAML. Never paste secrets into Git. A digest placeholder is not a valid digest.

## 6. KServe CRD schema and compatibility

**[SAFE-READ]**:

```bash
kubectl get crd inferenceservices.serving.kserve.io -o jsonpath='{.spec.versions[*].name}'
kubectl get crd servingruntimes.serving.kserve.io -o jsonpath='{.spec.versions[*].name}'
kubectl explain inferenceservice.spec.predictor --recursive | head -150
kubectl explain servingruntime.spec --recursive | head -150
```

Depending on release, predictor fields, storageUri, model/runtime references and runtime container specification may differ. Author a concrete deployable manifest only after validating the installed schema, runtime name and serving mode. Run `kubectl apply --dry-run=server` only in an authorized environment and recognize that server-side dry-run can invoke admission webhooks.

## 7. Storage identity and access

Prefer Kubernetes workload identity (IRSA/EKS Pod Identity, Azure Workload Identity, GKE Workload Identity or platform equivalent) over long-lived static keys. Scope permissions to exact object prefixes and read-only operations. Check service account binding, bucket policy, VPC endpoint/private routing, DNS, TLS, KMS decrypt rights and cross-account trust.

A 403 can originate from object storage, KMS, a proxy or an artifact registry. Determine the actual caller identity before changing permissions.

## 8. Storage integrity and provenance

Verify model and tokenizer hashes independently. Pin config files, generation parameters and any custom code dependencies. Validate signed artifact provenance where supported. Restrict remote-code execution and treat model packages as untrusted supply-chain inputs.

Example local checksum:

```bash
sha256sum /approved/local/path/model-config.json
sha256sum /approved/local/path/tokenizer.json
```

Hashing one file does not validate an entire model directory; use a signed manifest containing every file digest or a content-addressed artifact.

## 9. Startup sequence and budgets

```text
Pod scheduled → image pull → credentials available
 → artifact fetch/cache → checksum validation
 → tokenizer/config load → model weights load
 → CUDA/NCCL setup → warmup → readiness
 → synthetic golden inference → traffic eligibility
```

Track each phase separately. Cold-start failures often masquerade as probe failures. Set startup/readiness probes and timeouts to accommodate actual model size without hiding permanently unhealthy replicas.

## 10. GPU memory and parallelism

Estimate VRAM = weights + KV cache + activations + runtime workspace + CUDA graphs + communication buffers + headroom. TP/PP settings must match allocated GPU topology. A KServe InferenceService being Ready does not prove the backend is safe at peak context/concurrency.

## 11. Model server protocol contracts

KServe V1/V2, OpenAI-compatible chat/completions and custom gRPC/REST are distinct interfaces. Record route, authentication, request schema, streaming semantics, error mapping and terminal completion. Run a synthetic golden fixture after every runtime or artifact change.

## 12. Runtime security boundaries

Restrict ClusterServingRuntime creation to platform administrators. Enforce namespace quotas, admission policies, signed image digests, least-privilege service accounts, NetworkPolicies, private model storage, encryption and audited deployment approvals. Never expose model storage credentials in logs or traces.

## 13. Model caching and capacity

For each artifact record compressed/uncompressed size, ephemeral storage requirements, node cache reuse, eviction, inode pressure, download bandwidth and egress cost. Ensure multiple cold-start replicas do not saturate object storage or overwhelm node disks. Use controlled startup concurrency.

## 14. Observability and SLO

Measure artifact fetch latency/error, bytes downloaded, model initialization duration, Pod readiness, inference protocol errors, TTFT/TPOT/E2E, valid terminal completions, GPU VRAM and GPU-hours per valid completion. Inspect installed runtime exporters; never invent native metric names.

## 15. Read-only troubleshooting

```bash
date -u
kubectl config current-context
kubectl get inferenceservices -A
kubectl get servingruntimes -A
kubectl get clusterservingruntimes
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
```

For an approved namespace and known object:

```bash
kubectl -n APPROVED_NS describe inferenceservice APPROVED_MODEL
kubectl -n APPROVED_NS get pods -o wide
kubectl -n APPROVED_NS describe pod APPROVED_POD
kubectl -n APPROVED_NS logs APPROVED_POD --all-containers --tail=100 --since=15m
```

Review logs for secrets or prompt leakage before sharing. Discover Pod names rather than assuming them.

## 16. Troubleshooting decision tree

```text
InferenceService not serving?
 ├─ runtime absent → inventory and schema
 ├─ model format mismatch → supported formats
 ├─ Pod Pending → GPU quota/taints
 ├─ image pull error → registry IAM/digest
 ├─ artifact 403 → caller identity/bucket/KMS
 ├─ artifact 404 → URI/version/digest
 ├─ download timeout → network/egress/cache
 ├─ no disk space → ephemeral/PVC/inodes
 ├─ checksum mismatch → artifact corruption/drift
 ├─ CUDA OOM → VRAM/context/TP
 ├─ ready but invalid output → tokenizer/protocol
 ├─ TTFT regression → cold starts/queue
 └─ high cost → redundant downloads/idle GPU
```

## 17. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | runtime CRD missing | API discovery |
| F02 | runtime selection ambiguous | runtime inventory |
| F03 | model format unsupported | runtime status |
| F04 | runtime image digest drift | imageID |
| F05 | registry pull denied | Pod events |
| F06 | model URI points to mutable alias | artifact metadata |
| F07 | storage initializer not configured | Pod spec |
| F08 | workload identity misbound | service account |
| F09 | object storage 403 | storage access logs |
| F10 | KMS decrypt denied | KMS audit |
| F11 | model artifact 404 | object version |
| F12 | network timeout during download | network trace |
| F13 | ephemeral storage exhausted | eviction/events |
| F14 | cache serves stale model | digest audit |
| F15 | checksum mismatch | manifest hashes |
| F16 | tokenizer/model revision mismatch | golden fixture |
| F17 | custom model code unsafe | supply-chain review |
| F18 | GPU Pod Pending | scheduler events |
| F19 | CUDA/driver mismatch | startup logs |
| F20 | VRAM OOM during load | GPU metrics |
| F21 | probe kills slow model load | probe/startup timing |
| F22 | protocol mismatch | HTTP/gRPC response |
| F23 | rollback omits artifact revision | release bundle |
| F24 | cold downloads inflate GPU-hours | FinOps |

## 18. Sixteen operational runbooks

1. Verify installed CRD versions and runtime scope.
2. Diagnose runtime resolution/model format mismatch.
3. Validate runtime container image provenance.
4. Recover registry pull errors.
5. Validate immutable model/tokenizer release bundle.
6. Diagnose storage initializer and mount configuration.
7. Recover workload identity and object-store IAM.
8. Diagnose KMS, TLS, DNS and network failures.
9. Recover missing artifact/version 404.
10. Resolve cache corruption, checksum and stale revision.
11. Recover ephemeral storage/PVC exhaustion.
12. Diagnose GPU scheduling and driver compatibility.
13. Recover model load VRAM OOM and cold starts.
14. Validate inference protocol, golden output and streaming.
15. Restore telemetry, security posture and GPU FinOps.
16. Execute full artifact/runtime/image rollback and RCA.

## 19. Eight tabletop game days

1. ClusterServingRuntime is changed unexpectedly.
2. Model artifact URI resolves to a newer mutable revision.
3. Workload identity loses object-store read access.
4. KMS denies decrypt during scale-out.
5. Simultaneous GPU cold starts exhaust ephemeral storage.
6. Tokenizer mismatch causes invalid terminal responses.
7. GPU model load OOMs after a runtime upgrade.
8. Image rollback succeeds but model artifact remains changed.

Tabletop by default; live fault injection requires separately approved isolated staging.

## 20. Twenty acceptance gates

G01 approved scope/version; G02 runtime inventory; G03 model format/protocol; G04 image provenance; G05 immutable model revision; G06 tokenizer integrity; G07 workload identity; G08 storage/KMS/TLS; G09 network/download; G10 cache/disk capacity; G11 GPU scheduling; G12 CUDA/VRAM; G13 cold-start readiness; G14 golden inference; G15 streaming/terminal correctness; G16 security/privacy; G17 telemetry/SLO; G18 complete rollback; G19 FinOps; G20 independent review.

See [companion lab](../labs/042-kserve-runtime-selection-model-storage-check.md). All gates remain pending until independently evidenced.

**Next: Chapter 043 — KServe Networking, Security and Canary Rollouts.**
