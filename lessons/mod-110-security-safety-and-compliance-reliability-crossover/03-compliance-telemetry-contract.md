# The Compliance Telemetry Contract

## Why this chapter exists

A subset of regulatory and compliance obligations imposes a continuous demand on a model-backed service — not an incident-time disclosure, but a steady stream of monitoring data the organisation must maintain, inspect, and (sometimes) be able to produce on request. The EU AI Act's post-market monitoring regime, the FDA's expectation that an AI-enabled medical device remains inside its predetermined change-control envelope, the Federal Reserve's SR 11-7 expectation of ongoing model monitoring — each of these implies that *somebody* is emitting the telemetry that evidences continuous compliance, and *somebody* is reading it.

`mod-108` chapter 06 covered the incident-time handoff to legal / compliance. This chapter covers the steady-state: the telemetry contract the AI SRE owns with the `ai-risk-engineer` peer and legal, the specific signals the SRE function is on the hook for emitting, and the explicit line where the reporting obligation picks up on the peer / legal side. The AI SRE does not file; the AI SRE **ensures the signals legal needs to file accurately are present, correct, retained, and queryable** without taking on the reporting role.

The failure mode this chapter prevents: the AI SRE function turns its operational telemetry into a compliance-grade deliverable on demand, at 2 AM, under a regulator's deadline — because nobody wrote the contract when the service was built. The correct shape is the contract, the contract is written out of incident, and the telemetry is a by-product of the observability stack `mod-104` already runs.

## 1. The premise, repeated from `mod-108` chapter 06 §1

Three principles anchor the role at the compliance interface, repeated here because the next sections assume them:

- **Defer formal reporting.** The AI SRE does not file post-market monitoring reports, submit PCCP triggered-change notifications, or sign SR 11-7 management attestations. Those are the `ai-risk-engineer` / legal / MRM functions' outputs.
- **Produce accurate, retained telemetry.** The telemetry is the role's product. If the dashboard's numbers do not match the reality of the running service, legal files inaccurate reports.
- **Treat the telemetry contract as an SLI.** Telemetry freshness, completeness, and retention are themselves SLIs. A compliance telemetry contract whose availability SLO is undocumented is a contract waiting to break.

Those three are the vocabulary. Sections 2–4 write the compliance frameworks the SRE function most often touches and the specific telemetry each demands.

## 2. EU AI Act — the post-market monitoring half of the contract

Regulation (EU) 2024/1689 establishes a post-market monitoring regime for providers of high-risk AI systems on the EU market. Article 72 — post-market monitoring — requires providers to establish and document a post-market monitoring system that **actively and systematically collects, documents, and analyses** relevant data on the system's performance throughout its lifetime, to evaluate continuous compliance with the Regulation. <!-- needs-research: confirm the content and form of the Commission's implementing act under Art. 72(3) that establishes the detailed provisions for the post-market monitoring plan template and the status of its publication at the time of reading -->

Article 73 — the complementary serious-incident reporting duty — was covered in `mod-108` chapter 06 §2.1. Article 72 is the steady-state counterpart: *where the incident surfaces, 72 is the monitoring that was supposed to detect it; where the incident is reportable, 72 is the record of the monitoring system that was in place*.

### 2.1 What Article 72 implies for the AI SRE

The Act does not prescribe metric names. The regulatory shape is: a provider must be able to answer, on request, "what monitoring did you have in place to detect performance drift, safety regressions, or compliance breaches?" and "what did the monitoring data show for the period in question?"

Operationally, this collapses into three AI SRE deliverables:

- **A documented monitoring system.** The system is the `mod-104` observability stack plus the `mod-105` quality signals plus the `mod-108` runbook library — the AI SRE function already runs it. The compliance deliverable is a document that names the components, their coverage, and their retention.
- **A continuous data stream.** The data the monitoring system produces, retained for the duration legal specifies (frequently years for high-risk systems), queryable by `ai-risk-engineer` on its own cadence.
- **A tamper-resistant record.** The record must be defensible under audit. Logs that can be mutated silently, metrics that can be back-dated, dashboards that can be edited in place without history are not defensible.

### 2.2 The signal set

A representative high-risk-system monitoring telemetry inventory. The AI SRE emits each; `ai-risk-engineer` interprets and reports.

