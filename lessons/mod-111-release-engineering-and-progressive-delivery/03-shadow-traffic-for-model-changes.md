# Shadow Traffic for Model Changes

## Why this chapter exists

Canary (chapter 01) exposes a candidate deploy to a fraction of real users and reads the resulting signals. Shadow traffic does something structurally different: it **mirrors** production traffic to a candidate deploy *without client-visible responses*, lets the candidate serve the request in parallel to stable, and reads the comparison off-line. The user sees only the stable response; the candidate's response is discarded or routed to the analysis pipeline.

Shadow is load-bearing for model releases for two reasons that canary cannot match:

- **Paired comparison on live traffic.** The same request hits both versions. The response-diff pipeline (chapter 02) wants pairs, not population samples. Shadow is the delivery mechanism.
- **Zero user-visible blast radius for first exposure.** A new model is validated on real production shapes before any user sees its output. For releases where even a 1% canary is too risky (safety-adjacent routes, regulated jurisdictions, high-stakes domains), shadow is the only acceptable first exposure.

The complication is that **shadow traffic is not free**. The candidate deploy consumes capacity, the model-provider spend doubles on provider-backed routes, the discarded responses cost real tokens, and the side-effect surface of tool-using agents must be sandboxed or the shadow itself causes production effects. This chapter writes the shape of a well-run shadow program: what to mirror, what to discard, how to measure, how to feed the measurement back into the release gate.

The chapter's boundary: mirror-traffic plumbing in a service mesh is a solved problem (Istio's `VirtualService.mirror` field, Envoy's `shadow` upstream, NGINX's `mirror` directive). This chapter assumes the mesh is capable and writes the AI SRE's layer: **what makes a mirror correct on a model-backed service**, and **what makes the measurement useful**.

## 1. Shadow traffic in the canonical mesh

Istio and Envoy support shadow traffic as a first-class mesh primitive. The pattern on Istio:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: chat-serving
spec:
  hosts:
    - chat.inference.svc.cluster.local
  http:
    - route:
        - destination:
            host: chat-serving-stable
          weight: 100
      mirror:
        host: chat-serving-candidate
      mirrorPercentage:
        value: 100.0
