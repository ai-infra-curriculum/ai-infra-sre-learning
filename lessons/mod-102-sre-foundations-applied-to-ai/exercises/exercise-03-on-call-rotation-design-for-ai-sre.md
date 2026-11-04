# Exercise 03 — On-Call Rotation Design for an AI SRE Team

**Estimated effort:** 2 hours · **Output:** one Markdown file (~1500 words) + a hand-off template

## Objective

Design the on-call rotation that enforces the error-budget policy from exercise 02. The output is two artifacts:

1. A **rotation-design document** — shape, roles, escalation ladder, on-call-health metric plan, paging policy — for the team you have been carrying through this module.
2. A **hand-off checklist template** your team will use at every rotation hand-off.

Both artifacts should be operable by a team member reading them cold.

## Prerequisites

- Chapter 03 of this module.
- Exercises 01 and 02 (same workload + same policy).

## Problem statement

You have the SLI stack (exercise 01) and the policy that governs budget spend (exercise 02). You have not yet designed the rotation that executes the policy in real time. This exercise produces that design.

Work against a concrete team size — pick one of the three canonical cases:

- **Case A — small team (4–6 engineers).** Single-region default.
- **Case B — medium team (7–12 engineers).** Single-region with mature pager discipline, or early FTS if overnight volume justifies.
- **Case C — large team (≥ 13 engineers).** FTS-eligible across two or three regions.

Pick the case that matches your workload; if your workload is small (which it probably is), do not fight the math and pick Case A.

## Requirements

### Artifact 1 — Rotation-design document

#### Section 1 — Shape

Name your team size + region strategy. Justify in two sentences against chapter 03 section 1:

- If single-region, how do you cap overnight pages?
- If FTS, how many engineers per region, what is the hand-off cadence?

Explicitly address the "did you tune the pager before splitting?" test from the chapter.

#### Section 2 — Three roles

Per chapter 03 section 2, write:

- **Primary** — pager-holder, response-time SLO (MTTA numbers), triage boundary (when does a page become an incident?).
- **Secondary** — backup response-time SLO, when do they engage, how do they coordinate with the primary during an incident.
- **Incident commander** — rotation shape (shared with other teams? dedicated?), when is the IC paged, what is their authority in the three-way spend decision from exercise 02.

For each role, write a half-paragraph job description your rotation member will read on their first shift.

#### Section 3 — Escalation ladder (model-aware)

A table with eight rows, one per symptom class from chapter 03 section 5. Columns: symptom → SRE-side triage steps → when to escalate → which peer SME to page → how to page them.

Fill in the actual names / channels / rotations for your team — "Slack `#ai-infra-engine-oncall`" or "PagerDuty rotation `inference-engine-sme`," not "the inference-engine peer." If you do not know the real channel, use a placeholder `TBD-<peer>` and note it as a follow-up to resolve.

#### Section 4 — On-call-health metrics

A dashboard spec. For each metric from chapter 03 section 4 (page volume, actionable rate, sleep disruption, cognitive load), write:

- The source metric (where does the number come from — PagerDuty, Opsgenie, a survey).
- The target value.
- The alert / review trigger (when is the pager-tuning sprint called?).
- The owner (usually the SRE team's engineering manager, not the on-call rotation itself).

#### Section 5 — Paging policy

Enumerate the paging policy rules from chapter 03 section 6, specialized to your SLOs from exercise 01:

- Burn-rate alert thresholds (should match exercise 01).
- Multi-window silencing — the second-window requirement for each alert.
- Correlated-signal silencing — the implementation of the exercise-02 reconciliation rule inside the paging layer.
- Provider-status ingestion — which provider status pages are ingested, through what channel.
- Change-window routing — how does a declared deploy / model-promotion / prompt change disable certain alerts for the window.

### Artifact 2 — Hand-off checklist template

A six-section Markdown template matching chapter 03 section 3, ready to be copy-pasted by the outgoing on-call into a durable channel.

- Open incidents.
- Open pages.
- Known-bad state.
- In-flight changes.
- SLO budget state.
- Recent post-incident actions.

Each section should have a placeholder prompt (e.g., "List each open incident with state, next action, next-action owner, expected resolution time") so the outgoing on-call knows what to fill in.

## Starter guidance

- Resist the urge to design FTS before you need it. Case A is the honest answer for most Year-One AI SRE teams, and the pager-tuning work is more valuable than the rotation-splitting work at that stage.
- The escalation ladder rows are a translation of mod-101's deferral map into on-call language. If you have the mod-101 exercise 01 output, lift the peer owners directly.
- The on-call-health metrics are not vanity metrics — they drive the policy decisions on toil-reduction sprints (chapter 01 section 3). Wire them into the same SLO-review ritual as the error-budget policy.
- The hand-off checklist is the shortest artifact but the most-read one. A template that takes 15 minutes to fill in will be skipped; a template that takes 3 minutes will be adopted. Err toward brevity.

## Acceptance criteria

- Markdown file is self-contained and ≤ 2000 words total across both artifacts.
- Rotation shape named with justification.
- Three roles each have a half-paragraph job description.
- Escalation ladder covers at least the eight symptom classes from chapter 03 section 5.
- On-call-health dashboard has specific sources / targets / triggers / owners.
- Paging policy encodes the exercise-02 reconciliation rule.
- Hand-off checklist template is ready to use — no "fill in later" placeholders in required sections.
- A new team member can read both artifacts in 15 minutes and understand how the rotation works.

## Stretch goals

- **Pager-tuning retrospective template.** Author the retrospective format your team will run when actionable-rate drops below 50%. Cite one historical example (real or hypothetical) of what the output of that retrospective looks like.
- **Compensation / sustainability section.** Add a short appendix on shift compensation, overnight-page caps, and PTO / shift-swap policy. This is the part of the rotation most likely to be omitted from the technical design and most likely to determine whether the rotation is sustainable at two years.
- **Cross-team hand-off.** If your team shares incidents with product teams (which it will), author the one-paragraph protocol for *product* ↔ *SRE* hand-off during a cross-team incident.
- **Simulation drill.** Walk two incidents from exercise 02's hypothetical-scenario appendix (if you did the stretch there) through the rotation design. Who pages whom, what does the hand-off look like at shift change if the incident is still open? Any gaps revealed should be closed by revising the design.
