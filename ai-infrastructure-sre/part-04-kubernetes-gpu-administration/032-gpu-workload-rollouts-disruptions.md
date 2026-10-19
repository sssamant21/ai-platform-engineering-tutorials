# Chapter 032 — GPU Workload Rollouts and Disruptions

**AI Infrastructure Engineering & SRE · Part 04 — Kubernetes GPU Administration · Chapter 032/100**

**Status:** Published. Offline tests, live GPU staging, technical/security review and production acceptance are separate pending milestones.

## Objectives

Operate safe Kubernetes GPU workload releases and disruption recovery for inference and training. Understand Deployment strategies, StatefulSets, Jobs, distributed training ranks, checkpoint durability, PDBs, node drain, autoscaler consolidation, GPU Operator/driver upgrades, Karpenter disruption, rollback and SLO-driven acceptance. Diagnose failed rollouts and run a reproducible fail-closed acceptance lab.

## 1. Workload architecture and disruption boundaries

```text
GitOps / CI signed immutable image + manifest
  -> API admission (RBAC, PSA, quota, policy)
  -> Deployment / StatefulSet / Job / training controller
  -> scheduler (GPU resource, affinity, taints, topology)
  -> kubelet -> NVIDIA device plugin -> CDI/toolkit
  -> CUDA / NCCL / inference service
  -> readiness, application SLO, checkpoint / durable storage
  -> rollout / rollback / incident commander

Node disruptions:
  maintenance drain | GPU Xid | Spot interruption
  Karpenter consolidation/drift | AZ loss | driver upgrade
```

A Kubernetes Pod becoming Ready does **not** guarantee model correctness, inference p99 latency, distributed rank coordination, NCCL progress or checkpoint consistency. Treat Kubernetes availability, GPU health and application correctness as separate acceptance dimensions.

## 2. Workload controller selection

| Workload | Controller | Release concern |
|---|---|---|
| stateless GPU inference | Deployment | surge capacity, readiness, p99, model warm-up |
| stable identity / durable local state | StatefulSet | ordered replacement, PVC/identity |
| batch training | Job or supported training operator | checkpoint, completion, restart |
| distributed training | supported training operator / orchestrator | coordinated ranks, collective restart |
| GPU node platform | DaemonSet | driver/toolkit/device-plugin rollout and host impact |

**Do not use a generic Deployment rolling update as a safe substitute for coordinated distributed-training restart.** Each rank may require compatible process-group membership, checkpoint version and collective recovery.

## 3. Deployment rollout strategy

For GPU inference, `maxSurge` consumes extra GPU capacity; `maxUnavailable` trades availability for capacity. A rollout with `maxSurge: 1` and no spare GPU may stall indefinitely. A rollout with `maxUnavailable: 1` may temporarily reduce inference capacity and breach SLO. Choose based on measured headroom and load shedding, not a copied default.

Illustrative staging-only Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gpu-inference-canary
  namespace: gpu-staging
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: gpu-inference-canary
  template:
    metadata:
      labels:
        app: gpu-inference-canary
    spec:
      terminationGracePeriodSeconds: 120
      containers:
        - name: inference
          image: registry.example.invalid/inference@sha256:REPLACE_WITH_APPROVED_DIGEST
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            periodSeconds: 10
          startupProbe:
            httpGet:
              path: /ready
              port: 8080
            failureThreshold: 60
            periodSeconds: 5
          resources:
            requests:
              cpu: "2"
              memory: 8Gi
              nvidia.com/gpu: 1
            limits:
              memory: 16Gi
              nvidia.com/gpu: 1
```

**Non-deployable template:** replace image, endpoint, digest, securityContext, node selectors, tolerations, identity, resources and probes with reviewed application-specific values. Readiness must reflect actual model warm-up and ability to serve requests, not merely TCP listening. Use appropriate non-root Pod Security settings, service account controls and immutable artifacts. Ensure startup probes cover realistic model load time.

## 4. PodDisruptionBudget (PDB) semantics

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: gpu-inference-pdb
  namespace: gpu-staging
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: gpu-inference-canary
```

PDBs constrain **voluntary evictions through the eviction API** (such as a normal drain), not every pod termination. They do not prevent hardware failure, GPU Xid, OOM, node crash, direct deletion or all cloud interruption paths. A PDB can also block maintenance if replica count or readiness is insufficient. Confirm controller and drain behavior before relying on it.

## 5. Node drain and GPU maintenance

