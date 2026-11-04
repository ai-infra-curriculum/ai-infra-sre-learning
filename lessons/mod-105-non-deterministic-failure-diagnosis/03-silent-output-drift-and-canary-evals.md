# Silent Output Drift — Weights, Kernel, Numeric, and the Canary-Eval Detection Discipline

## Why this chapter exists

Of the ten failure classes in chapter 01, **silent output drift** (class 2.1) is both the most common and the hardest to detect, because — by definition — nothing is broken. No 500s. No 429s. No crashes. No stack traces. The service happily serves 200s all day. The output distribution has merely shifted in a way that, given enough traffic, moves one or more of the quality SLIs from `mod-103` chapter 02 across a threshold the business cares about. By the time the quality SLI burns the budget, the drift has already been live for hours or days.

The detection discipline for this class is therefore different from the rest of the module. For most AI-specific failures, the detection story is "a counter moves; the on-call sees it." For silent drift, the detection story is "a *reference* distribution is held somewhere, and a periodic comparison against it fires." No reference = no detection. No periodic comparison = no detection. This chapter authors both.

The chapter has three sub-classes to treat distinctly — model-weight change, kernel-regression, numeric-format change — because they have different root causes, different owners, and different mitigations, even though the on-call-visible surface looks similar at first glance. It also has three detection mechanisms — canary-set evals, output-distribution comparison, and proxy signals on cost and latency — because any one on its own has a blind spot the others cover.

## 1. The three sub-classes

### 1.1 Model-weight change

The weights served on the request path are literally different from what was served yesterday. Three provenance patterns:

- **Internal promotion.** The ML platform's model registry promoted a new version. The CI/CD pipeline deployed it. The registry change was logged; whether the quality-SLI-facing release gate caught the output shift depends on whether the gate had a canary-eval signal (chapter 03 of `mod-111` — release engineering — authors the gate). Most commonly, the gate was performance-only (TTFT / cost / availability pass) and the output-distribution signal was absent.
- **Provider alias move.** The model is hosted; the provider aliases a name (`gpt-4o`) to a dated snapshot (`gpt-4o-2024-08-06`). The provider silently moves the alias to a new snapshot (`gpt-4o-2024-11-20`). The `gen_ai.response.model` label shifts; `gen_ai.request.model` does not. This is the most common silent-drift root cause in a provider-hosted stack.
- **Base-model update without re-fine-tune.** The upstream base model was updated and a fine-tuned derivative was rebuilt on it, without updating the application's expectation of the derivative's behaviour. Formally an internal promotion, but with a provenance chain that touches the base-model vendor, the fine-tuning team, and the serving team.

**Root-cause owner.** ML platform (`ai-infra-ml-platform` peer at level 30) for internal promotions and re-fine-tunes; vendor-risk workflow (`mod-107`) for provider alias moves.

### 1.2 Kernel-regression drift

The weights are bit-for-bit identical. The kernel code path that runs them changed. Three canonical causes:

- **CUDA / cuDNN / toolkit bump.** The container image or the host kernel upgraded a library; numerically different implementations of the same operation (softmax, LayerNorm, matmul) now run under the same model file. Floating-point non-associativity means outputs change bit-wise and sometimes semantically.
- **Attention-kernel swap.** The engine switched between FlashAttention versions, between FlashAttention and cuDNN SDPA, between paged-attention block sizes, or between a fused and an unfused kernel path. Each has the same mathematical specification but a different numeric realization.
- **Precision-mode change at the kernel.** The engine's `--dtype` changed, or an FP8 path replaced an FP16 path for part of the forward pass. Arguably overlaps with 1.3; distinguished here by whether the *weights file* changed (1.3 yes, 1.2 no).

**Root-cause owner.** Inference-engine peer (`ai-infra-inference-engineer` at level 35) owns the kernel path selection; performance peer (`ai-infra-performance`) owns the kernel-level characterisation. The SRE's role is to recognise that the signal is a kernel-side change and route; they do not open Nsight.

### 1.3 Numeric-format change

The model file served is a numerically different representation of the same weights. Three canonical causes:

- **Re-quantization.** FP16 → BF16 → INT8 → FP8. The weights file is new; the "model" is logically the same; the output distribution shifts because of the quantization's error floor.
- **Calibration pass change.** A post-training-quantization (PTQ) pipeline recomputed scaling factors with a different calibration dataset. Same weights-at-full-precision; different quantized realization.
- **Rounding-mode change.** A toolchain bumped a rounding mode (round-to-nearest-even vs. truncate, stochastic rounding on / off). Rarely a conscious decision; usually a side effect of upgrading an export toolchain.

