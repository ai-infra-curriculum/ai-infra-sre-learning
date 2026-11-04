# The `ai-eval-engineer` Peer Boundary

## Why this chapter exists

Chapters 01–06 authored a multi-dimensional SLO stack whose quality dimension depends on signals the AI SRE does not construct — judge models, eval sets, calibration against human raters, pass-rate pipelines. Those signals belong to the `ai-eval-engineer` peer at level 30. If the boundary between "signal source" (peer) and "SLO as reliability contract" (you) is not explicit, three predictable failure patterns appear:

1. **SRE reaches into the judge pipeline.** The SRE starts tuning the judge prompt during an incident, the eval peer finds out at review, and the baseline is corrupted silently.
2. **Eval peer sets the SLO target.** The eval peer picks a quality target that reflects what the model *can achieve*, not what the business *requires* — and the SRE's error-budget policy inherits a number nobody defended against the sponsor.
3. **Nobody owns judge drift.** The judge model's agreement with humans decays over time, the signal silently drifts, and the SLO burns for the wrong reason. Each side thought the other was watching.

This chapter is the two-column contract that avoids all three. It is the quality-signal specialization of the peer deferral map from `mod-101` chapter 02 and the structural analogue to `mod-102` chapter 07 (the engine-authoring peer boundary) — same shape, different peer.

The sibling curriculum is `ai-eval-engineer-learning` at <https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning>. Read its charter once so the boundary is reciprocal in your head.

## 1. The two-column contract

| AI SRE owns (SLO as reliability contract) | `ai-eval-engineer` owns (signal source) |
|---|---|
| The SLI definition — good-event predicate, valid-event filter, window, target | The judge model, the eval set, the rule-based classifiers, the sampling methodology |
| The SLO target — what number the business sponsor signs off on | The achievable quality range given the current model + prompt + retrieval |
| Baselining the production SLI over business cycles | Judge-vs-human agreement on the judge pipeline, calibrated out-of-band |
| Prometheus counters, Grafana panels, Alertmanager routes for the SLI | The pipeline that emits per-response `PASS` / `FAIL` / `UNKNOWN` as structured output |
| Burn-rate alert thresholds and their multi-window tuning | Confidence bands and calibration reports accompanying the signal |
| The error-budget policy (chapter 05) that acts on the SLI | The release-gate eval suite (the version of the signal run pre-release) |
| The release-gate wiring — the SLO-shaped integration contract | The eval-set lifecycle — construction, drift monitoring, refresh |
| Rolling back a release that burns the SLO in production | Investigating *why* a gate-passing release burned the production signal |
| Co-investigation when the SLI burns | Co-investigation when the judge signal is suspect |
| On-call primary paging for SLI-burn | Owns the judge-drift incident class |

Every row on the left lands at the AI SRE on-call. Every row on the right lands at the eval peer. Both sides escalate to each other through a named channel (see section 4), not through ad-hoc Slack DMs during an incident.

## 2. Why the boundary sits here

The reliability charter from `mod-101` chapter 01 says the SRE owns the *operational reliability contract* and defers to specialist peers on *craft inside the contract*. The quality-signal construction is craft inside the contract.

Three specific reasons the split is cleaner this way than the alternatives:

- **Signal methodology is a research discipline.** Choosing between a judge-model and a rule-based filter, calibrating the judge prompt, estimating agreement with human raters — this is ML research work. The AI SRE is a reliability craft; grafting signal-methodology ownership onto the SRE gives an SRE on-call rotation that cannot credibly defend signal choices and gives an eval peer that cannot credibly operationalize them.
- **The SLO is a business contract.** The sponsor signs the SLO target; the sponsor does not sign the judge-prompt revision. Keeping the business-facing and craft-facing halves of the surface on separate shoulders means each conversation happens with the right counterpart.
- **Incident response splits cleanly along the line.** When the SLI burns and the question is "is the judge correct about the regression?" the eval peer answers. When the question is "has the release freeze taken effect?" the SRE answers. A single-owner version of the surface collapses the escalation tree.

## 3. The three canonical hand-offs

Three situations where recognizing a hand-off is the right move. Not optional; the whole point of the boundary is that these cases route cleanly.

### Hand-off 1 — A new quality SLI needs authoring

Symptom: the team wants to add a quality dimension to the SLO stack — say, "citation-correctness rate" for a RAG surface — and no signal exists today.

**Correct move.**

- The **AI SRE** writes the SLI *contract*: good-event predicate, valid-event filter, target shape, how it fits the stack, which release-gate decisions it informs.
- The **eval peer** builds the signal pipeline: judge prompt, sample selection, calibration against human labels, pass-rate emission.
- Both meet on the integration: the attribute name emitted on the trace, the metric name the SRE consumes, the versioning scheme on the pipeline.

**Incorrect moves.** The SRE picks the judge prompt without eval-peer involvement. The eval peer defines the SLO target without sponsor involvement. Either version ships a brittle SLI.

### Hand-off 2 — A quality SLI is burning and the signal is suspect

