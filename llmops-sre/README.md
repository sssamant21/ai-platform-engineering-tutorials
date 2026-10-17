# LLMOps & AI Inference Site Reliability Engineering

**Repository:** [AI Platform Engineering Tutorials](https://github.com/sssamant21/ai-platform-engineering-tutorials)  
**Track:** `llmops-sre/`  
**Scope:** 100 planned chapters, 12 parts. **Current status:** 71/100 chapters and companion labs published; Part 01 written curriculum complete (5/5); technical review and live execution pending.

## Mission

Build practical SRE skills for production LLM inference: vLLM, NVIDIA Triton, TensorRT-LLM, Ray Serve, KServe, Kubernetes GPU operations, observability, SLOs, performance, security, FinOps, incident response and enterprise acceptance.

This module complements [TrueFoundry](../truefoundry/) and [Databricks](../databricks/), rather than repeating platform setup.

## Start here

- [Master TOC](MASTER-TOC.md)
- [Status Tracker](STATUS-TRACKER.md)

## Chapter publishing standard

Each chapter should contain learning objectives, architecture and production tradeoffs, prerequisites, reproducible commands, safe staging lab, expected outputs, troubleshooting decision trees, failure scenarios, operational runbooks, security/FinOps controls, cleanup, acceptance gates and references to applicable upstream documentation. Do not invent CLI/API syntax or present theoretical tests as executed.

**[SAFE-READ]** approved read-only checks. **[LAB-MUTATE]** isolated staging with approval and cleanup. **[CHANGE-CONTROL]** production changes. No credentials, PHI, real prompts or sensitive model artifacts in examples.

## Parts

| Part | Area | Chapters |
|---|---|---|
| 01 | AI/LLM SRE Fundamentals | 001–005 |
| 02 | LLM Architecture and Inference Internals | 006–010 |
| 03 | GPU Architecture, CUDA and VRAM | 011–015 |
| 04 | vLLM Administration and Operations | 016–025 |
| 05 | NVIDIA Triton and TensorRT-LLM | 026–035 |
| 06 | Ray Serve and KServe | 036–045 |
| 07 | Kubernetes GPU Scheduling and Autoscaling | 046–055 |
| 08 | LLM Observability, Metrics and Tracing | 056–065 |
| 09 | Performance Engineering and Capacity | 066–075 |
| 10 | AI Security, Governance and FinOps | 076–085 |
| 11 | Incident Response, HA and Disaster Recovery | 086–095 |
| 12 | Final Production AI SRE Capstone | 096–100 |

## Workflow

1. Write one complete chapter and companion lab.
2. Publish both to GitHub and verify links.
3. Update this index, master TOC and status tracker.
4. Keep technical review, security review, actual lab execution and acceptance tracked separately.

**Published:** [Chapter 001 — Introduction to LLMOps and AI Inference SRE](part-01-fundamentals/001-introduction-to-llmops-ai-inference-sre.md) — [Lab](labs/001-introduction-to-llmops-ai-inference-sre-check.md).

**Published:** [Chapter 002 — Production LLM Service Architecture and SRE Responsibilities](part-01-fundamentals/002-production-llm-service-architecture-sre-responsibilities.md) — [Lab](labs/002-production-llm-service-architecture-sre-responsibilities-check.md).

**Published:** [Chapter 003 — SLIs, SLOs, Error Budgets and Inference User Journeys](part-01-fundamentals/003-slis-slos-error-budgets-inference-user-journeys.md) — [Lab](labs/003-slis-slos-error-budgets-inference-user-journeys-check.md).

**Published:** [Chapter 004 — Hands-On Lab Environment and Safety Controls](part-01-fundamentals/004-hands-on-lab-environment-safety-controls.md) — [Lab](labs/004-hands-on-lab-environment-safety-controls-check.md).

**Published:** [Chapter 005 — Fundamentals Acceptance Lab and Incident Walkthrough](part-01-fundamentals/005-fundamentals-acceptance-lab-incident-walkthrough.md) — [Capstone Lab](labs/005-fundamentals-acceptance-lab-incident-walkthrough-check.md).

**Part 01:** 5/5 tutorials and 5/5 labs published. Technical/security review and actual lab execution are pending.

**Published:** [Chapter 006 — Transformer Inference Request Lifecycle](part-02-inference-internals/006-transformer-inference-request-lifecycle.md) — [Lab](labs/006-transformer-inference-request-lifecycle-check.md).

**Part 02:** 1/5 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 007 — Tokenization, Context Windows and Generation Parameters](part-02-inference-internals/007-tokenization-context-windows-generation-parameters.md) — [Lab](labs/007-tokenization-context-windows-generation-parameters-check.md).

