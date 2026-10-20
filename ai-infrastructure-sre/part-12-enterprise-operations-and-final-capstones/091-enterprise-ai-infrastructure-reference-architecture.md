# Chapter 091 — Enterprise AI Infrastructure Reference Architecture

**Part 12 — Enterprise Operations and Final Capstones | AI Infrastructure Engineering & SRE**

**Publication:** PUBLISHED. Architecture review, lab execution, security verification, deployment and production acceptance **PENDING**.

## 1. Mission and scope

Design an enterprise-ready AI infrastructure platform spanning GPU compute, Kubernetes, distributed training, inference, data access, model lifecycle, networking, security, observability, SLOs, FinOps, multi-zone reliability and disaster recovery. This chapter is a **reference architecture and acceptance specification**, not a claim that any cluster has been built or validated.

**Prerequisites:** Chapters 001–090, particularly GPU infrastructure, networking, scheduling, security, training, serving, observability, capacity, FinOps and DR.

### Learning outcomes

1. Define business and technical requirements and threat model.
2. Produce logical, physical, network and data-flow architecture views.
3. Separate trust boundaries, tenant roles, secrets, control planes and GPU workloads.
4. Design compatible GPU pools for training and latency-sensitive serving.
5. Specify HA, DR, operational SLOs and evidence-backed acceptance.
6. Create reproducible GitOps/IaC modules and an actionable phased rollout plan.

## 2. Architecture principles

- **Security by default:** zero-trust service identity, least privilege, signed artifacts, isolation and encryption.
- **Workload isolation:** separate serving, training, research and platform system pools and admission policies.
- **SLO-driven capacity:** GPU quota, physical supply, reservations and scheduler placement must all be checked.
- **Failure-domain awareness:** AZ/rack, GPU node, NIC/fabric, storage, control plane and region boundaries.
- **Declarative operations:** versioned IaC/GitOps with policy checks, change approvals and rollback.
- **Measured performance:** benchmark throughput, TTFT/TPOT, p99, useful training steps and recovery time.
- **Auditable economics:** reconcile cloud invoice, GPU-hours, shared cost and cost per successful unit.
- **Evidence over diagrams:** architecture approval requires tests, ownership and independent review.

## 3. Logical reference architecture

