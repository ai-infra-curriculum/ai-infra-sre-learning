# Eval-Set Reliability

## Why this chapter exists

Every reliability decision this track has described — a canary pass / fail (`mod-111`), a release-gate promotion (`mod-111` chapter 05), a judge-score SLO burn (`mod-103` chapter 02), a prompt-change rollout (chapter 01), a retrieval probe-hit SLO (chapter 02), a feature-set release — ultimately resolves against **an eval**. The eval verdict is the predicate that decides whether a candidate ships, holds, or rolls back. Which means: **if the eval set itself is broken, the gate is broken, and every signal the gate feeds is wrong.**

Eval-set reliability is therefore the function's problem, even though eval-set *construction* is not. Eval methodology — writing the judge prompt, authoring the labels, calibrating against human raters, choosing the benchmark mix — is the `ai-eval-engineer` peer's and the `model-evaluation-engineer` peer's work (both at level 30 per `mod-101`). The function does not author eval sets. The function owns the reliability contract on the eval surface: **detecting when the eval set has drifted, rotted, or been poisoned, and refusing to let a compromised eval set function as a release predicate.**

This chapter is the authoring reference for that contract. It defines the three failure modes (drift, poisoning, rot), the detection signals, the peer handoff, and the operational response when an eval set is compromised. Primary references: [OpenAI Evals](https://github.com/openai/evals), [Inspect AI](https://inspect.aisi.org.uk/), [Promptfoo](https://www.promptfoo.dev/docs/), [LangSmith evaluations](https://docs.langchain.com/langsmith/evaluation-concepts), [Weights & Biases Weave Evaluations](https://wandb.ai/site/weave/), and the calibration literature — [Zheng et al., *Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena*](https://arxiv.org/abs/2306.05685) is a canonical reference for the agreement-with-human-labellers framing.

## 1. The three eval-set failure modes

Named up front, because each has a distinct detection path and a distinct remediation.

### 1.1 Eval-set drift

The eval set is static. Production traffic is moving. Over time, the distribution of what users actually ask drifts away from what the eval set represents. The eval predicate keeps passing, but it is passing on a world that no longer matches production. The gate approves candidates that look good on the eval and regress in production.

Drift is slow. Detecting it requires measuring the **representativeness** of the eval set against the current production distribution. §3 writes the detector.

### 1.2 Eval-set poisoning

An adversary (or an accident) injects labelled samples into the eval set such that a specific model or prompt family passes the eval while actually regressing. In the adversarial case the attacker's model is designed to recognise eval inputs; in the accidental case the eval author copy-pasted from a corpus the model under test was trained on.

Poisoning is the eval-side analogue of training-data poisoning, and the most load-bearing specific sub-case is **eval-train contamination** — the eval samples have leaked into some model's training data, so that model gets the right answer by memorisation rather than by capability. §4 writes the detector.

### 1.3 Eval-set rot

The eval set is still representative and uncontaminated, but every candidate now passes it trivially. The set no longer *distinguishes* candidates. Rot usually happens because the capability the eval tested has become table-stakes across the model population; the set was useful in 2024 and is uninformative in 2026.

Rot is the subtlest of the three. The eval passes every candidate, so the on-call sees no fire, but the *signal value* of the pass has collapsed to zero. The gate still issues a verdict, but the verdict carries no information. §5 writes the detector.

## 2. The eval-set-as-dependency framing

The function treats every eval set as a versioned dependency with a reliability contract. The contract shape:

- **Content-addressed.** The eval set is identified by a content hash. The release-gate verdict references the hash.
- **Immutable.** An eval set at hash `H` is never mutated. A revision produces a new hash `H'` and goes through a review flow.
- **Reviewed.** Changes to an eval set land through a PR with sign-off from the eval peer who owns it. The function reviews for scope and reliability-signal implications, not methodology.
- **Audit-logged.** Every eval-set change is recorded against the audit log at `mod-110` chapter 05.
- **SLI-instrumented.** The eval set emits signals the function reads: pass rate across candidates, discriminator score (§5), representativeness score (§3), contamination-check score (§4).
- **Rot-reviewed.** On a fixed cadence (quarterly by default), the function and the eval peer jointly review whether the set still discriminates and still represents production.

An eval set that does not satisfy the contract is not a release predicate. The function refuses to accept it as a gate input.

## 3. Eval-set drift detection

Drift is the easiest of the three to detect and the one where the function's signal is most directly usable.

### 3.1 The representativeness SLI

Define the production query embedding distribution as a baseline and measure how far the eval-set embedding distribution has drifted from it.

```
# compute on a schedule (daily)
eval_set_hash H
prod_sample  := sample N recent production queries (cohort-stratified)
eval_sample  := the eval set at hash H

for each sample, embed with the production embedding model
compute:
  mean_distance_eval_to_prod     := mean( min( dist(e, p) for p in prod_sample ) for e in eval_sample )
  prod_distribution_coverage     := frac( p in prod_sample where exists e in eval_sample with dist(p, e) < ε )
  cluster_coverage               := frac( k-means clusters of prod_sample containing >= 1 eval sample )

emit:
  eval_set_representativeness_score{eval_set, eval_hash}
  eval_set_prod_coverage{eval_set, eval_hash}
  eval_set_cluster_coverage{eval_set, eval_hash}
```

The representativeness score is a composite — any of PSI, Jensen-Shannon, or the cluster-coverage metric works; the eval peer picks the one that most closely tracks eval validity in their experience, and the function wires the emission.

### 3.2 The SLO

> Eval-set representativeness score stays above the per-eval-set floor over a rolling 30-day window.

When the score falls below the floor, the function fires an alarm — not a page, usually, because drift is slow — and opens a **representativeness-refresh** workitem routed to the eval peer. The eval set is still usable while it is being refreshed; the function tracks the lag with an SLO.

### 3.3 Cohort coverage

An eval set whose aggregate representativeness is adequate can still be unrepresentative of specific cohorts — a non-English language tier, a mobile-client cohort, a specific product surface. The function emits `eval_set_cohort_coverage{eval_set, cohort}` per cohort, and alerts per cohort rather than only on the aggregate.

The reason: a release-gate verdict that signs off the candidate on an eval set with 70% English and 30% Spanish is not evidence that the candidate works for the Japanese cohort. Cohort coverage makes the implicit assumption explicit.

### 3.4 Why the function and not the peer owns the emission

The eval peer owns the *content* of the eval set and the representativeness threshold. The function owns the *emission of the signal* for two reasons:

- The signal reads from production traffic, which lives in the function's observability stack.
- The signal routes to the function's alerting, which is the on-call path.

The function does not rewrite the representativeness metric; the function wires the metric the peer chose into the observability pane.

## 4. Eval-set poisoning and contamination detection

### 4.1 The two sub-classes

- **Eval-train contamination** — eval samples leaked into a model's training corpus (the model under test memorises the answer; the eval score is inflated).
- **Adversarial eval poisoning** — samples were added to the eval set with labels that favour a specific candidate. Rare in practice but structurally possible if the eval set has multiple author paths.

### 4.2 Contamination detection

The function wires two detectors:

- **Membership-inference probes.** For a held-out subset of the eval, check whether candidate models exhibit characteristic membership-inference signatures — the model's log-probability on the exact eval string is anomalously high, or the model's response is byte-identical to the eval's reference answer at a rate inconsistent with capability. Membership inference is an active research area; the function uses what the eval peer has calibrated, not a home-grown approach. <!-- needs-research: cite a specific membership-inference-for-contamination reference the field has adopted; recent surveys include works by Carlini et al., but the function should cite the specific technique in use -->
- **Canary strings.** The eval peer inserts a small number of synthetic canary strings into the eval set and checks whether candidate models have memorised them. A canary-string hit on a model is prima facie contamination evidence.

The signal emission:

```promql
eval_contamination_score{eval_set, eval_hash, candidate}
```

Scores above the pre-committed threshold block the release-gate verdict for that candidate. The gate does not downgrade silently.

### 4.3 Adversarial poisoning detection

The function's defence against adversarial poisoning is primarily procedural:

- **Codeowners on eval-set directories.** Every change goes through a PR with the eval peer as a reviewer.
- **Dual sign-off for samples that shift aggregate pass rate meaningfully.** A PR that moves the aggregate pass rate by more than a per-eval-set threshold requires a second reviewer.
- **Audit trail.** The audit log for eval-set changes is retained per `mod-110` chapter 05.

Technical detection of adversarial poisoning is harder and lives inside the eval peer's methodology; the function does not re-implement it. The function enforces the surface (codeowners, dual sign-off, audit) and reads the signal when the peer exposes one.

### 4.4 The quarantine action

When contamination is detected on an eval set, the function:

1. Marks the eval set as `quarantined` in the release-gate contract. Every verdict against it is suspect.
2. Fails the release-gate verdict for any candidate the contamination affects.
3. Routes to the eval peer with the specific contamination evidence.
4. Tracks remediation: the contaminated samples are identified and removed, a new eval-set hash is produced, the gate switches to the new hash.
5. Publishes a post-incident report with the contamination source and the remediation.

Quarantine is cheap for the function: the candidate holds. Quarantine is expensive for the organisation: no releases on that route. The function errs on the side of quarantine when evidence is ambiguous; the cost of a false positive is a delayed release, the cost of a false negative is a poisoned gate.

## 5. Eval-set rot detection

### 5.1 The discriminator SLI

Rot is detected by measuring whether the eval set *distinguishes candidates*. Compute:

```
for each eval set at hash H:
  for each candidate model in the active roster:
    compute pass_rate(candidate, H)
  
  discriminator := variance across candidates of pass_rate
  ceiling_fraction := fraction of candidates with pass_rate >= 0.98

emit:
  eval_set_discriminator_score{eval_set, eval_hash}
  eval_set_ceiling_fraction{eval_set, eval_hash}
```

A high ceiling fraction (most candidates pass trivially) and a low discriminator (candidates are not differentiated) are the rot pattern.

### 5.2 The rot SLO

> Eval-set discriminator score stays above the per-eval-set floor and ceiling fraction stays below the per-eval-set cap over a rolling 90-day window.

The 90-day window is longer than the drift SLO's 30-day window because rot is a slower phenomenon. When the SLO burns, the function opens a joint review with the eval peer to either:

- **Harden the eval** — add harder samples that the current model population finds difficult.
- **Retire the eval** — remove it from the release-gate contract; the signal it provides no longer has value.
- **Change the aggregation** — if the ceiling is only on the aggregate but harder subsets still discriminate, surface the subset signal.

### 5.3 The rot discussion is a joint discussion

Rot detection is where the function and the peer collaborate most actively. The function provides the signal (discriminator, ceiling); the peer decides the remediation (harden, retire, re-aggregate). The function does not retire an eval unilaterally; the function can, however, **demote** a rotten eval from `blocking` to `warning` in the release-gate contract — the gate no longer blocks on it, but the audit log still records it. Demotion is the operational action the function controls.

## 6. The eval-set release-gate contract

The release-gate contract (`mod-111` chapter 05) has specific eval-set clauses the function writes and the eval peer signs:

### 6.1 Named eval sets

The contract lists the specific eval sets that gate each route:

```yaml
route: chat
eval_sets:
  - name: chat-core-quality
    hash: sha256:3f91...
    role: blocking
    threshold: pass_rate >= 0.85
    representativeness_floor: 0.75
    ceiling_cap: 0.95
  - name: chat-safety-regression
    hash: sha256:6ac2...
    role: blocking
    threshold: pass_rate >= 0.99
  - name: chat-nonenglish-coverage
    hash: sha256:2b48...
    role: warning
    threshold: pass_rate >= 0.70
```

Each entry is auditable, pinned to a hash, and has an explicit role.

### 6.2 Verdict shape

The release-gate verdict references the eval-set hashes it signed off against:

```yaml
candidate: (model-v2, prompt-h1, config-h9)
verdict: pass
signed_against:
  - eval_set: chat-core-quality      @ hash sha256:3f91...
  - eval_set: chat-safety-regression @ hash sha256:6ac2...
  - eval_set: chat-nonenglish-coverage @ hash sha256:2b48...  (warning)
metadata:
  representativeness_at_verdict:
    chat-core-quality: 0.82
    chat-safety-regression: 0.91
  ceiling_fraction_at_verdict:
    chat-core-quality: 0.88
```

The verdict records *what the eval was like at verdict time*. A later investigation that pulls the verdict can see whether the eval was healthy when it signed off.

### 6.3 Contract revision cadence

- **Quarterly** — the function and the eval peer review the representativeness, discriminator, and ceiling scores. Roles may be promoted (warning → blocking) or demoted; sets may be added or retired.
- **On drift / rot alarm** — a focused review is triggered; the quarterly review is not where urgent changes happen.
- **On contamination** — an immediate quarantine; the revision is driven by the incident.

The cadence mirrors the chapter 01 prompt / config review and the `mod-111` chapter 05 release-gate review.

## 7. The eval-pipeline operational surface

The eval pipeline is itself a dependency with its own SLOs. The function treats it the same way it treats the retrieval ingest pipeline (chapter 02) and the feature pipeline (chapter 03).

### 7.1 Pipeline signals

- `eval_run_duration_seconds{eval_set, candidate}` — histogram.
- `eval_run_failure_total{eval_set, candidate, reason}` — counter.
- `eval_run_last_success_timestamp{eval_set, candidate}` — gauge.
- `judge_latency_seconds{eval_set, judge}` — the judge's call latency (the judge is itself a dependency per `mod-111` chapter 02 §7).
- `judge_availability_total{eval_set, judge}` — judge availability counter.

### 7.2 Pipeline SLOs

> 99% of eval runs against the release-gate's active eval sets complete within the committed duration budget over a rolling 30-day window.

> Judge availability > 99.9% over a rolling 30-day window.

A pipeline that can produce verdicts fast enough for the release cadence is the function's commitment; the chapter 06 peer worksheet writes the handoff if the eval pipeline cannot meet the cadence.

### 7.3 The pipeline runbook

`rb-eval-pipeline-degraded` is the function's runbook:

1. Identify the degraded component (eval runner, judge, data source).
2. If the judge is degraded, fall back to the pinned secondary judge per the peer's methodology, *if* the methodology supports it. The function does not pick a secondary judge unilaterally.
3. If the runner is degraded, scale horizontally (eval runs are typically embarrassingly parallel).
4. If the data source is degraded, hold the verdict and route to the owning peer.
5. Track the SLO burn through the degradation.
6. Post-incident review against the chapter 05 pane.

## 8. The eval-set reliability checklist

Before accepting an eval set into the function's release-gate contract:

1. **Content-addressed** — the eval set has a reproducible hash and is immutable at that hash.
2. **Reviewed** — codeowners on the directory; eval peer sign-off on every change; dual sign-off threshold for pass-rate-shifting changes.
3. **Representativeness SLI** emitted and the score is above the per-eval-set floor.
4. **Discriminator SLI** emitted and the score is above the per-eval-set floor.
5. **Contamination-check SLI** emitted (membership-inference or canary-string, per the peer's methodology).
6. **Cohort coverage SLIs** emitted for each cohort the eval is expected to represent.
7. **Pipeline SLOs** declared — eval-run duration, judge availability — and burn-rate alarms wired.
8. **Three runbooks** exist — `rb-eval-pipeline-degraded`, `rb-eval-contamination`, `rb-eval-rot-review` — and the pipeline-degraded runbook has been drilled.
9. **Release-gate contract entry** signed — role (blocking / warning), threshold, hash — and the quarterly review is on the function's calendar.

An eval set used as a release predicate without these items is a gate whose verdict carries no accountable signal.

## Summary

- An eval set is a dependency with a reliability contract, not just a dataset. **If the eval set is compromised, every gate verdict against it is suspect.**
- Three failure modes: **drift** (the set no longer represents production), **poisoning / contamination** (samples leaked into a model's training data or were injected adversarially), **rot** (every candidate passes trivially; the set no longer discriminates).
- Each failure mode has a detection signal — representativeness score, contamination score, discriminator score — the function emits into the single-pane observability stack.
- **Contamination triggers quarantine**: the function marks the eval set as suspect, fails the gate for affected candidates, and routes remediation to the eval peer. Quarantine is cheap for the function and expensive for the org, and the function errs toward it under ambiguity.
- **Rot triggers a joint review**: the function demotes the eval from `blocking` to `warning` in the release-gate contract, but the eval peer decides whether to harden, retire, or re-aggregate. Demotion is the function's unilateral action; retirement is not.
- The release-gate verdict records the eval-set hash and the representativeness / ceiling scores at verdict time, so a later incident investigation can see whether the gate was healthy when it signed off.
- The eval pipeline is itself a monitored dependency with duration, judge-availability, and failure-rate SLOs and a `rb-eval-pipeline-degraded` runbook.
- The function does **not** author eval sets, write judge prompts, or set methodology. The `ai-eval-engineer` and `model-evaluation-engineer` peers do. The function owns the reliability contract on the eval surface and the operational response when any of the three failure modes fires.
- The nine-point checklist governs an eval set's acceptance into the release-gate contract.

Chapter 05 pulls the surface together — one observability pane that reads prompts, configs, retrieval, features, and evals in the same stack as the serving tier does.
