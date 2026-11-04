# The AI-Specific Failure Taxonomy an AI SRE Must Carry

## Why this chapter exists

Classical SRE's incident vocabulary — `429`, `500`, `timeout`, `OOMKilled`, `CrashLoopBackOff`, `DNS resolution failure`, `cert expiry`, `disk-full`, `thundering herd`, `cascading failure` — is a complete taxonomy for a deterministic request/response service whose correctness is verified by the HTTP status code. An AI SRE will keep using every one of those terms. The problem is that the model-backed services the role operates against fail in ways that leave every one of those terms *clean*. The request succeeded. The response was returned. The status code was 200. The response was still wrong, or offensive, or late, or expensive, or refused, or looped, or hallucinated, or merely different from yesterday in ways that mattered to a user the SRE never meets.

The gap matters operationally. During an incident, the on-call must be able to *name* the failure within the first few minutes — the name tells them which dashboard to open, which peer to engage, and which playbook to run. A taxonomy whose names are "something is wrong with the model" and "latency is up but no errors" is operationally useless. The purpose of this chapter is to replace the empty taxonomy with ten concrete, named classes, each of which comes with a *signal*, a *suspect owner*, and a *time-to-recognition* budget. Subsequent chapters deepen each class in turn.

This is the first of two chapters that build the catalog. Chapter 02 grounds it in the authoritative external sources — NIST AI RMF Generative AI Profile, AI Incident Database, OECD.AI Incidents Monitor, MIT AI Risk Repository — so the vocabulary survives regulator / auditor review.

## 1. What classical SRE's taxonomy covers and where it stops

Classical SRE (the "SRE Book" canon from Google, the on-call culture of the 2010s cloud) names failure classes along four axes:

- **Availability failures.** The service is down or error-prone. Signals: HTTP 5xx rate, success-ratio SLI burn, probe failures.
- **Latency failures.** The service is slow. Signals: p50/p95/p99 histogram burn.
- **Capacity failures.** The service is overloaded. Signals: queue depth, admission rejects, autoscaler saturation.
- **Correctness failures (deterministic).** The service returned the wrong answer because of a code bug. Signals: a test fails, a canary diverges, a user reports a 1:1 reproduction, logs show a stack trace.

The four axes compose cleanly because each failure has an unambiguous *signal* and an unambiguous *diff* against the known good. The service was up; now it is down. The latency was 100ms; now it is 2s. The test passed; now it fails. The on-call's job is to confirm the signal, find the change, roll it back or route around it.

The problem starts when the service under operation returns a correctness answer whose *rightness* is not defined by a test. For a model-backed service:

- The response is a probability distribution over tokens, sampled. Two identical inputs yield two different outputs, both arguably correct.
- The correctness signal is a judge model's score, a human review, a user thumbs-up — all noisy, all expensive, all lagged.
- The failure modes include being *too helpful* (helpful with a disallowed request), *too refusing* (refusing a legitimate one), *too confident* (hallucinating), or *too similar to yesterday with a different model version* (silent drift). None of these raise a 500.
- A substantial fraction of incidents involve *no change in the code the SRE controls* — the provider's model moved, the embedding index drifted, the retrieval corpus went stale, the judge model recalibrated.

Classical SRE's four axes still fire — latency can regress, cost can explode, availability can still break — but they are now *lagging indicators*, not the direct signal. The direct signals live in the AI-specific taxonomy this chapter builds.

## 2. The ten failure classes

The ten classes below are the working taxonomy of this module. Each name is the handle an on-call uses on a page, in an incident channel, and in a retrospective. The classes are chosen to be (a) *mutually distinguishable* at triage time from signals in `mod-104`'s observability stack, (b) *routable* to a distinct peer or combination of peers, and (c) *grounded* in at least one of the external sources catalogued in chapter 02.

### 2.1 Silent output drift

The model's outputs have changed in a way no error surfaces. Three sub-classes at the root cause level:

- **Model-weight change.** A new model version was deployed — on your fleet, under a provider alias, or by a registry promotion — and the distribution of outputs for the same inputs shifted.
- **Kernel-regression drift.** The weights are identical but a kernel-level change (CUDA toolkit bump, attention-kernel swap, FlashAttention → cuDNN SDPA switch) produces bit-level different outputs because floating-point operations are non-associative.
- **Numeric-format change.** The weights were re-quantized or re-cast (FP16 → BF16, INT8 → FP8, a new calibration pass, a different rounding mode) and the output distribution shifted even though the "model" is logically the same.

