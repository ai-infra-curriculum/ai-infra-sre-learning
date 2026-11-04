# Exercise 03 — Compliance Telemetry Hooks for EU AI Act Article 72

**Estimated effort:** 2 hours · **Output:** a signed telemetry attestation (Article 72 slice), a signal inventory tied to the SRE-owned observability stack, and the pipeline-health SLO + runbook for the compliance telemetry pipeline itself

## Objective

Author the function's **Article 72 telemetry attestation** for a chosen (real or illustrative) high-risk AI system. The attestation is the AI SRE's compliance-facing deliverable: a signed, dated document that enumerates the signals the service produces for post-market monitoring, commits the retention, lineage, and query SLO for each, names the consumer peer, and declares the monitoring system's own availability SLO. The attestation feeds the `ai-risk-engineer` peer's Article 72 narrative; the AI SRE does not write the narrative.

A good deliverable is an attestation a legal reviewer can read, confirm against the running service, and rely on as the evidentiary input for an audit. A bad deliverable is a dashboard screenshot or a bulleted aspiration.

## Prerequisites

- Chapter 03 of this module.
- `mod-103` (SLO stack) for the serving-side signals.
- `mod-104` (observability stack) for the plumbing.
- `mod-105` (quality signals) for the model-behaviour half of the inventory.
- `mod-107` (fallback contract) for the vendor interaction signals.
- `mod-108` chapter 05 (fixture ledger) and chapter 06 (handoff) for the incident-lifecycle signals.
- The text of Regulation (EU) 2024/1689 Articles 72 and 73, read at least once at the chapter level.

The exercise assumes the fleet is in scope of the EU AI Act as a high-risk AI system. If the actual product is not, treat it as illustrative and state the assumption in the document.

## Problem statement

Work in five parts.

### Part 1 — Describe the high-risk AI system

In 200–300 words:

