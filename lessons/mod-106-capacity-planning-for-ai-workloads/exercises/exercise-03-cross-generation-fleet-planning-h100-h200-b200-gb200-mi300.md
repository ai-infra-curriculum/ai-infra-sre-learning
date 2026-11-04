# Exercise 03 — Cross-Generation Fleet Planning (H100 / H200 / B200 / GB200 / MI300)

**Estimated effort:** 3 hours · **Output:** one Markdown decision memo (~1500 words) with a cross-generation comparison table and a rollout plan

## Objective

Produce a defensible cross-generation planning memo for one workload in your fleet. Choose a hypothetical (or real) rollout: migrating from the current GPU generation to a candidate next generation, in whole or in part. The memo must separate capacity effects from performance effects, enumerate the cross-generation observability pitfalls, specify the six-piece swap checklist (chapter 03 §2.1), and recommend a rollout path with explicit failure-domain and SLO-equivalence choices.

The exercise exists to force the specific discipline of **translating** across generations rather than treating a hardware swap as a replace-in-place operation. A good deliverable is read by a reviewer who disagrees with the recommendation and can still point to where the disagreement lives.

## Prerequisites

- Chapter 03 of this module.
- One workload with a known profile (the output of exercise 01 or 02 works). The workload's bottleneck (prefill-bound, decode-bound, mixed) determines which generation benefits most.
- Current public benchmark data for the generations under consideration. MLPerf Inference, vLLM public benchmark leaderboards, cloud-provider SKU documentation, or engine-peer-published numbers. **Do not derive from TFLOPs specs.**

## Problem statement

Choose a workload and a candidate migration path. Acceptable examples:

- H100 → H200 for a decode-bound chat workload.
- H100 → B200 for a prefill-bound RAG workload.
- H200 → GB200 NVL72 for a 70B+-model workload that could benefit from rack-scale tensor parallelism.
- H100 → MI300X for a workload with a strong reason to move to AMD (cost, availability, strategic diversification).

Author the memo in six sections.

### Section 1 — The workload

One paragraph. Name the workload, classify its bottleneck (chapter 01), and cite the forecast number for peak demand (from exercise 02 or a stated number).

### Section 2 — The candidate generation(s)

Half a page. Describe the candidate generation, the specific SKU you would consume, the cloud-provider or on-prem availability shape (region, reservation path, lead time), and the software stack delta (CUDA version, cuDNN version, ROCm version, engine support matrix).

### Section 3 — Capacity effect vs. performance effect

The core of the memo. For the specific workload's bottleneck, compute:

- **Replica-equivalence ratios** per metric from chapter 03 §4.1: prefill-token-throughput ratio, decode-token-throughput ratio, KV-cache-budget ratio, composite capacity multiplier. Cite every number. Do not report a scalar replica-equivalence; report the ratio per metric.
- **Performance delta.** Does TTFT / TPOT improve at the same load, and by how much? Cite.
- **Classification.** State whether the swap is primarily a capacity event, primarily a performance event, or a composite. Name which operational decisions each triggers.

### Section 4 — The six-piece swap checklist

Apply chapter 03 §2.1 to this swap:

1. **Measured throughput curve** on the new generation — cite the source and note whether it is a public benchmark or a fleet-internal measurement. If internal data is not yet available, state what measurement must happen before the rollout starts.
2. **KV-cache budget per replica** on the new generation — how many in-flight sequences at the operating context length.
3. **Revised forecast** — pull the GPU count required from the exercise 02 forecast with the new throughput curves. State the delta.
4. **Revised SLO operating point** — do you tighten, hold, or bank the headroom? Commit to an answer with reasoning.
5. **Revised autoscaler thresholds** — what HPA target queue depth changes? If using TTFT-based autoscaling, does the target move?
6. **Revised failure-domain plan** — especially relevant for NVL72 racks; the failure-domain shape changes.

Each piece is a one-paragraph answer.

### Section 5 — Observability pitfalls

List the four cross-generation observability pitfalls (chapter 03 §3) and state which are relevant to this swap:

- GPU-util cross-generation incomparability.
- Per-token latency without context-length stratification.
- Prefix-cache behaviour change with KV-cache-budget change.
- Failure-domain shape change.

For each relevant pitfall, name the specific dashboard or metric panel that must be updated before the rollout begins.

### Section 6 — Rollout path and recommendation

Recommend one of:

- **Full migration** in one quarter, with a rollout calendar.
- **Partial migration** (e.g., migrate the RAG surface, keep chat on current generation), with the slice justified.
- **Hold** for the next cycle (not ready, allocation unavailable, software stack immature), with the trigger for revisiting.

Name the specific risks that could force you to walk back the recommendation (allocation does not land, engine support gap, observability pitfalls not mitigated in time).

## Requirements

1. **Replica-equivalence as a per-metric ratio, not a scalar.** Every number cited.
2. **Classification of the swap as capacity / performance / composite** with the operational decisions each triggers.
3. **The six-piece checklist is enumerated**, no skipping. If a piece is "out of scope for this memo, scheduled for quarter N+1," say so explicitly.
4. **Observability-pitfall mitigations named at the dashboard level.** Which panel updates, not a general statement.
5. **A recommendation that commits.** Full / partial / hold — not "it depends." Name the risks that would reverse it.

## Starter guidance

- Benchmarks age. Cite the date of the benchmark you pull from (e.g., "MLPerf Inference v5.0 datacentre submissions"). A number from a two-year-old benchmark should not drive a 2026 decision.
- The B200 / GB200 case is the most complex because the NVLink-5 fabric and NVL72 primitive change the planning math qualitatively, not just quantitatively. If you choose this, spend extra time on section 4 piece 6 (failure domain).
- The MI300X case requires evidence about engine support (vLLM on ROCm, SGLang on ROCm, TGI on ROCm) for your specific model at production-grade. Software-stack risk dominates hardware-spec gains for AMD migrations.
- The decision does not have to be adoption. "Hold for Q+1 because ROCm support for our MoE model is not yet at parity" is a complete recommendation.

## Acceptance criteria

- Markdown memo is ≤ 1800 words.
- Replica-equivalence table has at least three per-metric ratios (prefill, decode, KV-cache) with cited sources for each.
- Swap classification commits to capacity / performance / composite.
- All six swap-checklist pieces answered.
- Each relevant observability pitfall named with its dashboard mitigation.
- Recommendation is one of full / partial / hold with named reversing risks.
- A reviewer unfamiliar with the workload can read the memo in 15 minutes and understand the rollout proposal.

## Stretch goals

- **Cost-adjusted comparison.** Add a cost-per-request number on each generation using cloud-provider on-demand pricing (or your reservation rate). Note where the cheaper-per-request generation is also the higher-performance generation and where the two differ.
- **Multi-generation fleet operation.** Instead of a swap, model the steady-state operation of a mixed-generation fleet for the next three quarters. How are workloads routed to generations? How does a replica-equivalence failure (one generation under-capacity) rebalance to the other?
- **Rack-scale coherence experiment.** For the GB200 NVL72 case, model what a very-large-model deployment (e.g., 405B parameters with TP across the full rack) does to the forecast — one replica is 72 GPUs, the failure domain is the rack, the reservation calendar is in racks. How does the capacity math shift?
- **ROCm readiness checklist.** If MI300X is on the table, produce a readiness checklist for the software stack: engine, model, quantisation, tokenizer, observability, monitoring. What passes, what fails, what blocks adoption?
