# Exercise 04 — Provider-Timeout and Degraded-Mode Chaos Experiments

**Estimated effort:** 3 hours · **Output:** the provider-fallback contract-chaos suite (six experiments, one per failure shape), a baseline run of the three metrics (MTTD, MTTR, fallback-hit-rate), and a release-gate wiring that blocks router PRs on regression

## Objective

Build the chapter 04 contract-chaos suite for a chosen fleet. The deliverable is six authored experiments — one per provider-failure shape from chapter 04 §2 (5xx storm, 429 sustained, TCP stall, streaming stall mid-response, degraded-200, region-red-global-green) — each run against the router under blast-radius controls, with the three metrics (MTTD, MTTR, fallback-hit-rate) measured and persisted, and the suite wired as a pre-promotion gate on router changes.

This is the module's capstone operational artifact. Where exercise 03 verified serving-path reliability, this exercise verifies the fallback contract — the single piece of code whose silent failure is the most expensive AI-reliability bug the function can ship. A fleet that passes this exercise has a measurable, defendable provider-reliability posture.

## Prerequisites

- Chapter 04 of this module.
- The provider-fallback contract from `mod-107` chapter 02 for the fleet (real or illustrative).
- The chaos operator and auto-halt wiring from exercise 02.
- The baseline-measurement discipline from exercise 03 — the fleet's existing SLO panels and any judge-pipeline / cost-per-request metrics.
- A router / gateway implementation (LiteLLM-shape in-service, Portkey-shape as a gateway, or a bespoke equivalent) whose behaviour you can observe.
- Access to or a representation of at least two providers (primary + at least one alternate). If the fleet uses one provider, use a mock secondary endpoint for the exercise; note the limitation in the write-up.

## Problem statement

Work in seven parts.

### Part 1 — Inventory the contract

Read the fleet's fallback contract (`mod-107` chapter 02 §2–5) and extract the specific numeric commitments:

- Breaker threshold (default 50% errors over 30 s, ≥ 20 samples).
- Latency-breaker threshold (default 2× TTFT SLO for two consecutive 60-s windows).
- Fallback-hit-rate commitment (default ≥ 95% for tier-1 routes).
- Dampening period (default 5 minutes).
- Streaming splice vs. clean-terminate policy.
- Residency rules per tenant class.

The extracted list is the suite's hypothesis-source. An experiment whose hypothesis is not pinned to one of these numbers is unmoored.

### Part 2 — Author the six experiments

Produce six experiment specifications in `chaos/experiments/chaos-provider-*.{yaml,md}`. Each follows chapter 03 §1's shape and chapter 04 §2's fault description. Use the chapter 02 §4.1 egress-shaping pattern — `HTTPChaos` / `NetworkChaos` on the service's egress, scoped by label. Produce both the CRD and the documentation.

- `chaos-provider-5xx-storm` — `HTTPChaos` abort with `status: 503`.
- `chaos-provider-429-sustained` — `HTTPChaos` abort with `status: 429` and `Retry-After` header. If your operator does not support header injection, use a sidecar fault-injection proxy or a gateway's fault-injection mode.
- `chaos-provider-tcp-stall` — `HTTPChaos` delay the response body by 60s without closing the connection.
- `chaos-provider-streaming-stall` — The chapter 04 §2.4 case. This usually requires a custom chaos proxy; author the spec, document the mechanism limit, and if the direct mechanism is not available, substitute the closest `HTTPChaos`-based approximation and note the limitation.
- `chaos-provider-degraded-200` — Signal injection into the router's quality-health input (safer) or `HTTPChaos` body-rewrite to produce a regressed output (more realistic but tier-2 in some policies).
- `chaos-provider-region-red-global-green` — `NetworkChaos` partition of the primary's regional endpoint plus a chaos-wrapper that keeps the status-page ingestor's local state green for the duration. AWS FIS AZ-disruption is the alternative mechanism for AWS fleets.

Each spec names which contract clause it tests, what MTTD / MTTR / fallback-hit-rate budgets apply, and the three auto-halt alarms.

### Part 3 — Baseline the three metrics

Before wiring the gate, run each experiment three times in staging with full blast-radius controls. For each run, capture:

- **MTTD** — timestamp of first fault to timestamp of breaker-open.
- **MTTR** — timestamp of fault-end to timestamp of primary-traffic-ratio-recovered.
- **Fallback-hit-rate** — eligible-traffic-on-fallback-successful / eligible-traffic during the window.

Persist the numbers in a shared store (a Prometheus recording rule, a dashboard panel, or the fixture ledger). Three runs is enough to establish a mean and a variance; more if the variance is high.

### Part 4 — Wire the dashboard

Build a one-dashboard row per experiment with three panels (MTTD, MTTR, fallback-hit-rate), each with a shaded band for the baseline ± 20%. The dashboard is the function's quarterly-review-ready artifact.

