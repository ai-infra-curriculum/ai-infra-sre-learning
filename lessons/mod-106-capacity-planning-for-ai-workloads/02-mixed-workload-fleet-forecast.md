# Forecasting GPU-Fleet Capacity for a Mixed Workload

## Why this chapter exists

Chapter 01 built the first-order queueing model for a single workload. Real fleets do not serve a single workload. A typical LLM platform at a company that has adopted generative AI more than one quarter ago serves **chat**, **retrieval-augmented generation (RAG)**, **tool-calling and agent** traffic, and **batch** (summarisation, document processing, offline eval) from the same node pool — often the same replicas. Each surface has a different prompt-length distribution, a different completion-length distribution, a different prefix-cache profile, a different time-of-day shape, and a different sensitivity to TTFT vs. TPOT. Add that the output of the forecast is not a number — it is **a node-pool shape**: how many GPUs of which generation on which instance type, with what headroom.

A good capacity forecast for a mixed workload must do five things:

1. Decompose historical traffic into labelled workload classes.
2. Project each class's rate and shape forward against a defensible growth model.
3. Estimate per-class GPU cost — in prefill tokens, decode tokens, KV-cache bytes, and prefix-cache share.
4. Aggregate into a fleet-level requirement.
5. Translate the fleet-level requirement into a concrete node-pool shape with explicit headroom and failure-domain assumptions.

This chapter walks through all five steps using concrete examples. The output is the artifact an AI SRE brings to a quarterly capacity review.

## 1. Workload decomposition

The first job is to get rid of the aggregate. A single `requests_per_second` gauge and a single `avg_prompt_tokens` gauge across the fleet are useless for capacity planning because the numerator mixes incompatible units. Decompose by **workload class** at the ingestion label.

### 1.1 The four canonical classes

- **Chat.** Interactive, streaming, single-turn-to-multi-turn, moderate prompts (200–2000 tokens with growing context across turns), moderate completions (50–500 tokens, heavy-tailed). TTFT is the user-visible signal; TPOT matters for longer answers. Prefix cache can be high if a system prompt is stable.
- **RAG.** Retrieval-augmented, usually single-turn, large prompts dominated by retrieved documents (2K–16K tokens, sometimes higher), moderate completions (100–800 tokens). Prefill-bound by construction. Prefix cache can be high for the system prompt and low for the retrieved-document portion (every request has different documents unless the retrieval set is small and hot).
- **Tool-calling / agent.** Multi-step within one logical "task," where each step is a call to the model that may invoke tools. Each step has moderate prompts that grow across steps (tool outputs are appended), small-to-moderate completions. The surface signal is *steps per task* which multiplies effective RPS for a given user-facing rate.
- **Batch.** Offline, non-interactive, latency-insensitive beyond a per-job deadline. Prompts and completions vary; the capacity role of batch is as the *shock absorber* — it runs when the interactive fleet is not saturated and pauses when it is (chapter 05 returns to this in the autoscaling section).

A production fleet has at least one of each. Many have sub-classes (chat has multiple products; batch has nightly, weekly, ad-hoc). Keep the decomposition at the resolution that drives different capacity math; going more granular than that is noise.

### 1.2 How to get the labels onto traffic

The label must be stamped at ingest, not inferred at query time. Options in increasing order of reliability:

- **Per-route gateway label.** The API gateway (Envoy, Istio, an LLM gateway like LiteLLM or Portkey) attaches a `workload_class` attribute to every request based on the route or an application-supplied header.
- **OTel GenAI attribute.** The OpenTelemetry GenAI semantic conventions (see `mod-104` chapter 01) support custom attributes; `gen_ai.workload.class` is a natural one. Stamped at application boundary, propagated through the trace.
- **Application-supplied.** The upstream service passes a known header the gateway respects.

The label must survive onto the engine's metrics for the cost-stratification step (section 3). If the engine's Prometheus metrics do not carry the label today, the first chunk of the quarter's capacity work is wiring the label through. This is not optional; without it the forecast is guessing.

## 2. Per-class demand projection

For each class, project the rate and shape forward against your horizon. Two horizons are useful:

- **One quarter.** Operational — this is the forecast that drives purchase orders, reservation renewals, autoscaler-limit changes, and the next load test's target RPS.
- **One year.** Strategic — this is the forecast that drives capacity-reservation negotiations, GPU-generation planning (chapter 03), and the vendor-risk conversation (`mod-107`).

### 2.1 Rate projection

For each class, fit a growth curve to the last 12–24 weeks of request-per-second data at p50 and peak (p95 of the hourly peak). The simplest useful model is a monthly compound growth rate inferred from the trend; "we grew 18% MoM over the last six months" is often accurate enough as the starting point, bracketed with a ± range.

