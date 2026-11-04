# Exercise 01 — Prompt Template Versioning and Rollback as a Reliability Operation

**Estimated effort:** 3 hours · **Output:** a running Git-backed prompt registry wired to a toy serving tier, three prompt versions resolvable by content hash, one green progressive-delivery rollout and one timed rollback, plus a rollout note (≤ 600 words).

## Objective

Stand up a minimal, end-to-end **prompt-as-release-artifact** stack and prove the three operations the function must own on this surface: (a) ship a prompt change through the same progressive-delivery controller the model ships through, (b) tag every served response with a resolvable `prompt_hash`, and (c) roll back a prompt from the controller — without a container redeploy — against a measured time-to-rollback target.

The point is not to write a feature-rich prompt platform. The point is to have, at the end of the exercise, a **repeatable operation**: a prompt lands through CI, the rollout controller canaries it with an AI-adapted analysis predicate, the response-length / judge-score regression triggers an auto-rollback, and the audit trail shows exactly which hash was in effect when.

## Prerequisites

- Chapter 01 of this module.
- `mod-111` exercise 01 completed (Flagger or Argo Rollouts installed on a Kubernetes cluster you control). This exercise reuses the controller.
- Prometheus + Grafana available on that cluster.
- A toy model-backed service (the `mod-109` or `mod-111` sample is fine; the key behaviour is that it accepts a POST, calls out to a model, and emits Prometheus metrics with a `version` label).
- A Git repo you can commit to and that is connected to the cluster's CI / GitOps path (Argo CD or Flux is fine; a direct `kubectl apply` from CI works for the lab).

## Problem statement

Work in six parts.

### Part 1 — Author the prompt registry (Git-native)

Create `prompts/<route>/<name>.yaml` under your repo. Each prompt file has YAML frontmatter plus the template body:

```yaml
name: chat/core
owner: application-team-example
eval_set: chat-core-quality@sha256:3f91...
---
system: |
  You are a helpful assistant answering user questions concisely.
template: |
  User: {user_message}
  Assistant:
```

CI must, on merge:

- Compute the canonical content hash of each prompt file (whitespace-normalised).
- Publish the hash and the body to a tiny HTTP service you run on the cluster, exposing:
  - `GET /prompts/<hash>` → template body.
  - `GET /prompts/<route>/<channel>` → the hash currently bound to that channel.
  - `POST /prompts/<route>/<channel>/promote` → rebind a channel.
  - `GET /audit/<hash>` → audit record (author, timestamp, PR URL, reviewers).

The registry can be an in-memory Python / Go service backed by a ConfigMap or a small volume; persistence beyond the lab is not required.

### Part 2 — Integrate with the toy serving tier

Change the toy service so that on each request it:

1. Reads the current `stable` / `canary` channel binding via the registry.
2. Resolves the template body by content hash.
3. Renders the template with the request parameters.
4. Calls the model.
5. Emits the following Prometheus metrics with the `prompt_hash` label:
   - `request_total{route, version, prompt_hash}` (counter)
   - `ttft_seconds_bucket{route, version, prompt_hash}` (histogram)
   - `response_length_tokens_bucket{route, version, prompt_hash}` (histogram)
   - `refusal_rate_total{route, version, prompt_hash}` (counter)
   - `canary_judge_score{route, version, prompt_hash}` (gauge; synthetic is fine)

The response body (or a trailing header) includes `prompt_hash` so a replay against the audit endpoint resolves the full template.

### Part 3 — Define three prompt versions

Author three versions of the `chat/core` prompt:

- `v1` — a short, neutral system prompt.
- `v2` — adds a few-shot exemplar; otherwise equivalent.
- `v3` — deliberately regressing — a wordier system prompt that visibly inflates response length (ask the model to "always explain your reasoning in detail before answering" or equivalent). This is the regression candidate for part 5.

Each produces a distinct content hash via CI.

### Part 4 — Wire Flagger / Argo Rollouts to the registry

Extend the Flagger `Canary` or Argo `Rollout` resource from `mod-111` exercise 01 so its `targetRef` references a Kubernetes object you control (a `RouteConfig` ConfigMap is fine for the lab) whose `stable` and `canary` fields are the channel bindings in the registry. On a rollout:

- The controller "promotes" by calling the registry's `POST /promote` endpoint to rebind `stable`.
- The controller aborts by leaving `stable` bound to the previous hash; the serving tier reads the still-stable binding and continues serving.

Add four analysis metrics:

- `ttft-p95-vs-baseline` — same as `mod-111` exercise 01.
- `judge-score-delta` — same.
- `refusal-rate-delta` — same.
- `response-length-delta` — new in this exercise; this is the AI-adapted signal from chapter 01 §5.6.

Templates are Prometheus `MetricTemplate` (Flagger) or `AnalysisTemplate` (Argo Rollouts) resources committed under `release/analysis/`.

### Part 5 — Run a green rollout and a rollback

