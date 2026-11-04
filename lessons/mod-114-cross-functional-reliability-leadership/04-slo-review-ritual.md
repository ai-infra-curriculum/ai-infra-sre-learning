# The SLO Review Ritual: Monthly Facilitation at Team Scope

## Why this chapter exists

The monthly SLO review is the single ritual that most reliably separates a reliability programme at level 3 (programmatic) from one at level 2 (instrumented) or below. Mod-103 wrote the SLOs; mod-104 built the observability to compute them; mod-108 wrote the incident response that burns the budget; mod-111 wrote the release gate that respects it. Every one of those investments is theatre if the function does not have a monthly forum in which the budgets are *read, acted on, and recorded*.

The temptation on a new programme is to treat the SLO review as a status meeting. The dashboards get screen-shared, the budgets get walked, someone says "we're over budget on service X," heads nod, the meeting ends, no decision is made. Six months later the error-budget policy has silently become "we promise to care about budgets" rather than "we freeze releases when the budget runs out." The programme has drifted; nobody noticed because no artifact tracked the drift.

This chapter writes the facilitation shape. It describes the pre-meeting preparation, the agenda, the decisions the forum is explicitly authorised to make, the artifacts it produces, and the anti-patterns that most frequently degrade it. Chapter 01 of this module placed the ritual in the programme; this chapter writes what to do in the room.

