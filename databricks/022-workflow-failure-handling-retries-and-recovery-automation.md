# Chapter 022 — Workflow Failure Handling, Retries and Recovery Automation

**Track:** Databricks SRE & Infrastructure Engineering  
**Part:** 05 — Jobs, Workflows and Production Reliability  
**Clouds:** AWS, Azure, GCP  
**Prerequisites:** Chapters 016–021  
**Safety:** Production read-only; recovery tests only on approved staging jobs and synthetic data.

## 1. Objectives

At the end of this chapter, you can classify workflow failures, design bounded retries, distinguish repair from rerun, reconcile partial outputs, implement safe automation gates, and demonstrate operational recovery with measurable acceptance criteria.

## 2. Recovery architecture

    Trigger -> Run admission -> Task DAG -> Compute -> External systems
                      |             |             |
                      +----- run events and logs --+
                                     |
                             Failure classifier
                                     |
                          Safety / idempotency gate
                              /      |       \
                         Retry     Repair    Escalate
                                     |
                           Validate outputs and SLO
                                     |
                               Audit and close

**Principle:** A retry is not a recovery strategy unless it is safe to repeat the operation and the underlying condition is likely to change.

## 3. Failure taxonomy

| Category | Evidence | Default response |
|---|---|---|
| Transient network | Timeout, reset, intermittent upstream 5xx | Bounded backoff |
| Cloud capacity | Provisioning/instance shortage | Capacity investigation, limited retry |
| Identity/authentication | 401, token expiration, federation error | Repair identity before retry |
| Authorization | 403, Unity Catalog privilege denial | Least-privilege review |
| Deterministic code/schema | Reproducible exception | Fix code/schema |
| Data quality | Contract violation, invalid input | Quarantine or correct data |
| Resource exhaustion | OOM, executor loss, spill | Profile and rightsize |
| External side effect | Partial write/API call | Reconcile before rerun |
| Scheduling | Missed trigger, queue backlog | Correct trigger/admission |
| Platform disruption | Regional or service issue | Incident escalation and recovery plan |

HTTP codes and exception text are clues, not proof; correlate task, compute and dependency telemetry.

## 4. Retry engineering

Define maximum attempts, interval, backoff, jitter, overall deadline and retryable error classes. Databricks job/task settings may support retry count and interval, while sophisticated backoff/jitter may require an external orchestrator or supported task logic. **Do not claim native support for every policy.**

A practical retry budget is bounded by the business deadline:

    remaining_time = deadline - now
    retry_allowed = transient_failure
                    AND attempts_below_limit
                    AND remaining_time > expected_recovery_time
                    AND idempotency_confirmed

Avoid simultaneous retries across hundreds of jobs during a shared cloud or storage incident.

## 5. Idempotency and partial completion

Before retrying, determine whether the task:
- Reads only, writes an overwrite, performs MERGE, appends or invokes an external API.
- Uses a stable batch identifier or checkpoint.
- Can create duplicate rows or repeated notifications.
- Has already committed a transaction or external side effect.
- Is consumed by downstream tasks.

**Never assume that a failed task made no changes.** Record committed outputs and reconcile before retry, repair or backfill.

## 6. Repair versus rerun versus backfill

| Action | Appropriate when | Major risk |
|---|---|---|
| Task retry | Transient, idempotent attempt | Duplicate side effects |
| Run repair | Some tasks succeeded and repair is supported | Reused state and dependencies |
| Full rerun | All operations are safely repeatable | Reprocessing cost/duplicates |
| Targeted backfill | Known missing interval, bounded input | Overlap and ordering |
| Manual intervention | Irreversible or ambiguous side effects | Human error, delayed recovery |

Check the installed CLI/API for supported repair operations and task semantics. A repair may not undo prior successful tasks.

## 7. Lab A — Windows PowerShell preflight

~~~powershell
databricks version
databricks auth profiles
$env:DATABRICKS_CONFIG_PROFILE = "demo-staging"
databricks auth describe
databricks current-user me --output json
databricks jobs list --output json
databricks jobs list-runs --limit 25 --output json
databricks jobs get-run --help
databricks jobs get-run-output --help
databricks jobs repair-run --help
~~~

Confirm the profile points to an authorized staging workspace. Some commands and flags vary by CLI release. Listing may be paginated or permission-filtered.

## 8. Lab B — Investigate an authorized failed run

