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

Choose five runbooks from the twenty described in chapter 01 §3, covering at least three of the categories named in chapter 01 §2 (serving-path, quality-path, infrastructure-path). The three categories are the fixed grouping from chapter 01 §2 — reproduced here so your selection can be checked against it:

| Category | Runbooks (chapter 01 §3) |
|---|---|
| `serving-path` | `rb-ttft-regression`, `rb-tpot-regression`, `rb-cache-hit-cliff`, `rb-preemption-spike`, `rb-structured-output-break`, `rb-streaming-stall` |
| `quality-path` | `rb-hallucination-rate-spike`, `rb-toxic-output-blast`, `rb-refusal-rate-spike`, `rb-tool-call-cascade`, `rb-embedding-drift`, `rb-judge-model-drift`, `rb-eval-set-poisoning`, `rb-retrieval-index-staleness` |
| `infrastructure-path` | `rb-provider-degraded-fallback`, `rb-gpu-driver-ecc-xid`, `rb-nccl-hang-straggler`, `rb-weight-corruption`, `rb-model-swap-rollback`, `rb-cost-blow-up` |

A balanced default set:

- Serving-path: `rb-ttft-regression` and `rb-cache-hit-cliff`.
- Quality-path: `rb-hallucination-rate-spike`.
- Infrastructure-path: `rb-provider-degraded-fallback` and `rb-gpu-driver-ecc-xid`.

Alternative sets are acceptable if they still cover the three categories. Justify the choice briefly in the cover note.

### Part 2 — Author the runbooks