Three forces make the simple growth model miss, and the AI SRE must account for them explicitly:

- **Product launches and seasonality.** New product surfaces land; marketing campaigns shift demand; the batch workload tracks a monthly report cycle. These are step changes, not trend changes. Track them in the forecast's risk section.
- **Model promotions.** A new model version may have different prompt-length and completion-length distributions. A model that answers more concisely reduces decode cost even at the same request rate. Chapter 05 of `mod-105` names the "prompt-cache cliff" as the failure mode; capacity planning sees the mirror of it as a *demand shift*.
- **Downstream rate-limit changes.** The product team changes a free-tier limit, a paid-tier limit, or a per-user quota. The forecast must track these because they are the *actual* cap on demand, not whatever extrapolation the trend produced.

### 2.2 Shape projection

For each class, extract from production the following distributions (not means):

- **Prompt length.** Histogram of input tokens per request. Not mean — the shape matters because prefill compute is linear and the p99 tail drives prefill batch-time variance.
- **Completion length.** Histogram of output tokens per request. Mean drives decode throughput; the p95/p99 tail drives per-request service-time variance (chapter 01 section 2.2).
- **Multi-turn depth.** For chat, the distribution of context-length-at-turn-N across turns within a session. For agent, the distribution of steps per task and tokens per step.
- **Prefix-cache-worthy fraction.** The share of input tokens that are a cacheable prefix (system message, tool descriptions, few-shot examples, retrieved documents repeated across users). This requires the engine's prefix-cache metrics instrumented at the right granularity.

A healthy practice: report shapes as p50 / p95 / p99 numbers, not just means. "Chat: prompt p50 = 320, p95 = 1800, p99 = 4200; completion p50 = 110, p95 = 540, p99 = 1900" is a useful line for a capacity review. "Average prompt is 600 tokens" is not.

### 2.3 Prefix-cache-hit-rate forecast

Prefix cache hit rate is the single parameter that most-often breaks a naive forecast. If the measured hit rate today is 65% and your forecast assumes hit rate stays at 65%, you need to be explicit about **why**:

- **Template-driven.** The system prompt and tool descriptions are long and stable. The hit rate will stay high until a product change rewrites them. The forecast carries a risk line for the template-change event.
- **Content-driven.** Many users send similar prompts (RAG against a small corpus with a long-tail distribution of queries). The hit rate will track user behaviour and should be re-forecast quarterly.
- **Session-driven.** Multi-turn chat where each turn appends to the same conversation. The hit rate scales with turn depth; a change in average session length moves it.

For the forecast, state the hit-rate assumption and the sensitivity. "If prefix cache hit rate drops from 65% to 30% (chapter 05 of `mod-105`'s prompt-cache cliff scenario), prefill compute requirement rises by `(1 - 0.30) / (1 - 0.65) = 2×`." A number with a lever and a direction, not a guess.

## 3. Per-class GPU cost

For each class, compute an expected GPU cost per request from the shape distributions. The cost is denominated in **engine-native units** — prefill tokens, decode tokens, KV-cache bytes — and translated to wall-clock GPU-seconds only after the engine's throughput curves are brought in.

### 3.1 The per-request cost formula

For one request with prompt length `P`, completion length `O`, and prefix-cache hit rate `h` (fraction of prompt tokens served from cache):

- **Effective prefill tokens** `= P · (1 - h)`.
- **Decode tokens** `= O` (prefix cache does not help decode; speculative decoding changes effective per-token cost — see chapter 01 section 4.3).
- **KV-cache allocated bytes** `≈ (P + O) · bytes_per_token_per_layer × num_layers × 2 (for K and V)`. The engine's per-token KV-cache footprint depends on model architecture, numeric format (FP16 vs. FP8 KV-cache), and attention variant (MQA / GQA / MHA share the K and V across heads at different ratios). Use the engine's measured per-token KV-cache footprint; do not derive it.

For the class, expected per-request cost is the probability-weighted integral of these quantities over the class's distribution. In practice, a mean-based estimate is close enough for a first-order forecast; the tail is captured separately by headroom (section 5).

### 3.2 Translating to GPU-seconds

The engine's **throughput curve** tells you how many prefill tokens per second and decode tokens per second a given replica sustains at a given operating point. The curve depends on:

- **Model and quantisation.** FP16 vs. BF16 vs. FP8 vs. INT4.
- **Hardware.** H100 vs. H200 vs. B200 vs. MI300X vs. previous generations. Chapter 03 covers the cross-generation translation.
- **Batch size.** Decode throughput per replica rises with batch size up to the KV-cache budget.
- **Context length.** Both prefill and decode get slower at long context (attention is O(N²) in prefill, KV-cache reads are O(N) in decode).