- **Green rollout.** Promote `v1 → v2`. Trigger the canary. Observe the controller progress through the weight ramp (use `stepWeight=5`, `maxWeight=50` for the lab). Record per-step timings and the per-metric verdict.
- **Regression + rollback.** Promote `v2 → v3`. The response-length-delta threshold should fire. Observe:
  - The controller aborts the rollout within the analysis window.
  - The channel binding in the registry remains on `v2`.
  - The time-to-rollback is recorded (from the first threshold breach to the moment traffic is fully on `v2` again).
  - The audit trail for the attempted `v3` promotion exists.

### Part 6 — Prompt-only rollback drill (no controller involvement)

Simulate the on-call scenario: `v3` has already been promoted to `stable` without going through a canary (a manual promotion to simulate the "silent prompt edit" failure mode from chapter 01 §2.1). Using only a CLI against the registry — no container redeploy — rebind `stable` to `v2`. Measure the end-to-end time from CLI invocation to the serving tier returning the `v2` `prompt_hash` on fresh requests.

Record:

- The ConfigMap propagation time (if applicable).
- The registry cache invalidation time at the serving tier.
- The service-mesh steady-state reconvergence time.
- The total time-to-rollback.

A reasonable target for a mature team is **≤ 2 minutes** (chapter 01 §6.1). If you miss it, document the gap and the fix.

## Requirements

1. **The registry is Git-backed** — every prompt change lands through a PR, every merge produces a new hash, and the audit record includes the PR URL, the author, and the reviewers.
2. **The serving tier tags every response** with the resolved `prompt_hash` and emits the five Prometheus metrics with the `prompt_hash` label.
3. **Three prompt versions exist** and each resolves to a distinct content hash.
4. **The controller is wired to the channel binding**, not to a container image tag, for the rollout path.
5. **Four analysis metrics are configured** including `response-length-delta`.
6. **One green canary completed** with per-step timings.
7. **One auto-rollback completed** with the time-to-rollback measured.
8. **One manual prompt-only rollback completed** with an ≤ 2-minute target; actual time documented.
9. **An audit entry exists** for each promotion and rollback.
10. **A rollout note (≤ 600 words)** summarises the runs and names the gap-to-target items.

## Starter guidance

- Keep the registry service **simple**. It is 100 lines of Python with a dict, a Pod, and a ConfigMap-mount for persistence across restarts. The exercise is about the operation, not about rebuilding Promptfoo.
- Use `hashlib.sha256` on a deterministic serialisation (sort keys, strip trailing whitespace, LF line endings) — do not use Git blob hashing (`git hash-object`) unless you explicitly document that choice; the serialisation convention must be reproducible outside of Git.
- The `response-length-delta` query is a pair of histogram-quantile ratios between the `stable` and `canary` series. An example shape:

  ```promql
  (
    histogram_quantile(0.5, sum by (le) (
      rate(response_length_tokens_bucket{route="chat", version="canary"}[5m])
    ))
    /
    histogram_quantile(0.5, sum by (le) (
      rate(response_length_tokens_bucket{route="chat", version="stable"}[5m])
    ))
  )
  ```

  Threshold at `< 1.2` (canary cannot inflate median response length by more than 20%).
- The toy service can call any model you have credentials for. If you want to avoid real provider calls, mock the model with a function that returns a fixed response plus the system prompt's length in tokens; the regression candidate will still trip the length delta.
- For the prompt-only rollback timing, set the serving tier's registry-cache TTL to ~5 seconds for the lab. In a real deployment it would be 30–120 seconds; keep the target realistic to the setting.

## Acceptance criteria

- A Git-native prompt registry service running on the cluster, with three prompt versions resolvable by content hash and audit records attached.
- A toy serving tier that reads the channel binding on every request and tags responses with `prompt_hash`.
- Flagger or Argo Rollouts configured with four AI-adapted analysis metrics, including `response-length-delta`.
- One documented green canary and one documented auto-rollback, with per-step timings.
- One documented manual prompt-only rollback meeting (or missing with explanation) the 2-minute target.
- Audit records present for every promotion and rollback action.
- A rollout note (≤ 600 words) summarising the runs, the time-to-rollback, and the open items.

## Stretch goals

- **Codeowners enforcement.** Add `CODEOWNERS` entries and a required-reviewers rule on the prompts directory; demonstrate a merge blocked without eval-peer review.
- **Dual-channel resolver.** Extend the registry and serving tier to resolve both `stable` and `shadow` channels on the same request; call the model twice; emit a `shadow_of` correlation and compare response-length distributions on the shadow stream.
- **Platform compare.** Repeat Part 1 using a prompt-management platform (LangSmith, PromptLayer, or Agenta); write a 200-word compare-contrast on which satisfies chapter 01 §3.1's registry contract out of the box.
- **Prompt rollback chaos.** Wire the prompt-only rollback as an experiment in the `mod-109` chaos program — once a week, a known-regression prompt is auto-promoted and the drill measures time-to-rollback. Fail the drill if the target is missed.
- **Drift annotation.** Emit the registry-write event as a Grafana annotation (per chapter 05 §3.7) so the rollout and rollback times appear as event markers on the serving-SLI strip.
