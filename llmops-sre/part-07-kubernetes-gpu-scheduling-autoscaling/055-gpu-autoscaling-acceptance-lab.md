# Chapter 055 — GPU Autoscaling Acceptance Lab

**Part 07 — Kubernetes GPU Scheduling and Autoscaling | Chapter 055/100 | Part 07 written capstone**

> **Status:** Written acceptance project published. All live tests, independent technical/security reviews, and operational acceptance remain pending. This chapter defines acceptance criteria; it does **not** certify a real cluster. Run mutations or fault injections only in explicitly approved isolated staging with budget, rollback and security controls.

## Objectives and scope

Integrate Chapters 046–054 into a measurable production-style GPU inference platform acceptance exercise: device-plugin health, scheduling, taints/affinity, GPU quota, MIG/sharing, HPA/KEDA, CA/Karpenter, warmup, multi-zone resilience, PDBs and Pending Pod diagnosis.

## 1. Reference architecture

```text
Synthetic tenants → authenticated gateway → request/token quotas
                            │
                     model inference SLO
                            │
                  HPA/KEDA custom metrics
                            │
                  Deployment / GPU Pods
                            │
        ResourceQuota + RBAC + Pod Security
                            │
     scheduler / affinity / topology / PDB
                            │
         CA or Karpenter capacity owner
                            │
        AZ-a ───── AZ-b ───── AZ-c
        GPU        GPU         GPU
        │          │           │
        NVIDIA plugin/driver/MIG/shared policy
                            │
             image/model pull + CUDA warmup
                            │
          Prometheus/DCGM/OTel + FinOps
```

This design is not complete without workload identity, NetworkPolicy, artifact authorization, streaming-safe drain and human approval of disruptive operations.

## 2. Acceptance philosophy

**Written ≠ executed ≠ reviewed ≠ accepted.** Each gate needs a reproducible command/query, timestamp, cluster/version, expected result, actual result, evidence artifact, owner and reviewer. Failed, skipped or not-run gates are not passes.

## 3. Environment contract

Record provider, region/AZs, Kubernetes version, GPU SKU/driver, GPU Operator/device-plugin, MIG strategy, namespace owners, HPA/KEDA, CA/Karpenter, Prometheus/DCGM, model digest, synthetic request corpus, cloud quota, cost ceiling, SLO and rollback.

## 4. Prerequisite controls

Use isolated staging, dedicated test namespaces, least-privilege service accounts, synthetic non-sensitive prompts, approved GPU-hour budget, fixed image/model digests and rate limits. Never collect raw prompts, secrets or token values in logs.

## 5. Test phases

| Phase | Goal | Pass evidence |
|---|---|---|
| A | baseline inventory | GPU allocatable/allocated, controller versions |
| B | tenant isolation | quota/RBAC/Pod Security/NetworkPolicy |
| C | placement | taints, affinity, topology, MIG resource names |
| D | HPA/KEDA | correct metric and bounded replica scaling |
| E | node scale-out | GPU node ready and driver/plugin registered |
| F | model warmup | image/model/CUDA readiness measured |
| G | burst/load | terminal-valid throughput, TTFT/TPOT |
| H | zone/disruption | N-1 capacity, PDB, stream-safe drain |
| I | fault diagnosis | quota, Pending, plugin, cloud shortage |
| J | rollback/FinOps | restored config, cost attribution |

## 6. SLO and workload definition

Define **terminal-valid completion** as a request that reaches the intended terminal condition, passes model/API validation and has no unhandled stream interruption. Measure TTFT p95/p99, TPOT p95, E2E p95, queue wait, 429/error rate, terminal-valid goodput, GPU utilization, VRAM/KV pressure and GPU-hours per valid completion. Do not claim success from GPU utilization alone.

## 7. Capacity math

```text
required warm GPU equivalents =
 ceil(peak admitted token-work / measured sustainable token-work per GPU)
 + failure/maintenance reserve

N-1 condition:
 sum(usable zone capacity) - largest zone capacity
 >= peak GPU demand + maintenance reserve
```