```

Semantics the mesh guarantees:

- The client receives only the response from the primary (`chat-serving-stable`).
- The mirrored request is sent fire-and-forget to `chat-serving-candidate` with the same request body and headers (modulo mesh-added headers; many meshes rewrite `Host` to the mirror's host).
- The mirror's response is dropped at the mesh. The client is not blocked on it.
- Failures of the mirror do not affect the primary path.

These guarantees are what make mirror safe from the user's point of view. They are *not* what makes it correct for release gating. §§2–5 write the AI-specific corrections.

Argo Rollouts has a `setMirrorRoute` step that drives the same mesh primitive declaratively from the rollout controller. Flagger supports mirror traffic as a canary strategy alternative (`spec.analysis.mirror: true`).

## 2. What the mirror must preserve — and what it must not

A naive "mirror everything" introduces three classes of correctness bug on an AI route. The AI SRE's mirror configuration explicitly handles each.

### 2.1 Preserve request identity

The response-diff pipeline (chapter 02) requires the shadow request to be the *same* request the primary served. The mesh's mirror does this by default, but three things can drift:

- **Request ID propagation.** The mirror and the primary must share a request ID so the pair can be rejoined downstream. Istio does this via trace headers (`traceparent`, `b3`) that both branches carry; confirm the primary application does not rewrite the trace header on receipt.
- **Body identity.** Streaming request bodies (SSE uploads, large file uploads) are an edge case; some mesh implementations buffer the body before mirroring and change its characteristics (chunk boundaries, Content-Length handling). For most AI routes (JSON request bodies under a few KB) this is a non-issue; for multi-modal routes with image or audio payloads, confirm the mesh preserves the byte stream.
- **Timing identity.** The mirror receives the request at the moment the primary does, not after. A mesh that buffers the primary's response and only then issues the mirror is doing something different; the mirror's latency measurement will be correct, but the concurrency conditions the mirror sees are not the same ones the primary sees.

### 2.2 Isolate side effects

The mirror runs the candidate's full request path, including any tool calls the candidate makes. On an agent route, "same request mirrored" means two bookings, two emails, two charges — one from stable, one from candidate — unless the candidate path is explicitly isolated.

Three disciplines, pick the one that matches the route:

- **Side-effect-stripped shadow mode.** The candidate deploy runs with tool implementations replaced by no-ops or by recorded responses. The `send_email` tool returns a canned success; the `pay_invoice` tool records the intent and does not call the payment API. This is the default for any agent route in shadow.
- **Sandboxed external dependencies.** The candidate deploy points at sandbox endpoints (Stripe test mode, email-dev endpoint, test index) rather than production endpoints. Side effects happen but in a controlled environment.
- **Shadow-disabled for agent routes.** For routes whose tools cannot be mocked and whose sandbox is not production-like enough, shadow is not used; cohort-first canary (chapter 01 §6.1) is the alternative.

The decision is made per-route and recorded in the release-gate contract (chapter 05). A route that has tools but cannot be run in side-effect-stripped mode is a route that cannot use shadow for release gating; the function accepts the slower feedback loop or invests in the mocking.

### 2.3 Isolate telemetry

The candidate deploy emits metrics, logs, traces. By default those land in the production observability stack and confuse the stable version's dashboards — a 200% RPS on the Prometheus scrape target, a parallel set of logs with identical trace IDs, a doubled error-budget consumption.

The discipline:

- **Shadow telemetry is labelled.** Every metric, log, and span from the candidate carries `version=candidate` (or `release_mode=shadow`). The stable dashboards filter it out; the release-gate dashboards select on it.
- **Shadow logs go to a separate index or retention bucket.** The candidate's raw responses, judge scores, and tool-call records go into a shadow-specific storage path with its own retention policy (shorter — the shadow data is for the release gate, not for ongoing operations).
- **Shadow traces are sampled separately.** The primary's trace sampling policy may be 1%; the shadow's sampling is often 100% (small volume, expensive to miss) and lives in a separate trace back-end or namespace.

A shadow run that pollutes the production observability stack is a shadow run that produces a secondary incident while trying to validate a release.

## 3. The three measurements the shadow is for

A shadow run has value only to the extent its measurements land in the release gate. Three measurement axes are the standard; the fleet picks which ones apply.

### 3.1 Latency measurement

The mirror sees the real production prompt distribution, under real concurrent load (if the candidate is sized to match the primary — see §5). Shadow latency is therefore the most trustworthy pre-promotion latency estimate the function has.

What to measure:

- `ttft_seconds_shadow_bucket{route, version}` — TTFT of the candidate under live-shape traffic.
- `tpot_seconds_shadow_bucket{route, version}` — TPOT, same shape.
- `queue_depth_shadow{route, version}` — candidate's queue depth. A shadow that produces a 10× queue on the candidate is a shadow telling the fleet the candidate is under-provisioned, not that the model is slow.
- `streaming_stall_rate_shadow{route, version}` — same as chapter 01 §3.1.

The latency delta between stable and shadow is **not** a direct estimate of the user-facing latency delta after promotion, because the stable's latency reflects real client-side timing (TCP, TLS, keepalive reuse) that the shadow does not (the shadow is server-initiated). But the component the SRE most wants — the model's compute-plus-queue time — is directly comparable.

### 3.2 Quality measurement

The paired response from shadow is the input to the chapter 02 pipeline. Shadow is the natural home for the diff:

- Every shadowed pair is queued to the diff pipeline.
- Lexical / structural / semantic stages run as per chapter 02.
- Aggregated verdicts emit as `shadow_equivalence_score{route, version, cohort}` and feed the release gate.

Shadow's advantage over canary for quality measurement is that it is **paired** (§4.2 of chapter 02) and that it does not consume user attention — the user does not see the candidate's response and therefore cannot complain about it, regardless of quality. The release gate reads the paired signal as the strongest evidence for or against promotion.

### 3.3 Cost measurement

A shadow run doubles cost on provider-backed routes for the duration of the run. The cost is itself the measurement the gate consumes:

- `cost_per_request_shadow_usd{route, version, cost_source}` — the candidate's per-request cost.
- `tokens_per_request_shadow{route, version, kind}` — the candidate's token count.
- `tool_call_count_shadow{route, version}` — for agent routes, the number of tool calls the candidate made (the agent's path may be different from stable's even for the same outcome).

The cost-delta shadow metric is one of the two best predictors of a "this is cheap to roll out but expensive to run" regression. The other is the retrieval-amplification metric from §4 of chapter 06 of `mod-107`.

## 4. Sampling policy — how much to mirror

Mirror cost is proportional to the mirrored fraction. A 100% mirror doubles the fleet's cost and compute footprint during the shadow window; a 10% mirror halves the shadow's statistical power. The AI SRE sets the sampling policy to the point at which the measurement's confidence interval is tight enough for the gate.

Four standard policies:

- **Full mirror (100%) for a short window.** A few hours to a day; maximum statistical power; expensive. Appropriate for pre-GA canaries, major model upgrades, or rare high-stakes routes.
- **Fractional mirror (5–25%) for a longer window.** A week or two; lower cost; narrower confidence intervals per unit time but more time. Appropriate for regular model releases on a mature fleet.
- **Burst-only mirror.** The mesh mirrors only during specific windows (daily at 09:00 UTC for an hour) to capture peak-shape traffic without sustaining the cost 24/7.
- **Prompt-category mirror.** Mirror 100% of requests matching a specific pattern (prompts over 2000 tokens, requests to the RAG route, requests from a specific tenant class) and 0% of others. Narrows cost but focuses on the cohort the release gate cares about.

The default for most fleets is the fractional + prompt-category combination: 25% mirror overall plus 100% mirror of the specific cohorts in the release-gate contract.

## 5. Capacity planning for the shadow deploy

The candidate deploy must have capacity to serve the mirrored load. Two failure modes recur:

- **Under-provisioned shadow produces misleading latency.** The candidate runs at 50% of stable's replica count, queues under the mirror's load, and reports latency 3× stable's. The release gate reads a regression that is a capacity artifact.
- **Over-provisioned shadow produces misleading cost.** The candidate runs at the same replica count as stable but at a fraction of the load (the mirror doesn't reach it); per-replica GPU utilisation is low; the cost-per-request estimate is inflated.

The discipline:

- **Match the candidate's replica count and per-replica capacity to the mirror's load share.** A 25% mirror on a 20-replica stable deploys 5 candidate replicas, not 20 and not 2.
- **Confirm the candidate's cold-start, cache-warmup, and auto-scaling behaviour match stable.** A model with cold cache behaves differently from a model with a warm cache; the first ten minutes of shadow often throws away before the measurement is trustworthy.
- **Decouple the candidate's node pool from stable's.** The candidate on the same nodes as stable means an exhausted-memory candidate takes stable with it. On GPU-backed routes, use a separate node pool (`gpu.pool=candidate`) with explicit taints / tolerations.

The capacity-planning cycle for the shadow deploy uses the same tooling as `mod-106`: the unit-economics calculator sizes the candidate against the mirror fraction.

## 6. Feeding shadow into the release gate

The release gate (chapter 05) consumes two shadow outputs:

- The **equivalence verdict** from the response-diff pipeline (chapter 02 §2.4 aggregate shape).
- The **latency and cost verdicts** as deltas against stable, with confidence intervals.

The verdict is emitted as a structured document that chapter 05 §3 reads. A representative shape:

```yaml
release_candidate: chat-serving-v4
baseline: chat-serving-v3
shadow_window:
  started_at: 2026-02-10T09:00:00Z
  ended_at: 2026-02-11T09:00:00Z
  mirror_fraction: 0.25
  mirrored_requests: 2_413_000
