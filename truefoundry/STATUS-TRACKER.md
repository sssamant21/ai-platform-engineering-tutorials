# TrueFoundry Tutorial — Status Tracker

This tracker records the stage-gated review and publication status of the TrueFoundry production tutorial track.

## Review Workflow

Each tutorial follows:

**Draft → Technical/Vendor Validation → Production/SRE Review → Revised Final → Canonical → Hands-On Lab → Repository Validation**

---

## Part 1 — Foundations & Architecture

**Status:** ✅ Complete — 6/6

| # | Tutorial | Draft | Technical/Vendor Validation | Production/SRE Review | Revised Final | Canonical | Hands-On Lab | Repository Validation |
|---|---|---|---|---|---|---|---|---|
| 1.1 | What Is TrueFoundry? | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 1.2 | TrueFoundry Architecture — Control Plane, Compute Plane & `tfy-agent` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 1.3 | TrueFoundry Platform Components & Terminology | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 1.4 | TrueFoundry and Kubernetes — How They Work Together | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 1.5 | Workspaces, Environments & Deployment Concepts | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 1.6 | SRE Ownership, Failure Domains & Support Model | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

### Part 1 Completion

Part 1 establishes the production foundation for the TrueFoundry track:

- TrueFoundry platform purpose and operating model
- Control Plane and Compute Plane architecture
- `tfy-agent` and cluster-integration boundaries
- platform components and terminology
- TrueFoundry-to-Kubernetes relationships
- Workspaces, environments, deployment identity, and promotion concepts
- management path vs runtime path
- failure-domain isolation
- minimum supported blast radius
- Lowest Proven Healthy Layer
- First Failed Transition
- evidence confidence
- Current Actionable Owner
- evidence-based incident handoff and escalation
- authoritative remediation and end-to-end recovery validation

All Part 1 tutorials have completed:

**Draft → Technical/Vendor Validation → Production/SRE Review → Revised Final → Canonical → Hands-On Lab → Repository Validation**

---

## Part 2 — Kubernetes Integration

**Status:** ✅ Complete — 7/7

| # | Tutorial | Draft | Technical/Vendor Validation | Production/SRE Review | Revised Final | Canonical | Hands-On Lab | Repository Validation |
|---|---|---|---|---|---|---|---|---|
| 2.1 | Connecting a Kubernetes Cluster to TrueFoundry | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 2.2 | Understanding `tfy-agent` Deployment | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 2.3 | Namespaces & Workload Placement | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 2.4 | ServiceAccounts, RBAC & Permissions | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 2.5 | ConfigMaps, Secrets & Runtime Configuration | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 2.6 | Services, Ingress, DNS & TLS | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 2.7 | Storage, PVCs & Persistent Workloads | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

### Part 2 Completion

Part 2 establishes the production Kubernetes-integration operating model for TrueFoundry:

- Kubernetes cluster connection and integration validation
- `tfy-agent` deployment, health, authorization, and reconciliation
- namespaces and workload placement
- ServiceAccounts, RBAC, permissions, and cloud-IAM boundaries
- ConfigMaps, Secrets, runtime configuration, and desired-state ownership
- Services, Ingress, DNS, TLS, endpoints, and request-path validation
- StorageClasses, PVCs, PVs, CSI, topology, mounts, capacity, performance, and data-risk controls
- management path vs runtime path separation
- authoritative configuration ownership
- Lowest Proven Healthy Layer
- First Failed Transition
- Minimum Supported Blast Radius
- evidence confidence and Current Actionable Owner
- production-safe evidence collection and incident handoff

All Part 2 tutorials and `[SAFE-READ]` production validation labs have completed:

**Draft → Technical/Vendor Validation → Production/SRE Review → Revised Final → Canonical → Hands-On Lab → Repository Validation**

Part 2 canonical publication commits:

