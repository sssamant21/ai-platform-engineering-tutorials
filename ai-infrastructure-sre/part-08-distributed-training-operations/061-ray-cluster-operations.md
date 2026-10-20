# Chapter 061 — Ray Cluster Operations

**AI Infrastructure Engineering & SRE · Part 08 — Distributed Training Operations · Chapter 061/100**

> **Validation boundary:** Reference architecture, Kubernetes examples and offline synthetic acceptance only. No live KubeRay cluster, GPU autoscaling, object-store failure or disaster-recovery test has been performed.

## Objectives and prerequisites

Operate Ray as a production distributed AI execution platform on Kubernetes: Ray head and workers, Global Control Store (GCS), KubeRay operator, RayCluster/RayJob, GPU resource scheduling, placement groups, autoscaling, object-store memory/spilling, fault tolerance, Ray Train checkpoints, tenant isolation and SRE observability. Compare with Slurm and PyTorch DDP.

**Prerequisites:** Chapters 059–060, Kubernetes GPU administration, cloud quotas, NCCL networking, object storage and checkpoint recovery.

## 1. Enterprise reference architecture

```text
GitOps / approved RayJob / tenant identity
                |
    Kubernetes API + KubeRay operator
                |
     RayCluster / RayJob / RayService
                |
        Ray head pod (GCS)
        | control-plane state, jobs, dashboard
        |
  +-----+--------------+--------------+
  | Ray GPU worker A   | Ray GPU worker B
  | actors/tasks       | actors/tasks
  | object store       | object store
  | 8 GPUs             | 8 GPUs
  +-----+--------------+--------------+
        | placement groups / GPU resources
        | RDMA/NCCL where applicable
        v
 dataset FS / object store / checkpoints
        |
 DCGM / Prometheus / Ray metrics / OTel
        |
 on-call, incident command, SLO, FinOps
```

Ray logical GPU resources control scheduling, not physical isolation by themselves. Kubernetes GPU device plugin and container security must enforce device assignment.

## 2. Ray control plane and failure domains

Ray head provides control-plane services including GCS. Workers execute tasks and actors and maintain distributed object-store state. Head loss, GCS durability, worker eviction and Kubernetes operator failure have different blast radii. Confirm the chosen Ray/KubeRay versions and supported GCS fault-tolerance configuration; do not assume a restarted head transparently restores every job or actor.

| Component | Ownership | Key SLI |
|---|---|---|
| KubeRay operator | reconciles Ray custom resources | reconciliation errors |
| Ray head / GCS | metadata, scheduling and control | head/GCS availability |
| Ray workers | CPU/GPU tasks and actors | available resource slots |
| Placement groups | co-scheduling bundles | pending time |
| Object store | in-memory object references and transfers | utilization/spill latency |
| Ray Jobs | submit and track workloads | completion rate |
| Ray Train | distributed training coordination | restart/checkpoint success |
| Kubernetes autoscaler | provisions physical nodes | GPU node ready latency |

## 3. KubeRay Kubernetes deployment pattern

Illustrative RayCluster fragment; pin compatible Ray/KubeRay versions, image digests, service accounts, resource limits and namespace policies before deploying:

```yaml
apiVersion: ray.io/v1
kind: RayCluster
metadata:
  name: gpu-ray-example
spec:
  rayVersion: "<pin-supported-ray-version>"
  headGroupSpec:
    serviceType: ClusterIP
    rayStartParams: {}
    template:
      spec:
        serviceAccountName: ray-workload
        containers:
          - name: ray-head
            image: "<approved-ray-image-digest>"
            resources:
              requests: {cpu: "2", memory: 4Gi}
              limits: {cpu: "2", memory: 4Gi}
  workerGroupSpecs:
    - groupName: gpu-workers
      replicas: 2
      minReplicas: 1
      maxReplicas: 4
      rayStartParams: {}
      template:
        spec:
          serviceAccountName: ray-workload
          containers:
            - name: ray-worker
              image: "<approved-ray-image-digest>"
              resources:
                requests: {cpu: "8", memory: 32Gi, nvidia.com/gpu: "1"}
                limits: {cpu: "8", memory: 32Gi, nvidia.com/gpu: "1"}
```