**Primary signal.** A canary eval set's pass-rate drops, or the output-embedding distribution against a baseline diverges (chapter 05 of `mod-104`), or the response-token distribution (first-token histogram, average output length) shifts without a traffic change. Secondary signals: cost per request moves (longer completions), latency moves (different kernel hot path), prefix-cache hit rate moves (different template surfaces).

**Suspect owners.** ML platform / registry (if the deployment is yours), inference-engine peer (if the kernel changed), provider (if the model is hosted and the alias moved). Chapter 03 treats this class at depth.

### 2.2 Hallucination-rate spike

The judge / eval pipeline's "grounded-answer" or "factually-correct" pass-rate drops. The model is still answering — confidently — but the answer is wrong or fabricated at a rate higher than baseline.

**Primary signal.** The quality SLI from `mod-103` chapter 02 burns. A judge-pass-rate counter (from the eval peer's pipeline) drops below the SLO threshold. A retrieval-grounded-answer check fails at a higher rate — the retrieval returned useful documents but the model ignored them.

**Suspect owners.** Application / prompt layer (prompt changed, context shrank, system message was edited), eval peer (judge drifted and is calling correct answers wrong — chapter 07), retrieval layer (the documents the model needs stopped being retrieved — overlaps with 2.6), model layer (if the model changed — overlaps with 2.1).

### 2.3 Toxic / unsafe output blast

A safety classifier, moderation API, or incident report flags an unacceptable volume of unsafe outputs — outputs that violate your product's content policy, a legal constraint, or an external contractual safety floor.

**Primary signal.** A safety-category counter spikes (toxicity, self-harm, hate, sexual, violence, policy-defined category), either from your own moderation pipeline or from the model provider's. A user-report counter spikes. An external report (press, regulator, bug bounty, red team) escalates in.

**Suspect owners.** Security-safety-compliance peer (`mod-110`), application / prompt layer (jailbreak surface exposed), model layer (model is now more jailbreakable than baseline), red-team / adversarial peer. This class is a declared-incident class even when the rate is small, because the regulatory and reputational blast radius is nonlinear in rate.

### 2.4 Refusal-rate spike

The model is refusing requests it used to answer. Users see "I can't help with that" or an empty response on inputs that have been fine for months.

**Primary signal.** The `gen_ai.response.finish_reasons = ["content_filter"]` counter rises, or a product-side "refusal" classifier's rate rises, or the safety-SLI's refusal-dimension burns. May coincide with a drop in user-engagement metrics the SRE does not directly own (session length, task-completion).

**Suspect owners.** Model layer (safety training was tightened in a version bump — overlaps with 2.1 and 2.10), application / prompt layer (a prompt change made the model nervous), eval peer (a refusal classifier drifted — overlaps with 2.9). Refusal-rate spike is the mirror image of 2.3; both must be instrumented.

### 2.5 Prompt-cache cliff

The prefix / prompt cache hit rate collapses. TTFT regresses on the same deploy (because every request now does a full prefill). Cost per request rises because input tokens are no longer de-duplicated across requests.

**Primary signal.** Prefix-cache-hit-rate gauge drops sharply. TTFT p95 regresses with no traffic shape change. Input-token consumption rises with no output-token change.

**Three canonical root causes:**

- **Prefix invalidation.** The upstream prompt template changed and a token that used to be cached is now different (a date was added, a system message was reordered, a few whitespace tokens shifted).
- **Template change.** The application rolled out a new prompt template that is structurally different enough that no cache entry matches.
- **Tokenizer change.** The tokenizer version changed (model upgrade, library pin shift) and the token IDs for the same text are now different, invalidating every cache entry.

**Suspect owners.** Application / prompt layer (template change), inference-engine peer (tokenizer / cache implementation), provider (hosted cache invalidated silently). Chapter 05 treats this class at depth.

### 2.6 Retrieval-freshness regression

A RAG surface's retrieval step is returning the wrong documents or stale documents. The model's output quality falls as a function.

**Three sub-classes:**

- **Index staleness.** The vector index has not been refreshed on cadence; new documents are missing, old-and-updated documents are serving their old version.
- **Embedding drift.** The embedding model used for retrieval has changed (version bump, numeric-format change, provider-side move), so queries and indexed documents are in different embedding spaces. Recall collapses.
- **Document corruption.** A pipeline bug (chunking, OCR, encoding) produced garbage documents; the retriever faithfully returns them.

**Primary signal.** Retrieval-grounded-answer pass-rate drops. Retrieval-score-distribution shifts (top-k scores are lower overall or more bimodal). Document age-at-retrieval histogram skews old. Ingestion-pipeline freshness SLI burns (from `mod-112`).

**Suspect owners.** Data / config pipeline (`mod-112`), ML platform (embedding model promotion), application layer (query construction). Chapter 05 treats this class at depth.

### 2.7 Agent-tool-call cascade

An agent (a surface that lets the model call tools in a loop) is misbehaving. The failure mode is cost and latency, not a wrong answer per se — the agent is burning through tool calls, provider tokens, and dollars without converging.

**Three sub-classes:**

- **Recursive tool loop.** The agent calls a tool, reads the result, decides to call the same tool with a slightly different argument, loops indefinitely (or until a depth cap).
- **Rate-limit on nested tool.** A tool the agent uses hits its own upstream rate limit; the agent interprets the error as "try again," and the call retries forever.
- **Mis-plumbed tool schema.** A tool's JSON schema changed but the agent's tool descriptor did not; the model constructs invalid arguments; the tool errors; the agent tries again.

**Primary signal.** Tool-call-depth histogram shifts to the right (depth-10 bin suddenly populated). Provider-cost per user session spikes. Tool-call error rate rises. Agent wall-clock p99 regresses sharply.

**Suspect owners.** Application / prompt layer (agent framework), inference-engine peer (if structured-output conformance is engine-side). Chapter 06 treats this class at depth.

### 2.8 Judge-model drift

The eval pipeline's judge model has changed its calibration. The model under test is unchanged; the measurement is now wrong. The quality SLI appears to burn (or appears to recover) with no underlying change.

**Primary signal.** The judge-vs-human agreement from `mod-104` chapter 05 slips. The quality SLI moves on a dimension and nothing else in the stack explains it. Rolling judge-recalibration runs disagree across windows.

**Suspect owners.** Eval peer. The SRE's job is to recognise that the signal is suspect and to hold the SLO contract without rolling the model or the application in response. Chapter 07 treats this class at depth.

### 2.9 Provider-degraded mode

A hosted provider has silently degraded — latency up, quality down, no error code, no status-page incident declared, sometimes a response that is terser or otherwise off-distribution. The provider will often confirm it after the fact; during the incident, the provider is a black box.

**Primary signal.** TTFT / TPOT regression for requests slicing on `gen_ai.system=<provider>` and `gen_ai.response.model=<id>`. Judge-pass-rate drops on the same slice. The `gen_ai.response.model` label may *change* (the provider silently bumped the version). Status page is clean or lagging.

**Suspect owners.** Vendor-risk workflow from `mod-107`, provider's support channel, in the short term the SRE's fallback routing.

### 2.10 Policy / safety regression (shared with 2.3 and 2.4)

A model version bump changed what the model will refuse *and* what it will produce, in ways that invalidate your product's safety contract. Treated as the composition of 2.1 + 2.3 + 2.4; called out separately because the handling is distinct — a declared-safety-incident path.

**Primary signal.** Either 2.3 or 2.4 fires concurrently with a `gen_ai.response.model` label change.

**Suspect owners.** Security-safety-compliance (`mod-110`) + vendor-risk (`mod-107`) + application layer.

## 3. Why these ten, in this shape

The ten were chosen with three constraints.

- **Mutually distinguishable at triage time.** The point of a taxonomy is to make the on-call's first decision cheap. Two classes should not need the same signal to tell them apart. Prompt-cache cliff (hit-rate drop, TTFT up, cost up) is distinct at a glance from silent weight drift (cost up only if output length changed, canary eval drops, TTFT may not move). Agent cascade (tool-call depth histogram) is distinct from either. The matrix in chapter 08 enforces the distinguishability.
- **Routable to a distinct peer or composition.** Every class names one or two suspect owners. The on-call does not route to "the AI team"; they route to the engine peer, the eval peer, the ML platform peer, the application layer, the provider, or a composition. The routing survives at 3 AM with incomplete information.
- **Grounded.** Every class maps to one or more entries in NIST AI RMF, AIID, OECD, or MIT AI Risk Repository. Chapter 02 draws the mapping. The purpose is that when a regulator or auditor asks "what failure class is this incident?" the answer is a term they already recognise.

The taxonomy is explicitly *not* a tree. A real incident will often belong to two classes (a provider version bump causes a refusal-rate spike *and* a hallucination-rate spike *and* a provider-degraded-mode signal). The on-call names the class they will route first; they are allowed to add more as the incident unfolds.

## 4. What this taxonomy explicitly excludes

Several things are out of scope for this module, and are flagged so the on-call does not reach for them during an incident.

- **Hardware failures.** XID errors, ECC storms, thermal throttling, bad NVLink — the DCGM surface from `mod-104` chapter 03 owns these. They can look like 2.1 or 2.9 at first glance; chapter 08's matrix separates them in step one.
- **Kernel-level performance regressions as diagnosis depth.** Chapter 03 names kernel-regression as a *sub-class* of silent drift, but diagnosing *which kernel* and *why* is the `ai-infra-performance` peer's work. The SRE's job is to route to them with the right measurement envelope, not to open Nsight.
- **Model-training-time failures.** The ML platform's problem — failed jobs, bad data, wrong hyperparameters. This module operates *after* deployment.
- **Vendor commercial failures.** Billing disputes, contract breaches, SLO-credit negotiations. `mod-107` and `mod-114` handle these; the SRE's role is to produce the signal, not to negotiate the remedy.
- **Code bugs in the gateway or application that happen to touch the LLM.** A null-pointer in the gateway is still a null-pointer, routed to the gateway owner. The taxonomy here is for failures the LLM request itself produced, not failures that happened near the LLM request.

The inclusion / exclusion line is, roughly: *is the signal that something is wrong the model's output, the retrieval's output, or a derived quality / safety signal?* If yes, it is in this module. If the signal is a 500 or a `CrashLoopBackOff`, use the classical taxonomy.

## 5. Time-to-recognition budget

For each class, the on-call should be able to confirm the class is correct within a time budget. The budget is the gap between when the SLO burn-rate alert fires and when the on-call declares "this is a class-X incident" in the incident channel. Guidance:

| Class | Budget to recognition | Why |
|---|---|---|
| 2.1 Silent output drift | 10 min | Canary-eval, embedding-drift panel, cost-per-request panel are one click each in `mod-104` |
| 2.2 Hallucination spike | 5 min | Quality SLI pane is the triggering signal; recognition is confirming the dimension |
| 2.3 Toxic / unsafe blast | 2 min | This is a declared-incident class; recognition triggers the safety playbook immediately |
| 2.4 Refusal spike | 10 min | `finish_reasons=content_filter` counter is one query; confirming it is the dominant cause takes a slice |
| 2.5 Prompt-cache cliff | 5 min | Cache-hit rate + TTFT on the same panel; recognition is near-instant |
| 2.6 Retrieval-freshness regression | 15 min | Retrieval signals are a separate pane; freshness SLI may lag |
| 2.7 Agent cascade | 10 min | Tool-call-depth histogram + cost-per-session panel |
| 2.8 Judge drift | 30 min | Judge-vs-human agreement is a lagged signal; recognition requires a calibration spot-check |
| 2.9 Provider-degraded | 20 min | Slicing on `gen_ai.system` and `gen_ai.response.model`; provider status page cross-check |
| 2.10 Policy regression | 5 min | Composite of 2.3/2.4 + model-label-change; the composite is the recognition |

The budgets are operational targets, not SLO contracts. The point is that if the on-call has not named a class after 30 minutes, something is wrong with the dashboarding (the signal is missing) or with the on-call's training (they do not know where to look). Chapter 08 tests both.

## 6. The taxonomy is a working document

The ten classes above are the catalog *at the time of writing*. The field moves; the catalog will have to move with it. A reasonable cadence:

- **Quarterly.** Review the classes against incidents from the past quarter. Did any incident fail to map cleanly? Did a new pattern emerge?
- **On major model / engine / platform change.** A new agent framework, a new structured-output surface, a new multi-modal input (vision, audio) will create new classes. Add them, remove ones that have become irrelevant.
- **On regulatory change.** The NIST AI RMF, the EU AI Act implementing acts, and sector-specific regulators will add vocabulary. Chapter 02's mapping may need updating.

The point is not to produce a perfect taxonomy — it is to produce a taxonomy *good enough* that an on-call at 3 AM reaches a confident classification quickly, and then updates the taxonomy at next business day when a gap was exposed.

## Summary

- **Classical SRE's taxonomy is necessary but insufficient.** Availability, latency, capacity, and deterministic correctness still fire, but the direct signals for AI-specific failures live elsewhere.
- **Ten classes.** Silent output drift (weights / kernel / numeric); hallucination spike; toxic / unsafe blast; refusal spike; prompt-cache cliff (prefix / template / tokenizer); retrieval-freshness regression (staleness / embedding drift / corruption); agent-tool-call cascade (loop / rate-limit / schema); judge drift; provider-degraded mode; policy regression.
- **Each class carries a primary signal, a suspect owner, and a recognition budget.** All three are what the on-call needs at 3 AM.
- **Taxonomy is a working document.** Quarterly review, update on major change, keep the regulatory mapping current.

The next chapter grounds the catalog against the authoritative external taxonomies so the vocabulary is defensible against regulator and auditor review.
