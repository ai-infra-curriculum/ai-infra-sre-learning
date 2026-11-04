# GPU Telemetry with DCGM — Distinguishing a Bad GPU from a Bad Workload

## Why this chapter exists

Chapter 02 wired up the metric surface the engine emits — TTFT, TPOT, queue depth, cache hit, preemption rate. Those metrics tell you the *symptom* of an incident: "TTFT regressed," "preemption rate climbed," "stall rate spiked." They do not tell you whether the cause was a change in the workload (a bursty tenant on the chat surface, a traffic-mix shift toward long prompts, a cache-eviction storm) or a change in the hardware (a GPU with failing HBM, a thermal throttle, a PCIe bandwidth collapse after a BIOS update).

Distinguishing those two causes during an incident is often the difference between a five-minute remediation ("drain this one node") and a four-hour investigation ("the engine is misbehaving everywhere"). You cannot make the distinction from engine-side metrics alone. You need GPU-level telemetry, authored with intention, with alerts that fire on the hardware signal rather than the workload symptom.

This chapter is the authoring reference for DCGM — NVIDIA's Data Center GPU Manager — and the `dcgm-exporter` Prometheus integration. We cover what metrics to emit, what the XID error codes actually mean, how to compose GPU telemetry with the engine-side signals from chapter 02 so the on-call can route cleanly, and how to alert on "bad GPU" versus "bad workload" without creating a noise storm.

