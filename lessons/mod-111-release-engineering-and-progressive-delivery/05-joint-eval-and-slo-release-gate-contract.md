# The Joint Eval-and-SLO Release-Gate Contract

## Why this chapter exists

Chapters 01–04 built the delivery mechanisms (canary, response diff, shadow, blue-green). Each mechanism answers "how does the candidate reach production?" None of them answers the earlier question: **who signs that the candidate is allowed to**. That question is the release gate's, and on an AI service the release gate has two independent predicates, each owned by a different function. If the release pipeline evaluates only one, the other is implicit — and an implicit gate is a gate that is bypassed at 3 AM by an engineer under deadline pressure who did not know the predicate existed.

The two predicates:

- **Eval-gate.** Does the candidate meet the quality bar the `ai-eval-engineer` peer has defined for this route? Offline eval scores, canary-set thresholds, judge-pipeline verdicts, benchmark regressions. Methodology at the eval peer (level 30 by `mod-101`); this module consumes the verdict.
- **SLO-gate.** Will the candidate meet the reliability bar the AI SRE function owns? Serving-health SLOs (latency, availability, streaming health, structured-output conformance), cost SLOs, dependency health. Methodology at this module.

Neither predicate subsumes the other. An eval-passing candidate that doubles TTFT is a reliability regression that ships because the eval team's methodology does not include latency. An SLO-passing candidate that silently hallucinates 2× more is a quality regression that ships because the SRE's methodology does not include groundedness. Mature AI releases require **both** predicates to pass, and the contract that stitches them is **authored, signed, and machine-enforced** so no release path bypasses either.

This chapter writes the contract. It is the module's most load-bearing deliverable: canary, shadow, blue-green, DORA — all of them depend on a well-written gate contract, and the fleet that invests here first gets the most leverage out of the rest.

The chapter's boundary: the **eval-gate methodology** is not written here. That is at the `ai-eval-engineer` peer (level 30). This module owns the **contract shape** — what the eval team's verdict looks like when it reaches the release pipeline, what the SRE's verdict looks like, how the two compose, what the fallback is when one or both are unavailable, and how violations are attributed and fixed.

## 1. The contract's role in the release pipeline

A release pipeline for a model update has a canonical shape:

```
 commit ─→ CI build ─→ offline-eval (eval peer) ─→ artifact
                              │
                              ▼
                   release-gate decision ─── eval-gate verdict
                              │          ─── SLO-gate verdict
                              ▼
                     progressive delivery ─── shadow (ch 03)
                              │           ─── canary  (ch 01)
                              │           ─── blue-green (ch 04)
                              ▼
                         promote / hold / rollback
```

The release-gate decision is the explicit point where the two predicates meet. Everything before it (CI, eval, artifact build) is the upstream of each predicate; everything after is the delivery mechanism that depends on the verdict.

The contract document has four jobs:

1. **Name the two predicates** and the owners of each.
2. **Specify the predicates' thresholds** in terms both owners can measure.
3. **Define the composition rule** (AND, with explicit handling for a missing verdict).
4. **Prescribe the fallback** when either owner's pipeline is unavailable.

A contract missing any of the four is a contract that gets argued at incident time.

## 2. The contract's structure — a worked example

A release-gate contract is a versioned YAML (or equivalent structured) document checked into the service's Git repo at a well-known path (`release/gate-contract.yaml`). A representative example for a chat route:

```yaml
contract_version: v1.3
contract_owner: ai-sre-release-lead
last_reviewed: 2026-02-01
route: chat
applies_to:
  - canary_rollouts
  - shadow_runs
  - blue_green_cutovers
  - manual_hotfix_promotions

predicates:
  eval_gate:
    owner: ai-eval-engineer
    source_of_truth: eval-service.release/verdict
    verdict_schema: eval-gate-verdict-v1.schema.json
    required_thresholds:
      offline_eval:
        groundedness: { min_mean: 0.82, min_fraction_above: 0.95 }
        helpfulness:  { min_mean: 0.78 }
        safety_pass_rate: { min: 0.995 }
      canary_eval_set:
        regression_rate: { max: 0.02 }
        catastrophic_regression_count: { max: 0 }
    missing_verdict_behaviour: hold

  slo_gate:
    owner: ai-sre
    source_of_truth: slo-gate-service.release/verdict
    verdict_schema: slo-gate-verdict-v1.schema.json
    required_thresholds:
      serving:
        ttft_p95_ratio:    { max: 1.15 }
        tpot_p95_ratio:    { max: 1.10 }
        success_rate:      { min: 0.999 }
        streaming_stall_rate_ratio: { max: 1.20 }
      cost:
        cost_per_request_ratio: { max: 1.25 }
      quality_adjacent:
        refusal_rate_delta: { min: -0.05, max: 0.05 }
    missing_verdict_behaviour: hold

composition:
  rule: AND     # both predicates must pass
  tie_breaker: hold

fallbacks:
  eval_gate_unavailable:
    permit_promotion: false
    permit_rollback: true     # reliability-driven rollback does not need eval
    escalate_to: ai-eval-engineer-oncall
  slo_gate_unavailable:
    permit_promotion: false
    permit_rollback: true
    escalate_to: ai-sre-oncall

overrides:
  permitted_by: ai-sre-director AND ai-eval-engineer-director
  requires: written_justification + fixture_ledger_entry
  audit_trail: release-gate-overrides.log
```

