# Legal, Compliance, and Regulator-Facing Handoff for AI-Safety Incidents

## Why this chapter exists

A subset of AI incidents stops being an engineering problem the moment they resolve, because the organisation is obligated to tell someone outside it what happened. The regulatory surface for AI systems is growing: the EU AI Act creates post-market monitoring and serious-incident reporting duties for providers of high-risk AI systems; the US FDA's device framework treats changes to AI-enabled medical device software under predetermined change-control plans; the Federal Reserve's SR 11-7 guidance frames model-risk management for banks and shapes expectations in US financial services.

The AI SRE does not own the formal reporting. Legal, compliance, the `ai-risk-engineer` function, and executive leadership own it. What the AI SRE owns is **the technical narrative that those owners need to formally report accurately and on time**: the timeline, the scope, the detection signals, the mitigation steps, the user impact, the data classification, the artifact preservation. A handoff that gives legal the wrong numbers, incomplete timeline, or lost artifacts is a handoff that makes the organisation's reporting less accurate, more fragile, and more likely to trigger supplementary inquiries.

This chapter codifies the handoff. It names the three regulatory frameworks the role is most likely to touch, states the triggers that obligate handoff, describes the artifacts the AI SRE produces, and draws the line between "SRE prepares" and "legal reports." It is not a substitute for legal advice.

## 1. The handoff premise

Three principles organise the role's work at the regulatory interface:

- **Defer formal communication.** The AI SRE does not call a regulator, does not file a report, does not sign a disclosure, does not communicate with a competent authority under their own name. Those are legal / compliance / executive acts.
- **Produce accurate technical narrative.** The output the role owns is the technical narrative: the specific signal, the specific timestamp, the specific scope, the specific mitigation, and the specific artifact. Ambiguity that the engineer leaves in the narrative becomes risk legal must manage.
- **Preserve everything.** Discovery horizons in regulatory contexts are long. The retention policy for AI-incident artifacts is set with legal and may extend years after resolution; the AI SRE respects the retention end-to-end, even on artifacts the engineering team would otherwise prune.

The chapter's remaining sections turn these principles into concrete procedure.

## 2. The three regulatory frameworks most likely to apply

Each is sketched here to a level sufficient for the AI SRE to know *when* to hand off. Depth of interpretation lives with legal and the `ai-risk-engineer` function.

### 2.1 EU AI Act — post-market monitoring (Art. 72) and serious-incident reporting (Art. 73)

Regulation (EU) 2024/1689 (the EU AI Act) imposes obligations on providers of high-risk AI systems placed on the EU market. Two obligations directly affect incident response:

- **Post-market monitoring (Article 72).** Providers must maintain a post-market monitoring system that actively and systematically collects, documents, and analyses data on the performance of the AI system throughout its lifetime, to assess continuous compliance with the Act's requirements. Operationally, this means the telemetry the AI SRE already produces for `mod-104` is also a compliance deliverable: evidence that monitoring exists and is operating. <!-- needs-research: confirm the current Commission implementing act templates for the post-market monitoring plan under Art. 72 and verify effective dates for the various system classes -->
- **Serious-incident reporting (Article 73).** Providers of high-risk AI systems must report serious incidents to the market-surveillance authorities of the Member States in which the incident occurred. "Serious incident" is defined in the Act and is substantially tighter than "any availability regression"; it covers events causing or substantially contributing to death or serious harm to health, serious and irreversible disruption of critical infrastructure, serious violations of fundamental-rights obligations, or serious damage to property or the environment. Reporting windows are short (counted in days rather than weeks) and depend on the category of harm. <!-- needs-research: confirm the exact reporting windows under Art. 73(2) as currently in force (initial report, intermediate / definitive updates) and the list of competent authorities per Member State -->

The AI SRE's touch point is: when the retrospective's primary cause and impact line up with "serious incident" as defined, legal is engaged immediately and the technical narrative is prepared to the deadline-shortest window. The determination of whether the threshold is crossed is a legal call; the AI SRE's role is to provide the facts legal uses to make the call.

### 2.2 US FDA — Predetermined Change Control Plan (PCCP) for AI/ML-enabled device software functions

For AI-enabled medical devices in the US, the FDA's PCCP framework allows manufacturers to pre-specify the kinds of modifications they will make to a device after clearance without requiring a new submission each time. The PCCP includes:

- A **modification specification** — the planned changes, expressed in detail.
- A **modification protocol** — the methods used to develop, verify, and validate each modification.
- An **impact assessment** — the planned analysis of whether the modifications maintain safety and effectiveness.

