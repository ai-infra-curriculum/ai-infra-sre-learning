# Reconciling SLA with SLO: Error-Budget Math Across Upstreams

## Why this chapter exists

A downstream SLO is a promise about the end-to-end reliability of a user-visible surface. An upstream SLA is a promise about one leg of the chain that produces that surface. The two numbers do not combine by wishful thinking. If your chat product commits to a 99.9% availability SLO but calls one model provider with a 99.5% published SLA on a single route, the composed system **cannot deliver 99.9%** without redundancy. The SLO is already underfunded on the day it is signed, and the first vendor degradation will exhaust the error budget on its own.

This chapter does the math. It covers:

- The series-composition rule for independent dependencies.
- Why real dependencies are rarely independent, and the correction that implies.
- Reserved-budget discipline for provider degradation.
- When to add a redundant upstream to break the composition ceiling.
- The "hard no-op" cases from chapter 02 §4.5, now re-expressed as explicit error-budget line items.
- How to express the whole picture in a single budget table that an on-call can read during an incident.

The chapter assumes the SLO vocabulary from `mod-103`. It extends that vocabulary with the upstream-SLA accounting that model-backed services add on top of a classical SRE SLO.

## 1. The composed-SLA ceiling

The classical SRE observation: a service that depends on `N` independent upstreams in series, each with availability `a_i`, has an upper bound on its own availability of `∏ a_i`. (See "Site Reliability Engineering," Chapter 4, and the dependency-SLA math in the SRE Workbook.) For `a_i = 0.995` and `N = 1`, the ceiling is 0.995. For `N = 2`, the ceiling is 0.990. For `N = 5`, the ceiling is 0.975.

In model-backed services the series chain is often longer than it looks. A single user turn may hit, in order:

1. Your own ingress / auth / gateway.
2. The vector database (hosted or self-run).
3. A hosted embedding provider.
4. A hosted model provider (chat / completion).
5. Optionally, a second provider for a judge pass.
6. A moderation or safety API.
7. A tool-call target API.
8. Your own storage and response-formatting.

Not every turn hits every step, but the SLO-committed path does. The AI SRE's first duty is to enumerate the chain honestly; chapter 01's dependency map is the raw input.

### 1.1 The SLA units mismatch

Published SLAs rarely compare one-to-one with your SLO. Three alignment hazards:

- **Monthly uptime vs. rolling window.** AWS, Azure, and Google publish monthly uptime percentages. Many AI SRE SLO frameworks (`mod-103`) measure in a 28-day rolling window. The two are close but not identical; the error-budget math uses one or the other, not both.
- **Success-rate vs. latency.** Many provider SLAs are purely availability-based. Latency commitments appear in product-specific SLAs (Azure OpenAI's PTU has latency commitments; OpenAI's Scale Tier has latency commitments). A latency-sensitive SLO cannot be composed against an availability-only SLA.
- **Per-endpoint vs. per-service.** Some SLAs commit per API endpoint; some commit per service. A provider green on "overall API" with one endpoint at 90% does not violate its SLA but does violate your endpoint-specific SLO.

The dependency map (chapter 01) records the SLA language verbatim so the composed-math step does not silently normalise them away.

### 1.2 Doing the series composition

For a series chain of independent upstreams, the maximum attainable downstream availability is `∏ a_i`. For a chain of (your own 99.95%) → (hosted embedding provider 99.9%) → (hosted model provider 99.5%), the ceiling is `0.9995 × 0.999 × 0.995 ≈ 0.9935` — 99.35%. A downstream SLO of 99.9% on this chain is not possible without redundancy.

The composed ceiling is *before* consuming any error budget to your own bugs, your own change failures, or anything else the SRE function spends budget on. Realistically, your own operational failures take another 0.1–0.3% of availability. Which means on this chain, a feasible downstream SLO is ~99.0–99.2%, not 99.9%.

### 1.3 The air-gap check

Before committing to an SLO, do the composition explicitly and check the air gap:

- **Composed ceiling** = `∏ upstream_a_i`.
- **Target SLO** = the number you're committing to.
- **Air gap** = ceiling − SLO.