- 2.1 — `82b171b` — Kubernetes cluster integration tutorial and lab
- 2.2 — `0cca4d1` — `tfy-agent` deployment tutorial and lab
- 2.3 — `19bd7ff` — namespaces and workload placement tutorial and lab
- 2.4 — `fe35e45` — ServiceAccounts, RBAC, and permissions tutorial and lab
- 2.5 — `20bf51e` — ConfigMaps, Secrets, and runtime configuration tutorial and lab
- 2.6 — `bd57723` — Services, Ingress, DNS, and TLS tutorial and lab
- 2.7 — `e5f7370` — storage, PVCs, and persistent workloads tutorial and lab

---

## Part 3 — Compute & Scheduling

**Status:** ✅ Complete — 6/6

| # | Tutorial | Draft | Technical/Vendor Validation | Production/SRE Review | Revised Final | Canonical | Hands-On Lab | Repository Validation |
|---|---|---|---|---|---|---|---|---|
| 3.1 | CPU & Memory Resource Management | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 3.2 | Requests, Limits & Kubernetes QoS | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 3.3 | Node Selectors & Node Affinity | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 3.4 | Taints & Tolerations | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 3.5 | Dedicated Node Pools | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 3.6 | Workload Scheduling Troubleshooting | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

### Part 3 Completion

Part 3 establishes the production compute, placement, and scheduling operating model for TrueFoundry workloads:

- CPU and memory requests, limits, allocatable capacity, and runtime usage
- Kubernetes QoS and resource-accounting behavior
- node selectors and required/preferred node affinity
- taints, tolerations, and scheduling compatibility
- dedicated node pools and workload isolation patterns
- candidate-node reduction and resource-fit analysis
- topology, storage, and scheduling constraints
- dedicated-capacity and node-pool consistency validation
- scheduler evidence and exclusion-ledger analysis
- autoscaler, provisioning, registration, and Node readiness boundaries
- time-to-capacity investigation
- Lowest Proven Healthy Layer
- First Failed Transition
- Minimum Supported Blast Radius
- evidence confidence and Current Actionable Owner
- authoritative desired-state ownership
- production-safe evidence preservation and incident handoff
- service-restoration vs root-cause-remediation validation

All Part 3 tutorials and `[SAFE-READ]` production validation labs have completed:

**Draft → Technical/Vendor Validation → Production/SRE Review → Revised Final → Canonical → Hands-On Lab → Repository Validation**

Part 3 canonical publication commits:

- 3.1 — `1a03b8c` — CPU and memory resource management tutorial and lab
- 3.2 — `b6af212` — requests, limits, and Kubernetes QoS tutorial and lab
- 3.3 — `9fb75e0` — node selectors and node affinity tutorial and lab
- 3.4 — `9ef75cb` — taints and tolerations tutorial and lab
- 3.5 — `23d2d37` — dedicated node pools tutorial and lab
- 3.6 — `5a350e3` — workload scheduling troubleshooting tutorial and lab

## Part 4 — GPU Infrastructure

**Status:** ✅ Publication Complete — 7/7 published

| # | Tutorial | Draft | Technical/Vendor Validation | Production/SRE Review | Revised Final | Canonical | Hands-On Lab | Repository Validation |
|---|---|---|---|---|---|---|---|
| 4.1 | GPU Architecture and NVIDIA GPU Fundamentals | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 4.2 | GPU Nodes and Kubernetes GPU Scheduling | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 4.3 | NVIDIA Drivers, Device Plugin and GPU Operator | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 4.4 | CUDA, Container Runtime and Compatibility | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 4.5 | Dedicated GPU Node Pools and Resource Isolation | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 4.6 | GPU Validation, Monitoring and Troubleshooting | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 4.7 | Production GPU Infrastructure Recovery Lab | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |

Chapters 4.1–4.7 canonical tutorials and labs published; formal vendor and SRE reviews remain to be completed. See [Part 4 layout](part-04-gpu-infrastructure/PART-04-LAYOUT.md).

## Part 5 — Application Deployment

**Status:** ✅ Publication Complete — 7/7 published

