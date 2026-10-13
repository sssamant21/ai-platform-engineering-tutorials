# Chapter 001 — Databricks Platform for SREs

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 01 — Architecture and Platform Foundations  
**Audience:** SRE, Cloud Infrastructure, Platform Operations  
**Lab risk:** Read-only / low risk  
**Cloud coverage:** AWS, Azure, GCP

## 1. Learning objectives

By the end, you can:
1. Explain Databricks account, workspace, control plane, compute plane, and underlying cloud resources.
2. Distinguish provider-managed components from customer-managed security, network, access and compute configuration.
3. Inventory workspace and compute objects without changing production state.
4. Trace a failed compute start through identity, policy, capacity, networking, logs and cloud infrastructure.
5. Produce a repeatable SRE baseline and escalation package.

## 2. What is Databricks to an SRE?

Databricks is a managed data and AI platform. An SRE operates the platform boundary: provisioning and configuring workspaces, governing compute, protecting access and network paths, monitoring jobs and platform health, managing cost and handling incidents. An SRE need not write complex Spark pipelines to diagnose platform failures.

**Operational scope:** account/workspace configuration; identity and service principals; cloud IAM and storage permissions; network and private connectivity; compute policies and lifecycle; jobs; logs, telemetry and audit; incident response; upgrades; recovery; quotas and spend.

**Outside this track:** detailed notebook development, ML model training algorithms, application business logic. Tiny test workloads are used only to validate infrastructure.

## 3. Reference architecture and responsibility boundaries

```text
Operators / automation / GitLab CI
              |
       SSO / OAuth / CLI / API
              |
     Databricks account + workspace
       (managed control plane)
       /         |           \
  Jobs API   Governance    Compute configuration
       \         |           /
        Compute execution boundary
     (classic cloud compute or serverless)
              |
   Cloud network + identity + object storage
        AWS / Azure / Google Cloud
              |
         Logs / metrics / audit
```

**Important:** Databricks deployment and ownership models vary by cloud, workspace features, and classic vs serverless compute. Do not assume all compute runs in a customer-owned VPC/VNet. Confirm your workspace architecture in official cloud-specific documentation and its deployed configuration.

### Platform components

| Component | SRE operational questions |
|---|---|
| Account | Who administers account-level configuration? Which region and identity provider? |
| Workspace | Who has access? What is the workspace URL, ID and region? |
| Control plane | Are API/UI requests succeeding? Is there a vendor incident? |
| Compute | Can compute start? Which policy, runtime, capacity and permissions apply? |
| Network | Are DNS, egress, private connectivity and cloud endpoints reachable? |
| Storage | Can workloads reach authorized cloud object storage and volumes? |
| Jobs | Are schedules firing, tasks retrying, and dependencies available? |
| Telemetry | Can on-call engineers access logs, audit events and incident evidence? |

## 4. Cloud-specific operational view

| Area | AWS | Azure | GCP |
|---|---|---|---|
| Infrastructure | VPC, subnets, routes, security groups | VNet, subnets, NSGs, routes | VPC, subnets, firewall rules |
| Identity | IAM roles and instance profiles where applicable | Microsoft Entra ID and managed identities where applicable | IAM and service accounts |
| Storage | S3 | ADLS Gen2 | Cloud Storage |
| Capacity diagnosis | EC2 limits and instance availability | VM family quotas and availability | Compute Engine quotas and availability |
| Network diagnosis | NAT, endpoints, DNS, security groups | NAT, private endpoints, DNS, NSGs | Cloud NAT, Private Service Connect where configured, DNS |

Cloud-specific options differ; these are investigation categories, not claims that every workspace uses every component.

## 5. Prerequisites and safety

- Read-only workspace access to view compute and jobs; ask administrators for least-privilege access where needed.
- Databricks CLI installed and authenticated using an organization-approved method.
- Optional cloud CLI access for AWS/Azure/GCP infrastructure inventory.
- A non-production workspace for any subsequent changes.
- Never paste tokens, credentials, account identifiers or sensitive logs into a public repository.
- Never terminate production compute as part of this chapter.

### Install and authenticate the modern Databricks CLI

Use the supported installation instructions for your OS: https://docs.databricks.com/dev-tools/cli/install.html (select the appropriate cloud documentation).

```powershell
databricks version
databricks auth login --host https://<your-workspace-host>
databricks auth profiles
databricks auth describe
```

Replace the placeholder host. For CI, use workload identity or service principal OAuth according to the supported configuration; do not embed personal access tokens in scripts. The CLI syntax may vary with version: check `databricks --help` and `databricks auth --help`.

## 6. Hands-on lab — Read-only workspace inventory

**Objective:** establish an operational baseline without modifying resources.

### Step A — Validate authentication

```powershell
databricks version
databricks auth describe
databricks current-user me --output json
```

**Expected:** CLI version, active authentication method and current-user JSON. If authorization fails, stop and verify host/profile and permissions.

### Step B — Inventory compute and policies

```powershell
databricks clusters list --output json
databricks cluster-policies list --output json
```

**Record:** cluster IDs (privately), creator/owner, state, runtime, node type, autoscaling bounds and policy IDs. Some fields depend on compute type and permissions. Serverless compute is not necessarily represented in the classic clusters listing.

