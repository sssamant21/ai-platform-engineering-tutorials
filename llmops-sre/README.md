# LLMOps & AI Inference Site Reliability Engineering

**Repository:** [AI Platform Engineering Tutorials](https://github.com/sssamant21/ai-platform-engineering-tutorials)  
**Track:** `llmops-sre/`  
**Scope:** 100 planned chapters, 12 parts. **Current status:** 3/100 chapters and companion labs published; technical review and live execution pending.

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

**Next:** Chapter 004 — Hands-On Lab Environment and Safety Controls.
