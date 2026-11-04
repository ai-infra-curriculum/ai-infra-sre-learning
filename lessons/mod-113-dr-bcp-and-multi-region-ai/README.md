# mod-113-dr-bcp-and-multi-region-ai: DR, BCP, and Multi-Region for AI Workloads

**Estimated effort:** 14 hours

## What this module is

A regional disruption is the DR event an AI platform team is most likely to run in anger. Full cloud-provider collapses are rare; whole-region impairment — an AZ cascade, a hosted model-provider's regional endpoint returning 5xx while the global status page stays green (`mod-107` chapter 03), a bad control-plane deploy at the vendor — is routine enough that every tier-1 service has a nonzero chance of one each year. The classical DR canon is mature: NIST SP 800-34 Rev 1 names the process, ISO 22301 formalises the management-system shape, the AWS / Azure / Google Cloud reliability pillars encode the substrate mechanics. None of these documents writes the AI-specific extensions the function must author: the model-weight artifact replication cost, the cold-start-dominated RTO, the state surface that includes conversation and tool-call state, the data-residency constraint on failover direction, and the model-provider dependency as a first-class DR concern.

This module writes those. It grounds the function in the four primary references (chapter 01), writes the regional-outage playbook for AI workloads (chapter 02), writes the state-reconstitution plan for stateful agent systems (chapter 03), writes the residency-respecting failover design (chapter 04), writes the drill cadence that turns the plan into a practice (chapter 05), writes the BCP artifact pack that makes the practice visible to leadership and auditors (chapter 06), and closes on the peer boundaries that place this work in the ladder from `mod-101` (chapter 07).

## Learning objectives

- Ground the module in NIST SP 800-34 Rev 1 + the three cloud-provider reliability guides (AWS Well-Architected Reliability Pillar, Azure Reliability Principles, Google Cloud Reliability Guide) and adapt each to AI-specific concerns.
- Author regional-outage playbooks for AI workloads — DNS / Anycast / global-LB failover for the inference endpoint, cross-region replication of model-weight artifact stores (S3-class), warm-standby regions for cold-start-sensitive large models, provider-fallback across regions when the provider itself goes regional.
- Plan state reconstitution for stateful agent systems — conversation / memory state replication, KV-store replication across regions, tool-call-state continuity, trace-store replication; design around Recovery Point Objective (RPO) and Recovery Time Objective (RTO) targets per workload class.
- Respect data residency in failover — EU-resident traffic cannot silently failover to US; design per-region isolation with controlled bursting, and reason about the residency / reliability trade-off when the primary region is down.
- Rehearse DR as a regular practice — tabletop exercises (quarterly), controlled region-switch drills (semi-annually), full failover rehearsal (annually), integrated with the chaos program from `mod-109`.
- Build the business-continuity artifacts — DR plan document, RPO / RTO table per service, call tree, communication templates, regulator-facing summary scaffold — enough for a regulated-industry audit without taking on the audit itself (which sits with `ai-risk-engineer` + legal).
- Cite the boundaries — multi-region architecture at team-of-teams scope is at `ai-infra-architect` (level 40); multi-year DR strategy is at `ai-infra-senior-architect` (level 45); regulator-facing DR evidence is at legal / compliance.

## Chapter index

1. [Frameworks Grounding: NIST SP 800-34 and the Three Cloud Reliability Guides](01-frameworks-grounding.md) — the four primary references, the NIST seven-step process, the three-tier plan taxonomy (ISCP / DRP / BCP), the four DR strategies (backup / restore, pilot light, warm standby, multi-site active-active), and the five AI-specific extensions the frameworks assume but do not write.
2. [Regional Outage Playbooks for AI Workloads](02-regional-outage-playbooks.md) — the four impairment shapes, the three endpoint-failover substrates (DNS / Anycast / global LB), cross-region model-weight replication with hash integrity, warm-standby sizing for cold-start-sensitive models, provider-level fallback across regions, and the end-to-end failover sequence with a ten-item checklist.
3. [State Reconstitution for Stateful Agent Systems](03-state-reconstitution-for-stateful-agents.md) — the six state classes (conversation, GPU KV, distributed cache, tool-call, retrieval context, trace, config), per-class replication strategies, the three-way tool-call recovery decision, the trace-store write-pipeline survival pattern, and the per-service RPO / RTO matrix.
4. [Data Residency in Failover](04-data-residency-respecting-failover.md) — residency vs. sovereignty vocabulary, the regional-stamp pattern as the design primitive, the residency / reliability trade-off, three failover patterns (two-in-residency, one-in-residency with degraded fallback, pre-authorised cross-residency burst), and the implementation primitives that enforce the posture in code.
5. [DR Rehearsal as a Regular Practice](05-dr-rehearsal-cadence.md) — the three exercise tiers (tabletop, functional, full-scale), the cadence table per service tier, the scenario catalogue, auto-halt and blast-radius controls, integration with the `mod-109` chaos programme, the game-day shape, and the five-level maturity staircase.
6. [Business Continuity Artifacts](06-bcp-artifacts.md) — the ten-component artifact pack (policy, BIA, per-service ISCPs, RPO / RTO table, call tree, communication templates, regulator-facing summary scaffold, drill log, risk register, annual review record), each with its template, its living-document discipline, and its sign-off.
7. [Peer Boundaries for DR, BCP, and Multi-Region AI](07-peer-boundaries-for-dr-and-multi-region.md) — the three boundaries (`ai-infra-architect` for team-of-teams multi-region, `ai-infra-senior-architect` for multi-year strategy, legal / compliance for regulator-facing testimony), what the function hands up and receives back, and the four-question routing rule.

