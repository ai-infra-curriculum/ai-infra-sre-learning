# Chaos-Testing the Provider-Fallback Contract

## Why this chapter exists

`mod-107` chapter 02 wrote the provider-fallback contract: the per-route document that enumerates the health signals that make the function declare a provider unhealthy, the equivalence table of alternatives, the policy that chooses one, and the semantics (idempotency, retry, streaming, consistency) that must hold across any swap. That contract is a document. The fallback-executing code — the router, the breaker, the equivalence lookup, the retry handler, the streaming splicer — is the artifact that is supposed to implement it.

Between a document and the artifact that implements it, the gap is paved with real incidents. The breaker's threshold is too high for the failure mode the vendor actually has. The equivalence table lists a fallback whose quality tier shifted after the last eval. The streaming splice loses the half-delivered response and the retry double-bills. The router's own health signal is stale because the status-page ingestor is in the same cell as the primary provider.

Chaos engineering is the discipline that closes the gap. The chaos program takes the contract's commitments one by one, injects the failure the contract is written against, and measures whether the implementation meets the contract. The three key metrics it produces — MTTD (mean time to detect), MTTR (mean time to recover), fallback-hit-rate — are the surface the function reports to the quarterly reliability review.

This chapter is the specification of that program. Chapter 03 named the single-shot provider-timeout experiment; this chapter writes the full contract-chaos cluster around it. Exercise 04 of this module is the forcing function.

## 1. What a contract-chaos program is

A contract-chaos program is not a single experiment; it is a *suite* of experiments, one per clause of the fallback contract, run on a cadence, scored against a dashboard, and regressed against each release of the router. Four commitments:

1. **Every clause of the contract has a corresponding experiment.** If a clause says "the breaker opens at 50% error rate over 30 s with ≥ 20 samples," there is an experiment that produces exactly that failure shape and asserts the breaker opens. A clause with no experiment is unverified.
2. **The program is release-gating, not post-release.** Router changes land behind a canary that runs a contract-chaos smoke suite in staging before promotion. Shipping a router change without the suite having passed is a policy violation.
3. **Three metrics only.** MTTD, MTTR, fallback-hit-rate per failure class. More metrics is noise; fewer leaves gaps.
4. **Results flow back to the contract itself.** When an experiment fails, the fix is either in the router or in the contract — sometimes the contract's threshold was wrong. The program keeps both honest.

## 2. The six provider-failure shapes to inject

A good contract-chaos program covers the six shapes the vendor can present. Each is a separate experiment in the suite. Each uses the egress-shaping pattern from chapter 02 §4.1 (`HTTPChaos` / `NetworkChaos`) because the fault must be injected on the serving side's egress — that is where the router sees it.

### 2.1 Primary 5xx storm

- **Fault.** The primary provider returns 5xx (`500`, `502`, `503`, provider-specific `overloaded_error`) on ≥ 50% of calls for 2 minutes.
- **Hypothesis.** The error-rate breaker opens within the policy window (default 30 s with ≥ 20 samples per `mod-107` chapter 02 §2.1); fallback-hit-rate rises to ≥ 95% of eligible traffic within 1 minute of the breaker opening; MTTR to recover primary and re-close the breaker is bounded by the dampening period.
- **Mechanism.** `HTTPChaos` with `abort: true` and `status: 503` on the primary's egress, applied to a 1% label-selected pod slice.
- **Expected observations.** Breaker state transitions logged; fallback traffic ratio dashboard crosses threshold; per-route error-rate dips as traffic shifts; cost-per-request panel shifts to the fallback's `cost_ratio_vs_primary`.

### 2.2 Primary 429 sustained

- **Fault.** The primary returns `429 Too Many Requests` or the provider-specific quota error (`rate_limit_error`) with a non-trivial `Retry-After`, sustained for 5 minutes. This is a *retryable* class in `mod-107`'s taxonomy, so the retry budget must be consumed before the breaker opens.
- **Hypothesis.** Per-request retry budget is consumed (no infinite retry loop); after the budget, the breaker opens; the AI-SRE's rate-limit back-off respects `Retry-After` and does not cause a thundering herd on recovery.
- **Mechanism.** `HTTPChaos` with `status: 429` and a response header injection of `Retry-After: 30`.
- **Expected observations.** Retry counters move; retry budget exhaustion events logged; breaker eventually opens; herd avoidance verified (traffic to the primary recovers over the dampening window, not in a spike).

### 2.3 Primary stall (TCP open, body blocked)

- **Fault.** The primary's TCP connection stays open and the response body stalls for ≥ 30 seconds. This is the shape every generic timeout test misses because the connection is not closed and most naive breakers wait on the per-request deadline rather than the body-progress signal.
- **Hypothesis.** The per-request body-progress timeout fires (not only the connect timeout); the latency breaker (`mod-107` chapter 02 §2.2) correlates the stall with the error-rate counter; the breaker opens on the combined signal.
- **Mechanism.** `HTTPChaos` with `delay: 60s` on the response.
- **Expected observations.** p95 latency panel spikes before error-rate; breaker uses the combined signal; fallback engages.

