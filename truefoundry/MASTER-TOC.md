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
- [8.1 — Scaling Architecture and GPU-Aware Capacity Planning](part-08-scaling-performance/01-scaling-architecture-capacity-planning/01-scaling-architecture-capacity-planning.md) — [Hands-on Lab](part-08-scaling-performance/labs/01-scaling-architecture-capacity-planning-check.md) (published)
- [8.2 — Horizontal Pod Autoscaling for Inference Workloads](part-08-scaling-performance/02-horizontal-pod-autoscaling-inference-workloads/02-horizontal-pod-autoscaling-inference-workloads.md) — [Hands-on Lab](part-08-scaling-performance/labs/02-horizontal-pod-autoscaling-inference-workloads-check.md) (published)
- [8.3 — KEDA Event-Driven Scaling and Custom Metrics](part-08-scaling-performance/03-keda-event-driven-scaling-custom-metrics/03-keda-event-driven-scaling-custom-metrics.md) — [Hands-on Lab](part-08-scaling-performance/labs/03-keda-event-driven-scaling-custom-metrics-check.md) (published)
- [8.4 — GPU Node Autoscaling and Cluster Capacity](part-08-scaling-performance/04-gpu-node-autoscaling-cluster-capacity/04-gpu-node-autoscaling-cluster-capacity.md) — [Hands-on Lab](part-08-scaling-performance/labs/04-gpu-node-autoscaling-cluster-capacity-check.md) (published)
- [8.5 — Inference Concurrency, Latency and Throughput Optimization](part-08-scaling-performance/05-inference-concurrency-latency-throughput-optimization/05-inference-concurrency-latency-throughput-optimization.md) — [Hands-on Lab](part-08-scaling-performance/labs/05-inference-concurrency-latency-throughput-optimization-check.md) (published)
- [8.6 — Scaling Observability, Cost and Troubleshooting](part-08-scaling-performance/06-scaling-observability-cost-troubleshooting/06-scaling-observability-cost-troubleshooting.md) — [Hands-on Lab](part-08-scaling-performance/labs/06-scaling-observability-cost-troubleshooting-check.md) (published)
- [8.7 — Production Scaling and Recovery Capstone](part-08-scaling-performance/07-production-scaling-recovery-capstone/07-production-scaling-recovery-capstone.md) — [Hands-on Lab](part-08-scaling-performance/labs/07-production-scaling-recovery-capstone-check.md) (published)
- [Part 8 master layout](part-08-scaling-performance/PART-08-LAYOUT.md)

## Part 9 — Networking

- [9.1 — TrueFoundry Networking Architecture and End-to-End Request Path](part-09-networking/01-truefoundry-networking-architecture-request-path/01-truefoundry-networking-architecture-request-path.md) — [Hands-on Lab](part-09-networking/labs/01-truefoundry-networking-architecture-request-path-check.md) (published)
- [9.2 — Kubernetes Services, EndpointSlices and Service Discovery](part-09-networking/02-kubernetes-services-endpointslices-service-discovery/02-kubernetes-services-endpointslices-service-discovery.md) — [Hands-on Lab](part-09-networking/labs/02-kubernetes-services-endpointslices-service-discovery-check.md) (published)
- [9.3 — Ingress, Gateway API and Cloud Load Balancers](part-09-networking/03-ingress-gateway-api-cloud-load-balancers/03-ingress-gateway-api-cloud-load-balancers.md) — [Hands-on Lab](part-09-networking/labs/03-ingress-gateway-api-cloud-load-balancers-check.md) (published)
- [9.4 — DNS, TLS, Certificates and Secure Endpoint Exposure](part-09-networking/04-dns-tls-certificates-secure-endpoint-exposure/04-dns-tls-certificates-secure-endpoint-exposure.md) — [Hands-on Lab](part-09-networking/labs/04-dns-tls-certificates-secure-endpoint-exposure-check.md) (published)
- [9.5 — NetworkPolicy, Private Networking and Egress Security](part-09-networking/05-networkpolicy-private-networking-egress-security/05-networkpolicy-private-networking-egress-security.md) — [Hands-on Lab](part-09-networking/labs/05-networkpolicy-private-networking-egress-security-check.md) (published)
- [9.6 — Inference Streaming, Timeouts and Networking Troubleshooting](part-09-networking/06-inference-streaming-timeouts-networking-troubleshooting/06-inference-streaming-timeouts-networking-troubleshooting.md) — [Hands-on Lab](part-09-networking/labs/06-inference-streaming-timeouts-networking-troubleshooting-check.md) (published)
- [9.7 — Production Networking and Recovery Capstone](part-09-networking/07-production-networking-recovery-capstone/07-production-networking-recovery-capstone.md) — [Hands-on Lab](part-09-networking/labs/07-production-networking-recovery-capstone-check.md) (published)
- [Part 9 master layout](part-09-networking/PART-09-LAYOUT.md)