**Caution:** This is a design fragment, not a ready-to-apply production manifest. Version-specific required fields, autoscaler flags, persistent GCS state, pod security, probes, topology constraints, object-store sizing and service exposure must be validated against the installed CRD.

## 4. Read-only administration

```bash
kubectl get rayclusters,rayjobs -A
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl get nodes -o wide
kubectl describe raycluster gpu-ray-example -n <approved-namespace>
kubectl get deployment -A | grep -i kuberay
```

On an authorized Ray head, `ray status` and `ray list nodes` can provide useful inventory if the installed Ray CLI supports them. Protect dashboard and Jobs API with private network access and authenticated ingress; do not expose unauthenticated endpoints publicly.

## 5. GPU scheduling, placement groups and topology

Use explicit GPU requests, placement bundles, resource labels and affinity to express co-location or spreading. Ray's placement groups reserve logical bundles but do not inherently guarantee NUMA/NVLink/NIC locality; Kubernetes placement and actual node topology must also align. Confirm that each Ray worker advertises the GPUs actually isolated by the container runtime.

## 6. Autoscaling layers and quota management

Ray autoscaling responds to Ray resource demand; Kubernetes node autoscaling/Karpenter provisions actual machines. Define minimum/maximum worker replicas, cloud GPU quotas, startup time, consolidation safeguards and preemption behavior. Avoid conflicting autoscaler ownership and uncontrolled oscillation. A pending placement group can be caused by fragmentation, impossible resource shapes or exhausted cloud capacity.

## 7. Object store and data pipeline reliability

Ray's distributed object store uses memory and may spill to disk. Plan object size, pinned references, spill paths, local disk headroom, throughput, cleanup and cross-node transfer traffic. A spill storm can appear as GPU starvation and create storage/network contention with NCCL and checkpoint writes. Monitor object store utilization, spilled bytes, failures, task queue and per-worker memory.

## 8. Actors, tasks, retry semantics and checkpoints

Ray task/actor retry behavior is configurable and depends on failure mode and API; it does not make side effects exactly-once. Ensure idempotent writes, stable dataset IDs and checkpoint barriers. Ray Train restart/recovery must use a supported checkpoint format, durable storage, globally consistent state and tested dataset replay. A restarted actor is not automatically a recovered training step.

## 9. Observability and enterprise SLOs

Track Ray job success, task retries, actor restarts, placement pending p95, head/GCS availability, autoscaler scale-up p95, object spill rate, GPU utilization, NCCL communication, checkpoint RPO, restore RTO, telemetry freshness and $/successful training job. Correlate Ray job ID, Kubernetes namespace, worker pod, rank, node, dataset manifest and UTC.

## 10. Security and governance

Apply scoped Kubernetes service accounts and workload identity, RBAC, namespace quotas, network policies, image provenance, encryption, secret management and least-privilege checkpoint access. Limit dashboard/Jobs API exposure; treat arbitrary Ray job submission as remote code execution capability and restrict it to trusted users.

## 11. Fifteen troubleshooting scenarios

1. Ray head pod restarts and GCS state cannot be recovered.
2. KubeRay operator reconciles a stale RayCluster specification.
3. GPU workers join but advertise zero GPU resources.
4. Ray placement group remains pending under fragmentation.
5. Autoscaler cannot provision GPU nodes due to cloud quota.
6. Kubernetes Cluster Autoscaler/Karpenter and Ray autoscaler disagree.
7. Ray object store fills and spills overwhelm parallel storage.
8. Large object transfers saturate NICs and starve NCCL.
9. Actor restarts replay non-idempotent side effects.
10. Ray Job fails but Kubernetes pods remain healthy.
11. Worker node loss leaves orphaned tasks or placement bundles.
12. Ray dashboard or Jobs API is reachable without proper access control.
13. Mixed Ray/Python versions cause serialization failures.
14. Checkpoint is available but dataset cursor is inconsistent.
15. Prometheus metrics disappear during a cascading restart.

