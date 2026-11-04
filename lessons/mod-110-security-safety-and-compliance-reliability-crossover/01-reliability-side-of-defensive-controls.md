# The Reliability Side of Defensive Controls

## Why this chapter exists

Every model-backed service that ships to production carries a layer the classical SRE discipline does not teach: a stack of defensive controls that sit between the user and the model. Input guardrails, output guardrails, prompt-injection classifiers, PII filters, moderation filters, policy engines, rate-limit controllers aimed at model-extraction, retrieval-time access checks, agent tool allow-lists. The owner of the methodology behind those controls is the `ai-infra-security` peer, by the role split from `mod-101`. The AI SRE's role is not to author the attack catalogue or score the defensive posture; it is to **own the reliability dimension of the controls** — how much latency the guardrail is allowed to add, how much memory the PII filter is allowed to hold, how much tail-latency the rate limiter is allowed to produce, and how the policy engine behaves as a dependency that can itself fail.

The failure mode this chapter prevents is well-documented in the public post-mortem corpus: a defensive control is shipped, nobody owns its latency budget, and three weeks later the serving p99 is 40% worse while the serving team debugs the model and the security team debugs the classifier — neither reads the other's dashboard. The AI SRE's output here is a *contract* the security peer signs onto: "this guardrail may spend 40 ms at p95 and 120 ms at p99, may add this much cache footprint, may hold its own dependency with this fallback behaviour, and we will measure it with these SLIs." Without that contract, every defensive control is a load-bearing surface that no reliability engineer is accountable for.

Chapter 02 writes the joint post-mortem shape that takes over when the contract is violated. Chapter 03 turns the telemetry this chapter produces into the compliance-facing deliverable. Chapter 04 reconciles the reliability gate it defines with the safety gate maintained in parallel. This chapter is the entry point.

## 1. The guardrail stack as reliability surface

An input request to a model-backed service passes through, in order, several defensive hops before it hits the model, and several again on the way back. A reference shape (names vary; the principles do not):

1. **Edge rate limiter / anti-abuse.** Per-tenant, per-IP, per-API-key rate caps. Signals from WAF / anti-abuse upstream.
2. **Prompt-injection classifier.** A small classifier or rule engine that scores the incoming prompt for injection patterns (per the OWASP Top 10 for LLM Applications LLM01 category).
3. **PII filter — inbound.** Pattern matchers and entity recognisers that redact or block personally identifiable information before it is sent to the model or logged.
4. **Policy engine.** Rule evaluation (often OPA / Cedar / a custom DSL) that enforces per-tenant, per-route, per-data-class rules — "this tenant cannot use this model," "this route must have the safety mode on," "this data class cannot leave this region."
5. **Retrieval-time access check.** Row-level / document-level authorisation on the RAG corpus.
6. **Model call.** The model provider or self-hosted inference.
7. **Output guardrail.** Moderation classifier, grounding checker, structured-output validator, URL / tool-call sanitiser.
8. **PII filter — outbound.** The symmetric pass on the generated content before it reaches the user.
9. **Audit log emission.** Writing the request / response and classifier decisions to the audit log, under retention controlled by chapter 05.

Each hop adds latency, memory, a dependency, and a failure mode. From the reliability side, the stack is **not a security concern in isolation** — it is a chain of synchronous calls that sits on the critical path of every request and whose failure shapes the user experience. The AI SRE's working premise is: *every defensive control is a dependency; every dependency has a reliability contract*.

### 1.1 The three questions the AI SRE asks of each control

The AI SRE does not need to author the classifier. The AI SRE does need to answer, for each control:

- **What is its latency contribution at p50 / p95 / p99 of the request?** Measured as a per-hop histogram, not aggregated into "guardrail stack latency."
- **What is its failure behaviour?** When the classifier is unavailable, slow, or returns an error, what does the request do — fail closed (block the request), fail open (let it through and log), fail quiet (let it through silently), or retry? Each has a reliability signature.
- **What is its dependency surface?** Does it call an external model (another LLM-as-a-judge used as a safety classifier)? Does it hold a cache? Does it read a policy store at request time or at config time? Each answer determines a failure mode.

A defensive control whose three answers are not written down is a defensive control the function has agreed to run blind.

