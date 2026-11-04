# Exercise 02 — Error-Budget Policy Authoring for Non-Deterministic Systems

**Estimated effort:** 2 hours · **Output:** one Markdown file (~1500 words) — the policy document itself

## Objective

Author a complete, operable error-budget policy for the workload you picked in exercise 01. The output is a *policy document*, not a design discussion — it should be the kind of artifact the on-call primary reads in three minutes on a page and resolves what to do without further input.

## Prerequisites

- Chapter 02 of this module.
- Exercise 01 (same workload context carries forward).

## Problem statement

Your SLO review has landed and the product organization has agreed to the SLIs from exercise 01. The next deliverable is the policy that says what happens when those SLOs are burning. Author that policy with the seven sections named in chapter 02 section 5:

1. SLO surface (list, with brief restatement from exercise 01).
2. "Bad event" definitions (one per SLI; multi-SLO stack, not composite).
3. Reconciliation rule for multi-SLO burn.
4. Allocation rule for the quarter (canary / feature / refactor split).
5. Trigger thresholds (four bands keyed to budget remaining).
6. Review cadence.
7. Exception process.

## Requirements

### Section 1 — SLO surface

Restate the SLIs from exercise 01 compactly (bullet list is fine). Add a one-line note per SLO identifying which product surface it protects.

### Section 2 — Bad-event definitions

For each SLI, write the bad-event predicate in operational prose — not math. Example: "A latency bad event is a request whose TTFT exceeds 500 ms *excluding* requests throttled at admission (those are counted as valid-not-admitted and attributed to the capacity policy, not latency)."

### Section 3 — Reconciliation rule

Encode the rule from chapter 02 section 3, parameterized for your SLOs. Required pieces:

- What happens when a single SLO is under 50% budget remaining.
- What happens when two or more SLOs are under 50% simultaneously *and* the burn is time-correlated (within 2h or your chosen correlation window).
- What role in your on-call rotation decides this is correlated vs. independent (usually the incident commander).
- Explicit note on the cost SLO's unique behaviour (negotiation signal, not halt signal).

### Section 4 — Allocation rule

For the current quarter, propose a canary / feature / refactor split for the error budget, per chapter 02 section 4. Default: 40 / 40 / 20. Justify any deviation.

### Section 5 — Trigger thresholds

The four-band table from chapter 02 section 4, operationalized for your platform. Required columns: budget band, canary allowed, feature allowed, refactor allowed, additional action required (e.g., "SLO review scheduled within 48h").

Be concrete. "Canary allocation reduces by half" is weak; "Canary deploys limited to one per week, no new model rollouts, no new prompt templates, ongoing canaries continue" is operable.

### Section 6 — Review cadence

Name the three cadences:

- Monthly SLO review (attendees, standing agenda).
- Quarterly allocation refresh (who proposes the new numbers, who approves).
- Annual policy audit (what triggers a rewrite — typically a material platform change: new engine, new provider, new product).

### Section 7 — Exception process

The escape valve. How does an IC grant a time-boxed exception during an incident? How is the exception logged? When does it auto-expire? How does the post-mortem capture the fact that an exception was used?

## Starter guidance

- The policy should be legible to the product team, not only to SRE. Avoid jargon beyond what mod-101 chapter 04's two-column table defines.
- The reconciliation rule is the hardest section — spend extra time there. The correlated-burn case is the one that trips teams in practice because the default response is to treat each SLO's burn as an independent fire.
- The allocation rule must be *in writing before the quarter starts* to avoid being negotiated during outages. If this is your first quarter, mark it as proposed and schedule a mid-quarter retrospective to adjust.
- Trigger thresholds should be executable without human judgement. A primary on-call at 03:00 reading the policy should resolve the action for the band without asking a question.
- For the exception process, look at the way your organization treats emergency-change authority outside SRE (e.g., a product team's "hotfix with post-hoc review" process). The exception process here should rhyme with that; a standalone escape valve that no one has seen the shape of before will not survive its first use.

## Acceptance criteria

- Markdown file is self-contained and ≤ 2000 words.
- All seven sections present and populated.
- Reconciliation rule explicitly addresses correlated-burn routing to the IC.
- Trigger thresholds are concrete and operable (no "canary allocation reduces" without a specific number).
- Cost SLO treated distinctly from latency / quality / availability.
- Reviewer (another learner or yourself the next day) can read the policy in five minutes and resolve the action for a hypothetical burn state without asking a question.

## Stretch goals

- **Hypothetical-incident walk-through.** Append a short appendix with three hypothetical scenarios — a cost-only burn, a correlated availability+quality burn traced to a provider, a quality-only burn traced to a prompt regression — and resolve each under the policy. If any scenario is ambiguous, revise the policy until it is not.
- **Comparative policy.** Produce a one-line comparison against a classical SRE error-budget policy for a CRUD service. What is unchanged? What is new? This reinforces the mod-101 chapter 04 side-by-side.
- **Policy-as-code.** Encode the trigger-threshold table as a machine-readable manifest (YAML / JSON) your deploy tooling could consume to auto-gate rollouts under high-burn bands.
- **Peer-review simulation.** Have the policy reviewed by someone who does not share your mental model (a product engineer, a classical SRE, the eval peer). Capture one surprising objection in the appendix with your revision.
