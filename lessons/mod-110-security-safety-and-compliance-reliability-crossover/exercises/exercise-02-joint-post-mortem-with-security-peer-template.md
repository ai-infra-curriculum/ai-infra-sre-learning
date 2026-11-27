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

- Header (co-authors, shape, status, window, legal reviewer field). The header reproduces the chapter 02 §2.1 block verbatim as fillable placeholders — field slugs are the parse contract:

  ```
  Incident: <id>
  Shape: A | B | C | other
  Primary surface(s): <reliability | defensive | compliance | multiple>
  AI SRE co-author: <name>
  Security co-author: <name>
  Legal reviewer (if triggered): <name>
  Status: Draft | In review | Published
  Window (UTC): <start> → <stable> → <resolved>
  ```

  The `Shape` field is a single-select from {`A`, `B`, `C`, `other`} — `other` is the parking space for ambiguous first-fifteen-minute classifications, re-classified at the day-1 review per chapter 02 §4.2. The `Primary surface(s)` field is a multi-select from {`reliability`, `defensive`, `compliance`, `multiple`}. Both fields are filled in the retro header, not left blank for the reader to infer.

- Joint narrative section with the observation-vs-inference rule and signature lines. The chapter 02 §2.2 worked example is the shape to imitate: each narrative line pairs a metric / source citation (observation) with a one-sentence inferential clause (implication). Signature lines belong at the end of the narrative, naming both co-authors as the signing roles.
- Three function-specific detail blocks (AI SRE, security, joint) with authoring guidance. The chapter 02 §2.3 voice assignment is the parse contract — reproduced here as the authoring guidance the template embeds (in italics beside each block header, no example data):

  | Block | Voice / owner | Facts it carries |
  |---|---|---|
  | `ai-sre-detail` | AI SRE co-author | Reliability facts: SLO burns, runbook firings, mitigations tried, mitigations that failed. |
  | `security-detail` | Security co-author | Defensive facts: controls affected, controls bypassed, alerts fired, alerts missed, classifier / policy decision-log excerpts for the window. |
  | `joint-detail` | Both co-authors, written to the seam | Shared / interaction facts: for Shape A, the "classifier deploy vs. serving SLO alarm" seam; for Shape B, the "fallback contract vs. defensive-parity requirement" seam; for Shape C, the "ingest discipline vs. serving/defensive surface" seam. |

  Neither block owns the other. The joint block exists only on multi-function incidents and is the natural home for the seam that made the incident joint in the first place.

- Root-cause classification using the extended taxonomy (base categories from `mod-108` chapter 03 §3 plus the chapter 02 §2.4 joint categories, reproduced below as the selectable list).
- Action-item table with mandatory `Owner`, `Owning-function`, `Due`, `Fixture-class` columns. `Fixture-class` draws from the controlled vocabulary reproduced below. The column slugs and controlled vocabularies are the parse contract so a downstream tracker can diff rows against the template:

  | Column | Content |
  |---|---|
  | `id` | Stable identifier, prefixed `AI-` / `SEC-` / `JOINT-` depending on `Owning-function`. |
  | `action` | One-sentence imperative, present tense. |
  | `owner` | Named human (name or role-handle), never `TBD`. |
  | `owning_function` | One of {`AI SRE`, `Security`, `AI SRE + Security`}. A joint owner is permitted only for signing actions (chapter 02 §2.5 rule 1); implementation actions are decomposed into function-owned sub-items. |
  | `due` | ISO date; distance from incident day expressed in days or weeks per chapter 02 §2.5 rule 3 (no quarter-grain dates). |
  | `fixture_class` | One of the nine-slug vocabulary below (or `other` with justification). |

- Lessons block with "Shared" and "Function-specific" subsections. Order is non-trivial: `Shared` comes first per chapter 02 §2.6 — a joint post-mortem whose function-specific blocks swallow the shared block is a document the joint shape failed to produce.

The extended root-cause taxonomy the template exposes as a single selectable list:

```
# Base categories (mod-108 chapter 03 §3)
- model-weight
- kernel
- scheduler
- prompt-or-config
- retrieval
- provider
- agent-cascade
- eval-drift

# Joint categories (chapter 02 §2.4)
- defensive-change-untested-budget
- defensive-change-untested-latency-tail
- fallback-without-defensive-parity
- fallback-timeout-defeats-defensive-heuristic
- ingest-poisoning-cross-surface
- provider-change-cross-surface
- shared-tool-call-path-cross-surface
```