## 2. The guardrail-sidecar latency budget

The single most commonly violated contract in this surface is the latency budget. A pattern that recurs across production AI stacks:

- A security team ships a guardrail as a *sidecar* — a separate container in the same pod as the serving process, invoked over localhost HTTP / gRPC. The sidecar runs a small classifier (sometimes a transformer, sometimes a logistic model, sometimes rules).
- The sidecar is tested on a representative latency distribution, passes its functional eval, is shipped.
- Two weeks after rollout, the serving-team p99 is 40% worse than baseline. The dashboard for the model shows no change. The dashboard for the sidecar shows acceptable p95 but a long tail.
- Nobody formally owns the p99 of the sidecar.

The reliability contract fixes this by committing the budget at the SLI level, before the control ships, and by instrumenting each hop so the contract is measurable continuously. The commitment has four parts.

### 2.1 The budget allocation

Decompose the end-to-end TTFT and TPOT SLOs from `mod-103` into the per-hop budget. A fictional but representative allocation on a 2.0 s TTFT p95 budget for a chat route:

| Hop | p50 budget | p95 budget | p99 budget | Owner |
|---|---|---|---|---|
| Edge rate limiter | 2 ms | 5 ms | 10 ms | Platform |
| Prompt-injection classifier | 8 ms | 25 ms | 60 ms | `ai-infra-security` |
| Inbound PII filter | 5 ms | 15 ms | 40 ms | `ai-infra-security` |
| Policy engine | 3 ms | 10 ms | 25 ms | Platform / Security |
| Retrieval auth check | 4 ms | 12 ms | 30 ms | `ai-infra-platform` |
| Model call (TTFT portion) | 800 ms | 1700 ms | 2200 ms | AI SRE / provider |
| Output moderation (pre-stream) | 0 ms | 0 ms | 0 ms | Streamed hop |
| PII filter outbound (pre-stream) | 0 ms | 0 ms | 0 ms | Streamed hop |
| Audit log emission | 0 ms (async) | 0 ms | 0 ms | Platform |

The table is a contract, not a report. Each cell is a commitment the owner signs onto; the serving SLO cannot hold if the sum of the signed cells exceeds the SLO.

### 2.2 Per-hop SLI emission

The `mod-104` observability stack must produce a histogram per hop per route, with the same labels used by the SLO dashboards. A representative Prometheus shape:

```
guardrail_hop_latency_seconds_bucket{hop="prompt_injection_classifier",route="chat",outcome="allow",le="0.025"} 12043
guardrail_hop_latency_seconds_bucket{hop="prompt_injection_classifier",route="chat",outcome="allow",le="0.060"} 12441
guardrail_hop_latency_seconds_bucket{hop="prompt_injection_classifier",route="chat",outcome="block",le="0.025"} 112
guardrail_hop_latency_seconds_count{hop="prompt_injection_classifier",route="chat",outcome="allow"} 12889
guardrail_hop_latency_seconds_sum{hop="prompt_injection_classifier",route="chat",outcome="allow"} 118.4
```

The `outcome` label matters. A block path that averages 200 ms while the allow path averages 25 ms will not show in the aggregate — users of blocked requests also have a latency experience, and that experience affects the service's P99. The same applies to the `cache_status` label for anything that caches (next section).

### 2.3 Error-budget alarms on the guardrail hops

The chapter 03 SLO burn-rate alarms from `mod-103` extend naturally: author one alarm per hop per route, fast-burn and slow-burn, firing when that hop's latency crosses its signed budget. The `ai-infra-security` peer is paged for their hops; the AI SRE is paged for provider and routing hops; the serving p99 regression no longer sits unowned. A guardrail-hop fast-burn alarm is signalled to both peers because recovery often requires both.

### 2.4 Negotiation discipline

The budget is negotiated out of incident. The sequence:

1. The security peer proposes a new control and estimates its latency profile at p50 / p95 / p99.
2. The AI SRE allocates the budget from the TTFT / TPOT SLO; if the allocation is insufficient, the control is reshaped (quantised model, early-exit rules, async emission) or the SLO is renegotiated with the product owner.
3. The control ships behind the budget. The chapter 03 continuous-chaos runbook includes an experiment that proves the control respects its budget under load.
4. Changes to the control that move its latency distribution are treated as a change to the SLO contract — a reviewed change, not a unilateral one.