\`\`\`text
                   Enterprise users / CI / MLOps
                               |
                Identity, SSO, RBAC, policy, audit
                               |
                     GitOps / IaC control plane
                               |
         ┌─────────────────────┴──────────────────────┐
         |                                            |
  Data + model supply chain                    AI workload plane
  ├─ dataset catalog / ACL                     ├─ Inference ingress / auth
  ├─ object storage / lineage                  ├─ Model gateway / routing
  ├─ model registry / signing                  ├─ GPU serving deployments
  ├─ image registry / SBOM                     ├─ Training jobs / gang scheduler
  └─ secrets / KMS                             └─ Checkpoint / artifact writers
         |                                            |
         └─────────────────────┬──────────────────────┘
                               |
                     Kubernetes platform
          ┌────────────────────┼────────────────────┐
          |                    |                    |
     GPU serving pool     GPU training pool     System/CPU pool
     multi-zone, warm     topology-aware        DNS, metrics, controllers
     p99 + HA             RDMA/NCCL, queues     hardened, autoscaled
          └────────────────────┼────────────────────┘
                               |
               Cloud/on-prem GPU + NIC + storage
                               |
     Telemetry / SLOs / incident response / FinOps / DR
\`\`\`

**Important:** the diagram is conceptual. Actual cloud services, GPU models, network interfaces, identity federation, data paths and control-plane topology are design decisions requiring validation.

## 4. Physical deployment and failure domains

| Layer | Recommended design questions | Evidence |
|---|---|---|
| Region/AZ | Which AZs have compatible GPU stock? | capacity/reservation register |
| GPU nodes | GPU SKU/HBM/driver, CPU/RAM, PCIe/NVLink, NIC | validated hardware matrix |
| Training fabric | RDMA topology, bandwidth, congestion, placement | NCCL/fabric benchmark |
| Serving | warm model replicas, LB routing, peak p99 | staged traffic test |
| Control plane | etcd/managed K8s HA, API rate limits | control-plane SLO |
| Storage | checkpoint throughput, model distribution, encryption | restore/checksum test |
| Identity | SSO, workload identity, KMS, break-glass | access/deny tests |
| Observability | independent metrics/logs/traces and alerts | incident drill |
| DR | recovery GPU compatibility, RTO/RPO, fencing | full recovery evidence |

Never assume spare GPUs are interchangeable: HBM size, CUDA compatibility, MIG profile and interconnect can make otherwise idle capacity unusable.

## 5. Kubernetes platform architecture

### Cluster boundaries

Choose single versus multiple clusters based on tenancy, blast radius, compliance, control-plane scale, GPU topology and cost. Do not treat namespaces as equivalent to hard security isolation. High-risk tenants may require dedicated clusters/nodes and stronger runtime controls.

### Node pools

- **System pool:** DNS, CNI, monitoring, controllers; separate from scarce GPU nodes.
- **Serving GPU pool:** protected minimum replicas, multi-zone placement, startup/readiness probes, fast routing.
- **Training GPU pool:** topology-aware whole-node allocation, gang scheduling, checkpoint-safe preemption.
- **Research GPU pool:** explicit quotas and borrowing, isolation and cost caps.

Use device plugins/GPU Operator and validated driver/CUDA/container-runtime versions. Restrict privileged GPU management components to approved namespaces/nodes.

### Admission and resource governance

Combine Namespace ResourceQuota, LimitRange, PriorityClass restrictions, Kueue or equivalent queue admission, network policy and pod security admission. Validate CRDs and version-specific APIs before deployment. ResourceQuota does **not** guarantee physical capacity.

## 6. Network and data flow design

### Control plane

SSO/IdP → CI/GitOps → Kubernetes API → admission/controllers → workloads. Authenticate each hop, authorize narrowly and log privileged changes.

### Inference data path

Client → WAF/API gateway → authentication/authorization → model router → inference replicas → approved feature/data access → response. Apply request limits, tenancy, encryption, observability and prompt/model-data privacy controls.

### Training data path

Approved dataset → versioned loader → GPU training job → checkpoint/object store → model registry → evaluation/security gates → promoted serving artifact. Enforce dataset lineage, checksum integrity, secrets isolation and controlled promotion.

### Network segmentation

Deny-by-default east-west policies where CNI supports enforcement; allow only necessary ingress/egress for registry, object storage, metrics, DNS, KMS and approved external endpoints. Validate with **positive and negative connectivity tests**; a manifest alone is not evidence of enforcement.

## 7. Identity, security and supply chain

- SSO/MFA for humans, short-lived workload identity for services.
- Kubernetes RBAC, cloud IAM and tenant-specific policy with break-glass audit.
- Secrets/KMS rotation and cross-region recovery access.
- Signed container images, pinned digests, SBOM, provenance and vulnerability scanning.
- Admission controls for privileged pods, host paths, unapproved registries and priority escalation.
- Data residency, encryption, retention and deletion policy for training data and checkpoints.
- GPU multi-tenancy threat model for MIG/time slicing, memory isolation, side channels and device access.
- Incident logging that avoids exposing prompts, PII or proprietary model artifacts.

**Security signoff:** validate threat model and access-denial cases in staging. Do not label a cluster zero trust solely because it uses RBAC.

## 8. Training architecture

Distributed training requires compatible node bundles, high-bandwidth fabric, gang scheduling, rank/world-size consistency, NCCL health and checkpoint durability. Define:
- job templates, GPU SKU/HBM, topology and queue class;
- reproducible image/model/data versions;
- checkpoint interval, atomic commit marker, optimizer/RNG/dataset state;
- failure restart behavior and RPO/RTO;
- success measured as **cost to target model quality**, not just GPU utilization.

## 9. Inference architecture

Define:
- model-serving runtime, versioned routing and canary strategy;
- request/sequence batching, KV-cache budget and model warmup;
- HPA/KEDA metric adapter and measured SLO-compliant replica throughput;
- per-model TTFT, TPOT, p99, errors, token throughput and cost;
- multi-zone placement, protected headroom and bounded overload behavior;
- rollback for accuracy, safety, latency or memory regression.

## 10. Observability and SRE operations

| Signal | Examples | Owner |
|---|---|---|
| GPU hardware | ECC/Xid, HBM, power, throttling | infrastructure |
| Kubernetes | Pending reasons, node readiness, restarts | platform |
| Training | steps/sec, checkpoint age, NCCL retries | ML training |
| Serving | p99, TTFT, TPOT, error rate, output tokens | inference SRE |
| Security | privileged admission, IAM denial, drift | security |
| FinOps | billed GPU-hours, idle reservation, $/step, $/1M tokens | FinOps |
| DR | restore age, RPO/RTO, replication lag | incident/DR owner |

SLOs need a clear service boundary, measured baseline, error budget and alert route. Establish incident severity, on-call rotation, escalation and post-incident corrective action.

## 11. Capacity, HA and DR

Use Chapter 083 demand forecasts, Chapter 084 rightsizing, Chapter 085 reservations, Chapter 086 N-1/AZ tests, Chapter 087 cost accounting, Chapter 088 scaling/admission and Chapter 089 recovery tests.

### Synthetic sizing illustration

Two AZs × four 8-GPU nodes each = **64 installed GPUs**. One node loss leaves **56**; one AZ loss leaves **32**. If protected inference needs 24 GPUs and non-preemptible training needs 16, the **40-GPU combined workload fails** the 32-GPU AZ-loss aggregate capacity test. Validate physical compatibility, placement and end-user p99 before any HA claim.

### DR contracts

Document per-service RTO/RPO, immutable checkpoint replication, artifact/secret availability, recovery-region GPU capacity, traffic failover and dual-writer fencing. Run isolated restore drills; replication without restore verification is insufficient.

## 12. FinOps and unit economics

Reconcile GPU/CPU/RAM node cost, storage, network, shared services, commitments and idle reservations to a versioned ledger. Track:
- cost per successful training optimizer step and target-quality run;
- cost per million successful inference tokens and SLO-compliant request;
- GPU allocated versus useful GPU-hours;
- realized invoice savings versus capacity freed;
- forecast error, cost anomalies and chargeback disputes.

Budget enforcement must not silently sacrifice protected customer SLOs or training integrity.

## 13. Implementation blueprint

\`\`\`text
platform/
  terraform/
    networking/
    identity/
    kubernetes/
    gpu-nodepools/
    storage/
    observability/
  gitops/
    clusters/
    system/
    gpu-operator/
    policies/
    serving/
    training/
  policies/
    rbac/
    pod-security/
    network/
    quotas/
  tests/
    security/
    performance/
    resilience/
    dr/
  runbooks/
  evidence/
\`\`\`

This is a **proposed repository layout**. It does not imply any files or infrastructure have been created.

### Phased rollout

1. Requirements, service ownership and threat model.
2. Cloud/on-prem GPU stock, network and data-residency feasibility.
3. IaC identity/network/cluster foundations with policy checks.
4. GPU nodes, drivers, device plugins and operator validation.
5. Security admission, workload identity and namespace/cluster isolation.
6. Training queue and checkpoint pipeline.
7. Serving, traffic routing and SLO instrumentation.
8. Observability, FinOps, incident response and change control.
9. Staging performance, security, node/AZ and DR drills.
10. Independent approval and gradual production promotion.

## 14. Six production runbooks

**R1 — GPU hardware degradation:** isolate failing GPU/node; inspect Xid/ECC and scheduler; protect inference; verify healthy replacement.

**R2 — Serving SLO breach:** check p99/TTFT, queue age, HBM, model warmup and routing; use safe backpressure/scale-up; rollback bad model or config.

**R3 — Distributed training failure:** inspect rank/NCCL/fabric and last committed checkpoint; resume with verified state and record lost steps.

**R4 — Security policy violation:** deny unapproved workload, preserve audit trail, rotate compromised credentials and escalate to security.

**R5 — GPU quota/capacity shortage:** distinguish quota, physical supply and reservation; prioritize protected workloads and approved alternatives.

**R6 — Regional DR:** declare incident, verify recovery capacity, artifact integrity, KMS and routing; measure end-to-end RTO/RPO and fence writers.

## 15. Sixteen architecture lab exercises

1. Document stakeholders and workload SLOs.
2. Build physical/logical architecture diagrams.
3. Inventory GPU SKUs, zones, fabric and storage.
4. Define cluster/node-pool tenancy boundaries.
5. Draft identity/RBAC and admission threat model.
6. Design deny-by-default network policies.
7. Define artifact provenance and secret management.
8. Model training queues and gang placement.
9. Design checkpoint integrity and restore flow.
10. Define inference serving/canary/rollback.
11. Design GPU and workload telemetry dashboards.
12. Calculate N-1/AZ capacity headroom.
13. Define cost ledger and unit-economics KPIs.
14. Draft multi-region DR and failback plan.
15. Execute architecture review tabletop and failure scenarios.
16. Assemble evidence, score and independent signoff.

## 16. Twelve architecture failure scenarios

| Scenario | Required response |
|---|---|
| GPU SKU incompatible across AZs | capacity redesign |
| Shared KMS unavailable in DR | recovery identity redesign |
| Training gang fragments across nodes | topology-aware admission |
| Serving p99 worsens after canary | rollback |
| Tenant gains PriorityClass escalation | admission/RBAC fix |
| GPU quota approved but no stock | provider supply investigation |
| Checkpoint partially written | reject and restore committed version |
| Model registry compromised | signed artifact verification |
| Metrics show healthy GPUs but requests fail | end-to-end SLI debugging |
| AZ loss exceeds protected capacity | HA gap and remediation |
| Chargeback duplicates shared node cost | ledger correction |
| Failback creates duplicate writers | fencing/idempotency fix |

## 17. Architecture acceptance (100 points)

| Gate | Points |
|---|---:|
| Requirements, service ownership and threat model | 15 |
| Physical/GPU/network/data architecture | 20 |
| Kubernetes tenancy, identity and supply chain | 20 |
| Training, inference and model lifecycle | 15 |
| Observability, SLOs and operations | 10 |
| Capacity, FinOps, HA and DR | 15 |
| Review evidence and rollback | 5 |
| **Total** | **100** |

**Pass:** ≥85/100 with no critical tenant-isolation, data-residency, checkpoint-integrity, unverified failover capacity, SLO, KMS or rollback failure. **Current state: design published, not accepted.**

## 18. References

- [Kubernetes Production Environment](https://kubernetes.io/docs/setup/production-environment/)
- [Kubernetes Multi-Tenancy](https://kubernetes.io/docs/concepts/security/multi-tenancy/)
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)
- [Kueue](https://kueue.sigs.k8s.io/)
- [OpenTelemetry](https://opentelemetry.io/)
- [FinOps Foundation Framework](https://www.finops.org/framework/)
- [Google SRE Workbook](https://sre.google/workbook/table-of-contents/)

**Next:** Chapter 092 — Build a Secure GPU Cluster Platform.