| Signal class | Specific signals | Source |
|---|---|---|
| Serving health | TTFT p50/p95/p99, TPOT p50/p95/p99, success-rate, 5xx rate, 429 rate, structured-output conformance rate | `mod-104` |
| Model behaviour | Judge-pipeline quality scores (groundedness, faithfulness, helpfulness) per route, per cohort; refusal rate; safety-classifier output rate per class | `mod-105` and defensive stack from chapter 01 |
| Defensive stack | Per-hop latency histograms (chapter 01 §2); classifier precision / recall on sampled production traffic (owned by security, exposed in telemetry) | Joint with security |
| Fallback behaviour | Fallback engagement rate, fallback MTTR, fallback-hit-rate, provider identity per request | `mod-107` |
| Data handling | Data-residency tag per request, retention bucket per request, tenant / cohort ID per request (hashed) | `mod-104` and platform |
| Drift signals | Judge-score distribution shift vs 30-day baseline; embedding-space drift on retrieval corpus; prompt-template version in use per request | `mod-105` and config |
| Incident lifecycle | Counts of incidents by severity, root-cause category, MTTD, MTTR, action-item completion rate, outstanding fixtures | `mod-108` chapter 05 fixture ledger |
| Governance | Model version, model-card link, change-log entry, approval gate signature per deploy | Platform / release |

The inventory is a contract column. The second column (specific signals) comes from the function's own observability stack; the third column (source) is the module in this track that already defines the signal. Chapter 03 of this module **does not re-author the SLIs**; it names which SLIs are compliance-grade and commits to their retention.

### 2.3 Retention and queryability

The default retention for `mod-104` dashboards is weeks to months, sized for debugging. Compliance retention is years. The AI SRE contract commits to:

- **Short-horizon retention** (hot, fully indexed, dashboard-queryable) sized for operational response (30–90 days typical).
- **Long-horizon retention** (cold, slower, bulk-queryable) sized for the legal retention period. Options include object-locked storage of aggregated time-series, exported Prometheus / Parquet snapshots, a dedicated long-term store (Thanos / Cortex / Mimir / equivalent). The storage is append-only; mutations are not permitted.
- **Lineage.** A record that ties a compliance snapshot ("telemetry state on 2026-11-04") to the specific recording rules, transformation code versions, and source series that produced it. Without lineage, a regulator-facing claim about a historical state is unverifiable.
- **Query SLO.** A commitment that a well-formed query over the long-horizon store returns a bounded result in a bounded time (e.g., "a 24-hour window on any compliance signal returns within 60 seconds from cold"). `ai-risk-engineer` depends on this to meet their own deadlines.

### 2.4 The AI SRE's output artifact

The deliverable the AI SRE signs for Article 72 purposes is a one-page **telemetry attestation** that reads:

> The following signals are emitted continuously by the production monitoring system of the {product / high-risk system name}. They are retained in {short-horizon store} for {N} days and in {long-horizon store} for {M} years under {policy reference}. The monitoring system's availability is targeted at {SLO}, measured and reported quarterly. Access is available to the roles enumerated below for the purposes of compliance evaluation.

Attached to the attestation: the §2.2 inventory table with current coverage status, the retention schedule, the query SLO, the role / access matrix. `ai-risk-engineer` or legal consumes the attestation as the evidentiary input to the Article 72 narrative; they do not re-derive the inventory. <!-- needs-research: confirm whether the Commission implementing act under Art. 72(3), when published, imposes a specific format for the post-market monitoring plan that supersedes the free-form attestation shape -->

### 2.5 The fall-backs the Act is read with

The Act interacts with existing regimes the role should be aware of as *context*, not as additional reporting duties:

- **GDPR.** Processing of personal data by a monitoring system is itself a data-protection surface. The telemetry contract's data-handling columns (residency, retention) carry GDPR-colour implications. Legal / DPO owns the interpretation.
- **Non-high-risk AI systems** have lighter obligations under the Act. The attestation discipline is still a good hygiene choice; the Act's weight is not.
- **General-purpose AI model providers** (Chapter V of the Act) have a separate duty regime. If the organisation is a provider of a general-purpose AI model in addition to a product built on one, the telemetry contract has two surfaces — the model-provider surface and the system-provider surface. The `ai-risk-engineer` peer navigates which obligations apply. <!-- needs-research: confirm the current status of the Code of Practice for general-purpose AI models under Art. 56 and any specific telemetry expectations it contains -->

