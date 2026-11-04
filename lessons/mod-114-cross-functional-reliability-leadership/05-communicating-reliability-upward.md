# Communicating Reliability Upward: Product, Architect, Leadership, Board

## Why this chapter exists

A reliability function that produces excellent artifacts and makes the right operational decisions but cannot translate any of it into language the product lead, the architect peer, or the board can act on is a function that will be routed around when the hard decisions get made. The error budget is spent and the function says so, in engineering language, in the quarterly review; six weeks later the product organisation ships the big launch anyway because nobody at the launch-approval meeting understood that the budget was the issue. The next year's hiring round is approved for every team except the SRE team because the business case for reliability engineering was never translated into business-impact language. The function is right, correctly, repeatedly — and ignored.

Upward communication is the function's responsibility. The audience is not the function; the function cannot expect the audience to translate on its behalf. This chapter writes the three audiences the function translates to — the **product lead**, the **architect peer**, and **leadership / board** — and the specific translation move that each requires. It also writes the boundary against the `head-of-ai-governance` and CAIO scopes: the function authors reliability narratives, not governance narratives, and the difference matters.

Primary references: the [Google SRE Workbook chapter 6 (Reliability of the business, not the product)](https://sre.google/workbook/eliminating-toil/) and chapter 15 (Managing SRE adoption) for the audience-translation vocabulary; mod-101 chapter 01 and chapter 02 for the charter boundaries this chapter respects; mod-110 chapter 06 for the governance-versus-reliability boundary.

## 1. Three audiences, three translation moves

Each audience hears reliability signal differently. The function tailors the frame.

- **The product lead** hears reliability as a *constraint on feature shipping*. The translation move is from error-budget-burn to business-impact: "This month's spend of the budget corresponds to N hours of degraded user experience, affecting M user cohorts. Here is what rebalancing for the next 30 days will enable us to prevent."
- **The architect peer** (level 40) hears reliability as *patterns in incident data pointing at architectural limitations*. The translation move is from recurring-incident-class to architecture-ask: "We have seen four incidents in the last quarter that trace back to [architectural property X]. Here is the specific architecture ask that would remove the class."
- **Leadership and the board** hear reliability as a *quarterly narrative that composes with the organisation's risk posture*. The translation move is from the monthly minutes and the drill log into a compressed quarterly narrative that fits in the leadership review packet.

Three different stories, each sourced from the same operational signal. A function that tells all three in the same engineering language is a function that is heard by none of the three.

## 2. Translating for the product lead

### 2.1 What the product lead needs

Product leads make decisions about feature prioritisation, launch timing, customer communication, and engineering resourcing. Reliability signals affect each of these; the function's job is to make the signal actionable at the product lead's abstraction level.

Three questions the function answers for the product lead on a monthly cadence:

- **Are we safely spending the error budget?** Not "what is the SLO at 99.5%" but "are we on-track to deliver the product experience we promised, and if not, what is the cost of recovery?"
- **What is the business-impact decomposition of this month's budget-burn?** Which user cohorts, which routes, which features bore the burn. A budget spent on a long-tail of transient blips affects very differently from a budget spent on a single multi-hour degradation of the premium tier.
- **What does the next 30 days need from product?** Deferred feature-work, communication to specific customers, scope reduction on an upcoming launch, agreement on a red window, authorisation to invest in a specific reliability improvement.

### 2.2 The monthly product-lead brief

A one-page written document the function delivers to the product lead after the monthly SLO review (chapter 04 §5). The document has four sections:

- **Headline.** One sentence stating the current budget state and its implication. "We spent 60% of this month's tier-1 budget; feature deploys continue; we are watching a trend that could force a red window next month." Or: "We spent 100% of this month's tier-1 budget; non-critical deploys are frozen for 30 days per the error-budget policy; the top three contributors below."
- **Budget decomposition.** A short list — three to five lines — of the largest budget-burning events of the month, each translated into business-impact language. Not: "15-minute P99 breach of the chat-service SLO on 2026-11-02 due to vLLM scheduler regression." Yes: "15-minute degradation of the chat product on 2026-11-02 affecting approximately N premium-tier users during their peak-usage window, root-caused to a serving-engine regression that has since been fixed."
- **What changes now.** The three decisions the SLO review authorised that affect the product organisation. "Red window declared for 30 days, feature deploys frozen except incident response"; "Launch-X is still planned for 2026-11-20; we assess it as safe inside the current budget"; "We are escalating the recurring class-Y incident to the architect peer and expect a design response within one quarter."
- **What the function asks of product.** Explicit asks: acknowledge the red window; co-sign the customer communication for the 2026-11-02 incident; attend next month's SLO review; support the architect escalation when it lands on your plate.

The document is written for a 10-minute read and a 30-minute meeting. It is Git-tracked but is also delivered in the product lead's preferred medium (shared doc, email, Slack) the day of the review.

### 2.3 Converting error-budget-burn to business impact

The translation is mechanical once the signal infrastructure is in place. Three primitives:

- **Per-SLO cost of burn.** For each SLO, the function pre-computes the "cost of a percentage point of burn" in business terms: user-minutes degraded, request count affected, revenue-at-risk (where product can share the number), customer-tier-weighted impact. Mod-103 chapter 05's SLO-authoring discipline includes this costing where product has provided the inputs.
- **Burn attribution.** Per incident or per burn-event, the function attributes the burn: which route, which service tier, which user cohort, which time window. Mod-104's observability pipeline is the source; mod-108's post-mortem structure (mod-108 chapter 04) is the formal attribution record.
- **Composition.** The monthly brief composes per-incident attributions into a decomposition. The three or four events that dominated the month get named; the long-tail gets summarised.

The function does not fabricate business-impact numbers it does not have. If the per-SLO cost-of-burn has not been co-authored with product, the brief states this and names the ask: "we do not yet have a shared cost-of-burn figure for route X; we propose to co-author one before the next review."

### 2.4 Composing with release-freeze decisions

When the forum (chapter 04 §6) declares a red window, the function's communication to product follows a specific sequence:

1. The forum's written minute is delivered to product within 24 hours.
2. A one-paragraph summary of the decision, the trigger, and the scope is included at the top of the monthly brief.
3. The function proposes — in writing — the specific feature-work-to-reliability-work rebalancing the policy authorises. Product decides how to implement it.
4. If product disputes the decision, the escalation path is director → architect, not re-litigation in the SLO review.

The error-budget policy was signed by both leads (mod-103 chapter 04); the forum applies the policy; the brief delivers the decision. The product lead's authority is in *how* to implement the rebalancing, not in whether to apply the policy at all.

### 2.5 Anti-patterns

- **The "we had a bad month" summary.** The brief describes the pain in engineer-language without translating to business impact. Product nods and does nothing.
- **The surprise red window.** The function declares a freeze that product first hears about in the brief. The policy was signed; the signal was trending; the function did not pre-flag the likelihood. Trust erodes.
- **The engineering self-blame.** The brief dwells on "we should have caught this earlier." Product's response is to feel bad on the function's behalf, not to make a product decision. The brief states the facts, the decision, and the ask.
- **The overclaim on business impact.** The function attributes revenue loss it cannot verify. Product finance disputes the number; the whole brief loses credibility.

## 3. Translating for the architect peer

### 3.1 What the architect needs

The `ai-infra-architect` peer (level 40) operates at the scope where no single service's design is sufficient. The architect's input from the function is a *pattern claim* — not a single incident, but a class of incidents whose recurrence points at a limitation of the current architecture. The architect's output is a decision on whether the class is worth addressing architecturally and what the design response is.

Three questions the function answers for the architect on a quarterly cadence:

- **What recurring incident classes are we seeing?** Classes defined by failure mode (mod-105's catalogue), not by date.
- **Which of these classes point to an architectural limitation the function cannot fix inside its own scope?** The function's self-honesty here is what makes the escalation credible.
- **What specific architecture ask would remove the class?** Not a solution, a proposal — the architect may redesign the ask, but the function names what would, in principle, work.

### 3.2 The architecture-ask ticket

The function's escalation to the architect is a specific artifact: a written ticket in the format the organisation uses for architecture decisions (RFC, ADR, design doc — whichever is the convention). The ticket has four sections:

- **The class.** A one-paragraph description of the incident class, using mod-105's failure-class vocabulary. Representative incident IDs linked. Frequency of the class per quarter, trend over the last year.
- **Why the function cannot close it.** The architectural property that makes the class recurring. This is the honest claim the function must stand behind.
- **The impact.** The composition of this class's budget cost over the past quarter: incidents, MTTR total, budget-percentage burned. The chapter-04 §4.4 "recurring-class → architect escalation" decision authorises the ticket; the ticket carries the quantified impact.
- **The ask.** A proposal, not a demand. "A design response that addresses [specific architectural property] would remove this class." The function names the shape of the response and names the SRE-side owner who will stay in the conversation.

The ticket is filed by the SRE lead, co-reviewed by the SRE team before filing, and escalated into the architect's intake process. The function's SLO-review minutes are the primary evidence pack.

### 3.3 Standing time in the architect's intake rhythm

Many architecture functions have their own quarterly rhythm — architecture review board, ADR triage, roadmap planning. The function proposes a standing slot in that rhythm — one hour per quarter — in which the function walks the past-quarter escalation tickets. The slot is predictable; the content varies.

If the architect's intake does not have a slot, the function proposes one. The alternative is ad-hoc escalations that go stale.

### 3.4 Composing with the maturity ladder

Chapter 01 §5's maturity ladder sits above the function at level 4 and level 5. The architect peer's ask of the function, in return, is often "where are you on the ladder; what are the gaps that are keeping you off the next level?"

The function answers in writing. The annual programme review document (chapter 01 §2.5) is the artifact that carries the current level and the gap list. The architect reads it; the architect's roadmap for the function's substrate is informed by it; the architect's response to the escalation tickets is scoped by it.

### 3.5 Anti-patterns

- **The single-incident escalation.** One incident is not a class. The architect correctly bounces it back as an incident-response question (mod-108) not an architecture question.
- **The "fix our problem" demand.** The function demands a solution, not a design response. The architect's authority to shape the response is what the architect brings; a demand abuses the escalation.
- **The hidden fix.** The function proposes an architectural change and the suggested solution is really within the function's own scope (a different HPA, a different autoscaler, a different runbook). The architect sees through it; the function's credibility declines.
- **The absent follow-through.** The function files the escalation and then disengages. The architect designs a response; the response needs an SRE-side co-owner; the function was not present. The response goes stale.

## 4. Translating for leadership and the board

### 4.1 What leadership needs

Director / VP / C-suite / board audiences read reliability as one line item in a quarterly leadership packet alongside security, compliance, engineering productivity, and roadmap progress. The function's quarterly narrative is one or two pages inside a 30-page packet, read in three to five minutes.

Three questions the function answers at this altitude on a quarterly cadence:

- **Is our reliability posture improving or declining?** A single-line answer with a trend arrow.
- **What is the biggest reliability risk in the next quarter?** Named, with a one-paragraph explanation and the function's response plan.
- **What investment is the function asking for?** Headcount, tooling spend, cross-team support — specific asks, each with a cost and a return.

### 4.2 The quarterly reliability narrative

A two-page document for the quarterly leadership review. The document's shape:

- **Headline (one line).** "This quarter: reliability posture steady; one tier-1 service missed its SLO; red window invoked once; we are investing in [X] for next quarter."
- **The three numbers that matter.** SLO attainment rate across tier-1 services (percentage), count of SLO breaches, DORA four-keys trend (improving / stable / declining per key). Each with a quarter-over-quarter delta. Mod-111 chapter 06 is the source for DORA.
- **The quarter's reliability narrative (half page).** Prose. Three to five sentences on the quarter's largest events, their resolution, the lessons captured. No engineer vocabulary; no acronym without expansion.
- **The top three risks for next quarter (half page).** Named risks with likelihood / impact. For each, the function's planned response.
- **The ask (one paragraph).** Specific, named, costed. "Headcount: one additional SRE for the inference tier. Tooling: approval of the observability-stack RFC [chapter 02]. Support: executive sponsorship for the architect-escalation ticket on class-Y."
- **The programme-maturity score (one line).** Chapter 01 §5's score, with last-quarter delta.

The narrative is Git-tracked and is the input to the next quarter's planning. It is also the document the senior architect (chapter 03 §3 of mod-113) reads as the annual review input.

### 4.3 Board-adjacent narratives

For board-adjacent audiences — the company's board or a frontier-safety or risk committee — the narrative is a further compression. One slide, plus the function's quarterly document as an appendix. The slide has:

- **The one-line headline.** Reliability-posture trend.
- **The three numbers.** As above.
- **The one risk the board should know.** The highest-likelihood-and-impact risk from the quarterly narrative, framed in business terms.
- **The one ask.** The single ask that the function's leadership wants the board's support for.

Board audiences are not debugging the SRE function; they are confirming that the function is doing its job and signalling where they need to intervene. The compression is what makes the signal legible.

### 4.4 The boundary with governance narratives

**This is the critical boundary.** The function's quarterly and board-adjacent narratives are about **reliability** — the operational posture of the production AI systems. They are not about **AI governance** — the organisation's stance on model-safety classification, responsible-release-tier commitments, regulatory compliance posture, or the ethical frame for the AI products.

Governance is the domain of the `head-of-ai-governance`, the CAIO, the chief ethics officer, the legal and compliance peers. The function **composes** with these peers (mod-110 chapter 06; chapter 03 §4 of this module) and emits signals they consume (audit log, drill outcomes, incident classifications), but the function does not author the governance narrative.

The function's board-adjacent narrative therefore stays inside its charter:

- **Yes.** "Our reliability posture declined this quarter because of [specific technical cause]; here is our response."
- **Yes.** "Our DR drill outcome was [specific score]; here is the posture for the next region-outage class we have not yet drilled."
- **Yes.** "Our [specific] investment ask would reduce [specific operational risk]."
- **No.** "Our AI safety posture is [X]." — governance.
- **No.** "Our regulatory compliance against [regulation] is [X]." — legal and compliance.
- **No.** "Our product-tier risk classification is [X]." — governance and product-safety.

A function that drifts into governance language in its board-adjacent narrative is a function that will be told, politely, to step back. The anti-pattern is addressed by discipline: the function's narrative is reliability, scoped to the mod-101 chapter 01 charter, with explicit handoffs to the governance peers for anything outside.

### 4.5 Anti-patterns

- **The engineering lecture.** The narrative reads like a technical post-mortem. Leadership stops reading halfway through.
- **The governance drift.** The narrative claims authority the function does not have (see §4.4). Governance peers push back; the function's credibility on actual reliability declines.
- **The no-ask narrative.** The function reports status and asks for nothing. The next quarter's planning allocates nothing to reliability; the function's scope narrows.
- **The overclaimed improvement.** The function claims a reliability win that one or two months later turns out to have been a measurement artifact or a lucky-month. Leadership's trust in the function's numbers declines.
- **The missing bad news.** The function omits a reliability failure from the narrative in the hope that it will not surface. It surfaces anyway (via incident reporting, media, or regulatory filing); the function's credibility is damaged irretrievably. Leadership's rule: no surprises.

## 5. The written-cadence calendar

The function tracks the upward-communication cadence explicitly:

| Artifact | Audience | Frequency | Triggering ritual |
|---|---|---|---|
| Monthly SLO-review minute | SRE team + product | Monthly | Monthly SLO review (chapter 04) |
| Monthly product-lead brief | Product lead | Monthly | Monthly SLO review |
| Architect-escalation ticket | Architect peer | As authorised | Monthly SLO review §4.4 |
| Architect-standing-slot walkthrough | Architect peer | Quarterly | Quarterly retrospective (chapter 01 §2.4) |
| Quarterly reliability narrative | Leadership / director | Quarterly | Quarterly retrospective |
| Board-adjacent slide | Board / exec committee | Quarterly or annually | Annual programme review (chapter 01 §2.5) |
| Senior-architect roadmap input | Senior architect | Annually | Annual programme review |

The cadence is Git-tracked in the programme's RACI document (chapter 01 §4). The function does not slip the cadence under meeting pressure; the cadence is the commitment.

## 6. The written-standards calendar

Each artifact has a length target, a format convention, and a review loop:

- **Monthly product-lead brief.** One page. Written doc (shared with product), Git-mirrored. Reviewed by SRE team before send.
- **Architect-escalation ticket.** One to two pages. In the organisation's ADR / RFC format. Reviewed by SRE team before filing.
- **Quarterly reliability narrative.** Two pages. In the organisation's quarterly review format. Reviewed by SRE team and by the director's chief of staff (if present) before submission.
- **Board-adjacent slide.** One slide plus the quarterly narrative as appendix. Reviewed by the director before the board packet.

A function that produces excellent artifacts but delivers them in formats the audience does not read loses the information. The standards calendar is the function's self-discipline.

## 7. The first-quarter of upward communication

For a new programme or a new team lead with no previous upward-communication rhythm:

- **Month 1.** Stand up the monthly product-lead brief. First brief will be rough; the data is incomplete; the business-impact translation is uncertain. Deliver anyway; iterate.
- **Month 2.** First architecture-escalation ticket candidate from the SLO review. Draft it; walk it with the SRE team; file it. The first ticket is less important than the signal that the function is willing to escalate.
- **Month 3.** First quarterly reliability narrative. Keep it short. The narrative's credibility compounds over the next three to four quarters.
- **Month 6.** Standing slot in the architect's intake rhythm, if not already present. Agreed.
- **Month 12.** First board-adjacent narrative (if the function is operating at that altitude) or first senior-architect roadmap input (chapter 03 §3 of mod-113). The written record of the previous twelve months is the evidence.

By end of year one, the function has a Git-tracked record of its upward communication — the briefs, the tickets, the narratives — that is itself the evidence of the programme's maturity.

## Summary

- Upward communication is the function's responsibility, not the audience's. Each audience — product lead, architect peer, leadership / board — hears reliability signal differently and the function tailors the frame to each.
- The product-lead translation converts error-budget-burn into business-impact language, delivered as a monthly one-page brief with budget decomposition, the month's decisions, and an explicit ask.
- The architect-peer translation converts recurring incident classes into architecture-ask tickets, filed through the architect's intake rhythm with a standing quarterly slot.
- The leadership / board translation converts a quarter of operational signal into a two-page reliability narrative — headline, three numbers, narrative, top-three risks, ask, maturity-score — with further compression to one slide for board audiences.
- The function's board-adjacent narrative stays inside the reliability charter. Governance narratives — AI safety classification, regulatory posture, ethics — belong to `head-of-ai-governance`, CAIO, legal, and compliance. The function composes with those peers but does not claim their authority.
- A written-cadence calendar names every upward artifact, its audience, its frequency, and the triggering ritual. The function treats the cadence as a commitment, not a best-effort.
- A written-standards calendar names each artifact's length, format, and review loop. Excellent content in the wrong format is information that is not received.
- The first year of the function's upward communication builds a Git-tracked written record that is itself the evidence of the programme's maturity and the input to the next year's investment decisions.

Chapter 06 closes the module by placing the function's career trajectory, the certifications that carry weight in the enterprise-hiring market, and the open-source contribution surfaces that compose with the function's day job.
