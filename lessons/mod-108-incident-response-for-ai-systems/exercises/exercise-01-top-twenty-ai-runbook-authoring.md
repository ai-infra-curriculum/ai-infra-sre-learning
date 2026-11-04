# Exercise 01 — Top-Twenty AI Runbook Authoring

**Estimated effort:** 4 hours · **Output:** a runbook library with at least five fully-authored runbooks, plus a one-page cover note

## Objective

Produce a usable, environment-grounded fragment of the top-twenty AI-specific runbook library described in chapter 01. The deliverable is not twenty full runbooks — that would be a quarter of operational engineering compressed into an afternoon — but **five runbooks in production-ready detail**, chosen to cover three of the runbook library's three categories (serving-path, quality-path, infrastructure-path). Each runbook follows the seven-section shape (header, detection signals, confirm checklist, mitigation menu, handoff, user communication, fixture back-feed) and is specific to a product fleet real or realistic enough that the on-call could execute it at 3 AM without judgement calls.

A good deliverable is one the learner's would-be on-call peer can read without pre-explanation and be able to execute. A bad deliverable is a generic template filled with "TBD" or "the metric that would be relevant here." The exercise rewards specificity over coverage.

## Prerequisites

- Chapters 01 and 02 of this module (shape and command context for the runbooks).
- Access to (or plausible representation of) a model-backed service with observability signals. If you do not have a real service, use an illustrative RAG-chat product backed by a hosted model (OpenAI, Anthropic, or Vertex AI) with embedding and moderation routes.
- Familiarity with the SLI / SLO stack vocabulary from `mod-103` and the observability signals from `mod-104` so that the runbook's detection section references real metric names.

## Problem statement

Work in three parts.

### Part 1 — Pick the five runbooks

Choose five runbooks from the twenty described in chapter 01 §3, covering at least three of the categories named in chapter 01 §2 (serving-path, quality-path, infrastructure-path). A balanced default is:

- Serving-path: `rb-ttft-regression` and `rb-cache-hit-cliff`.
- Quality-path: `rb-hallucination-rate-spike`.
- Infrastructure-path: `rb-provider-degraded-fallback` and `rb-gpu-driver-ecc-xid`.

Alternative sets are acceptable if they still cover the three categories. Justify the choice briefly in the cover note.

### Part 2 — Author the runbooks

Each runbook is a Markdown file in a directory named `runbooks/` (or an equivalent location in the learner's environment). Each file begins with the YAML header from chapter 01 §4 and includes all seven sections from chapter 01 §1 populated with environment-specific content:

- **Header.** `runbook_id`, title, symptom, default severity, severity ladder, peer owner, related runbooks, mapped `mod-105` class, fixture back-feed hints, `last_reviewed` date.
- **Detection signals.** Specific metric or query names. Not "the time-to-first-token metric" — the exact name, dashboard panel, or PromQL / LogQL / DataDog / Honeycomb-shaped query. If the metric does not yet exist in the chosen environment, name it and mark `(proposed)`. Include the expected lag between root cause and signal.
- **Confirm checklist.** Four to six specific questions the on-call answers yes / no / unknown, each with the system they consult.
- **Mitigation menu.** A ranked list of 3–5 mitigations, each tagged with blast radius (user / dev / fleet), reversibility (one-command vs. coordinated unwind), pre-conditions, and expected time-to-effect.
- **Handoff.** The specific rotation to page (by role, not by name), the structured payload shape to send them, and the explicit severity-escalation conditions.
- **User communication.** A one-paragraph status-page template with named placeholders, or an explicit statement that this class does not require user-facing communication.
- **Fixture back-feed.** A one-line note gesturing to chapter 05's six-class decision tree with candidates specific to this runbook class.

### Part 3 — Cover note

A one-page cover note (≤ 700 words) that explains:

- Which product / fleet the runbooks are authored for (real or illustrative).
- Which observability stack the signal names reference.
- Why the chosen five cover the three categories adequately.
- Which gaps remain — runbooks you would write next, and signals you would need to add to the observability stack before those runbooks could be executed.
- How the runbooks compose with the service's existing on-call runbook library (if any) and how you would introduce them without displacing existing documentation.

## Requirements

1. **Five runbooks, covering three categories.** Serving-path, quality-path, infrastructure-path — at least one from each.
2. **Every runbook has all seven sections.** Not optional.
3. **Detection signals name real-or-proposed metric names.** "The metric for X" is not enough; provide the exact identifier.
4. **Mitigation menu is ranked** by blast radius and reversibility, with pre-conditions and expected time-to-effect.
5. **Handoff is by role, not name.** `ai-infra-inference-engineer-oncall`, not Sam.
6. **YAML header parses.** Treat it as machine-readable; the CI test for runbook-vs-alert coverage reads this.
7. **Cover note is specific.** Not a template; a concrete narrative tied to the chosen product.

## Starter guidance

- Begin with the quality-path runbook — `rb-hallucination-rate-spike` is the one most learners find hardest, because the detection signal is a judge pipeline and the mitigations are capability-reducing. Writing it first calibrates the difficulty of the others.
- Write the fixture back-feed section last. Reading your own draft reveals the real candidates.
- A runbook whose mitigation menu's first entry is "roll everything back" is almost always wrong. Roll-back is a rare correct first move for AI incidents; the state involved (model weights, caches, indices) does not roll back the way container images do.
- A detection signal that no one in the environment can actually observe is not a detection signal. If the metric is not in the stack, write `(proposed)` and name it in the cover note's gap list.
- Steal from the vendor's engineering blogs. If OpenAI's or Anthropic's public write-ups describe a signal they monitored, your runbook can reference an analogous signal you would build.

## Acceptance criteria

- Five Markdown runbooks in `runbooks/` named `rb-<slug>.md`, each with parseable YAML header, all seven sections populated, and the environment-specific content described above.
- Coverage of serving-path, quality-path, and infrastructure-path categories, with at least one runbook per category.
- Cover note that names the environment, the observability stack, the gaps, and the introduction plan.
- No "TBD" or placeholder content in any of the five runbooks.
- Runbook IDs are stable, kebab-case, and prefixed `rb-`.

## Stretch goals

- **CI enforcement.** Write a small test (Python, shell, or the environment's CI language) that asserts every alert in the alert rules references a `runbook_id` present in the directory. Running this on PR catches alert / runbook drift.
- **Walk a real incident.** Pick a real past incident (from the function's history or a dissected public post-mortem) and demonstrate that one of your five runbooks would have been the right one, with a hypothetical IC execution trace.
- **Chaos-verify one runbook.** For one of the five, sketch a chaos experiment (`mod-109`) that would exercise the runbook's confirm checklist and mitigation menu in a staging environment.
- **Multi-persona dry-run.** Have a colleague who is not the author read one of the runbooks and execute the confirm checklist against a fake alert payload. Note where the runbook required clarification; revise.
- **Localise the status-page templates** into the languages the product supports. The communication section is often dropped from non-English surfaces; the stretch surfaces that gap.
