# Exercise 05 — Cache and Retrieval Chaos Experiments

**Estimated effort:** 2 hours · **Output:** two authored chaos experiments (`chaos-prompt-cache-invalidation`, `chaos-retrieval-index-drop`), one tier-2 staging-only experiment (`chaos-prompt-cache-poisoning`), run logs for each, and fixture-ledger entries for the findings

## Objective

Author and run the two cache / retrieval chapter 03 catalogue entries — `chaos-prompt-cache-invalidation` (chapter 03 §4, the invalidation variant) and `chaos-retrieval-index-drop` (chapter 03 §10). Separately, author the tier-2 staging-only variant `chaos-prompt-cache-poisoning` (chapter 03 §4, the poisoning variant) without running it in production — the purpose is to specify what the staging drill looks like and why it must not be tier-1.

Cache and retrieval are the surfaces where quality and serving metrics interact most tightly on an AI fleet: a cache miss manifests as a TTFT regression, a retrieval drop manifests as a quality regression (or, more dangerously, as a false-confidence response). This exercise forces the function to confront the quality half of the three-family steady state (chapter 01 §2) under chaos.

## Prerequisites

- Chapter 01 (steady state), chapter 02 (operator), chapter 03 §4 and §10 of this module.
- Exercises 02 and 03 complete — the chaos operator and the serving-path experience base are in place.
- A product with a prompt cache (either a vendor-side prefix cache, an in-service cache like a Redis or an application-layer cache with stable prefix keys) **and** a retrieval pipeline (vector index with an application that RAGs against it). If the environment has only one of the two, run the half you have and document the gap.
- The runbooks `rb-cache-hit-cliff`, `rb-ttft-regression`, `rb-retrieval-index-staleness`, `rb-embedding-drift` from `mod-108` chapter 01 for the handoff mapping.
- A canary query set with a judge-pipeline that scores groundedness and / or a hallucination proxy — exercise 05 cannot reason about the retrieval experiment's quality hypothesis without this.
- The function's safety / policy peer's signature on the policy that keeps the poisoning variant tier-2 (chapter 05 §3).

## Problem statement

Work in six parts.

### Part 1 — Author `chaos-prompt-cache-invalidation`

Spec under `chaos/experiments/chaos-prompt-cache-invalidation.{yaml,md}`. Mechanism options:

- **Admin-API purge.** A chaos-wrapper job that calls the cache admin API to purge a labelled slice (one route, one tenant). Cleanest; requires the cache to expose an admin API.
- **HTTPChaos on the cache service.** Intercept the cache-lookup responses on a label-selected pod slice, returning cold-miss for the duration. Works when the cache is in-service; more invasive for distributed caches.
- **Template perturbation.** The chaos-wrapper rotates a per-route "cache-salt" value that invalidates the prefix without touching the cache itself. The closest analogue to the production failure (a template change).

Pick one. Hypothesis is that TTFT regresses by (cache-miss-fraction × cache-miss-latency) on the slice and recovers within the configured warm-up window. Blast radius: 5% of a single route, 20-minute window, auto-halt on TTFT > 3× baseline for 5 min.

### Part 2 — Author `chaos-retrieval-index-drop`

Spec under `chaos/experiments/chaos-retrieval-index-drop.{yaml,md}`. Two variants, run both:

- **Pod-kill variant.** `PodChaos` kills the vector-index pod. Reproduces a service-gone failure.
- **Partition variant.** `NetworkChaos` partitions the application pod from the retrieval service. Reproduces a reachable-but-unreachable failure.

The hypothesis is central: the application degrades *honestly* — the response includes an explicit "I don't have retrieval data" shape rather than confident-wrong. The judge-pipeline on groundedness stays within tolerance on the degraded-shape responses. The runbook ties are `rb-retrieval-index-staleness` and (adjacent) `rb-embedding-drift`.

Blast radius: single replica or single pod, 10-minute window, auto-halt on groundedness dropping below the floor (false-confidence is the dangerous mode, not latency).

### Part 3 — Author `chaos-prompt-cache-poisoning` (tier-2, staging-only)

Spec under `chaos/experiments/chaos-prompt-cache-poisoning.{yaml,md}` with a **bold tier-2 banner** at the top of the doc. The spec's point is to document what a staging run would do, not to run it in production:

- Mechanism: a test harness that injects a crafted cache key via the cache admin API (not reachable from production chaos operator), then queries with a different tenant's credentials.
- Hypothesis: the system fails closed. Cache lookups scope by tenant; a cross-tenant hit is rejected and increments `prompt_cache_cross_tenant_lookup_rejected_total` (or the equivalent safety counter).
- Blast radius: staging only, no production cohort. Explicitly marked no-production.

The exercise does **not** run this experiment unless the environment has an isolated staging cluster plus the safety peer's sign-off. The deliverable is the spec and a one-paragraph explanation of why the tier-2 marking is appropriate.

