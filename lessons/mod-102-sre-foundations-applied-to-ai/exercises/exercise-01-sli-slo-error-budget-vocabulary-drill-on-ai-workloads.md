# Exercise 01 — SLI / SLO / Error-Budget Vocabulary Drill on AI Workloads

**Estimated effort:** 2 hours · **Output:** one Markdown file (~1000 words) with four sections

## Objective

Produce the AI-shaped SLI / SLO / error-budget stack for a concrete AI workload you can name. This exercise is the warm-up for exercise 02 (the policy document) and exercise 03 (the rotation that enforces the policy). Pick a workload here; carry it through the module.

A good deliverable from this exercise is **concrete**: specific SLIs with specific numbers, specific valid-event filters, specific burn-rate alert thresholds. "Hallucination rate SLO" is not an answer; "judge-pass-rate ≥ 97% over 28 rolling days, measured on a 1% continuous production sample of request-types A, B, C, with burn-rate alerts at 14.4×/1h, 6×/6h, 1×/3d" is.

## Prerequisites

- Chapters 01 and 02 of this module.
- A concrete workload context — your current team's AI surface, a prior team's, or a well-documented illustrative one from a public engineering blog. If you cannot pick one, use "a hypothetical internal LLM gateway serving a chat product, a code-assist product, and an internal retrieval-augmented knowledge-base product" and be explicit that it is illustrative.

## Problem statement

Your team is staffing up its first SLO review and the product organization has asked for a one-pager that enumerates the reliability dimensions, the targets, and the current burn state. Your output of this exercise *is that one-pager*.

Concretely, author a Markdown file with these four sections:

### Section 1 — The workload

Half a page. Name the workload, describe the request shape (streaming / non-streaming, single-turn / multi-turn, synchronous / async), enumerate the external dependencies (models, retrieval indexes, providers, guardrails), and note the user-visible contract (latency expectation, correctness expectation, cost expectation).

### Section 2 — The SLI stack

At minimum one SLI from each of the four dimensions in chapter 02:

- **Availability** — protocol-level good-event SLI.
- **Latency** — at least TTFT and TPOT; add ITL and E2E if applicable.
- **Quality** — at least one constructed quality signal with a judge / rule / user-feedback pipeline; name the eval-peer owner of the signal and the SRE-side wiring.
- **Cost** — cost-per-1M-successful-tokens or equivalent.

For each SLI, write:

- SLI definition (ratio of good events to valid events).
- The valid-event filter (what counts in the denominator, what is excluded).
- The target SLO (percentage / number) and the window (typically 28 rolling days).
- The burn-rate alert thresholds (fast-burn page, medium-burn page, slow-burn ticket).

### Section 3 — The MTTR / MTBF / MTTA / MTTD table, stratified

A small table with one row per failure class you care about and columns for MTTD, MTTA, MTTR targets. Use chapter 01's guidance that AI-side MTTD is stratified — a quality MTTD and an availability MTTD are not the same number, and reporting them as one is misleading.

Minimum rows: classical infra (5xx / scheduler / GPU fault), latency burn, quality burn, provider-degraded, prompt-regression. Add rows for your workload's specifics.

### Section 4 — The toil enumeration

List at least five items of AI-specific toil visible on your workload *today*, with the one-line automation plan for each (see chapter 01, section 3 for the canonical list). Candidates: manual canary analysis, manual quota ticket filing, manual drift chart review, manual model promotion, manual prompt hotfix, manual GPU reboot. Pick from your workload; invent a sixth if the five above do not cover something you've actually seen.

## Requirements

1. **Named workload.** Specific, not "LLM service."
2. **At least four SLIs**, one per dimension. Multi-SLO stack, not composite.
3. **Valid-event filters made explicit** — not "we measure quality" but "we measure the judge pass rate on sampled responses where the prompt is longer than 20 tokens and not a known-refusal."
4. **Burn-rate thresholds numeric** — not "page on fast burn" but "page at 14.4×/1h *and* 6×/6h with the two-window rule."
5. **MTTR table stratified** — one row per failure class, not one aggregate.
6. **Toil enumeration with automation plan** — not a list of complaints.

## Starter guidance

- The eval peer (level 30) owns the quality signal's methodology. You are consuming the signal, not authoring it. Write the SLI such that if the eval peer's judge model changes, the SLI definition is still intact — i.e., name the *inputs to the pipeline* (sample, filter, judge interface) not the implementation inside the pipeline.
- For the burn-rate thresholds, use the [SRE Workbook, Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) multi-window / multi-burn-rate table directly. Adjust the thresholds per-SLO; a quality SLO at 97% will have a different burn arithmetic than an availability SLO at 99.9%.
- For the MTTR table, draw failure-class rows from chapter 01 section 2 and from `mod-101` chapter 04 right column.
- For the toil enumeration, be specific. "Manual model promotion" is weak; "every Thursday afternoon the primary on-call manually copies the model artifact from the training registry to the serving registry and edits the model-card YAML" is strong.

## Acceptance criteria

- Markdown file is self-contained and ≤ 1500 words.
- Section 1 names a specific workload.
- Section 2 contains at least four SLIs with explicit valid-event filters and burn-rate thresholds.
- Section 3 is a stratified table with at least five failure-class rows and MTTD / MTTA / MTTR numeric targets.
- Section 4 lists at least five toil items with named automation plans.
- Reviewer can read the file in ten minutes and understand the reliability contract of the workload without further context.

## Stretch goals

- **Vendor / provider column.** For workloads using a model provider, add a column to the MTTR table for "does the provider's SLA cover this failure class?" The answer is usually no; the exercise is making that visible.
- **Historical burn-rate look-back.** If you have access to real production telemetry, compute what the error-budget burn would have looked like over the last 28 days under your proposed SLOs. Did you meet them? If not, are the targets wrong or is the reliability work incomplete? The honest answer informs exercise 02.
- **Composite good-event alternate formulation.** For contrast, author one extra SLI that uses the *composite* good-event form (chapter 02 section 1 Pattern 1) and note what diagnostic information is lost compared to the multi-SLO stack.
