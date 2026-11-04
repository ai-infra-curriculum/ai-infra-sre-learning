# Exercise 03 — Availability SLO for Streaming Responses

**Estimated effort:** 2 hours · **Output:** one Markdown file (~1200 words) — the streaming-availability block of the SLO stack plus the failure-walkthrough appendix

## Objective

For the workload from exercise 01, author the three streaming-availability SLIs — partial-success, graceful-degradation, streaming-stall — including the per-SLI valid-event filter, target, and burn-rate alert config. Then produce a short failure-walkthrough appendix that traces at least four concrete failure modes through all four availability-block SLIs (protocol-level + three streaming SLIs), demonstrating that the SLI surface diagnoses by burn pattern.

## Prerequisites

- Exercises 01 and 02 completed (you are now extending the stack rather than starting fresh).
- Chapter 03 of this module.

## Problem statement

The protocol-level `2xx-or-3xx` SLI is not sufficient on your streaming surface. You know this because: (a) the chat UI displays partial responses and users complain about frozen streams while the gateway reports 2xx, (b) when the primary provider degrades the fallback path serves without the error-budget stack noticing, and (c) scheduler interference between tenants produces stalls that no SLI currently catches.

Author the streaming-availability block of the SLO stack and the failure-walkthrough that shows the block does its job.

### Section 1 — The streaming-availability block

A table extending the stack from exercise 02. Four rows (protocol-level availability for completeness, then the three streaming-specific SLIs). Columns: SLI definition, valid-event filter, target, 28-day shortfall budget, burn-rate thresholds (fast / medium / slow).

### Section 2 — The fallback-path catalog (for graceful-degradation SLI)

A table. Rows are the fallback paths available on your workload (primary → secondary provider, primary → smaller model, retrieval-unavailable fallback, guardrail-fail-open, cached-response, etc.). Columns: fallback name, trigger condition, fail-open vs. fail-closed decision, quality-vs-primary delta (qualitative is acceptable; chapter 06 covers how to measure it), per-path sub-SLI target.

Minimum three fallback paths. If your workload has fewer, invent plausible ones for the exercise and mark them illustrative.

### Section 3 — Streaming-stall threshold calibration

For the streaming-stall SLI, defend the gap threshold you picked. Minimum content:

- What UI consumes the stream, and what the user experiences at the chosen threshold.
- What engine-side events could produce gaps of that length (scheduler preemption, KV-cache eviction, upstream network burst).
- What the trade-off is between threshold (lower threshold = more sensitive to minor interference = more pages).

### Section 4 — Measurement decisions

Three decisions you must codify explicitly in the SLI definition:

1. **Server-side vs. client-side.** Which number is authoritative? What is the delta signal and where does it live in the stack?
2. **Client cancellation vs. server truncation.** How does your gateway distinguish them? If it does not, what instrumentation gap must land before the partial-success SLI is operable?
3. **Tagging the fallback path on every response.** What attribute, where is it emitted, how does the graceful-degradation SLI consume it?

### Section 5 — Failure walkthrough appendix

For each of the four failure cases below, walk through all four availability-block SLIs and say whether each burns or stays clean.

- **Failure A.** Classical 5xx storm — gateway returning 503 on 30% of requests for 15 minutes.
- **Failure B.** Mid-stream server crash — scheduler preempted a request and did not resume it after 400 of 1000 tokens.
- **Failure C.** Provider fallback — primary provider returned 5xx on 20% of requests for 10 minutes; LiteLLM routed to secondary.
- **Failure D.** Scheduler-induced stall — a noisy co-tenant caused a 4-second inter-token gap in the middle of a stream.

Then add a fifth of your own, specific to your workload. Candidates: guardrail-sidecar timeout, retrieval-index outage, cache-corruption on prefix cache, provider rate-limit threshold crossed.

For each failure, name:

- Which SLIs burn.
- Which SLI burns *first* in the alert ladder.
- Which runbook the on-call primary routes to (reference `mod-102` chapter 03 escalation ladder if you did not re-write it for your workload).

## Requirements

- **Four-row stack table** with per-SLI threshold values.
- **Three or more fallback paths cataloged** with fail-open / fail-closed decision explicit per path.
- **Streaming-stall threshold defended** in UI terms and engine-event terms.
- **Three measurement decisions codified** with the instrumentation-gap flag where applicable.
- **Five failure cases walked** — the four prescribed plus one of your own.
- **Burn-pattern diagnosis is the point.** Each failure case should burn a *different* SLI (or a different combination), proving the SLI surface diagnoses by burn pattern rather than reporting one aggregate signal.

## Starter guidance

- The partial-success SLI is the one most teams miss. Spend extra effort on the valid-event filter — in particular the distinction between client-closed and server-closed termination. If your gateway does not emit the distinction, this exercise just exposed the instrumentation gap; document it.
- The graceful-degradation SLI is unusual in that it is *not* a straight good/bad SLI. The aggregate rate is the primary number, but the per-path sub-SLIs are often what the review cares about. Be explicit about which path is in each sub-SLI.
- The streaming-stall threshold should be chosen for *your* UI, not for a generic "1 second." A voice UI's threshold is ~200ms; a background-batch pipeline's threshold is ~10s. Picking the right number is part of the exercise.
- For the fallback-path catalog, distinguish between *reliability* fallbacks (provider swap, model-fallback) and *safety* fallbacks (guardrail fail-open). Both are in the catalog but the ownership routing may differ — safety fallbacks typically route a notification to `ai-infra-security` as well as the SRE on-call.
- The failure walkthrough is the acceptance test for the SLI surface. If two of your five failures burn *the same set* of SLIs, you probably have a redundant or missing SLI; iterate on the definitions in section 1 until the failures diagnose cleanly.

## Acceptance criteria

- Markdown file is self-contained and ≤ 1500 words.
- Section 1 has the four-row table with explicit valid-event filters.
- Section 2 has three or more fallback paths with per-path target.
- Section 3 defends the stall threshold on two axes (UI, engine events).
- Section 4 addresses all three measurement decisions.
- Section 5 walks five failure cases; each burns a distinct combination of SLIs.
- A reviewer can use the SLI surface to diagnose a new hypothetical failure after reading the document.

## Stretch goals

- **Streaming-stall as a leading indicator.** For the streaming-stall SLI, propose a lookahead: a short-window (30-second) stall-rate signal that leads the 28-day SLI by minutes. Useful for on-call paging ahead of the formal SLI burn.
- **Fallback-path quality delta measured.** For at least one fallback in the catalog, propose how to *measure* the quality-vs-primary delta (judge-model compare, side-by-side eval, user-feedback rate on fallback-served responses). Chapter 06 covers the instrumentation but you can describe the methodology here.
- **Sixth failure — correlated cross-surface.** A failure that burns the availability block on one surface *and* the quality block on another — e.g., a provider-degraded state where the chat surface sees availability burn and the RAG surface sees quality burn simultaneously. Walk through how the SLI surface handles the cross-surface correlation.
- **Protocol-level vs. streaming trade-off commentary.** Note one SLO numeracy artifact: the protocol-level availability SLO (99.9%+) sets a higher bar than the streaming-availability SLOs (which admit graceful degradation in their denominator). Discuss what the sponsor needs to understand about this trade-off.
