# Lab 097 — GPU Hardware and Driver Incident Exercises

**Chapter:** [097 — Run GPU Hardware and Driver Incident Exercises](../part-12-enterprise-operations-and-final-capstones/097-gpu-hardware-driver-incident-exercises.md)

**Execution:** NOT RUN. Staging fault simulations, actual recovery measurements, on-call routing tests and independent review remain pending.

## 1. Safety

Do not induce ECC faults, overheat GPUs, unload host drivers, alter firmware or drain production GPU nodes for practice. Prefer synthetic events, recorded logs and disposable staging workloads. Any live node cordon, GPU reset or operator change requires explicit authorization, spare capacity, rollback and abort criteria.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | Status |
|---|---|---|---|
| E01 | Scope/roles/abort | approved game-day plan | Pending |
| E02 | Hardware/driver inventory | GPU UUID/SKU/version | Pending |
| E03 | Xid/ECC alerts | rules and dashboards | Pending |
| E04 | Evidence collection | sanitized checklist | Pending |
| E05 | Recorded Xid triage | event classification | Pending |
| E06 | ECC classification | vendor guidance | Pending |
| E07 | Driver incompatibility | simulated diagnosis | Pending |
| E08 | Device plugin outage | root cause | Pending |
| E09 | Thermal/interconnect | metrics/topology | Pending |
| E10 | Synthetic page | route/ack timeline | Pending |
| E11 | Staging cordon/failover | protected capacity | Pending |
| E12 | Inference recovery | p99/TTFT/errors | Pending |
| E13 | Training restore | checkpoint hashes/step | Pending |
| E14 | CUDA/NCCL retest | baseline comparison | Pending |
| E15 | RCA/COAs | incident document | Pending |
| E16 | Independent review | score/signoff | Pending |

## 3. Read-only Kubernetes preflight

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable.nvidia\.com/gpu}{"\n"}{end}'
\`\`\`

Record missing access or absent GPU devices as BLOCKED. These commands do not validate hardware or recovery.

## 4. GPU node diagnostics (approved access only)

\`\`\`bash
nvidia-smi -L
nvidia-smi -q -d ECC,ERROR,PERFORMANCE,TEMPERATURE,POWER
journalctl -k --no-pager | grep -Ei 'NVRM|Xid|nvidia' | tail -100
\`\`\`

Sanitize GPU hostnames, IPs and workload metadata before attaching evidence. Record exact Xid code and vendor-specific interpretation; do not label all Xid errors hardware failures.

## 5. Synthetic incident timeline exercise

\`\`\`text
T+00:00  synthetic Xid alert received
T+00:02  on-call acknowledges
T+00:05  GPU UUID and workload impact identified
T+00:10  incident commander chooses safe containment
T+00:20  protected serving rerouted / training checkpoint located
T+00:30  staging workload restored
T+00:40  CUDA/NCCL and end-user SLO revalidated
\`\`\`

The timestamps are **illustrative tabletop objectives**, not measured results.

## 6. Fault scenarios and abort gates

| Exercise | Safe injection | Abort when |
|---|---|---|
| Xid alert | synthetic metric/log payload | route pages wrong owner |
| Driver mismatch | incompatible disposable test image | host driver at risk |
| GPU missing | simulated allocatable loss | protected capacity threatened |
| Inference GPU failure | synthetic Pod failure | p99 exceeds approved budget |
| Training rank failure | disposable job termination | checkpoint state at risk |
| Node evacuation | approved staging cordon | no compatible spare GPUs |

## 7. Evidence manifest

\`\`\`yaml
chapter: "097"
environment: "GPU_STAGING_REQUIRED"
publication: "PUBLISHED"
execution: "NOT_RUN"
execution_date_utc: null
cluster: null
incident_commander: null
gpu_platform_owner: null
security_reviewer: null
gpu_inventory: []
xid_ecc_triage: "NOT_RUN"
driver_fault_drill: "NOT_RUN"
operator_outage_drill: "NOT_RUN"
alert_delivery: "NOT_TESTED"
node_evacuation: "NOT_RUN"
inference_slo_recovery: "NOT_RUN"
training_checkpoint_restore: "NOT_RUN"
cuda_nccl_revalidation: "NOT_RUN"
rca_review: "PENDING"
independent_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
corrective_actions: []
\`\`\`

## 8. Acceptance

Score **20 + 20 + 20 + 20 + 10 + 10 = 100**. Pass ≥85 only after safe fault triage, protected SLO recovery, training checkpoint integrity, evidence, corrective actions and independent review. **Publishing a game-day plan is not execution.**

**Next:** Chapter 098 — Run Distributed Training Failure Recovery.