The AI SRE does not derive the throughput curve. The AI SRE **measures** the curve — on your model, on your engine, on your hardware, with your traffic shape — using the load-testing harness from chapter 06. Document the curve as a table or a function. Consume it in the forecast.

Per-class GPU-seconds is then:

- **Prefill GPU-s per request** = `effective_prefill_tokens / prefill_throughput`.
- **Decode GPU-s per request** = `decode_tokens / decode_throughput_at_working_batch`.

Sum across classes, multiply by projected rate, you have aggregate GPU-seconds per second (i.e., the number of fully-utilised GPUs required at mean load).

### 3.3 Which workload dominates

A common finding of this exercise is that the GPU-second budget is dominated by one or two classes. RAG with 8K-token prompts and 10% hit rate frequently consumes 60–80% of the prefill budget while being 20% of requests. Chat with 500-token completions can dominate the decode budget at a different RPS mix. **Say which class dominates, by how much, under what assumption.** This is the input to the GPU-generation choice in chapter 03 (decode-bound workloads benefit disproportionately from H200 bandwidth; prefill-bound workloads benefit from B200 tensor cores).

## 4. Fleet-level aggregation

Sum the per-class GPU-seconds-per-second into a fleet-level mean GPU count. Then add the non-ignorable pieces that are not covered by the mean:

- **Time-of-day peak.** The mean captures the 24-hour mean; the fleet must survive the peak hour. Multiply by `peak-hour-RPS / mean-hour-RPS`, which is typically 2–3× for an interactive workload with a geographic concentration, 1.5× for a workload distributed globally, and nearly 1× for a batch-dominated workload.
- **Time-of-week peak.** Weekly cyclicality is usually small enough to be absorbed in the daily peak factor, but for B2B workloads with weekday-only traffic the Monday morning spike can dominate.
- **Tail latency headroom.** Section 5 of chapter 01: running at `ρ = 0.95` means TTFT p99 blows up. To hold a tight TTFT p99, run the fleet at `ρ ≤ 0.70` at peak. The headroom multiplier is `1 / ρ_peak`; at 0.70 that is 1.43×.
- **Failure-domain headroom.** The fleet must survive the loss of one availability zone, one rack, one node pool — whichever is the declared failure domain. If the fleet is spread across 3 AZs, loss of one AZ loses 33% of capacity; headroom multiplier is `1 / (1 - 0.33) ≈ 1.5×`. Smaller failure domains need smaller headroom; larger ones need more.
- **Warm-start / rolling-update overhead.** A fleet doing weekly rolling model updates has some fraction of replicas not fully utilisable during the rollout window. Add a few percent.

These multipliers are not stacked blindly; some overlap. A defensible aggregation:

- `GPU_mean = Σ_classes (rate × per_request_GPU_s)`
- `GPU_peak = GPU_mean × (peak_factor) × (1 / ρ_target) × (1 / (1 - failure_domain_fraction))`

Document each multiplier with a one-line justification. "We run at 2.5× peak-to-mean because our workload is US-business-hours dominated; we target ρ ≤ 0.70 because our TTFT SLO is 2s at p99; we provision for loss of one AZ out of three." Those three lines are the forecast's defensibility.

## 5. Translating to a node-pool shape

A number ("we need 140 GPU-equivalents at peak") is not a capacity plan. A node-pool shape is the plan. A node-pool shape specifies:

- **GPU generation** (H100, H200, B200, GB200, MI300X — see chapter 03 for selection criteria).
- **Instance type / SKU** (p4de, p5, p5e on AWS; a3-highgpu / a3-mega on GCP; ND-H100-v5, ND-H200-v5 on Azure; or an on-prem SKU).
- **Node count** at the generation × instance-type intersection.
- **Replicas per node** (how many model replicas a single node hosts; a function of VRAM per GPU, tensor parallelism, and KV-cache budget).
- **Tensor-parallel degree / pipeline-parallel degree** for the model (an engine-side concern — see peer boundary in chapter 07).
- **Node pools.** A fleet is often split into pools — e.g., a reserved on-demand pool for baseline, a spot pool for burst, a disaggregated-prefill pool if the engine supports P/D disaggregation.

### 5.1 The arithmetic

GPU-count from section 4 → node count via:

```
nodes = ceil(GPU_peak / GPUs_per_node)
```

Then validate replicas-per-node by checking the model's memory footprint fits: `replicas_per_node × (model_weights_memory + KV_cache_budget + activation_workspace) ≤ node_VRAM`. If the model does not fit a single GPU, apply tensor parallelism and solve `replicas_per_node = ceil(GPUs_per_node / TP_degree)`.

