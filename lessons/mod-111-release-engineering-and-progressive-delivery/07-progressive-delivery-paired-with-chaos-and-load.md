# Progressive Delivery Paired with Chaos and Load Programs

## Why this chapter exists

A release gate that passes only in steady-state conditions is a gate that passes the wrong thing. Steady-state production traffic rarely reveals a release's capacity ceiling, its fallback behaviour under provider failure, or its burn profile when a scheduled job stacks onto normal traffic. Mature teams catch these before the user does by **running the candidate through load and chaos conditions as part of the release** — not as a separate quarterly drill but as a pipeline stage that must pass alongside eval and SLO.

The tooling for load and chaos already exists in this track: `mod-106` chapter 05 covers load testing for capacity planning, and `mod-109` writes the chaos program in detail. This chapter does not re-derive them. It writes the **release-time composition** — what runs before promotion, what runs during ramp, and what runs after — so the release-gate contract (chapter 05) and the delivery mechanisms (chapters 01, 03, 04) incorporate chaos and load as built-in predicates rather than side activities.

The chapter's three core deliverables:

1. **Pre-promotion load test at canary scale** — the candidate is run at the canary's target load before weight is ramped, so capacity ceilings are discovered off the user's path.
2. **Chaos probe of the new deploy before weight ramp** — a small set of chaos experiments fire against the candidate to verify fallbacks, breakers, and degraded-mode routing hold on the new deploy.
3. **SLO-burn as circuit breaker during ramp** — the ramp itself is instrumented with a hair-trigger burn-rate monitor that stops promotion (not rollback; a different control from chapter 01 §5) when the budget is spent faster than expected.

All three are mechanically enforced through the release pipeline, and all three produce signals the release-gate contract consumes.

## 1. Why release-time chaos and load are different from steady-state programs

The chaos program from `mod-109` runs on a four-tier cadence (per-change, daily, weekly, quarterly). The per-change tier (chapter 05 §1.1 of mod-109) is the hook this chapter plugs into. The release-time use is deliberately narrower than the full catalogue:

- **Small, deterministic suite.** Three to five experiments that take less than 20 minutes end-to-end. Not the full catalogue.
- **Pre-promotion only.** The candidate has not yet taken live traffic; the experiments run against the pre-warmed green environment (chapter 04 §3) or the pre-canary candidate.
- **Pass / fail as a gate predicate.** The outcome is boolean: did the hypothesis hold? If no, the release-gate holds.

The chaos program's quarterly game-day still runs; this chapter does not replace it, only bolts a specific pre-release use case on.

Similarly, the load-test program from `mod-106` has a steady-state role (capacity planning) and a release-time role (pre-promotion verification). The release-time use is narrower:

- **At the canary's scale, not the full fleet's.** A load test that runs the candidate at the full fleet's RPS requires the full fleet's capacity; we are not provisioning that just for the release.
- **Realistic-shape, not synthetic.** The load generator uses recorded production prompts (`mod-106` chapter 05 §4's traffic-shaping discipline), not uniform synthetic load.
- **Measured against the same SLIs as the release-gate contract.** No new signal is invented at release time; the signals are the ones the contract already reads.

## 2. Pre-promotion load test at canary scale

### 2.1 The goal

Before any user traffic hits the candidate, run the candidate under the load it would see at its first canary step. If the candidate holds the SLO at that load, promote it to the canary step with confidence. If it does not, hold the release and route the finding to the capacity-planning team (`mod-106`).

Expected catches:

- **Capacity ceiling moved.** A new model with larger KV cache footprint fits fewer concurrent sessions per replica than the previous model; the per-replica capacity dropped and the autoscaler's HPA target is wrong. Steady-state eval never hits this because eval is single-request-at-a-time.
- **GPU-saturation TTFT regression.** The new model's compute profile saturates the GPU differently; p95 TTFT at 70% utilisation is fine, p95 at 90% is not.
- **Memory headroom collapse.** The new model's KV cache grows faster with context length; a load mix that hits long-context requests at the same rate as before now pages into swap.
- **Downstream rate-limit discovery.** The candidate hits the provider's rate limit sooner because it retries more aggressively; the provider-side 429 rate appears only under load.

### 2.2 The mechanics

The load-test runs as a pipeline stage before the canary promotes to 1%. The release pipeline:

1. **Deploys the candidate** to a staging or pre-canary environment with the same per-replica capacity as the canary will have.
2. **Starts the load generator** with a recorded trace from `mod-106` chapter 05 §4, replayed at the target canary RPS.
3. **Runs for a bounded window** (10–30 minutes) with warmup bake-in.
4. **Reads SLIs** through the window: TTFT, TPOT, success rate, streaming stall, cost per request. Same signal names as the SLO-gate verdict (chapter 05 §3.2).
5. **Emits a verdict** in the same schema as the SLO-gate verdict — pass / fail against the thresholds in the release-gate contract.
6. **Attaches the verdict** to the release-gate decision.

```yaml
pre_promotion_load_test_verdict:
  verdict_schema: slo-gate-verdict-v1       # same schema as live SLO-gate
  candidate_sha: ef56ab78
  measured_from: pre-promotion-load-test
  measurement_window_seconds: 1800
  load_trace: prod-chat-replay-2026-02-14-peak
  target_rps: 48                             # target canary-step RPS
  dimensions:
    ttft_p95_ratio: 1.08
    tpot_p95_ratio: 1.03
    success_rate: 0.9998
    streaming_stall_rate_ratio: 1.11
    cost_per_request_ratio: 1.19
  verdict: pass
```

The verdict feeds the release gate exactly like the live SLO-gate verdict; the composition rule from chapter 05 §4 treats it the same way.

### 2.3 The trace replay discipline

The quality of the pre-promotion verdict depends on the trace. Four disciplines:

- **Representative prompt mix.** A recent production trace (last 24 h) captures the current prompt-length distribution, language mix, retrieval-topic coverage, and tenant mix. A replay of last quarter's traffic underestimates today's load.
- **Scrubbed for PII.** The trace is redacted per the compliance-telemetry contract (`mod-110` chapter 03). A replay that leaks PII into the pre-promotion environment is a secondary incident waiting to happen.
- **Concurrency-preserving.** The trace replays requests at the original concurrency pattern, not as a uniform rate. A naive uniform replay misses burst-shape effects that the recorded trace preserves.
- **Side-effect-stripped.** For agent routes, the trace's tool calls are mocked (chapter 03 §2.2). The replay must not call `send_email` 10,000 times.

The trace itself is a versioned artifact in a well-known storage path; the release-gate contract references it by name (`load_trace: prod-chat-replay-2026-02-14-peak`).

### 2.4 Scale beyond canary

A richer practice runs the load test at **multiple scales** and emits a capacity-ceiling signal:

- At the 1% canary target RPS: pass.
- At the 5% canary target RPS: pass.
- At the 25% canary target RPS: fail — TTFT p95 exceeded threshold.

The capacity-ceiling result tells the function the candidate can safely reach 5% but not 25% without additional replicas. The release pipeline either holds the canary at 5%, or schedules an autoscaler-target revision before the canary passes 5%. The conversation is at the `mod-106` chapter 05 unit-economics calculator.

## 3. Chaos probe of the new deploy before weight ramp

### 3.1 The release-time chaos suite

A small, fixed suite of experiments runs against the candidate before weight is ramped. The exact suite is per-fleet, but the canonical shape:

- **`provider-primary-timeout`.** The primary provider endpoint is replaced with a timeout sink for 60 seconds. Hypothesis: the fallback contract (`mod-107` chapter 02) engages, requests succeed via secondary, user-visible success rate stays above the SLO.
- **`pod-kill-one-replica`.** One candidate replica is killed mid-request. Hypothesis: the mesh retries to a sibling replica, in-flight requests complete (via idempotent retry) or fail gracefully with a known error class.
- **`prompt-cache-flush`.** The candidate's prompt cache is flushed mid-run. Hypothesis: TTFT p95 degrades but not beyond the pre-committed cliff; success rate holds; the fleet's `rb-cache-hit-cliff` runbook's early-warning fires correctly.
- **`retrieval-index-timeout`.** The retrieval service is forced to timeout. Hypothesis: the degraded-mode fallback (return answers without retrieval, or return a user-safe "cannot retrieve" message) engages.
- **`auto-halt-smoke`.** The chaos auto-halt itself is exercised: a synthetic SLO-burn signal is forced, the chaos operator aborts the experiment cleanly.

Each experiment is from the `mod-109` chapter 03 catalogue, run under the per-change tier's cadence (chapter 05 §1.1 of mod-109). The suite is a subset, not a new catalogue.

### 3.2 The verdict shape

Each experiment emits:

```yaml
release_chaos_experiment_verdict:
  verdict_schema: release-chaos-v1
  candidate_sha: ef56ab78
  experiment: provider-primary-timeout
  started_at: 2026-02-15T13:52:00Z
  duration_s: 60
  hypothesis: fallback engages within 2s, success rate stays >= 0.995
  observations:
    time_to_fallback_s: 1.3
    success_rate_during: 0.998
    mttr_s: 0.4
  verdict: pass
```

The aggregate across the suite is the gate's chaos-predicate:

```yaml
release_chaos_verdict:
  candidate_sha: ef56ab78
  experiments:
    - provider-primary-timeout: pass
    - pod-kill-one-replica: pass
    - prompt-cache-flush: pass
    - retrieval-index-timeout: pass
    - auto-halt-smoke: pass
  overall: pass
```

The release-gate reads the aggregate as part of the SLO-gate verdict (chapter 05 §3.2 `dependency_health` dimension), not as a separate predicate — the chaos probe is validating the reliability guarantees the SLO-gate already depends on.

### 3.3 The chaos probe runs against a hot green, not production

The chaos suite runs against the pre-warmed green environment (chapter 04 §3) *before* any production traffic is routed. This is the only tier where chaos fires against a candidate that is not serving real users. The experiments are tier-1 by `mod-109` chapter 01 §4 (production-safe blast radius) but on a controlled non-user-serving path.

An alternative composition some fleets use: the chaos suite runs against the shadow candidate (chapter 03) *during* the shadow window. This is cheaper (no extra pre-warmed environment needed) but has a complication: a chaos experiment during shadow may be interpreted by the response-diff pipeline as a candidate-side regression. The pipeline must be aware of the chaos window (annotation on the shadow log) so it does not mis-attribute the induced failure. The default is the dedicated green probe; the shadow-composed variant is an optimisation.

### 3.4 What is explicitly not in the release-time chaos suite

Per `mod-109` chapter 06's peer-boundary discipline, the release-time chaos suite does **not**:

- Inject prompt-injection payloads (security peer).
- Run jailbreak suites (security peer).
- Run benchmark sweeps against the model (eval peer).
- Probe safety-classifier gaps (safety peer).

The release-time suite is a reliability-only subset; the other peers' release-time checks run as part of the eval-gate verdict (chapter 05 §3.1) if at all.

## 4. SLO-burn as circuit breaker during ramp

### 4.1 The role

During the canary ramp (chapter 01) or during the first N minutes after a blue-green cutover (chapter 04), the function wants a hair-trigger burn-rate monitor that *stops the ramp* (not rolls back, not triggers a full incident) when the error budget is being consumed faster than expected. The control:

- **Fast-burn alarm on a short window.** The `mod-103` multi-window multi-burn-rate practice provides the shape. A 5-minute fast-burn at 14× steady-state fires; the canary's weight ramp pauses.
- **The ramp pauses rather than rolls back.** A pause allows the function to investigate (is this a transient? is the candidate actually worse? is the autoscaler catching up?) without discarding the release.
- **Pause has a dwell-timer.** The pause cannot exceed a fixed window (e.g., 30 minutes); past the dwell, the release pipeline either resumes the ramp (if the alarm cleared) or declares a failed release and rolls back.

### 4.2 The three circuit-breaker triggers

Three burn-rate families fire the pause:

- **Serving SLO burn.** Latency or availability burn at the fast-rate threshold.
- **Quality SLO burn.** Judge-score regression that crosses the chapter 05 contract's threshold (short-window version).
- **Cost SLO burn.** Cost per request exceeds the contract threshold over a 5-minute window.

Each is wired as an alarm the rollout controller reads. Flagger and Argo Rollouts both support pause-on-analysis-fail; the AI-adapted shape is that the analysis is on the SLO-burn-rate signal, not on the raw metric.

### 4.3 The pause's visibility

A pause during ramp is operationally visible:

- **Rollout dashboard shows `paused`.** Not `promoted`, not `rolling-back`.
- **The on-call channel receives an event.** Not an alarm (not a page); an informational event with a link to the rollout and the burn-rate graph.
- **The release pipeline refuses to advance** until a human or the dwell-timer resolves the pause.

### 4.4 Distinction from auto-rollback

Chapter 01 §5 described auto-rollback as the control that shifts traffic back to stable without human action. The circuit-breaker pause is different:

- **Rollback shifts traffic.** Pause does not; the canary continues to serve its current weight slice.
- **Rollback is a hard verdict.** Pause is a hold pending more information.
- **Rollback triggers are harder thresholds.** Pause triggers are faster, lower thresholds — the function expects pauses to be more frequent than rollbacks.

The composition: a pause that exceeds the dwell and does not clear becomes a rollback. A pause that clears resumes the ramp.

## 5. Composing the three release-time stages

The three stages — pre-promotion load test, chaos probe, circuit-breaker on ramp — fire in sequence:

```
 [eval-gate pass]
        │
        ▼
 ┌──────────────────────────────┐
 │ pre-promotion load test      │   pass → continue
 │ (chapter 07 §2)              │   fail → hold at gate
 └──────────────────────────────┘
        │
        ▼
 ┌──────────────────────────────┐
 │ chaos probe on hot green     │   pass → continue
 │ (chapter 07 §3)              │   fail → hold at gate
 └──────────────────────────────┘
        │
        ▼
 ┌──────────────────────────────┐
 │ canary ramp (chapter 01)     │   pause on fast-burn
 │ with SLO-burn circuit breaker│   (chapter 07 §4)
 │ (chapter 07 §4)              │
 └──────────────────────────────┘
        │
        ▼
    promote 100%
```

The sequence is enforced by the release pipeline — a candidate cannot skip a stage without an explicit override (chapter 05 §4.6).

## 6. The load-and-chaos-pairing in the release-gate contract

The release-gate contract (chapter 05 §2) adds two predicates to the contract document:

```yaml
predicates:
  pre_promotion_load_test:
    owner: ai-sre
    verdict_schema: slo-gate-verdict-v1
    source_of_truth: load-gate-service.release/verdict
    required_thresholds:
      serving:
        ttft_p95_ratio: { max: 1.15 }
        tpot_p95_ratio: { max: 1.10 }
        success_rate:   { min: 0.999 }
    trace: prod-chat-replay-2026-02-14-peak
    missing_verdict_behaviour: hold

  release_chaos_probe:
    owner: ai-sre-chaos-lead
    verdict_schema: release-chaos-v1
    source_of_truth: chaos-gate-service.release/verdict
    required_experiments:
      - provider-primary-timeout
      - pod-kill-one-replica
      - prompt-cache-flush
      - retrieval-index-timeout
      - auto-halt-smoke
    missing_verdict_behaviour: hold
```

The composition is AND with the existing predicates: eval-gate ∧ slo-gate ∧ pre_promotion_load_test ∧ release_chaos_probe. The pause-on-burn during ramp is enforced at the delivery-mechanism layer (the rollout controller), not as a release-gate predicate — the rollout controller is the one that can mechanically read the burn-rate alarm and pause.

## 7. Failure modes named explicitly

Five anti-patterns the discipline prevents:

- **"The pre-promotion load test is a smoke test."** It tests only liveness, not SLI. A pre-promotion load that passes with p99 = 10× baseline is not a pre-promotion load; it is a liveness check. The verdict schema (§2.2) names the SLIs explicitly to close this.
- **"The chaos probe is skipped because the candidate is 'just a config change.'"** Config changes are the stream with the fastest change-failure feedback and the easiest fallback regression. A config change goes through the same chaos probe or the gate rejects it.
- **"Pauses during ramp are rare, so we don't invest in the signal."** Pauses are the control that lets the ramp run autonomously; a function that cannot pause mid-ramp is a function that either rolls back on small signals (noisy) or ignores small signals (dangerous).
- **"We run chaos quarterly; we don't need a release-time probe."** The quarterly chaos (`mod-109` chapter 05 §1.4) is a different cadence; the release-time probe fires per release. Quarterly chaos finds known-class regressions; release-time chaos finds this-release regressions.
- **"The load test uses synthetic prompts."** Synthetic prompts miss the long-tail behaviours that production prompts hit. The trace-replay discipline (§2.3) is non-negotiable.

## 8. The AI SRE's pairing checklist

Before enabling load-and-chaos pairing on a fleet's release pipeline:

1. **A production-shape trace exists**, scrubbed, versioned, stored, and referenced by the release-gate contract.
2. **The pre-promotion load test runs as a pipeline stage** with the verdict emitted in the same schema as the SLO-gate.
3. **The chaos probe suite is defined** (three to five experiments from `mod-109` chapter 03's catalogue) and runs on hot green or shadow.
4. **The chaos probe does not inject prompt-injection / jailbreak / safety-classifier probes** per the peer-boundary discipline.
5. **The circuit-breaker on ramp is wired** to fast-burn alarms on serving, quality, and cost signals.
6. **Pauses are visible** in the rollout dashboard and the on-call channel; auto-rollback is distinct from pause.
7. **The release-gate contract names the two new predicates** with thresholds, sources of truth, and missing-verdict behaviour.
8. **A dry-run** proves the pipeline holds on a synthetic failing verdict for each of the two new predicates.
9. **The quarterly chaos program (`mod-109` chapter 05 §1.4) remains in place** and is not displaced by the release-time probe.
10. **The DORA change-failure attribution (chapter 06 §4) includes** auto-rollback triggered by a circuit-breaker-after-dwell failure.

A pairing that passes the ten is a pairing that catches the class of regression that slips past eval and SLO verdicts in isolation.

## Summary

- Release-time load-and-chaos pairing is what keeps the DORA four-keys from telling a story the function would rather not tell.
- The pre-promotion load test runs the candidate at canary-scale load against a production-shape trace, emits a verdict in the SLO-gate schema, and gates the first canary step.
- The chaos probe is a small suite of experiments from the `mod-109` chapter 03 catalogue, running against hot green or shadow, with a verdict that composes into the release gate.
- SLO-burn during ramp fires a circuit-breaker **pause**, distinct from auto-rollback — the ramp holds, a human or the dwell-timer decides whether to resume or roll back.
- The three stages compose with the eval-gate and SLO-gate from chapter 05 as AND predicates; the release pipeline enforces the sequence mechanically.
- Named anti-patterns (smoke-test load, skipped chaos on config changes, pause invested-under, quarterly-only chaos, synthetic prompts) are what the discipline prevents.
- A ten-point checklist gates the first enablement per fleet.

Chapter 08 closes the module with the peer-boundary discipline: which release-engineering work is this module's, and which is at the inference-engine, mlops, and eval-engineer peers.