### Part 5 — Wire the release gate

Add a CI job to the router repo that runs the reduced suite (five-minute windows, staging environment) on every router PR. The job:

- Fails merge if any MTTD regresses ≥ 20% from baseline.
- Fails merge if any MTTR exits the ± 20% band.
- Fails merge if any fallback-hit-rate drops below budget.
- Posts the three-metric table on the PR as a check.

GitHub Actions, GitLab CI, or whatever the function uses — the choice is immaterial; the gating is the artifact.

### Part 6 — Document the findings

For each experiment, write a one-paragraph finding. Expected (chapter 04 §5 is the catalogue):

- The breaker's minimum-volume rule masks the stall shape.
- The retry budget is unbounded for the 429 case.
- The status-page signal is a hard dependency (region-red experiment catches this).
- The streaming splice drops context (streaming-stall catches it).
- The quality signal is stale by more than the breaker window (degraded-200 catches it).
- The fallback is silently degraded (fallback-hit-rate below budget on multi-provider failure).

If a finding is real, ledger it with the `mod-108` chapter 05 schema, name the owning peer (usually `ai-infra-inference-engineer` or `ai-risk-engineer`), and record the remediation plan. If no finding — the contract holds — write that up too; the exercise's result is a known-good baseline.

### Part 7 — Wire the suite into the quarterly review

Add the dashboard row (part 4) to the function's quarterly reliability review template. Note the review cadence and the escalation rule (a trend that worsens over three quarters escalates to Principal / Staff attention per chapter 05 §4.4).

## Requirements

1. **Six experiments authored and stored in the chaos-program repo** with CRD + doc each.
2. **Baselines established** with three runs per experiment, persisted in a shared store.
3. **Dashboard built** with one row per experiment and the three panels per row.
4. **Release gate wired** on the router PR path, with the three-metric check posted.
5. **At least one experiment's finding ledgered** (the catalogue in chapter 04 §5 virtually guarantees at least one real finding).
6. **No experiment runs in production without the blast-radius controls from chapter 05 §3.**
7. **Auto-halt verified for every experiment** before the baseline runs.

## Starter guidance

- **Begin with the 5xx-storm experiment.** It is the easiest shape to inject, has the cleanest breaker-open signal, and calibrates the suite's tooling before you tackle the harder cases.
- **The streaming-stall experiment is the hardest.** Expect to spend half your time here. The chaos operator's native primitives rarely handle "start a stream, send a few chunks, then stall the body" directly; you will likely need a custom fault-injection proxy or a gateway's test mode. Document the choice.
- **Be honest about the degraded-200 mechanism.** Body-rewrite is more realistic but opens a safety-adjacent surface. Signal injection into the router's health input is safer and still exercises the breaker's logic. Pick, defend the choice in the write-up.
- **Wire the gate before the first production-slice run.** The gate is cheaper than the three-runs baseline; if you have to redo baselines later, you want the gate already working.
- **Three metrics only.** More is noise. If a run surfaces a signal the dashboard does not track (cost-per-fallback, cross-region latency), ledger it as a fixture candidate for next quarter — do not grow the suite's primary dashboard.

## Acceptance criteria

- The six experiment artifacts in the chaos-program repo.
- A baseline table (nine numbers minimum: three per experiment × six experiments is 18 if you want both pre- and post-fault, but at minimum one baseline mean per metric per experiment) persisted.
- A dashboard with six rows and 18 panels (or the equivalent compact view).
- A CI gate on the router PR path that runs the reduced suite and posts the three-metric check.
- At least one ledgered finding, named owner, remediation plan.
- A one-page suite write-up (≤ 600 words) covering: the contract clauses tested, the baseline numbers, the findings, the gate's first PR run outcome, the next quarter's planned improvements.

## Stretch goals

- **Add a cross-provider failure experiment.** `chaos-provider-primary-and-fallback-both-degraded` — both providers degrade at once; the suite should verify the degraded-mode tier-3 fallback engages (the `degraded_mode: true` equivalence-table row).
- **Ledger the vendor-correlation finding.** If the dashboard reveals that your primary and fallback are hosted on the same cloud / same AZ / same shared dependency (`mod-107` chapter 01 §3 shared-substrate), file that as a tier-1 chaos finding for `ai-risk-engineer`.
- **Add a cost-aware budget check.** Fallback-hit-rate above budget but cost per request on fallback 3× the primary's breaks the economic steady state. Measure and ledger.
- **Chaos-verify the breaker's dampening period.** Run `chaos-provider-5xx-storm` twice in quick succession (fault, recover, fault again inside the dampening window) and verify the breaker's hysteresis prevents a thundering herd.
- **Author the quarterly-game-day scenario.** Compose two experiments into one scenario (`chaos-provider-5xx-storm` + `chaos-judge-model-unavailable` running simultaneously) and run it as the input to exercise 06.
