# Exercise 03 — Reliability Charter Authoring for an AI Team

**Estimated effort:** 2 hours · **Output:** one reliability charter — the full seven-section artifact

## Objective

Write the full reliability charter for an AI SRE team, using the seven-section structure from chapter 05. Exercise 01 produced the Section 5 piece (deferral table); this exercise produces the whole document, including the owned-list, the SLO map, the routing rule, and the review cadence.

The charter you write here is the capstone artifact for this module. Everything in the rest of the curriculum (`mod-102` through `mod-114`) is downstream of this charter: the SLOs live inside Section 3, the runbooks live inside Section 4, the vendor contract and release gate live inside Section 4, the chaos calendar and DR plan live inside Section 4. If the charter is clear, the rest of your platform's documents hang off it cleanly.

## Prerequisites

- Chapter 05 (authoring the reliability charter) in this module.
- Exercise 01 completed — the deferral table from that exercise plugs directly into Section 5 of the charter.
- A concrete team context (reuse the one from exercises 01 and 02).

## Problem statement

You are the tech lead of an AI SRE team at a named company (or a documented open-source project, or the illustrative team from the module chapters — any of the three is fine, as long as you commit to one). The team has decided to publish a reliability charter to end the recurring "whose problem is this" discussion that keeps happening after incidents. Write the charter.

The chapter-05 template is the structure. Do not deviate from the seven sections unless you have a specific, defensible reason. (A specific, defensible reason to deviate is *"my platform has three customer tiers with different SLOs, so Section 3 is three tables rather than one."* Not a defensible reason: *"my platform does not have a vendor dependency"* — it does; name it even if it's a self-hosted model, in which case the dependency is on your own training team.)

## Requirements

Produce a single Markdown file with the following sections (exactly from chapter 05):

1. **One-sentence mission.** Rules from chapter 05: one sentence, names the platform, one explicit *does not own* clause.
2. **What this platform is.** Short enumeration of the surfaces. Four to eight items. Anything not on the list is explicitly out of scope.
3. **SLOs the platform publishes.** One line per SLO, with a pointer to the authoritative definition (even if that definition is "to be authored in `mod-103`-shape"). At least four SLOs across latency, quality, availability, and cost.
4. **What this team owns.** Eight to twelve items, each answerable with a specific artifact or page (even if the artifact is "to be authored in `mod-X`-shape").
5. **What this team does not own, with routing target.** The deferral table from exercise 01, cleaned up and integrated.
6. **The routing rule for ambiguous cases.** The three-question rule, plus at least two case-law entries (from exercise 01 or newly authored).
7. **Review cadence and version history.** Quarterly review on the calendar; version 0.1 entry at the bottom with today's date and your author name.

## Starter guidance

- Chapter 05's "common failure modes" section is the pre-flight checklist. Before marking the exercise complete, re-read each failure mode and verify your charter does not fall into it.
- The single most common failure mode in a first-draft charter is *aspirational scope* — claiming to own things the team is not actually staffed for. Honest small scope beats aspirational large scope. If your team is three engineers, do not claim to own chaos engineering on a monthly cadence; name chaos engineering as a planned investment with a target quarter instead.
- For Section 3, do not restate SLO numbers in the charter — numbers change often; charters should change rarely. The charter *commits that* there is an SLO for a dimension and points at the authoritative definition.
- For Section 4, each item should have a verb — "we author X", "we run Y", "we respond to Z within W". A passive noun list ("SLOs", "runbooks", "capacity plan") reads as a wish list, not an ownership statement.
- For Section 6, two case-law entries can be real or illustrative. If illustrative, mark them explicitly (`<!-- illustrative case -->`). Illustrative precedents are still useful for onboarding; they just do not carry the *"this is settled in our team"* weight that real ones do.

## Acceptance criteria

- Markdown file is a single-page charter (target: 500–1500 words).
- All seven sections present in the chapter-05 order.
- Section 1 is one sentence with an explicit *does not own* clause.
- Section 2 names at least four platform surfaces.
- Section 3 lists at least four SLOs across latency, quality, availability, and cost dimensions.
- Section 4 has 8–12 ownership items, each with a verb and pointer to artifact.
- Section 5 is at least seven rows with unambiguous routing targets.
- Section 6 contains the three-question rule and at least two case-law entries.
- Section 7 has quarterly review on the calendar and a version 0.1 entry.
- The ten-concern test from exercise 01 still passes against the integrated Section 5 table.
- A reader unfamiliar with the platform can route a sample incoming concern using only this document, in under ten minutes.

## Stretch goals

- **Peer-review the charter.** Send to a peer learner or a senior engineer and ask them to find the one thing they would push back on at review. Incorporate (or explicitly decline, with a reason) the pushback.
- **Draft the quarterly review script.** The chapter says "quarterly review is on the calendar." Write the agenda: who attends, what the review covers, what the deliverable of the review is. Attach it to the charter as an appendix.
- **Integration map.** Draw a one-page diagram showing how the charter's sections hook into the rest of the curriculum: Section 3 → mod-103, Section 4 runbooks → mod-108, Section 4 chaos → mod-109, Section 4 DR → mod-113, Section 5 → mod-101 (this module), Section 7 → mod-114.