## Structure

- `01-…md` … `07-…md`: lecture chapters.
- `exercises/`: per-exercise prompts for hands-on practice.
- `labs/`: long-form hands-on labs (scaffolded).
- `quizzes/`: knowledge checks (scaffolded).
- `resources.md`: external references.

## Exercises

1. [NIST SP 800-34 applied to an AI workload](exercises/exercise-01-nist-sp-800-34-applied-to-an-ai-workload.md) — take the seven-step process + three-tier plan taxonomy and map it onto a specific AI service of the student's choice.
2. [Regional outage playbook authoring](exercises/exercise-02-regional-outage-playbook-authoring.md) — write the `rb-regional-failover` runbook for one tier-1 service at command-level specificity against the chapter 02 substrates.
3. [Stateful agent state reconstitution plan](exercises/exercise-03-stateful-agent-state-reconstitution-plan.md) — produce the per-service RPO / RTO matrix (chapter 03 §7) and validate the three-way tool-call recovery against the agent's tool catalogue.
4. [Data residency-constrained failover design](exercises/exercise-04-data-residency-constrained-failover-design.md) — write the per-stamp residency posture (chapter 04 §5) for an EU + US + APAC deployment, including the pre-authorised cross-residency burst design.
5. [DR rehearsal schedule from tabletop to full failover](exercises/exercise-05-dr-rehearsal-schedule-table-top-to-full-failover.md) — produce the annual drill schedule, author one tabletop and one functional-drill scenario, and design the auto-halt wiring for each.
6. [BCP artifact pack for a regulated-industry audit](exercises/exercise-06-bcp-artifact-pack-for-regulated-industry-audit.md) — assemble the ten-component pack from chapter 06 at audit-ready quality and walk it through a mock audit interview with a function member playing the auditor.

## How this module composes with the track

- **Input from `mod-101`.** The ladder positioning and peer slugs (ai-infra-architect, ai-infra-senior-architect, ai-risk-engineer) read in chapter 07 are canonical in `mod-101`.
- **Input from `mod-103`.** SLO / error-budget practice is the vocabulary chapter 02's cutover criteria and chapter 05's auto-halt wiring inherit.
- **Input from `mod-104`.** Observability — metrics, traces, logs — is the substrate chapter 03 §5's trace-store survival pattern and the entire drill-signal set depend on.
- **Input from `mod-105`.** The idempotency contract for tool calls is the upstream of chapter 03 §4's three-way tool-call recovery.
- **Input from `mod-107`.** The provider-fallback contract and provider-status ingestion are the inputs for chapter 02 §5's provider-level failover.
- **Input from `mod-108`.** The runbook library (chapter 01 of that module) is where this module's playbooks live; the incident command structure (chapter 02 of that module) is what the drill game-days inherit.
- **Input from `mod-109`.** The chaos programme is this module's weekly cadence partner; the game-day shape, the three-family alarm, and the auto-halt wiring are shared.
- **Input from `mod-110`.** Audit emission and retention is the discipline this module's trace-store survival and residency-scoped audit emission feed into.
- **Input from `mod-111`.** Release engineering is the discipline chapter 03 §6.3's "pause non-urgent promotions during DR" composes with.
- **Input from `mod-112`.** The prompt registry, config CRDs, retrieval index, and feature store are the data / config pipelines whose cross-region replication this module writes.
- **Output to `mod-114`.** Cross-functional reliability leadership operates at the level chapter 07's peer boundaries define.
