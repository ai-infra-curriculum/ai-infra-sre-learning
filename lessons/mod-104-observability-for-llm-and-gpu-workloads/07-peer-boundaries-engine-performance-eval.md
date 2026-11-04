# Peer Boundaries — Engine, Performance, and Eval

## Why this chapter exists

Chapters 01–06 built the observability stack an AI SRE owns: OTel semantic conventions, Prometheus metrics, GPU telemetry, LLM-trace platforms, drift signals, structured logs. Along the way, each chapter marked spots where the work belonged to a peer at a different level on the ladder — the inference-engine team, the performance-engineering team, the eval-engineering team. Those marks were deliberate. The AI SRE at level 35 is a *reliability generalist* who integrates signals; the peers at adjacent levels are *specialists* who construct signals.

If the boundaries are not explicit, three predictable failure patterns appear:

1. **SRE reaches into the engine or the kernel.** The on-call starts editing a vLLM scheduler flag during an incident, the engine team finds out at the post-mortem, and the engine team's change-management discipline is corrupted silently.
2. **Peer rotations are handed the SLO contract.** The performance engineer, correctly identifying that a kernel is slow, decides to lower the TTFT SLO target — a decision that is the SRE's and the sponsor's, not theirs.
3. **Nobody owns the handoff artifacts.** The DCGM dashboard the SRE operates and the Nsight trace the performance peer operates do not join to each other; during an incident the two teams diverge instead of composing.

This chapter authors the three two-column contracts — SRE / inference-engineer, SRE / performance, SRE / eval — that avoid all three. It is the observability-specific specialization of the peer deferral map from `mod-101` chapter 02, the structural analogue to `mod-102` chapter 07 (engine-authoring peer boundary) and `mod-103` chapter 07 (eval-engineer peer boundary), and the capstone of this module.

The sibling curricula referenced below are:

- **`ai-infra-inference-engineer-learning`** — the inference-engine peer.
- **`ai-infra-performance-learning`** (or equivalent — the kernel / profiling peer at the `ai-infra-performance` role).
- **`ai-eval-engineer-learning`** at <https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning>.

Read the orientation chapter of each once; the boundaries work best when they are reciprocal.

## 1. Boundary A — the inference-engine peer

The inference-engine peer owns the serving runtime itself: vLLM, SGLang, TRT-LLM, Triton, NVIDIA Dynamo, or whatever next-generation serving engine your fleet runs on. From chapters 02 and 03, that peer owns the surface that *produces* most of the engine-side observability signal.

### Two-column contract

| AI SRE owns (consumer of the signal) | Inference-engine peer owns (producer of the signal) |
|---|---|
| Choosing which engine metrics compose into which SLI | The `/metrics` endpoint — which metrics are exposed, what they are named, what their units are |
| PromQL recipes against the engine-emitted metrics (chapter 02) | Instrumenting new counters / histograms when a new engine feature is added |
| Burn-rate alerts on SLIs computed from the engine metrics | Keeping the metric surface backward-compatible across engine versions, or flagging breaking changes |
| Grafana dashboards that overlay engine state with GPU state | Engine-side logging schema and severity levels |
| On-call routing when the engine's exposed metric indicates a workload issue | On-call routing when the engine's internal state indicates an engine issue (scheduler bug, cache corruption, admission regression) |
| Integration contract with chapter 01 OTel spans — which trace attributes the engine propagates | Context propagation across the engine's internal boundaries (admission → scheduler → prefill → decode → emission) |
| Fleet-wide load testing to produce baselines (chapter 02) | Engine-side benchmarks (throughput, goodput) producing the shape of the baseline |
| Capacity-planning math from `mod-106` | Engine configuration knobs that shift the planning math (max-batch-size, max-num-seqs, prefix-cache size) |

Every row on the left lands with the AI SRE. Every row on the right lands with the engine peer.

### The three canonical hand-offs

