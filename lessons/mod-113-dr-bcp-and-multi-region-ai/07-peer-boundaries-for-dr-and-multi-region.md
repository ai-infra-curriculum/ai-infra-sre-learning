# Peer Boundaries for DR, BCP, and Multi-Region AI

## Why this chapter exists

The chapters up to now have written the DR / BCP practice as if the function owned the whole surface. In the ladder context of `mod-101`, it does not. Multi-region architecture at team-of-teams scope is at the `ai-infra-architect` (level 40). Multi-year DR strategy — the three-year roadmap that picks the regions, chooses the provider portfolio, and sizes the investment against portfolio risk — is at the `ai-infra-senior-architect` (level 45). Regulator-facing evidence — the part of the regulator-facing scaffold (chapter 06 §7) that is *shown to the regulator*, with the testimony and attestations that carry legal weight — is at legal and compliance. The function's work from chapters 01–06 produces the inputs for all three; it does not substitute for any of them.

This chapter writes the boundaries. For each peer, the function names what it hands over, what the peer hands back, what the shared surfaces are, and what the common anti-patterns look like. The result is the routing table the function uses during a DR event, during a planning cycle, and during an audit — the same routing-table pattern as `mod-101` chapter 02 and `mod-108` chapter 06, specialised for DR / BCP / multi-region work.

Primary references: `mod-101` chapters 01 and 02 (the role scope and peer deferral map), `mod-110` chapter 06 (the legal / compliance handoff for AI incidents), `mod-108` chapter 06 (the incident-side legal handoff this chapter composes with), `mod-107` chapter 07 (the vendor-side peer boundary this chapter does not duplicate).

## 1. The three boundaries this chapter writes

- **ai-infra-architect (level 40).** Multi-region architecture at team-of-teams scope. The function's per-service ISCP (chapter 06 §3) composes a *fleet*; the architect composes *the fleets*.
- **ai-infra-senior-architect (level 45).** Multi-year DR strategy. The function's annual review (chapter 06 §10) is a yearly artifact; the senior architect's roadmap is a multi-year commitment with capital implications.
- **Legal / compliance (ai-risk-engineer, DPO, general counsel).** Regulator-facing DR evidence. The function's regulator-facing scaffold (chapter 06 §7) is honest and current; the regulator-facing *testimony* is at the legal / compliance peers.

Each boundary gets §2–§4 below. The chapter closes with a four-question routing rule (§5) the function applies in real time.

## 2. The ai-infra-architect boundary

### 2.1 What the architect owns

The `ai-infra-architect` peer owns multi-region architecture at the scope where no single service's ISCP is sufficient. The subjects at this scope:

- **Regional portfolio.** Which regions the organisation operates in, which combinations provide residency-respecting resilience, how the portfolio evolves when a region is added or deprecated.
- **Shared-platform multi-region design.** The multi-region design of the shared platform components (identity, authentication, routing layer, release pipeline, observability roll-up) that every service's ISCP depends on.
- **Cross-service dependency DR.** When service A's ISCP targets 15-minute RTO and service A depends on service B whose ISCP targets 60-minute RTO, the composition is at the architect. The function owns each ISCP; the composition is at the architect.
- **Capacity reservation across regions.** Team-of-teams-level reserved-capacity contracts with cloud providers for DR scenarios.
- **The team-of-teams DR cadence.** Multi-service game-days (the chapter 05 §6 game-day shape, expanded to multiple services simultaneously) where the composition is what is being tested.

### 2.2 What the function hands up

- The per-service ISCP (chapter 06 §3), as the input to any multi-service composition.
- The RPO / RTO table (chapter 06 §4) — the architect reads the table to find composition conflicts.
- The drill scores (chapter 05 §6.4) — the architect uses the trend to resource the architecture programme.
- The risk-register entries (chapter 06 §9) for gaps that the function cannot close within its own scope.

### 2.3 What the architect hands back

- The multi-region reference architecture — the function builds each service's ISCP *inside* the reference architecture.
- The cross-service dependency graph with DR annotations — the function uses it to confirm the service's own ISCP targets are compatible with its upstream and downstream dependencies.
- The regional portfolio with the approved-region set — the function's residency posture (chapter 04) is scoped to this set.
- The team-of-teams game-day schedule — the function's annual full-scale exercise (chapter 05 §1.3) composes with it.

### 2.4 The shared surfaces

