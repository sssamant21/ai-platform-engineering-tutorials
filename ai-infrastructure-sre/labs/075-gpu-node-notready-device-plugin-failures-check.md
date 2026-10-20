# Lab 075 — GPU Node NotReady and Device Plugin Failures Acceptance Gate

**Chapter:** [075 — GPU Node NotReady and Device Plugin Failures](../part-10-incident-response-and-troubleshooting/075-gpu-node-notready-device-plugin-failures.md)

**Status:** NOT EXECUTED. This lab describes safe validation; no live cluster or GPU fault has been tested.

## 1. Preconditions and safety

Read-only Kubernetes and host diagnostics first. Any component restart, cordon, drain, driver change or reboot needs an approved staging change and workload owner. Never disrupt production training or inference to complete a tutorial. Keep incident evidence access-controlled and redact secrets, hostnames and tenant metadata.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | GPU/driver/operator inventory | version and node mapping | Pending |
| E02 | Node readiness | conditions, lease, taints, pressure | Pending |
| E03 | GPU resource reconciliation | physical/MIG vs allocatable | Pending |
| E04 | Device plugin | DaemonSet, pod logs, socket status | Pending |
| E05 | GPU Operator | ClusterPolicy and components | Pending |
| E06 | Kubelet diagnostics | journal and event timestamps | Pending |
| E07 | NVML/Xid evidence | redacted driver logs | Pending |
| E08 | Synthetic Ready/no GPU | fault tree and safe response | Pending |
| E09 | Synthetic NotReady/GPU healthy | non-GPU root-cause tree | Pending |
| E10 | Pending GPU workloads | taints/quotas/topology | Pending |
| E11 | Staging plugin restart | approved change and rollback | Pending |
| E12 | Driver mismatch tabletop | image/version matrix | Pending |
| E13 | Xid 79 checkpoint tabletop | job recovery decisions | Pending |
| E14 | Alert routing | test receiver evidence | Pending |
| E15 | Recovery go/no-go | rollback and health criteria | Pending |
| E16 | Acceptance | reviewer and score | Pending |

## 3. Read-only commands

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe node <gpu-node>
kubectl get node <gpu-node> -o yaml
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get pods -A -o wide --field-selector spec.nodeName=<gpu-node>
kubectl get pods -A --field-selector=status.phase=Pending
kubectl get pods -A | grep -Ei 'nvidia|gpu-operator|device-plugin|dcgm'
kubectl get daemonsets -A | grep -Ei 'nvidia|gpu'
kubectl get resourcequota -A
\`\`\`

Host access **only when approved**:

\`\`\`bash
uname -r
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version --format=csv
systemctl status kubelet --no-pager
journalctl -u kubelet --since '-2 hours' --no-pager
journalctl -k --since '-2 hours' | grep -Ei 'NVRM|Xid|nvidia|AER'
\`\`\`

## 4. Synthetic troubleshooting cases

**Case A:** Node Ready, physical GPU visible in \`nvidia-smi\`, but \`nvidia.com/gpu\` missing. Inspect plugin logs, device registration, GPU Operator configuration and MIG strategy. Do not reset GPU by default.

**Case B:** Node NotReady, \`nvidia-smi\` healthy. Inspect kubelet lease, network, disk/memory pressure and cloud node health. Do not assume a GPU issue.

**Case C:** Node Ready, allocatable GPUs present, pod Pending. Check taints, node affinity, quota, GPU SKU and topology. Do not assume more generic GPUs solve the issue.

**Case D:** Xid 79 during distributed training. Preserve driver and checkpoint evidence; coordinate vendor and workload owners; no unapproved drain or reset.

## 5. Staging-only component drill

Use a disposable GPU staging node with no production jobs. Document existing plugin DaemonSet, node allocatable, GPU inventory and baseline CUDA smoke test. Under approved change, restart **only** the staging device-plugin pod, observe re-registration and alert behavior, and confirm GPU allocatable returns to baseline. Abort if unexpected workloads or unrelated nodes are affected. A tabletop is acceptable as training but does **not** count as live recovery validation.

## 6. Evidence record

\`\`\`yaml
chapter: "075"
environment: "STAGING_OR_OFFLINE"
execution_date_utc: null
reviewer: null
cluster_version: null
node_id_redacted: null
gpu_model: null
driver_version: null
gpu_operator_version: null
device_plugin_version: null
node_ready_before: null
node_ready_after: null
physical_gpu_count: null
mig_profile_inventory: {}
gpu_allocatable_before: null
gpu_allocatable_after: null
plugin_logs_reviewed: "PENDING"
kubelet_evidence_reviewed: "PENDING"
staging_restart: "NOT_RUN"
synthetic_cases: "NOT_RUN"
alert_delivery: "NOT_RUN"
security_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
open_risks: []
\`\`\`

## 7. Acceptance gate

Chapter scoring: **20 + 20 + 15 + 15 + 20 + 10 = 100**. Pass at **≥85** with no critical safety/security failure and documented rollback. Record publication, offline execution, staging validation and independent review as separate milestones.

**Next:** Chapter 076 — CUDA Initialization and Driver Errors.