Every field in the contract is intentional. The next sections unpack the ones that carry the most weight.

## 3. The two verdicts — shape and transport

Each predicate emits a **structured verdict** that the release-gate decision reads. Both verdicts share a shape (so the composition code is simple), and both are produced by the owner's pipeline to a well-defined schema.

### 3.1 Eval-gate verdict

```yaml
eval_gate_verdict:
  verdict_schema: eval-gate-verdict-v1
  candidate_sha: ef56ab78
  baseline_sha:  ab12cd34
  produced_at: 2026-02-15T13:45:12Z
  judge_version_tag: 2026-01
  eval_sets_run:
    - name: chat-offline-core-v7
      fraction_passed: 0.985
    - name: chat-canary-set-v12
      fraction_passed: 0.971
  dimensions:
    groundedness: { mean: 0.84, fraction_above_threshold: 0.962 }
    helpfulness:  { mean: 0.81 }
    safety_pass_rate: { value: 0.998 }
    refusal_rate_delta: 0.012
  verdict: pass          # pass | fail | abstain
  reason: null           # populated when fail or abstain
```

The verdict is produced by the eval peer's pipeline (offline eval, canary eval, judge aggregation). The AI SRE's role is to confirm the pipeline emits the schema and that the release-gate service reads it.

### 3.2 SLO-gate verdict

```yaml
slo_gate_verdict:
  verdict_schema: slo-gate-verdict-v1
  candidate_sha: ef56ab78
  baseline_sha:  ab12cd34
  produced_at: 2026-02-15T13:48:03Z
  measured_from: shadow_window | canary_window | pre-promotion-load-test
  measurement_window_seconds: 7200
  dimensions:
    ttft_p95_ratio: 1.05
    tpot_p95_ratio: 1.02
    success_rate:   0.9994
    streaming_stall_rate_ratio: 1.08
    cost_per_request_ratio: 1.18
    refusal_rate_delta: -0.004
  dependency_health:
    judge_pipeline_available: true
    provider_fallback_healthy: true
  verdict: pass
  reason: null
```

The SLO-gate verdict is produced by the SRE's release-gate service — a small service that reads the Prometheus / observability stack over the measurement window and compares to the thresholds in the contract. The verdict is emitted on demand by the release pipeline.

### 3.3 Transport

Both verdicts live in a signed, machine-readable form in a well-known location:

- **Pull.** The release pipeline fetches the latest verdict for the candidate SHA from each service's HTTPS endpoint.
- **Push.** The pipelines write verdicts to a shared audit bucket keyed by candidate SHA.

Either is fine; the key is that the release pipeline **cannot decide to promote without both verdicts present**. §4 writes the composition.

## 4. Composition — the AND rule and its edge cases

The default composition is strict AND: both predicates must emit `pass`, and the release pipeline promotes only on `pass ∧ pass`. Every other combination holds or rolls back.

Edge cases worth naming:

### 4.1 One verdict missing

A missing verdict is not treated as `pass`. The contract specifies `missing_verdict_behaviour: hold` for both predicates (§2). "No news is good news" is the failure mode this clause prevents.

### 4.2 Verdict older than acceptable window

A verdict is produced against a specific candidate SHA, but the candidate may be rebuilt (image rebuild, config change) without re-running eval. The contract specifies a **freshness window** per predicate:

```yaml
predicates:
  eval_gate:
    verdict_freshness_seconds: 86400   # verdict must be ≤ 24 h old
  slo_gate:
    verdict_freshness_seconds: 7200    # verdict must be ≤ 2 h old
```

An expired verdict is treated as missing. This closes the "we ran eval last week and the gate is still green" bypass.

### 4.3 Verdict for a different SHA

A verdict for a different `candidate_sha` than the one being promoted is a mismatch. The release pipeline rejects promotion. This closes the "we ran eval on an older build of the candidate" bypass.

### 4.4 Partial pass within a predicate

A predicate may pass on aggregate but fail on a stratified cohort (see chapter 03 §6). The schema accommodates both: `verdict: pass` on the aggregate requires `all cohorts pass` within the predicate's logic, not AND at the composition layer. Cohort handling is the predicate owner's responsibility; the contract requires the predicate to emit `verdict: fail` if any required cohort fails.

