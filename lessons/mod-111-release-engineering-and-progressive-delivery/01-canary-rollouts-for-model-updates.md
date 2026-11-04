# Canary Rollouts for Model Updates

## Why this chapter exists

Canary deployment is a solved problem for stateless web services. There is a decade of practice, a shelf of CNCF tools (Flagger, Argo Rollouts, Istio's own traffic-splitting primitives), and a settled vocabulary around weight ramps, SLO-based promotion, and automatic rollback. An AI SRE arriving at this topic does not get to re-derive the field. The job is to **take the canonical progressive-delivery pattern as it is**, understand the one place where its assumptions break when the artifact being rolled out is a model, and extend the pattern with the minimum set of AI-specific controls that preserve the promotion-or-rollback decision under non-deterministic outputs.

The break is sharp and specific: **two model versions never produce byte-identical responses**, and the baseline canary-analysis mechanisms (success-rate / latency / HTTP error-rate deltas; response-equality probes) either keep working as-is, or they need a quality signal grafted in because equality has stopped being a well-defined predicate. Chapter 02 goes deep on the semantic-equivalence problem that follows. This chapter writes the surrounding progressive-delivery shape: what tool draws the traffic, how the weight ramps, which SLOs gate the step-up, what triggers the rollback, and which signals belong on the canary dashboard that the one-click promotion reads from.

The chapter's boundary is tight. It does **not** teach Istio request routing in general, does **not** rewrite Argo Rollouts' documentation, does **not** take sides on Flagger-vs-Argo. It assumes the function has already picked a tool from the canonical shelf and writes the AI SRE's layer on top. CI/CD plumbing — the Git-to-manifest promotion pipeline, the image-build-and-tag workflow — is at the `ai-infra-mlops` peer (chapter 08 of this module draws that line). The in-flight-swap and drain primitives at the inference engine are at the `ai-infra-inference-engineer` peer; blue-green for large models (chapter 04) touches that boundary more directly than canary does.

## 1. The canonical canary, in one page

A canary rollout, in its classical form, has five mechanical pieces:

1. **Two serving populations.** A stable deployment serving ~all production traffic, and a canary deployment serving a small fraction.
2. **Traffic splitter.** A mechanism — service mesh (Istio, Linkerd), ingress (NGINX, Contour), or a dedicated rollout controller — that routes a weighted percentage of requests to the canary.
3. **Canary analysis.** A set of metrics compared between stable and canary over a rolling window: request success-rate, latency percentiles, error-rate, (classical) output equivalence. A regression on any metric fails the canary.
4. **Promotion policy.** A deterministic ramp — e.g., 1% → 5% → 25% → 50% → 100% — with a dwell time at each step, promotion only after the dwell window passes clean.
5. **Rollback policy.** If any step fails the analysis, the controller shifts traffic back to stable and declares the canary failed. Human review happens on the failure, not on each promotion.

Flagger and Argo Rollouts are the two CNCF implementations of this loop. Flagger integrates with service meshes (Istio, Linkerd, App Mesh, Kuma, NGINX, Contour, Gloo, Traefik — the exact mesh matrix is in the Flagger docs; the point is that Flagger does not ship its own data plane). Argo Rollouts ships its own `Rollout` CRD, drives traffic through the mesh *or* via replica-count manipulation, and integrates with analysis templates in both Prometheus and third-party metric providers. Istio can be driven directly with `VirtualService` weight edits, but the canary-analysis loop then has to be written by hand; most mature teams prefer Flagger or Argo on top.

For this chapter, assume the function has picked one of Flagger or Argo Rollouts; the two produce equivalent AI-adapted shapes, and chapter 08 writes the mapping to the mlops peer that chose the tool.

## 2. Where "canary analysis" breaks for model updates

Classical canary analysis compares the stable and canary populations on three families of signal:

- **Serving health** — HTTP 5xx rate, request duration p95 / p99, success-rate.
- **Business metrics** — revenue per request, conversion, add-to-cart rate; the signal the business cares about.
- **Equivalence** — for a non-trivial fraction of PRs (configuration changes, refactors, minor logic changes), the response bodies should match byte-for-byte. Classical response-diffing tools (Diffy at Twitter, Scientist-style parallel-run harnesses) existed exactly to validate this.

On a model-backed route, the **serving-health** family keeps working. A canary that doubles 5xx rate, triples TTFT p95, or halves success-rate is unambiguously broken; the SLO burn-rate alarms from `mod-103` fire; the rollout controller aborts. Nothing AI-specific is needed to catch this shape.

The **business-metrics** family keeps working in principle but with a latency-of-signal problem: conversion-rate deltas on a 1% canary take hours to be statistically meaningful, and the canary window is typically shorter than that. Teams typically rely on business metrics as a longer-horizon post-promotion check rather than a per-step gate.

The **equivalence** family *does not work*. Two model versions, even under identical decoding settings, produce different outputs because:

- Decoding is sampled — temperature, top-p, top-k inject randomness. Setting `temperature=0` reduces but does not eliminate variance; batch-size effects, FP8 / BF16 reduction orders, non-deterministic kernels, and provider-side changes keep outputs non-identical even at zero temperature on some stacks (`mod-105` chapter 02 covers the full taxonomy of non-determinism sources).
- Even without sampling, model-weight changes alter the output distribution. A new model version by construction produces different token sequences for the same input; that is the entire point of shipping it.

Classical response-diffing therefore produces a wall of false positives on an AI canary. A naive "did the canary respond with the same bytes as stable?" check fails on nearly every request. The rollout controller sees the equality fail, calls the canary broken, and rolls back. Shipping is blocked by a gate that is not actually measuring regression.

The AI-adapted analysis **retains serving-health and business-metrics checks unchanged** and **replaces equivalence with a quality predicate** that is tolerant of non-determinism. Chapter 02 writes the quality predicate; this chapter describes how the canary analysis loop consumes it.

## 3. The AI-adapted canary analysis signal set

For a model-backed route, the canary-analysis dashboard reads from four signal families, each of which the rollout controller treats as an independent fail-the-canary predicate.

### 3.1 Serving-health signals (unchanged from classical)

The SLI set from `mod-103` and `mod-104` carries over directly:

- `request_success_rate{route, version}` — not a model-specific signal; works for any HTTP route.
- `ttft_seconds_bucket{route, version}` — time to first token. The dominant latency SLI on a streaming route.
- `tpot_seconds_bucket{route, version}` — time per output token.
- `streaming_stall_rate{route, version}` — fraction of streams that paused longer than the per-route stall budget.
- `structured_output_conformance_rate{route, version}` — for routes that must return JSON / function calls, the parse-rate.

Each is emitted with a `version` label that distinguishes stable from canary. The canary-analysis query is the ratio (or absolute delta) between the two populations over the analysis window.

### 3.2 Quality signals (new for AI canaries)

Non-deterministic outputs cannot be compared for equality, but their *distributions* can be compared. The quality signal family lives in this layer:

- `canary_judge_score{route, version, judge_id, dimension}` — online judge pipeline scores (groundedness, faithfulness, helpfulness, safety-pass-rate) emitted as a histogram per canary request. The `ai-eval-engineer` peer owns the judges; the AI SRE wires the emission.
- `refusal_rate{route, version}` — fraction of requests refused by the model. A canary that spikes refusals to 2× baseline has regressed in a product-relevant way even if latency is identical.
- `safety_classifier_trigger_rate{route, version}` — the rate at which the output guardrail blocks or flags; same shape.
- `response_length_distribution{route, version}` — a canary that silently truncates or inflates output length changes cost, latency, and product experience simultaneously.

The canary-analysis query reads the KL divergence, the distribution shift, or a pre-committed bounded delta on each of these, not the equality of individual samples. Chapter 02 is the place where the exact predicate shape is written.

### 3.3 Cost signals (new for AI canaries)

Cost per request is a reliability signal on AI services in a way that it is not on classical services. A canary that answers correctly but costs 3× more per request is a regression — against the economic SLO (`mod-106` chapter 03's cost-per-request guardrail) and against the cost-per-outcome contract in `mod-107` chapter 05.

- `cost_per_request_usd{route, version, cost_source}` — the sum of model-provider cost, retrieval cost, and (for self-hosted) GPU-second cost attributed per request.
- `tokens_per_request{route, version, kind="prompt|completion"}` — the raw input, which drives most of the cost delta on provider-backed routes.

The cost delta is often the *first* signal to move on a model upgrade: the new model has different tokenisation, a different verbose-vs-terse profile, or a different routing to a more expensive variant.

### 3.4 Business signals (classical, long-horizon)

These keep working but on a longer window:

- `conversion_rate{route, version}` — if the fleet has one.
- `user_feedback_rate{route, version, polarity="positive|negative"}` — thumbs-up / thumbs-down on the response.
- `session_continuation_rate{route, version}` — did the user ask a follow-up?

On a 1% canary, these take 24–72 hours to become statistically meaningful. They belong on a post-promotion dashboard with the promotion held open under watch, not on the per-step gate.

## 4. The weight-ramp policy, adapted

The classical ramp is `1% → 5% → 25% → 50% → 100%` with a dwell of (say) 10 minutes per step and a promotion-pass criterion of "no fail in dwell window." The AI-adapted ramp changes two things.

### 4.1 Longer dwell at low weights

Quality signals have higher variance than serving-health signals. A 1% canary that sees 1000 requests in 10 minutes produces an estimate of judge-score mean whose confidence interval is wide. The dwell at 1% should be long enough that the quality signal's confidence interval is tighter than the pre-committed regression threshold. For a judge whose per-request score has standard deviation ≈ 0.15 (a typical value for a scalar 0–1 judge on a mature evaluation set — <!-- needs-research: cite representative judge-score standard-deviation data from a public eval-harness paper or report -->), detecting a shift of 0.05 at 95% confidence on 1000 requests is feasible; detecting a shift of 0.01 is not. The ramp must either dwell longer, dwell wider (start at 5% instead of 1%), or use a less-sensitive threshold.

A representative AI-adapted ramp:

| Step | Weight | Dwell | Promotion predicate |
|------|--------|-------|---------------------|
| 1 | 1% | 30 min | Serving-health SLOs green AND judge-score delta within ±0.05 |
| 2 | 5% | 60 min | Same |
| 3 | 25% | 2 h | Same, plus cost delta within +20% of stable |
| 4 | 50% | 4 h | Same, plus refusal-rate delta within ±10% relative |
| 5 | 100% | — | Promotion complete; business-metric watch continues for 72 h |

The exact numbers are per-fleet and are written in the release-gate contract (chapter 05).

### 4.2 Hold at a non-trivial weight for the long-horizon signals

On classical services, 100% means the release is done. On AI services, the signals that matter most (conversion, long-session satisfaction, cost-per-outcome at a weekly granularity) take days to resolve. The AI-adapted policy explicitly adds a **long-horizon watch** after 100%: the previous version's deployment is kept warm (not torn down) for a rollback window of 72 hours, during which the release is monitored on business and cost signals that the per-step gate cannot see. A regression during that window triggers a *post-promotion* rollback, which is a different operational shape — chapter 05 §6 writes it.

## 5. SLO-based auto-rollback — the control that lets a human sleep

Auto-rollback is the control that makes continuous canary promotion viable. It is distinct from auto-halt (which pauses a chaos experiment, `mod-109` chapter 05) and from circuit-breaking (which rejects traffic without unwinding a deploy). An auto-rollback means the rollout controller shifts traffic back to stable without human action and declares the canary failed.

### 5.1 The three rollback triggers

1. **SLO-burn fast-rate alarm.** A canary that burns the error budget at 14.4× the steady-state rate over 5 minutes is already unacceptable (per `mod-103`'s multiwindow multi-burn-rate practice). Fire → rollback.
2. **Quality-regression alarm.** A canary whose judge-score distribution shifts by more than the per-fleet bounded delta over the dwell window. Fire → rollback.
3. **Cost-regression alarm.** A canary whose cost-per-request exceeds the pre-committed delta over the dwell window. Fire → rollback.

Each has to be wired as an alarm the rollout controller can read. Flagger's analysis templates and Argo Rollouts' analysis template both support Prometheus queries; both will abort the rollout on a failing analysis. The query is the same shape as the per-step gate; the difference is the alarm's severity and the controller's reaction to it.

### 5.2 The "rollback is cheap, don't be clever" rule

A tempting failure mode is to add logic to the rollback trigger: "if the regression is small, hold instead of rolling back; if the judge is a new model that we don't trust yet, wait." The rule is the opposite. Rollback is cheap — the stable deployment is still running at the serving-weight the canary hasn't claimed yet; routing back is a mesh config change. The cost of a false-positive rollback is a delayed release; the cost of a false-negative promotion is user-visible regression. The rollout controller rolls back on any pre-committed alarm and the function investigates on the rollback, not on the request path.

An exception the policy allows: a *flaky* alarm (fires three times in a week without a real regression) is tuned, not ignored. The chaos program's "learnings back to the runbook" principle (`mod-109` chapter 01 §7) applies to auto-rollback triggers too.

### 5.3 The rollback's data trail

Every auto-rollback produces a record that includes:

- The canary version, the stable version, and the manifest hash of each.
- The trigger (which alarm fired) and the metric trace at the moment of firing.
- The canary weight at the moment of rollback.
- The time-to-rollback (from trigger to traffic shift).

The rollout controller writes this; the fixture back-feed from `mod-108` chapter 05 reads it; chapter 06 of this module (DORA four-keys) uses it to compute change-failure-rate and MTTR.

## 6. Progressive exposure on top of the ramp

Weight is one axis of exposure. On AI services, two other axes are often more productive because they localise the blast radius to a cohort that is explicitly opt-in to the risk.

### 6.1 Cohort-first canaries

Instead of a random 1% of all requests, route 100% of a specific cohort — internal users, dog-food tenants, a specific free-tier region, a specific product surface — to the canary. The cohort is:

- **Opt-in.** Known-tolerant to variation, consented-to-telemetry, reachable for qualitative feedback.
- **Large enough to be statistically meaningful.** A 50-person dog-food cohort produces 50 × N requests; if N is small, extend the cohort or extend the dwell.
- **Representative of the production distribution** on the axes that matter — prompt-length distribution, language mix, retrieval-topic coverage. A cohort of English-only internal engineers is not a good canary for a Japanese consumer surface.

Istio supports cohort-first routing via `VirtualService` `match` rules on headers (`x-user-class: dogfood`), source IP ranges, or JWT claims; Flagger and Argo Rollouts both compose on top of this (Flagger's `canaryAnalysis.match` field; Argo Rollouts' `setHeaderRoute` and `setMirrorRoute` steps).

### 6.2 Route-by-route canaries

A model update rarely regresses uniformly. A new retrieval-tuned model may improve groundedness on the RAG route and silently regress on the free-text chat route. Promoting the model on a per-route basis (RAG first, chat second, agent third) lets each route's signal set validate the model independently. The `version` label on each SLI then carries the route dimension; the canary-analysis query is per-route.

### 6.3 Progressive exposure as the default

Weight ramp alone is easy to reason about but produces a lot of false-negative rollbacks on routes where the aggregate signal hides a localised regression. Cohort-first and route-by-route layering localises the signal; the default AI SRE canary policy combines all three (narrow cohort first, one route at a time, weight ramp on top).

## 7. A worked example — canary config for a chat route

A representative Flagger `Canary` resource for a chat route (abbreviated to the AI-adapted parts; the full resource includes standard health checks and the mesh-specific gateway bindings):

```yaml
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: chat-serving
  namespace: inference
spec:
  provider: istio
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: chat-serving
  service:
    port: 8080
  analysis:
    interval: 2m           # analysis window
    threshold: 5           # consecutive failures before rollback
    stepWeight: 5          # weight increase per analysis pass
    maxWeight: 50          # stop auto-promotion at 50%; manual promotion to 100%
    match:
      - headers:
          x-user-class:
            regex: "^(dogfood|internal)$"   # cohort-first, see §6.1
    metrics:
      - name: ttft-p95
        templateRef:
          name: ttft-p95-vs-baseline
        thresholdRange: { max: 1.15 }       # p95 must stay within 15% of stable
        interval: 2m
      - name: success-rate
        templateRef:
          name: success-rate
        thresholdRange: { min: 99.0 }
        interval: 2m
      - name: judge-score-delta
        templateRef:
          name: judge-score-delta
        thresholdRange: { min: -0.05, max: 0.1 }   # bounded delta; §3.2
        interval: 5m
      - name: cost-per-request-ratio
        templateRef:
          name: cost-per-request-ratio
        thresholdRange: { max: 1.20 }        # cost must not exceed 1.2× stable
        interval: 5m
      - name: refusal-rate-delta
        templateRef:
          name: refusal-rate-delta
        thresholdRange: { min: -0.1, max: 0.1 }  # absolute delta on refusal rate
        interval: 5m
    webhooks:
      - name: eval-gate
        type: pre-rollout
        url: http://eval-gate.release/eval
      - name: load-test-canary-scale
        type: rollout
        url: http://load-gen.release/run
        timeout: 15m
```

Each named `templateRef` resolves to a Prometheus query. The two webhooks compose with chapters 05 (eval gate as pre-rollout) and 07 (load-test at canary scale) respectively.

## 8. The AI SRE's canary-rollout checklist

Before enabling a canary on an AI route for the first time, the function confirms:

1. **The serving-health SLIs are in place and emit a `version` label.** The `mod-104` observability stack covers this; the canary just requires the label.
2. **The quality signal is wired.** Online judge pipeline per route, score emission per request, histogram available in Prometheus. The `ai-eval-engineer` peer owns the judge; the SRE wires the emission path.
3. **The cost signal is wired.** Per-request cost attribution from `mod-106` and `mod-107`.
4. **The rollout controller is installed and tested.** Flagger or Argo Rollouts; one completed non-AI canary run against a sample workload to confirm the mesh plumbing.
5. **The analysis templates are authored and in Git.** Not in a dashboard; in the same Git repo as the service manifests.
6. **The rollback is tested.** A dry-run rollout with a synthetic regression (a canary image that emits deliberately-high latency) confirms the controller rolls back within the time budget.
7. **The release-gate contract names this canary as its promotion mechanism.** Chapter 05 writes the contract.

A canary that runs without this checklist is a rollout waiting to produce an incident.

## Summary

- Canary rollout is a solved problem; the AI-specific break is that response-equality is no longer a valid gate between stable and canary.
- Replace equality with a **distribution-level quality predicate** (chapter 02 writes the predicate itself); keep serving-health and business-metric checks unchanged; add a cost-delta check that AI canaries uniquely need.
- The weight ramp is longer at low weights than the classical default, because judge-signal variance demands wider dwell for a tight confidence interval.
- Auto-rollback is wired to three families — SLO burn, quality regression, cost regression — and rolls back cheaply on any pre-committed alarm; do not add rollback-side logic.
- Progressive exposure uses cohort-first and route-by-route layering on top of weight; this localises the blast radius and the signal.
- The Flagger / Argo Rollouts resource composes with the webhook pattern that chapters 05 and 07 use to bind the eval gate and the pre-promotion load test.
- Before enabling an AI canary for the first time, the function runs the eight-point checklist — SLIs with version label, quality signal wired, cost signal wired, controller installed and dry-run-tested, templates in Git, rollback tested, contract in place.

Chapter 02 writes the response-diffing layer the canary-analysis signal set depends on.