Primary references: the [Google SRE Workbook chapter 3 (SLO engineering)](https://sre.google/workbook/implementing-slos/) and chapter 4 (SLO engineering case studies); the [Google SRE Book chapter 4 (Service Level Objectives)](https://sre.google/sre-book/service-level-objectives/) for the vocabulary; mod-103 chapters 01–05 for the per-SLO mechanics this ritual reads from; mod-108 chapter 04 for the post-mortem structure feeding the "top five open incidents" section.

## 1. Purpose and scope

The monthly SLO review has one purpose: to turn operational signal into decisions that change the function's behaviour over the next 30 days. Three decision classes the ritual is explicitly authorised to make:

- **Release-freeze / feature-work rebalancing.** When the error budget is spent for a tier-1 service, the forum declares a red window on the deployment-change calendar (chapter 01 §3.2) and reallocates engineering time from feature work to reliability work for the next 30 days.
- **Architect-peer escalation.** When a recurring incident class points to an architectural limitation the function cannot fix inside its own scope, the forum authorises an escalation ticket to `ai-infra-architect` (chapter 05 §2) and names the SRE-side owner who will walk the ticket.
- **Runbook-gap prioritisation.** When the top-five runbook gaps have been stable for two reviews running, the forum reallocates on-call engineering time to close the top two. The forum is the only place in the programme where this trade-off is made explicitly.

Decisions outside these three go to other rituals. An SLO definition change (mod-103 chapter 02) is reviewed here but signed in a separate change-control record. A retrospective on a specific incident runs in a separate post-mortem meeting (mod-108 chapter 04). A multi-year roadmap decision lives at the senior-architect peer (chapter 05 §3), not here.

Keeping the scope narrow is what keeps the ritual on cadence. A meeting that drifts into general operational discussion is a meeting that drops back to monthly-status and loses its decision authority.

## 2. Attendees and standing roles

- **Facilitator.** The SRE team lead or a rotating facilitator from the team. The facilitator owns the agenda, the pre-reading, and the artifact. Rotating the facilitator (quarterly) spreads the ritual's institutional knowledge.
- **Attendees (required).** All members of the AI SRE team. A quorum of at least two-thirds of the team is required for the forum to make binding decisions.
- **Attendees (standing invitation).** Product-engineering lead (or named delegate) — required at least quarterly, invited monthly. Peer-team leads (`ai-infra-inference-engineer`, `ai-infra-performance`, `ai-eval-engineer`, `ai-infra-mlops`, `ai-infra-ml-platform`, `ai-infra-security`) — standing invitation, attendance expected at least quarterly.
- **Observers.** The director / engineering leadership — standing invitation. The `ai-infra-architect` peer — standing invitation, attendance expected at least quarterly.
- **Note-taker.** Rotates. The note-taker produces the artifact (§5) and posts it to Git by end of the next business day.

The required attendees are the SRE team; a review that happens without the SRE team is not an SRE review. The product-engineering representative's required-quarterly attendance is what makes the release-freeze / feature-work rebalancing decision (§4.1) binding on the product side.

## 3. Pre-meeting preparation

A 60–90-minute meeting works only when the inputs are prepared in advance. The pre-reading is distributed 48 hours before the meeting; participants are expected to read before the slot.

### 3.1 Pre-reading pack

- **Current-month SLO attainment table.** Per SLO, per service, per tier: the 28-day attainment, the current error-budget-burn trend, the delta against the previous month.
- **Top-N dashboards.** Per tier-1 SLO: a snapshot of the dashboard at the time of pack generation, with annotations for any notable events.
- **Top five open incidents.** Status, owner, next action, days-open. "Open" is defined as mod-108 chapter 04 status `open` or `in-review`; `closed` incidents enter the top-5 only if their action items are still open.
- **Top five open runbook gaps.** Gaps identified in post-mortems, drill retrospectives, or during on-call. Each entry: the gap description, the service affected, the detection mode, the open-since date.
- **In-flight-migration status.** A one-line status per entry in the migration tracker (chapter 01 §3.3). Phase, lead, exit criteria, open risks.
- **Deployment-change calendar for the next 30 days.** Chapter 01 §3.2's view at the agenda's horizon. Red / yellow / green windows called out.
- **Peer-interface health summary.** One line per interface from chapter 03 §10's quarterly health check, updated with monthly delta.
- **Previous-review action items.** The open action items from the last three reviews. Each with status (done / in-flight / blocked / dropped).

The pack is the pre-reading, not the meeting. A facilitator who finds themselves walking the pack live in the meeting is wasting the team's time.

### 3.2 Facilitator homework

Before the meeting the facilitator pre-identifies:

- Which SLOs are near a burn-rate-threshold decision (§4.1).
- Which incident classes have appeared in more than one of the top-5 lists across the last three reviews (§4.2 escalation candidate).
- Which runbook gaps have been top-5 for two or more consecutive reviews (§4.3 prioritisation candidate).
- Which migration is at a phase-gate this month and needs review time.
- Which peer-interface signal is declining and needs 10 minutes at the table.

These are the facilitator's *questions for the room*. The meeting's quality depends on the facilitator having framed them in advance.

## 4. The agenda

A 60-minute meeting runs eight items; a 90-minute meeting allows deeper discussion on two or three. The structure is deliberately tight; the facilitator defends it.

### 4.1 SLO attainment and error-budget-burn trends (15–20 min)

Walk the top-N SLOs. For each:

- 28-day attainment against target.
- Current burn-rate — 1-hour, 6-hour, 24-hour, 7-day — using the Google SRE Workbook chapter 5 multi-window multi-burn-rate framework.
- 90-day trend, visually: improving / stable / worsening.
- Composition of burn — which failure classes (mod-105) contributed, which peer slice (chapter 03 §9) was the causal path.

The output of this section is a written decision per SLO:

- **Within budget, trending healthy.** No action; move on.
- **Within budget, trending toward breach.** Name the trend; schedule a focused investigation; assign an owner; put on the next review's agenda for verification.
- **Over budget.** Declare the red window on the deployment-change calendar for the next 30 days per the error-budget policy. Reallocate engineering time (§4.1 below). The policy is not negotiable in the room — the policy was written by the function and signed by product (mod-103 chapter 04) in advance; the forum applies it.

**The error-budget policy's role.** The policy is Git-tracked (mod-103 chapter 04), signed by the SRE lead and product lead, and names, per budget state, the automatic consequence. "When a tier-1 SLO's quarterly budget is exhausted, we freeze non-critical deploys and reallocate X% of feature-engineering capacity to reliability work until the budget recovers." The ritual *applies* the policy; it does not rewrite the policy on the fly.

### 4.2 Top five open incidents (15 min)

Per incident, 3–4 minutes. For each:

- Current status (open / in-review / closed-with-open-actions).
- Days open; owner.
- Root-cause class per mod-105's catalogue.
- Status of action items.
- Whether the incident fits a recurring class (§4.4 below).

A single incident open for more than 30 days without a written reason is an escalation to the director; the forum is where the escalation is agreed.

### 4.3 Top five open runbook gaps (10 min)

Per gap, 2 minutes. For each:

- The scenario the runbook is missing (or the existing runbook is wrong on).
- When the gap was identified.
- Who owns closing it.
- Rough effort to close.

If the top-five list is stable for two consecutive reviews, the forum reallocates on-call engineering time to close the top two. Stability is a signal that the backlog is priority-starved, not technique-limited.

### 4.4 Recurring incident classes → architect escalations (5–10 min)

The facilitator presents the incident classes that have appeared across more than one top-5 list in the last three reviews. For each recurring class:

- Does the class point to an architectural limitation the function cannot fix inside its own scope?
- If yes, authorise an escalation ticket to the `ai-infra-architect` peer (chapter 05 §2).
- Name the SRE-side owner who walks the ticket.

A pattern that is recurring but is still fixable inside the function's scope stays with the function; the facilitator pushes back on reflexive escalation.

### 4.5 In-flight migrations (5–10 min)

Walk the migration tracker (chapter 01 §3.3). Per migration at a phase-gate:

- Phase, lead, exit criteria.
- Open risks.
- Decision authorised here: advance / hold / rollback.

Migrations not at a phase-gate are mentioned only if they have aged beyond their expected phase duration.

### 4.6 Deployment-change calendar review (5 min)

Walk the next-30-day view. Named changes and their windows:

- Red windows declared under §4.1 — confirmed.
- Yellow windows — change-admission decision.
- High-concentration windows (several changes in a short period) — the forum asks "do we want to spread these out?"

### 4.7 Peer-interface signal (5 min)

The facilitator walks the one or two interfaces with declining signal. Each is a 2–3 minute discussion:

- What the signal is.
- The immediate next action.
- Whether to open a focused discussion with the peer lead outside the forum.

The forum does not relitigate peer contracts here; chapter 03's quarterly health check and annual review are the venues for that.

### 4.8 Previous-review action items and close-out (5 min)

- Walk the open action items from the last three reviews. Each: still-open? Done? Blocked?
- Close dropped items with a written reason.
- Add new action items from this review, each with owner and target date.

An action item without an owner is not an action item. The facilitator defends this.

## 5. The ritual's artifact

The note-taker produces a one-document record by end of the next business day. The document lives in the team's Git repository under `slo-reviews/YYYY-MM.md` (or equivalent). Its sections mirror the agenda:

1. SLO attainment table (snapshot).
2. Decisions per SLO (budget, trend, action).
3. Top-five-incident walk (status per incident, action per incident).
4. Top-five-runbook-gap walk (owner per gap).
5. Recurring-class escalation decisions (authorised, owner, target date).
6. Migration phase-gate decisions.
7. Deployment-change-calendar updates.
8. Peer-interface signals addressed.
9. New action items (owner, target date, review date).
10. Facilitator / note-taker names; next review date.

The document is signed by the facilitator and the note-taker before Git commit. It is **not** a slide deck. A slide deck is ephemeral; a Git-tracked written record outlives the meeting and is linkable from every subsequent ritual's agenda.

## 6. The error-budget policy and the release-freeze decision

The forum's authority to freeze releases is derived from the error-budget policy (mod-103 chapter 04) signed in advance by the SRE lead and the product lead. Without that signed policy, the forum does not have the authority; with it, the forum applies it.

### 6.1 The policy's shape

A one-page document stating, per service tier, the budget target, the measurement window (typically 28 or 30 days), and the automatic consequence per budget state:

- **Budget healthy (> 50% remaining).** Green window default; no reliability-side intervention required.
- **Budget threatened (20–50% remaining).** Yellow window; the forum authorises the function to raise the question with product-engineering about feature-work rebalancing.
- **Budget critical (< 20% remaining).** Red window; non-critical deploys frozen for 30 days; a named percentage of feature-engineering capacity is reallocated to reliability work until the budget recovers.
- **Budget exhausted (0% remaining).** All deploys except incident-response frozen; a named escalation to the director runs in parallel; the function prepares a focused action plan within one week.

### 6.2 The forum's role

- The forum reads the policy at the start of §4.1.
- The forum applies the policy to the current budget state, per SLO, per service.
- The forum authorises the specific red / yellow / green window decisions on the deployment-change calendar.
- The forum does not rewrite the policy on the fly; a proposed policy change goes to a separate signoff event (mod-103 chapter 04 §x) with the product lead.

### 6.3 Feature-work rebalancing

When the forum authorises a red window, the function's engineers are reallocated per the policy. The specific rebalancing — which features defer, which reliability work accelerates — is negotiated between the SRE lead and the product-engineering lead within the following week, with reference to the forum's minute as authority. The chapter 05 upward-communication path (chapter 05 §1) is how the product lead hears this; the forum is how the authority is anchored.

## 7. Facilitator discipline: anti-patterns the ritual most often drifts into

### 7.1 The "status meeting" drift

The forum reads dashboards and walks incident lists without producing decisions. Three months in, the forum feels less useful; attendance slips. **Fix.** The facilitator pre-identifies §3.2's questions and insists on per-SLO decisions in §4.1.

### 7.2 The "re-litigate the SLO definition" drift

An uncomfortable budget state produces a push to redefine the SLO. The forum spends 30 minutes debating the SLO's threshold instead of acting on the current state. **Fix.** SLO-definition changes are out of scope for the ritual. The facilitator parks the discussion into a separate signoff event (mod-103 chapter 02).

### 7.3 The "every incident gets re-discussed" drift

The top-5 walk turns into mini-post-mortems. The forum runs long; later agenda items get cut. **Fix.** Post-mortems happen in mod-108 chapter 04's post-mortem meeting, not here. The top-5 walk is 3 minutes per incident, by the clock.

### 7.4 The "we'll come back to it" drift

A hard decision is deferred to the next review. At the next review, deferred. Six months later, the decision still unmade. **Fix.** The facilitator insists that every agenda item produces either a decision or a dated owner for a follow-up. "Defer" is not a valid outcome without a date.

### 7.5 The "product wasn't here" drift

The forum made a release-freeze decision; the product lead was absent. The decision stands on the signed policy; product finds out after and pushes back. **Fix.** The policy was signed; the forum has standing authority. But the facilitator ensures the product lead is on the distribution list for the Git-committed minute within 24 hours.

### 7.6 The "everybody is present but nobody is accountable" drift

The forum has excellent attendance but action items are owned by "the team." Three months in, nothing on the action list has moved. **Fix.** Every action item names one person. The facilitator does not accept "team" as an owner.

## 8. Composing with the chaos and drill programmes

The SLO review consumes signals the chaos (mod-109) and drill (mod-113 chapter 05) programmes produce. Specifically:

- A chaos experiment that caused a near-breach of an SLO surfaces in §4.1 as a trend; the forum decides whether the experiment's finding needs action.
- A drill that caught an auto-halt-wiring gap surfaces in §4.3 as a runbook gap.
- A drill scenario failure mode that surfaced a recurring architectural limitation surfaces in §4.4 as an escalation candidate.

The forum does not run drills; it reads them. The two rituals are complementary and the function's leadership value is in the composition.

## 9. Composing with the DORA four-keys trend

Mod-111 chapter 06 writes DORA four-keys instrumentation for model releases. The quarterly retrospective (chapter 01 §2.4) is where the DORA trend is reviewed in depth; the monthly SLO review consumes only the deltas that affect budget:

- Change-failure-rate for the past month — if elevated, it enters §4.1 as a budget-trend driver.
- MTTR for the past month — if elevated, it enters §4.2 as a cross-incident pattern.
- Deployment frequency delta — if a change in frequency is driving burn, enter §4.1 as a cause.

The forum does not run a DORA review; it reads the DORA signal where it intersects budget.

## 10. The first-year of the ritual

For a programme at level 1 or 2 (chapter 01 §5) inheriting no previous SLO review ritual:

- **Reviews 1–2.** Attendance is light; most pre-reading is being produced for the first time; the dashboards are imperfect. The facilitator's job is to protect the shape even when the content is weak. Decisions are minimal but formal.
- **Reviews 3–6.** The pre-reading stabilises; the top-5 lists get real content. The first red-window decision usually falls here. The team-product trust in the ritual's authority is established or lost in this range.
- **Reviews 7–12.** The ritual produces architect-escalation tickets, runbook-gap prioritisation, and quarterly-review inputs. The forum's decisions outlast individual facilitators; the artifact becomes the function's institutional memory.

At 12 months the function should be able to point to a Git directory of 12 signed monthly SLO review documents and say "this is how the team's reliability posture evolved." At that point the ritual is load-bearing and the function is at level 3 (chapter 01 §5) on the programme's reliability dimension.

## Summary

- The monthly SLO review is the ritual that turns SLO measurement into decisions. It is authorised to make three decision classes — release-freeze / feature-work rebalancing, architect-peer escalation, and runbook-gap prioritisation — and keeps its scope narrow to protect its cadence.
- Required attendees are the SRE team; product-engineering is required quarterly, invited monthly; peer leads and leadership have standing invitations with quarterly attendance expected.
- Pre-meeting preparation — a pre-reading pack distributed 48 hours ahead, with facilitator homework to pre-identify the forum's decisive questions — is what makes the 60-minute agenda work.
- The eight-item agenda runs on a tight clock: SLO attainment and budget-burn, top-five incidents, top-five runbook gaps, recurring-class escalations, migrations at phase-gates, deployment-change-calendar review, peer-interface signal, previous-review close-out.
- The artifact is a Git-tracked monthly document, signed by facilitator and note-taker, that mirrors the agenda. It outlives the meeting and is linkable from every subsequent ritual.
- Release-freeze authority is derived from a signed error-budget policy (mod-103 chapter 04); the forum applies the policy, it does not rewrite the policy.
- Facilitator discipline is the ritual's moat against six recurring anti-patterns: the status drift, the SLO-redefinition drift, the mini-post-mortem drift, the deferral drift, the product-absent drift, and the diffused-accountability drift.
- The forum composes with chaos / drill programmes (reads their signals), the DORA four-keys trend (reads budget-affecting deltas), and the peer-interface health check (triages declining interfaces).
- A twelve-month track record of signed monthly minutes is the function's evidence that its reliability posture is anchored in practice, not aspiration — and the input the chapter 05 upward-communication story draws from.

Chapter 05 writes the upward-communication path by which the forum's signals become business-language stories the product lead, the architect peer, and the board can read and act on.
