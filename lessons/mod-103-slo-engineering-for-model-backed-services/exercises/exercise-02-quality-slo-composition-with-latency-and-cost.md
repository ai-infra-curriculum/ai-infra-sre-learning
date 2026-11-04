# Exercise 02 — Quality SLO Composition with Latency and Cost

**Estimated effort:** 3 hours · **Output:** one Markdown file (~1800 words) — the composed multi-dimensional SLO stack for the workload from exercise 01

## Objective

Extend the latency SLI matrix from exercise 01 with the six quality SLIs from chapter 02 and the four cost SLIs from chapter 04 (for the latter, you only name them here; cost targets are fully derived in exercise 04). Compose the three dimensions into a single error-budget stack that the on-call primary can reason about on a page. Author the eval-peer handoff paragraph for every quality SLI.

Carry the workload from exercise 01 forward. You will build on this document in every subsequent exercise.

## Prerequisites

- Exercise 01 completed.
- Chapters 01, 02, and 04 of this module.
- A working mental model of the eval-peer boundary from `mod-101` chapter 02 (chapter 07 of this module extends it).

## Problem statement

You carried the latency SLI matrix out of exercise 01. The product sponsor's next question is *the whole quality picture* — "what hallucination rate are you willing to commit to, and how does it compose with latency into the user experience we promise?" Produce the answer.

Author a Markdown file with these five sections.

### Section 1 — The quality SLI table

A table. Rows are the six quality SLIs from chapter 02 (hallucination, toxic-output, tool-call-success, structured-output-conformance, refusal-two-sided, format-drift). Columns: SLI definition, target shape, window, sampling methodology, burn-rate thresholds. For each row, either propose an SLI with concrete numbers or mark N/A with a one-line reason (e.g., "format-drift is warn-only on this surface because the downstream consumer is a UI that re-formats anyway").

For the refusal SLI, remember it is *two-sided*. Author both the warranted-refusal and the over-refusal sub-SLIs as separate rows.

### Section 2 — The eval-peer handoff paragraph per quality SLI

For each quality SLI that is not N/A, write the "signal source" paragraph from chapter 02 section 5. Name the pipeline, the version, the attribute surface, and the change-coordination process. If the eval peer does not exist on your team today, use the illustrative name `eval-pipeline-vN` and mark the dependency as "to be stood up alongside the SRE-side work."

### Section 3 — The composed stack

Produce the full table from chapter 02 section 3, extended with your latency SLIs from exercise 01 and your cost SLIs named (targets to be filled in exercise 04). Columns: dimension, SLI, target, 28-day shortfall budget, window, burn-rate thresholds. Include a bottom row showing the sum-of-shortfalls composite ceiling; this is the single number you can hand the sponsor as the overall user-experience commitment.

### Section 4 — The reconciliation sketch

Preview of exercise 05: describe how a correlated-burn incident (two or more SLIs burning simultaneously within the 2-hour window) would route through this stack. Minimum: trace two concrete scenarios.

- **Scenario A.** A hallucination burn correlated with a TTFT burn. What does on-call do first?
- **Scenario B.** A tool-call-success burn correlated with a cost burn. What does on-call do first?

You do not have to write the full error-budget policy here (that is exercise 05); you just have to show the stack admits sensible reconciliation.

### Section 5 — The composition arithmetic

Show the sum-of-shortfalls calculation from chapter 02 section 3 against your SLO values. Call out which SLI has the largest shortfall contribution (likely the hallucination SLI) and whether the composite ceiling meets the user-experience contract named in exercise 01 section 1.

If the composite is unacceptably low, propose where to tighten — and name the cost of the tightening (which dimension it tightens, what it costs in reliability engineering capacity or in cloud spend).

## Requirements

