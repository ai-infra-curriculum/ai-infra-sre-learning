# Error-Budget Policy in Practice

## Why this chapter exists

The previous four chapters authored the SLI surface — latency, quality, availability-of-streaming, cost. The error-budget policy from `mod-102` chapter 02 named the policy *shape*. This chapter is where the SLI surface and the policy shape meet: concrete decision rules for when to freeze releases, when to spend budget on canary, how to weight latency burn against quality burn against cost burn, and how the policy reconciles with the eval peer's release gate. The output is a policy that an on-call primary reads at 03:00 and resolves to action without asking a question.

The classical references are [SRE Book, Chapter 3 — Embracing Risk](https://sre.google/sre-book/embracing-risk/), [SRE Workbook, Chapter 2 — Implementing SLOs](https://sre.google/workbook/implementing-slos/), and [SRE Workbook, Chapter 5 — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/). The AI-side error-budget policy from `mod-102` chapter 02 is the prerequisite for this chapter; this module's chapter 05 specializes it with the multi-dimensional stack from chapters 01–04.

## 1. The three decisions an error-budget policy makes

Before the specifics: an error-budget policy is a decision function from *state of the budgets* to *allowed actions*. Three decisions are always present:

1. **The release-freeze decision.** When a burn warrants halting new feature work, new model rollouts, or new prompt / config / retrieval changes on the affected surface.
2. **The budget-spend decision.** When budget is available, what is it spendable on — canary rollout, feature deploy, reliability refactor — and in what proportion.
3. **The burn-weighting decision.** When multiple dimensions are burning simultaneously (correlated or independent), which gets priority attention, which triggers which runbook.

Classical SRE policies address (1) with a two-state halt-or-ship rule and (2) only implicitly. The AI-shaped policy has to address all three explicitly because the dimensions are more varied and the failure modes more correlated.

## 2. The release-freeze decision — when to halt what

A naive policy says "if budget < 50%, halt all releases." On a multi-dimensional surface that is both too aggressive (one SLI burning should not freeze an unrelated surface) and not aggressive enough (a correlated burn is a bigger risk than an independent one).

### The surface-level freeze pattern

Rather than a global freeze, freeze *the systems most directly influencing the burning SLI*. A reusable mapping:

| Burning SLI | Freezes what |
|---|---|
| Protocol-level availability | Infrastructure changes to the serving fleet (manifest / Helm / engine-version / HPA policy changes). Feature deploys on the same surface. |
| TTFT, TPOT, ITL, E2E, streaming-stall | Engine configuration changes (batch size, scheduling, prefix-cache knobs). New model rollouts. Context-length expansions. |
| Partial-success | Engine version bumps. Scheduler / preemption policy changes. |
| Graceful-degradation (above threshold) | Provider routing changes. Fallback-path changes. Not an SRE-side freeze; an incident flag. |
| Hallucination rate | New model rollouts. New prompt template changes. New retrieval-index pins. Changes to judge-model / eval pipeline that could shift the signal. |
| Toxic-output rate | Model / prompt / guardrail-sidecar changes. Usually paged, not just frozen. |
| Tool-call-success | Model changes on the surface with tool-calling. Tool schema changes. |
| Structured-output-conformance | Model changes. Schema changes. Constrained-decoding configuration. |
| Refusal (both sides) | Model changes. Prompt-template changes. Guardrail changes. |
| Format-drift | Model changes. Prompt-template changes. UI contract changes. |
| Cost (per 1M input/output tokens) | *Not a freeze.* Negotiation trigger — chapter 04 section 5. |
| Cost (per request / session) | Same. |

The pattern: the freeze targets the smallest set of change types that could have *caused* the burn. Changes to unrelated systems are allowed to continue.

### Multi-tier freeze bands

Within the surface-level freeze, the classical four-band table from `mod-102` chapter 02 section 4 applies — but with cost SLIs excluded (they do not halt) and quality SLIs having their own burn-rate numeracy (see chapter 02 of this module, section 4). A concrete rendering:

| Remaining budget on the SLI | Canary rollouts | Feature deploys on surface | Reliability refactor on surface |
|---|---|---|---|
| ≥ 80% | normal cadence | normal cadence | normal cadence |
| 50–80% | reduced — one canary per week, no new model rollouts | unchanged | unchanged |
| 25–50% | halted (except rollbacks) | halted on this surface | doubled — this is where the fix lives |
| < 25% | halted | halted | reliability sprint; SLO review called early |

Keep in mind: the bands apply *per SLI*. A policy that freezes the whole surface when any SLI drops below 50% is too coarse; one that only looks at the aggregate budget misses the correlated-burn case below.

## 3. The budget-spend decision — allocating under the quarter

The AI-shaped policy from `mod-102` chapter 02 introduced the three-way spend split (canary / feature / refactor). The latency + quality + streaming-availability + cost stack from this module makes the allocation more specific.

### The default split

For a steady-state platform with a mature model-rollout cadence:

- **40% of expected monthly budget** to canary work — model rollouts, prompt / template changes, retrieval-index refreshes, engine version bumps.
- **40% to feature deploys** — classical SRE allocation, applied to new product capability on the surface.
- **20% reserved for reliability refactor** — scheduler tuning, engine-config optimization, guardrail hardening, SLO instrumentation improvements.

These numbers are *starting points*, not universals. A platform in early-stage rollout mode may run 60 / 20 / 20 (more canary, less feature). A platform under reliability pressure may run 20 / 30 / 50 (less canary, more refactor). The exercise asks you to defend your team's numbers.

### The per-SLI budget allocation is implicit

Some teams try to pre-allocate budget *per SLI* ("quality budget is for canaries, latency budget is for feature deploys"). This is almost always a mistake — the budgets burn in correlated ways and the pre-allocation becomes an accounting fiction. Run the allocation at the *overall budget* level and let the surface-level freeze from section 2 route the spend decision by *which SLI is burning*.

### Documenting the allocation

The policy should name the allocation in writing *before the quarter starts*. During an incident, no one should be arguing over whether a specific rollout is a canary or a feature — that was decided when the allocation was written. The negotiation happens at the quarterly allocation refresh, not live.

## 4. Weighting latency burn vs. quality burn vs. cost burn

When a single SLI is burning, the response is straightforward — the surface-level freeze from section 2 kicks in. When multiple SLIs burn, the policy has to weight them. Four cases, with explicit guidance.

### Case 1 — latency burn only

Infrastructure response. The on-call primary looks at:

1. The engine-side knobs (`mod-102` chapters 04–06).
2. The admission policy and the fleet capacity.
3. The scheduler interference between tenants.

Classical SRE playbook applies. Spend reliability refactor budget on the fix.

### Case 2 — quality burn only

Not infrastructure. The on-call primary routes to:

1. The recent release events — model, prompt, retrieval index — anything that changed within the burn window.
2. The eval peer, with the signal sample and the before/after artifacts.
3. If necessary, a model / prompt / index rollback via the rollback machinery from `mod-112`.

**Key policy rule.** Quality-only burn triggers the eval peer *as a co-investigator*, not as a page. The eval peer owns the signal; you own the SLI. See chapter 07 and section 7 of this chapter.

### Case 3 — latency + quality correlated burn

Dangerous case. A single underlying event is most likely — a provider partial outage (long-context prompts slow *and* degraded), a scheduler regression that affects both latency and the quality of responses that completed under preemption, a retrieval regression that both slowed the service *and* reduced grounding quality.

**Policy rule.** When two or more SLIs are simultaneously under 50% budget remaining *and* the burn is time-correlated (within 2 hours), escalate to the on-call incident commander for root-cause investigation before applying single-SLO responses. Do not treat correlated burn as two independent incidents.

The 2-hour correlation window is a starting point; some teams prefer 1 hour (tighter) or 4 hours (wider). The window should be in the policy document, not left as an implicit judgment call.

### Case 4 — cost burn concurrent with other burn

Two sub-cases:

- **Cost burn + quality burn.** Often a sign of a prompt-regression or model-upgrade that added output verbosity *and* reduced usability. Treat the quality burn as primary; the cost burn is a confirming signal.
- **Cost burn + latency burn.** Often a sign that a fleet-scale response to latency (adding capacity) has been applied without an authorization from the cost SLO's sponsor. Route to the SLO review with both facts; the response may be to accept the cost impact (if the latency fix was urgent) or to roll back the capacity add (if the cost impact is unacceptable).

Cost burn *never pages* on its own, per chapter 04. In the correlated cases above, the paging SLI is the latency / quality one; cost rides along as diagnostic.

## 5. Reconciling with the eval peer's release gate

The `ai-eval-engineer` peer (level 30) owns a release-gate pipeline: before a new model, prompt, or retrieval index can be rolled out, it must pass the peer's eval suite with a signal above threshold. This gate is the peer's deliverable; your SLO is downstream of it.

Two coordination rules the policy should encode.

### Rule 1 — a passing release-gate does not guarantee a non-burning canary

The release gate validates against a *known eval set* at a *point in time*. Production traffic distribution drifts, judge models drift, and a release that passed the gate may still burn the hallucination SLO in production. The on-call response if that happens:

1. Roll back the release (via the mechanism in `mod-112`).
2. File an incident with the eval peer: the release gate signal passed, the production SLI burned, and the artifact set (the traffic slice that burned, the judge calls on it, the pass-rate differential vs. gate) is attached.
3. The eval peer then owns the question of *why the gate did not catch this* — a gap in the eval set, a drift between the gate's judge and the production judge, a workload-mix shift. The answer updates the gate; your SLI is restored by the rollback.

The policy should say explicitly that a release-gate-passing release *can still burn the SLO* and the rollback authority belongs to on-call, not to a re-review of the gate.

### Rule 2 — a non-passing release-gate does not grant a budget-spend

A release that failed the eval peer's gate cannot be shipped by invoking "we have budget." The reverse rule: budget authorizes *risk* on changes that have passed the gate. It does not override the gate itself.

Why this rule: the release gate is a *quality floor* set by the peer; the error budget is a *tolerance envelope* around that floor. Allowing budget to override the gate would mean the SRE side is overruling the peer on their own craft, which breaks the reliability charter from `mod-101` chapter 01.

The exception process from section 7 handles the rare case where a gate-failing release must ship (e.g., a security hotfix). It is explicitly logged and reviewed.

## 6. The on-call primary's three-minute decision ladder

The policy's usefulness is proportional to how fast the on-call primary can resolve an action under it. A reference three-minute decision ladder the policy should produce:

1. **Which SLI is burning?** (The alert content names it.)
2. **What is the burn band?** (≥ 80 / 50–80 / 25–50 / < 25.)
3. **Is any other SLI also burning within the correlation window?** (If yes: IC escalation per section 4 case 3.)
4. **Is this latency / quality / availability / cost?** (Routes the response: infra / eval-peer / IC / review-queue per section 4.)
5. **Are release / canary / refactor actions permitted on the surface at this band?** (Table from section 2.)

Five questions, one action. The ladder is what the policy is *for*; everything else is supporting detail.

## 7. The exception process and its auditability

Every error-budget policy has an escape valve — the case where an exception to the policy is justified. Without one, the policy becomes brittle and is routed around informally, which is worse than not having one at all. The exception process the policy should encode:

- **Who grants it.** Typically the incident commander for in-incident exceptions, the SRE lead for planned exceptions.
- **For how long.** Time-boxed (24 or 48 hours is typical). Auto-expires.
- **What triggers it.** A named scenario — "a provider outage requires a non-gate-passing model fallback to serve traffic," "a security hotfix overrides the normal release gate," "a cost burn requires a capacity add without SLO sign-off."
- **How it is logged.** Explicitly in the incident channel or the SLO review log. The exception's existence is the artifact.
- **How it is reviewed.** At the next SLO review, every exception is read aloud and either absorbed into the policy (if it recurs) or marked as one-off (if it does not).

An exception is not a failure of the policy; it is a *known branch* in it. What makes it work is the auditability — every use is visible at the next review, and if the exception is used repeatedly, the policy gets updated to remove the need.

## 8. A worked example — the three-minute response to an incident

To make the policy concrete, walk one incident end-to-end.

**Incident.** At 02:17 UTC, two alerts fire:

- Hallucination rate SLI: fast-burn — 2% of budget in 1 hour.
- TTFT (cache-miss slice) SLI: fast-burn — 2% of budget in 1 hour.

### The primary's three-minute path

**Minute 1** — read the alert content. Two SLIs. Both within the 2-hour correlation window. Correlated-burn rule from section 4 case 3 applies: escalate to IC before single-SLO response.

**Minute 2** — page the IC. IC pulls up the recent-change log: a model-provider failover occurred 90 minutes ago due to primary-provider 5xx. The graceful-degradation SLI is also burning (confirming the fallback took effect).

**Minute 3** — the correlated signal is now: provider-fallback is serving 60% of traffic from a secondary with slower TTFT and worse hallucination performance. The right action is not to roll back the fallback (then availability tanks) and not to roll back the primary model (it was not the regression). The right action is to invoke the provider-fallback incident runbook from `mod-107`: coordinate with provider status, watch for primary-restoration, and absorb the degraded-serving burn against the explicit graceful-degradation SLO.

**What the policy produced.** A three-minute escalation to IC, no false action against the (unrelated) model-rollout surface, and a routing to the right runbook. No two-dimensional debugging in the dark.

**What a single-SLO policy would have produced.** Two independent responses — one freezing model rollouts (quality SLI burning), one adding fleet capacity (latency SLI burning). Both actions wrong. The fallback path, the real cause, gets attention only after a longer investigation.

## 9. The policy document — section outline

The artifact your team produces using this chapter. Eight sections:

1. **SLI surface** — enumerated from chapters 01–04. Each SLI listed with target, window, and burn-rate alert config.
2. **Freeze-targeting map** — the per-SLI freeze mapping from section 2 of this chapter.
3. **Allocation rule** — canary / feature / refactor split for the quarter.
4. **Burn bands** — the four-band table from section 2, operationalized.
5. **Correlated-burn rule** — the correlation window, the IC escalation path, the handoff to the IC's runbook.
6. **Eval-peer coordination rule** — the two rules from section 5, with the peer's channel and co-investigation process named.
7. **Exception process** — who grants, for how long, how logged.
8. **Review cadence** — monthly SLO review, quarterly allocation refresh, annual policy audit.

Exercise 05 of this module produces the full policy document against the workload picked in exercise 01.

## Summary

- **Three decisions** — release-freeze, budget-spend, burn-weighting. A classical policy handles the first; the AI-shaped policy addresses all three explicitly.
- **Freeze-targeting map** — per burning SLI, freeze the specific change types that could have caused the burn. Not a global freeze.
- **Four-band table** — per-SLI budget-remaining bands drive what canary / feature / refactor actions are permitted on the surface.
- **Correlated-burn rule** — two or more SLIs burning within the correlation window route to IC before single-SLO response. Prevents two-dimensional debugging in the dark.
- **Two eval-peer coordination rules** — a gate-passing release can still burn the SLO (rollback authority on-call-side); a gate-failing release cannot ship by spending budget.
- **Three-minute decision ladder** — the policy's usefulness is proportional to how fast on-call resolves an action under it.
- **Exception process** — time-boxed, logged, reviewed. Not a failure of the policy; a known branch.
- **Eight-section policy document** — the artifact exercise 05 produces.

The next chapter wires the SLO lifecycle from this chapter end-to-end — define, baseline, instrument, alert, review, retire — to the Prometheus / Grafana / OpenTelemetry artifacts the policy is operationalized through.
