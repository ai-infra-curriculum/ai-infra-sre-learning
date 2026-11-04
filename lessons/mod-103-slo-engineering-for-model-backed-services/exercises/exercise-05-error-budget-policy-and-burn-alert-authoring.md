# Exercise 05 — Error-Budget Policy and Burn-Alert Authoring

**Estimated effort:** 3 hours · **Output:** one Markdown file (~2000 words) — the full error-budget policy document plus the burn-alert configuration

## Objective

Produce the complete error-budget policy document from chapter 05 section 9 for the SLO stack you have been building in exercises 01–04. Then produce the Prometheus-shaped burn-rate alert configuration per SLI from chapter 06 section 5. The result is the operable artifact the on-call rotation reads during an incident — not a design discussion, not a conceptual essay.

Chapter 05 is the prerequisite. If you have not read it, do so before starting.

## Prerequisites

- Exercises 01, 02, 03, and 04 completed. (Your SLO stack must be complete and composed.)
- Chapters 05 and 06 of this module.
- Working knowledge of multi-window / multi-burn-rate alerting ([SRE Workbook, Chapter 5](https://sre.google/workbook/alerting-on-slos/)).

## Problem statement

You have a composed SLO stack across latency, quality, streaming-availability, and cost. The on-call rotation from `mod-102` chapter 03 exists but has no written policy against your specific SLIs. Author the policy and the alert config.

Author a Markdown file with these two parts.

## Part A — the policy document

The eight-section policy from chapter 05 section 9. Treat each section as a 2–4 paragraph artifact the primary on-call reads at 03:00.

### Section 1 — SLO surface

Compact restatement of the stack from exercises 01–04. No new SLIs; refer to those documents for detail. A single table is sufficient: SLI name, target, window, burn-rate-alert severity.

### Section 2 — Freeze-targeting map

The per-SLI freeze map from chapter 05 section 2. One row per SLI. Columns: SLI, change types frozen on burn, who authorizes exceptions (usually the incident commander).

Address every SLI in your stack, including the cost SLIs (which do *not* freeze).

### Section 3 — Allocation rule

Canary / feature / refactor split for the quarter. Justify any deviation from the 40 / 40 / 20 default. If you are in a different platform phase (early rollout, reliability sprint, cost-tightening quarter), the numbers may differ; defend them.

### Section 4 — Burn bands

The four-band table from chapter 05 section 2. Columns: budget-remaining band, canary allowed, feature allowed, refactor allowed, additional required action.

Be operable. "Canary allocation reduces by half" is vague. "At most one canary deploy per week; no new model rollouts; no new prompt templates; existing canaries continue" is operable.

### Section 5 — Correlated-burn rule

Explicit rule per chapter 05 section 4 case 3. Required pieces:

- The correlation window (2 hours by default; justify a different window if you pick one).
- The escalation path (IC call, co-investigation with eval peer for correlated quality burn).
- The handoff from single-SLI runbooks to the correlated-burn runbook.

### Section 6 — Eval-peer coordination rule

The two rules from chapter 05 section 5, parameterized for your workload:

1. A gate-passing release that burns the production SLO can be rolled back by on-call without re-invoking the gate.
2. A gate-failing release cannot ship by spending budget.

Name the eval peer's channel (if one exists on your team) or mark the dependency as "to be stood up."

### Section 7 — Exception process

The five-element exception process from chapter 05 section 7: who grants, for how long, what triggers, how logged, how reviewed.

### Section 8 — Review cadence

Monthly SLO review, quarterly allocation refresh, annual policy audit. Name attendees per chapter 06 section 6.

## Part B — the burn-rate alert configuration

Prometheus-shaped alert rules, one YAML block per SLI. Per SLI, three rules: fast-burn (page), medium-burn (page), slow-burn (ticket). Cost SLIs omit fast-burn per chapter 04.

For each rule, include:

- The alert name.
- The PromQL expression (the two-window guard from chapter 06 section 5).
- The threshold multiplier computed against the specific SLO target.
- Labels — severity, slo, surface, runbook URL, co-owner where applicable.
- One-line annotation for the on-call.

The YAML does not have to be deployable against your real cluster; it has to be *readable and correct* for the SLI targets you set in exercises 01–04. A reviewer should be able to drop the rule into a Prometheus config and it should parse.

At minimum, produce rules for five SLIs covering at least three dimensions (e.g., TTFT-cache-miss, hallucination, partial-success, cost-per-1M-output-tokens, structured-output-conformance). If you want to cover the full stack, do so.

## Requirements

- **Policy document has all eight sections.** No TBD, no placeholder.
- **Freeze-targeting map addresses every SLI** in the stack, including cost (does-not-freeze).
- **Burn-band table is operable.** Specific numbers, specific actions, no handwaves.
- **Correlated-burn rule names the correlation window** and the escalation path.
- **Eval-peer coordination rule is concrete.** If no peer exists, the dependency is flagged.
- **Alert YAML parses** (even if not deployed). Thresholds are computed per SLO, not copy-pasted.
- **Every alert has a runbook URL** (can be a placeholder path — e.g., `runbooks/ttft-cache-miss-burn.md` — the point is the alert is never fired without one).
- **Three-minute decision ladder visible.** A new primary on-call reading the policy for the first time should resolve action for the five-step ladder (SLI → band → correlation? → dimension → permitted actions) within three minutes.

## Starter guidance

- Chapter 05 section 8 walks a worked example end-to-end. Use it as the shape; do not copy the numbers.
- The hardest section to write is section 5 (correlated-burn). Spend extra time there. The correlation-window number is a design decision — too tight (30 minutes) and you miss slow-correlated incidents; too loose (8 hours) and you treat unrelated incidents as correlated. Justify your choice.
- For the allocation rule, be realistic. A team shipping its first SLO stack is probably not in equilibrium; the first quarter's split may be 20 / 20 / 60 (heavy reliability investment) until the stack is stable. Note that you expect to re-set in the second quarter review.
- For the exception process, look at how your organization handles emergency-change authority outside the SRE team. Rhyme with that; a standalone escape valve that nobody has seen before will not survive its first use.
- Burn-rate alert thresholds are the most common bug in operable policies. Compute them fresh per SLO. A fast-burn threshold of `14.4 × (1 - 0.999) = 0.0144` on a 99.9% SLO becomes `14.4 × (1 - 0.97) = 0.432` on a 97% SLO — a 30× difference. Copy-pasting one threshold across the stack is the fastest way to a broken alerting system.
- For the alert YAML, model on the chapter 06 section 5 example. The two-window guard pattern (`and` between the long and short window) is non-negotiable; single-window burn-rate alerts are noisy.

## Acceptance criteria

- Markdown file is self-contained and ≤ 2500 words.
- Part A has all eight sections populated.
- Part B has three alert rules per SLI for at least five SLIs.
- Every alert rule has a threshold value computed against its specific SLO (not a generic number).
- A primary on-call can resolve a hypothetical burn (pick any SLI, any band) in three minutes against the policy without asking.
- Correlated-burn rule names the window and routes to the IC.
- Cost SLIs appear in the freeze map as explicit "does not freeze" entries, not omitted.

## Stretch goals

- **Policy-as-code manifest.** Encode the burn-band table from section 4 as a machine-readable YAML or JSON manifest your deploy tooling could consume to auto-gate rollouts at specific bands. Chapter 05 did not require this; the exercise rewards it.
- **Policy walk-through with three hypothetical incidents.** Append a short appendix: resolve three scenarios under the policy — a cost-only burn, a correlated latency+quality burn traced to a provider, and a hallucination-only burn traced to a prompt regression. Each scenario should resolve to a clear action within the three-minute ladder.
- **Peer review simulation.** Have the policy reviewed by someone who does not share your mental model — a product engineer, a classical SRE, the eval peer — and capture one surprising objection in the appendix with your revision.
- **Policy legibility test.** Time how long it takes a reader to find the response to "hallucination rate burning at 40% budget, no other SLIs burning." If longer than three minutes, restructure. A tested policy is better than a comprehensive one.
