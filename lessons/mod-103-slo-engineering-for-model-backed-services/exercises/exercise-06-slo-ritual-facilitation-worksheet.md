# Exercise 06 — SLO Review Ritual Facilitation Worksheet

**Estimated effort:** 2 hours · **Output:** two Markdown files — (1) the facilitation worksheet template, and (2) a dry-run meeting transcript with resolution notes

## Objective

Produce the two artifacts that make the monthly SLO review ritual from chapter 06 section 6 operable: (1) a reusable facilitation worksheet the chair of the review uses to run the meeting, and (2) a dry-run of one meeting against the SLO stack from exercises 01–05, walking through a plausible set of burn events, exceptions, and target adjustments.

The worksheet is the artifact your SRE team uses every month. The dry-run is the proof the worksheet works.

## Prerequisites

- Exercises 01–05 completed. (You need the stack and the policy to review.)
- Chapter 06 of this module, especially section 6.

## Problem statement

Your SLO review is scheduled for the first Thursday of every month. Nobody has run it before. The attendees — primary and secondary on-call, SRE lead, business sponsor, eval peer, capacity owner — have thirty minutes together. You are the chair, and if you do not produce an agenda, the meeting runs long, the sponsor feels their time was wasted, and the next month's attendance drops.

Produce the two artifacts.

## Part A — the facilitation worksheet template

A reusable document the chair fills in for each monthly review. Six sections, keyed to the standing agenda from chapter 06 section 6.

### Section 1 — SLI health snapshot

A pre-populated table template. One row per SLI. Columns: SLI name, 28-day value, target, budget remaining, trajectory (up / flat / down vs. prior review), owner. The worksheet template provides the column headers; the chair fills values before the meeting from the Grafana dashboards.

### Section 2 — Burn-event review

For each SLI that was in burn-rate alert since the last review, a short block to be populated:

- SLI and incident reference.
- Duration of burn.
- Root cause (one line).
- Action taken (rollback / capacity add / exception grant / no action).
- Lessons captured and where (post-mortem link, runbook update, SLI-definition change).

### Section 3 — Exception log review

Every exception granted under the policy (chapter 05 section 7) since the last review. Columns: exception, who granted, how long, triggered by, resolution (absorbed into policy / marked one-off).

Section-ending question: were any exceptions granted *more than once* for similar triggers? If so, the policy needs an update — flag for section 6.

### Section 4 — Target-adjustment proposals

The agenda item where SLO targets get revised. For each proposed adjustment:

- SLI affected.
- Current target.
- Proposed target.
- Justification — baseline re-measured, model change, workload shift, sponsor renegotiation.
- Who signs off (SRE + sponsor + eval peer for quality SLIs).

### Section 5 — New SLIs / retired SLIs

Any additions or retirements. The retirement checklist from chapter 06 section 7: alerts removed? panels archived? baseline archived? retirement recorded?

### Section 6 — Policy adjustments

Any policy changes coming out of this review. This is where recurring exceptions get absorbed into the policy, where new SLIs trigger policy updates, where the correlation window gets re-set.

### Appendix to the worksheet — meeting-mechanics notes

A one-paragraph section with the chair's practical notes:

- Who runs the meeting (chair, co-chair if applicable).
- How long each agenda item is boxed.
- What happens when an item overruns.
- Where the minutes are published and the circulation list.
- Quarterly allocation refresh — what triggers moving the agenda item up and who proposes the new numbers.

## Part B — the dry-run meeting

A transcript (bullet-point form is fine) of one hypothetical monthly review against the SLO stack from exercises 01–05. Populate the worksheet with plausible values for an *interesting* month — not an all-green month, not an all-on-fire month.

Minimum scenarios to include:

- **One SLI burned fast in the window.** A latency or quality incident that was caught, mitigated, and is now recovered.
- **One SLI burned slow in the window.** A cost SLI drift that triggered a ticket but no page. The review must decide: accept, renegotiate target, or reduce traffic.
- **One exception granted.** The IC granted a 24-hour exception during an incident. Review decides whether it was a one-off or recurring pattern.
- **One target adjustment proposal.** A baseline refresh (model upgraded last month, baseline measurably better now) triggers a tightening proposal.
- **One new SLI proposal.** A product surface launched a new modality and needs a quality SLI the current stack does not cover.

For each, write the actual resolution produced by the meeting — not just the scenario, but the decision the chair drove.

### Timing

Note how many minutes each agenda item took in your dry-run. The meeting is thirty minutes. If your dry-run runs over, cut *content*, not *time* — the discipline of running the review on a clock is part of the ritual.

## Requirements

- **Worksheet template is reusable.** Not filled with exercise-specific values; a chair can take it as-is next month and fill in new data.
- **All six worksheet sections present** with the structure from Part A.
- **Appendix-mechanics paragraph populated** with a realistic process.
- **Dry-run has at least five scenarios** covering the listed categories.
- **Each dry-run scenario has a resolution**, not just a statement.
- **Timing budget respected.** The dry-run meeting fits in thirty minutes (or you cut content to make it fit).
- **Attendee list is explicit.** The sponsor and eval peer are visible in the attendee column; a review without the sponsor is not a review.

## Starter guidance

- The worksheet template should be legible to the chair *two weeks before* the meeting, when they start filling it from Grafana. The chair should not need to interpret anything; they copy values from dashboards into cells.
- The dry-run is the hard part. Pick scenarios that *conflict* with each other — e.g., a latency burn triggered by capacity shortfall, concurrent with a cost SLI drift that argues *against* adding capacity. The resolution is the interesting content; a dry-run of five uncorrelated scenarios is less useful than three correlated ones.
- The target-adjustment scenario should make the tightening-vs-loosening trade-off visible. If baselines have moved in the better direction, tightening is defensible against the sponsor (they get a stricter commitment); if they have moved worse, loosening requires explicit sponsor sign-off because it is a weakening of the user contract.
- The exception review is where recurring patterns get caught. If an IC granted the same shape of exception twice in a quarter, the policy should be updated. Make the dry-run surface at least one such candidate.
- For the appendix mechanics, model on how your organization runs other cross-functional rituals (production readiness review, architecture review, security review). The SLO review rhymes with those rather than being novel.

## Acceptance criteria

- Part A worksheet template is self-contained and ≤ 1500 words.
- Part B dry-run is self-contained and ≤ 2000 words.
- Worksheet has all six sections with template-shaped (not filled) content.
- Dry-run covers all five required scenarios with resolutions.
- Timing budget is explicit and respected.
- Attendees named per scenario (not just at the top of the dry-run).
- A reviewer could pick up the worksheet and chair the next month's review without you being in the room.

## Stretch goals

- **Pre-read packet.** A short document (one page) the chair circulates 48 hours before the review. Contains the SLI health snapshot pre-filled, the exception log, and the proposed agenda additions. Reduces in-meeting discovery time.
- **Decision-log format.** For the dry-run, structure each resolution as a formal decision record: context / decision / consequences. The format makes the review minutes machine-searchable for future reference.
- **Retrospective on the ritual itself.** Add a section to the worksheet appendix: a quarterly meta-review of the ritual — attendance, timing, usefulness. If the sponsor skipped twice, the ritual is failing and needs renegotiation.
- **Sponsor-facing summary.** A one-paragraph post-meeting summary written for a sponsor who cannot attend the full review. Contains the SLI changes, the big decision, the asks of the sponsor. Decouples sponsor engagement from meeting attendance.