If air gap < 0.3%, the SLO is at the edge; any upstream degradation in the quarter exhausts the budget. If air gap < 0.0%, the SLO is infeasible on paper. The remedy is either lower the SLO, add redundancy to raise the ceiling (section 3), or renegotiate an upstream SLA (section 6).

## 2. Independence is a lie — the correlation correction

The series-product rule assumes upstream failures are independent. They are not.

### 2.1 Why upstreams correlate

- **Shared cloud.** Anthropic runs on AWS. Together, Fireworks, and others run on the same handful of cloud providers for their own GPU capacity. A sufficiently broad AWS incident takes multiple of your "diversified" upstreams at once.
- **Shared infrastructure component.** Multiple providers use Cloudflare, Fastly, Akamai, or a shared CDN. A Cloudflare incident in July 2019, November 2023, and others produced simultaneous reachability issues across many "independent" SaaS vendors.
- **Shared auth / certificate / DNS.** A DNS provider incident or a widely-used certificate authority event correlates every upstream that depends on it.
- **Shared model.** A fallback contract that routes between Anthropic direct and Anthropic-on-Bedrock is one model, two endpoints. The model itself is the single point of failure — a quality regression in the model propagates to both endpoints.

The AI SRE treats the independence assumption as **optimistic by default**. The series-product ceiling is an upper bound; reality is somewhat worse.

### 2.2 The correlation correction

A pragmatic correction: for every pair of upstreams in your series chain, estimate `P(both fail | one fails)`. If it is non-negligible (>5%), add a "common-mode" availability deduction:

- Hard common mode (same cloud region, same CDN, same model). Treat the two upstreams' availabilities as **the min, not the product**: `min(a_1, a_2)` instead of `a_1 × a_2`. This is pessimistic; the truth is between.
- Soft common mode (different clouds, similar infrastructure families). Add a 0.1–0.3% common-mode failure rate to the composed availability.

The exact arithmetic is less important than the discipline: the planner should name the common-mode pairs rather than pretending they are independent.

### 2.3 Example

Primary: `anthropic` direct, 99.5% SLA (illustrative). Fallback: `anthropic-on-bedrock`, 99.9% SLA. Naively, the composed system has `1 − (1 − 0.995)(1 − 0.999) = 99.9995%` availability — astonishing. Realistically, because the model artifact is shared, a quality regression or model-specific capacity issue in Anthropic's infrastructure affects both; the right composed availability is closer to `max(0.995, 0.999) = 0.999` for model-level failures and `0.9995%` for pure delivery-infrastructure failures. The dependency map (chapter 01) records the common-mode clearly so this does not get hidden.

## 3. Adding redundancy to raise the ceiling

Redundancy is the only way to break the series-product ceiling. Three redundancy patterns, each with a math change.

### 3.1 Active-active across two independent providers

Two providers, each receiving a share of traffic; the router falls over on breaker-open. If the failure modes are independent and the fallback is automatic:

```
composed_availability ≈ 1 − (1 − a_1) × (1 − a_2)
```

Two 99.5% providers compose to `1 − 0.005 × 0.005 = 99.9975%` in the fully-independent case, before common-mode. With a 20% common-mode (shared cloud provider), the practical availability is closer to `1 − 0.005 × 0.2 = 99.9%`. Still better than either alone, but not the paper number.

### 3.2 Cold-standby with imperfect switchover

The secondary is wired but unused. Switchover takes some time `t`, during which the system is degraded. If the switchover succeeds reliably:

```
composed_availability ≈ a_primary + (1 − a_primary) × (1 − P(switchover_fails)) × (1 − t/MTBF)
```

The dominant penalty is `P(switchover_fails)`. In practice, this is higher than most teams estimate because the switchover path is rarely tested (chapter 02 §7 and `mod-109` game-day are the counter-measures). A `P(switchover_fails)` of 10% is defensible for a rarely-tested path; 1% requires drill discipline.

### 3.3 Quality-tiered degradation

A tier-A primary falling to a tier-B alternate increases availability but *reduces quality*. The SLO column that matters depends on which quality threshold the service team commits to. If the SLO is "serve the request with any quality," composition raises the number. If the SLO is "serve at tier A," the tier-B interval is still a budget-consuming degraded period. Chapter 02 §4.3 pins this trade-off per route; chapter 05's math respects it.

