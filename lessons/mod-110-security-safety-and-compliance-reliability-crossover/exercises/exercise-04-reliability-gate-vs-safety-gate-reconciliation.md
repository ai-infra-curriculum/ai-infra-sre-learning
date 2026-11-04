# Exercise 04 — Reliability Gate / Safety Gate Reconciliation

**Estimated effort:** 2 hours · **Output:** the joint release checklist, the capability-tier change protocol, the pre-committed decision tree, the joint-dashboard definition, and the escalation path — signed by the AI SRE function lead and the safety / policy peer

## Objective

Produce the function's **gate-reconciliation artifact set** — the five documents from chapter 04 §5 that commit the AI SRE function and the safety / policy peer to a shape where the reliability gate and the safety gate compose cleanly. The set is the thing a release calendar reads before scheduling a model release, and the thing a capability-tier change references when the mitigations list shifts. It is not a safety-framework document (the safety peer authors those) nor a reliability-SLO document (chapter 01 authors those); it is the seam.

A good deliverable is an artifact set a release manager can walk at 9 AM on release day and produce a go / no-go decision from. A bad deliverable is a set of aspirational documents that have no consumer.

## Prerequisites

- Chapter 04 of this module.
- The organisation's safety-tiering document (RSP-equivalent, Preparedness-Framework-equivalent, Frontier-Safety-Framework-equivalent, or the internal analogue). If none is public, draft a short synthetic one based on the three published frameworks for the exercise.
- Exercise 01 (the latency-budget contract) and exercise 03 (the compliance telemetry attestation) — chapter 04's checklists consume those artifacts as evidence.
- `mod-109` chapter 03 (chaos experiments) for the evidence-of-mitigation surface.
- `mod-111` (release engineering) at the chapter-index level — the exercise's output is a release-gate input.

## Problem statement

Work in five parts.

### Part 1 — Author the joint release checklist

Produce the chapter 04 §5.1 checklist for a chosen route / model. The checklist has three columns:

- Operational mitigation (from the framework's requirements for the model's current capability tier).
- Signal / artifact that evidences the mitigation is in effect (dashboard panel, alarm, drill result, chaos-run verdict, PR link).
- Current status (in effect / partially in effect / not in effect, with evidencing artifact link).

At least six mitigations. Each row names a concrete SRE-owned artifact; mitigations without an operational artifact are flagged "not in effect" and tracked in the gap list.

### Part 2 — Author the capability-tier change protocol

Produce the chapter 04 §5.2 protocol document:

- The change-notice template the safety peer uses to notify the AI SRE of a tier change (fields: previous tier, new tier, driving evidence, new mitigations, changed mitigations, target effective date).
- The AI SRE's return-notice template (acknowledgement, implementation plan, expected effective date, required engineering support from peers).
- The committed response window (e.g., 48 hours from change notice to return notice).
- The tracking cadence for in-progress mitigation implementation.

### Part 3 — Fill the pre-committed decision tree

Produce the chapter 04 §5.3 decision tree with at least six entries. Each entry has:

- Trigger (a specific detectable condition).
- Owning function for the primary decision (AI SRE / safety / joint).
- Pre-committed action (what the function does out-of-incident, so the in-incident decision is faster).
- Secondary notification (who is informed).
- Reference to the runbook or runsheet that implements the action.

Representative entries include: safety-classifier false negative detected, reliability regression correlates with safety-telemetry shift, kill-switch drill latency drifts past SLO, new capability discovered in a deployed model, operational mitigation uncommitted at release time.

### Part 4 — Specify the joint dashboard

Produce the chapter 04 §5.4 joint-dashboard definition:

- The dashboard's name and location.
- The panels, each rendering one operational mitigation as a pass / warn / fail state.
- The underlying signals per panel (PromQL / equivalent expressions).
- The dashboard-as-code commitment (the YAML / JSON / Terraform / equivalent that lives in version control).
- The weekly / bi-weekly joint review attendance roster and agenda.

### Part 5 — Publish the escalation path

Produce the chapter 04 §5.5 escalation document:

- The "defer-to-stricter-gate" default rule.
- The named escalation (the cross-functional lead's role-handle).
- The window after which an unresolved joint block escalates.
- The co-authored escalation note template.
- The commitment that no executive override proceeds without a committed mitigating action.

All five parts are signed jointly by the AI SRE function lead and the safety / policy peer. The signatures bind the function to the reconciliation discipline.

## Requirements

1. **The target route / model is named** and its current capability tier is stated.
2. **The joint release checklist has ≥ 6 mitigations** with concrete evidencing artifacts.
3. **The change-notice and return-notice templates are fillable forms**, not aspirations. Response window is a committed number.
4. **The decision tree has ≥ 6 entries**, each specific and actionable. No "we'll figure it out" entries.
5. **The joint dashboard is specified in code.** PromQL expressions (or equivalent) for each panel. Panels live in version control.
6. **The escalation path is a single document** with the four elements from chapter 04 §5.5.
7. **All five artifacts are signed** by both function leads (names or role-handles).
8. **No safety-framework authoring.** The exercise references the framework; it does not write it.
9. **No reliability-SLO re-authoring.** The exercise references the latency-budget contract from exercise 01; it does not re-derive it.

## Starter guidance

- The joint release checklist is the most load-bearing artifact. Fill it for a specific route first; the other four artifacts are structurally simpler once the checklist is grounded.
- When filling the checklist, be honest about which mitigations are "not in effect" today. A gap column is better than inflation.
- The capability-tier change protocol is the artifact that prevents a surprise at the next tier change. Draft it when there is no surprise.
- The decision tree is best authored by walking recent incidents (the function's own, or public post-mortems) and asking "what would we have pre-committed about this?" Each walked case becomes an entry.
- The joint dashboard is the surface both functions read weekly. A dashboard that is defined in text but not in code drifts.
- The escalation path should feel uncomfortable — it exists because the normal path can fail. If it looks comfortable, it is probably ornamental.

## Acceptance criteria

- Five documents in the exercise workspace: `joint-release-checklist.md`, `capability-tier-change-protocol.md`, `pre-committed-decision-tree.md`, `joint-dashboard.yaml` (or equivalent), `escalation-path.md`.
- The joint release checklist has ≥ 6 rows, each with a concrete evidencing artifact.
- The change-notice and return-notice templates are complete fillable forms with a committed response window.
- The decision tree has ≥ 6 entries, each with trigger / owner / action / notification / runbook reference.
- The joint dashboard is specified as code with PromQL-equivalent expressions for every panel.
- The escalation path has the four chapter 04 §5.5 elements.
- All five artifacts are signed by both function leads (role-handles acceptable).
- `<!-- needs-research: ... -->` comments used for specific framework claims that cannot be verified at time of writing.

## Stretch goals

- **Walk the three worked examples from chapter 04 §6** with the artifact set. The exercise is informal; the findings are template gaps.
- **File the joint release checklist into the release pipeline.** Chapter 04 §5.1 is only effective if a release pipeline consults it; the stretch is to wire it as a release-gate check.
- **Stress-test the capability-tier change protocol.** With the safety peer, run a hypothetical tier change through the protocol and confirm the response window is achievable.
- **Author the quarterly joint-review agenda.** The standing review from chapter 04 §5.4 benefits from a standing agenda; draft the first version.
- **Extend the escalation path to compliance.** For incidents where the compliance surface (chapter 03) escalates in parallel, the escalation path and chapter 03 §6 interact; document the interaction.
