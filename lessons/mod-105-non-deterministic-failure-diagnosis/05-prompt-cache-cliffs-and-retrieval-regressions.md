# Prompt-Cache Cliffs and Retrieval-Freshness Regressions

## Why this chapter exists

Two failure classes from chapter 01 — prompt-cache cliff (2.5) and retrieval-freshness regression (2.6) — are what engineers who come to AI SRE from classical SRE recognise fastest and triage slowest. Fastest, because the signals look familiar: a hit-rate dropped, a cache missed, an index got stale. Slowest, because the root causes are subtle in model-specific ways — a tokenizer version shifted, an embedding model's dimensionality is different, a retrieval score distribution bimodalised in a window where the index was being rebuilt.

Both classes share a *two-signal* joint-detection property that distinguishes them from other classes. Prompt-cache cliff: cache-hit rate *and* TTFT *and* input-token cost all move together; any one alone is not enough. Retrieval-freshness regression: retrieval-score distribution *and* retrieval-grounded-answer pass-rate *and* document-age-at-retrieval histogram all move together. The chapter authors the joint signals and the triage patterns.

The signals come from `mod-104`: engine-side metrics for the cache (chapter 02), structured logs for retrieval (chapter 06), drift pipelines for embedding spaces (chapter 05).

## 1. Prompt-cache cliff (class 2.5)

### 1.1 What a prompt / prefix cache is, briefly

Serving engines — vLLM, SGLang, TRT-LLM, Triton, NVIDIA Dynamo, hosted provider stacks — accelerate common prefixes by caching the KV state produced during prefill. When two requests share a prefix (system message, few-shot examples, retrieval context), the engine reuses the cached KV for the shared tokens and only runs prefill on the suffix. The cache is addressed by the token-ID sequence; a mismatch at any position invalidates the suffix (and everything after).

The operational surface:

- **Cache-hit rate.** The fraction of prefix tokens that hit the cache across requests. Engine metric (`vllm.cache.prefix_hit_rate` or equivalent; chapter 02 of `mod-104` enumerates per-engine names).
- **TTFT.** Time-to-first-token. Highly sensitive to cache hits — a full cold prefill on a long prompt is seconds; a hit reduces the prefill to the uncached suffix only.
- **Input-token cost.** For provider-hosted models, prompt-caching discounts input tokens (provider-side pricing; e.g., "cached input tokens are billed at 10% of uncached"). A cache cliff on a hosted provider shows as a cost spike even if TTFT does not move much (hosted providers batch differently from your intuition).

A "cache cliff" is the on-call-visible event when the cache-hit rate falls from stable-high (60–90% is common for prompt-heavy surfaces) to near-zero in a short window on a specific deploy or at a specific template rollout. Every request now does a full prefill; TTFT regresses; cost regresses; the deploy looks like a latency and cost incident even though nothing is actually broken.

### 1.2 Three canonical root causes — same chapter-01 sub-classes

#### 1.2.1 Prefix invalidation

Something in the prefix changed, position-dependent. Examples:

- A date was prepended to the system message (`"Today's date: 2026-11-04"`). Every day, the cache invalidates globally.
- A tenant-ID or session-ID moved from a suffix slot into the system prefix.
- A whitespace token drifted (an extra newline, a trailing space the renderer trims differently).
- The system message added an "assistant persona" string that is per-request (`"You are talking to Alice"`).

The cliff is *gradual* if the invalidating change rolls out gradually, *sharp* if it is a full deploy.

#### 1.2.2 Template change

The prompt template was restructured. The new structure has no prefix overlap with the old; no cache entry matches. Examples:

- A system message reordered.
- A few-shot example set changed.
- The context window formatting changed (XML tags added / removed).

Template changes often look identical to prefix invalidation in metric shape; the difference is scope — a template change moves every request, a prefix invalidation often only moves requests that hit the changed path.

#### 1.2.3 Tokenizer change

The tokenizer version shifted. The *bytes* in the prompt are the same but the *token IDs* are different; the cache is keyed on IDs, so no cache entry matches.

Three ways a tokenizer shifts:

- **Model upgrade.** A new model has a new tokenizer. If the cache entries are keyed per model this does not cause a cliff on the new model, but if the cache was keyed globally or the config change was partial, cliffs appear.
- **Library pin shift.** A tokenizer library's minor version shifted and changed the tokenization of some edge cases (unicode normalization, BPE merges, special-token handling).
- **Pre-processing change.** A gateway or SDK upgraded a text pre-processing step (normalization, truncation, chat-template application).

