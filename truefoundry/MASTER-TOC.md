# TrueFoundry — Master Table of Contents

## Part 1 — Foundations & Architecture
- 1.1 — What Is TrueFoundry?
- 1.2 — TrueFoundry Architecture: Control Plane, Compute Plane & `tfy-agent`
- 1.3 — TrueFoundry Platform Components & Terminology
- 1.4 — TrueFoundry and Kubernetes: How They Work Together
- 1.5 — Workspaces, Environments & Deployment Concepts
- 1.6 — SRE Ownership, Failure Domains & Support Model

## Part 2 — Kubernetes Integration
2.1 Cluster Connection; 2.2 `tfy-agent`; 2.3 Namespaces & Placement; 2.4 RBAC; 2.5 ConfigMaps/Secrets; 2.6 Services/Ingress/DNS/TLS; 2.7 Storage.

## Part 3 — Compute & Scheduling
CPU/memory, requests/limits/QoS, affinity, taints/tolerations, dedicated node pools, scheduling troubleshooting.

## Part 4 — GPU Infrastructure
- [4.1 — GPU Architecture and NVIDIA GPU Fundamentals](part-04-gpu-infrastructure/01-gpu-architecture-nvidia-fundamentals/01-gpu-architecture-nvidia-fundamentals.md) — [Hands-on Lab](part-04-gpu-infrastructure/labs/01-gpu-architecture-nvidia-fundamentals-check.md) (published)
- [4.2 — GPU Nodes and Kubernetes GPU Scheduling](part-04-gpu-infrastructure/02-gpu-nodes-kubernetes-scheduling/02-gpu-nodes-kubernetes-scheduling.md) — [Hands-on Lab](part-04-gpu-infrastructure/labs/02-gpu-nodes-kubernetes-scheduling-check.md) (published)
- [4.3 — NVIDIA Drivers, Device Plugin and GPU Operator](part-04-gpu-infrastructure/03-nvidia-drivers-device-plugin-gpu-operator/03-nvidia-drivers-device-plugin-gpu-operator.md) — [Hands-on Lab](part-04-gpu-infrastructure/labs/03-nvidia-drivers-device-plugin-gpu-operator-check.md) (published)
- [4.4 — CUDA, Container Runtime and Compatibility](part-04-gpu-infrastructure/04-cuda-container-runtime-compatibility/04-cuda-container-runtime-compatibility.md) — [Hands-on Lab](part-04-gpu-infrastructure/labs/04-cuda-container-runtime-compatibility-check.md) (published)
- [4.5 — Dedicated GPU Node Pools and Resource Isolation](part-04-gpu-infrastructure/05-dedicated-gpu-node-pools-isolation/05-dedicated-gpu-node-pools-isolation.md) — [Hands-on Lab](part-04-gpu-infrastructure/labs/05-dedicated-gpu-node-pools-isolation-check.md) (published)
- [4.6 — GPU Validation, Monitoring and Troubleshooting](part-04-gpu-infrastructure/06-gpu-validation-monitoring-troubleshooting/06-gpu-validation-monitoring-troubleshooting.md) — [Hands-on Lab](part-04-gpu-infrastructure/labs/06-gpu-validation-monitoring-troubleshooting-check.md) (published)
- [4.7 — Production GPU Infrastructure Recovery Lab](part-04-gpu-infrastructure/07-production-gpu-infrastructure-recovery/07-production-gpu-infrastructure-recovery.md) — [Hands-on Lab](part-04-gpu-infrastructure/labs/07-production-gpu-infrastructure-recovery-check.md) (published)
- [Part 4 master layout](part-04-gpu-infrastructure/PART-04-LAYOUT.md)

