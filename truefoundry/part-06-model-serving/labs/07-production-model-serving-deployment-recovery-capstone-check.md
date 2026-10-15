# Lab 6.7 — Production Model-Serving Deployment and Recovery Capstone

**Canonical:** [Chapter 6.7](../07-production-model-serving-deployment-recovery-capstone/07-production-model-serving-deployment-recovery-capstone.md)  
**Mode:** **[SAFE-READ]** on existing systems; **[LAB-MUTATE]** only on isolated approved nonproduction TrueFoundry resources. Production changes require **[CHANGE-CONTROL]**.

## Mission

Deploy an approved model-serving release A, validate provenance, identity, CPU/GPU runtime, health, endpoint and SLO, promote candidate B, isolate two or three safe lab-only failures, roll back through the authoritative deployment source and prove recovery.

## 1. Preflight and approval

```bash
kubectl config current-context
kubectl -n <namespace> get deployments,replicasets,pods,services -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> get resourcequota,limitrange,networkpolicies
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Record TrueFoundry version, workspace, cluster, namespace, owner, approved synthetic model, image digest, immutable model version, rollback release, SLO and authorized client. Stop if the environment is shared production or ownership is unclear.

## 2. Artifact and identity gate

| Evidence | Value | Pass/Fail |
|---|---|---|
| Image digest and provenance | | |
| Model immutable version and manifest | | |
| Verified artifact SHA-256 | | |
| Approved model runtime and schema | | |
| Scoped image pull identity | | |
| Scoped model storage identity | | |
| Authorized client identity | | |
| TLS and network exposure | | |

Never print credential values or use sensitive inference payloads.

## 3. Runtime and capacity gate

```bash
kubectl -n <namespace> describe deployment <deployment>
kubectl -n <namespace> describe pod <pod>
kubectl -n <namespace> get events --sort-by=.lastTimestamp
kubectl -n <namespace> top pods --containers
```

Record CPU/memory requests/limits, node placement, quota and model cache. If GPU is used, record GPU SKU, allocatable resources, device plugin, compatible CUDA/framework and VRAM headroom. Metrics Server may be unavailable.

## 4. Release A — known-good baseline [LAB-MUTATE]

1. Deploy the vetted model via the supported installed-version TrueFoundry workflow.
2. Confirm image pull, artifact download, checksum, model load, warmup and startup probe.
3. Verify readiness becomes true only after model is usable.
4. Confirm Ready EndpointSlice and authorized TLS/auth route.
5. Execute approved synthetic golden inference; record expected schema/model version.
6. Record request P95/P99, errors, queue, CPU/RAM and GPU/VRAM if applicable.
7. Preserve the immutable release A manifest and rollback instructions.

```bash
kubectl -n <namespace> rollout status deployment/<deployment> --timeout=180s
kubectl -n <namespace> get deployments,replicasets,pods -o wide
kubectl -n <namespace> get endpointslices -o wide
kubectl -n <namespace> logs <pod> --tail=150
```

## 5. Release B — controlled promotion [LAB-MUTATE]

Deploy a compatible approved change with distinct image/model/config revision. Confirm extra Pod and GPU capacity before rollout. Compare:

| Signal | Release A | Release B | Pass/Fail |
|---|---|---|---|
| Immutable loaded model version | | | |
| Golden correctness | | | |
| Ready endpoints | | | |
| 5xx/timeout rate | | | |
| P95/P99 latency | | | |
| Queue depth/wait | | | |
| Model load/warmup time | | | |
| CPU/RAM/GPU/VRAM | | | |
| Stream completion if used | | | |

Do not promote a failing candidate; preserve known-good replicas.

## 6. Fault exercises [LAB-MUTATE or tabletop]

Choose two or three **lab-owned** exercises: incorrect readiness path on a disposable revision, missing synthetic model object, mismatched selector on a disposable Service. Observe failure, gather events/logs/metrics, identify the first failed transition and recover. Do not mutate shared gateways, secrets, storage, GPU nodes, device plugins or production traffic.

## 7. Incident classification worksheet

| ID | Symptom | Evidence | Lowest healthy layer | First failed transition | Recovery |
|---|---|---|---|---|---|
| C1 | ImagePullBackOff | | | | |
| C2 | Pod Pending | | | | |
| C3 | Model storage 403 | | | | |
| C4 | Missing artifact version | | | | |
| C5 | Checksum mismatch | | | | |
| C6 | CUDA/runtime mismatch | | | | |
| C7 | Host OOMKilled | | | | |
| C8 | GPU VRAM OOM | | | | |
| C9 | Startup probe restarts | | | | |
| C10 | Premature readiness | | | | |
| C11 | No Service endpoints | | | | |
| C12 | TLS/auth failure | | | | |
| C13 | 429/503 queue growth | | | | |
| C14 | P99 regression, GPU idle | | | | |
| C15 | Wrong output with HTTP 200 | | | | |
| C16 | Reconciler overwrites patch | | | | |
| C17 | Mutable model alias on rollback | | | | |
| C18 | Stream interrupted on termination | | | | |

Use tabletop for cases unsafe to inject. **Do not claim live validation for tabletop-only results.**

## 8. Authoritative rollback and recovery [LAB-MUTATE]

1. Record rollback trigger, owner, timestamp and affected SLO.
2. Restore release A via TrueFoundry/GitOps, including image digest, model version, runtime/resources, probes and endpoint policy.
3. Verify actual loaded model version, not merely the configured alias.
4. Verify authorized golden inference, Ready EndpointSlices, stable P95/P99, error ratio, queue and memory.
5. Compare elapsed recovery time with the agreed RTO.
6. Confirm the rollback survives platform reconciliation.

## 9. Acceptance scorecard

| Category | Max | Awarded |
|---|---:|---:|
| Design/provenance/integrity | 15 | |
| Identity/network/security | 15 | |
| Runtime/capacity | 15 | |
| Probes/readiness/correctness | 15 | |
| Endpoint SLO/telemetry | 15 | |
| Rollout/failure isolation/recovery | 20 | |
| Cleanup/evidence/handoff | 5 | |
| **Total** | **100** | |

Pass at **85+** only when all safety gates also pass. Missing execution evidence is **Not run**, not a pass.

## 10. Final evidence and cleanup

```text
Date/time, owner and approval:
TrueFoundry workspace/cluster/namespace:
Image digest and model version/checksum:
Workload identity, endpoint auth/TLS:
Runtime CPU/GPU, RAM/VRAM:
Release A baseline and Release B comparison:
Model load/warmup, probe/EndpointSlice:
Golden inference and latency/error SLO:
Selected live fault exercises and tabletop IDs:
Lowest Proven Healthy Layer:
First Failed Transition:
Rollback owner/release and recovery duration:
Post-recovery model version and SLO:
Lab-owned resources removed:
Unexecuted checks and formal review owners:
```

Remove only lab-owned resources through the approved platform workflow. Do not delete shared PVCs, model registry versions or caches. Formal vendor review, SRE review and actual lab execution are tracked independently of GitHub publication.

**Next:** Part 07 — vLLM and LLM Serving.
