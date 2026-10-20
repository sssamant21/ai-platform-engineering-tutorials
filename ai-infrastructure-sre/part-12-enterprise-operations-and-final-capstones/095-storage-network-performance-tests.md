# Chapter 095 — Implement Storage and Network Performance Tests

**Part 12 — Enterprise Operations and Final Capstones | AI Infrastructure Engineering & SRE**

**Status:** Documentation published. Runtime benchmarks, real GPU fabric tests, security review and independent production acceptance **PENDING**.

## 1. Mission

Design and execute a **reproducible storage and network performance acceptance program** for AI inference, distributed GPU training, dataset loading, model artifact distribution and checkpoint recovery on Kubernetes. Establish realistic throughput/latency baselines, expose tail-risk and identify infrastructure bottlenecks before production rollout.

**Prerequisites:** Chapters 091–094, approved staging GPU nodes, network/storage topology, Kubernetes access, object storage/PVC test areas, approved benchmark images, NCCL tooling and cost/change approvals.

### Learning outcomes

1. Map end-to-end data flows, I/O patterns and performance budgets.
2. Test object storage, block/PVC, parallel filesystems and local NVMe safely.
3. Distinguish throughput, IOPS, latency, queue depth and consistency/durability.
4. Validate network bandwidth, RTT, retransmits, MTU, RDMA and NCCL transport.
5. Correlate storage/network performance with training steps/sec and serving p99.
6. Define evidence, thresholds, rollback and acceptance without inventing benchmark results.

## 2. End-to-end performance architecture

