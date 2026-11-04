# Unified Pipeline Observability in One Pane

## Why this chapter exists

Chapters 01–04 named four new reliability surfaces the AI SRE function owns: prompts and configs, retrieval, feature and metadata stores, and eval sets. Each has its own SLOs, its own signals, its own runbooks. In the wrong hands, that becomes four disconnected dashboards — a prompt dashboard in a vendor UI, a retrieval dashboard in Pinecone / Weaviate's console, a feature dashboard in the ML platform's console, an eval dashboard in LangSmith / Weights & Biases. On-call sees a page, opens five tabs, correlates by hand, and spends the first ten minutes of the incident just assembling the picture.

That is not the function's observability contract. The contract is: **every pipeline signal flows into the same Prometheus / Grafana / OTel stack the serving tier uses** (`mod-104`), **under the same alert-rule discipline** (`mod-104` chapter 05), **routed by the same Alertmanager** (`mod-108`), **so an incident surfaces in a single pane**. The on-call reads one dashboard, correlates four signals against the serving SLIs on the same time axis, and reaches a hypothesis in minutes, not tabs.

This chapter writes that unification. It defines the common signal schema, the Grafana layout, the alert-rule discipline, and the operational doctrine for what belongs in the pane and what does not. The chapter deliberately does not redefine observability primitives — those are in `mod-104`. It extends `mod-104` with the specific signal set from the four pipeline surfaces.

