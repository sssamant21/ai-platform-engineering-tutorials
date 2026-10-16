# LLMOps & AI Inference Site Reliability Engineering

**Repository:** [AI Platform Engineering Tutorials](https://github.com/sssamant21/ai-platform-engineering-tutorials)  
**Track:** `llmops-sre/`  
**Scope:** 100 planned chapters, 12 parts. **Current status:** 13/100 chapters and companion labs published; Part 01 written curriculum complete (5/5); technical review and live execution pending.

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

**Next:** Chapter 014 — Multi-GPU Parallelism and Topology.