**Part 02:** 2/5 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 008 — Prefill, Decode, KV Cache and Continuous Batching](part-02-inference-internals/008-prefill-decode-kv-cache-continuous-batching.md) — [Lab](labs/008-prefill-decode-kv-cache-continuous-batching-check.md).

**Part 02:** 3/5 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 009 — Streaming, Backpressure and API Gateway Behavior](part-02-inference-internals/009-streaming-backpressure-api-gateway-behavior.md) — [Lab](labs/009-streaming-backpressure-api-gateway-behavior-check.md).

**Part 02:** 4/5 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 010 — Inference Internals Performance Lab and Reliability Acceptance Project](part-02-inference-internals/010-inference-internals-performance-reliability-acceptance.md) — [Capstone Lab](labs/010-inference-internals-performance-reliability-acceptance-check.md).

**Part 02:** 5/5 chapters and companion labs published. Reviews, live execution and acceptance remain pending.

**Published:** [Chapter 011 — GPU Architecture, SMs, HBM and Memory Bandwidth](part-03-gpu-cuda-vram/011-gpu-architecture-sms-hbm-memory-bandwidth.md) — [Lab](labs/011-gpu-architecture-sms-hbm-memory-bandwidth-check.md).

**Part 03:** 1/5 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 012 — CUDA Runtime, Drivers, Containers and Compatibility](part-03-gpu-cuda-vram/012-cuda-runtime-drivers-containers-compatibility.md) — [Lab](labs/012-cuda-runtime-drivers-containers-compatibility-check.md).

**Part 03:** 2/5 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 013 — GPU VRAM Sizing, Precision and Quantization](part-03-gpu-cuda-vram/013-gpu-vram-sizing-precision-quantization.md) — [Lab](labs/013-gpu-vram-sizing-precision-quantization-check.md).

**Part 03:** 3/5 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 014 — Multi-GPU Parallelism and Topology](part-03-gpu-cuda-vram/014-multi-gpu-parallelism-topology.md) — [Lab](labs/014-multi-gpu-parallelism-topology-check.md).

**Part 03:** 4/5 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 015 — GPU Diagnostics, DCGM and Failure Lab](part-03-gpu-cuda-vram/015-gpu-diagnostics-dcgm-failure-lab.md) — [Capstone Lab](labs/015-gpu-diagnostics-dcgm-failure-lab-check.md).

**Part 03:** 5/5 chapters and companion labs published. Technical/security review, live execution and operational acceptance remain pending.

**Published:** [Chapter 016 — vLLM Architecture and Production Deployment](part-04-vllm/016-vllm-architecture-production-deployment.md) — [Lab](labs/016-vllm-architecture-production-deployment-check.md).

**Part 04:** 1/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 017 — vLLM Model Loading and Runtime Configuration](part-04-vllm/017-vllm-model-loading-runtime-configuration.md) — [Lab](labs/017-vllm-model-loading-runtime-configuration-check.md).

**Part 04:** 2/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 018 — Continuous Batching, PagedAttention and KV Cache](part-04-vllm/018-continuous-batching-pagedattention-kv-cache.md) — [Lab](labs/018-continuous-batching-pagedattention-kv-cache-check.md).

