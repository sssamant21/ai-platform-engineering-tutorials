# LLMOps & AI Inference SRE — Master Table of Contents

**Curriculum:** 100 planned chapters across 12 parts. **Publication:** 17/100 chapters; **Lab execution:** 0; **Technical/security review:** pending. This is a proposed learning plan, not a claim of completed tutorials.

## Part 01 — AI/LLM SRE Fundamentals (Chapters 001–005)

- [001 — Introduction to LLMOps and AI Inference SRE](part-01-fundamentals/001-introduction-to-llmops-ai-inference-sre.md) — [Hands-on Lab](labs/001-introduction-to-llmops-ai-inference-sre-check.md) (published)
- [002 — Production LLM Service Architecture and SRE Responsibilities](part-01-fundamentals/002-production-llm-service-architecture-sre-responsibilities.md) — [Hands-on Lab](labs/002-production-llm-service-architecture-sre-responsibilities-check.md) (published)
- [003 — SLIs, SLOs, Error Budgets and Inference User Journeys](part-01-fundamentals/003-slis-slos-error-budgets-inference-user-journeys.md) — [Hands-on Lab](labs/003-slis-slos-error-budgets-inference-user-journeys-check.md) (published)
- [004 — Hands-On Lab Environment and Safety Controls](part-01-fundamentals/004-hands-on-lab-environment-safety-controls.md) — [Hands-on Lab](labs/004-hands-on-lab-environment-safety-controls-check.md) (published)
- [005 — Fundamentals Acceptance Lab and Incident Walkthrough](part-01-fundamentals/005-fundamentals-acceptance-lab-incident-walkthrough.md) — [Capstone Lab](labs/005-fundamentals-acceptance-lab-incident-walkthrough-check.md) (published)

## Part 02 — LLM Architecture and Inference Internals (Chapters 006–010)

- [006 — Transformer Inference Request Lifecycle](part-02-inference-internals/006-transformer-inference-request-lifecycle.md) — [Hands-on Lab](labs/006-transformer-inference-request-lifecycle-check.md) (published)
- [007 — Tokenization, Context Windows and Generation Parameters](part-02-inference-internals/007-tokenization-context-windows-generation-parameters.md) — [Hands-on Lab](labs/007-tokenization-context-windows-generation-parameters-check.md) (published)
- [008 — Prefill, Decode, KV Cache and Continuous Batching](part-02-inference-internals/008-prefill-decode-kv-cache-continuous-batching.md) — [Hands-on Lab](labs/008-prefill-decode-kv-cache-continuous-batching-check.md) (published)
- [009 — Streaming, Backpressure and API Gateway Behavior](part-02-inference-internals/009-streaming-backpressure-api-gateway-behavior.md) — [Hands-on Lab](labs/009-streaming-backpressure-api-gateway-behavior-check.md) (published)
- [010 — Inference Internals Performance Lab and Reliability Acceptance Project](part-02-inference-internals/010-inference-internals-performance-reliability-acceptance.md) — [Capstone Lab](labs/010-inference-internals-performance-reliability-acceptance-check.md) (published)

## Part 03 — GPU Architecture, CUDA and VRAM (Chapters 011–015)

- [011 — GPU Architecture, SMs, HBM and Memory Bandwidth](part-03-gpu-cuda-vram/011-gpu-architecture-sms-hbm-memory-bandwidth.md) — [Hands-on Lab](labs/011-gpu-architecture-sms-hbm-memory-bandwidth-check.md) (published)
- [012 — CUDA Runtime, Drivers, Containers and Compatibility](part-03-gpu-cuda-vram/012-cuda-runtime-drivers-containers-compatibility.md) — [Hands-on Lab](labs/012-cuda-runtime-drivers-containers-compatibility-check.md) (published)
- [013 — GPU VRAM Sizing, Precision and Quantization](part-03-gpu-cuda-vram/013-gpu-vram-sizing-precision-quantization.md) — [Hands-on Lab](labs/013-gpu-vram-sizing-precision-quantization-check.md) (published)
- [014 — Multi-GPU Parallelism and Topology](part-03-gpu-cuda-vram/014-multi-gpu-parallelism-topology.md) — [Hands-on Lab](labs/014-multi-gpu-parallelism-topology-check.md) (published)
- [015 — GPU Diagnostics, DCGM and Failure Lab](part-03-gpu-cuda-vram/015-gpu-diagnostics-dcgm-failure-lab.md) — [Capstone Lab](labs/015-gpu-diagnostics-dcgm-failure-lab-check.md) (published)

