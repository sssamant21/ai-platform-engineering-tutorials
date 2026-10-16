# Chapter 12.2 — GPU Capacity Planning, Quotas and Production Scaling

**Part 12 — Production Operations | Canonical SRE tutorial**

## Objectives, prerequisites and guardrails

Design and operate predictable GPU capacity for TrueFoundry-hosted vLLM inference on Kubernetes. Quantify workload demand in tokens and requests, translate model memory and performance into feasible GPU placement, plan quota and autoscaler headroom, avoid overcommit and capacity deadlocks, and make scaling changes safely.

**[SAFE-READ]** inspect inventory and telemetry. **[LAB-MUTATE]** synthetic traffic on approved staging resources. **[CHANGE-CONTROL]** GPU node pools, quotas, cluster autoscalers, model replicas, routing and production cost budgets. Verify Kubernetes, TrueFoundry, NVIDIA driver/device plugin, vLLM engine, cloud GPU SKU and Karpenter/Cluster Autoscaler versions before using any provider-specific option. Do not record real prompts, completions, PHI, credentials or model weights.

## 1. Capacity as a multi-layer constraint

```text
Customer request distribution
  ├─ peak RPS and concurrent streams
  ├─ input tokens, output tokens, context length
  └─ SLO: TTFT, TPOT, error rate, completion
           │
   vLLM runtime and batching
  ├─ model weights / quantization / KV cache
  ├─ prefill vs decode / scheduler limits
  └─ GPU memory + throughput envelope
           │
   Kubernetes placement
  ├─ GPU request and MIG profile
  ├─ node CPU/RAM, taints, affinity, topology
  ├─ namespace quota and node-pool limits
  └─ autoscaler lead time and available cloud quota
           │
    available replicas + warm spare
           │
   ingress admission control / fallback
```

A Pod may be schedulable yet fail CUDA initialization, model loading or the TTFT SLO. Node allocatable GPUs, memory fit and verified inference throughput are **different** capacity constraints.

## 2. Demand forecast and workload profiles

Build a demand matrix by service/model, region, environment, time-of-day and traffic tier. Collect peak/steady requests per second, input/output token distributions (p50/p95/p99), context length, streaming duration, concurrency, retries, burstiness, growth rate and seasonality. Use approved synthetic workloads to reproduce representative token mixes.

Apply Little's Law as an approximation for stable systems: `average_concurrency ≈ arrival_rate × average_duration`. Do not substitute p99 latency for the average in this formula. Plan queueing separately; streaming request lifetime may be much longer than TTFT.

## 3. GPU memory and model fit

Account for model weights, CUDA runtime/context, activation/workspace, attention/KV cache, graph capture, fragmentation and safety reserve. Quantization, tensor/pipeline parallelism and vLLM engine flags change memory requirements. GPU utilization is not free-memory headroom.

```text
usable_gpu_memory = physical_memory - driver/runtime_reserved - safety_margin
required_memory = weights + runtime_workspace + kv_cache + fragmentation
fit only if required_memory <= usable_gpu_memory under tested concurrency
```

This is a planning model, not a guarantee: memory spikes, context length, graph capture and CUDA allocator behavior can invalidate estimates. Test real model/runtime versions in staging. For MIG, treat profiles as distinct resource types and verify scheduling/resource advertisement; do not count MIG slices as full GPUs.

## 4. GPU throughput and replica sizing

Measure sustained accepted output tokens/sec and requests/sec **while meeting TTFT/TPOT and error targets**. For a steady target of 1,200 output tokens/sec and a validated capacity of 300 output tokens/sec per replica, the nominal requirement is 4 replicas. With a 25% operating headroom target, a conservative planning estimate is `ceil(1200 / (300 × 0.75)) = 6` replicas. This is illustrative and excludes failover, burst capacity and cloud SKU fragmentation.

For high availability, verify that losing the largest failure domain leaves enough *ready* replicas to serve priority traffic. A reserved node without loaded model weights may not meet recovery objectives.

## 5. Kubernetes quota and scheduling inventory [SAFE-READ]

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl describe nodes
kubectl get pods -A -o wide
kubectl get resourcequota,limitrange -A
kubectl get priorityclass
kubectl get pdb -A
kubectl get hpa -A
kubectl get events -A --sort-by=.lastTimestamp | tail -50
```

For a specific approved namespace:
```bash
kubectl -n MODEL_NAMESPACE describe pod MODEL_POD
kubectl -n MODEL_NAMESPACE get pods -o custom-columns='NAME:.metadata.name,NODE:.spec.nodeName,PHASE:.status.phase'
```

Check allocatable `nvidia.com/gpu` or provider/MIG extended resources, GPU requests/limits, CPU/memory fit, taints/tolerations, affinity, topology, maxPods, PVC zone binding and namespace quotas. GPU extended resources are discrete and generally cannot be fractionally requested without an approved sharing mechanism.

## 6. ResourceQuota, LimitRange and tenant isolation

Namespace ResourceQuota can constrain GPU requests/limits, CPU/memory and object counts, subject to resource availability and Kubernetes version. For extended GPU resources, validate exact quota keys and admission behavior in staging; do not copy untested manifests into production.

An illustrative quota, only for clusters that support the named resource:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: inference-gpu-budget
  namespace: inference-staging
spec:
  hard:
    requests.nvidia.com/gpu: "4"
    requests.cpu: "16"
    requests.memory: 64Gi
```