- **The RPO / RTO table.** Per-service values are at the function; the table itself is often rolled up at the architect level for leadership reporting.
- **The regional portfolio.** The function reads the portfolio; the architect sets it.
- **The reference architecture repository.** Function PRs against service-specific manifests; architect PRs against reference manifests.

### 2.5 The anti-patterns

- **The function makes a unilateral regional decision.** A team that moves to a new region without the architect's regional-portfolio approval creates an operational island that the shared platform does not integrate with. The function escalates; it does not decide.
- **The architect writes individual ISCPs.** Each service's ISCP is at the service's function. An architect writing an ISCP for a service is doing the function's job and is doing it without the function's operational knowledge.
- **The function tolerates a cross-service composition gap.** A composition gap (service A's RTO cannot be met because service B's RTO is longer) is escalated, not silently absorbed. The function surfaces the composition; the architect resolves it.

## 3. The ai-infra-senior-architect boundary

### 3.1 What the senior architect owns

The `ai-infra-senior-architect` operates at the horizon where the current architecture itself is what is being redesigned. The subjects:

- **Multi-year DR strategy.** The three-to-five-year plan for how the organisation's DR posture evolves — new regions, new providers, new residency boundaries as the business opens new markets, new regulatory regimes.
- **Provider-portfolio strategy.** The choice of primary and secondary model providers, the exit-cost analysis per provider, the diversification policy.
- **Regulatory-posture strategy.** As new regulations land (EU AI Act, sector-specific AI regulation, data-protection regimes), how the organisation's DR posture evolves to meet them.
- **Capital allocation.** The investment case for the next generation of the DR programme — new infrastructure, new tooling, new regions — that needs executive funding.

### 3.2 What the function hands up

- The annual review record (chapter 06 §10) — the primary artifact the senior architect reads to understand the programme's current state.
- The multi-year drill-score trend — the senior architect sees the direction and uses it to resource.
- The risk-register resurfacing (chapter 06 §9.3) — unaccepted but intolerable gaps rise to the senior architect.
- The programme's maturity-ladder level per service (chapter 05 §8) — the senior architect's roadmap targets a maturity level per service.

### 3.3 What the senior architect hands back

- The multi-year strategy document — the function's annual planning builds inside it.
- The provider-portfolio decisions — the function's `mod-107` chapter 02 fallback contracts are scoped to the portfolio.
- The regulatory-posture decisions — the function's chapter 04 residency pattern per stamp is scoped to them.
- The capital allocation decisions — the function builds the current year's programme to the allocated budget.

### 3.4 The shared surfaces

- **The risk register.** Day-to-day entries are at the function; the long-tail entries that persist across annual reviews land at the senior architect's horizon.
- **The maturity ladder.** The function reports its level; the senior architect targets its level.

### 3.5 The anti-patterns

- **The function tries to write the multi-year strategy.** A DR programme at the function's level is annualised; longer horizons are the senior architect's. The function's inputs to the strategy are invaluable; the strategy itself is above the function's pay grade.
- **The senior architect writes annual plans.** Annual plans are at the function; the senior architect approves the frame and funds the budget.

## 4. The legal / compliance boundary

### 4.1 What legal / compliance owns

Three roles converge here: the `ai-risk-engineer` peer (as scoped in the `mod-101` ladder), the data protection officer (DPO), and the general counsel / legal peer.

- **Regulator-facing testimony.** Any statement made to a regulator about the DR programme's compliance with a specific regulation is at legal, not the function. The function's scaffold (chapter 06 §7) is the input; the legal filing is at legal.
- **Standard Contractual Clauses (SCCs) and equivalents.** The legal basis for cross-residency burst (chapter 04 §3.2) is drafted, negotiated, and signed by legal; the function references the signed instrument.
- **Breach-notification determinations.** Whether a DR event rises to a GDPR Article 33 breach is a legal determination. The function provides the technical facts; legal makes the call.
- **Audit-firm relationships.** The SOC 2 audit, the ISO 27001 audit, sector-specific audits, external penetration tests — the firms and the relationships are at legal and risk.
- **Contractual DR commitments.** The RPO / RTO commitments made to specific customers under contract are owned by the legal / sales interface; the function builds the ISCP to those commitments, not the other way around.

### 4.2 What the function hands up

- The regulator-facing scaffold (chapter 06 §7).
- The ISCPs, the drill log, the risk register — the full artifact pack from chapter 06 — on request.
- The technical facts of any event that requires a legal determination.
- The DPO escalation input during a cross-residency DR event (chapter 04 §3.3).

