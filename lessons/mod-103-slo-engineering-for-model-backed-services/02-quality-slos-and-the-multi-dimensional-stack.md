# Quality SLOs and the Multi-Dimensional Stack

## Why this chapter exists

Chapter 01 authored the latency SLI surface. Latency SLIs alone are not sufficient — a model-backed service can be fast and still fail users by hallucinating, producing invalid JSON, over-refusing, or drifting from its format contract. The classical SRE answer "did the request return 2xx?" is blind to every one of those failure modes.

This chapter authors the *quality* SLI surface — hallucination rate, toxic-output rate, tool-call-success rate, structured-output-conformance rate, refusal rate, format-drift rate — and shows how to compose them with the latency SLIs from chapter 01 into a single multi-dimensional error-budget stack. The composition is the hard part: a quality SLI with an SLO target of 97% and a latency SLI with a target of 99.9% cannot be summed into one number without losing the diagnostic signal both were authored to provide.

The classical reference is still [SRE Workbook, Chapter 2 — Implementing SLOs](https://sre.google/workbook/implementing-slos/); the AI-shaped treatment of quality signals as reliability contracts is partly covered in NIST's [AI Risk Management Framework Generative AI Profile](https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile), especially its treatment of hallucination, safety, and output-integrity as risk classes.

A reminder on scope. Measurement methodology for quality signals (how the judge model is prompted, how the eval set is calibrated, how judge-vs-human agreement is estimated) belongs to the `ai-eval-engineer` peer at level 30. This chapter and chapter 07 are explicit about that handoff. What you own as the AI SRE is the SLI *contract*: the valid-event filter, the target percentile, the window, the alerting, and the position in the error-budget stack.

## 1. The six quality SLIs

### Hallucination rate

A "hallucination" is a response containing a factually unsupported claim. The quality signal is *constructed*, not directly observable — the eval peer publishes a pipeline that takes a response and emits `HALLUCINATED` / `NOT_HALLUCINATED` / `UNKNOWN`.

- **Valid event.** A sampled production response of a type the judge is calibrated to score. Typically: responses to questions in a known domain, excluding refusals, excluding trivially short outputs, excluding responses to prompts the judge would categorize as out-of-distribution.
- **Good event.** Valid event where the judge returned `NOT_HALLUCINATED`.
- **Sampling.** Hallucination judging is expensive; you cannot judge every production response. The SLI runs on a sampled slice — typical starting point is 1% of production traffic, stratified by request-type. Document the sampling methodology in the valid-event filter.
- **Target shape.** Not a classical availability number. A hallucination-rate SLO of 99.9% is unreachable on most domains and most model classes; a realistic starting point is in the high-90s but reflects the actual performance of the model on your traffic, baselined before the SLO is signed.
- **Window.** 28 rolling days is the classical default; for low-traffic surfaces where sampling yields too few events for statistical power, extend the window (56 or 90 days) or drop the SLO tier to a warn-only signal.

### Toxic-output rate

A "toxic output" is a response containing content classified as harmful under the organization's policy — typically includes toxicity, harassment, PII leakage, and other policy-defined harms.

- **Valid event.** Any response that reaches the client. (Unlike hallucination, you *can* run a toxicity classifier on 100% of production; it is cheap.)
- **Good event.** Valid event where the toxicity classifier returned below-threshold on every policy dimension.
- **Target shape.** Much higher than hallucination. Toxic outputs are typically treated as policy violations, not probabilistic artifacts; a reasonable starting point is 99.9%+ of outputs classified below-threshold, with the toxic-output count treated as an incident-triggering signal in its own right. Correlates with the `ai-infra-security` peer's PII / safety sidecars.
- **Watch for.** Classifier false-positive rate. A toxicity classifier that over-flags sends your SLO into a loop where real responses are blocked, user experience degrades, and the "toxicity rate" number becomes meaningless because it is dominated by false alarms. The eval peer owns calibrating the classifier; the SLI you author assumes a calibrated signal.

### Tool-call-success rate

For services that invoke external tools (function-calling, agent orchestration), the fraction of tool-calls where the arguments were well-formed, the called tool completed, and the result was consumed by the next step of the flow.

- **Valid event.** A request that attempts a tool call. (Requests that complete without needing a tool call are excluded.)
- **Good event.** Valid event where the first tool-call attempt returned a usable result — no JSON parse failure, no tool-side 5xx, no argument-schema violation.
- **Why "first attempt."** Retries hide the signal. If the SLI counts "eventually succeeded after N retries" as good, you miss a serious regression (e.g., the model produces invalid JSON 50% of the time but retry policy absorbs it). The retry rate is a separate SLI; aim to publish both.
- **Target shape.** Depends heavily on the complexity of the tool surface. For a one-function tool library, 99%+ is reachable; for a 50-function library with complex argument schemas, 95% may be realistic. Baseline before locking.

### Structured-output-conformance rate

For services that return JSON / YAML / structured envelopes, the fraction of responses that validate against the declared schema on the first attempt.

- **Valid event.** Any response for an endpoint declared as structured-output.
- **Good event.** Response parses successfully *and* validates against the declared schema (JSON Schema, Pydantic model, protobuf, etc.) *and* contains all fields marked required by the application contract.
- **Why distinct from tool-call-success.** Tool-call-success wraps structured-output-conformance plus the downstream tool-side execution. You want both — a drop in structured-output-conformance isolates the failure to the model's output integrity; a drop in tool-call-success isolates it to the downstream tool. The composition in section 3 relies on this.
- **Target shape.** For models with native structured-output support (constrained decoding, JSON mode), reachable targets are high — 99%+. For models without, the rate is lower and often model-version-specific; a model upgrade can shift this SLI materially and that is the specific case the release gate in chapter 05 catches.

### Refusal rate

The fraction of responses where the model declined to answer. Refusals are a two-sided SLI: too few is a safety failure (the model is answering things it should not), too many is a usability failure (the model is over-refusing).

- **Valid event.** Any response.
- **Good event, lower bound.** The refusal was warranted — the request violated policy. (Measured against a rule-based filter or a judge model; the eval peer owns both.)
- **Good event, upper bound.** The refusal was *unwarranted* and the model over-refused. This is often tracked as a separate SLI: *over-refusal rate*, with its own target.
- **Target shape.** Two numbers in the SLO document: "warranted-refusal rate ≥ X% on policy-violating prompts" (a true-positive-style metric) and "over-refusal rate ≤ Y% on benign prompts" (a false-positive-style metric). Both come from eval sets the peer maintains.
- **Watch for.** Model-upgrade regressions. A new model release often shifts the refusal distribution in one direction or the other; the release gate from chapter 05 catches the regression before it reaches the full fleet.

### Format-drift rate

The fraction of responses that silently change *style or structure* without the application contract changing. Includes: output suddenly includes markdown when previously it did not, structured output adds optional fields the client does not consume, apology prefixes appear on previously direct responses, citations shift format.

- **Valid event.** Any response.
- **Good event.** Response matches the format fingerprint established at the last release. The format fingerprint is a rule-based summary (does it contain a code fence? does it start with "I'd be happy to"? is the length distribution within the expected band?) authored by the eval peer.
- **Why a separate SLI.** Format drift does not fail hard. The structured-output parser still succeeds; the tool call still fires; the latency is fine. But the downstream consumer (a UI, a parser in the client application) may be depending on a format the model has quietly moved off of. The SLI catches the silent regression before it becomes a user-visible bug.
- **Target shape.** Depends on how brittle the downstream consumers are. For a surface where the consumer is a UI that re-formats anyway, format-drift can be a warn-only signal. For a surface where a downstream agent parses specific sentinels, format-drift is a hard SLO.

## 2. The composition decision — multi-SLO stack, revisited

Chapter 02 of `mod-102` introduced the choice between a *composite good event* and a *multi-SLO stack*. For the quality + latency surface, that choice is more consequential than for latency alone.

### The composite good event

One predicate: a request is "good" if every one of (A) protocol-good ∧ (B) latency-good ∧ (C) quality-good ∧ (D) cost-good. One SLI, one SLO target, one budget. Simple policy, poor diagnostic.

**What breaks.** The failure pattern of a model-backed service is dimensional — a prefill-queueing incident breaks latency but not quality; a model upgrade can break quality but not latency; a provider degradation can break both in correlated ways; a schema change can break structured-output-conformance while all other dimensions are clean. A composite SLO fires on every one of those but tells you nothing about which dimension is the cause. The on-call primary spends their first ten minutes on the page just deciding which dashboard to look at.

### The multi-SLO stack

Each dimension owns its own SLI, its own SLO target, its own window, its own budget, its own burn-rate alerts. The policy from chapter 05 then reconciles across them.

**What this buys.** Every page tells you the dimension of the regression. "Latency SLO 500ms-TTFT is burning fast" is a different action tree than "structured-output-conformance SLO 99% is burning fast," and the alert content reflects it. The reconciliation logic (chapter 05) handles correlated burn.

**What this costs.** More SLIs to instrument. More burn-rate alerts to tune. A longer SLO review meeting.

### The recommended default

Multi-SLO stack, reconciled in policy. The authoring overhead is linear in the number of SLIs; the diagnostic value is the whole reason you are composing quality with latency in the first place. If you default to a composite you have saved paperwork and lost the ability to say which dimension burned.

The remaining chapters of this module assume the multi-SLO-stack default.

## 3. Composing latency and quality into a single budget — the arithmetic

Even with a multi-SLO stack, the policy has to be able to answer: *is the composite user experience within the overall error budget?* That requires a composition rule.

### The independence approximation

If the failure modes of latency and quality are approximately *independent* on the request level — a slow request is no more or less likely to be a bad-quality request than a fast one — then the composite "all dimensions good" rate factors approximately as the product of the individual rates. With rates near 1, that is well-approximated by the sum of the shortfalls:

```
composite_shortfall  ≈  latency_shortfall + quality_shortfall + cost_shortfall + protocol_shortfall
```

So an availability SLO of 99.9% (0.1% shortfall), a latency SLO of 99% (1% shortfall), a quality SLO of 97% (3% shortfall), and a structured-output SLO of 99% (1% shortfall) have a *composite* shortfall ceiling of roughly 5.1%. The composite SLO is ≥ 94.9%.

**Why this matters.** The composite ceiling is what you promise the business sponsor against user experience. The individual SLIs are what the on-call rotation acts against. If the composite ceiling is lower than the business can accept, you tighten *the dimension you can tighten cheapest* — tightening a 97% quality target to 98% costs eval capacity and model improvements; tightening a 99% latency target to 99.5% costs fleet capacity.

### The correlation caveat

The independence approximation is a floor, not a ceiling. Many realistic failure modes *correlate* latency and quality — a long-context prompt is both slower *and* more hallucination-prone; a provider-degraded state slows requests *and* worsens their quality. In correlated-burn, the composite shortfall is *less* than the sum (a single underlying event causes both). The policy in chapter 05 handles correlated burn explicitly.

Do not spend effort authoring a closed-form composite SLO. The useful artifact is the sum-of-shortfalls ceiling (quick, defensible against the sponsor) plus the reconciliation rule in the policy (operates in correlated-burn cases).

### Visualizing the stack

A simple stacked table in your SLO document conveys the composition better than arithmetic:

| Dimension | SLI | Target | 28-day shortfall budget | Window | Burn-rate thresholds |
|---|---|---|---|---|---|
| Availability | 2xx-or-3xx rate on `/v1/chat/completions` | 99.9% | 0.1% | 28d | 14.4×/1h, 6×/6h, 1×/3d |
| Latency — TTFT | p95 TTFT ≤ *T1* ms, cache-miss slice | 99% | 1% | 28d | 14.4×/1h, 6×/6h, 1×/3d |
| Latency — TPOT | p95 TPOT ≤ *T2* ms/token | 99% | 1% | 28d | 14.4×/1h, 6×/6h, 1×/3d |
| Latency — ITL stall | fewer than 0.1% of requests contain a gap > 1s | 99.9% | 0.1% | 28d | 14.4×/1h, 6×/6h, 1×/3d |
| Quality — hallucination | judge `NOT_HALLUCINATED` on 1% sample | 97% | 3% | 28d | burn thresholds adjusted for 3% SLO |
| Quality — toxicity | toxicity classifier below threshold | 99.9% | 0.1% | 28d | immediate-page on any breach |
| Quality — structured output | first-attempt schema validation | 99% | 1% | 28d | 14.4×/1h, 6×/6h, 1×/3d |
| Quality — tool call | first-attempt tool invocation success | 98% | 2% | 28d | adjusted |
| Quality — refusal (warranted) | refusals-on-policy-violations | ≥ 99% | 1% | 28d | slow-burn only |
| Quality — over-refusal | refusals-on-benign | ≤ 1% | 1% | 28d | slow-burn only |
| Quality — format drift | fingerprint match | 99% | 1% | 28d | slow-burn; warn-only initially |
| Cost | see chapter 04 | — | — | — | — |

The specific SLO values above are *illustrative*. Chapter 06 of this module authors the lifecycle for baselining, instrumenting, alerting, and reviewing them; your real document comes from running that lifecycle against your workload.

## 4. Burn-rate alerts for a 97% SLO are not the same as for a 99.9% SLO

The multi-window / multi-burn-rate table from [SRE Workbook, Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) assumes a particular SLO numeracy — the canonical examples use 99.9% availability. A quality SLO at 97% *has 30× more budget* than an availability SLO at 99.9%, so a given absolute number of bad events burns a very different fraction of budget.

Two practical consequences:

1. **Recompute the burn-rate thresholds per SLO.** The multiplier (14.4×, 6×, 1×) stays — they correspond to "2% of budget in 1 hour," "5% of budget in 6 hours," "10% of budget in 3 days." But the *absolute* rate those multipliers translate to depends on the SLO. Do the arithmetic once per SLI; do not copy-paste.
2. **Noisy SLIs need wider windows.** A quality SLI constructed from a 1% production sample will be inherently noisier than an availability SLI measured on 100% of traffic. Lengthen the burn-rate alert windows (1h → 6h for the fast burn) to raise the statistical power of the alert; otherwise you get noisy pages.

Document the alert thresholds *per SLI* in the SLO document, not as a single reused table.

## 5. The eval-peer handoff explicit at the SLI level

Every quality SLI depends on a constructed signal — judge model, rule-based classifier, user-feedback pipeline. The signal is owned by the `ai-eval-engineer` peer. The SLI is owned by you. The handoff lives inside the SLI definition and should be visible when the SLO document is read.

A reusable paragraph to include with every quality SLI:

> **Signal source.** The `NOT_HALLUCINATED` judgment is produced by the eval-peer-owned judge pipeline `judge-pipeline-vN` at commit `SHA`, prompted against the sampled response as described in the pipeline's interface document. The SLI consumes the pipeline's structured output; it does not reach into the pipeline's internals. Changes to the judge model, prompt, or calibration are coordinated via the eval peer's change log.

Two things that paragraph does for you:

1. **It names the owner of the signal.** When the SLI starts burning in a way that looks like a judge-drift (the eval-peer-owned side), the on-call primary routes the page to the peer rather than debugging the LLM.
2. **It names the version of the signal.** A judge-pipeline bump is a *release event* that can shift the SLI baseline. If the version is not in writing, the SLI silently inherits the shift and the baseline is lost. The coordination chapter (chapter 07) covers the full peer-ownership contract.

## Summary

- **Six quality SLIs** — hallucination, toxic-output, tool-call-success, structured-output-conformance, refusal (two-sided), format-drift. Each with explicit valid-event filter, sampling methodology where relevant, and target shape grounded in baseline rather than intuition.
- **Multi-SLO stack, not composite** — each dimension stands alone with its own target, window, and burn-rate alerts. Composition happens in policy (chapter 05), not in the SLI definition.
- **Composition arithmetic** — sum-of-shortfalls as a composite ceiling for business-side defence; correlation caveat handled in reconciliation rule. No closed-form composite SLO.
- **Burn-rate alerts per SLO target** — the multi-window / multi-burn-rate multipliers stay, but the absolute thresholds differ for a 97% SLO and a 99.9% SLO. Recompute once per SLI; widen windows for noisy sampled SLIs.
- **Eval peer handoff explicit at the SLI level** — every quality SLI names the signal source, the pipeline version, and the peer's change-log ownership. Covered in depth in chapter 07.

The next chapter addresses the third SLI dimension that did not have a classical equivalent: availability for streaming responses, where "the request succeeded" is not binary when the user already saw half the response.