| # | Tutorial | Draft | Technical/Vendor Validation | Production/SRE Review | Revised Final | Canonical | Hands-On Lab | Repository Validation |
|---|---|---|---|---|---|---|---|---|
| 5.1 | Application Deployment Architecture and First Service | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 5.2 | Container Images, Registries and Supply Chain | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 5.3 | Application Configuration, Environment Variables and Secrets | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 5.4 | Resource Requests, Limits, Probes and Workload Reliability | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 5.5 | Services, Endpoints, Routing and Application Connectivity | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 5.6 | Deployment Revisions, Rollouts, Rollbacks and Troubleshooting | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 5.7 | Production Application Deployment and Recovery Capstone | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |

Formal technical/vendor and SRE reviews and actual lab execution remain pending. See [Part 5 layout](part-05-application-deployment/PART-05-LAYOUT.md).

## Part 6 — Model Serving

**Status:** ✅ Publication Complete — 7/7 published

| # | Tutorial | Draft | Technical/Vendor Validation | Production/SRE Review | Revised Final | Canonical | Hands-On Lab | Repository Validation |
|---|---|---|---|---|---|---|---|---|
| 6.1 | Model-Serving Fundamentals and Architecture | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 6.2 | Model Artifacts, Registries and Storage Integration | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 6.3 | CPU and GPU Inference Runtimes and Scheduling | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 6.4 | Inference Endpoints, Request Lifecycle and API Reliability | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 6.5 | Model Loading, Warmup, Health Probes and Readiness | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 6.6 | Model-Serving Observability, Performance and Troubleshooting | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 6.7 | Production Model-Serving Deployment and Recovery Capstone | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |

See [Part 6 layout](part-06-model-serving/PART-06-LAYOUT.md). Formal technical/vendor validation, SRE review and live lab execution remain pending.

## Part 7 — vLLM & LLM Serving

**Status:** ✅ Publication Complete — 7/7 published

| # | Tutorial | Draft | Technical/Vendor Validation | Production/SRE Review | Revised Final | Canonical | Hands-On Lab | Repository Validation |
|---|---|---|---|---|---|---|---|---|
| 7.1 | vLLM Fundamentals and Serving Architecture | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 7.2 | vLLM Deployment with TrueFoundry and Kubernetes | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 7.3 | GPU CUDA PyTorch and Device Detection | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 7.4 | Model Loading Tokenizers and Tensor Parallelism | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 7.5 | PagedAttention KV Cache and Continuous Batching | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 7.6 | vLLM Performance Tuning Observability and Troubleshooting | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 7.7 | Production vLLM Deployment and Recovery Capstone | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |

See [Part 7 layout](part-07-vllm-llm-serving/PART-07-LAYOUT.md). Formal reviews and live lab execution remain pending.

## Part 8 — Scaling & Performance

**Status:** 🟡 Publication Complete — 7/7 published

| # | Tutorial | Draft | Technical/Vendor Validation | Production/SRE Review | Revised Final | Canonical | Hands-On Lab | Repository Validation |
|---|---|---|---|---|---|---|---|---|
| 8.1 | Scaling Architecture and GPU-Aware Capacity Planning | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 8.2 | Horizontal Pod Autoscaling for Inference Workloads | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 8.3 | KEDA Event-Driven Scaling and Custom Metrics | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 8.4 | GPU Node Autoscaling and Cluster Capacity | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 8.5 | Inference Concurrency, Latency and Throughput Optimization | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 8.6 | Scaling Observability, Cost and Troubleshooting | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 8.7 | Production Scaling and Recovery Capstone | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |

See [Part 8 layout](part-08-scaling-performance/PART-08-LAYOUT.md). Formal reviews and live lab execution remain pending.

## Part 9 — Networking

**Status:** ✅ Publication Complete — 7/7 published