The chapter 09 pattern from `mod-108` (post-mortems blame the absence of this contract, not the individual control) depends on this discipline being in place before the first incident.

## 3. PII-filter cache-miss impact on TTFT

PII filters are the control most likely to bite TTFT in a non-obvious way. The reason is a cache.

Most production PII filters do two things in sequence: a fast-path classifier (regex, trie, learned classifier) that is cheap to run, and a slow-path validator (an LLM call, a hosted service like a cloud DLP API, an internal entity-resolution service) that is only invoked when the fast-path indicates a probable match. The validator result is cached (keyed on a hash of the text snippet) so repeated queries with similar content do not re-invoke the slow path.

The cache has two failure modes that produce reliability regressions:

- **Cache-miss storm.** A promotion event (marketing push, scheduled report, batch job firing through the API) sends a burst of distinct-content requests, each missing the cache and invoking the slow path. The p95 for the PII-filter hop during the burst is the p95 of the slow path (sometimes seconds), not of the fast path. The chapter 03 `rb-ttft-regression` and `rb-cache-hit-cliff` runbooks from `mod-108` fire, but on the AI SRE's side the signal is "TTFT up," not "PII cache miss rate up."
- **Cache invalidation on filter rules update.** The security team updates the filter rules (new pattern, new entity type), which (depending on the implementation) invalidates the cache. For the next window, every request is a slow-path call. The TTFT regression is step-shaped, correlates in time with a security deploy, and is diagnosable only if the two sides share the deploy timeline.

The reliability instrumentation for this control therefore adds one metric family to §2.2:

```
pii_filter_cache_total{route="chat",outcome="hit|miss|slowpath_called"}
pii_filter_cache_slowpath_latency_seconds_bucket{route="chat",le="..."}
pii_filter_cache_invalidation_total{reason="rules_update|schema_change|manual"}
```

And one explicit runbook (`rb-pii-filter-slowpath-storm`) with mitigations: cache warming before known-burst windows, slow-path fallback (fast-path-only with an async validator audit), cache-replica sharding to amortise an invalidation event. The runbook is co-owned with the security peer per chapter 02's joint post-mortem pattern; the filter's correctness is theirs, but the cache's reliability is the AI SRE's.

## 4. Model-extraction query-rate control and P99

Model-extraction attacks are described in NIST AI 100-2 and OWASP's LLM Top 10 (LLM10 in the current edition at the time of writing — <!-- needs-research: confirm the latest OWASP Top 10 for LLM Applications version and the stable code for model-extraction / unbounded consumption; the list has rotated between versions -->). The `ai-infra-security` peer owns the detection and the methodology. The common defensive control is a **query-rate shaper** — per-API-key, per-tenant, per-session rate caps combined with pattern-based anomaly detection that slows the response when a client's query sequence resembles an extraction pattern.

The reliability consequence is non-obvious and matters: a rate shaper that delays rather than blocks adds a *tail* to the latency distribution for the exact clients the service is trying to retain. The AI SRE's role is to make sure the shaper's behaviour is specified and measured, not to re-derive the shaper.

Three specifications the AI SRE requests from the security peer, with the reliability consequence of each:

- **The delay function.** Fixed delay (adds a constant to P99), token-bucket shaping (adds variable delay with a known ceiling), queue-and-reject (converts latency pressure into a visible error class). Each has a different serving SLO shape.
- **The target cohort.** Whole tenant? Whole API key? Just the extraction-matching queries in a session? The narrower the cohort, the more the P99 dashboard surfaces the control as a distinct signal rather than an unexplained tail.
- **The visibility.** Does the client see a header indicating the shaper is engaged (`Retry-After`, a custom `X-RateLimit-*` header, a specific error code)? Clients that silently see higher p99 blame the model; clients that see a header blame themselves.

A fourth decision — part of the auto-halt chapter 05 of `mod-109` — is the circuit-breaker the AI SRE installs around the shaper itself: if the shaper's own classifier malfunctions and starts delaying every request, there must be a path to disable it. This is not the security peer's circuit; it is the AI SRE's. The shaper's availability does not get to defeat the serving SLO.