ResourceQuota is not a cloud GPU inventory guarantee. Cloud account/region quota, node-pool max, provisioning permissions and GPU SKU availability may be tighter. Document fair-share policy, reserved priority capacity and protection from a noisy tenant.

## 7. Autoscaling architecture

**HPA:** scales replicas using CPU/custom/external metrics; GPU utilization alone may not correlate with latency or queue pressure. Confirm the custom metrics adapter and HPA version/behavior.

**KEDA:** event-driven scaling based on supported external metrics; verify scaler type, auth, polling, cooldown and metric adapter.

**Cluster Autoscaler / Karpenter:** adds GPU nodes to satisfy pending Pods, subject to instance availability, IAM, subnet IPs, taints, daemon overhead and quotas. Node provisioning + image pull + model load can dominate total scale-up latency.

Avoid scale-to-zero for latency-critical large models unless cold-start objectives explicitly permit it. Use minimum warm replicas, bounded max replicas, scale-down stabilization, disruption budgets and conservative cooldowns. Test whether node and workload autoscalers oscillate together.

## 8. Capacity model and scaling policy

Document:
- **Baseline:** demand, verified per-replica throughput, TTFT/TPOT SLO and error rate.
- **Headroom:** peak reserve, N+1 or zone-failure reserve, GPU memory safety margin.
- **Provisioning:** cloud quota, instance supply, launch/image/model warmup times.
- **Admission:** concurrency caps, backpressure, rate limits, request priorities and graceful rejection.
- **Cost:** GPU-hour rate, replicas × hours, warm capacity, data transfer and storage.
- **Fallback:** model/region failover only if functional/security and SLO constraints allow.

Promote scaling changes with versioned configuration, staged load test, clear rollback, budget approval and SLO observation.

## 9. Capacity dashboards and alerting

Required panels: eligible RPS, concurrent streams, input/output tokens/sec, queue depth, TTFT/TPOT p95, error ratio, running/waiting requests, KV cache pressure, GPU utilization/memory, GPU health, allocatable vs requested GPUs, pending Pods, autoscaler decisions, node provision latency, cloud quota, warm replicas and estimated GPU cost. Metric names depend on installed vLLM/DCGM/Prometheus integrations. Distinguish absent telemetry from zero demand.

## 10. Twelve production runbooks

1. **GPU Pods Pending:** inspect events, allocatable GPU, affinity, taints, quotas and cloud supply.
2. **Namespace GPU quota exhausted:** validate requested resources, owners, approval and fair-share policy.
3. **Cloud GPU quota exhausted:** inspect region/SKU quotas, capacity reservations and approved fallback.
4. **Karpenter/CA cannot provision:** inspect node class, IAM, subnet IPs, instance offerings and logs.
5. **Node Ready but GPU unavailable:** inspect NVIDIA device plugin, runtime and driver health.
6. **Model cannot fit GPU memory:** inspect weights, quantization, context length, KV cache and CUDA OOM.
7. **GPU memory fragmented:** inspect runtime allocator, preemption and model restart impact.
8. **TTFT worsens during burst:** correlate queue, prefill tokens, concurrency and warm replica count.
9. **Throughput plateaus:** find latency-throughput knee and CPU/network/decoder constraints.
10. **Scaling oscillates:** inspect HPA/KEDA stabilization, cooldown, metric freshness and node consolidation.
11. **Zone failure leaves insufficient replicas:** verify placement, PDBs, reserved capacity and recovery time.
12. **GPU costs exceed budget:** compare utilization, idle warm capacity, demand forecast and rightsizing policy.

## 11. Acceptance and change control

Require approved forecast, reproducible per-replica benchmark, quota inventory, GPU memory fit, failure-domain capacity, scale-up timing, admission-control policy, alert ownership, cost estimate and rollback. Do not approve production based solely on nominal GPU count.

Complete the [hands-on lab](../labs/02-gpu-capacity-planning-quotas-production-scaling-check.md): **32 failure scenarios and 12 acceptance gates**. Formal technical review and live execution pending.

**Next:** Chapter 12.3 — High Availability, Resilience and Failure Domains.
