# Vendor-Eval Discipline: Single-Source, Multi-Source, or Self-Host

## Why this chapter exists

The previous chapters assumed the vendor set was given. This one goes up one level: **choosing the vendor set** is itself a reliability decision, with the longest half-life of any decision in the stack. A single-source choice is a trade of flexibility for rate-card simplicity, a feature frontier, and cleaner support relationships. A multi-source default is a trade of operational surface area for resilience and negotiating leverage. A self-host choice is a trade of cost and headcount for control. None of these is universally right; all of them are specific to the workload, the organisation's scale, the regulatory posture, and the quality of the engineering team.

This chapter builds the decision framework. It covers:

- When single-source is defensible, and when it is not.
- When multi-source is the right default, and the cost of that default.
- When self-hosting changes the shape of the problem, and where the hand-off to the inference-engineer discipline sits.
- Version-locking a vendor API against breaking changes.
- Modelling exit cost as a first-class metric.
- The vendor-eval scorecard — the document the AI SRE produces to inform the decision.

The chapter is explicitly at the reliability layer. The cost-performance-quality router, the serving-system build decision, and the deeper economics of self-hosting all sit with the ai-infra-inference-engineer peer (see §3.4 and chapter 07).

## 1. Single-source: defensible scenarios

The default advice in many engineering-blog posts is "always multi-vendor." That advice is not wrong, but it is incomplete. Single-source is the right choice in defensible scenarios:

### 1.1 Very small scale

A team of 2 shipping a product with <1000 daily users has no engineering bandwidth for a multi-vendor router; the operational surface of the router is likely to cause more incidents than it prevents. Single-source with a clear migration plan if scale changes is reasonable. Review the choice at every 10x scale increase.

### 1.2 Unique vendor capability on the critical path

If your product depends on a capability only one vendor offers — a specific long-context frontier, a specific multimodal shape, a specific real-time API, a reasoning model with no equivalent elsewhere — multi-vendor becomes impossible for that leg of the chain. The right posture is single-source for the irreplaceable capability + multi-source for everything else. The dependency map (chapter 01) names the irreplaceable rows explicitly.

### 1.3 Deep enterprise integration