## 3. FDA PCCP — the triggered-change half of the contract

For AI-enabled medical devices in the US, the FDA's Predetermined Change Control Plan framework allows manufacturers to pre-specify the modifications they intend to make after clearance without requiring a new submission each time. The PCCP includes a Modification Specification, a Modification Protocol, and an Impact Assessment, per the final guidance the FDA published in December 2024. <!-- needs-research: confirm the final guidance document title, publication date, and whether any updates have superseded it at the time of reading; the current document is "Marketing Submission Recommendations for a Predetermined Change Control Plan for Artificial Intelligence-Enabled Device Software Functions" -->

### 3.1 What PCCP implies for the AI SRE

The AI SRE's touch point is not the submission itself. The function's role is to ensure the production reality of the device software remains inside the modification envelope the PCCP committed to, and to **trigger the triggered-change workflow** when the envelope is approached or breached.

Three conditions produce a triggered-change signal:

- **A planned modification is about to occur.** A model retrain, a prompt-template change, a threshold adjustment, a data-pipeline update that the PCCP covers. The AI SRE emits a pre-change telemetry snapshot.
- **A modification occurred that was not in the PCCP's Modification Specification.** An operational change (a hotfix, a config tweak, an emergency revert) that the PCCP does not pre-specify. The AI SRE flags this to regulatory affairs immediately.
- **Post-change telemetry drifts outside the PCCP's acceptance criteria.** The Impact Assessment sets numeric acceptance criteria; production telemetry drifts past them. The AI SRE's drift alarms fire.

### 3.2 The signal set

PCCP telemetry is a subset of Article 72 telemetry, re-cut for device-specific acceptance criteria:

- **Pre-change state capture.** A snapshot of the signals the Impact Assessment enumerates, with timestamps, model version, data version, and configuration version. The snapshot is retained for the device's full post-market lifecycle.
- **Post-change state capture.** The same snapshot, taken on a cadence the PCCP's Modification Protocol specifies (frequently daily or weekly for the window following a change).
- **Acceptance criterion alarms.** For each numeric criterion in the Impact Assessment, a Prometheus / equivalent alarm that fires when the criterion drifts, on the fast-burn and slow-burn cadences from `mod-103`.
- **Unplanned-change detection.** A hash of the deployed model binary, prompt-template version, configuration, data pipeline version — any change not accompanied by a modification ticket matched to the Modification Specification fires a `rb-unplanned-change-on-pccp-device` runbook.

### 3.3 The handoff pattern

The AI SRE emits; the regulatory affairs function (sometimes called "QA / RA" in the device world) consumes and decides whether an MDR filing, a supplementary submission, or a recall is required. The AI SRE's runsheet includes the regulatory-affairs on-call and the specific trigger wording: "the drift detector on `{criterion}` has crossed the PCCP's acceptance ceiling at `{timestamp}`; please confirm whether this is a reportable event under the current PCCP." The AI SRE does not make the reportability determination; the AI SRE ensures it is made.

### 3.4 What PCCP does not ask the AI SRE to do

- **Interpret the Modification Specification.** The regulatory-affairs function owns the interpretation.
- **Author the Impact Assessment.** The device-software product team and regulatory affairs author it jointly, with the AI SRE as a reviewer for operational feasibility.
- **File the MDR report.** The quality system owns MDR.

These exclusions are explicit in the attestation the AI SRE signs so the function's scope is on the record.

## 4. SR 11-7 — the model-risk half of the contract

The Federal Reserve's SR 11-7 (2011), adopted in parallel by the OCC's 2011-12 bulletin, is the model-risk-management guidance US banks apply to models in production. Its treatment of AI models is interpretive; MRM functions use its framework by analogy for AI. <!-- needs-research: confirm whether an interagency AI-specific supplement to SR 11-7 has been issued at the time of reading; as of early 2026 the long-standing SR 11-7 remains the primary guidance for US banks, interpreted for AI by MRM functions -->

SR 11-7 expects a model inventory, documentation, independent validation, ongoing monitoring, and governance. The AI SRE's role maps to **ongoing monitoring**.

### 4.1 What SR 11-7 implies for the AI SRE

