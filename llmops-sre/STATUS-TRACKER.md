# LLMOps & AI Inference SRE — Status Tracker

**Status:** Chapter publication in progress. **Published:** 2/100. **Live labs executed:** 0. **Reviews:** pending.

Status definitions: Planned → Drafted → Technical reviewed → Security reviewed → Lab published → GitHub verified → Lab executed → Accepted. Publication and lab execution are separate.

| Chapter | Topic | Draft | Review | Tutorial | Lab | Live execution |
|---|---|---|---|---|---|---|
| 001 | Introduction to LLMOps and AI Inference SRE | ✅ | ⬜ Pending | ✅ Published | ✅ Published | ⬜ Pending |
| 002 | Production LLM Service Architecture and SRE Responsibilities | ✅ | ⬜ Pending | ✅ Published | ✅ Published | ⬜ Pending |
| 003 | SLIs, SLOs, Error Budgets and Inference User Journeys | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 004 | Hands-On Lab Environment and Safety Controls | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 005 | Fundamentals Acceptance Lab and Incident Walkthrough | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 006 | Transformer Inference Request Lifecycle | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 007 | Tokenization, Context Windows and Generation Parameters | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 008 | Prefill, Decode, KV Cache and Continuous Batching | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 009 | Streaming, Backpressure and API Gateway Behavior | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 010 | Inference Internals Performance Lab | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 011 | GPU Architecture, SMs, HBM and Memory Bandwidth | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 012 | CUDA Runtime, Drivers, Containers and Compatibility | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 013 | GPU VRAM Sizing, Precision and Quantization | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 014 | Multi-GPU Parallelism and Topology | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 015 | GPU Diagnostics, DCGM and Failure Lab | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 016 | vLLM Architecture and Production Deployment | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 017 | vLLM Model Loading and Runtime Configuration | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 018 | Continuous Batching, PagedAttention and KV Cache | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 019 | vLLM Streaming and OpenAI-Compatible APIs | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 020 | vLLM GPU Memory and Context Tuning | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 021 | vLLM Tensor/Pipeline Parallelism and Distributed Serving | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 022 | vLLM Security, Authentication and Network Isolation | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 023 | vLLM Metrics, Logging and Distributed Tracing | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 024 | vLLM Slowness, CUDA OOM and Recovery Runbooks | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 025 | vLLM Production Acceptance Project | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 026 | Triton Inference Server Architecture | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 027 | Triton Model Repository and Model Lifecycle | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 028 | Triton Scheduling, Dynamic Batching and Ensembles | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 029 | Triton GPU Metrics and Model Performance | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 030 | Triton Kubernetes Deployment and Operations | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 031 | TensorRT-LLM Architecture and Compilation | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 032 | TensorRT-LLM Quantization and Engine Compatibility | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 033 | TensorRT-LLM Throughput, Latency and KV Cache | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 034 | Triton and TensorRT-LLM Troubleshooting | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 035 | NVIDIA Inference Stack Acceptance Lab | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 036 | Ray Serve Architecture and Deployment | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 037 | Ray Actors, Placement Groups and GPU Resources | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 038 | Ray Serve Routing, Batching and Backpressure | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 039 | Ray Serve Autoscaling and Fault Tolerance | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 040 | Ray Serve Observability and Recovery | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 041 | KServe Architecture, CRDs and InferenceServices | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 042 | KServe Runtime Selection and Model Storage | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 043 | KServe Networking, Security and Canary Rollouts | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 044 | KServe Autoscaling and GPU Scheduling | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 045 | Ray Serve and KServe Operations Project | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 046 | Kubernetes GPU Scheduling Fundamentals | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 047 | NVIDIA Device Plugin and GPU Operator | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 048 | GPU Node Pools, Taints, Tolerations and Affinity | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 049 | GPU Resource Quotas and Namespace Isolation | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 050 | MIG, GPU Sharing and Multi-Tenant Risks | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 051 | HPA, KEDA and Custom Inference Metrics | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 052 | Cluster Autoscaler, Karpenter and GPU Warmup | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 053 | Multi-Zone GPU Capacity and Disruption Budgets | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 054 | GPU Scheduling, Quota and Pending Pod Troubleshooting | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 055 | GPU Autoscaling Acceptance Lab | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 056 | Golden Signals for LLM Inference | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 057 | TTFT, TPOT, ITL, E2E Latency and Completion | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 058 | vLLM, Triton and Ray Serve Metrics | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 059 | GPU DCGM, Node and Kubernetes Observability | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 060 | OpenTelemetry Traces Across Inference Paths | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 061 | Prompt-Safe Logging, Redaction and Cardinality | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 062 | Prometheus Recording Rules and Alert Design | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 063 | SLO Burn Rates and Error Budget Policy | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 064 | Dashboards, Synthetic Probes and On-Call | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 065 | Inference Observability Acceptance Lab | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 066 | Inference Benchmark Design and Traffic Cohorts | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 067 | Prefill-Heavy Versus Decode-Heavy Workloads | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 068 | Concurrency, Queueing and Backpressure | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 069 | Batching, KV Cache and Memory Tuning | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 070 | Model Precision, Quantization and Quality Gates | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 071 | GPU Throughput and Cost per Successful Token | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 072 | Cold Start, Warm Pools and Model Loading | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 073 | Load, Stress, Soak and Failure Testing | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 074 | Capacity Forecasting, N-1 and Rollout Headroom | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 075 | Performance and Capacity Acceptance Lab | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 076 | LLM Threat Modeling and Trust Boundaries | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 077 | Inference Authentication, Authorization and Tenant Isolation | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 078 | Prompt Injection, Data Leakage and Model Abuse | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 079 | Secrets, Workload Identity and Network Policies | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 080 | Model Artifact Supply Chain, SBOM and Provenance | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 081 | Privacy-Safe Telemetry and Data Retention | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 082 | AI Gateway Rate Limits and Denial-of-Wallet Protection | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 083 | GPU FinOps, Budget Alerts and Rightsizing | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 084 | Policy-as-Code, Change Management and Audit | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 085 | Security, Governance and FinOps Acceptance Lab | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 086 | LLM Incident Command, SEV and Escalation | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 087 | Inference 429, 5xx, Timeout and SSE Incident Runbooks | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 088 | GPU XID, ECC, Driver and CUDA Incident Response | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 089 | Model Quality and Release Regression Incidents | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 090 | TrueFoundry, Ray Serve and KServe Control Plane Failures | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 091 | High Availability, N-1 and Multi-Zone Recovery | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 092 | Model Artifact, Registry and Configuration Recovery | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 093 | Backup, DR, RTO/RPO and Region Failover | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 094 | Chaos Engineering and Recovery Game Days | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 095 | Incident Response and Disaster Recovery Acceptance Lab | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 096 | Enterprise LLM Inference Reference Architecture | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 097 | Build a Secure Multi-Environment GPU Serving Platform | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 098 | Implement SLOs, Load Testing and Capacity Governance | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 099 | Run Compound Failure and Incident Recovery Exercises | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 100 | Final AI Inference SRE Production Readiness Assessment | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |

## Chapter 001 publication evidence
- [Canonical tutorial](part-01-fundamentals/001-introduction-to-llmops-ai-inference-sre.md) — published and verified.
- [Companion lab](labs/001-introduction-to-llmops-ai-inference-sre-check.md) — published and verified.
- Technical/security review, live lab execution and acceptance remain pending.
- Next: Chapter 003 — SLIs, SLOs, Error Budgets and Inference User Journeys.

## Chapter 002 publication evidence
- [Canonical tutorial](part-01-fundamentals/002-production-llm-service-architecture-sre-responsibilities.md) — published and verified.
- [Companion lab](labs/002-production-llm-service-architecture-sre-responsibilities-check.md) — published and verified.
- Technical/security review, live lab execution and acceptance remain pending.
- Next: Chapter 003 — SLIs, SLOs, Error Budgets and Inference User Journeys.
