# Exercise 02 — Mixed-Workload Capacity Forecast Worksheet

**Estimated effort:** 4 hours · **Output:** one Markdown forecast (~2000 words) and one spreadsheet with per-class arithmetic and sensitivity analysis

## Objective

Produce a one-quarter capacity forecast for a mixed-workload AI fleet that you serve (or can plausibly represent). The forecast must decompose by workload class, project rate and shape forward with a defensible growth model, compute per-class GPU cost in engine-native units, aggregate with explicit headroom multipliers, and translate to a concrete node-pool shape. It must include a prefix-cache-hit-rate sensitivity analysis and a risk section.

This is the module's signature artifact. It is longer than exercise 01 and benefits from the Little's-Law groundwork from there — pull your workload classification into this exercise as the starting point.

## Prerequisites

- Chapter 01 (queueing-theory base) and chapter 02 (mixed-workload forecast).
- Exercise 01 completed — you have at least one workload classified and modelled.
- Access to (or plausible numbers for) a fleet serving at least three distinguishable workload classes. If you cannot source real traffic, use a composition of chat + RAG + batch from a documented public example and attribute.
- Access to engine-side throughput numbers for the model(s) you serve on the hardware you use. If measured numbers are not available, cite a public benchmark and bracket.

## Problem statement

Author a quarterly capacity forecast as a Markdown document, structured in six sections, with the arithmetic in an attached spreadsheet.

### Section 1 — Workload decomposition

Identify at minimum three workload classes the fleet serves. For each, state:

- The user-visible surface (product or internal service name).
- The ingestion-time label used to identify the class (header, route, OTel attribute).
- A prompt-length distribution (p50, p95, p99) and a completion-length distribution (p50, p95, p99).
- A prefix-cache hit-rate estimate with the rationale (template-driven, content-driven, session-driven — see chapter 02 §2.3).
- A time-of-day shape sketch (peak factor over mean; US-business-hours vs. global vs. batch).

### Section 2 — Growth projection

For each class, project the request rate one quarter forward:

- Fit a simple growth curve to the last 12 weeks of request-rate data (or state the growth assumption with defence if historical data is unavailable).
- State the external forces that could override the trend: product launches, campaigns, rate-limit changes.
- Produce a point projection for mean hourly RPS and peak hourly RPS at the quarter's end, with a ± range.

### Section 3 — Per-class GPU cost

For each class, compute the expected per-request GPU cost in engine-native units:

- Effective prefill tokens = `(1 - hit_rate) × prompt_length` (mean).
- Decode tokens = completion_length (mean).
- KV-cache bytes per request = per-token-KV-footprint × (prompt_length + completion_length). Use the engine-peer-provided per-token KV-footprint for your model, quantisation, and attention variant (MQA / GQA / MHA).

Multiply by projected RPS → aggregate GPU-seconds per second (prefill and decode separately).

### Section 4 — Fleet-level aggregation

Sum the per-class GPU-second budgets. Then apply the chapter 02 §4 multipliers:

- **Peak-to-mean factor.** Document the factor and its justification.
- **Utilisation headroom.** Target `ρ ≤ 0.70` → multiplier `1 / 0.70 = 1.43×`.
- **Failure-domain headroom.** AZ-loss / rack-loss, state your assumption, compute the multiplier.
- **Rolling-update overhead.** A few percent, cited.

Produce `GPU_peak` as the aggregated requirement, with each multiplier called out separately.

### Section 5 — Node-pool shape

Translate `GPU_peak` into a concrete node-pool shape:

- GPU generation(s) proposed (consistent with your fleet; chapter 03 covers mixed-generation considerations).
- Instance type / SKU on your cloud provider.
- Node count (ceiling of `GPU_peak / GPUs_per_node / TP_degree × replicas_per_node`).
- Tensor-parallel and pipeline-parallel degree if applicable.
- Pool composition: baseline (on-demand or reserved), burst (spot or on-demand headroom), batch (preemptible or shared).
- Reservation / capacity block details and renewal window.

### Section 6 — Sensitivity analysis and risks

Produce a table with sensitivity to at least these levers:

- Prefix-cache hit rate drops by 50% (template-change scenario).
- Completion length rises 30% (longer-reasoning model version bump).
- Peak factor rises 20% (geographic concentration shift).
- Growth is 2× the point projection (bull case).

For each row, compute the implied `GPU_peak` and the node-pool delta. End with a prose risk section that names the three most likely ways this forecast will be wrong.

## Requirements

1. **At least three workload classes.** Decomposition is the exercise.
2. **Distributions, not means, in section 1.** p50 and p95 minimum per prompt and completion length.
3. **Explicit prefix-cache hit-rate assumption with rationale.**
4. **Growth projection with ± range**, not a single point.
5. **Per-class GPU-second arithmetic in the spreadsheet.** Reader can inspect.
6. **Each headroom multiplier called out separately**, with its justification.
7. **Node-pool shape specifies all six fields** in chapter 02 §5.1 (generation, SKU, node count, replicas/node, TP/PP, pool role).
8. **Sensitivity table with four rows minimum.**
9. **Risk section names three specific failure modes.** Generic "the forecast could be wrong" does not count.

## Starter guidance

- The spreadsheet does the heavy lifting; the Markdown document narrates. The arithmetic should be in one place and traceable; the narrative should be readable without the spreadsheet open.
- For a first pass, workload classes can be coarse: chat / RAG / batch. If your fleet has more structure (multiple products, multiple tenants), use the structure that drives different capacity math.
- Prefix-cache hit rate is the lever that most-often breaks a naive forecast. Spend time on the rationale; a single sentence like "we assume 65% because that is what we measured in week 42 and the template is stable" is adequate.
- For throughput numbers, name the source: measured in-production, measured in-staging, from the engine peer's benchmark, from MLPerf, from a vendor-published number. Cite in the spreadsheet.
- The node-pool shape does not have to match cloud SKUs you have access to. The exercise's output is defensibility; if the right SKU is "p5en.48xlarge with 8× H200 141GB" on AWS, name it even if your production is on GCP's a3-ultragpu-8g.

## Acceptance criteria

- Markdown document is ≤ 2500 words.
- Attached spreadsheet has per-class arithmetic reproducing section 3, aggregation reproducing section 4, and sensitivity reproducing section 6.
- At least three workload classes with distribution shapes, growth projections, and hit-rate assumptions.
- Per-class GPU cost computed in engine-native units; translated to GPU-seconds via a cited throughput source.
- Each of the four headroom multipliers is called out with a one-line justification.
- Node-pool shape specifies all six required fields.
- Sensitivity table has four rows showing the delta in `GPU_peak` and node count.
- Three specific named risks in the final paragraph.
- A reviewer unfamiliar with your fleet can read the document in 20 minutes and understand the quarter's capacity contract.

## Stretch goals

- **Multi-region forecast.** Repeat sections 3–5 per region. Note the capacity reservations that are per-region (every major cloud) and the reservation / burst mix per region.
- **Cost accounting.** Translate the node-pool shape into a monthly cost estimate under reasonable pricing (vendor-published on-demand / reservation rates with appropriate asterisks). Compare to the previous quarter's actuals if available.
- **Forecast-error tracking.** If you have a previous-quarter forecast, compute forecast error on rate, hit rate, completion length, and peak factor. Which input was the most wrong? Commit to reducing that gap for this quarter's forecast.
- **Simulator calibration.** Run your forecast's inputs through a serving simulator (vidur, LLMServingSim) and compare the simulator's TTFT / TPOT predictions against the forecast's SLO targets. Where they disagree, decide which to trust and document.
