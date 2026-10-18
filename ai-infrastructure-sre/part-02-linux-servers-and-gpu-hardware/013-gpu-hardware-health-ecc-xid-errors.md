# Chapter 013 — GPU Hardware Health, ECC and Xid Errors

**Part 02 — Linux, Servers and GPU Hardware · Chapter 013 of 100**  
**Publication:** Complete. **Offline tests, real hardware fault injection, independent technical review and production acceptance:** Pending.

## Objectives
Understand NVIDIA GPU hardware-health signals, ECC error semantics, Xid/NVRM kernel events, retired pages and row remapping, PCIe/AER, thermals, power and NVLink health. Correlate these signals with Kubernetes workload impact; implement safe triage, quarantine and vendor escalation; distinguish synthetic policy checks from real device diagnostics.

## 1. Health architecture and failure domains
```text
GPU HBM / SM / interconnect / power / thermals
                   |
          firmware and driver
           /               \
   NVML / nvidia-smi     NVRM/Xid kernel log
           \               /
          DCGM exporter / host logs
                   |
       Prometheus / alert routing
                   |
 Kubernetes node + workload telemetry
                   |
       incident commander / GPU SRE
                   |
  safe containment / vendor RMA / restore
```
A GPU can remain visible in `nvidia-smi` while its workloads fail, and a device can show low utilization while being unhealthy. Health assessment must combine device errors, hardware counters, driver logs, workload failures and current incident context.

## 2. Key signal interpretation
| Signal | Meaning | Important caveat |
|---|---|---|
| Correctable ECC | memory errors corrected by hardware | a single event is not always a device failure; trends matter |
| Uncorrectable ECC | memory error not corrected | may cause workload failure; treat as high severity pending vendor guidance |
| Retired pages / row remapping | memory reliability mitigation | device- and architecture-specific; inspect pending remapping and reboot requirements |
| Xid event | NVIDIA driver-reported GPU error event | code meaning varies; do not classify every Xid as hardware failure |
| PCIe AER | PCIe link or transaction errors | can originate from GPU, root complex, riser or platform |
| NVLink errors | GPU interconnect health | only applicable where NVLink is present |
| GPU temperature / throttle | thermal or power operating constraint | compare against supported limits and workload conditions |
| GPU reset state | recoverability/availability indicator | resets can disrupt all users of a device |

**ECC semantics:** distinguish volatile versus aggregate counters, correctable versus uncorrectable events, and whether the GPU supports the queried metric. `N/A` must not become zero. Counter resets, driver restarts, GPU replacements and MIG configurations can affect interpretation.

**Xid semantics:** Xid numbers are diagnostic categories, not universal root causes. Use NVIDIA's current Xid documentation, relevant kernel log context, application failures, driver version and vendor diagnostics to decide remediation. Preserve the original event code and timestamp.

## 3. Read-only Linux and NVIDIA diagnostics
Run only on approved GPU hosts. Preserve UTC timestamps and redact asset/process details.
```bash
date -u
uname -r
lspci -nn | grep -Ei 'nvidia|3d controller'
nvidia-smi -L
nvidia-smi --query-gpu=index,uuid,name,driver_version,temperature.gpu,power.draw --format=csv
nvidia-smi -q -d ECC,PAGE_RETIREMENT,ROW_REMAPPER,PERFORMANCE,POWER,TEMPERATURE,PCI
nvidia-smi topo -m
journalctl -k -b --no-pager | grep -Ei 'NVRM|Xid|AER|pcie|nvidia' | tail -n 200
```
Supported `nvidia-smi -q -d` display categories depend on driver version and hardware; use `nvidia-smi -h` and `nvidia-smi -q` if a category is unsupported. Kernel logs may require elevated permissions; request approved read-only access rather than bypassing controls.

**Do not** run `nvidia-smi --gpu-reset`, ECC configuration changes, firmware flashing or GPU diagnostic stress tools against production without an approved change and explicit workload-safety review.

## 4. Kubernetes and application correlation
```bash
kubectl config current-context
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -A -o wide
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe node APPROVED_GPU_NODE
kubectl get daemonsets -A
```
Compare node allocatable GPU count, device plugin and GPU Operator health with the affected GPU UUID and PCI bus ID. Inspect application CUDA failures, NCCL rank logs, inference error rates, queue growth, TTFT/TPOT, training checkpoint age and recent driver changes. Kubernetes Node `Ready` is not proof of GPU health.

## 5. Alert and severity design
Use thresholds approved for the GPU model, driver and vendor support contract. Avoid one universal ECC/Xid threshold across all architectures.

| Condition | Initial response | Policy |
|---|---|---|
| unsupported/missing metric | telemetry investigation | do not treat as healthy |
| correctable ECC increase | trend and correlate | investigate if sustained or workload-impacting |
| uncorrectable ECC | high-priority triage | quarantine candidate, assess blast radius |
| repeated critical Xid | immediate correlation | stop rollout and consider isolation |
| PCIe/NVLink errors | hardware/network review | compare baseline and workload impact |
| thermal/power throttle | capacity/performance triage | inspect airflow, caps, workload |
| device disappears | incident and capacity review | do not automatically reboot repeatedly |

Alert on **telemetry freshness**, unexpected GPU count and GPU device loss separately. Device fault alerts should include host, GPU UUID, failure domain, driver, workload impact, escalation owner and incident link; avoid high-cardinality request labels.

