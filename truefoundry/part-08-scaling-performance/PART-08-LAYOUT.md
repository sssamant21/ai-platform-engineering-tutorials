# Part 08 — Scaling & Performance: Master Layout

**Scope:** TrueFoundry inference scaling, Kubernetes HPA/KEDA, GPU node provisioning, capacity, workload performance, cost governance and operational recovery.

| Chapter | Topic | Canonical | Hands-on Lab |
|---|---|---|---|
| 8.1 | Scaling Architecture and GPU-Aware Capacity Planning | [Tutorial](01-scaling-architecture-capacity-planning/01-scaling-architecture-capacity-planning.md) | [Lab](labs/01-scaling-architecture-capacity-planning-check.md) |
| 8.2 | Horizontal Pod Autoscaling for Inference Workloads | [Tutorial](02-horizontal-pod-autoscaling-inference-workloads/02-horizontal-pod-autoscaling-inference-workloads.md) | [Lab](labs/02-horizontal-pod-autoscaling-inference-workloads-check.md) |
| 8.3 | KEDA Event-Driven Scaling and Custom Metrics | [Tutorial](03-keda-event-driven-scaling-custom-metrics/03-keda-event-driven-scaling-custom-metrics.md) | [Lab](labs/03-keda-event-driven-scaling-custom-metrics-check.md) |
| 8.4 | GPU Node Autoscaling and Cluster Capacity | [Tutorial](04-gpu-node-autoscaling-cluster-capacity/04-gpu-node-autoscaling-cluster-capacity.md) | [Lab](labs/04-gpu-node-autoscaling-cluster-capacity-check.md) |
| 8.5 | Inference Concurrency, Latency and Throughput Optimization | [Tutorial](05-inference-concurrency-latency-throughput-optimization/05-inference-concurrency-latency-throughput-optimization.md) | [Lab](labs/05-inference-concurrency-latency-throughput-optimization-check.md) |
| 8.6 | Scaling Observability, Cost and Troubleshooting | [Tutorial](06-scaling-observability-cost-troubleshooting/06-scaling-observability-cost-troubleshooting.md) | [Lab](labs/06-scaling-observability-cost-troubleshooting-check.md) |
| 8.7 | Production Scaling and Recovery Capstone | Planned | Planned |

**Publication:** 6/7 chapters and 6/7 labs published. Formal technical/vendor review, production/SRE review, revised-final approval and live lab execution remain separate pending milestones.

**Operating principles:** measure SLO-constrained throughput; separate engine batching, serving replicas and GPU nodes; plan N-1 headroom and cold starts; validate metrics; protect streaming requests; recover through authoritative TrueFoundry/GitOps.