The canonical references are the [NVIDIA DCGM documentation](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/index.html), the [dcgm-exporter GitHub](https://github.com/NVIDIA/dcgm-exporter), the [NVIDIA XID error reference](https://docs.nvidia.com/deploy/xid-errors/index.html), and the [NVIDIA GPU operator for Kubernetes](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/overview.html). The depth beyond DCGM (kernel-level traces via Nsight Systems, NVTX, CUPTI) is at the `ai-infra-performance` peer — chapter 07 marks the boundary.

## 1. The GPU observability surface — what DCGM gives you

DCGM is NVIDIA's management daemon for data-center GPUs (Tesla, A100, H100, H200, B200, L40S, etc.). It wraps the lower-level NVML (NVIDIA Management Library), adds policies for health checks, grouping, and configuration, and exposes a scrape endpoint via `dcgm-exporter` that fits cleanly into the Prometheus / OTel stack from chapters 01–02.

The surface splits into five groups of signals. Each group answers a different diagnostic question.

### Utilization — "is the GPU busy doing work?"

- `DCGM_FI_DEV_SM_ACTIVE` — fraction of time at least one warp is active on each SM (streaming multiprocessor). The *compute occupancy* signal. A low `SM_ACTIVE` on a GPU that is scheduled a workload means the workload is blocked on something other than compute.
- `DCGM_FI_DEV_SM_OCCUPANCY` — fraction of the *resident* warps vs. the max resident on each SM. Deeper diagnostic for kernel-launch efficiency; the application / kernel peer cares more about this one than the SRE.
- `DCGM_FI_PROF_SM_ACTIVE` / `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` — tensor-core activity, as a fraction. Decode-heavy LLM workloads produce a specific `SM_ACTIVE` / `PIPE_TENSOR_ACTIVE` ratio; a drift in that ratio across the fleet is a signal that the kernel path changed.

Diagnostically: *low* utilization on a scheduled GPU with pending work in the queue means the GPU is stalled on memory, PCIe, or NVLink. *High* utilization with a bad TPOT means the workload is compute-bound and you need a different model or more hardware. The utilization reading by itself doesn't tell you which; combine with the next group.

### Memory — "is HBM healthy and uncontended?"

- `DCGM_FI_DEV_FB_USED` / `DCGM_FI_DEV_FB_FREE` / `DCGM_FI_DEV_FB_TOTAL` — frame-buffer (HBM) usage. Set the alert on "used / total > 0.95 for N minutes" as a leading indicator of OOM.
- `DCGM_FI_DEV_FB_RESERVED` — reserved frame-buffer. Changes with driver / MIG / vGPU configuration; a sudden change can mean MIG partitioning drifted.
- `DCGM_FI_DEV_MEM_COPY_UTIL` — fraction of time the memory copy engine is active. Combined with SM activity, splits "compute-bound" from "memory-bound."
- `DCGM_FI_PROF_DRAM_ACTIVE` — DRAM activity, as a fraction. Direct HBM-bandwidth saturation signal; the number you want for the "is HBM the bottleneck" question.
- `DCGM_FI_DEV_ECC_SBE_VOL_TOTAL` — single-bit ECC errors, volatile counter. Non-zero single-bit errors are a *leading* indicator of an HBM failure; they get corrected transparently by ECC but accumulate toward a replacement decision.
- `DCGM_FI_DEV_ECC_DBE_VOL_TOTAL` — double-bit ECC errors, volatile counter. A non-zero value here is a *failure* — DBE events are uncorrectable and have corrupted a memory read. Any DBE is a page-the-oncall event; the GPU is likely coming out of the pool regardless.

<!-- needs-research: cross-check the exact DCGM field names against the installed dcgm-exporter version — NVIDIA has renamed and reorganized fields across DCGM 2.x to 3.x; verify against the current docs before shipping alert rules. -->

### Interconnect — "is the data getting to and between GPUs?"

- `DCGM_FI_PROF_PCIE_TX_BYTES` / `DCGM_FI_PROF_PCIE_RX_BYTES` — PCIe transmit / receive bytes. Combined with the host's PCIe generation and lane count, gives the saturation fraction.
- `DCGM_FI_PROF_NVLINK_TX_BYTES` / `DCGM_FI_PROF_NVLINK_RX_BYTES` — NVLink TX / RX bytes, per link. Sum across links to get the aggregate; the per-link counters surface uneven loading from a bad link.
- `DCGM_FI_DEV_NVLINK_REPLAY_ERROR_COUNT_TOTAL` — NVLink replay errors. Growing replay-error counts are a signal of a failing link; the fabric is correcting via retry but the link is degrading.
- `DCGM_FI_DEV_NVLINK_CRC_DATA_ERROR_COUNT_TOTAL` / `DCGM_FI_DEV_NVLINK_CRC_FLIT_ERROR_COUNT_TOTAL` — NVLink CRC errors at data and flit (flow control) levels. Non-zero is already-corrupted-traffic-retried; sustained growth is a link to drain.

On multi-GPU model-parallel deployments (tensor-parallel 4-way or 8-way, pipeline-parallel across nodes), interconnect bandwidth saturation is often the *primary* bottleneck and your TTFT regression traces straight back to an NVLink hiccup. Alerts on the interconnect family are paging alerts on multi-GPU workloads and ticket alerts on single-GPU ones.

### Thermal and power — "is the GPU running within its envelope?"

- `DCGM_FI_DEV_GPU_TEMP` — core temperature, °C.
- `DCGM_FI_DEV_MEMORY_TEMP` — HBM temperature, °C. On H100/H200, this can be the limiting thermal signal rather than the core temp.
- `DCGM_FI_DEV_POWER_USAGE` — instantaneous power draw, watts.
- `DCGM_FI_DEV_POWER_VIOLATION` — time (in microseconds) the GPU was clock-capped by the power limit.
- `DCGM_FI_DEV_THERMAL_VIOLATION` — time (in microseconds) the GPU was clock-capped by thermal throttling.
- `DCGM_FI_DEV_SYNC_BOOST_VIOLATION` — time clock-capped by sync-boost governor.
- `DCGM_FI_DEV_BOARD_LIMIT_VIOLATION` — time clock-capped by board-level limit.

Thermal throttling on an LLM workload reads as a quiet, distributed TPOT regression — the GPU is doing the work, just slower, and the engine's throughput metric drops a few percent. Without the DCGM signal, it reads as "the service got slower"; with the signal, it reads as "node X-37 is in a datacenter aisle that is 2°C warmer than the rest; coordinate with facilities." The alert is high-impact-per-page.

### Health and error — "did the GPU hit a hard failure?"

- `DCGM_FI_DEV_XID_ERRORS` — the XID error code of the most recent GPU-side event. This is the primary hardware-health signal.
- `DCGM_FI_DEV_ROW_REMAP_PENDING` / `DCGM_FI_DEV_ROW_REMAP_FAILURE` — memory row-remapping status. Non-zero pending means the GPU has memory rows waiting to be remapped on next reset; failures are unrecoverable.
- `DCGM_FI_DEV_RETIRED_SBE` / `DCGM_FI_DEV_RETIRED_DBE` / `DCGM_FI_DEV_RETIRED_PENDING` — retired memory pages, single-bit and double-bit, and pending retirements.

XID codes are the single highest-signal hardware diagnostic on the GPU. Section 3 covers them in detail — they are specific enough that your alert rules can key on them.

## 2. Installing dcgm-exporter — the operator's view

The `dcgm-exporter` is maintained by NVIDIA and packaged for Kubernetes via the [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/overview.html). The GPU Operator also installs the driver, the container-toolkit, and the MIG manager; on a fleet that is managing multiple of those surfaces, the operator is the honest choice. On a non-K8s fleet (bare-metal, Slurm, Ray), install `dcgm-exporter` directly.

Key deployment choices:

- **Scrape interval.** 10–30 seconds is the typical default. Shorter intervals produce more time-series writes but catch transient thermal throttle events that a 60-second interval would miss. Lean shorter on inference fleets; the metric volume is modest relative to Prometheus's overall scrape load.
- **Metric selection.** The exporter supports a config file that selects which DCGM fields to expose. Default field lists are large; a production inference fleet rarely needs more than 30–50 fields. Trim for cardinality — each field × each GPU is a time series.
- **Label surface.** The exporter emits `gpu` (index), `UUID`, `device`, `modelName`, `Hostname`, optional `pci_bus_id`. On a Kubernetes fleet, join against the node label surface (`kubernetes.io/hostname`, `node.kubernetes.io/instance-type`, `topology.kubernetes.io/zone`) via the kube-state-metrics or node-exporter data. On a non-K8s fleet, inject your inventory labels at scrape time.
- **MIG partitioning.** On H100/H200/B200 with MIG (Multi-Instance GPU) enabled, dcgm-exporter can emit per-MIG-slice metrics. If your fleet runs MIG, align the label surface so the engine's per-pod labels can join to the per-slice DCGM metrics.
- **Running the collector alongside Triton / vLLM / SGLang.** Some engine-side configurations use an application-level CUPTI or NVML binding for their own profiling; `dcgm-exporter` and these can coexist, but on a heavily contended node the DCGM scrape itself can become non-trivial. Measure the overhead on a canary before fleetwide rollout.

Reference the [dcgm-exporter installation guide](https://github.com/NVIDIA/dcgm-exporter/blob/main/README.md) for the current flags and the Grafana dashboard templates NVIDIA ships. The dashboards are a solid starting point; customize once your SLO stack is defined.

## 3. XID codes — the hardware-health vocabulary

XID codes are NVIDIA's enumerated set of GPU-side error events, emitted by the driver when the hardware raises an interrupt or a software fault is detected. The full reference is [docs.nvidia.com/deploy/xid-errors/](https://docs.nvidia.com/deploy/xid-errors/index.html); the operator-relevant subset:

### Fatal or near-fatal (drain the GPU and page)

- **XID 48 — Double-bit ECC error.** Uncorrectable memory corruption. Drain immediately; take the GPU out of the pool for RMA-or-reset decision.
- **XID 63 / 64 — Row-remapping event.** The GPU has detected a failing memory row and is scheduling a remap on next reset. Follow the vendor's remap policy — on H100/H200 this is a healthy mechanism up to a quota; above the quota, the card needs replacement.
- **XID 74 — NVLink error.** NVLink has a persistent fault. On multi-GPU workloads, degrades the whole tensor-parallel group — drain the entire group, not just the one GPU.
- **XID 79 — GPU has fallen off the bus.** The PCIe link to the GPU has dropped. Catastrophic for the running workload; page immediately; node needs reboot and often an RMA.
- **XID 92 — High single-bit error rate.** HBM is throwing more SBE than the correction mechanism is comfortable with. Drain and flag for replacement.
- **XID 94 — Contained ECC error / unrecoverable.** Similar to XID 48 for newer architectures; policy is drain.

### Software-faulting or workload-correlated (ticket, investigate)

- **XID 13 — Graphics engine exception.** Often a CUDA kernel crash. The workload crashed, not the hardware. Investigate the engine logs, not the GPU.
- **XID 31 — GPU memory page fault.** Application-side out-of-bounds access. Workload bug, not hardware.
- **XID 43 — Reset channel verify failure.** Reset path confusion; often transient. Log and watch.
- **XID 45 — Preemptive cleanup.** The GPU was asked to preempt a workload. Expected during scheduler-initiated preemptions on some drivers; confirm against your workload's scheduler behavior.

### Heat / power (environment)

- **XID 61 / 62 — Internal micro-controller warning / breakpoint.** Firmware-level; rare, driver-vendor investigation.
- **XID 83 — Fatal error, GPU hung.** Combined signal — look at power/thermal immediately adjacent in time.

The *alerting rule you want* keys on a severity bucket, not the raw XID. In your fleet metadata, keep a lookup `xid_severity[code]` with `fatal / workload / ticket` as the three buckets, and emit an engineered label `xid_severity` alongside the raw code. Then the alert expression is one rule per bucket rather than one per code.

## 4. Alerts that distinguish a bad GPU from a bad workload

The clinical question — bad GPU or bad workload — is answerable with a handful of composed signals. The alert set below is the defendable minimum for an inference fleet.

### Alert 1 — hardware fatal (page)

Fires on any XID in the fatal bucket, any DBE, any row-remap failure.

```promql
(
  increase(DCGM_FI_DEV_ECC_DBE_VOL_TOTAL[5m]) > 0
) or (
  DCGM_FI_DEV_ROW_REMAP_FAILURE > 0
) or (
  # XID in fatal bucket — using an auxiliary recording rule that
  # joins DCGM_FI_DEV_XID_ERRORS against a bucketing table
  dcgm:xid_in_fatal_bucket > 0
)
```

Routing: page the hardware-on-call rotation. Drain the GPU out of the pool via the cluster scheduler; take it out of the model-serving ingress; do not clear the alert until the hardware is remediated or replaced.

### Alert 2 — ECC single-bit rate (ticket, not page)

Non-zero SBE rate accumulating. ECC corrected the read; the user did not see corruption; but the GPU is a candidate for replacement.

```promql
increase(DCGM_FI_DEV_ECC_SBE_VOL_TOTAL[1h]) > 10
```

Threshold is fleet-specific — anchor to your card type's tolerance from the hardware vendor's guidance. 10 per hour on an H100 is likely a GPU trending toward failure; 10 per hour on an older A100 is possibly within normal bounds. Ticket, not page.

### Alert 3 — thermal throttling (page under sustained, ticket under transient)

```promql
# Page: sustained throttle > 1% of wall-clock for 10 minutes
(
  increase(DCGM_FI_DEV_THERMAL_VIOLATION[10m]) / (10 * 60 * 1e6) > 0.01
)
```

Thermal throttling has a direct TPOT impact that will not surface clearly in engine-side metrics (it reads as "TPOT drifted 3% on this one node"). Catching it at the GPU signal routes correctly — environment / facilities / cooling — rather than engine / workload. Pair with the host's ambient-temperature signal if available.

### Alert 4 — PCIe bandwidth saturation (ticket)

```promql
# Fraction of theoretical PCIe bandwidth saturated, averaged 5 minutes
(
  rate(DCGM_FI_PROF_PCIE_TX_BYTES[5m])
  + rate(DCGM_FI_PROF_PCIE_RX_BYTES[5m])
) / <theoretical_peak_bytes_per_sec_for_card> > 0.85
```

Replace the denominator with the card's theoretical PCIe peak (PCIe gen × lanes × bytes/lane/sec). Sustained > 85% means the PCIe path is the bottleneck; investigate whether the workload is memory-pinned-bad or whether the host-to-GPU data path has a hardware issue.

### Alert 5 — NVLink error accumulation (ticket, escalating to page)

```promql
# Ticket when CRC errors accumulate; page when replay errors accumulate above threshold
(
  rate(DCGM_FI_DEV_NVLINK_CRC_DATA_ERROR_COUNT_TOTAL[5m]) > 0
) or (
  rate(DCGM_FI_DEV_NVLINK_REPLAY_ERROR_COUNT_TOTAL[5m]) > <threshold>
)
```

Tune thresholds to vendor guidance. Non-zero replay errors on multi-GPU workloads correlate with specific TTFT regression patterns — the tensor-parallel workload is partially-degrading.

### Alert 6 — GPU appears idle while workload is scheduled (ticket, workload-side)

```promql
# SM active < 5% while engine-side queue > N
(
  avg_over_time(DCGM_FI_DEV_SM_ACTIVE[5m]) < 0.05
  and on (pod)
  vllm_num_requests_waiting > 10
)
```

This one is crucial for the bad-GPU vs. bad-workload distinction. If the queue is deep but the SMs are idle, the workload is blocked — PCIe, retrieval, admission misconfiguration — not GPU-bound. Route to the engine / platform team, not the hardware team.

### Alert 7 — HBM pressure (page before OOM)

```promql
DCGM_FI_DEV_FB_USED / DCGM_FI_DEV_FB_TOTAL > 0.97 for 5m
```

Leading indicator of the engine OOMing. On vLLM and similar engines, the KV-cache budget is derived from this; running close to the limit causes preemptions and admission rejects. The alert is often accompanied by the preemption-rate alert from chapter 02 recipe 4.5; cross-reference them in Grafana.

### Alert 8 — power draw outside expected envelope (ticket)

```promql
# Power draw < 50% of expected floor for the workload class
avg_over_time(DCGM_FI_DEV_POWER_USAGE[15m])
< on (card_type) group_left power_floor_watts
```

Power draw on a serving workload should be in a known band; a GPU drawing much less than expected is often scheduling-starved or thermally-capped. Not an urgent page but a useful fleet-trend signal.

## 5. Composing GPU telemetry with engine-side telemetry

The *routing* value of this chapter is composing the GPU signal with the engine-side signal from chapter 02. A few canonical compositions worth having in the on-call's pocket.

### Composition A — TTFT regression with no engine-queue change

"TTFT p95 degraded from 400ms to 900ms. Queue depth, admission reject, cache hit rate all unchanged."

Look at:

- `DCGM_FI_DEV_THERMAL_VIOLATION` increase in the time window — thermal throttle.
- `DCGM_FI_PROF_DRAM_ACTIVE` — HBM bandwidth saturation.
- `DCGM_FI_DEV_SM_ACTIVE` — GPU occupancy change?
- XID codes raised on the affected nodes.

If any of those moved, the cause is GPU-side, not engine-side.

### Composition B — preemption rate spike

"vLLM preemption rate climbed from 0.2% to 5%."

Look at:

- `DCGM_FI_DEV_FB_USED` — KV cache memory pressure.
- SBE rate — memory errors pushing KV-cache evictions.
- NVLink replay / CRC — bad interconnect degrading large batches.

Preemption rate without a hardware signal is an engine / workload problem; with one, it is a hardware incident.

### Composition C — stall rate spike

"Streaming stall rate jumped on one availability zone."

Look at:

- Thermal violation across all GPUs in that AZ — rack cooling event.
- NVLink errors — fabric problem.
- Power violation — datacenter power envelope issue.

Stall rate is often the *first* symptom the engine-side sees; the GPU-side telemetry is where you find the actionable signal.

### The dashboard pattern

A "fleet health" dashboard that overlays GPU and engine signals on the same time axis:

- Top row: TTFT p95, TPOT p95, stall rate (engine-side, from chapter 02).
- Middle row: SM active, HBM used, DRAM active, thermal violation (DCGM).
- Bottom row: XID event stream, ECC single-bit rate, NVLink error rate.

When the top row moves, the on-call scans down. 80% of inference incidents resolve at that scan.

## 6. Capacity planning and MIG — the operator surface

DCGM also feeds capacity planning from `mod-106`. Three specific signals the planning math consumes:

- **Sustained SM-active distribution across the fleet.** Your realized utilization vs. the fleet size. Low utilization with capacity complaints means scheduling is wasting capacity; high utilization with headroom complaints means you need more fleet.
- **HBM headroom distribution.** How close to the ceiling the KV cache is running. The capacity math treats this as a tail-risk signal.
- **Power-draw distribution vs. the TDP.** A fleet drawing well below TDP at peak has scheduling headroom; a fleet near TDP is already at the hardware limit even if SM-active is not.

On **MIG-partitioned fleets** (H100/H200/B200 cards split into smaller GPU slices), the DCGM surface emits per-slice metrics. The capacity planning joins to the slice, not the card, and the alerting fires on the slice if the workload is allocated to the slice. Reference the [MIG user guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/) for the slice-to-field mapping.

## 7. The peer boundary — what DCGM does and does not give you

DCGM is a *management* and *telemetry* surface. It tells you the GPU's state as the driver sees it. It does not tell you what any specific kernel was doing, which CUDA stream was blocked on what, or how a particular matmul kernel's memory access pattern lined up against the L2 cache.

Those deeper questions are at the `ai-infra-performance` peer, who uses:

- **NVIDIA Nsight Systems** — system-wide tracing and profiling, correlates CUDA kernel execution against CPU activity and interconnect.
- **NVTX (NVIDIA Tools Extension)** — in-application markers that annotate the Nsight timeline.
- **CUPTI (CUDA Profiling Tools Interface)** — the low-level API Nsight sits on.

Chapter 07 covers the peer boundary in detail. The practical rule for the AI SRE:

- **If the question is "is this GPU healthy?"** — DCGM answers it. You own it.
- **If the question is "is this workload using the GPU well?"** — the engine metrics plus DCGM utilization answer it at the first level. You own that.
- **If the question is "why does this specific kernel underperform?"** — Nsight / NVTX answers it. The performance peer owns that; your job is to produce the trace artifact (via Nsight-on-production-sampled techniques) and route it to them.

Most incident investigations stay above the Nsight line. When they cross — a model that is 20% slower than its benchmark for no clear reason — the peer hand-off is the right move.

## 8. One worked example — a quiet TPOT regression

Concrete example of how the full composition pays off. The incident shape:

- **Day 1 09:00.** Prometheus alert fires: TPOT p95 on `gpt-4o-mini-equivalent-local-model` has drifted from 42ms/token (28-day baseline) to 48ms/token over a 48-hour window. No release in the window; no traffic-mix change in the gateway logs; no cache-hit rate change.
- **09:05.** On-call opens the engine-state dashboard. Preemption rate normal. Admission reject rate normal. Queue depth normal.
- **09:08.** On-call opens the GPU health dashboard. `DCGM_FI_DEV_THERMAL_VIOLATION` has climbed on 6 out of 48 nodes. The 6 nodes are all in one rack.
- **09:12.** Cross-reference with facility inventory: that rack has a cooling-coil issue flagged by the datacenter team as "being monitored." Ambient temperature in the aisle is 4°C above the rest.
- **09:20.** Drain the 6 affected nodes; repopulate from the pool; TPOT p95 recovers within one scrape window.
- **09:30.** Facilities team invoked; cooling-coil repair scheduled for the maintenance window.

Without DCGM telemetry, this incident reads as "the model got slower; investigate the engine" — a path that leads to the inference-engine team, hours of engine-side investigation, and no finding. With the telemetry composed into the on-call workflow, the right diagnosis arrives in twelve minutes and the mitigation is immediate. The GPU surface is often the first *actionable* signal in an inference-latency incident even when the symptom is not GPU-shaped.

## Summary

- **Five families of GPU signal** — utilization (SM active, tensor-pipe active), memory (HBM, ECC, DRAM active), interconnect (PCIe, NVLink, replay / CRC), thermal / power, health / error (XID, row-remap).
- **XID codes are the hardware-health vocabulary.** Partition into fatal / workload / ticket buckets; alert off the bucket, not the raw code.
- **DCGM-aware alerts distinguish bad-GPU from bad-workload.** Hardware-fatal pages; thermal / NVLink / PCIe / HBM-pressure tickets; idle-with-queue alert routes to engine side.
- **Compose with engine telemetry.** TTFT regression + thermal violation = facilities; preemption spike + HBM pressure = capacity; stall rate + NVLink CRC = fabric. Overlay on one dashboard.
- **MIG partitions the surface.** Per-slice metrics; align labels for the join; capacity planning works on the slice.
- **Peer boundary.** DCGM answers "is this GPU healthy?" and "is this fleet utilized?" Nsight / NVTX / CUPTI at the performance peer answer "why does this kernel underperform?" The AI SRE escalates when the question crosses the line, not before.
- **The quiet regressions are where the signal pays off.** Thermal throttling and HBM errors show up as 3–5% drift in user-facing metrics; without the DCGM composition they're hours to diagnose.

The next chapter lifts up from the metric layer to the LLM-trace observability platform — Langfuse, Phoenix, LangSmith — and reasons about when a dedicated trace platform earns its keep vs. staying with OTel + Tempo / Jaeger.
