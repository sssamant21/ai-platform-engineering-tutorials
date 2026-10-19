# Chapter 019 — CUDA Processes, Streams and Contexts Executable Lab

**Python 3.11+ · pytest · offline synthetic DAG checks.** Live CUDA execution is optional and requires an approved staging GPU.

## Model
A CUDA stream orders work submitted to that stream; cross-stream ordering requires an explicit dependency or synchronization. This offline lab validates an abstract graph of operations, not actual CUDA execution.

Save as `stream_policy.py`:
```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Operation:
    name: str
    stream: str
    reads: frozenset[str]
    writes: frozenset[str]
    after: frozenset[str]

def evaluate(ops):
    if not isinstance(ops, (tuple, list)) or not ops:
        return {"accepted": False, "reason": "empty"}
    if any(not isinstance(o, Operation) for o in ops):
        return {"accepted": False, "reason": "invalid_operation"}
    names = [o.name for o in ops]
    if any(not isinstance(n, str) or not n for n in names) or len(names) != len(set(names)):
        return {"accepted": False, "reason": "invalid_names"}
    if any(not isinstance(o.stream, str) or not o.stream
           or not isinstance(o.reads, frozenset)
           or not isinstance(o.writes, frozenset)
           or not isinstance(o.after, frozenset)
           or any(not isinstance(v, str) or not v for v in (o.reads | o.writes | o.after))
           for o in ops):
        return {"accepted": False, "reason": "invalid_fields"}
    index = {o.name: i for i, o in enumerate(ops)}
    if any(dep not in index or dep == o.name for o in ops for dep in o.after):
        return {"accepted": False, "reason": "unknown_dependency"}
    edges = {o.name: set(o.after) for o in ops}
    for i, o in enumerate(ops):
        for previous in ops[:i]:
            if previous.stream == o.stream:
                edges[o.name].add(previous.name)
    visiting, visited = set(), set()
    def ancestors(name):
        if name in visiting:
            raise ValueError("cycle")
        if name in visited:
            return visited[name]
        visiting.add(name)
        result = set(edges[name])
        for dep in edges[name]:
            result.update(ancestors(dep))
        visiting.remove(name)
        visited[name] = result
        return result
    try:
        for name in names:
            ancestors(name)
    except ValueError:
        return {"accepted": False, "reason": "dependency_cycle"}
    conflicts = []
    for i, a in enumerate(ops):
        for b in ops[i + 1:]:
            overlap = bool((a.writes & (b.reads | b.writes))
                           or (b.writes & (a.reads | a.writes)))
            ordered = a.name in visited[b.name] or b.name in visited[a.name]
            if overlap and not ordered:
                conflicts.append((a.name, b.name))
    return {"accepted": not conflicts, "conflicts": conflicts}

def example():
    return (
        Operation("load", "s1", frozenset(), frozenset({"buffer"}), frozenset()),
        Operation("compute", "s2", frozenset({"buffer"}), frozenset({"output"}), frozenset({"load"})),
        Operation("readback", "s2", frozenset({"output"}), frozenset(), frozenset()),
    )
```

Save as `test_stream_policy.py`:
```python
from dataclasses import replace
import pytest
from stream_policy import Operation, evaluate, example

def test_baseline():
    assert evaluate(example())["accepted"]

def test_empty():
    assert evaluate([])["reason"] == "empty"

def test_invalid_operation():
    assert evaluate([{}])["reason"] == "invalid_operation"

def test_duplicate():
    o = example()[0]
    assert evaluate([o, o])["reason"] == "invalid_names"

def test_blank_name():
    assert evaluate([replace(example()[0], name="")])["reason"] == "invalid_names"

def test_blank_stream():
    assert evaluate([replace(example()[0], stream="")])["reason"] == "invalid_fields"

def test_unknown_dependency():
    assert evaluate([replace(example()[0], after=frozenset({"missing"}))])["reason"] == "unknown_dependency"

def test_self_dependency():
    assert evaluate([replace(example()[0], after=frozenset({"load"}))])["reason"] == "unknown_dependency"

def test_cycle():
    a, b, _ = example()
    assert evaluate([replace(a, after=frozenset({"compute"})), b])["reason"] == "dependency_cycle"

def test_unordered_write_read():
    a, b, _ = example()
    assert not evaluate([a, replace(b, after=frozenset())])["accepted"]

def test_unordered_writes():
    a = example()[0]
    b = replace(a, name="write2", stream="s2")
    assert not evaluate([a, b])["accepted"]

def test_same_stream_order():
    a = example()[0]
    b = replace(a, name="write2")
    assert evaluate([a, b])["accepted"]

def test_disjoint_streams():
    a = example()[0]
    b = replace(a, name="other", stream="s2", writes=frozenset({"other"}))
    assert evaluate([a, b])["accepted"]

def test_explicit_dependency():
    a = example()[0]
    b = replace(a, name="other", stream="s2", after=frozenset({"load"}))
    assert evaluate([a, b])["accepted"]

@pytest.mark.parametrize("field", ["reads", "writes", "after"])
def test_invalid_field_type(field):
    a = example()[0]
    assert evaluate([replace(a, **{field: set()})])["reason"] == "invalid_fields"

def test_invalid_resource():
    a = example()[0]
    assert evaluate([replace(a, writes=frozenset({""}))])["reason"] == "invalid_fields"
```

Run:
```bash
python -m pip install pytest
python -m pytest -q
```
**Expected 18 pytest cases** (15 standalone + 3 parameterized); execution not yet performed.

### Incident simulations
1. Remove the dependency between load and compute: policy rejects cross-stream read/write hazard.
2. Add two writers on separate streams: policy rejects unordered write/write conflict.
3. Add a cycle: policy rejects invalid dependency graph.
4. Move operations to the same stream: implicit submission order passes.
5. Add explicit event-style dependency: cross-stream graph passes.

### Staging validation
Compile and run the chapter's `stream_demo.cu` on an authorized GPU. Validate output, event ordering, CUDA error checks and stream synchronization. Compare CPU-side enqueue time and CUDA event GPU time with warm-up; avoid claiming overlap without measurement. Correlate process/context counts and GPU UUID with the active workload.

### 100-point rubric
Architecture 20; stream correctness 20; lifecycle/tenancy 20; incidents/runbooks 20; lab evidence 20. A passing synthetic graph test does not establish real CUDA concurrency or security isolation.
