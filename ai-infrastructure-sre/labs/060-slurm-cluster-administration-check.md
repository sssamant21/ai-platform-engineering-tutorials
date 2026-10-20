# Lab 060 — Slurm Cluster Administration Gate

**Python 3.11+ · pytest · offline synthetic Slurm readiness.** No scheduler API calls, node drains, GPU jobs or cloud changes.

## Goal

Fail closed on invalid GPU/node allocation geometry, N-1 GPU capacity, queue wait SLO, controller recovery, accounting lag, checkpoint RPO/RTO, telemetry staleness and critical Slurm security/governance evidence. Numeric limits are illustrative site policies; Boolean attestations do not independently prove live compliance.

## File: `slurm_admin_policy.py`

```python
from dataclasses import dataclass
import json
import math

@dataclass(frozen=True)
class Plan:
    cluster: str
    nodes: int
    gpus_per_node: int
    registered_gpus: int
    requested_gpus: int
    surviving_gpu_capacity: int
    required_n_minus_one_gpus: int
    queue_wait_p95_min: float
    queue_wait_slo_min: float
    controller_recovery_min: float
    controller_rto_min: float
    accounting_lag_min: float
    accounting_lag_slo_min: float
    checkpoint_age_min: float
    checkpoint_rpo_min: float
    restore_minutes: float
    restore_rto_min: float
    telemetry_age_s: float
    telemetry_max_age_s: float
    controller_ha_reviewed: bool
    slurmdbd_accounting_verified: bool
    munge_identity_verified: bool
    gpu_gres_config_verified: bool
    gpu_cgroup_isolation_verified: bool
    node_feature_inventory_verified: bool
    partitions_qos_approved: bool
    fairshare_policy_verified: bool
    preemption_policy_verified: bool
    backfill_policy_verified: bool
    gpu_topology_verified: bool
    cuda_nccl_compatibility_verified: bool
    network_fabric_verified: bool
    checkpoint_signal_verified: bool
    job_requeue_tested: bool
    drain_resume_runbook_verified: bool
    node_health_checks_verified: bool
    tenant_identity_verified: bool
    accounting_audit_verified: bool
    monitoring_alerts_verified: bool
    security_review_approved: bool
    incident_drills_completed: bool
    change_approved: bool
    rollback_tested: bool
    independent_review_approved: bool

def evaluate(p):
    if not isinstance(p, Plan):
        raise TypeError("Plan required")
    if not isinstance(p.cluster, str) or not p.cluster.strip():
        return {"accepted": False, "reason": "missing_cluster"}
    counts = (p.nodes, p.gpus_per_node, p.registered_gpus,
              p.requested_gpus, p.surviving_gpu_capacity, p.required_n_minus_one_gpus)
    if any(type(v) is not int or v <= 0 for v in counts):
        return {"accepted": False, "reason": "invalid_count"}
    metrics = (p.queue_wait_p95_min, p.queue_wait_slo_min,
               p.controller_recovery_min, p.controller_rto_min,
               p.accounting_lag_min, p.accounting_lag_slo_min,
               p.checkpoint_age_min, p.checkpoint_rpo_min,
               p.restore_minutes, p.restore_rto_min,
               p.telemetry_age_s, p.telemetry_max_age_s)
    if (any(type(v) not in (int, float) or not math.isfinite(v) or v < 0 for v in metrics)
            or min(p.queue_wait_slo_min, p.controller_rto_min,
                   p.accounting_lag_slo_min, p.checkpoint_rpo_min,
                   p.restore_rto_min, p.telemetry_max_age_s) <= 0):
        return {"accepted": False, "reason": "invalid_metric"}
    gates = {
        "gpu_inventory": p.registered_gpus == p.nodes * p.gpus_per_node,
        "allocation_capacity": p.requested_gpus <= p.registered_gpus,
        "n_minus_one_capacity": p.surviving_gpu_capacity >= p.required_n_minus_one_gpus,
        "queue_slo": p.queue_wait_p95_min <= p.queue_wait_slo_min,
        "controller_rto": p.controller_recovery_min <= p.controller_rto_min,
        "accounting_lag": p.accounting_lag_min <= p.accounting_lag_slo_min,
        "checkpoint_rpo": p.checkpoint_age_min <= p.checkpoint_rpo_min,
        "restore_rto": p.restore_minutes <= p.restore_rto_min,
        "telemetry_freshness": p.telemetry_age_s <= p.telemetry_max_age_s,
        "controller_ha_reviewed": p.controller_ha_reviewed is True,
        "slurmdbd_accounting_verified": p.slurmdbd_accounting_verified is True,
        "munge_identity_verified": p.munge_identity_verified is True,
        "gpu_gres_config_verified": p.gpu_gres_config_verified is True,
        "gpu_cgroup_isolation_verified": p.gpu_cgroup_isolation_verified is True,
        "node_feature_inventory_verified": p.node_feature_inventory_verified is True,
        "partitions_qos_approved": p.partitions_qos_approved is True,
        "fairshare_policy_verified": p.fairshare_policy_verified is True,
        "preemption_policy_verified": p.preemption_policy_verified is True,
        "backfill_policy_verified": p.backfill_policy_verified is True,
        "gpu_topology_verified": p.gpu_topology_verified is True,
        "cuda_nccl_compatibility_verified": p.cuda_nccl_compatibility_verified is True,
        "network_fabric_verified": p.network_fabric_verified is True,
        "checkpoint_signal_verified": p.checkpoint_signal_verified is True,
        "job_requeue_tested": p.job_requeue_tested is True,
        "drain_resume_runbook_verified": p.drain_resume_runbook_verified is True,
        "node_health_checks_verified": p.node_health_checks_verified is True,
        "tenant_identity_verified": p.tenant_identity_verified is True,
        "accounting_audit_verified": p.accounting_audit_verified is True,
        "monitoring_alerts_verified": p.monitoring_alerts_verified is True,
        "security_review_approved": p.security_review_approved is True,
        "incident_drills_completed": p.incident_drills_completed is True,
        "change_approved": p.change_approved is True,
        "rollback_tested": p.rollback_tested is True,
        "independent_review_approved": p.independent_review_approved is True,
    }
    failed = sorted(k for k, ok in gates.items() if not ok)
    return {"accepted": not failed, "failed_gates": failed,
            "total_gates": len(gates),
            "total_gpu_capacity": p.nodes * p.gpus_per_node}

def example():
    return Plan(
        "synthetic-slurm-gpu-cluster",
        8, 8, 64, 32, 56, 48,
        5.0, 15.0, 2.0, 10.0,
        1.0, 5.0, 5.0, 30.0,
        60.0, 120.0, 10.0, 120.0,
        *([True] * 25)
    )

if __name__ == "__main__":
    print(json.dumps(evaluate(example()), indent=2))
```