| # | Tutorial | Draft | Technical/Vendor Validation | Production/SRE Review | Revised Final | Canonical | Hands-On Lab | Repository Validation |
|---|---|---|---|---|---|---|---|---|
| 9.1 | TrueFoundry Networking Architecture and End-to-End Request Path | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 9.2 | Kubernetes Services, EndpointSlices and Service Discovery | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 9.3 | Ingress, Gateway API and Cloud Load Balancers | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 9.4 | DNS, TLS, Certificates and Secure Endpoint Exposure | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 9.5 | NetworkPolicy, Private Networking and Egress Security | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 9.6 | Inference Streaming, Timeouts and Networking Troubleshooting | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 9.7 | Production Networking and Recovery Capstone | ✅ | ⬜ Pending formal review | ⬜ Pending formal review | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |

See [Part 9 layout](part-09-networking/PART-09-LAYOUT.md). Formal reviews and live lab execution remain pending.

## Part 10 — Security

**Status:** 🟡 In Progress — 4/7 published

| Chapter | Topic | Draft | Vendor review | SRE/security review | Live lab | Canonical | Lab | Verified |
|---|---|---|---|---|---|---|---|---|
| 10.1 | Security Architecture, Trust Boundaries and Threat Modeling | ✅ | ⬜ Pending | ⬜ Pending | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 10.2 | Identity, SSO, Authentication and Authorization | ✅ | ⬜ Pending | ⬜ Pending | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 10.3 | Kubernetes RBAC, ServiceAccounts and Workload Identity | ✅ | ⬜ Pending | ⬜ Pending | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 10.4 | Secrets Management, Encryption and Credential Rotation | ✅ | ⬜ Pending | ⬜ Pending | ⬜ Pending | ✅ Published | ✅ Published | ✅ Verified |
| 10.5 | Cloud IAM, Supply Chain and Runtime Security | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 10.6 | Security Observability, Audit and Incident Response | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 10.7 | Production Security Hardening and Recovery Capstone | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |

See [Part 10 layout](part-10-security/PART-10-LAYOUT.md).

## Part 11 — Observability

**Status:** ⬜ Not Started

## Part 12 — Production Operations

**Status:** ⬜ Not Started

## Part 13 — Troubleshooting

**Status:** ⬜ Not Started

## Part 14 — SRE Runbooks

**Status:** ⬜ Not Started

## Part 15 — End-to-End Production Project

**Status:** ⬜ Not Started

---

## Overall Track Status

| Part | Area | Status |
|---|---|---|
| 1 | Foundations & Architecture | ✅ Complete — 6/6 |
| 2 | Kubernetes Integration | ✅ Complete — 7/7 |
| 3 | Compute & Scheduling | ✅ Complete — 6/6 |
| 4 | GPU Infrastructure | ✅ Publication Complete — 7/7 published |
| 5 | Application Deployment | ✅ Publication Complete — 7/7 published |
| 6 | Model Serving | ✅ Publication Complete — 7/7 published |
| 7 | vLLM & LLM Serving | ✅ Publication Complete — 7/7 published |
| 8 | Scaling & Performance | 🟡 Publication Complete — 7/7 published |
| 9 | Networking | 🟡 In Progress — 2/7 published |
| 10 | Security | ⬜ Not Started |
| 11 | Observability | ⬜ Not Started |
| 12 | Production Operations | ⬜ Not Started |
| 13 | Troubleshooting | ⬜ Not Started |
| 14 | SRE Runbooks | ⬜ Not Started |
| 15 | End-to-End Production Project | ⬜ Not Started |

---

**Current milestone:** Part 09 publication complete (7/7); Part 10 — Security Chapters 10.1–10.4 and labs published (4/7), Chapter 10.5 next. Formal reviews and live lab execution remain pending. Part 08 publication complete. Part 07 publication complete. Formal vendor/SRE reviews and live lab execution remain separate pending milestones.

