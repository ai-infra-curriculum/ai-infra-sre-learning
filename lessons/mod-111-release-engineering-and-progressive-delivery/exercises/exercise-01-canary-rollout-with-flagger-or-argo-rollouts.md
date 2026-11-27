# Exercise 01 — Canary Rollout With Flagger or Argo Rollouts

**Estimated effort:** 3 hours · **Output:** a working canary rollout on a Kubernetes cluster, one green canary run for a model-backed route, the canary analysis templates in Git, and a short rollout note (≤ 600 words)

## Objective

Install one of the two canonical progressive-delivery controllers (Flagger or Argo Rollouts) on a Kubernetes cluster you control, author the canary configuration for a model-backed route with the AI-adapted analysis signal set from chapter 01 §3, run one green canary on a non-regressing candidate, and dry-run an auto-rollback on a synthetic regression. The deliverable is not production deployment — it is the **mechanics competence** that lets any later chapter build on a working canary substrate.

The point is that at the end of the exercise the function can take any release-gate verdict (chapter 05) and drive it through the controller with signal-grounded analysis templates. The route itself is intentionally small; the surrounding analysis templates and the dry-run discipline are where the work is.

## Prerequisites

- Chapter 01 of this module and the skim of chapter 02's semantic-equivalence section (the exercise uses a placeholder judge rather than a real one; if your environment has a real judge available, prefer it).
- A Kubernetes cluster you can install into — a local kind / minikube / k3d cluster is acceptable; a non-production cluster with real serving-like replicas is better.
- Prometheus + Grafana installed (`kube-prometheus-stack` chart is the fastest path if not already present).
- A sample model-backed workload. If you have no real inference pod, the illustrative chat-via-provider workload from `mod-109` exercise 02 is sufficient; a minimal Python service that wraps an Anthropic or OpenAI endpoint behind an HTTP handler works. The sample must emit TTFT / success-rate / cost-per-request Prometheus metrics (proposed shapes are acceptable — `ttft_seconds_bucket{route, version}`, `request_success_total{route, version}`, `cost_per_request_usd{route, version}`).
- Istio or Linkerd (or NGINX ingress with Flagger's NGINX provider) installed — any of the meshes listed in the controller's provider matrix. For Argo Rollouts, the mesh is still required for traffic splitting.
- Admin kubectl on the cluster.

## Problem statement

Work in six parts.

### Part 1 — Install the controller

Pick Flagger *or* Argo Rollouts and install it following the official docs. Record:

- The exact version installed.
- The Helm chart or manifest URLs used.
- The namespace the controller landed in.
- The provider / integration enabled (Istio, Linkerd, NGINX, service-mesh of choice).

If the controller's default install is cluster-wide-RBAC, scope it to the release namespace + the target serving namespaces before any production use.

### Part 2 — Instrument the sample workload

Confirm the sample workload emits these Prometheus metrics with a `version` label distinguishing stable and canary:

- `ttft_seconds_bucket{route, version}` (histogram)
- `request_success_total{route, version}` and `request_total{route, version}` (counters)
- `cost_per_request_usd{route, version}` (gauge or summary)
- A placeholder `canary_judge_score{route, version}` emitted as a gauge with synthetic values in `[0, 1]`. A real judge is a stretch goal; the exercise focuses on mechanics.

If the workload does not emit any of these, add the instrumentation before continuing. Commit the instrumentation to Git.

### Part 3 — Author the canary resource

Author the canary configuration under `release/canary-<route>.yaml`:

- **Flagger route.** A `Canary` resource targeting the sample workload's `Deployment`, with `analysis.interval=2m`, `analysis.stepWeight=5`, `analysis.maxWeight=50`, and `analysis.match` scoping to a dogfood cohort via a header (`x-user-class: dogfood`). The abbreviated example from chapter 01 §7 is the starting point.
- **Argo Rollouts route.** A `Rollout` resource with an analysis template-referenced `canaryStep` sequence, equivalent cohort matching via `setHeaderRoute`, and the same analysis metric queries.

Author at least four `AnalysisTemplate`s (Argo Rollouts) or `MetricTemplate`s (Flagger). Each template is a named Prometheus query with a pre-committed threshold. The query shapes below are the parse contract — the exact label matcher set, aggregation, and over-window are what the exercise grades against:

```promql
# ttft-p95-vs-baseline
# p95 TTFT of canary divided by p95 TTFT of stable, over the analysis window.
histogram_quantile(
  0.95,
  sum by (le) (rate(ttft_seconds_bucket{route="$route", version="canary"}[$interval]))
)
/
histogram_quantile(
  0.95,
  sum by (le) (rate(ttft_seconds_bucket{route="$route", version="stable"}[$interval]))
)

# success-rate
# Canary 2xx rate as a percentage, over the analysis window.
100 *
sum(rate(request_success_total{route="$route", version="canary"}[$interval]))
/
sum(rate(request_total{route="$route", version="canary"}[$interval]))

# judge-score-delta
# Mean judge-score of canary minus mean judge-score of stable, over the analysis window.
avg_over_time(canary_judge_score{route="$route", version="canary"}[$interval])
-
avg_over_time(canary_judge_score{route="$route", version="stable"}[$interval])

# cost-per-request-ratio
# Mean cost-per-request of canary divided by mean cost-per-request of stable.
avg_over_time(cost_per_request_usd{route="$route", version="canary"}[$interval])
/
avg_over_time(cost_per_request_usd{route="$route", version="stable"}[$interval])
```

The pre-committed thresholds the canary resource references these templates with (from chapter 01 §7, reproduced as the contract):

| Template | `thresholdRange` | Analysis interval | Rationale |
|----------|------------------|-------------------|-----------|
| `ttft-p95-vs-baseline` | `{ max: 1.15 }` | 2m | p95 TTFT must stay within 15% of stable. |
| `success-rate` | `{ min: 99.0 }` | 2m | Canary 2xx rate must hold above 99%. |
| `judge-score-delta` | `{ min: -0.05, max: 0.1 }` | 5m | Bounded quality drift; §3.2 of chapter 01 writes the bound. |
| `cost-per-request-ratio` | `{ max: 1.20 }` | 5m | Cost must not exceed 1.2× stable. |

A fleet whose observed variance differs from these defaults is free to widen or tighten — but the delta from these defaults is documented in the rollout note, not silently chosen.

Commit all templates to `release/analysis/` in Git. Dashboards are for inspection; the templates are the source of truth.

### Part 4 — Run a clean canary

Deploy the sample workload as stable at version `v1`. Deploy a canary at version `v2` with no regression (identical binary, different version label). Trigger the canary by applying the manifest.

Observe:

- The controller progresses the canary through the weight steps.
- Each step's analysis passes (metrics within thresholds).
- The canary promotes to 50% (the `maxWeight` ceiling) and holds; the final promotion to 100% is manual (a human-reviewable step).

Record the timings per step and the per-step verdict trace to `release/runs/<canary>-clean.md`. The row shape is the parse contract — a downstream tracker (and the chapter 06 DORA four-keys computation) reads these rows:

| Column | Content |
|---|---|
| `step` | 1-indexed integer matching the controller's step counter. |
| `weight_percent` | The canary weight the controller applied at this step (matches `stepWeight` × step, capped at `maxWeight`). |
| `dwell_start` / `dwell_end` | ISO-8601 UTC timestamps bracketing the analysis window. `dwell_end - dwell_start` must be ≥ `analysis.interval`. |
| `ttft_p95_ratio` | The value the `ttft-p95-vs-baseline` template returned for this step (≤ `1.15` to pass). |
| `success_rate_pct` | The value the `success-rate` template returned (≥ `99.0` to pass). |
| `judge_score_delta` | The value the `judge-score-delta` template returned (in `[-0.05, 0.1]` to pass). |
| `cost_ratio` | The value the `cost-per-request-ratio` template returned (≤ `1.20` to pass). |
| `outcome` | One of {`pass`, `fail`, `hold`}. `hold` is reserved for the `maxWeight` ceiling step pending manual promotion. |

The chapter 01 §4.1 representative ramp — `1% → 5% → 25% → 50% → 100%` with longer dwells at low weights — is the shape the recorded trace is compared against. This exercise's `stepWeight=5`/`maxWeight=50` configuration necessarily produces more steps with shorter dwells; a per-fleet deviation from the §4.1 defaults (wider dwell, narrower step, different starting weight) is documented in the rollout note (Part 6), not silently chosen.

### Part 5 — Dry-run an auto-rollback

Deploy a canary at `v3` that *does* regress — the simplest way is to add artificial latency in the handler (sleep 500ms before responding) so TTFT fires the `ttft-p95-vs-baseline` threshold. Trigger the canary.

Confirm:

- The analysis fires `failed` on the regressed metric within the analysis interval.
- The controller aborts the rollout and shifts traffic back to stable.
- The event stream shows the rollback (including the metric that fired).
- The time-to-rollback is recorded.

Record the rollback as `release/runs/<canary>-rollback.md` with the fields from chapter 01 §5.3. The field slugs are the parse contract:

| Field | Content |
|---|---|
| `canary_version` | The version label applied to the canary pod template (e.g., `v3`). |
| `stable_version` | The version label the controller rolled traffic back to (e.g., `v1`). |
| `canary_manifest_sha` | `sha256` of the canary `Deployment` / `Rollout` manifest as committed to Git. |
| `stable_manifest_sha` | `sha256` of the stable manifest at the time of rollback. |
| `trigger_template` | Which of the four analysis templates fired (one of `ttft-p95-vs-baseline`, `success-rate`, `judge-score-delta`, `cost-per-request-ratio`). |
| `trigger_metric_value` | The value the template returned when the threshold was crossed. |
| `trigger_threshold` | The `thresholdRange` the template was configured with (reproduced from the Part 3 contract table). |
| `canary_weight_at_rollback_percent` | The canary weight the controller had reached when the alarm fired. |
| `time_to_rollback_seconds` | Seconds from trigger fire to traffic shift completion. |
| `controller_event_trace` | Raw controller events for the rollback window (stdout of `kubectl describe canary <name>` for Flagger, `kubectl argo rollouts status <name>` for Argo Rollouts). |

The record is the fixture the chapter 06 DORA four-keys computation (`change-failure-rate`, `MTTR`) consumes; its slugs are what the aggregator expects. Narratives around the rollback belong in the rollout note (Part 6), not in this record.

### Part 6 — Rollout note

A short rollout note (≤ 600 words) at `release/runs/<canary>-rollout-note.md`. The section slugs are the parse contract so a reader (or a future chapter 06 rollup) can locate each fact without reading prose:

| Section | Required content |
|---|---|
| `controller` | Chosen controller and provider with versions (e.g., `flagger 1.37.0, istio provider, chart X.Y.Z`); namespace it landed in; RBAC scope (namespace-scoped or documented deviation from §Part 1). |
| `analysis-templates` | Each of the four templates authored, with the Prometheus query shape and the `thresholdRange` applied. Any per-fleet delta from the Part 3 default-threshold table is called out here with a one-sentence justification. |
| `clean-canary-run` | The step table from Part 4 (or a link to `<canary>-clean.md`), the total wall-clock for the clean run, and the step at which the controller held for manual promotion. |
| `rollback-dry-run` | Link to the `<canary>-rollback.md` record from Part 5; the template that fired; the time-to-rollback; whether the alarm fired inside the single analysis interval it should have. |
| `cohort-verification` | The verified header-based routing from the Part 3 `x-user-class: dogfood` match — one request with the header reached the canary, one without reached stable. Record the two `curl` lines and the per-pod log lines that confirm the routing. |
| `open-items` | The judge score is synthetic; the next step is to wire a real judge from the `ai-eval-engineer` peer (chapter 02). Any `canary-analysis` deviation from the chapter 01 §3 signal set is named here. |
| `known-limitations` | The exercise's cost metric is nominal; a real cost source joins provider-cost APIs or self-hosted GPU-second attribution. The exercise did not exercise the long-horizon post-promotion watch from chapter 01 §4.2. |

## Requirements

1. **Controller installed and version-recorded.** A one-line `controller: flagger 1.37.0 (helm chart X.Y.Z)` is enough.
2. **RBAC is scoped, not cluster-wide.** Document deviations.
3. **Analysis templates are in Git**, not only in the controller's in-memory config.
4. **The sample workload emits the four Prometheus metrics** with the `version` label.
5. **One green canary run completed** with the weight ramp documented as the nine-column verdict trace (`step`, `weight_percent`, `dwell_start`, `dwell_end`, `ttft_p95_ratio`, `success_rate_pct`, `judge_score_delta`, `cost_ratio`, `outcome`) at `release/runs/<canary>-clean.md`.
6. **One rollback dry-run completed** with the rollback record authored as the ten-field parse contract from Part 5 at `release/runs/<canary>-rollback.md`. The record names which of the four templates fired and reports a `time_to_rollback_seconds` value.
7. **The rollout note exists** at `release/runs/<canary>-rollout-note.md` with all seven named sections from the Part 6 parse contract (`controller`, `analysis-templates`, `clean-canary-run`, `rollback-dry-run`, `cohort-verification`, `open-items`, `known-limitations`) and is specific (no "TBD").
8. **The dogfood cohort routing was verified** at least once (a request with the `x-user-class: dogfood` header reaches the canary; without it, the request reaches stable). The two `curl` lines and the per-pod log lines land in the rollout note's `cohort-verification` section.
9. **Any per-fleet deviation from chapter 01's defaults is documented, not silent.** This includes: the ramp shape vs. §4.1 (step width, dwell length, starting weight), the four templates' `thresholdRange` values vs. the Part 3 contract table, and the §3 signal set vs. the four templates actually authored. Each deviation is called out in the rollout note with a one-sentence rationale.

## Starter guidance

- Default to Flagger if the function has no prior opinion; the CRD vocabulary is richer out of the box and the dashboard is approachable. Switch to Argo Rollouts later if the function also runs the Argo CD / Argo Workflows stack and wants a single tool family.
- `kind create cluster --name canary-lab` + `helm install` is the shortest path to a working controller. Reserve production for later.
- Build the analysis templates incrementally: get `success-rate` and `ttft-p95-vs-baseline` passing first; add `judge-score-delta` and `cost-per-request-ratio` once the first two work.
- Do not skip the rollback dry-run. A canary that has never been rolled back is a canary whose rollback has never been tested.
- Keep the first canary deliberately small. The `maxWeight: 50` ceiling and `stepWeight: 5` default are fine for the lab; the full progression to 100% is for later chapters.
- The synthetic judge-score emission is fine for this exercise — the mechanics of wiring the signal is the lesson, not the quality of the signal itself.

## Acceptance criteria

- A controller installed on a non-production cluster with scoped RBAC, version documented.
- Analysis templates committed to Git whose Prometheus queries match the four parse-contract shapes above (label matchers, aggregation, over-window), and whose `thresholdRange` values either match the pre-committed table or document the per-fleet delta in the rollout note.
- One clean canary run with per-step verdicts, recorded as the nine-column trace (`step`, `weight_percent`, `dwell_start`, `dwell_end`, `ttft_p95_ratio`, `success_rate_pct`, `judge_score_delta`, `cost_ratio`, `outcome`), with `outcome ∈ {pass, fail, hold}` and the final step carrying `hold` at the `maxWeight` ceiling.
- One rollback dry-run with timings and metric trace, including which of the four templates fired. The record uses the ten-field schema from Part 5 (`canary_version`, `stable_version`, `canary_manifest_sha`, `stable_manifest_sha`, `trigger_template`, `trigger_metric_value`, `trigger_threshold`, `canary_weight_at_rollback_percent`, `time_to_rollback_seconds`, `controller_event_trace`).
- A sample workload emitting the four signals with a `version` label.
- The rollout note (≤ 600 words) summarising the run and open items, with the seven named sections from the Part 6 parse contract present and populated (no "TBD").
- The `cohort-verification` section of the rollout note contains two `curl` lines (with-header and without-header) and the per-pod log lines that confirm the header-based routing.
- Per-fleet deviations from the chapter 01 §3 signal set, the §4.1 ramp defaults, and the Part 3 threshold defaults are each surfaced with a one-sentence rationale, not silently applied.

## Stretch goals

- **Wire a real judge.** Replace the synthetic `canary_judge_score` with a real online judge (LLM-as-a-judge on a cheap provider) and compare the behaviour of the analysis template on real signal.
- **Add route-by-route canary.** Deploy two routes on the sample workload (chat and summarise), and demonstrate a canary that promotes on chat but holds on summarise due to a route-specific regression.
- **Wire the pre-promotion load test.** Add the chapter 07 §2 load test as a `pre-rollout` webhook on the Flagger resource (or a `BackgroundAnalysisRun` on Argo Rollouts) so the canary does not start until the load test passes.
- **GitOps the install.** If the environment uses Argo CD, commit the controller install and the canary resource under an Argo application and verify the drift-detection catches a hand-edited canary.
- **Second controller side-by-side.** Install the other controller in a different namespace and author the equivalent canary. Write a 300-word compare-contrast on what each controller does better for this fleet.
