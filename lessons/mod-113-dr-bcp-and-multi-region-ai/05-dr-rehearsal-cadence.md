# DR Rehearsal as a Regular Practice

## Why this chapter exists

A DR plan the function has never run is a document. A DR plan the function has run is a capability. NIST SP 800-34 Rev 1 §3.5 is explicit on this — "a system must be tested, trained upon, and exercised regularly" — and names a progression of exercise types: **tabletop**, **functional**, **full-scale**. The classical SRE canon reaches the same conclusion from a different angle: if a system is not routinely failed, the mechanisms that would recover it have decayed. The chaos programme from `mod-109` operates on the same premise at a smaller scope; this chapter writes the DR extension — the cadence, the exercise types, the integration with chaos, the auto-halt discipline, and the learnings feedback loop — so that DR is a weekly / quarterly / annual practice, not a thing the team will "get to next quarter."

The chapter is substrate-agnostic. It tells the function what to run, when to run it, who to involve, how to score it, and how to feed the output back into the ISCP, the runbook library, and the drill programme. The chapter 02 playbook, the chapter 03 state plan, and the chapter 04 residency constraints are the content that is being exercised; this chapter is the exercise.

Primary references: [NIST SP 800-84](https://csrc.nist.gov/pubs/sp/800/84/final) (*Guide to Test, Training, and Exercise Programs for IT Plans and Capabilities*), [NIST SP 800-34 Rev 1 §3.5](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final), [ISO 22301 §8.5](https://www.iso.org/standard/75106.html) (exercise programme), the [Principles of Chaos Engineering](https://principlesofchaos.org/) (`mod-109` chapter 01), [AWS DR game-day guidance](https://aws.amazon.com/builders-library/testing-site-recovery-and-failover/) <!-- needs-research: validate the exact AWS reference on game-day testing and update the URL if the canonical reference has moved -->, and the `mod-109` chapter 05 continuous-chaos cadence this chapter composes with.

## 1. The three exercise tiers

NIST SP 800-84 and ISO 22301 agree on three progressive exercise types. Each costs more than the one before and buys a different confidence class.

### 1.1 Tabletop exercises

A **tabletop exercise** is a discussion-based walkthrough. Participants talk through a scenario against the DR plan, surfacing gaps in authority, information, detection, and action. No systems are touched.

- **Duration.** 60–120 minutes.
- **Participants.** On-call lead, incident commander, service owner, representative from legal / compliance (for residency-relevant scenarios), executive sponsor on the quarterly cadence.
- **Facilitator.** A rotating function member (not the service owner); the facilitator reads the scenario, pushes on "what would you do here," and takes notes.
- **Output.** A list of gaps: a missing runbook, an unclear decision right, a stale contact in the call tree. Each gap becomes a ticket.
- **Cadence.** Quarterly per tier-1 service; semi-annually per tier-2.

Tabletop exercises are cheap. They catch the surprisingly large class of problems that are purely about information flow, authority, and documentation. The function runs them first and often.

### 1.2 Functional exercises (controlled region-switch drills)

A **functional exercise** performs the drill on real systems, with controlled scope. A region-switch drill shifts traffic from a primary to a secondary region on a scheduled window, measures the mechanism, and shifts back.

- **Duration.** Typically a 30–60 minute switch window plus 60 minutes of observation.
- **Participants.** On-call rotation, service owner, SRE lead, chaos programme facilitator.
- **Mechanism.** The chapter 02 playbook is invoked — the actual DNS / Anycast / LB change, the actual registry channel rebind, the actual provider-region breaker trip.
- **Blast-radius controls.** Scheduled low-traffic window. One service at a time. Immediate rollback mechanism confirmed before the window opens. Auto-halt wired to the three-family SLI alarm (§4).
- **Scope.** Within-residency where possible; cross-residency bursts drilled separately per chapter 04 §3.4 with legal present.
- **Output.** Measured RPO, RTO, and quality delta from the drill. Any step that deviated from the plan is a correction.
- **Cadence.** Semi-annually per tier-1 service; annually per tier-2.

### 1.3 Full-scale exercises

A **full-scale exercise** failover the whole service, end-to-end, as if the primary were lost — and leaves the secondary as the primary for a defined period before failing back. This is the DR event the team wishes it had practiced before the real one.

- **Duration.** A business day of elevated readiness. The failover itself is a 10–30 minute window; the secondary runs as primary for hours before fail-back.
- **Participants.** On-call, service owner, SRE function lead, incident command, legal / compliance (if cross-residency or on a residency-sensitive stamp), executive sponsor, communications team.
- **Mechanism.** Everything in the ISCP. Not a subset.
- **Blast-radius controls.** Announced window (internal and, if applicable, status-page-posted). Rollback path confirmed and timed. Auto-halt alarms live and the game-day commander has the authority to halt.
- **Scope.** The entire tier-1 service. Dependencies not included in the drill are documented as "not drilled this cycle."
- **Output.** A post-mortem (`mod-108` chapter 03), a revised ISCP, a list of gaps feeding the next year's drill plan.
- **Cadence.** Annually per tier-1 service. A function with multiple tier-1 services staggers them across the year.

The three tiers form a funnel. Tabletops catch documentation and authority gaps; functional drills catch mechanism gaps; full-scale exercises catch the composition gaps that only appear end-to-end.

## 2. The cadence table

The function's cadence per service tier:

| Exercise | Tier-1 | Tier-2 | Tier-3 |
|----------|--------|--------|--------|
| Tabletop | Quarterly | Semi-annually | Annually (optional) |
| Functional region-switch drill | Semi-annually | Annually | None required (run when design changes) |
| Full-scale failover | Annually | None required (unless BIA demands) | None |
| Cross-residency burst drill (where authorised) | Annually (with legal present) | Annually or per policy | n/a |
| State-reconciliation drill (per chapter 03 §2.4) | Annually | Annually | n/a |

The cadence is a floor, not a ceiling. A service undergoing significant change (new region, new state store, new regulator relationship) adds an off-cycle exercise against the change.

## 3. Exercise scenario catalogue

A DR programme that runs the same drill every quarter trains the team to the drill, not the real failure. The function rotates scenarios from a documented catalogue.

### 3.1 Scenarios from the regional-outage shapes

Each of the chapter 02 §1 impairment classes is drilled:

- **Full regional loss.** The primary region is marked unreachable. The playbook invokes endpoint failover, the state surface moves, the serving tier serves from the secondary.
- **Partial regional impairment.** A specific AZ or service in the region is impaired. The three-family alarm fires on the SLI, not on the aggregate status. The on-call decides on hold vs. shift.
- **Degraded-200.** The provider returns 2xx with quality collapse. The quality SLI breaches. The provider-regional breaker trips; traffic shifts to the alternate provider region.
- **Control-plane-only outage.** Control plane is impaired; data plane runs. The on-call practices the hold-and-observe path.

### 3.2 Scenarios from the state taxonomy

Each state class from chapter 03 §1 is exercised:

- **Conversation store partition.** A cross-region replication path is interrupted; writes diverge; the reconciliation policy is exercised on recovery.
- **Tool-call interruption.** An in-flight agent run is checkpointed, the serving region fails, the run resumes in the secondary against the three-way recovery decision.
- **Trace-ingest gap.** The regional ingest pipeline is impaired; the collector's local buffer absorbs. On recovery, the buffer drains. The exercise verifies no trace loss within the RTO target.
- **Registry unreachable.** The prompt / config registry is impaired. The serving tier's static-mode cache carries the serving path. New promotions are held per chapter 03 §6.3.

### 3.3 Scenarios from the provider-fallback contract

The chapter 02 §5 provider-regional posture is exercised:

- **Single provider regional outage.** The primary provider's primary region is red; the router shifts to the provider's other region.
- **Multi-provider regional overlap.** Both the primary provider's region and the fallback provider's region in the same geography are impaired; the router escalates to a further-away provider region, within the residency constraint.
- **Provider platform outage.** The entire provider is unavailable; the fallback provider is used.

The `mod-109` chapter 04 provider-fallback-contract chaos suite runs these at smaller scope weekly; the DR programme runs them at full regional scope in the functional and full-scale exercises.

### 3.4 Residency-sensitive scenarios (chapter 04)

- **In-residency failover (two-region stamp).** The default exercise; tier-1 semi-annual.
- **Degraded-mode in-residency.** One-region-stamp drill; the degraded path is exercised.
- **Pre-authorised cross-residency burst.** Annual, with legal / DPO present. The burst is invoked, the user-notice template is drafted (not sent — this is a drill), the teardown is timed.

### 3.5 The no-fly list

Scenarios the function does *not* drill end-to-end without explicit executive approval:

- **A real cross-residency burst without a pre-authorised SCC.** The drill would itself be a compliance event. The tabletop walks through the posture; the mechanism is not actually exercised.
- **Full-provider-platform-outage against the only provider.** If the function does not have a tested alternate provider, the drill is a tabletop until the alternate is wired.
- **Any drill that cannot be rolled back within the RTO window.** NIST SP 800-84 §3.1's "exercise control" principle; a drill must be stoppable.

## 4. Auto-halt and blast-radius controls

A drill that produces a real user-visible outage is a mis-drilled drill. The function wires the exercise to the three-family alarm (`mod-109` chapter 05 §2):

- **Serving-family alarm.** Burn-rate on the primary SLO exceeds the drill-admissible threshold; auto-halt fires.
- **Quality-family alarm.** Judge-score, refusal-rate, or safety-classifier flag change exceeds threshold; auto-halt fires.
- **Cost-family alarm.** Cost per 1k successful responses spikes beyond threshold; auto-halt fires.

The alarms are independent; any one triggers an immediate halt. The halt reverses the drill's action — traffic snaps back to the primary, the secondary ramps down — on a documented time budget (within one minute). The halt is drilled before the full drill: the function invokes an artificial breach to confirm the halt wiring on each full-scale exercise's day-of readiness check.

### 4.1 The blast-radius policy

Per the chaos programme's blast-radius policy (`mod-109` chapter 05 §3), the DR exercise's blast-radius is sized:

- **Traffic percentage.** The functional-drill cutover is 100% within the drilled service's traffic, 0% of other services. A full-scale exercise is 100% of the drilled service; if the service has dependencies that are themselves drilled, each is sized independently.
- **Duration.** The drill has a maximum-permitted duration before an automatic abort regardless of signals. A functional drill that hangs is not a drill in progress, it is an incident.
- **Scope.** One service at a time per drill. A drill that touches two services simultaneously is a composition experiment and requires explicit approval.

### 4.2 Scheduled off-peak windows

Drills run during the service's off-peak traffic window. The function's analytics peer provides the window curve; the DR programme's scheduling runs during the lowest-traffic quarter-hour the schedule supports. A drill against peak traffic is reserved for the chaos programme's maturity level 4 (`mod-109` chapter 05 §5), not a default.

### 4.3 Pre-drill readiness check

The ten-minute pre-drill readiness is a named step. On every functional and full-scale exercise:

- The on-call rotation is confirmed and paged into the drill channel.
- The dashboards are open and the three-family SLI is confirmed green.
- The warm-standby readiness (`chapter 02 §4.4`) is confirmed green.
- The auto-halt is tested with a synthetic breach and confirmed firing.
- The rollback path is confirmed.
- The incident commander for the drill is named.

A readiness check that fails cancels or defers the drill; it does not proceed with a known warning.

## 5. Integration with the chaos programme

The DR programme is not a parallel initiative to the chaos programme from `mod-109`; it is its larger scope. Chaos runs per-change, daily, weekly, quarterly (chapter 05 §1 of `mod-109`). DR runs quarterly, semi-annually, annually. The function composes them:

- **Chaos's provider-fallback contract suite (`mod-109` chapter 04) is the weekly cadence that keeps the provider-regional pathways warm.** A full-scale DR exercise's provider-regional swap should surprise nobody; the suite has been exercising it weekly.
- **Chaos's game-day shape (`mod-109` chapter 05 §4) is the structural pattern for the DR functional drill.** The same roles, the same facilitation, the same post-event discipline.
- **Chaos's auto-halt wiring (`mod-109` chapter 05 §2) is the DR drill's auto-halt wiring.** Two parallel alarm paths are not safer; the function reuses the one path and verifies it in both programmes.
- **Chaos's fixture back-feed (`mod-109` chapter 05 §6) applies.** Every DR gap becomes a chaos experiment that keeps the gap closed. A missed detection in a drill becomes an alerting-rule chaos fixture the following cycle.

The DR programme's quarterly review (§7 below) is a joint review with the chaos programme owner. The question "which DR gaps are already covered by chaos experiments" is answered on each cycle; the ones that are not become the next cycle's chaos fixtures or the next cycle's DR drill scenarios.

## 6. Game-day shape

A DR functional drill or full-scale exercise runs as a **game day** — a structured, facilitated event with explicit roles, timelines, and deliverables. The pattern is the `mod-109` chapter 05 §4 game-day shape extended for the broader stakeholder set.

### 6.1 Roles

- **Incident commander (IC).** Runs the drill as if it were an incident. Assigns actions, sequences decisions, calls holds.
- **Scribe.** Records the timeline, the decisions, the deviations from the plan, the signals observed. The scribe's log is the primary artifact after the drill.
- **Service owner.** Invokes the technical actions under the IC's direction.
- **Facilitator.** Rotates between function members; manages the scenario, injects new conditions ("the secondary's warm-standby alarm is also firing — how do you proceed?"), times the actions.
- **Observer roles.** Legal / compliance (for residency scenarios), executive sponsor (for full-scale exercises), subject-matter experts for specific dependencies (model-provider contact, retrieval-platform owner).

### 6.2 Timeline

A typical functional drill runs on the following timeline:

- **T-10 min:** Pre-drill readiness check (§4.3).
- **T-0:** The scenario is declared. The IC opens the drill channel and reads the scenario.
- **T+0 to T+10:** Detection phase. The three-family alarm should fire; the on-call acknowledges per `mod-108` chapter 02's incident-command drill.
- **T+10 to T+30:** Decision phase. The IC decides on the playbook action (shift, hold, degraded).
- **T+30 to T+50:** Execution phase. The service owner invokes the playbook. The signals are observed. Auto-halt is armed.
- **T+50 to T+90:** Observation phase. The secondary runs; the SLI is monitored.
- **T+90 to T+120:** Fail-back phase. Traffic returns to primary. The drill closes.
- **T+120 and later:** Hot-wash — a 30-minute discussion while the memory is fresh — followed by a formal post-mortem within 48 hours.

### 6.3 The hot-wash

The hot-wash is a short, structured conversation immediately after the drill. Questions:

- What did we detect, and did the detection fire as expected?
- What decision did the IC make, and what inputs were missing?
- What actions ran cleanly, and which required improvisation?
- What signals surprised us?
- What is the first change we should make to the ISCP as a result?

The hot-wash output seeds the post-mortem; the post-mortem produces tickets.

### 6.4 The scoring rubric

The function scores each drill against a short rubric:

- **Detection.** Did the alarm fire within the ISCP target time?
- **Decision.** Did the IC decide within the ISCP target time?
- **RPO.** Was the measured data loss within the target?
- **RTO.** Was the measured service restoration within the target?
- **Quality.** Did the quality signals remain within the drill-admissible band?
- **Communication.** Were the stakeholder updates sent on the ISCP cadence?
- **Rollback.** Was the fail-back clean?

The scores feed a rolling trend (quarterly dashboard) that leadership reviews on the §7 cadence.

## 7. The learnings-feedback loop

A drill that produces findings and no follow-through is a drill that happened. The function closes the loop:

- **Every drill produces tickets** with named owners and deadlines. Tickets are tracked in the function's standard tracker.
- **Every ticket closure updates the ISCP.** A fix that is not reflected in the plan is a fix that will not survive personnel turnover.
- **Every unrecoverable gap** — a drill that failed and the fix is not within the drill-cycle's budget — produces an explicit risk entry in the function's risk register; the next business-continuity review (chapter 06 §6) resolves it against business trade-offs.
- **Every new runbook or playbook change** goes through the `mod-108` chapter 01 top-20 runbook library's review discipline.
- **Every residency-touching finding** is reviewed with the DPO and the legal peer before the ISCP update lands.

The quarterly review — one or two hours, co-attended by the function lead, the chaos programme owner, and the service owners — confirms:

- The cadence was met.
- The scores improved or the regressions are understood.
- The tickets from previous cycles are closed or explicitly carried.
- The next cycle's drill schedule is set.

## 8. Maturity staircase

A DR programme matures over quarters. The function tracks its own maturity against a documented ladder:

- **Level 0 — plan exists.** An ISCP has been written. Nothing has been drilled.
- **Level 1 — tabletop cadence.** Quarterly tabletops are running. Documentation gaps are closing. No real failovers have been attempted.
- **Level 2 — functional drills.** Semi-annual region-switch drills are completing cleanly. RPO / RTO are measured. Auto-halt is wired and tested.
- **Level 3 — full-scale annual.** The annual full-scale exercise runs end-to-end. The secondary serves real traffic for hours. The post-mortem is routine.
- **Level 4 — continuous.** Game-day frequency approaches the chaos programme's weekly cadence; DR drills are small and often; the full-scale annual is a confirmation, not a surprise.

A programme at level 0 or 1 on a tier-1 service is an uninsured service. The function's reporting to leadership is the maturity level per service; leadership's job is to resource the programme to the required level.

## 9. The rehearsal-programme checklist

Before claiming the DR programme is in steady state, the function confirms:

1. **The cadence table** (§2) is agreed with service owners and leadership.
2. **The scenario catalogue** (§3) is in Git, with named rotations.
3. **The auto-halt and blast-radius policy** (§4) is wired for every drill and tested at pre-drill.
4. **The chaos programme is composed** (§5) — overlapping drills are coordinated, not duplicated.
5. **Each drill has an IC, scribe, service owner, and facilitator** (§6.1) named in advance.
6. **The hot-wash and post-mortem cycle** (§6.3, §7) are routine and produce tickets.
7. **The scoring rubric** (§6.4) runs on every drill; a quarterly trend dashboard exists.
8. **Every ISCP is updated** against the tickets from its drills within one quarter.
9. **The maturity level per service** (§8) is reported to leadership quarterly.
10. **The legal peer is a standing observer** on residency-sensitive drills.

A programme that cannot pass the checklist is a programme that is in motion, not in operation.

## Summary

- Three exercise tiers compose the DR programme: tabletop (quarterly, cheap, catches documentation / authority gaps), functional region-switch drill (semi-annually, catches mechanism gaps), full-scale failover (annually, catches composition gaps).
- The cadence per service tier is tabulated (§2); tier-1 services carry the heaviest cadence, tier-3 services are exercised only when design changes.
- A scenario catalogue (§3) rotates through regional-outage shapes, state-class failures, provider-fallback shapes, and residency-sensitive drills. A no-fly list names what is drilled only as a tabletop.
- Auto-halt is wired to the three-family SLI alarm from `mod-109`; every drill is stoppable within one minute of a breach.
- The DR programme composes with the chaos programme — same game-day shape, same auto-halt wiring, same fixture back-feed. Chaos keeps the mechanisms warm; DR confirms they hold at scope.
- A game-day is a structured, facilitated event with named roles, a documented timeline, a hot-wash, and a scoring rubric. The learnings feed back into the ISCP, the runbook library, and the next cycle's drill plan.
- A five-level maturity staircase (plan → tabletop → functional → full-scale → continuous) measures the programme's progress; the function reports the per-service level to leadership.

Chapter 06 writes the business-continuity artifact set — the ISCP template, the RPO / RTO table, the call tree, the communication templates, the regulator-facing summary scaffold — that holds the programme together and makes it visible to the external audit.