## 6. Failure mode and effects analysis
| Failure | Blast radius | Detection | Containment |
|---|---|---|---|
| uncorrectable HBM ECC | one GPU, job or tenant | NVML, Xid, CUDA error | checkpoint-safe drain and isolate |
| PCIe link regression | node or affected GPU | AER, link diagnostics | hardware triage, stop rollout |
| GPU overheating | GPU/node | thermal and clocks | reduce unsafe load, vendor escalation |
| NVLink failure | multi-GPU job | NVLink errors, NCCL | topology-aware recovery |
| driver Xid storm | node pool if shared release | NVRM logs and error rate | halt rollout, revert image |
| GPU telemetry loss | observability blind spot | missing scrape | independent health checks |

## 7. Hands-on labs
**Lab A — Baseline:** capture sanitized GPU UUID, model, driver, ECC support, row-remap support, Xid history, power/temperature, PCIe and NVLink topology.

**Lab B — Time correlation:** join GPU fault timestamps with kubelet events, GPU Operator/device plugin logs, workload CUDA/NCCL failures and SLO burn windows.

**Lab C — Offline policy:** use the companion Python 3.11+ validator to evaluate synthetic fault snapshots. Preserve unsupported fields as `None`; fail closed on missing required evidence.

**Lab D — Controlled staging exercise:** with approved GPU capacity, simulate an *alert stream* containing a correctable ECC increase, uncorrectable ECC, critical Xid, stale telemetry and missing GPU. Do not inject real ECC faults or reset production GPUs.

**Lab E — Recovery:** rehearse approved cordon/drain, verified checkpoint restore, node replacement and workload smoke tests in staging.

**Lab F — Acceptance:** record evidence, owner, vendor ticket, root-cause hypothesis, rollback and final signoff.

## 8. Production incident simulations
**A — Correctable ECC spike:** compare volatile/aggregate counters, workload error rate and prior baseline; do not RMA solely on a synthetic count.

**B — Uncorrectable ECC:** preserve Xid/NVRM evidence, stop risky rollout, identify affected GPU UUID and verify checkpoint-safe recovery.

**C — Xid storm after driver rollout:** correlate version change, workload and device count; isolate canary and revert approved image if indicated.

**D — GPU disappears from inventory:** distinguish PCIe/hardware, kernel driver and device-plugin failure before replacing hardware.

**E — NVLink/NCCL stall:** compare topology, NVLink counters and rank logs; recover distributed job from verified checkpoint.

**F — Thermal throttling:** correlate temperature, clocks, power and application latency; inspect airflow and power policy.

**G — Telemetry pipeline outage:** detect stale scrape; rely on independent probes and stop risky maintenance.

**H — Persistent faults after replacement:** verify asset/UUID mapping, motherboard/riser, firmware and platform-level PCIe errors.

## 9. Troubleshooting decision matrix
| Symptom | Evidence | Likely domain | Escalation |
|---|---|---|---|
| ECC unavailable | NVML support, MIG mode | metric support | GPU platform |
| uncorrectable ECC | ECC/Xid, workload failure | GPU memory | hardware/vendor |
| repeated Xid | code, NVRM, CUDA logs | driver/device/app | GPU SRE/vendor |
| PCIe AER | kernel logs, link state | host/riser/GPU | hardware/cloud |
| NCCL timeout | rank logs, NVLink/NIC | interconnect | HPC/network |
| thermal throttle | clocks, power, fans | cooling/power | datacenter/cloud |
| no GPU allocatable | driver/plugin, node | GPU integration | Kubernetes |
| no metrics | scrape freshness | observability | telemetry SRE |

## 10. Operational runbooks
**RB-013-A — Initial triage:** capture UTC incident time, GPU UUID, node, Xid code, ECC status, workload impact, driver and change history.

**RB-013-B — ECC escalation:** distinguish correctable/uncorrectable and volatile/aggregate counters; check row remapping and vendor guidance, preserve evidence.

**RB-013-C — Xid investigation:** inspect exact Xid code in official guide; correlate kernel, application, PCIe and recent releases; avoid unsupported blanket interpretations.

**RB-013-D — GPU isolation:** verify N-1 capacity, tenant impact, checkpoint readiness, owner approval and rollback before cordon/drain/replacement.

**RB-013-E — Hardware/vendor case:** attach sanitized diagnostics, GPU UUID/serial via approved channel, firmware, driver, error timeline and reproducibility.

**RB-013-F — Post-repair validation:** verify new inventory, GPU health, CUDA smoke, NCCL where relevant, telemetry, application SLO and rollback plan.

**RB-013-G — RCA/COA:** document contributing factors, recurrence prevention, ownership, alert tuning and change control.

## 11. Executable lab
[Chapter 013 — GPU Health, ECC and Xid Acceptance Lab](../labs/013-gpu-hardware-health-ecc-xid-check.md) provides a synthetic policy evaluator with Python and pytest. It is **not** an NVML integration or vendor diagnostic.

## 12. Final 100-point assessment
| Domain | Points |
|---|---:|
| ECC/Xid hardware theory | 20 |
| Host and Kubernetes diagnostics | 20 |
| Alerting and fault correlation | 20 |
| Incident recovery and runbooks | 20 |
| Executable lab and evidence | 20 |
| **Total** | **100** |

Pass >=90 with no critical safety, missing telemetry, data integrity or tenant isolation blocker.

## References
- [NVIDIA Xid Errors](https://docs.nvidia.com/deploy/xid-errors/)
- [NVIDIA System Management Interface](https://docs.nvidia.com/deploy/nvidia-smi/)
- [NVIDIA NVML](https://docs.nvidia.com/deploy/nvml-api/)
- [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

**Next:** Chapter 014 — GPU Firmware, BIOS and Host Maintenance.
