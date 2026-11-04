# Owning the Reliability Program at Team Scope: Cadence, Rituals, and Rotation

## Why this chapter exists

Mod-101 through mod-113 taught the mechanics. SLO definitions, observability pipelines, non-deterministic failure diagnosis, capacity planning, vendor-risk mapping, incident response, chaos engineering, safety-reliability crossover, progressive delivery, data-and-config pipelines, DR and multi-region. By the end of mod-113 the function knows how to operate each surface. What remains is the question of **programme** — the operating rhythm under which the whole function runs, year after year, visible to leadership, debuggable by a new hire, and durable through the inevitable roster churn.

A reliability function without a programme is a set of individual heroics. The dashboards exist, the runbooks exist, the error budgets exist, but there is no ritual that forces the function to *read* them. Decisions about feature-work rebalancing, release freezes, cross-peer escalations, and migration planning get made ad-hoc, under whichever incident is most recent, by whichever engineer happens to care most. Six months in, the function has drifted. The SLOs are now aspirational, the budgets are now theatre, and the on-call rotation is burning out the two people who still take it seriously.

The programme fixes this. It is a small number of durable rituals, run on an explicit cadence, with named owners and named artifacts, that keep the function's attention on the right surfaces at the right frequency. This chapter writes the shape — not any one ritual in isolation (the SLO review gets its own chapter in §04, the deployment-change calendar composes with mod-111 chapter 01, the on-call rotation composes with mod-108 chapter 03), but the **composition** of all of them into a weekly / monthly / quarterly / annual rhythm that fits on one page and that a new team lead can read on their first day.