GPU equivalents require calibration for MIG, time-slicing, model size and batching. Physical GPUs and logical advertised shares are not interchangeable.

## 8. Cold-start budget

```text
queue observation → HPA/KEDA decision → node provision
→ kubelet Ready → GPU allocatable → image pull
→ model download → CUDA warmup → readiness
```

Record actual p50/p95 durations for each stage. Minimum warm capacity must reflect user-facing TTFT objectives; scale-to-zero may be unsuitable for interactive inference.

## 9. Safe-read evidence commands

```bash
date -u
kubectl config current-context
kubectl version --client
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get nodes --show-labels
kubectl get pods -A -o wide
kubectl get resourcequota,limitrange -A
kubectl get hpa -A
kubectl get pdb -A
kubectl get networkpolicy -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get crd | grep -Ei 'scaledobject|nodepool|nodeclaim|karpenter'
kubectl describe node APPROVED_GPU_NODE
kubectl -n APPROVED_NS describe pod APPROVED_GPU_POD
```

Do not assume installed CRDs, metrics APIs or privileges. Scope commands to authorized namespaces where required.

## 10. Staging execution sequence [APPROVAL REQUIRED]

1. Freeze baseline manifests and obtain rollback owner approval.
2. Verify GPU allocatable, tenant policies and cloud quota.
3. Deploy approved synthetic model replica(s) with immutable digests.
4. Measure baseline terminal-valid inference and cost.
5. Introduce bounded synthetic burst and record metric-to-replica lag.
6. Confirm GPU Pod Pending → node provisioning → warmup critical path.
7. Exercise approved tenant quota/placement negative tests.
8. Exercise stream-safe scale-down without truncating active requests.
9. Simulate zone loss **only if approved**; otherwise use tabletop evidence.
10. Restore original configuration and verify SLO, capacity and GPU spend.

Never inject failures into production or a shared staging cluster without explicit authorization.

## 11. HPA/KEDA acceptance

Validate queue/token metric semantics, Prometheus freshness, custom/external metrics APIs, correct scale target ownership, min/max replicas, stabilization windows, scale-to-zero policy and absence of competing controllers. A correct HPA recommendation is not proof that GPU nodes or model replicas are ready.

## 12. GPU node provisioning acceptance

Verify CA or Karpenter ownership, compatible node templates, taints/labels, cloud GPU quota, zone capacity, driver/plugin registration, model artifact IAM/KMS and measured cold-start. Check that scale-out does not exceed GPU-hour ceiling.

## 13. Isolation acceptance

Audit ResourceQuota, RBAC, Pod Security, NetworkPolicy, workload identity, model artifact IAM, tenant token/concurrency budgets and GPU sharing/MIG policy. Namespaces, taints and time-slicing alone are not tenant security boundaries.

## 14. Disruption acceptance

Inspect PDB selector, `DisruptionsAllowed`, zone spread, N-1 headroom, graceful termination and streaming drain. PDB does not prevent involuntary node or zone failure. Verify warm replacement before planned eviction.

## 15. Incident response and rollback

Freeze changes, preserve events/metrics, classify admission versus Pending versus runtime/model warmup, identify owner, execute smallest approved remediation, canary recovery and compare against pre-change SLO. Restore GitOps source of truth and reconcile orphaned GPU nodes.

## 16. Twenty-four production failure scenarios

