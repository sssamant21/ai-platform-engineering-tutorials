# Lab 080 — GPU Failure Triage and Escalation Acceptance Gate

**Chapter:** [080 — GPU Failure Triage and Escalation](../part-10-incident-response-and-troubleshooting/080-gpu-failure-triage-escalation.md)

**Status:** NOT EXECUTED. Offline synthetic exercises, live staging checks, independent technical/security review and acceptance remain pending.

## 1. Safety and prerequisites

Read-only collection only in production with approved access. **Never intentionally trigger Xid, ECC, PCIe, NVLink or thermal failures** on live GPUs. No unapproved reset, reboot, firmware update, driver reinstall, drain, forced pod deletion or hardware diagnostic stress. Protect customer data, training checkpoints and multi-tenant identifiers.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | State |
|---|---|---|---|
| E01 | GPU inventory | UUID, SKU, node, driver | Pending |
| E02 | Job-to-GPU mapping | pod UID, node, GPU UUID | Pending |
| E03 | Kernel event timeline | UTC NVRM/Xid/AER logs | Pending |
| E04 | Synthetic Xid 79 | fault hypothesis and escalation | Pending |
| E05 | Synthetic ECC trend | correctable/uncorrectable classification | Pending |
| E06 | Synthetic row remap | vendor-guided decision | Pending |
| E07 | Synthetic PCIe fault | host/riser/device hypothesis | Pending |
| E08 | Synthetic NVLink fault | rank and topology impact | Pending |
| E09 | Synthetic thermal throttle | power/cooling analysis | Pending |
| E10 | Healthy peer comparison | differences and confidence | Pending |
| E11 | DCGM alert review | metric availability and semantics | Pending |
| E12 | Severity/impact matrix | customer and job impact | Pending |
| E13 | Vendor case bundle | redacted evidence package | Pending |
| E14 | Quarantine tabletop | checkpoint-safe approval plan | Pending |
| E15 | Return-to-service gate | staged CUDA/NCCL/allocatable check | Pending |
| E16 | Acceptance | RCA/COA, reviewer, score | Pending |

## 3. Read-only diagnostics

\`\`\`bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe node <gpu-node>
kubectl get pods -A -o wide --field-selector spec.nodeName=<gpu-node>
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl get pods -A | grep -Ei 'nvidia|gpu-operator|device-plugin|dcgm'
\`\`\`

Approved GPU host:

\`\`\`bash
nvidia-smi -L
nvidia-smi --query-gpu=uuid,name,driver_version,temperature.gpu,power.draw --format=csv
nvidia-smi -q -d ECC,PCI,PERFORMANCE,POWER,TEMPERATURE
journalctl -k --since '-4 hours' | grep -Ei 'NVRM|Xid|AER|PCIe|nvidia'
\`\`\`

Record unsupported GPU/driver fields as \`NOT_AVAILABLE\`, not PASS.

## 4. Synthetic failure fixtures

**A — Xid 79:** kernel logs report Xid 79, GPU disappears from \`nvidia-smi\`, and one distributed training rank fails. Preserve evidence, protect checkpoint and escalate hardware/PCIe without reset.

**B — ECC:** volatile correctable ECC rises but no application error; distinguish warning trend from an uncorrectable event with workload impact. Follow model-specific vendor guidance.

**C — PCIe AER:** link errors appear after host maintenance. Compare host PCIe evidence and GPU UUID timeline before assuming GPU silicon failure.

**D — NVLink:** interconnect error counters rise and NCCL collective latency spikes. Identify GPU/NVLink topology and fabric owner.

**E — Thermal:** GPU clocks drop under high temperature while Xid absent. Investigate cooling/power limits rather than automatic RMA.

## 5. Vendor escalation template

\`\`\`yaml
case_type: "GPU_HARDWARE_TRIAGE"
severity: "UNASSESSED"
customer_impact: "REDACTED"
incident_start_utc: null
gpu_model: null
gpu_uuid_redacted: null
node_id_redacted: null
driver_version: null
kernel_version: null
gpu_operator_version: null
xid_codes: []
ecc_summary: "NOT_COLLECTED"
pcie_nvlink_summary: "NOT_COLLECTED"
thermal_power_summary: "NOT_COLLECTED"
last_verified_checkpoint_utc: null
mitigation: "NONE"
vendor_case_id: null
case_owner: null
next_update_utc: null
evidence_links: []
\`\`\`

## 6. Acceptance evidence

\`\`\`yaml
chapter: "080"
environment: "STAGING_OR_OFFLINE"
execution_date_utc: null
reviewer: null
gpu_inventory: "NOT_RUN"
xid_ecc_analysis: "NOT_RUN"
pcie_nvlink_analysis: "NOT_RUN"
vendor_bundle_review: "PENDING"
quarantine_tabletop: "NOT_RUN"
cuda_smoke: "NOT_RUN"
nccl_smoke: "NOT_RUN"
return_to_service: "NOT_EVALUATED"
security_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
open_risks: []
\`\`\`

## 7. Acceptance

Chapter 080 gates: **20 + 25 + 15 + 15 + 15 + 10 = 100**. Pass at ≥85, with no critical data loss, unauthorized GPU reset, production disruption or privacy exposure. Do not mark live hardware fault injection complete; this lab uses synthetic fixtures for unsafe scenarios.

**Next:** Chapter 081 — Recovery Runbooks and RCA COA.