Primary references for this chapter: the [Google SRE Book](https://sre.google/sre-book/table-of-contents/) chapters 11 (Being On-Call), 15 (Postmortem Culture), and 29 (Dealing with Interrupts); the [Google SRE Workbook](https://sre.google/workbook/table-of-contents/) chapter 8 (On-Call) and chapter 14 (Managing Load); mod-101 chapter 01 for the role's charter, mod-108 chapter 03 for the on-call-rotation mechanics this chapter composes with, mod-111 chapter 06 for the DORA metrics the deployment-change calendar reads.

## 1. What a programme is, and what it is not

A reliability programme is the set of recurring activities, with explicit cadence and ownership, through which the function produces, consumes, and acts on the reliability signal. It is **not** a list of tools, a set of dashboards, or a project plan. Tools, dashboards, and project plans are *inputs* to the programme; the programme is the discipline that keeps the inputs honest.

Three properties separate a programme from a collection of habits:

- **Cadence.** Each activity has a fixed interval — weekly, bi-weekly, monthly, quarterly, annual — that does not slide because the week is busy. The cadence is the commitment; the content varies.
- **Named owner.** Each activity has one accountable owner on the team, named in a Git-tracked RACI document (§4), who is responsible for the activity running whether or not they personally facilitate it.
- **Artifact.** Each activity produces an artifact — meeting notes, a signed decision, a dashboard snapshot, a ticket — that outlives the meeting and is linkable from the next activity's agenda.

An activity missing any of the three is a habit, not a programme element. Habits are fragile; programmes survive roster changes.

## 2. The five-ritual core

A team-scope reliability programme runs on five rituals. More rituals do not help — they compete for the team's attention and the lowest-leverage one gets dropped silently. Fewer rituals leave a gap that something ad-hoc grows to fill. The five below are the stable shape; everything else is a sub-activity inside one of them.

### 2.1 Weekly operational review (30–45 min)

**Purpose.** Keep the function synchronised on the current operational state: open incidents, in-flight deploys, SLO breaches that are trending but not yet in page-range, chaos experiments scheduled this week, drill outcomes from last week.

**Frequency.** Weekly, same slot. Missing the slot more than twice in a quarter is a programme failure that goes on the quarterly retrospective agenda.

**Attendees.** The AI SRE team. Optional standing observers: the `ai-infra-inference-engineer` peer tech lead (for engine-adjacent discussion), the `ai-eval-engineer` peer lead when release-gate activity is high.

**Agenda.**

1. Error-budget snapshot per tier-1 SLO (current burn, 7-day trend, 28-day trend).
2. Open incidents from the past 7 days (status, owner, next action).
3. In-flight deploys / canaries / rollouts on the deployment-change calendar for the next 7 days.
4. Chaos / drill activity planned this week — owner, blast radius, auto-halt wiring.
5. Interrupts above the Google SRE Workbook chapter 29 threshold (if `ai-eval-engineer`, `ai-infra-inference-engineer`, or product-engineering filed > N tickets this week, surface and triage).
6. One-item retrospective on the previous week's unexpected signal.

**Artifact.** A one-page meeting note in the team wiki, posted within 24 hours. The dashboard snapshot at the top; the action items at the bottom; linked from the next week's agenda.

**Why weekly.** Incidents, canaries, and chaos run at daily tempo; a monthly rhythm leaves the function reactive. A daily rhythm imposes meeting overhead that erodes the function's engineering time. Weekly is the natural compromise and matches the Google SRE Book's description of oncall-handoff cadence.

### 2.2 Monthly SLO review (60–90 min)

**Purpose.** Step back from the operational tempo. Read the top-N SLOs, their 28-day burn trend, the top five open incidents, the top five open runbook gaps, and the pipeline of migrations and major deploys for the next month. Decide release freezes, feature-work rebalancing, and escalations to the architect peer.

**Frequency.** Monthly, same week. Does not slide because an incident ran long; the incident is the input, not the excuse.

**Attendees.** The AI SRE team. Product-engineering representative (the product lead or their delegate). Standing invitation to `ai-infra-inference-engineer`, `ai-infra-performance`, `ai-eval-engineer`, and `ai-infra-mlops` peer leads; attendance expected at least quarterly.

**Chapter 04 of this module** writes the ritual in detail. For the programme-level view, the output is: a signed decision on freeze state (if any), a short list of architect escalations, and a priority list of runbook gaps for the next quarter's planning.

**Artifact.** The signed SLO-review minutes in Git — not just in the team wiki — with the budget-burn figures, the decisions, and the rationale. Auditors and successor team leads read these; a wiki-only record is not enough.

### 2.3 Weekly on-call handoff (15–30 min)

**Purpose.** Hand the pager cleanly from the outgoing primary to the incoming primary. Transfer context on open incidents, trending alerts, in-flight remediation, planned work that will affect the week's alerting.

**Frequency.** Weekly, at the rotation boundary. Google SRE Workbook chapter 8 §"Transitioning to On-Call" is the canon.

**Attendees.** Outgoing primary, incoming primary, outgoing secondary, incoming secondary, team lead or delegate as standing observer.

**Agenda.**

1. Walk the open-alert list; status and next action per item.
2. Walk the open-incident list; status and owner per item.
3. Review the deployment-change calendar (§3.2) for the week; flag windows of elevated risk.
4. Review chaos / drill schedule for the week; the on-call is the human-in-loop.
5. Confirm pager routing, phone-notification settings, backup escalation.

**Artifact.** The handoff ticket (or chat-thread) with the walked items, signed off by both outgoing and incoming primaries.

### 2.4 Quarterly retrospective (2–3 hours)

**Purpose.** The ninety-day step back. Review the quarter's incidents (counts, classes, MTTR trends), the drill-log, the error-budget consumption trend by service tier, the DORA four-keys trend from mod-111 chapter 06, the on-call burden per rotator, and the state of the roadmap. Generate the input to the next quarter's planning and to the annual review.

**Frequency.** Quarterly, aligned with the organisation's planning cadence.

**Attendees.** AI SRE team. Director or engineering-leadership representative. Peer-team leads as standing invitees.

**Agenda.**

1. Quarterly metrics review (incidents, MTTR, SLO attainment, DORA, on-call load).
2. "What we committed to last quarter and what landed" audit.
3. Retrospective: start / stop / continue, by individual and by programme ritual.
4. Peer-interface health check (§Chapter 03 artifacts — the interface contracts, the escalation counts, the mutual deliverables).
5. Next-quarter commitments, roadmap item deltas, hiring / staffing asks.

**Artifact.** The quarterly review document in Git (not a slide deck — a signed written artifact that leadership reads).

### 2.5 Annual programme review (half-day offsite)

**Purpose.** The yearly reset. Look at the annual DR drill outcome (mod-113 chapter 05 §1.3), the annual on-call rotation design review (§3.1 below), the annual build-vs-buy re-evaluation (chapter 02 of this module), the maturity-ladder score (§5), and the multi-year roadmap input to the `ai-infra-senior-architect` peer (chapter 05 §3 of this module).

**Frequency.** Once per calendar year, in the planning window before the next fiscal year begins.

**Attendees.** AI SRE team. Director / leadership. Standing invitation to the `ai-infra-architect` and `ai-infra-senior-architect` peers.

**Artifact.** The annual programme review document — the input to the following year's planning, the certification of the maturity-ladder score, the decisions that go to the senior architect (chapter 05 §3).

## 3. The programme's three operational artifacts

Three artifacts carry the programme between rituals. Each is Git-tracked, versioned, and reviewed on a cadence tied to a ritual above.

### 3.1 The on-call rotation design

**What it is.** A written document, under 10 pages, that specifies the rotation's shape: who is in the pool, primary / secondary / escalation tiers, follow-the-sun or regional-anchored, shift length, hand-off slot, compensation policy, interrupt-budget per rotator, the "stop the bleed" authority of the primary, the escalation tree to secondary / IC / director / provider support.

**Composition with mod-108 chapter 03.** Mod-108 chapter 03 wrote the incident-response mechanics the on-call executes. This document is the *organisational* layer — who takes the pager, in what pattern, under what compensation, with what limits.

**Reviewed.** Annually, in the annual programme review (§2.5). Partial updates at the quarterly retrospective (§2.4) when the retrospective surfaces a burden imbalance.

**Primary references.** Google SRE Book chapter 11 (Being On-Call); Google SRE Workbook chapter 8 (On-Call). The function cites both and does not reinvent the mechanics they already wrote; the function's work is the AI-specific adaptations:

- **Alert-pool size.** The function runs the Workbook chapter 8 §"Managing operational overload" math on the alert pool; for an AI service with non-deterministic quality alerts (mod-105) and GPU-telemetry alerts (mod-104), the per-rotator alert budget is tighter than for a classical web service because each alert tends to take longer to triage.
- **AI-domain context ramp.** A new rotator cannot be dropped straight into the primary slot. The function runs a shadow-oncall period — new rotator pairs with primary for one rotation, takes secondary for one or two, then takes primary. The shadow period's length is a function of the service's failure-class catalogue (mod-105).
- **Peer-provider escalation.** The escalation tree includes the hosted model provider's support channel (mod-107 chapter 03). The function knows, before an incident, which provider's support tier gets paged, through which account, on which SLA, with what minimum response time.

### 3.2 The deployment-change calendar

**What it is.** A shared calendar — a view, not a prose document — that shows every planned change that will touch a production reliability surface in the next 30 to 60 days. Model-version promotions, infrastructure deploys, tooling-stack changes, chaos / drill windows, configuration rollouts, provider-side announced changes.

**Composition with mod-111 chapter 01 and mod-112 chapter 05.** Mod-111 wrote the release pipeline that produces model-version candidates; mod-112 wrote the config / data pipeline that produces config and index changes. The deployment-change calendar is the **cross-pipeline view** — the place where the function sees that a prompt-registry rollout (mod-112 chapter 02) is scheduled in the same 48-hour window as a provider-advertised regional maintenance (mod-107 chapter 06) *and* the quarterly chaos game day. Each of the three is individually safe; the composition is not.

**Reviewed.** Weekly, at the operational review (§2.1). Weekly on-call handoff (§2.3) walks the coming week's window. Monthly SLO review (§2.2) reads the next-month pipeline to inform freeze decisions.

**Change-window conventions.**

- **Green window.** Low-risk changes allowed; routine deploys proceed.
- **Yellow window.** Only pre-approved changes; new changes require explicit approval at the weekly operational review.
- **Red window.** Change freeze. Only incident-response or explicitly-approved exceptions.

Red windows are declared at the monthly SLO review (§2.2) based on error-budget-burn state (chapter 04 §5). Yellow windows are often default during known high-risk periods — a pending regulatory filing deadline, an executive-presence event, the first 72 hours after a major provider-version pin change (mod-107 chapter 06).

**Format.** A Git-tracked YAML or an authoritative shared calendar — not a slide deck, not a wiki page that gets stale. The format must be queryable; the weekly operational review's agenda-automation reads it directly.

### 3.3 The migration tracker

**What it is.** A Git-tracked table of every active or planned cross-track migration — SRE-tooling changes (chapter 02 of this module), observability-stack evolutions (mod-104), release-pipeline migrations (mod-111), inference-engine moves (owned by the `ai-infra-inference-engineer` peer but visible to the SRE), chaos-tool changes (mod-109), and multi-region expansions (mod-113).

**Why a separate tracker.** Migrations are long-running. A migration from self-hosted Prometheus to a vendor observability stack may run for six to nine months (chapter 02 §5). The deployment-change calendar (§3.2) shows individual *cutover windows*; the migration tracker shows the *overall arc* — which phase, who the lead is, what the exit criteria are, what risks are open.

**Fields per migration.**

- `name`, `owner` (one named person), `phase` (one of: proposed / approved / in-flight / cut-over / wind-down).
- `exit_criteria` — the explicit conditions under which this migration is declared complete.
- `rollback_plan` — the path back if the migration fails at cutover.
- `risk_register_link` — the risks from chapter 02 §4 that this migration carries.
- `peer_dependencies` — the peer tracks whose work this migration depends on (performance, inference-engineer, mlops, eval, ml-platform, security).

**Reviewed.** Monthly, in the SLO review (§2.2). The migration tracker is on the monthly agenda unconditionally; the function does not let a multi-month migration go unreviewed for two consecutive months.

## 4. The RACI document

One Git-tracked document, usually one or two pages, that specifies — per programme ritual, per operational artifact, per cross-peer interface — who is **R**esponsible (does the work), **A**ccountable (one named owner; can be the same as R or different), **C**onsulted (input is sought before the work finishes), and **I**nformed (receives the output).

The RACI is not a bureaucratic flourish. It is the document that keeps three failure modes out of the programme:

- **Diffused accountability.** A ritual whose accountable owner is "the team" is a ritual that quietly stops happening. Every programme element names one A.
- **Peer surprise.** A migration whose C list did not include the `ai-infra-performance` peer is a migration where the peer first hears about the change from a Slack message post-cutover. The RACI forces the function to enumerate the C list before the migration starts.
- **Decision drift.** An incident-response action whose decision authority is unclear falls to whoever is loudest. The RACI pre-decides.

The RACI covers:

- Each of the five rituals (§2).
- Each of the three operational artifacts (§3).
- Each of the seven interface contracts in chapter 03 (one per peer).
- The RFC authoring authority (chapter 02).
- The budget-burn communication loop (chapter 05).

Reviewed annually; updates at any quarterly retrospective where a new ritual, artifact, or peer is added.

## 5. The maturity ladder

A five-level ladder — adapted from Google SRE Workbook chapter 2's "SLO engineering" ladder, Dickerson's hierarchy (mod-101 chapter 03), and the DR maturity ladder from mod-113 chapter 05 §8 — that places the function's programme on an axis. The function scores itself at the annual programme review (§2.5) and against this ladder leads the senior-architect roadmap conversation (chapter 05 §3).

**Level 1 — Reactive.** Incidents are handled as they occur. Dashboards exist but are not read on a cadence. SLOs may be aspirational. No weekly ritual; no monthly ritual; on-call is ad-hoc.

**Level 2 — Instrumented.** Dashboards are read weekly. SLOs are defined for tier-1 services. The weekly operational review runs. The on-call rotation is written down. Incidents produce post-mortems; post-mortem actions are inconsistently completed.

**Level 3 — Programmatic.** All five rituals from §2 run on cadence. All three artifacts from §3 are Git-tracked and current. The RACI exists and is referenced. Error-budget policy is in place and respected (chapter 04 §5). Peer interfaces (chapter 03) are written and signed. Drill cadence (mod-109, mod-113) is on schedule.

**Level 4 — Composed.** The programme composes with the organisation's wider rhythm — product planning, security, legal — through documented handoffs. Cross-service SLO composition (chapter 03 §7) is articulated. The deployment-change calendar is queried by neighbouring teams, not only the SRE. The migration tracker's cycles are completed on their stated exit-criteria without silent timeline slippage.

**Level 5 — Anticipatory.** The programme is used *predictively* — the monthly SLO review consistently catches a trend before it becomes a breach; the annual DR drill exercises the actual failure classes that occur in the subsequent year with > 50% coverage; the error-budget policy's freeze is invoked sparingly because the earlier-cadence rituals catch issues upstream.

The function does not pretend to level 5 on a new programme. Level 3 is the realistic target for the first year of a programme build; level 4 is the two-to-three-year target; level 5 is where senior-architect peer conversations open.

## 6. Composing with the peer tracks

The programme is not an island. Each ritual and artifact composes with a peer's programme; the function names the composition explicitly to avoid duplicating work the peer already does.

- **`ai-infra-inference-engineer` peer.** The engine team has its own release pipeline (mod-111 chapter 04 handoff). The deployment-change calendar is where the two programmes meet — a new engine release goes on the SRE calendar before it is promoted; the SRE calendar flags a chaos / drill window that the engine team's canary should avoid.
- **`ai-infra-performance` peer.** The performance team has its own regression-tracking rhythm (kernel-release notes, Nsight-Compute benchmark trend). The monthly SLO review is where the SRE surfaces performance regressions that leaked past the pre-release gate; the ticket flow (chapter 03 §2) is the mechanism.
- **`ai-infra-security` peer.** The security team runs its own change-management ritual (threat modelling, security review before significant changes). The deployment-change calendar is where the SRE sees security-driven changes landing; the joint post-mortem convention (chapter 03 §4) is where the shared-incident surface lives.
- **`ai-infra-ml-platform` peer.** The ML-platform team owns the artifact promotion pipeline (mod-111 chapter 08 §1.2). The monthly SLO review reads the pipeline's health; the migration tracker shows platform-pipeline migrations the SRE must plan around.
- **`ai-eval-engineer` peer.** The eval team owns the release gate (mod-111 chapter 05). The deployment-change calendar is where the SRE sees eval-gate activity; the monthly review reads eval-gate throughput and false-positive / false-negative rate.
- **`ai-infra-mlops` peer.** The mlops team owns CI/CD (mod-111 chapter 08 §1.2). DORA four-keys (mod-111 chapter 06) are read at the quarterly retrospective; the deployment-change calendar is populated partly by the mlops pipeline's events.

The function reads each peer's programme once (in the annual programme review (§2.5)) and names, in the RACI, the per-peer interface. It does not rebuild the peer's programme; it composes with it.

## 7. The first-90-days version

A new team lead inheriting a function with no programme does not build the full §2–§6 shape in month one. Three ordered milestones carry a programme from "nothing" to "level-3 ready":

**Weeks 1–4 — the weekly rhythm and the on-call document.**

- Stand up the weekly operational review (§2.1). First few meetings will be rough; keep them short, iterate the agenda.
- Stand up the weekly on-call handoff (§2.3). Codify the existing rotation into the on-call rotation document (§3.1) even if the document is imperfect.
- Produce the initial RACI (§4) covering just the two weekly rituals and the on-call document.

**Weeks 5–12 — the monthly rhythm and the change calendar.**

- Run the first monthly SLO review (§2.2). First one surfaces the SLO-definition gaps; chapter 04 of this module writes the facilitation shape.
- Stand up the deployment-change calendar (§3.2). Populate from existing pipelines (mod-111) and mlops tickets; convene the first weekly review of the calendar.
- Extend the RACI to cover the monthly review and the calendar.

**Weeks 13+ — the quarterly rhythm and the full artifact set.**

- Run the first quarterly retrospective (§2.4).
- Stand up the migration tracker (§3.3). Backfill the active migrations; set exit criteria for each.
- Extend the RACI to cover the migration tracker and the peer-interface routing (chapter 03).
- Score the programme against the maturity ladder (§5); aim for level 3 within the first year.

The annual programme review (§2.5), the annual on-call design review, and the annual build-vs-buy re-evaluation (chapter 02) follow naturally once the quarterly rhythm is in place.

## Summary

- A reliability programme is the composition of five rituals (weekly operational review, monthly SLO review, weekly on-call handoff, quarterly retrospective, annual programme review), three operational artifacts (on-call rotation design, deployment-change calendar, migration tracker), and one RACI document. Everything else fits inside this shape.
- Each ritual has an explicit cadence, a named owner, and a Git-tracked artifact. Habits without these three properties are not programme elements.
- The on-call rotation design composes Google SRE Book / Workbook canon with AI-specific adaptations: tighter per-rotator alert budget, shadow-oncall ramp calibrated to the failure-class catalogue, provider-support channels wired into the escalation tree.
- The deployment-change calendar is the cross-pipeline view. It composes the release pipeline (mod-111), the data / config pipeline (mod-112), chaos / drill windows (mod-109), and provider-announced changes (mod-107). Green / yellow / red windows mediate change-admission.
- The migration tracker is the long-arc view of cross-track SRE-stack evolution, reviewed monthly, with explicit exit criteria and rollback plans per migration.
- A five-level maturity ladder — reactive / instrumented / programmatic / composed / anticipatory — gives the function a score it reports to leadership and that feeds the senior-architect roadmap conversation (chapter 05 §3).
- A new team lead builds the programme in three ordered milestones: weekly rhythm in weeks 1–4, monthly rhythm and change calendar in weeks 5–12, quarterly rhythm and the full artifact set in weeks 13+.

Chapter 02 writes the first large-scale decision the programme carries: the build-vs-buy RFC for the SRE tooling stack, with its TCO model, risk register, and migration plan.
