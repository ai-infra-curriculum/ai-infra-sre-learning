# DORA Four-Keys for Model Releases

## Why this chapter exists

DORA — the DevOps Research and Assessment team's multi-year *State of DevOps* reports and the book *Accelerate* (Forsgren, Humble, Kim) — established four metrics that predict the performance of software delivery organisations: **deployment frequency**, **lead time for changes**, **change-failure rate**, and **mean time to restore**. The research is grounded in survey data from hundreds of organisations over a decade and has become the standard quantitative narrative for release-engineering programs. The metrics are unambiguous, comparable across teams, and link engineering practice to business outcomes.

The four-keys hold for classical software delivery with no translation needed. They hold for AI service delivery too, but **the units need redefinition** or the metrics report the wrong thing. "Deployment frequency" for a fleet whose service image deploys daily but whose model weights change weekly is three different numbers depending on what the function calls a deployment. "Change-failure rate" for a release where canary auto-rolled back on quality regression is different from one where the release shipped and a user complained three days later. "Lead time" for a change that is a prompt-template edit is different from a change that is a seven-day fine-tune followed by a release.

The function's job is not to re-derive the metrics. The job is to **define, per fleet, what each DORA key means on this fleet**, instrument to produce the metric, and report it in a narrative the organisation's leadership and the function's peers can read in a single glance. This chapter writes the definitions, the instrumentation, and the narrative shape.

The chapter's boundary: this is a measurement chapter, not a methodology chapter. The DORA methodology is settled; the AI SRE's work is adaptation. External references (the DORA reports, the `dora` CLI tool from Google Cloud, the Four Keys reference implementation) are cited in the resources file.

## 1. The four keys, in one page

For reference:

1. **Deployment frequency.** How often the organisation successfully releases to production. DORA buckets: Elite (multiple per day), High (between once per day and once per week), Medium (between once per week and once per month), Low (less than once per month).
2. **Lead time for changes.** Time from code committed to code successfully running in production. DORA buckets: Elite (< 1 hour), High (1 day to 1 week), Medium (1 week to 1 month), Low (> 1 month). <!-- needs-research: confirm the exact bucket thresholds in the latest DORA State of DevOps report; the thresholds have moved slightly between editions -->
3. **Change-failure rate.** Percentage of changes that result in a failure in production (requiring hotfix, rollback, or incident). DORA buckets: Elite (0–15%), High (16–30%), Medium (16–30%), Low (46–60%).
4. **Mean time to restore (MTTR).** Time from detection of a production failure to service restoration. DORA buckets: Elite (< 1 hour), High (< 1 day), Medium (< 1 week), Low (> 6 months).

The buckets are context — the function's goal is improvement against its own baseline, not a target bucket. Reporting the bucket alongside the number signals maturity.

## 2. Deployment frequency — three fleets of change

A single "deployment frequency" number on an AI fleet undercounts the real release cadence. On a typical model-backed service, three streams of change ship independently and each needs its own count:

### 2.1 Service-image deployments

The classical signal: the container image that runs the inference wrapper, the retrieval service, the agent scaffolding, the structured-output validator. Deploys on the normal CI/CD cadence (per-commit, per-merge, per-sprint depending on the team). This is the number that matches the DORA definition directly.

**Measurement.** Count successful promotions of the serving-image to production per calendar period. "Successful" = the release-gate contract (chapter 05) passed and the canary / blue-green cutover completed. A deploy that auto-rolled back is counted as a deploy *and* as a change-failure (§4).

**Expected rate for an AI fleet on a mature team.** Daily to multiple-per-day on service-image. The service image is just software; it has the same delivery-performance profile as any other microservice.

### 2.2 Model / weight updates

The AI-specific signal: the model being served changes. This is a different artifact, deployed through the same release-gate contract but with a different frequency and a different failure profile.

**Measurement.** Count successful cutovers to a new model version per calendar period. The audit event from chapter 04 §4.4 is the authoritative source.

**Expected rate.** Weekly to monthly on most fleets. Model updates are rarer than service-image updates because the upstream pipeline (training, eval, approval) is longer. On provider-backed routes, the rate is driven by the provider's release cadence; the function counts each provider-model-version pin change as a deployment.

### 2.3 Prompt / config changes

The third signal: prompt templates, agent scaffolding, retrieval config, output-filter thresholds, routing rules. Config changes typically deploy more frequently than service-image changes on an actively-developed fleet.

