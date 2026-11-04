# Response Diffing with Semantic-Equivalence Tolerance

## Why this chapter exists

Chapter 01 declared the AI-adapted canary analysis depends on a quality predicate that tolerates non-determinism. This chapter writes the predicate. It is the single most AI-specific piece of the module: classical progressive delivery has no equivalent to it, because classical services produce comparable responses and the tooling (Twitter's Diffy, GitHub's Scientist, LinkedIn's Darkstar, Facebook's Sparts) treats byte-level diff as the ground truth. On a model-backed route, byte-level diff is noise. The release-gate pipeline needs a shape of diffing that says "these two versions produce *equivalent* responses" where equivalent is defined against the product's quality intent rather than against the text.

Three audiences read this chapter:

- The AI SRE authoring the canary-analysis signal that chapter 01 wires in. The output here is the `judge-score-delta` and `distribution-shift` metrics referenced by the analysis template.
- The engineer running a shadow-traffic experiment (chapter 03), who needs the same diff layer applied to mirrored requests.
- The release-gate owner (chapter 05), who treats the semantic-equivalence verdict as one predicate in the two-predicate contract.

The chapter is **bounded**. It does not teach how to train a judge model; that is at the `ai-eval-engineer` peer. It does not define what "quality" means on a given product; that is the product owner and the eval team. It does define the **engineering shape** of the diff pipeline — what gets compared, by what predicate family, with what statistical discipline, and how the verdict is emitted so the rollout controller and the release gate can consume it.

## 1. What "diff" means on an AI response

A classical response diff asks a single question: *do the bytes match?* For a model-backed response, three questions are live, and the engineer picks which ones the pipeline asks:

- **Lexical diff.** Byte-level or token-level comparison. Almost always a full failure. Useful only as a binary "did the model actually change?" smoke test.
- **Structural diff.** For responses with a structured envelope (JSON function-call arguments, structured output under a schema, tool-call sequences), diff the structure while tolerating variance in the free-text slots. This recovers a lot of signal for routes where structure is the contract.
- **Semantic diff.** Compare whether the two responses *mean the same thing* for the user's purpose: do they ground the same claims, refuse the same requests, cover the same required topics, hit the same safety threshold, cost roughly the same tokens to produce? This is the AI-native question.

The three are not mutually exclusive; a production pipeline runs the cheap checks (lexical, structural) first as filters and routes the expensive check (semantic) through the judge layer. §4 writes the composition.

## 2. The response-diff pipeline — shape and stages

A representative shape for an AI response-diff pipeline applied to a canary or shadow request:

```
 request ──> stable serving ──┐
                              ├──> response-pair collector
 request ──> canary serving ──┘           │
                                           ▼
                           ┌───────────────┴───────────────┐
                           │  Stage 1 — trivial filters    │   (bytes match → score=1.0, exit)
                           ├───────────────────────────────┤
                           │  Stage 2 — structural diff    │   (schema / tool-call shape)
                           ├───────────────────────────────┤
                           │  Stage 3 — semantic judge     │   (LLM-as-judge / rules / human)
                           ├───────────────────────────────┤
                           │  Stage 4 — distribution agg   │   (histogram over many requests)
                           └───────────────────────────────┘
                                           │
                                           ▼
                           ┌───────────────┴───────────────┐
                           │  release-gate / canary signal │
                           └───────────────────────────────┘
```

Stages 1–3 run per response pair. Stage 4 aggregates over the canary-analysis window and emits the signal chapter 01 §3.2 reads. Each stage either passes-through, assigns a score, or emits a verdict.

### 2.1 Stage 1 — trivial filters

Three filters catch the non-regression cases cheaply before the judge runs:

- **Bytes match.** On occasions where the model is deterministic enough (temperature 0, same KV cache, cached provider response), the responses can be byte-identical. If so, equivalence=1.0 and the response does not spend judge cost.
- **Trivial normalisation match.** After stripping whitespace, normalising Unicode, canonicalising JSON key order, and sorting tool-call arguments, do the responses match? Many "non-identical" responses differ only in cosmetic ways that no product would consider a regression.
- **Empty on both sides / refusal on both sides.** If both canary and stable refused, that is equivalence (though it may still be an overall-refusal spike, which the refusal-rate SLI chapter 01 §3.2 catches separately).