## File: `test_slurm_admin_policy.py`

```python
from dataclasses import replace
import pytest
from slurm_admin_policy import evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_type():
    with pytest.raises(TypeError):
        evaluate({})

def test_missing_cluster():
    assert evaluate(replace(example(), cluster=" "))["reason"] == "missing_cluster"

def test_bool_count():
    assert evaluate(replace(example(), nodes=True))["reason"] == "invalid_count"

def test_fractional_count():
    assert evaluate(replace(example(), registered_gpus=1.5))["reason"] == "invalid_count"

def test_nan():
    assert evaluate(replace(example(), queue_wait_p95_min=float("nan")))["reason"] == "invalid_metric"

def test_inf():
    assert evaluate(replace(example(), accounting_lag_min=float("inf")))["reason"] == "invalid_metric"

def test_bool_metric():
    assert evaluate(replace(example(), controller_rto_min=True))["reason"] == "invalid_metric"

def test_gate_count():
    assert evaluate(example())["total_gates"] == 34

def test_nonboolean_review():
    assert "independent_review_approved" in evaluate(
        replace(example(), independent_review_approved="yes"))["failed_gates"]

@pytest.mark.parametrize("field,gate,value", [
    ("registered_gpus", "gpu_inventory", 63),
    ("requested_gpus", "allocation_capacity", 65),
    ("surviving_gpu_capacity", "n_minus_one_capacity", 40),
    ("queue_wait_p95_min", "queue_slo", 16.0),
    ("controller_recovery_min", "controller_rto", 11.0),
    ("accounting_lag_min", "accounting_lag", 6.0),
    ("checkpoint_age_min", "checkpoint_rpo", 31.0),
    ("restore_minutes", "restore_rto", 121.0),
    ("telemetry_age_s", "telemetry_freshness", 121.0),
])
def test_numeric_gate(field, gate, value):
    assert gate in evaluate(replace(example(), **{field: value}))["failed_gates"]

@pytest.mark.parametrize("field,gate", [
    ("controller_ha_reviewed", "controller_ha_reviewed"),
    ("slurmdbd_accounting_verified", "slurmdbd_accounting_verified"),
    ("munge_identity_verified", "munge_identity_verified"),
    ("gpu_gres_config_verified", "gpu_gres_config_verified"),
    ("gpu_cgroup_isolation_verified", "gpu_cgroup_isolation_verified"),
    ("node_feature_inventory_verified", "node_feature_inventory_verified"),
    ("partitions_qos_approved", "partitions_qos_approved"),
    ("fairshare_policy_verified", "fairshare_policy_verified"),
    ("preemption_policy_verified", "preemption_policy_verified"),
    ("backfill_policy_verified", "backfill_policy_verified"),
    ("gpu_topology_verified", "gpu_topology_verified"),
    ("cuda_nccl_compatibility_verified", "cuda_nccl_compatibility_verified"),
    ("network_fabric_verified", "network_fabric_verified"),
    ("checkpoint_signal_verified", "checkpoint_signal_verified"),
    ("job_requeue_tested", "job_requeue_tested"),
    ("drain_resume_runbook_verified", "drain_resume_runbook_verified"),
    ("node_health_checks_verified", "node_health_checks_verified"),
    ("tenant_identity_verified", "tenant_identity_verified"),
    ("accounting_audit_verified", "accounting_audit_verified"),
    ("monitoring_alerts_verified", "monitoring_alerts_verified"),
    ("security_review_approved", "security_review_approved"),
    ("incident_drills_completed", "incident_drills_completed"),
    ("change_approved", "change_approved"),
    ("rollback_tested", "rollback_tested"),
    ("independent_review_approved", "independent_review_approved"),
])
def test_boolean_gate(field, gate):
    assert gate in evaluate(replace(example(), **{field: False}))["failed_gates"]
```

Run offline:

```bash
python -m pip install pytest
python slurm_admin_policy.py
python -m pytest -q
```

**44 planned pytest cases:** 10 standalone + 9 numeric + 25 Boolean. No runtime tests, live scheduler failover or independent review have been executed.

## Ten synthetic incident drills

1. GPU inventory mismatch.
2. Requested GPU overcommit.
3. N-1 GPU shortage.
4. Queue SLO breach.
5. Controller recovery timeout.
6. Accounting lag.
7. Checkpoint RPO breach.
8. Restore RTO breach.
9. Stale telemetry.
10. Missing MUNGE, cgroup, security, rollback or reviewer evidence.

## 100-point scorecard

Slurm services 20; GPU scheduling/fair-share 20; checkpoint/SLO 20; security/incidents 20; tests/live staging/review 20. Require >=90, all critical gates and signed staging evidence.
