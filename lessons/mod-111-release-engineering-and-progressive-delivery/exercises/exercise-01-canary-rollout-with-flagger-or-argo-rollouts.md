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

Author at least four `AnalysisTemplate`s (Argo Rollouts) or `MetricTemplate`s (Flagger):

1. `ttft-p95-vs-baseline` — Prometheus query that computes the p95 TTFT ratio of canary vs stable.
2. `success-rate` — Prometheus query for 2xx-rate of canary.
3. `judge-score-delta` — Prometheus query for the delta of judge score canary minus stable.
4. `cost-per-request-ratio` — Prometheus query for the ratio of canary cost vs stable.

Commit all templates to `release/analysis/` in Git. Dashboards are for inspection; the templates are the source of truth.

### Part 4 — Run a clean canary

Deploy the sample workload as stable at version `v1`. Deploy a canary at version `v2` with no regression (identical binary, different version label). Trigger the canary by applying the manifest.

Observe:

- The controller progresses the canary through the weight steps.
- Each step's analysis passes (metrics within thresholds).
- The canary promotes to 50% (the `maxWeight` ceiling) and holds; the final promotion to 100% is manual (a human-reviewable step).

Record the timings per step and the per-step verdict trace.

### Part 5 — Dry-run an auto-rollback

Deploy a canary at `v3` that *does* regress — the simplest way is to add artificial latency in the handler (sleep 500ms before responding) so TTFT fires the `ttft-p95-vs-baseline` threshold. Trigger the canary.

Confirm:

- The analysis fires `failed` on the regressed metric within the analysis interval.
- The controller aborts the rollout and shifts traffic back to stable.
- The event stream shows the rollback (including the metric that fired).
- The time-to-rollback is recorded.

### Part 6 — Rollout note

A short rollout note (≤ 600 words) covering:

- The chosen controller and provider, with versions.
- The analysis template authored, with the Prometheus query shape for each.
- The clean-canary run timings and the per-step verdicts.
- The dry-run rollback — the metric that fired, the time-to-rollback, and the timings of each propagation step.
- Open items: the judge score is synthetic; the next step is to wire a real judge from the `ai-eval-engineer` peer (chapter 02).
- Known limitations: the exercise's cost metric is nominal; a real cost source joins provider-cost APIs or self-hosted GPU-second attribution.

## Requirements

1. **Controller installed and version-recorded.** A one-line `controller: flagger 1.37.0 (helm chart X.Y.Z)` is enough.
2. **RBAC is scoped, not cluster-wide.** Document deviations.
3. **Analysis templates are in Git**, not only in the controller's in-memory config.
4. **The sample workload emits the four Prometheus metrics** with the `version` label.
5. **One green canary run completed** with the weight ramp documented.
6. **One rollback dry-run completed** with the timings and the metric trace.
7. **The rollout note exists** and is specific (no "TBD").
8. **The dogfood cohort routing was verified** at least once (a request with the `x-user-class: dogfood` header reaches the canary; without it, the request reaches stable).

## Starter guidance

- Default to Flagger if the function has no prior opinion; the CRD vocabulary is richer out of the box and the dashboard is approachable. Switch to Argo Rollouts later if the function also runs the Argo CD / Argo Workflows stack and wants a single tool family.
- `kind create cluster --name canary-lab` + `helm install` is the shortest path to a working controller. Reserve production for later.
- Build the analysis templates incrementally: get `success-rate` and `ttft-p95-vs-baseline` passing first; add `judge-score-delta` and `cost-per-request-ratio` once the first two work.
- Do not skip the rollback dry-run. A canary that has never been rolled back is a canary whose rollback has never been tested.
- Keep the first canary deliberately small. The `maxWeight: 50` ceiling and `stepWeight: 5` default are fine for the lab; the full progression to 100% is for later chapters.
- The synthetic judge-score emission is fine for this exercise — the mechanics of wiring the signal is the lesson, not the quality of the signal itself.

## Acceptance criteria

- A controller installed on a non-production cluster with scoped RBAC, version documented.
- Analysis templates committed to Git with Prometheus query shapes documented.
- One clean canary run with per-step verdicts.
- One rollback dry-run with timings and metric trace.
- A sample workload emitting the four signals with a `version` label.
- The rollout note (≤ 600 words) summarising the run and open items.

## Stretch goals

- **Wire a real judge.** Replace the synthetic `canary_judge_score` with a real online judge (LLM-as-a-judge on a cheap provider) and compare the behaviour of the analysis template on real signal.
- **Add route-by-route canary.** Deploy two routes on the sample workload (chat and summarise), and demonstrate a canary that promotes on chat but holds on summarise due to a route-specific regression.
- **Wire the pre-promotion load test.** Add the chapter 07 §2 load test as a `pre-rollout` webhook on the Flagger resource (or a `BackgroundAnalysisRun` on Argo Rollouts) so the canary does not start until the load test passes.
- **GitOps the install.** If the environment uses Argo CD, commit the controller install and the canary resource under an Argo application and verify the drift-detection catches a hand-edited canary.
- **Second controller side-by-side.** Install the other controller in a different namespace and author the equivalent canary. Write a 300-word compare-contrast on what each controller does better for this fleet.
