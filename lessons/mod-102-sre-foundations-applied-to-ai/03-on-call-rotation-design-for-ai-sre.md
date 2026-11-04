# On-Call Rotation Design for an AI SRE Team

## Why this chapter exists

A rotation written on a whiteboard the first week of a team's life is the single artifact most responsible for whether the team is alive in two years. Sustainable rotation shape protects the people in it; unsustainable shape produces burnout, erodes response quality, and makes the error-budget policy unenforceable. The AI surface adds three complications to the classical rotation-design problem:

1. The escalation tree has peer SMEs at the *same level* you need to page (kernel SME, inference-engine SME, ML-platform SME, eval SME, security SME) — more than classical SRE does, because the AI stack has more owned peer surfaces.
2. The failure classes are more varied (classical infra + non-deterministic quality + provider-degraded), so the "who is the right pager" problem is harder.
3. The on-call-health signal has to be more nuanced, because the page-volume ratio of "truly actionable AI quality signal" to "false-positive drift alert" is a tuning problem that lives inside your dashboard, not inside the pager.

The reference for the classical rotation design is [SRE Workbook, Chapter 8 — On-Call](https://sre.google/workbook/on-call/) and [SRE Book, Chapter 11 — Being On-Call](https://sre.google/sre-book/being-on-call/).

## 1. Rotation shape — follow-the-sun vs. single-region

### Follow-the-sun

Two or three regions (typical: Americas, EMEA, APAC) with hand-off between them so no engineer is paged outside business hours.

**When it fits an AI SRE team.** You have ≥ 5 engineers per region, the product has global 24×7 user traffic, and the platform has enough load to produce realistic page volume in every region's working hours. Frontier labs and large model-hosting providers run FTS because their serving loads do not quiet overnight.

**Costs.** Significant. You need ≥ 3 engineers per region (two primaries + one secondary, with coverage for PTO and illness), plus hand-off ritual every 8–12 hours, plus cross-region shared state (runbooks, incident channels, oncall platform).

### Single-region

One region, one rotation, engineers go on-call for overnight hours with compensation / sustainability discipline (the SRE Book's "overnight page quota" rule of thumb: ≤ 2 interrupts per overnight shift sustained; more than that and the shape is broken).

**When it fits an AI SRE team.** Smaller teams (typical: 4–8 engineers total), traffic has a diurnal pattern (business-hours heavy, overnight light), and the platform's failure modes are mostly classical infra that auto-remediation can handle until morning. Most Year-One AI SRE teams start here.

**Costs.** Overnight pages consume engineer sleep and compound into on-call avoidance behaviour. A single-region rotation with poor paging discipline fails fast.

### The practical default for a new AI SRE team

Start single-region. Drive the page volume down to ≤ 2 actionable pages per 24h *in the rotation's business hours* (not overnight — overnight pages are the top-of-the-stack problem); split to FTS only when overnight page volume crosses the sustainability threshold.

**Common anti-pattern.** Splitting into FTS *before* page volume is tuned. You now have twice the rotation overhead and the same noisy pager, scaled across two regions. Fix the pager first.

## 2. Primary + secondary + incident commander

The three distinct roles in a healthy rotation.

### Primary on-call

Pager-holder. First responder. Owns triage. Expected response: MTTA within 5 minutes during business hours, 15 minutes overnight (adjust for your SLO). Does the first pass of the runbook; declares an incident if the page cannot be resolved within 30 minutes of triage.

### Secondary on-call

Backup. Two functions:

1. **Pager failover.** If primary does not acknowledge within the alert window (typical: 10 minutes), the alert fans out to secondary. Prevents single-person failure.
2. **Rolling incident shadow.** When primary declares an incident, secondary joins as a second brain — reads the runbook alongside, picks up adjacent on-call work (so primary can focus), takes over as primary if primary needs to tap out.

Secondary is not "the other oncall who is sleeping." Secondary is a role with its own expected response, its own compensation, and its own shift. Teams that under-staff secondary end up with single-engineer incident responses, which is where mistakes come from.

### Incident commander (IC)

Separate role. Not necessarily an engineer. During a declared incident, the IC owns:

- **Communication.** Status page updates, internal stakeholder comms, cross-team coordination.
- **Decision authority.** Who is paged next (the SMEs below), when the fallback contract is activated, when the postmortem is scheduled.
- **Timebox discipline.** Keeping the technical responder focused on the technical work by taking all non-technical work off their queue.

The IC is a separate role *even when the same engineer plays both IC and primary*. The role separation is in writing because the role mixing produces well-documented failure modes (the technical responder gets pulled into stakeholder comms and loses the thread of the diagnosis).

For an AI SRE team at level 35, the IC rotation is typically a small shared rotation across senior engineers from the SRE team *and* senior adjacent engineers (an eval peer, a platform peer); not every page escalates to an incident, so the IC on-call volume is lower than primary volume.

## 3. Hand-off discipline

Every rotation has hand-offs. Every hand-off is an opportunity for state to drop. Enforce a hand-off ritual.

### The hand-off checklist

Written, filed in a channel, reviewed by both outgoing and incoming on-call:

1. **Open incidents.** List every incident still open at hand-off, with current state, next action, next-action owner, and expected resolution time.
2. **Open pages.** Alerts currently firing or recently resolved but not yet root-caused.
3. **Known-bad state.** Any known degraded state that is *intentional* or *awaiting fix* that must not trigger a confused response (e.g., "a KV cache was manually flushed on pool B, TTFT is elevated as prefix-caching re-warms — expected to normalize by 14:00").
4. **In-flight changes.** Any deploy, model promotion, prompt template update, or provider route change in flight during the shift.
5. **SLO budget state.** Any SLO under 50% budget remaining, and the policy band in effect (chapter 02).
6. **Recent post-incident actions.** Fixes landed in the last 24h that could still produce confused behaviour if another incident fires against the same surface.

A hand-off longer than 10 minutes is a signal that your runbooks or your dashboards are not doing their job and the human is carrying state that should be in the system.

### Hand-off anti-patterns to avoid

- **Chat-DM hand-offs.** Keep them in a durable channel or document so the incoming on-call can review *before* the shift starts and so the state is auditable.
- **Oral-only.** Written log is the source of truth; voice is for emphasis, not primary signal.
- **Shift-start reads.** The incoming on-call should read the hand-off *before* their shift starts, not as the first act of it. Build this into the schedule.

## 4. On-call-health metrics

[SRE Workbook, Chapter 8 — On-Call](https://sre.google/workbook/on-call/) names the core signals. The AI surface adds a quality dimension to each.

### Page volume

**Signal.** Pages per week, stratified by shift (business-hours vs. overnight) and by alert source (SLO burn, infra alert, quality-signal alert, provider-status alert).

**Target.** The SRE Book's rule of thumb — no more than two interrupts per on-call shift — applies. AI surfaces frequently violate this early because quality-signal alerts are poorly tuned. Treat quality-alert noise as a dashboard problem to fix, not as a load-shedding problem for the human.

### Actionable rate

**Signal.** Of the pages received, what fraction resulted in a meaningful response (investigation, fix, documentation update)? The complement is the false-positive rate — pages that resolved to "nothing to do."

**Target.** ≥ 70% actionable. Below 50% is a signal to audit the alerting layer; the humans will start to treat the pager like background noise.

**AI-specific hazard.** Quality signals constructed from judge models have inherent noise. If your hallucination SLO fires on a single 5-minute window, you will page on judge-model variance. Multi-window / multi-burn-rate alerting (see [SRE Workbook, Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)) is mandatory on constructed signals.

### Sleep disruption

**Signal.** Count of overnight pages per week per rotation member, with their explicit well-being self-report at the end of shift.

**Target.** ≤ 2 overnight pages per week per primary. Overnight pages are the first lever to pull when the rotation is unhealthy.

### Cognitive load

**Signal.** Qualitative, asked at the end of each shift. "On a 1-5 scale, how exhausted are you?" Not a KPI; a weather report. A pattern of 4s and 5s is the signal to re-shape.

## 5. Model-aware escalation — the SME ladder

The AI stack has more owned peer surfaces than classical infra, so the escalation tree has more branches. This is the model-aware part of AI SRE on-call design.

### Who is paged for what

| Symptom | Primary diagnosis owner | Who the IC pages next if primary needs help |
|---|---|---|
| Classical infra (CPU / memory / network / scheduler / 5xx) | AI SRE primary | Platform SRE, on-call engineer for the cloud provider |
| GPU fault (XID, DCGM alarm) | AI SRE primary | GPU peer (ai-infra-performance peer at level 35), NVIDIA GPU Operator maintainer, cloud GPU support |
| Serving-engine misbehaviour (vLLM OOM, SGLang scheduler thrash, TRT-LLM engine crash) | AI SRE primary (triage with knob fluency — chapters 04–06) | **Inference-engine peer** (ai-infra-inference-engineer peer at level 35) |
| Kernel regression (new engine release with slow FlashAttention path) | AI SRE primary (confirms the regression surface) | **Kernel SME** (ai-infra-performance peer at level 35) |
| Quality regression (hallucination spike, judge-pass-rate burn) | AI SRE primary (confirms it is real, not judge-model drift) | **Eval SME** (ai-eval-engineer peer at level 30) + product owner |
| Model-artifact handoff issue (promotion from training to serving) | AI SRE primary (confirms infra-side wiring) | **ML-platform SME** (ai-infra-ml-platform peer at level 30) |
| Provider degradation / outage | AI SRE primary + IC | Vendor support, product owner (if user-visible), security peer (if data-handling change) |
| Guardrail / PII filter misbehaviour | AI SRE primary (confirms reliability knock-on) | **Security peer** (ai-infra-security peer at level 35) |

This is the same deferral map from `mod-101` chapter 02, re-shaped for live-page routing.

### The escalation runbook per SME

For each SME on the ladder, write a one-paragraph runbook that includes:

- **How to page.** The specific channel / rotation handle / escalation policy.
- **What to send.** Minimum context (incident ID, affected SLO, hypothesized failure class, what you have already tried).
- **Expected response window.** Primary SME MTTA on their rotation.
- **What they will ask you for.** The telemetry / logs / repro they will need to pick up the ball. Pre-collect so the hand-off is clean.

The SMEs are peer-level engineers on their own rotations. Respect their contract: do not page the kernel SME for a prompt regression.

## 6. The paging-policy minimum viable set

A rotation is only as good as the paging policy that feeds it. The AI-specific additions to the classical paging policy:

- **Page on burn rate, not single-sample excursion** — standard. More critical on AI because quality signals are noisy.
- **Multi-window alerts** — e.g., 2% in 1h *and* 5% in 6h for a fast burn; 10% in 3d *for* a slow burn. [SRE Workbook, Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/).
- **Correlated-signal silencing.** If availability and quality are both burning, page once (as an incident), not twice. See chapter 02's reconciliation rule.
- **Provider-status ingestion.** The status pages of your providers (OpenAI, Anthropic, Google, AWS, Azure) are a *signal*. Wire them into your paging stack (`mod-107`) so the correlation "provider X reported degraded at 03:12 UTC, our quality SLO started burning at 03:14 UTC" fires one page, not two uncorrelated ones.
- **Change-window paging.** During a declared deploy / model-promotion / prompt change window, alerts on the burning surface are routed to the deployer, not the oncall. The oncall is the escalation, not the primary triage.

## Summary

- **Shape.** Start single-region; split to follow-the-sun only when overnight page volume crosses the sustainability threshold. Tune the pager before you split.
- **Three roles.** Primary (pager-holder), secondary (backup + incident shadow), IC (comms + decision + timebox). Written separately even when staffed by the same engineer.
- **Hand-offs.** Written, durable, pre-read before shift start. Six-section checklist: open incidents, open pages, known-bad state, in-flight changes, SLO budget state, recent remediation.
- **On-call health.** Page volume, actionable rate, sleep disruption, cognitive load. AI surfaces frequently violate the two-interrupts-per-shift rule early via noisy quality alerts — fix the dashboard, do not load-shed to the human.
- **Model-aware escalation.** Peer SME ladder with eight canonical symptom routes. Each SME has a one-paragraph runbook describing the page.
- **Paging policy.** Burn-rate, multi-window, correlated-signal silencing, provider-status ingestion, change-window routing.

The next three chapters (04, 05, 06) build the specific engine-knob fluency the primary on-call needs to triage the inference-engine symptoms in the table above without escalating to the inference-engine peer on every page.
