# Exercise 03 — Public Incident Case Study: OpenAI 2023 and Subsequent Frontier-Lab Outages

**Estimated effort:** 2 hours · **Output:** one Markdown case-study report (~1500 words) covering two incidents and tying each to a specific clause in your fallback contract

## Objective

Walk through two public provider-incidents from the historical record (chapter 03), reconstruct the on-caller's perspective, and translate the lessons into concrete clauses in your own fallback contract (exercise 02) or dependency map (exercise 01). The purpose is calibration: unless the AI SRE has internalised how long and how broad a real outage can be, the contract is written to the wrong severity.

A good deliverable reads like a learning-from-incidents report: timeline, impact, the gap each incident exposed, and the specific change it would have forced in *your* system.

## Prerequisites

- Chapter 03 of this module.
- Access to the public status pages (status.openai.com, status.anthropic.com, status.cloud.google.com, status.azure.com, health.aws.amazon.com) — their historical incident views are public.
- Your exercise-01 surface map and exercise-02 fallback contract for the route you will analyse against. If you have not completed those, use an illustrative route (e.g., "chat.completions primary = OpenAI, secondary = Anthropic").

## Problem statement

Choose two incidents from the list below (or an equivalent documented event) and author one case study per incident. One incident must be from the frontier-lab tier (OpenAI / Anthropic / Google DeepMind product); one may be from the hyperscaler infrastructure tier (GCP / AWS / Azure).

### Candidate incident list

Pick two. Each is public and researchable; the write-up must cite primary sources (status page history, the vendor's own post-mortem if one exists, or a reputable summary with a link to the status entry).

- **OpenAI, November 2023.** Multi-day DDoS and post-DevDay degradation. Multiple products affected; sustained 5xx and connection drops across the API.
- **Anthropic, launch-period overload.** One of the Claude 3 / 3.5 / 4 family launch-period overload windows with elevated `overloaded_error` responses. Pick a specific month-event from status.anthropic.com history.
- **Google Cloud regional incident.** A documented regional Vertex AI or related Google Cloud networking / compute event; select from status.cloud.google.com history.
- **AWS Bedrock throttling event.** A documented throttling or capacity event on Bedrock; select from AWS Health history or AWS post-incident summaries.
- **Azure OpenAI capacity event.** A documented capacity-contention or regional-degradation event on Azure OpenAI; select from status.azure.com and Azure OpenAI service-health advisories.
- **Cloudflare / Fastly correlated event.** A CDN or WAF incident that affected multiple AI providers simultaneously (common-mode failure).

### Report structure per incident

Author each case study with the following sections.

#### Section 1 — Incident summary

One paragraph: when it happened, how long it lasted, which products / services were affected, how the vendor framed it. Cite the primary source (status-page URL, vendor blog post, or archived capture) with a date-of-check.

#### Section 2 — Reconstructed timeline from the caller side

Hour-by-hour (or minute-by-minute if the incident was shorter): what the caller observed, what the status page said, how far apart the two were. Use the vendor's status updates as the spine; annotate with what a careful operator would have observed on their own dashboard (error rate, p95 latency, queue depth, retry rate).

If exact numbers are not public, use qualitative descriptors ("sustained elevated 5xx") rather than invented specifics.

#### Section 3 — Gap analysis

Walk through the fallback-contract sections from chapter 02 and name the gap this incident would have exposed in *your* current setup:

- **Health signals.** Would the breaker have opened in time? Was the error class in your taxonomy? Did the status-page lag matter?
- **Equivalence table.** Was the secondary available during the event? Were the quality tiers calibrated correctly?
- **Policy.** Did the policy match the event's shape (long tail, correlated failures, residency-forced hard-fail)?
- **Semantics.** Would idempotency, streaming resumption, or tool-use multi-turn have failed under the pressure of the event?
- **Error budget (chapter 05).** How much of a monthly budget would this event have consumed? Was your reserve sized to absorb it?

#### Section 4 — Changes forced

A list of specific, named changes the case study would have forced in your artifacts:

- Map changes (exercise 01).
- Contract changes (exercise 02).
- Error-budget table changes (exercise 05).
- Scorecard changes (exercise 06).

Each item is a one-line "now I would do X because of Y."

## Requirements

1. **Two incidents.** One frontier-lab, one hyperscaler-or-CDN (common-mode or regional).
2. **Primary-source citations.** Each incident references the vendor's status entry or post-mortem with a URL and a date-of-check.
3. **Timeline is reconstructed, not invented.** Gaps in the public record are stated as such; do not fabricate specifics.
4. **Gap analysis covers all five contract sections** (health signals, equivalence table, policy, semantics, budget).
5. **Changes forced section is actionable.** Each bullet is a concrete edit to a named artifact, not a sentiment.
6. **Reasoning is grounded in your route.** The report references the exercise-01 route and exercise-02 contract where applicable.

## Starter guidance

- Status pages archive incident updates after resolution. Start from the status-page history page and work backward. Many incidents have a "post-mortem" entry days later.
- The 2023-11 OpenAI event is well-documented and the easiest first-case; the Anthropic launch-period overloads require more work to pin to specific dates — look for correlated spikes in third-party community forums (dev.to, r/OpenAI, Hacker News) that date-stamp observed problems.
- If you cannot find reliable primary sources for a specific incident, pick a different one. Do not infer numbers.
- Avoid armchair quarterbacking. The case study's value is in the "what *you* would do," not in critiquing the vendor's response.
- The multi-day shape of the 2023 OpenAI event is the most important calibration point: most teams plan for 90-minute outages and are surprised by multi-day degradations.

## Acceptance criteria

- Markdown report ≤ 2000 words covering two incidents.
- Each incident has summary, timeline, gap analysis, and changes-forced sections.
- Every factual claim cites a primary source with a date-of-check.
- Gap analysis touches all five contract sections for each incident.
- Changes-forced section has at least three concrete edits per incident to named artifacts.
- No invented numbers; qualitative descriptors where the record is thin.

## Stretch goals

- **Correlated-event case study.** Pick a Cloudflare, Fastly, or DNS-provider event that affected multiple AI vendors at once. Analyse how much of your "diversified" vendor set would have survived; name the common-mode in your map.
- **Replay as a tabletop.** Run the case study as a 30-minute tabletop exercise with the service team: present the incident's early signals and let the team describe what they would do, step by step, with their current tools. Capture the outcomes.
- **Status-page-lag measurement.** For your chosen incident, measure the lag between the first observed effect (in community forums or your archived logs) and the first status-page update. Use the result to calibrate the chapter 04 ingestor's expectations.
- **Deprecation case study.** Pick one documented model deprecation (OpenAI, Anthropic, Google) and walk through the vendor's notice-to-shutdown timeline from the perspective of a dependent service. What migration path would have been forced? What was the exit cost?
- **Vendor post-mortem readout.** Choose a vendor whose post-incident report you find well-written and one you find poorly-written. Analyse what made the difference and what that tells you about the vendor's operational culture.