## Part 10 — Security
- [10.1 — Security Architecture, Trust Boundaries and Threat Modeling](part-10-security/01-security-architecture-trust-boundaries-threat-modeling/01-security-architecture-trust-boundaries-threat-modeling.md) — [Hands-on Lab](part-10-security/labs/01-security-architecture-trust-boundaries-threat-modeling-check.md) (published)
- [10.2 — Identity, SSO, Authentication and Authorization](part-10-security/02-identity-sso-authentication-authorization/02-identity-sso-authentication-authorization.md) — [Hands-on Lab](part-10-security/labs/02-identity-sso-authentication-authorization-check.md) (published)
- [10.3 — Kubernetes RBAC, ServiceAccounts and Workload Identity](part-10-security/03-kubernetes-rbac-serviceaccounts-workload-identity/03-kubernetes-rbac-serviceaccounts-workload-identity.md) — [Hands-on Lab](part-10-security/labs/03-kubernetes-rbac-serviceaccounts-workload-identity-check.md) (published)
- [10.4 — Secrets Management, Encryption and Credential Rotation](part-10-security/04-secrets-management-encryption-credential-rotation/04-secrets-management-encryption-credential-rotation.md) — [Hands-on Lab](part-10-security/labs/04-secrets-management-encryption-credential-rotation-check.md) (published)
- [10.5 — Cloud IAM, Supply Chain and Runtime Security](part-10-security/05-cloud-iam-supply-chain-runtime-security/05-cloud-iam-supply-chain-runtime-security.md) — [Hands-on Lab](part-10-security/labs/05-cloud-iam-supply-chain-runtime-security-check.md) (published)
- [10.6 — Security Observability, Audit and Incident Response](part-10-security/06-security-observability-audit-incident-response/06-security-observability-audit-incident-response.md) — [Hands-on Lab](part-10-security/labs/06-security-observability-audit-incident-response-check.md) (published)
- [10.7 — Production Security Hardening and Recovery Capstone](part-10-security/07-production-security-hardening-recovery-capstone/07-production-security-hardening-recovery-capstone.md) — [Hands-on Lab](part-10-security/labs/07-production-security-hardening-recovery-capstone-check.md) (published)
- [Part 10 master layout](part-10-security/PART-10-LAYOUT.md)

## Part 11 — Observability