**Root-cause owner.** ML platform (owns the registry entry including the numeric format); inference-engine peer (if the engine is responsible for the format conversion at load time).

### 1.4 How the sub-classes present differently

The three sub-classes present with distinct secondary signals, and the on-call should learn to read them.

| Signal | Weight change (1.1) | Kernel regression (1.2) | Numeric-format change (1.3) |
|---|---|---|---|
| Canary-eval pass rate | Shifts, often meaningfully | Shifts, usually small but detectable | Shifts, often small and uniform across inputs |
| Output-embedding drift | Clear shift on most inputs | Small but detectable | Usually visible, uniform |
| Output-length distribution | Often shifts (different model, different verbosity) | Usually unchanged | Usually unchanged |
| Refusal rate | May shift | Usually unchanged | Usually unchanged |
| TTFT / TPOT | May shift (different model size or arch) | Shifts (different kernel speed) | Usually small change; may improve |
| Cost per request | Shifts (verbosity + possibly pricing) | Usually unchanged | Usually improves on provider price per-token basis if same model, but internal GPU-cost may improve or worsen |
| `gen_ai.response.model` label | Changes (if provider-side or label-emitting internal) | Unchanged | Unchanged |
| `gen_ai.request.model` label | Unchanged | Unchanged | Unchanged |

The distinguishing columns are "response-model label" (a change implies 1.1), "TTFT / TPOT" (a performance change with unchanged response-model label implies 1.2), and "output-length" (a clear change implies 1.1). An on-call who memorises the shape of the table can classify at a glance.

## 2. The reference problem — what you compare *against*

No reference, no detection. The chapter's central operational claim is that every production deployment serving a generative model must have a **canary set** — a small, versioned, representative sample of production-like inputs whose outputs on the current production model have been captured, scored, and stored as the "known-good" distribution.

### 2.1 What a canary set is (and is not)

A canary set is:

- A **frozen** collection of inputs — typically 100 to 5,000 examples, drawn to be representative of production traffic along the dimensions that matter (task type, input length, language, tenant class).
- **Versioned** — the set has an identifier; it is checked into source control; its contents never change silently.
- **Scored** — a reference run of the current production model on the canary set has been captured. Scores may include (a) model outputs themselves, (b) output embeddings, (c) judge-model verdicts, (d) regex / exact-match / pattern-match signals, (e) output length, (f) token counts, (g) finish reasons.
- **Re-runnable** — a scheduled job can replay the canary set against the current production path and produce a new scored run in minutes.
- **Low-cost** — small enough that running it every 15 minutes, every hour, or on-release is affordable (hundreds of requests at cheapest-model prices).

A canary set is *not*:

- The release-gate eval set. The release gate (owned by `mod-111` and the eval peer) uses a larger, slower, offline eval set to decide whether a model *should promote*. The canary set is the online, in-production sentinel that detects drift *after promotion*.
- The quality-SLI's judge pipeline. The judge pipeline scores *production traffic*; the canary set scores *reference traffic*. Both are needed. Production-traffic judge signals are the SLI; canary-set signals are the drift sentry.
- The user-facing traffic. The canary set is *shadow traffic* — synthetic, tagged, routed to the same production path as a user request but never shown to a user.

### 2.2 Building a canary set

The right time to build a canary set is before the first production deploy; the next-best time is now. A reasonable construction procedure:

1. **Sample production inputs.** Take a representative window (e.g., one week of production traffic). Draw a stratified sample across input length, language, task type, tenant class, and whatever dimensions your surface slices on. Target 500–2000 examples to start.
2. **Scrub for PII / PHI / secrets.** The canary set lives in source control, in logs, in the drift dashboard. It cannot contain user content without consent. Options: synthesize paraphrases of real inputs; use a public-corpus analog (e.g., from the same domain); use internal test-account traffic marked as sharable.
3. **Freeze and version.** Give it a version; store it in source control or an eval-dataset registry; never mutate.
4. **Score on the current model.** Run the production path on it; capture outputs, scores, metadata. This is the reference distribution.
5. **Document the "known-good" ranges.** For each measurable signal, document the expected central tendency and spread. Judge-pass rate is X%, output length is Y ± Z tokens, output-embedding-centroid is such-and-such.
6. **Wire the schedule.** The canary set runs on a schedule (every N minutes / hours), on every deploy, and on-demand during an incident. Results feed Prometheus counters and/or an LLM-trace platform.

