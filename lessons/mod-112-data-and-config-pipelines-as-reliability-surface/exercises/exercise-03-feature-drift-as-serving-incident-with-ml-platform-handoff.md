# Exercise 03 — Feature Drift as a Serving Incident, with ML-Platform Handoff

**Estimated effort:** 3 hours · **Output:** a minimal feature-serving stack with the chapter 03 SLO set, a drift scenario captured on the dashboard, a parity audit run, and an incident timeline (≤ 600 words) that routes to the ML-platform peer.

## Objective

Instrument a toy feature-serving path so the function can observe **feature drift, pipeline stalls, and training / serving parity** as reliability signals. The deliverable is not a production-grade feature store — it is the **SLO, signal, and handoff contract** from chapter 03. At the end of the exercise, the function can take any feature set the `ai-infra-ml-platform` peer ships and enforce the reliability contract on it without re-authoring the pipeline.

The handoff shape is deliberate: the peer runs the pipeline; the function runs the drift / stall / parity alarms; an incident routes the fix to the peer; the function keeps the gate while the fix lands.

## Prerequisites

- Chapter 03 of this module.
- Prometheus + Grafana + Alertmanager available on a cluster you control.
- A key-value store that can play the role of the online store — Redis is the shortest path; DynamoDB Local is acceptable.
- Any batch job substrate — a simple cronjob, an Airflow DAG, or a Python script in a CronJob Kubernetes resource. The pipeline is simple; the signal set is where the work is.
- A toy serving service (reuse from previous exercises if available) that will call the feature-lookup path on every request.
- Optional: a Feast install. Feast is the shortest path to a realistic feature-store abstraction, but any key-value store with a pipeline writing to it satisfies the exercise.

## Problem statement

Work in six parts.

### Part 1 — Author the feature set

Author a **feature set** `rank-v1` consisting of three features per `user_id`:

- `user_tenure_days` (integer; days since account creation).
- `user_session_count_7d` (integer; count of sessions in the last 7 days).
- `user_cohort` (categorical: "free", "pro", "enterprise").

Author a synthetic source-of-truth — a CSV or SQLite table — with 1,000 synthetic users. The values are what the compute pipeline reads.

### Part 2 — Author the compute pipeline

Author a periodic job that:

- Reads the source-of-truth.
- Computes the three features per user.
- Writes to the online store under keys of the form `rank-v1:{user_id}` with a per-feature value plus `materialised_at`.
- Writes the same `(user_id, timestamp, features)` rows to an **offline store** (parquet files under a `/features/offline/` path, or a SQLite table) so the parity audit (part 5) has something to replay against.
- Emits Prometheus metrics:
  - `feature_pipeline_run_last_success_timestamp{feature_set}`
  - `feature_pipeline_lag_seconds{feature_set, stage}` (histogram)
  - `feature_pipeline_backlog{feature_set, stage}` (gauge)
  - `feature_pipeline_failure_total{feature_set, stage, reason}` (counter)
  - `feature_materialisation_rows_per_minute{feature_set}` (gauge)

Run the job on a 5-minute cadence.

### Part 3 — Instrument the serving-side feature lookup

Modify the toy serving service so each request:

1. Resolves the entity key (the test harness can pass `user_id` via a header).
2. Calls the online store for `rank-v1:{user_id}`.
3. Emits:
   - `feature_lookup_seconds_bucket{feature_set, feature}` (histogram)
   - `feature_lookup_null_ratio_total{feature_set, feature, outcome="hit|miss|error"}` (counter)
   - `feature_drift_psi{feature_set, feature}` — see part 4 for the emitter.
4. Uses the feature values in the response (even a toy use is fine — e.g., modifies the response prefix based on `user_cohort`).

### Part 4 — Author the drift monitor

Author a drift-monitor job that runs on a 15-minute cadence and:

- Reads the last-N-hour distribution of each feature from the online-store-fed stream (sample at serving time and emit to Prometheus as `rate` counters, or sample from the offline store).
- Compares against a frozen baseline distribution captured at onboarding.
- Computes PSI (Population Stability Index), or any of KL / JS / KS as a substitute.
- Emits `feature_drift_psi{feature_set, feature}` and `feature_drift_null_rate{feature_set, feature}`.

Use the open-source Evidently library (`mod-104` chapter 05 catalog) to compute PSI if you prefer not to implement from scratch; emit the metric via a small exporter.

### Part 5 — Author the parity audit

Author a parity-audit job that runs on a 10-minute cadence and:

- Samples 100 recent serving lookups from the previous 10 minutes (from logs, or from a kept sample in Redis).
- For each `(user_id, timestamp)`, reads the online store's returned value and the offline-store's value for the same `(user_id, timestamp)`.
- Classifies the pair: `match`, `value_mismatch`, `missing_online`, `missing_offline`, `tolerance_exceeded`.
- Emits:
  - `feature_parity_match_total{feature_set, feature}` (counter)
  - `feature_parity_mismatch_total{feature_set, feature, mismatch_type}` (counter)

### Part 6 — Induce the incident scenarios

Run each of the following and capture the dashboard + alerts + handoff artifact:

- **Pipeline stall.** Stop the compute pipeline. Wait 30+ minutes. Observe:
  - `feature_pipeline_run_last_success_timestamp` ages.
  - The warning fires at the per-pipeline threshold.
  - Feature-freshness burns; the SLO alarm fires.
  - `feature_lookup_null_ratio_total{outcome="miss"}` rises as newer users have no backing data.
  - Open an incident ticket **routed to the ML-platform peer**. The ticket includes the Prometheus trace, the pipeline UI link, and a clear statement of what the function expects (restart the pipeline, backfill the gap).
  - Restart the pipeline; recovery arrives; record the end-to-end timeline.