Stage 1 is cheap, composable, and catches a double-digit percentage of pairs on routes that have significant deterministic structure (RAG routes with grounded answers, function-calling routes with small-domain outputs).

### 2.2 Stage 2 — structural diff

For responses with a structured envelope, the structural layer is high-signal and low-cost. Representative checks:

- **JSON schema conformance.** Both responses must parse and conform. Stable valid, canary invalid → clear regression. Both valid → continue.
- **Field-level structural comparison.** For the known fields of the schema, compare values with per-field tolerances: exact-match on enum fields, numeric-tolerance on numeric fields, set-equality on list fields, semantic-diff only on free-text fields.
- **Tool-call sequence comparison.** For agent routes, compare the sequence of tool names and argument shapes (not the exact arg values). A canary that calls `search_docs` then `summarise` where stable called `summarise` directly is doing something different; a diff may be benign (it may also be a regression), but it is *information* for the gate.

Structural diff is often expressed as a per-field weighted score in `[0, 1]`. The output of stage 2 is a per-response structural-equivalence score, which stage 3 consumes as context (or skips if the structural score is below a hard threshold — a structural regression is often a regression full stop and does not need the semantic layer).

### 2.3 Stage 3 — semantic judge

The semantic layer is the expensive one, and its methodology is *not* the AI SRE's to author. The shape is:

- **LLM-as-judge.** A strong model (frequently a larger / more expensive one than the model under test) is prompted with the user's original request, the stable response, and the canary response, and asked: are these equivalent for the user's purpose? The judge emits a score (typically `[0, 1]`) and, often, a rationale.
- **Rule-based semantic checks.** For narrow domains (classification outputs, extraction outputs, bounded-answer QA), a rule or a classical NLP metric (ROUGE / BLEU variants, embedding cosine) is faster and sometimes more reliable than an LLM-as-judge.
- **Human-in-the-loop sampling.** A fraction of pairs are routed to a human reviewer for ground-truth labels; those labels calibrate the LLM-as-judge and the rule-based checks.

The AI SRE does *not* author the judge prompt, train the judge model, or set the scoring rubric. The AI SRE **consumes** the judge's score, enforces its runtime characteristics (latency, cost, availability), and feeds the score into the aggregate.

The eval-engineer peer owns the judge (see chapter 08 §3 for the boundary). The relevant external references for judge methodology are the LLM-as-a-judge literature — the Zheng et al. 2023 *Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena* paper is the canonical starting point — and the practices around judge calibration in `mod-105` chapter 06. <!-- needs-research: confirm the exact title / venue for Zheng et al. 2023 before publication -->

### 2.4 Stage 4 — distribution aggregation

The release-gate signal is not a per-request verdict; it is a distribution-level comparison over the canary-analysis window. Stage 4 aggregates stage-3 scores and emits:

- **Mean equivalence score** over the window, with confidence interval.
- **Fraction of pairs below a per-fleet equivalence threshold** (e.g., "what fraction of pairs scored below 0.7?").
- **Stratified metrics** by prompt category, language, retrieval-topic, user cohort.

The stratification matters. A 0.05 mean-equivalence drop is tolerable if it's uniform; a 0.05 mean drop concentrated in Japanese-language prompts is a regression on a cohort that the aggregate hides.

## 3. Semantic-equivalence tolerance — the policy layer

"Tolerance" is a policy, not a measurement. The pipeline measures equivalence; the function *decides* how far from equivalent a canary may drift and still promote. Four decisions make the policy explicit:

### 3.1 The equivalence threshold

A single number in `[0, 1]`: the minimum per-pair equivalence score below which a pair is counted as regressed. Representative defaults:

- **RAG routes with high-ground-truth tasks (summarisation, extraction):** threshold 0.8–0.9. The user expects a specific answer; drift is costly.
- **Open chat routes:** threshold 0.5–0.7. The user tolerates stylistic variation; the judge should not count stylistic rewrites as regressions.
- **Safety-adjacent routes:** threshold 0.95. A safety regression is not tradeable against a quality improvement; if the canary refuses or warns less than stable on a safety-sensitive prompt, it is a regression regardless of mean quality.

The threshold is set in the release-gate contract (chapter 05) and signed by the product owner, the eval team, and the AI SRE function.

