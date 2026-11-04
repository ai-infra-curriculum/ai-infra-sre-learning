# Error-Budget Policy for Non-Deterministic Systems

## Why this chapter exists

Classical error-budget policy is a one-page document that says: *if the budget is burned, feature work stops and reliability work takes priority until the budget is restored*. The budget is driven by a single SLO (request availability) with a well-defined binary event (success or 5xx). Writing the policy is straightforward because the event is unambiguous.

Writing the policy for an AI surface is harder in three specific ways:

1. The output of a "successful" request is a *probabilistic string*, not a status code — so "bad event" is not a free dictionary lookup against the response.
2. There are multiple SLO dimensions (availability, latency, quality, cost) that can burn independently — so you need a reconciliation rule across them.
3. The budget-spend decision is three-way (canary vs. feature vs. refactor), not two-way (feature vs. reliability).

This chapter authors the policy end-to-end. The resulting artifact is what exercise 02 asks you to produce for a team you name.

The classical reference is [SRE Book, Chapter 3 — Embracing Risk](https://sre.google/sre-book/embracing-risk/) and [SRE Workbook, Chapter 2 — Implementing SLOs](https://sre.google/workbook/implementing-slos/).

## 1. What is a "bad event" when the output is a probabilistic string?

### The three honest answers

There is no unique right answer. Each team's error-budget policy picks one or more of:

**(A) Protocol-level bad.** The HTTP response was non-2xx, the stream broke mid-response, the structured-output schema failed to parse, the tool-call JSON was malformed. This is the easiest to measure (classical rules apply) and the least AI-aware. It should always be included but it is never sufficient.

**(B) Latency-bad.** TTFT, TPOT, or ITL exceeded threshold. See `mod-103` for the construction of each. Latency SLIs are *per-request* for TTFT and *per-token* (or per-request aggregated — document which) for TPOT / ITL.

**(C) Quality-bad.** The response violated a constructed quality signal:

- *Judge-model pass-rate on a continuously-sampled production slice.* The eval peer owns the judge; you own the sample ratio, the sampling methodology, and the SLO threshold.
- *Golden-stimulus replay.* A frozen set of inputs is replayed against production; judge checks the pass rate. The signal is less noisy than production sampling but misses true production distribution.
- *Rule-based conformance.* Did the structured-output JSON validate against the schema? Did the output contain a required citation? Did the response hit a toxicity classifier? These are *deterministic given the output* even though the output is stochastic.
- *User-signal feedback.* Thumbs-down rate, abandoned-stream rate, retry-rate from the UI. These are the ground truth but they lag.

**(D) Cost-bad.** The successful request's cost-per-token is over the budget envelope. See chapter 01, section 6.

### The combinator — "composite good event" vs. "multi-SLO stack"

Two patterns, both valid:

**Pattern 1 — Composite good event.** Define *one* good-event predicate that requires *all* of (A) ∧ (B) ∧ (C) ∧ (D). The error budget is a single number. Simpler policy, weaker diagnostic — a burn tells you the composite is bad but not which dimension.

**Pattern 2 — Multi-SLO stack.** Keep each dimension as its own SLO with its own error budget. Policy says what happens when *any single* dimension's budget is burned, *and* a separate rule for cross-dimension correlated burn. More policy to write, much better diagnostic signal. This is the recommended default — covered in depth in `mod-103`.

Pick explicitly; document the choice at the top of the policy document.

## 2. How a quality signal becomes an SLI-shape ratio

A quality signal typically starts life as a dashboard number — "judge pass-rate was 93.1% yesterday." To put it into the error-budget stack it has to become a *ratio of good events to valid events over a window*.

### A worked construction

Target signal: hallucination rate on a continuously-sampled 1% production slice.

- **Valid event.** A sampled production response of a type the judge is calibrated for (e.g., not trivially short, not a known refusal). Document the inclusion filter.
- **Good event.** Valid event where the judge model returned `PASS` *and* the response was not flagged by the rule-based factuality check *and* no user feedback within 10 minutes flagged a hallucination.
- **Window.** 28 rolling days.
- **SLO.** `SLI ≥ 97.0%` over the 28-day window.
- **Error budget.** 3.0% of valid events per 28-day window may be judged hallucinated.
- **Burn-rate alerts.** 14.4 (2% burn in 1h — page), 6 (5% burn in 6h — page), 1 (10% burn in 3d — ticket). See [SRE Workbook, Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/).

Two things to notice:

1. The SLO target is *not* 99.9%. It is 97%. Quality-signal SLOs are set based on what the eval peer and the product team agree is "good enough to ship" — *not* on classical availability-SLO numeracy. A quality SLO of 99.9% would be burned every day on realistic traffic.
2. The valid-event filter is explicit. Signals the judge is not calibrated for *cannot* be in the denominator or the whole SLO becomes noise.

## 3. Reconciling availability SLO with quality SLO

The core question of this chapter: what do you do when two SLOs disagree?

### The three canonical scenarios

**Scenario 1 — Availability is in-SLO, quality is burning.** Serving fleet is healthy, 2xx rate is clean, TTFT is fine; judge pass-rate dropped from 97% to 93% yesterday. The policy MUST allow this to halt feature work *even though classical signals look fine*. This is the whole point of adding a quality SLO: it catches the degradations classical signals cannot see. See `mod-105` for the common causes (prompt regression, retrieval freshness, provider silent swap, judge drift).

**Scenario 2 — Availability is burning, quality is in-SLO.** Infrastructure incident — a GPU pool lost capacity, scheduler backpressure kicked in, 5xx rate spiked on throttled traffic; quality of the requests that *did* succeed is unchanged. Standard classical response: halt risky deploys, spend budget on remediation. Nothing AI-specific here.

**Scenario 3 — Both are burning, correlated.** The dangerous case. A model-provider partial outage (provider is up but degraded) can burn both availability (timeouts, 5xx on long contexts) *and* quality (the answers that return are subtly worse). Your policy needs a cross-SLO correlation rule that recognizes this pattern and routes to the provider-fallback contract (`mod-107`) rather than treating it as two unrelated regressions.

### The reconciliation rule template

Write it as explicit as this, in the policy document:

> If any one SLO's 28-day budget is under 50% remaining, feature work on *the systems that most directly influence that SLO* is halted until the budget is at ≥ 75%.
>
> If two or more SLOs are simultaneously under 50% remaining *and* the burn is time-correlated (within 2 hours), escalate to the on-call incident commander for root-cause investigation before applying single-SLO rules. Correlated burn is almost always a single upstream (provider, retrieval, config push) and should not be treated as two problems.
>
> If the cost SLO is under 50% remaining but no other SLO is burning, raise with the platform team at the next SLO review — this is a *right-sizing* problem, not a reliability halt.

That last point is important. A classical error-budget policy is a two-state halt-or-ship machine. The cost SLO introduces a third state (negotiate the envelope) because cost does not fail users immediately — it fails the platform's long-run viability.

## 4. When to spend the budget on canary, feature, or refactor

The classical error-budget policy's spend decision is binary: either feature work ships or it does not. The AI error-budget policy's spend decision is three-way because there are three claimants to the budget:

- **Canary / model-rollout.** Rolling a new model build, a new prompt template, a new retrieval index, a new quantization. The canary *will* burn some budget — stochastic outputs mean the semantic-diff gate will never be perfectly tight. The policy has to pre-commit to how much budget is allocable to canary.
- **Feature work.** Classical SRE meaning — new product capability, new endpoint, new agent. Shipped under the standard deploy pipeline.
- **Refactor / reliability work.** Scheduler tuning, engine upgrade, retrieval-index optimization, guardrail hardening. Classical SRE spends budget on this; AI SRE does the same.

### The allocation rule

A reasonable default for a healthy team:

- **40% of expected monthly budget** allocated to model / prompt / index rollouts (canary tier).
- **40% allocated to feature deploys** (classical tier).
- **20% reserved for reliability refactor** (the "fund the SRE wishlist" tier).

The numbers are team-specific. What is universal is that *they are set in writing before the quarter starts*, so budget-spend decisions during the quarter are routed against a prior commitment, not negotiated during an outage.

### Trigger thresholds

Encode spend-trigger rules in the policy:

- **Budget remaining ≥ 80%.** All three tiers may spend, with their monthly allocations as rate caps. Canaries ship weekly, features ship on normal cadence, refactor PRs land.
- **Budget remaining 50–80%.** Canary allocation reduces by half (ship fewer, more conservative). Feature allocation unchanged. Refactor allocation unchanged.
- **Budget remaining 25–50%.** Canary halts except for roll-backs. Feature work halts for anything touching the burning surface. Refactor allocation doubles — this is the time to fix what is burning.
- **Budget remaining < 25%.** All non-rollback work halts. Reliability sprint. SLO review called early.

These are illustrative; the exercise calls for your team-specific numbers.

## 5. The policy document outline

The artifact you produce in exercise 02 should have these sections:

1. **SLO surface** — the list of SLOs the policy covers, each with SLI definition, target, window, and burn-rate alert thresholds. Prefer the multi-SLO-stack pattern.
2. **"Bad event" definitions** — one per SLI, with the valid-event filter made explicit.
3. **Reconciliation rule** — what happens when multiple SLOs burn, including the correlated-burn escalation to the IC.
4. **Allocation rule** — the canary / feature / refactor split for the quarter.
5. **Trigger thresholds** — the four bands above (or equivalent), with concrete actions per band.
6. **Review cadence** — monthly SLO review, quarterly allocation refresh, annual policy audit.
7. **Exception process** — how a time-boxed exception is granted (e.g., "the IC may grant a 24-hour exception for a provider-outage workaround, logged in the incident channel").

The policy is a contract between SRE, product, and on-call. Its usefulness is proportional to how specific its numbers are, not how general its principles sound.

## Summary

- **"Bad event" on an AI surface is a choice, not a given.** Pick explicitly from the (A)/(B)/(C)/(D) menu — protocol, latency, quality, cost — and document.
- **Composite good event vs. multi-SLO stack** — pick one. The multi-SLO stack is the recommended default for diagnostic power.
- **Reconciliation rules** — single-SLO halt is classical; correlated-burn escalation is AI-specific; the cost SLO is a negotiation signal, not a halt signal.
- **Three-way spend decision** — canary, feature, refactor — with per-quarter allocations and trigger thresholds that depend on budget remaining.
- **Document the policy** with seven sections: SLO surface, bad-event definitions, reconciliation rule, allocation rule, trigger thresholds, review cadence, exception process.

Next chapter applies this against the on-call rotation that *enforces* the policy in real time.