### 2.4 Streaming stall mid-response

- **Fault.** A streaming response starts normally, then the provider stops sending chunks after 500 ms of normal streaming for ≥ 15 s. Observed with every major provider at some point; it is the pattern that the public incident logs cite most often for "the integration looked fine, but users saw a half-sentence."
- **Hypothesis.** The streaming splicer detects the inter-chunk gap exceeding the SLO threshold; the client receives either a splice-to-fallback (if the splicer supports it) or a clean terminal with a "retry on fallback" signal; the retry does not double-bill.
- **Mechanism.** A chaos proxy sits in the egress path and lets the first N bytes pass, then holds the body. Either a custom test proxy (because `HTTPChaos` does not natively mid-splice) or a dedicated egress proxy with a fault-injection mode. Some mature fleets run LiteLLM-shape gateways with a chaos hook explicitly for this case.
- **Expected observations.** Inter-chunk-time histogram spikes; streaming-splice-success counter increments (or clean-terminate counter does if splicing is not supported); client retry succeeds on fallback.

### 2.5 Primary degraded (200 but wrong)

- **Fault.** The primary returns 200 but with regressed quality — higher refusal rate, higher hallucination score on the judge pipeline, higher structured-output schema-violation rate. This is `mod-105` chapter 07 and `mod-107` chapter 02 §2.4's territory.
- **Hypothesis.** The quality-regression signal (judge-based) reaches the router's health input within the breaker window; the router demotes the route to an equivalent per the equivalence table; the application's capability-reduced flag propagates to downstream consumers.
- **Mechanism.** Synthetic injection at the judge-pipeline layer (the chaos-wrapper lowers the judge score for the labelled slice), or `HTTPChaos` body-rewrite on the primary's response to produce a regressed output shape. The body-rewrite is more realistic but more invasive; the judge-signal injection is safer and covers the breaker's own logic.
- **Expected observations.** Judge-score panel drops; router demotion event; capability-reduced flag visible to the application; safety signals stay flat.

### 2.6 Primary region red, global green

- **Fault.** The primary's regional endpoint is unreachable while the vendor's global status page reports green. `mod-107` chapter 03 §4 cites Bedrock us-east-1 and Vertex us-central1 as historical exemplars.
- **Hypothesis.** The router treats the status-page signal as soft (`mod-107` chapter 02 §2.3); the locally observed error elevation triggers the breaker independent of the status page; residency-aware fallback kicks in for tenants whose contract permits it.
- **Mechanism.** Combined: `NetworkChaos` partitioning egress to the primary's regional endpoint, plus a chaos-wrapper that *keeps* the status-page ingestor reporting green for the duration (to prove the router does not depend on it). AWS FIS can simulate the regional-isolation part.
- **Expected observations.** Primary error-rate rises; status-page ingestor stays green; breaker fires on local signal; fallback chosen respects residency per row.

## 3. The three metrics

A contract-chaos program's output is a small, boring dashboard. Three numbers per experiment, trended over time.

### 3.1 MTTD — mean time to detect

The time from fault injection to the router declaring the primary unhealthy. Measured as `breaker_open_timestamp - fault_inject_timestamp`.

- **Budget.** Set per failure class. For 5xx and stall, MTTD ≤ 30 s is the floor. For sustained 429, MTTD ≤ 2 minutes (retry budget first). For streaming stall, MTTD ≤ 15 s (inter-chunk threshold). For degraded-200, MTTD ≤ 5 minutes (quality signal is slow).
- **Alarm.** A chaos run where MTTD regresses by ≥ 20% against its baseline is a chaos failure; the result goes to the router team as a bug, not as a flake.

### 3.2 MTTR — mean time to recover

The time from fault recovery (fault injection ended, primary is healthy again) to the router closing the breaker and re-routing primary traffic. Measured as `primary_traffic_ratio_recovered_timestamp - fault_end_timestamp`.

- **Budget.** The dampening period from `mod-107` chapter 02 §2.5 is the floor. MTTR below the dampening period is a bug (the router is flapping); MTTR well above it is a stuck-breaker bug.
- **Alarm.** MTTR outside the ± 20% band from baseline fails the chaos run.

### 3.3 Fallback-hit-rate

The fraction of eligible traffic that successfully executed on the fallback route during the chaos window, over total eligible traffic. "Eligible" excludes traffic that cannot fall back (residency-locked, tier-specific) and is defined in the contract.

- **Budget.** ≥ 95% for tier-1 routes during a sustained primary failure. Fallback-hit-rate below 95% means the fallback itself is failing or the equivalence row is misconfigured; either is a contract bug.
- **Alarm.** Fallback-hit-rate below budget fails the chaos run.

### 3.4 The dashboard shape

The three metrics are plotted on the chaos dashboard, one row per experiment, one column per metric, with the baseline-trend as a shaded band. A chaos run outside the band is red; a run inside is green. The operations review reads it in five minutes.

## 4. The suite as a release gate