### 3.2 The regression-rate budget

The second number: what fraction of pairs may regress and still promote. A pipeline that counts only the regressed fraction (ignoring improvements) is tempting because "improvements are fine, right?" — but a model that improves 70% of responses and regresses 10% is a more complicated promotion than one that moves 80% evenly upward. Representative defaults:

- **Regressed fraction ≤ 5%** on canary routes.
- **Regressed fraction ≤ 1%** on safety-adjacent routes.
- **Net-positive sign on mean score** on all routes (a model that is "roughly as good" overall but with higher variance is not a promotion).

### 3.3 The cohort breakdown

The per-cohort equivalence is evaluated, not only the aggregate. A canary promotes only when every cohort in the pre-committed list passes the threshold and the regression-rate budget. The cohorts come from §4 of chapter 05's release-gate contract.

### 3.4 The judge-source discipline

The judge itself is a dependency. Three disciplines defend it:

- **The judge is not the model-under-test.** Canary evaluating itself is a well-known failure mode; the judge must be a different model (ideally a stronger one) or a rule-based check.
- **The judge is versioned.** A release-gate comparison between stable and canary uses judge version `J_n`; a later comparison between the new stable (formerly canary) and the next canary uses `J_n` for consistency until a new judge is explicitly promoted. Changing the judge changes the signal.
- **The judge is calibrated.** Human-labelled pairs establish the judge's agreement with ground truth; the agreement is reported with the release-gate verdict so the function knows how much weight to put on the judge's score.

The AI SRE does *not* set these disciplines but *enforces* their presence before shipping the pipeline to production.

## 4. The statistical layer — making the verdict defensible

A per-pair score is not a verdict. The release gate reads a *distributional* verdict, and the shape of that verdict is where many AI canary pipelines go wrong. The AI SRE's checklist:

### 4.1 The sample size is large enough

The confidence interval on the mean equivalence score narrows with the number of pairs. For a judge whose per-pair standard deviation is 0.15 and a threshold of 0.05, detecting a shift at 95% confidence requires on the order of 150–300 pairs per cohort per window (back-of-envelope; precise number depends on the within-cohort variance). A 1% canary that sees 1000 requests in the analysis window and splits into 5 cohorts has 200 pairs per cohort — right at the edge. Either narrow the cohort list, extend the window, or widen the threshold.

### 4.2 The comparison is paired, not aggregate

Compare the same request's stable and canary responses, not the aggregate of stable responses against the aggregate of canary responses. A paired comparison controls for request-side variation (prompt-length distribution, topic distribution, time-of-day effects). Unpaired comparisons smuggle those confounds into the signal.

In practice, pairing requires that both stable and canary see the same request; shadow-traffic patterns (chapter 03) are the default way to produce the pair. Chapter 01's weight-ramp canary does *not* produce paired data (one user hits stable, another hits canary); if pairing is required, run the diff against shadow traffic rather than against the live canary split. The release gate (chapter 05) reads both signals.

### 4.3 The confidence interval is published

The verdict emitted to the release gate is not a point estimate; it is `(mean_delta, confidence_interval, sample_size)`. The gate's predicate reads the lower bound of the confidence interval, not the point estimate. A canary whose point-estimate delta is +0.02 but whose lower bound is −0.04 has not yet produced enough evidence to promote; the gate holds.

### 4.4 The test is one-sided

The hypothesis is "the canary is not worse than stable." A two-sided test that treats improvement and regression symmetrically produces a conservative threshold that holds the release even when the canary is unambiguously better. One-sided: "is the canary worse by more than the tolerance?" — if no, the gate passes.

## 5. Evaluating equivalence on streaming responses

Streaming responses have a time axis the diff pipeline must respect. Two complications:

- **Inter-token pause structure.** A canary that produces the same tokens but with 3× the TTFT is "equivalent" semantically but regressed operationally. The streaming-health SLIs (chapter 01 §3.1) catch this; the diff pipeline does not need to.
- **Partial-stream diff.** Diffing a 1000-token response against another 1000-token response is one check; diffing a 1000-token response against a 700-token truncation is another. The pipeline must treat a completed response and a truncated response differently from a completed response and a completed response of different length.