Some customers require that a specific vendor is on the contract (Azure for a customer committed to Microsoft's enterprise agreement, Google Cloud for a customer standardised there). The contract decides; your fallback contract must remain within the vendor's product boundary.

### 1.4 Zero-retention / highly sensitive data

A workload whose data must stay within a specific vendor's data-processing agreement, or whose customer requires named-vendor certification (FedRAMP High, IL-5), may be single-source by constraint. Secondary vendors need equivalent certifications; if none exist, single-source is forced.

### 1.5 When single-source is **not** defensible

- When the workload is tier-1 and SLO-committed, and the vendor's SLA ceiling is below your SLO target (chapter 05 §1). You are already failing the composed-ceiling math on day one.
- When the vendor has a multi-day-outage history in the last 24 months (chapter 03 §1) and your SLO cannot absorb a multi-day event in the current budget. You need redundancy.
- When the vendor's deprecation cadence (chapter 03 §7) is faster than your migration cadence.
- When the vendor's rate-card is the only pricing input and you have no negotiating leverage for a reserved tier.

## 2. Multi-source as default: the cost and the benefit

Multi-source — two or more vendors serving the same route, either active-active or primary-with-fallback — is the AI SRE's typical recommendation for tier-1 routes. The benefits are plain; the costs are often underweighted.

### 2.1 The benefits

- **Availability ceiling raised** (chapter 05 §3).
- **Quality diversification.** A quality regression in one vendor (silent model-version update, a bad SFT pass on the provider side) is caught earlier via A/B comparison against the other.
- **Pricing leverage.** The ability to shift a substantial share of traffic to the secondary in weeks, not quarters, materially changes the negotiation. Vendors with no realistic second source charge accordingly.
- **Deprecation insurance.** When one vendor deprecates a model, the other may not, which buys migration time.

### 2.2 The costs

- **Router complexity.** Chapter 02's semantics are not free. Idempotency, streaming resumption, function-calling consistency, context-window mismatch — each is engineering work.
- **Observability doubled.** Each vendor's request-id, error taxonomy, latency shape, and quota headers lives in the trace alongside the others. The observability stack (`mod-104`) must normalise across providers.
- **Feature-frontier lag.** You use the features both vendors support, not the newest capability of either, unless you also accept cross-vendor feature inconsistency.
- **Quality-eval work doubled.** The canary-eval suite (`mod-105` chapter 03) runs against each vendor independently. Divergent results across the two are signal, but they also demand time to interpret.
- **Contract overhead.** Two commercial agreements, two legal reviews, two security reviews, two procurement cycles. Non-trivial for a lean org.

The AI SRE's job is to acknowledge these costs explicitly in the decision document, not to pretend multi-source is free.

### 2.3 How many is too many

More vendors ≠ better. Diminishing returns set in sharply after two, and operational surface grows linearly. A pragmatic shape:

- **Two vendors for the tier-1 chat / completion route.** Primary + active-active or primary + cold-standby, depending on the §4 trade-off from chapter 02.
- **One vendor for second-order dependencies** (embeddings, moderation, image, rerank) unless the workload is specifically sensitive to that second-order failure (e.g., embeddings for a search-critical surface).
- **A qualified third vendor on standby.** Not routed-to, but with a kept-current fallback-contract entry and a quarterly-re-run canary eval. The migration path is short if the primary two both degrade.

## 3. Self-hosting: when the problem shape changes

Self-hosting a model (or a model family) with your own GPU fleet is qualitatively different from choosing between hosted vendors. The AI SRE's role here is to know when the question is worth asking, not to own the build decision end-to-end.

### 3.1 Signals that self-host is worth evaluating

- **Predictable, sustained spend.** If your hosted-vendor bill on a single model crosses the high six-figure range per month and the traffic shape is predictable, the break-even calculation often favours self-hosting the base demand and bursting to hosted. Specific numbers vary; the math is in the inference-engineer discipline.
- **Latency floor.** Hosted APIs add network round-trip and shared-fleet queuing. For some workloads (interactive agents, voice, real-time), the latency floor of a co-located self-host is irreplaceable.
- **Residency / compliance.** When no vendor has a region that satisfies your constraint, self-host is the only option.
- **Model-weight access.** When your workload requires a model with no hosted equivalent (specific fine-tunes, specific open-weight checkpoints, your own continued-pretraining derivatives).
- **IP / data-moat.** When the use of your data inside a hosted vendor's platform is a commercial risk, even with no-training commitments.

### 3.2 Signals that self-host is not the right answer

- **Variable or bursty traffic.** A workload that goes from 0 to 10k rps and back three times a day wastes self-hosted capacity. Hosted is the right primary; reservation + burst to on-demand is the right shape.
- **Fast model-release cadence sensitivity.** If the product's competitive position depends on being on the newest frontier model within weeks of release, a self-host that takes months to qualify a new base model is the wrong fit.
- **Small team.** Self-hosting at scale is a specialised discipline — GPU operations, inference-engine tuning, capacity planning, incident response for hardware faults. A team without dedicated inference engineers should not take this on.

### 3.3 The hybrid shape

The common shape for a mature org is **hybrid**: self-hosted for the base-rate workload where the economics are clear, hosted for burst and for the frontier-model surfaces. The fallback contract (chapter 02) often puts self-hosted as the primary and hosted as the fallback, reversing the usual intuition. The AI SRE's cross-reference to the inference-engineer peer owns this shape.

### 3.4 Where this hands off

The specific build-vs-buy decision, the inference-engine selection (vLLM / SGLang / TensorRT-LLM / TGI / Dynamo), the engine-tuning knobs (max_num_seqs, quantisation, speculative decoding), and the GPU-fleet economics are the ai-infra-inference-engineer peer's domain. The AI SRE receives their output — "this base model on this hardware at this throughput for this cost" — and composes it with the vendor-risk picture.

## 4. Version-locking a vendor API against breaking changes

A vendor-risk posture that survives multiple years is one that locks against two kinds of change: the *wire-protocol change* (API version) and the *model-behaviour change* (model id).

### 4.1 API-version locking

Every hosted provider versions its API in some form:

- **OpenAI.** The Python / Node SDK is pinned via dependency; the wire protocol is stable within a major version. OpenAI publishes deprecations for endpoints; new behaviour is typically opt-in. Pin the SDK major version in dependencies.
- **Anthropic.** `anthropic-version` HTTP header (e.g., `2023-06-01`). Pin explicitly in requests; SDK does this on your behalf but verify.
- **Google (Vertex AI).** SDK versioning; wire protocol with versioned endpoints.
- **AWS Bedrock.** SDK versioning via AWS SDKs; model-id in requests is the primary variable; `InferenceConfiguration` schema is stable within SDK major versions.
- **Azure OpenAI.** `api-version` query parameter (e.g., `2024-10-21`) on every request. Pin explicitly; Azure publishes compatibility matrices for api-version vs. feature availability.

The dependency map's `api_version` field (chapter 01 §1.3) is enforced in production: a canary job runs daily against the pinned version, and a CI check fails the build on an SDK bump that changes the pinned version implicitly.

### 4.2 Model-version locking

Every provider has model-id families (`gpt-4o`, `claude-sonnet-4-6`, `gemini-2.5-pro`) and specific dated snapshots (`gpt-4o-2024-11-20`, `claude-sonnet-4-6-2025-xx-xx`). The choice of pinning strategy:

- **Pin to a specific snapshot.** Deterministic; immune to silent vendor refresh. Requires manual migration on deprecation. Default for tier-1 SLO-committed routes.
- **Track the alias.** Automatically picks up the newest snapshot of the family. Convenient; opens the door to silent quality regression (`mod-105` chapter 03). Reasonable for tier-3 and some tier-2 routes.
- **Hybrid.** Pin production; track alias in a shadow / eval environment that automatically runs the canary suite against new snapshots; promote by hand after eval passes.

### 4.3 Deprecation calendar

The map (chapter 01) records the current announced deprecation date for each row. Vendor deprecation announcements land on the provider's docs; the quarterly audit refreshes the map. Published deprecation pages:

- OpenAI: `platform.openai.com/docs/deprecations`.
- Anthropic: `docs.anthropic.com/en/docs/about-claude/model-deprecations`.
- Google Vertex AI: `cloud.google.com/vertex-ai/generative-ai/docs/learn/model-versions`.
- AWS Bedrock: `docs.aws.amazon.com/bedrock/latest/userguide/model-lifecycle.html`.
- Azure OpenAI: Azure OpenAI service documentation — model lifecycle appears per-model.

Each deprecation ticket on the backlog has: vendor, model, deprecation date, successor candidate, canary-eval status, migration owner. The ticket is scheduled such that migration completes ≥ 30 days before the deprecation date, to leave slack for discovered quality regressions.

### 4.4 Breaking-change canaries

A canary that runs a frozen test suite against the current pinned (version, model) daily is the sentinel. If the vendor silently changes behaviour under the pinned snapshot — rare but documented — the canary catches it within a day. `mod-105` chapter 03 covers the eval shape.

## 5. Exit cost as a first-class metric

Vendor-risk engineering is incomplete without an estimate of the cost to **exit** each vendor. Lock-in is a reliability risk: it bounds how quickly you can respond to a vendor's price change, deprecation, or sustained degradation.

### 5.1 The axes of exit cost

- **Engineering effort.** Person-weeks to migrate every route off vendor X. Dominated by: unique features (function-calling schema, structured outputs, vision shape, thinking-block handling) and the number of call sites. The abstraction layer (chapter 02 §5) is what compresses this; absent it, exit cost grows with surface area.
- **Quality-regression risk.** Probability that the move degrades quality enough that users notice. Mitigated by the canary-eval suite having the receiver vendor in it already.
- **Prompt rewriting.** Prompts that encode the primary vendor's quirks (reasoning-trace conventions, caching-break hints, specific refusal patterns) must be reworked. Few organisations track this as a line item.
- **Fine-tuned / distilled artifacts.** If you have a fine-tune on vendor X, exiting means retraining elsewhere (or losing the custom-model moat). Non-recoverable if the base model is not published.
- **Contract exit.** Reservation commitments have term lengths. An exit mid-term may forfeit the committed-spend prepayment.
- **Data residency.** Data at rest in the vendor's embedding store or vector DB may need to be re-embedded on a different model (mod-112 covers the pipeline shape).

### 5.2 Expressing exit cost

A one-row summary per vendor:

```yaml
vendor: openai
exit_cost:
  eng_weeks: 6
  prompt_rewrite_lines: ~400
  fine_tunes_affected: 2  # gpt-4o-ft:acme:summariser:ABC; gpt-4o-mini-ft:acme:classifier:DEF
  reservation_forfeit_usd: 120_000  # if exit mid-term
  quality_regression_risk: medium  # we have a canary-eval against anthropic+bedrock
  data_re_embedding_required: false
  contract_notice_days: 30
  total_exit_lead_time_days: 60
```

The exit-cost row is reviewed annually, and whenever a major change makes it materially different (new fine-tune shipped, new reservation signed, new unique-feature dependency introduced).

### 5.3 Why bother

Two concrete uses:

- **Negotiation leverage.** The AI SRE's view of exit cost is the lower bound on the vendor's pricing power. If exit cost is $200k and vendor Y offers a $50k/year savings, the move takes four years to pay back; the vendor can charge accordingly. Awareness of the number is leverage to lower it.
- **Incident response.** When a vendor has a multi-day outage (chapter 03) and the SLO is bleeding, the question "can we shift to the fallback more permanently" is answered against the exit-cost number. If exit cost is 6 weeks and the outage is already 3 days in, permanent shift is feasible. If exit cost is 6 months, permanent shift is not on the table and the response is contractual (service credits, escalation).

## 6. The vendor-eval scorecard

The artifact that captures the chapter's decisions is a per-vendor scorecard, re-run annually and whenever the vendor landscape changes. Every tier-1 route has at least two scorecards: one for the primary and one for the committed secondary.

### 6.1 The scorecard shape

```markdown
# Vendor eval — anthropic — for route summariser-chat

## Capability
- Model ids qualified: claude-sonnet-4-6, claude-opus-4-8.
- Context window: 200K / 200K.
- Function-calling: native; schema matches our abstraction with minor translation.
- Structured outputs: supported via tool-use; verified in canary.
- Vision input: supported; not used on this route.
- Reasoning / thinking blocks: not currently used on this route.

## Reliability
- SLA (as of 2026-11-04): <vendor's current language; see docs>.
- Historical availability (last 12 months, observed from our traffic): 99.72%.
- Multi-day outage events in last 24 months: 1 (2024 Sept, ~4h; 2026 Jan, ~12h).  <!-- needs-research: specifics from status.anthropic.com -->
- Deprecation cadence: prior major family retired 10 months after successor GA.

## Quality
- Canary eval score vs. workload prompts: tier A.
- Divergence from OpenAI primary on golden set: 4.2% disagreement rate (2026-10 run).

## Cost
- Input/Output: $X/$Y per MTok (as-of 2026-11-04).
- Reservation product: Priority Tier; current commitment $K/mo.
- Blended per-request cost vs. primary: 0.90x.

## Residency / compliance
- US data zone: yes.
- EU data zone: yes.
- Zero-retention: yes under enterprise contract (agreement #XYZ).
- HIPAA BAA: yes.
- SOC 2 / ISO 27001: yes.

## Exit cost
- Engineering effort to exit: 4 weeks.
- Fine-tunes affected: 0.
- Reservation forfeit if mid-term: negligible (month-to-month).

## Decision
- Fit for use on route summariser-chat as primary: yes.
- Fit for use as active-active fallback: yes, in place today.
- Review date: 2027-11-04.
- Owner: @ai-sre-lead
```

The scorecard is the input to the service team's primary-choice decision and the AI SRE's SLO-math exercise (chapter 05).

### 6.2 What the scorecard excludes

- **Deep inference-engine economics.** The scorecard reports hosted costs; self-hosted economics are the inference-engineer peer's.
- **Commercial-contract terms.** Service-credit structures, remedies, and MSA amendments live in legal / procurement documents; the scorecard summarises the SLA language but does not restate the contract.
- **Security review.** Vendor security review is a separate artifact by the security function; the scorecard references its conclusion, not its content.

## 7. The decision cadence

- **Annually.** Every scorecard refreshed. Decision confirmed or changed.
- **Quarterly.** Deprecation calendar reviewed. Canary-eval divergence re-checked. Residency landscape reviewed (new regions, new certifications).
- **On major vendor event.** Multi-day outage, significant deprecation, major model release — trigger a mini-review of the affected scorecards within two weeks.
- **On major internal event.** New regulatory requirement, new enterprise customer with a residency clause, new product line — trigger a scorecard refresh for the affected routes.

## 8. The AI SRE's vendor-eval worksheet

1. **For each tier-1 route**, list the current primary and committed secondary vendor.
2. **For each listed vendor**, maintain a scorecard with the eight sections in §6.1.
3. **Compute the exit cost** per vendor per route; keep it current.
4. **Pin API and model versions** (§4). Enforce in CI.
5. **Track deprecation calendars** in the dependency map (chapter 01); schedule migrations ≥ 30 days ahead.
6. **Run the quarterly re-eval**: refresh canary scores, re-check residency / certifications, confirm contract-term currency.
7. **Document the primary-choice decision** and its date, so a reviewer a year on knows what the context was.

## Summary

- **Single-source is defensible in named scenarios**, not as a default. Small scale, unique capability, enterprise-mandated integration, or compliance-forced singletons.
- **Multi-source is the AI SRE's default for tier-1 routes**, but it is not free. Operational surface, observability doubled, canary-eval doubled, two contracts — all explicit costs.
- **Self-hosting is a different shape of problem** that hands off to the inference-engineer peer. The AI SRE's role is to recognise when the question is worth asking.
- **Version-lock both the API and the model**. Pin in CI; audit on deprecation.
- **Exit cost is a first-class metric.** Measure it, keep it current, and use it as both negotiation leverage and incident-response input.
- **The scorecard is the artifact.** Per-vendor, per-route, annually refreshed, with sections for capability, reliability, quality, cost, residency, and exit.

Chapter 07 draws the peer-boundary lines explicitly: what belongs to the ai-infra-inference-engineer, what belongs to legal / vendor-management, and what the AI SRE must *not* over-reach into even when adjacent to the problem.
