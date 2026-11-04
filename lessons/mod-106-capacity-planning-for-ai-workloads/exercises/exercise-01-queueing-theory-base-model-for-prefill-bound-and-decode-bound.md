# Exercise 01 — Queueing-Theory Base Model for Prefill-Bound and Decode-Bound Workloads

**Estimated effort:** 3 hours · **Output:** one Markdown note (~1200 words) plus a worked spreadsheet or notebook

## Objective

Produce the first-order queueing-theory model for *two* named workloads that your fleet serves (or that you choose from a documented public example) — one you classify as prefill-bound and one you classify as decode-bound. For each, compute the Little's-Law budget, estimate the utilisation operating point, identify the dominant bottleneck, and name the features of the production engine that violate the independence assumption of M/M/c so you know when to prefer simulation.

A good deliverable makes the arithmetic **concrete**: specific prompt / completion distributions, a specific `max_num_seqs` or equivalent engine parameter, a specific SLO, a specific headroom. "We are decode-bound" is not an answer; "Chat surface A: median prompt 420 tokens, median completion 180 tokens, `max_num_seqs=128`, TTFT SLO 2s at p95, TPOT SLO 60 ms, operating at estimated `ρ = 0.63` on decode slots" is.

## Prerequisites

- Chapter 01 of this module.
- Access to (or plausible numbers for) two production or illustrative workloads with distinguishable profiles. If you cannot source real traffic shape, use the documented chat / RAG / batch profiles from a public engineering blog and attribute them.
- A working knowledge of the serving engine the workload is on (vLLM / SGLang / TensorRT-LLM / TGI). You do not need to be able to tune it; you need to be able to read its metrics vocabulary.

## Problem statement

Two workloads will be modelled: one prefill-bound, one decode-bound. For each, author the first-order model as four sections of a short Markdown note, with the arithmetic in an attached spreadsheet or notebook.

### Section 1 — Classify the workload

Name the workload. Describe the user-visible surface, the typical prompt and completion length distributions (histogram or at least p50 / p95 / p99), and the time-of-day shape. State your classification (prefill-bound, decode-bound, mixed) and the measured or estimated split of GPU-seconds between prefill and decode. Show your reasoning — a prefill-dominated RAG surface with 8K prompts and 150-token completions does not need defending; a borderline surface does.

### Section 2 — Compute Little's Law

For each workload, state `λ` (target request rate per replica), `W` (end-to-end service time = TTFT + decode time for an average completion), and derive `L = λ · W`. Compare `L` to the engine's `max_num_seqs` or equivalent in-flight-request cap. State whether the fleet is comfortably within the cap, close to the cap, or over it.

### Section 3 — Estimate utilisation and name the operating point

Estimate `ρ` for the dominant bottleneck:

- Decode-bound: `ρ = L / max_num_seqs` (per replica), or `ρ = (λ · W) / (max_num_seqs · replicas)` for the fleet.
- Prefill-bound: `ρ = prefill_tokens_per_second_demand / prefill_tokens_per_second_capacity`.

State the target `ρ` from chapter 01 §3.1 (0.70 is the conservative choice for p99 survival). Report whether the fleet is at, below, or above the target. If above, state the fleet-size adjustment required to bring it below.

### Section 4 — Enumerate the simple-model violations

List which of the modern engine features your fleet uses: continuous batching (nearly certainly yes), prefix caching, speculative decoding, chunked prefill, prefill/decode disaggregation, MultiLoRA. For each one present, write one sentence on how it violates the independence assumption of M/M/c and whether that violation biases the first-order model high or low for your workload.

End the note with a one-paragraph verdict: is the first-order model accurate enough for your current capacity-review cadence, or does the number of violations suggest you need to upgrade to simulation (chapter 01 §5)?

## Requirements

1. **Two workloads**, one prefill-bound and one decode-bound, with concrete numbers. Mixed-only is not acceptable; the exercise exists to make the asymmetry visible.
2. **Shape distributions, not means.** At minimum p50, p95 for prompt and completion length. Means alone will not catch the tail-driven planning risks.
3. **Little's Law numbers that match or exceed engine parameters.** If `L > max_num_seqs`, name it explicitly as a sizing issue.
4. **Utilisation estimate with its bottleneck.** Decode-slots for decode-bound, prefill-token-budget for prefill-bound. No `nvidia-smi` GPU-util.
5. **Named engine features that violate M/M/c.** With direction of bias for each.
6. **The attached spreadsheet / notebook shows the arithmetic.** Reader can reproduce the numbers.

## Starter guidance

- The Little's-Law arithmetic is deliberately simple. Resist the urge to include a per-request-cost model; that is exercise 02. Here the goal is to internalise the queueing shape.
- For the decode-time component of `W`: `W_decode ≈ completion_length × TPOT_measured`. If you do not have measured TPOT, cite a plausible range from the engine peer's benchmarks (e.g., vLLM public benchmarks) and bracket your answer.
- For prefill-bound workloads, `W` is TTFT-dominated; `decode_time` is small. The arithmetic simplifies.
- Do not derive throughput from theory. Use measured numbers from the engine's Prometheus metrics (e.g., vLLM's `vllm:gen_throughput_tokens_per_second`) or from a public benchmark cited inline.
- The "which engine features violate independence" list is an understanding check, not a research exercise. If your fleet uses continuous batching + prefix caching + chunked prefill, state so and move on; depth belongs in exercise 02 and chapter 02.

## Acceptance criteria

- Markdown note is ≤ 1500 words.
- Both workloads classified with reasoning; prefill-bound and decode-bound each have a worked example.
- Each workload has prompt / completion distribution (p50 and p95 minimum), `λ`, `W`, `L`, and `ρ`.
- Each workload states whether the fleet is at, below, or above the target utilisation; names the fleet-size adjustment if above.
- Each workload lists the modern engine features in use with a one-sentence bias note.
- Verdict paragraph commits to a position — first-order is good enough, or upgrade to simulation.
- Spreadsheet / notebook attached and reproducible.

## Stretch goals

- **Pollaczek–Khinchine refinement.** Replace M/M/c with M/G/c (general service-time distribution) using a measured variance-to-mean ratio from production. Does the predicted waiting-time change by more than 30%? If yes, the heavy-tailed service time is a first-order concern.
- **Cross-generation translation.** Pick one of your workloads and recompute the Little's Law budget on a hypothetical H200 fleet (same model, different hardware). Use measured H200 throughput from a current public benchmark (MLPerf, vLLM leaderboards) rather than guessing. How does `ρ` change at the same traffic?
- **Simulation calibration.** If you have access to a simulator (vidur, LLMServingSim) or can build a minimal one, calibrate it against your production throughput measurements and report the gap between its predicted TTFT p95 and the measured TTFT p95 at the same offered load. Beyond 30% gap, the model needs more work; within 30%, you have a usable simulator.
