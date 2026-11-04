# Feature and Metadata Store Reliability

## Why this chapter exists

A large category of AI serving regressions are not model regressions. They are **data regressions** — a feature a model relies on stopped updating, the pipeline that computes it stalled, the serving-time value drifted from the training-time value, a metadata join quietly started returning nulls. The model is doing exactly what it was trained to do; the inputs it is being handed have changed, and the user sees a worse answer.

In ML-platform-mature organisations, feature stores (Feast, Tecton, Vertex AI Feature Store, SageMaker Feature Store, Databricks Feature Store) are the layer that serves these inputs. They sit between the training pipeline and the serving tier and promise two things: **feature availability** at serving time (a feature lookup returns a value fast enough for the request path) and **training / serving parity** (the serving-time value of a feature equals the training-time value for the same entity-at-time). When either promise breaks, serving regresses in a way the serving-tier's own SLIs do not show.

This chapter writes the reliability contract on feature and metadata stores from the AI SRE's seat. The contract is: feature-drift signals are **serving incidents**, pipeline stalls are **page-worthy events**, training / serving parity is a **monitored property**, and the whole surface is observable in the single pane with the serving stack. The boundary is explicit. Feature-pipeline authoring — choosing the features, writing the transformations, selecting the online store, operating the registry, running backfills — is the **`ai-infra-ml-platform`** peer's work (level 30 per `mod-101`, slug per the ladder document). The function does not author features; the function owns the reliability contract on the feature surface the serving tier reads.