Tokenizer-driven cliffs are the most pernicious because the source-control diff between old and new typically does not include the prompt text; it includes a dependency bump nobody connected to the cache.

### 1.3 Joint-detection shape

A prompt-cache-cliff alert should fire on the *joint* of three signals:

- **Cache-hit-rate drop** (engine-side metric). Threshold: a drop of 20+ percentage points from the baseline for the same model / surface over a 15-minute window.
- **TTFT regression** on the same slice over the same window.
- **Input-token cost regression** (provider-hosted) or **prefill-share-of-GPU-time regression** (self-hosted) on the same slice.

A single-signal alert will fire during many normal events (a traffic shape change naturally drops hit-rate; a scheduler reconfig naturally moves TTFT). The joint signal is the recognition: "all three moved together on the same deploy; this is a cliff."

### 1.4 Diagnosis in the first fifteen minutes

1. **Confirm the joint signal.** Cache-hit rate, TTFT, cost all moved together.
2. **Identify the deploy boundary.** The cliff starts at a timestamp; find the deploy, template push, or dependency bump at that timestamp. If none is visible, suspect provider-side invalidation (hosted providers invalidate caches on their schedule, usually without notification).
3. **Classify the sub-class.** Look at diff:
   - Prompt / system-message diff → prefix invalidation (1.2.1)
   - Template structural diff → template change (1.2.2)
   - Tokenizer / SDK / gateway dependency diff → tokenizer change (1.2.3)
   - No diff on your side → provider-side invalidation; escalate per `mod-107`.
4. **Measure blast radius.** Which model slices are affected; what fraction of traffic; what is the TTFT / cost delta.

By minute 15: sub-class classification + reversible-lever decision.

### 1.5 Mitigation and route

- **Prefix invalidation.** Move the varying content to a suffix slot. If the change was "add the date to the system prefix," move it to the user-turn prefix. This restores most of the cache in minutes.
- **Template change.** Roll back to the previous template, or configure the engine to warm the cache against the new template before the next traffic wave.
- **Tokenizer change.** Roll back the tokenizer / library / SDK bump. Provider-side: pin the tokenizer version.
- **Provider-side invalidation.** Route-around if possible; otherwise, absorb the cost / latency shift for the cache-rebuild window. Provider-support ticket in parallel.

Route-to: inference-engine peer owns the engine-side cache implementation; application layer owns prompt templates; vendor-risk owns provider interactions.

### 1.6 Why this class is easy to miss

The quality SLI does not fire — outputs are fine. The availability SLI does not fire — requests succeed. Only TTFT and cost fire. If the operator only watches quality and availability, the cache cliff lives for days until the finance team notices the bill. The dashboard panel for class 2.5 needs to be in the on-call's default view, not three clicks away.

## 2. Retrieval-freshness regression (class 2.6)

### 2.1 What a RAG retrieval surface looks like, briefly

A retrieval-augmented-generation (RAG) surface has three components the SRE operates alongside:

- An **ingestion pipeline** that takes source documents (knowledge base, product catalog, documents uploaded by users), chunks them, embeds them, and writes to an index.
- An **index** (vector database, inverted index, or hybrid) that answers nearest-neighbour or ranked-retrieval queries.
- A **query path** that takes the user's input, constructs a retrieval query (sometimes identical to the user text, sometimes rewritten), retrieves top-k documents, and injects them into the model's prompt.

Three freshness dimensions matter operationally:

- **Index staleness.** The index is behind the source of truth. New documents are missing; updated documents are serving their old version.
- **Embedding drift.** The embedding model used for retrieval has changed. Queries and documents are in different embedding spaces. Recall collapses.
- **Document corruption.** The ingestion pipeline produced garbage documents (chunking bug, OCR failure, encoding issue). The retriever faithfully returns garbage.

### 2.2 Index staleness

The ingestion pipeline has lagged or stopped. Operational signals:

