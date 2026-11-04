# Judge-Model Drift and Provider-Degraded Mode

## Why this chapter exists

The two failure classes this chapter treats — judge-model drift (class 2.8) and provider-degraded mode (class 2.9) — share a property that distinguishes them from the rest of the taxonomy and makes them hard: the signal the on-call is looking at is not fully trustworthy. For 2.8, the *measurement* is wrong — the judge model that scores production outputs has changed, and the quality SLI it drives is now biased. For 2.9, the *provider* is degrading — the hosted model behind an API is slower, or dumber, or both, but the provider has not changed any contract the SRE can read.

Both cases strain the triage matrix because the usual pattern — "signal fires, classify, route to owner" — depends on the signal being a reliable reflection of reality. When the signal is suspect, the on-call must either (a) corroborate the signal from an independent source, or (b) hold position without over-correcting. Over-correcting on a suspect signal is how reliability teams make incidents worse — rolling a model that was fine because the judge recalibrated, or routing around a provider that was fine because the drift-pipeline drifted.

This chapter authors the specific techniques for both classes: for 2.8, the discipline of judge calibration and the when-to-believe-the-signal checklist; for 2.9, the slicing-by-`gen_ai.system`-and-`gen_ai.response.model` discipline and the mitigation posture when the upstream is a black box.

## 1. Judge-model drift (class 2.8)

### 1.1 What a judge is, briefly

The eval peer (per `mod-103` chapter 07) builds and operates the **judge pipeline**: for each production response (or a sampled subset), a scoring process emits a verdict — pass / fail / unknown on one or more dimensions. The dimensions typically include:

- Factuality / groundedness.
- Refusal rate / helpfulness.
- Safety-category classifications.
- Task-specific rubrics (did the model answer the question, did it follow the format, did it succeed at the task).

The scoring process is often itself an LLM — a "judge model" with a prompt that asks it to score the response under test. Alternative scoring processes include regex / rule-based classifiers, embedding-similarity to known-good references, and human-in-the-loop sampling. Most production pipelines mix.

The judge is the primary producer of the quality SLIs the AI SRE operates. The quality SLI's reliability is bounded by the judge's reliability.

### 1.2 What "judge drift" means

Judge drift is a change in the judge's behaviour over time such that the verdicts it emits are no longer calibrated against the same ground truth. The visible symptom is that the quality SLI moves (up or down) with no corresponding change in the model under test or in the input distribution. Possible root causes:

- **Judge model version change.** The judge is an LLM (say, a specific GPT-4o snapshot) and the provider moved the alias. The judge's scoring behaviour shifted.
- **Judge prompt change.** The eval peer edited the judge's scoring prompt — a legitimate iteration but one that invalidates comparisons against pre-change baselines.
- **Judge rubric change.** The rubric definition shifted (what counts as "grounded"; what counts as "refusal"; what the safety-category thresholds are).
- **Judge-input distribution shift.** The judge is being asked to score a different mix of outputs than it was calibrated against; its reliability on out-of-distribution inputs is worse.
- **Judge-vs-human agreement decay.** The judge has drifted silently; without periodic human-annotation calibration, the drift is not visible until it is large.

### 1.3 The fundamental sign — judge-vs-human agreement

