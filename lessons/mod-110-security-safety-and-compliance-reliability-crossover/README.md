# mod-110-security-safety-and-compliance-reliability-crossover: Reliability Crossover with Security, Safety, and Compliance

**Estimated effort:** 12 hours

An AI SRE operates next to three programs whose work shares the same substrate but whose ownership is distinct: the `ai-infra-security` peer's defensive-controls program, the `ai-risk-engineer` peer's AI risk management framework, and the legal / compliance function's regulator-facing reporting. Classical SRE sees these programs only through the back door of incident response. On an AI fleet, the programs cross the serving path every request: guardrail sidecars sit on the critical path, compliance obligations impose continuous-monitoring duties, safety commitments condition releases on operational mitigations. The AI SRE's job is to **own the reliability dimension of those programs' work** — the latency budget the guardrails live inside, the telemetry contract the compliance surface depends on, the operational mitigations the safety gate counts on — without inheriting the programs' primary responsibilities. This module writes the discipline.

## Learning objectives

- Own the reliability dimension of defensive controls — guardrail-sidecar latency budget (how many ms the guardrail is allowed to cost), PII-filter cache-miss impact on TTFT, model-extraction query-rate control impact on P99, adversarial-input-induced timeouts, policy-engine load as a reliability event
- Co-author joint post-mortems with the ai-infra-security peer (level 35) — when a defensive-control change caused a reliability regression, when a reliability remediation weakened a defensive posture, when a shared root cause (e.g., retrieval index poisoning) crosses both domains
- Operate the reliability side of the compliance contract — EU AI Act Article 72 post-market-monitoring hooks, FDA PCCP triggered-change hooks, SR 11-7 model-risk notification; produce the AI-specific telemetry the ai-risk-engineer peer and legal teams need without taking on their reporting responsibility
- Reconcile the reliability contract with safety / responsible-AI programs — Anthropic RSP, OpenAI Preparedness Framework, Google DeepMind Frontier Safety Framework-style release tiering, where the reliability gate overlaps with the safety gate and where they intentionally diverge
- Handle incident-sensitive data correctly — prompt / completion content capture with retention limits, chain-of-custody discipline for AI-safety-adjacent incidents, redaction and selective sharing with SecOps / IR; defer to those teams for evidence handling
- Cite the boundaries — adversarial-defence methodology is at ai-infra-security peer; AI risk management framework ownership is at ai-risk-engineer peer; legal / regulator-facing output is at legal / compliance out of scope

## Chapter index

- [`01-reliability-side-of-defensive-controls.md`](./01-reliability-side-of-defensive-controls.md) — The guardrail stack as a reliability surface; the three questions (latency contribution, failure behaviour, dependency surface) asked of each control; the per-hop latency-budget contract with allocation table, SLI emission, SLO burn alarms, and negotiation discipline; PII-filter cache-miss as a non-obvious TTFT regressor; model-extraction rate-shaper P99 tail behaviour; adversarial-input-induced timeouts and the AI SRE's three mitigations; policy engines as compute with their own failure modes.
- [`02-joint-post-mortems-with-security-peer.md`](./02-joint-post-mortems-with-security-peer.md) — The three incident shapes (defensive change caused reliability regression, reliability remediation weakened defensive posture, shared root cause) with worked archetypes; the joint template extending `mod-108` chapter 03 with co-author header, joint narrative, function-specific detail blocks, extended root-cause taxonomy, ownership-tagged action items, twin lesson blocks; the per-shape template branches; the drafting cadence and publication discipline; the quarterly joint retrospective as the compounding mechanism.
- [`03-compliance-telemetry-contract.md`](./03-compliance-telemetry-contract.md) — The attestation premise; Article 72 steady-state monitoring with the signal inventory, retention, lineage, and query SLO; FDA PCCP triggered-change workflow and the drift / unplanned-change signals; SR 11-7 ongoing-monitoring surface; the monitoring system's own SLO as a compliance surface; the compliance-telemetry attestation as the AI SRE's product; anti-patterns and the quarterly re-sign worksheet.
- [`04-reliability-gate-vs-safety-gate.md`](./04-reliability-gate-vs-safety-gate.md) — The two gates, defined; three public frameworks (Anthropic RSP, OpenAI Preparedness Framework, Google DeepMind Frontier Safety Framework) and their common shape; where the gates overlap (operational mitigations) and where they intentionally diverge (pre-deployment eval, capability-threshold response, purely reliability-shaped regressions); the reconciliation discipline (joint release checklist, capability-tier change protocol, pre-committed decision tree, shared evidence surface, published escalation path); three worked examples.
- [`05-incident-sensitive-data-discipline.md`](./05-incident-sensitive-data-discipline.md) — The six classes of incident-sensitive data; the five principles (minimum necessary content, named storage not ambient channels, chain of custody, legal-set retention, selective sharing); the class-by-class application; the operational runsheet; the specific anti-patterns; the tooling (named store, redaction library, chain-of-custody workflow, safety-samples sub-store, legal-hold register, automated share-expiry); the quarterly data-handling review.
- [`06-peer-boundaries-for-the-crossover.md`](./06-peer-boundaries-for-the-crossover.md) — The six peers on the crossover surface (`ai-infra-security`, `ai-risk-engineer`, legal / compliance, safety / policy, SecOps / IR, MRM / regulatory affairs / quality system); the AI SRE's irreducible crossover deliverables; the three ownership shapes (AI SRE drafts / peer approves, peer drafts / AI SRE reviews, joint draft); the anti-patterns at the boundary; the three bend-the-boundary patterns; the dashboard-and-review surface; the crossover worksheet.

