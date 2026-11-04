# Exercise 01 — Principles of Chaos, Adapted to Your AI Fleet

**Estimated effort:** 2 hours · **Output:** a signed "AI chaos principles" document for a chosen fleet, plus a one-page gap-list

## Objective

Produce the function's extended principles document — the artifact that commits the chaos program to a specific interpretation of the five canonical Principles of Chaos Engineering plus the AI-only sixth principle from chapter 01 §7 — for a chosen (real or illustrative) AI fleet. The document is signed by the AI SRE function lead and the safety / policy peer and is the input to exercises 02–06.

A good deliverable is a document a would-be chaos operator can read once and run against: every principle names the specific metric, cohort, or threshold the program will enforce. A bad deliverable is a restating of the canonical principles with "for AI" appended to each.

## Prerequisites

- Chapter 01 of this module.
- `mod-103` (SLI / SLO vocabulary) and `mod-104` (observability stack) for the serving-side steady-state metric names.
- `mod-105` (non-deterministic failure taxonomy, judge-pipeline shape) for the quality-side steady-state metric names.
- `mod-107` (fallback contract) and `mod-108` chapter 01 (runbook library) for the integration points.
- Access to or a plausible representation of a model-backed product (routes, providers, observability stack).

## Problem statement

Work in four parts.

### Part 1 — Describe the fleet

In 200–300 words, describe the fleet the principles document is being authored for. Name:

- The routes (e.g., chat, summarisation, structured-extraction, RAG-search).
- The providers in use (self-hosted model on GPUs, hosted provider, hybrid).
- The observability stack (Prometheus / OTel / specific LLM-trace platform, judge-pipeline).
- The fallback contract shape (link the specific `mod-107` chapter 02 document or describe the equivalent).
- The runbook library's current state (link to the inventory from `mod-108` chapter 01 or state that it is being authored in parallel).

### Part 2 — Walk the five canonical principles

For each of the five canonical principles (steady state, real-world events, production first, automate continuously, minimise blast radius), produce **three subsections**:

1. **The canonical statement.** One sentence, paraphrasing principlesofchaos.org.
2. **The AI-specific extension.** The extension that chapter 01 writes, adapted to the specific fleet. Not boilerplate: the metric names, the cohort list, the cadence numbers, and the auto-halt alarms the program will actually enforce on this fleet.
3. **The concrete artifact.** The one or two lines that make the extension executable — e.g., "steady state = `{inference_ttft_seconds_p95, judge_groundedness_score, cost_per_request_usd}` on the baseline-7d window published by `prom://chaos/baseline-7d`."

### Part 3 — Write the sixth principle

Write the function's commitment to the "learnings back to the runbook" principle (chapter 01 §7). Name:

- Which fixture-ledger entry format (from `mod-108` chapter 05) the chaos program will produce.
- Which channel the ledger lives in.
- Who the ledger owner is and how the quarterly review consumes it.

### Part 4 — Gap list

A one-page list of the artifacts the chaos program *cannot yet enforce* and what would need to land for each. Representative entries:

- "Quality steady-state baseline: the judge-pipeline runs on 20% of production traffic; the baseline-7d is not yet persisted in Prometheus. Blocked on a pipeline change by `ai-eval-engineer`."
- "Auto-halt on cost spike: per-request cost attribution exists per-route but not per-session; `rb-tool-call-cascade` experiments cannot auto-halt on per-session cost until that lands."
- "Production tier for cache-poisoning experiments: not enabled; moved to tier-2 (staging) per policy."

## Requirements

1. **The fleet is named, not generic.** A reader should be able to tell which routes, which providers, which observability stack.
2. **Every principle has the three subsections.** Canonical, extension, artifact.
3. **The extension subsections cite metric names, cohorts, cadences.** No hand-waving.
4. **The sixth principle is committed.** Fixture format, ledger location, owner, review cadence.
5. **The gap list is honest.** At least three real gaps, each with the blocker and the owning peer.
6. **The document is signed.** Function lead and safety peer (names or role-handles, not "TBD").
7. **No adversarial / jailbreak / model-extraction content.** If a draft wanders into those, trim them; they belong to `ai-infra-security`.

## Starter guidance

- Begin with the fleet description. Vagueness there cascades into vagueness everywhere else.
- Steady state is the hardest section for AI fleets because the quality baseline is often not yet persisted. Be explicit about the shape even if the artifact is on the gap list.
- Production-first is the section where the safety peer's signature matters most. The tier-1 / tier-2 split and the no-chaos list are the parts they will read closely.
- Keep the extension subsections short. One paragraph per principle is often enough; the point is to be concrete, not long.
- The gap list is not a weakness; it is the document's honesty. A principles document with no gap list is a document nobody has read critically.

## Acceptance criteria

- A single document (≤ 2000 words) titled `chaos-principles-<fleet-slug>.md` in the function's documentation repo or the exercise workspace.
- All five canonical principles present with the three subsections each.
- The sixth principle explicitly written with artifact, owner, cadence.
- The gap list has ≥ 3 entries with blocker and peer owner.
- Signed by the function lead and the safety / policy peer.
- No content that strays into adversarial-robustness or model-eval-robustness territory.

## Stretch goals

- **Cross-reference every principle to a chapter-03 experiment.** The "real-world events" principle lists the first five experiments the program will run; the "production first" principle names which ones are tier-1 and which tier-2.
- **Draft the chaos-program policy stub.** Chapter 05 §3's policy document is the natural follow-on. The principles document is its preamble; a one-page stub of the full policy is the next artifact.
- **Dry-run the gap list with the owning peers.** Walk each gap to the peer named as the blocker and confirm the blocker is correctly stated. The surface of corrections is often the most useful finding.
- **Translate the principles into a one-page on-call briefing.** The runbook `rb-chaos-experiment-active` from chapter 02 §6 reads this briefing for its "what is an experiment" paragraph.