Read-only preflight:

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide --field-selector spec.nodeName=APPROVED_GPU_NODE
kubectl get pdb -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe node APPROVED_GPU_NODE
kubectl get daemonsets -A | grep -Ei 'nvidia|gpu-operator'
```

Authorized staging maintenance, after checkpoint and capacity approval:

```bash
kubectl cordon APPROVED_GPU_NODE
kubectl drain APPROVED_GPU_NODE --ignore-daemonsets --timeout=15m
# Verify workload recovery, GPU health and replacement capacity.
kubectl uncordon APPROVED_GPU_NODE
```

**Do not add `--force`, `--delete-emptydir-data` or `--disable-eviction` reflexively.** Review unmanaged pods, local data, PDBs, finalizers and checkpoint integrity. Draining does not automatically migrate GPU memory or training process state.

## 6. Training checkpoint and recovery contract

Define durable checkpoint storage, checkpoint version/schema, atomic commit or manifest publication, rank coordination, optimizer state, RNG state, dataset progress, model weights, object-storage consistency, checksum and encryption. Set RPO as maximum acceptable training progress lost; RTO as maximum recovery time. For distributed training, require coordinated termination/restart and NCCL communicator reinitialization where needed.

Record `last_verified_checkpoint_utc`, age, restore test, artifact digest and resume throughput. A checkpoint file existing does not prove restorable recovery.

## 7. Karpenter, Spot and GPU Operator disruptions

Karpenter consolidation/drift, node termination and Spot interruption may cause evictions or abrupt loss. Disruption budgets and do-not-disrupt annotations are version-specific and not absolute protections. GPU Operator driver/toolkit changes may require draining/reboot and have host-level rollback requirements beyond Helm rollback. Separate workload rollout ownership from GPU platform maintenance ownership.

**N-1:** verify remaining eligible GPU capacity by resource type, zone, CPU/memory, affinity, quotas and NCCL topology. A spare physical GPU in another incompatible node pool is not usable headroom.

## 8. Rollout observability

| Signal | Acceptance criterion |
|---|---|
| updated/ready/available replicas | controller converged |
| rollout progress deadline | no stalled rollout |
| GPU allocatable / requested | sufficient surge/recovery headroom |
| GPU UUID and CUDA smoke | expected device and runtime |
| inference p50/p95/p99, error rate | within approved SLO |
| model load/warm-up time | readiness reflects real service |
| training step progress / NCCL heartbeat | no deadlock or silent stall |
| checkpoint age and restore result | RPO/RTO protected |
| Xid/ECC/DCGM health | no hardware regression |
| node disruption and evictions | within budget |
| telemetry freshness | evidence current |

Rollback triggers should be explicit and automated where appropriate; a Kubernetes rollout undo may not revert database schema, model artifact, driver or checkpoint format. Use compatible artifacts and staged promotion.

## 9. Twelve troubleshooting scenarios

1. Rolling update stalls because no surge GPU capacity exists.
2. Readiness succeeds before model warm-up; inference 5xx spikes.
3. PDB blocks planned node drain.
4. PDB fails to protect an involuntary node loss.
5. Karpenter consolidation interrupts a training rank.
6. Spot termination arrives before checkpoint completion.
7. Training resumes with incompatible optimizer/checkpoint schema.
8. GPU Operator driver upgrade breaks CUDA compatibility.
9. Node GPU resource key changes after MIG reconfiguration.
10. Deployment rollout succeeds but NCCL distributed workers hang.
11. Failed readiness causes repeated rollout and resource contention.
12. Rollback restores pods but model quality or p99 remains degraded.

## 10. Ten production incident simulations

1. Insufficient GPU headroom for zero-unavailable rollout.
2. Inference p99 regression during canary.
3. PDB deadlock during node maintenance.
4. Unexpected GPU Xid forces immediate node loss.
5. Karpenter drift evicts a critical training worker.
6. Spot interruption races checkpoint commit.
7. Multi-rank training restart loses collective consistency.
8. Driver rollout makes model container incompatible.
9. Rollback points to unavailable model artifact.
10. AZ failure exceeds N-1 GPU reserve.

Capture UTC timeline, pod UID, GPU UUID, node, rollout revision, model/checkpoint digest, user impact, RPO/RTO, SLO, rollback and RCA/COA.

## 11. Eight runbooks

**RB-032-A — Rollout preflight:** verify approved image/model digest, quota, surge GPU headroom, SLO and rollback.

**RB-032-B — Stalled rollout:** inspect Deployment conditions, Pending events, allocatable GPU, affinity and readiness.

**RB-032-C — Safe GPU drain:** checkpoint, validate PDB, cordon/drain and prove workload recovery.

**RB-032-D — Spot/Karpenter interruption:** correlate termination signals, checkpoint age, worker coordination and replacement.

**RB-032-E — Training rank recovery:** validate durable checkpoint and restart collective participants consistently.

**RB-032-F — Driver/operator change:** canary, compatibility, node-image rollback, CUDA/NCCL verification.

**RB-032-G — Inference rollback:** shift traffic to known-good artifact, verify model correctness and p99/error budget.

**RB-032-H — Incident closure:** restore SLO, reconcile checkpoint/RPO, validate N-1 and record approvals.

## 12. Hands-on staging labs

1. Inventory GPU node supply, workload controller, PDB, readiness, image/model digest and SLO.
2. Render a pinned-image Deployment with surge budget and realistic warm-up probes.
3. Perform an authorized staging rollout and record rollout progress, latency and GPU UUID.
4. Simulate insufficient surge capacity **offline** and verify fail-closed policy.
5. Test a controlled drain with PDB and checkpoint evidence.
6. Validate restart from a durable checkpoint for a disposable training job.
7. Simulate canary SLO regression and perform approved rollback.
8. Record RPO/RTO, N-1, security review, cost impact and signoff.

[Lab 032 — GPU Workload Rollout and Disruption Gate](../labs/032-gpu-workload-rollouts-disruptions-check.md) implements a Python 3.11+ offline evaluator and pytest suite. It does not contact Kubernetes or provision GPUs.

## 13. 100-point assessment

| Domain | Points |
|---|---:|
| Controller and rollout architecture | 20 |
| PDB, disruption and capacity planning | 20 |
| Checkpoint and recovery correctness | 20 |
| Troubleshooting, incidents and runbooks | 20 |
| Tests, staging and SLO evidence | 20 |
| **Total** | **100** |

Pass >=90 with no unresolved data-loss, GPU correctness, tenant-security, rollback or SLO blocker.

## References

- [Kubernetes Deployment Rolling Updates](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Kubernetes PodDisruptionBudgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [Kubernetes Safely Drain a Node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [Karpenter Disruption](https://karpenter.sh/docs/concepts/disruption/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)

**Next: Chapter 033 — GPU Operator and Device Plugin Troubleshooting.**
