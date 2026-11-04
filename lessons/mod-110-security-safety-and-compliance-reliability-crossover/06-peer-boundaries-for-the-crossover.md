# Peer Boundaries for the Security / Safety / Compliance Crossover

## Why this chapter exists

The crossover surface is seductive. The AI SRE has read the OWASP LLM Top 10 and NIST AI 100-2; the AI SRE has an opinion on the classifier deploy pipeline; the AI SRE can draft the first paragraph of a regulator-facing filing and wants to help. The function's output quality on all three surfaces is improved by the AI SRE's involvement. The function's output quality on each surface is **degraded** if the AI SRE takes ownership.

Chapters 01–05 wrote what the AI SRE *does* on the crossover: own the reliability dimension of defensive controls, co-author joint post-mortems, operate the compliance telemetry contract, reconcile the reliability gate with the safety gate, operate the incident-sensitive data discipline. This chapter closes the module by writing what the AI SRE **does not do** — which peers own what, which artifacts are jointly owned on which pattern, and which anti-patterns at the boundary the function recognises and refuses.

The chapter mirrors the shape of `mod-107` chapter 07, `mod-108` chapter 07, and `mod-109` chapter 06. Where those modules wrote the vendor, incident-response, and chaos boundaries, this chapter writes the security, safety, and compliance boundaries, with specific attention to where the incident-adjacent and regulator-adjacent surfaces end.

## 1. The peers on this surface

The crossover involves more peers than most modules on this track. Each is sketched to the depth needed to recognise the handoff.

### 1.1 `ai-infra-security` peer (level 35)

- **What they own.** Adversarial-defence methodology: the attack catalogue (OWASP LLM Top 10, NIST AI 100-2, MITRE ATLAS, private catalogues), the classifier portfolio (prompt-injection classifier, PII filter, moderation classifier, policy engine), the red-team program, model-extraction defences, agent-tool authorisation design, supply-chain-security for AI artifacts, secret-handling in prompts / completions, DPIAs on AI-specific processing paths.
- **What they produce.** Defensive controls (as classifiers, rules, rate shapers, policy rules), the attack test suite, red-team reports, security runbooks, security dashboards (precision / recall / coverage / time-to-detection).
- **What they do not own.** Serving SLOs, the latency budget the defensive controls operate within, the reliability contract the controls must respect, the chaos-side verification of fallback behaviour, the compliance-telemetry inventory.

### 1.2 `ai-risk-engineer` peer

- **What they own.** The AI risk management framework for the organisation — the risk inventory, the risk register, the risk-tiering methodology, the risk-to-mitigation mapping, the control catalogue (as a management-of-risk artifact, distinct from the engineering-of-controls artifact the security peer owns). The framework the role maintains is often grounded in NIST AI RMF, ISO/IEC 42001, and the organisation's own adaptation. The role coordinates with legal on regulator-facing output.
- **What they produce.** Risk registers, risk assessments, model cards, model-risk reports, pre-deployment risk reviews, compliance-program design, management-of-risk attestations, the formal outputs that satisfy a regulator's "describe your risk management system" question.
- **What they do not own.** The operational monitoring that evidences the controls are in effect (that is the AI SRE), the engineering of the defensive controls themselves (that is the security peer), the serving reliability contract (that is the AI SRE).

### 1.3 Legal / compliance function

