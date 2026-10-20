# Lab 086 — N-1 and Multi-Zone GPU Resilience Acceptance Gate

**Chapter:** [086 — N-1 and Multi-Zone GPU Resilience](../part-11-capacity-finops-and-reliability/086-n-1-multi-zone-gpu-resilience.md)

**Execution status:** NOT RUN. Offline modeling, isolated staging failure drills, independent technical/security review and acceptance are pending.

## 1. Safety contract

Read-only production inventory is permitted only with authorization. Failure injection is **staging-only**, isolated and bounded; no live AZ disablement, production node drain, GPU reset, fabric disruption or checkpoint deletion. Obtain platform, serving, ML workload, network and storage approvals. Preserve last complete checkpoint and a tested rollback path.

## 2. Sixteen-exercise checklist

| ID | Exercise | Required evidence | State |
|---|---|---|---|
| E01 | GPU node/rack/AZ inventory | topology map | Pending |
| E02 | SLO/RTO/RPO | workload contracts | Pending |
| E03 | N-1 node calculation | math and assumptions | Pending |
| E04 | AZ-loss calculation | worst-zone result | Pending |
| E05 | Compatible node bundles | SKU/HBM/MIG feasibility | Pending |
| E06 | Topology constraints | manifests and events | Pending |
| E07 | PDB review | voluntary/involuntary distinctions | Pending |
| E08 | Inference failover | peak p99/error evidence | Pending |
| E09 | Checkpoint restore | checksum and lost steps | Pending |
| E10 | Gang job restart | rank/world-size evidence | Pending |
| E11 | Quota/reservation audit | per-zone capacity | Pending |
| E12 | Shared dependency review | zonal SPOF map | Pending |
| E13 | Failure-domain alerting | alert delivery proof | Pending |
| E14 | Staging node-loss drill | recovery timeline | Pending |
| E15 | Synthetic AZ-loss tabletop | abort/rollback decision | Pending |
| E16 | Independent review | signed score and COAs | Pending |

## 3. Offline N-1 and AZ-loss calculation

\`\`\`python
nodes_per_zone = {"az-a": 4, "az-b": 4}
gpus_per_node = 8
protected_inference_gpus = 24
non_preemptible_training_gpus = 16

total = sum(nodes_per_zone.values()) * gpus_per_node
after_one_node = total - gpus_per_node
survivors = {
    failed: sum(n*gpus_per_node for zone, n in nodes_per_zone.items() if zone != failed)
    for failed in nodes_per_zone
}
worst_survivor = min(survivors.values())
print(total, after_one_node, survivors, worst_survivor)
assert (total, after_one_node, worst_survivor) == (64, 56, 32)
assert protected_inference_gpus <= worst_survivor
assert protected_inference_gpus + non_preemptible_training_gpus > worst_survivor
\`\`\`

Expected: 64 installed, 56 after node failure, 32 after either AZ loss. **Inference-only aggregate capacity passes**, but simultaneous 24 inference + 16 non-preemptible training GPUs fails the AZ-loss aggregate test. Neither outcome proves GPU SKU, model memory, node-bundle, fabric or SLO feasibility.

## 4. Read-only Kubernetes checks

\`\`\`bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get pods -A -o wide
kubectl get pdb -A
kubectl get priorityclass
kubectl get resourcequota -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl -n <namespace> get deployment <inference-deployment> -o yaml
\`\`\`

For GPU devices, collect authorized \`nvidia-smi -L\` and node device-plugin telemetry. Do not infer a GPU SKU from node name alone.

## 5. Offline tabletop fixtures

**Scenario A — Node failure:** 8-GPU node disappears during an 8-rank job. Determine checkpoint-safe restart and protected serving impact.

**Scenario B — AZ loss:** 32 GPUs survive, with 24 inference and 16 non-preemptible training GPU demand. Explain which policy must change and who approves it.

**Scenario C — Wrong GPU SKU:** spare nodes have insufficient HBM. Reject misleading aggregate capacity pass.

**Scenario D — Shared checkpoint storage:** only one AZ contains checkpoint volume. Calculate RPO/RTO risk and propose verified cross-zone artifact replication.

**Scenario E — PDB misconception:** node fails involuntarily despite PDB. Explain why PDB is not an HA guarantee.

## 6. Staging game-day evidence

\`\`\`yaml
chapter: "086"
environment: "OFFLINE_OR_STAGING"
execution_date_utc: null
incident_commander: null
platform_owner: null
ml_owner: null
security_reviewer: null
node_topology: []
protected_slo: null
rto_target_seconds: null
rpo_target_steps: null
node_loss_drill: "NOT_RUN"
az_loss_tabletop: "NOT_RUN"
inference_peak_p99_test: "NOT_RUN"
checkpoint_restore: "NOT_RUN"
gang_restart: "NOT_RUN"
capacity_reservation_review: "PENDING"
rollback_validation: "NOT_RUN"
independent_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 7. Acceptance

Score **15 + 25 + 20 + 15 + 15 + 10 = 100**. Pass at ≥85 with no critical unsafe change, checkpoint loss, privacy breach, unverified zone capacity or missing rollback. A tabletop or published manifest is **not** proof of live failover.

**Next:** Chapter 087 — GPU Cost Allocation and Unit Economics.