- **New engine metric requested.** The SRE needs a metric the engine does not currently emit (e.g., per-tenant queue depth, prefix-cache hit rate per prompt-length bucket). Correct move: file a ticket at the engine team with the SLI contract — "the SLI is X; I need the metric Y with labels Z; here is the PromQL it will feed." The engine team instruments; the SRE wires the consumption.
- **Engine-side metric regression.** The engine team releases a version that renames or drops a metric; the SRE's dashboard breaks. Correct move: the engine release should have flagged the break; the engine team restores the old metric as an alias for one release and the SRE migrates by the deprecation deadline. Codify this in the engine-release-process doc.
- **Suspected engine bug mid-incident.** The SLI is burning; the engine state dashboard shows something anomalous (queue climbing with no admission-reject climb, cache hit rate dropping in a way that doesn't match traffic). Correct move: the SRE engages the engine on-call rotation; both parties investigate along their columns. The SRE handles the reliability response (rollback, admission throttle, capacity expansion) while the engine team diagnoses the engine state.

Reference `mod-102` chapter 07 for the engine-release coordination template; this chapter extends that boundary into the observability surface.

### Where it drifts

- **SRE edits engine flags during incident.** Fix: incident playbook codifies which flags are reversible SRE-authored emergency levers (admission rate limit, routing weight) and which are not (scheduler / batching / preemption). Engine-peer-authored flags only touch via a documented incident procedure.
- **Engine team authors SLO targets.** Fix: the engine team's benchmark numbers inform the baseline; they do not set the sponsor-facing target. Chapter 02's PromQL recipes against engine metrics are SLO-facing; the SRE owns the target.

## 2. Boundary B — the performance / kernel peer

The `ai-infra-performance` peer owns the deep-performance surface: CUDA kernel design and tuning, memory-access patterns, L2 cache behaviour, warp scheduling, Tensor Core utilization, NCCL collective tuning, NVTX annotation of the engine's hot paths. From chapter 03, the peer's tools are **NVIDIA Nsight Systems** (system-wide tracing), **Nsight Compute** (kernel-level metrics), **NVTX** (in-application markers), and **CUPTI** (the low-level profiling API Nsight sits on).

The AI SRE's DCGM surface (chapter 03) answers "is this GPU healthy?" The performance peer's Nsight surface answers "why does this specific kernel underperform on this GPU?" These are distinct observability surfaces that connect only where the AI SRE's data acts as the *trigger* for the performance peer's work.

### Two-column contract

| AI SRE owns (fleet-level signal) | Performance peer owns (kernel-level signal) |
|---|---|
| DCGM telemetry — SM activity, HBM pressure, interconnect saturation, thermal, power, XID codes | Nsight Systems traces, Nsight Compute kernel metrics, NVTX annotations |
| Alerting on hardware-bucket signals (chapter 03 alerts 1–7) | Kernel-level performance analysis — occupancy, memory-coalescing, Tensor Core usage, achieved vs. theoretical FLOP/s |
| Detection of fleet-wide regression (chapter 02 dashboard panels, drift detection on TPOT) | Root-causing a kernel-level slowdown — register spills, bad launch configurations, suboptimal collective patterns |
| Production-sampling methodology for performance traces — when to capture, how often, how much overhead is acceptable | Analyzing captured traces, authoring kernel-tuning recommendations, validating with before/after benchmarks |
| Communicating performance incidents to the business (SLO burn framing) | Producing the engineering artifact — "this kernel is 20% slow because of X; here is the diff; here is the benchmark" |
| Routing a performance incident to the peer when DCGM + engine signal does not resolve | Engaging when a workload's performance is a function of the specific kernel path, not the fleet state |
| Capacity planning (mod-106) using baselined utilization | Benchmark artifacts that feed capacity planning — what shape of workload achieves what utilization |

### The three canonical hand-offs

- **Hand-off 1 — Model is 20% slower than its benchmark, no hardware or workload signal.** The SRE has looked: DCGM clean, engine metrics clean, drift signals clean. TPOT is nonetheless 20% worse than the model's published benchmark on this hardware. Correct move: file a ticket at the performance peer with the measurement envelope — workload shape, hardware, engine version, baseline reference, current measurement, absence of hardware signal. The peer engages Nsight, identifies the kernel path, proposes a fix. Peer's investigation lands as a merge request to the engine; the SRE's job is complete when the fleet-wide metric recovers.
- **Hand-off 2 — Capacity planning asks for a new GPU shape.** `mod-106` planning math wants to know "how does this workload perform on H200 vs. H100?" The fleet is on H100; a small H200 pool exists for pilot. Correct move: performance peer runs a benchmark suite under production-like load on both; SRE operates the measurement pool and ensures fleet isolation; peer produces the per-shape benchmark artifact; SRE consumes it into the capacity model. The peer is the measurement craft; the SRE is the measurement environment.
- **Hand-off 3 — Nsight capture in production.** The SRE needs a production-sampled Nsight trace to characterize a slow-path pattern. Correct move: the performance peer defines the capture methodology (which workload, what overhead cap, what redaction over kernel names if they are sensitive); the SRE operates the capture (deploys the Nsight agent to the sampled pool, enables for a configured window, routes the output to the peer's storage); the peer analyzes. Neither tries to do the other's part.

### Where it drifts

- **SRE tries to read Nsight traces.** The peer's tool, the peer's craft. The SRE's job is to produce the trace, not interpret it. If the SRE develops the skill to interpret it, that is useful, but it is not the SRE's primary responsibility.
- **Performance peer alerts on DCGM.** DCGM is fleet-level; the peer's alerts, if any, should be on their own kernel-benchmark pipelines. If the peer builds a DCGM-alert layer, coordinate with the SRE so it doesn't duplicate the chapter 03 alert set.
- **Nsight overhead in production.** Nsight Systems in default-settings has non-trivial overhead; running it continuously across the fleet is not acceptable. The capture discipline is the peer's responsibility, but the AI SRE owns the fleet-side cap (per-agent CPU / memory, per-fleet aggregate bandwidth to the capture store). Enforce the cap.

### Named channels and ticket template

Ticket from SRE to performance peer when hand-off 1 fires:

- **Title.** One-line symptom (workload X, N% slow vs. benchmark).
- **Measurement envelope.** Model, engine version, hardware, prompt-length distribution, QPS, duration of measurement.
- **Ruled out.** DCGM clean, engine metric clean, drift clean, config unchanged in the window. Include links.
- **Request.** Nsight production sample + analysis.
- **Response hold.** What the SRE is doing in the meantime (routing weight shift, per-request overhead cap).

## 3. Boundary C — the eval-engineer peer

The eval-engineer peer owns the quality-signal surface: judge models, eval sets, calibration, pass-rate pipelines, dataset curation, release-gate eval suites. From chapters 01, 04, and 05 — the trace-overlay attributes for quality scores, the LLM-trace platform, the drift signals on output distribution and judge agreement — the AI SRE *consumes* a lot of eval-authored signal.

This boundary is extensively covered in `mod-103` chapter 07. The specialization here is **the observability-side boundary** rather than the SLO-contract-side.

### Two-column contract (observability-specific)

| AI SRE owns (observability of the signal) | Eval peer owns (construction of the signal) |
|---|---|
| OTel span attributes that carry score verdicts | The judge pipeline that emits the verdicts |
| Prometheus counters aggregating verdicts for the quality SLI | Judge-vs-human agreement calibration |
| Drift alerts on judge agreement (chapter 05 alert 4) | Investigation of judge drift when the alert fires |
| LLM-trace platform deployment reliability (chapter 04) | Dataset and annotation workflow inside the platform |
| Retention / redaction / audit policy on content in the platform | Content curation for the release-gate eval sets |
| Observability-side incident response when the quality SLI burns | Diagnosis of whether the signal or the model is the cause |
| Alerting on drift pipeline staleness | Health of the judge pipeline itself |

### The two-way hand-off worth naming

**Hand-off: eval peer publishes a new judge version.**

- Peer announces via the signal-change channel from `mod-103` chapter 07 section 4.
- SRE schedules the baseline refresh — the quality SLI's baseline is re-measured against the new judge.
- SRE updates the SLI definition's "signal source and version" field.
- Co-review happens at the next SLO ritual.

**Hand-off: SRE observes judge-agreement slip (chapter 05 alert 4).**

- SRE pages primary for routing; eval peer is notified as co-receiver; auto-rollback automation keyed on this judge is paused.
- Eval peer investigates calibration, prompt revisions, underlying model version.
- Peer and SRE co-decide whether to (a) accept the new judge as a baseline shift, (b) recalibrate, or (c) replace.

### Where it drifts

- **SRE edits the judge prompt.** No. The judge is the peer's craft.
- **Eval peer tunes Alertmanager rules.** No. The reliability contract is the SRE's.
- **Neither owns the content retention policy.** This is a shared ownership failure — in the two-column contract, retention / redaction / audit *is* the SRE's. Explicitly, in writing.

## 4. Cross-boundary scenarios — three worked examples

### Scenario A — TTFT burn with DCGM-clean, engine-metric-clean, Nsight-flagged

The SLI burns; chapters 02 and 03 show no actionable signal; drift signals clean. The SRE's investigation stops here. Scenario routes to hand-off 1 of the performance peer boundary. Nsight analysis identifies a kernel path where a recent engine-version bump introduced a worse launch configuration. Engine team merges the fix; SRE operates the rollout; SLI recovers.

The composition walked: SRE (observability) → performance peer (kernel diagnosis) → engine peer (code fix) → SRE (rollout and SLI closure). Four participants, each inside their column.

### Scenario B — Quality SLI burn with input-drift + judge-drift suspected

The hallucination SLI burns. Chapter 05 alert 1 (input drift) fires at the same time. Drift alert 4 (judge agreement) is not firing but the most recent agreement number was close to threshold.

Correct composition:

- SRE declares incident; holds the SLO contract; investigates on the SRE column (recent deploys, retrieval index version, traffic source).
- Eval peer investigates on the eval column (judge calibration run on the sampled window, judge-vs-human spot check, pipeline version).
- Decision at co-review: if the eval peer confirms judge drift concurrent with input drift, the signal is unreliable; retro-adjust the SLI over the overlapping window per `mod-103` chapter 05 section 7. If the eval peer confirms the judge is stable, the SRE investigates traffic further.

### Scenario C — Vendor-hosted model version bump, output-drift fires

The provider silently bumps `gpt-4o` under the alias, no notification. Output embedding drift fires (chapter 05 alert 2). Input drift does not. DCGM clean (this is a provider-hosted model; DCGM is irrelevant). Engine metric only reflects the gateway, not the provider.

Composition:

- SRE investigates the `gen_ai.response.model` label — chapter 01's instrumentation captures the served-model version. The label value has shifted.
- SRE escalates to the vendor-risk workflow from `mod-107` — the version change is now a vendor event.
- Eval peer runs the production sample against both the old pinned version and the new version on a known-good calibration set; computes the quality delta.
- Co-decision: pin to the old version via the provider's version-pinning API (if available) or absorb the new version with an updated SLO baseline (if quality delta is acceptable).

The first *observability signal* — the `gen_ai.response.model` label — was what routed the incident correctly. Without it, the scenario reads as "quality got worse; nobody knows why."

## 5. Peer-review cadence and cross-training

The boundaries hold because people on both sides periodically read each other's work. Four cross-training mechanisms:

- **SLO ritual attendance.** Eval peer is a standing attendee on quality-SLI agenda items (`mod-103` chapter 06 section 6). Performance peer is a standing attendee when fleet-utilization data is reviewed.
- **Reciprocal on-call shadow.** Once a quarter, an SRE attends an engine-team incident retrospective; an engine engineer attends an SRE incident. One session per direction.
- **Shared dashboards.** The engine-state dashboard from chapter 02 is linked from the engine team's internal dashboard; the DCGM dashboard is linked from the performance team's. Each side sees what the other sees.
- **Joint runbook authoring.** The playbooks for the cross-boundary scenarios (section 4) are authored jointly; both parties sign off; both parties review on cadence.

The signal that the boundaries are healthy: when an incident of scenario A / B / C shape happens, the parties know who to engage without a meeting. The signal that they are drifting: an incident retrospective includes "we spent 2 hours figuring out who to engage."

## 6. One rule-of-thumb per boundary

Fast mental tests for the SRE in the moment:

- **Engine boundary.** If the question is "which metric should I query?" — SRE. If the question is "why does the engine's internal state look like this?" — engine peer.
- **Performance boundary.** If the question is "is this GPU fleet healthy?" or "is this workload utilizing the fleet well at the aggregate?" — SRE. If the question is "why does this specific kernel underperform?" — performance peer.
- **Eval boundary.** If the question is "is the quality SLI burning and what does the burn mean for the user?" — SRE. If the question is "is the signal measuring what we think it is measuring?" — eval peer.

Three questions, three boundaries. When in doubt, say it out loud — "I think this is the engine peer's question" — and route accordingly.

## 7. Where the role grows next

This is the capstone of the observability module. The surface it authored — OTel spans, Prometheus metrics, DCGM telemetry, LLM-trace platform, drift signals, structured logs, peer boundaries — composes with the rest of the AI SRE track:

- **`mod-105` — non-deterministic failure diagnosis.** The observability surface from this module is the input to the next module's incident-forensics pattern library.
- **`mod-106` — capacity planning.** The utilization and goodput signals from chapters 02 and 03 feed the capacity math.
- **`mod-107` — vendor and model-provider risk.** The vendor-side instrumentation from chapter 01 (`gen_ai.system`, `gen_ai.response.model`) feeds the vendor-risk dashboards.
- **`mod-108` — incident response.** Every incident playbook references the dashboards and runbooks from this module.
- **`mod-111` — release engineering.** Canary analysis and progressive delivery reads the SLIs computed here.

The observability surface is load-bearing for everything after. If a reader finishes this module able to instrument an LLM request path, author the PromQL recipes, operate DCGM alerts, pick an LLM-trace platform, wire drift into reliability alerts, and route cross-peer boundaries without a meeting, the subsequent modules land on solid foundation.

## Summary

- **Three peer boundaries.** Inference-engine (producer of engine metrics; consumer, the SRE). Performance (owner of kernel-level analysis; the SRE routes to them when fleet-level signal doesn't resolve). Eval (owner of quality-signal construction; the SRE owns the observability-side reliability of the signal).
- **Two-column contracts** structure each. Signal construction on one side; reliability contract and consumption on the other. No overlap.
- **Hand-offs are named.** New metric request, metric regression, engine-bug co-investigation. Nsight production sample, capacity benchmark, hardware-vs-workload routing. Judge version bump, judge-drift incident, quality SLI burn with signal suspect.
- **Cross-training cadence.** SLO ritual attendance, reciprocal on-call shadow, shared dashboards, joint runbook authoring.
- **Three rule-of-thumb questions** — which metric vs. engine internal state; fleet health vs. kernel diagnosis; SLI burn vs. signal validity. Route accordingly.

This closes the module. The exercises walk the full stack against a workload you carry across all seven.
