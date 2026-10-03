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

**Status:** ⬜ Not Started

## Part 5 — Application Deployment

**Status:** ⬜ Not Started

## Part 6 — Model Serving

**Status:** ⬜ Not Started

## Part 7 — vLLM & LLM Serving

**Status:** ⬜ Not Started

## Part 8 — Scaling & Performance

**Status:** ⬜ Not Started

## Part 9 — Networking

**Status:** ⬜ Not Started

## Part 10 — Security

**Status:** ⬜ Not Started

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
| 4 | GPU Infrastructure | ⬜ Not Started |
| 5 | Application Deployment | ⬜ Not Started |
| 6 | Model Serving | ⬜ Not Started |
| 7 | vLLM & LLM Serving | ⬜ Not Started |
| 8 | Scaling & Performance | ⬜ Not Started |
| 9 | Networking | ⬜ Not Started |
| 10 | Security | ⬜ Not Started |
| 11 | Observability | ⬜ Not Started |
| 12 | Production Operations | ⬜ Not Started |
| 13 | Troubleshooting | ⬜ Not Started |
| 14 | SRE Runbooks | ⬜ Not Started |
| 15 | End-to-End Production Project | ⬜ Not Started |

---

**Current milestone:** Part 3 — Compute & Scheduling complete. Part 4 — GPU Infrastructure is next.
