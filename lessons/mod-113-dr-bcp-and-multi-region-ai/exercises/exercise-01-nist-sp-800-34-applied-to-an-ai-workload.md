# Exercise 01 — NIST SP 800-34 Applied to an AI Workload

**Estimated effort:** 2 hours · **Output:** a NIST SP 800-34 §4 ISCP skeleton for one AI service of the student's choice, plus a one-page "where NIST does not quite fit" annex, plus the §5 design-time checklist from chapter 01 completed for the service.

## Objective

Take NIST SP 800-34 Rev 1 — the US federal standard for IT contingency planning — and apply its seven-step process and ISCP template to a *specific* AI workload. The exercise is design-level, not implementation; the artifact is the annotated plan skeleton that the subsequent exercises will fill in. The point is to force the student to translate the federal-IT vocabulary into the AI-platform vocabulary and to name — explicitly, in writing — the places where the translation does not survive.

## Prerequisites

- Chapter 01 of this module.
- Read [NIST SP 800-34 Rev 1](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final) cover-to-cover, paying particular attention to §3 (planning process) and §4 (ISCP template). Download the document; cite section numbers in the output.
- The student has chosen an AI service to carry through this module's exercises. A real service on the team is preferred; a reference service (an internal chat assistant, a RAG-backed Q&A over a documentation corpus, a code-assistance route) is acceptable if real work is not available. Fix the service for all six exercises.

## Problem statement

Work in five parts.

### Part 1 — Service characterisation

Write a one-page characterisation of the chosen service. Include:

- **Purpose.** What business problem the service solves.
- **Scale.** Rough request rate, latency expectations, geographic footprint.
- **Dependencies.** Model provider(s), retrieval index, feature / metadata stores, tools invoked.
- **State surfaces.** Which of the chapter 03 §1 state classes are in use (conversation / memory, KV, distributed prefix cache, tool-call, retrieval context, trace, config).
- **Residency commitments.** Any explicit regional or residency commitment in contracts, customer agreements, or regulatory regimes.
- **Classification.** Tier-1 / tier-2 / tier-3, with a one-line rationale.

The characterisation is the input to every subsequent part.

### Part 2 — The seven-step process walkthrough

For each of NIST SP 800-34 §3's seven steps (develop policy, BIA, preventive controls, contingency strategies, develop ISCP, testing / training / exercises, maintenance), write one paragraph that states:

- What the step produces as an artifact.
- Who in the function or peer graph would produce it.
- What is currently in place on the student's team against the step (honestly — "nothing" is a valid answer).
- One concrete next action to close the gap.

### Part 3 — ISCP skeleton (NIST SP 800-34 §4 template, AI-adapted)

Produce the six-section ISCP skeleton from chapter 06 §3.1, filled in to the level of "here is what will go in each section" (not the detailed content — that is exercises 02–05). For each section, write:

- The section heading.
- Two to four bullet points on what the section will contain, specific to the student's service.
- A citation to the chapter and section of this module that produces the detailed content.
- At least one open question the student does not yet have the answer to.

### Part 4 — "Where NIST does not quite fit" annex

Write a one-page annex identifying — specifically for the student's service — the places where NIST SP 800-34's implicit assumptions break on an AI workload. Chapter 01 §2.3 lists four classes; the student may add service-specific ones. For each, state:

- The NIST assumption.
- Why it does not quite fit for this service.
- The AI-adapted mechanic that replaces it (citing chapters 02–05).

### Part 5 — The chapter 01 §5 design-time checklist

Complete the five-item checklist from chapter 01 §5 for the service:

- Classification tag.
- Shared-responsibility note (AWS / Azure / Google, whichever is the primary vendor; cite the vendor's own shared-responsibility documentation).
- Chosen DR strategy in AWS vocabulary (backup / restore, pilot light, warm standby, multi-site active-active), with a one-line justification against the RPO / RTO.
- The AI-specific extensions from chapter 01 §4 that apply.
- Framework citations backing each choice (NIST section numbers, vendor-doc URLs).

## Requirements

1. **The service is specific.** "A chat assistant" is not specific; "the internal T&S triage assistant deployed to the EU and US stamps" is specific.
2. **NIST SP 800-34 is cited by section number** at least five times across the output.
3. **Vendor documentation is cited** at least twice (shared-responsibility and chosen-strategy mapping).
4. **The ISCP skeleton covers all six sections** from chapter 06 §3.1.
5. **The "does not quite fit" annex names at least four assumptions** and states a specific replacement for each.
6. **The §5 checklist has no "TBD" entries** — unknowns are written as open questions with a target resolution date.
7. **The output is in Git.** Commit the artifacts; the subsequent exercises will edit them.

## Starter guidance

- The hardest part is the BIA step. The function does not usually have formal access to business-impact numbers; write the step as if you did. If you have access to the actual BIA at your organisation, use it (redact where required).
- The chosen DR strategy from AWS's four should feel obvious once the RPO / RTO are set. If it does not, the RPO / RTO are not yet set correctly.
- The "does not quite fit" annex is an exercise in honest translation. A reader familiar with classical IT contingency planning should learn something new from each entry.
- Do not attempt to write the detailed mechanics in this exercise. Exercises 02–05 do that.

## Acceptance criteria

- A one-page service characterisation.
- A seven-paragraph NIST process walkthrough with a concrete gap-closing action per step.
- A six-section ISCP skeleton with open questions enumerated.
- A one-page "does not quite fit" annex with four or more entries.
- A completed chapter 01 §5 design-time checklist with full citations.
- All artifacts in a Git repository with the service name in the path and the module slug in the parent directory.

## Stretch goals

- **Multi-service mapping.** Repeat Part 3 for a second service on the same team; identify where the two ISCPs share content and where they diverge.
- **ISO 22301 cross-walk.** Write a one-page cross-walk between NIST SP 800-34's seven steps and ISO 22301's clauses (Plan-Do-Check-Act, 6–10). A reader of both standards should be able to use your walkthrough to translate.
- **Framework comparison table.** Produce the three-column table (NIST / ISO 22301 / vendor-specific) for the student's primary cloud vendor; cite the exact section each column references.
- **BIA interview.** Schedule a 30-minute interview with the product owner for the service; produce a one-page BIA draft from the conversation; attach to the exercise output.
