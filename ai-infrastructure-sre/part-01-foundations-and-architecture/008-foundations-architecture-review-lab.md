# Chapter 008 — Foundations Architecture Review Lab

**Part 01 — Foundations and Architecture · Chapter 008 of 100 · Part 01 Capstone**  
**Publication:** Complete. **Hands-on execution, staging evidence, independent technical/security approval and production acceptance:** Pending.

## Mission and prerequisites
This capstone integrates Chapters 001–007: enterprise AI infrastructure architecture, workload taxonomy, GPU reference architectures, service ownership, dependency mapping, SLO/error-budget governance and security trust boundaries. The learner must produce an auditable architecture decision record (ADR), executable acceptance evidence, incident simulation results and a go/no-go recommendation.

**Scenario:** A regulated enterprise runs latency-sensitive inference and scheduled distributed training across GPU Kubernetes node pools in AWS EKS or Azure AKS. It requires independent zones, controlled model provenance, tenant separation, recoverable checkpoints, reliable on-call escalation and cost governance. All quantities below are **synthetic design assumptions**, not observed performance or provider guarantees.

## 1. Reference architecture and data flows
```text
       users / applications / training clients
                       |
          WAF + gateway + authentication
                       |
         admission / rate limit / identity
                       |
             Kubernetes control plane
                 /            \
       inference pool       training pool
        zone A/B/C           topology-aware
          |     |             GPU workers
        GPU hosts  <------>  network/RDMA
          |                     |
        signed models       datasets/checkpoints
          \                     /
          object storage / registry / KMS
                       |
       OTel + DCGM + Prometheus + alerting
                       |
       on-call / incident commander / FinOps
```
**Trust boundaries:** public ingress, tenant identity, Kubernetes API/RBAC, GPU host/device isolation, model registry/artifacts, object storage, RDMA fabric, telemetry and outbound tools. Identify which dependencies are zonal, regional or global; do not assume zone spread eliminates shared failure modes.

## 2. Workload requirements
| Contract | Inference | Training |
|---|---|---|
| Goal | completed streams, TTFT/TPOT | job success, useful tokens/s |
| Scheduling | warm capacity and N-1 admission | topology-aware placement and quotas |
| Data | signed weights, prompts, cache | datasets, optimizer, checkpoints |
| Network | gateway, model fetch, RAG | collective traffic, RDMA/NCCL |
| Recovery | bounded retries, canary rollback | consistent restore including data cursor |
| Telemetry | stream completion, queue, GPU | rank step time, GPU, I/O, checkpoints |
| Cost | cost per successful output token | cost per completed training objective |

Define actual targets and measurement populations before provisioning. No synthetic result may be described as a production SLO measurement.

## 3. Architecture decision record
**ADR-008-01 — Separate serving and training GPU capacity.**  
*Context:* serving requires low-tail latency and warm replicas; training is bursty and may saturate memory/network.  
*Decision:* isolate production inference and training by node pool, identity, quotas, network policy and scheduling priority.  
*Alternatives:* shared pool with priority/preemption, separate clusters, Slurm for training.  
*Tradeoffs:* more idle capacity and administration in exchange for isolation and predictable recovery.  
*Rollback:* move only approved workloads to previous pinned pool configuration after validating security and capacity.

**ADR-008-02 — Multi-zone inference.** Maintain independent warm replicas where supported, reserve enough safe capacity to survive largest zone failure, and verify shared IAM, KMS, DNS, registry and load-balancer dependencies.

**ADR-008-03 — Immutable artifacts.** Require pinned container/model digests, verified signatures, scoped workload identity and auditable promotion; never use floating tags in production.

**ADR-008-04 — Checkpoint integrity.** Require tested restore of model, optimizer, scheduler, RNG and data cursor. Uploaded bytes are not evidence of recoverability.

## 4. Capacity and SLO math
For independent zones with measured safe throughput `C_i`:
```text
N-1 capacity = sum(C_i) - max(C_i)
admission ceiling = N-1 capacity × (1 - reserve)
request error budget = floor(eligible requests × (1 - SLO))
burn rate = observed bad fraction / (1 - SLO)
```
Illustrative zones A/B/C = 100 requests/s each, reserve = 20%, forecast = 150 requests/s. N-1 capacity = 200 requests/s and admission ceiling = 160 requests/s: **synthetic capacity gate passes**. For 1,000,000 eligible requests and 99.9% SLO, 1,000 bad requests are budgeted; 700 consume 70%. Different model mixes and streaming patterns require new measured safe-throughput values.

## 5. Hands-on capstone plan
**Lab 1 — Inventory and owners:** list services, teams, support tiers, cloud accounts/subscriptions, region/zone, versions and on-call coverage. Flag unowned critical services.

**Lab 2 — Dependency graph:** record service → dependency edges for GPU host, driver, CUDA, device plugin, scheduler, registry, KMS, object storage, fabric, telemetry and DNS. Mark runtime/startup/recovery dependency classes and shared failure domains.

**Lab 3 — Topology and sizing:** record GPU type, count, memory, PCIe/NUMA, NVLink/NVSwitch where available, NIC/RDMA capabilities, CPU/RAM, dataset throughput and checkpoint capacity. Use vendor-supported matrices, not assumed compatibility.

**Lab 4 — Reliability:** calculate error budget and N-1 capacity using the companion Python evaluator. Design end-to-end stream-completion, TTFT/TPOT, scheduling and checkpoint-restore SLIs.

**Lab 5 — Security:** verify namespace and service-account separation, workload identity, least-privilege RBAC, NetworkPolicy, signed images/models, storage ACLs, secret boundaries and residency.