**Part 04:** 3/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 019 — vLLM Streaming and OpenAI-Compatible APIs](part-04-vllm/019-vllm-streaming-openai-compatible-apis.md) — [Lab](labs/019-vllm-streaming-openai-compatible-apis-check.md).

**Part 04:** 4/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 020 — vLLM GPU Memory and Context Tuning](part-04-vllm/020-vllm-gpu-memory-context-tuning.md) — [Lab](labs/020-vllm-gpu-memory-context-tuning-check.md).

**Part 04:** 5/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 021 — vLLM Tensor/Pipeline Parallelism and Distributed Serving](part-04-vllm/021-vllm-tensor-pipeline-parallelism-distributed-serving.md) — [Lab](labs/021-vllm-tensor-pipeline-parallelism-distributed-serving-check.md).

**Part 04:** 6/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 022 — vLLM Security, Authentication and Network Isolation](part-04-vllm/022-vllm-security-authentication-network-isolation.md) — [Lab](labs/022-vllm-security-authentication-network-isolation-check.md).

**Part 04:** 7/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 023 — vLLM Metrics, Logging and Distributed Tracing](part-04-vllm/023-vllm-metrics-logging-distributed-tracing.md) — [Lab](labs/023-vllm-metrics-logging-distributed-tracing-check.md).

**Part 04:** 8/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 024 — vLLM Slowness, CUDA OOM and Recovery Runbooks](part-04-vllm/024-vllm-slowness-cuda-oom-recovery-runbooks.md) — [Lab](labs/024-vllm-slowness-cuda-oom-recovery-runbooks-check.md).

**Part 04:** 9/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 025 — vLLM Production Acceptance Project](part-04-vllm/025-vllm-production-acceptance-project.md) — [Lab](labs/025-vllm-production-acceptance-project-check.md).

**Part 04:** 10/10 tutorials and companion labs published (written curriculum complete); technical/security review, live execution and production acceptance pending.

**Published:** [Chapter 026 — Triton Inference Server Architecture](part-05-triton-tensorrt-llm/026-triton-inference-server-architecture.md) — [Lab](labs/026-triton-inference-server-architecture-check.md).

**Part 05:** 1/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 027 — Triton Model Repository and Model Lifecycle](part-05-triton-tensorrt-llm/027-triton-model-repository-model-lifecycle.md) — [Lab](labs/027-triton-model-repository-model-lifecycle-check.md).

**Part 05:** 2/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 028 — Triton Scheduling, Dynamic Batching and Ensembles](part-05-triton-tensorrt-llm/028-triton-scheduling-dynamic-batching-ensembles.md) — [Lab](labs/028-triton-scheduling-dynamic-batching-ensembles-check.md).

**Part 05:** 3/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 029 — Triton GPU Metrics and Model Performance](part-05-triton-tensorrt-llm/029-triton-gpu-metrics-model-performance.md) — [Lab](labs/029-triton-gpu-metrics-model-performance-check.md).

**Part 05:** 4/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 030 — Triton Kubernetes Deployment and Operations](part-05-triton-tensorrt-llm/030-triton-kubernetes-deployment-operations.md) — [Lab](labs/030-triton-kubernetes-deployment-operations-check.md).

**Part 05:** 5/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 031 — TensorRT-LLM Architecture and Compilation](part-05-triton-tensorrt-llm/031-tensorrt-llm-architecture-compilation.md) — [Lab](labs/031-tensorrt-llm-architecture-compilation-check.md).

**Part 05:** 6/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 032 — TensorRT-LLM Quantization and Engine Compatibility](part-05-triton-tensorrt-llm/032-tensorrt-llm-quantization-engine-compatibility.md) — [Lab](labs/032-tensorrt-llm-quantization-engine-compatibility-check.md).