The gold standard for judge calibration is agreement against human annotation. Periodically — typically weekly or monthly — a sampled set of production outputs is scored by both the judge and by qualified human annotators. The agreement rate (Cohen's κ or a product-specific statistic) is tracked over time.

A drop in judge-vs-human agreement is the leading signal of judge drift. `mod-104` chapter 05 wired this into drift pipelines as alert 4 ("judge agreement slip"). The chapter 04 / 05 in this module reference it as the signal the on-call must cross-check whenever the quality SLI moves and no other cause is visible.

### 1.4 Why the AI SRE must recognise this class even though the eval peer owns the fix

The SRE's *mitigation* for 2.8 is to *not roll the model*. That is a non-trivial action: the quality SLI is burning; the natural operational instinct is to roll back whatever changed recently. If the on-call rolls the model when the actual cause is judge drift, they have made no reliability improvement, have paid the cost of a rollback, and have burned engineering time on a wild-goose chase. Recognising the class is the mitigation.

The eval peer owns the *fix* — recalibrate, replace, pin, roll back the judge change. The SRE owns the *hold* — don't react to a suspect signal; don't let the SLO burn-rate alert push the team into action when the action would be wrong; declare the signal suspect, communicate clearly, and let the eval peer's forensic re-run resolve.

### 1.5 The when-to-believe-the-signal checklist

When a quality SLI burns, before taking any mitigating action on the model side, the on-call should walk a short checklist:

1. **Check judge-vs-human agreement, most recent window.** If agreement dropped recently, the signal may be judge drift. Flag it.
2. **Check judge version pin.** Is the judge's version the same today as yesterday? If not, this is strong evidence of 2.8.
3. **Check judge prompt / rubric version.** Was the judge prompt or rubric edited in the window? The eval peer's change log is the source.
4. **Spot-check 10–20 flagged outputs manually.** The on-call (or the eval peer on page) looks at 10–20 production outputs the judge marked as failing. If the outputs look fine to a human eye, the judge has drifted or the rubric has drifted. If the outputs really are failing, the model or the application is at fault.
5. **Compare the judge's drift pattern against known regressions.** If the judge is suddenly failing all outputs on a specific dimension, the rubric for that dimension may have shifted. If the judge is suddenly failing outputs randomly, the judge model may have drifted.

The checklist is a 10-minute addition to the hallucination-spike / refusal-spike / toxic-blast first-15-minutes checklists from chapter 04. It is cheap; skipping it is expensive.

### 1.6 Diagnostic patterns

Three patterns the on-call will recognise once they have seen them a few times.

**Pattern A — The judge changed, the model did not.** Quality SLI shifts; no deploy on the application / prompt / model side; `gen_ai.response.model` label unchanged; judge-vs-human agreement is lower than the previous calibration window; judge version shifted. Diagnosis: 2.8. Mitigation: pin judge; eval peer re-calibrates; retro-adjust SLI over the affected window per `mod-103` chapter 05 section 7.

**Pattern B — Everything is changing at once.** Quality SLI shifts; model deployed recently; judge pipeline was updated this week; input traffic mix shifted (new campaign, new tenant). Nothing can be isolated. Diagnosis: ambiguous. Mitigation: hold; the eval peer's forensic re-run with a judge pinned to the previous version isolates the model-side cause from the judge-side cause. The SRE's job is to not act on ambiguous signal.

**Pattern C — The judge is *more* permissive than yesterday.** Quality SLI *improves* with no explanation. This is almost always judge drift, not a model improvement. Mitigation: same as pattern A but with less urgency — the business is not visibly hurt, but the signal is uncalibrated and future burns will be based on a wrong baseline.

### 1.7 Judge drift and the retrospective

A retrospective on an incident later found to be judge-drift-driven should include:

- A clear statement that the quality SLI was uncalibrated during the window.
- The corrected pass-rate numbers after the judge is recalibrated.
- SLO retro-adjustment per the error-budget policy from `mod-103` chapter 05.
- An update to the judge-calibration cadence if the drift was large.
- A note in the eval peer's change log and the SRE's dashboard documentation about the affected window.

Judge drift is a *measurement incident*, not a service incident. It belongs in the retrospective record for two reasons: (a) auditors need to know that quality numbers in the affected window are suspect; (b) the team's muscle memory for recognising the pattern improves with every case.

## 2. Provider-degraded mode (class 2.9)

### 2.1 What provider degradation looks like

A provider-hosted LLM is a black box with a contract — an API, a model name, a pricing surface, sometimes an SLA. The provider degrades silently when it does not change any contract but the service nonetheless behaves worse than it did yesterday. Three sub-patterns:

- **Latency regression.** TTFT and/or TPOT regress on the same `gen_ai.system=<provider>` and `gen_ai.response.model=<id>` slice. No capacity signal from your side; no traffic shape change. The provider is slower.
- **Quality regression.** Judge-pass rate drops on the same slice; input mix unchanged. The provider's model, under the same name, is producing worse outputs. Often due to an undisclosed snapshot change.
- **Both together.** Slower *and* worse; often the hallmark of a provider rolling a more aggressive quantization, batching, or routing change behind the model name.

In all three cases, the provider's status page is clean or lagging. The provider's support channel either does not confirm, confirms after the fact ("we had a backend rebalance yesterday"), or confirms but under NDA.

### 2.2 Why it's hard

Three characteristics make 2.9 operationally hard.

- **The upstream is a black box.** The provider does not expose the model version it is serving, the quantization, the batching, the backend hardware, the routing. Your `gen_ai.response.model` label captures whatever the provider chooses to tell you.
- **The status page is lagging.** Providers typically declare incidents after sustained user impact. A "degraded" state that is not a clean outage may not appear on the status page at all.
- **Attribution to provider vs. network vs. gateway is costly.** A TTFT regression against a hosted provider could be the provider, your network path, your gateway, or a region issue. Sorting is non-trivial.

### 2.3 Signals and slicing

The core diagnostic discipline is **slicing by `gen_ai.system` and `gen_ai.response.model`**, both captured on the OTel spans per `mod-104` chapter 01.

Useful dashboard panels:

- TTFT p95 per `gen_ai.system` × `gen_ai.response.model` slice.
- TPOT p95 per same slice.
- Quality SLI (per quality dimension) per same slice.
- `gen_ai.response.model` label value over time — is the label itself changing?
- Provider-reported error codes per same slice (429, 500, 503, provider-specific codes).
- Cost per request per same slice.

A provider-degraded signal is a shift in the TTFT / TPOT / quality panel on the provider slice that is *not* present on other slices (your self-hosted models, other providers) and *not* explainable by your own stack's changes.

### 2.4 Triangulation signals

When the provider slice is suspect, the on-call triangulates:

- **Does the regression affect only one provider?** If yes, probably the provider; if multiple providers regress together, probably your side (network, gateway, prompt-template changes).
- **Does the regression affect only one region?** Many providers run multiple backend regions; a region-specific issue is suggestive of a provider backend issue.
- **Does the provider's model-name label change?** A `gen_ai.response.model` shift, even if the provider's alias did not visibly change, is strong evidence of a provider-side snapshot move.
- **Are other users of the same provider seeing the same?** Public signals: provider's community forums, Twitter / X, Reddit, LLM-observability vendors' industry reports.
- **Can you reproduce against a reference input?** A canary set (chapter 03) hitting the hosted endpoint will reproduce the regression.

Section 2.6 treats the mitigation posture; the triangulation above feeds it.

### 2.5 Interactions with other classes

- **2.9 + 2.1.** When a provider snapshot moves silently, the `gen_ai.response.model` label moves, which is also a silent-weight-change signal. The incident is both 2.1 (silent weight change) and 2.9 (provider is now serving a different thing without warning). The classification is composite; the routing (vendor-risk) is common.
- **2.9 + 2.2 / 2.3 / 2.4.** A provider snapshot move often moves hallucination / toxic / refusal rates together. Treat the composite accordingly.
- **2.9 + 2.5.** Providers that offer prompt caching sometimes silently invalidate caches during backend changes. A cache-hit-rate drop concurrent with a provider-slice latency shift is a composite signal.

### 2.6 Mitigation and route — the vendor-risk workflow

The SRE's short-term levers:

- **Route around.** If you have a multi-provider setup via a gateway (LiteLLM, Portkey, your own) and an alternative provider / model is a viable fallback, shift traffic. Fallbacks have their own quality baselines; validate against your eval set before shifting.
- **Pin the provider's version** if the provider offers version pinning. Most major providers offer dated snapshot names (e.g., `gpt-4o-2024-08-06`) alongside the floating alias (`gpt-4o`). Switch your `gen_ai.request.model` from the floating alias to the dated snapshot. This prevents future silent moves on the same surface.
- **Scale a self-hosted fallback.** If you have a self-hosted alternative in reserve (even a smaller model), scaling it and shifting traffic is the ultimate black-box mitigation.
- **Throttle the surface** if no fallback is available and quality / cost impact is bounded.
- **Open a vendor support ticket.** With the evidence envelope (slice, timing, diff). Follow the vendor-risk workflow from `mod-107`.

The SRE's mitigation ends when user impact is bounded. The *remediation* — provider commits to a fix; SLA credits are negotiated; contract clauses are activated — belongs to vendor-risk and leadership (`mod-107`, `mod-114`).

### 2.7 Comms posture

Provider-degraded incidents have a specific comms posture:

- **Internal:** incident channel, standard severity per burn size, note that the upstream is suspect.
- **External users:** if user-visible, acknowledge in product (status page, in-product banner) with careful phrasing: "we are seeing elevated latency with one of our model providers; working on a mitigation." Do *not* name the provider publicly without legal / vendor-contract review.
- **To the vendor:** evidence envelope with the full technical story. Providers are more responsive when your evidence is better than their internal signal.
- **To leadership / sponsors:** framed as vendor-risk; the SLA-credit, contract-renegotiation, or multi-provider-strategy decision is at this level.

### 2.8 Prevention — architecture choices that reduce 2.9 severity

The AI SRE's architectural voice in `mod-107` on vendor-risk and `mod-113` on multi-region DR / BCP is to argue for architectures that reduce 2.9 blast radius:

- **Multi-provider gateway.** Requests route via a layer that can shift traffic without application-side changes.
- **Version-pinned requests by default.** Floating aliases are an operational hazard; pin unless there is a specific reason not to.
- **Self-hosted fallback for critical surfaces.** A smaller self-hosted model that can absorb traffic for the most critical surfaces when the primary provider is unavailable.
- **Regular dry runs.** Periodically exercise the fallback path (chaos engineering, `mod-109`) so it is known to work when needed.

The SRE's job is not to build these (the application / platform teams do); the SRE's job is to argue for them with incident-based evidence, as `mod-114` cross-functional-reliability-leadership develops.

## 3. The pattern that unifies 2.8 and 2.9 — degraded signal + degraded upstream

Both classes share an operational pattern: **the on-call cannot fully trust what they see**. For 2.8, the measurement is suspect; for 2.9, the upstream is a black box. In both, the correct posture is:

- Do not over-react to a single signal.
- Triangulate from multiple independent signals before taking action that is costly or hard to reverse.
- Mitigate the user-visible impact first; resolve the attribution question second.
- Route to the owner with the forensic context to resolve attribution — the eval peer for 2.8, vendor-risk for 2.9.

An on-call who has internalised this pattern responds well to any failure class where the signal is not fully reliable. That includes future failure classes the module has not yet named — multi-modal failures (vision / audio / video outputs with their own quality signals), long-horizon-agent failures, model-of-model-composition failures. The discipline generalises; the specifics will come from quarterly taxonomy reviews.

## 4. The role of calibration cadence

Both classes are managed, long-run, by calibration cadence:

- **Judge calibration.** Weekly or monthly human-annotated calibration run against the judge; agreement is tracked; drift is corrected proactively instead of discovered in an incident.
- **Provider calibration.** Monthly canary-set runs against each provider; baseline quality / latency / cost is captured; a shift is detected before user impact.

The cadence is operational hygiene; the chaos / release-engineering / vendor-risk modules (`mod-109`, `mod-111`, `mod-107`) treat specific calibration programs. This chapter notes that the cadence prevents both classes from becoming incidents.

## 5. The retrospective-template fields

For a 2.8 or 2.9 incident, the retrospective template should carry fields specific to signal-trust:

- **Signal trustworthiness.** Was the primary alerting signal reliable during the window? (For 2.8: no — judge drifted. For 2.9: partial — our signal was reliable; the upstream contract was not.)
- **Independent corroboration.** What independent signals confirmed or refuted the primary? (Human annotation; multi-provider comparison; canary re-run.)
- **Hold decision.** Was there a moment when the on-call chose to hold instead of act? Why; what was the time-to-resolution impact.
- **Signal improvement action item.** What is the plan to make the signal more trustworthy? (Tighter judge calibration; multi-provider gateway; version pinning policy.)

These fields are specific to this chapter's classes; they are not useful for all incidents. `mod-108` on incident response treats the retrospective template at depth; this chapter contributes the specific fields.

## Summary

- **Judge-model drift (2.8)** is a *measurement* failure. The quality SLI moves because the judge changed, not because the model did. Recognition signals: judge-vs-human agreement drop, judge version shift, judge prompt / rubric edit, spot-check of flagged outputs.
- **Provider-degraded mode (2.9)** is a *black-box upstream* failure. The hosted provider is slower / dumber / both without a visible contract change. Recognition signals: slice-by-provider-and-response-model TTFT / TPOT / quality shifts; `gen_ai.response.model` label changes; triangulation against self-hosted and other providers.
- **In both, signal is suspect.** The SRE's mitigation is to not over-react; triangulate; mitigate user-visible impact; route to the peer with the forensic context.
- **Judge-drift mitigation: hold.** The SRE's job is to recognise the class, pin judges where possible, and let the eval peer recalibrate. SLO retro-adjustment follows.
- **Provider-degraded mitigation: route around, pin versions, scale fallback.** Short-term SRE levers; long-term remediation is vendor-risk (`mod-107`).
- **Calibration cadence prevents both.** Judge human-annotation cadence; provider canary cadence.
- **Retrospective template carries signal-trust-specific fields.** Signal trustworthiness; independent corroboration; hold decision; signal-improvement action item.

The next chapter brings the module together — a triage decision matrix the on-call follows at 3 AM, and ten mock-incident walkthroughs across the ten classes, each with trace / metric / log artifacts.