## Part 5 — Application Deployment
- [5.1 — TrueFoundry Application Deployment Architecture and First Service](part-05-application-deployment/01-truefoundry-application-deployment-first-service/01-truefoundry-application-deployment-first-service.md) — [Hands-on Lab](part-05-application-deployment/labs/01-truefoundry-first-application-deployment-check.md) (published)
- [5.2 — Container Images, Registries and Supply Chain](part-05-application-deployment/02-container-images-registries-supply-chain/02-container-images-registries-supply-chain.md) — [Hands-on Lab](part-05-application-deployment/labs/02-container-images-registries-supply-chain-check.md) (published)
- [5.3 — Application Configuration, Environment Variables and Secrets](part-05-application-deployment/03-application-configuration-environment-secrets/03-application-configuration-environment-secrets.md) — [Hands-on Lab](part-05-application-deployment/labs/03-application-configuration-environment-secrets-check.md) (published)
- [5.4 — Resource Requests, Limits, Probes and Workload Reliability](part-05-application-deployment/04-resource-requests-limits-probes-reliability/04-resource-requests-limits-probes-reliability.md) — [Hands-on Lab](part-05-application-deployment/labs/04-resource-requests-limits-probes-reliability-check.md) (published)
- [5.5 — Services, Endpoints, Routing and Application Connectivity](part-05-application-deployment/05-services-endpoints-routing-connectivity/05-services-endpoints-routing-connectivity.md) — [Hands-on Lab](part-05-application-deployment/labs/05-services-endpoints-routing-connectivity-check.md) (published)
- [5.6 — Deployment Revisions, Rollouts, Rollbacks and Troubleshooting](part-05-application-deployment/06-deployment-revisions-rollouts-rollbacks-troubleshooting/06-deployment-revisions-rollouts-rollbacks-troubleshooting.md) — [Hands-on Lab](part-05-application-deployment/labs/06-deployment-revisions-rollouts-rollbacks-troubleshooting-check.md) (published)
- [5.7 — Production Application Deployment and Recovery Capstone](part-05-application-deployment/07-production-application-deployment-recovery-capstone/07-production-application-deployment-recovery-capstone.md) — [Hands-on Lab](part-05-application-deployment/labs/07-production-application-deployment-recovery-capstone-check.md) (published)
- [Part 5 master layout](part-05-application-deployment/PART-05-LAYOUT.md)

## Part 6 — Model Serving
- [6.1 — Model-Serving Fundamentals and Architecture](part-06-model-serving/01-model-serving-fundamentals-architecture/01-model-serving-fundamentals-architecture.md) — [Hands-on Lab](part-06-model-serving/labs/01-model-serving-fundamentals-architecture-check.md) (published)
- [6.2 — Model Artifacts, Registries and Storage Integration](part-06-model-serving/02-model-artifacts-registries-storage/02-model-artifacts-registries-storage.md) — [Hands-on Lab](part-06-model-serving/labs/02-model-artifacts-registries-storage-check.md) (published)
- [6.3 — CPU and GPU Inference Runtimes and Scheduling](part-06-model-serving/03-cpu-gpu-inference-runtimes-scheduling/03-cpu-gpu-inference-runtimes-scheduling.md) — [Hands-on Lab](part-06-model-serving/labs/03-cpu-gpu-inference-runtimes-scheduling-check.md) (published)
- [6.4 — Inference Endpoints, Request Lifecycle and API Reliability](part-06-model-serving/04-inference-endpoints-request-lifecycle-api-reliability/04-inference-endpoints-request-lifecycle-api-reliability.md) — [Hands-on Lab](part-06-model-serving/labs/04-inference-endpoints-request-lifecycle-api-reliability-check.md) (published)
- [6.5 — Model Loading, Warmup, Health Probes and Readiness](part-06-model-serving/05-model-loading-warmup-health-readiness/05-model-loading-warmup-health-readiness.md) — [Hands-on Lab](part-06-model-serving/labs/05-model-loading-warmup-health-readiness-check.md) (published)
- [6.6 — Model-Serving Observability, Performance and Troubleshooting](part-06-model-serving/06-model-serving-observability-performance-troubleshooting/06-model-serving-observability-performance-troubleshooting.md) — [Hands-on Lab](part-06-model-serving/labs/06-model-serving-observability-performance-troubleshooting-check.md) (published)
- [6.7 — Production Model-Serving Deployment and Recovery Capstone](part-06-model-serving/07-production-model-serving-deployment-recovery-capstone/07-production-model-serving-deployment-recovery-capstone.md) — [Hands-on Lab](part-06-model-serving/labs/07-production-model-serving-deployment-recovery-capstone-check.md) (published)
- [Part 6 master layout](part-06-model-serving/PART-06-LAYOUT.md)