The pragmatic shape: the diff pipeline consumes the *final* response (after the terminal event closes the stream), not the stream. The streaming SLIs capture the time-axis signal; the diff pipeline captures the content-axis signal; they are orthogonal.

## 6. Evaluating equivalence on agent / tool-using routes

Agent routes extend the problem surface. The canary may have taken a different path to the same answer, taken the same path but different tools, or produced side effects (tool calls with external consequences) that the user will notice regardless of whether the final text matches.

Three additional diffs apply:

- **Tool-call graph diff.** Compare the sequence of tool invocations, their arguments, and their ordering. A canary that reaches the same answer through different tools is doing something the gate should consider.
- **Side-effect attribution.** Any tool that has external effects (send_email, pay_invoice, publish_post) is called in-band on the canary path and in-band on the stable path — double side effects, by default. The diff layer must run tool calls through a **mock sandbox** or **side-effect-stripped shadow mode** (chapter 03 §4) during the canary analysis; chapter 03 writes the discipline.
- **Cost-to-answer attribution.** Agent routes amplify cost deltas because each tool call is a cost line; diff the per-session cost, not only the per-request cost.

For routes where side effects are irreversible and cannot be sandboxed (finance, email, moderation decisions), canary analysis cannot be used at all; the function falls back to shadow-with-side-effects-disabled or to staged rollout to internal cohorts first (chapter 01 §6.1).

## 7. The equivalence pipeline is a dependency

A judge model with its own availability, latency, and cost profile sits on the release-gate critical path. Treat it as the module's own dependency:

- **Judge SLOs.** The judge has an availability SLO, a latency SLO, and a cost budget. If the judge is down, the release gate degrades to "cannot evaluate quality predicate" — not "promote anyway."
- **Judge failover.** A secondary judge (a different model, a rule-based check, a sampled human queue) is pre-wired. The `mod-107` chapter 02 fallback contract shape applies to judges.
- **Judge version-change protocol.** Changing the judge changes the signal; chapter 05 §4 includes a judge-version field in the release-gate contract so the human reviewer knows which judge gated the promotion.

An equivalence pipeline the function cannot explain the health of is a pipeline the function should not depend on.

## 8. The AI SRE's response-diffing checklist

Before relying on a response-diff pipeline as a canary gate for the first time:

1. **The pipeline is staged** (lexical → structural → semantic → aggregate), not a single monolithic LLM judge call.
2. **Stage 1 / 2 filter rates are measured** so the function knows what fraction of pairs reach the expensive semantic stage.
3. **The judge is not the model-under-test** and is a separately-owned dependency.
4. **Judge version and judge calibration are published** with every release-gate verdict.
5. **The equivalence threshold and regression-rate budget are signed** into the release-gate contract (chapter 05) by the product owner, the eval team, and the AI SRE function.
6. **The comparison is paired** and the sample size is pre-computed for the chosen tolerance; the confidence interval is included in the emitted verdict.
7. **Stratified cohorts** are evaluated, not only the aggregate.
8. **Streaming responses are diffed on their completed form**, with streaming-health captured in the orthogonal SLIs.
9. **Agent / tool-using routes diff the tool-call graph and the cost** in addition to the final response; side effects are sandboxed in the canary path.
10. **The judge's own SLOs, fallback, and version-change protocol are documented.**

A pipeline missing any of the ten is a pipeline whose verdicts the release gate should not trust.

## Summary

- Response diffing on AI routes replaces byte-level equality with a staged pipeline: lexical → structural → semantic → distributional aggregate.
- The semantic stage is owned by the eval peer, not by the SRE; the SRE owns the engineering shape, the dependency management, and the signal emission.
- Semantic-equivalence tolerance is a **policy**: threshold, regression-rate budget, cohort breakdown, judge-source discipline. The policy is signed into the release-gate contract.
- The statistical layer makes the verdict defensible: paired comparisons, confidence intervals, one-sided tests, pre-computed sample sizes.
- Streaming responses diff the final content; streaming health is captured in orthogonal SLIs.
- Agent / tool-using routes add tool-call-graph diff, side-effect sandboxing, and cost diff.
- The judge is itself a dependency with its own SLOs and failover; the release gate must know what to do when the judge is unavailable.

Chapter 03 writes the shadow-traffic pattern that this pipeline is most naturally paired with.
