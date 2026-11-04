# Exercise 04 — Model-Provider Quota Pool and Rate-Limit Plan

**Estimated effort:** 3 hours · **Output:** one Markdown quota plan (~1500 words) and a tier / quota lookup table

## Objective

Produce the quota plan artifact described in chapter 04 §6 for a named workload that consumes at least one hosted model provider (OpenAI, Anthropic, Google / Vertex AI, AWS Bedrock, or Azure AI Foundry). The plan must size the reservation, the burst, the fall-through, and the residency constraints, with dates and renewal owners. The output is what you bring to a quarterly capacity review for the provider-side of the fleet.

A good deliverable is read by a reviewer who hands it back and says "I believe this plan would survive next quarter's growth without a 429 incident, and if it does not, I know who pays the error-budget cost."

## Prerequisites

- Chapter 04 of this module.
- A workload or product whose traffic shape against a hosted provider you know or can plausibly represent. If you do not have real numbers, use the illustrative chat workload from chapter 04 §4.3 as a starting point and change the specifics to fit your context.
- Current provider documentation open in another tab. Quota tier structures, reservation product names, and residency primitives change; the plan must cite the current state.

## Problem statement

Pick one workload and one provider. Author the quota plan in eight sections corresponding to the chapter 04 §6 artifact.

### Section 1 — Workload identification

One paragraph. Name the product, surface (chat / RAG / agent / batch), region(s), model version(s), and the ingestion label or routing key used to identify this traffic.

### Section 2 — Forecast

Project the quarter-forward demand on the provider:

- Baseline (p90 of hourly rate over the last 28 days).
- Peak (p99 of hourly rate, or the forecast peak from exercise 02).
- Split into ITPM (input tokens per minute) and OTPM (output tokens per minute) if the provider distinguishes (Anthropic and OpenAI do; Google / Bedrock / Azure often report a single TPM).
- Concurrent-request estimate under Little's Law using the average per-request latency.

### Section 3 — Reservation

Specify the reservation product and sizing:

- Provider's reservation product name (OpenAI Scale Tier, Anthropic Priority Tier, Vertex AI Provisioned Throughput, Bedrock Provisioned Throughput in MU, Azure PTU — current name at time of writing; refresh from docs).
- Units and commitment term.
- Expiry / renewal date.
- Renewal owner (person or team).
- Expected cost under current provider pricing (bracket with "as of date X" to acknowledge pricing drift).

### Section 4 — Burst pool

Specify the burst (on-demand / standard) pool:

- Provider's current standard tier RPM / ITPM / OTPM for your account at your usage tier.
- Projected burst demand above baseline.
- Headroom percentage (burst-pool-ceiling / projected-burst).

### Section 5 — Fall-through

Document what happens when the primary provider 429s persistently:

- Secondary provider (if available) with its own quota plan reference.
- Secondary region (for providers that support cross-region inference profiles — Bedrock, Azure Global Standard).
- Degraded-mode path: shorter prompts, smaller model, reduced feature set.
- Explicit annotation: does the fall-through cross a data-residency boundary? If yes, named compliance approval or named restriction.

### Section 6 — Residency annotations

For each pool (reservation, burst, fall-through), state the data-residency zone it serves (EU, US, global) and any contracts that pin traffic to a zone (GDPR, HIPAA BAA, customer contract clause X).

### Section 7 — Reservation calendar

A table with columns: provider, product, start date, expiry date, renewal decision date (typically ≥ 30 days before expiry), owner. Include every reservation in play for this workload.

### Section 8 — Sensitivities and risks

Three to five concrete risks:

- Growth exceeds forecast by factor X → reservation is undersized by Q+1.
- Provider tier promotion restarts the standard-tier ceiling (relevant for OpenAI / Anthropic).
- Provider deprecates the model version → reservation must be renegotiated against the successor.
- A residency contract tightens → fall-through becomes invalid; the burst pool must grow to compensate.
- Co-tenant inside your own org surges → gateway shaping needs tightening.

Each risk has (a) the trigger, (b) the impact, (c) the mitigation.

## Requirements

1. **Numbers per provider axis, not aggregate.** RPM, ITPM, OTPM, concurrent requests. If the provider does not distinguish all four, state which they enforce.
2. **Reservation product named explicitly** with its commitment term and units.
3. **Fall-through path is concrete** — "fall through to Bedrock cross-region EU-wide inference profile" is specific; "we have fallbacks" is not.
4. **Residency annotations on every pool.** Including the fall-through.
5. **Reservation calendar with renewal dates and owners.** Dates are absolute, not relative.
6. **At least three concrete risks**, each with a trigger, impact, and mitigation.
7. **Current provider documentation cited** — URL and date-of-check.

## Starter guidance

- Provider quota models change every quarter. The plan is a snapshot — note the date-of-check prominently so a reader a year from now knows how fresh the numbers are.
- Co-tenancy inside your own org (chapter 04 §3.1) is the most-often-missed risk. If your workload shares an OpenAI organisation with other products, name the gateway-side shaping that keeps the surge of product A from starving product B.
- For Azure AI Foundry specifically, the plan is denominated in PTU units, not raw TPM. The PTU-to-TPM conversion depends on the model; use Azure's published conversion table.
- For Bedrock, the data-zone restriction on cross-region inference profiles is documented. Verify the current set of zones — EU, US — before relying on it for a residency-constrained workload.
- The quota plan is a different document than the vendor-risk plan (`mod-107`). Keep them separate; cross-reference where they overlap (e.g., "vendor-risk plan for Anthropic owns the deprecation calendar; this plan consumes it").

## Acceptance criteria

- Markdown document is ≤ 1800 words.
- Numerical forecast for ITPM / OTPM / RPM / concurrent requests in section 2.
- Reservation product named with commitment and units in section 3.
- Burst-pool ceiling and current headroom in section 4.
- Fall-through path is concrete (not "we have redundancy").
- Residency annotations present on all pools including fall-through.
- Reservation calendar table with absolute dates and named renewal owners.
- Three or more concrete risks with trigger, impact, and mitigation each.
- Provider docs cited with date-of-check.

## Stretch goals

- **Multi-region quota plan.** Repeat sections 2–5 for each region your workload serves. Note which providers support cross-region pooling and which require per-region reservation.
- **Multi-provider plan for a single workload.** Model the workload as routed across two providers (e.g., 70% Anthropic, 30% Bedrock-Anthropic) with the gateway's routing logic documented. What's the fall-through if one provider degrades (`mod-105` chapter 07)?
- **Cost comparison.** For the reservation choice, compare committed-spend of reserved vs. expected on-demand spend over the commitment period. Name the break-even utilisation.
- **Quota-poll alerting.** Design the alert that fires when the burst-pool headroom drops below 20%. What's the signal? Where does the alert route? What's the response playbook?
- **Co-tenancy audit.** If your organisation has multiple products on one provider's quota pool, enumerate them with their current usage share. Propose a per-product budget that sums to ≤ the pool's ceiling with headroom.