For each record symptom, SLI, read-only inspection, hypothesis, containment, restore, rollback and owner.

## 12. Ten incident simulations

1. **INC-061-01:** Head/GCS failure.
2. **INC-061-02:** Operator reconciliation drift.
3. **INC-061-03:** GPU discovery failure.
4. **INC-061-04:** Placement-group deadlock.
5. **INC-061-05:** GPU autoscaling quota exhaustion.
6. **INC-061-06:** Object store OOM and spill storm.
7. **INC-061-07:** Worker eviction during training.
8. **INC-061-08:** Non-idempotent actor restart.
9. **INC-061-09:** Tenant network/security boundary failure.
10. **INC-061-10:** Monitoring outage during checkpoint RPO breach.

Use isolated staging and explicit authorization for disruptive tests. Capture incident commander, UTC timeline, impacted jobs, data-loss window, RPO/RTO, GPU-hours lost and RCA/COA.

## 13. Eight production runbooks

1. **RB-061-A — Head and GCS triage:** Inspect Ray head, GCS health, recovery configuration and pod events.
2. **RB-061-B — Worker and GPU discovery:** Compare Ray advertised GPUs, Kubernetes resources and nvidia-smi.
3. **RB-061-C — Pending placement groups:** Inspect bundles, node shape, quotas, topology and autoscaler events.
4. **RB-061-D — Autoscaler and Karpenter:** Identify which layer owns worker replicas versus physical node capacity.
5. **RB-061-E — Object store and spilling:** Inspect object memory, spill I/O, disk headroom and object lineage.
6. **RB-061-F — Actor/task recovery:** Confirm retry semantics, idempotency, checkpoint and data cursor.
7. **RB-061-G — Security and tenant isolation:** Validate dashboard exposure, RBAC, workload identity and network policy.
8. **RB-061-H — Incident closure:** Measure job SLO, RPO/RTO, rollback, audit and independent review.

## 14. Twelve hands-on labs

1. Draw KubeRay, head/GCS, GPU workers, object store and failure domains.
2. Inspect a RayCluster CRD and review GPU resource limits.
3. Inventory Ray jobs, placement groups and worker resources.
4. Simulate a GPU placement-group scheduling deadlock.
5. Model Ray worker scaling and Kubernetes GPU node capacity.
6. Model object store spill and disk capacity headroom.
7. Review task/actor idempotency and retry policies.
8. Simulate Ray head/GCS failure and recovery constraints.
9. Verify dataset and checkpoint lineage for a Ray Train job.
10. Simulate network isolation, dashboard exposure and quota failure.
11. Run the companion offline fail-closed policy and pytest suite.
12. Produce incident evidence and a 100-point readiness review.

**Companion:** [Lab 061 — Ray Cluster Operations Gate](../labs/061-ray-cluster-operations-check.md).

## 15. Enterprise acceptance — 100 points

| Domain | Points |
|---|---:|
| KubeRay control plane, GCS and RayJob lifecycle | 20 |
| GPU scheduling, autoscaling and object-store performance | 20 |
| Fault tolerance, checkpoints and RPO/RTO | 20 |
| Security, monitoring and incident response | 20 |
| Offline tests, staging evidence and independent review | 20 |
| **Total** | **100** |

Require >=90 and zero critical control-plane, GPU isolation, object-store, security, checkpoint or rollback failures. Offline policy passing is not production acceptance.

## References

- [Ray documentation](https://docs.ray.io/en/latest/)
- [KubeRay overview](https://docs.ray.io/en/latest/cluster/kubernetes/index.html)
- [Ray scheduling and placement groups](https://docs.ray.io/en/latest/ray-core/scheduling/placement-group.html)
- [Ray fault tolerance](https://docs.ray.io/en/latest/ray-core/fault_tolerance.html)
- [Kubernetes GPU scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)

**Next: Chapter 062 — PyTorch Distributed and NCCL.**
