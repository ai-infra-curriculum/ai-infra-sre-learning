# The Continuous Chaos Program and the Game-Day Shape

## Why this chapter exists

A chaos catalogue is a shelf. A chaos program is the operational discipline that makes sure experiments from the shelf run on a cadence, under blast-radius controls, with auto-halt wired to the SLO stack, and with results that close the loop by producing permanent fixtures. Google's Site Reliability Engineering tradition — specifically the DiRT (Disaster Recovery Testing) program — is the shape most mature engineering organisations borrow; Netflix's Simian Army and the follow-on at LinkedIn / Alibaba / Microsoft are variants. This chapter writes the shape for the AI SRE function.

The chapter sits where the module's abstract pieces become a program the organisation *runs*. It lays out the four-tier cadence, the governance (policy, review, Principal / Staff involvement), the auto-halt wiring, and the feedback loop that back-feeds the fixture ledger from `mod-108` chapter 05. The exercise for this chapter (exercise 06) authors the game-day plan for a chosen scenario.

## 1. The four-tier cadence

The cadence from chapter 01 §5 is the floor. The four tiers below are the production shape.

### 1.1 Per-change (continuous)

- **Scope.** A reduced suite — three to five fast experiments — runs on every router / serving-image PR before promotion. Chapter 04's provider-fallback contract smoke suite is here.
- **Environment.** Staging or a canary slice.
- **Duration.** 5-minute windows per experiment; suite ≤ 20 minutes end-to-end.
- **Mechanism.** CI pipeline job that calls the chaos operator with a scripted suite CRD; results post to the PR as a check.
- **Gate.** A regression on any metric blocks merge.

### 1.2 Daily (overnight)

- **Scope.** A rotating 25% slice of the full chapter 03 catalogue, plus the contract-chaos suite from chapter 04.
- **Environment.** Pre-production cluster with production-like traffic replay (shadow traffic, if available).
- **Duration.** 2-hour window overnight. Each experiment runs for 10–15 minutes with blast-radius controls per its catalogue entry.
- **Mechanism.** Chaos Mesh `Schedule` CRD or LitmusChaos `ChaosSchedule`. Results post to a chaos channel (dedicated, not the on-call channel) by morning.
- **Gate.** A regression opens a ticket to the owning peer; a trend of regressions escalates to the quarterly review.

### 1.3 Weekly (production slice)

- **Scope.** The same catalogue, run against a 1% production slice with tight blast-radius controls and auto-halt.
- **Environment.** Production.
- **Duration.** 30-minute total window during the week's lowest-traffic period. One or two experiments per window.
- **Mechanism.** Same operator CRDs, with explicit cohort filters and the auto-halt wired to SLO-burn and safety-signal alarms.
- **Gate.** Auto-halt is a chaos result, not a failure of the program. A chaos run that was auto-halted produces a ticket to the peer who owns the symptom.

### 1.4 Quarterly (game-day)

- **Scope.** A composed scenario drawn from the chapter 03 catalogue and / or chapter 04's suite, with SME participation across peers, a documented runsheet, and a published retrospective.
- **Environment.** A mix of production and staging as the scenario requires. Full-scale cross-AZ or cross-region drills run in staging with production-parity data; smaller drills run in production under policy.
- **Duration.** Half a day, with a 60–90-minute active-experiment window inside a longer review-and-post-mortem block.
- **Mechanism.** The function's chaos lead orchestrates; chapter 05 §4 of this chapter writes the game-day shape in detail.
- **Gate.** Every game-day produces a published retrospective with at least one fixture (`mod-108` chapter 05).

## 2. Auto-halt — the control that lets the program run

A chaos program without a reliable auto-halt is a program the organisation cannot run in production. Auto-halt is the mechanism that mechanically terminates an experiment when a pre-declared threshold is crossed. It must satisfy:

1. **Independent of the experiment itself.** The halt signal cannot live in the chaos controller alone; it is an alarm on a production dashboard that the chaos controller subscribes to. If the chaos controller is wedged, the alarm reaches it or kills the experiment via a side channel.
2. **Multiple thresholds.** Serving (SLO burn), quality (judge / safety spike), and cost (per-request cost). Each is wired separately. Any of the three firing halts the experiment.
3. **Verifiable in a dry-run.** The halt is tested on a staging run before the experiment is promoted to production.
4. **Observable in the incident channel.** When an experiment auto-halts, the event appears in the incident / chaos channel within seconds, with a link to the chaos run, the alarm that fired, and the metric trace.

### 2.1 Example wiring: Prometheus → Chaos Mesh

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: HTTPChaos
metadata:
  name: inject-primary-stall
  namespace: inference
spec:
  mode: fixed
  value: "1"
  selector:
    labelSelectors:
      app: summariser
      chaos.target: "true"
  target: Request
  port: 443
  method: POST
  delay: "45s"
  duration: "15m"
  stopCondition:
    # Chaos Mesh's workflow allows a webhook or Prometheus alert to halt
    prometheusAlert:
      - name: slo-burn-5min-summariser
        namespace: monitoring
      - name: safety-signal-spike-inference
        namespace: monitoring
      - name: cost-per-request-spike-summariser
        namespace: monitoring
