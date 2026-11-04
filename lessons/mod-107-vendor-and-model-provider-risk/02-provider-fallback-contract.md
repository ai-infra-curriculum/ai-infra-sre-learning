# The Provider-Fallback Contract

## Why this chapter exists

A dependency map (chapter 01) tells you *what you depend on*; a fallback contract tells you *what happens when a dependency fails*. The failure modes are distinct enough that a single hand-waved "we have fallbacks" does not survive a real incident:

- The primary provider returns 429 for a sustained window (quota breach, see `mod-106` chapter 04).
- The primary provider returns 5xx or `overloaded_error` for a sustained window (capacity event on the vendor's side, chapter 03).
- The primary provider returns 2xx with degraded quality — slower, more refusals, more empty completions, more schema violations (`mod-105` chapter 07).
- The primary provider version-deprecates the model id (chapter 06 §3) and the new default is a different model.
- The primary region goes red while the global API still reports green (Bedrock us-east-1 or Vertex us-central1 degraded, chapter 03 §4).
- Streaming connections drop mid-response at a rate above baseline.

Each has a different right answer. A good fallback contract enumerates them and spells out the policy per route: what we try next, which error budgets we are willing to spend, what quality floor we refuse to go below, how we preserve idempotency, and how we handle a streamed response that was half-delivered when the swap happened. This chapter writes that contract.

Implementation lives in a router or gateway — LiteLLM-shape in the service, Portkey-shape as a gateway, or a bespoke equivalent. The contract is the input to that implementation; the implementation is an artifact of the chapter 02 artifact, not a replacement for it.

## 1. What a fallback contract is

A fallback contract is a per-route document with four parts:

1. **The health signal** — what makes us declare the primary unhealthy and switch, and what makes us declare it healthy again and switch back.
2. **The equivalence table** — the ordered list of alternatives, each with its own SLA, its own quality profile, its own cost profile, and its own residency profile.
3. **The policy** — which alternative kicks in on which failure class, and under what quality floor.
4. **The semantics** — idempotency, retry, streaming, and consistency rules that must hold across any swap.

The contract is short enough to read in one sitting (one to three pages per route) and precise enough to generate router configuration from. It is signed by the service owner and reviewed by the AI SRE function on each change.

## 2. The health signal: when do we switch?

A fallback that fires on a single bad response is a self-inflicted thundering herd; a fallback that fires only after a 10-minute sustained outage misses every 90-second degradation. The right signal is request-aggregate, not per-request.

### 2.1 Error-rate circuit breaker

For every (route, upstream) pair, maintain a **short-window error-rate counter** and open the circuit when the rate exceeds a threshold. Classical breaker shape (Hystrix / resilience4j / go-resiliency):

- **Window.** 30 s rolling, or an equivalent `k`-of-`n` exponential-decay counter.
- **Threshold.** 50% of calls in the window are non-retryable errors (5xx, provider-specific `overloaded_error`, unavailable).
- **Minimum volume.** Do not trip on three samples. Require ≥ 20 calls in the window to avoid flapping on quiet routes.
- **Probe cadence.** When open, allow 1 probe every 30 s to detect recovery. Three consecutive probe successes close the breaker.

Retryable classes (429, transient TLS, gateway 502 on first try) do *not* trip the breaker on their own; they consume the per-request retry budget first and only flow to the breaker if the final attempt failed. `mod-108` chapter 03 walks through the retry taxonomy in more depth.

### 2.2 Latency breaker

Degraded latency is a quality event that an error-rate breaker misses. A second breaker tracks **p95 latency relative to the route's SLO**:

- When p95 TTFT (for streaming) or p95 full-completion latency (for non-streaming) over the last 60 s exceeds the route's TTFT SLO by 2× for two consecutive 60-s windows, declare degraded.
- The response is usually not to switch primaries immediately — latency alone may be a cache miss or a burst — but to raise an alert and preempt the circuit-open decision if an error rate follows.

### 2.3 Provider status signal

The vendor's own status page is a third input. Chapter 04 wires status.openai.com / status.anthropic.com / status.cloud.google.com / status.azure.com / health.aws.amazon.com into the pipeline; the breaker reads a `provider.degraded == true` flag from that ingestor and treats it as a soft cue (not an automatic switch). The reason to treat it as soft: status-page updates often lag the incident by 10–30 minutes, and sometimes a region is degraded but your specific endpoint is not. The AI SRE's default: status-page red combined with any locally observed error elevation is enough to open the breaker; status-page red on its own raises the on-call page.

### 2.4 Quality-regression signal

Quality degradation is `mod-105`'s subject in depth (chapter 07, "Provider Degraded Mode"). The fallback-contract-relevant part: a slow-moving signal (judge drift, refusal-rate spike, schema-violation-rate spike) informs the breaker that the primary is *returning* but *returning wrong*. The contract specifies whether quality-regression alone is sufficient to switch primaries (usually yes for a tier-1 route with a strict quality SLO; usually no for a tier-2 route where the alternative is cheaper but noticeably worse).

### 2.5 Hysteresis

Switching back is harder than switching away. Avoid oscillation: after the breaker closes on the primary, hold on the secondary for a dampening period (e.g., 5 minutes) to avoid flapping during a recovering outage. Record the dampening period in the contract.

## 3. The equivalence table

Not all models that answer a prompt are equivalent. A fallback contract that promotes `gpt-4o-mini` over `gpt-4o` because the primary is 429 may save availability at the cost of a quality SLO that only the user will notice. The equivalence table makes the compromise explicit.

### 3.1 Row shape

For each route, the equivalence table has rows:

```yaml
route_id: summariser-primary
primary:
  provider: openai
  model: gpt-4o-2024-11-20
  region: data_zone_us
  quality_tier: A    # see §3.2
alternates:
  - id: alt-anthropic-sonnet
    provider: anthropic
    model: claude-sonnet-4-6
    region: us
    quality_tier: A
    residency_ok: true
    cost_ratio_vs_primary: 0.9
    activation_triggers: [primary_breaker_open, primary_429_sustained]
    quality_floor_ok: true
  - id: alt-azure-gpt4o
    provider: azure_openai
    model: gpt-4o-2024-11-20
    region: eastus2
    quality_tier: A
    residency_ok: true   # with US-only contract
    cost_ratio_vs_primary: 1.05
    activation_triggers: [primary_breaker_open]
    quality_floor_ok: true
  - id: alt-openai-mini
    provider: openai
    model: gpt-4o-mini-2024-07-18
    region: data_zone_us
    quality_tier: B
    residency_ok: true
    cost_ratio_vs_primary: 0.15
    activation_triggers: [all_quality_a_unavailable]
    quality_floor_ok: false_for_tier1_user_facing
    degraded_mode: true
```

The row's fields are the inputs the router reads. The policy (§4) is the sequencing.

### 3.2 Quality tiers

An equivalence table is only meaningful if "quality" is pinned. The fallback contract commits to a tier scheme — A / B / C, or a numeric score from an offline eval — and every row is placed in it. The pinning is an **offline eval against your workload's representative prompts**, not the model's public benchmark score. See `mod-105` chapter 03 for the canary-eval harness; the same evals power the tier assignment.

A model that benchmarks at 95% on a public LMSYS leaderboard may land at tier B on your workload if your workload has a mix the public benchmark does not reflect (long structured JSON, domain jargon, a specific tool-calling schema). The tier is yours, not the vendor's.

### 3.3 Cost, residency, and license rows

- **Cost ratio.** Normalised to the primary's blended price. Rows below 1 are cheaper-and-worse; rows above 1 are more-expensive-and-same-tier. The policy (§4) uses the ratio to decide "which equivalent do we try first."
- **Residency.** Boolean per row, considering the specific customer-contract or regulatory boundary. A row that violates residency is not an alternate — it is a *declined* alternate, present for reference but excluded from any policy under the relevant tenant.
- **License / content.** Some alternates (e.g., open-weight models served by a third party) have license terms that constrain commercial use or output redistribution. Record the constraint.

### 3.4 Stale alternates

Alternates drift. A model you qualified a year ago may now be the primary of another service with different tier-B behaviour. A quarterly re-evaluation of the equivalence table — running the same offline eval across all table rows — keeps it honest.

## 4. The policy: when do we switch to what?

The policy is a decision tree with the triggers from §2 as inputs and the equivalence-table rows as outputs. Three policy archetypes cover most tier-1 routes.

### 4.1 Mirror-provider active-active

A two-provider policy where primary and secondary run with shared traffic (say 95% / 5% or 90% / 10%). The breaker opens the primary; the router shifts the full share to the secondary; health-probes to the primary bring the share back. The 5% baseline on the secondary buys you continuous validation that the secondary works (no "untested fallback that turns out to be broken on the day") and some natural warming of its prompt cache (chapter 02 §6.2).

**When to use.** Top-tier user-facing routes where a 60-second cold failover is unacceptable. Cost penalty: ~5% on the secondary's rate card. Also the simplest contract to explain to an auditor.

### 4.2 Cold standby

Primary gets 100% of traffic; secondary is wired but idle. On breaker open, router switches 100% to secondary. On breaker close plus dampening period, 100% back to primary.

**When to use.** Routes where cost sensitivity is real and the secondary has roughly equivalent cold-start latency. Downside: the secondary may surface bugs only on the day of the failover ("we thought we routed to Bedrock but the Bedrock IAM role was never provisioned").

### 4.3 Quality-tiered graceful degradation

A tier-A alternate is tried first; if all tier-A alternates are unavailable, the router *may* fall back to tier-B with an explicit degraded-mode flag that the downstream service respects — e.g., the UI shows "Running in degraded mode — results may be less accurate" or a feature is disabled. The contract names the quality SLO's fate: **does this route accept tier-B output, and if so, how is the user informed?**

**When to use.** Routes where "no answer" is worse than "a worse answer" (search, autocomplete, content classification). Not appropriate for tier-1 routes where a worse answer is more damaging than no answer at all (medical, legal, financial decision-support).

### 4.4 Composite policies

Most real routes combine. A tier-1 chat route may run active-active across two tier-A providers and only fall through to a tier-B degraded-mode on the rare event that both tier-A providers are red at once. The contract names the composite: the router's shape (§5) encodes it.

### 4.5 Hard no-op cases

Not every failure deserves a fallback. If the user is in the EU under a residency contract, falling through to a US-only region is *more harmful* than failing the request. The policy names these cases: on a failure of the only residency-compliant upstream, the route **fails the request with a 503** rather than silently switching regions. Chapter 05 §4 treats this as a separate error-budget consumer.

## 5. The router shape: LiteLLM and Portkey as reference implementations

The contract is codified in a router. Two common shapes:

### 5.1 LiteLLM-shape (in-process library)

LiteLLM is an open-source Python library and proxy that exposes a unified `completion()` / `chat.completions.create()` surface across most hosted providers and open-weight hosters. Its router supports fallbacks, retries, routing strategies (round-robin, weighted, latency-based), and model groups.

A LiteLLM-shape fallback contract fragment:

```python
from litellm import Router

router = Router(
    model_list=[
        {
            "model_name": "summariser-primary",
            "litellm_params": {
                "model": "openai/gpt-4o-2024-11-20",
                "api_key": OPENAI_KEY,
                "rpm": 10000,
                "tpm": 2_000_000,
            },
        },
        {
            "model_name": "summariser-primary",     # same group, different backend
            "litellm_params": {
                "model": "azure/gpt-4o-2024-11-20",
                "api_base": AZURE_ENDPOINT,
                "api_key": AZURE_KEY,
                "api_version": "2024-10-21",
            },
        },
        {
            "model_name": "summariser-fallback-a",
            "litellm_params": {
                "model": "anthropic/claude-sonnet-4-6",
                "api_key": ANTHROPIC_KEY,
            },
        },
    ],
    fallbacks=[{"summariser-primary": ["summariser-fallback-a"]}],
    routing_strategy="latency-based-routing",
    num_retries=2,
    timeout=20,
    allowed_fails=3,
    cooldown_time=30,
)
```

The library's fallback is a single-step promotion — on exhausted retries against the primary group, call the fallback group once. Multi-step chains (`primary → tier-A alt → tier-B degraded`) are expressed by chaining fallback lists.

**Caveats.** LiteLLM's abstraction smooths provider APIs but does not perfectly hide them. Streaming semantics, function-calling schema, image inputs, and reasoning-traces (o-series, Claude thinking blocks, Gemini thinking) all have provider-specific shape. Your abstraction either flattens to a lowest-common-denominator surface or exposes provider-specific extensions; the contract names which. Verify current feature parity against the LiteLLM docs — the project moves fast.

### 5.2 Portkey-shape (external gateway)

Portkey is a commercial AI gateway (with a self-hostable OSS edition) that sits in front of your provider clients and centralises routing, fallbacks, caching, guardrails, observability, and cost tracking. The service calls the gateway's unified API; the gateway applies the contract.

A Portkey-shape config fragment (illustrative — refresh schema from portkey.ai docs):

```json
{
  "strategy": { "mode": "fallback" },
  "targets": [
    {
      "virtual_key": "openai-prod",
      "override_params": { "model": "gpt-4o-2024-11-20" },
      "retry": { "attempts": 2, "on_status_codes": [429, 500, 502, 503, 504] }
    },
    {
      "virtual_key": "anthropic-prod",
      "override_params": { "model": "claude-sonnet-4-6" }
    }
  ]
}
```

**Trade-off.** An external gateway adds one network hop and a dependency *on the gateway*. The gateway becomes its own single point of failure; its SLA is a row on your dependency map; its outage is a cross-service event. Deciding gateway vs. library is an architecture call; self-hosted gateways are a middle ground that keeps the gateway-as-contract-point but keeps it on your infra.

### 5.3 Cloud-native gateways

Cloudflare AI Gateway, Kong AI Gateway, AWS Bedrock's cross-region inference profiles (which are themselves a limited fallback mechanism inside a single vendor), Google's Vertex AI endpoint-level routing — each is a point on the same spectrum. The contract is portable across the choices; the implementation is not.

### 5.4 Writing your own

For organisations with unusual constraints (strict residency, offline-air-gapped, novel observability needs), writing the router in your own service code is a defensible choice. The invariants to preserve are the ones the next section documents.

## 6. The semantics: what must hold across any swap?

Routers exist to hide complexity; the complexity does not vanish, it just concentrates in these invariants.

### 6.1 Idempotency

Retries and fallback both repeat work. If the user's request has side effects — a tool-call wrote to a database, an embedding was indexed, a payment was authorised — repeating the request is a correctness bug, not a retry. Three mitigations, each a contract clause:

- **Idempotency keys.** Every request carries a client-generated UUID. The downstream tool executor dedupes by key. Provider-side some APIs honour idempotency keys (Stripe, some AWS, OpenAI's `Idempotency-Key` header has been documented for certain endpoints — verify current scope).
- **Pre-flight separation.** Inference is pure; tool-calls happen downstream of inference. The router may retry/switch freely on the inference step; the tool-execution step has its own idempotency discipline. The contract names where inference ends and side-effects begin.
- **Compensation.** Where idempotency keys are not available, the service's tool layer records every effectful call and compensates on repeat (reverse the write, void the payment). Expensive; reserved for cases where idempotency keys cannot be threaded.

### 6.2 Streamed-response resumption

A streamed response that partially succeeded before the primary failed is the hardest case. Three realistic policies:

- **Discard-and-retry-from-zero.** The simplest. On stream break, drop the partial, call the fallback from the start. User sees a stutter; cost is 1 wasted partial. Appropriate for short responses (<500 tokens).
- **Resume-from-partial.** The router prepends the partial response to the fallback request as context ("previous assistant partial: `<partial>`. Continue from where the assistant left off, without repeating.") Provider-specific — Anthropic and OpenAI chat APIs accept this via a crafted `messages` payload; the continuation's tokenisation and style rarely match perfectly. Use only when the partial-cost outweighs the stutter.
- **Replay-with-cache.** If prompt caching is on (OpenAI prompt caching, Anthropic prompt caching, Google context caching, Bedrock prompt caching), the fallback's re-run is cheaper and faster because the prefix is cached. Not a correctness policy; a cost mitigation.

The contract names the policy per route. UIs that render chat token-by-token usually choose discard-and-retry-from-zero for responses under a threshold and resume-from-partial above it; the threshold is a route-level knob.

### 6.3 Function-calling / structured-output consistency

A primary that returns a structured JSON per schema may fall through to a secondary with slightly different schema compliance. The contract names:

- **Which schema validator is authoritative.** Pydantic / JSON Schema / provider-native structured outputs — exactly one is the source of truth.
- **Behaviour on validator failure.** Retry, fall through, or surface the failure. (`mod-105` chapter 03 §6 treats schema-violation rate as a canary signal.)

### 6.4 Tool-use multi-turn consistency

An agentic turn with 3 tool calls may complete 2 against primary and the third against fallback. The fallback must be able to read the first 2 turns' tool results as context. In practice this means the message array is serialised in a provider-neutral shape, with the router translating to provider-native tool-call formats on every call. LiteLLM and Portkey both do this; writing it by hand is non-trivial — verify round-trip equivalence under a test harness before relying on it.

### 6.5 Context-window mismatch

Fallback alternates may have different context-window ceilings. A primary with 200K context falling through to an alternate with 128K will drop tokens; the contract specifies the truncation policy and whether a request too large for the fallback should hard-fail instead of silently truncate.

### 6.6 Rate-limit header propagation

Providers return rate-limit headers (`x-ratelimit-remaining-tokens`, `retry-after`). The router consumes them for its own health signals; it does not pass them through to the downstream service verbatim, because the headers have different semantics per provider. Normalise to a single router-emitted header (e.g., `x-llm-retry-after-seconds`) or none at all.

### 6.7 Observability continuity

Every call through the router carries a trace-id that propagates into the provider's request-id (OpenAI returns `x-request-id`; Anthropic returns `request-id`; Google and others similar). The router records the mapping. During an incident, the ability to say "this trace-id hit Anthropic request-id R at 14:12:03" shortens triage from hours to minutes. `mod-104` chapter 05 details the trace plumbing; the contract requires it.

## 7. Testing the contract

A contract that is not tested is not a contract. Three test levels:

### 7.1 Config validation

A CI step parses the router's config, asserts every alternate is reachable (keys present, endpoints resolvable), and asserts every route in the dependency map (chapter 01) has a matching contract.

### 7.2 Synthetic fallback drill

Weekly or daily, an automated test sends traffic to a non-production copy of the router, injects a forced 5xx on the primary, and asserts the fallback is exercised, that the response arrives within budget, that the trace is continuous, and that idempotency holds. Non-production because real-provider traffic for a drill is expensive; use a mock provider that returns the error on command.

### 7.3 Game day

Quarterly, the real router in production is nudged: increase the primary's error-rate injection from 0% to a configurable share (e.g., 10%), observe the breaker open, observe the fallback shoulder the load, observe the breaker close. Chapter 09 (chaos engineering, `mod-109`) is the home for the game-day discipline.

## 8. Writing the contract down

The artifact itself is a Markdown or YAML file per route under `fallback_contracts/`, co-located with the service. Its template:

```markdown
# Fallback Contract — summariser-primary

## Route
See dependency-map entry `summariser-gpt4o-prod`.

## Health signals
- Error-rate breaker: 50%/30s window, min 20 calls, 30 s probe.
- Latency breaker: p95 TTFT > 2 × SLO for 2 × 60 s windows.
- Status-page signal: openai[chat].degraded == true from chapter 04 ingestor.
- Quality signal: schema-violation-rate > 2% rolling-60s (from mod-105 canary).

## Equivalence table
| Row | Provider | Model | Quality tier | Cost | Residency OK | Notes |
|-----|----------|-------|--------------|------|--------------|-------|
| primary | openai | gpt-4o-2024-11-20 | A | 1.00 | US-only | |
| alt-1 | azure_openai | gpt-4o-2024-11-20 | A | 1.05 | US-only | same model, different provider |
| alt-2 | anthropic | claude-sonnet-4-6 | A | 0.90 | US-only | different model |
| degraded | openai | gpt-4o-mini-2024-07-18 | B | 0.15 | US-only | user shown "degraded" banner |

## Policy
- Active-active 95/5 across primary + alt-1.
- On primary breaker open → 100% alt-1.
- On alt-1 also breaker open → 100% alt-2.
- On all tier-A unavailable → degraded with UI banner.
- Residency violation → 503 to caller; no silent fallback.

## Semantics
- Idempotency keys: on; passed through to tool layer.
- Streaming resumption: discard-and-retry-from-zero < 500 tokens; resume-from-partial above.
- Schema authority: service-side Pydantic validator.
- Context-window: truncation hard-fail if request > alternate ceiling.

## Testing
- Config validation: `router_contract_test.py`, CI-blocking.
- Synthetic drill: `litellm_fallback_drill.py`, nightly.
- Game-day: quarterly, see mod-109.

## Owners
- Service: @team-summariser
- AI SRE: @ai-sre-oncall
- Last reviewed: 2026-10-14 — scheduled: 2027-01-14
```

## Summary

- **A fallback contract is a per-route document.** It enumerates the health signals that fire the switch, the ordered table of alternates, the policy per failure class, and the semantics the system must preserve across any swap.
- **Equivalence is a quality statement.** An equivalence table that is not backed by an offline eval against your workload is wishful thinking; "equivalent" is yours to prove.
- **Choose the policy, not the library.** LiteLLM-shape, Portkey-shape, and bespoke router are interchangeable implementations of the same contract. Pick the one with the right operational footprint.
- **The invariants cost real engineering.** Idempotency, streaming resumption, function-calling consistency, tool-use multi-turn, context-window mismatch, rate-limit header normalisation, and trace continuity are seven places where naive implementations go wrong. Each is a contract clause.
- **Untested is unreliable.** CI config validation, nightly synthetic drills, and quarterly game-days keep the contract from drifting into fiction.

Chapter 03 turns to the public incident record — the real events that make these contracts non-optional.
