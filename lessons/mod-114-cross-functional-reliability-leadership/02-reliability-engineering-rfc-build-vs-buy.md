# Authoring the Reliability-Engineering RFC: Build-vs-Buy for the SRE Tooling Stack

## Why this chapter exists

Once per year — sometimes twice, when the vendor contract renews on an aggressive uplift or the infra bill becomes a line item in the quarterly leadership review — the AI SRE function faces a decision that it cannot settle informally. Do we continue to self-host Prometheus, Grafana, and OpenTelemetry Collector, or do we move to a vendor observability stack like Datadog, New Relic, or Chronosphere? Do we continue to run Chaos Mesh on our own cluster, or do we buy Gremlin? Do we keep Flagger as our progressive-delivery controller, or do we move to a cloud provider's managed progressive-delivery service?

These decisions are not neutral. Each one commits the function to a multi-year operating posture, a specific migration cost, a specific capital-vs-operating-expense profile, a specific portability posture, a specific set of failure modes during the next vendor outage. They are also decisions that produce the strongest "vendor dumping" invitations the function will see — the free professional-services onboarding, the aggressive first-year discount, the account-executive attention that *feels* like leadership attention. A function that makes these decisions on calendar-urgency, on vendor-courtship, or on one senior engineer's Twitter opinion is a function that will regret the decision within eighteen months.

The **reliability-engineering RFC** is the artifact that forces the decision to be made on written evidence. It is long-form, Git-tracked, explicit about assumptions, explicit about reversibility, and reviewed by peers and by leadership before any procurement or any migration begins. This chapter writes the RFC's shape. Chapter 01 of this module gave the programme ritual (the annual build-vs-buy review in §2.5); this chapter writes the artifact that ritual produces.