```

(The CRD shape shown is a close approximation; the precise schema is in Chaos Mesh's docs. Exercise 02 wires a real one.)

### 2.2 Example wiring: AWS FIS → CloudWatch

AWS FIS has first-class stop-conditions that tie experiments to CloudWatch alarms. Chapter 02 §4.3 shows the template. The AI SRE's addition: every FIS experiment references *three* alarms, not one — the serving, quality, and cost alarms.

### 2.3 Example wiring: Human-visible kill switch

Even with auto-halt, the chaos program ships a one-command abort for the on-call. `kubectl delete chaos ...` works; a dedicated `chaos abort <run-id>` CLI is cleaner. The on-call knows where to find it from the runbook `rb-chaos-experiment-active` (named in chapter 02 §6).

## 3. The blast-radius policy document

Every chaos program has a written policy. It is one document, signed by the function lead and the safety / policy peer, reviewed quarterly, and consulted before each experiment is promoted to a new tier. The sections:

1. **Scope.** Which fleets, which clusters, which cohorts the policy covers.
2. **Tier definitions.** Tier-1 (production-safe) and tier-2 (staging-only) per chapter 01 §4. The rubric for deciding tier is spelled out: side-effect-creating? → tier-2; safety-filter-dependent? → tier-2; weight-level corruption? → tier-2 (chapter 03's no-chaos list).
3. **Per-tier caps.** Cohort, traffic fraction, time, and auto-halt per tier. The weekly-production experiments from §1.3 are tier-1 with 1% / 15-minute caps.
4. **The no-chaos list.** Chapter 03 §11's entries plus any environment-specific additions.
5. **Reviewer slate.** Who signs off on a new experiment entering each tier. Changes from tier-2 to tier-1 require two signatures (function lead + safety peer); a tier-1 catalogue addition requires one (function lead).
6. **Incident interaction.** What the on-call does if a real incident fires during a chaos window. Default: auto-halt is expected to fire; if it does not and the incident is unrelated, the chaos run is aborted manually and the result is not held against the program. If the chaos run may be contributing, the incident takes priority per `mod-108` chapter 02.
7. **Audit.** Every chaos run is logged with its ID, operator, scope, result, and ledger linkage.

The policy is version-controlled in the function's documentation repo alongside `mod-108` chapter 02's command structure doc. Treat it as a hard dependency, not a nice-to-have.

## 4. Game-day shape

The quarterly game-day is the practice that catches what continuous automation cannot — the human-in-the-loop decisions, the cross-peer coordination, the comms-under-pressure muscle. The Google SRE DiRT program and Netflix's ChAP (Chaos Automation Platform) describe their variants publicly; the shape below is the AI SRE adaptation.

### 4.1 Pre-game-day (one to two weeks ahead)

- **Pick a scenario.** Draw from a single, composed experiment (e.g., "primary provider region-red with a correlated judge-model unavailability"). The scenario is written down; participants read it in advance.
- **Name the roles.** IC (`mod-108` chapter 02), OL for the AI SRE function, SMEs from `ai-infra-inference-engineer`, `ai-infra-performance`, `ai-eval-engineer`, `AI Engineering`, `ai-risk-engineer`, FinOps, safety / policy (as scenario requires), scribe. A Principal / Staff engineer participates as observer or IC per §4.4.
- **Agree the blast radius.** Cohort, traffic fraction, time, auto-halt. Signed in the policy document.
- **Agree the kill criteria.** "If customer-reported safety events exceed N in the window, we end." "If fallback-hit-rate drops below 90%, we end."
- **Pre-committed decision tree.** `mod-108` chapter 02 §6's safety-adjacent tree is read in advance so a decision is not improvised during the drill.

### 4.2 Game-day execution

- **Open with context.** The IC states the scenario, the roles, the kill criteria, the scope, and the clock. Five minutes.
- **Trigger.** The chaos operator runs the composed experiment. The event stream appears in the incident channel.
- **Observation.** Each role runs against the scenario as if real: on-call reads the runbook (`mod-108` chapter 01), IC runs the channel per `mod-108` chapter 02, OL coordinates mitigations, SMEs diagnose. Comms stays quiet (this is a drill) but drafts the status-page template as if public.
- **Checkpoints.** Every 15 minutes the IC calls a checkpoint: "where are we against the hypothesis? are kill criteria at risk?"
- **End.** Either the hypothesis is validated, the kill criteria are hit, or the time cap expires. Any of the three is a valid ending.

### 4.3 Post-game-day

- **Retrospective within 48 hours.** Blameless post-mortem shape (`mod-108` chapter 03). Attendees: all game-day participants, plus any peer the fixtures land on.
- **Fixtures.** Every game-day produces at least one of each from the fixture ledger (alert, canary, eval entry, load shape, chaos amendment, runbook update). The Candidate → Owned → Implemented → Verified → Active → Measured cycle from `mod-108` chapter 05 §2 runs on them.
- **Publication.** The retrospective is published internally (or externally if the scenario does not reveal sensitive information). Pattern-matching against public post-mortems (`mod-108` chapter 04) should be able to reference the game-day over time.

### 4.4 Principal / Staff-engineer involvement

The quarterly game-day is a Principal / Staff-visible event because:

- **It exercises cross-peer judgement that no one else owns.** The IC role is often a Principal or Staff engineer by rotation — the game-day is the practice that keeps the muscle current.
- **It surfaces architectural risk that only a Principal / Staff engineer can act on.** "The router's quality signal is stale by 10 minutes" is a finding whose fix lands at a scope that below-Principal / Staff engineers rarely own.
- **It defends the chaos program's political capital.** A Principal / Staff engineer who ran the last game-day is the right person to defend the program at budget time.

Scheduling: one Principal / Staff engineer attends every quarterly game-day in a named role (IC, OL, or observer with sign-off on the retrospective). The expectation is written in the function's charter.

## 5. The policy–execution–feedback cycle

The program is not three isolated pieces; it is a cycle.

```
    Policy (this chapter §3)
          │
          ▼
    Execution (§1 cadence × chapter 03 catalogue × chapter 04 suite)
          │
          ▼
    Fixture back-feed (mod-108 chapter 05)
          │
          ▼
    Runbook library (mod-108 chapter 01)   ──→   next policy review