- Name the system (product, route, intended purpose per the Act's taxonomy).
- State the applicable Annex III category (or the illustrative category the exercise is using).
- Name the provider / deployer relationship — is the organisation a provider, a deployer, or both?
- Note any adjacent obligations the Act imposes (technical documentation, logging capability, human oversight, accuracy / robustness / cybersecurity requirements) and link where the function's observability intersects them.

### Part 2 — Fill the signal inventory

Produce the chapter 03 §2.2 inventory table for the system, filled. Columns:

- Signal class (serving health / model behaviour / defensive stack / fallback behaviour / data handling / drift signals / incident lifecycle / governance).
- Specific signals (metric names, label sets).
- Source module in the track.
- Short-horizon retention (days, where stored).
- Long-horizon retention (years, where stored, lineage reference).
- Query SLO (bounded-time commitment on a well-formed query over the long-horizon store).
- Current coverage status (operational / aspirational / blocked-on-X).

Every row cites a specific `mod-104` / `mod-105` / `mod-107` / `mod-108` artifact. No invented metric names.

### Part 3 — Author the pipeline-health SLO

Write the SLO contract for the compliance telemetry pipeline itself:

- The "telemetry delivered end-to-end within N minutes" SLI expression.
- The target SLO (e.g., 99.9% of events delivered within 5 minutes).
- The fast-burn and slow-burn burn-rate alarms.
- The pager routing.
- The `rb-compliance-telemetry-gap` runbook: detection, confirm checklist, mitigation menu, handoff (to `ai-risk-engineer` and legal), post-incident fixture commitment.

The pipeline-health SLO is itself part of the attestation; the compliance surface cannot be audited if the audit's data source is intermittently unreliable.

### Part 4 — Commit the retention, lineage, and access matrix

Three subsections:

- **Retention schedule.** For each signal class, the short-horizon retention (operational) and the long-horizon retention (compliance), with legal reviewer sign-off reference.
- **Lineage artifact.** The format of the lineage record (which recording rules, which code versions, which source series produced the long-horizon snapshot). A sample lineage record is included.
- **Access matrix.** Roles that may read compliance telemetry, with purpose statements and the audit-logging path. Role list is signed by legal.

### Part 5 — Draft the attestation

Produce the signed one-page attestation per chapter 03 §2.4, with:

- The §2.4 attestation text, filled.
- Attached: the signal inventory, the retention schedule, the query SLO, the role / access matrix.
- Signatures: AI SRE function lead, `ai-risk-engineer`, legal. The signatures are role-handles if named individuals are not applicable to the exercise.
- The next review date (quarterly or on the regulatory surface's cadence, whichever is tighter).

## Requirements

1. **The system is named.** Even if illustrative, it is not generic — product, intended purpose, Annex category, provider / deployer relationship.
2. **Every signal row cites a specific source.** No invented metrics. Where a signal does not yet exist, the row's status reads "aspirational, blocked on X."
3. **Retention columns distinguish short-horizon from long-horizon.** Both are committed, with storage locations.
4. **The query SLO is quantitative.** A time budget for a bounded query over the long-horizon store.
5. **The pipeline-health SLO is authored** with fast-burn and slow-burn alarms and a working runbook.
6. **The access matrix has roles, purposes, and an audit path.**
7. **The attestation is signed.** Three signatures (names or role-handles).
8. **The document uses the `<!-- needs-research: ... -->` form** for any specific regulatory claim it cannot verify at the time of writing (e.g., the content of a Commission implementing act under Art. 72(3) that has not been published).
9. **No regulator-facing narrative.** The exercise authors the telemetry attestation, not the Article 72 report; the attestation is an input to the report, not a substitute.

## Starter guidance

- Start with the system description. Vagueness there cascades into the inventory.
- The inventory is the main artifact. Every signal is a row; every row is a commitment.
- For signals that do not yet exist, be honest. "Aspirational, blocked on X" is better than listing a signal that is not produced.
- Retention is longer than you think. Legal will frequently ask for the longest plausible window; err long rather than short.
- The pipeline-health SLO is the non-obvious load-bearing part. A compliance stack whose pipeline health is uncommitted is a stack that will gap during an audit window.
- The lineage artifact is what distinguishes a defensible record from a dashboard. Draft it even if it looks like overkill.
- The attestation is the AI SRE's compliance product. One page, signed, dated, versioned. Not a 20-page design document.

## Acceptance criteria

- One attestation document in the exercise workspace, with the §2.4 chapter 03 text filled and three signatures.
- Signal inventory attached, with at least eight signal classes and ≥ 25 specific signal rows across the classes.
- Retention schedule attached, distinguishing short-horizon and long-horizon.
- Query SLO quantified.
- Access matrix attached with ≥ 5 roles, each with a purpose and an audit path.
- Pipeline-health SLO authored with burn-rate alarm expressions and a runbook draft (`rb-compliance-telemetry-gap`).
- `<!-- needs-research: ... -->` comments used for any specific regulatory claim that cannot be verified at the time of writing.
- The next review date is set.

## Stretch goals

- **Extend to the Art. 73 (serious-incident reporting) hook.** Author the trigger protocol that pages legal when the serious-incident definition is approached by a running incident. Chapter 06 of `mod-108` is the base; chapter 03 of this module extends.
- **Extend to a PCCP slice.** If the product is also FDA-cleared AI-enabled device software, add the PCCP-specific signals, triggered-change workflow, and regulatory-affairs handoff.
- **Extend to an SR 11-7 slice.** If the product is also in MRM scope, add the model-inventory alignment, materiality-flag detection, and challenger-model telemetry.
- **Dry-run an audit.** Walk the attestation with `ai-risk-engineer` or a legal reviewer; the questions they ask are the gaps the next revision closes.
- **Build the attestation in version control.** Markdown in a repo with a CI check that the attached inventory references only metrics that exist in the function's observability config. The CI check is the strongest guard against aspirational signals making it into a signed attestation.
