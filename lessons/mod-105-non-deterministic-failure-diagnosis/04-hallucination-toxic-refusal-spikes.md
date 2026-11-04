# Quality-Shaped Failures — Hallucination, Toxic/Unsafe, and Refusal Spikes

## Why this chapter exists

The three failure classes this chapter treats — hallucination-rate spike (2.2), toxic / unsafe output blast (2.3), and refusal-rate spike (2.4) — share a shape that distinguishes them from the drift family in chapter 03. For drift, the SRE builds the reference distribution themselves. For hallucination / toxic / refusal, the reference signal is already a running rate on the quality SLI dashboard from `mod-103` chapter 02. The alert that fires is a burn-rate alert, not a distribution-drift alert. The diagnostic question is not "did something drift?" but "which quality dimension burned and what fraction of the burn is attributable to which cause?"

That shift of question matters operationally. It changes the first-fifteen-minutes checklist, the suspect owners, and especially the comms posture — a toxic-output blast is a declared-incident class from minute one, even at small rate, because the regulatory and reputational blast radius is nonlinear in rate. The chapter's goal is to teach the on-call to triage the three classes together (they often co-occur) and route each to the right peer without duplicating the eval-peer's forensic work.

The companion signals come from `mod-104` chapter 05 (drift / judge agreement), chapter 01 (`gen_ai.response.finish_reasons`), and the eval-peer's judge pipeline (per `mod-103` chapter 07). Those signals must be live before this module can run; if they are not, the gap is upstream.

## 1. The three classes at operational depth

### 1.1 Hallucination-rate spike (class 2.2)

"Hallucination" in industry use covers two overlapping phenomena:

- **Unfounded claim.** The model asserts a fact it has no grounding for. Sometimes the fact is wrong (contradicted by ground truth); sometimes it is unverifiable.
- **Ungrounded with retrieval.** For RAG surfaces, the model produced an answer not supported by the retrieved context, even though the context was adequate. The model ignored or over-summarised the retrieval.

NIST AI 600-1 uses the term **confabulation** for this phenomenon (chapter 02). Operationally, your judge pipeline will have one or more of three signals:

- **Factuality / correctness verdict.** A judge model or rubric scores "is the claim correct against known ground truth." Usable where ground truth exists.
- **Groundedness verdict.** For RAG, "is every claim in the answer supported by the retrieved context." Judge compares answer to context.
- **Self-consistency score.** Sample the model N times on the same input; measure semantic agreement. A drop in self-consistency is a hallucination-adjacent signal — the model is uncertain in ways that often correlate with wrong answers.

**Primary alert source.** The quality SLI for the "factuality" or "groundedness" dimension burns. Alertmanager rule points at the hallucination-triage channel.

### 1.2 Toxic / unsafe output blast (class 2.3)

The model produced content that violates a policy — your product's content policy, a legal constraint, or an external contractual safety floor. Categories typically tracked:

- Hate speech / discrimination
- Harassment / threats
- Sexual content (especially outside the product's permitted scope)
- Self-harm / suicide-adjacent content
- Violence / graphic content
- Illegal-activity guidance (CSAM, weapons, cyberoffense, fraud)
- CBRN uplift (chemical, biological, radiological, nuclear capabilities)
- Jurisdiction-specific forbidden content

**Primary alert source.** A safety classifier (yours, a provider's, a third-party service such as Perspective API, OpenAI moderation endpoint, Azure Content Safety) raises a per-category rate above threshold. Or an external report (user complaint, press, regulator, bug bounty, red team) escalates in.

The "unsafe output blast" terminology emphasises the *rate* dimension. A single unsafe output is a bug; a measurable rate shift over baseline is a class-2.3 incident.

### 1.3 Refusal-rate spike (class 2.4)

The model is refusing requests it used to answer. Two shapes:

- **Explicit refusal.** The model returns a canned "I can't help with that" / "I'm not able to…" response. `gen_ai.response.finish_reasons = ["content_filter"]` on some providers; a product-side refusal classifier on others.
- **Over-cautious response.** The model answers but with so many disclaimers, hedging, or scope-shrinks that the response does not meet the user's need. Harder to detect; often requires a quality judge with a "usefulness" or "helpfulness" rubric.

**Primary alert source.** The `gen_ai.response.finish_reasons = content_filter` counter rises, or a product-side "refusal" classifier's rate rises. The refusal dimension of the quality SLI burns.

Refusal-rate spike is the mirror of toxic-blast: both are safety-training outcomes, both often co-occur after a model version bump, both must be instrumented together.

## 2. Why the three classes must be instrumented together

The three classes share a trade-off surface that makes them operationally entangled.

Tightening safety training (to reduce 2.3) tends to *increase* 2.4 — the model refuses more, including on legitimate inputs. Relaxing safety to reduce 2.4 tends to *increase* 2.3. Changes in retrieval, prompt templates, system messages, and base-model versions move all three dimensions at once.

An on-call who alerts on 2.3 alone will not see a 2.4-induced regression; the users complaining that the product no longer answers their questions will reach customer-support before they reach the SLO burn. An on-call who alerts on 2.4 alone will miss a 2.3-induced regression; one toxic output sampled into a press story will reach the executive team before SLO burn.

The joint instrumentation:

- **Safety-category counters** (fine-grained, per category, labelled by model and tenant class).
- **Refusal-rate counter** (both explicit-refusal and over-cautious if the eval peer's pipeline supports it).
- **Hallucination / groundedness pass-rate** (the factuality axis).
- **Quality-SLI dashboard panel that overlays all three.**

Section 6 treats the overlayed dashboard shape.

## 3. Detection discipline — three patterns

### 3.1 Burn-rate alerts (fast + slow window)

For each dimension, the standard SRE Workbook multi-window / multi-burn-rate pattern from `mod-103` chapter 05:

- **Fast window** (5–15 min) catches sudden breaks (model version bump, prompt template change).
- **Slow window** (6 h / 24 h) catches gradual regressions (drift, eval-set shift, user population shift).

Burn thresholds reflect business impact. Safety SLIs (2.3) typically carry the tightest thresholds — a 14× burn on the slow window is already severe. Hallucination SLIs (2.2) typically have a more permissive tail. Refusal SLIs (2.4) depend on the surface (a finance-chat with low refusal tolerance vs. a creative-writing surface with high tolerance).

### 3.2 Categorical spike detection (for 2.3)

For safety, the useful signal is often not a smoothed rate but a *spike* in a specific category. A sudden jump in `safety.category=self_harm` is more actionable than a flat-average "safety-event rate." Instrument:

- Per-category counters.
- Per-category rate calculations with short time windows (minute-level, not hour-level) for the categories with lowest tolerance (CSAM, CBRN, self-harm).
- Cross-reference with user-submitted reports.

### 3.3 Side-signal detection for 2.4

Over-cautious refusal is often invisible to the simple "finish_reasons=content_filter" counter because the model does not refuse explicitly — it produces an unhelpful response. Side signals:

- **User session abandonment.** The user asked something and did not follow up; session ended short.
- **Thumbs-down rate** or product-specific negative feedback.
- **Retry rate.** The user rephrased the same question (classified via embedding similarity across sequential turns).
- **Eval-pipeline helpfulness verdict.** The judge pipeline's "did the response meet the user's intent" verdict.

The SRE's dashboard surfaces these; the eval peer's pipeline computes them.

## 4. Root-cause taxonomy for each class

Each class has a set of canonical root causes. The on-call does not fix them; they classify so the right peer is engaged.

### 4.1 Root causes of 2.2 — hallucination

- **Model change.** A new model is more confabulatory than the previous on your domain. Overlaps with 2.1.
- **Prompt / system-message change.** A system-message edit shifted the model's grounding behaviour.
- **Retrieval regression.** The retrieval step started returning less-relevant documents — the model fills the gap by fabricating. Overlaps with 2.6; the joint signal is what distinguishes.
- **Temperature / sampling change.** Someone raised temperature or changed top-p / top-k.
- **Context truncation.** The input context grew and the model is dropping grounding material to fit.
- **Judge drift.** The judge is now calling correct answers wrong. Overlaps with 2.8.

### 4.2 Root causes of 2.3 — toxic / unsafe blast

- **Model change.** A new model's safety training is weaker on your category.
- **Prompt-injection surface.** Attackers found an input pattern that bypasses the safety training. Instrumentation: cross-reference 2.3 spike with inputs that match known injection patterns (from `mod-110`).
- **Prompt / system-message change.** A system-message edit weakened the safety posture (common in "we tried to make it less restrictive" edits).
- **Retrieval-injected content.** A RAG surface retrieved a document containing unsafe content, and the model regurgitated it.
- **Tool-call result-injected content.** An agent-tool returned content that got laundered through the model.
- **Safety-classifier regression.** The classifier is now over- or under-flagging. Signal: external reports do not match the internal rate.

### 4.3 Root causes of 2.4 — refusal

- **Model change.** A new model refuses more. Very common after safety-training updates by providers.
- **Prompt / system-message change.** A system-message edit tightened the model's refusal posture.
- **Content-filter threshold change.** The provider or your product changed a filter threshold.
- **Tenant / user population change.** New users are asking questions the model treats as adjacent to disallowed categories.
- **Refusal-classifier regression.** The classifier is counting the wrong things. Overlaps with 2.8.

## 5. Diagnosis — the first fifteen minutes

### 5.1 Hallucination spike

1. Confirm which quality dimension burned (factuality vs. groundedness vs. self-consistency).
2. Check the model version label — if it shifted, this is 2.2 + 2.1; route accordingly.
3. Check retrieval signals — if retrieval-score distribution dropped or retrieval-grounded-answer passage dropped, this is 2.2 + 2.6.
4. Check the prompt / system-message deployment history — any config push in the window?
5. Check the sampling parameters (temperature, top-p) — any change in the window?
6. Check judge-vs-human agreement (chapter 07) — if the judge has recalibrated, the burn may be a judge signal, not a model signal.
7. By minute 15, classify: 2.2-only, 2.2+2.1, 2.2+2.6, 2.2+2.8, or ambiguous.

### 5.2 Toxic / unsafe blast

1. **Immediately declare incident.** Minute 1, not minute 15. Toxic-output blast is a declared-incident class. Even if the rate is small, the retrospective-cost of *not* declaring is higher than the cost of declaring.
2. Identify the category (hate, self-harm, CSAM-adjacent, etc.). The playbook depends on the category.
3. Check whether the outputs came from a prompt-injection pattern — cross-reference inputs against known injection classifiers.
4. Check whether a specific tenant / user class is producing the inputs — may be red-team activity, may be an adversarial user.
5. Check the model-version label and the system-message deployment history.
6. Check whether the safety classifier is firing correctly — a few spot-check examples.
7. Engage security-safety-compliance peer (`mod-110`) and vendor-risk workflow (`mod-107`) immediately.

### 5.3 Refusal spike

1. Confirm it is finishing-reasons-driven refusal vs. over-cautious response. Look at both counters.
2. Check the model-version label — a new provider snapshot is the most common cause.
3. Check the system-message / prompt deployment history.
4. Check whether the spike is concentrated on a tenant / query class — may be a legitimate-use pattern the model started treating as borderline.
5. Check the refusal classifier's behaviour — if it recalibrated, overlaps with 2.8.
6. By minute 15, classify: 2.4-only, 2.4+2.1 (provider bump), 2.4+2.8, or ambiguous.

## 6. Dashboard shape — the quality overlay

A dashboard panel that supports the triage above has roughly five rows, overlayed in time:

| Row | Signal | Purpose |
|---|---|---|
| 1 | Hallucination / factuality / groundedness pass-rate (per model slice) | Primary 2.2 signal |
| 2 | Safety-category counters (per category, per model slice) | Primary 2.3 signal |
| 3 | Refusal rate (both finish_reasons=content_filter and product-side classifier) | Primary 2.4 signal |
| 4 | `gen_ai.response.model` label value over time (per model slice) | Overlay with 2.1 — model version shifts |
| 5 | Config deploy markers (prompt / system message / safety filter changes) | Overlay with application-layer causes |

With the five rows side by side, a 3 AM on-call glancing at the panel can read in seconds: "quality dimension A burned; model label did not shift; config was pushed at T-15 minutes; the burn started at T-8 minutes." Hypothesis forms before the first query is typed.

## 7. Comms posture — different for 2.3

The on-call's comms posture differs across the three classes.

- **2.2 Hallucination.** Internal incident channel, standard severity by burn size. External comms only if a user-visible pattern was reported.
- **2.4 Refusal.** Internal incident channel; may include customer-support team if user-visible. Rarely external.
- **2.3 Toxic / unsafe.** Internal incident channel *plus* security-safety-compliance channel *plus* legal / PR / executive channel at a severity-threshold the compliance playbook specifies (which is low — often "any confirmed category-X event" counts). The retrospective is under legal-privilege-adjacent procedures; capture every signal, do not improvise comms.

The reason for the elevated comms posture on 2.3 is regulatory: the NIST AI 600-1 "Dangerous / violent / hateful" and "Obscene / degrading" categories, the EU AI Act's "unacceptable risk" provisions, and sector-specific regulations (child-safety, financial-advice, medical) all treat these events as reportable. The reporting obligation runs from the first confirmed event; waiting to see "if it recurs" is not an option. `mod-110` authors this discipline; this chapter's job is to plug it into the on-call's first minute.

## 8. Mitigation and route — the SRE's lanes

### 8.1 For 2.2 — hallucination

- **Rollback** of a recent model promotion, prompt change, or retrieval-config change.
- **Route around** to a fallback model or route-weight shift.
- **Throttle** affected tenants / query classes if a bounded blast radius is identified.
- **Hand off** to the application layer (prompt change), eval peer (judge calibration), ML platform (model promotion), or vendor-risk (provider change).

### 8.2 For 2.3 — toxic / unsafe

- **Short-circuit** the output path for the affected category. Many safety architectures include a kill-switch for a category — route to a canned response, block the output, or escalate to a human-review queue.
- **Pin the model version** if a provider alias move is the cause.
- **Disable a tool-call path** if a tool-injection is the cause.
- **Rate-limit or block** the specific tenant / user class producing the inputs (with legal sign-off for customer accounts).
- **Hand off** to security-safety-compliance (`mod-110`) and vendor-risk (`mod-107`); the SRE's lane ends when the user-facing impact stops and the compliance disclosure process has started.

### 8.3 For 2.4 — refusal

- **Rollback** of the model / prompt / filter-threshold change.
- **Route around** to a model with looser refusal behaviour (if quality / safety trade-offs allow).
- **Adjust the product-side filter** if the filter is the cause.
- **Hand off** to the application layer or vendor-risk.

In all three cases, the SRE owns rollback / route-around / throttle. The SRE does not edit the safety filter's threshold at 3 AM; they roll back the change that moved it. The SRE does not edit the system prompt at 3 AM; they roll back the last config push. If no reversible lever is available, the SRE declares the incident mitigated-pending and hands the forensic / fix work to the peer with the daytime context to do it right.

## 9. Interaction with the chaos program

`mod-109` on chaos engineering uses this taxonomy as the menu of failures to inject. For 2.2/2.3/2.4 specifically, chaos-eng patterns include:

- **Red-team injection runs.** Pre-written adversarial inputs replayed against the production safety pipeline; the pass/fail becomes a periodic reliability signal.
- **Synthetic refusal probe set.** A set of legitimate inputs that *should not* be refused; a scheduled run measures the refusal rate against them.
- **Groundedness probe set.** A set of RAG-style inputs with known-answer context; a scheduled run measures the hallucination rate against them.

Each is a scheduled job that writes to the same quality SLI dashboard. The chaos signal gives the on-call confidence that the detection pipeline is live and sensitive; a chaos pass rate that falls over time is itself a signal the pipeline is degrading. Chapter 02 of `mod-109` authors the chaos program; this chapter notes that the chaos signal is part of the detection stack.

## 10. The eval peer's lane — explicit

This chapter is the SRE's view. The eval peer owns:

- The design of the judge rubrics (factuality, groundedness, refusal, helpfulness, safety).
- The judge model(s) and their calibration against human annotation.
- The content-safety classifier selection and calibration.
- The release-gate eval sets (which the SRE's canary set is distinct from — see chapter 03).
- The periodic human-annotation workflow that keeps the pipeline honest.
- The forensic re-run when a 2.2/2.3/2.4 incident lands.

The SRE owns:

- The dashboard that overlays the eval peer's signals with the model/config/deploy markers.
- The alert rules on the quality SLI burn.
- The incident declaration, mitigation, and comms.
- The post-incident disclosure to compliance / legal / vendor-risk.

The peer-boundary content in `mod-103` chapter 07 and `mod-104` chapter 07 holds; this chapter's job is to operate it during a quality-shaped incident.

## Summary

- **Three quality-shaped failure classes, highly entangled.** Hallucination (2.2), toxic / unsafe (2.3), refusal (2.4). They share root causes and often co-occur.
- **Instrument all three jointly.** The trade-off surface between them means alerting on one hides regressions on another.
- **2.3 is a declared-incident class from minute one.** The regulatory / reputational blast radius is nonlinear in rate; comms posture is elevated even at small rate.
- **First-fifteen-minutes checklists per class.** Confirm which dimension burned, check model-version label, check config deploy history, cross-reference adjacent signals.
- **Dashboard shape overlays five rows.** Hallucination, safety-category, refusal, model-version label, config markers.
- **SRE mitigates — rollback, route-around, throttle, short-circuit — and hands off.** Depth lives with the eval peer, the application layer, the security-safety-compliance peer, or the vendor.
- **Eval peer owns judge construction, classifier selection, release-gate design, and human-annotation workflow.** SRE owns dashboarding, alerting, incident operation, and comms.

The next chapter opens prompt-cache cliffs and retrieval-freshness regressions — infra-shaped failures whose signals live on the cache-hit-rate and retrieval-score panels but whose business impact lands on quality and cost.