## Structure

- `01-…md` … `06-…md`: lecture chapters.
- `exercises/`: per-exercise prompts (guardrail latency-budget authoring, joint post-mortem template, Article 72 telemetry hooks, reliability-gate / safety-gate reconciliation, incident-sensitive data discipline).
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.

## How this module composes with the track

- **Input from `mod-101`.** The AI SRE's role-split with `ai-infra-security` (level 35) and `ai-risk-engineer` is the premise under which chapter 06 draws the peer boundaries.
- **Input from `mod-102`.** Classical SRE discipline — SLOs as contracts, error budgets, non-abstract large-systems design — is the ancestry the latency-budget contract in chapter 01 §2 extends.
- **Input from `mod-103`.** The SLO stack is where the per-hop guardrail budgets are signed into production and where chapter 01 §2.3's burn-rate alarms live.
- **Input from `mod-104`.** Every per-hop SLI and compliance telemetry signal cited in chapters 01 and 03 is grounded in a `mod-104` observability artifact. A signal that does not exist in the stack is not a compliance deliverable.
- **Input from `mod-105`.** The quality-signal set from the non-deterministic-failure taxonomy is the half of the compliance telemetry inventory (chapter 03 §2.2) that is not serving-side.
- **Input from `mod-107`.** The fallback contract is the artifact chapter 02 Shape B's joint retros amend with defensive-parity clauses, and the artifact chapter 04 §5's capability-tier change notice extends with safety-gate commitments.
- **Input from `mod-108`.** The incident-command structure, blameless post-mortem template, and the chapter 06 legal / compliance handoff are the surfaces this module extends with joint-authorship (chapter 02), steady-state telemetry (chapter 03), and in-incident data-handling discipline (chapter 05).
- **Input from `mod-109`.** The chaos program is what verifies each operational mitigation referenced in chapter 04 is actually in effect; chaos runs are the evidence base of the reliability-gate verdict.
- **Output to `mod-111`.** Release engineering uses chapter 04's joint release checklist as a release-gate component; the compliance telemetry attestation is a release-time deliverable.
- **Output to `mod-112`.** Data and config pipelines are the upstream for chapter 02 Shape C's shared root causes; this module names ingest discipline as the primary fix.
- **Output to `mod-113`.** DR / BCP for AI consumes chapter 04's operational mitigations as a readiness checklist for degraded-mode operation.
- **Output to `mod-114`.** Cross-functional leadership operates at the level this module's peer-boundary chapter sets and reads the quarterly joint retrospective readouts as leadership intake.