The contract-chaos suite is not an aspiration; it is a gate on router releases. The gate shape:

1. **Pre-promotion smoke.** Every router change runs a reduced suite (five-minute windows, staging, all six shapes) and must pass MTTD / MTTR / fallback-hit-rate per the budgets.
2. **Post-promotion verification.** The full suite (15-minute windows, 1% production slice, auto-halt wired) runs within 24 hours of promotion. A regression blocks the next change.
3. **Quarterly game-day.** A full-scale cross-provider failover drill (chapter 05 §4) exercises the suite end-to-end plus the human side.
4. **Regression prevention.** Each experiment's baseline is tracked in a persistent store (the fixture ledger from `mod-108` chapter 05). A trend that moves the wrong way over three runs opens a bug on the router without waiting for a user-visible failure.

A router change that regresses contract-chaos by ≥ 10% on any metric is rolled back; the fixture ledger records it as a chaos finding.

## 5. Common failure modes this program catches

The experiments in §2 catch a predictable set of bugs. Naming them helps the first-time author of this suite recognise them in the result:

- **The breaker's minimum-volume requirement masks the stall.** The stall shape (§2.3) has low request volume per unit time (because the server is holding on to calls), so the breaker's "≥ 20 samples" rule does not trip. The fix is a per-request body-progress timeout that counts as an error independent of volume.
- **The retry budget is unbounded.** The 429 experiment (§2.2) finds the loop; the fix is a per-request retry cap and a per-client backoff.
- **The status-page signal is a hard dependency.** The region-red-global-green experiment (§2.6) finds this; the fix is treating the status-page as a soft cue per `mod-107` chapter 02 §2.3.
- **The streaming splice drops the response context.** The streaming-stall experiment (§2.4) finds this; the fix is either a splicer or a clean-terminate-and-retry convention.
- **The quality signal is stale.** The degraded-200 experiment (§2.5) finds this; the fix is tightening the judge-pipeline's refresh cadence or adding a canary-eval short loop.
- **The fallback is silently degraded in the same window.** Fallback-hit-rate drops to 60% because the fallback is also 429ing. The fix is multi-provider health awareness in the router and a tier-3 fallback that is cheaper-and-worse but *available* (the `degraded_mode: true` row in the equivalence table).

## 6. What this program does not test

The contract-chaos suite does *not* cover:

- **Adversarial robustness** (prompt injection, model extraction, jailbreak). Those are `ai-infra-security`'s program (chapter 06 of this module).
- **Model-eval robustness** (benchmark robustness, red-team coverage). Those are the `ai-eval-engineer` / `model-evaluation-engineer` peer's program (chapter 06).
- **Business-contract SLAs vs. SLOs.** The reconciliation of the vendor's contractual SLA with the function's internal SLO is `mod-107` chapter 05's subject.
- **Multi-region / multi-cloud DR.** Full DR scenarios are `mod-113`'s territory. Contract-chaos exercises region-local failover (§2.6); multi-region failover drills live in DR.

## 7. The suite-authoring worksheet

Exercise 04 produces the environment-specific version of this suite. The worksheet shape:

1. **Enumerate the clauses.** Read the fallback contract (`mod-107` chapter 02) and list every assertion — "breaker opens at X," "fallback-hit-rate ≥ Y," "no double-billing on retry." Each assertion is a chaos target.
2. **Pick the mechanism.** From chapter 02 §4.1's egress-shaping pattern for the six shapes in §2.
3. **Set the budgets.** MTTD, MTTR, fallback-hit-rate per failure class. Pin to the contract's numbers.
4. **Wire the auto-halt.** Each experiment auto-halts on a budget breach on the dashboard-critical metric (fallback-hit-rate for shift experiments, MTTD regression for breaker experiments).
5. **Wire the release gate.** The CI or CD system runs the reduced suite on every router PR, blocking merge on regression.
6. **Schedule the full suite.** Weekly on a 1% production slice, quarterly as a game-day.
7. **Hook the ledger.** Each run's three metrics are persisted; trends drive the quarterly review.

## Summary

- The provider-fallback contract (`mod-107` chapter 02) is a document; a chaos suite is what verifies it. The suite enumerates six provider-failure shapes — 5xx storm, 429 sustained, TCP stall, streaming stall, degraded-200, region-red-global-green — and exercises each against the router.
- Three metrics are enough: MTTD, MTTR, fallback-hit-rate per shape. Budgets come from the contract; regressions are router bugs.
- The suite is a release gate. Pre-promotion smoke in staging, post-promotion verification in production, quarterly game-day, continuous regression prevention via the fixture ledger (`mod-108` chapter 05).
- The suite catches a predictable set of real bugs: volume-masked breakers, unbounded retry budgets, hard status-page dependencies, lost streaming splices, stale quality signals, co-degraded fallbacks.
- The suite does not test adversarial robustness, eval robustness, or full DR. Those boundaries are chapter 06.

Chapter 05 turns the catalogue (chapter 03) and the suite (chapter 04) into a continuous program with scheduled game-days, blast-radius governance, and auto-halt wiring.