Primary references: [Prometheus best practices](https://prometheus.io/docs/practices/naming/), [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/), [Grafana](https://grafana.com/docs/grafana/latest/), [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/), the [OpenTelemetry `gen_ai.*` semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) that `mod-104` chapter 01 already builds on.

## 1. The unification principle

Four principles govern the pane. Each is a decision the function makes once and holds.

### 1.1 One metric store, one trace store, one log store

- **Metrics** in Prometheus (or a Prometheus-compatible TSDB — Mimir, Thanos, VictoriaMetrics). No vendor's bespoke metric store except as a source that feeds Prometheus via an exporter.
- **Traces** in the OTel-ingesting backend of the fleet's choice (Tempo, Jaeger, a commercial platform). The `gen_ai.*` semantic conventions are the vocabulary; pipeline spans extend them.
- **Logs** in the fleet's structured-log store (Loki, Cloud Logging) with trace-ID correlation.

A pipeline signal that lives only inside a vendor UI is not observable in the fleet. The function pulls it out through an exporter or does not use it as a reliability signal.

### 1.2 One label vocabulary

Every pipeline metric carries the labels that let it join against the serving SLIs. The `mod-104` chapter 01 / 02 serving labels are the baseline:

- `route` — the route the signal attributes to.
- `tenant` — the tenant (or `_all`).
- `cohort` — the cohort (or `_all`).
- `model_version`, `prompt_hash`, `config_hash` — from chapter 01.
- `corpus`, `eval_set`, `feature_set` — the pipeline-side identifiers from chapters 02, 03, 04.
- `version` — stable / canary / shadow (from `mod-111`).

A metric that omits a required label is a metric that does not join. The function reviews the label discipline at metric-onboarding time.

### 1.3 One alert routing

Pipeline burn-rate alarms route through the same Alertmanager as the serving SLO burn-rate alarms. The paging path from `mod-108` applies unchanged. A `document_staleness_slo_burn` fires into the same on-call queue as a `ttft_slo_burn`; the on-call does not discover pipeline incidents from a separate queue.

### 1.4 One incident surface

When an incident opens, the single pane is the first artifact the incident commander looks at. The pane is maintained as a function asset the same way the serving dashboard is — in Git, review-gated, version-controlled.

## 2. The signal taxonomy

The four surfaces contribute signal families that the pane organises on a single time axis against the serving SLIs.

### 2.1 Prompt / config surface (chapter 01)

Metrics:

```
prompt_registry_lookup_seconds_bucket{route, outcome}
prompt_registry_lookup_total{route, outcome}
prompt_cache_hit_ratio{route}
config_admission_rejection_total{route, reason}
rollout_controller_canary_state{route, state}         # gauge: 0=idle, 1=progressing, 2=succeeded, 3=rolled-back
rollout_controller_rollback_total{route, trigger}
```

Trace spans — `prompt.resolve` (span), `prompt.render` (span), both parented under the request span.

Logs — audit log emission from the registry, with `prompt_hash`, `author`, `reviewers`, correlated by trace-ID.

### 2.2 Retrieval surface (chapter 02)

Metrics:

```
document_staleness_seconds{corpus, doc_id}
ingest_lag_seconds{corpus, stage}
ingest_backlog_count{corpus, stage}
ingest_run_last_success_timestamp{corpus}
retrieval_recall_probe_hit_ratio{corpus, k}
embedding_index_distinct_model_count{corpus}
embedding_index_minority_model_doc_count{corpus}
retrieval_query_embedding_seconds_bucket{corpus}
retrieval_index_search_seconds_bucket{corpus, index_type}
retrieval_rerank_seconds_bucket{corpus, reranker_model}
retrieval_context_assembly_seconds_bucket{corpus}
```

Trace spans — `retrieval.embed`, `retrieval.search`, `retrieval.rerank`, `retrieval.assemble`, all parented under the request span. Each span has the `corpus` and `index_version` attributes.

Logs — ingest-pipeline run logs, with `corpus`, `stage`, `run_id`.

### 2.3 Feature / metadata surface (chapter 03)

Metrics:

```
feature_lookup_seconds_bucket{feature_set, feature}
feature_lookup_null_ratio{feature_set, feature}
feature_pipeline_run_last_success_timestamp{feature_set, pipeline}
feature_pipeline_lag_seconds{feature_set, pipeline, stage}
feature_pipeline_backlog{feature_set, pipeline, stage}
feature_parity_match_total{feature_set, feature}
feature_parity_mismatch_total{feature_set, feature, mismatch_type}
feature_drift_psi{feature_set, feature}
metadata_store_lookup_seconds_bucket{store, operation}
metadata_store_error_total{store, operation, reason}
```

Trace spans — `feature.lookup` (per feature set, with `feature_set` and `feature_set_version` attributes), `metadata.resolve` (per store).

### 2.4 Eval surface (chapter 04)

Metrics:

```
eval_set_representativeness_score{eval_set, eval_hash}
eval_set_discriminator_score{eval_set, eval_hash}
eval_set_ceiling_fraction{eval_set, eval_hash}
eval_set_cohort_coverage{eval_set, cohort}
eval_contamination_score{eval_set, eval_hash, candidate}
eval_run_duration_seconds_bucket{eval_set, candidate}
eval_run_failure_total{eval_set, candidate, reason}
eval_run_last_success_timestamp{eval_set, candidate}
judge_latency_seconds_bucket{eval_set, judge}
judge_availability_total{eval_set, judge}
```

Traces — eval-run spans emitted by the eval pipeline, tagged with `eval_set`, `eval_hash`, `candidate`, and `judge`.

### 2.5 Joining the surfaces

The pipeline metrics share labels with the serving metrics on `route`, `cohort`, `tenant`, `model_version`, `prompt_hash`, `config_hash`, `version`. A PromQL query like "show the retrieval-recall probe hit ratio on the chat route for the canary channel" resolves without a schema translation step:

```promql
retrieval_recall_probe_hit_ratio{corpus="chat-kb", k="10"}
  * on(corpus) group_right()
  label_replace(
    sum by (route) (rate(request_total{route="chat", version="canary"}[5m])),
    "corpus", "chat-kb", "route", "chat"
  )
```

The join is not elegant, but it is possible because the two metrics live in the same TSDB with consistent labels. The function's label discipline is what makes the join possible at all.

## 3. The Grafana pane

The single pane is laid out so the on-call reads it top-to-bottom during an incident and reaches a hypothesis in one scroll.

### 3.1 Row 1 — the serving SLI strip

- Request rate by route and version.
- TTFT p50 / p95 / p99 by route and version.
- Success rate by route and version.
- Judge score by route and version.
- Cost per request by route and version.

This row is the serving dashboard from `mod-104` chapter 02, pulled in unchanged. It establishes what the user is experiencing.

### 3.2 Row 2 — the SLO burn strip

- Current burn rate per fleet-level SLO (TTFT, success, judge, cost, availability).
- Pipeline-SLO burn rates (freshness, recall, feature-freshness, feature-parity, eval-pipeline-duration).

Burn rate normalises the view. A 1.0× rate is "spending budget at the sustainable long-run rate"; a 14.4× is the fast-burn page threshold from `mod-103`. The pane shows which SLO is burning alongside which.

### 3.3 Row 3 — the pipeline-stall strip

A dedicated row for "when did each pipeline last complete successfully?" A single stat panel per pipeline shows `time() - ingest_run_last_success_timestamp`, `time() - feature_pipeline_run_last_success_timestamp`, `time() - eval_run_last_success_timestamp`. Each panel is colour-coded against its per-pipeline threshold.

Why this row exists: pipeline stalls are among the most common root causes of pipeline-SLO burns, and the "age of last success" is the first thing the on-call looks at when a freshness alarm fires. The row's job is to make that lookup zero-click.

### 3.4 Row 4 — the retrieval-pipeline detail

- Retrieval-recall probe hit ratio per corpus.
- Ingest-lag per corpus and stage.
- Embedding-index model-count per corpus.
- Retrieval-latency decomposition (query-embed, index-search, re-rank, context-assembly) per corpus.
- Document staleness distribution (p50 / p95) per corpus.

A route-filter at the top of the dashboard applies a `route="chat"` to the whole pane; the retrieval row then shows the corpora that route reads from.

### 3.5 Row 5 — the feature-pipeline detail

- Feature-lookup latency and null-ratio per feature set.
- Feature-pipeline lag and backlog per feature set and stage.
- Parity-match ratio per feature set with breakdown by mismatch type.
- Drift PSI per monitored feature.

### 3.6 Row 6 — the eval-pipeline and registry detail

- Representativeness, discriminator, ceiling for each gating eval set.
- Eval-run duration and judge-availability.
- Prompt-registry lookup latency and cache hit ratio.
- Config-admission rejection rate.
- Rollout-controller state per route and recent rollback count.

### 3.7 Row 7 — recent change events

An annotation / event row showing recent prompt-registry writes, config CRD applies, rollout promotions / rollbacks, eval-set revisions. When the pane shows a regression at a specific time, the row answers "what changed near that time?" without the on-call opening Git.

Events are emitted into Prometheus via the standard practice of Grafana annotations backed by a timestamped metric:

```promql
# a gauge whose value changes on every event, labelled with the change kind
pipeline_change_event{kind, route, hash, actor}
```

Alternatively via Loki's log stream with annotation queries.

### 3.8 One pane, multiple views

The pane is one dashboard. The `route` filter is the primary pivot. Per-tenant and per-cohort views are variants reached by the standard Grafana variable mechanism; they are not separate dashboards (and in particular, a separate dashboard per team is the drift pattern the pane is explicitly fighting).

## 4. The alert-rule discipline

### 4.1 Alert rules in Git

Every pipeline-SLO alert rule is Git-tracked under the same convention as the serving-SLO rules (`mod-104` chapter 05). A representative rule:

```yaml
groups:
  - name: pipeline-retrieval
    rules:
      - alert: DocumentStalenessFastBurn
        expr: |
          slo:document_staleness:burn_rate_5m{corpus="chat-kb"} > 14.4
          and
          slo:document_staleness:burn_rate_1h{corpus="chat-kb"} > 14.4
        for: 2m
        labels:
          severity: page
          slo: document-staleness
          surface: retrieval
          team: ai-sre
        annotations:
          summary: Document staleness burning fast on chat-kb
          runbook: https://runbooks/rb-ingest-stalled
          dashboard: https://grafana/d/pipelines?var-route=chat

      - alert: EmbeddingIndexModelMismatch
        expr: embedding_index_distinct_model_count{corpus="chat-kb"} > 1
        for: 5m
        labels:
          severity: page
          surface: retrieval
          team: ai-sre
        annotations:
          summary: Embedding-index has multiple embedding-model versions
          runbook: https://runbooks/rb-embedding-version-mismatch
```

Each rule links to its runbook and the pane variant that filters to its context. The function does not accept a rule without a runbook link.

### 4.2 The multiwindow multi-burn-rate shape

The `mod-103` chapter 05 practice applies unchanged on pipeline SLOs. Fast-burn uses 5m / 1h windows at a 14.4× threshold; slow-burn uses 1h / 6h at a 6× threshold. The function does not re-derive the thresholds per surface; the SLO-burn mathematics is independent of the SLO's domain.

### 4.3 The anti-noise discipline

- **No alert without a runbook.** The runbook must be more than a stub; a page without a diagnostic is a page the on-call cannot resolve.
- **No alert without an owner.** The function owns pipeline alerts; the ML-platform peer, the rag-engineer peer, the eval peer own the fix. The alert labels name the escalation path.
- **Noisy alerts are tuned, not ignored.** An alert that fires three times in a week without a real incident is reviewed at the weekly operational review (see §6). Thresholds are widened, dwell is added, or the alert is demoted.
- **Routing per surface.** `surface: retrieval`, `surface: feature`, `surface: eval` labels enable per-surface escalation; the rag-engineer peer can subscribe to retrieval-surface alarms without noise from the feature surface.

### 4.4 Change-correlation annotations

Alertmanager annotations include a link to the "recent change events" row of the pane filtered to the time window of the alert. The on-call reads the annotation, lands on the row, and sees what shipped in the last 30 minutes. This is the single most consistent accelerant of pipeline incident response; the function invests in it.

## 5. Trace correlation across pipelines

A trace that starts at the user's request should propagate through every pipeline it touches. Concretely:

```
POST /chat
├─ prompt.resolve   (chapter 01)
├─ retrieval.embed  (chapter 02)
├─ retrieval.search (chapter 02)
├─ retrieval.rerank (chapter 02)
├─ feature.lookup   (chapter 03)
├─ model.call       (mod-104 chapter 01)
└─ response.emit
```

The function uses OTel's context propagation to ensure every pipeline-span is parented under the request-span. A dropped-context pipeline (one whose trace starts over inside the pipeline) is an onboarding gap the function fixes.

Eval-run traces are a separate class — they are not in the user's request path — but they are correlated to the candidate they evaluated by `candidate_id`, which joins to the release-gate verdict's `candidate_id`. A later investigation can walk from a serving response to the eval run that signed off its (model, prompt, config) triple.

## 6. Operational doctrine

### 6.1 The pane is reviewed weekly

The function runs a weekly operational review: the pane is the agenda. The review:

- Confirms every SLO's current burn.
- Acknowledges each alarm fired in the last week with the incident write-up.
- Reviews the "recent change events" row for events whose impact was not caught by alarms.
- Reviews the noisy-alert list and schedules tuning.
- Reviews the pipeline-stall strip for drift patterns.

The ML-platform peer, the rag-engineer peer, and the eval peer rotate as attendees when their surface is on the agenda.

### 6.2 The pane is version-controlled

The dashboard JSON is stored in Git under `grafana/dashboards/pipeline-observability.json`. A change to the pane is a PR. The function does not hand-edit the dashboard in the Grafana UI except for exploration; changes land through CI.

### 6.3 The pane is the incident's first artifact

An incident opening against any SLO this module covers opens the pane first. The incident commander times to hypothesis from the pane; a time-to-hypothesis over the fleet's target is a signal the pane itself needs work (missing signal, wrong layout, missing event annotation). The post-incident review includes a pane-improvement item when relevant.

### 6.4 The pane has an SLO on itself

The Prometheus / Grafana stack is a dependency. Its own availability affects the function's ability to respond. The function sets an availability SLO on the observability stack per `mod-104`, and the pane is included in the fleet's DR drill (`mod-113`) as an asset.

## 7. The anti-patterns

Named explicitly because each erodes the pane's value:

- **Per-team pane drift.** The rag-engineer peer builds their own retrieval dashboard; the ML-platform peer builds their own feature dashboard. The on-call ends up with four tabs again. The function's response: converge on the shared pane; the peer dashboards remain as deep-dive views but the on-call reads the shared pane first.
- **Vendor-UI-only signals.** A pipeline signal that only exists in LangSmith's UI or Pinecone's console is not alertable. Export it to Prometheus or treat it as not-a-reliability-signal.
- **Labels that don't join.** A metric with `corpus_name` instead of `corpus`, or with `release_id` instead of `version`. The join at query time silently fails to find the match and the on-call sees a flat line that looks healthy. The function's metric review catches this at onboarding.
- **Alerts without runbooks.** A pager-bomb in waiting. The function does not accept new alert rules into production without a runbook link that resolves.
- **"Pipeline dashboards are for the ML team."** The claim this module refutes most directly. A pipeline failure that regresses the user is the on-call's incident. If the signal routes anywhere other than the function's pane and the function's queue, the on-call will discover it from users.
- **Alert fatigue tolerated.** Alerts that fire without action for weeks. The function tunes or retires on a cadence; a noisy alert never becomes background hum without a decision.

## 8. The single-pane checklist

Before the function claims a route is covered by its pipeline observability:

1. **Serving SLIs** from `mod-104` are emitting with the full label set (`route`, `tenant`, `cohort`, `model_version`, `prompt_hash`, `config_hash`, `version`).
2. **Pipeline SLIs** from this module emit with labels that join the serving SLIs (`corpus`, `feature_set`, `eval_set` where applicable).
3. **The single pane** (`grafana/dashboards/pipeline-observability.json`) is deployed and version-controlled.
4. **Alert rules** for every pipeline SLO are in Git under the Alertmanager repo, each with a runbook link and a `surface` label.
5. **Change-event annotations** stream into the pane; every prompt / config / retrieval / feature / eval change is visible on the events row.
6. **Trace propagation** is complete — pipeline spans parent under the request span; a recent trace exemplar is linked from each row.
7. **Weekly operational review** has the pane on the agenda; peers attend when their surface is on the agenda.
8. **The pane's own availability** is covered by the observability-stack SLO and included in the DR drill scope.

A route served without the single pane is a route whose incidents the function is not prepared to investigate within a time budget.

## Summary

- Pipeline observability is unified in **one Prometheus / Grafana / OTel stack**, one label vocabulary, one Alertmanager, one Grafana pane. Four disconnected dashboards is the anti-pattern this chapter exists to prevent.
- The pane's layout follows the incident path: **serving SLIs first, SLO burn next, pipeline-stall ages, then per-surface detail, then recent change events**. The on-call reads top-to-bottom.
- Pipeline SLO burn-rate alarms use the same multiwindow multi-burn-rate shape as serving SLOs, route through the same Alertmanager, and require a runbook link to be accepted into production.
- Trace propagation stitches every pipeline span under the request span, so a single trace shows the full path from user request to retrieval to feature lookup to model call.
- The pane is **version-controlled**, **weekly-reviewed**, and **has its own SLO**. It is maintained as a function asset the same way the serving dashboard is.
- Six anti-patterns are named: per-team pane drift, vendor-UI-only signals, non-joining labels, runbook-less alerts, "pipeline dashboards are for the ML team," and tolerated alert fatigue.
- The eight-point checklist governs whether a route is covered by the function's pipeline observability contract.

Chapter 06 closes the module by naming the peer boundaries — the four surfaces each hand off to a specific peer, and the function's irreducible pipeline-reliability surface is what remains after the handoffs are explicit.