- **Six quality SLIs addressed.** N/A is acceptable with justification; empty rows are not.
- **Refusal SLI is two-sided.** Over-refusal and warranted-refusal appear as separate rows.
- **Eval-peer handoff paragraph per non-N/A quality SLI.** Named pipeline, named version, named channel.
- **Composed stack table is complete.** All dimensions represented, sum-of-shortfalls row at the bottom.
- **Two reconciliation scenarios traced.** Not resolved in full (that is exercise 05), but the ordering of on-call response is shown.
- **Composition arithmetic matches the targets.** If your numbers do not sum to your composite ceiling, the arithmetic is wrong.
- **Burn-rate thresholds recomputed for each quality SLI's target.** A 97% target and a 99.9% target have different absolute thresholds; do not reuse the latency numbers from exercise 01.

## Starter guidance

- The hallucination SLI target is the one most likely to feel aggressive. Resist the temptation to set it at 99.9%. Baseline the current model + prompt against production traffic (you can use an illustrative baseline for the exercise); set the target at that baseline minus a small headroom factor. Chapter 06 covers the full baselining methodology; here you are *using* the methodology.
- The toxic-output SLI target is the opposite — aim high (99.9%+). Toxic outputs are typically treated as incidents, not probabilistic artifacts. If your workload is sensitive to a specific toxicity dimension (child-safety, PII, etc.), note the sensitivity and the paging policy.
- For structured-output-conformance, note whether your model uses constrained decoding / JSON mode. If so, the target can be high (99%+); if not, the target may need to be in the 90s and the application layer will absorb retries. The distinction matters for the tool-call-success SLI, which composes the schema-validity with the tool-side execution.
- For the refusal over-refusal sub-SLI, be explicit about the eval set that drives it. "Over-refusal on benign prompts measured against eval set `eval-benign-vN`" is a real SLI; "over-refusal rate" without a named denominator is not.
- The composition arithmetic is the sum of shortfalls near 1 (chapter 02 section 3). It is a *ceiling* on the composite "all dimensions good" rate; do not try to claim it is the exact value. If the sponsor asks for a single number, the sum-of-shortfalls ceiling is what you promise against.
- For the reconciliation scenarios, do not solve them — just show the ordering. Scenario A's correct ordering is "declare IC escalation first, investigate traffic shift second, consider rollback third"; scenario B's correct ordering is probably "page on quality, route cost to ticket." The exercise is showing the stack admits the correct ordering.

## Acceptance criteria

- Markdown file is self-contained and ≤ 2500 words.
- Section 1 table has all six quality SLIs (eight rows, with refusal split).
- Section 2 has a handoff paragraph per non-N/A quality SLI.
- Section 3 composed stack has every dimension; sum-of-shortfalls row is present.
- Section 4 traces two scenarios with on-call response ordering.
- Section 5 arithmetic is correct and matches the stack.
- A reviewer unfamiliar with the workload can understand the full quality + latency + cost commitment within fifteen minutes of reading.
- The eval-peer dependency is visible — not hidden in a footnote.

## Stretch goals

- **Composite good-event alternate.** For contrast with the multi-SLO stack, author one *composite* SLI (chapter 02 section 2 Pattern 1) and note what diagnostic information is lost compared to the stack. Useful to show the sponsor why you chose the stack.
- **Judge-vs-rule split per quality SLI.** For each quality SLI, note whether the signal is judge-model-based (expensive, sampled) or rule-based (cheap, 100% coverage). The distinction shapes the sampling methodology and the alert tuning.
- **Hallucination sub-SLIs by request type.** Chapter 02 section 1 noted that the valid-event filter for hallucination rate is "request-type the judge is calibrated for." If your workload has heterogeneous request types, break hallucination rate into per-type SLIs and note which ones are N/A because the judge is uncalibrated.
- **Signal-health sub-SLI.** Add a meta-SLI: what fraction of quality-SLI samples the judge returned `UNKNOWN` on. A growing `UNKNOWN` rate is a signal-health regression (eval peer concern) before it is an SLI concern.