### 4.3 What legal / compliance hands back

- The SCCs, DPAs, and contractual commitments that constrain the ISCPs.
- The regulatory thresholds that constrain the notification cadences in chapter 06 §6.
- The audit-firm's findings against the artifact pack — findings become drill scenarios or ISCP updates.
- The determination on a specific DR event's regulatory classification — the function acts on it.

### 4.4 The shared surfaces

- **The communication templates (chapter 06 §6).** The function drafts; legal approves; the function sends.
- **The user-notice trigger condition (chapter 04 §3.4).** Jointly owned; neither alone can set it.
- **The regulator-notice (chapter 06 §6.3).** Function drafts the technical content, legal shapes the regulatory framing, legal files.

### 4.5 The anti-patterns

- **The function makes a regulatory determination.** "This probably doesn't need a breach notice" is a sentence the function may think and never say. The decision is at legal.
- **Legal / compliance tries to make a technical DR decision.** Whether to invoke the cross-residency burst is partly a compliance decision (chapter 04 §3.3); the decision not to cross is often also a reliability decision. The two inputs come from different owners and are integrated at the incident commander (`mod-108` chapter 02), not by one side overriding the other.
- **Either side leaves the other out of the room on a residency-sensitive drill.** Chapter 05 §3.4 has legal as a standing observer. A drill run without legal is a drill that does not exercise the escalation pattern the real event will need.

## 5. The four-question routing rule

When a DR / BCP / multi-region question arrives, the function applies four questions to route it:

1. **Is the question about mechanics within one service's ISCP?** → Function owns. Chapters 02–06 are the material.
2. **Is the question about composition across services, or about shared-platform multi-region design?** → `ai-infra-architect`. Function escalates with the ISCP and RPO / RTO table in hand.
3. **Is the question about the multi-year horizon, provider-portfolio, or capital allocation?** → `ai-infra-senior-architect`. Function escalates with the annual review record and the maturity-ladder level.
4. **Is the question about a regulator, a contract, or a legal determination?** → Legal / compliance. Function escalates with the artifact pack and the specific technical facts.

A question that touches more than one lane is routed to all applicable owners, with the function as the initial router. The function does not sit on an ambiguous routing; it asks explicitly.

## 6. The function's irreducible DR / BCP surface

Having named everything that is not the function's, the chapter closes by naming what is:

- **The per-service ISCP** at command-level specificity (chapter 06 §3.2).
- **The per-service state-reconstitution plan** (chapter 03) and the three-way tool-call recovery for every tool with side effects (chapter 03 §4.2).
- **The per-service residency posture** at the implementation layer (chapter 04 §6) — the residency tag on every request, the routing-layer enforcement, the egress controls, the IaC admission policy.
- **The per-service drill cadence** (chapter 05) and the quality of each drill (chapter 05 §6.4).
- **The artifact pack** (chapter 06) — complete, truthful, current.
- **The on-the-ground DR event execution** — the function's on-call is the IC; the function's playbook is what gets invoked.

Nobody else owns these. The architect writes the frame; the senior architect writes the horizon; legal writes the testimony. The function writes — and runs — the mechanism.

## Summary

- Three peer boundaries sit above the function's DR work: `ai-infra-architect` for multi-region architecture at team-of-teams scope, `ai-infra-senior-architect` for multi-year strategy, legal / compliance for regulator-facing testimony and contractual commitments.
- The function hands up per-service ISCPs, the RPO / RTO table, drill scores, and the risk register. Each peer hands back the frame the function builds against: reference architecture, multi-year strategy, contractual constraints, regulatory determinations.
- The shared surfaces (the RPO / RTO table, the communication templates, the user-notice trigger, the regulator-facing scaffold) are jointly owned; neither side acts without the other.
- A four-question routing rule (§5) is applied in real time: within-service → function, cross-service → architect, multi-year → senior architect, regulator / contract → legal.
- The function's irreducible surface is the per-service ISCP, the state-reconstitution plan, the residency implementation, the drill cadence, the artifact pack, and the on-call DR event execution. Everything else composes with peers; these do not.

The module is now complete. The reader who has absorbed chapters 01–07, worked through the exercises, and participated in a full drill cycle has the AI SRE's DR / BCP / multi-region capability as a practice, not a document — ready to stand against the kind of regional disruption the next year is likely to produce.