**Measurement.** Count successful promotions of the config artifact per period. The release-gate contract (chapter 05) applies to config promotions as well as model promotions; a config change without a gate run is an uncounted deployment with a hidden risk.

**Expected rate.** Multiple per day on a healthy fleet, with each change under the full release gate. A fleet whose config changes bypass the gate under a "it's just a prompt" exception is a fleet whose change-failure rate for config will dominate the other two in the next quarter.

### 2.4 Reporting the three

The function reports deployment frequency as **three numbers side by side**, not an aggregate. The quarterly narrative:

```
Deployment frequency (Q1 2026):
  service-image:  4.2 per day    (Elite)
  model-version:  1.3 per week   (Medium — inter-fleet variance high)
  config:         11 per day     (Elite)
```

A single aggregated number hides the maturity of the service-image stream behind the slower model-version stream (or vice versa) and masks the actual distribution of risk the function is managing.

## 3. Lead time — four clocks

Lead time on an AI fleet has a different shape per change stream. The function tracks four clocks.

### 3.1 Service-image lead time

Commit-to-production for the serving-image. Same definition as classical DORA. The clock starts at the first commit on the change's branch and stops when the release-gate's cutover event is recorded.

**Expected.** Hours on a mature fleet; Elite is less than a day. The service image is normal software.

### 3.2 Config-change lead time

Commit-to-production for a prompt / config change. Often faster than service-image because the artifact is smaller and the release-gate run is shorter (no model warm-up; often no shadow run required). Many fleets reach a 10–30 minute config-change lead time.

### 3.3 Model-update lead time

Weight-ready to production for a model change. This is the AI-specific clock:

- **Clock starts** when the training / fine-tune run's final checkpoint is tagged as a release candidate.
- **Clock stops** when the cutover event is recorded.
- **Intermediate stops** are the eval-gate verdict (how long did eval take?), the SLO-gate verdict (how long did shadow + canary take?), and the blue-green warm-up (how long did pre-warm take?).

A model-update lead time of 48–96 hours is common on a fleet with offline eval + shadow + canary + blue-green. The decomposition lets the function see which stage is the bottleneck and target it; `mod-114` chapter 03 writes the quarterly reliability narrative that reads this breakdown.

### 3.4 Data / retrieval-index lead time

For fleets that serve retrieval-augmented responses, a change to the retrieval index (new document ingested, embedding update, re-chunking) is a production change. It is at the `ai-infra-mlops` peer for pipeline wiring (chapter 08 §2), but this module's release-gate contract applies: a retrieval index that ships without a gate is a change that can shift quality without the function noticing.

**Expected.** Hours on a fast-ingest fleet (RAG with high-frequency ingest) to weeks on a slow-change corpus. The decomposition helps the function identify whether the retrieval surface is contributing more to quality variance than any of the other streams.

### 3.5 Reporting lead time

```
Lead time for changes (Q1 2026 p50 / p95):
  service-image:    3.1 h / 11.4 h    (Elite / High)
  config:           22 min / 1.8 h    (Elite / Elite)
  model-version:    68 h / 7.2 d      (Medium — shadow dominates)
  retrieval-index:  4.1 h / 29 h      (High / Medium)
```

Both p50 and p95 — the p95 is where the long-tail work happens; the p50 is the typical case.

## 4. Change-failure rate — what counts as a failure

On a classical service, a change-failure is a release that rolled back, hotfixed, or produced an incident. The DORA definition is operational, not strict. On an AI fleet, three failure modes need to be treated as change-failures even though they look harmless to a classical SRE:

### 4.1 Auto-rollback during canary

Any release that triggered the canary's auto-rollback mechanism (chapter 01 §5) is a change-failure, even though no user saw the regression. The function counts these because:

- The release-gate contract did not prevent the regression from reaching canary.
- The engineering work on the release was wasted (and must be redone).
- Repeated auto-rollbacks on the same stream suggest the gate upstream of canary (shadow, eval) is not catching what it should.

### 4.2 Quality regressions caught in the long-horizon window

A release that passed the full gate, promoted to 100%, and was rolled back 48 hours later on business or quality signals (chapter 01 §4.2 long-horizon watch) is a change-failure. The release should have been held at the earlier stage; a slow regression that reached GA is the gate's failure.

### 4.3 Model-attributable incidents post-release

An incident whose root cause is traced to the released model (judged by the `mod-108` chapter 03 post-mortem) is a change-failure for the release that introduced the model. The post-mortem process sets the attribution; the DORA report inherits it.

