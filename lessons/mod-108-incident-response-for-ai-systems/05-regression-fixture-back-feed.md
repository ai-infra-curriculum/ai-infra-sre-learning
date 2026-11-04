# Regression-Fixture Back-Feed Discipline

## Why this chapter exists

A post-mortem that produces no durable change is a cost without a return. The organisation paid the cost of the incident — user impact, engineering time, trust — and if the only artifact is a filed document, the same incident will recur. Classical SRE knows this; the back-feed discipline is the glue. The AI SRE's version of the discipline has to work in a wider set of substrates, because the signals that catch AI incidents live in more places than the classical SRE's two surfaces (metrics / logs). This chapter codifies the back-feed cycle, defines what counts as a fixture, specifies the six fixture classes, and gives the SLO-stack closure contract that makes the cycle auditable.

The premise is one sentence: **every post-mortem produces at least one permanent fixture that, if it had existed, would have caught the incident earlier or mitigated it faster**. The remainder is engineering practice.

## 1. The six fixture classes

The AI SRE function's fixture-back-feed vocabulary has six classes. Each class is tied to a surface that is already standing — the SLO stack, the observability stack, the eval system, the load-test harness, the chaos program, or the runbook library. New surfaces are rare; the back-feed usually adds to an existing one.

### 1.1 Alert (SLO / metric / log)

A new or amended threshold alert or burn-rate alert in the observability stack. The alert fires on a signal that was present during the incident but was not alerted on. Example: the `prompt_cache_hit_ratio` was visible on a dashboard but no alert existed; the incident proved the metric is worth alerting on; the fixture is a 10-minute threshold alert at a configured drop percentage.

An alert fixture is owned by the metric's maintainer and verified by exercising a staged dip in staging.

### 1.2 Canary eval case

A new eval case (prompt, retrieval query, structured-output example, tool-call shape) added to the canary eval set that runs on a cadence — pre-promotion for the weights registry, on every deploy for the application, nightly for the vendor-eval surface. The case is the exact failing shape from the incident, generalised to the smallest class that would catch the pattern.

A canary eval fixture is owned by the eval engineer and verified by running the new case against the pre-incident model / config (should fail) and against the post-fix model / config (should pass).

### 1.3 Eval-set entry

A permanent addition to the regression eval set used by `ai-eval-engineer` for model-promotion decisions. Differs from a canary case in persistence and purpose: canary cases live in a fast, small, frequent-run set; eval-set entries live in the larger, slower, authoritative set used to make a promotion call. The same incident may produce both — a fast-signal canary and a permanent eval-set case.

### 1.4 Load-test shape

A new load-test pattern — arrival shape, prompt-length distribution, batch composition, long-tail concurrency burst — that reproduces the resource or latency behaviour from the incident. Lives in the load-test harness `mod-106` chapter 06 teaches.

A load-test fixture is owned by performance / inference engineering and verified by running the shape against the pre-incident configuration (should reproduce the symptom) and the post-fix configuration (should not).

### 1.5 Chaos experiment

A reproducible fault-injection experiment that creates the incident's root-cause shape on a canary or staging environment on a scheduled cadence. The chaos program (`mod-109`) is the home. Examples: inject an XID error on a GPU to verify the drain-and-reschedule runbook, return a looping error shape from a tool to verify the tool-call cap, flip the vendor's fallback breaker to verify the router.

A chaos fixture is owned by the AI SRE function (or the chaos team if the org has one); verification is that the experiment ran on schedule and the detection and mitigation both worked.

### 1.6 Runbook update

A new or amended runbook in the top-twenty library (chapter 01) or a related class-specific runbook. Updates include new confirm-checklist steps, new mitigation options, severity-ladder refinements, and corrections to the peer-owner routing.

A runbook fixture is owned by the AI SRE function, reviewed by the peer owner of the surface the runbook touches, and verified by replaying the incident timeline against the new runbook with a different on-call persona.

### 1.7 Fixtures that are usually *not* valid on their own

The following are frequently proposed and should be challenged:

- **"Document the lesson learned."** Documentation is a complement to a fixture, not a fixture. If the documentation is not consumed by code (a lint, a schema check, a template rule), it is cosmetic.
- **"Educate the team."** Chapter 03 §7 covers this. If humans must remember to avoid a recurrence, the system is unchanged.
- **"Open a vendor support ticket."** A ticket is not a fixture. A fixture that depends on the vendor landing a change is a plan with a dependency, not a durable change.
- **"Tighten the review process."** Unless it is implemented in CI or a PR template that is enforced, review tightening is a wish.

The reviewer's job is to spot these and ask for the implementation-shaped version.

## 2. The back-feed cycle

The cycle is a small state machine. The post-mortem fills in the fixtures section (chapter 03 §2.9) with candidates; the cycle turns candidates into landed, verified, measured fixtures.