\`\`\`text
 Dataset source / object store / model registry
                  |
         private network + IAM/KMS
                  |
        data loader / artifact cache
                  |
       ┌──────────┴──────────┐
       |                     |
   GPU training          GPU inference
   all-reduce fabric     model load + request path
       |                     |
  checkpoints/PVC       model artifact distribution
       └──────────┬──────────┘
                  |
   CSI / block / shared FS / NVMe / network
                  |
      metrics + traces + cost + SLO gates
\`\`\`

### Performance dimensions

| Workload | Primary concern | Supporting metrics |
|---|---|---|
| Dataset reads | sustained throughput and tail stalls | MB/s, p95/p99 read latency, cache hit rate |
| Random training reads | metadata/IOPS bottlenecks | IOPS, ops latency, file-open rate |
| Checkpoint write | bounded pause and durable commit | GiB/s, completion time, integrity |
| Checkpoint restore | RTO and data integrity | restore duration, checksum, resumed step |
| Model loading | cold-start impact on p99 | bytes/sec, model ready time |
| Inference request path | latency and loss | RTT, p99, retransmits |
| Distributed all-reduce | fabric/collective efficiency | NCCL algbw/busbw, timeouts |
| Storage/egress cost | useful work economics | $/training run, $/million tokens |

## 3. Workload characterization and budget

Capture real **staging** workload sizes and request profiles. Example synthetic budgets:

- Dataset: 2 TiB, 8 GPU workers, 400 MiB/s aggregate sustained read target.
- Checkpoint: 64 GiB, maximum 120-second checkpoint pause; required effective throughput **64 × 1024 / 120 ≈ 546 MiB/s**.
- Model artifact: 32 GiB, maximum 90-second transfer time; required transfer throughput **≈364 MiB/s** before load/warmup.
- Network: measured all-reduce bandwidth must meet a **same-SKU/same-topology approved baseline**, not a generic vendor peak.

These are **fictional test assumptions**, not observed environment measurements. Add headroom for encryption, protocol overhead, retries, contention, metadata and storage consistency.

## 4. Test safety and isolation

- Benchmark **only** an approved isolated staging bucket/PVC/namespace and preallocated budget.
- Use synthetic random data. Never overwrite customer datasets, checkpoints, snapshots or model weights.
- Verify capacity, write IOPS, cost and network egress quotas before high-load tests.
- Record node/zone, GPU/NIC SKU, kernel, driver, filesystem, CSI version, encryption and instance type.
- Do not disable TLS, encryption, network policy, KMS, audit or RDMA safeguards to inflate scores.
- Run warm/cold cache tests separately; avoid dropping caches on shared production nodes.
- Capture test duration, concurrent jobs, background load, retries, errors and billable usage.

## 5. Storage test matrix

| Storage tier | Test | Critical caveat |
|---|---|---|
| Local NVMe | sequential/random fio | ephemeral loss on node replacement |
| Block/PVC | IOPS, throughput, p99, fsync | provisioned tier/limits, AZ attachment |
| Shared filesystem | metadata, parallel read/write | lock/cache behavior and topology |
| Object store | upload/download, list, range GET | not a POSIX filesystem; API and consistency semantics |
| Checkpoint store | atomic/committed manifest restore | durable state, optimizer/RNG/sampler |
| Model artifact cache | cold and warm load | integrity, cache isolation and freshness |

**Never use destructive fio commands on a block device or existing production path.** The following pattern targets a disposable **regular file** inside a dedicated test directory.

## 6. Safe file-backed fio example

In a pre-approved disposable filesystem with sufficient capacity:

\`\`\`bash
mkdir -p /mnt/perf-test
fio --name=ai-sequential-write \
  --filename=/mnt/perf-test/fio-test.bin \
  --size=2G --rw=write --bs=1M \
  --direct=1 --ioengine=libaio --iodepth=16 \
  --time_based --runtime=60 --ramp_time=10 \
  --group_reporting --output-format=json \
  --output=/mnt/perf-test/fio-write.json
\`\`\`

**Important:** this test **overwrites the named test file** and may allocate 2 GiB; confirm \`/mnt/perf-test\` is disposable and that the target filesystem supports \`O_DIRECT\` and \`libaio\`. Some CSI/filesystems need different I/O engines. Do not point \`--filename\` at a raw device. Record write throughput, IOPS, latency percentiles, queue depth and errors.

For read benchmarking, first prepare and verify a disposable file, then use a separate \`--rw=read\` job; distinguish warm cache from cold cache. Compare against provisioned throughput/IOPS and filesystem semantics.

## 7. Checkpoint throughput and integrity

A checkpoint test must measure **write + durability/commit + restore**, not just buffered write speed. A robust flow:

1. Generate deterministic synthetic weights/optimizer/RNG/sampler state.
2. Write versioned temporary objects/files.
3. Flush/sync according to the storage backend's durability contract.
4. Record SHA-256 and manifest for each component.
5. Atomically publish a completion marker or versioned manifest after all objects succeed.
6. Restore into an isolated location and verify hashes.
7. Resume a small training workload and validate next step/optimizer consistency.
8. Measure elapsed wall-clock checkpoint pause and restore RTO.

For object stores, define backend-specific multipart upload, versioning, integrity and commit semantics. Do not assume POSIX rename or \`fsync\` exists for object storage.

## 8. Object storage benchmarking

Measure separately: PUT/GET throughput, p95/p99 API latency, request concurrency, multipart behavior, retry rate, throttling, metadata/list operations and KMS overhead. Use an **approved test bucket** and lifecycle cleanup. Tool choices include provider SDKs/CLIs or an approved benchmark harness.

Do not assume \`aws s3 cp\`, \`azcopy\` or \`gsutil\` results are comparable without normalizing region, instance network, object size, concurrency and encryption. Capture object version, ETag/checksum semantics and billed request/egress costs.

## 9. Network baseline: latency and throughput

### Non-destructive host/network inspection

\`\`\`bash
ip -br link
ip route
ss -s
ethtool -S <APPROVED_INTERFACE> | head -100
\`\`\`

The \`ethtool\` command may require privileges and not all drivers expose the same counters. Replace the placeholder with the approved interface.

### iperf3 isolated network test

On an authorized dedicated staging receiver:

\`\`\`bash
iperf3 -s -p 5201
\`\`\`

On an authorized sender:

\`\`\`bash
iperf3 -c <STAGING_RECEIVER_IP> -p 5201 -t 30 -P 4 --json
\`\`\`

The receiver must have an approved firewall rule and no untrusted access. The test can saturate the path; limit duration/concurrency and schedule it away from production. Record sender/receiver throughput, retransmits, CPU, NIC speed, RTT and background contention. TCP bandwidth is **not** proof of RDMA performance.

## 10. RDMA, InfiniBand and RoCE verification

Inspect installed tools and fabric only when supported:

\`\`\`bash
ibv_devinfo
ibstat
rdma link
\`\`\`

These commands may be unavailable in managed/cloud environments. Validate NIC/GPU locality, link state, GID, PFC/ECN where relevant, MTU, firewall, RDMA permissions and GPUDirect compatibility. Run provider-approved RDMA bandwidth/latency tests in staging; do not infer successful GPUDirect from an active NIC link.

## 11. NCCL multi-node performance

Use Chapter 093's approved \`nccl-tests\` harness with verified rank-to-GPU mapping, matching GPU SKUs and a validated launcher. Capture:

- \`algbw\` and \`busbw\` by payload size;
- collective correctness errors and timeouts;
- NCCL transport selected (socket, IB/RDMA, etc.);
- fabric errors/retransmits, GPU/NIC topology and contention;
- warmup, repetitions, median, dispersion and worst case.

Avoid tuning \`NCCL_IB_DISABLE\`, \`NCCL_SOCKET_IFNAME\` or \`NCCL_P2P_DISABLE\` without understanding the actual network path. A successful TCP fallback does not validate RDMA.

## 12. End-to-end data pipeline benchmark

A storage or network microbenchmark may pass while training throughput fails. Measure together:

1. GPU steps/sec and useful accelerator occupancy.
2. Data-loader wait time and CPU saturation.
3. Storage p99 latency, IOPS and throughput.
4. Network retransmits, RTT and all-reduce time.
5. Checkpoint pause, age and retry rate.
6. Inference model cold-start and request p99.
7. Unit cost per successful step/request.

Use tracing/correlation identifiers across Pod UID, node, volume, model version and storage request. Confirm performance regressions with repeated runs and controlled variables.

## 13. Performance troubleshooting decision tree

\`\`\`text
Training slow / inference cold-start high
               |
        Is GPU compute busy?
          /            \
        no              yes
        |                |
 Data-loader wait?   NCCL/compute bottleneck?
  /        \           /          \
 yes        no       NCCL        kernels
  |          |         |            |
 storage    CPU      NIC/MTU      precision/
 latency    decode   topology     occupancy
\`\`\`

Storage queue depth, CPU decompression, small-file metadata storms, model distribution, cross-AZ traffic and GPU fabric congestion can all cause similar symptoms. Verify bottleneck with correlated measurements rather than a single dashboard.

## 14. Six production runbooks

**R1 — Dataset read slowdown:** inspect p99 reads, metadata/IOPS, cache hit, data-loader wait, node/zone locality, throttling and storage service health; protect running jobs and revert recent change.

**R2 — Checkpoint timeout/corruption:** halt unsafe promotion, inspect commit marker, KMS, object versions, storage latency and hashes; restore last complete checkpoint.

**R3 — Model cold-start regression:** compare artifact size, transfer speed, cache, KMS, registry and warmup; preserve protected serving replicas and rollback bad release.

**R4 — Network throughput collapse:** inspect NIC link/errors, CPU saturation, retransmits, MTU, congestion, security policy and cross-AZ path; validate baseline before tuning.

**R5 — NCCL/RDMA performance incident:** isolate intra-node vs inter-node, rank mapping, NIC affinity, transport selection and fabric health; avoid disabling RDMA without change approval.

**R6 — Storage/network cost anomaly:** reconcile cross-AZ egress, object requests, retries, cache misses, provisioned IOPS and orphaned test data; confirm invoice impact.

## 15. Sixteen hands-on exercises

1. Approve test scope, cost and safety.
2. Inventory storage classes, PV/PVC and backend semantics.
3. Map data, model and checkpoint flows.
4. Define synthetic I/O workloads and performance budgets.
5. Benchmark disposable block/PVC sequential writes.
6. Benchmark random reads and metadata operations.
7. Measure object storage GET/PUT and retries.
8. Test checkpoint durable commit and hash restore.
9. Measure model artifact cold and warm load.
10. Inspect NIC, route, MTU and error counters.
11. Run bounded staging TCP throughput/RTT tests.
12. Validate RDMA fabric and GPU/NIC locality.
13. Run intra-node and inter-node NCCL collectives.
14. Correlate storage/network with training and serving SLOs.
15. Inject one controlled performance regression and rollback.
16. Assemble evidence and independent acceptance review.

## 16. Twelve failure scenarios

| Symptom | Root-cause focus |
|---|---|
| High fio throughput, slow training | data-loader/metadata wait |
| High IOPS but checkpoint timeout | sequential bandwidth/durability |
| PVC Pending | storage class/zone/capacity |
| Checkpoint incomplete | missing atomic commit marker |
| Object PUT succeeds, restore fails | version/checksum/KMS |
| Model cold-start p99 spike | artifact transfer/cache |
| iperf3 passes, NCCL slow | RDMA/GPU-NIC topology |
| RDMA link up, transport uses TCP | NCCL selection/permissions |
| TCP retransmits rise | congestion/MTU/NIC errors |
| Cross-AZ throughput degrades | placement/egress |
| NVMe fast, recovery fails | ephemeral storage loss |
| Storage bill rises after tuning | provisioned IOPS/egress |

## 17. Acceptance rubric (100 points)

| Domain | Points |
|---|---:|
| Workload characterization and budgets | 15 |
| Block/PVC/object storage benchmarks | 20 |
| Checkpoint durability and restore | 20 |
| Network TCP/RDMA/NCCL verification | 20 |
| End-to-end workload correlation | 15 |
| Rollback, evidence and independent review | 10 |
| **Total** | **100** |

**Pass:** ≥85/100 and no critical data corruption, unverified checkpoint restore, network security bypass, uncontrolled test impact, failed protected workload SLO or absent rollback. **Current state: documentation only; tests not executed.**

## 18. References

- [fio Documentation](https://fio.readthedocs.io/en/latest/fio_doc.html)
- [iperf3](https://software.es.net/iperf/)
- [NVIDIA NCCL User Guide](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/)
- [NVIDIA nccl-tests](https://github.com/NVIDIA/nccl-tests)
- [Kubernetes Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [NVIDIA GPUDirect RDMA](https://docs.nvidia.com/cuda/gpudirect-rdma/)

**Next:** Chapter 096 — Implement GPU SLOs and Observability.
