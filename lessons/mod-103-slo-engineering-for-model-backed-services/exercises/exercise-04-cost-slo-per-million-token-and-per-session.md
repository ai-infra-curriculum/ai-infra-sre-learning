# Exercise 04 — Cost SLO per Million Tokens and per Session

**Estimated effort:** 2 hours · **Output:** one Markdown file (~1200 words) — the cost SLI block and the business unit-economics derivation

## Objective

For the workload from exercise 01, author the four cost SLIs from chapter 04, derive each target from an explicit business unit-economics model, and tie the resulting block into the composed stack from exercise 02. The exercise is less about numbers and more about *defensibility*: the targets must trace back to a revenue unit, a margin assumption, and a traffic model — not an SRE's intuition.

## Prerequisites

- Exercises 01 and 02 completed.
- Chapter 04 of this module.

## Problem statement

Your SLO stack from exercises 01 and 02 has cost SLIs named with empty targets. Finance has asked for the cost contract in parallel with the latency and quality contracts. Produce it.

Author a Markdown file with these five sections.

### Section 1 — The business unit-economics model

For the workload you picked, name:

- **The revenue unit.** Dollars per subscriber per month, dollars per API call, dollars per transaction completed, dollars per batch job. One sentence.
- **The target gross margin on inference.** Typically expressed as a percentage; may vary by product tier.
- **The cost ceiling per revenue unit.** Revenue × (1 − gross margin). The arithmetic should be visible, not just the result.
- **The traffic profile per revenue unit.** How many requests, how many tokens (input + output), how many sessions does a single revenue unit correspond to? For a chat subscriber this is monthly; for an API customer it may be per-minute.

If you do not know your organization's numbers, use illustrative values and mark them as such. The purpose is the *shape of the derivation*, not the specific number.

### Section 2 — The four cost SLIs

A table. Rows are the four cost SLIs from chapter 04. Columns: SLI definition, target ceiling, derivation (explicit arithmetic chain from the section 1 model), 28-day window, burn-rate alert config (ticket-only per chapter 04).

For each SLI, the derivation column should read: "cost ceiling per revenue unit ÷ (SLI unit per revenue unit) = SLI target ceiling." For example: "$8 per subscriber per month ÷ 20,000 output tokens per subscriber per month = $400 per 1M output tokens."

### Section 3 — The product-ask pushback

Pick one plausible product-ask that implies over-provisioning. Candidates:

- "Reduce TTFT to 200 ms on all chat traffic."
- "Enable 32k-context on all queries by default."
- "Use the frontier-tier model on 100% of traffic."
- "Eliminate all over-refusals."

Write the three-step pushback from chapter 04 section 4:

1. Model the cost of the ask (added capacity, added per-token cost, added latency-budget headroom — whichever applies).
2. Translate into business margin terms.
3. Show the alternative and defend it.

Half a page. The purpose is to practice the translation; do not over-solve.

### Section 4 — The ticket-only alerting config

Chapter 04 section 5 said cost SLI burn routes to tickets, not pages — except in the three cases where cost burn correlates with abuse / safety / quality. For your workload:

- Which cost burn triggers *do* page? Name the specific correlated signal (toxic-output SLI burning simultaneously, input-token rate spiking without traffic growth, provider-pricing change notification). You may have zero cost-paging triggers; that is a valid answer if defended.
- Which cost burn triggers ticket to the SRE queue?
- Which cost burn triggers route directly to the SLO review?

The output is three lists, each with the burn condition and the recipient.

### Section 5 — Instrumentation gaps

Chapter 04 section 7 listed the instrumentation each cost SLI depends on. For your workload, identify which of these is *not* currently in place:

- Provider billing feed ingested.
- Per-request token-count on the trace / log (OTel `gen_ai.usage.*` attributes).
- Model + provider tag on every response.
- Session identifier linking related requests.
- Self-hosted cost model (GPU-hour × fraction-utilized) wired, if you run self-hosted serving.

Flag each gap and the exercise or ticket needed to close it. These gaps are what blocks the SLI from being operable; naming them explicitly is the point.

## Requirements

- **Business unit-economics model with explicit arithmetic.** No "we aim for a healthy margin"; the math is in the document.
- **Four cost SLIs with target derivation visible.** Not just the target — the derivation column shows where it came from.
- **Product-ask pushback on at least one real ask.** Translated into margin terms, not latency terms.
- **Ticket-only alerting config with correlated-paging exceptions explicit.**
- **Instrumentation gaps named.** If no gaps exist, say so (and the reviewer will be skeptical).
- **The cost block integrates into the exercise-02 stack.** When you drop this block in, the composition arithmetic from exercise 02 does not need to change (cost is a one-sided ceiling, not in the sum-of-shortfalls); but the stack table does need the new rows.

## Starter guidance

- The derivation is the whole exercise. Any target not derived from the business model is a target you cannot defend — and the moment the business sponsor asks "why that number?" you fail. Spend the first half of your time on the derivation.
- The cost-per-successful-session SLI is often the most *useful* SLI for a multi-turn surface. Subscriber-level revenue maps naturally to session-level cost. If your surface is single-turn, that SLI may legitimately be N/A; say so.
- For the product-ask pushback, do not pick a straw-man ask you know is wrong. Pick an ask your product team might realistically make and that *has* a defensible cost case. The pushback is a negotiation tool, not a refusal tool.
- For the self-hosted cost model, the formula is `GPU-hour-cost × fraction-of-GPU-used-by-workload / tokens-produced-per-hour`. Chapter 04 section 1 and `mod-106` cover the details. For this exercise, a one-paragraph description of how you would compute it (if you self-host) is acceptable.
- The paging exceptions in section 4 are often more important than the ticket thresholds. A cost spike that correlates with a toxicity classifier failure is an abuse incident; name the cross-SLI logic that triggers the page.

## Acceptance criteria

- Markdown file is self-contained and ≤ 1500 words.
- Section 1 has the four-element unit-economics model with arithmetic.
- Section 2 four-row table with derivation column populated per SLI.
- Section 3 pushback on one ask covers all three steps.
- Section 4 has three alerting lists (page, ticket, review).
- Section 5 names at least two instrumentation gaps (unless truly none exist, in which case defend).
- A finance peer unfamiliar with the engineering can resolve "what is the cost contract on this surface" within fifteen minutes.

## Stretch goals

- **Sensitivity analysis on the gross-margin assumption.** Show how the cost SLI targets shift if the business changes the target gross margin by 10 percentage points. The exercise is making the margin assumption visible so it can be renegotiated.
- **Provider-pricing-change playbook.** A one-paragraph runbook: if the primary provider raises prices, which SLIs are affected, what the notification to the sponsor says, what interim steps the SRE takes before the next SLO review. Chapter 04 section 5 mentions the case; author the response.
- **Per-model cost SLI breakouts.** The aggregate cost-per-1M-output-tokens number can hide a model-mix shift (cheap-model share rising) that looks fine at the aggregate but reflects a quality regression. Break the SLI into per-model sub-SLIs and discuss when the aggregate is still useful vs. when the breakouts are.
- **The abuse-detection cross-SLI.** For the cost-correlated-with-abuse case, propose the specific signal — a sudden rise in per-user input-token rate without a parallel rise in session count, a cluster of requests with a shared adversarial prefix — and the SLI that would catch it before the cost burn does.
