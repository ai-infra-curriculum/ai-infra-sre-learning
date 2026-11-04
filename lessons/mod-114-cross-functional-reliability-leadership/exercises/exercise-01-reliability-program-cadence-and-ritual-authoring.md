# Exercise 01 — Reliability Program Cadence and Ritual Authoring

**Estimated effort:** 3 hours · **Output:** a Git-tracked `programme/` directory containing the five-ritual cadence document, the three operational artifacts (on-call rotation design, deployment-change calendar schema, migration tracker), the RACI document, and the first-90-days plan — all scoped to a specific AI SRE team you name in Part 1.

## Objective

Author the full reliability programme document for a specific AI SRE team. The exercise forces you to translate chapter 01's abstract programme shape into the concrete artifacts your team would actually commit to Git on day one of a team lead's tenure. The exercise does not ask you to *run* the rituals; it asks you to produce the documents from which the rituals can run.

## Prerequisites

- Chapter 01 of this module.
- Mod-101 chapter 01 (the role charter) and chapter 02 (the peer deferral map).
- Mod-103 chapter 04 (the error-budget policy) — the chapter 01 §2.5 and §4 of this exercise read from it.
- Mod-108 chapter 01 (runbook library) and chapter 03 (incident command / on-call) — the on-call rotation design (chapter 01 §3.1 of this module) composes with these.
- Mod-111 chapter 01 (release pipeline overview) and mod-112 chapter 05 (config-pipeline change patterns) — the deployment-change calendar reads both.
- Read the [Google SRE Workbook chapter 8 (On-Call)](https://sre.google/workbook/on-call/) cover-to-cover, paying particular attention to the operational-overload section.

## Context

You have been hired as the first AI SRE team lead at a 400-engineer AI-platform company. The team you lead is six engineers. You run production for two tier-1 AI services (an internal copilot assistant and an external chat-based customer-support product), one tier-2 service (a RAG-backed documentation search), and two tier-3 batch jobs (nightly embedding regeneration and a weekly eval pipeline). Your fleet runs on Kubernetes across two AWS regions. You have hosted-model dependency on two providers and a self-hosted vLLM fleet for the copilot.

No reliability programme exists. The team has dashboards (a Prometheus + Grafana stack), a rotation (informally run on PagerDuty), and runbooks of inconsistent quality. The previous engineering manager left three months ago; incidents have been handled ad-hoc. You start Monday.

## Problem statement

Work in five parts. The deliverables land in a directory `programme/` in your Git repository.

### Part 1 — Team characterisation and inherited state (one page)

Write a one-page description of the team you are authoring for. Cover:

- The five services (the two tier-1, one tier-2, two tier-3) — purpose, scale, dependencies.
- The six engineers — rough seniority distribution, geographic distribution, on-call-eligible subset.
- The inherited state — dashboards, alerting, rotation, runbooks, open incidents, open post-mortem action items.
- The three constraints that bound your programme design (headcount budget, geographic distribution limiting follow-the-sun, inherited vendor contracts, etc.).

You may use the context above verbatim and extend it, or you may substitute a real team you have operated. Fix the description for the remainder of the exercise.

### Part 2 — The five-ritual cadence document (two pages)

Produce a `programme/rituals.md` document describing the five rituals from chapter 01 §2 as you would run them on your team:

- Weekly operational review: slot, length, attendees, agenda, artifact location.
- Monthly SLO review: slot, length, attendees (required / standing-invitation / observers), agenda, artifact location. Reference forward to exercise 04.
- Weekly on-call handoff: slot, length, attendees, agenda, artifact location.
- Quarterly retrospective: slot, length, attendees, agenda, artifact location.
- Annual programme review: slot, length, attendees, agenda, artifact location.

For each ritual, name one measurable property by which you would judge "this ritual is still working" six months in — an attendance threshold, an artifact-timeliness threshold, a decision-production threshold.

### Part 3 — The three operational artifacts

Produce three sub-documents.

#### 3.1 On-call rotation design (`programme/on-call.md`, 2–4 pages)

Follow chapter 01 §3.1. Include:

- Rotation shape (primary / secondary / escalation). Shift length. Hand-off slot.
- Pool composition — who is in, who is being onboarded, who is excluded and why.
- Follow-the-sun vs. regional-anchored. Justify the choice against the geographic distribution.
- Interrupt-budget per rotator (hours per week), citing the Workbook chapter 8 operational-overload math.
- Shadow-oncall convention — ramp length calibrated to your failure-class catalogue.
- Peer-provider escalation — the account, the SLA, the minimum response time for each hosted provider.
- Compensation and time-off policy references.

#### 3.2 Deployment-change calendar schema (`programme/change-calendar.md`, 1–2 pages)

Follow chapter 01 §3.2. Specify:

- The format (Git-tracked YAML, shared calendar, etc.). Justify the choice.
- The schema for a change entry — required fields, optional fields.
- The green / yellow / red window conventions and the default windows across the year.
- The query interfaces — how the weekly operational review, the weekly on-call handoff, and the monthly SLO review each read the calendar.
- A worked example of the next-30-days view for your team with at least eight realistic entries (some green, some yellow, at least one red).

#### 3.3 Migration tracker (`programme/migrations.md`, 1–2 pages)

Follow chapter 01 §3.3. Include:

- The tracker format and fields (per chapter 01 §3.3).
- At least three realistic in-flight migrations for your team — e.g. the observability-stack RFC from exercise 02, an engine-version migration, a DR region addition.
- Per migration: name, owner, phase, exit criteria, rollback plan, open risks, peer dependencies.

### Part 4 — The RACI document (`programme/raci.md`, 1–2 pages)

Follow chapter 01 §4. Produce a table covering:

- Each of the five rituals (R, A, C, I named per ritual).
- Each of the three operational artifacts.
- The reliability-engineering RFC authoring authority.
- The budget-burn communication loop.
- The six peer-interface contracts from chapter 03 (one row per peer).

Every row has one named A (person or role, not "the team"). C and I lists may have multiple entries.

### Part 5 — The first-90-days plan (`programme/first-90-days.md`, 1–2 pages)

Follow chapter 01 §7. Produce a week-by-week plan for weeks 1 through 13 of your tenure, with explicit milestones at week 4 (weekly rhythm live), week 12 (monthly rhythm live), and week 13+ (first quarterly retrospective scheduled). For each week, name:

- The ritual being stood up or iterated on.
- The artifact being produced or updated.
- The people being engaged.
- One measurable outcome of the week.

## Requirements

1. **All five rituals and three artifacts are produced.** No "TBD" entries.
2. **Every ritual names one named owner (an A in the RACI).** "The team" is not an owner.
3. **The on-call rotation design cites the Google SRE Workbook chapter 8 at least twice** and cites your team's specific operational constraints.
4. **The deployment-change calendar includes a worked next-30-days view** with at least eight entries across all three window types.
5. **The migration tracker lists at least three realistic migrations** with exit criteria and rollback plans.
6. **The RACI covers the five rituals, three artifacts, RFC authoring, budget-burn communication, and six peer contracts** — minimum 15 rows.
7. **The first-90-days plan has explicit week-by-week milestones.** "Set up rituals in month 1" is not acceptable.
8. **All artifacts are in Git**, in the `programme/` directory, with a top-level `programme/README.md` indexing them.

## Starter guidance

- Resist the temptation to copy chapter 01 verbatim. Chapter 01 is the generic shape; the exercise is the application.
- The on-call rotation design is the hardest piece. Be honest about the constraint of a six-engineer team and a 24/7 service. The Workbook's operational-overload math is uncomfortable; sit with the discomfort and name what the team cannot sustainably cover.
- The RACI is the document where diffused-accountability anti-patterns hide. Read every row and ask: "if this ritual did not happen next month, whose name is on the hook?" If the answer is unclear, the row needs rewriting.
- The deployment-change calendar schema is most useful when it is queryable. Spending ten minutes on a format that supports `query --next-30-days --tier=1` pays off at every weekly review.
- The first-90-days plan should be ambitious but not unrealistic. If every week has three milestones, the first incident or outage will collapse the plan. Build in slack.

## Acceptance criteria

- A `programme/` directory in Git with `README.md`, `rituals.md`, `on-call.md`, `change-calendar.md`, `migrations.md`, `raci.md`, `first-90-days.md`.
- Each file meets the length targets above.
- Each file cites at least one reference (chapter of this module, chapter of another module, or external reference — the Workbook, SRE Book, or your organisation's existing docs).
- The RACI has no `team` entries in the A column.
- A reviewer with the team-characterisation (Part 1) context could read the programme and run the week-1 rituals on Monday morning.

## Stretch goals

- **The second-team variant.** Rewrite the programme for a 15-engineer team running five tier-1 services. Note the places where the programme shape changes with team scale.
- **The frontier-lab variant.** Rewrite the programme for a 20-engineer team running a frontier-model serving fleet with a weekly model-version-promotion cadence. Note the places where the deployment-change calendar and the release-freeze policy adapt.
- **The regulated-industry variant.** Rewrite the programme for a team deploying AI in a regulated environment (healthcare, financial services, public sector). Note the places where the RACI expands to include legal and compliance peers.
- **The measurement instrument.** Propose a one-page "programme health dashboard" that a successor team lead could read to know, in 10 minutes, whether the programme is still functioning.
