# Retrieval-Freshness SLOs for RAG

## Why this chapter exists

A retrieval-augmented route is only as good as its index. The model that serves it may be perfectly healthy, the prompt perfectly stable, the runtime config untouched — and the user still sees stale answers, wrong citations, or "I don't know" refusals, because the vector index the retrieval step reads from was last rebuilt three weeks ago and the source corpus has moved on.

On a classical search service this would be a search-quality problem, owned by a search-relevance team, routed to a backlog. On an AI-service surface, where the user does not see the retrieval layer directly and the model confidently paraphrases whatever it is handed, retrieval staleness becomes a quality incident that the SRE is paged on. The function's dashboard says "p95 TTFT green, success-rate 99.98%, judge score dropping slowly for three days." That slow drop is a retrieval-freshness burn, and if the function cannot read it, cannot measure it, and cannot alert on it, the incident has no owner until users complain in support tickets.

This chapter writes the reliability contract on retrieval freshness: the SLO shape, the signals, the index-version-mismatch detection, the re-rank-latency contribution to TTFT budget, and the on-call behaviour when any of them burns. The boundary is sharp. Retrieval-pipeline authoring — the choice of embedding model, the chunker, the chunk-size tuning, the re-rank model, the hybrid-search layer — is the `rag-engineer` peer's work (`mod-101` places them at level 25, consumer role). The function owns the *reliability contract on what the retrieval pipeline exposes to serving*, the same way `mod-103` owns the reliability contract on the model's serving path.

