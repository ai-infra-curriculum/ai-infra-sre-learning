# Exercise 05 — Non-Abstract Large-System Design Review of an AI Stack

**Estimated effort:** 2 hours · **Output:** one Markdown design-review file (~2000 words) + a residual-risks appendix

## Objective

Drive a **non-abstract** design review against the AI stack behind the workload you have been carrying through this module. The output is the review artifact — the one that would be the input to a launch coordination engineering checklist (chapter 01 section 5) and the one that would be audited at a readiness review.

"Non-abstract" means the review is full of **numbers**. No "we'll size it later;" no "we can add retries if needed;" no "capacity will not be a problem." Every claim in the review either has a number behind it or is marked as a residual risk to be measured before launch.

The reference is [SRE Workbook, Chapter 12 — Introducing Non-Abstract Large System Design](https://sre.google/workbook/non-abstract-design/).

## Prerequisites

- Chapter 01 of this module (especially sections 5 and 6).
- Exercises 01–04 (the workload, the SLOs, the policy, the rotation, the engine cheatsheets — all carry in).

## Problem statement

You have been invited to the launch review of a new (or next-version) AI product on your platform. Your charter (`mod-101` chapter 01) says the AI SRE team owns the reliability contract; you have been asked to run the non-abstract design review before the launch gate. The output of this exercise is that review artifact.

Author a Markdown document with the eight sections below. Every section should have at least one back-of-envelope number or a residual-risk entry explaining why the number is not yet available.

## Requirements

### Section 1 — The system under review

Describe the stack as a block diagram in prose: ingress / gateway → auth / rate-limit → routing → engine pools (which engines, which model artifacts) → retrieval (if applicable) → model provider (if applicable) → guardrails → egress.

Include the data-flow for one representative request path end-to-end. Name every component.

### Section 2 — Traffic and token-throughput back-of-envelope

Per chapter 01 section 6, the right unit of load is token throughput, not RPS. Produce:

- Expected request volume (RPS) at launch, at 90-day projection, at 1-year projection.
- Expected prompt-length distribution (mean, p50, p95, p99).
- Expected generation-length distribution.
- **Derived**: expected input and output tokens/sec at each horizon.
- **Derived**: expected cost per month at each horizon, given your model-provider pricing or your self-hosted cost model.

If any input number is not known, state the assumption range (e.g., "We assume p95 prompt length is 2,048 tokens based on the eval-set distribution; this is the first residual risk to validate against beta traffic").

### Section 3 — Capacity, fleet, and KV-cache footprint

Back-of-envelope the serving fleet:

- GPU shape chosen and why.
- Fleet size at launch (replicas, GPUs per replica).
- KV-cache footprint per concurrent sequence at your `max-model-len` (see chapter 01 section 6 for the formula).
- Maximum concurrent sequences per replica (bounded by KV pool).
- Headroom % at launch volume.
- Trigger points for scale-out (HPA metric, Karpenter policy — pointer forward to `mod-106`).

### Section 4 — Prefill vs. decode compute mix

Characterize the workload's prefill-to-decode ratio. Does it justify disaggregated serving (NVIDIA Dynamo — chapter 06), or is a co-located vLLM / SGLang / TRT-LLM deployment the right call?

The honest answer here is often "we don't know yet, we'll measure during beta." That is fine — mark it as a residual risk to measure. The exercise is making the uncertainty visible, not pretending it is resolved.

### Section 5 — Dependencies and failure domains

Enumerate every external dependency:

- Model artifact(s) and their training-serving handoff (`mod-101` deferral map).
- Model provider(s), if applicable — quota pool, SLA, fallback contract (`mod-107`).
- Retrieval index (if applicable) — ownership, refresh cadence, failure mode.
- Guardrail sidecars (`mod-110`).
- Observability pipeline.
- Billing / cost accounting pipeline.

For each dependency, name a failure mode (what goes wrong), a detection signal (how do we know), and a response (what the on-call does). This is a compressed version of the chapter-02 reconciliation rule applied to dependencies.

### Section 6 — SLO contract (from exercise 01) + policy (from exercise 02)

Compressed restatement. The point of this section in a design review is to make visible that the launch is bound to the SLOs and the policy, not negotiating new ones.

### Section 7 — Rollout plan

Progressive delivery path (pointer forward to `mod-111`):

- Pre-launch: staging / shadow traffic plan.
- Beta: percentage, selection, success criteria.
- Progressive deploy: canary percentages, dwell times, auto-rollback triggers.
- GA: launch criteria, specifically which residual risks must be resolved before GA.

### Section 8 — Residual risks

A ranked list of residual risks — things you cannot back-of-envelope at the design review and must measure before or during beta. Each risk has:

- Description.
- Severity (low / medium / high / critical).
- Owner.
- Measurement plan.
- Deadline (relative to launch).

A design review with zero residual risks has not been done honestly; a design review with ten critical residual risks is not ready to launch. Three to six is a healthy range.

## Starter guidance

- The back-of-envelope sections (2 and 3) are where it is tempting to hand-wave. Resist. A missed KV-cache calculation is the single most common capacity-review failure.
- For Section 2's cost projection, use the actual model-provider pricing tables (OpenAI, Anthropic, Google, Bedrock, Azure) or your self-hosted cost-per-token model. If you are doing self-hosted on GPU, build a simple cost model: GPU-hour cost × GPU-count × utilization / token-throughput.
- Section 4's prefill-vs-decode question does not need exotic benchmarking to answer at design time — a rough token-count ratio from the eval set is enough to decide between "co-located with chunked prefill" and "disaggregated serving." Err on the side of simpler (co-located) for Year-One platforms.
- Section 7's auto-rollback triggers should reference the error-budget policy from exercise 02 — specifically which budget bands halt the rollout. Do not invent a new release-gate language here.
- Section 8's residual risks should be *written down so they can be assigned* — a residual risk that lives in an engineer's head for three months before launch is a bug.

## Acceptance criteria

- Markdown file is self-contained and ≤ 2500 words.
- All eight sections populated with numbers where numbers are claimed, or residual risks where they are not yet available.
- Section 2 produces a token-throughput projection, not an RPS projection.
- Section 3 produces a KV-cache footprint calculation.
- Section 5 names at least four external dependencies with failure-mode / detection / response triples.
- Section 7 references the exercise-02 policy as the auto-rollback authority.
- Section 8 contains between three and six residual risks with owners and measurement plans.
- A reviewer can read the artifact in 20 minutes and resolve whether the launch is design-ready.

## Stretch goals

- **Pre-mortem.** Add a section where you write the post-mortem for the *most likely* launch-failure scenario before launch. What signal would fire first? Which part of the rotation is responsible? Does the response work?
- **Peer-reviewed back-of-envelope.** Have the capacity section reviewed by the kernel SME / the inference-engine peer. Capture one surprising objection and revise.
- **Launch-coordination checklist.** Convert the review into the LCE checklist form from chapter 01 section 5. This is the handoff artifact to the launch gate itself; `mod-111` deepens the checklist.
- **Compare to a classical SRE design review.** One-line restatement of what sections 2–5 would look like for a classical CRUD service — concretely, which numbers change and which disappear. Reinforces the chapter-01 vocabulary rebase.