### 4.5 One predicate `abstain`

The verdict may be `abstain` when the predicate cannot be computed — insufficient data, judge unavailable, measurement window too short. The contract treats `abstain` as `hold`, not as `pass` or `fail`. The release does not promote; the owning oncall is paged.

### 4.6 Forced override

Overrides exist. A release-gate override requires:

- **Two signatures** from the predicate owners' directorate (not oncall, not lead — directorate). The contract names the roles (not people).
- **A written justification** stored in the audit log.
- **A fixture-ledger entry** committing to the follow-up work that would allow the gate to pass in future.
- **A time-bounded window** (the override expires; the fix is in place before it does).

Overrides are tracked and reviewed quarterly. A fleet whose override rate is > 5% of releases is a fleet whose gate is too strict or whose predicates are not trusted — both of which are the quarterly review's job to investigate.

## 5. Writing the thresholds

The threshold values in the contract are not arbitrary. They come from four sources, and the contract document notes the source per threshold for durability.

### 5.1 SLO thresholds

The serving-health and cost thresholds are **derived from the SLOs**. An SLO of "TTFT p95 ≤ 2 s" with an error budget of 1% over 30 days produces a per-release budget: a release that burns more than N% of the budget in its first days violates the SLO contract. The gate threshold for `ttft_p95_ratio` is the ratio at which the burn is unacceptable; representative value is 1.10–1.25 depending on the fleet's error-budget policy.

The thresholds are not published in isolation; the contract references the `mod-103` SLO document and includes the burn-rate calculus:

```yaml
slo_gate:
  derivation_note: |
    ttft_p95_ratio threshold of 1.15 corresponds to burning 0.5 of the 30-day
    error budget in the first week post-promotion at the current p95 of the
    stable baseline. See slo/chat/ttft-p95.md.
```

### 5.2 Eval thresholds

The eval thresholds are at the eval peer. The contract **restates** them so a reader has the full picture, but the authoritative source is the eval team's document. Any change to the eval threshold is a change to the contract, not a unilateral eval-side edit; the AI SRE signs off that the new threshold is operationally reasonable (will not produce a gate that rejects every release).

### 5.3 Cost thresholds

The cost threshold comes from the `mod-106`/`mod-107` cost SLO stack. The ratio in the contract (e.g., `cost_per_request_ratio ≤ 1.25`) is the per-release cost-tolerance; the SLO itself is the longer-horizon budget that this tolerance rolls up to.

### 5.4 Quality-adjacent thresholds

Refusal-rate delta, safety-classifier-trigger rate, output-length distribution — these sit in the SLO gate because they are observable on the serving layer and because they have reliability consequences (a refusal rate spike is often a serving-side regression, not a quality one). The eval peer may have overlapping thresholds; the contract's §2 notes which side owns each to avoid double-counting.

## 6. The contract and the delivery mechanisms

The contract is the input to every delivery mechanism; each reads a different slice of it.

### 6.1 Canary (chapter 01)

The canary's per-step analysis reads the SLO-gate verdict as the step-by-step predicate. The eval-gate verdict is checked **before the canary starts** and again at **promotion to 100%**:

```
 CI build ─→ eval-gate ──(pass)──→ canary step 1 ──(SLO-gate green)──→ step 2 ...
                                                                       ──→ step N
                                                                       ──→ eval-gate re-check
                                                                       ──→ promote 100%
```

The eval re-check at 100% catches the case where the canary's judge-online signal has drifted from the offline-eval signal; the gate requires both to pass.

### 6.2 Shadow (chapter 03)

Shadow produces the SLO-gate verdict for a candidate that has not yet served user traffic. The eval-gate verdict is independently produced from offline eval + the canary eval set run against the candidate's shadow responses. Both must pass before the candidate is permitted to canary.

### 6.3 Blue-green (chapter 04)

The cutover is predicated on both verdicts. The contract's `applies_to: [blue_green_cutovers]` field requires the cutover record (§4.4 of chapter 04) to include both verdicts. A cutover without both is an override, not a routine release.

### 6.4 Rollback

A rollback does not require eval-gate verdict (the baseline has already passed; rolling back to it is reliability-driven). The contract explicitly says `permit_rollback: true` even when eval is unavailable, so the oncall can execute a rollback without blocking on the eval pipeline.

## 7. Contract maintenance — the living document

The contract rots if it is written once and forgotten. Four disciplines keep it current:

### 7.1 Quarterly review

Every quarter, the predicate owners (eval-engineering lead and AI SRE release lead) review the contract. The review covers:

- **Override rate.** Any override in the quarter is reviewed for cause; a pattern of overrides on the same predicate triggers a threshold revision.
- **Pass rate.** The fraction of releases that pass the gate on first try. Target 80–95%; very low pass rate suggests the gate is too strict; very high pass rate (near 100%) suggests the gate is too loose.
- **Verdict freshness.** How often a release is held on expired verdicts; if frequent, the freshness windows are too tight for the release cadence.
- **Fixture back-feed.** New failure modes observed in post-release windows translate into new thresholds or new dimensions on the verdict schema.

### 7.2 Change-controlled edits

Edits to the contract are reviewed changes, like any other Git-tracked artifact. The commit touching the contract requires:

- A PR with both predicate owners' approval.
- A reference to the quarterly review or the specific incident driving the change.
- A dry-run against the last N releases to confirm the new threshold would not have changed historical outcomes in an unexpected way.

### 7.3 Schema versioning

The verdict schema version (`verdict_schema: eval-gate-verdict-v1`) is bumped when the schema changes. A release gate during a schema transition reads both versions; a schema older than the gate expects is treated as `missing_verdict`.

### 7.4 The reliability-contract archive

Each version of the contract is retained in Git; releases carry a reference to the contract version they were gated on. A post-mortem for a regression that the gate missed can read "the gate was contract v1.3; the threshold for X was Y; here is what Y should have been."

## 8. Anti-patterns and the discipline that prevents them

Named failure modes:

- **"Eval said yes, so we're good."** Promoting on eval-pass alone. The SLO-gate is skipped. The next-day serving-latency regression is the result; the post-mortem blames "no reliability check," which is correct — the contract wasn't in place.
- **"SLO said yes, so we're good."** The symmetric. Quality regressed; ship-rate stayed up; a public post-mortem is the delayed outcome.
- **"The gate is advisory."** The release pipeline reads the verdicts but promotes anyway when they fail. The gate is not a gate; it is a dashboard. The contract's §2 `missing_verdict_behaviour` and `fallbacks` close this specifically.
- **"We override every Friday."** The override path becomes the default path. §4.6 and §7.1 catch this at the quarterly review.
- **"The verdict is a PDF."** The verdict is human-readable but not machine-readable; the release pipeline cannot parse it; the gate is executed by a human reading the PDF. Schema the verdict (§3) so the pipeline can enforce.
- **"The thresholds are nobody's."** The contract does not name an owner per predicate, so when a threshold is argued at incident time, no one has the authority to adjust it. §2's `owner` field is the fix.
- **"We don't have a rollback path on gate failure."** The contract specifies `permit_rollback: true` under both unavailable-predicate branches; a release pipeline that cannot roll back when the gate is red is a release pipeline that cannot be safely driven.

## 9. The AI SRE's gate-contract checklist

Before relying on the release gate for the first time on a fleet:

1. **The contract exists in Git**, at a well-known path, under version control.
2. **Both predicate owners have signed** the initial version.
3. **The thresholds have a derivation note** for each (SLO, eval, cost, quality-adjacent).
4. **The verdict schemas are published** and both pipelines emit against them.
5. **The freshness windows** and the SHA-matching discipline are enforced.
6. **The `missing_verdict_behaviour` is `hold`**, not `pass`, for both predicates.
7. **The override path is documented, named by role, and audited.**
8. **The quarterly review cadence is on the calendar.**
9. **The rollback path is allowed without both predicates.**
10. **The release pipeline refuses to promote** on `fail ∨ abstain ∨ missing` — tested by injecting a synthetic failing verdict and confirming the pipeline holds.

A gate that cannot be mechanically verified by this checklist is a gate that will be bypassed when the next release is behind schedule.

## Summary

- The AI release gate has two independent predicates — eval-gate (quality) and SLO-gate (reliability) — owned by different functions; neither subsumes the other.
- The contract is a Git-tracked, versioned document specifying predicates, thresholds, composition, fallbacks, and overrides.
- Each predicate emits a structured verdict against a published schema; the release pipeline reads both and cannot promote without either.
- Composition is strict AND; missing / stale / mismatched-SHA / abstained verdicts are treated as `hold`, not as `pass`.
- Thresholds are derived from the SLO stack, the eval team's canary thresholds, and the cost SLOs; every threshold has a derivation note.
- The contract is the input to canary, shadow, and blue-green mechanisms; each reads a different slice of the verdict stream.
- The contract is maintained — quarterly review, change-controlled edits, schema versioning, archive for post-mortems.
- Anti-patterns (advisory gate, PDF verdicts, unowned thresholds, missing rollback path) are named so the function recognises them; the contract's fields close each one.
- A ten-point checklist gates the first use of the contract per fleet.

Chapter 06 writes DORA four-keys adapted to this release shape — the quantitative narrative the function reports up on release reliability.
