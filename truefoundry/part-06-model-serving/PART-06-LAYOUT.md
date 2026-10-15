# Part 06 — Model Serving: Master Layout

**Scope:** model-serving architecture, artifacts and storage, CPU/GPU runtimes, inference endpoints, lifecycle/health, observability and incident recovery.

| Chapter | Topic | Canonical | Hands-on Lab |
|---|---|---|---|
| 6.1 | Model-Serving Fundamentals and Architecture | [Tutorial](01-model-serving-fundamentals-architecture/01-model-serving-fundamentals-architecture.md) | [Lab](labs/01-model-serving-fundamentals-architecture-check.md) |
| 6.2 | Model Artifacts, Registries and Storage Integration | [Tutorial](02-model-artifacts-registries-storage/02-model-artifacts-registries-storage.md) | [Lab](labs/02-model-artifacts-registries-storage-check.md) |
| 6.3 | CPU and GPU Inference Runtimes and Scheduling | [Tutorial](03-cpu-gpu-inference-runtimes-scheduling/03-cpu-gpu-inference-runtimes-scheduling.md) | [Lab](labs/03-cpu-gpu-inference-runtimes-scheduling-check.md) |
| 6.4 | Inference Endpoints, Request Lifecycle and API Reliability | [Tutorial](04-inference-endpoints-request-lifecycle-api-reliability/04-inference-endpoints-request-lifecycle-api-reliability.md) | [Lab](labs/04-inference-endpoints-request-lifecycle-api-reliability-check.md) |
| 6.5 | Model Loading, Warmup, Health Probes and Readiness | Planned | Planned |
| 6.6 | Model-Serving Observability, Performance and Troubleshooting | Planned | Planned |
| 6.7 | Production Model-Serving Deployment and Recovery Capstone | Planned | Planned |

**Publication:** 4/7 chapters and 4/7 labs published. Formal technical/vendor validation, SRE review and live lab execution are separate pending milestones.

**Operating principles:** immutable image and model versions; identity isolation; production-safe discovery; lowest proven healthy layer and first failed transition; authoritative rollback and verifiable end-to-end inference.
