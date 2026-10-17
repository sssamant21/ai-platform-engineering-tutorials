# Chapter 059 — Hands-On Lab: GPU DCGM, Node and Kubernetes Observability

**Part 08 | Chapter 059 | Written lab published; execution pending**

## Goal

Inspect DCGM exporter, GPU device plugin, node/Kubernetes telemetry, correlate hardware with inference SLOs, validate MIG/topology labels, alerting and GPU cost accounting.

## Safety

**[SAFE-READ]** authorized Kubernetes/Prometheus/GPU inventory. **[LAB-MUTATE]** approved isolated staging. **[CHANGE-CONTROL]** production GPU Operator, DCGM exporter, device plugin, MIG, driver, scrape targets and alerts. Never reset/reboot production GPUs or expose raw prompts, credentials or PHI.

## Lab 0 — Preflight

Record UTC, cluster/context, cloud/region/AZ, GPU SKU, driver/DCGM/device-plugin versions, MIG mode, Prometheus/DCGM endpoints, model digest, approval, budget and rollback.

## Lab 1 — GPU and node inventory [SAFE-READ]

```bash
date -u
kubectl config current-context
kubectl get nodes -o wide
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -A -o wide
kubectl get daemonsets -A
kubectl get events -A --sort-by=.lastTimestamp | tail -100
kubectl describe node APPROVED_GPU_NODE
```

Record GPU Capacity/Allocatable, Pod requests, node labels/taints, driver/plugin status and AZ.

## Lab 2 — DCGM metric discovery [SAFE-READ]

```bash
# Approved internal metrics endpoint only; port illustrative
curl -fsS --max-time 10 http://127.0.0.1:9400/metrics | grep -E '^(# HELP|# TYPE|DCGM_)' | head -100
```

Document actual metric names, units, types, labels, supported GPU/MIG device and scrape freshness.

## Lab 3 — Hardware-to-inference correlation [DESIGN]

| Signal | Evidence source | Interpretation |
|---|---|---|
| TTFT/TPOT | inference gateway/server | user latency |
| queue/KV | model runtime | scheduler pressure |
| GPU SM | DCGM | compute activity |
| VRAM | DCGM | model/KV headroom |
| ECC/Xid | GPU diagnostics | hardware health |
| node CPU/network | node exporter | preprocessing/download |
| Pod readiness | kube-state/Kubernetes | serving capacity |

Do not assume high GPU utilization alone is a failure.

## Lab 4 — PromQL design [VERIFY LABELS]

```promql
avg by (instance) (DCGM_FI_DEV_GPU_UTIL)
```

```promql
sum by (instance) (DCGM_FI_DEV_FB_USED)
```

Validate exporter units, duplicate targets, MIG granularity and actual labels.

## Lab 5 — Topology and FinOps [CALCULATION]

```text
GPU-hours = sum(device allocated duration in hours)
Cost per valid completion = allocated GPU cost / terminal-valid completions
```

Separate physical GPU, MIG slice and time-sliced resource accounting. Include idle warm pool, zone reserve and telemetry cost.

## Lab 6 — Eight game days

01 DCGM absent; 02 plugin missing; 03 VRAM OOM; 04 MIG drift; 05 ECC/Xid; 06 CPU bottleneck; 07 duplicate scrape; 08 idle GPU spend.

## Twenty-four failure checklist

F01 exporter; F02 target; F03 counter; F04 compatibility; F05 allocatable; F06 plugin; F07 VRAM OOM; F08 utilization; F09 bandwidth; F10 ECC/Xid; F11 thermal; F12 power; F13 MIG; F14 sharing; F15 Pod/GPU join; F16 duplicate; F17 CPU; F18 network; F19 restarts; F20 zone; F21 cardinality; F22 privacy; F23 alerts; F24 FinOps.

## Twenty acceptance gates

| Gate | Evidence | Status |
|---|---|---|
| G01 | Environment/versions | Pending |
| G02 | DCGM exporter | Pending |
| G03 | Scrape freshness | Pending |
| G04 | GPU inventory | Pending |
| G05 | Allocatable/requests | Pending |
| G06 | GPU utilization | Pending |
| G07 | VRAM | Pending |
| G08 | ECC/Xid | Pending |
| G09 | Thermal/power | Pending |
| G10 | MIG/sharing | Pending |
| G11 | Pod-to-GPU mapping | Pending |
| G12 | Node CPU/memory | Pending |
| G13 | Network/storage | Pending |
| G14 | Pod readiness/restarts | Pending |
| G15 | TTFT/TPOT correlation | Pending |
| G16 | Labels/privacy | Pending |
| G17 | Alerting | Pending |
| G18 | Rollback | Pending |
| G19 | GPU FinOps | Pending |
| G20 | Independent review | Pending |

## Evidence worksheet

```text
UTC/context/approval:
Cloud/region/AZ/GPU SKU:
Driver/DCGM/plugin/MIG versions:
GPU inventory and allocatable:
DCGM HELP/TYPE/units/labels:
SM/VRAM/ECC/Xid/thermal/power:
Node CPU/memory/network/storage:
Pod readiness/restarts:
Model TTFT/TPOT/queue/KV:
Pod-to-GPU correlation:
PromQL/alerts/cardinality/privacy:
GPU-hours and cost:
Eight game-day findings:
Rollback/reviewer/gates:
```

## Cleanup

Read-only/tabletop work creates no resources. Approved staging changes must restore original exporter/plugin/scrape/alert configuration, remove only owned test workloads, validate GPU availability and reconcile cost. No acceptance gate passes without evidence and independent review.

**Next: Chapter 060 — OpenTelemetry Traces Across Inference Paths.**