## Part 04 — vLLM Administration and Operations (Chapters 016–025)

- [016 — vLLM Architecture and Production Deployment](part-04-vllm/016-vllm-architecture-production-deployment.md) — [Hands-on Lab](labs/016-vllm-architecture-production-deployment-check.md) (published)
- [017 — vLLM Model Loading and Runtime Configuration](part-04-vllm/017-vllm-model-loading-runtime-configuration.md) — [Hands-on Lab](labs/017-vllm-model-loading-runtime-configuration-check.md) (published)
- 018 — Continuous Batching, PagedAttention and KV Cache (planned)
- 019 — vLLM Streaming and OpenAI-Compatible APIs (planned)
- 020 — vLLM GPU Memory and Context Tuning (planned)
- 021 — vLLM Tensor/Pipeline Parallelism and Distributed Serving (planned)
- 022 — vLLM Security, Authentication and Network Isolation (planned)
- 023 — vLLM Metrics, Logging and Distributed Tracing (planned)
- 024 — vLLM Slowness, CUDA OOM and Recovery Runbooks (planned)
- 025 — vLLM Production Acceptance Project (planned)

## Part 05 — NVIDIA Triton and TensorRT-LLM (Chapters 026–035)

- 026 — Triton Inference Server Architecture (planned)
- 027 — Triton Model Repository and Model Lifecycle (planned)
- 028 — Triton Scheduling, Dynamic Batching and Ensembles (planned)
- 029 — Triton GPU Metrics and Model Performance (planned)
- 030 — Triton Kubernetes Deployment and Operations (planned)
- 031 — TensorRT-LLM Architecture and Compilation (planned)
- 032 — TensorRT-LLM Quantization and Engine Compatibility (planned)
- 033 — TensorRT-LLM Throughput, Latency and KV Cache (planned)
- 034 — Triton and TensorRT-LLM Troubleshooting (planned)
- 035 — NVIDIA Inference Stack Acceptance Lab (planned)

## Part 06 — Ray Serve and KServe (Chapters 036–045)

- 036 — Ray Serve Architecture and Deployment (planned)
- 037 — Ray Actors, Placement Groups and GPU Resources (planned)
- 038 — Ray Serve Routing, Batching and Backpressure (planned)
- 039 — Ray Serve Autoscaling and Fault Tolerance (planned)
- 040 — Ray Serve Observability and Recovery (planned)
- 041 — KServe Architecture, CRDs and InferenceServices (planned)
- 042 — KServe Runtime Selection and Model Storage (planned)
- 043 — KServe Networking, Security and Canary Rollouts (planned)
- 044 — KServe Autoscaling and GPU Scheduling (planned)
- 045 — Ray Serve and KServe Operations Project (planned)

## Part 07 — Kubernetes GPU Scheduling and Autoscaling (Chapters 046–055)

- 046 — Kubernetes GPU Scheduling Fundamentals (planned)
- 047 — NVIDIA Device Plugin and GPU Operator (planned)
- 048 — GPU Node Pools, Taints, Tolerations and Affinity (planned)
- 049 — GPU Resource Quotas and Namespace Isolation (planned)
- 050 — MIG, GPU Sharing and Multi-Tenant Risks (planned)
- 051 — HPA, KEDA and Custom Inference Metrics (planned)
- 052 — Cluster Autoscaler, Karpenter and GPU Warmup (planned)
- 053 — Multi-Zone GPU Capacity and Disruption Budgets (planned)
- 054 — GPU Scheduling, Quota and Pending Pod Troubleshooting (planned)
- 055 — GPU Autoscaling Acceptance Lab (planned)

