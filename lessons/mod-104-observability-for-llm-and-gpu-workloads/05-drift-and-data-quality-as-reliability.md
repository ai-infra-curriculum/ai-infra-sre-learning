# Drift and Data Quality as Reliability Signals

## Why this chapter exists

A classical SRE page fires on a visible symptom — a 500 rate, a latency spike, a queue depth over threshold. An LLM-fleet's failure surface is broader. A model that silently starts returning worse answers on the same traffic is a reliability incident that will not trip any of the chapter 02 PromQL recipes. Users feel it as "the product got dumber"; the on-call never got paged.

The signal class that catches these quiet failures is **drift**: the slow change over time in the *inputs* the model sees (prompt distribution, retrieved context distribution, user demographics), the *outputs* the model produces (completion length, format conformance, judge pass rate, embedding distribution), and the *measurement apparatus* itself (the judge model's agreement with human raters). When drift crosses a threshold that materially affects the user, it is a reliability event. It belongs in the on-call's queue the same way a latency SLO burn belongs there, routed to the same incident commander.

This chapter is the authoring reference for how to instrument drift so it reads as reliability signal — a Prometheus-backed, Alertmanager-routed signal, not a Monday-morning ML-team chart. It covers the three drift classes (input, output, judge), the open-source tooling (Evidently, Arize, WhyLabs), and the wiring that turns a slow-moving statistical signal into a workable SRE alert without creating a storm of noise.

The eval-peer ownership boundary is strict here — judge quality and drift methodology are craft inside the signal, which is the peer's work (see `mod-103` chapter 07). Your work is the *reliability contract on the drift signal*: how it enters the SLO stack, how it pages, how it rolls back a release, how it co-investigates when the signal itself may be drifting.

Primary references: [Evidently documentation](https://docs.evidentlyai.com/), [Arize documentation](https://docs.arize.com/), [WhyLabs documentation](https://docs.whylabs.ai/), and the classical ML survey literature — Lu et al., *["Learning under Concept Drift: A Review"](https://arxiv.org/abs/2004.05785)*, which is the general statistical framing.

## 1. The three drift classes

A production LLM surface experiences drift on three distinct axes. Each produces different symptoms, each requires different detection, and each routes to a different incident playbook.

### Drift A — input distribution shift

The *prompts the model is being asked* change. More users ask about a new topic (product launch); the mean prompt length grows (users learn to be verbose); one tenant's share of traffic changes; a new UI surface starts routing to the model with a different prompt template.

Why it reads as reliability — the model's quality is only as good as its training distribution and its tuning distribution. If the production distribution drifts away from what was baselined, quality regresses invisibly. Latency metrics may also drift (long prompts cost more TTFT) — but latency catches a narrow slice of the harm.

Detection signals:

- Mean prompt length and length distribution (percentiles).
- Prompt-embedding distribution — `L2` or cosine distance from a baseline centroid, Population Stability Index (PSI), KL divergence over a sliding window.
- Prompt-category distribution if you have a topic classifier running.
- Language / locale distribution, if your product supports multilingual.
- Tool-call request distribution — which tools, which arguments.

### Drift B — output distribution shift

The *completions the model produces* change on the same inputs. Output length drifts up (model gets more verbose), refusal rate climbs, format-conformance rate falls, specific phrase-use shifts, embedding distribution of completions shifts.

Why it reads as reliability — a model's behavior can drift even with no provider-side rollout (sampling non-determinism); with a rollout (quiet version bump on a vendor-hosted model), it can drift hard. Output drift is also the signal that a judge model is *agreeing less with humans* — the judge sees the same completions but its verdict shifts.

Detection signals:

- Mean completion length and percentile distribution.
- Refusal rate (`gen_ai.response.finish_reasons = "content_filter"` from chapter 01).
- Length-finish rate (`"length"` finish reason — hit the max-tokens cap).
- Judge pass rate per dimension (hallucination, toxicity, structured-output conformance, tool-call success).
- Completion-embedding distribution — same statistics as prompt-embedding above.
- User-feedback rate (thumbs-down), where available.

### Drift C — judge / measurement drift

The *measurement apparatus* changes. A judge LLM that previously classified 94% of a calibration set correctly now classifies 86% correctly. An embedding model that the vector store uses was quietly updated by the provider. A rule-based classifier's regex falls out of sync with the current response format.

Why it reads as reliability — if the judge drifts, every quality SLI the judge feeds burns for the wrong reason. The model is not getting worse; the measurement is. A rollback decision triggered by a drifted judge is a rollback for no reason.

Detection signals:

- Judge-vs-human agreement on a stable calibration set, measured out-of-band on a cadence (weekly / monthly).
- Judge-pass-rate on a stable "golden" set — the number should be flat; any movement is a judge-side signal, not a model-side one.
- Embedding-model-version signal — ideally a fingerprint hash across an unchanging text sample, compared to baseline.

Judge drift is slow (months, usually), but when it crosses a threshold it has already been corrupting downstream signals for a while. The alerting cadence here is weekly / monthly review rather than minute-by-minute, but it is still on-call-adjacent because when it fires, pending rollback decisions have to be re-evaluated.

## 2. The Evidently / Arize / WhyLabs tool surface

Three open-source or open-core tools dominate drift-detection in production ML. Each has a different design center; the one your team should pick is determined by workflow more than feature matrix.

### Evidently ([docs.evidentlyai.com](https://docs.evidentlyai.com/), [GitHub](https://github.com/evidentlyai/evidently))

- **Shape.** Python library + a server UI (Evidently Cloud and self-hosted Evidently OSS platform).
- **Design center.** Statistical test-suite catalog — PSI, KS, Chi-squared, Wasserstein, Jensen-Shannon — plus LLM-specific metrics (semantic similarity, format checks, custom LLM-judge evaluations). You compose reports from the catalog.
- **When it fits.** Teams comfortable with a batch, Python-first workflow — scheduled reports run against a snapshot dataset, upload results to a UI, trigger alerts. Fits well with Airflow / Dagster / dbt workflows.
- **Operator posture for the SRE.** Fully open-source under Apache 2.0; self-host is straightforward. The metrics emit to Prometheus via custom exporters you write around the report outputs; this is a lightweight integration but you own the glue.

### Arize AX / Arize Phoenix ([docs.arize.com](https://docs.arize.com/))

- **Shape.** Arize AX is the commercial platform; Phoenix (introduced in chapter 04) is the open-core companion that overlaps the drift surface.
- **Design center.** Continuous, embedding-centric drift and performance monitoring. Strong UMAP / embedding-drift views; strong on live-traffic streaming ingest.
- **When it fits.** Teams that already run Phoenix for LLM traces and want to extend into drift without adopting a second platform. Also teams that prize the embedding-space exploration UI.
- **Operator posture for the SRE.** Phoenix open-source handles ingest and views; the commercial product owns the continuous-monitoring / alerting side. Metrics exposed via API; Prometheus bridge is possible but platform-first alerting is the vendor-blessed path.

### WhyLabs ([docs.whylabs.ai](https://docs.whylabs.ai/), [whylogs](https://github.com/whylabs/whylogs))

- **Shape.** `whylogs` is an open-source profiling library; WhyLabs AI Observatory is the commercial monitoring platform the profiles feed into. (The commercial platform was, per vendor announcements, discontinued as a standalone offering in 2025; verify the current status before adopting.)
- **Design center.** Statistical *profiling* — summarizing a dataset into a compact profile (sketches, histograms) that is cheap to ship and cheap to compare over time. Strong on batch and streaming both.
- **When it fits.** Teams that need profiling as a *library* embedded in pipelines (so profiles are an artifact in your data pipeline), with monitoring potentially elsewhere. The library stands on its own regardless of platform state.
- **Operator posture for the SRE.** The library is Apache 2.0. The platform-side story is uncertain as of writing; use the library and route the profiles to your own Prometheus / Grafana stack or to a different monitoring platform.

<!-- needs-research: WhyLabs' commercial platform status and roadmap has shifted recently (announcements of discontinuation in 2025 have circulated); verify the current state of the commercial product before committing to it as the monitoring backend. -->

### Other tools worth knowing

- **Alibi Detect** — open-source library of drift detectors (CVM, KS, MMD, LSDD, learned kernel) — more statistically rich than Evidently's catalog on specific detectors. Worth reading in combination.
- **NannyML** — focuses on *performance estimation* under drift (what would the model's accuracy be, given the drift, without waiting for labels).
- **River** — online-learning library; also has streaming drift-detection primitives.
- **Model-provider native drift tools** — Vertex AI Model Monitoring, SageMaker Model Monitor, Azure ML data-drift monitoring. If you are already on a cloud ML platform, the native tool may be the cheapest adoption path.

### Picking one

The trade-offs mirror chapter 04's platform decision. Three practical pulls:

- **Pure OSS self-host is a hard constraint** — Evidently is the easiest; whylogs library can stand alone.
- **Already running Phoenix for LLM traces** — extend Phoenix's drift surface before adopting a second tool.
- **Need continuous (minute-by-minute) drift detection on live traffic** — commercial tier of one of the three (Evidently Cloud, Arize AX, or a provider-native tool) is honest.

For the exercises in this module, prefer Evidently — it has the broadest open-source story and the cleanest "export to Prometheus" integration.

## 3. The metrics — what to compute and emit

For the AI SRE, the drift signal is whatever ends up as a time-series the Prometheus stack can query. That means every drift detector you run should produce *one or more* scalar metrics per detection run, exported to Prometheus with a bounded label set.

### Metric A — distribution-distance scalar per detector

For every detector you run (prompt-length, prompt-embedding, completion-length, completion-embedding, refusal rate, per-category distribution), emit:

```
drift_distance{detector, surface, model, baseline_window, comparison_window}
```

The scalar is the statistical test's output — the KS statistic, the PSI, the Jensen-Shannon divergence, the Wasserstein distance. Each detector has a natural scale; document it in the detector's definition.

Also emit:

```
drift_pvalue{detector, surface, model, ...}       # for detectors that produce one
drift_threshold{detector, ...}                    # the threshold the alert uses
```

The threshold is a configuration-driven metric so Grafana can draw the "alert when above this" line next to the signal.

### Metric B — detector run metadata

```
drift_detector_runs_total{detector, outcome}      # counter: outcome = drift | no_drift | error
drift_detector_latency_seconds{detector, quantile}  # histogram
drift_detector_last_success_timestamp{detector}   # gauge
```

These are the operator metrics for the drift pipeline itself — how often it runs, whether runs succeed, how fresh the signal is. You will need them when the pipeline falls behind and the drift signal silently goes stale.

### Metric C — judge-vs-human agreement (judge-drift signal)

```
judge_agreement_rate{judge_version, calibration_set, measurement_batch}
```

Measured cadence (weekly / monthly). The judge-drift SLO sits on this; chapter 05 of `mod-103` names the policy around it.

### Metric D — baseline refresh

Baselines are not static. They refresh on a cadence — monthly for fast-moving products, quarterly for stable ones. Emit:

```
drift_baseline_age_days{detector, surface}
drift_baseline_refresh_scheduled_timestamp{detector, surface}
```

A baseline that is 400 days old is probably no longer the right reference; the alert on `drift_baseline_age_days > 90` catches the "our baseline is stale" case before the signal becomes meaningless.

### What NOT to emit as a metric

- **Per-request drift score.** That is a trace-level attribute, not a metric; emit it to the OTel span from chapter 01, not to Prometheus. Cardinality explosion risk.
- **Full per-token embedding vectors.** The platform keeps these; your metric store holds only scalars.
- **Raw judge verdict counts at per-request granularity.** Counter aggregated at the detector's window is sufficient.

## 4. The alert set — paging the SRE on drift

Drift alerts are a specific authoring problem because:

- Statistical significance (small p-value) is not the same as operational significance (the user is harmed).
- Drift detectors are noisy on short windows and insensitive on long ones.
- Drift is usually *gradual*, so the right alert shape is "sustained drift," not a single-sample excursion.
- A drift alert that pages for every statistical blip is noise; one that fires only when the signal has already flipped quality metrics is late.

The following alert set is the defendable minimum. Each alert names an on-call routing.

### Alert 1 — material input drift, sustained

Fires when the input-distribution distance exceeds a surface-specific threshold for a sustained window (e.g., 24 hours) *and* the baseline is still fresh.

```promql
avg_over_time(drift_distance{detector="prompt_embedding_js", surface="chat"}[24h])
  > drift_threshold{detector="prompt_embedding_js", surface="chat"}
and
drift_baseline_age_days{detector="prompt_embedding_js", surface="chat"} < 90
```

Routes to: AI SRE on-call. Playbook: investigate traffic source (product launch? abuse campaign? broken client that sends malformed prompts?); co-review with product / ML; decide whether the drift is a traffic event that invalidates the current baseline or an incident that warrants release-freeze.

### Alert 2 — material output drift, sustained, without input drift

The same shape against `detector="completion_embedding_js"`, with an *inverse* condition on the input drift — fires when output has drifted but input has not.

Routes to: AI SRE on-call. Playbook: this is the classic "provider silently rolled a model version" signal, or a prompt-template change that was not coordinated with the eval peer. First investigate the model-version signal (chapter 06 cross-ref); then the retrieval signal (what if the retrieved context distribution shifted?); then escalate to eval peer if judge-drift is suspect.

### Alert 3 — refusal rate spike

Fires when the fraction of requests with `gen_ai.response.finish_reasons = "content_filter"` moves out of band.

```promql
(
  sum(rate(gen_ai_responses_total{finish_reason="content_filter"}[1h]))
  /
  sum(rate(gen_ai_responses_total[1h]))
)
>
quantile(0.95, refusal_rate_7d_baseline)
```

Routes to: AI SRE on-call. Could be a traffic-abuse event, a provider-side safety-filter tune, or a prompt-template change. The three branch quickly in the playbook; the alert exists to get a human looking.

### Alert 4 — judge-vs-human agreement slip

Fires on the weekly / monthly cadence when `judge_agreement_rate` drops below a threshold.

```promql
judge_agreement_rate{judge_version="v3", calibration_set="2024q4"} < 0.88
```

Routes to: eval peer *primary*, AI SRE *co-receiver*. The eval peer owns the fix; the AI SRE needs to know because every quality SLI's trust level just changed. Suspend auto-rollbacks that key on this judge until the peer resolves.

### Alert 5 — drift pipeline stale

Operator alert — the drift pipeline itself hasn't produced fresh data.

```promql
time() - drift_detector_last_success_timestamp > 3600 * 2   # 2 hours stale
```

Routes to: AI SRE on-call, ticket not page. Fix the pipeline; the drift signal is not trustworthy until it recovers.

### Alert 6 — baseline stale

```promql
drift_baseline_age_days > 90
```

Routes to: AI SRE on-call, ticket. Schedule a baseline refresh with the eval peer and the product team.

### Threshold-setting discipline

The tempting mistake is to pick a threshold from the Evidently default and ship. Instead:

1. **Baseline first.** Measure the detector on production for 2–4 weeks before setting a threshold. The threshold lives in the right tail of the measured baseline distribution; p99 or p99.9 of the baseline is a defensible starting point.
2. **Review in the SLO ritual.** The drift thresholds belong on the SLO review agenda from `mod-103` chapter 06 section 6. They move as the product moves.
3. **Tune on false-positive rate.** If the first month of alerts is more noise than signal, raise the threshold or lengthen the sustained window. If an incident went un-paged, lower it and shorten.

## 5. Composing with the SLO stack

Drift alerts compose with the SLO stack from `mod-103` in a specific way: they are *leading indicators* on the quality SLI burn. The composition in your runbook:

- **Drift alert fires first (hours in).** The on-call investigates traffic and configuration; often contains the issue before the SLI burns.
- **Quality SLI starts burning (hours later).** The on-call already has context from the drift alert; the SLI burn is a confirming signal.
- **Burn-rate alert pages on the SLI (depending on rate).** Response is coordinated with what the drift investigation found.

If drift and SLI burn fire together, the drift alert's traffic-side investigation often yields the root cause the SLI burn alone would take hours to find.

If drift fires and the SLI never burns, the drift was a traffic event that the model handled well — no incident, but the baseline probably needs a refresh.

If the SLI burns and no drift fires, the cause is probably hardware-side, engine-side, or retrieval-side — route to chapter 03 (DCGM) and chapter 02 (engine state).

### Error-budget interaction

Drift events can *invalidate* a stretch of SLI measurement. If your input distribution changed materially — a product launch that tripled the long-prompt share — the SLI was measuring a new workload against an old baseline. The error-budget policy from `mod-103` chapter 05 handles this through the retroactive-adjustment mechanism: document the drift event, flag the affected measurement window, re-baseline, and either retroactively credit the budget or hold the SLO constant depending on policy.

The important point is that drift events are *documented* in the error-budget log, not silently ignored. The next SLO review pulls them in.

## 6. The judge-drift special case

Judge drift deserves its own treatment because the detection methodology is subtly different and because the incident class routes differently.

### Why judge drift is hard

The judge is the measurement apparatus for the quality SLI. If the judge drifts, the quality SLI's burn / no-burn status is untrustworthy — but you don't know the judge has drifted *because* the SLI's burn status is your signal. The chicken-and-egg problem requires an out-of-band signal.

### The out-of-band signal — judge-vs-human agreement

The eval peer maintains a calibration set — a few hundred to a few thousand responses that have been manually labelled by human raters. The judge is run against this calibration set on a cadence; the agreement rate with the human labels is the judge-drift signal.

When the agreement drops below threshold:

1. Pause rollback automation keyed on this judge.
2. Eval peer investigates: judge prompt revision, underlying model version, calibration-set sample bias.
3. AI SRE reviews the quality-SLI history over the drift window; if the SLI burned during the drift and the judge is the cause, retroactive adjustment per `mod-103` chapter 05 section 7.
4. Judge is recalibrated or replaced; baseline and SLO definition update per `mod-103` chapter 06.

### The silent-judge-swap case

Some providers rev their underlying models silently. A judge that was running on `gpt-4-0613` is now running on `gpt-4-0125` because the provider deprecated the pinned version. The agreement drops; you don't know why until you check the vendor's deprecation notes.

Instrumentation defense: pin judge model versions explicitly (do not use latest aliases), and emit `judge_model_version` as a label on every judge-verdict metric. Any change in the label value is a tracked event, even if the eval peer thinks they didn't change anything.

## 7. One worked example — the agentic-chat-refusal-rate incident

Illustrative shape of a real-feeling drift incident:

- **Day 1 11:00.** Evidently report, running as a daily job, flags that the refusal rate on the agentic-chat surface has shifted from 1.8% ± 0.3% (14-day baseline) to 3.1% in the last 24 hours.
- **11:05.** Alert fires to AI SRE on-call (Alert 3). Playbook routing: refusal-rate incident.
- **11:10.** On-call opens the trace platform (chapter 04), filters for traces with `content_filter` finish reason over the drift window, reads the first 20. The rejected prompts share a common pattern: queries about a specific public-figure event that was in the news that morning.
- **11:20.** Hypothesis: traffic shift toward politically-sensitive queries combined with the provider's recent safety-filter tune. On-call checks the provider's model-version metric — no version change. Checks the safety-filter version if exposed — not available on this provider.
- **11:35.** On-call pings product team: is there a traffic campaign or a news-cycle effect? Product confirms the news event is causing a burst of related queries. Not a defect; a traffic event.
- **12:00.** On-call documents the drift event, holds the SLO, flags the window for retroactive-adjustment review; refusal-rate alert threshold temporarily raised with an auto-expiry at 24 hours.
- **Day 3.** News cycle passes; refusal rate returns to baseline. Threshold auto-restores. SLO review agenda picks up the event for documentation.

No model change, no engineering incident, but a signal that the on-call got and triaged in under an hour. Without the drift alert, the refusal rate would have been visible only on a weekly dashboard review; the product-team conversation would have happened days later.

## Summary

- **Three drift classes.** Input distribution, output distribution, judge / measurement drift. Each has distinct symptoms, distinct detection, distinct routing.
- **Tool surface.** Evidently (strongest OSS batch workflow), Arize / Phoenix (continuous + embedding-centric), WhyLabs library (`whylogs`) — commercial platform uncertain. Also Alibi Detect, NannyML, provider-native tools.
- **Metric surface.** Distance scalar per detector, detector-run metadata, judge-agreement rate, baseline-age gauges. Never per-request as a metric; that goes on the span.
- **Six alerts.** Sustained input drift, sustained output drift, refusal-rate spike, judge-agreement slip, pipeline stale, baseline stale. Thresholds are baselined, not defaulted.
- **Composition with the SLO stack.** Drift alerts lead SLI burns by hours; together they pinpoint traffic-side vs. model-side causes. Document drift events in the error-budget log for retroactive adjustment.
- **Judge-drift is a special case.** Out-of-band judge-vs-human agreement is the only honest signal; pause rollback automation when it slips; pin judge model versions explicitly.
- **Owner split.** Eval peer owns the judge and the methodology; the AI SRE owns the reliability contract on the signal — how it pages, how it rolls back, how it enters the SLO stack.

The next chapter drops back to the log layer — structured logging discipline, PII redaction at high QPS, trace-ID correlation, cost-per-log — so the Loki / Cloud Logging surface composes with the metric and trace stacks rather than fighting them.