Symptom: the hallucination-rate SLI dropped from 97% to 92% over 48 hours. No model change, no prompt change, no retrieval change. Either the production traffic distribution shifted (real regression) or the judge model drifted (signal regression).

**Correct move.**

- The **AI SRE** declares the incident, holds the SLO contract, and spends reliability-side investigation on the production-traffic side (traffic mix, retrieval freshness, recent deploys adjacent to but not on the quality signal).
- The **eval peer** investigates the signal side: judge-model version, prompt revisions, pass-rate on the calibration set, agreement with human spot-checks.
- Both share the artifact set (sampled responses with judge verdicts, before-and-after metric slices, calibration deltas).

The decision on whether to roll back a recent release stays with the on-call primary — the SLO burns the error budget regardless of whether the signal is accurate. If the signal turns out to have drifted, the eval peer fixes the pipeline *and* the SRE restores the historical SLI over the drift window (section 6 retroactive adjustments).

**Incorrect move.** The SRE tries to debug the judge pipeline; the eval peer tries to decide the rollback. Both overstep.

### Hand-off 3 — A gate-passing release burned the SLO

Symptom: a new model was promoted through the release-gate (eval suite passed), rolled out to canary, promoted to full, and the production hallucination SLI began burning within 24 hours.

**Correct move.**

- The **AI SRE** rolls back (via `mod-112` machinery) under the policy from chapter 05 — the SLO burn takes precedence over the gate-passing signal.
- The **eval peer** investigates the gate-vs-production divergence: was the eval set stale against production traffic? Did a judge calibration shift occur between gate run and production run? Was there a sampling bias in the gate?
- The output of the eval peer's investigation updates the gate (adds coverage, re-calibrates). The SRE's output is the restored SLO and the incident post-mortem.

**Incorrect move.** The SRE waives the rollback because "the gate said it was good." The gate is an *upstream signal*; the SLO is the *downstream contract*. The SRE honors the contract.

### A fourth, implicit hand-off — judge-drift as an incident class

Judge-drift is slow by nature — the judge's agreement with human raters decays over months, not hours. There is no fast-burn alert for it; the eval peer owns a slower cadence of recalibration work. But when the drift *crosses a threshold that material affects the SLI*, the eval peer notifies the SRE, and the SRE either (a) recognizes a baseline shift and updates the SLI target via the lifecycle review (chapter 06 section 6), or (b) flags the drift as an exception under the policy (chapter 05 section 7) until recalibration lands.

The hand-off is: eval peer notifies, SRE updates the SLI definition or baseline, both artifacts get recorded in the SLO review log. No fire, but it is still a managed event.

## 4. Named channels and ticket templates

The boundary works because escalation has a known path, not because both sides pay attention in general.

### The three channels

- **Signal-change channel.** Eval peer announces pipeline version bumps, calibration updates, eval-set revisions. SRE watches; the SLI definition's "signal source and version" field (chapter 06 section 2) gets updated when a bump lands.
- **Incident channel.** Joint incident response when a quality SLI burns and either side of the boundary is in play. Typically your shared incident commander channel from `mod-102` chapter 03.
- **Review channel / cadence.** Monthly SLO review per chapter 06 section 6; eval peer is a standing attendee for the quality-SLI agenda items.

### The ticket template — SRE to eval peer

When the SRE files a ticket at the eval peer (signal is suspect, new SLI needed, judge-drift observed), use a standard template:

- **Title.** One-line symptom.
- **SLI.** Exact SLI identifier from the SLO document.
- **Burn pattern.** The metric trace — value over window, timestamp of first excursion, size.
- **What changed on the SRE side.** Recent deploys, infra events, config changes.
- **What the SRE is asking for.** Diagnosis of signal validity, re-calibration, pipeline-version bump, new SLI.
- **Response hold.** What the SRE is doing in the meantime — rollback, freeze, exception grant.

Boring on purpose, same as the engine-peer template from `mod-102` chapter 07 section 3. The peer should be able to pick up without a meeting.

### The ticket template — eval peer to SRE

The reverse direction is just as important. Common cases:

- **New judge version available.** Peer publishes the new pipeline; SRE schedules the baseline refresh and SLI-definition update.
- **Signal recalibration proposal.** Peer proposes shifting the pass-rate interpretation; SRE evaluates whether the SLO target needs re-negotiating with the sponsor.
- **Eval-set refresh.** Peer refreshes the production-traffic eval set; SRE confirms the release-gate wiring is still intact and schedules the baseline refresh.

A healthy relationship has both arrows populated. If you only *send* tickets to the eval peer and never receive any, you are probably missing signal-change events that should be flowing to you.

## 5. The reverse direction — what the eval peer routes to you

For symmetry with `mod-102` chapter 07, the cases the eval peer sends into your queue:

- **Signal-pipeline rollouts.** New judge model, updated prompt, re-calibration — the peer cuts the version, you own the rollout plan (SLI-definition update, baseline re-run, release-gate rewire). Analogous to the engine-release-rollout hand-off in `mod-102`.
- **Release-gate configuration recommendations.** Peer publishes a recommended eval-suite configuration for a new product surface; you operationalize it (wire to the deploy pipeline, define the SLO it feeds into).
- **Eval-set evolution.** Peer refreshes the eval set to track production distribution; you confirm the SLI's valid-event filter still aligns.
- **Quality-regression incidents the peer surfaces first.** Peer's offline calibration catches a model regression before the production SLI burns; you decide whether to pre-emptively halt the rollout.

