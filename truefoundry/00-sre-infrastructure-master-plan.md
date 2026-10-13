# TrueFoundry — SRE and AI Infrastructure Engineering Master Plan

## Audience and boundaries
SREs, Kubernetes administrators, cloud engineers, GPU infrastructure operators and AI platform engineers. Prioritize platform lifecycle, compute, GPU scheduling, model-serving reliability, networking, security, troubleshooting, incident response and FinOps. Model development is outside scope.

## Standard chapter format
Objective; architecture; prerequisites; commands/manifests; reproducible lab; failure simulation; monitoring and triage; remediation/rollback; acceptance checks; production runbook.

## Learning path (60 chapters)
### Part 1 — Foundations
001 TrueFoundry for SREs
002 Control plane, agent and Kubernetes architecture
003 Workspaces, environments and resource boundaries
004 Platform components and dependency mapping
005 Architecture inspection lab

### Part 2 — Kubernetes integration
006 EKS integration
007 AKS integration
008 GKE integration
009 Namespaces, service accounts and workload isolation
010 Kubernetes integration acceptance lab

### Part 3 — Compute and scheduling
011 CPU workload scheduling and resource limits
012 Node pools, taints, tolerations and affinity
013 Autoscaling, disruption and capacity reservations
014 Quotas, priorities and workload placement
015 Unschedulable workload recovery lab

### Part 4 — GPU infrastructure
016 GPU node provisioning and drivers
017 NVIDIA device plugin and GPU discovery
018 CUDA/runtime/container compatibility
019 GPU sharing, topology and resource isolation
020 No-CUDA-runtime incident lab

### Part 5 — Networking and ingress
021 Service connectivity and ingress
022 DNS, TLS and certificate operations
023 Private connectivity and egress controls
024 Load balancing and inference traffic routing
025 Network outage diagnosis lab

### Part 6 — Identity and security
026 Platform RBAC and least privilege
027 Cloud workload identity and service accounts
028 Secrets and registry access
029 Network policies and supply-chain controls
030 Authorization and credential failure lab

### Part 7 — Deployment operations
031 Deployment lifecycle and health checks
032 Revisions, rollouts and rollback
033 Environment configuration and drift
034 Deployment automation and GitOps integration
035 Failed deployment recovery lab

### Part 8 — Model-serving infrastructure
036 Inference serving architecture
037 vLLM runtime deployment operations
038 GPU memory, batching and concurrency
039 Model artifact loading and storage
040 Inference startup failure lab

### Part 9 — Observability and SLOs
041 Platform, Kubernetes and agent metrics
042 GPU utilization, memory and saturation
043 Request latency, errors and throughput
044 Logs, traces, dashboards and alerts
045 SLO and alert acceptance lab

### Part 10 — Performance and capacity
046 Inference load testing
047 Autoscaling behavior and cold starts
048 Capacity planning and GPU sizing
049 Cost-per-request and GPU FinOps
050 Performance regression project

### Part 11 — Reliability and disaster recovery
051 Failure domains and high availability
052 Backup of configuration and dependencies
053 Cluster and region recovery strategy
054 Incident response and operational change management
055 Multi-component outage simulation lab

### Part 12 — Final production capstone
056 Production readiness checklist
057 Security and resilience review
058 GPU exhaustion and service degradation drill
059 Upgrade, rollback and post-change validation
060 Final TrueFoundry SRE operational assessment

## Lab rules
Use neutral demo-app and sample-model naming only. Separate EKS, AKS and GKE platform specifics. Assume dev/test/staging/prod may be namespace-isolated within one cluster. Run destructive tests only in disposable environments. Include cleanup, evidence and acceptance criteria.