The MRM function maintains the model inventory and the model documentation. Ongoing monitoring is the surface the AI SRE's observability work already produces. The AI SRE's responsibilities here are:

- **Inventory alignment.** For each model the service invokes (self-hosted or provider-hosted), the inventory entry references the SRE-owned telemetry signal set for that model.
- **Monitoring state attestation.** On the MRM function's cadence (typically quarterly for first-line and annually for second-line review), the AI SRE attests to the operational monitoring posture: which models are monitored, with what signals, with what retention, with what acceptance thresholds.
- **Material-deviation notification.** When a model's production behaviour drifts materially — as defined by the model documentation's acceptance envelope, which MRM maintains — the AI SRE's detection pipeline flags the deviation to the model's MRM owner. The deviation itself is quantified in the function's telemetry; the materiality determination is MRM's.

### 4.2 The signal set

SR 11-7 telemetry is close to Article 72 telemetry for financial-services workloads, plus the model-governance extension:

- **Model identity per request.** Which model (name, version, config) served which request. Non-trivial for multi-model routing, degraded-mode routing, and provider fallback.
- **Model-specific performance signals.** Per model (not per route), the quality signals and the serving signals.
- **Data-distribution signals.** Input-distribution monitoring where the MRM documentation specifies an expected data distribution. Drift against the baseline is a standing alarm.
- **Challenger-model signals.** Where MRM maintains a challenger model, the challenger's output and performance are also telemetered, by the same pipeline.
- **Model-change audit trail.** Every change to a production model's identity, configuration, or data is an audit-logged event the MRM function can query.

### 4.3 What SR 11-7 does not ask the AI SRE to do

- **Maintain the model inventory.** MRM maintains the inventory. The AI SRE provides inventory-aligned telemetry.
- **Conduct independent validation.** Independent validation is an MRM team, by regulatory design. The AI SRE is first-line; validation is second-line.
- **Author the model documentation.** The model owner (data science / ML engineering) authors documentation. The AI SRE reviews for operational feasibility.
- **Decide materiality.** MRM determines materiality; the AI SRE flags candidates.

## 5. The contract, generalised

Across Article 72, PCCP, and SR 11-7, the AI SRE's compliance role has one shape:

1. **Enumerate the signals the service produces** that are compliance-relevant. The §2.2 inventory is the template; the specific table is per product.
2. **Commit retention, lineage, and query SLO** for each signal class.
3. **Commit the monitoring-system availability SLO.** The monitoring system is itself a load-bearing part of the compliance posture; its downtime is a compliance event.
4. **Name the consumer peer** for each compliance surface. `ai-risk-engineer` is the primary; legal / MRM / regulatory affairs are the frameworks-specific extensions.
5. **Name the triggers that obligate handoff.** Chapter 06 of `mod-108` enumerates the incident-time triggers; this chapter adds the steady-state triggers (drift alarms crossing acceptance criteria, unplanned-change detection, telemetry-system availability breach).
6. **Attest in writing.** The attestation is the function's compliance product. It is signed, dated, versioned, and reviewed on a cadence matched to the regulatory surface.

The playbook is one artifact: the **telemetry contract**, signed by the AI SRE function lead, `ai-risk-engineer`, legal where appropriate, and the model / product owner. Exercise 03 of this module authors the Article 72 slice of this contract for a representative service.

## 6. The monitoring-system SLO as compliance surface

The non-obvious load-bearing part of this chapter: the monitoring system has its own SLO, and when it burns, it is a compliance surface, not only a reliability one.

Three specific failure modes have compliance implications:

- **Telemetry pipeline gaps.** A Prometheus scrape failed; a retention tier lost a day; a recording rule mis-emitted. For a non-compliance observability stack, this is operationally annoying. For a compliance observability stack, a gap in the series is a gap in the record the organisation must defend.
- **Dashboard definition drift.** A dashboard the compliance function uses is edited, the edit is not versioned, a historical screenshot differs from the current rendering. The record is unstable.
- **Access control drift.** Access to the compliance telemetry widens (a new engineer added, a new service account authorised) without an audit trail. The chain-of-custody discipline from chapter 05 of this module applies here too, continuously.

The AI SRE commits to three practices against these:

- **Pipeline-health SLO.** A monitored SLO on "telemetry delivered end-to-end within N minutes of event" for the compliance signal set. Burn on this SLO is a compliance-grade incident with its own runbook (`rb-compliance-telemetry-gap`).
- **Dashboard-as-code.** Compliance-facing dashboards live in version control, are reviewed on change, and have a full edit history. Freehand dashboards are not compliance-grade.
- **Access audit.** Access to compliance telemetry is RBAC'd, logged, and periodically reviewed (quarterly). The audit log itself is retained under the compliance retention schedule.

The function's quarterly reliability review (and the `mod-114` cross-functional cadence) reads the compliance-telemetry SLO alongside the serving SLOs.

## 7. The anti-patterns at the compliance surface

Observed on functions new to the compliance crossover:

- **"We'll copy the Article 72 inventory from a template."** Templates are useful; the inventory the function commits to must be *the actual signals the function produces*. A template with aspirational entries produces audit discoveries months later.
- **"We'll retain everything forever."** Retention is not free. Over-retention has privacy, cost, and legal-discovery implications; legal sets the retention windows, not engineering intuition.
- **"The attestation is a legal document; we'll let legal write it."** The attestation's technical content is the AI SRE's. Legal cannot author the signal list, retention schedule, or SLO numbers correctly without the function's input.
- **"The MRM reviewer is annoying; we'll give them a dashboard URL and move on."** MRM reviewers are consumers; a well-made inventory table, a signed attestation, and an SLA for query response is a much lower-friction interface than ad-hoc dashboard questions.
- **"PCCP triggered-change is a device-team problem."** On any team that ships a cleared device software function, the AI SRE is on the handoff bridge. The function's runsheet must include the regulatory-affairs on-call.

## 8. The AI SRE's compliance-telemetry worksheet

Use this during initial authoring and during the quarterly re-sign:

1. **List the compliance surfaces** that apply to each product the function serves (EU AI Act, FDA PCCP, SR 11-7, state AI laws, sectoral regulations, customer contracts referencing NIST AI RMF / ISO/IEC 42001).
2. **For each surface, name the consumer peer** — `ai-risk-engineer`, legal, MRM, regulatory affairs — and the regulatory cadence the peer operates on.
3. **Fill the §2.2 inventory table** for each product, pinning every signal to a `mod-104` / `mod-105` / `mod-107` / `mod-108` source. Flag any that are aspirational (not yet produced) as gaps.
4. **Sign the retention, lineage, and query SLO** per signal class. If the function cannot meet a required retention, raise it in the quarterly review before the compliance surface demands it.
5. **Author the pipeline-health SLO** for the compliance telemetry pipeline and wire the `rb-compliance-telemetry-gap` runbook.
6. **Author the dashboard-as-code commitment** and the quarterly access audit.
7. **Sign the attestation** jointly with `ai-risk-engineer` and legal where appropriate. The attestation goes to the compliance archive and is reviewed on the surface's cadence.
8. **Rehearse the triggered-change workflow** (PCCP) and the serious-incident handoff (Art. 73) on a game-day cadence from `mod-109` chapter 05 §4.

## Summary

- **The AI SRE owns continuous compliance telemetry; the AI SRE does not file.** The output is a signed attestation and a retained data stream.
- **Three frameworks cover most of the surface** — EU AI Act Article 72 (post-market monitoring), FDA PCCP (triggered-change), SR 11-7 (ongoing model monitoring). Each has a telemetry subset; each has a consumer peer.
- **The telemetry inventory is a table of signals pinned to existing SRE-owned sources.** The function does not re-author SLIs for compliance; it names which SLIs are compliance-grade.
- **Retention is longer than operational retention; lineage and query SLO are explicit.** Append-only storage, versioned recording rules, bounded query response time are the baseline.
- **The monitoring system has its own SLO**, with a specific runbook (`rb-compliance-telemetry-gap`) when it burns. A gap in the compliance record is itself a compliance surface.
- **The attestation is the AI SRE's compliance product.** Signed, dated, versioned, reviewed on the surface's cadence; co-signed with `ai-risk-engineer` and legal.
- **The anti-patterns at this surface are preventable** with a worksheet, a template with honest gaps, and a joint cadence with `ai-risk-engineer` and legal.

Chapter 04 reconciles the compliance gate the attestation defends with the safety gate the responsible-AI programs run in parallel.