## Part 7 — vLLM & LLM Serving
- [7.1 — vLLM Fundamentals and Serving Architecture](part-07-vllm-llm-serving/01-vllm-fundamentals-serving-architecture/01-vllm-fundamentals-serving-architecture.md) — [Hands-on Lab](part-07-vllm-llm-serving/labs/01-vllm-fundamentals-serving-architecture-check.md) (published)
- [7.2 — vLLM Deployment with TrueFoundry and Kubernetes](part-07-vllm-llm-serving/02-vllm-deployment-truefoundry-kubernetes/02-vllm-deployment-truefoundry-kubernetes.md) — [Hands-on Lab](part-07-vllm-llm-serving/labs/02-vllm-deployment-truefoundry-kubernetes-check.md) (published)
- [7.3 — GPU CUDA PyTorch and Device Detection](part-07-vllm-llm-serving/03-gpu-cuda-pytorch-device-detection/03-gpu-cuda-pytorch-device-detection.md) — [Hands-on Lab](part-07-vllm-llm-serving/labs/03-gpu-cuda-pytorch-device-detection-check.md) (published)
- [7.4 — Model Loading Tokenizers and Tensor Parallelism](part-07-vllm-llm-serving/04-model-loading-tokenizers-tensor-parallelism/04-model-loading-tokenizers-tensor-parallelism.md) — [Hands-on Lab](part-07-vllm-llm-serving/labs/04-model-loading-tokenizers-tensor-parallelism-check.md) (published)
- [7.5 — PagedAttention KV Cache and Continuous Batching](part-07-vllm-llm-serving/05-pagedattention-kv-cache-continuous-batching/05-pagedattention-kv-cache-continuous-batching.md) — [Hands-on Lab](part-07-vllm-llm-serving/labs/05-pagedattention-kv-cache-continuous-batching-check.md) (published)
- [7.6 — vLLM Performance Tuning Observability and Troubleshooting](part-07-vllm-llm-serving/06-vllm-performance-tuning-observability-troubleshooting/06-vllm-performance-tuning-observability-troubleshooting.md) — [Hands-on Lab](part-07-vllm-llm-serving/labs/06-vllm-performance-tuning-observability-troubleshooting-check.md) (published)
- [7.7 — Production vLLM Deployment and Recovery Capstone](part-07-vllm-llm-serving/07-production-vllm-deployment-recovery-capstone/07-production-vllm-deployment-recovery-capstone.md) — [Hands-on Lab](part-07-vllm-llm-serving/labs/07-production-vllm-deployment-recovery-capstone-check.md) (published)
- [Part 7 master layout](part-07-vllm-llm-serving/PART-07-LAYOUT.md)

## Part 8 — Scaling & Performance
Scaling, HPA, GPU autoscaling, concurrency, latency/throughput, GPU utilization, performance troubleshooting.

## Part 9 — Networking
Architecture, Services, Ingress, DNS, TLS, internal/external endpoints, troubleshooting.

## Part 10 — Security
Architecture, authentication/authorization, RBAC, ServiceAccounts, secrets, cloud IAM, isolation, production checklist.

## Part 11 — Observability
Architecture, logs, Kubernetes events, platform/GPU/model metrics, latency/throughput/errors, dashboards/alerts.

## Part 12 — Production Operations
Readiness, capacity, GPU capacity, HA, upgrades, FinOps, operational runbooks.

## Part 13 — Troubleshooting
Methodology, `tfy-agent`, deployment, Pending, CrashLoopBackOff, ImagePullBackOff, RBAC, networking, GPU, NVIDIA/CUDA, vLLM, inference performance, GPU OOM, incidents.

## Part 14 — SRE Runbooks
Platform health, `tfy-agent`, GPU health, model deployment, vLLM health, incidents, post-incident validation.

## Part 15 — End-to-End Production Project
Architecture, Kubernetes, GPU, TrueFoundry, vLLM deployment, networking/security, observability, load testing, failure/recovery, readiness, incident simulation, acceptance lab.