- **What they own.** Regulator-facing output end to end. Reportability determinations (is this incident a serious incident under Art. 73? does this modification require an FDA supplementary submission? does this performance drift materially shift an SR 11-7 model's risk profile?), formal reports, disclosure decisions, retention determinations, legal-hold decisions, contract-level commitments, response to a regulator inquiry.
- **What they produce.** Formal filings, disclosures, retention policies, legal-hold orders, counsel's opinions on specific questions.
- **What they do not own.** The operational telemetry the filings depend on, the incident-time data-handling discipline, the engineering of defensive controls or monitoring, the reliability contract.

### 1.4 Safety / policy / responsible-AI peer

- **What they own.** The organisation's safety commitments, the safety-tiering framework (where the organisation operates one), the safety-eval suite, the policy committee, the review of capability-level changes, the relationship between the model's capability posture and deployment commitments. On many organisations this function and `ai-infra-security` overlap operationally; naming the function distinctly keeps the responsibilities clear.
- **What they produce.** Safety evaluations, safety-tier classifications, policy-committee verdicts, public safety commitments (RSP-equivalent documents), joint release checklists with the AI SRE.
- **What they do not own.** The operational mitigations themselves (those are shared or AI SRE-owned per chapter 04 of this module), the reliability contract, the compliance-telemetry inventory.

### 1.5 SecOps / IR function

- **What they own.** Security incident response end to end — the SIEM, the security-incident runbooks, the chain-of-custody discipline for security-flagged artifacts, the forensic tooling, the external communication for security incidents (where distinct from the regulator-facing path). On many organisations this function is separate from `ai-infra-security`; the SRE function interoperates with both.
- **What they produce.** Security-incident timelines, forensic reports, chain-of-custody records, security-incident communications.
- **What they do not own.** AI-specific detection content, the reliability contract on defensive controls, the AI-specific incident taxonomy.

### 1.6 MRM (model risk management) / regulatory affairs / quality system

- **What they own.** Framework-specific functions the AI SRE's function interacts with per chapter 03 of this module. MRM in US financial services; regulatory affairs and the quality system in FDA-cleared device software; sector-specific reviewers in healthcare, insurance, lending.
- **What they produce.** The formal filings their regulators expect, the internal certifications their frameworks require, the independent-validation reports their frameworks mandate.
- **What they do not own.** Everything the AI SRE owns.

## 2. The AI SRE's irreducible crossover surface

Re-stating the function's deliverables on this crossover, now that the peers are explicit:

- **The guardrail latency-budget contract** per chapter 01 of this module.
- **The per-hop SLI emission and SLO burn-rate alarm set** on defensive controls.
- **The joint post-mortem template and workflow** per chapter 02.
- **The compliance-telemetry inventory, retention commitment, and attestation** per chapter 03.
- **The reliability gate and joint release checklist** per chapter 04.
- **The incident-sensitive data discipline and named-store operation** per chapter 05.
- **The runbook library entries** for `rb-guardrail-hop-latency-breach`, `rb-pii-filter-slowpath-storm`, `rb-model-extraction-shaper-induced-tail`, `rb-adversarial-input-induced-timeout`, `rb-policy-engine-reload-stall`, `rb-compliance-telemetry-gap`, `rb-cross-surface-corpus-event` (as added to `mod-108` chapter 01's library).

Everything above is wholly within the AI SRE function. None of it is co-owned on implementation; several items are jointly consumed.

## 3. The shared artifacts and their ownership shape

Three ownership shapes recur across the crossover. Being explicit about the shape prevents one-sided drift.

### 3.1 AI SRE drafts / peer approves

The AI SRE authors the artifact; the peer reviews and signs. Examples:

- **The guardrail latency-budget contract.** AI SRE drafts, `ai-infra-security` reviews and co-signs (chapter 01 §2.4).
- **The compliance telemetry attestation.** AI SRE drafts the technical content, `ai-risk-engineer` reviews, legal reviews.
- **The joint dashboard for operational mitigations** (chapter 04 §5.4). AI SRE authors the dashboard definitions in code, safety peer reviews, both sign.

### 3.2 Peer drafts / AI SRE reviews

The peer authors the artifact; the AI SRE reviews for operational feasibility and co-signs. Examples:

- **The defensive-control roadmap.** `ai-infra-security` authors; AI SRE reviews for latency-budget feasibility and operational-response implications.
- **The risk register and risk-tiering methodology.** `ai-risk-engineer` authors; AI SRE reviews the operational monitoring implications.
- **The safety-tiering policy (RSP / Preparedness / Frontier Safety Framework-equivalent).** Safety peer authors; AI SRE reviews for the operational-mitigation commitments the function would be asked to operate.

### 3.3 Joint draft

The artifact has no single author; both functions author and both sign. Examples:

- **The joint post-mortem** (chapter 02). Shape A / B / C all require joint authorship by construction.
- **The joint release checklist** (chapter 04 §5.1). Both functions fill the three-column table; both sign.
- **The pre-committed decision tree** for the crossover (chapter 04 §5.3, chapter 02 of `mod-108` for IC decisions). Jointly authored, jointly reviewed.
- **The quarterly joint retrospective** (chapter 02 §5 of this module).

A function that drifts from "AI SRE drafts / peer approves" to "AI SRE authors alone" on §3.1 artifacts is a function that will soon author an artifact the peer disowns. A function that drifts from "peer drafts / AI SRE reviews" to "AI SRE writes the first draft to speed things up" on §3.2 artifacts ends up with the wrong function's framing in load-bearing documents.

## 4. The anti-patterns at the boundary

Each anti-pattern below has a specific symptom the function recognises and a specific correction the function applies.

- **Owning the defensive control instead of its latency budget.** The AI SRE starts commenting on the classifier's precision; the classifier evolves away from what the security peer thinks is correct. Correct: the AI SRE's lane is the latency budget, failure mode, and dependency surface. The content of the classifier is the peer's.
- **Owning the risk register instead of the monitoring.** The AI SRE updates the risk register because `ai-risk-engineer` is slow this quarter. The register's framing drifts; the next review finds a risk the function considers mitigated and legal considers open. Correct: the AI SRE feeds signals; `ai-risk-engineer` maintains the register.
- **Writing the filing instead of the technical narrative.** The AI SRE drafts the specific Art. 73 or MDR language. Legal rewrites it and files; later a regulator question reveals the facts are stated to a standard the engineer-authored draft got wrong. Correct: the AI SRE writes technical narrative (chapter 02 of this module, chapter 06 of `mod-108`), legal writes the filing.
- **Treating SecOps / IR as "the security team."** SecOps / IR and `ai-infra-security` are distinct roles on most organisations. Chain-of-custody discipline runs with SecOps / IR; defensive-control methodology runs with `ai-infra-security`. Conflating them routes incidents to the wrong peer.
- **Collapsing the safety peer into security.** Safety / policy / responsible-AI is a function distinct from security on frontier-lab-shaped organisations. Even where they overlap operationally, treating them as one function collapses the chapter 04 reconciliation discipline.
- **Treating "compliance" as a single function.** Legal, MRM, regulatory affairs, the quality system, `ai-risk-engineer` — each of these is a distinct consumer of the AI SRE's compliance telemetry. The attestation is routed to the right one, not to a catch-all "compliance" address.
- **Running an AI SRE-only pre-mortem for a jointly-shaped risk.** The function pre-mortems a release without the security or safety peer in the room. The pre-mortem is incomplete. Correct: pre-mortems for crossover-shaped releases are jointly scheduled.
- **"Reading" a framework for the first time during an incident.** The function's on-call does not learn the EU AI Act, PCCP, SR 11-7, RSP-equivalent, or the organisation's internal risk framework at 3 AM. Chapter 03 and chapter 04 of this module are out-of-incident reading; the game-day cadence from `mod-109` chapter 05 drills the triggered-change and serious-incident workflows.
- **Volunteering as the PR / external-comms lead on an AI incident.** An AI incident with external visibility has a communications owner (frequently CommsOps, sometimes legal, rarely engineering). The AI SRE provides technical narrative on request; the function does not sign the external communication.
- **Treating the joint dashboard as "security's dashboard."** The joint dashboard renders both functions' signals. It is read in the standing joint review (chapter 04 §5.4) with both functions present; it is not a hand-off surface where one function updates and the other consumes passively.

## 5. When the boundary bends

Boundary chapters over-fence if not read carefully. Three cases where the boundary bends, and the pattern for bending it correctly:

### 5.1 A reliability-shaped incident uncovers a defensive gap

The AI SRE is investigating a serving regression; while tracing, the engineer notices the output moderation classifier would have accepted the regression's wrong-output case. The classifier is the security peer's; the finding is the AI SRE's. The pattern:

- The AI SRE escalates the finding to the security peer immediately.
- The AI SRE adds the detection (a telemetered signal for the observed case) to the dashboard so the function sees recurrence.
- The AI SRE does **not** propose the classifier fix.
- The retro, if the incident surfaces to post-mortem, uses Shape B or Shape C from chapter 02 as appropriate.

### 5.2 A compliance-shaped surface uncovers a reliability gap

`ai-risk-engineer` is preparing a quarterly attestation; reviewing the retained telemetry, the engineer notices a signal class has a 12-hour gap three weeks ago. The gap was not an SRE incident at the time (dashboards scrolled past it). The pattern:

- The AI SRE investigates the gap and files a post-incident runbook (`rb-compliance-telemetry-gap` extension) if the cause is operational.
- The AI SRE commits a fixture per `mod-108` chapter 05 so a future gap triggers an alarm, not an audit discovery.
- The attestation reflects the gap honestly; the function does not amend history.

### 5.3 A safety-shaped surface commits an operational mitigation that the AI SRE is not currently able to operate

The safety function has committed in the RSP-equivalent to "safety-relevant-event MTTD under 15 minutes." The current fleet's instrumentation on the relevant signal has an MTTD of 45 minutes. The pattern:

- The AI SRE does not disown the commitment ("we can't do that").
- The AI SRE commits to a plan to close the gap (specific engineering work, timeline).
- Until the plan lands, the joint release checklist (chapter 04 §5.1) shows "mitigation not yet in effect" and releases that depend on it are paused or scoped.
- The function's quarterly readout escalates the gap if it does not close.

Across the three: the pattern is the same as `mod-109` chapter 06 §5 — the SRE function detects or escalates the cross-boundary signal, names the peer, takes the SRE-side action, and does not take the peer-side action.

## 6. The dashboard-and-review surface

The crossover's dashboards and review cadences are jointly owned enough that they deserve explicit naming so neither function ends up as the lone operator.

- **Joint operational dashboard** (chapter 01 §7.6): guardrail health + serving SLO. AI SRE operates the dashboard infrastructure; security peer operates the classifier-side signals. Both review weekly.
- **Joint release checklist dashboard** (chapter 04 §5.4): mitigation state per operational commitment. AI SRE operates; safety peer co-reads.
- **Compliance telemetry dashboard** (chapter 03): the signal inventory and retention state. AI SRE operates; `ai-risk-engineer` co-reads; legal reviews.
- **Data-handling dashboard** (chapter 05 §6): artifact inventory, legal-hold state, access anomalies. AI SRE operates; SecOps / IR co-reads; legal reviews.
- **Quarterly joint retrospective readout** (chapter 02 §5): trends in joint post-mortems. AI SRE and security peer jointly produce; function leaders consume.

A function that cannot name which dashboard exists for which boundary surface is running the crossover from incident to incident.

## 7. The AI SRE's crossover peer-boundary worksheet

Use this at the function's quarterly review and when onboarding to the crossover role:

1. **Name the peers** with their current role-handles (names and function handles). The crossover has six peers listed in §1; the function's actual org may name some differently.
2. **List the shared artifacts** and classify each as §3.1, §3.2, or §3.3. Review the classification with the peers.
3. **List the AI SRE's irreducible deliverables** from §2. Confirm each is current and operational.
4. **Walk the anti-patterns in §4.** Any observed in the past quarter? If yes, the correction is a committed action.
5. **Walk the bend-the-boundary cases in §5.** Any observed? Each should have produced a specific AI SRE-side action and a specific peer handoff.
6. **Confirm the dashboards from §6 are deployed, operational, and reviewed on cadence.** Any that are stubs or stale are the next engineering priority.
7. **Sign the worksheet** jointly with the security peer, the safety peer, `ai-risk-engineer`, and legal. The joint signature is itself a boundary-aligning act.

The worksheet is the artifact that keeps the crossover role honest over multi-quarter horizons.

## Summary

- **Six peers share the crossover surface** — `ai-infra-security`, `ai-risk-engineer`, legal / compliance, safety / policy, SecOps / IR, MRM / regulatory affairs / quality system. Each owns a distinct responsibility.
- **The AI SRE's irreducible crossover deliverables** are the guardrail latency-budget contract, the per-hop SLI set, the joint post-mortem template, the compliance-telemetry inventory and attestation, the reliability-gate / joint release checklist, the incident-sensitive data discipline, and the runbook library entries for this module's named failure modes.
- **Shared artifacts fall into three ownership shapes** — AI SRE drafts / peer approves, peer drafts / AI SRE reviews, joint draft. The shape is explicit per artifact.
- **The anti-patterns at the boundary are specific and preventable** — owning the control instead of its budget, owning the risk register instead of the monitoring, writing the filing instead of the narrative, collapsing SecOps / IR with `ai-infra-security`, collapsing safety with security, collapsing compliance into one function, running AI SRE-only pre-mortems, learning frameworks in-incident, volunteering as external comms, treating the joint dashboard as a hand-off surface.
- **The boundary bends in three repeatable patterns** — reliability-shaped incident uncovers defensive gap, compliance-shaped surface uncovers reliability gap, safety-shaped surface commits an un-operated mitigation. The AI SRE detects, acts on the SRE side, hands off cleanly.
- **The dashboard-and-review surface is jointly owned** and reviewed on an explicit cadence so the crossover does not reduce to incident-to-incident coordination.
- **The worksheet in §7** is the artifact that keeps the crossover honest over time.

This closes module 110. The crossover the module writes is the one where an AI SRE function most often *over-reaches* into peer domains and most often *underserves* its own domain; the chapters here commit the function to a shape where both failure modes are prevented, so the function can compose cleanly with the security, safety, and compliance programs operating in parallel and the next modules (111 release engineering, 112 data / config pipelines, 113 DR / BCP, 114 cross-functional leadership) can inherit the crossover posture without re-authoring it.