### 4.4 What does *not* count as a change-failure

Named to keep the count defensible:

- **A deploy that completed cleanly and was later rolled back for a non-regression reason** (product decision, business change) is not a change-failure.
- **A canary that held at an intermediate weight and then promoted after a longer dwell** is not a change-failure; it is a slow promotion.
- **An infrastructure incident unrelated to the deploy** is not a change-failure; it is counted under the fleet's broader reliability metrics.

### 4.5 Target

Elite fleets land in the 0–15% band on DORA's current rubric. For AI fleets, the practical target is **≤ 15% per change stream**, with service-image typically lower (3–8%) and model-version typically higher (15–25% is common on fast-iterating teams, dropping as the eval and shadow stages mature). A model-version stream above 30% is a signal that the eval / shadow stages are not producing trustworthy verdicts.

### 4.6 Reporting

```
Change-failure rate (Q1 2026):
  service-image:   4.1%    (Elite)
  config:          6.9%    (Elite)
  model-version:  18.4%    (High — shadow coverage improving; see Q4 plan)
  retrieval-index: 9.2%    (Elite)
```

## 5. MTTR — mean time to restore when a model release triggers a rollback

Classical MTTR is the time from detection to restoration of service. On an AI fleet, the clock has an AI-specific structure because the "restoration" of a model release is not always a container restart.

### 5.1 The four MTTR branches

The function tracks MTTR as a composite of the delivery mechanism used:

- **Canary auto-rollback MTTR.** From the auto-rollback trigger to traffic shifted back to stable. Typically 10–60 seconds (mesh config propagation). The canary's rollback is the fastest path; the function targets ≤ 2 minutes.
- **Blue-green rollback MTTR.** From rollback decision to the symmetric cutover complete, including blue's warmth check if blue was already decommissioned. Typically 2–15 minutes on a hot-blue retention policy, longer if blue must be re-warmed. The function targets ≤ 15 minutes with hot-blue retention.
- **Model-rollback MTTR under weight-swap.** When the inference engine supports in-place weight swap (chapter 08 §1 at the engineer peer) and the fleet uses the `mod-108` `rb-model-rollback-in-place` runbook — the time from decision to all replicas serving the previous weights. Typically 5–30 minutes depending on replica count and swap primitive.
- **Full re-deploy MTTR.** When rollback requires rebuilding and re-deploying the previous artifact (rollback window expired, blue decommissioned, no engine-level swap). Typically 30 minutes to hours.

### 5.2 Reporting

```
MTTR (Q1 2026 p50 / p95):
  canary-auto-rollback:        32 s / 2.1 min   (Elite / Elite)
  blue-green-rollback:         6.8 min / 14 min (Elite / Elite)
  engine-weight-swap-rollback: 11 min / 28 min  (Elite / Elite)
  full-redeploy:               47 min / 2.3 h   (Elite / High)
```

The distinction matters because the four represent different investment levels: a fleet whose canary MTTR is 3 hours has not wired the auto-rollback properly; a fleet whose engine-swap MTTR is 2 hours has an inference-engine-peer issue to raise.

### 5.3 The "no-rollback-needed" case

A release that triggers a rollback and the rollback succeeds is a counted MTTR event. A release that triggers an *alert* but did not need rollback (e.g., a transient spike that corrected without intervention) is not counted as MTTR unless the function's on-call declared an incident. The post-mortem decides.

## 6. The AI-specific fifth key — reliability-weighted release velocity

DORA's four keys do not capture one AI-specific signal that mature teams track: **the fraction of releases that promoted on the first gate run**. A release that passes the gate first try has validated the upstream work (eval, pre-promotion load test, shadow); a release that holds, iterates, and passes on the third try is a release that produced wasted engineering time.

The metric is not canonical DORA, but the practice is well-established. Report alongside the four:

```
Gate-pass rate (Q1 2026):
  first-try:     81%
  second-try:    14%
  third-plus:     5%
```

A first-try rate above 90% suggests the gate is too permissive (too few holds); below 70% suggests the gate is catching regressions that should have been caught further upstream (offline eval, pre-promotion load test). The target range is 75–90%, calibrated per fleet.

## 7. Setting targets by maturity

DORA buckets are descriptive, not prescriptive. For an AI SRE function on a specific fleet, targets are set against the fleet's current baseline with a quarterly improvement goal. A representative progression:

- **Year 1.** Instrument all four keys per change stream; publish quarterly; targets are "don't regress from baseline." Change-failure rate on model-version is the single metric to prioritise improving.
- **Year 2.** Target Medium-to-High across all keys on all streams; reduce model-version change-failure rate by 50% from Year 1 baseline; add gate-pass-rate as the fifth key.
- **Year 3.** Target High-to-Elite on service-image and config streams; High on model-version; MTTR on all branches within the SLO's error-budget discipline. The quarterly narrative is where the function demonstrates compounding improvement.

The maturity curve is not uniform. A fleet that lands on Elite service-image frequency in Year 1 may still be at Medium model-version frequency in Year 3 because the model-update pipeline has fundamental constraints (eval latency, training cost) that service-image does not. The function reports the asymmetry, not a blended number.

## 8. Instrumenting the keys

The four keys are measured as a by-product of the systems the previous chapters built:

- **Deployment frequency.** Count of cutover events from chapter 04 §4.4 and canary completion events from chapter 01. Both are structured, both land in the audit log; the metric is a SQL / log-query against the log.
- **Lead time.** Join the first commit on the change's branch with the cutover event timestamp. The Git commit history and the audit log are both required; the `dora` CLI from Google Cloud and the OpenSSF SCALE metrics project implement the join. <!-- needs-research: confirm the current name and status of the Google Cloud 'dora' CLI tool and OpenSSF's four-keys offerings before publication -->
- **Change-failure rate.** Count of change-failure-attributed events (from §4) divided by count of releases. The post-mortem system from `mod-108` chapter 03 is the attribution ground truth; the release audit log is the denominator.
- **MTTR.** Timestamp of rollback trigger minus timestamp of traffic-restored. Both are in the audit log for canary and blue-green; the engine-swap and full-redeploy branches require the `rb-model-rollback` and `rb-full-redeploy` runbook's timestamp capture.

A small reporting service reads the audit log and the post-mortem store on a weekly cron and publishes the metrics to a dashboard. The function presents the dashboard at the quarterly review (`mod-114` chapter 03).

## 9. The AI SRE's DORA checklist

Before relying on the four-keys as a function-level metric:

1. **All four keys are instrumented** per change stream (service-image, model-version, config, retrieval-index).
2. **Lead time is reported as p50 and p95**, not mean.
3. **Change-failure rate's attribution rules are written** (§4) and signed by the function lead.
4. **MTTR is broken down by delivery mechanism** (§5).
5. **The gate-pass rate (fifth key) is tracked** (§6).
6. **The audit-log and post-mortem schemas are stable** so the metric queries do not break on schema drift.
7. **The quarterly narrative report is on the calendar** and includes a trend chart per key per stream over the last 4–8 quarters.
8. **The targets are explicit per stream** and tied to the fleet's error-budget policy where applicable.
9. **The report is shared** with the function's leadership and the predicate-owner peers (eval-engineer, mlops, inference-engineer) so the metric drives cross-peer conversation.
10. **The metric does not displace qualitative review.** A post-mortem that reveals a pattern the four-keys hide is more valuable than a dashboard; the function reads both.

## Summary

- DORA's four keys hold on AI fleets with per-fleet definitions of deployment, lead time, failure, and restore.
- Deployment frequency is reported as three (or four) numbers per stream — service-image, model-version, config, retrieval-index — never aggregated.
- Lead time is tracked as four clocks with p50 and p95 reporting; the model-update clock decomposes into eval, shadow, and warm-up stages.
- Change-failure rate on AI releases includes auto-rollbacks during canary, post-promotion long-horizon regressions, and model-attributed incidents from the `mod-108` post-mortem store.
- MTTR branches by delivery mechanism (canary auto-rollback, blue-green rollback, engine weight-swap, full re-deploy) because the four branches represent different investment levels.
- An AI-specific fifth key — gate-pass rate on first try — tracks whether the pre-gate stages are catching regressions early enough.
- Maturity targets are set per fleet against baseline, with a quarterly improvement goal; the function reports the asymmetry across streams rather than a blended number.
- Instrumentation is a by-product of the earlier chapters' audit log and post-mortem store; a small reporting service aggregates for the quarterly narrative.
- A ten-point checklist gates the first function-level DORA report.

Chapter 07 writes how progressive delivery is paired with the chaos and load-test programs — the pre-promotion verification that keeps the four-keys from telling a story the function would rather not tell.