### Chapter 9.3 — Ingress, Gateway API and Cloud Load Balancers
- Canonical: [Published](part-09-networking/03-ingress-gateway-api-cloud-load-balancers/03-ingress-gateway-api-cloud-load-balancers.md)
- Lab: [Published](part-09-networking/labs/03-ingress-gateway-api-cloud-load-balancers-check.md)
- Repository validation: pending final link verification; vendor/SRE review and live execution pending.
- Part 09 publication: 7/7 chapters and labs; curriculum: 61 chapters published.

### Chapter 9.4 — DNS, TLS, Certificates and Secure Endpoint Exposure
- Canonical: [Published](part-09-networking/04-dns-tls-certificates-secure-endpoint-exposure/04-dns-tls-certificates-secure-endpoint-exposure.md)
- Lab: [Published](part-09-networking/labs/04-dns-tls-certificates-secure-endpoint-exposure-check.md)
- Publication verified; formal reviews and live lab execution pending.

### Chapter 9.5 — NetworkPolicy, Private Networking and Egress Security
- Canonical: [Published](part-09-networking/05-networkpolicy-private-networking-egress-security/05-networkpolicy-private-networking-egress-security.md)
- Lab: [Published](part-09-networking/labs/05-networkpolicy-private-networking-egress-security-check.md)
- Part 09: 5/7 chapters and labs published; curriculum: 59 chapters. Formal reviews and live lab execution pending.

### Chapter 9.6 — Inference Streaming, Timeouts and Networking Troubleshooting
- Canonical: [Published](part-09-networking/06-inference-streaming-timeouts-networking-troubleshooting/06-inference-streaming-timeouts-networking-troubleshooting.md)
- Lab: [Published](part-09-networking/labs/06-inference-streaming-timeouts-networking-troubleshooting-check.md)
- Part 09: 7/7 chapters and labs; curriculum: 61 chapters. Formal reviews and live lab execution pending.

### Chapter 9.7 — Production Networking and Recovery Capstone
- Canonical: [Published](part-09-networking/07-production-networking-recovery-capstone/07-production-networking-recovery-capstone.md)
- Lab: [Published](part-09-networking/labs/07-production-networking-recovery-capstone-check.md)
- Part 09 publication complete: 7/7 chapters and labs; curriculum: 61 chapters published. Formal vendor/SRE reviews and live lab execution remain pending.

### Chapter 10.1 — Security Architecture, Trust Boundaries and Threat Modeling
- Canonical: [Published](part-10-security/01-security-architecture-trust-boundaries-threat-modeling/01-security-architecture-trust-boundaries-threat-modeling.md)
- Lab: [Published](part-10-security/labs/01-security-architecture-trust-boundaries-threat-modeling-check.md)
- Part 10: 4/7 chapters and labs; curriculum: 65 chapters published. Formal reviews and live execution pending.

### Chapter 10.2 — Identity, SSO, Authentication and Authorization
- Canonical: [Published](part-10-security/02-identity-sso-authentication-authorization/02-identity-sso-authentication-authorization.md)
- Lab: [Published](part-10-security/labs/02-identity-sso-authentication-authorization-check.md)
- Part 10: 4/7 chapters and labs; curriculum: 65 chapters. Formal reviews and live execution pending.

### Chapter 10.3 — Kubernetes RBAC, ServiceAccounts and Workload Identity
- Canonical: [Published](part-10-security/03-kubernetes-rbac-serviceaccounts-workload-identity/03-kubernetes-rbac-serviceaccounts-workload-identity.md)
- Lab: [Published](part-10-security/labs/03-kubernetes-rbac-serviceaccounts-workload-identity-check.md)
- Part 10: 4/7 chapters and labs; curriculum: 65 chapters. Formal reviews and live execution pending.

### Chapter 10.4 — Secrets Management, Encryption and Credential Rotation
- Canonical: [Published](part-10-security/04-secrets-management-encryption-credential-rotation/04-secrets-management-encryption-credential-rotation.md)
- Lab: [Published](part-10-security/labs/04-secrets-management-encryption-credential-rotation-check.md)
- Part 10: 4/7 chapters and labs; curriculum: 65 chapters. Formal reviews and live execution pending.