- **Ingestion-freshness SLI** burns. The `mod-112` module (data / config pipelines) authors this SLI: for each document source, measure "how long since this source was last fully reprocessed." Threshold is domain-specific (news: minutes; product catalog: hours; policy documents: days).
- **Document-age-at-retrieval** histogram skews old. For each retrieval, the retrieved documents' timestamps feed a histogram. A shift of the p95 age upward is the signal.
- **New-document-coverage** drops. For a set of known-new documents (tracked via the ingestion pipeline's write log), measure the fraction retrievable. A drop is a staleness signal.

Root causes include ingestion-pipeline failures (DAG stopped, worker OOM, upstream source unavailable), queue backup, embedding-service outage, index-write throttling.

Owner: data / config pipeline (`mod-112`), ML platform (if the embedding service is there). The SRE's role is to catch the burn and route.

### 2.3 Embedding drift

The embedding model changed. A corpus embedded under model M₁ and queries embedded under model M₂ do not share a space; cosine similarities are near-random. This is the most catastrophic of the three freshness failures because the index is *not detectably broken* — queries return results — but recall collapses.

Three canonical patterns:

- **Query-side change.** The query-path embedding model was upgraded; the index was not re-embedded. Every query now lives in a different space from the index.
- **Index-side change.** A new embedding model was promoted on the ingestion path; the query path was not updated. New documents live in a different space from old; and from queries.
- **Numeric-format change.** The embedding model's weights or quantization changed; outputs are nearly but not quite the same; nearest-neighbour rankings shift.

Operational signals:

- **Retrieval-score-distribution shift.** Top-k scores are lower overall or more bimodal. Compared against a reference window, a material shift is the signal. Drift pipelines (`mod-104` chapter 05) catch this.
- **Retrieval-grounded-answer pass-rate** drops. For a RAG surface, the groundedness judge (from chapter 04) will burn because the retrieved documents no longer support the generated answer.
- **Query-document embedding-centroid distance** rises. For a sampled set of (query, retrieved-document) pairs, the distance distribution is monitored; a drift is the signal.

Owner: ML platform (embedding model promotion), application layer (query path pipeline), data / config pipeline.

### 2.4 Document corruption

The ingestion pipeline produced garbage. Signals:

- **Document-content-shape** anomalies. Chunk-length distribution becomes bimodal (tiny-fragment chunks or giant-blob chunks). Encoding-detection flags rise.
- **Retrieval-grounded-answer** pass-rate drops — the retrieved documents do not contain the answer because the ingestion pipeline mis-chunked them.
- **User-reported irrelevance** rises.

Root causes include OCR / parsing bugs, chunker mis-configuration, encoding issues (UTF-8 BOM, Windows-1252 misinterpretation), upstream-content changes (the source site now returns an SPA shell instead of HTML).

Owner: data / config pipeline (`mod-112`).

### 2.5 Joint-detection shape

A retrieval-regression alert should fire on the joint of two or three signals:

- **Retrieval-grounded-answer pass-rate drops** (eval-peer signal).
- **Retrieval-score-distribution shifts** (drift pipeline signal).
- **Document-age-at-retrieval histogram** shifts (for staleness) *or* **document-content-shape signals** (for corruption).

Individual signals alone will mis-classify: a score shift alone may be a traffic pattern change; a grounded-answer drop alone may be a model regression. The joint classifies the incident as retrieval-side.

### 2.6 Diagnosis in the first fifteen minutes

1. **Confirm the joint signal.** Retrieval-grounded-answer passage dropped; retrieval-score distribution shifted or document-age / corruption signals moved.
2. **Check the ingestion pipeline's freshness SLI.** If staleness, this is 2.6.a (index staleness).
3. **Check the embedding model version on both paths.** Query-path embedding model; index-build embedding model. If mismatch, this is 2.6.b (embedding drift).
4. **Spot-check retrieved documents.** If the documents look right but the model is ignoring them → this may be a hallucination issue (2.2), not retrieval. If the documents look wrong (garbage, truncated, wrong encoding) → 2.6.c (corruption).
5. **Check retrieval-rewrite step.** If the query path rewrites user input before retrieval, a rewrite-prompt change can look like a retrieval regression.

Classification by minute 15.

### 2.7 Mitigation and route

- **Staleness.** Trigger the ingestion-pipeline recovery. Route-to data-pipeline peer. If the staleness is bounded (e.g., "news surface has 2-hour lag"), note the SLO impact and let the pipeline catch up.
- **Embedding drift.** Pin the embedding model version on the path that regressed. Re-embed the index against the correct model as a scheduled catch-up. Route-to ML platform peer.
- **Corruption.** Isolate the corrupted batch; roll back the ingestion-pipeline change that caused it; re-ingest the affected sources. Route-to data-pipeline peer.

## 3. Interactions between 2.5 and 2.6 (and other classes)

### 3.1 Tokenizer change invalidates both cache and retrieval

A tokenizer change (2.5.c) commonly also breaks retrieval: the retrieval-path query embedding is produced against the tokenizer; a tokenizer shift changes the inputs to the embedding model, which changes the embeddings. A single root cause (a library bump that moved the tokenizer) produces 2.5 *and* 2.6.b signals simultaneously. The on-call should recognise the double-signal and route both to the dependency/platform owner.

### 3.2 Retrieval regression masquerades as hallucination

2.6 produces 2.2 symptoms — the model hallucinates because the retrieval did not give it the facts. Only the retrieval-score / document-age / grounded-answer signals distinguish. Chapter 04 section 4.1 listed "retrieval regression" as a root cause of hallucination; chapter 05 is where you close the loop.

### 3.3 Template change invalidates cache and is correlated with prompt-system-message changes that affect quality

A template rollout often correlates with a hallucination rate change (the new template elicits different model behaviour) and a cache cliff. Treat these as a single compound incident; mitigate both.

## 4. Dashboard shape — the cache / retrieval panel

A panel that supports both classes has roughly these signals, laid out side by side:

| Row | Signal | Covers |
|---|---|---|
| 1 | Prefix-cache hit-rate (per engine / per model) | 2.5 |
| 2 | TTFT p95 (per model) | 2.5 (cache cliff); 2.1 (kernel) |
| 3 | Input-token cost per request (per model) | 2.5; 2.1 |
| 4 | Retrieval-grounded-answer pass-rate (per RAG surface) | 2.6; 2.2 |
| 5 | Retrieval-score-distribution (p50 / p95 of top-k scores) | 2.6.b |
| 6 | Document-age-at-retrieval histogram p95 | 2.6.a |
| 7 | Ingestion-freshness SLI burn | 2.6.a |
| 8 | Deploy / template / ingestion-pipeline / dependency-bump markers | overlay for all |

The panel sits next to the quality panel (chapter 04 section 6). Both together give the on-call most of the signal they need to classify any incident involving 2.2, 2.5, or 2.6.

## 5. Peer boundary — who you route to

- **Inference-engine peer.** Engine-side cache implementation; cache-hit-rate metric correctness; engine-side tokenizer plumbing.
- **Application / prompt layer.** Prompt templates; system-message changes; retrieval-query construction; agent-tool prompts.
- **ML platform peer.** Embedding model promotion / version management; registry entries for embedding and base models.
- **Data / config pipeline peer (`mod-112`).** Ingestion-pipeline health; chunking; OCR; freshness SLIs; document-source adapters.
- **Vendor-risk workflow (`mod-107`).** Provider-side cache behaviour; provider-side tokenizer changes; provider-side embedding-service outages.

The on-call will often route to two of these for the same incident; chapter 08's triage matrix supports the common compositions.

## 6. Why these two classes are called out together

Operationally, 2.5 and 2.6 look superficially like "performance incidents" to engineers who come from classical SRE. They are not — they are *content-plumbing incidents* whose symptoms happen to include a performance regression. Treating them as performance-only means mitigating a symptom (route-around, autoscale) without fixing the root cause (the cache is still cold; the retrieval is still broken).

The chapter's thesis is: the signals belong together in the on-call's mind, because the root causes often overlap (tokenizer bumps, template rollouts, provider-side changes), and because the mitigations belong to adjacent peers.

## Summary

- **Prompt-cache cliff (2.5)** is a joint signal: cache-hit rate down, TTFT up, cost up. Three sub-classes: prefix invalidation, template change, tokenizer change.
- **Retrieval-freshness regression (2.6)** is a joint signal: retrieval-grounded-answer pass-rate down, retrieval-score shift, document-age / corruption signal. Three sub-classes: index staleness, embedding drift, document corruption.
- **Both classes present as performance or cost incidents to classical-SRE eyes; they are content-plumbing incidents with performance symptoms.** Mitigate the root cause, not the symptom.
- **Tokenizer and template changes often cause both classes at once.** Recognise the compound-incident shape.
- **Dashboard panel rows for cache-hit, TTFT, cost, grounded-answer, retrieval-score, document-age, and deploy markers support the triage.** Sit next to the quality panel.
- **Route-to.** Engine peer (cache implementation / tokenizer), application layer (templates / queries), ML platform (embedding models), data pipeline (ingestion / corruption), vendor-risk (provider-side changes).

The next chapter opens agent-tool-call cascades — a class where the failure is cost and latency, not a wrong answer, and where the recognition signal is a tool-call-depth histogram and a provider-cost blow-up.
