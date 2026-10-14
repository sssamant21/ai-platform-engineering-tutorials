# Lab 4.6 — GPU Validation, Monitoring and Troubleshooting Check

**Canonical:** [Chapter 4.6](../06-gpu-validation-monitoring-troubleshooting/06-gpu-validation-monitoring-troubleshooting.md)  
**Mode:** Production **[SAFE-READ]**. No node, exporter, driver or alert changes.

## Objectives

Discover GPU monitoring components, validate DCGM metric freshness/schema, correlate GPU and inference health, distinguish missing telemetry from failed GPU operation, and write an actionable incident report.

## 0. Safety and preflight

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get namespaces
```

Record cluster, namespace, TrueFoundry deployment and timestamp. Use approved RBAC. Do not install exporters, edit scrape configuration or create privileged debug Pods for this lab.

## 1. Inventory GPU resources [SAFE-READ]

```bash
kubectl get nodes -L node-pool,gpu,gpu_type
kubectl describe node <gpu-node>
kubectl get node <gpu-node> -o jsonpath='{.status.capacity.nvidia\.com/gpu}{" capacity\n"}{.status.allocatable.nvidia\.com/gpu}{" allocatable\n"}'
```

Record node Ready, advertised GPU capacity, allocatable, allocated requests and pool labels/taints. **Do not** call allocatable “free GPUs.”

## 2. Discover monitoring stack [SAFE-READ]

```bash
kubectl get daemonsets -A
kubectl get pods -A -o wide
kubectl get services -A
kubectl get servicemonitors -A
```

The last command works only if ServiceMonitor CRDs exist. Identify the actual DCGM exporter namespace, Pod and service. If no exporter exists, record **Not installed/Not accessible** rather than fabricating telemetry.

```bash
kubectl -n <gpu-namespace> describe pod <dcgm-exporter-pod>
kubectl -n <gpu-namespace> logs <dcgm-exporter-pod> --tail=100
```

## 3. Validate metrics endpoint [SAFE-READ]

If approved, port-forward to the existing exporter Pod:

```bash
kubectl -n <gpu-namespace> port-forward pod/<dcgm-exporter-pod> 9400:9400
```

In another terminal:

```bash
curl -fsS http://127.0.0.1:9400/metrics | grep -E '^(# (HELP|TYPE) DCGM_FI_DEV_|DCGM_FI_DEV_(GPU_UTIL|FB_USED|GPU_TEMP|XID_ERRORS))' | head -80
```

Use the actual configured port if not 9400. Confirm HELP/TYPE, units, labels, GPU identity and metric freshness. A successful endpoint response does not prove Prometheus is scraping it.

## 4. Inspect application impact [SAFE-READ]

```bash
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> describe pod <gpu-pod>
kubectl -n <namespace> logs <gpu-pod> --tail=150
kubectl -n <namespace> get events --sort-by=.lastTimestamp
```

Capture Pod state, CUDA initialization, vLLM readiness, recent failures, and available inference latency/throughput/error metrics. Avoid logging credentials or patient data.

## 5. Build dashboard requirements

Complete this table with **real** series names/labels from your environment:

| Panel | Metric or log source | Labels | Unit / metric type | Owner |
|---|---|---|---|---|
| GPU utilization | | | | |
| VRAM used/free | | | | |
| GPU temperature | | | | |
| Xid/ECC | | | | |
| GPU resource allocation | | | | |
| Pending GPU workloads | | | | |
| CUDA/vLLM failures | | | | |
| Inference P99/error rate | | | | |

Example PromQL only if metrics exist:

```promql
DCGM_FI_DEV_GPU_UTIL
DCGM_FI_DEV_FB_USED
DCGM_FI_DEV_GPU_TEMP
```

Verify exporter labels before adding `by (node)` aggregations. Do not infer physical GPU counts from unfiltered time-series counts.

## 6. Non-destructive incident scenarios

For each scenario, identify first evidence, failure boundary, current owner, user impact and safe remediation:

1. **Exporter down, inference healthy:** distinguish monitoring outage from serving outage.
2. **GPU node Ready, no advertised GPU:** inspect plugin/driver registration.
3. **GPU Pod Pending:** inspect allocations, taints, affinity and autoscaler evidence.
4. **GPU visible, vLLM fails:** investigate PyTorch/CUDA and vLLM initialization.
5. **High GPU utilization, P99 regression:** correlate VRAM, batching, queueing and model saturation.
6. **Xid error burst:** record code, frequency, device identity, workload symptoms and NVIDIA guidance; do not reset hardware as an experiment.
7. **Low utilization, high GPU cost:** compare reserved/allocated GPU-hours, demand and inference SLOs.

## 7. Alert acceptance worksheet

```text
Alert name:
Signal and source:
Metric type / unit / labels verified:
Trigger duration:
Hardware-specific threshold evidence:
User impact / SLO relationship:
Noise / maintenance handling:
Owner and escalation:
Runbook link:
Recovery condition:
```

Do not deploy alert rules in this lab. Thresholds must be validated against actual hardware, baseline and metric semantics.

## 8. Incident evidence report

```text
Timestamp / context / namespace:
TrueFoundry workspace / deployment:
GPU node / GPU identity:
Driver / device plugin / exporter status:
GPU capacity / allocatable / assigned requests:
GPU utilization / VRAM / temperature:
Xid / ECC data:
Prometheus scrape health / freshness:
Pod and CUDA state:
vLLM model readiness:
Inference latency / throughput / error rate:
Recent changes:
Lowest Proven Healthy Layer:
First Failed Transition:
Current actionable owner:
Approved remediation / rollback:
Post-recovery validation:
```

**Pass:** produce a source-backed GPU health picture, correctly separate monitoring failures from GPU/runtime/model failures, and propose a safe recovery path without production mutation.

**Next:** Chapter 4.7 — Production GPU Infrastructure Recovery Lab.