- **Feature drift.** Introduce a step-change in one feature's distribution — e.g., replace half the source-of-truth's `user_cohort` values from "free" to "pro". Let a few compute runs materialise. Observe:
  - `feature_drift_psi` crosses the chosen threshold.
  - The drift alarm fires.
  - The incident commander follows the `rb-feature-drift` runbook stub.
  - Hypothetical remediation: route to the peer; the peer confirms the change is intended (a product cohort migration), the drift is legitimate, and the baseline is refreshed. Document the decision.

- **Parity mismatch.** Introduce a code skew — in the compute pipeline, change the transformation so `user_cohort` is uppercased in the online store but not in the offline store. Observe:
  - `feature_parity_mismatch_total{mismatch_type="value_mismatch"}` grows.
  - The parity SLO alarm fires.
  - Hypothetical remediation: route to the peer with the classification; the peer fixes the transformation; a backfill resolves the mismatch window; the function tracks recovery.

### Part 7 — Dashboard and incident write-up

Author a Grafana dashboard under `grafana/dashboards/feature-pipeline.json` with the chapter 05 row layout for the feature surface:

- Feature-lookup null-ratio and latency.
- Feature-pipeline lag, backlog, time-since-last-success.
- Drift PSI per monitored feature.
- Parity-match ratio with breakdown by mismatch type.

Write an incident write-up (≤ 600 words) covering:

- The three induced scenarios with timelines (alarm time, incident-commander acknowledgment, peer handoff time, remediation time, recovery time).
- The handoff artifact for each — what was the function's ticket to the ML-platform peer, what information did it include, and what did the function *not* do (that is explicitly the peer's work)?
- Open items — baselines to refresh, SLO thresholds to tune, runbook additions.

## Requirements

1. **A feature set is defined** (`rank-v1`) and a compute pipeline writes it on a 5-minute cadence.
2. **The serving service calls the online store** on every request and emits the feature-lookup signal set.
3. **A drift monitor** emits `feature_drift_psi` per feature on a 15-minute cadence.
4. **A parity audit** runs on a 10-minute cadence and emits parity match / mismatch counters with classification.
5. **The three incident scenarios** (stall, drift, parity mismatch) are induced, captured, and documented.
6. **Each incident has a handoff artifact** — a ticket with the Prometheus trace, the pipeline link, and a clear ask of the peer.
7. **A Grafana dashboard** covers the feature surface per chapter 05's row layout.
8. **An incident write-up (≤ 600 words)** names the timelines, the handoffs, and the open items.
9. **Three runbook stubs** exist: `rb-feature-pipeline-stalled`, `rb-feature-drift`, `rb-feature-parity-mismatch`.

## Starter guidance

- Keep the compute pipeline **trivial**. A Python script in a Kubernetes CronJob that reads the SQLite source and writes to Redis is 100 lines. The exercise is not about the pipeline.
- Redis has native TTL support; use it to simulate materialisation-lag naturally rather than coding a separate decay path.
- For the parity audit, writing to the offline store from the same job that writes to the online store is the simplest setup; the skew you introduce in part 6 is between the two writes in that single job.
- Evidently's `DataDriftPreset` can produce the PSI you need with minimal code; alternatively, `scipy.stats.ks_2samp` and a hand-coded PSI function give full control.
- When routing an incident to the "ML-platform peer," write the ticket **as if to a real person**. The exercise's point is not technical realism but handoff discipline — the ticket must be self-contained, cite the signal that fired, name the ask, and not try to do the peer's job.
- The exercise's dashboards and runbook stubs merge upward into the unified pane in exercise 05; keep the file structure consistent.

## Acceptance criteria

- A toy feature set with a compute pipeline, serving-side lookup, drift monitor, and parity audit — all running on the cluster.
- Full chapter 03 signal set emitted into Prometheus.
- Three induced scenarios captured on the dashboard with timelines.
- Three handoff artifacts (tickets to the ML-platform peer) committed to the repo as markdown or templated Jira / GitHub issues.
- A Grafana feature-surface dashboard committed to Git.
- An incident write-up (≤ 600 words) covering the three scenarios.
- Three runbook stubs committed to Git.

## Stretch goals

- **Feast integration.** Rebuild part 2 on Feast; demonstrate that the chapter 03 signal set is emitted via Feast's registry plus a small adapter. Write 200 words on where Feast's built-in observability stops and the function's signal extension begins.
- **Streaming features.** Replace the batch cron with a small Kafka-driven stream (Redpanda as a local substitute). Demonstrate a lag-and-backlog alarm firing when the consumer stalls.
- **Entity-drift detection.** Add an entity-drift signal that detects when the mix of user cohorts hitting the serving tier shifts, independent of any one feature's distribution. Alarm and investigate.
- **Metadata-store SLO.** Stand up a toy model-registry (MLflow local) and declare its SLOs separately. Demonstrate a registry-lookup failure blocking the release-gate verdict — the metadata-store-in-the-critical-path pattern from chapter 03 §6.
- **Cross-exercise integration.** Wire the feature-pipeline change-events into Grafana annotations on the unified pane from exercise 05, so a parity-mismatch incident lights up alongside the prompt / retrieval / eval change events.