Section 5 drills into the signal design; section 6 drills into the alert design.

### 2.3 Canary sets for provider-hosted models

A subtle operational point: for provider-hosted models, the canary set is the *only* defense against alias moves. The provider is not going to tell you they changed the snapshot behind `gpt-4o` before they do it. A canary set that runs every hour against the hosted endpoint will detect the shift the same way it detects an internal promotion. Without the canary set, the shift is invisible until user quality complaints lag in.

The flip side: canary runs against hosted providers cost money. Budget it — a 500-example hourly canary against a cheap-tier model is a few dollars a day; a 5000-example canary against a frontier model is hundreds of dollars a month. The cost is operational insurance; frame it that way to the sponsor.

## 3. Output-distribution comparison as the on-traffic signal

The canary set is reference-traffic detection. **Output-distribution comparison** is production-traffic detection — it detects drift by watching the shape of the actual output stream, not a reference. The two are complementary; use both.

### 3.1 Signals that are cheap to compute

Chapter 05 of `mod-104` introduced drift detection as a reliability signal. For the drift-detection case, the cheapest signals are:

- **Output-token length distribution.** Histogram of `gen_ai.usage.output_tokens` per model slice. A shift in the mean or in the tails of this histogram, with no traffic shape change, is drift.
- **Finish-reason distribution.** Fraction of `gen_ai.response.finish_reasons = stop` vs. `length` vs. `content_filter` vs. `tool_calls`. A material shift is drift (and a `content_filter` shift is also a refusal-spike signal — chapter 04).
- **Output-embedding centroid.** Embed a sample of outputs; track the running mean embedding. A large angle shift against the baseline centroid is drift.
- **First-token histogram.** The distribution of the *first token* generated. This is surprisingly sensitive to numeric-format and kernel changes; it is almost free to compute.
- **Vocabulary-coverage profile.** For each window, the fraction of vocabulary tokens used in outputs. Model changes often shift which parts of the vocabulary get used.

### 3.2 Signals that are more expensive but stronger

- **Judge-pass rate on production traffic (per-slice).** The quality SLI from `mod-103` chapter 02. Slower, more expensive, and a lagging signal, but direct.
- **Human spot-check sampling.** Every N hours, a sampled set of outputs goes to a human reviewer. Operationally expensive; useful as the final backstop.
- **Output-embedding distribution tests.** Formal statistical tests (KS, MMD) on the output-embedding distribution against a reference window. Chapter 05 of `mod-104` has the recipes.

### 3.3 The cheapest-signal-first discipline

The drift dashboard should put cheap signals first. The on-call looking at the panel in order should see:

1. Output-token-length distribution (cheap, fast, high-sensitivity to 1.1).
2. Finish-reason distribution (cheap, fast, high-sensitivity to 2.4-adjacent effects).
3. First-token histogram (cheap, fast, high-sensitivity to 1.2 and 1.3).
4. Output-embedding centroid drift (medium cost).
5. Judge-pass rate (expensive, but the ground truth).
6. Human spot-check (manual).

If the first three show a clear shift, the on-call is confident in a drift classification before the expensive signals land.

## 4. Cost and latency as proxy signals

The least-direct but often-fastest signal that a drift has happened is **cost and latency**. The reason is that both are already instrumented at high resolution (every request has token counts and timings) and both are already on the oncall's dashboard.

### 4.1 Cost proxy

A silent model-weight change frequently moves the output-token distribution: a newer / older / more-verbose / more-terse model emits different-length completions on the same inputs. The cost-per-request SLI from `mod-103` chapter 04 — denominated in input-tokens-plus-output-tokens-times-provider-price — will move. A cost shift with unchanged traffic shape is a drift signal.

Operationally:

- **Set a cost-per-request bollard**. A burn-rate-style alert on cost-per-request-per-model-slice. A 10–20% shift in cost-per-request on a stable traffic surface is the kind of signal the on-call wants to see.
- **Alert lives in the reliability surface, not the finance surface**. Finance alerts on absolute spend; reliability alerts on *unit* cost. The finance alert fires after the bill lands; the reliability alert fires in the window where you can route it.

### 4.2 Latency proxy

A silent kernel-regression often moves latency: a different kernel path takes different time per token. The TTFT and TPOT SLIs (`mod-103` chapter 01) will move. A latency shift with unchanged workload shape and unchanged engine config is a kernel-side drift signal.