Primary references: the open-source retrieval stacks the function treats as substrate — [FAISS](https://github.com/facebookresearch/faiss), [Milvus](https://milvus.io/docs), [Weaviate](https://weaviate.io/developers/weaviate), [Qdrant](https://qdrant.tech/documentation/), [Pinecone](https://docs.pinecone.io/), [pgvector](https://github.com/pgvector/pgvector), [Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/semantic-search.html); re-rank models from [Cohere Rerank](https://docs.cohere.com/docs/overview-rerank) and [BGE / Jina rerankers](https://huggingface.co/BAAI); the general retrieval-evaluation literature at [BEIR](https://github.com/beir-cellar/beir) and [MTEB](https://huggingface.co/spaces/mteb/leaderboard).

## 1. The retrieval-pipeline model

Before the SLO shape, the function needs a shared picture of the pipeline it is measuring. A canonical RAG retrieval path has five stages:

```
request
  -> query embedder        (embeds the user query with embedding-model-V)
  -> vector index search   (top_k by cosine against index-V)
  -> re-ranker             (reorders top_k' with a cross-encoder)
  -> context assembler     (slots retrieved chunks into the prompt template)
  -> model call            (the serving tier from mod-103 / mod-104)
```

Behind this live-path, two offline pipelines feed it:

- **Document ingestion** — source documents land in a staging area, are chunked, the chunks are embedded with embedding-model-V and written to index-V.
- **Index build** — the vector index is (re)built from the embedded chunks. For in-memory indexes (FAISS flat, HNSW) this may be a snapshot build; for managed services (Pinecone, Weaviate) this is incremental writes to a hosted index.

Everything the function measures is a property of one of these stages or the relation between them. The function does *not* own the stages' internals (the `rag-engineer` peer does); the function owns the SLOs that read their health.

## 2. The retrieval-freshness SLO

Freshness is the primary reliability signal the function owns. The core question is: **how stale may the content of the retrieval index be before the fleet is out-of-SLO?**

### 2.1 Why freshness is a fleet SLO, not a product-manager preference

Treating freshness as a product preference ("we'd like docs to be fresh") puts no operational pressure on the ingest pipeline. Treating freshness as a reliability SLO — error-budgeted, burn-rate-alarmed, on-call-paged — puts the ingest pipeline on the same operational footing as the serving tier. That is the right footing: an ingest pipeline that stalls silently is as much of an incident as a model deployment that stalls silently.

### 2.2 The freshness SLI

Define `document_staleness_seconds` as the age of a document, measured as:

```
staleness(doc) = indexed_at(doc) - source_last_modified(doc)
```

That is: the lag between when the source last changed and when the index last reflected that change. For a document that is up-to-date, the staleness is zero (or small, equal to the ingest latency). For a document whose source has changed but the index has not been rebuilt, the staleness grows with time.

The fleet-level SLI is the distribution of staleness over the *actively-retrieved* documents (weighted by retrieval frequency, not the raw document count — stale documents that are never retrieved do not affect users). Prometheus-friendly shape:

```promql
# per-document staleness, exposed by the ingest pipeline
document_staleness_seconds{corpus, doc_id}

# fleet-level freshness SLI, weighted by retrieval frequency
# (join with a retrieval_count_by_doc counter emitted by the retrieval service)
sum by (corpus) (
  document_staleness_seconds * on(doc_id) retrieval_count_by_doc_total
) / sum by (corpus) (retrieval_count_by_doc_total)
```

### 2.3 The freshness SLO shape

A representative SLO for a fleet whose corpus changes daily:

> **Document-staleness SLO** — 99% of documents retrieved over a rolling 7-day window are indexed within 24 hours of their source-last-modified timestamp.

The exact numbers are per-corpus: a product-knowledge-base corpus that changes weekly can run on a 168-hour staleness bound; a pricing-and-availability corpus may need a 15-minute bound. The function writes one SLO per corpus; the `rag-engineer` peer owns the target choice with the product owner.

Burn-rate alarms (`mod-103` chapter 05) apply unchanged. A 14.4× burn over 1 hour on the staleness SLO fires a page; the on-call reads the `rb-ingest-stalled` runbook.

### 2.4 The retrieval-recall SLO

Staleness measures *whether* the index contains the right document. Recall measures *whether the retrieval step finds the right document*. Both are reliability signals; both have SLOs.

Define a **retrieval-recall probe set** — a curated set of (query, expected-doc-id) pairs maintained by the `rag-engineer` peer, run against the live retrieval stack on a cadence (every 5 minutes is a typical sampling rate — <!-- needs-research: cite a retrieval-probe cadence recommendation from a published RAG-reliability reference; current statement is reasoned, not cited -->). The SLI is the fraction of probes where the expected document appears in the `top_k` retrieved set:

```promql
retrieval_recall_probe_hit_ratio{corpus, k}
```

The SLO:

> **Retrieval recall SLO** — `recall@10` over the probe set stays above a pre-committed floor (typical: 0.95) over a rolling 24-hour window.

Recall drift — the probe-hit ratio drops without a deploy — is a reliability incident. The chapter's §4 writes the on-call response.

## 3. Index-rebuild cadence

The ingest pipeline decides when to rebuild the index. The function does not own that decision. The function owns the SLO that constrains it and the alert path when the pipeline falls behind.

### 3.1 Rebuild styles

The function sees three operational shapes:

- **Full rebuild.** The index is dropped and rebuilt from the source corpus on a cadence (nightly, weekly). Simple, deterministic, atomic — but the full-rebuild window is a staleness floor (a nightly rebuild means documents can be up to 24h stale even in the healthy case).
- **Incremental upsert.** New / changed documents are embedded and written to the existing index as they arrive. Managed services (Pinecone, Weaviate, Qdrant) default to this; local indexes (FAISS HNSW, Elastic BM25) support it with some operator care.
- **Shadow rebuild.** A new index is built alongside the current one; a cutover swap promotes the shadow once it passes validation. This is the retrieval-side analogue of blue-green (`mod-111` chapter 04) and is the correct shape for embedding-model upgrades.

The function reads each style through the same freshness SLO. A nightly full-rebuild with a 24h SLO is consistent; a 1h SLO against a nightly-rebuild pipeline is a misaligned contract that will burn the budget every single day and should be caught at SLO-setting time.

### 3.2 The ingest pipeline as a reliability dependency

The ingest pipeline is a *dependency* the serving tier reads through the index. It has its own health signals the function wires into the observability stack (`mod-104`):

- `ingest_lag_seconds{corpus, stage}` — lag at each stage (staging-arrival, embedding, index-write). Prometheus histogram.
- `ingest_backlog_count{corpus, stage}` — queue depth at each stage.
- `ingest_failure_total{corpus, stage, reason}` — counter per failure type.
- `ingest_run_last_success_timestamp{corpus}` — the Unix timestamp of the last successful run; a gauge whose current-time-minus-value is the time-since-last-success.

The function alerts on:

- Backlog above a pre-committed threshold.
- `time() - ingest_run_last_success_timestamp` above a threshold (ingest has not completed successfully recently).
- `ingest_failure_total` rate above a threshold.

These are not the same as freshness-SLO burn — a healthy pipeline that has been down for 15 minutes may not yet have breached freshness, but it is a leading indicator that freshness is about to burn. The leading indicators go to a warning channel; the SLO burn-rate alarm goes to the page.

### 3.3 Pipeline-stall runbook

The `rb-ingest-stalled` runbook is a first-class operation:

1. Identify the stalled stage from `ingest_backlog_count` and `ingest_lag_seconds`.
2. Check the stage's upstream dependencies (object store availability, embedding-provider health, index-service availability).
3. If the embedding provider is degraded, fall back to the pinned secondary embedding provider per `mod-107` chapter 04 — provided the index was built with the secondary model, or a hybrid path is available.
4. Resume the stage; measure the drain time; track freshness-SLO burn through the drain.
5. If freshness-SLO burn exceeds the fast-burn threshold, treat as a serving incident per `mod-108` — the retrieval failure is user-affecting even without a 5xx.

The runbook is drilled in the `mod-109` chaos program (freshness-chaos is a legitimate experiment shape).

## 4. Retrieval-recall drift as a reliability incident

Recall drift is the quiet failure mode where the probe hit-ratio falls over time with no deploy to blame. The function treats it as a reliability incident — paged, investigated, root-caused, with a post-incident learning loop.

### 4.1 Why recall drifts without a deploy

Without any code or config change, probe-hit ratio can drift because:

- **Corpus content changed.** New documents pushed the probe's expected document out of the `top_k` (a new document is closer to the query's embedding than the expected one).
- **Query distribution changed.** The production query distribution has drifted; the probe set no longer represents what users are actually asking.
- **Embedding-provider drift.** A hosted embedding model has been silently updated (`mod-107` chapter 04); the embeddings of new ingested documents are no longer directly comparable to the embeddings of old ones.
- **Index degradation.** HNSW graphs can degrade under sustained incremental upserts; recall-at-k on an HNSW index that has not been periodically recompacted is known to drift. <!-- needs-research: cite a specific source on HNSW quality degradation under heavy upsert load; the pattern is widely reported but the function should cite a canonical reference (e.g. Malkov & Yashunin 2018, or a specific blog post from Weaviate / Qdrant engineers) -->

