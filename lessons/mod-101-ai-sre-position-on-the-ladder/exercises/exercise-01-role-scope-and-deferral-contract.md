# Exercise 01 — Role Scope and Deferral Contract

**Estimated effort:** 2 hours · **Output:** one Markdown file (the deferral table) + a short route-ten-concerns rubric

## Objective

Produce the *deferral contract* for an AI SRE team you can name — a routing table that resolves every reliability-adjacent concern to a specific owner, inside or outside the team, without debate. This is the Section-5 part of the reliability charter you will author in exercise 03, scoped down so you can land it in one sitting.

A good deferral contract is **testable**. Ten concerns go in; the table resolves them without asking a human. You will do the ten-concern test at the end.

## Prerequisites

- Chapter 01 (what AI SRE owns) and chapter 02 (the peer deferral map) in this module.
- A concrete team context in mind — your current team, a prior team, or a well-documented open-source project (e.g. an inference-platform build-out at a named company with a public engineering blog you have read).

## Problem statement

You have joined (or are standing up) an AI SRE team. Your first real-week deliverable is a one-page deferral table that the product team, the inference-engineer team, the security team, the ML-platform team, and the SRE team's own on-call rotation can all read and agree on.

Concretely: produce a Markdown file with one `## Deferral table` section, structured as a two-column table (*concern → routing target*) with a short qualifier for each row, that covers at least the following seven categories of concern. Expand as needed for the team you are modelling.

- Serving-engine source-code bug
- Kernel / GPU-performance regression
- Security control (guardrail / PII filter / supply-chain)
- Model-artifact handoff (format, metadata, promotion manifest)
- Eval-signal / release-gate
- CI/CD pipeline wiring
- Prompt / tool-definition / UX concern

Each row must name a routing target — a team or role — and a qualifier that disambiguates the row from the *yes* column (what AI SRE owns).

## Requirements

1. **Pick a concrete platform.** Write one paragraph at the top naming the platform, its surfaces (serving fleet, retrieval pipeline, agent orchestration, etc.), and its primary consumers. If you can't pick a concrete platform, use the illustrative platform in chapter 05 ("the internal LLM gateway serving teams X, Y, Z"); be explicit that it is illustrative.
2. **Author the deferral table.** Minimum seven rows across the categories above. Expand with team-specific rows where appropriate — for example, if your platform has a data-pipeline team that owns retrieval-index refresh, add it.
3. **Write the three-question routing rule** at the top of the table. Verbatim from chapter 02 is acceptable; a team-specific rewording is better.
4. **Case-law section.** One worked example of a historically-ambiguous case (either real from your team or an illustrative hypothetical, clearly marked), with the reasoning that resolved it. A good case shows *why* the routing decision is non-obvious.
5. **Ten-concern test.** List ten plausible incoming concerns (not the ones in the Requirements list — invent them for your platform). For each, write the resolved routing target in one line. If three or more of the ten are ambiguous under your table, revise the table until they are not.

## Starter guidance

- Begin from chapter 02's peer map, but *rewrite* it in your team's voice. A table lifted verbatim from a curriculum chapter will not survive the first hostile review from a peer team; a table authored in your own naming will.
- When in doubt about whether a row belongs in your table or inside a peer's, apply the three-question rule from chapter 02. If the fix lives inside a peer craft, route; if not, keep.
- For the ten-concern test, draw from real-world cases you have seen: an on-call page from a recent quarter, a thread in a channel you follow, a public post-mortem from a frontier lab or model-hosting provider. The more realistic your ten, the more the table will be stressed.

## Acceptance criteria

- Markdown file is self-contained and under 800 words.
- Deferral table has at least seven rows, each with an unambiguous routing target.
- Three-question routing rule is stated at the top of the table.
- At least one case-law precedent is included, with the reasoning that resolved it.
- Ten-concern test is included, with the resolved routing target for each of the ten.
- No concern appears both in the "owned" implicit frame of the document and in the routing table — the two sides are disjoint.
- Reviewer (another learner, or yourself the next day) can read the file in ten minutes and route all ten test concerns without asking a question.

## Stretch goals

- **Per-concern severity dimension.** Add a third column — *severity if mis-routed* — ranking each row from low to critical. Use this to highlight which routing decisions are highest-leverage to get right.
- **Reverse routing.** Produce the mirror table from the perspective of one named peer team (e.g., `ai-infra-inference-engineer`): what *they* route to *you*. A healthy platform has both sides of the mirror.
- **Charter-boundary PR.** If you have a real team, open a draft PR against your team's internal docs with the deferral table as a candidate section. Note the review comments.