Operationally:

- **Watch TPOT more than TTFT** for kernel drift. TTFT is dominated by prefill and admission; TPOT is dominated by the decode kernel's per-token speed. TPOT is the more-specific signal.
- **Cross-reference with engine version.** The engine release manifest is the first thing to check when TPOT drifts without a weight change.

### 4.3 The proxy signals are not sufficient on their own

Proxy signals are high-cadence and low-noise, but they do not tell you *what changed about the output*. A cost shift says "output length moved," not "output quality moved." A latency shift says "kernel path moved," not "output quality moved." Proxy signals route the on-call to the canary set and to the output-distribution panel; those are the signals that tell you whether users are affected.

## 5. Signal architecture — what you actually build

A drift-detection architecture that works at operational scale has roughly four components:

### 5.1 Canary-run orchestration

- A scheduled job (Kubernetes CronJob, Airflow DAG, Argo Workflow, whatever your platform runs) that takes the canary set, hits the production model endpoint, and writes the results.
- Results written to a trace store (LLM-trace platform or OTel backend) for per-example inspection, and summarised into Prometheus counters / histograms for alerting.
- Identified as canary traffic via a tag (`request.source = canary` or similar) — never counted in user-facing SLIs, never costed against user-facing budgets.

### 5.2 Reference store

- The last-known-good run of the canary set. A JSON / Parquet file in object storage, or a row set in a dataset registry (MLflow, Weights & Biases datasets, a Feast-like store).
- Versioned alongside the model version the reference was captured against. When the model version intentionally changes, the reference run is re-captured and promoted; the pre-promotion version is kept so a rollback can re-pin.
- Access control: reference runs can contain model outputs that reveal capabilities, prompt wording, and sometimes sensitive patterns. Treat them as production artifacts.

### 5.3 Comparison and metric emission

For each canary run, compare against the reference:

- Per-example scores (judge pass, output-length delta, output-embedding cosine distance) → Prometheus histograms labelled by canary-set-version, model-version, and slice.
- Aggregate signals (canary-pass-rate, canary-embedding-centroid-shift, canary-output-length-mean-delta) → Prometheus gauges.
- Alerting is on the aggregate signals; drill-down is to the per-example scores and the LLM-trace records.

### 5.4 Production-traffic drift pipeline

Independently of the canary pipeline, the production-traffic drift pipeline watches live traffic:

- Output-token-length histogram per slice.
- Output-embedding centroid per slice, windowed.
- Finish-reason distribution per slice.
- First-token distribution per slice.
- Judge-pass rate (if the eval pipeline is online; otherwise lagged).

Both pipelines feed the same drift dashboard. The canary signal is lower-noise and gives a known-good comparison; the production signal is higher-coverage and sees changes in traffic the canary does not cover. They disagree sometimes; the disagreement itself is informative.

## 6. Alert design

Three alert patterns worth codifying.

### 6.1 Canary-pass-rate burn

The canary set's judge-pass rate is a proportion; apply the same multi-window / multi-burn-rate logic as any SLO (see `mod-103` chapter 05). Thresholds are typically tighter than the production quality SLI because the canary set is a known distribution — a 5-percentage-point drop is meaningful.

```
# Prometheus (illustrative — exact metric names depend on your pipeline)
canary_pass_rate{model="prod-model", canary_set_version="v3"}
```

Alert fires when:

- `canary_pass_rate` drops below `baseline_pass_rate - 5%` for 15 minutes (fast window), *and*
- below `baseline_pass_rate - 2%` for 6 hours (slow window).

The classic multi-burn pattern from the SRE Workbook, applied to the canary signal.

### 6.2 Output-distribution-shift alert

For the production-traffic signals, alert on *distribution-distance* metrics from the drift pipeline (`mod-104` chapter 05 wired Evidently / Arize / WhyLabs).

```
# e.g., PSI (population stability index) on output-token-length distribution
output_token_length_psi{model=~"prod-.*"} > 0.2
```

Thresholds are domain-specific; PSI > 0.2 as a "material shift" is a widely-cited threshold but not sacrosanct. Pick thresholds against your baseline noise and document the choice.

### 6.3 Proxy-signal alert

Cost-per-request and TPOT already have alerts from `mod-103`. Add a drift-specific joint alert:

- Cost-per-request shift of > 10% AND no traffic-shape change (input-token distribution stable) → page the drift-triage channel, not the cost-governance channel.
- TPOT shift of > 10% AND no weight change (model-version label stable) AND no capacity signal → page the kernel-regression branch of the drift-triage channel (which routes to the inference-engine peer in chapter 08).

## 7. Diagnosis — the first fifteen minutes

Once a drift alert fires, the on-call's first-15-minute checklist:

1. **Confirm the alert is on reference signal, production signal, or proxy.** This decides which sub-class is most likely.
2. **Check `gen_ai.response.model` label distribution.** If the label value shifted in the last window, this is almost certainly sub-class 1.1 (weight change). Capture the old and new values; the next step is "who promoted?"
3. **Check the engine version / image tag.** If the engine or its image rolled in the window, this is suspicious for sub-class 1.2 (kernel regression). Cross-check with the engine-release channel.
4. **Check the model-format metadata.** If the registry entry is a newly-quantized artifact or a new calibration pass, this is sub-class 1.3.
5. **Check the canary signals and the production signals side by side.** If both move together, the drift is real. If canary moves and production does not, the production traffic may not be hitting the affected path. If production moves and canary does not, the canary set may not be representative — a canary-set gap.
6. **Confirm the user-facing impact.** Look at the quality SLI; look at customer-support signal; look at the eval peer's spot-check dashboard.

By minute 10–15, the on-call should have:

- A sub-class classification (1.1, 1.2, 1.3, or "still ambiguous").
- A suspect owner (ML platform, engine peer, provider).
- A rough blast radius (which slices are affected, how badly).
- A mitigation option on hold (rollback, route-around, throttle).

## 8. Mitigation and route — the SRE's lanes

Recognise-and-route is the module's contract. For silent drift, the SRE's mitigations are:

- **Roll back.** If the drift is a new model / new engine version and the previous version is still available, roll back. The on-call owns this lever because it is reversible and because the fix (figure out why the new version drifted) belongs to the owning peer.
- **Route around.** If the drift is a provider alias move and the provider offers version pinning, pin to the previous snapshot. This is a short-term fix; the long-term pinning policy belongs to vendor-risk (`mod-107`).
- **Throttle.** If the drift's business impact is bounded and immediate rollback is not available, throttle or route a fraction of traffic to a fallback (another provider, a smaller model) to limit exposure.

The SRE's mitigation ends when user impact stops. The drift's *root-cause fix* belongs to the owning peer. The on-call hand-off ticket includes (a) the sub-class classification, (b) the canary-run evidence, (c) the production-signal evidence, (d) the blast-radius estimate, (e) the mitigation in place.

## 9. Why canary evals are a quarterly investment, not a one-shot

A canary set decays. Four decay modes:

- **Representation decay.** Production traffic shifts over months; the canary set that was representative a year ago no longer is. Quarterly re-draw against production traffic.
- **PII drift.** A canary set scrubbed against yesterday's PII policy may not meet today's. Quarterly scrub review.
- **Reference drift.** The baseline model's reference run is still in use months after the model changed. Re-capture on every intentional model change.
- **Operational trust.** A canary set that nobody looks at for months loses the "I trust the signal" property. Include canary-set review in the quarterly SLO ritual (`mod-103` chapter 06).

Set a calendar: quarterly canary-set refresh, per-release reference re-capture, on-demand expansion when a new workload class is added.

## Summary

- **Silent drift is three sub-classes.** Model-weight change (1.1), kernel regression (1.2), numeric-format change (1.3). Each has a different owner.
- **Detection requires a reference.** Canary sets are the reference. Frozen, versioned, scored, cheap to re-run, PII-scrubbed. No canary set = no silent-drift detection.
- **Production-traffic signals complement canary.** Output-length distribution, finish-reason distribution, output-embedding centroid, first-token histogram, judge-pass rate.
- **Proxy signals are fastest.** Cost per request and TPOT are already on the dashboard; they catch drift via second-order effects.
- **Alert on canary burn, production distribution distance, and joint proxy signals.** Page the drift-triage channel; sub-class classification happens within 15 minutes.
- **The SRE mitigates — rollback, route-around, throttle — and hands off root cause.** Depth lives with the ML platform, engine, or vendor peer.
- **Canary sets decay.** Quarterly refresh, per-release reference re-capture, included in the SLO ritual.

The next chapter opens the quality-shaped failures — hallucination, toxic / unsafe, and refusal — where the signal is already a rate on the quality SLI dashboard and the diagnostic question is *which* rate moved and *why*.
