# Exercise 02 — Retrieval-Freshness SLO for RAG

**Estimated effort:** 3 hours · **Output:** a working RAG retrieval path with freshness and recall SLOs declared, a stall-induced freshness burn captured on the dashboard, an embedding-version-mismatch alarm fired and remediated, and an SLO report (≤ 600 words).

## Objective

Instrument a minimal retrieval pipeline so the function can read freshness and recall as first-class reliability signals. The deliverable is not an optimised RAG stack — it is the **SLO and signal shape** from chapter 02: per-document staleness, probe-based recall, embedding-version mismatch detection, and the burn-rate alarm that routes an ingest stall to the on-call before the user feels it.

At the end of the exercise, the function can take any retrieval pipeline the `rag-engineer` peer ships and plug it into the same signal and alert surface in minutes.

## Prerequisites

- Chapter 02 of this module.
- Prometheus + Grafana + Alertmanager available on a cluster you control (reuse the setup from exercise 01 if present).
- A vector store you can write to and query — pick any one from `pgvector`, `Qdrant`, `Weaviate`, `Milvus`, `FAISS` embedded in your service, or a managed Pinecone / Qdrant Cloud index. The exercise is substrate-independent.
- An embedding model with two callable versions. Easiest path: the provider's `-v3` and `-v4` named embedding models (e.g., OpenAI `text-embedding-3-small` and `text-embedding-3-large`, or Cohere `embed-english-v2.0` and `embed-english-v3.0`). Self-hosted BGE / E5 models work too. The point is that you have **two distinguishable embedding-model identities**.
- A small corpus — 50–200 documents is enough. Any text corpus with timestamps works (Wikipedia snapshots, an internal docs export, a synthetic set of generated statements).

## Problem statement

Work in six parts.

### Part 1 — Build the ingest pipeline

Author a small ingest service that:

- Reads a configurable corpus from object storage / a directory.
- For each document, records `source_last_modified` (from the source's timestamp).
- Chunks the document into reasonably-sized chunks (any chunker; a 500-token naive chunker is fine for the lab).
- Embeds each chunk with the **pinned embedding model** declared in config.
- Writes the chunk and vector to the index with:
  - `doc_id`
  - `source_last_modified`
  - `indexed_at`
  - `embedding_model` (string identifier of the pinned model)
  - `embedding_dim` (integer)
- Emits Prometheus metrics:
  - `ingest_run_last_success_timestamp{corpus}` (gauge)
  - `ingest_lag_seconds{corpus, stage}` (histogram)
  - `ingest_backlog_count{corpus, stage}` (gauge)
  - `ingest_failure_total{corpus, stage, reason}` (counter)
  - `document_staleness_seconds{corpus, doc_id}` (gauge; emitted on a scrape loop that reads the current time and the stored `source_last_modified` / `indexed_at`)

Commit the ingest service to Git and run it on the cluster.

### Part 2 — Build the retrieval service

Author a tiny retrieval service that:

- Accepts a query string.
- Embeds the query with the same pinned model.
- Searches the index, returns `top_k=10`.
- Optionally re-ranks (any reranker; a cross-encoder from HuggingFace will do, or skip the re-rank for simplicity).
- Emits per-stage latency histograms:
  - `retrieval_query_embedding_seconds_bucket{corpus}`
  - `retrieval_index_search_seconds_bucket{corpus, index_type}`
  - `retrieval_rerank_seconds_bucket{corpus, reranker_model}` (only if you implement re-rank)
  - `retrieval_context_assembly_seconds_bucket{corpus}`
- Emits `embedding_index_distinct_model_count{corpus}` and `embedding_index_minority_model_doc_count{corpus}` as gauges, computed from the index metadata.

### Part 3 — Declare the SLOs

Author two SLO specs under `slo/retrieval/` and commit them to Git:

- **Document-staleness SLO.** The target reflects the ingest cadence. If your ingest runs hourly, a reasonable target is: 99% of actively-retrieved documents are indexed within 2 hours of their `source_last_modified` over a rolling 7-day window.
- **Retrieval-recall SLO.** `recall@10 ≥ 0.95` over a rolling 24-hour window on the probe set from part 4.

Each SLO has:

- A PromQL definition of the SLI.
- The burn-rate alert rules (fast-burn 5m/1h and slow-burn 1h/6h — `mod-103` chapter 05's shape).
- A runbook link (author a stub if the full runbook is not ready).

### Part 4 — Author the probe set

Author 25 probes under `probes/retrieval/chat-kb.yaml`. Each probe is a (query, expected_doc_id) pair. The set must:

- Cover the major content clusters in the corpus (not all of one topic).
- Include at least two probes that depend on a document whose source has been updated recently (so a stale index misses them).

Author a probe runner that:

- Runs every 5 minutes.
- Submits each probe's query to the retrieval service.
- Records whether the expected doc appears in `top_k`.
- Emits `retrieval_recall_probe_hit_ratio{corpus, k="10"}` as a gauge.

### Part 5 — Induce and observe three failure modes

Run each of the following and capture the dashboard + alerts:

- **Ingest stall.** Stop the ingest service. Update a document the probe set depends on (change its `source_last_modified`). Observe:
  - `ingest_run_last_success_timestamp` falls behind "now."
  - The warning on time-since-last-success fires.
  - The document-staleness SLI breaches.
  - The burn-rate alarm fires.
  - The on-call hypothetically opens `rb-ingest-stalled` and resumes the service; freshness recovers.
  - Record the end-to-end timeline.

- **Embedding-version mismatch.** Switch the ingest service's pinned embedding model to the second model without a full rebuild. Ingest 5 new documents. Observe:
  - `embedding_index_distinct_model_count{corpus} == 2`.
  - The mismatch alarm fires.
  - A probe whose expected doc was re-embedded under the new model fails (recall drops).
  - Remediate by either reverting the pin + reingesting the new docs, or by running a full rebuild under the new model and updating all records.
  - Record the recall trajectory through the remediation.

- **Rerank latency regression (optional if re-rank implemented).** Switch the rerank model to a larger one (or add a synthetic 300 ms delay). Observe:
  - `retrieval_rerank_seconds_bucket` p95 shifts.
  - The TTFT-decomposition view on the dashboard shows rerank consuming a larger budget share.
  - Remediate by reducing the rerank candidate count via config (chapter 02 §6.2).

### Part 6 — Dashboard and SLO report

Author a Grafana dashboard under `grafana/dashboards/retrieval-pipeline.json` with the chapter 05 row layout for the retrieval surface:

- A pipeline-stall strip (time-since-last-success per corpus).
- Freshness SLO burn rate.
- Recall probe hit ratio.
- Embedding-index distinct-model count.
- Retrieval-latency decomposition (per stage).

Write an SLO report (≤ 600 words) covering:

- The two declared SLO targets and their rationale.
- The ingest-stall scenario timeline — stall start, alarm fire, remediation, recovery — with the observed burn rate.
- The embedding-version-mismatch scenario — alarm fire, recall drop, remediation.
- The rerank-latency scenario (if attempted).
- Open items — probes to add, runbook gaps, a signal the exercise skipped.

## Requirements

1. **Ingest and retrieval services both emit** the metric set from chapter 02 §2, with the `corpus` label.
2. **Two SLOs are declared** (staleness + recall) with burn-rate rules in Git.
3. **A probe set of ≥ 25 entries** exists and the probe runner emits `retrieval_recall_probe_hit_ratio`.
4. **An embedding-model pin** is enforced in the ingest service's config, and the mismatch detector runs.
5. **Three failure modes are induced and captured** (ingest stall, embedding mismatch, rerank-latency if feasible).
6. **A Grafana dashboard** covers the retrieval surface per chapter 05's row layout.
7. **An SLO report (≤ 600 words)** is written, with timelines and open items.
8. **Runbook stubs exist** for `rb-ingest-stalled`, `rb-recall-drift`, `rb-embedding-version-mismatch`.

## Starter guidance

- Keep the corpus **small** (≤ 200 documents). The exercise is about the signal set, not about building a large index.
- `pgvector` on an existing Postgres is the lowest-overhead substrate; Qdrant via its Docker container is nearly as light. Avoid managed services for the lab unless you already have an account; the lab is not worth the credit spend.
- For `document_staleness_seconds`, emit the metric per document only for a small, actively-retrieved subset (the probe-set documents and a bounded random sample). Emitting one Prometheus timeseries per document in a large corpus is a cardinality mistake.
- Define the `embedding_model` identifier as the full provider+model+version string. A raw model name without a provider prefix will collide when you try two providers later.
- When you induce the embedding mismatch, **do not just toggle the config and wait** — ingest at least one new document so the mismatch detector has evidence (two distinct model values in the index) rather than only an audit-log signal.
- The dashboard can reuse panels from exercise 05's unified pane; keep the retrieval-specific pane in Git so it can be merged with exercise 05 later.

## Acceptance criteria

- Ingest and retrieval services running on the cluster with the full chapter 02 signal set emitted.
- Two SLOs with burn-rate alerts declared and firing correctly on induced regressions.
- A probe set with ≥ 25 entries and a probe runner emitting the hit-ratio metric.
- Embedding-model pin enforced, mismatch detector running, and the mismatch alarm demonstrated firing.
- Three failure scenarios captured with timelines.
- A retrieval-surface Grafana dashboard committed to Git.
- An SLO report (≤ 600 words) with the rationale, scenario timelines, and open items.
- Runbook stubs for the three failure modes committed to Git.

## Stretch goals

- **Shadow rebuild drill.** Build a parallel index under a second embedding model, run probe recall against both, and perform a channel-cutover (chapter 02 §3.1 "shadow rebuild"). Measure the cutover time and the recall delta.
- **Freshness-chaos experiment.** Wire the ingest-stall scenario as a scheduled `mod-109` chaos experiment; the drill fails if the alarm does not fire within a target window.
- **Probe-set drift.** Add a `probe_set_representativeness_score` emitter that compares the probe-set embedding centroid to a rolling-window production-query centroid; alarm if the probe set stops representing production queries (chapter 04's rot pattern applied to a probe set).
- **HNSW degradation under upsert.** Perform ≥ 1,000 incremental upserts against an HNSW index and measure recall change pre- / post-upsert. Trigger a compaction / full rebuild and re-measure. Document the finding.
- **Hybrid retrieval.** Add a BM25 layer alongside the vector search and emit `retrieval_recall_probe_hit_ratio` per retrieval mode (`bm25`, `vector`, `hybrid`). Observe which mode is most robust under the induced failures.