The fixture-class controlled vocabulary the `Fixture-class` column selects from:

```
# Base classes (mod-108 chapter 05 §1)
- alert
- canary-eval-case
- eval-set-entry
- load-test-shape
- chaos-experiment
- runbook-update

# Joint-specific classes (chapter 02 §2.5)
- contract
- gate
- dashboard
```

A root cause or fixture class that does not fit the list is marked `other` with a short justification, and is nominated as a candidate taxonomy entry at the next quarterly joint retrospective (chapter 02 §5).

The template is written as a fillable form — placeholders in `<angle brackets>`, inline guidance in short italics, no example data. Example data belongs in Part 2.

### Part 2 — Worked examples

Produce three worked examples, one per shape, in a companion file `joint-post-mortem-examples.md`:

- **Shape A example.** A defensive-control change caused a reliability regression. Use the archetype from chapter 02 §1.1 or an invented but plausible analogue (classifier deploy, PII-filter rules update, policy-engine rule-set reload).
- **Shape B example.** A reliability remediation weakened the defensive posture. Use the chapter 02 §1.2 archetype or an invented analogue (fallback contract engaged on a provider lacking defensive parity, worker-circuit breaker tripped past a defensive heuristic window, timeout widened to a value that defeats an injection-suppression rule).
- **Shape C example.** A shared root cause crossed both domains. Use the chapter 02 §1.3 archetype (retrieval-index poisoning) or an invented analogue (upstream provider's silent model change crossed both quality and defensive surfaces, agent-tool-contract change produced both a cost cascade and a privilege-escalation path).

Each worked example fills the template end to end — narrative, three detail blocks, extended root cause selected from the Part 1 taxonomy, and five or more action items drawing from the Part 1 fixture-class vocabulary. Each example is at least 600 words of substantive content. Narratives separate observation from inference. Action items have specific owners (names or role-handles), specific due dates (days or weeks from the incident), and specific fixture classes; at least one action item per worked example uses a joint-specific fixture class (`contract`, `gate`, or `dashboard`), because the joint retro is the natural home of those artifacts.

Each worked example must also cover its shape's chapter 02 §3 branch — the must-name facts, must-reference artifacts, and expected action-item classes for that shape, reproduced here as a checklist the retro fills (if an artifact genuinely does not exist, say so explicitly and name the gap as an action item — silence is not an option):

| Shape | Must-name facts (chapter 02 §3) | Must-reference artifacts | Expected action-item classes |
|---|---|---|---|
| A | Specific control that changed (version, deploy time, author); per-hop latency contract in force before the change (§2.1 of chapter 01); measured per-hop latency after the change (histogram link); SLO burn trajectory and budget at mitigation; release process the change followed and gates passed. | Signed budget contract (or its absence); security peer's eval report for the change; AI SRE's SLO burn dashboard for the window. | Budget contract (if missing or stale); canary gate at the defensive deploy level (chapter 01 §2.4); fast-burn alarm on the per-hop budget; joint shared dashboard panel. |
| B | Remediation taken (fallback engaged, timeout widened, control disabled, worker breaker tripped); defensive controls affected and how; scope of the defensive gap in requests / users / data classes / durations; whether a compliance trigger (chapter 03 of this module; chapter 06 of `mod-108`) fires — if yes, legal is a co-reviewer. | Fallback contract (`mod-107` chapter 02) for the route; defensive-parity annex to the fallback contract (may not yet exist — if so, the primary action item); classifier / policy decision log for the window. | Defensive-parity annex authored for the fallback contract; provider-aware defensive stack (if the gap was in provider differences); chaos experiment added to `mod-109`'s provider-fallback-contract suite that verifies defensive parity; pre-committed decision-tree update for the IC runbook (chapter 02 of `mod-108`). |
| C | Upstream artifact that was poisoned / drifted / changed (document, model version, template, tool contract); the two downstream signals (one reliability-coloured, one defensive-coloured) and how long each existed before correlation; the detection path that connected them and the time to correlation. | Ingest / release / provider-change audit trail; both functions' detection dashboards for the window; any user reports or support escalations that preceded detection. | Ingest discipline (validation, sanitisation, quarantine); cross-surface detection (a signal that fires on correlated tilts); shared dashboard panel designed to surface the correlation; joint runbook (`rb-cross-surface-corpus-event`-shaped) added to the `mod-108` chapter 01 library. |

Narratives must respect the chapter 02 §2.2 rewrite rule — any sentence of the form "X was Y because Z" is rewritten into an observation line (metric + source citation) and an inference line. The chapter's worked example (the classifier p95 breach) is the pattern. Individuals do not appear as narrative actors per chapter 02 §6; role handles (`the on-call AI SRE`, `the classifier deploy pipeline`) do. Named individuals appear only in the co-author block and in action-item `owner` cells.

### Part 3 — Commit the workflow

Produce the drafting cadence document the function signs. The document commits to:

- The trigger protocol (when is a joint retro filed, who pages whom).
- Authorship rules (co-authors from both functions, drafting responsibility).
- The day-0 / day-1-2 / day-3 / day-5 drafting cadence from chapter 02 §4.2.
- Publication rules (one document, one URL, one history).
- Legal review step for compliance-triggered incidents.
- The quarterly joint retrospective schedule and its readout format.

Each commitment lands as a named section in the workflow document so the signing leads are signing a contract and not a paragraph. The parse contract (reproduced from chapter 02 §4) — sections and the fields each section commits:

| Section | Required fields |
|---|---|
| `trigger-protocol` | The chapter-06-of-`mod-108` crossover subset that nominates the incident; the specific signals (shapes from §1 of chapter 02) that force a joint-retro page; the rotation the IC pages (both `ai-infra-security` on-call and the AI SRE on-call) within the first hour. |
| `authorship-rules` | Default drafter (function holding primary incident command); co-author from the other function (role, not name); inline-review rule on the shared document; rule that named individuals are co-authors and action-item owners, not narrative actors. |
| `drafting-cadence` | Day 0 — timeline and narrative skeleton (both functions contribute). Day 1–2 — function-specific detail blocks, root-cause draft, candidate action items. Day 3 — joint review with both function leads; root cause classified; action items committed. Day 5 — legal review (if triggered); publication to both functions' retrospective archives. |
| `publication-rules` | One document, one URL, one history (chapter 02 §4.3). Both functions' archives link the same canonical. Action items are mirrored into each function's issue tracker but retain the single canonical ID from the post-mortem. |
| `legal-review` | Trigger conditions (chapter 03 of this module; chapter 06 of `mod-108`). Named legal reviewer role. Timebox for the review. Rule that for incidents without a legal trigger, publication to the two-function archives does not wait on legal per chapter 02 §6. |
| `quarterly-joint-retro` | Scheduled cadence (quarter-grain). Attendees (both function leads; the shared stakeholders from chapter 02 §4.3 distribution list). Readout format — the four questions from chapter 02 §5 (shape distribution, recurring seams, outstanding action items, candidate taxonomy entries). Single-page readout to function leaders as the input to the mod-114 cross-functional leadership cadence. |

The workflow is signed by the AI SRE function lead and the `ai-infra-security` lead. The signatures are themselves a preventive artifact. Signatures list role-handle or name, date of signing, and the review cadence at which the workflow itself is re-ratified (default: annually, at the first quarterly joint retro of the calendar year).

## Requirements

1. **The template is a fillable form**, not a filled example. Placeholders are explicit.
2. **The header reproduces the chapter 02 §2.1 block verbatim** with the eight field slugs. `Shape` is a single-select from {`A`, `B`, `C`, `other`}; `Primary surface(s)` is a multi-select from the four-slug vocabulary.
3. **The three detail blocks carry the authoring-guidance table** — each block's voice, owner, and the fact-class it carries, per the Part 1 reproduction of chapter 02 §2.3.
4. **The three worked examples exercise all three shapes** and fill the template to the depth a real post-mortem would.
5. **Each worked example covers its shape's chapter 02 §3 branch checklist** — must-name facts, must-reference artifacts, and expected action-item classes. Missing artifacts are named as gaps and action items, not silently omitted.
6. **Narratives separate observation from inference.** Any sentence of the form "X was Y because Z" is rewritten into observation (metric + source citation) and inference. Narrative actors are role handles, not named individuals.
7. **Action items have specific owners.** No "TBD owner." Only signing actions may carry a joint `owning_function`; implementation actions are decomposed into function-owned sub-items per chapter 02 §2.5 rule 1.
8. **Action-item table uses the six column slugs** (`id`, `action`, `owner`, `owning_function`, `due`, `fixture_class`). Row IDs are prefixed `AI-` / `SEC-` / `JOINT-` matching `owning_function`.
9. **Action items have due dates in days or weeks**, not quarters. No multi-quarter deferrals in the worked examples.
10. **Every action item has a fixture class** from the six in `mod-108` chapter 05 plus the three joint classes (`contract`, `gate`, `dashboard`). Any `other` value carries a one-sentence justification and is nominated to the quarterly joint retro.
11. **The drafting cadence is a signed workflow** with the six named sections (`trigger-protocol`, `authorship-rules`, `drafting-cadence`, `publication-rules`, `legal-review`, `quarterly-joint-retro`). Not a wish list.
12. **The template references the extended root-cause taxonomy** (the eight base categories plus the seven joint categories from chapter 02 §2.4).
13. **The lessons block puts `Shared` first and `Function-specific` second.** Order is load-bearing per chapter 02 §2.6.

## Starter guidance

- Begin with the template, not the examples. The examples reveal gaps in the template, and iterating on the template once the examples exist is cheaper than iterating on both.
- Shape A examples are the easiest; Shape C is the hardest because shared root causes require two voices in the narrative without privileging either.
- The observation-vs-inference discipline is the single most important narrative rule. Practice it on the worked examples until it is a habit.
- Action items that span quarters are a smell. If the real fix needs a quarter of engineering, decompose it into a stabilising sub-item (days) and a structural sub-item (weeks) so the retro is not deferred into irrelevance.
- The quarterly joint retrospective is the compounding mechanism. The workflow document commits to it even though the exercise does not run it.
- Keep the template short. The function fills this template dozens of times per year; a template that takes two hours to fill is a template that will get skipped.

## Acceptance criteria

- Two documents land in the exercise workspace: `joint-post-mortem-template.md` and `joint-post-mortem-examples.md`, plus the workflow commitment.
- The template has all sections from chapter 02 §2 as fillable placeholders, each using the parse-contract field slugs / column slugs given in Part 1.
- The header block carries all eight fields from §2.1 with the two controlled-vocabulary fields (`Shape`, `Primary surface(s)`) selectable from their reproduced slug sets.
- The three detail blocks are labelled `ai-sre-detail`, `security-detail`, `joint-detail` and carry the voice-and-facts guidance from Part 1 as inline italics.
- The three worked examples cover Shape A, B, C and each is substantive (≥ 600 words of content) with five or more action items.
- Each worked example covers the chapter 02 §3 branch checklist for its shape — must-name facts, must-reference artifacts, and at least one action item in each of the expected action-item classes for that shape (with missing artifacts surfaced as gap-closing action items).
- Every action item in the worked examples uses the six-column schema (`id`, `action`, `owner`, `owning_function`, `due`, `fixture_class`), with `owner` naming a human (name or role-handle), `due` expressed in days or weeks from the incident, and `fixture_class` drawn from the nine-class vocabulary (six base + three joint).
- Every action-item `id` uses the `AI-` / `SEC-` / `JOINT-` prefix matching its `owning_function`. Only signing actions carry a joint `owning_function`.
- Each worked example uses at least one joint-specific fixture class (`contract`, `gate`, or `dashboard`).
- Each worked example's narrative contains no sentence of the form "X was Y because Z" (observation and inference are separated per chapter 02 §2.2), and no narrative actor is a named individual.
- The drafting cadence document is signed by both function leads (names or role-handles), lists each of the six required sections, and names a re-ratification cadence.
- The extended root-cause taxonomy (eight base + seven joint categories) is reproduced in the template as a selectable list, matching the Part 1 reference block.
- The fixture-class vocabulary (six base + three joint) is reproduced in the template as a selectable list, matching the Part 1 reference block.
- The lessons block in the template orders `Shared` first and `Function-specific` second.

## Stretch goals

- **Author a fourth worked example** for the "crossover with compliance trigger" shape (where legal becomes a co-reviewer). This tests the chapter 06 of `mod-108` handoff from inside the joint shape.
- **File the template into the function's post-mortem tooling.** Templates in a doc tree are easier to drift from than templates in the IR tooling; integrate it.
- **Build a joint-retro dashboard.** Count joint retros by shape, by root-cause category, by quarter. The dashboard is the input to the quarterly joint retrospective (chapter 02 §5).
- **Run a tabletop exercise.** With the security peer, walk through a hypothetical Shape A or Shape C event using the template. The exercise is the fastest way to find template gaps.
- **Cross-reference the template to `mod-109` chaos runs.** A joint retro whose root cause is "the chaos program never injected this" produces a chaos-program action item; the template should surface the hook.