~~~powershell
# Replace the example run ID.
databricks jobs get-run --run-id 67890 --output json
~~~

Collect UTC trigger/start/end timestamps, lifecycle and result states, task run IDs, attempt counts, error messages, cluster/warehouse details and recent configuration changes. Review task output with the supported CLI command and authorized task run ID. Sanitize evidence before sharing.

### Failure worksheet

| Item | Finding |
|---|---|
| Job/run/task | Restricted identifiers |
| First failure UTC | |
| Failed dependency | |
| Retryable? | |
| Committed side effects | |
| Safe repair scope | |
| Deadline remaining | |
| Proposed mitigation | |
| Approval and rollback | |

## 9. Lab C — Offline retry decision simulator

This lab makes **no Databricks API calls** and performs **no recovery action**.

~~~powershell
$cases = @(
  [pscustomobject]@{ Error="NETWORK_TIMEOUT"; Attempts=1; MaxAttempts=3; Idempotent=$true; MinutesLeft=25 },
  [pscustomobject]@{ Error="PERMISSION_DENIED"; Attempts=1; MaxAttempts=3; Idempotent=$true; MinutesLeft=25 },
  [pscustomobject]@{ Error="NETWORK_TIMEOUT"; Attempts=2; MaxAttempts=3; Idempotent=$false; MinutesLeft=25 },
  [pscustomobject]@{ Error="NETWORK_TIMEOUT"; Attempts=3; MaxAttempts=3; Idempotent=$true; MinutesLeft=25 }
)
$retryable = @("NETWORK_TIMEOUT","TEMPORARY_CAPACITY")
$cases | ForEach-Object {
  $decision = if (
    $_.Error -in $retryable -and
    $_.Attempts -lt $_.MaxAttempts -and
    $_.Idempotent -and
    $_.MinutesLeft -gt 5
  ) { "RETRY_CANDIDATE" } else { "STOP_AND_TRIAGE" }
  [pscustomobject]@{ Error=$_.Error; Attempts=$_.Attempts; Decision=$decision }
}
~~~

A retry candidate is not permission to automatically rerun production. Human approval and full dependency checks still apply.

## 10. Lab D — Synthetic recovery acceptance

On a disposable staging job with synthetic input and a defined cost cap:

1. Record current job settings, run-as identity, retry policy and baseline.
2. Introduce a controlled synthetic transient task failure.
3. Verify retry count and alert delivery.
4. Verify failure stops after the configured attempt budget.
5. Inspect task history and identify any committed outputs.
6. Correct the synthetic failure.
7. Select task repair or a full rerun **only after idempotency review**.
8. Confirm final output has no duplicates or missing records.
9. Restore original staging settings and remove disposable resources.

Use a tabletop exercise if staging mutation is not approved.

## 11. Lab E — Recovery automation design

A safe recovery controller should be **read-only by default** and require explicit approval for state-changing operations.

~~~text
Observe run -> classify -> correlate dependency health
                  |
           Is failure transient?
             no -> escalate
             yes
                  |
        Is task repeatable and bounded?
             no -> escalate
             yes
                  |
        Is deadline and retry budget valid?
             no -> escalate
             yes
                  |
        Is approved recovery enabled?
             no -> suggest action
             yes -> execute supported action
                         |
                 verify outputs / alert
~~~

Required safeguards: deduplication key, per-job cooldown, max actions per window, circuit breaker, global outage suppression, audit trail, least-privilege identity and kill switch.

## 12. Lab F — Repair command inspection

~~~powershell
databricks jobs repair-run --help
databricks jobs run-now --help
databricks jobs cancel-run --help
~~~

**Inspection only.** Do not execute repair, run-now or cancellation in production as part of this chapter. For an approved staging repair, follow the installed CLI/API contract, identify the latest repair attempt and validate which tasks will rerun.

## 13. Incident A — Repeated storage timeouts

**Triage:** inspect storage latency, throttling, network, executor logs and shared workload patterns.

**Response:** limit retries, protect storage, apply backoff and repair the actual dependency.

**Acceptance:** storage errors normalize and output completeness is verified.

## 14. Incident B — Expired service principal credential

**Triage:** identity, OAuth/federation, expiry, workspace assignment and audit events.

**Response:** restore approved authentication or rotate credential; do not keep retrying 401 errors.

**Acceptance:** intended workload succeeds and unauthorized operations remain denied.

## 15. Incident C — Unity Catalog privilege regression