- [11.1 — Observability Architecture and Telemetry Foundations](part-11-observability/01-observability-architecture-telemetry-foundations/01-observability-architecture-telemetry-foundations.md) — [Hands-on Lab](part-11-observability/labs/01-observability-architecture-telemetry-foundations-check.md) (published)
- [11.2 — Logging Architecture, Structured Logs and Kubernetes Events](part-11-observability/02-logging-architecture-structured-logs-kubernetes-events/02-logging-architecture-structured-logs-kubernetes-events.md) — [Hands-on Lab](part-11-observability/labs/02-logging-architecture-structured-logs-kubernetes-events-check.md) (published)
- [11.3 — Prometheus Metrics, Exporters and Recording Rules](part-11-observability/03-prometheus-metrics-exporters-recording-rules/03-prometheus-metrics-exporters-recording-rules.md) — [Hands-on Lab](part-11-observability/labs/03-prometheus-metrics-exporters-recording-rules-check.md) (published)
- [11.4 — OpenTelemetry Distributed Tracing and Correlation](part-11-observability/04-opentelemetry-distributed-tracing-correlation/04-opentelemetry-distributed-tracing-correlation.md) — [Hands-on Lab](part-11-observability/labs/04-opentelemetry-distributed-tracing-correlation-check.md) (published)
- [11.5 — GPU, vLLM and Inference Performance Observability](part-11-observability/05-gpu-vllm-inference-performance-observability/05-gpu-vllm-inference-performance-observability.md) — [Hands-on Lab](part-11-observability/labs/05-gpu-vllm-inference-performance-observability-check.md) (published)
- [11.6 — Dashboards, SLIs, SLOs, Alerting and Error Budgets](part-11-observability/06-dashboards-slis-slos-alerting-error-budgets/06-dashboards-slis-slos-alerting-error-budgets.md) — [Hands-on Lab](part-11-observability/labs/06-dashboards-slis-slos-alerting-error-budgets-check.md) (published)
- [11.7 — Enterprise Observability and Incident Detection Capstone](part-11-observability/07-enterprise-observability-incident-detection-capstone/07-enterprise-observability-incident-detection-capstone.md) — [Hands-on Lab](part-11-observability/labs/07-enterprise-observability-incident-detection-capstone-check.md) (published)
- [Part 11 master layout](part-11-observability/PART-11-LAYOUT.md)

## Part 12 — Production Operations
- [12.1 — Production Operations Architecture and Readiness](part-12-production-operations/01-production-operations-architecture-readiness/01-production-operations-architecture-readiness.md) — [Hands-on Lab](part-12-production-operations/labs/01-production-operations-architecture-readiness-check.md) (published)
- [12.2 — GPU Capacity Planning, Quotas and Production Scaling](part-12-production-operations/02-gpu-capacity-planning-quotas-production-scaling/02-gpu-capacity-planning-quotas-production-scaling.md) — [Hands-on Lab](part-12-production-operations/labs/02-gpu-capacity-planning-quotas-production-scaling-check.md) (published)
- [12.3 — High Availability, Resilience and Failure Domains](part-12-production-operations/03-high-availability-resilience-failure-domains/03-high-availability-resilience-failure-domains.md) — [Hands-on Lab](part-12-production-operations/labs/03-high-availability-resilience-failure-domains-check.md) (published)
- [12.4 — Production Releases, Upgrades and Rollback](part-12-production-operations/04-production-releases-upgrades-rollback/04-production-releases-upgrades-rollback.md) — [Hands-on Lab](part-12-production-operations/labs/04-production-releases-upgrades-rollback-check.md) (published)
- [12.5 — FinOps, GPU Cost Governance and Rightsizing](part-12-production-operations/05-finops-gpu-cost-governance-rightsizing/05-finops-gpu-cost-governance-rightsizing.md) — [Hands-on Lab](part-12-production-operations/labs/05-finops-gpu-cost-governance-rightsizing-check.md) (published)
- 12.6 — Operational Runbooks, Maintenance and Incident Command (planned)
- 12.7 — Production Readiness and Recovery Capstone (planned)
- [Part 12 master layout](part-12-production-operations/PART-12-LAYOUT.md)

## Part 13 — Troubleshooting
Methodology, `tfy-agent`, deployment, Pending, CrashLoopBackOff, ImagePullBackOff, RBAC, networking, GPU, NVIDIA/CUDA, vLLM, inference performance, GPU OOM, incidents.

## Part 14 — SRE Runbooks
Platform health, `tfy-agent`, GPU health, model deployment, vLLM health, incidents, post-incident validation.

## Part 15 — End-to-End Production Project
Architecture, Kubernetes, GPU, TrueFoundry, vLLM deployment, networking/security, observability, load testing, failure/recovery, readiness, incident simulation, acceptance lab.
