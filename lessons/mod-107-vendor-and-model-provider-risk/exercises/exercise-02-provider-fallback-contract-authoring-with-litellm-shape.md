# Exercise 02 — Provider-Fallback Contract Authoring with LiteLLM-Shape

**Estimated effort:** 3 hours · **Output:** one Markdown contract (~2000 words) and a working LiteLLM (or Portkey) router config that implements it

## Objective

Author the fallback-contract artifact described in chapter 02 §8 for one tier-1 route from your surface map (exercise 01). The contract must enumerate the four parts (health signals, equivalence table, policy, semantics) and be paired with a working router config — LiteLLM Python or Portkey JSON, your choice — that implements the policy. The two must be consistent: a reviewer can read the contract, open the config, and see the enforcement.

A good deliverable survives an incident walk-through: "primary is 503 for 10 minutes — what does the router do?" "primary returns tier-B-quality for an hour — what does the router do?" "EU request comes in, primary is red — what does the router do?" The answers are in the contract and demonstrated by the config.

## Prerequisites

- Chapters 01 and 02 of this module.
- One tier-1 route from your exercise-01 surface map, with a known primary vendor and at least one candidate alternate.
- A working Python 3.10+ environment with `litellm` installed (or a Portkey account / self-hosted Portkey OSS).
- A sense of the route's quality floor — either a canary-eval score or a qualitative "we commit to tier A" statement.

## Problem statement

Author the contract in six sections and produce the paired config.

### Section 1 — Route reference

Link to the surface-map row (route_id). Restate the primary vendor, model, region, and SLO target in one paragraph.

### Section 2 — Health signals

Define the three-to-four health inputs the router uses to decide whether the primary is healthy:

- **Error-rate breaker.** Window length, threshold, minimum-volume guard, probe cadence, close criteria. Name the retryable error classes that *do not* trip the breaker on their own (429 with retry-after; transient TLS; gateway 502 on first try) and the non-retryable classes that *do* (sustained 5xx, provider-specific `overloaded_error` / `ThrottlingException` / `RESOURCE_EXHAUSTED` / 529).
- **Latency breaker.** The p95 threshold relative to the route's TTFT SLO, the window length, the number of windows for a declaration.
- **Status-page signal.** How the chapter-04 ingestor's event enters the breaker's state; whether it is sufficient alone or only in combination.
- **Quality signal (optional).** If the canary-eval harness emits a schema-violation-rate or judge-divergence signal, state the threshold.

### Section 3 — Equivalence table

Author the ordered table of alternates per chapter 02 §3.1. Each row has:

- id, provider, model (dated snapshot), region.
- quality_tier (A / B / C, grounded in a canary-eval score if you have one).
- residency_ok (boolean for the route's declared residency).
- cost_ratio_vs_primary (number; your estimate is acceptable, note the date-of-check).
- activation_triggers (which health signals activate this alternate).
- quality_floor_ok (whether the route's SLO accepts this tier's output).
- degraded_mode (boolean + what the user sees if true).

A minimum of **two tier-A alternates** and one **tier-B degraded** row for a tier-1 route. Fewer is acceptable only with explicit justification (irreplaceable capability on primary, no viable second tier-A, etc.).

### Section 4 — Policy

The decision tree per chapter 02 §4. Pick one of the archetypes (mirror active-active, cold standby, quality-tiered degradation) or a composite. State:

- Steady-state traffic split.
- Breaker-open action.
- Breaker-recovery (hysteresis) action.
- Hard-no-op cases (residency, quality-floor) — explicit list of when the route **fails the request** instead of falling over.

### Section 5 — Semantics

The seven invariants per chapter 02 §6. For each, state the chosen policy:

- Idempotency keys — on/off, scope, who dedupes.
- Streaming resumption — discard-and-retry vs. resume-from-partial vs. hybrid; the threshold.
- Function-calling / structured-output schema — authoritative validator and behaviour on validator failure.
- Tool-use multi-turn consistency — whether the fallback can read primary's tool results; how the message array is serialised cross-vendor.
- Context-window mismatch — truncation policy; hard-fail threshold.
- Rate-limit header propagation — normalised header name or none.
- Observability continuity — how the trace-id maps to each provider's request-id.

### Section 6 — Testing

The three test levels per chapter 02 §7:

- Config validation step (what the CI checks).
- Synthetic fallback drill (frequency, injection method, success criteria).
- Game-day plan (who runs it, when, what is injected).

## Paired artifact — router config

Produce a working config in one of two flavours:

### Option A — LiteLLM (Python)

A `router.py` module that:

- Defines a `Router` with `model_list` containing the primary and alternates from the equivalence table.
- Sets `fallbacks`, `num_retries`, `timeout`, `allowed_fails`, and `cooldown_time` consistent with the health signals (section 2) and the policy (section 4).
- Includes a minimal test (pytest or `if __name__ == "__main__"`) that hits a mock provider to demonstrate the fallback fires on an injected 503.
- Comments link each config choice back to the contract's section.

### Option B — Portkey (JSON)

A `portkey-config.json` using Portkey's fallback / conditional-routing strategy that encodes the equivalence table and policy. Include a note on how a local Portkey OSS gateway is wired to the service's provider clients.

Either option is acceptable; choose the one that matches your platform.

## Requirements

1. **Contract covers all four parts** (health signals, equivalence table, policy, semantics) plus the route-reference and testing sections.
2. **Equivalence table has at least two tier-A alternates** or an explicit justification.
3. **Hard-no-op cases are named.** If none apply, explicitly state why none apply.
4. **Semantics section addresses all seven invariants.** Partial answers for a specific invariant are acceptable only with "not applicable — <reason>."
5. **Router config exists and parses.** A LiteLLM `Router(...)` instantiates without error; a Portkey config is schema-valid.
6. **Config and contract are consistent.** If the contract says 50%/30s breaker, the config matches. If the contract says `discard-and-retry < 500 tokens`, the code shows the branch.
7. **A mock test runs.** A small script demonstrates the fallback firing on an injected failure (503, 529, `overloaded_error`). The output is captured in the deliverable.

## Starter guidance

- LiteLLM's abstraction smooths provider APIs but is imperfect. Verify current feature parity for your specific feature set (structured output, tool-calling, vision) in the LiteLLM docs. Record any gap in the Semantics section rather than discovering it in production.
- Portkey is a gateway hop. If you choose it, note the gateway's own SLA and availability on the dependency map (exercise 01).
- The hardest semantic is streaming resumption. For first-pass, choose discard-and-retry-from-zero globally and note the limitation; upgrade to resume-from-partial only with a test harness that verifies tokenisation and style continuity.
- Idempotency keys cost nothing to add and save correctness during retries. If your route has any side effects (tool calls, writes), turn them on in the contract.
- Do not over-specify. A contract that enumerates ten alternates without a quality basis is worse than one that names two with a canary-eval score behind each.
- The quality floor is yours to decide; the vendor's benchmark is not sufficient. If you do not have a canary eval yet, note it as a gap and schedule it.

## Acceptance criteria

- Markdown contract ≤ 2500 words covering all six contract sections.
- Equivalence table with ≥ 2 tier-A alternates (or explicit justification for fewer).
- All seven semantics invariants addressed.
- Router config (LiteLLM Python or Portkey JSON) that parses and encodes the policy.
- Mock test that demonstrates fallback firing on an injected failure, with captured output in the deliverable.
- Contract-to-config cross-references on every policy choice.

## Stretch goals

- **Multi-route contract.** Author contracts for a chat route *and* an embedding route *and* a moderation route. Note where the pattern repeats and where it diverges.
- **Streaming resumption with provider-partial.** Implement resume-from-partial for responses above a threshold. Write the integration test that compares a resumed stream's full text against a non-interrupted baseline; measure the stylistic continuity quantitatively.
- **Status-page signal wiring.** Combine with exercise 04's ingestor: subscribe the router to the normalised provider-health event bus and use the signal as a third input to the breaker.
- **Quality-floor guardrail.** Hook a judge model into the router so degraded tier-B output is post-validated and, if it fails, the route returns a hard error rather than the tier-B response. State the trade-off (latency cost of the judge pass) in the contract.
- **Game-day drill script.** Author the chapter 02 §7.3 game-day plan as a runnable notebook/script that injects errors against a staging router, measures the breaker's time-to-open, and verifies the dampening period. Produce a one-page game-day report template.