Operationally, if an incident surfaces a modification that was not covered by the PCCP, or if post-market data suggests the device's performance drifted outside the PCCP's stated bounds, the manufacturer may need to notify FDA, file a supplementary submission, or trigger a recall. For an AI SRE working on an FDA-regulated product, the handoff triggers are:

- Any observed deviation from the device's stated performance envelope.
- Any modification that was executed outside the PCCP's modification specification.
- Any serious adverse event associated with the device output.

Depth and the specific Medical Device Reporting (MDR) obligations live with the quality / regulatory affairs function; the AI SRE hands off at the trigger with the technical narrative. <!-- needs-research: confirm the current FDA final guidance publication (released December 2024) on PCCP for AI/ML-enabled device software functions and the specific modification-protocol expectations -->

### 2.3 US financial services — SR 11-7 model risk management

The Federal Reserve's SR 11-7 (2011), adopted in parallel by the OCC (OCC 2011-12), is the long-standing model-risk-management guidance for US banks. Its applicability to AI is interpretive — the guidance pre-dates the modern generative wave — but banks apply its framework to AI models by analogy. SR 11-7 expects:

- A model inventory that records every model in production use.
- Model documentation covering purpose, assumptions, data, methodology, and limitations.
- Independent validation.
- Ongoing monitoring of model performance.
- Explicit governance of model-risk decisions.

Operationally, when an AI incident touches a model in SR 11-7 scope, the handoff triggers include:

- Material model performance deviation from documented expectations.
- An incident that would materially change the model's documented risk profile.
- Any event an independent validation or internal audit team is entitled to see.

The institution's MRM (Model Risk Management) function is the consumer of the narrative; the AI SRE's output is the model inventory entry, the monitoring record, and the incident technical narrative at a specificity the MRM reviewer can consume. <!-- needs-research: cite the exact current OCC/FRB/FDIC guidance text and clarify whether SR 11-7 or its successors (e.g., any post-2023 interagency AI-focused supplement) is currently authoritative at the time of reading -->

### 2.4 Other frameworks the role may encounter

- **NIST AI RMF and the Generative AI Profile (NIST-AI-600-1).** A voluntary framework. Many organisations use it internally as the risk vocabulary and some customer contracts require alignment. It is not a reporting regime but informs the shape of internal documentation and categories. Grounded in `mod-105` chapter 02.
- **ISO/IEC 42001:2023.** The AI management-system standard. If the organisation is certified, incidents may trigger a non-conformance that must be documented in the AI management system. <!-- needs-research: confirm ISO/IEC 42001's incident-related clauses relevant to AI SRE handoff -->
- **State and sectoral regulators.** New York DFS cybersecurity rule, California CCPA / CPRA incident provisions if user data was involved, Colorado AI Act (SB 24-205), and sectoral regulators in healthcare, education, employment, and lending. <!-- needs-research: confirm the current list and incident-reporting triggers for state-level AI laws that have passed between 2024 and today -->
- **Existing incident-reporting duties that AI incidents may trip.** Data-breach notification laws (GDPR Art. 33, state breach-notification laws in the US) if an AI incident involved unauthorised disclosure; critical-infrastructure incident-reporting regimes if the AI system is in scope (CIRCIA in the US). <!-- needs-research: confirm the CIRCIA final-rule applicability and reporting windows at the time of reading -->

The AI SRE's working knowledge needs to be *enough to know when to page legal*, not enough to interpret the rule.

## 3. The triggers that obligate handoff

The following triggers are pre-committed in the AI SRE function's doctrine (chapter 02 §6 pre-commits the operational decision tree). They are the AI SRE's "page legal / compliance now" conditions, read independently of severity labels.

### 3.1 Safety-trigger conditions

- **`rb-toxic-output-blast` firing on a user-facing route.** Any sustained firing — or any single egregious user-reported event.
- **A model output that materially contributed to a health, safety, or financial harm.** Reported by a user, support, or detected via a classifier / monitor.
- **A regulated-domain route emitting an incorrect or unsafe answer** (medical, legal, financial advice depending on jurisdiction and contract).
- **A suspected or confirmed jailbreak that defeats a safety control** in production.

### 3.2 Compliance-trigger conditions

- **A data-residency violation** (EU customer data served through a non-EU route against contract).
- **A ZDR / no-training commitment potentially violated** (customer data potentially reached a provider path not covered by ZDR).
- **A potential exfiltration of user-interaction data** through a provider or vendor path.
- **Loss of audit-log integrity** on a regulated path.