### Part 4 — Run the two tier-1 experiments

For `chaos-prompt-cache-invalidation` and both variants of `chaos-retrieval-index-drop`:

- Dry-run.
- Run under blast-radius controls with a load generator sending canary queries to the affected route.
- Observe: cache-hit-rate panel, TTFT panel, judge-pipeline groundedness panel, application's "retrieval unavailable" counter.
- Record timings, dashboard screenshots, and any auto-halt firings.

### Part 5 — Interpret

For each run, identify at least one finding. Representative:

- Cache warm-up takes longer than the hypothesis — fixture: adjust the warm-up window in the cache config, or add a pre-warm canary for the slice.
- Retrieval drop produces confident-wrong responses on a non-trivial fraction of the canary — fixture: a CI test on the application's retrieval-unavailable response shape, plus an alert on judge-groundedness in the retrieval-degraded branch.
- Retrieval drop recovers cleanly — fixture: a chaos-amendment to add to the daily cadence.

### Part 6 — Ledger

Each finding gets a fixture-ledger entry per the `mod-108` chapter 05 schema. The entries cross-link to the chaos run ID and the owning peer (`ai-infra-pipeline` for retrieval, `ai-infra-inference-engineer` and `AI Engineering` for cache).

## Requirements

1. **Three experiments authored** (`chaos-prompt-cache-invalidation`, `chaos-retrieval-index-drop`, `chaos-prompt-cache-poisoning`); two run, one documented tier-2 only.
2. **Chapter 03 §1 shape for each** — eight sections, environment-specific.
3. **Tier-2 experiment banner is explicit** — the doc clearly signals that this experiment does not run without staging + safety peer sign-off.
4. **Load generator used for the tier-1 runs**, with query shape and RPS recorded.
5. **Auto-halt wired to the quality-floor signal**, not only to serving metrics. The retrieval experiment's dangerous mode is confident-wrong at normal latency — the halt must detect that.
6. **At least one finding per tier-1 experiment ledgered.**
7. **The safety peer's acknowledgement of the tier-2 spec** is recorded (a Slack message, a comment on the doc, a signed PR).
8. **No run touches live customer data.** The canary query set is synthetic or anonymised.

## Starter guidance

- **Start with the invalidation experiment.** The cache cold-miss is the gentlest shape and gives the operator a first look at the quality-side steady state in action.
- **For the retrieval experiment, instrument the judge-pipeline carefully.** The hypothesis is about the *shape* of the response, not its latency. A judge that only scores final-answer quality without regard to honesty will mis-rank "confident wrong" above "honest decline."
- **The poisoning tier-2 spec is a thought experiment the first time through.** Write it with the same rigour as the tier-1 specs; the safety peer will read it precisely because it is not going to run.
- **If the environment has no judge-pipeline, use the exercise 04 fallback metric (cost-per-request) as a secondary health signal.** Document the gap and ledger a fixture for adding a judge.
- **Pair the retrieval experiment with the fallback contract from exercise 04.** If retrieval has a fallback (e.g., a replica index or a different search backend), the chaos run exercises that fallback's contract. That is a chapter 04 §2 adjacent finding.

## Acceptance criteria

- Three experiment artifacts in the chaos-program repo.
- Run logs for the two tier-1 experiments (including both retrieval-drop variants).
- The tier-2 spec is explicitly marked and has recorded safety-peer acknowledgement.
- At least one ledgered finding per tier-1 experiment.
- Dashboards show the quality-signal panel alongside the serving-signal panel for each tier-1 run.
- A one-page exercise write-up covering the environment, the three specs, the tier-1 run results, the tier-2 rationale, and the next experiments the function would add.

## Stretch goals

- **Compose the cache and retrieval experiments.** A chained scenario — `chaos-prompt-cache-invalidation` on a RAG route *while* `chaos-retrieval-index-drop` is active — exercises the hardest double-fault condition. Observe whether the application's degraded-mode response composes (two "sorry, I don't have X" signals in one response is a real failure mode).
- **Shadow-replay a tier-2 scenario.** With the safety peer's sign-off, use shadow traffic (inputs replayed, outputs discarded) to verify the tier-2 poisoning experiment's hypothesis in staging without affecting any real response.
- **Ledger the embedding-model-pin signal.** If the retrieval chaos run reveals that an embedding-model version change would be ambiguous (the system has no `embedding_model_version` counter), file that as a fixture for the observability stack.
- **Wire the cache chaos into the pre-promotion gate for template changes.** `AI Engineering`'s template-deploy pipeline should run the invalidation experiment on a staging cache before promoting a template that moves the prefix hash.
- **Test the application's retrieval-unavailable response shape against the eval set.** The canary queries should have "honest-decline" ground truth entries so the judge can score the degraded responses correctly. If they do not, author them and ledger the eval-set extension.