```
Candidate ─▶ Owned ─▶ Implemented ─▶ Verified ─▶ Active
                                                   │
                                                   └─▶ Measured (did it fire / detect / hold?)
```

### 2.1 Candidate

The post-mortem's fixture back-feed list. Each candidate has a class, a proposed system, and a one-line description. The candidate may not yet have an owner; the retro reviewer assigns one before the retro closes.

### 2.2 Owned

A named owner on a named rotation, and a due date. The owner is often not the retro author — fixtures cross systems and the eval entry may belong to the eval engineer while the alert belongs to the SRE team.

### 2.3 Implemented

The fixture is in code / config / schedule. A PR has merged, an alert is deployed, a canary case is in the eval set, a chaos experiment is in the scheduler.

### 2.4 Verified

The fixture has been exercised against the pre-incident baseline and shown to catch the incident. The verification is explicit: a staging run of the chaos experiment, a replay of the pre-incident trace through the new alert, a run of the canary set against the pre-fix model. Fixtures that are not verified are silent.

### 2.5 Active

The fixture is on the production cadence, firing / running / evaluating against real traffic or scheduled exercise.

### 2.6 Measured

Over time, did the fixture fire? Did it catch anything? If it never fires, it is either calibrated too loose to be useful or the class it was built for does not recur. Both are valid outcomes; the function tracks the fire rate on the quarterly review.

## 3. The fixture schema

Fixtures are machine-readable so the function can audit them. The schema matches the post-mortem block from chapter 03 §2.9 and adds verification / tracking fields:

```yaml
fixture_id: fx-2026-11-04-structured-output-01
origin_incident: inc-2026-11-04-structured-output-summariser
class: alert
system: prometheus
name: structured_output_validator_error_rate_high
threshold: ">= 3× 7-day baseline for 10m"
runbook: rb-structured-output-break
owner: ai-infra-inference-engineer-oncall
due_date: 2026-11-18
status: active
verification:
  method: replayed pre-incident trace through alert rule
  date: 2026-11-10
  outcome: alert would have fired 23 minutes before the user-visible impact
measurement:
  first_live_fire: 2026-12-02 on the summariser route, mitigated within SLO
  subsequent_fires: 2
```

Fixture IDs are durable and are referenced by the post-mortem that produced them and by any subsequent incident whose root-cause class matches.

## 4. Closing the SLO-stack loop

The SLO stack (from `mod-103`) is the organising surface the fixtures live in. The back-feed closes the loop:

- A **burn-rate alert** fired → an **incident** → a **post-mortem** → a **new alert / canary / eval / load / chaos / runbook** → the next incident of the same class is caught by the new fixture or the fixture measures its absence.

This is the SRE classical closed loop, specialised with the AI-specific fixture classes. The closure is visible in the SLO stack when:

- The burn-rate alert has a linked runbook that is current.
- The runbook references detection signals that are all live.
- The canary eval set includes cases from prior incidents.
- The load-test harness runs shapes that reproduce prior resource events.
- The chaos program exercises the failure classes the fixtures cover.

The AI SRE function's leadership dashboard shows this closure explicitly: for each SLO, the number of linked fixtures and the last-fire / last-exercise date per fixture class. A SLO with no linked fixtures is a SLO without memory.

## 5. Back-feed hygiene

Two forces erode the back-feed over time. The function mitigates them explicitly.

### 5.1 Alert fatigue

Every incident wants a new alert. Over months, the alert noise grows faster than real signal, the on-call pages on cosmetics, and the system is fatigued into ignoring all alerts equally. Chapter 01's runbook library and `mod-103`'s SLO discipline bound this — alerts must correspond to user-visible SLO-committed symptoms, not arbitrary metrics — and the back-feed cycle respects that bound.

- **Prefer burn-rate alerts.** They are self-adjusting to severity. Threshold alerts are reserved for hardware / infrastructure classes where burn-rate is not meaningful (e.g., ECC double-bit).
- **Prefer multi-signal joint conditions.** A cache-hit-rate drop alone is an anomaly; a cache-hit-rate drop *coincident with a TTFT regression on the same route* is an incident. Joint conditions reduce false-positive pages.
- **Tune down before off.** When an alert is noisy, raise its window or its threshold; do not delete it. The signal is still a signal.
- **Review quarterly.** Any alert that has not fired in six months is reviewed for relevance; the decision is explicit (keep, tune, retire) and recorded.

### 5.2 Fixture obsolescence

Systems change. A fixture bound to a specific metric name / dashboard / runbook / eval slug rots when the underlying surface moves. Fixtures are versioned with their origin incident and reviewed on the same cadence as the runbook library (chapter 01 §5). A fixture whose target has moved is either re-pointed or retired explicitly.

## 6. Scaling the back-feed across many incidents

A function that lands two or three fixtures per month will grow a healthy library in a year. The failure mode is not under-production; it is under-coordination. Two practices:

### 6.1 The fixture ledger

A single structured store (a YAML / JSON directory, a Backstage plugin, a lightweight service) that holds every fixture. The ledger is the source the leadership dashboard reads; the post-mortem block chapter 03 §2.9 is the ingest; the quarterly review of the library is the audit.

### 6.2 Cross-incident de-duplication

When two incidents in a quarter produce overlapping fixture candidates, the function reviews and merges. Two runbooks for the same class become one; two alerts for the same signal become one; two canary cases testing the same regression become one. The reviewer's test: can one on-call person follow the fixture without being confused by the duplicates?

## 7. The AI-specific back-feed traps

The classical back-feed cycle has traps the AI SRE inherits and three traps that are AI-specific:

### 7.1 Eval-set contamination

A naive back-feed adds a failing input to the eval set. If that input is also in the training set used to fine-tune the next model version, the model sees the fix at training time and the eval no longer tests what it claims to test. Chapter 03 §3.8's eval-set poisoning class covers the surface; the back-feed cycle mitigates by:

- Keeping an explicit training-eval boundary (an index that enforces "do not train on items in the regression eval set").
- Hashing the eval items and checking training data against the hash set on ingest.
- Preferring behavioural / templated eval cases over verbatim-input cases when the model can memorise.

### 7.2 Judge-model drift invalidating a fixture

An eval or canary that depends on a judge model is only as reliable as the judge's stability. A judge-model drift (`rb-judge-model-drift`) invalidates every fixture that reads the judge until recalibration. The back-feed should pin the judge model id and version in the fixture definition and raise an alarm if the judge changes without recalibration.

### 7.3 Prompt-cache invalidation via the fixture

An eval that is structured with a leading canary token or test marker may itself invalidate the production prompt-cache prefix (if it shares a template namespace) or degrade production latency if run on the live route. Fixtures that touch the production path must either run on a separate cache namespace or be run against a traffic-shadowed route.

## 8. The incident → fixture decision tree

The retro owner walks a short decision tree per contributing factor:

- **Is the factor an observability gap?** → alert fixture.
- **Is it a quality / behaviour gap?** → canary eval case and / or eval-set entry.
- **Is it a capacity / performance gap?** → load-test shape.
- **Is it a resilience / failure-mode gap?** → chaos experiment.
- **Is it a procedural / routing gap?** → runbook update.
- **Is it a policy / contract gap?** → not a fixture in this cycle; a separate policy ticket with legal / safety.

Multiple factors produce multiple fixtures. One contributing factor that touches two surfaces (e.g., "silent model-version update that evaded both the alert and the canary") produces two fixtures.

## 9. Verifying the cycle externally

Every fixture claim is verifiable:

- **The alert fired at the expected scope in staging.**
- **The canary case fails on the pre-incident configuration and passes on the fix.**
- **The load shape reproduces the symptom in a load-test environment.**
- **The chaos experiment runs on a scheduled cadence and catches the fault.**
- **The runbook change is reviewed and signed off by the peer owner and the AI SRE function.**

Verification is recorded in the fixture record (§3). The audit test for the library's health is: for each fixture tagged "verified," the record cites a specific exercise and a specific outcome. "Verified by inspection" is not verification; it is a placeholder for not having done the work.

## 10. The AI SRE's back-feed worksheet

1. **For each post-mortem**, walk chapter 03 §7's six-class decision tree (§8 above) per contributing factor.
2. **Write the fixture candidates** into the post-mortem's §2.9 block with class, system, and one-line description.
3. **Assign owners and due dates** before closing the retro.
4. **Implement**, **verify**, and **activate** on the schedule.
5. **Register in the fixture ledger** with the schema from §3.
6. **Measure.** Track live fires, exercised detections, and absent fires on the quarterly review.
7. **Prune.** Retire fixtures whose surface has moved; de-duplicate overlapping fixtures.
8. **Report.** The SLO-stack dashboard shows the number of linked fixtures and their last-exercise dates per SLO.

## Summary

- **Every post-mortem produces at least one durable fixture.** No fixture means the retrospective is incomplete.
- **Six classes.** Alert, canary eval, eval-set entry, load-test shape, chaos experiment, runbook update. Each maps to a standing surface.
- **The cycle is a state machine.** Candidate → Owned → Implemented → Verified → Active → Measured. Each transition is explicit.
- **Close the SLO-stack loop.** Burn-rate alert → incident → post-mortem → fixture → the next incident of the same class is caught earlier.
- **The AI-specific traps are real.** Eval-set contamination, judge-model drift, and prompt-cache invalidation require explicit discipline.
- **Fixtures are auditable.** Verified means exercised and recorded, not asserted.

Chapter 06 picks up the thread the back-feed cycle does *not* close: the incidents that cross into legal, compliance, and regulator-facing surface, where the AI SRE's output is not a fixture but a precise handoff to the people who formally report.
