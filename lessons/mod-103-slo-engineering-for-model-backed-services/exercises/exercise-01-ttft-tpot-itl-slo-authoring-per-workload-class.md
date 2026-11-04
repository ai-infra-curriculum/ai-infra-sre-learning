# Exercise 01 — TTFT / TPOT / ITL SLO Authoring per Workload Class

**Estimated effort:** 3 hours · **Output:** one Markdown file (~1500 words) — the per-workload-class latency SLO one-pager

## Objective

Produce a defensible per-token latency SLO authoring document that picks, for a named workload, the TTFT / TPOT / ITL / E2E / goodput SLIs, the percentile targets, and the valid-event filters. The output is the *latency section* of what will become the full SLO document in subsequent exercises — a reviewer should be able to read it in ten minutes and understand what you are promising the user, how you measure it, and what you traded off to arrive at those numbers.

You pick the workload in this exercise and carry it through exercises 02–07. Pick something you can describe concretely — your current team's AI surface, a prior team's, or a well-documented illustrative one from an engineering blog. If none fits, use "a hypothetical internal LLM gateway serving a chat product, a tool-calling coding-assist product, and an internal batch summarization pipeline" and mark it illustrative.

## Prerequisites

- Chapter 01 of this module.
- Familiarity with the OpenTelemetry Semantic Conventions for Generative AI (<https://opentelemetry.io/docs/specs/semconv/gen-ai/>) at the level of knowing the attribute names.
- Working knowledge of the burn-rate alerting from [SRE Workbook, Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/).

## Problem statement

Your SLO review kicks off next sprint. The product organization has asked for a per-surface latency contract that names the SLIs, the targets, and the user-experience justification. Produce that artifact.

Author a Markdown file with these six sections:

### Section 1 — The workloads

For the workload you picked, enumerate the *distinct product surfaces* and classify each by workload class from chapter 01 section 3 — chat / batch / RAG / tool-calling / agentic / multimodal. Minimum three surfaces; a monolithic "single chat endpoint" is not enough detail to produce an interesting SLI table.

For each surface include a one-line user-experience contract. "Users expect responses to start within one second and complete within ten seconds on 95% of chat requests."

### Section 2 — The per-surface SLI matrix

A table. Rows are surfaces; columns are the five latency SLIs from chapter 01 (TTFT, TPOT, ITL, E2E, goodput). Each cell is either:

- A proposed SLI with target (percentile and threshold), **or**
- An explicit "N/A — this SLI does not apply to this surface" with a one-line reason.

Surfaces that are non-streaming (batch, async) should cleanly say so and derive their primary SLI from E2E. Surfaces that are agentic should push the primary SLI up to task-completion rather than per-call.

### Section 3 — Valid-event filters

For each SLI in the matrix that is *not* N/A, write the valid-event filter in prose. Minimum items to address:

- **Measurement side.** Server, client, or both? Where is the authoritative number?
- **Cache slicing.** For TTFT: are prefix-cache hits in the denominator or excluded? Why?
- **Chunked prefill.** How is the "first token" defined on surfaces using chunked prefill?
- **Speculative decoding.** For TPOT: server-side or client-side TPOT, given that rejected speculative tokens differ?
- **Abandoned streams.** Client-cancelled streams — how are they treated across TTFT / TPOT / ITL / E2E?
- **Retries.** If the client retries on admission-reject, does the retry's latency roll into the primary request?

Not every filter applies to every SLI; address the ones that do.

### Section 4 — Burn-rate alert thresholds

Per SLI, the three multi-window / multi-burn-rate thresholds (fast burn: 2% in 1h, medium burn: 5% in 6h, slow burn: 10% in 3d). You must *recompute the absolute threshold* against each SLI's target rather than reusing a single generic table — a 99.9% SLO and a 99% SLO have different absolute burn thresholds for the same multiplier.

Note which alerts page and which ticket.

### Section 5 — Threshold defence

For at least two of the SLIs, write the four-point defence template from chapter 01 section 4:

1. What the user experiences at this threshold.
2. What the business loses when this threshold is breached.
3. What it costs to deliver the threshold.
4. What the fallback is if the threshold is missed.

The point is to practice the translation into user-terms and cost-terms, not to write a comprehensive business case. Two SLIs (pick ones where the trade-off is live) is enough.

### Section 6 — Measurement gotchas worth writing into the SLI spec

One short list — the gotchas from chapter 01 section 5 that apply to your workload. For each, one sentence of how your SLI definition handles it. Prefix-cache, chunked prefill, speculative decoding, abandoned streams, retries, and anything workload-specific you noticed while writing sections 2–3.

## Requirements

- **Named workload with distinct surfaces.** "A chat product" is not enough; "a chat product streaming responses, a tool-calling coding-assist returning structured output, and an overnight batch pipeline summarizing support tickets" is.
- **Matrix is populated.** Every cell is either a proposed SLI with number or an explicit N/A with reason. No empty cells.
- **Percentile targets justified.** Not "p99 because p99 is standard." A sentence about the workload class is required.
- **Valid-event filters explicit.** Every SLI in the matrix has its filter addressed in section 3.
- **Burn-rate thresholds recomputed per SLI.** Not a copy-paste of the SRE Workbook table.
- **At least two threshold defences.** Each defence covers all four points.
- **Gotchas addressed.** At least five from the chapter list plus any workload-specific ones.
- **Reviewer can read in ten minutes and understand the contract.** No hidden assumptions.

## Starter guidance

- Pick workloads where you expect the matrix to be *different* across them. A chat surface, a batch surface, and a tool-calling surface will yield interesting differences; three variants of the same chat product will yield the same column repeated three times.
- For the N/A entries, be honest. Batch does not need TTFT; agentic surfaces often do not need TPOT at the per-call level. The exercise is not to fill the matrix dense; it is to defend the shape of the SLI surface for each workload class.
- Baseline numbers can be illustrative for this exercise — you are not expected to go measure production for three hours — but note the baseline-report dependency explicitly where your target implicitly assumes one. Chapter 06 exercise walks the full baseline methodology.
- For the threshold defence, pick SLIs where the trade-off is *actually live* in your workload: e.g., chat TTFT (where a user-perceived threshold and a cost threshold are both at play), or structured-output E2E on tool-calling (where a downstream parser timeout is a hard dependency). Pick ones with interesting trade-offs over ones with trivial ones.
- For abandoned-stream treatment, note whether your gateway distinguishes client-closed from server-closed stream termination. If it doesn't, the SLI's valid-event filter is weaker than it should be — flag this as an instrumentation gap to be addressed.
- The OpenTelemetry Generative AI semconv attributes give you the authoritative names for your schema. Use them where they match; extend them with `slo.*` attributes where you need SLI-specific tags (chapter 06 section 4 is the reference).

## Acceptance criteria

- Markdown file is self-contained and ≤ 2000 words.
- Section 1 names at least three distinct surfaces across at least two workload classes.
- Section 2 matrix is fully populated, with every N/A justified in-line.
- Section 3 addresses the six listed filter items where they apply.
- Section 4 has three multi-window thresholds per SLI with absolute values recomputed.
- Section 5 has two SLIs defended on all four points.
- Section 6 lists at least five gotchas with your SLI-spec responses.
- A reviewer unfamiliar with your product can resolve "what are you promising, how are you measuring it, what did you trade off" within ten minutes.

## Stretch goals

- **Cache-hit vs. miss break-out for every surface.** TTFT published as two SLIs per surface (cache-hit, cache-miss) with the different percentile targets and different valid-event filters. Harder to defend to the sponsor, cleaner diagnostic signal.
- **Workload-mix sensitivity analysis.** Pick one SLI and compute how its target shifts under a plausible traffic mix change (e.g., 20% growth in long-prompt share). A sensitivity analysis protects the target from a single traffic shift silently breaking the SLO.
- **Client-side vs. server-side delta tracking.** Add a low-priority SLI to the matrix: the delta between client-measured TTFT and server-measured TTFT. A growing delta is a gateway / egress incident; tracking it continuously surfaces it early.
- **Peer review by someone from another workload class.** Have an SRE who works on a different workload class (e.g., classical CRUD) read your document and resolve "what does this service promise users?" without asking. Capture one surprising objection with your revision.
