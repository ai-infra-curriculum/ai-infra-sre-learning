# Exercise 02 — Classical SRE vs AI SRE Side-by-Side

**Estimated effort:** 2 hours · **Output:** one Markdown file — the two-column comparison + annotations + the lateral-transfer checklist

## Objective

Produce a defensible, team-specific version of the classical-SRE-vs-AI-SRE comparison from chapter 04 that an engineer transitioning laterally (either direction) can read and know *exactly* which of their current reflexes survive and which have to be re-authored.

The chapter-04 table is generic. A team's version is specific: it uses your actual SLOs, your actual runbooks, your actual vendor-dependency map. That specificity is what makes the comparison useful for onboarding.

## Prerequisites

- Chapter 04 (classical SRE vs AI SRE) in this module.
- The Google SRE Book and SRE Workbook at reading depth — you do not need to re-read them for this exercise, but you should be able to name the chapter in the book that corresponds to each row.
- A concrete team context (same as exercise 01 or a different one if you prefer).

## Problem statement

Imagine you are the senior engineer running the onboarding session for a lateral hire. They have five to ten years of classical SRE at a web / database / data-pipeline company. They have passed the hiring loop; they are starting Monday. You have one hour with them to walk the gap between their reflexes and the AI stack.

Produce the handout for that hour.

## Requirements

Produce a single Markdown file with the following sections.

1. **Context one-liner.** One sentence naming the platform you are onboarding into (reuse the one from exercise 01 if you like).
2. **Left column — carries verbatim.** At least nine practices, each with:
   - The practice name.
   - A one-line description (what it is).
   - A Google SRE Book / Workbook chapter reference.
   - A one-line note on how it shows up on *your* platform specifically (name an SLO, a runbook, or a ritual).
3. **Right column — re-invented.** At least seven re-authored artifacts, each with:
   - The classical form it replaces.
   - The AI-specific form.
   - A one-line "why the classical form breaks" rationale.
   - A pointer to the module in this curriculum that teaches the re-authored form.
4. **The gap column.** For each row in the right column, add one sentence on the *onboarding gap* — what the lateral hire specifically has to unlearn or re-learn.
5. **The lateral-transfer checklist.** A short checklist (eight to twelve items) the new hire can use to self-assess at the four-week mark: *"am I over the gap on this one yet?"* Each item is yes/no and refers back to a specific row in the comparison.

## Starter guidance

- The left column is mostly an exercise in correct citation. Lift the practice names from chapter 04 and the SRE Book table of contents; one-line descriptions should be self-contained.
- The right column is where the work is. For each row, ask: *what is the exact classical artifact?* and *what is the exact AI-specific artifact?* If you are tempted to write "and adapt for AI", push harder — a lateral hire cannot *adapt* a reflex they have not yet broken.
- The onboarding-gap sentences should be concrete. "Latency SLO" vs. "TTFT/TPOT/ITL stack" has an onboarding gap that reads: *"when your reflex says 'look at request latency', ask instead which of TTFT/TPOT/ITL moved; the single-number instinct will miss two-thirds of the real signal."*
- The lateral-transfer checklist is a tool, not a certificate. Items should be behaviours a four-week hire can honestly self-check — "I can read a dashboard with TTFT / TPOT / ITL tiles and point to the one that would move for a prefix-cache cliff" is stronger than "I understand TTFT / TPOT / ITL."

## Acceptance criteria

- Markdown file is self-contained and under 1500 words.
- Left column has at least nine practices, each with a SRE Book / Workbook reference.
- Right column has at least seven re-authored artifacts, each with a classical-form, AI-form, rationale, and a module pointer.
- Each right-column row has a one-sentence onboarding-gap note.
- Lateral-transfer checklist has 8–12 yes/no items, each tied to a specific row.
- A classical SRE reading this file would leave able to name the three to five most important things they have to re-learn — without the file telling them they are wrong, which they are not.

## Stretch goals

- **Second lateral direction.** Add a mirror section for the *AI → classical-SRE* direction: an AI engineer joining a classical SRE team for a tour. Which reflexes carry (most of the AI-specific detection discipline is a pure *addition* to classical SRE; it does not interfere), which have to be dropped (the entire quality-SLO frame is probably not useful on a database team).
- **Timed table.** Add a third column: "weeks to close the gap" for each right-column row. Use it to inform your onboarding plan — which gaps should be closed in week one, week four, month three.
- **Real artifact reference.** For each right-column row, link to an actual artifact in your platform (an SLO document, a runbook, a dashboard) that demonstrates the AI form. If you cannot find an artifact for a row, that row is a candidate for the next quarter's work.
