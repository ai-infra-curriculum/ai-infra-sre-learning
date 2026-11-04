# Cost SLOs Tied to Unit Economics

## Why this chapter exists

A classical SRE track treats cost as a non-functional concern tracked on a separate dashboard, with occasional "we're spending too much" conversations handled outside the error-budget framework. For a model-backed service, that separation is fatal — the cost per request can swing 10× between a small model and a frontier model, between a well-tuned prefix-cache and a cold cache, between a primary provider and a fallback. The business unit economics of the surface are inseparable from the reliability contract, and so cost belongs in the SLI surface, not on a sidecar dashboard.

This chapter authors the four core cost SLIs, ties each to a business unit-economics expression, and makes explicit the one place where cost behaves differently from availability / latency / quality — the *response to burn*. A cost SLO that is burning is a negotiation trigger, not a page-the-oncall trigger. Chapter 05 then integrates that distinction into the error-budget policy.

The classical reference for treating cost as a reliability concern is [SRE Workbook, Chapter 18 — Managing Load](https://sre.google/workbook/managing-load/) (admission control, graceful degradation under cost pressure) and [Chapter 26 — Managing Code](https://sre.google/workbook/managing-code/) (the cost-of-change framing). The AI-shaped treatment here extends those ideas with the token / inference economics the LLM surface introduces.

## 1. Why cost is an SLI, not a KPI

The classical distinction: a *KPI* (key performance indicator) is a business metric reviewed quarterly; an *SLI* is a reliability-contract metric measured continuously and enforced via error budget. On a model-backed surface, three things push cost from the KPI column into the SLI column:

1. **The cost of a single request varies by 1–2 orders of magnitude.** A 100-token summary from a small model with a cache hit might cost 10⁻⁵ dollars; a 8k-token reasoning-chain from a frontier provider might cost 10⁻² dollars. A product launch that shifts the traffic mix can move the cost-per-request by 50× without changing the SLO surface the user sees. KPIs reviewed quarterly cannot catch this; SLIs with burn-rate alerts can.
2. **Cost regressions are often silent.** A model upgrade that introduces 20% more output tokens per request (longer reasoning chains, more verbose format) regresses cost silently while quality looks the same or better. A prompt-template change that adds a few hundred tokens of system instructions regresses every subsequent request. Without an SLI watching, these land in the monthly bill, not in the release-gate.
3. **The business has unit-economics bounds the service must live inside.** A chat product monetized at $X/month-per-user has an implicit ceiling of $X × margin on inference cost per user per month. If the service exceeds that, the product is unprofitable. This bound belongs in the reliability contract the same way an uptime bound belongs.

Cost treated as an SLI is defensive against each of these. Cost treated as a KPI gives you a one-month lag on each.

## 2. The four cost SLIs

### Cost per 1M input tokens

Dollars (or local currency) consumed per million input tokens processed, aggregated over a window.

- **Definition.** `total_dollars_on_input / (total_input_tokens / 1_000_000)` over 28 rolling days, broken out per model + provider pair.
- **What it measures.** The *input-side pricing efficiency* of the surface. Includes the direct per-token input price from the provider, the overhead of the gateway and routing, and amortized fixed costs (if self-hosted, the GPU-hour cost divided by token throughput).
- **Why "per 1M tokens."** Provider pricing conventionally lists per-million-token rates; aligning the SLI to the convention makes the number legible to the sponsor and comparable against the pricing-page baseline.
- **Target shape.** The *target is a ceiling*, not a floor. "Cost per 1M input tokens ≤ $X on 28 rolling days" where X is set from the business unit-economics model.
- **Watch for.** Prefix-cache hit rate shifts. A drop in cache hit rate can double effective input cost without any price change on the provider side; your SLI will catch it before the bill does. (Prefix cache references: [vLLM automatic prefix caching](https://docs.vllm.ai/en/latest/features/automatic_prefix_caching.html), [SGLang RadixAttention](https://arxiv.org/abs/2312.07104).)

### Cost per 1M output tokens

Dollars per million output tokens generated.

- **Definition.** Analogous to input. `total_dollars_on_output / (total_output_tokens / 1_000_000)` over 28d, per model + provider.
- **Why it is a separate SLI from input.** Provider output pricing is typically 3–5× the input pricing; the ratio of input to output tokens varies dramatically between workload classes (chat is input-heavy, agentic is output-heavy); and regressions on the output side (more verbose responses, longer reasoning chains) are a different failure mode than on the input side (longer prompts, retrieval bloat).
- **Target shape.** Again a ceiling. "Cost per 1M output tokens ≤ $Y on 28d."
- **Watch for.** Chain-of-thought regressions. A model upgrade that adds a 500-token reasoning preamble to every response can materially regress this SLI; so can a prompt change that encourages longer responses. The release gate from chapter 05 catches these.

### Cost per completed request

Dollars per clean request, aggregated.

- **Definition.** `total_dollars / count_of_clean_completed_requests` over 28d. A "clean completed request" is one that satisfied the partial-success SLI from chapter 03 (did not truncate) *and* the structured-output / tool-call SLIs from chapter 02 (did not fail validation on first attempt).
- **Why "completed" and not "any request."** Costs on failed requests, retries, and rejected tool calls are *waste*. A per-request cost that includes waste is less useful for unit-economics tracking than one that reports the cost of *useful outcomes* only. The failed requests are counted against the quality and availability SLIs; they are not double-counted here.
- **Target shape.** Ceiling, in dollars per request, calibrated to the business's revenue-per-request if monetized at the request level.
- **Watch for.** Retry amplification. A quality regression (schema conformance drops) that sends the application layer into N× retry loops will inflate this SLI even though the per-token prices did not change. The SLI catches the operational cost of quality failures, which is a view the per-token SLIs miss.

### Cost per successful session

Dollars per complete user *session*, where "session" is a surface-level grouping of related requests — a chat conversation, an agent task, a batch job.

- **Definition.** `total_dollars_on_sessions / count_of_successful_sessions` over 28d, where "successful session" is defined per surface (a completed chat conversation that did not abandon, a completed agent task, a finished batch job).
- **Why it exists.** For multi-turn surfaces, the unit-economics ceiling applies at the session level, not the request level. A chat product may budget $0.50 per session even if one of ten requests in that session was unusually expensive. A batch pipeline may budget $5 per 1000-document job. The SLI anchors to the business model, not the engineering abstraction.
- **Target shape.** Ceiling, denominated in whatever the business's revenue-per-session unit is.
- **Watch for.** Session-length drift. A UX change that lengthens the average conversation (more messages, more context accumulated, higher per-request cost as context grows) can regress this SLI without any per-request regression visible. The SLI surfaces the compound effect.

## 3. Tying each SLI to the business unit economics

Every one of the SLIs above has a number attached. The SLO document must say *where that number came from*. The reusable pattern:

1. **Identify the business revenue unit.** Dollars per subscriber per month, dollars per API customer per month, dollars per transaction, dollars per session. One of these is the number the business sells against.
2. **Identify the cost ceiling per unit.** Revenue × (1 - target gross margin) = cost budget per unit. If the product is targeted at a 60% gross margin and the sell-price is $20/user/month, the inference-cost ceiling per user per month is $8.
3. **Translate to the SLI unit.** If users average 100 requests per month with 200 output tokens each, that is 20,000 output tokens / user / month. $8 / 20k tokens = $400 per 1M output tokens as the cost SLO ceiling. If the current rate is $500 per 1M, the SLO is already burning; if it is $200 per 1M, there is headroom.
4. **Document the translation chain.** The SLO document contains the arithmetic. When the sponsor asks "why $400?" the answer is in the document, not in your head.

This is the arithmetic that makes the cost SLO *defensible* against the sponsor and against the finance team. "Our cost-per-1M-output-tokens target is $400 because we sell $20/user/month at 60% margin and users average 20k output tokens per month" is a legible statement. "Our target is $400 because that is what I think we can hit" is not.

## 4. The "product-ask over-provisioning" trap

Product teams routinely ask for latency targets that imply over-provisioning. "We want p95 TTFT ≤ 200 ms on all workloads." A reasonable-sounding ask that, for a workload with 50% cache-miss rate on 4k-token prompts, might require 3× the fleet. The cost of that fleet is the cost SLI's problem.

This is a specific negotiating posture the SRE has to be able to hold:

1. **Model the cost of the ask.** "To hold p95 TTFT ≤ 200 ms at current traffic, we need *N* GPUs at $X/month each, for a total of $Y/month added. That translates to $Z per 1M output tokens added to the cost SLI."
2. **Translate into business terms.** "At our sell-price of $20/user/month and 100 requests/user/month, the latency improvement from 400 ms to 200 ms adds $Z' per user per month in COGS. The gross margin impact is $Z'/$20."
3. **Show the alternative.** "A target of p95 ≤ 400 ms, with the current fleet, holds all users served by the primary path and leaves budget for a canary slot and a 20% headroom. The user-perceived difference between 200 ms and 400 ms is small on a chat UI; the business model implication is material."

If you cannot do this translation, the latency SLO negotiation is a one-way push from product to SRE and the cost SLI absorbs the overrun silently. The cost SLI is your *tool for pushing back*; use it.

Three product-ask patterns to be ready for:

- **"Make it as fast as possible."** Not a target. A target has a number, a margin at that number, and a user-experience justification.
- **"Match our competitor's p95."** Possibly a real target, but one that requires modeling the competitor's cost structure, which you do not have. Counter with your own user-measured threshold and defend it on your unit economics.
- **"Zero over-refusals."** A quality target that has a cost implication through the model class (more capable models over-refuse less but cost more per 1M tokens). Translate the quality target to the cost SLI and show the impact.

## 5. Cost-SLO behavior on burn — the "negotiation trigger" distinction

As `mod-102` chapter 02 noted and this module's chapter 05 will develop, the cost SLO *does not halt releases* the way latency / quality / availability SLOs do. The reason is simple: cost does not immediately fail users. It fails the platform's long-run viability.

The burn response is correspondingly different:

- **Availability / latency / quality burn.** Page on-call, freeze releases on the burning surface, spend reliability engineering capacity on remediation until budget is restored. (Policy from chapter 05.)
- **Cost burn.** Ticket (not page). Flag at the next SLO review. Invoke one of three response paths:
  1. **Right-size the fleet.** The cost regression is a provisioning problem; reduce capacity to the right level. Covered in `mod-106`.
  2. **Right-size the workload.** The cost regression is a workload-shape problem (longer prompts, bigger responses); negotiate with product on token-budgeting.
  3. **Re-negotiate the SLI target.** The cost regression reflects a business-model change the SLI target did not anticipate; the sponsor and SRE re-set the target.

A cost SLO that breaches overnight and triggers a page would be the wrong operational pattern; a cost SLO that silently absorbs a 2× overrun because no one looks at it is the *other* wrong pattern. The design here puts cost in the error-budget framework (visibility, burn-rate alerts, policy response) while routing the response to *reviews and negotiations* rather than *pages and freezes*.

### What this means for alerting

Cost SLI burn-rate alerts should still exist, but at the **slow-burn, ticket-only** end of the multi-window / multi-burn-rate table from [SRE Workbook, Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/). A reasonable configuration:

- **Fast-burn alert (2% in 1h): do not configure.** Cost does not need a page.
- **Medium-burn alert (5% in 6h): ticket, not page.** Opens a ticket for the SRE queue.
- **Slow-burn alert (10% in 3d): ticket, assigned to the SLO review queue.** Lands on the next SLO review's agenda.

Three exceptions where a cost burn *does* page:

- **Suspected egress cost.** A classifier regression or prompt-injection bug that causes the model to respond with pathological length (10k-token responses to simple questions) is a cost *and* quality incident. Page on the quality SLI; the cost SLI is a confirming signal.
- **Suspected adversarial usage.** A spike in input tokens unexplained by traffic growth — a bot flooding the surface with long prompts — is an abuse incident that pages on the security side. The cost burn is a secondary signal.
- **Provider-pricing change.** If a provider raises prices, the cost SLI will burn even with a clean workload. Not a page, but a notification for the sponsor and the capacity planner. (Covered in `mod-107`.)

## 6. Cost SLIs in the error-budget-stack table

Extending the table from chapter 02:

| Dimension | SLI | Target | 28-day shortfall budget | Window | Burn-rate response |
|---|---|---|---|---|---|
| Cost | Cost per 1M input tokens | ≤ $X_1 | — (one-sided ceiling) | 28d | slow-burn ticket only |
| Cost | Cost per 1M output tokens | ≤ $X_2 | — | 28d | slow-burn ticket only |
| Cost | Cost per completed request | ≤ $X_3 | — | 28d | slow-burn ticket only |
| Cost | Cost per successful session | ≤ $X_4 | — | 28d | slow-burn ticket only |

The "shortfall budget" column is blank intentionally — cost SLIs are *one-sided ceiling* SLOs (lower is always better), not two-sided SLIs with budget to burn. The burn is measured as "how far over the ceiling for how long"; the budget is implicit in that drift.

You can, if you prefer, convert the cost SLI to a two-sided form with a budget: "cost-per-1M-output-tokens stays within $X ± $tolerance over 28d." This is less common in practice because the business cares about the *ceiling*, not deviation in either direction. Use the one-sided form unless the sponsor specifically wants the two-sided framing.

## 7. What you measure, and where

A short checklist for the cost SLI instrumentation:

- **Provider billing feed.** Most providers (OpenAI, Anthropic, Azure OpenAI, Bedrock, Vertex) publish per-request cost feeds or billing dashboards; wire these into your telemetry. For self-hosted serving, the cost model is GPU-hour × fraction-utilized, built from the capacity planning work in `mod-106`.
- **Per-request tokenization accounting.** Every request's input and output token count must be tagged on the trace / log event. The OpenTelemetry Generative AI semantic conventions define `gen_ai.usage.input_tokens` and `gen_ai.usage.output_tokens` attributes — use these.
- **Model + provider tag.** The per-model, per-provider breakout of the SLI requires every request to be tagged with which model + provider served it. Same attribute set as the graceful-degradation SLI from chapter 03.
- **Session identifier.** The cost-per-successful-session SLI requires a session-id joining key. For a chat surface, this is the conversation id; for an agent surface, the task id; for a batch surface, the job id.

Prometheus can hold the aggregate counters (`total_dollars_on_input`, `total_output_tokens`, etc.) as the backbone of the SLI; the detailed per-request traces live in your tracing backend. Chapter 06 wires the full lifecycle.

## Summary

- **Cost is an SLI, not a KPI.** The variance of per-request cost, the silence of cost regressions, and the business-side unit-economics ceiling all push it into the error-budget framework.
- **Four cost SLIs** — per 1M input tokens, per 1M output tokens, per completed request, per successful session. Each with a derived ceiling, each aggregated over 28 days.
- **Derive the SLI number from the business unit economics.** Document the arithmetic chain — revenue unit, target gross margin, cost per unit, translation to the SLI unit — in the SLO document. The ceiling is not a guess; it is a model.
- **Defend against product-ask over-provisioning.** The cost SLI is the pushback tool for latency / quality asks that imply fleet growth. Translate the ask into cost-per-token impact and present the business-margin consequence.
- **Cost burn routes to negotiation, not pages.** Slow-burn ticket alerts only; fast-burn pages reserved for cost burns correlated with abuse / safety / quality incidents.
- **Instrumentation points to the OpenTelemetry Generative AI semantic conventions** and the provider billing feeds; the aggregated counters live in Prometheus; chapter 06 wires the lifecycle end-to-end.

The next chapter integrates the four SLI dimensions (availability, latency, quality, cost) into an executable error-budget policy, with explicit rules for release freezes, canary spend, burn weighting, and the handoff to the eval peer's release gate.
