# mod-114-cross-functional-reliability-leadership: Cross-Functional Reliability Leadership at Team Scope

**Estimated effort:** 14 hours

## What this module is

Mod-101 through mod-113 taught the mechanics of AI-specific reliability: SLO engineering, observability, non-deterministic failure diagnosis, capacity planning, vendor risk, incident response, chaos, security-reliability crossover, progressive delivery, data-and-config pipelines, and DR / multi-region. Each module ended with a per-chapter peer-boundary that bounded the chapter's scope. This module composes those mechanics into the operating posture of a function **at team scope** — the leadership-shaped work of running the reliability programme, authoring the cross-cutting decisions, holding the peer interfaces, communicating up, and positioning the career trajectory the role opens onto.

The module is explicitly not another mechanics module. It is the document the AI SRE team lead reads on day one and returns to monthly. It closes the track.

## Learning objectives

- Own an AI-reliability program at team scope — cadence, SLO-review ritual, on-call rotation design, deployment-change calendar, migration planning across the ladder of peer tracks (inference-engineer, performance, security, ML-platform, eval, mlops).
- Author a reliability-engineering RFC — build-vs-buy framing for the SRE-tooling stack (self-host Prometheus / Grafana / OTel vs. Datadog / New Relic / Chronosphere; self-host Chaos Mesh vs. Gremlin; self-host Flagger vs. a cloud provider's progressive-delivery service), total-cost-of-ownership model, risk register, migration path.
- Run cross-peer interface contract design — what tickets this team files to the performance peer (kernel regressions, Nsight-level profiling asks) and how, what the ML-platform peer owes this team (artifact lineage, eval evidence, rollback hook), what the security peer co-owns (joint post-mortems, shared latency budget), what the eval peer owes (release-gate signal, drift alert) — explicit role boundaries, mutual deliverables, escalation paths.
- Run the SLO-review ritual at team scope — monthly review of top-N SLOs, error-budget-burn trends, top five open incidents, top five open runbook gaps, feed into release-freeze / feature-work rebalancing decisions.
- Communicate reliability upward — translate error-budget burn into business impact for the product lead, translate recurring incident classes into architecture asks for the architect peer (level 40), author board-adjacent reliability narratives without crossing into head-of-ai-governance / CAIO scope.
- Position the AI-SRE career trajectory — what senior-engineer / staff / architect / principal looks like next, what certifications are load-bearing in enterprise hiring (CKA / CKAD, NVIDIA DLI GPU courses, (ISC)² CISSP for BCP / incident-response credibility, cloud-provider reliability specialty certs), and how to contribute upstream to the open-source reliability toolchain (OTel, DCGM-exporter, Prometheus exporters, chaos-mesh experiments).

## Chapter index

1. [Owning the Reliability Program at Team Scope — Cadence, Rituals, and Rotation](01-reliability-program-cadence-and-ritual.md) — the five-ritual core (weekly operational review, monthly SLO review, weekly on-call handoff, quarterly retrospective, annual programme review), the three operational artifacts (on-call rotation design, deployment-change calendar, migration tracker), the RACI document, the five-level maturity ladder, and the first-90-days plan for a new team lead.
2. [Authoring the Reliability-Engineering RFC — Build-vs-Buy for the SRE Tooling Stack](02-reliability-engineering-rfc-build-vs-buy.md) — the seven-section RFC shape (context, options, TCO, operational qualities, risk register, migration path, recommendation), the five decision axes, the per-component specialisations (observability, chaos, progressive delivery), the five-phase migration path, and the three-ring review loop.
3. [Cross-Peer Interface Contracts — Performance, Inference-Engineer, Security, ML-Platform, Eval, MLOps](03-cross-peer-interface-contracts.md) — the one-page interface-contract shape, the six peer contracts the function owns (with mutual deliverables, SLAs, and escalation paths per peer), the cross-service SLO composition problem, and the quarterly peer-interface health check.
4. [The SLO Review Ritual — Monthly Facilitation at Team Scope](04-slo-review-ritual.md) — the forum's three authorised decision classes, pre-meeting preparation, the eight-item agenda, the Git-tracked artifact, the error-budget policy's role, facilitator discipline against six recurring anti-patterns, and the first-year trajectory of the ritual.
5. [Communicating Reliability Upward — Product, Architect, Leadership, Board](05-communicating-reliability-upward.md) — the three audiences (product lead, architect peer, leadership / board), the specific translation move for each, the monthly product-lead brief, the architecture-ask ticket, the quarterly reliability narrative, and the governance boundary.
6. [Positioning the AI-SRE Career Trajectory — Levels, Certifications, Open-Source Contribution](06-ai-sre-career-trajectory.md) — the four altitudes above level 35, the load-bearing enterprise certifications (CKA / CKAD, NVIDIA DLI, (ISC)² CISSP, cloud-provider operations-specialty), the portfolio evidence that signals level, the open-source toolchain contribution surfaces (OpenTelemetry, Prometheus ecosystem, Chaos Mesh, Flagger / Argo Rollouts), and the anti-patterns that stall the trajectory.

## Structure

- `01-…md` … `06-…md`: lecture chapters.
- `exercises/`: per-exercise prompts for hands-on practice.
- `labs/`: long-form hands-on labs (scaffolded).
- `quizzes/`: knowledge checks (scaffolded).
- `resources.md`: external references.

## Exercises

1. [Reliability program cadence and ritual authoring](exercises/exercise-01-reliability-program-cadence-and-ritual-authoring.md) — produce the full programme document for a specific team: the five-ritual cadence, the three operational artifacts, the RACI, and the first-90-days plan from chapter 01 §7.
2. [SRE-tooling build-vs-buy RFC authoring](exercises/exercise-02-sre-tooling-build-vs-buy-rfc-authoring.md) — write a complete chapter-02-shape RFC for one of the three large build-vs-buy decisions, including TCO, risk register, migration path, and recommendation.
3. [Cross-peer interface contracts: performance, inference-engineer, security, ML-platform, eval, mlops](exercises/exercise-03-cross-peer-interface-contracts-performance-inference-security-mlplatform-eval-mlops.md) — author the six one-page interface contracts and the composition artifact from chapter 03 §8, signed by named peer leads.
4. [SLO review ritual facilitation drill](exercises/exercise-04-slo-review-ritual-facilitation-drill.md) — run a complete SLO review from pre-reading pack through signed minute, including a mock red-window decision and an architect-escalation authorisation.
5. [Reliability communication upward to product and architect](exercises/exercise-05-reliability-communication-upward-to-product-and-architect.md) — author one monthly product-lead brief, one architecture-ask ticket, and one quarterly reliability narrative for leadership, with a mock-review loop against the chapter 05 §5 standards.
6. [AI-SRE career trajectory and certification portfolio](exercises/exercise-06-ai-sre-career-trajectory-and-certification-portfolio.md) — produce a personal three-year trajectory plan with certification mix, portfolio-evidence targets, and an open-source contribution plan composed with the day job.

## How this module composes with the track

- **Input from `mod-101`.** The ladder, the role charter, and the peer deferral map that this module's cross-peer interface contracts (chapter 03) and chapter 06 career discussion inherit.
- **Input from `mod-103`.** The SLO engineering vocabulary and the error-budget policy that chapter 04's ritual applies.
- **Input from `mod-104`.** The observability pipeline the chapter 04 SLO review reads from and the chapter 02 observability RFC debates the shape of.
- **Input from `mod-105`.** The failure-class catalogue that the chapter 04 ritual uses to characterise recurring incident classes and that chapter 05 §3 translates into architecture asks.
- **Input from `mod-106`.** The capacity-planning posture chapter 02's self-host TCO numbers inherit.
- **Input from `mod-107`.** The vendor-risk vocabulary chapter 02's risk register and chapter 03 §3 and §4's peer-interface boundaries inherit.
- **Input from `mod-108`.** The incident response / post-mortem discipline the chapter 04 ritual reads from and the chapter 03 §4 joint post-mortem convention composes with.
- **Input from `mod-109`.** The chaos programme the chapter 02 RFC applies to chaos tooling and the chapter 03 §9 joint-drill-design pattern composes with.
- **Input from `mod-110`.** The security-reliability crossover the chapter 03 §4 security peer contract inherits and the chapter 05 §4.4 governance boundary explicitly respects.
- **Input from `mod-111`.** The release engineering practice the chapter 02 progressive-delivery RFC debates, the chapter 03 §6 eval and §7 mlops contracts inherit, and the chapter 05 §4.2 DORA-signal composition reads from.
- **Input from `mod-112`.** The data / config pipeline practice chapter 01's deployment-change calendar composes with and chapter 03 §5 ML-platform contract touches on.
- **Input from `mod-113`.** The DR / BCP practice chapter 01's annual programme review inherits and chapter 03 §9's cross-service SLO composition problem elevates.

## What this module does *not* cover

- **New reliability mechanics.** The mechanics were authored in mod-102 through mod-113; this module composes them at team scope.
- **Multi-year strategy.** The chapter 06 principal / architect altitude references the `ai-infra-senior-architect` peer (mod-113 chapter 07 §3); that peer's scope is not re-authored here.
- **Governance narratives.** Mod-110 chapter 06 and chapter 05 §4.4 of this module explicitly draw the boundary against the `head-of-ai-governance` / CAIO scope. The function composes with those peers but does not substitute for them.
- **Specific posting evidence for the AI-SRE title.** Mod-101 chapter 06 is the home for the hiring-market research; this module references but does not re-author it.