**Part 05:** 7/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 033 — TensorRT-LLM Throughput, Latency and KV Cache](part-05-triton-tensorrt-llm/033-tensorrt-llm-throughput-latency-kv-cache.md) — [Lab](labs/033-tensorrt-llm-throughput-latency-kv-cache-check.md).

**Part 05:** 8/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 034 — Triton and TensorRT-LLM Troubleshooting](part-05-triton-tensorrt-llm/034-triton-tensorrt-llm-troubleshooting.md) — [Lab](labs/034-triton-tensorrt-llm-troubleshooting-check.md).

**Part 05:** 9/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 035 — NVIDIA Inference Stack Acceptance Lab](part-05-triton-tensorrt-llm/035-nvidia-inference-stack-acceptance-lab.md) — [Acceptance worksheet](labs/035-nvidia-inference-stack-acceptance-lab-check.md).

**Part 05:** 10/10 written tutorials and companion labs published (written curriculum complete); live execution, technical/security review and operational acceptance pending.

**Published:** [Chapter 036 — Ray Serve Architecture and Deployment](part-06-ray-serve-kserve/036-ray-serve-architecture-deployment.md) — [Lab](labs/036-ray-serve-architecture-deployment-check.md).

**Part 06:** 1/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 037 — Ray Actors, Placement Groups and GPU Resources](part-06-ray-serve-kserve/037-ray-actors-placement-groups-gpu-resources.md) — [Lab](labs/037-ray-actors-placement-groups-gpu-resources-check.md).

**Part 06:** 2/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 038 — Ray Serve Routing, Batching and Backpressure](part-06-ray-serve-kserve/038-ray-serve-routing-batching-backpressure.md) — [Lab](labs/038-ray-serve-routing-batching-backpressure-check.md).

**Part 06:** 3/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 039 — Ray Serve Autoscaling and Fault Tolerance](part-06-ray-serve-kserve/039-ray-serve-autoscaling-fault-tolerance.md) — [Lab](labs/039-ray-serve-autoscaling-fault-tolerance-check.md).

**Part 06:** 4/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 040 — Ray Serve Observability and Recovery](part-06-ray-serve-kserve/040-ray-serve-observability-recovery.md) — [Lab](labs/040-ray-serve-observability-recovery-check.md).

**Part 06:** 5/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 041 — KServe Architecture, CRDs and InferenceServices](part-06-ray-serve-kserve/041-kserve-architecture-crds-inferenceservices.md) — [Lab](labs/041-kserve-architecture-crds-inferenceservices-check.md).

**Part 06:** 6/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 042 — KServe Runtime Selection and Model Storage](part-06-ray-serve-kserve/042-kserve-runtime-selection-model-storage.md) — [Lab](labs/042-kserve-runtime-selection-model-storage-check.md).

**Part 06:** 7/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 043 — KServe Networking, Security and Canary Rollouts](part-06-ray-serve-kserve/043-kserve-networking-security-canary-rollouts.md) — [Lab](labs/043-kserve-networking-security-canary-rollouts-check.md).

**Part 06:** 8/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 044 — KServe Autoscaling and GPU Scheduling](part-06-ray-serve-kserve/044-kserve-autoscaling-gpu-scheduling.md) — [Lab](labs/044-kserve-autoscaling-gpu-scheduling-check.md).

**Part 06:** 9/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 045 — Ray Serve and KServe Operations Project](part-06-ray-serve-kserve/045-ray-serve-kserve-operations-project.md) — [Lab](labs/045-ray-serve-kserve-operations-project-check.md).

**Part 06:** 10/10 tutorials and companion labs published — written curriculum complete; technical/security review, live execution and operational acceptance pending.

**Published:** [Chapter 046 — Kubernetes GPU Scheduling Fundamentals](part-07-kubernetes-gpu-scheduling-autoscaling/046-kubernetes-gpu-scheduling-fundamentals.md) — [Lab](labs/046-kubernetes-gpu-scheduling-fundamentals-check.md).