**Lab 6 — Observability:** specify DCGM/Prometheus/OpenTelemetry signals, client synthetic probes, missing-telemetry alerts, pager delivery and dashboard ownership.

**Lab 7 — Failure simulation:** in an approved staging environment, simulate loss of a GPU node or zone, device-plugin outage, model registry failure, NCCL stall, checkpoint corruption and security policy rejection. Predefine abort conditions.

**Lab 8 — Change and recovery:** exercise immutable deployment, canary verification, rollback, escalation and incident commander communication. Record UTC timeline and customer-impact evidence.

**Lab 9 — Cost:** attribute GPU-hours, idle reservation, checkpoint/storage/network cost and retry waste; document quota and spend approval.

**Lab 10 — Architecture review:** assemble ADR, diagram, risk register, test results, SLO dashboards, security signoff, operational runbooks and go/no-go decision.

## 6. Safe discovery commands
Execute only against approved contexts and redact infrastructure details:
```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get daemonsets -A
kubectl get resourcequota -A
kubectl get networkpolicy -A
kubectl get events -A --sort-by=.lastTimestamp
nvidia-smi -L
nvidia-smi topo -m
nvidia-smi --query-gpu=index,name,driver_version,memory.total --format=csv
```
`nvidia-smi` requires compatible hardware and driver; Kubernetes commands do not prove cloud IAM or network-policy enforcement. Avoid printing secrets or regulated data.

## 7. Production incident simulations and expected decisions
| Injected fault | Evidence | Expected containment | Owner |
|---|---|---|---|
| GPU node unavailable | Xid/ECC, allocatable GPU, node events | bound admission, shift only to healthy capacity | GPU/Kubernetes SRE |
| Device plugin failure | DaemonSet health, GPU resources | pause unsafe rollout, restore compatible plugin | GPU platform |
| Zone loss | queue age, stream SLO, N-1 headroom | activate verified surviving capacity | inference SRE |
| Registry/KMS unavailable | model pull and auth logs | preserve signed warm cache, pause cold rollouts | ML platform/security |
| NCCL hang | rank traces, fabric metrics | checkpoint-safe recovery | HPC/network |
| Corrupt checkpoint | restore checksum, optimizer state | reject restore, preserve last verified snapshot | storage/ML owner |
| Telemetry outage | absent scrape, synthetic probe | independent health check, stop game day | observability SRE |
| Cross-tenant access | IAM audit, denied access test | security containment and evidence preservation | security/incident command |

Do not run destructive fault injection in production without an explicit change, approved blast radius, rollback and abort authority.

## 8. Troubleshooting and recovery runbooks
**RB-008-A — Preflight:** confirm target context, owners, versions, quotas, data classification, evidence capture, approvals, abort conditions and backup restore points.

**RB-008-B — GPU availability:** correlate Linux host, driver, plugin, scheduler, workload placement and quota; avoid blind node restarts.

**RB-008-C — Model/data path:** check registry digest/signature, identity, KMS, DNS, cache, object store and tenant authorization; do not bypass integrity controls.

**RB-008-D — Distributed training:** inspect rank logs, NCCL/fabric, dataset I/O, checkpoint age and consistent restore; require ML owner acceptance.

**RB-008-E — SLO breach:** validate eligible request accounting and complete streams, check burn windows and missing telemetry, restrict risky changes and confirm rollback.

**RB-008-F — Security incident:** contain identity or workload, preserve audit chain, involve security/privacy, authorize recovery and revalidate isolation.

**RB-008-G — Review closure:** attach UTC evidence, ADR, risk acceptance, SLO/FinOps measurements, RCA/COA owners and signed go/no-go.

## 9. Architecture review checklist
- [ ] Named service and backup owners, escalation and change approver
- [ ] Workload taxonomy and latency/throughput objectives
- [ ] Supported GPU/driver/CUDA/NCCL/operator version matrix
- [ ] Network, storage, model, IAM and cloud dependency graph
- [ ] Failure domains, largest-zone loss and shared services tested
- [ ] Request SLI, error budget, training and checkpoint SLOs
- [ ] Tenant identity, network, GPU isolation and artifact signatures
- [ ] Telemetry health, alert routing, incident command and rollback
- [ ] Recovery targets, checkpoint integrity and restore drills
- [ ] Cost ownership, quotas, budgets and risk register

## 10. Final assessment — 100 points
| Domain | Points |
|---|---:|
| Architecture, workload contracts and ADR | 20 |
| GPU, cloud, networking and dependency topology | 20 |
| SLO, N-1, DR and recovery design | 20 |
| Security, ownership and incident response | 20 |
| Executable lab, evidence, troubleshooting and FinOps | 20 |
| **Total** | **100** |

**Passing standard:** >=90 points with no critical security, data-integrity, unowned-service or recovery blocker. A high numerical score cannot override a failed hard gate. Distinguish **documentation complete**, **offline tests executed**, **staging validated**, and **production accepted**.

## 11. Executable acceptance lab
[Chapter 008 — Architecture Review Acceptance Lab](../labs/008-foundations-architecture-review-check.md) implements deterministic policy checks for N-1 capacity, error budgets, ownership, security and recovery with Python 3.11+ and pytest. Inputs are synthetic and do not certify real GPU hardware or cloud controls.

## References
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [Kubernetes Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [NVIDIA Data Center Documentation](https://docs.nvidia.com/datacenter/)

**Next:** Chapter 009 — GPU Server Hardware, PCIe and NUMA, beginning Part 02 — Linux, Servers and GPU Hardware.