Each cause has a different remediation. The on-call's job is not to pick a cause from memory; the job is to walk the diagnostic in the `rb-recall-drift` runbook.

### 4.2 The recall-drift diagnostic

```
page: retrieval_recall_probe_hit_ratio burning
-> which probes are failing?                       (per-probe breakdown)
-> is the expected doc indexed at all?             (query the index for its doc_id)
   -> no: ingest-side problem; route to rb-ingest-stalled
   -> yes: continue
-> is the expected doc's embedding fresh?          (compare indexed_at to source_last_modified)
   -> no: ingest-side problem
   -> yes: continue
-> did the embedding-model version change?         (check embedding-pipeline audit log)
   -> yes: the index is mixed; route to rb-embedding-version-mismatch (§5)
   -> no: continue
-> is the production query distribution different  (compare rolling-window query embeddings
   from the probe set?                              against the probe set's centroid)
   -> yes: probe set is stale; route to rag-engineer peer with the evidence
   -> no: continue
-> are there recent incremental upserts at volume? (check ingest_backlog and upsert rate)
   -> yes: HNSW degradation suspected; schedule compaction / full rebuild
   -> no: open a joint investigation with the rag-engineer peer
```

The runbook branches are each a short operation. The function's job is to drive the diagnostic to the correct remediation, not to re-engineer the retrieval pipeline.

### 4.3 The probe set is the function's artifact

The probe set itself is a shared surface. The `rag-engineer` peer authors the probes (the content). The function:

- Operates the probe runner (schedules it, emits its metrics, pages on burn).
- Treats the probe set as versioned (its hash is tagged on every probe emission; a probe-set change is a config-change candidate per chapter 01).
- Maintains a probe-set-refresh SLO — the probe set itself can rot (chapter 04's eval-set rot pattern applies here identically), and the function's SLO target forces refresh cadence.

## 5. Embedding-index-version-mismatch detection

The most expensive retrieval failure the function sees is an **embedding-version mismatch** — part of the index was built with embedding-model-A and part with embedding-model-B, and vector distances between A-vectors and B-vectors are not meaningful. Recall collapses on exactly the documents that were re-embedded without a full rebuild.

### 5.1 How mismatch happens

- A hosted embedding model is silently upgraded by the provider; new ingests pick up the new model; old vectors stay under the old model.
- A library upgrade in the ingest pipeline switches the embedding call's default model.
- A failover to a secondary embedding provider happens mid-ingest (`mod-107` chapter 04) and is not followed by a rebuild.
- A dimensionality change (768 → 1024) is pushed through without updating the index schema; the write fails loudly for some stacks and silently for others.

### 5.2 Mismatch detection

The function requires every vector written to the index to be tagged with:

- `embedding_model` — the model identifier (provider, model name, version).
- `embedding_model_pin` — a content hash of the model if the model is self-hosted, or the provider's version string if hosted.
- `embedding_dim` — the vector's dimension.
- `ingested_at` — the timestamp of the ingest.

The mismatch detector is a periodic check that emits a gauge per index:

```promql
embedding_index_distinct_model_count{corpus}      # should be 1 in the healthy case
embedding_index_distinct_dim_count{corpus}        # should be 1
embedding_index_minority_model_doc_count{corpus}  # should be 0 (or small, during a migration)
```

Any value above 1 (or above the migration-tolerance cap) fires a page. The page's runbook is `rb-embedding-version-mismatch`:

1. Confirm the mismatch (which models, how many documents each).
2. Identify the ingest event that introduced the second model (provider auto-upgrade, library default change, failover, misconfiguration).
3. Decide: rebuild the full index with the new model (if the new model is validated), or quarantine the new-model vectors and retry under the pinned model.
4. Track the rebuild to completion against the freshness SLO.

### 5.3 The embedding-model pin

The function enforces an **embedding-model pin** as config (chapter 01 §4) — the ingest pipeline's config declares the exact embedding model it writes with, and a config drift fires the alarm before any vectors land. Hosted providers that do not expose a pinnable version are a vendor-risk signal (`mod-107` chapter 06) and the function escalates to the vendor contract.

## 6. Re-rank latency and the TTFT budget

The re-ranker is the retrieval stage most commonly overlooked in the latency budget. A cross-encoder re-rank over 100 candidates can add 200–500ms to the request path (<!-- needs-research: cite representative re-rank latency numbers from Cohere Rerank, BGE-reranker, or Jina-reranker benchmark pages; the specific range depends on model size and hardware -->), which is a large fraction of a tight TTFT SLO.

### 6.1 The latency decomposition

The function instruments the retrieval path such that the TTFT budget is decomposable:

```
TTFT = query_embedding_latency
     + index_search_latency
     + rerank_latency
     + context_assembly_latency
     + model_call_first_token_latency
```

Each component emits a histogram metric:

```promql
retrieval_query_embedding_seconds_bucket{corpus}
retrieval_index_search_seconds_bucket{corpus, index_type}
retrieval_rerank_seconds_bucket{corpus, reranker_model}
retrieval_context_assembly_seconds_bucket{corpus}
model_first_token_seconds_bucket{route, model_version}
```

The SLO on TTFT (`mod-103` chapter 01) is decomposed into per-component sub-budgets, and the sub-budgets are the engineering levers. When TTFT burns, the per-component view says which stage is responsible — the function does not stare at a single aggregate number and guess.

### 6.2 Re-rank as a tunable control

The re-rank stage is controllable at the config layer (chapter 01):

- **Candidate count.** How many of the `top_k` are re-ranked. Reducing from 100 to 30 cuts re-rank latency ~3× at a cost in recall.
- **Re-rank model size.** A smaller cross-encoder is faster at a cost in quality.
- **Re-rank skip threshold.** If the initial retrieval score for the top-1 is above a threshold, skip re-rank entirely for that query.

Each is a config value under chapter 01's CRD. A re-rank-latency burn triggers a well-defined set of mitigations the function can apply through the config-change release path. The function does not re-invent the re-ranker; the function tunes the exposed knobs under a release-gate contract.

### 6.3 Cost of re-rank

The re-rank layer has a cost dimension. A re-rank call on a hosted provider (Cohere Rerank, Voyage) is a billable operation per query. The function's cost SLO (`mod-103` chapter 04, `mod-106`) captures it as a line item in `cost_per_request`:

```promql
cost_per_request_usd{route, cost_source="rerank"}
```

A re-rank that is 50% of the request's cost is a deployed configuration the function can read and challenge.

## 7. The retrieval-pipeline failure modes list

The function keeps a one-page reference of the retrieval-pipeline failure modes it has seen in production, each tagged with its signal, SLO, and runbook. The initial list:

| Failure mode | Primary signal | SLO affected | Runbook |
|---|---|---|---|
| Ingest pipeline stalled | `ingest_run_last_success_timestamp` age | Freshness | `rb-ingest-stalled` |
| Embedding provider latency spike | `retrieval_query_embedding_seconds_bucket` p99 | TTFT | `rb-embedding-provider-degraded` |
| Index service unavailable | `retrieval_index_search_seconds_bucket` error-rate / timeout | TTFT, availability | `rb-index-service-down` |
| Embedding-model version mismatch | `embedding_index_distinct_model_count > 1` | Recall | `rb-embedding-version-mismatch` |
| Recall drift | `retrieval_recall_probe_hit_ratio` | Recall | `rb-recall-drift` |
| HNSW degradation under upsert | Recall drift with no other signal moving | Recall | `rb-recall-drift` (branch) |
| Re-rank latency regression | `retrieval_rerank_seconds_bucket` p95 | TTFT | `rb-rerank-latency` |
| Context-window over-fill | Prompt-token-count p95 approaches model limit | Quality, cost | `rb-context-overfill` |
| Probe-set rot | Probe hit-ratio plateau over months | Recall (signal validity) | Joint with `rag-engineer`; see chapter 04 for the rot pattern |

Each entry is backed by a Grafana panel in the single-pane dashboard (chapter 05) and a runbook under version control.

## 8. The retrieval-reliability checklist

Before accepting the retrieval reliability contract for a new RAG route:

1. **Freshness SLO defined** per corpus, with the target negotiated against the ingest-rebuild cadence.
2. **Document-staleness SLI emitted** — per-document staleness is instrumented; the fleet SLI is weighted by retrieval frequency.
3. **Retrieval-recall probe set exists** and the probe runner emits `retrieval_recall_probe_hit_ratio` at the committed cadence.
4. **Embedding-model pin in config** per chapter 01 §4; the mismatch detector is wired.
5. **Latency decomposition in place** — each retrieval stage emits a histogram; TTFT budget is per-component.
6. **Ingest-pipeline SLI set emitted** — lag, backlog, last-success, failure rate; alerts wired.
7. **Four runbooks exist** — `rb-ingest-stalled`, `rb-recall-drift`, `rb-embedding-version-mismatch`, `rb-rerank-latency` — and the ingest-stalled and embedding-version-mismatch runbooks have been drilled at least once.
8. **Peer handoff with `rag-engineer`** signed — probe-set authoring at the peer, probe-runner operation at the function, SLO targets jointly owned.
9. **Retrieval-pipeline changes go through chapter 01's progressive-delivery path** — an embedding-model upgrade or a re-rank-model swap is a candidate the release gate evaluates.

A RAG route served without these items in place is served without a retrieval reliability contract; the function does not represent it as covered.

## Summary

- Retrieval freshness, recall, and latency are reliability signals the function owns on a RAG route — the serving-side SLOs from `mod-103` are necessary but not sufficient.
- The **document-staleness SLO** constrains the ingest pipeline; the **retrieval-recall SLO** constrains the live retrieval stack; both burn independently and both page the on-call.
- **Embedding-index-version-mismatch** is the most expensive retrieval-side failure — the function enforces an embedding-model pin as config, tags every vector with its embedding-model identifier, and runs a mismatch detector that fires an alarm on any drift.
- **Re-rank latency** is a decomposable component of the TTFT budget. The function instruments it per-stage and treats the re-rank's candidate count, model size, and skip threshold as config-layer controls under the chapter 01 release path.
- The probe set that validates recall is itself a versioned artifact with its own rot pattern — chapter 04 makes the rot discipline explicit; this chapter establishes the operational dependency.
- The function does **not** own retrieval-pipeline authoring. The `rag-engineer` peer owns chunking, embedding selection, re-rank choice, and probe-set content. The function owns the reliability contract on what the pipeline exposes to serving and the on-call response when those signals burn.
- The nine-point checklist governs a route's acceptance into the function's retrieval-reliability contract.

Chapter 03 moves from the retrieval surface to the feature-store surface, where the handoff is to the ML platform peer and the drift signals look similar but route differently.