**Part 07:** 1/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 047 — NVIDIA Device Plugin and GPU Operator](part-07-kubernetes-gpu-scheduling-autoscaling/047-nvidia-device-plugin-gpu-operator.md) — [Lab](labs/047-nvidia-device-plugin-gpu-operator-check.md).

**Part 07:** 2/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 048 — GPU Node Pools, Taints, Tolerations and Affinity](part-07-kubernetes-gpu-scheduling-autoscaling/048-gpu-node-pools-taints-tolerations-affinity.md) — [Lab](labs/048-gpu-node-pools-taints-tolerations-affinity-check.md).

**Part 07:** 3/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 049 — GPU Resource Quotas and Namespace Isolation](part-07-kubernetes-gpu-scheduling-autoscaling/049-gpu-resource-quotas-namespace-isolation.md) — [Lab](labs/049-gpu-resource-quotas-namespace-isolation-check.md).

**Part 07:** 4/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 050 — MIG, GPU Sharing and Multi-Tenant Risks](part-07-kubernetes-gpu-scheduling-autoscaling/050-mig-gpu-sharing-multi-tenant-risks.md) — [Lab](labs/050-mig-gpu-sharing-multi-tenant-risks-check.md).

**Part 07:** 5/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 051 — HPA, KEDA and Custom Inference Metrics](part-07-kubernetes-gpu-scheduling-autoscaling/051-hpa-keda-custom-inference-metrics.md) — [Lab](labs/051-hpa-keda-custom-inference-metrics-check.md).

**Part 07:** 6/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 052 — Cluster Autoscaler, Karpenter and GPU Warmup](part-07-kubernetes-gpu-scheduling-autoscaling/052-cluster-autoscaler-karpenter-gpu-warmup.md) — [Lab](labs/052-cluster-autoscaler-karpenter-gpu-warmup-check.md).

**Part 07:** 7/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 053 — Multi-Zone GPU Capacity and Disruption Budgets](part-07-kubernetes-gpu-scheduling-autoscaling/053-multi-zone-gpu-capacity-disruption-budgets.md) — [Lab](labs/053-multi-zone-gpu-capacity-disruption-budgets-check.md).

**Part 07:** 8/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 054 — GPU Scheduling, Quota and Pending Pod Troubleshooting](part-07-kubernetes-gpu-scheduling-autoscaling/054-gpu-scheduling-quota-pending-pod-troubleshooting.md) — [Lab](labs/054-gpu-scheduling-quota-pending-pod-troubleshooting-check.md).

**Part 07:** 9/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 055 — GPU Autoscaling Acceptance Lab](part-07-kubernetes-gpu-scheduling-autoscaling/055-gpu-autoscaling-acceptance-lab.md) — [Lab](labs/055-gpu-autoscaling-acceptance-lab-check.md).

**Part 07:** 10/10 tutorials and companion labs published (written curriculum complete); technical/security review and live execution pending.

**Published:** [Chapter 056 — Golden Signals for LLM Inference](part-08-llm-observability-metrics-tracing/056-golden-signals-llm-inference.md) — [Lab](labs/056-golden-signals-llm-inference-check.md).

**Part 08:** 1/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 057 — TTFT, TPOT, ITL, E2E Latency and Completion](part-08-llm-observability-metrics-tracing/057-ttft-tpot-itl-e2e-latency-completion.md) — [Lab](labs/057-ttft-tpot-itl-e2e-latency-completion-check.md).

**Part 08:** 2/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 058 — vLLM, Triton and Ray Serve Metrics](part-08-llm-observability-metrics-tracing/058-vllm-triton-ray-serve-metrics.md) — [Lab](labs/058-vllm-triton-ray-serve-metrics-check.md).

**Part 08:** 3/10 tutorials and companion labs published; technical/security review and live execution pending.