### 3.3 Reporting-trigger conditions

- **An incident that may meet the EU AI Act Art. 73 "serious incident" threshold** (serious harm, critical-infrastructure disruption, fundamental-rights violation, serious property or environmental damage). Legal makes the final determination.
- **An incident on an FDA-cleared AI device that exceeds the PCCP's stated performance envelope** or involves a serious adverse event.
- **An SR 11-7-scope model regression** that materially shifts the model's risk profile relative to its documented posture.
- **A regulator inquiry** opened in connection with the incident.

Any trigger firing does not mean reporting *will* occur — it means legal engages and decides. The AI SRE's duty is engagement, not the decision.

## 4. The artifact package the role produces

When a handoff trigger fires, the AI SRE prepares a structured package. The package is similar to (and often derived from) the internal post-mortem (chapter 03 §2), with legal-focused specificity on dates, scope, and evidence integrity.

### 4.1 The technical narrative

A clean prose document, 1–3 pages, written in past tense, actor-role voice, with observation and inference separated. Contents:

- **Incident identifiers** (internal IDs, channel references, model version IDs, configuration hashes).
- **Chronology.** UTC timestamps for first signal, declare, mitigation steps, stable, resolve.
- **Scope.** Specific user cohorts (counts if known), regions, product surfaces, data classes touched.
- **Technical root cause** classified in the internal taxonomy (chapter 03 §3) with the specific instance named.
- **Detection signal** that fired, with the system of record.
- **Mitigation** taken, with timestamps and effect.
- **Residual exposure** after mitigation.

### 4.2 The impact assessment

Quantitative where possible:

- **Number of affected users / requests / sessions.**
- **Duration of exposure.**
- **Categories of impact** — availability (SLO burn), quality (eval regression), safety (classifier events), privacy (data exposure), financial (cost or contractual).
- **Known or suspected third-party dependency involvement.**

### 4.3 The artifact inventory

A list of the preserved artifacts with their locations and retention:

- **Timeline source** (incident channel export, pinned timeline, scribe notes).
- **Telemetry snapshots** (dashboards, Grafana panels pinned to the incident window, trace exports).
- **Sample outputs** (if customer-facing content is at issue, exported to a controlled-access store, never the open incident channel).
- **Request / response logs** for the affected path within the window.
- **Model artifacts** — weights version, prompt version, config version — pinned to the incident.
- **Vendor correspondence** — support ticket IDs, provider status-page captures, vendor incident IDs.

Every artifact entry has a path, an access-control note, and a retention date. Nothing on this list is deleted without legal's clearance.

### 4.4 The classification map

For the regulatory-reporting consumer, a one-table map from the technical fact base to the framework's categories. Example for an Art. 73-trigger evaluation:

| Art. 73 element | Fact from the narrative |
|---|---|
| Date and place of incident | 2026-11-04, EU-region traffic on the summariser route |
| Serious harm category | None identified; health and safety reviews requested |
| Information identifying the system | Model version ID, product surface, intended purpose |
| Any corrective action taken | Reverted to prior model version at 14:12 UTC |

The table is a drafting aid for legal, not a filing. Legal rewrites into the actual filing shape.

### 4.5 The data-handling attestation

A brief attestation describing the handling of any customer data during the incident:

- **What data was processed** during the affected window, by which systems.
- **What data was retained**, where, and under what access control, for the incident analysis.
- **What data was shared** with any party outside the organisation (support, vendor, auditor) and under what legal basis.

The attestation is factual, not interpretive. Legal is the consumer.

## 5. Preserving artifacts

Preservation is a hard discipline because the day-to-day engineering culture is "delete logs older than N days to control storage." The AI SRE function overrides on incident artifacts.

- **Legal hold on identified artifacts.** When a trigger fires, legal issues a hold with the specific artifact IDs; those are explicitly exempt from the regular deletion policy. The AI SRE function maintains a legal-hold list.
- **Immutable storage.** Preserved artifacts land in append-only / object-locked storage. Retention is legal's call, not the engineering team's.
- **Access control.** Preserved artifacts are accessible only to named roles with a documented need. The incident channel's open visibility is not a substitute; sample content is not pasted into the open channel in the first place (chapter 02 §5.3 is explicit).
- **Chain of custody.** For artifacts that may be presented externally, access logs are preserved. "Who read what, when" is part of the record.