verdicts:
  equivalence:
    judge_id: judge-sonnet-4-6
    judge_version_tag: 2026-01
    mean_score: 0.84
    mean_delta_vs_baseline: -0.02
    ci_lower: -0.035
    ci_upper: -0.005
    regressed_fraction: 0.038
    threshold: 0.8
    regression_budget: 0.05
    passed: true
  latency:
    ttft_p95_shadow_ms: 1540
    ttft_p95_baseline_ms: 1480
    delta_ratio: 1.041
    threshold: 1.15
    passed: true
  cost:
    cost_per_request_shadow_usd: 0.0087
    cost_per_request_baseline_usd: 0.0075
    delta_ratio: 1.160
    threshold: 1.20
    passed: true
  cohorts:
    - name: ja_JP
      equivalence_mean: 0.69
      regressed_fraction: 0.11
      passed: false            # <-- cohort regression, gate holds
    - name: en_US
      equivalence_mean: 0.86
      regressed_fraction: 0.025
      passed: true
overall: hold                  # cohort failure blocks the aggregate pass
```

The release-gate pipeline reads `overall: hold | promote | rollback` and acts; the detail is for the human reviewer and the fixture back-feed.

## 7. Where shadow does not replace canary

Shadow is powerful but does not retire canary. Three signals canary gives that shadow cannot:

- **Client-side latency.** The shadow's TTFT is measured server-side; real client-side TTFT (DNS, TCP, TLS, retry behaviour under actual network conditions) is only observable with real user traffic.
- **Downstream behaviour.** A user who hits the canary and clicks a follow-up link, who thumbs-down the response, who opens a support ticket, or who converts — that signal is not available in shadow because the shadow response is discarded.
- **Regressions that only appear at user-facing scale.** A canary run at the same RPS as production produces concurrency and cache-interaction effects the shadow's lower-RPS mirror does not.

The AI SRE's default is therefore **shadow then canary**: shadow validates the candidate on paired equivalence and server-side latency; a successful shadow promotes the candidate to a weight-ramped canary; the canary validates on client-side and downstream signals; the full ramp completes. Chapter 05 writes the two-stage gate contract that encodes the sequence.

## 8. Shadow-specific anti-patterns

Named so the function recognises them:

- **The shadow becomes a secondary production.** The candidate deploy stays on indefinitely because "we're always evaluating something," cost doubles permanently, and the analysis pipeline stops being treated as a release gate. A shadow is a bounded operation; end it when the release decision is made.
- **The shadow's side effects leak.** A tool call in the candidate path is not sandboxed and reaches production (duplicate emails, duplicate charges). Per §2.2, the contract is explicit; audit the candidate's outbound network connections before enabling shadow.
- **The shadow's telemetry pollutes the primary dashboard.** The `version` label is missing; the Prometheus scrape target is shared; the dashboard shows aggregate metrics that include the candidate's. The primary SLO dashboard becomes untrustworthy during the shadow window.
- **The shadow measures "the model is better" without measuring cost.** A model that is marginally better but 2× more expensive is a release-gate decision, not a "promote and figure out cost later." Cost sits in the verdict; it does not live in a separate review.
- **The shadow window is too short for the eval window.** The equivalence pipeline has a judge-latency of its own; a shadow that runs for an hour but whose judges take four hours to finish produces a verdict after the release decision would otherwise be made. Size the shadow window to the full judge-latency chain.

## 9. The AI SRE's shadow-traffic checklist

Before enabling shadow on an AI route for the first time:

1. **The mesh supports mirror** (Istio / Envoy / NGINX / a dedicated shadow sidecar) and the primary-vs-shadow isolation guarantees are verified on a sample workload.
2. **The candidate deploy is on its own node pool / replica set**, not on the stable's replicas.
3. **Side-effect-stripped mode is configured** on any tool-using route, or shadow is disabled for that route.
4. **Telemetry carries a `version=candidate` label**; dashboards filter stable-vs-shadow correctly.
5. **Mirror fraction and window are sized** against the required confidence interval for the release-gate thresholds.
6. **The candidate's capacity matches the mirror fraction** — not over-provisioned, not under-provisioned.
7. **The equivalence pipeline is live** (chapter 02 pre-work) and consuming the shadow pairs.
8. **The release-gate verdict shape is emitted** as a structured document (§6), not as a dashboard screenshot.
9. **The shadow window has an end condition** — a release decision triggers teardown; indefinite shadow is not allowed by policy.
10. **The cost of the shadow window is budgeted** and the finance / product owner knows the number before the mirror is enabled.

## Summary

- Shadow traffic mirrors production requests to a candidate deploy without client-visible responses, enabling paired comparison on live traffic.
- Correctness depends on preserving request identity, isolating side effects (sandboxed tools or shadow-off on tool routes), and isolating telemetry.
- Shadow produces three measurement families — latency, quality (via the chapter 02 diff pipeline), cost — that the release gate reads as a structured verdict.
- Sampling policy ranges from full mirror for a short window to prompt-category-targeted mirror; the choice balances cost and statistical power.
- Capacity planning for the shadow deploy mirrors the mirror fraction and uses the `mod-106` calculator; under- or over-provisioning produces misleading signals.
- Shadow does not replace canary — it is the pre-canary stage. The two-stage gate (shadow then canary) is the default.
- Anti-patterns: perpetual shadow, leaking side effects, polluted telemetry, cost-ignorant verdict, under-sized window relative to judge latency.
- A ten-point checklist gates the first shadow run per fleet.

Chapter 04 writes blue-green for large models — the pattern that runs when neither canary nor shadow fits because the artifact is too large to run both versions at full capacity.