**Published:** [Chapter 059 — GPU DCGM, Node and Kubernetes Observability](part-08-llm-observability-metrics-tracing/059-gpu-dcgm-node-kubernetes-observability.md) — [Lab](labs/059-gpu-dcgm-node-kubernetes-observability-check.md).

**Part 08:** 4/10 tutorials and companion labs published; technical/security review and live execution pending.

**Next:** Chapter 060 — OpenTelemetry Traces Across Inference Paths.

- [Chapter 060 — OpenTelemetry Traces Across Inference Paths](part-08-llm-observability-metrics-tracing/060-opentelemetry-traces-across-inference-paths.md) — [Lab](labs/060-opentelemetry-traces-across-inference-paths-check.md) (published; review and execution pending).

- [Chapter 061 — Prompt-Safe Logging, Redaction and Cardinality](part-08-llm-observability-metrics-tracing/061-prompt-safe-logging-redaction-cardinality.md) — [Lab](labs/061-prompt-safe-logging-redaction-cardinality-check.md) (published; reviews and execution pending).

- [Chapter 062 — Prometheus Recording Rules and Alert Design](part-08-llm-observability-metrics-tracing/062-prometheus-recording-rules-alert-design.md) — [Lab](labs/062-prometheus-recording-rules-alert-design-check.md) (published; reviews and execution pending).

- [Chapter 063 — SLO Burn Rates and Error Budget Policy](part-08-llm-observability-metrics-tracing/063-slo-burn-rates-error-budget-policy.md) — [Lab](labs/063-slo-burn-rates-error-budget-policy-check.md) (published; reviews and execution pending).

- [Chapter 064 — Dashboards, Synthetic Probes and On-Call](part-08-llm-observability-metrics-tracing/064-dashboards-synthetic-probes-on-call.md) — [Lab](labs/064-dashboards-synthetic-probes-on-call-check.md) (published; reviews and execution pending).

- [Chapter 065 — Inference Observability Acceptance Lab](part-08-llm-observability-metrics-tracing/065-inference-observability-acceptance-lab.md) — [Lab](labs/065-inference-observability-acceptance-lab-check.md) (published; Part 08 chapters 056–065 complete; reviews and execution pending).

- [Chapter 066 — Inference Benchmark Design and Traffic Cohorts](part-09-performance-engineering-capacity/066-inference-benchmark-design-traffic-cohorts.md) — [Lab](labs/066-inference-benchmark-design-traffic-cohorts-check.md) (published; Part 09 started; reviews and execution pending).

- [Chapter 067 — Prefill-Heavy Versus Decode-Heavy Workloads](part-09-performance-engineering-capacity/067-prefill-heavy-versus-decode-heavy-workloads.md) — [Lab](labs/067-prefill-heavy-versus-decode-heavy-workloads-check.md) (published; reviews and execution pending).

- [Chapter 068 — Concurrency, Queueing and Backpressure](part-09-performance-engineering-capacity/068-concurrency-queueing-backpressure.md) — [Lab](labs/068-concurrency-queueing-backpressure-check.md) (published; reviews and execution pending).

- [Chapter 069 — Batching, KV Cache and Memory Tuning](part-09-performance-engineering-capacity/069-batching-kv-cache-memory-tuning.md) — [Lab](labs/069-batching-kv-cache-memory-tuning-check.md) (published; reviews and execution pending).

- [Chapter 070 — Model Precision, Quantization and Quality Gates](part-09-performance-engineering-capacity/070-model-precision-quantization-quality-gates.md) — [Lab](labs/070-model-precision-quantization-quality-gates-check.md) (published; reviews and execution pending).

- [Chapter 071 — GPU Throughput and Cost per Successful Token](part-09-performance-engineering-capacity/071-gpu-throughput-cost-per-successful-token.md) — [Lab](labs/071-gpu-throughput-cost-per-successful-token-check.md) (published; reviews and execution pending).
