# Databricks — SRE and Infrastructure Engineering Master Plan

## Audience and boundaries
SREs, cloud infrastructure engineers, platform engineers, and on-call operators. Focus on provisioning, governance, reliability, security, cost, incident response and hands-on operational labs. Application development, notebooks, and ML algorithms appear only as minimal test workloads.

## Standard chapter format
Objective; architecture and failure domains; prerequisites; operational commands and IaC; reproducible lab; failure injection; detection and triage; recovery and rollback; verification and acceptance criteria; production checklist.

## Learning path (60 chapters)
### Part 1 — Architecture and platform foundations
001 Databricks platform for SREs
002 Account, workspace and regional architecture
003 Control plane vs compute plane and cloud responsibilities
004 Workspace and compute resource inventory
005 Infrastructure architecture validation lab

### Part 2 — Cloud provisioning and networking
006 AWS workspace provisioning
007 Azure workspace provisioning
008 GCP workspace provisioning
009 VPC/VNet, private endpoints, DNS and egress
010 Cross-cloud networking acceptance lab

### Part 3 — Identity and security
011 Account and workspace identities
012 Service principals, workload identities and automation
013 Unity Catalog governance for operators
014 Secrets, keys and credential rotation
015 Access-denied incident response lab

### Part 4 — Compute and capacity
016 Classic and serverless compute models
017 Compute policies and guardrails
018 Cluster startup, autoscaling and pools
019 Runtime compatibility and upgrade planning
020 Capacity pressure and failed startup lab

### Part 5 — Jobs and platform operations
021 Jobs orchestration and task dependencies
022 Job retries, concurrency and idempotency
023 Operational APIs and CLI
024 Maintenance windows and change control
025 Failed job and dependency recovery lab

### Part 6 — Storage and data reliability
026 Cloud object storage integration
027 External locations, volumes and access paths
028 Delta table reliability for operators
029 Data retention, lifecycle and cleanup
030 Storage and permissions incident lab

### Part 7 — Observability
031 Audit and diagnostic logs
032 System tables and operational telemetry
033 Job and compute health dashboards
034 SLI, SLO, alerting and escalation
035 Monitoring acceptance lab

### Part 8 — Performance and troubleshooting
036 Compute saturation and resource contention
037 Slow jobs and Spark UI interpretation
038 Network, DNS and storage bottlenecks
039 Common platform error signatures
040 End-to-end slowness incident lab

### Part 9 — Reliability and recovery
041 Availability architecture and failure domains
042 Backup, recovery and data protection boundaries
043 Workspace and infrastructure disaster recovery
044 RPO, RTO and recovery drills
045 Regional outage simulation lab

### Part 10 — Automation and GitOps
046 Terraform provisioning and drift detection
047 Workspace configuration as code
048 CI/CD for infrastructure changes
049 Policy validation and operational testing
050 Rebuild-from-code acceptance lab

### Part 11 — FinOps and governance
051 DBU and cloud cost model
052 Compute rightsizing and budget guardrails
053 Cost attribution and anomaly detection
054 Capacity forecasting and quotas
055 Cost optimization project

### Part 12 — Production SRE capstone
056 Production readiness assessment
057 Incident command, escalation and communications
058 Multi-failure scenario and recovery
059 Upgrade, rollback and post-change verification
060 Final Databricks SRE operational assessment

## Lab rules
Use generic demo names only. Keep AWS, Azure and GCP variations explicit. Never run destructive failure tests in production. Record expected outputs, teardown commands, safety limits and validation evidence.