```

The cycle produces:

- **Fewer incidents** of the classes the catalogue covers, because the runbook has been exercised against a staged version of the incident class.
- **Faster mitigation** when an incident of a new class fires, because the operational muscle for recognising a chaos signal and executing against a runbook is current.
- **A quantitative story** (MTTD / MTTR / fallback-hit-rate trends from chapter 04, auto-halt rates from §2) the function brings to leadership on the quarterly cadence.

### 5.1 Reading a program's maturity

A rough staircase, so a function can place itself:

- **Level 0.** No chaos program; failure injection happens during real incidents.
- **Level 1.** Ad-hoc game-days once or twice a year, no policy document, results not persisted.
- **Level 2.** Scheduled game-days, policy document exists, results back-fed by hand.
- **Level 3.** Daily automated runs on pre-production, weekly on production slice, auto-halt wired, fixture ledger operational, Principal / Staff involvement in quarterly game-days.
- **Level 4.** Contract-chaos suite is a release gate; chaos findings routinely originate bug fixes in the owning peer's code; the function presents chaos trends at the quarterly reliability review.

The AI SRE function should be at level 2 within six months of adopting this module, level 3 within a year, level 4 within two years. Faster is possible; much faster is unlikely to be stable.

## 6. Pitfalls this chapter is calibrated against

A short list of failure modes the program must defend against. Each shows up at every function that adopts chaos; naming them makes them less likely to land here.

- **The theatre pitfall.** Game-days become press releases ("we ran a chaos drill") without a published retrospective, fixtures, or trend. Defence: §4.3 is mandatory; the ledger linkage is non-negotiable.
- **The auto-halt-is-flakey pitfall.** Auto-halt fires on noise, the on-call tunes it out, the next genuine halt is ignored. Defence: §2.1's dry-run protocol, and the policy §3.6 rule that a flakey halt is a chaos bug, not a program failure.
- **The accumulation pitfall.** The catalogue grows faster than the organisation can run it. Defence: chapter 03's "run three times before adding one" rule.
- **The over-ambition pitfall.** A new program schedules a cross-region game-day in month one. Defence: §5.1's maturity staircase; a level-3 practice is a year of groundwork.
- **The owner-less-fixture pitfall.** Chaos finds something, nobody owns it, it sits in the ledger forever. Defence: `mod-108` chapter 05 §2's state machine, with the Owner field required.
- **The decoupling pitfall.** Chaos lives in the SRE function's documentation; the owning peers never read it. Defence: §4's Principal / Staff and SME participation; §5's cross-peer sign-offs on new experiments.

## 7. The program-authoring worksheet

Exercise 06 produces the environment-specific version of this program. The worksheet shape:

1. **Draft the policy document (§3).** Scope, tier definitions, caps, no-chaos list, reviewer slate, incident interaction, audit. One document, version-controlled.
2. **Pick the cadence (§1).** Per-change, daily, weekly, quarterly. Name the environments.
3. **Wire the auto-halt (§2).** Three alarms per experiment (serving, quality, cost). Dry-run the halt.
4. **Pick the first quarterly scenario (§4).** A composed scenario within the function's current maturity. Document the roles, the kill criteria, the pre-committed decisions.
5. **Hook the ledger (§5).** Every run's ID logs into the fixture ledger with the outcome; trend reports feed the quarterly reliability review.
6. **Secure the Principal / Staff slot (§4.4).** In writing, in the function's charter.

## Summary

- The continuous chaos program has four cadences: per-change CI smoke, daily pre-production, weekly 1% production slice, quarterly game-day.
- Auto-halt is the control that lets the program run in production. It uses three independent alarms (serving / quality / cost), is wired to Prometheus / CloudWatch / equivalent, and is dry-run on every new experiment.
- The blast-radius policy is a signed document (function lead + safety peer) with tier definitions, caps, a no-chaos list, a reviewer slate, and incident-interaction rules.
- The game-day shape is a cross-peer drill with SME participation, a pre-committed decision tree, and a published retrospective producing fixtures.
- Principal / Staff involvement in every quarterly game-day is a function-charter commitment, not a convention.
- The program's maturity moves on a staircase (levels 0–4). Six months to level 2, a year to level 3, two years to level 4 — faster is rarely stable.

Chapter 06 closes the module by drawing the boundary between this program (reliability chaos) and the peer programs — `ai-infra-security`'s adversarial robustness and `ai-eval-engineer`'s model-eval robustness.