Each runbook is a Markdown file in a directory named `runbooks/` (or an equivalent location in the learner's environment). Each file begins with the YAML header from chapter 01 §4 (reproduced below as the parse contract) and includes all seven sections from chapter 01 §1 populated with environment-specific content:

```yaml
---
runbook_id: rb-<slug>
title: <human-readable title>
symptom: <one-sentence symptom with the specific signal name>
severity_default: <SEV-1 | SEV-2 | SEV-3>
severity_ladder:
  sev2: <condition that escalates to SEV-2>
  sev1: <condition that escalates to SEV-1>
peer_owner: <peer-role-slug, not a person>
related_runbooks:
  - rb-<slug>
mod_105_class: <recognise-and-route class this maps to>
fixture_backfeed: [<alert | canary-prompt-set | eval-case | load-shape | chaos-experiment | runbook-amendment>, ...]
last_reviewed: <YYYY-MM-DD>
---
```

- **Header.** All fields above populated. `peer_owner` is a rotation slug, not a named human. `mod_105_class` is one of the recognise-and-route classes from `mod-105`.
  - `peer_owner` draws from the chapter 07 peer-role slug set (reproduced from chapter 01 §3 usage — pick one; compound owners go in the handoff section, not the header): `ai-infra-inference-engineer`, `ai-infra-performance`, `ai-infra-platform`, `ai-infra-pipeline`, `ai-infra-ml-platform`, `ai-eval-engineer`, `ai-risk-engineer`, `ai-engineering`, `finops`, `serving-gateway-owner`.
  - `mod_105_class` draws from the ten failure classes in `mod-105` chapter 01 §2: `silent-output-drift`, `hallucination-rate-spike`, `toxic-output-blast`, `refusal-rate-spike`, `prompt-cache-cliff`, `retrieval-freshness-regression`, `agent-tool-call-cascade`, `judge-model-drift`, `provider-degraded-mode`, `policy-safety-regression`. If the runbook class has no matching `mod-105` entry (the infrastructure-path hardware runbooks typically do not), set the value to `n/a` and name the grounding chapter in the body.
  - `fixture_backfeed` draws from the six-class vocabulary in chapter 05 §1: `alert`, `canary-prompt-set`, `eval-case`, `load-shape`, `chaos-experiment`, `runbook-amendment`. List 1–3 candidates; the specific entries land in the Fixture back-feed section.
- **Detection signals.** Specific metric or query names. Not "the time-to-first-token metric" — the exact name, dashboard panel, or PromQL / LogQL / DataDog / Honeycomb-shaped query. Each signal entry carries the four sub-fields from chapter 01 §1.2 as its parse contract:

  | Sub-field | Content |
  |---|---|
  | `name` | Exact metric / query identifier (e.g., `inference_ttft_seconds`, not "TTFT"). Mark `(proposed)` if not yet in the stack. |
  | `shape` | Threshold or anomaly shape — absolute value, relative to baseline, z-score, or EWMA burn. |
  | `panel` | Durable dashboard-panel link (named panel URL, not a Grafana ad-hoc URL). |
  | `expected_lag` | Expected delay between root cause and signal firing. |

- **Confirm checklist.** Four to six specific questions the on-call answers yes / no / unknown. Each row has the question, the system consulted, and the default answer space:

  | Sub-field | Content |
  |---|---|
  | `question` | Specific yes/no/unknown question (scope, correlation, baseline, upstream, downstream — the five from chapter 01 §1.3). |
  | `system` | The named system the on-call opens to answer it (dashboard, status page, deploy-event log, vendor ingestor, support queue). |
  | `answer_space` | `yes` / `no` / `unknown`. Free-form answers are not a checklist. |

- **Mitigation menu.** A ranked list of 3–5 mitigations. Each entry carries the four tags from chapter 01 §1.4:

  | Sub-field | Vocabulary |
  |---|---|
  | `blast_radius` | `user` (user-visible) · `dev` (dev / internal only) · `fleet` (fleet-wide). |
  | `reversibility` | `one-command` (single revert / breaker-close) · `coordinated-unwind` (requires peer handoff to reverse). |
  | `preconditions` | The signal or state that makes the mitigation safe (e.g., "prior version's eval score still green"). |
  | `time_to_effect` | Expected propagation time (e.g., `30 s` for a router breaker, `2–3 min` for pod restart, `10 min` for model-swap). |

  Rank by lowest-blast-radius-first, then one-command-first. Roll-back as the first entry is almost always wrong for AI incidents — see Starter guidance.

- **Handoff.** The specific rotation to page (by role, not by name), the structured payload shape to send them, and the explicit severity-escalation conditions. The payload shape is the chapter 01 §1.5 contract:

  ```yaml
  handoff_payload:
    incident_id: <IC-assigned ID>
    runbook_id: rb-<slug>
    symptom_window: <start_ts .. end_ts, UTC>
    scope: <routes / models / regions / cohorts affected>
    correlated_change: <deploy / config / vendor event in the window, or null>
    confirm_state: <summary of yes/no/unknown answers>
    mitigations_tried: [<mitigation_rank>, ...]
    next_decision: <the question the peer must answer>
  ```

  Severity-escalation conditions are explicit: name the metric / state transition that moves SEV-3 → SEV-2 or SEV-2 → SEV-1.

- **User communication.** A one-paragraph status-page template with named placeholders, or an explicit statement that this class does not require user-facing communication. For safety-adjacent runbooks (`rb-toxic-output-blast`, `rb-weight-corruption`), note that legal / PR owns the externally visible statement per chapter 01 §3.6 and chapter 06.
- **Fixture back-feed.** A one-line note gesturing to chapter 05's six-class decision tree with candidates specific to this runbook class. Name each candidate by its chapter 05 §1 class (`alert`, `canary-prompt-set`, `eval-case`, `load-shape`, `chaos-experiment`, `runbook-amendment`) and the surface it lands on.

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
- Coverage of serving-path, quality-path, and infrastructure-path categories, with at least one runbook per category drawn from the Part 1 category table.
- Every YAML header populates all ten fields from the §4 contract. `peer_owner` uses one of the ten chapter-reproduced peer-role slugs. `mod_105_class` is one of the ten `mod-105` chapter 01 §2 class slugs or the explicit `n/a` value. `fixture_backfeed` uses only the six class slugs from chapter 05 §1.
- Every detection signal cites a real metric / query / panel identifier, or an identifier marked `(proposed)` with a matching gap entry in the cover note. Each signal entry includes all four sub-fields (`name`, `shape`, `panel`, `expected_lag`).
- Every confirm-checklist row has the three sub-fields (`question`, `system`, `answer_space`) and the answer space is restricted to `yes` / `no` / `unknown`.
- Every mitigation carries all four tags using the controlled vocabulary above: `blast_radius` ∈ {`user`, `dev`, `fleet`}, `reversibility` ∈ {`one-command`, `coordinated-unwind`}, `preconditions`, `time_to_effect`.
- Mitigation entries are ranked lowest-blast-radius-first. Roll-back appears as the first entry only when a correlated recent deploy is named in the preconditions.
- Every runbook includes the handoff payload block with all eight keys from the §1.5 contract and names the severity-escalation metric / state transition explicitly.
- Cover note that names the environment, the observability stack, the gaps, and the introduction plan.
- No "TBD" or placeholder content in any of the five runbooks.
- Runbook IDs are stable, kebab-case, and prefixed `rb-`.

## Stretch goals

- **CI enforcement.** Write a small test (Python, shell, or the environment's CI language) that asserts every alert in the alert rules references a `runbook_id` present in the directory. Running this on PR catches alert / runbook drift.
- **Walk a real incident.** Pick a real past incident (from the function's history or a dissected public post-mortem) and demonstrate that one of your five runbooks would have been the right one, with a hypothetical IC execution trace.
- **Chaos-verify one runbook.** For one of the five, sketch a chaos experiment (`mod-109`) that would exercise the runbook's confirm checklist and mitigation menu in a staging environment.
- **Multi-persona dry-run.** Have a colleague who is not the author read one of the runbooks and execute the confirm checklist against a fake alert payload. Note where the runbook required clarification; revise.
- **Localise the status-page templates** into the languages the product supports. The communication section is often dropped from non-English surfaces; the stretch surfaces that gap.