## 4. Reserved-budget discipline for provider degradation

A classical SRE error-budget policy allocates the monthly / quarterly budget across its consumers: your own change failures, your bugs, your dependency failures, planned maintenance, unknown-unknowns. The AI SRE's modification is to **make provider-degradation an explicit named line item** in the budget allocation, with a reserved share.

### 4.1 Why reserve explicitly

Two symmetric failure modes without the reserve:

- **Spending it all on your own failures.** Your team ships with less rigour, consumes the whole budget, and then the first provider degradation pushes the composed availability past the SLO. Users are harmed; your team argued the budget was theirs.
- **Spending none of it on your own failures.** Your team becomes risk-averse because "we might need the budget for a provider incident." Release velocity suffers; the budget is unused on the day the quarter closes.

Neither is the intent of an error budget. The reserve forces the conversation.

### 4.2 Sizing the reserve

A defensible starting point: **the reserve for provider degradation is the sum of the unmet air-gaps** (section 1.3) across all upstreams, plus a small common-mode allowance, plus the expected downtime from last year's incidents in the same vendor set.

For a 99.9% monthly SLO with a composed ceiling of 99.5%:

- Air gap = 99.5% − 99.9% = −0.4%. The budget is already infeasible without redundancy.

After adding a redundant upstream, suppose the composed ceiling rises to 99.95%:

- Air gap = 99.95% − 99.9% = 0.05%, or ~22 minutes per month.
- Historical provider-caused downtime at your vendor set: say 60 min/quarter = 20 min/month.
- Reserve for provider degradation: 20 min + 10 min buffer = 30 min/month of the 44-minute budget. ~70% of the budget is reserved for upstreams.

That is a sobering share. It is also realistic. If the share feels wrong, the response is not to reduce the reserve — it is to either lower the SLO or further diversify upstream (section 3).

### 4.3 Error-budget alert policy

A budget policy that includes the reserve has four alerting levels:

- **Burn rate normal.** All consumers under their allocation; no action.
- **Operational burn above target** (your own failures). Freeze risky changes; standard error-budget policy applies (`mod-103`).
- **Provider burn above reserve.** The provider's own incidents are consuming more than their reserved share; the response is not to freeze your changes but to escalate **vendor-side**: open a ticket with the account manager, trigger the fallback-contract review, potentially shift the primary.
- **Combined burn > 100%.** All-hands incident. The SLO is already violated; the service team's release cadence is the first lever; the second lever is the fallback contract.

Note the asymmetry in the "provider burn above reserve" response: the on-call cannot fix the upstream. The response is to spend the budget deliberately — swap to the fallback early, open the vendor dialogue early, and (in extreme cases) accept the violation and communicate to users.

## 5. Hard no-op cases as explicit line items

Chapter 02 §4.5 named cases where the right response to an upstream failure is to fail the request rather than silently fall over — residency-constrained traffic, high-stakes decisions, tier-B degraded output being more harmful than no output. These are not "degradation"; they are deliberate error-budget expenditure to preserve a different property (compliance, correctness).

The error-budget table records them as a separate line:

- **Residency-forced hard fail.** Budget: N minutes/quarter. Policy: fail with 503 on residency-only upstream degraded; do not switch regions. Trigger: dependency map's residency column = `strict`. Owner: service + legal.
- **Quality-floor hard fail.** Budget: N minutes/quarter. Policy: return error / degraded UI if all tier-A alternates are unavailable. Trigger: fallback-contract tier == A only. Owner: service.

Explicit named line items make the trade-off visible. An auditor reads them and understands the budget allocation; an on-call reads them and knows not to switch.

## 6. Renegotiating the SLA

Sometimes the composed ceiling is too low and adding redundancy is impractical. The remaining lever is to raise an upstream's SLA.

### 6.1 Available instruments

