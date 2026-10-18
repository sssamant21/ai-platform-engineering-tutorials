# Chapter 005 — Executable Linux, Kubernetes and Cloud Dependency Mapping Lab

**Status:** Published. Offline pytest execution, live infrastructure discovery and independent review pending.

## Setup
Python 3.11+ and pytest:
```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install pytest
```
PowerShell: `.venv\Scripts\Activate.ps1`.

## 1. Deterministic dependency graph evaluator
Save as `dependency_map.py`:
```python
from dataclasses import dataclass
import json

@dataclass(frozen=True)
class Component:
    name: str
    owner: str
    failure_domain: str
    critical: bool = False

def validate(components, edges):
    if not isinstance(components, dict) or not isinstance(edges, list):
        raise TypeError("components dict and edges list required")
    if not components:
        return {"accepted": False, "reason": "empty_inventory"}
    for key, c in components.items():
        if not isinstance(c, Component) or not isinstance(key, str) or not key:
            return {"accepted": False, "reason": "invalid_component"}
        if c.name != key or not c.failure_domain.strip():
            return {"accepted": False, "reason": "invalid_component"}
    for edge in edges:
        if not isinstance(edge, tuple) or len(edge) != 2:
            return {"accepted": False, "reason": "invalid_edge"}
        a, b = edge
        if a not in components or b not in components or a == b:
            return {"accepted": False, "reason": "invalid_edge"}
    missing_owners = sorted(k for k, c in components.items()
                            if c.critical and not c.owner.strip())
    graph = {k: set() for k in components}
    for consumer, dependency in edges:
        graph[consumer].add(dependency)
    state = {}
    def visit(n):
        if state.get(n) == 1:
            return True
        if state.get(n) == 2:
            return False
        state[n] = 1
        if any(visit(dep) for dep in sorted(graph[n])):
            return True
        state[n] = 2
        return False
    cyclic = any(visit(k) for k in sorted(graph) if not state.get(k))
    failed = []
    if missing_owners:
        failed.append("critical_owner")
    if cyclic:
        failed.append("dependency_cycle")
    return {"accepted": not failed, "failed_gates": failed,
            "missing_owners": missing_owners,
            "component_count": len(components),
            "dependency_count": len(edges)}

def blast_radius(components, edges, failed):
    if failed not in components:
        raise ValueError("unknown failed component")
    reverse = {k: set() for k in components}
    for consumer, dependency in edges:
        if consumer not in components or dependency not in components:
            raise ValueError("invalid dependency")
        reverse[dependency].add(consumer)
    impacted = set()
    queue = [failed]
    while queue:
        node = queue.pop(0)
        for consumer in sorted(reverse[node]):
            if consumer not in impacted and consumer != failed:
                impacted.add(consumer)
                queue.append(consumer)
    return sorted(impacted)

def example():
    nodes = {
        "inference": Component("inference", "llmops-sre", "zone-a", True),
        "gpu-driver": Component("gpu-driver", "gpu-sre", "zone-a", True),
        "model-cache": Component("model-cache", "ml-platform", "region", True),
        "registry": Component("registry", "ml-platform", "region", True),
        "telemetry": Component("telemetry", "observability", "region"),
    }
    edges = [("inference", "gpu-driver"), ("inference", "model-cache"),
             ("model-cache", "registry"), ("inference", "telemetry")]
    return nodes, edges

if __name__ == "__main__":
    nodes, edges = example()
    print(json.dumps({"validation": validate(nodes, edges),
                      "registry_blast_radius": blast_radius(nodes, edges, "registry")}, indent=2))
```
**Model limitation:** The graph treats all edges as potential dependencies. It does not distinguish runtime from startup or observability impact, prove actual outage propagation or inspect real cloud/Kubernetes resources. Blast radius is **potential exposure**, not confirmed customer impact.

## 2. Automated pytest suite
Save as `test_dependency_map.py`:
```python
from dataclasses import replace
import pytest
from dependency_map import Component, validate, blast_radius, example

def test_baseline():
    nodes, edges = example()
    assert validate(nodes, edges)["accepted"]

def test_registry_blast_radius():
    nodes, edges = example()
    assert blast_radius(nodes, edges, "registry") == ["inference", "model-cache"]

def test_driver_blast_radius():
    nodes, edges = example()
    assert blast_radius(nodes, edges, "gpu-driver") == ["inference"]

def test_telemetry_exposure():
    nodes, edges = example()
    assert blast_radius(nodes, edges, "telemetry") == ["inference"]

def test_empty_inventory():
    assert validate({}, [])["reason"] == "empty_inventory"

def test_missing_critical_owner():
    nodes, edges = example()
    nodes["gpu-driver"] = replace(nodes["gpu-driver"], owner="")
    assert "critical_owner" in validate(nodes, edges)["failed_gates"]

def test_noncritical_missing_owner_allowed():
    nodes, edges = example()
    nodes["telemetry"] = replace(nodes["telemetry"], owner="")
    assert validate(nodes, edges)["accepted"]

def test_cycle_detected():
    nodes, edges = example()
    edges.append(("registry", "inference"))
    assert "dependency_cycle" in validate(nodes, edges)["failed_gates"]

def test_self_edge_rejected():
    nodes, edges = example()
    assert validate(nodes, edges + [("inference", "inference")])["reason"] == "invalid_edge"

def test_unknown_edge_rejected():
    nodes, edges = example()
    assert validate(nodes, edges + [("unknown", "registry")])["reason"] == "invalid_edge"

def test_invalid_edge_shape():
    nodes, edges = example()
    assert validate(nodes, edges + [("inference",)])["reason"] == "invalid_edge"

def test_component_key_mismatch():
    nodes, edges = example()
    nodes["registry"] = replace(nodes["registry"], name="other")
    assert validate(nodes, edges)["reason"] == "invalid_component"

def test_missing_domain():
    nodes, edges = example()
    nodes["registry"] = replace(nodes["registry"], failure_domain="")
    assert validate(nodes, edges)["reason"] == "invalid_component"

def test_unknown_failure():
    nodes, edges = example()
    with pytest.raises(ValueError):
        blast_radius(nodes, edges, "unknown")

def test_wrong_type():
    with pytest.raises(TypeError):
        validate([], [])

@pytest.mark.parametrize("component", ["inference", "gpu-driver", "model-cache", "registry", "telemetry"])
def test_known_component_impact(component):
    nodes, edges = example()
    assert isinstance(blast_radius(nodes, edges, component), list)
```

Run:
```bash
python dependency_map.py
python -m pytest -q
```
Expected: **20 pytest cases** (15 standalone + 5 parameterized). Execution not performed.

## 3. Approved staging extension
1. Export sanitized Linux host, Kubernetes node/pod and cloud resource inventories.
2. Join by stable service IDs; do not infer ownership from labels alone.
3. Classify every edge as runtime, startup, control plane, identity, network, storage or telemetry.
4. Add owner, failure domain, SLO impact, runbook and evidence link.
5. Identify critical unowned dependencies, cycles, shared regional services and high-fanout nodes.
6. Run a separately approved synthetic outage game day with abort controls; record actual impact.
7. Compare observed failures with predicted blast radius, update graph and RCA/COA.

## 4. 100-point rubric
Inventory and ownership (20), discovery (20), failure-domain analysis (20), recovery/security (20), evidence and tests (20). Pass >=90 with no critical unowned service or unsafe action.