Primary references: [Feast documentation](https://docs.feast.dev/), [Tecton documentation](https://docs.tecton.ai/), [Vertex AI Feature Store](https://cloud.google.com/vertex-ai/docs/featurestore), [SageMaker Feature Store](https://docs.aws.amazon.com/sagemaker/latest/dg/feature-store.html), [Databricks Feature Store](https://docs.databricks.com/aws/en/machine-learning/feature-store/). The training / serving skew concept has its canonical write-up in Google's [*Rules of Machine Learning*](https://developers.google.com/machine-learning/guides/rules-of-ml) (Rule #29 is the one most directly relevant). The incident-response attitude mirrors `mod-108`.

## 1. Where feature stores sit in the fleet

The serving tier reads features the same way it reads retrieval results — through a lookup against a dependency that must be fast, correct, and fresh. A typical flow for a model that uses features (classification / scoring / recommendation, routing / tool-selection inside an LLM agent, or hybrid retrieval that uses user-side features for re-rank):

```
request (entity: user_id)
  -> feature client: GET /online-store/features?entity=user_id&feature_set=rank-v4
  -> model inference with (prompt, retrieved_context, features)
  -> response
```

Behind this, two offline pipelines keep the online store fresh and feed training:

- **Feature compute pipeline** — a stream (Flink, Beam, Kafka Streams) or batch (Airflow, Spark, dbt) job computes feature values from source events and writes them to the online store.
- **Materialisation into the online store** — a write path from the compute pipeline into a low-latency key / value store (Redis, DynamoDB, Bigtable, Cassandra, managed-tier of the feature-store product).

The training path reads the *same* feature definitions from the offline store (parquet on object store, BigQuery, Snowflake) to assemble training data. Training-serving parity is the property that **the same `(entity, feature, timestamp)` tuple returns the same value from both paths**.

The AI SRE's observational seat is the serving-side read. The function's SLIs read from there, back-propagate to pipeline stalls, and route alerts to the ML-platform peer when the underlying pipeline is the cause.

## 2. The feature-serving SLO set

Four SLOs cover the serving-side reliability contract. The function defines each per feature-set (not per feature — the feature set is the deploy unit).

### 2.1 Feature-availability SLO

> 99.95% of feature lookups against feature-set `V` return within the p99 latency target over a rolling 30-day window.

The latency target is per-set because feature sets differ widely: a small set of user-profile features served from Redis has a 2–5 ms target; a large set of real-time-aggregated features may have a 50–100 ms target. The target is negotiated with the ML-platform peer at feature-set onboarding time.

### 2.2 Feature-freshness SLO

Analogous to the retrieval-freshness SLO in chapter 02:

> 99% of features in feature-set `V` have `materialisation_lag_seconds` under the per-feature bound over a rolling 7-day window.

Where `materialisation_lag_seconds = now - feature_last_materialised_at(entity, feature)`. For streaming features, the bound might be 60 seconds; for daily-batch features, the bound is 24 hours. The per-feature bound is declared in the feature registry and enforced here.

### 2.3 Feature-completeness SLO

> 99.9% of feature lookups against feature-set `V` return a value for every feature in the set over a rolling 7-day window.

Missing features (nulls, lookup failures, entity-not-found) often collapse to a silent default in the serving tier, and the model behaves as if the user has no history. The function alerts on the ratio of lookups that return a complete feature vector.

### 2.4 Training-serving parity SLO

> For a weekly audit sample of 10,000 entity-at-time lookups, parity-match rate between the serving store and the offline store is above 99% over a rolling 30-day window.

Parity-match is a stronger predicate than "both returned a value": it means the values are equal within a per-feature tolerance (exact match for categorical, within ε for numeric). Chapter §5 writes the parity audit.

## 3. Feature drift as a serving incident

Feature drift is the slow change in the *distribution* of feature values over time. On classical ML, drift is monitored by the ML team on a Monday-morning dashboard. On AI-served products where feature-driven decisions propagate into the user-visible response, drift is an on-call signal.

### 3.1 The drift signal families

For each feature in a monitored set:

- **Population drift.** The distribution of feature values across the serving traffic has moved vs. the training distribution (or vs. a rolling baseline). Measured by PSI, KL divergence, Jensen-Shannon, Wasserstein, Kolmogorov–Smirnov. `mod-104` chapter 05's tooling catalog applies directly — Evidently, Arize Phoenix, WhyLabs are the go-to open / open-core stacks.
- **Entity drift.** The mix of entities being served has changed (new user cohorts, new tenants) — feature distributions may look fine on the surface but the composition under them has shifted.
- **Null-rate drift.** The fraction of lookups returning null has moved. A feature whose null-rate climbs from 1% to 20% is a reliability incident even if the non-null distribution is unchanged.

### 3.2 Drift detection cadence

The function routes drift at two cadences:

- **Fast cadence** — null-rate and feature-availability-latency per feature, alerted on 5-minute burn-rate windows. These are the signals that catch pipeline-stall-shaped failures.
- **Slow cadence** — distribution-shift statistics per feature, computed on hourly or daily rollups, alerted when PSI (or equivalent) exceeds a per-feature bound sustained over a rolling window.

The drift-detection tool emits Prometheus-friendly signals; the function wires them into Alertmanager under `mod-104` chapter 05's routing discipline. A drift alarm that fires at 03:00 routes to the on-call the same as a 5xx spike — not to an email digest.

### 3.3 Drift as an incident, not a backlog item

The function's claim is specific: **when a drift alarm fires on a production feature set, the response is an incident response, not a ticket-file-and-triage workflow**. The incident commander follows `mod-108`:

1. Acknowledge the page.
2. Scope the incident — which feature set, which feature, which route, which cohort.
3. Determine blast radius — are serving responses measurably worse, or is the drift purely on the input side?
4. Decide the action: roll back the model (if the model was recently promoted on a different feature distribution and is sensitive), roll back the feature pipeline (if the pipeline change shifted the distribution), or hold and investigate (if the drift is a legitimate production shift the model must learn).
5. Route to the ML-platform peer for pipeline-side remediation.

Not every drift fires a page. Noisy features that drift constantly are tuned at SLO-setting time (the bound is widened; a sustained-window requirement is added). The function does not tolerate "drift always fires, we ignore it" — that pattern is dead-signal, same as a flapping serving alert.

## 4. Feature-pipeline-stall detection

A stalled compute pipeline is the most common root cause of the drift-burn and null-rate-climb alarms. The function instruments the pipeline the same way it instruments the ingest pipeline in chapter 02.

### 4.1 Signals

- `feature_pipeline_run_last_success_timestamp{feature_set, pipeline}` — gauge; `time() - value` is time-since-last-success.
- `feature_pipeline_lag_seconds{feature_set, pipeline, stage}` — histogram per stage.
- `feature_pipeline_backlog{feature_set, pipeline, stage}` — queue depth.
- `feature_pipeline_failure_total{feature_set, pipeline, reason}` — counter.
- `feature_materialisation_rows_per_minute{feature_set}` — write rate into the online store.

### 4.2 Alerts

A stalled pipeline is detected before the freshness SLO burns:

- `time() - feature_pipeline_run_last_success_timestamp > per_pipeline_threshold` — warning.
- Burn-rate on the freshness SLO — page.

Treating the time-since-last-success as a warning and the SLO burn as a page is deliberate: a short stall is tolerable (the feature's freshness bound absorbs it); a sustained stall is the SLO burn.

### 4.3 The stall runbook

`rb-feature-pipeline-stalled` is the function's runbook:

1. Identify the stalled stage. Pull the pipeline UI (Flink UI, Airflow, Dataflow / Beam console, the feature-store product's pipeline view).
2. Identify the immediate cause — upstream source dead, worker pool at capacity, dependency service degraded, schema-change rejection.
3. If the root cause is in the peer's scope, hand off to the ML-platform peer with the specific stage and the Prometheus trace. The SRE does not rewrite the Flink job; the peer does.
4. If the root cause is in the function's scope (e.g., the online-store database is at capacity), remediate within the runbook.
5. Monitor the freshness SLO burn through remediation.
6. After recovery, post-incident review against the chapter 05 observability pane.

The handoff to the peer is explicit; the function is not a passive escalation point but it is also not the owner of the fix.

## 5. Training / serving parity monitoring

Training / serving skew is the most-cited failure mode in the ML-platform literature and the hardest to catch because it is invisible to the serving tier. Two ways it emerges:

- **Code skew.** The transformation applied to a feature at serving time differs from the one applied at training time — a different library, a different default, a different unit. Feature stores whose purpose is to unify the two paths (a single transformation written once) exist precisely to prevent this.
- **Data skew.** Even with identical code, the serving distribution differs from the training distribution because the training data was sampled at a particular time or with a particular filter.

The function's role is to monitor parity; the peer's role is to fix a parity violation.

### 5.1 Parity audit

Once per week (or faster for high-stakes feature sets), the function samples `N` recent serving lookups, replays each as an offline-store lookup against the same `(entity, feature, timestamp)`, and emits:

```promql
feature_parity_match_total{feature_set, feature}
feature_parity_mismatch_total{feature_set, feature, mismatch_type}
```

Where `mismatch_type` is `missing_online`, `missing_offline`, `value_mismatch`, or `tolerance_exceeded`.

The parity-match ratio is an SLI. A ratio below the parity SLO threshold burns the budget and fires a page.

### 5.2 Parity failure classification

- `missing_online` — the online store did not return a value for an entity the offline store has. Usually pipeline-stall or materialisation lag.
- `missing_offline` — the offline store does not have the entity-at-time the online store returned. Usually a backfill gap.
- `value_mismatch` (categorical) — the serving value differs from the training value for the same key. Usually code skew — a transformation default changed between paths.
- `tolerance_exceeded` (numeric) — the serving value is outside the per-feature tolerance of the training value. Usually floating-point precision, unit change, or time-window edge.

Each is a different peer handoff; the function's alert includes the classification so the peer can jump to the right investigation.

### 5.3 The parity audit's reliability

The audit job is itself a dependency. The function runs it, times it, and alerts if it has not completed on cadence. A silent parity audit is worse than no parity audit — it tells the function things are fine when they are not.

## 6. Metadata-store reliability

Metadata stores (model registry, feature registry, lineage store) are a smaller surface than feature stores but the same discipline applies. They are the dependency the release gate (`mod-111` chapter 05), the audit log (`mod-110` chapter 05), and the serving tier all read from.

### 6.1 Signals

- `metadata_store_lookup_seconds_bucket{store, operation}` — latency per operation.
- `metadata_store_error_total{store, operation, reason}` — error counter.
- `metadata_store_record_count{store, record_type}` — gauge for drift on record counts (a model-registry whose record count drops is suspicious).

### 6.2 SLOs

> Metadata-store lookup availability > 99.99%, p99 latency within the per-operation budget, over a rolling 30-day window.

The target is tighter than feature-store availability because the metadata store is in the release-gate critical path: a release-gate call that times out on a model-registry lookup blocks every promotion.

### 6.3 The registry-mismatch class

A specific failure the function watches for: the serving tier's resolved model identifier does not match any record in the registry. This is the metadata-layer analogue of the chapter 02 embedding-version-mismatch. It fires when:

- The registry was pruned by retention policy while a long-running pod still had the old identifier cached.
- A deploy wrote a new model to the serving tier without registering it first.
- A replica of the registry is stale.

The detector emits:

```promql
serving_model_identifier_unregistered_total{route}
```

Any non-zero value is a page.

## 7. The feature / metadata handoff with the ML-platform peer

The ML-platform peer is a level-30 peer on the engineering ladder (`mod-101`). <!-- needs-research: confirm the exact slug and level of the ML-platform peer in the current org ladder — the number is stated in the module spec and should match --> The handoff is written in chapter 06; this section anchors what the function owns versus what the peer owns at the feature surface.

### 7.1 The function owns

- The feature-serving SLO set (availability, freshness, completeness, parity).
- The signal emission from the online-store read path.
- The pipeline-stall detection and alert routing.
- The on-call incident response.
- The single-pane observability dashboard (chapter 05).
- The release-gate contract for feature-set promotions (the peer runs the pipeline; the function's gate signs off on the resulting feature-set version).

### 7.2 The peer owns

- Feature authoring and transformation code.
- Feature registry content.
- Pipeline operation (Flink / Beam / Spark / Airflow / dbt).
- Backfill strategy and execution.
- Online-store sizing and schema.
- Offline-store partitioning and retention.
- The feature-set versioning scheme.

### 7.3 The shared surface

Three artifacts are shared:

- **The feature SLO contract** — negotiated jointly, written once, signed by both peers.
- **The parity audit** — the function operates it; the peer authors the feature-specific tolerance values.
- **The feature-set release gate** — the peer runs the pipeline; the gate reads a verdict that includes the function's reliability signals and the peer's eval signals.

A disagreement between the two peers at a release decision routes to the function's release-engineering lead and the ML-platform lead jointly — the same model as the `mod-111` chapter 05 contract review cadence.

## 8. The feature-reliability checklist

Before accepting the feature-reliability contract for a new feature-set:

1. **Feature-serving SLO set** defined with per-feature-set targets, signed by the ML-platform peer.
2. **Signals emitted** — availability, freshness, completeness; lookup latency histogram per feature set.
3. **Drift monitors** running — PSI / KL / null-rate per monitored feature, with Prometheus-friendly emission into the serving-stack observability pane.
4. **Pipeline-stall detection** wired — last-success-timestamp, lag, backlog, failure counters emitted and alerted on.
5. **Parity audit** scheduled and running — sample size and cadence declared, Prometheus emission of match / mismatch classifications.
6. **Three runbooks** exist — `rb-feature-pipeline-stalled`, `rb-feature-drift`, `rb-feature-parity-mismatch` — and the stall and parity-mismatch runbooks have been drilled at least once.
7. **Metadata-store SLOs** separately defined for the model registry, feature registry, and lineage store the fleet depends on.
8. **Peer handoff with `ai-infra-ml-platform`** signed, including who the function escalates to when a pipeline-side remediation is required.
9. **Feature-set changes flow through the release gate** per chapter 01's progressive-delivery path.

A feature-set served without these items in place is served without a reliability contract; the function does not represent it as covered.

## Summary

- Feature drift and pipeline stalls are **serving incidents** on AI fleets that depend on feature stores. The function pages on them, holds an incident response, and routes pipeline-side remediation to the ML-platform peer.
- Four SLOs cover the feature-serving surface: **availability**, **freshness**, **completeness**, and **training / serving parity**. Each is per-feature-set, negotiated with the peer, and burn-alarm-routed into the on-call queue.
- Pipeline-stall detection runs alongside the SLO burn-rate alarms — time-since-last-success is a warning, SLO burn is a page. The `rb-feature-pipeline-stalled` runbook routes to the peer explicitly.
- **Training / serving parity** is audited on a weekly cadence (faster for high-stakes feature sets) by replaying serving lookups against the offline store; mismatches are classified and routed to the peer for root-cause.
- **Metadata-store reliability** (model registry, feature registry, lineage) gets its own SLO because it sits in the release-gate critical path.
- The function does **not** author feature transformations, run the pipeline, or own the registry's content. The ML-platform peer does. The function owns the reliability contract on what the feature surface exposes to serving and the on-call response when those signals burn.
- The nine-point checklist governs a feature-set's acceptance into the function's reliability contract.

Chapter 04 moves to the third data surface the function reads reliability from: the eval sets themselves, which have their own rot, drift, and poisoning failure modes.