### 4.1 The SLI set for the shaper

Add to the per-route SLI set:

```
model_extraction_shaper_action_total{route="chat",action="pass|delay|reject",tenant_class="..."}
model_extraction_shaper_added_latency_seconds_bucket{route="chat",tenant_class="...",le="..."}
```

The "added latency" histogram is explicitly the *delta* the shaper added, measured at the shaper and not inferred from end-to-end latency. Inference-by-subtraction is unreliable when the shaper interacts with retries.

## 5. Adversarial-input-induced timeouts

Not every adversarial input is handled by a classifier. Some fail in the serving path: a prompt with a context-length-exceeded payload, a prompt with a pathological Unicode normalisation that explodes tokenisation time, a prompt template that triggers an output-length blow-up that saturates a worker's KV cache, a prompt whose decoding settings trigger a retry loop in the structured-output validator. These are the reliability shape of what the security peer catalogues under "unbounded consumption" / "denial-of-wallet" / "resource exhaustion attacks" (OWASP LLM04 at time of writing — <!-- needs-research: confirm OWASP LLM Top 10 numbering for Model Denial of Service / Unbounded Consumption in the currently published version -->).

The AI SRE owns three mitigations even though the detection is often the security peer's:

- **Per-request wall-clock timeout.** Every request has a maximum wall-clock budget (TTFT timeout + total response timeout). The timeout is enforced in the serving layer, not in the client. The chapter 01 `rb-streaming-stall` runbook from `mod-108` fires when the timeout trips above baseline rate.
- **Per-request resource budget.** Max input tokens, max output tokens, max retrieval-expansion count, max tool-call depth. These are reliability caps, not security caps — the security peer may raise or lower them for defensive reasons, but the serving layer enforces them for reliability reasons regardless.
- **Worker-level circuit breaker.** A worker that observes a single request consume more than N× the median wall time for its route is allowed to abort the request rather than let it starve co-tenant requests on the same GPU. `mod-108` chapter 01's `rb-worker-crash-under-stream` and `rb-kv-pressure-graceful-preemption` cover the operational response.

The reliability contract names, for each mitigation, the numeric cap and the error code the user sees. An adversarial-input-induced timeout that surfaces as `context_length_exceeded` at the model layer is a defensive signal; the same timeout that surfaces as a generic `504 Gateway Timeout` at the edge is a reliability bug. The AI SRE owns which path the shape takes.

## 6. Policy engine load as a reliability event

Policy engines (OPA, Cedar, home-rolled rule evaluators) are the control the AI SRE is most likely to under-weight because they look like config, not compute. They are compute, and under concurrent load they fail the way any cached-but-not-free dependency fails.

Three failure shapes the AI SRE must plan for:

- **Rule-set reload stall.** The policy engine reloads its rule set from a bundle (OPA's bundle API, Cedar's policy store). A large bundle (thousands of policies, with ReBAC-style graph expansion) takes seconds to compile and evaluate; during the reload, the engine's p99 spikes. Serving requests queue. The chapter 03 `rb-cache-hit-cliff` and `rb-ttft-regression` runbooks fire.
- **Request-scope evaluation blow-up.** A single policy that iterates over a per-request graph (tenant membership, document-level ACL, data-residency tag) has an evaluation cost proportional to the graph size. A particular tenant's graph grows unmonitored; a single request's policy-eval latency crosses the SLO. The signal is per-tenant p99 drift; the detection is a histogram keyed by tenant class.
- **Engine unavailability.** OPA sidecar dies, Cedar service times out, in-process evaluator hits a resource cap. The policy engine was a load-bearing synchronous dependency and the serving layer must now decide whether to fail open (reliability-preserving, defensive-weakening) or fail closed (reliability-degrading, defensive-preserving). This decision is pre-committed with the security peer, not improvised at 3 AM.

The reliability contract therefore includes:

- **A latency budget for the policy engine** in the §2.1 allocation table.
- **A pre-committed open-vs-closed failure mode** per route (chat route fails closed; status-endpoint route fails open; batch-ingest route fails closed with retry-queue). The failure mode is on the runsheet.
- **A specific SLI** `policy_engine_evaluation_seconds_bucket{route,outcome,rule_pack_version}` and a corresponding error-budget alarm.
- **A rule-set reload drill** that runs in the chaos program from `mod-109` chapter 03 as a documented experiment. The drill confirms the hypothesis "a bundle reload of size N completes within M seconds and TTFT does not regress by more than K%."

Treating the policy engine as "security's thing" is the default that produces the public post-mortem the function will later write.

## 7. The broader reliability-side playbook

Across the five surfaces above, the AI SRE's playbook has one shape:

1. **Name each defensive control** present in the request path (or added by a security change) and list it against §1's four questions: latency contribution, failure behaviour, dependency surface, retention cost.
2. **Negotiate a per-hop latency budget** (§2) with the security peer and sign it into the SLO stack from `mod-103`. Author the per-hop SLIs in `mod-104`.
3. **Author the control-specific runbook** for each control's failure mode. The runbook is co-owned with the security peer per the shape chapter 02 writes.
4. **Add the control to the chaos program.** Chapter 03 of `mod-109` includes a guardrail-latency experiment (verify the budget holds under load) and a guardrail-unavailability experiment (verify the fail-open / fail-closed decision is what the runsheet says).
5. **Produce the AI-specific compliance telemetry** the `ai-risk-engineer` and legal peers need (chapter 03 of this module). The telemetry is a by-product of good per-hop instrumentation, not a separate stream.
6. **Establish the shared dashboard.** A joint dashboard with the security peer shows guardrail health (the security peer's view) side-by-side with serving SLO (the AI SRE's view). The dashboard is reviewed every week in the joint operational review (chapter 06 §3 draws the boundary).

The playbook is a three-month arc on a mature fleet. Exercise 01 of this module writes the first column — the latency budget contract — for a chosen fleet.

## 8. What this chapter is not

To keep the boundary from §1 explicit:

- **It does not catalogue attack patterns.** OWASP LLM Top 10 and NIST AI 100-2 are inputs for the security peer; the AI SRE reads them for context (and §4 / §5 cite them) but does not run the attack catalogue.
- **It does not author the classifier.** Prompt-injection classifier training, PII-pattern authoring, moderation-model fine-tuning are the security peer's work.
- **It does not grade defensive effectiveness.** Precision / recall / F1 on an attack suite are the security peer's metrics. The AI SRE reads them as a context signal — a low-precision classifier produces more requests on the block path, which affects §2.2's `outcome="block"` latency distribution — but does not sign off on them.
- **It does not set the retention policy.** Chapter 05 of this module writes the handling discipline; the retention windows themselves are negotiated with legal.

The crossover works because each role contributes the expertise their function was hired for.

## Summary

- **Every defensive control is a reliability dependency.** The AI SRE owns the latency budget, the failure behaviour, the dependency surface, and the retention cost of each control — not the attack catalogue or the classifier itself.
- **The guardrail-sidecar latency budget is the single most load-bearing artifact.** A per-hop allocation table, per-hop SLI emission with `outcome` and `cache_status` labels, and per-hop SLO burn-rate alarms turn the guardrail stack from "security's thing" into a reliability contract the function owns.
- **PII-filter cache behaviour produces non-obvious TTFT regressions.** Cache-miss storms and invalidation events on rules updates need their own SLIs and their own runbook (`rb-pii-filter-slowpath-storm`).
- **Model-extraction rate shaping adds a tail to the P99** for exactly the retained-client cohort. The shaper's delay function, target cohort, visibility, and breaker belong in the reliability contract.
- **Adversarial-input-induced timeouts are a reliability shape of security-adjacent attacks.** Per-request wall-clock timeouts, per-request resource budgets, and worker circuit breakers are the AI SRE's mitigations.
- **Policy engines are compute, not config.** Reload stalls, request-scope evaluation blow-ups, and engine unavailability each need a signed contract — budget, SLI, fail-mode decision, chaos drill.
- **The shape generalises.** For every defensive control that lands on the critical path, the function names the four questions, signs the budget, authors the runbook, drills it, and shares the dashboard.

Chapter 02 writes the joint post-mortem pattern that applies when any of the above contracts is violated in production.
