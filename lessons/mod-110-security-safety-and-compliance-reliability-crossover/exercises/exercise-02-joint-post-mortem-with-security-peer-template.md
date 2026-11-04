# Exercise 02 — Joint Post-Mortem with the Security Peer: the Template

**Estimated effort:** 2 hours · **Output:** the function's joint post-mortem template (one artifact), a worked Shape A / B / C example on illustrative facts, and the committed drafting cadence

## Objective

Produce the **joint post-mortem template** the function will file against for crossover-shaped incidents, and demonstrate it with one worked example per shape (A, B, C) on an illustrative incident. The template is the artifact that commits both functions — AI SRE and `ai-infra-security` — to a shape where the shared facts are told once, the function-specific facts are told in each function's voice, the root cause is classified on a joint taxonomy, and action items are ownership-tagged with due dates in days or weeks.

The joint template does not replace the single-function post-mortem template from `mod-108` chapter 03; it extends it. A bad deliverable is a parallel-universe template that forks from the single-function one. A good deliverable composes cleanly.

## Prerequisites

- Chapter 02 of this module.
- `mod-108` chapter 03 (blameless post-mortem template) — the base template this exercise extends.
- `mod-108` chapter 05 (regression-fixture back-feed) — the fixture-class vocabulary the action items tag.
- `mod-108` chapter 06 (legal / compliance handoff) — the legal-review step when a trigger fires.
- Exercise 01 of this module — the latency-budget contract referenced by Shape A worked examples.

## Problem statement

Work in three parts.

### Part 1 — Author the template

Produce `joint-post-mortem-template.md` with the structure from chapter 02 §2:

- Header (co-authors, shape, status, window, legal reviewer field).
- Joint narrative section with the observation-vs-inference rule and signature lines.
- Three function-specific detail blocks (AI SRE, security, joint) with authoring guidance.
- Root-cause classification using the extended taxonomy (base categories from `mod-108` chapter 03 §3 plus the chapter 02 §2.4 joint categories).
- Action-item table with mandatory `Owner`, `Owning-function`, `Due`, `Fixture-class` columns.
- Lessons block with "Shared" and "Function-specific" subsections.

The template is written as a fillable form — placeholders in `<angle brackets>`, inline guidance in short italics, no example data. Example data belongs in Part 2.

### Part 2 — Worked examples

Produce three worked examples, one per shape, in a companion file `joint-post-mortem-examples.md`:

- **Shape A example.** A defensive-control change caused a reliability regression. Use the archetype from chapter 02 §1.1 or an invented but plausible analogue (classifier deploy, PII-filter rules update, policy-engine rule-set reload). Fill the template end to end: narrative, three detail blocks, extended root cause, five or more action items tagged with owner / function / due / fixture-class.
- **Shape B example.** A reliability remediation weakened the defensive posture. Use the chapter 02 §1.2 archetype or an invented analogue (fallback contract engaged on a provider lacking defensive parity, worker-circuit breaker tripped past a defensive heuristic window, timeout widened to a value that defeats an injection-suppression rule).
- **Shape C example.** A shared root cause crossed both domains. Use the chapter 02 §1.3 archetype (retrieval-index poisoning) or an invented analogue (upstream provider's silent model change crossed both quality and defensive surfaces, agent-tool-contract change produced both a cost cascade and a privilege-escalation path).

Each worked example is at least 600 words of substantive content. Narratives separate observation from inference. Action items have specific owners (names or role-handles), specific due dates (days or weeks from the incident), and specific fixture classes.

### Part 3 — Commit the workflow

Produce the drafting cadence document the function signs. The document commits to:

- The trigger protocol (when is a joint retro filed, who pages whom).
- Authorship rules (co-authors from both functions, drafting responsibility).
- The day-0 / day-1-2 / day-3 / day-5 drafting cadence from chapter 02 §4.2.
- Publication rules (one document, one URL, one history).
- Legal review step for compliance-triggered incidents.
- The quarterly joint retrospective schedule and its readout format.

The workflow is signed by the AI SRE function lead and the `ai-infra-security` lead. The signatures are themselves a preventive artifact.

## Requirements

1. **The template is a fillable form**, not a filled example. Placeholders are explicit.
2. **The three worked examples exercise all three shapes** and fill the template to the depth a real post-mortem would.
3. **Narratives separate observation from inference.** Any sentence of the form "X was Y because Z" is rewritten into observation + inference.
4. **Action items have specific owners.** No "TBD owner."
5. **Action items have due dates in days or weeks**, not quarters. No multi-quarter deferrals in the worked examples.
6. **Every action item has a fixture class** from the six in `mod-108` chapter 05 plus the three joint classes (contract, gate, dashboard).
7. **The drafting cadence is a signed workflow.** Not a wish list.
8. **The template references the extended root-cause taxonomy** (the eight base categories plus the seven joint categories from chapter 02 §2.4).

## Starter guidance

- Begin with the template, not the examples. The examples reveal gaps in the template, and iterating on the template once the examples exist is cheaper than iterating on both.
- Shape A examples are the easiest; Shape C is the hardest because shared root causes require two voices in the narrative without privileging either.
- The observation-vs-inference discipline is the single most important narrative rule. Practice it on the worked examples until it is a habit.
- Action items that span quarters are a smell. If the real fix needs a quarter of engineering, decompose it into a stabilising sub-item (days) and a structural sub-item (weeks) so the retro is not deferred into irrelevance.
- The quarterly joint retrospective is the compounding mechanism. The workflow document commits to it even though the exercise does not run it.
- Keep the template short. The function fills this template dozens of times per year; a template that takes two hours to fill is a template that will get skipped.

## Acceptance criteria

- Two documents land in the exercise workspace: `joint-post-mortem-template.md` and `joint-post-mortem-examples.md`, plus the workflow commitment.
- The template has all sections from chapter 02 §2 as fillable placeholders.
- The three worked examples cover Shape A, B, C and each is substantive (≥ 600 words of content).
- Every action item in the worked examples has owner, function, due date (days / weeks), and fixture class.
- The drafting cadence document is signed by both function leads (names or role-handles).
- The extended root-cause taxonomy is reproduced in the template as a selectable list.

## Stretch goals

- **Author a fourth worked example** for the "crossover with compliance trigger" shape (where legal becomes a co-reviewer). This tests the chapter 06 of `mod-108` handoff from inside the joint shape.
- **File the template into the function's post-mortem tooling.** Templates in a doc tree are easier to drift from than templates in the IR tooling; integrate it.
- **Build a joint-retro dashboard.** Count joint retros by shape, by root-cause category, by quarter. The dashboard is the input to the quarterly joint retrospective (chapter 02 §5).
- **Run a tabletop exercise.** With the security peer, walk through a hypothetical Shape A or Shape C event using the template. The exercise is the fastest way to find template gaps.
- **Cross-reference the template to `mod-109` chaos runs.** A joint retro whose root cause is "the chaos program never injected this" produces a chaos-program action item; the template should surface the hook.