The preservation rules are the single most-often-missed step by engineering teams entering the regulatory surface for the first time. The exercise for this chapter asks the learner to draft the organisation's version.

## 6. The communication protocol with legal and compliance

How the AI SRE talks to legal and compliance matters as much as the content. Four conventions the function observes:

- **Named routing.** Legal / compliance teams have an incident contact; the AI SRE pages that contact at trigger, not an individual by name. The contact is on a rotation visible in the IR tooling.
- **Facts first, implications later.** The initial brief is five bullets: what happened, when, scope, detection, mitigation. Implications are discussed after the fact base is clear.
- **Written primary, verbal secondary.** The written record is the source of truth. Verbal briefings supplement; they do not replace.
- **Clarify unknowns explicitly.** "We do not yet know whether EU customer data was affected" is better than "we believe EU customers were not affected." Overstating certainty is a professional failure in this register.

Legal and compliance in return commit to:

- **Responding within the AI SRE's handoff window** — defined in the function's doctrine and matched to the applicable regulatory deadline.
- **Making the determination** on whether a reportable threshold is crossed and communicating back.
- **Owning the external communication** end-to-end.

## 7. The post-handoff follow-through

After the handoff lands, the AI SRE function does not disengage. Three follow-through practices:

### 7.1 Co-authored retro with legal awareness

The post-mortem (chapter 03) proceeds normally, with legal as a review step before publication. The retro's language is factual and avoids speculative framing that would complicate a formal filing. The fixture back-feed (chapter 05) proceeds; some fixtures may themselves be regulatory-required ("we will add the following monitor"), and those become internal commitments the organisation stands behind.

### 7.2 Regulator-facing narrative preparation

If legal determines a filing will occur, the AI SRE supports:

- Technical review of the draft filing for factual accuracy (legal owns the submission; the engineer reviews the facts).
- Preparation of any supplementary technical annex the filing requires.
- Readiness for follow-up questions from the competent authority through legal.

### 7.3 Reporting the patterns forward

Over time the function tracks which incident classes have led to legal / regulatory handoff. The function's leadership uses this to prioritise: if a particular surface produces repeated regulator-facing events, the preventive fixtures for that surface are prioritised, and `mod-110`'s security-safety-compliance module and `mod-111`'s release-engineering module may receive input from this module's experience.

## 8. The AI SRE's regulatory-handoff worksheet

1. **Know the triggers** in §3 by rote. Decisions at the 3 AM moment are not the time for case-by-case interpretation.
2. **Page legal / compliance immediately at trigger.** Do not wait for the engineering diagnosis.
3. **Preserve artifacts** under the function's legal-hold procedure as soon as the trigger fires; nothing is pruned until legal clears.
4. **Draft the technical narrative** to the template in §4 as the incident resolves. The window for first-draft is hours, not days, for serious-incident-shaped events.
5. **Produce the artifact inventory** with paths, access controls, and retention dates.
6. **Produce the classification map** against the applicable framework.
7. **Hand off to legal / `ai-risk-engineer`** with the written package. Verbal follow-up supplements, does not replace.
8. **Support the filing** through technical review; do not contact the regulator directly.
9. **Execute the post-mortem** with legal review before publication.
10. **Track the pattern** forward for future prevention priorities.

## Summary

- **The AI SRE prepares; others formally report.** The role's output is an accurate, complete technical narrative. The role does not file, call, or sign.
- **Three frameworks are the most likely to apply** — the EU AI Act (Art. 72 post-market monitoring and Art. 73 serious-incident reporting), the FDA PCCP framework for AI-enabled medical devices, and SR 11-7 for US bank models. Others (NIST AI RMF, ISO/IEC 42001, state AI laws, breach-notification laws) may apply.
- **Triggers are pre-committed.** Safety, compliance, and reporting triggers are authored out-of-incident and consulted in-incident.
- **The artifact package is structured.** Technical narrative, impact assessment, artifact inventory, classification map, data-handling attestation.
- **Preservation is a hard discipline.** Legal hold, immutable storage, access control, chain of custody.
- **Written communication is primary.** Clarify unknowns; defer the determination to legal.
- **Follow through post-handoff.** Co-author the retro with legal awareness, support the filing, track the patterns forward.

Chapter 07 takes the full incident-response surface this module has covered — runbooks, command, retrospectives, back-feed, regulatory handoff — and draws the peer boundaries explicitly, so the AI SRE's authored output composes cleanly with the peers who own each of the surrounding surfaces.