## Part 08 — LLM Observability, Metrics and Tracing (Chapters 056–065)

- 056 — Golden Signals for LLM Inference (planned)
- 057 — TTFT, TPOT, ITL, E2E Latency and Completion (planned)
- 058 — vLLM, Triton and Ray Serve Metrics (planned)
- 059 — GPU DCGM, Node and Kubernetes Observability (planned)
- 060 — OpenTelemetry Traces Across Inference Paths (planned)
- 061 — Prompt-Safe Logging, Redaction and Cardinality (planned)
- 062 — Prometheus Recording Rules and Alert Design (planned)
- 063 — SLO Burn Rates and Error Budget Policy (planned)
- 064 — Dashboards, Synthetic Probes and On-Call (planned)
- 065 — Inference Observability Acceptance Lab (planned)

## Part 09 — Performance Engineering and Capacity (Chapters 066–075)

- 066 — Inference Benchmark Design and Traffic Cohorts (planned)
- 067 — Prefill-Heavy Versus Decode-Heavy Workloads (planned)
- 068 — Concurrency, Queueing and Backpressure (planned)
- 069 — Batching, KV Cache and Memory Tuning (planned)
- 070 — Model Precision, Quantization and Quality Gates (planned)
- 071 — GPU Throughput and Cost per Successful Token (planned)
- 072 — Cold Start, Warm Pools and Model Loading (planned)
- 073 — Load, Stress, Soak and Failure Testing (planned)
- 074 — Capacity Forecasting, N-1 and Rollout Headroom (planned)
- 075 — Performance and Capacity Acceptance Lab (planned)

## Part 10 — AI Security, Governance and FinOps (Chapters 076–085)

- 076 — LLM Threat Modeling and Trust Boundaries (planned)
- 077 — Inference Authentication, Authorization and Tenant Isolation (planned)
- 078 — Prompt Injection, Data Leakage and Model Abuse (planned)
- 079 — Secrets, Workload Identity and Network Policies (planned)
- 080 — Model Artifact Supply Chain, SBOM and Provenance (planned)
- 081 — Privacy-Safe Telemetry and Data Retention (planned)
- 082 — AI Gateway Rate Limits and Denial-of-Wallet Protection (planned)
- 083 — GPU FinOps, Budget Alerts and Rightsizing (planned)
- 084 — Policy-as-Code, Change Management and Audit (planned)
- 085 — Security, Governance and FinOps Acceptance Lab (planned)

## Part 11 — Incident Response, HA and Disaster Recovery (Chapters 086–095)

- 086 — LLM Incident Command, SEV and Escalation (planned)
- 087 — Inference 429, 5xx, Timeout and SSE Incident Runbooks (planned)
- 088 — GPU XID, ECC, Driver and CUDA Incident Response (planned)
- 089 — Model Quality and Release Regression Incidents (planned)
- 090 — TrueFoundry, Ray Serve and KServe Control Plane Failures (planned)
- 091 — High Availability, N-1 and Multi-Zone Recovery (planned)
- 092 — Model Artifact, Registry and Configuration Recovery (planned)
- 093 — Backup, DR, RTO/RPO and Region Failover (planned)
- 094 — Chaos Engineering and Recovery Game Days (planned)
- 095 — Incident Response and Disaster Recovery Acceptance Lab (planned)

## Part 12 — Final Production AI SRE Capstone (Chapters 096–100)

- 096 — Enterprise LLM Inference Reference Architecture (planned)
- 097 — Build a Secure Multi-Environment GPU Serving Platform (planned)
- 098 — Implement SLOs, Load Testing and Capacity Governance (planned)
- 099 — Run Compound Failure and Incident Recovery Exercises (planned)
- 100 — Final AI Inference SRE Production Readiness Assessment (planned)

