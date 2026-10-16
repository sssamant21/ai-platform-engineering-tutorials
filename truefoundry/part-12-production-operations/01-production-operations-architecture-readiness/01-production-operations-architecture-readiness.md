# Chapter 12.1 — Production Operations Architecture and Readiness

**Part 12 — Production Operations | TrueFoundry SRE curriculum**

## Learning objectives

Define production ownership, readiness gates, release safety, operational capacity and incident interfaces for TrueFoundry-hosted inference on Kubernetes. Produce a repeatable readiness review and rollback plan that separates platform, Kubernetes, GPU, model-serving and cloud dependency responsibilities.

**Safety:** [SAFE-READ] inspection; [LAB-MUTATE] isolated staging; [CHANGE-CONTROL] any production deployment, scaling, credential, routing or runtime change. Verify deployed TrueFoundry capabilities and versions rather than assuming universal configuration. Never collect credentials, PHI, raw prompts or model weights in evidence.

## 1. Production service topology

```text
Users → DNS/TLS/Ingress → inference gateway → TrueFoundry model deployment
                                               │
                            Kubernetes scheduling/readiness/rollouts
                                               │
                      GPU node/device plugin/driver → vLLM runtime
                                               │
                         model artifacts/storage/cloud identity
                                               │
                 logs + Prometheus + traces + SLO alerting
```

**Ownership:** service team owns model correctness and application errors; platform owns TrueFoundry integration and deployment controls; Kubernetes SRE owns cluster health; GPU team owns device/runtime health; cloud security owns IAM/network guardrails. Assign a primary and backup on-call for each dependency, and record shared-responsibility boundaries.

## 2. Production readiness scorecard

| Domain | Evidence | Blocking condition |
|---|---|---|
| Architecture | Diagram, regions/zones, failure domains | Unknown single point of failure |
| Deployment | Immutable image/model digest, config revision, rollback | No verified rollback |
| Identity | Workload identity, least privilege, secret rotation | Long-lived embedded credentials |
| Networking | Private endpoint policy, TLS, DNS, ingress timeouts | Uncontrolled public exposure |
| Scheduling | GPU allocatable, quotas, affinities, taints | No feasible placement |
| Health | Startup/readiness/liveness tuned for model load | Premature restart loops |
| Data | Model artifact integrity, storage capacity | Missing integrity/restore path |
| Observability | SLI, TTFT, errors, GPU, alert ownership | Silent customer-impact failures |
| Capacity | Peak/steady RPS, tokens/sec, queue, GPU headroom | No load baseline |
| Resilience | Failure drills, zone/node recovery, RTO/RPO where relevant | Unverified critical recovery |
| Operations | Change calendar, incident commander, escalation | No responsible owner |
| FinOps | GPU-hours, idle cost, utilization, budget | Unbounded scaling/cost |

Readiness requires **evidence**, not a green dashboard alone. Production acceptance is blocked by unresolved critical findings.

## 3. Preflight commands [SAFE-READ]

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -o custom-columns='NAME:.metadata.name,GPU:.status.allocatable.nvidia\\.com/gpu'
kubectl get pods -A -o wide
kubectl get deployments,statefulsets -A
kubectl get pdb -A
kubectl get hpa -A
kubectl get events -A --sort-by=.lastTimestamp | tail -50
kubectl top nodes 2>/dev/null || true
```

Check access scope before querying cluster-wide resources. Identify actual TrueFoundry agent namespace/deployment via inventory, not a hardcoded name. Confirm installed monitoring CRDs before using ServiceMonitor/PrometheusRule commands.

## 4. Workload and GPU readiness

For each model record digest, context length, token distribution, vLLM version, tensor parallelism, quantization, GPU type/MIG profile, requested GPU count, memory limits, cold-start duration, warmup procedure and concurrency policy. Distinguish CUDA out-of-memory from Kubernetes OOMKilled and model-loading failures.

Readiness probes must account for cold loading; startup probes can protect slow initialization. Liveness must not repeatedly kill healthy but busy inference Pods. Use PDBs, topology spread and multi-zone placement only where real capacity supports them.

## 5. Capacity and availability design

Measure accepted RPS, input/output tokens/sec, TTFT p95/p99, TPOT, error rate, waiting requests, KV cache pressure, GPU utilization/memory, node capacity and autoscaler lead time under **synthetic, bounded staging traffic**. Find the throughput/latency knee. Document burst allowance, scale-out time, model warm-pool requirements, reserved GPU headroom and admission control. A high GPU utilization sample alone is not proof of saturation.

Plan graceful draining of streaming requests, termination grace period, readiness removal and gateway timeouts. Ensure rollouts preserve enough capacity to avoid queuing regressions.

## 6. Release and change management

1. Create change record with risk, owner, affected model/tenant, maintenance window and rollback trigger.
2. Pin immutable image/model/config revisions and verify SBOM/scanning policies.
3. Validate staging acceptance, smoke tests and synthetic inference.
4. Confirm GPU/node headroom and Pod disruption safety before rollout.
5. Canary a limited route/traffic share using a supported deployment mechanism; do not assume TrueFoundry implements a particular traffic-split feature.
6. Watch availability, TTFT/TPOT, queue, GPU memory, model load, alert firing and log/trace anomalies.
7. Promote only after approved observation period; roll back if SLO burn or functional regression crosses threshold.
8. Record post-change validation and close the change.

## 7. Incident readiness and communications

Maintain severity matrix, incident commander, on-call routing, vendor escalation, communication templates, rollback owner and dependency contacts. Every alert needs a runbook and a customer-impact check. For incident evidence, record UTC timestamps, environment, Pod/deployment revision, trace ID where safe, symptom, mitigation and verified recovery.

## 8. Twelve production runbooks

1. **Release readiness failure:** stop promotion; inspect blockers and owner sign-off.
2. **GPU capacity unavailable:** check allocatable, taints, quotas, autoscaler and node pool limits.
3. **Model cold-start timeout:** inspect artifact fetch, CPU/storage and startup probe budget.
4. **CrashLoopBackOff:** inspect previous logs, probe configuration and resource exhaustion.
5. **CUDA unavailable:** validate device plugin, driver/runtime, CUDA compatibility and image.
6. **Gateway 5xx/timeout:** correlate upstream, model readiness, streaming and network.
7. **TTFT regression:** compare input tokens, queue, prefill, GPU and deployment revision.
8. **GPU OOM:** inspect model/KV fit and concurrent sequences; avoid blind limit increases.
9. **Node drain stalls:** inspect PDBs, long streams, termination and remaining capacity.
10. **Autoscaling lag:** inspect pending Pods, node provisioning, image/model load and quotas.
11. **Telemetry missing:** validate exporter/collector and treat SLO state as unknown.
12. **Rollback ineffective:** verify immutable revision, dependency drift, model artifacts and traffic route; escalate.

## 9. Acceptance and operational evidence

Required outputs: signed ownership matrix, deployment manifest/digests, service map, SLO and alert inventory, capacity baseline, safe release and rollback plan, resilience assumptions, incident contacts, GPU cost estimate and unresolved risk register. Track approval separately from publication.

Complete the [companion lab](../labs/01-production-operations-architecture-readiness-check.md): **30 failure scenarios, 12 acceptance gates**. Formal review and live execution pending.

**Next:** Chapter 12.2 — GPU Capacity Planning, Quotas and Production Scaling.
