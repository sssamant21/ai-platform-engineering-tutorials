# Lab 088 — Scaling and Queue Admission Control Acceptance Gate

**Chapter:** [088 — Scaling and Queue Admission Control](../part-11-capacity-finops-and-reliability/088-scaling-queue-admission-control.md)

**Status:** NOT EXECUTED. Offline calculations, isolated GPU staging, queue admission tests, independent technical/security review and acceptance remain pending.

## 1. Safety

Use synthetic traffic and isolated staging for scaling/admission tests. Production is read-only unless an independently approved change window exists. Never trigger a production scale-in, GPU node drain, queue preemption, quota modification or load storm. Keep rollback artifacts, checkpoints, workload owners and blast-radius limits.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | GPU node pools and priorities | inventory | Pending |
| E02 | Inference/queue metrics | signal definitions | Pending |
| E03 | SLO replica floor | benchmark and math | Pending |
| E04 | N-1 replica headroom | placement model | Pending |
| E05 | Backlog GPU-hours | queue model | Pending |
| E06 | Provisioning/cold start | lead-time distribution | Pending |
| E07 | HPA/KEDA integration | metrics adapter check | Pending |
| E08 | HPA schema review | staged manifest | Pending |
| E09 | Provisioner/quotas | Pending reason matrix | Pending |
| E10 | Kueue borrowing | ResourceFlavor/cohort model | Pending |
| E11 | Quota exhaustion | admission evidence | Pending |
| E12 | Inference overload | p99/rejection/backpressure | Pending |
| E13 | Training fairness | starvation analysis | Pending |
| E14 | Oscillation control | cooldown test | Pending |
| E15 | Safe scale-in | rollback and cost review | Pending |
| E16 | Peer review | score and open risks | Pending |

## 3. Offline serving replica sizing

\`\`\`python
from math import ceil
peak_rps = 900
slo_compliant_rps_per_replica = 150  # synthetic benchmark
replicas_per_failed_node = 2

base = ceil(peak_rps / slo_compliant_rps_per_replica)
n_minus_one_node = base + replicas_per_failed_node
print("base:", base, "N-1-node:", n_minus_one_node)
assert (base, n_minus_one_node) == (6, 8)
\`\`\`

Expected: **6 base replicas, 8 replicas for simplified one-node loss**. Assumes at most two replicas per node and equal replica throughput; real node/AZ placement and peak p99 must be tested.

## 4. Offline training backlog

\`\`\`python
jobs = [
    {"gpus": 8, "runtime_hours": 4},
    {"gpus": 16, "runtime_hours": 2},
    {"gpus": 4, "runtime_hours": 6},
]
gpu_hours = sum(j["gpus"] * j["runtime_hours"] for j in jobs)
print("queued_gpu_hours:", gpu_hours)
assert gpu_hours == 88
\`\`\`

Expected: **88 GPU-hours**. A 16-GPU gang job may be unschedulable on eight free GPUs even if backlog accounting looks favorable.

## 5. Read-only Kubernetes inventory

\`\`\`bash
kubectl config current-context
kubectl get hpa -A
kubectl get scaledobjects.keda.sh -A 2>/dev/null || true
kubectl get nodes -o wide
kubectl get pods -A --field-selector=status.phase=Pending
kubectl get priorityclass
kubectl get resourcequota -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

Inspect installed CRDs before querying Kueue and Karpenter resources; record \`NOT_INSTALLED\` if absent. Do not claim an HPA metric works without a registered custom/external metrics adapter.

## 6. Staging-only test matrix

| Test | Injection | Required outcome |
|---|---|---|
| Traffic burst | synthetic 2× request load | p99/rejection within approved policy |
| GPU shortage | simulate incompatible node pool | accurate Pending reason |
| Queue fairness | submit P1/P3 synthetic jobs | policy-respecting admission |
| Quota ceiling | request beyond staging limit | predictable denial |
| Metric lag | delay custom metric in test harness | no uncontrolled oscillation |
| Scale-in | remove synthetic demand | no lost requests/checkpoints |
| Node loss | staging node unavailable | protected serving floor survives |

Record measured times and SLO results; never mark PASS based on a design diagram.

## 7. Evidence template

\`\`\`yaml
chapter: "088"
environment: "OFFLINE_OR_STAGING"
execution_date_utc: null
platform_owner: null
ml_owner: null
security_reviewer: null
inference_slo_definition: null
benchmark_per_replica_rps: null
gpu_sku_and_topology: []
hpa_metric_adapter: "NOT_VERIFIED"
queue_admission_test: "NOT_RUN"
provisioning_lead_time_seconds: null
overload_test: "NOT_RUN"
n_minus_one_test: "NOT_RUN"
scale_in_rollback: "NOT_RUN"
cost_governance_review: "PENDING"
independent_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 8. Acceptance

Score **20 + 20 + 20 + 20 + 10 + 10 = 100**. Pass at ≥85 with no critical inference SLO breach, unauthorized priority/preemption, checkpoint loss, unbounded spend or untested rollback. Documentation publication is not staging acceptance.

**Next:** Chapter 089 — GPU Platform DR and Continuity.