### Step C — Inventory jobs

```powershell
databricks jobs list --output json
```

Record job ownership, enabled schedules, retry settings and recent failures using the UI or relevant Jobs API/CLI commands. If the listing is paginated, follow CLI help and pagination options; do not assume the first response is complete.

### Step D — Capture cloud dependency map

In a private operations worksheet, document:
- workspace host and cloud/region;
- classic vs serverless compute in use;
- identity model and responsible admin team;
- network connectivity and DNS path;
- storage endpoints and authorization path;
- monitoring, audit logs, dashboards and on-call owner;
- quotas, compute policies and cost guardrails.

### Step E — Check provider status

Consult the appropriate Databricks status page and your cloud provider status page. Treat a green vendor status as one signal, not proof that your own network, identity or quotas are healthy.

## 7. Failure simulation — Safe, non-destructive triage exercise

**Scenario:** a new non-production job cannot start because the assigned compute is unavailable.

Do **not** intentionally break a production cluster. Instead, use an instructor-provided sample incident or a disposable test environment.

Work through this decision tree:

```text
Job failed to start
  |
  +-- Is the workspace API/UI healthy?
  |      +-- No -> check vendor status, auth, DNS, network
  |
  +-- Does the job have permission to use its compute/policy?
  |      +-- No -> check principal, grants and policy
  |
  +-- Is the requested node type/region capacity available?
  |      +-- No -> check cloud quotas and capacity; choose approved alternative
  |
  +-- Are network and storage dependencies reachable?
  |      +-- No -> check private endpoints, routes, DNS and credentials
  |
  +-- Inspect job events, compute events and cloud logs
         -> isolate cause, remediate in test, re-run and validate
```

### Evidence checklist

| Evidence | Why it matters |
|---|---|
| UTC incident start/end | Aligns events across services |
| Workspace and region | Establishes scope |
| Job run ID and compute ID | Correlates events |
| Sanitized error text | Identifies failure signature |
| Recent changes | Reveals policy, IAM, network or runtime regressions |
| Cloud quota and capacity | Rules in/out infrastructure shortage |
| Network and identity checks | Separates authentication from connectivity |
| Recovery validation | Confirms service restored |

## 8. Troubleshooting matrix

| Symptom | Likely investigation | Safe first action |
|---|---|---|
| CLI unauthorized | Wrong host, expired credentials, missing grants | Verify `auth describe` and identity |
| Cluster pending | Capacity, quota, policy, image/startup issue | Inspect compute events and cloud quotas |
| Job cannot access data | Storage credentials, permissions, network | Review scoped access and logs |
| UI works but job fails | Workload identity, job owner, compute policy | Compare job principal and policy |
| Intermittent startup | Capacity variability, transient network failures | Correlate timestamps and retries |
| Slow job | Compute saturation, I/O, skew, external dependencies | Establish baseline before tuning |

## 9. Operational recovery runbook

1. **Declare scope:** affected workspace, cloud, region, jobs and users.
2. **Preserve evidence:** job IDs, timestamps, error messages and recent changes.
3. **Classify layer:** provider platform, identity, network, compute, storage or workload.
4. **Choose least disruptive fix:** renew authorized credentials, restore approved network configuration, request quota increase or change to an approved compute type.
5. **Use change control:** obtain approval before modifying production policies, compute or connectivity.
6. **Validate:** API reachable, authorized test job succeeds, alerts normalize, no regressions.
7. **Document:** timeline, contributing factors, remediation, rollback and follow-up owners.

**Rollback:** revert only the configuration changed under the approved incident/change record. A rollback is not a substitute for verifying that the original configuration is still safe.

## 10. Lab acceptance criteria

- [ ] CLI version and authentication method captured without exposing secrets.
- [ ] Workspace inventory completed or access gaps documented.
- [ ] Compute and jobs inventoried, with pagination/permission limitations noted.
- [ ] Cloud dependencies and responsibility boundaries documented.
- [ ] Sample compute-start incident triaged with evidence.
- [ ] Safe recovery and validation steps written.
- [ ] No production resources modified.

## 11. Production readiness checklist

- Named workspace/account administrators and on-call owners
- Least-privilege access and emergency access procedures
- Approved compute policies and runtime lifecycle process
- Network, DNS, storage and identity dependency map
- Centralized logs, audit access and actionable alerting
- Quota and cost monitoring
- Tested incident escalation and recovery playbooks
- Documented RPO/RTO expectations where applicable

## 12. Review questions

1. Which parts of Databricks are managed by the vendor, and which depend on your cloud configuration?
2. Why might a classic cluster fail while the workspace UI remains reachable?
3. How would you distinguish an identity failure from a network failure?
4. Which evidence is required before escalating a capacity incident?
5. What makes a read-only infrastructure inventory safe for production?

## 13. References

- Databricks documentation: https://docs.databricks.com/
- Databricks CLI: https://docs.databricks.com/dev-tools/cli/
- Azure Databricks documentation: https://learn.microsoft.com/azure/databricks/
- Databricks on Google Cloud: https://docs.databricks.com/gcp/
- Databricks status: https://status.databricks.com/

**Next:** Chapter 002 — Account, Workspace and Regional Architecture.
