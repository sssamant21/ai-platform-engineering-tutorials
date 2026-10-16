# Chapter 12.4 — Hands-on Lab: Production Releases, Upgrades and Rollback

**Part 12 | Approved staging | Live execution pending**

## Mission and safety

Practice version-pinned release promotion, compatibility review, synthetic canary, streaming drain, upgrade preflight and verified rollback for TrueFoundry/vLLM inference. **[SAFE-READ]** inspect; **[LAB-MUTATE]** isolated staging; **[CHANGE-CONTROL]** production routing, node upgrades or secret changes. No customer traffic, model weights, PHI or credentials in evidence.

## Lab 0 — Preflight

```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get deployments -A
kubectl get pods -A -o wide
kubectl get pdb -A
kubectl get events -A --sort-by=.lastTimestamp | tail -40
```

For an approved deployment, inspect `kubectl rollout history` only if Kubernetes Deployment is the authoritative resource. Otherwise use the actual TrueFoundry release/rollback mechanism.

## Lab 1 — Version and release manifest

Record Git commit, immutable image/model digests, tokenizer, vLLM/PyTorch/CUDA/driver, TrueFoundry agent, Kubernetes version, GPU type, gateway config and compatibility sources. Capture baseline availability, TTFT/TPOT, stream completion, queue, GPU memory and cost.

## Lab 2 — Five game days

1. **Canary release:** deploy a disposable staging revision and compare synthetic metrics.
2. **Bad runtime version:** tabletop or isolated deliberately incompatible revision; detect and abort.
3. **Streaming drain:** test active synthetic stream completion during a safe staging rollout.
4. **GPU node upgrade:** tabletop driver/device plugin compatibility, PDB and spare capacity; optional approved canary pool.
5. **Rollback:** restore known-good image/model/config/route and verify SLO recovery.

Record approvals, start/end UTC, actual/simulated actions, alert latency, user-impact proxy, decision, recovery and rollback evidence.

## Thirty-two failure scenarios

| # | Scenario | Focus |
|---|---|---|
| 01 | Mutable image tag changed | Reproducibility |
| 02 | Model digest unavailable | Artifact |
| 03 | Tokenizer/model mismatch | Compatibility |
| 04 | CUDA/driver mismatch | GPU runtime |
| 05 | PyTorch/vLLM mismatch | Engine |
| 06 | Removed vLLM flag | Configuration |
| 07 | TrueFoundry agent version incompatible | Platform |
| 08 | Kubernetes API deprecated | Upgrade |
| 09 | Canary Pod Pending | GPU capacity |
| 10 | maxSurge lacks GPU headroom | Rollout |
| 11 | PDB blocks eviction | Maintenance |
| 12 | Readiness probe premature | Startup |
| 13 | Liveness kills loading model | Probe |
| 14 | ImagePullBackOff | Registry |
| 15 | Model artifact access denied | Identity |
| 16 | Model cold-start too slow | Warmup |
| 17 | CUDA OOM after model change | Memory |
| 18 | TTFT regression | Prefill/queue |
| 19 | TPOT regression | Decode |
| 20 | Streaming completion falls | Drain |
| 21 | Gateway route stale | Traffic |
| 22 | Canary metrics not labeled by revision | Observability |
| 23 | Alert routing disabled | Detection |
| 24 | Canary sample too small | Statistics |
| 25 | Rollback model incompatible with config | Recovery |
| 26 | Prior image digest garbage-collected | Registry |
| 27 | Migration not backward compatible | Data |
| 28 | Node upgrade loses GPU allocatable | Driver |
| 29 | Autoscaler drains warm GPU reserve | Capacity |
| 30 | Security patch blocked by freeze | Governance |
| 31 | Cost doubles during blue/green | FinOps |
| 32 | No-data treated as healthy | SLO |

For each scenario capture expected/actual, UTC evidence, impact, hypothesis, mitigation, rollback and reviewer. Unsafe scenarios are tabletop.

## Twelve acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G1 | Scope, change owner and rollback approved | Pending |
| G2 | Immutable image/model/config references verified | Pending |
| G3 | Full version compatibility matrix reviewed | Pending |
| G4 | Security and supply-chain checks passed | Pending |
| G5 | Staging smoke/functional tests completed | Pending |
| G6 | Canary SLO/quality metrics compared | Pending |
| G7 | GPU headroom and PDBs validated | Pending |
| G8 | Streaming drain verified | Pending |
| G9 | Upgrade and node-pool plan approved | Pending |
| G10 | Alert routing and no-data behavior tested | Pending |
| G11 | Rollback and recovery demonstrated | Pending |
| G12 | SRE/platform/security/service owner sign-off | Pending |

Pass requires executed timestamped evidence and sign-off. A tabletop rollback is not an executed recovery.

## Cleanup and evidence

Restore staging revision and routes, remove disposable workloads, confirm GPU resources and no unexpected charges. Save sanitized manifest, metrics, decision log, rollback commit and reviewer decisions.

**Next:** Chapter 12.5 — FinOps, GPU Cost Governance and Rightsizing.