**Triage:** compare effective grants, run-as principal and recent governance changes.

**Response:** restore minimum authorized privileges through change control.

**Acceptance:** job accesses approved data; negative permission test remains denied.

## 16. Incident D — Partial external write

**Triage:** identify committed rows/files/API requests, checkpoints, batch IDs and downstream consumers.

**Response:** reconcile and deduplicate, then repair the smallest safe scope.

**Acceptance:** exact expected output cardinality and no duplicate external effects.

## 17. Incident E — Retry storm and capacity exhaustion

**Triage:** shared upstream failure, job attempt frequency, concurrent compute and cloud quotas.

**Response:** activate cooldown/circuit breaker, reduce repeated attempts and fix root cause.

**Acceptance:** queue, compute and cloud costs return toward baseline.

## 18. Incident F — Repair causes downstream inconsistency

**Triage:** compare task DAG, previously successful outputs, repair scope and consumer state.

**Response:** stop further automated repairs, reconcile data and choose controlled replay/backfill.

**Acceptance:** upstream and downstream state agrees for the affected interval.

## 19. Troubleshooting decision tree

    Failed workflow
      |
      +-- Credential or permission error? -> repair IAM/UC, no blind retry
      |
      +-- Transient dependency error? -> inspect health, bounded retry
      |
      +-- Deterministic data/code error? -> fix and validate
      |
      +-- Partial side effects? -> reconcile before any rerun
      |
      +-- Capacity/queue pressure? -> mitigate contention and retry storm
      |
      +-- Unclear cause? -> preserve evidence and escalate

## 20. Production recovery runbook

1. Detect and classify impact; identify job, run, task and business deadline.
2. Preserve UTC timeline, error signatures, task graph and configuration changes.
3. Assess external dependency health and shared incidents.
4. Determine whether the operation is idempotent and whether outputs committed.
5. Evaluate retry budget, cost and deadline.
6. Choose the smallest approved retry/repair/backfill action.
7. Verify output correctness, downstream consistency and security boundaries.
8. Record action, actor, approval, rollback and residual risk.
9. Update automation rules and incident documentation.

## 21. SLOs and monitoring

Monitor task failure rate, retry attempts per run, repair frequency, duplicate-prevention incidents, time to validated recovery, missed deadlines, queue depth and incremental compute cost. Alert on correlated failure spikes, exhausted retry budgets, unexpected repair activity and missing output.

Define SLOs by workload criticality; do not adopt universal thresholds without baseline evidence.

## 22. Change and rollback

Before enabling recovery automation:
- Document permitted jobs, error classes and approved actions.
- Validate synthetic failure cases and negative cases.
- Test circuit breaker, deduplication and kill switch.
- Limit blast radius to staging, then controlled production cohorts.
- Keep audit and incident escalation independent of the automation.
- Roll back the automation configuration if it increases failures or duplicates.

Disabling automation cannot reverse already committed external writes.

## 23. Operational acceptance checklist

- [ ] Failure taxonomy and ownership documented.
- [ ] Retryable versus non-retryable errors mapped.
- [ ] Timeouts and retry budgets aligned to deadlines.
- [ ] Idempotency and partial side-effect controls validated.
- [ ] Repair/rerun/backfill decision rules approved.
- [ ] Staging synthetic failure test completed.
- [ ] Recovery automation is read-only by default.
- [ ] Cooldown, circuit breaker, deduplication and kill switch tested.
- [ ] Six incident scenarios reviewed.
- [ ] Audit, notifications and rollback procedures validated.
- [ ] No production jobs or credentials changed in lab.

## 24. Review questions

1. Why is a transient error not sufficient reason to retry?
2. How does repair differ from a full rerun?
3. What evidence is required before replaying a partially completed job?
4. How can recovery automation amplify a cloud incident?
5. Why must output validation be part of the recovery SLO?

## 25. References

- Databricks Jobs: https://docs.databricks.com/aws/en/jobs/
- Databricks Jobs CLI: https://docs.databricks.com/aws/en/dev-tools/cli/reference/jobs-commands
- Databricks authentication: https://docs.databricks.com/aws/en/dev-tools/auth/
- Azure Databricks Jobs: https://learn.microsoft.com/azure/databricks/jobs/
- Databricks on GCP: https://docs.databricks.com/gcp/

**Next:** Chapter 023 — Workflow Monitoring, Alerting and SLA Management.