| ID | Failure | Evidence |
|---|---|---|
| F01 | GPU device plugin absent | DaemonSet |
| F02 | GPU allocatable zero | node |
| F03 | GPU driver/CUDA mismatch | runtime |
| F04 | wrong MIG profile | resource keys |
| F05 | time-slicing VRAM contention | OOM |
| F06 | tenant GPU quota denial | admission |
| F07 | CPU/memory quota denial | admission |
| F08 | RBAC/Pod Security denial | policy |
| F09 | taint mismatch | scheduler |
| F10 | required affinity impossible | events |
| F11 | topology spread deadlock | Pending |
| F12 | PVC zone mismatch | PVC |
| F13 | custom metrics adapter absent | HPA |
| F14 | KEDA metric stale | scaler |
| F15 | competing HPA ownership | audit |
| F16 | cloud GPU quota exhausted | autoscaler |
| F17 | Karpenter/CA node launch failure | controller |
| F18 | GPU plugin late after node Ready | allocatable |
| F19 | model download/IAM failure | startup |
| F20 | CUDA warmup breaches SLO | readiness |
| F21 | scale-down truncates streams | gateway |
| F22 | zone loss exceeds N-1 reserve | capacity |
| F23 | rollback leaves GitOps drift | diff |
| F24 | orphaned GPU nodes exceed budget | FinOps |

## 17. Sixteen operational runbooks

1. Baseline inventory and freeze.
2. Restore driver/device-plugin GPU registration.
3. Recover MIG/profile and sharing policy.
4. Recover tenant GPU/CPU/memory quota.
5. Restore RBAC/Pod Security/workload identity.
6. Diagnose taint, selector and affinity Pending.
7. Resolve topology/PVC zone deadlock.
8. Restore custom/external metrics adapter.
9. Recover KEDA and metric freshness.
10. Resolve HPA ownership and oscillation.
11. Recover CA/Karpenter/cloud GPU capacity.
12. Restore model artifact download and warmup.
13. Protect streaming scale-down and PDB.
14. Recover zone outage and N-1 headroom.
15. Reconcile orphaned GPU spend and FinOps.
16. Execute end-to-end rollback, RCA and acceptance sign-off.

## 18. Eight acceptance game days

1. GPU device plugin disappears from one node.
2. Tenant GPU quota blocks a critical rollout.
3. Required GPU affinity cannot be satisfied.
4. Prometheus/KEDA metric becomes stale during burst.
5. Cloud quota prevents GPU scale-out.
6. Model warmup exceeds TTFT objective.
7. Scale-down interrupts active streams.
8. One zone becomes unavailable and rollback leaves drift.

Tabletop by default; live injection requires explicit staging authorization.

## 19. Twenty acceptance gates

| Gate | Required evidence | Status |
|---|---|---|
| G01 | Cluster/versions and scope | Pending |
| G02 | GPU driver/plugin healthy | Pending |
| G03 | GPU allocatable/allocated | Pending |
| G04 | MIG/sharing policy | Pending |
| G05 | Tenant ResourceQuota | Pending |
| G06 | RBAC/Pod Security/identity | Pending |
| G07 | Taints/affinity | Pending |
| G08 | Topology/PVC | Pending |
| G09 | HPA/KEDA ownership | Pending |
| G10 | Custom metric correctness | Pending |
| G11 | GPU node autoscaler | Pending |
| G12 | Cloud GPU quota/zone capacity | Pending |
| G13 | Image/model/CUDA warmup | Pending |
| G14 | TTFT/TPOT/goodput SLO | Pending |
| G15 | Streaming-safe drain | Pending |
| G16 | PDB and N-1 resilience | Pending |
| G17 | Fault game-day results | Pending |
| G18 | Complete rollback | Pending |
| G19 | GPU-hours/FinOps | Pending |
| G20 | Independent technical/security approval | Pending |

**Pass rule:** All required gates evidenced and reviewed; any failed, skipped or unexecuted mandatory gate prevents operational acceptance. An exception requires documented owner, risk and expiry and does not silently count as a pass.

## 20. Capstone deliverables

Produce an environment inventory, approved manifests, sanitized event/metric bundle, cold-start timeline, load-test report, zone capacity worksheet, game-day record, rollback diff, GPU cost sheet and signed acceptance matrix.

See [companion acceptance lab](../labs/055-gpu-autoscaling-acceptance-lab-check.md).

**Part 07 written curriculum complete (10/10); execution and reviews pending. Next: Chapter 056 — Golden Signals for LLM Inference.**
