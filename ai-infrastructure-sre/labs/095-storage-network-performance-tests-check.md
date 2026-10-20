# Lab 095 — Storage and Network Performance Acceptance Gate

**Chapter:** [095 — Implement Storage and Network Performance Tests](../part-12-enterprise-operations-and-final-capstones/095-storage-network-performance-tests.md)

**Status:** NOT EXECUTED. Requires authorized GPU staging, disposable storage, approved network endpoints, performance test images and independent review.

## 1. Safety

Never run fio against a production block device or existing checkpoint file. Use a disposable regular file, isolated staging PVC and synthetic data. Bound throughput tests to approved duration, size, bandwidth, egress and cost. Do not disable security controls or overwrite model/data artifacts.

## 2. Sixteen-exercise checklist

| ID | Exercise | Evidence | Status |
|---|---|---|---|
| E01 | Scope/cost approval | change record | Pending |
| E02 | Storage inventory | class/PVC/backend | Pending |
| E03 | Dataflow map | topology diagram | Pending |
| E04 | Performance budgets | SLO worksheet | Pending |
| E05 | Sequential write fio | JSON metrics | Pending |
| E06 | Random read/metadata | p99/IOPS | Pending |
| E07 | Object GET/PUT | throughput/retries | Pending |
| E08 | Checkpoint restore | hashes/RTO | Pending |
| E09 | Model cold/warm load | elapsed time | Pending |
| E10 | NIC/MTU counters | interface metrics | Pending |
| E11 | TCP throughput | iperf3 JSON | Pending |
| E12 | RDMA locality | fabric evidence | Pending |
| E13 | NCCL collectives | algbw/busbw | Pending |
| E14 | End-to-end correlation | training/serving SLO | Pending |
| E15 | Regression/rollback | controlled drill | Pending |
| E16 | Independent review | rubric/COAs | Pending |

## 3. Read-only preflight

\`\`\`bash
kubectl config current-context
kubectl get storageclass
kubectl get pvc -A
kubectl get pv
kubectl get nodes -L topology.kubernetes.io/zone -o wide
kubectl get events -A --sort-by=.lastTimestamp | tail -100
\`\`\`

No command above validates performance. Record unavailable storage classes or privileges as BLOCKED.

## 4. Offline performance budget sanity check

\`\`\`python
checkpoint_gib = 64
checkpoint_budget_s = 120
model_gib = 32
model_transfer_budget_s = 90
checkpoint_mib_s = checkpoint_gib * 1024 / checkpoint_budget_s
model_mib_s = model_gib * 1024 / model_transfer_budget_s
assert round(checkpoint_mib_s) == 546
assert round(model_mib_s) == 364
print(f"synthetic checkpoint: {checkpoint_mib_s:.1f} MiB/s")
print(f"synthetic model transfer: {model_mib_s:.1f} MiB/s")
\`\`\`

These are arithmetic targets, **not observed throughput**.

## 5. Safe staging fio test

\`\`\`bash
# Run only in an approved disposable mount with >2 GiB free.
mkdir -p /mnt/perf-test
fio --name=write --filename=/mnt/perf-test/fio-test.bin \
  --size=2G --rw=write --bs=1M --direct=1 \
  --ioengine=libaio --iodepth=16 --runtime=60 \
  --time_based --group_reporting --output-format=json \
  --output=/mnt/perf-test/fio-write.json
\`\`\`

This overwrites the **named test file**. Verify mount and target before running. \`libaio\`/\`O_DIRECT\` may not be supported on every filesystem. Retain actual JSON output and errors.

## 6. Isolated network test

\`\`\`bash
# Receiver in approved staging:
iperf3 -s -p 5201
# Sender in approved staging:
iperf3 -c <STAGING_RECEIVER_IP> -p 5201 -t 30 -P 4 --json
\`\`\`

Capture endpoint identity, network path, link rate, throughput, retransmits and cost. Do not equate TCP throughput with RDMA or NCCL.

## 7. Checkpoint integrity and restore

Generate synthetic checkpoint components, store checksums and a completion marker, then restore into a separate location. Verify **all** hashes and optimizer/RNG/sampler metadata; measure full wall-clock restore. An incomplete manifest must fail closed.

## 8. Evidence manifest

\`\`\`yaml
chapter: "095"
environment: "STAGING_REQUIRED"
publication: "PUBLISHED"
execution: "NOT_RUN"
execution_date_utc: null
cluster: null
storage_backend: null
storage_tier: null
network_topology: null
fio_sequential: "NOT_RUN"
fio_random: "NOT_RUN"
object_storage: "NOT_RUN"
checkpoint_integrity: "NOT_RUN"
checkpoint_restore_rto: null
model_cold_start: "NOT_RUN"
iperf3_tcp: "NOT_RUN"
rdma_validation: "NOT_RUN"
nccl_collectives: "NOT_RUN"
end_to_end_slo: "NOT_RUN"
rollback_drill: "NOT_RUN"
independent_review: "PENDING"
score: null
acceptance: "NOT_EVALUATED"
evidence_links: []
corrective_actions: []
\`\`\`

## 9. Acceptance

Score **15 + 20 + 20 + 20 + 15 + 10 = 100**. Pass ≥85 with every critical durability, restore, security, workload SLO and rollback gate independently verified. **Documentation publication is not performance acceptance.**

**Next:** Chapter 096 — Implement GPU SLOs and Observability.