### 5.2 The common gotchas

- **Networking is a capacity constraint.** For P/D disaggregation, KV-cache shipping requires high-bandwidth interconnect (RDMA, NVLink-across-nodes on GB200 / NVL72). The nominal GPU count is not reachable if the interconnect is the bottleneck.
- **Instance types are not uniform within a cloud.** AWS p5 and p5e have different per-GPU memory even on the same H100/H200 generation (p5: 80GB H100 or H200; p5e: 141GB H200). GCP a3-highgpu-8g (H100 80GB) and a3-ultragpu-8g (H200) differ. Azure's ND series similarly. Verify at the SKU level, not the generation level.
- **Spot capacity is variable.** A node-pool shape that counts on 30% spot capacity for burst must have a tested fall-through to on-demand when spot is unavailable (chapter 05 covers Karpenter patterns).
- **Capacity reservations have lead time.** Reserved instances, capacity blocks (AWS Capacity Blocks for ML, GCP dynamic workload scheduler), and bare-metal lease renewals have weeks-to-months of lead time. The quarterly forecast must feed the reservation calendar; the reservation renewal cycle must feed back into the forecast's input assumptions.

### 5.3 The deliverable shape

A one-page node-pool proposal looks like this (illustrative):

```
Fleet: eu-west, model-A, baseline + burst
  Baseline pool
    - 24 × a3-highgpu-8g (H100 80GB), 3-year CUD reservation
    - 3 replicas per node, TP=2, decode-batch 128
  Burst pool
    - Up to 16 × a3-highgpu-8g, on-demand
    - Karpenter-managed; scale target: queue_depth p95 > 20
  Rationale
    - Peak demand projected 140 GPU-eq at Q2-end
    - Baseline provisions 72 GPU-eq (3 AZs × 24 × 8 / 3 replicas-per-GPU),
      covers 60% of peak at ρ=0.65
    - Burst pool covers the +20 GPU-eq peak hour at ρ=0.80
    - Headroom budget: 1.43× (ρ-target 0.70) × 1.5× (1-AZ-loss)
    - Prefix-cache sensitivity: at h=0.30 instead of 0.65, prefill
      requirement rises 2×; current pool does not hold, add 24 nodes
```

A reviewer should be able to read this and know (a) the target, (b) the assumption set, (c) the operating point, (d) the fallback, and (e) the sensitivities. All five are required.

## 6. The forecast as a living document

Capacity forecasts rot. The three most common rot modes:

- **Hit-rate drift.** The assumed prefix-cache hit rate was 65% in Q1 and is 42% by Q3 because a template change landed quietly. The forecast still assumes 65%. The fleet is underprovisioned on prefill and the AI SRE discovers it during the holiday load test.
- **Shape drift.** A product change doubled the average completion length. The forecast still assumes the Q1 distribution. The fleet is underprovisioned on decode.
- **Rate shift.** A product launch moved demand from US-business-hours to global-24-hours. Peak factor went from 2.5× to 1.4×, meaning the fleet is *overprovisioned*. The spot pool is sitting idle; finance is asking why.

The response: **re-forecast monthly, review quarterly**. Keep the forecast under version control. Record the actuals at each cadence step and track forecast error (`projected_peak_RPS / actual_peak_RPS`). A forecast that is consistently high is overspending; one that is consistently low is a reliability risk. Chapter 06's load-test regime feeds the actuals.

## Summary

- **Decompose by workload class.** Chat, RAG, tool-calling / agent, batch — each has distinct prompt / completion / hit-rate / time-of-day shape. An aggregate RPS number is unusable.
- **Project rate and shape separately.** Simple growth model for rate, distribution-level forecasts for prompt and completion lengths, an explicit prefix-cache-hit-rate assumption with sensitivity.
- **Compute per-class GPU cost in engine-native units.** Effective prefill tokens, decode tokens, KV-cache bytes. Translate to GPU-seconds via the engine's measured throughput curves, not derived ones.
- **Aggregate with explicit multipliers.** Peak-to-mean, utilisation target, failure-domain loss, rolling-update overhead. Document each; do not hide them in a round-up.
- **Translate to a node-pool shape.** GPU generation × instance SKU × node count × replicas-per-node × TP/PP × pool role (baseline, burst, P/D-disaggregated). Validate memory and networking.
- **The forecast is a living document.** Re-forecast monthly against actuals, review quarterly, feed the capacity-reservation calendar. Track forecast error as its own reliability metric.

The next chapter addresses the first choice the node-pool shape forces: which GPU generation, in what mix, under what cross-generation-translation discipline.
