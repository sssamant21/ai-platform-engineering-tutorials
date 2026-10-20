# Chapter 086 — N-1 and Multi-Zone GPU Resilience

**Part 11 — Capacity, FinOps and Reliability | AI Infrastructure Engineering & SRE**

**Status:** Published documentation; staging drills, live failover tests, independent security/technical review and production acceptance **pending**.

## 1. Mission

Design, calculate, implement and validate GPU platform fault tolerance for the loss of **one node (N-1)**, one accelerator, a rack, an availability zone (AZ), and selected shared dependencies. Protect customer-facing inference SLOs and checkpoint-safe distributed training recovery without assuming that a count of spare GPUs equals usable failover capacity.

**Prerequisites:** Chapters 075–085, Kubernetes scheduling and topology, GPU Operator/device plugin, Kueue, storage and network fundamentals, SLOs, incident command, cloud quotas and capacity reservations.

**Learning outcomes:** define precise failure domains; calculate N-1 and AZ-loss survivability; distinguish serving continuity from training restartability; audit placement and capacity reservations; perform bounded staging game days; and establish rollback, evidence and operational acceptance gates.

## 2. Resilience contract and failure domains

\`\`\`text
Region
 ├─ AZ-A
 │   ├─ rack / power / fabric-A
 │   │   ├─ GPU node A1 (8 GPUs)
 │   │   └─ GPU node A2 (8 GPUs)
 │   └─ storage/network dependencies
 ├─ AZ-B
 │   ├─ GPU node B1 (8 GPUs)
 │   └─ GPU node B2 (8 GPUs)
 └─ Shared services: API, identity, image registry, checkpoints,
     DNS, model artifacts, queues, observability, quota and control plane
\`\`\`

**N-1** must specify what \`N\` means: one GPU, one GPU node, one rack or one independent AZ. A platform surviving one failed GPU does not necessarily survive a failed 8-GPU node. An AZ-loss objective is not implied by N-1 node tolerance.

| Failure | Desired service behavior | Required evidence |
|---|---|---|
| Single GPU | workload isolation or restart on compatible device | UUID health and application progress |
| GPU node | serving replicas remain available; training safely recovers | node capacity, PDB, checkpoint |
| Rack/fabric | alternate topology avoids shared failure | topology and link map |
| AZ | protected inference SLO sustained; training RTO/RPO met | AZ-specific capacity and storage |
| Control plane | existing serving remains healthy; recovery operations bounded | dependency failover test |
| Checkpoint store | last committed checkpoint survives | cross-zone restore checksum |

**Availability objective:** SLI, SLO, RTO and RPO must be stated per workload. A training job can be recoverable without uninterrupted execution; inference usually requires continuous serving.

## 3. Capacity feasibility mathematics

For a homogeneous pool with \`N\` nodes of \`g\` GPUs each and protected demand \`D\`:

\[
C_{\mathrm{N-1}}=(N-1)g,\qquad D\leq C_{\mathrm{N-1}}
\]

This is a **necessary aggregate condition**, not a sufficient placement guarantee. A 16-GPU gang job may require two compatible nodes on the same high-bandwidth fabric island. Count complete node bundles and compatible GPU memory/SKU/MIG resources.

For AZ loss:

\[
C_{\mathrm{remaining}}=\sum_{z\ne z_{\mathrm{failed}}} C_z,\quad
D_{\mathrm{protected}}\leq\min_z C_{\mathrm{remaining}}(z)
\]

Evaluate each AZ-loss case separately and include capacity already consumed, placement constraints and failover overhead.

### Worked example: two zones, eight-GPU nodes

| Zone | Nodes | Installed GPUs | Protected inference |
|---|---:|---:|---:|
| AZ-A | 4 | 32 | 12 |
| AZ-B | 4 | 32 | 12 |
| Total | 8 | 64 | 24 |

- **N-1 node:** 56 installed GPUs remain, before health/fragmentation deductions.
- **AZ-A loss:** only 32 GPUs remain in AZ-B. If 24 inference GPUs are needed to serve *full* traffic after failure, 32 is an aggregate upper bound; verify traffic routing, replica placement and CPU/NIC/HBM headroom.
- If 16 additional training GPUs are **non-preemptible**, total protected demand is 40 and **AZ loss fails** the aggregate test (40 > 32). A documented training pause/checkpoint policy could reduce protected failover demand, but cannot be silently assumed.
- A capacity reservation in AZ-A cannot be treated as AZ-B supply. Confirm physical stock and cloud quota for failover zone.

## 4. Placement and topology

Use labels and node affinity to bind GPU SKU, memory, fabric topology, availability zone and dedicated pools. Prefer topology spread for inference replicas so one node/AZ cannot remove all serving capacity.

### Example: zone-aware inference spread

\`\`\`yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-gpu
  namespace: ai-serving
spec:
  replicas: 4
  selector:
    matchLabels: {app: inference-gpu}
  template:
    metadata:
      labels: {app: inference-gpu}
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels: {app: inference-gpu}
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels: {app: inference-gpu}
      containers:
        - name: server
          image: registry.example.invalid/ai/serving@sha256:REPLACE_WITH_REAL_DIGEST
          resources:
            limits:
              nvidia.com/gpu: "1"
\`\`\`

**Illustrative manifest only:** replace image with a valid pinned digest, add CPU/memory requests, readiness/startup probes, security context, service routing, real model configuration and access controls. Test against your cluster and scheduler. Hard spread constraints can intentionally leave pods Pending if the required zones are unavailable.

### PDB and termination behavior

A PodDisruptionBudget can constrain **voluntary evictions**, but does not prevent involuntary node/AZ failures. Readiness probes and graceful termination protect routing during planned rollouts; they cannot create capacity in a failed AZ.

## 5. GPU workload distinctions

### Online inference

- Minimum ready replicas across independent nodes and zones.
- Load-balancer health checks, traffic routing and zone evacuation behavior.
- Model warmup/cold start, HBM fit, KV cache and p99 at **survivor-zone peak load**.
- Autoscaler reaction time, reserved compatible GPU capacity and scale-from-zero risks.
- Model artifact availability, image pull rate, secrets, DNS and shared control-plane dependencies.
- Confirm correctness and safety policies remain active after failover.

### Distributed training

- Gang-scheduled ranks can fail together when one node disappears; **do not assume automatic rank healing**.
- Preserve last complete checkpoint including optimizer/scheduler/RNG/shard metadata and dataset version.
- Check world-size/topology compatibility before restarting; elastic training is framework-dependent.
- Set RPO as maximum acceptable lost training progress and RTO as time to restart and resume stable steps.
- Prioritize inference during AZ loss; training preemption only through approved checkpoint-safe controls.

## 6. Shared-dependency risk

Multi-zone GPUs are not resilient if all workloads depend on a single-zone:
- model registry or object-store endpoint;
- checkpoint volume or metadata service;
- NAT/DNS/identity path;
- RDMA fabric gateway;
- scheduler/controller;
- key management or secret provider.

Build a dependency graph with explicit failure domains and restore sequence. Validate object-store durability/availability semantics and cross-zone network cost/latency; avoid assuming a volume can attach across zones.

## 7. Cloud and on-prem capacity

Verify **physical supply**, account quota, reservation matching rules, node SKU, fabric availability and provisioning lead time in every target zone. Separate billing commitments from actual capacity guarantees.

For on-prem: validate rack/power diversity, NIC/fabric redundancy, spare GPU nodes, firmware compatibility, storage access and replacement lead time. Reserve whole compatible nodes when gang placement demands them.

## 8. Observability and SLO signals

- ready serving replicas by node/AZ and GPU SKU;
- compatible allocatable versus requested GPUs by zone;
- GPU health/Xid/ECC, node readiness and device-plugin registration;
- serving request success, p99, TTFT/TPOT and routing distribution;
- training checkpoint age, restart count, lost steps and queue delay;
- fabric packet loss/retries, object-store error rate and model load time;
- autoscaler/provisioner failures, quota denials and capacity reservation coverage.

Alert on **failure-domain headroom breach** before customer SLO degradation. Do not aggregate away a zero-capacity AZ behind healthy regional totals.

## 9. Six production runbooks

### A. Single GPU/device failure
**Detect:** UUID/Xid, device-plugin event, job error. **Mitigate:** isolate affected placement under change control. **Validate:** healthy CUDA smoke and unaffected service SLO. **Rollback:** stop unsafe changes; escalate hardware owner.

### B. GPU node failure
**Detect:** node NotReady and rank/serving loss. **Mitigate:** protect inference routing, check survivor node capacity, recover training from verified checkpoint. **Validate:** ready replicas, p99 and progress. **Escalate:** capacity shortage, repeated Xid or unsafe restart.

### C. AZ loss
**Detect:** zone-wide node/network failures. **Mitigate:** IC declares incident, route traffic to surviving AZ, enforce protected priority and capacity. **Validate:** survivor-zone peak SLO and checkpoint accessibility. **Rollback:** revert traffic only after zone stable and change approved.

### D. GPU quota/reservation shortfall
**Detect:** provisioning denied despite pending demand. **Diagnose:** physical stock vs quota vs reservation scope. **Mitigate:** approved compatible pool/region; protect inference. **Validate:** actual node/GPU availability.

### E. Shared checkpoint/model-store outage
**Detect:** model load/checkpoint errors. **Mitigate:** fail over to verified replicated artifact path; avoid unverified restore. **Validate:** checksum, RPO and restart. **Rollback:** return to known-good artifact.

### F. Multi-zone game day
**Preflight:** staging only, owners, blast radius, PDBs, rollback, capacity. **Inject:** synthetic zone unavailability or controlled isolated node stop. **Measure:** time to detection, traffic recovery, p99, RTO/RPO, checkpoint integrity. **Abort:** any unapproved impact.

## 10. Controlled game-day design

1. Choose **isolated staging** with at least two zones and synthetic workloads.
2. Capture baseline load, GPU health, topology and model/checkpoint versions.
3. Define explicit pass/fail thresholds and abort conditions before injecting failure.
4. Start with offline tabletop, then one staging node failure, then a simulated zone outage.
5. Validate surviving GPU availability and **actual placement**, not just arithmetic.
6. Measure end-user SLO, recovery time, checkpoint recovery and residual queue backlog.
7. Restore original configuration and prove no orphaned nodes, reservations or model versions.
8. Preserve redacted evidence, review with platform/ML/security and file COAs.

Never disable an actual production AZ, reset GPUs or disrupt shared fabric to demonstrate resilience.

## 11. Sixteen hands-on exercises

1. Map nodes, GPUs, MIG profiles, racks and zones.
2. Define SLO/RTO/RPO per workload class.
3. Calculate homogeneous N-1 node capacity.
4. Calculate worst-case AZ-loss remaining capacity.
5. Validate GPU SKU/HBM and node-bundle compatibility.
6. Inspect topology spread and affinity.
7. Inspect PDBs and voluntary versus involuntary disruptions.
8. Test synthetic inference survivor-zone peak traffic.
9. Test checkpoint integrity and restart.
10. Model gang scheduling failure under node loss.
11. Audit quotas and reservations per zone.
12. Map single-zone shared dependencies.
13. Build failure-domain dashboards and alerts.
14. Conduct staging node-loss game day.
15. Conduct synthetic zone-loss tabletop with rollback.
16. Score and peer-review acceptance package.

## 12. Twelve failure scenarios

| Scenario | Expected diagnosis |
|---|---|
| One GPU Xid 79 | hardware/device-level failure |
| GPU node NotReady | node loss and compatible spare nodes |
| AZ-A power loss | AZ-B survivor capacity |
| All replicas in one zone | invalid placement policy |
| PDB blocks planned drain | voluntary disruption protection |
| AZ failure bypasses PDB | involuntary loss distinction |
| Spare GPUs wrong SKU | incompatibility |
| Spare MIG slices fragmented | full-GPU job unschedulable |
| GPU quota approved, no stock | physical availability |
| Checkpoint only on zonal volume | DR gap |
| Inference p99 spikes after failover | survivor-zone load/warmup |
| Training rank disappears | checkpoint-safe gang restart |

## 13. Acceptance gate (100 points)

| Gate | Points |
|---|---:|
| Failure-domain inventory and dependency map | 15 |
| N-1 and AZ-loss math with placement feasibility | 25 |
| Serving SLO and training RPO/RTO design | 20 |
| Cloud quota, reservation and topology validation | 15 |
| Staging drills, rollback and evidence | 15 |
| Runbooks, security and independent review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 and no critical unapproved disruption, checkpoint integrity, tenant security, unsupported failover capacity or missing rollback. Mark runtime acceptance **PENDING** until real staging results and independent review exist.

## 14. References

- [Kubernetes Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Kubernetes Disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Kubernetes Assign Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [Kueue Documentation](https://kueue.sigs.k8s.io/)
- [Google SRE — Addressing Cascading Failures](https://sre.google/sre-book/addressing-cascading-failures/)

**Next:** Chapter 087 — GPU Cost Allocation and Unit Economics.