## 6. Signals the boundary is drifting

### You are drifting into the peer's work if

- You are editing the judge prompt directly during an incident.
- You are running a judge-vs-human agreement study.
- You are reasoning about statistical calibration of the eval signal.
- You are authoring new evaluation metrics (not SLIs) from scratch.

### The peer is drifting into your work if

- The peer is tuning Prometheus alert rules on the quality SLI.
- The peer is proposing SLO targets directly to the business sponsor without SRE involvement.
- The peer is making rollback decisions during incidents.
- The peer is editing the error-budget policy.

Both drifts are solvable with the two-column contract from section 1 as the artifact. During a cross-team sync, walking the two columns is fifteen minutes; letting the drift continue is a quarter of absorbed work on both sides.

## 7. Shared sources of truth

Both sides read and update the same documents:

- **The SLO document.** Owned by SRE, co-reviewed by eval peer. Quality SLIs name the pipeline version.
- **The pipeline version log.** Owned by eval peer, consumed by SRE. Every pipeline version bump is dated and summarized.
- **The baseline reports.** Owned by SRE, inputs provided by eval peer for quality baselines.
- **The SLO review minutes.** Co-attended; co-signed on quality-SLI agenda items.
- **The incident post-mortems involving quality signals.** Co-authored.

The document set is small on purpose. Every artifact is addressable; every one of them crosses the boundary at least once.

## 8. One worked example — an SLI co-authored across the boundary

Concrete case to close the chapter: a RAG surface adds a "citation-correctness rate" quality dimension to its SLO stack.

**Day 1 — kickoff.** SRE and eval peer meet. SRE writes the draft SLI contract: valid events are RAG responses on knowledge-grounded questions; good events are responses whose citations were verifiable against the retrieval corpus; target: baselined after instrumentation; pages on fast-burn. Eval peer proposes signal: a rule-based citation-extractor plus a judge model that verifies each citation against the retrieved passage.

**Days 2–8 — instrumentation.** Eval peer builds the pipeline, emits `PASS` / `FAIL` / `UNKNOWN` per response as structured output. SRE wires Prometheus counters, Grafana panel, and the OTel attribute on the request span.

**Days 9–36 — baseline.** Four weeks of production measurement. Baseline p (citation-correct) = 91% on average, with a long tail of multi-citation responses at 72%. SRE proposes target: ≥ 90% overall with a stretch warn-only target of ≥ 80% on multi-citation responses. Sponsor signs off.

**Day 37 — go-live.** Alerts activated, SLO added to the stack, release-gate pipeline wired to the signal.

**Day 45 — first burn.** Citation-correctness drops to 84%. SRE pages, investigates traffic — no deploys, no retrieval change. Hands to eval peer. Peer discovers the retrieval-index refresh pushed documents with new citation-format conventions; judge model's citation-extractor misparses the new format. Peer updates the extractor; signal recovers; SRE retroactively adjusts the 28-day SLI to exclude the drift window (documented in the next review).

**Day 65 — SLO review.** Incident reviewed, extractor change documented, target held at 90%. Baseline flagged for refresh in month four to capture the new retrieval distribution.

Six weeks, two peers, one new SLI. The two-column contract routed each decision cleanly; no one stepped on the other's work.

## Summary

- **Two-column contract.** SRE owns the SLI contract, the SLO target, the baseline, the instrumentation, the alerts, the error-budget policy, the rollback authority. The eval peer owns the judge, the eval set, the sampling methodology, the pipeline version, the calibration, the release-gate eval suite.
- **Three canonical hand-offs.** New SLI authoring (co-built), quality-SLI burning with signal suspect (joint investigation along column lines), gate-passing release burns in production (SRE rolls back, peer fixes the gate).
- **Judge-drift as an implicit hand-off.** Peer notifies; SRE updates SLI definition or flags an exception until recalibration lands.
- **Named channels.** Signal-change channel, incident channel, SLO review channel. Ticket templates both directions.
- **Reverse direction.** Peer routes signal-rollouts, gate-configuration, eval-set evolution, offline-caught regressions to the SRE. Both arrows populated in a healthy relationship.
- **Shared sources of truth.** SLO document, pipeline version log, baseline reports, SLO review minutes, joint incident post-mortems.
- **Drift signals.** SRE editing judge prompts; peer tuning alert rules — both solvable with the two-column contract as the conversation artifact.

This closes the module. You now have the latency SLI surface (chapter 01), the quality SLI surface (chapter 02), the streaming availability triad (chapter 03), the cost SLI surface (chapter 04), the operable error-budget policy (chapter 05), the SLO lifecycle wired to the Prometheus / Grafana / OTel stack (chapter 06), and the explicit peer boundary to the eval engineer (this chapter). The exercises walk the full stack against a workload you pick in exercise 01 and carry through.