Primary references: the [Google SRE Workbook chapter 9 (SRE Engagement Model)](https://sre.google/workbook/engagement-model/) for the frame of SRE taking on or declining a surface; the [IETF RFC style and process](https://datatracker.ietf.org/doc/html/rfc7322) for the written-decision convention this chapter borrows; mod-101 chapter 02 for the peer boundaries the RFC's cross-track dependencies map to; mod-107 chapter 01 and chapter 02 for the vendor-risk vocabulary the RFC's risk register inherits.

## 1. The three decisions this chapter covers

The function's SRE tooling stack has many components; three of them concentrate the largest build-vs-buy decisions because they have the largest TCO, the sharpest lock-in risk, and the strongest substitution paths.

- **Observability.** Self-host [Prometheus](https://prometheus.io/docs/introduction/overview/) + [Grafana](https://grafana.com/docs/) + [OpenTelemetry](https://opentelemetry.io/docs/) + a chosen trace / log backend ([Loki](https://grafana.com/docs/loki/), [Tempo](https://grafana.com/docs/tempo/), [Mimir](https://grafana.com/docs/mimir/), [VictoriaMetrics](https://docs.victoriametrics.com/), [Jaeger](https://www.jaegertracing.io/docs/), [Elastic](https://www.elastic.co/guide/)); or buy [Datadog](https://docs.datadoghq.com/), [New Relic](https://docs.newrelic.com/), [Chronosphere](https://docs.chronosphere.io/), [Honeycomb](https://docs.honeycomb.io/), [Dynatrace](https://docs.dynatrace.com/), or [Splunk Observability Cloud](https://docs.splunk.com/Observability/).
- **Chaos engineering.** Self-host [Chaos Mesh](https://chaos-mesh.org/docs/) or [LitmusChaos](https://litmuschaos.io/docs/); or buy [Gremlin](https://www.gremlin.com/docs/) (or a cloud-provider's chaos service like [AWS Fault Injection Service](https://docs.aws.amazon.com/fis/latest/userguide/) or [Azure Chaos Studio](https://learn.microsoft.com/en-us/azure/chaos-studio/)).
- **Progressive delivery.** Self-host [Flagger](https://docs.flagger.app/) or [Argo Rollouts](https://argoproj.github.io/argo-rollouts/); or buy a cloud provider's progressive-delivery service (e.g. [AWS App Mesh](https://docs.aws.amazon.com/app-mesh/) + [CodeDeploy canary](https://docs.aws.amazon.com/codedeploy/latest/userguide/), [Google Cloud Deploy](https://cloud.google.com/deploy/docs)).

The same RFC shape applies to each; the arguments and the TCO numbers differ. This chapter writes the shape once, uses observability as the worked example, and names at §8 the two specialisations for chaos and progressive delivery.

## 2. The RFC's seven sections

An AI SRE build-vs-buy RFC runs seven sections. The sections are ordered so that a reviewer who stops reading halfway through still has a coherent view.

### 2.1 Context and problem statement

- **Why now.** The event that triggered the RFC: contract renewal, cost overrun, operational pain, a cross-team migration, an architect's roadmap item, a regulatory deadline, or a scheduled annual review (chapter 01 §2.5).
- **Current state.** What the stack is today: components, versions, operators, approximate scale (cardinality for observability, blast-radius envelopes for chaos, rollout rate for progressive delivery), approximate team-time spent per week on the stack's own reliability.
- **Problem.** The specific pain the status quo produces — not a vague "it's getting hard," but a measurable claim: a cardinality explosion that is costing N GB/day on the TSDB, a Grafana upgrade that has been blocked for X months because of dashboard breakage, an on-call burden of Y hours per week on the stack itself.
- **Scope.** What this RFC is about and what it is not. An observability RFC is not a logging-retention policy change (that is an operational doc); a chaos RFC is not a game-day cadence change (that is chapter 01 §2); a progressive-delivery RFC is not a release-gate contract change (that is mod-111 chapter 05).

The problem statement is the section future readers will quote. It must be defensible against "but we always did X" — the answer to "why now" is the pivot that forced the writing.

### 2.2 Options under consideration

Enumerate the options. For the observability case:

- **Option A — stay self-hosted** (the default; include always, even if clearly suboptimal, because the reviewer must see what the function is comparing against).
- **Option B — move to a vendor SaaS observability stack** (name the vendor explicitly; "Datadog" is a concrete option, "a SaaS observability tool" is not).
- **Option C — hybrid** (self-host metrics; vendor logs / traces; or vice versa). Always include a hybrid when the components have independent decisions; do not include a hybrid where it would create two sets of on-call surfaces to the same problem with no clear boundary.
- **Option D — alternative open-source** (e.g. move from Prometheus + Thanos to VictoriaMetrics; move from Loki to Elastic; move from Grafana-stack to the cloud provider's managed open-source offering).

For each option, write one paragraph on what the option is, two to three bullet points on its distinguishing advantages, and two to three bullet points on its distinguishing disadvantages. Keep the TCO out of this section — that is §2.3. The purpose here is to make the option space explicit.

### 2.3 TCO model

A total-cost-of-ownership model, run on a three-year horizon, broken out per option. The model must be reproducible — a spreadsheet or notebook linked from the RFC — and must show the sensitivity of the answer to its three or four largest inputs.

**Cost dimensions.**

- **Infrastructure cost.** Compute, storage, network egress, cross-region replication. For self-hosted, the node count × instance cost + the storage cost + the egress from scraping GPU exporters and from cross-region federation. For vendor, the ingest rate × per-GB or per-datapoint cost + the per-user license + the custom-metric cost + the APM host cost + the network-egress cost to ship data to the vendor (often forgotten; non-trivial for high-cardinality AI telemetry).
- **Engineering cost.** The team-time spent operating the stack. For self-hosted, count: upgrade / patch cadence, cardinality-review cadence, Grafana-dashboard maintenance, alert-rule authoring and tuning, on-call for the stack itself, training onboarding time for a new hire. Convert to engineer-days / year × loaded engineer cost. For vendor, count: configuration-authoring time, integration maintenance, dashboard migration cost, on-call for the stack's *integration* (not the stack's substrate, which is the vendor's problem), vendor-account management overhead.
- **Migration cost.** One-time, in the first 6–18 months. For a vendor move: data-shipper deployment (OTel-Collector configuration, agent rollouts), dashboard porting, alert-rule porting, runbook updates, team training, parallel-run duration (double-paying infra during the overlap). For a self-host rebuild: capacity planning, new-component evaluation, deployment automation, migration-script authoring.
- **Egress and exit cost.** The cost of *leaving* the chosen option two or three years out. Vendor contracts frequently include non-trivial exit — bulk data egress, re-shipping historical data, vendor-specific ingestion pipelines to unwind. Self-hosted exit is often lower but not zero (you have cardinality-explosive dashboards you authored assuming unlimited retention).
- **Risk-weighted cost.** Not a dollar figure but a surcharge applied to high-risk options. See §4.

**Example TCO shape for observability.** For a team of five engineers running an AI serving fleet with ~200 nodes, ~10k active series per node, 1–3 TB/day of traces, the three-year TCO ballparks (orders of magnitude; the function runs its own numbers):

| Option | Y1 infra | Y1 eng | Y1 migration | Y2–Y3 infra | Y2–Y3 eng | 3-yr total order |
|---|---|---|---|---|---|---|
| A — self-host | medium | high | low | medium | high | medium |
| B — vendor SaaS | medium–high | low | medium | high–very high | low | medium–high |
| C — hybrid | medium | medium | medium | medium | medium | medium |
| D — alt OSS | medium | medium | medium | medium | medium | medium |

The function fills in the dollars; the spreadsheet shows the sensitivity. The number that comes out is rarely the deciding factor on its own — a 1.5× TCO difference is usually dwarfed by the operational and strategic factors in §3 and §4 — but writing it down forces the function to notice when the dollars have become decisive, and to say so.

<!-- needs-research: a public 2026 vendor-cost benchmark for observability (Datadog / New Relic / Chronosphere) sized against a frontier-model serving fleet. Ingest-rate, custom-metric cost, and APM-host cost all moved in the 2024–2026 window; cite current rate cards rather than relying on a frozen table. -->

**Reproducibility.** The TCO must be reproducible by the reader. Vendor pricing changes, salary bands change, workload scale changes. The RFC links the spreadsheet; the spreadsheet documents its assumptions; the leadership reviewer can re-run the numbers with their own inputs.

### 2.4 Operational qualities (beyond TCO)

A pure TCO comparison misses dimensions that matter more over a three-year horizon than any dollar difference. Rate each option on each dimension as *better / comparable / worse* and write one sentence of evidence per rating.

- **Time to first value.** How long before a new signal, dashboard, or alert is live. Vendor often wins on this axis for standard use cases; self-host wins for custom AI-specific signals (per-token latency, cache-hit depth, GPU-telemetry correlation) that vendor dashboards do not know about natively.
- **Cardinality and custom-metric support.** The dominant AI-platform observability cost driver. Vendor billing often penalises high-cardinality labels in ways that force the function to prune signals the stack needs (mod-104 chapter 03 §4's per-model-version / per-tenant / per-route cardinality decision).
- **Integration with AI-specific signals.** OpenTelemetry [GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/), NVIDIA [DCGM-exporter](https://github.com/NVIDIA/dcgm-exporter), [node-exporter](https://github.com/prometheus/node_exporter), engine-specific Prometheus exporters (vLLM, TensorRT-LLM metrics endpoints). Which option supports these today, how well, through what mechanism.
- **Query-language and alerting.** [PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/) vs. vendor-specific DSLs. The function has existing alert rules, runbook references, and team mental models in one; migrating to another has a six-to-twelve-month familiarity cost.
- **Portability and lock-in.** How hard is it to leave. OpenTelemetry on the ingestion path keeps the function's instrumentation portable; proprietary SDKs do not.
- **Data residency and sovereignty.** Where does the data land. For EU-resident traffic (mod-113 chapter 04), a vendor that does not offer an EU-resident tenancy or does not contractually commit to EU-only storage is disqualified for that surface.
- **On-call surface.** With self-host, the function is on-call for the observability stack itself; with vendor, the function is on-call for the integration layer and for the vendor's own incidents (a vendor observability outage during an SRE-visible incident is a specific class of pain).
- **Multi-tenant and multi-team properties.** The function may be building for team scope now but the stack composes with the organisation's wider platform. The RFC names how this option composes with the architect peer's roadmap (chapter 05 §2 and §3).

### 2.5 Risk register

An explicit list of risks per option. See §4 for the structure the function uses; the register is embedded in the RFC and signed off by the risk-engineer peer and the architect peer.

### 2.6 Migration path

The written sequence by which the function would move from current state to the recommended option, with cutover criteria and rollback paths. See §5 for the shape.

### 2.7 Recommendation and decision

A single-paragraph recommendation with the rationale compressed. A list of reviewers (named engineers and named leadership) with signoff slots. A date by which the decision is final. A pointer to the follow-up RFC (if any — a decision to buy may spawn a sub-RFC on the vendor-procurement details; a decision to self-host may spawn a sub-RFC on the next-generation architecture).

The recommendation is the function's *written opinion*. It must be defensible under cross-examination and it must be reversible — the RFC says, explicitly, what evidence would change the decision and when the function will re-evaluate.

## 3. The build-vs-buy decision axes

Before the TCO spreadsheet produces a number, the function should know on which of five axes the decision will land. The same spreadsheet can produce the same TCO and the decision can go either way depending on which axis dominates.

### 3.1 Cost at scale

The axis the TCO model (§2.3) measures directly. Most functions over-weight this axis early, then under-weight it at the three-year mark when vendor pricing has escalated or self-host operational cost has compounded. The TCO model's three-year horizon is what forces the honest comparison.

### 3.2 Control over the signal surface

Self-host gives the function direct control: the Prometheus scrape config, the retention tier, the recording rule, the alert expression, the trace sampler. Vendor gives the function control through the vendor's abstractions, which may or may not expose the primitive the function needs. For AI-specific signals (per-token latency distribution with per-provider / per-tenant / per-route labels; GPU-telemetry correlated to model-version; trace spans with the OpenTelemetry GenAI conventions) the primitives matter more than for a conventional web service; this axis frequently tilts the decision.

### 3.3 Portability and reversibility

Self-host with standard open-source components is portable: if the function is unhappy with the Grafana dashboard experience in two years, moving to a different dashboarding layer costs porting-the-dashboards, not re-instrumenting-the-fleet. Vendor with proprietary instrumentation is less portable; the function's agents, SDKs, and dashboards are vendor-specific and the data is in the vendor's store. The function reads portability as a hedge against a bad decision — *the decision we make today may be wrong; how expensive is it to undo three years out?*

### 3.4 On-call burden

Self-host means the function is on-call for the observability stack's own outages, cardinality explosions, upgrade failures, and capacity miscalculations. Vendor shifts the primary on-call to the vendor — but introduces a different on-call pattern: the function is now on-call for the integration layer (OTel Collector, agents, vendor-side limits) and for incidents that occur *while the vendor is down*. For a team of three SREs operating a frontier serving fleet, the self-host burden is often the differentiator; for a team of eight operating a mature stack, it rarely is.

### 3.5 Strategic positioning

The decision composes with the architect peer's roadmap (chapter 05 §3) and with the senior architect's multi-year strategy. A function that is one of three teams moving to the same vendor, with the architect's active sponsorship, has leverage the function gains from the shared buying power; a function that is the only team on a bespoke self-host while the organisation moves to vendor has a maintenance liability it may not be resourced to carry.

The function names the decisive axis explicitly in §2.7 of the RFC. A recommendation on "control over the signal surface" and a recommendation on "cost at scale" look identical on the summary line and feel completely different under three years of operation.

## 4. The risk register

The RFC includes a risk register per option. Each entry has:

- **ID.** A stable reference the reviewer can cite (`R-OBS-01`, `R-OBS-02`, ...).
- **Statement.** One sentence describing the risk ("Vendor pricing increases by > 25% per year at renewal, not covered by the current contract").
- **Likelihood** × **impact.** A lightweight scoring (low / medium / high) each. The product is a heat-map cell; the function does not pretend to quantitative precision the data does not support.
- **Owner.** One named person on the function or peer graph who is accountable for watching this risk and surfacing it if it materialises.
- **Mitigation.** The specific action — contractual, technical, organisational — that reduces the likelihood or the impact. "We will evaluate quarterly" is not a mitigation; "we include a multi-year pricing cap in the contract and self-host a parallel high-cardinality signal set for three months of the year as a benchmark" is.
- **Trigger.** The observable condition under which the mitigation is invoked.
- **Escalation.** Where the risk goes if it exceeds the function's authority (usually `ai-infra-architect` for substrate-level, `ai-infra-senior-architect` for multi-year, legal for contractual, product for business-impact).

**Example risks for an observability RFC.**

- `R-OBS-01` — Vendor cardinality cost balloons as AI traffic grows → mitigation: contractual cardinality cap or committed-use discount; owner: SRE lead; trigger: ingest-cost run-rate exceeds 120% of plan for two consecutive months; escalation: architect.
- `R-OBS-02` — Vendor instrumentation locks the fleet to the vendor's SDK → mitigation: standardise all instrumentation on OpenTelemetry; owner: SRE lead; trigger: a proposed change adds a vendor-proprietary SDK; escalation: function can block the change unilaterally.
- `R-OBS-03` — Vendor observability outage during an SRE incident → mitigation: maintain a minimal self-host "last-resort" dashboard sourced from Prometheus scrape of a small signal subset, available during vendor outages; owner: SRE lead; trigger: vendor-advertised regional outage; escalation: incident commander (mod-108 chapter 02).
- `R-OBS-04` — Data residency violation when EU traffic metadata lands in the vendor's US tenancy → mitigation: EU-tenant separation with contractual commitment; owner: legal + SRE lead; trigger: new EU regulatory filing; escalation: legal (mod-113 chapter 07 §4).
- `R-OBS-05` — Team skill atrophy on Prometheus / PromQL after a two-year vendor migration → mitigation: maintain the internal learning path (this curriculum's mod-104); owner: SRE team lead; trigger: the next build-vs-buy review (chapter 01 §2.5); escalation: none, informational.

The risk register is the piece the RFC reviewer will remember. A well-written register says, explicitly, what could go wrong, how the function would notice, and what it would do. An absent or hand-waved register is the function telling the reviewer that it has not thought seriously about the decision.

## 5. The migration path

Any buy-or-swap decision produces a migration. The RFC's §2.6 writes it as five phases, each with written entry conditions, written exit conditions, and a written rollback path. The phases below use a vendor observability move as the worked example; the shape generalises.

### 5.1 Phase 1 — Prepare (4–8 weeks)

- Vendor contract signed with the negotiated terms (data residency, cardinality cap, pricing cap, SSO integration, support tier, exit clause).
- Non-production tenant stood up; small pilot team using the vendor for one service's non-production environment.
- OpenTelemetry Collector architecture finalised — the agents on the fleet ship to both the current self-host and the vendor in parallel.
- Entry condition: §2.7 of the RFC signed off.
- Exit condition: vendor tenant is live; a pilot team has written its first vendor-side dashboard and alert; dual-ship is working for one non-production service.
- Rollback: discard vendor tenant, do not migrate traffic.

### 5.2 Phase 2 — Parallel run (8–12 weeks)

- Instrument production fleet to dual-ship (OTel Collector forks the stream). The self-host remains the system of record for alerting.
- Port the top-N dashboards and the top-N alert rules from self-host to vendor. Validate that the vendor's rendering of each matches the self-host's.
- Train the team on vendor-side primitives (query language, alerting, dashboards). Chapter 06 §5 of this module compositions with this — the team's upskilling is a reliability-career investment.
- Entry condition: phase 1 exit met.
- Exit condition: top-N dashboards and alerts are mirrored and validated; team has run one on-call rotation with vendor-side primary dashboards while self-host remains the alerting system of record; the dual-ship cost overrun is within budget.
- Rollback: discard vendor dashboards and alerts, continue on self-host; the decision has been made but the migration has not completed.

### 5.3 Phase 3 — Cutover (2–4 weeks)

- Switch alerting system-of-record to the vendor for one service tier at a time. Start with tier-2 or tier-3; promote tier-1 only after two weeks of stable vendor-side alerting on the lower-tier.
- Each service cutover: re-point alert destinations, update the runbook library's dashboard links (mod-108 chapter 01), update the on-call documentation (chapter 01 §3.1).
- Entry condition: phase 2 exit met.
- Exit condition: all service tiers alerting through the vendor; the deployment-change calendar (chapter 01 §3.2) shows no red flags for two consecutive weeks.
- Rollback: revert the alert-system-of-record switch per service tier in reverse order; vendor remains live but passive.

### 5.4 Phase 4 — Decommission self-host (4–8 weeks)

- Wind down the self-host stack. Reduce retention to a short "last-resort" window (R-OBS-03). Retire the dedicated capacity. Preserve only the minimal set required for the vendor-outage fallback dashboard.
- Entry condition: phase 3 exit met plus one month of stable vendor-primary operation.
- Exit condition: self-host capacity reduced to the agreed minimum; the capacity recovered is re-deployed; the operational overhead is measurably lower.
- Rollback: expensive. Explicitly document the recovery cost: restoring full self-host after decommission is a quarter-length project.

### 5.5 Phase 5 — Post-migration review (4 weeks)

- Score the migration against the original RFC's success criteria.
- Update the RFC document in Git with the actuals: what was the realised TCO, what risks materialised, what new risks emerged, what the function would do differently.
- Place the output of this phase in the annual programme review record (chapter 01 §2.5).
- Entry condition: phase 4 exit met.
- Exit condition: the post-migration review is signed by the function lead and the architect peer; the lessons are reflected in the next build-vs-buy RFC cycle.

The timeline is six to nine months end-to-end for a well-run observability migration on a team of this scale. A faster timeline usually means the function has skipped the parallel-run phase; a slower timeline means an unresolved risk is driving the delay.

## 6. The review and signoff loop

The RFC is reviewed in three rings before any procurement or any migration begins.

- **Ring 1 — team review.** The AI SRE team reads the full RFC. Comments inline in the Git document. The function lead resolves each comment with a written reply (accept, decline, defer).
- **Ring 2 — peer review.** The `ai-infra-architect` peer reads for composability with the organisation's roadmap. The `ai-infra-security` peer reads for the data-handling implications. The `ai-infra-mlops` peer reads for the pipeline implications. The `ai-eval-engineer` peer reads for the release-gate implications if the RFC touches progressive delivery. The `ai-risk-engineer` peer reads the risk register. Each peer signs off or raises a blocking issue.
- **Ring 3 — leadership review.** The director / VP level reviews the recommendation, the TCO summary, and the top three risks. Leadership signs off on the commitment — procurement, migration cost, team-time allocation — and on the re-evaluation date.

A blocking comment at any ring returns the RFC to the author with a written reply obligation. Comments do not disappear by being closed without a reply; the review loop produces a durable record.

## 7. When the function declines to buy

The RFC may conclude that the recommendation is to *stay self-hosted*. Two patterns in which this is the right call:

- **The TCO difference is small and the control / portability axis matters.** A 20–30% TCO delta in favour of vendor is often eaten by cardinality-cost escalation within 18 months; the function is better off investing the engineering time to make the self-host stack sustainable.
- **The AI-specific signal surface is central to the function's mandate.** For a team whose SLOs include per-token latency distributions, per-provider failure classes, GPU-correlated latency breakdowns, and prompt-registry cache-hit depth, the function's dashboards and alerts are a *product* the function is actively iterating on. Vendor abstractions often lag the AI-specific signal surface by 12–24 months.

When the function declines to buy, it still produces the RFC — the written record of the decision protects the function the next time the question is raised. Chapter 06 §5 of this module composes with this: the self-host posture is an upstream contribution opportunity for the function (OTel Collector configurations, DCGM-exporter patches, custom exporters) that doubles as a career-development surface for the team.

## 8. Specialisations for chaos and progressive delivery

The same seven-section RFC shape applies with two local specialisations.

### 8.1 Chaos engineering (self-host Chaos Mesh / LitmusChaos vs. buy Gremlin / cloud-provider chaos)

- **Dominant decision axis.** Blast-radius safety and audit trail. Buy-side Gremlin and cloud-provider chaos services often ship stronger safety primitives (pre-approved experiment catalogues, built-in auto-halt, audit emission that satisfies compliance). Self-host gives you unconstrained experiment authoring and tighter integration with the deployment-change calendar (chapter 01 §3.2) but requires the function to build the safety guardrails itself.
- **Specific risks to add.** The buy-side risk that the vendor's safety abstractions do not match the AI-specific failure classes (GPU-telemetry-driven auto-halt, per-model-version experiments, prompt-registry-correlated experiments). The self-host risk that an experiment escapes its blast-radius envelope (mod-109 chapter 02).
- **Migration path specialisation.** The parallel-run phase (§5.2) runs the new chaos tool in *shadow* — experiments are planned and simulated against the new tool but executed on the old tool — for one full quarterly drill cycle before the first live experiment on the new tool.

### 8.2 Progressive delivery (self-host Flagger / Argo Rollouts vs. buy a cloud-provider managed service)

- **Dominant decision axis.** Release-gate composition and rollback primitives. Mod-111 chapter 05's release-gate contract is a property of the *progressive delivery controller*; the controller's extensibility determines what the release-gate can read. Self-host Flagger / Argo Rollouts exposes the controller's webhooks and the custom-metric adapters directly; vendor services expose the vendor's abstraction.
- **Specific risks to add.** The buy-side risk that custom AI-specific release-gate signals (per-model-version latency delta, judge-confidence calibration drift, provider-fallback health) are not first-class in the vendor's abstraction. The self-host risk that the controller's own reliability becomes an SRE-team on-call surface.
- **Migration path specialisation.** The cutover phase (§5.3) is per-service and starts from the service with the simplest release-gate contract; services with multi-signal gates migrate last.

## 9. Anti-patterns the RFC format prevents

- **The "vendor opinion piece."** An argument framed as "we should move to Datadog" with no §2.1 context, no §2.2 options, no §2.3 TCO, and no §2.5 risks. The RFC format forces the comparison to be written down.
- **The "self-host religion."** A refusal to even run the TCO on a vendor option because the function "doesn't use proprietary tools." The §2.2 option enumeration is explicit; refusing to list an option is a reviewable act.
- **The "decision by exhaustion."** A migration that started informally, has run for eight months, and is now too far along to reverse. The §5 phased migration path with explicit rollback at each phase makes "we've already spent three months on this" an auditable statement the function reviews before each next phase.
- **The "silent lock-in."** A vendor move that proceeded without a §4 risk register. Two years later, the contract renews on a 30% uplift and the function discovers the exit cost is six months of engineering time. The risk register surfaces this at decision time.
- **The "no re-evaluation date."** A decision made once and never revisited. The §2.7 reviewer list includes the next build-vs-buy review date (chapter 01 §2.5); every build-vs-buy decision is explicitly expiring.

## Summary

- A reliability-engineering RFC is the Git-tracked, long-form artifact through which the function makes the SRE-tooling stack build-vs-buy decisions. It is the output of the annual programme review (chapter 01 §2.5) and the input to multi-month migrations.
- Three components concentrate the largest decisions: observability (Prometheus / Grafana / OTel vs. Datadog / New Relic / Chronosphere), chaos (Chaos Mesh / LitmusChaos vs. Gremlin / cloud-provider chaos), and progressive delivery (Flagger / Argo Rollouts vs. cloud-provider managed service).
- The RFC's seven sections — context, options, TCO, operational qualities, risk register, migration path, recommendation — are each defensible on their own; a reviewer who stops halfway through has a coherent view.
- The TCO model runs on a three-year horizon, is reproducible by the reader, and includes migration and exit costs, not only run-rate.
- Five decision axes carry more weight than any single dollar figure: cost at scale, control over the signal surface, portability and reversibility, on-call burden, and strategic positioning. The RFC names which axis dominated.
- The risk register has per-entry statements, likelihood × impact, owner, mitigation, trigger, and escalation path. It is the piece reviewers remember and the piece that protects the function when a risk materialises.
- The migration path is five phases (prepare, parallel run, cutover, decommission, post-migration review) with explicit entry / exit conditions and explicit rollback at each phase.
- The review and signoff loop runs three rings — team, peer, leadership — with written-reply discipline on every blocking comment.
- Declining to buy is a valid recommendation, delivered through the same RFC, and protects the function the next time the question is raised.

Chapter 03 writes the cross-peer interface contracts — the documents that make the §2.5 risk-register's "owner" and "escalation" fields something more than a pointer to a job title.