- **Reservation products.** OpenAI Scale Tier, Anthropic Priority Tier, Google Vertex AI Provisioned Throughput, AWS Bedrock Provisioned Throughput, Azure PTU — each carries a different SLA than the on-demand tier. Typically higher availability and lower latency commitments in exchange for committed spend. Include the specific product's SLA in the map.
- **Enterprise agreements.** Named-account agreements with the vendor that commit to a specific availability, latency, support response, and (occasionally) a notification lead time on deprecations and model changes. This is a legal/procurement activity, not an engineering activity.
- **Dedicated capacity / single-tenant.** For some providers (Google's Vertex AI Dedicated, Azure Confidential Compute, Bedrock Custom Model Units), traffic is served on isolated infrastructure. SLAs are different.

### 6.2 Boundary note

The actual contractual-SLA conversation — term sheet, remedies, service credits, cap-on-liability, force-majeure carve-outs — is **out of scope** for AI SRE and lives with legal, procurement, and vendor management. The AI SRE's output to that conversation is the SLA number required to meet the downstream SLO, with the composition math showing why.

## 7. The error-budget table

The artifact that encodes this chapter is one table per SLO, visible to the service team and on-call:

```markdown
# SLO: summariser-chat-availability
- Target: 99.9% monthly, 28-day rolling window.
- Composed ceiling: 99.95% (primary + active-active alt-1 + alt-2 degraded fallback).
- Budget per month (if 99.9%): ~44 min.

## Budget allocation

| Consumer                     | Allocation  | Policy on breach                     |
|------------------------------|-------------|--------------------------------------|
| Own operational failures     | 15 min/mo   | Standard error-budget freeze.        |
| Provider degradation reserve | 25 min/mo   | Escalate vendor-side; flip primary.  |
| Residency-forced hard fail   | 2  min/mo   | Allow 503; do not cross-region.      |
| Quality-floor hard fail      | 2  min/mo   | Allow 503; do not fall to tier-B.    |
| Unknown-unknowns             | Remaining   | Post-incident review required.       |

## Alerting
- Operational burn > 50% of allocation → page service team.
- Provider burn > 50% of reserve → page AI SRE + vendor-manager.
- Any hard-fail burn > 50% → page service + compliance owner.
- Total burn > 100% → all-hands incident.
```

The table lives in the SLO repo (same repo as the SLIs, if you have one) and is reviewed monthly.

## 8. The AI SRE's SLA-SLO worksheet

1. **Enumerate the chain** from the dependency map (chapter 01). Include your own hops.
2. **Compute the composed ceiling** `∏ a_i` under independence.
3. **Mark the common-mode pairs.** Shared cloud, shared CDN, shared model. Recompute the ceiling under the pessimistic `min` or an added common-mode deduction.
4. **Compute the air gap** against the target SLO. Negative or near-zero means the SLO is infeasible without redundancy.
5. **Decide redundancy.** Active-active, cold standby, or quality-tiered. Recompute the ceiling.
6. **Allocate the budget.** Named line items for provider degradation and for hard-fail cases.
7. **Set the alerting thresholds.** Separate alerts for own burn, provider burn, and hard-fail burn.
8. **Review monthly.** Compare actual burn to allocation; adjust allocation if the historical shape is wrong.

## Summary

- **The series-product rule bounds you.** A downstream SLO cannot exceed `∏ upstream_a_i` without redundancy. The AI SRE does the math honestly, before signing the SLO.
- **Independence is optimistic.** Shared cloud, shared CDN, shared model — all common modes. Apply a correction; don't pretend the product is the truth.
- **Redundancy raises the ceiling.** Active-active with independent-enough providers is the standard lever. Switchover reliability is the primary unknown; game-day drills are how you pin it down.
- **Reserve the budget for providers explicitly.** A named line item for provider degradation forces the conversation. Size from the composed-ceiling air gap plus historical provider incidents.
- **Hard no-ops are explicit budget line items.** Residency-forced and quality-floor hard fails consume budget deliberately to preserve a different property. Name them in the table.
- **Renegotiating the SLA is a lever.** Reservation products, enterprise agreements, dedicated capacity — the AI SRE feeds the required number into legal/vendor-management, who owns the negotiation.

Chapter 06 moves from "how to survive a known vendor" to "how to decide which vendor, how many vendors, and when to self-host."
