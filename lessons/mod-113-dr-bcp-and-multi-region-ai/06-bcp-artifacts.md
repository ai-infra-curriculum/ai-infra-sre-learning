# Business Continuity Artifacts

## Why this chapter exists

The chapters so far have written the practice: the frameworks (01), the regional-outage playbooks (02), the state reconstitution (03), the residency posture (04), and the rehearsal cadence (05). What they have *not* produced is the pack of artifacts the function hands to a successor on-call engineer, to leadership, to the function's risk / compliance peer, or to an external auditor. The practice is unverifiable without the artifact pack. A DR capability that lives in individual heads and ad-hoc runbooks is a capability that disappears with personnel turnover and fails under an audit the function did not expect.

This chapter writes the artifact set. It names each artifact, states its scope, templates its structure, and specifies where it lives. It stops at the artifact — the audit *of* the artifact (whether the pack satisfies SOC 2 CC9.1, ISO 22301 clause 8.5.4, HIPAA §164.308(a)(7), or sector-specific regulators) is at the `ai-risk-engineer` and legal peers, cited in chapter 07. The function's responsibility is that the pack is complete, truthful, and current. The determination that it is *sufficient* is downstream.

Primary references: [NIST SP 800-34 Rev 1 §4](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final) (the ISCP template), [ISO 22301:2019 §7–8](https://www.iso.org/standard/75106.html), [NIST Cybersecurity Framework v2.0](https://www.nist.gov/cyberframework), [AWS — Writing a disaster recovery plan](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/plan-for-disaster-recovery-dr.html), the [SOC 2 Trust Services Criteria](https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services) (CC9.1 on risk mitigation, A1 on availability), and `mod-110` chapters 02 and 05 for the audit and retention surface this artifact pack cites.

## 1. The pack at a glance

The function's BCP / DR artifact pack contains the following, stored in a single Git repository (or an equivalent documentation substrate) under access controls that match the function's standard engineering policies plus explicit read access for the risk and compliance peers:

1. **The contingency-planning policy** — one or two pages authorising the programme.
2. **The business impact analysis (BIA)** — the per-mission-process impact classification and RPO / RTO assignment.
3. **The information system contingency plan (ISCP) per service** — the NIST SP 800-34 §4 template, AI-adapted.
4. **The RPO / RTO table** — a consolidated, cross-service view of recovery targets.
5. **The call tree and roster** — who to contact in what order during a DR event.
6. **The communication templates** — status-page, user-notice, regulator-notice, executive-update.
7. **The regulator-facing summary scaffold** — a one-document landing page for an auditor or regulator.
8. **The drill log** — a running record of every exercise, score, and ticket.
9. **The risk register entries for open DR gaps** — the items leadership has accepted and not resourced.
10. **The annual review record** — the sign-off that the pack has been reviewed in the past year.

The pack is reviewed annually (at minimum) and after every significant architecture change, personnel change, or incident. The review record is itself an artifact.

## 2. The contingency-planning policy

The policy is a short authorising document. It states that the function operates a DR / BCP programme, names the owners, defines the scope, and commits to a review cadence.

### 2.1 Structure

- **Purpose.** One paragraph on why the programme exists. Cites the relevant regulation, customer commitment, or executive directive that drives it.
- **Scope.** Which services, which teams, which regions, which data classifications are covered.
- **Roles and responsibilities.** The SRE function lead, the on-call rotation, the incident commander, the service owner, the risk / compliance peer, the legal peer, the executive sponsor. Each has one or two sentences of named responsibility.
- **Programme components.** Enumerates the artifacts (§1) and their owners.
- **Review cadence.** Annually, and after significant change.
- **Policy approval.** Signatures (or Git-committed equivalent) from the SRE function lead and the executive sponsor. Dated.

### 2.2 Length and tone

The policy is intentionally short (one to two pages). A longer policy is mistaken for the plan itself. The ISCP is where the mechanics go; the policy is only the frame.

## 3. The information system contingency plan (ISCP) per service

The ISCP is the central artifact. NIST SP 800-34 Rev 1 §4 provides the template; the function AI-adapts it. One ISCP per tier-1 service, one per tier-2 service, one shared ISCP for a class of tier-3 services.

### 3.1 The AI-adapted ISCP template

The function's template:

**Section 1 — Introduction.**

- Service name, owner, tier.
- Scope (functions covered, functions explicitly out of scope).
- References (framework citations per chapter 01, dependencies on sibling ISCPs).
- Document control (author, approvers, revision history, next review date).

**Section 2 — Concept of operations.**

- System description — architecture, dependencies, data flows.
- The stamp topology (chapter 04 §2.1) — regions, residency boundary, warm-standby configuration.
- Interfaces to peer teams.
- Recovery objectives (RPO, RTO per state class; references the §4 table).
- Recovery-strategy selection (one of the four AWS-vocabulary strategies from chapter 01 §3.1).

**Section 3 — Activation and notification.**

- Activation criteria: what signal triggers the plan (chapter 02 §1 impairment classes, chapter 05 §3 scenario rotation).
- Notification sequence: the call tree (§5), the status-page cadence, the regulator thresholds.
- Decision authority: who declares the DR event, who authorises the failover, who closes it.

**Section 4 — Recovery (the playbook).**

- Detection — the specific alerts and dashboards that signal the event.
- Decision — the decision tree from chapter 02 §6.2.
- Execution — the ordered sequence from chapter 02 §6.3, with command-level specificity (named CLI invocations, specific route names, specific channel bindings).
- State preservation — the chapter 03 contract per class.
- Residency — the chapter 04 constraint and the DPO escalation (where applicable).
- Communication — the §6 templates and cadence.
- Recovery — the fail-back sequence.
- Post-mortem — the handoff to `mod-108` chapter 03.

**Section 5 — Reconstitution.**

- The fail-back criteria.
- The staged return to primary.
- The reconciliation of state (chapter 03 §2.4).
- The teardown of cross-residency burst (chapter 04 §3.2), if invoked.
- The closure of the incident.

**Section 6 — Plan appendices.**

- Contact list (§5).
- Vendor contact and support procedures (`mod-107` chapter 04).
- Documented drill log excerpts (§9).
- References to runbooks (`mod-108` chapter 01).
- References to chaos experiments (`mod-109` chapter 03).

### 3.2 The level of specificity

An ISCP that says "failover to the secondary region" is not an ISCP. An ISCP that says "invoke `rb-regional-failover` for the `chat-core` route; the runbook cites `promptctl rebind chat-core/stable --region eu-west-2` as the first step; the auto-halt is wired on the `svc_slo_burn_rate` alarm in dashboard DR-1" is an ISCP. Specificity to the command level is the test.

### 3.3 The living-document discipline

The ISCP is the top-of-the-runbook-library artifact. It is reviewed:

- Annually.
- On every architecture change.
- After every incident of any size where the ISCP was referenced (even if DR was not declared).
- After every drill.

Changes land as PRs under the same review discipline as code. The function treats a stale ISCP as a reliability bug.

## 4. The RPO / RTO table

A consolidated, cross-service table. The function maintains one.

### 4.1 Structure

Columns: service, tier, state class, RPO target, RTO target, measured RPO (latest drill), measured RTO (latest drill), substrate, residency, replication mechanism, reconciliation policy, link to ISCP.

Rows: one per (service, state class) pair. A tier-1 service with six state classes (chapter 03 §1) produces six rows.

### 4.2 The business-input column

A column — "business justification" — cites the BIA entry that produced the target. A reader should trace each number to its business rationale, not to engineering intuition. "RPO 30 s" alone is a number; "RPO 30 s because the customer contract clause 4.3 requires conversation continuity under disruption" is a defensible number.

### 4.3 The measured-vs-target delta

The columns "measured RPO" and "measured RTO" come from the most recent drill (chapter 05 §6.4). A row where the measured value exceeds the target is a visible gap, tracked as a ticket, resurfaced on the quarterly review (chapter 05 §7). Leadership sees the gap; the risk register logs it until closed.

### 4.4 Access and refresh

The table is in Git, rendered to a dashboard the function and leadership can read. It is refreshed on every drill and on every ISCP change. The refresh cadence is enforced by a scheduled CI job that fails if the table is older than 30 days.

## 5. The call tree and roster

The DR event's first hour is dominated by who to contact. A stale call tree is one of the most frequent findings in post-incident reviews.

### 5.1 Structure

The call tree includes, in named order:

- **The on-call rotation.** The primary and secondary on-call for the current window, with phone, SMS, and Slack / chat identity. Pulled from the function's paging substrate.
- **The service owner.** Named human plus escalation.
- **The incident commander.** The designated IC for the rotation.
- **The function lead.** On a tier-1 event, the function lead is paged.
- **Peer owners.** The `ai-infra-inference-engineer`, `ai-infra-performance`, `ai-infra-security`, `ai-infra-mlops` peers whose systems are commonly involved in an AI DR event.
- **Legal / compliance.** The DPO (per chapter 04 §3.3), the risk peer, the general counsel escalation. Each has a response-time SLA.
- **Executive sponsor.** The engineering leadership contact for the event class.
- **Communications.** The customer-communications owner, the status-page editor.
- **Vendor support.** The cloud-provider, model-provider, and major-dependency support contacts with account identifiers.
- **Regulator liaisons** (if applicable). The pre-registered regulator contact per regime.

### 5.2 Freshness

The call tree has contact information that goes stale on every personnel move. The function:

- Pulls on-call and peer identities from the live paging substrate, not from a static file.
- Reviews static entries (legal, executive, regulator) quarterly.
- Runs a contactability check — a drill phone call to each static entry — semi-annually.
- Treats a failed contactability check as an incident.

### 5.3 Confidentiality

The call tree contains personal contact information. It lives in a restricted-access repository or documentation substrate. External access (by auditors) is through the risk peer, not the public documentation.

## 6. Communication templates

A DR event requires simultaneous communication to multiple audiences at different cadences. The function pre-authors each template so that the on-call does not draft under incident pressure.

### 6.1 The status-page template

The customer-facing status update. Published on the function's status page.

- **Initial update.** Issued within the ISCP's target time (often 10–20 minutes from detection on tier-1).
- **Progress update cadence.** Every 30–60 minutes while the event is active.
- **Resolution update.** Issued on recovery, with a short description of the event.
- **Post-mortem link.** Added after the post-mortem publishes.

Template language is pre-authored per severity and per scenario class. The on-call fills in the variables; they do not write the message.

### 6.2 The user-notice template

The direct-to-user notification. Applies on residency-affecting events, data-boundary events, or events that affect a bounded user set in a way the status page does not address.

- **Trigger condition.** Pre-agreed with legal; cited in the ISCP.
- **Timeline.** Per the regulatory regime (GDPR Article 34, equivalents elsewhere) or the contractual commitment.
- **Content.** The event description, the impact on the user's data, the mitigation, the user's options (if any), the contact for questions.
- **Approval.** Legal signs off before the notice fires. The template exists so the sign-off is on the variable parts, not the entire draft.

### 6.3 The regulator-notice template

The regulator-facing notification. Applies on events that meet the regulator's reporting threshold (GDPR Article 33: supervisory authority, 72 hours; sector-specific thresholds differ).

- **Content.** The event's regulatory classification, scope, affected data categories, timeline, mitigation, remediation, lessons learned.
- **Delivery.** Through the pre-registered regulator channel (in some regimes, a portal; in others, a named contact).
- **Authority.** The function drafts; legal approves; the designated officer files.

### 6.4 The executive-update template

The internal-to-executive update. Applies on tier-1 events.

- **Cadence.** Every 30 minutes while active; immediately on resolution.
- **Content.** Impact, status, decisions needed from the executive, forward plan.
- **Distribution.** A pre-named mailing list and / or executive chat channel.

### 6.5 The internal-stakeholder update

A broader internal update — product, customer success, sales. Content is bounded to what those teams need to represent to customers.

### 6.6 The language-neutrality rule

Templates avoid speculation. They state what is known, what is not known, and what is being done. They do not attribute cause until root cause is confirmed. The `mod-108` chapter 03 blameless post-mortem discipline extends to the live-event communications.

## 7. The regulator-facing summary scaffold

An external auditor or regulator who arrives at the function asking "show me your DR capability" cannot read ten chapters. The function maintains a single document — one to three pages — that is the entry point for that conversation.

### 7.1 Structure

- **Overview.** One paragraph on the DR / BCP programme, its authority, its scope.
- **Framework mapping.** A short table: the NIST SP 800-34 seven-step process, the ISO 22301 clauses, and the SOC 2 / sector-specific criteria the programme addresses, with pointers to the artifacts that satisfy each.
- **Service coverage.** A table of services with their tier classification, their RPO / RTO target, and their ISCP link.
- **Drill cadence.** The current cadence per tier, the last drill per service, the next scheduled drill.
- **Audit retention.** A pointer to `mod-110` chapter 05's retention policy.
- **Known gaps.** The open items from the risk register (§9), honestly stated.
- **Contact.** The function lead's contact for follow-up.

### 7.2 The honesty principle

The scaffold states gaps where they exist. An auditor is more suspicious of a programme that claims no gaps than of one that shows a tracked risk register with owners and dates. The function does not hide; the function documents.

### 7.3 Who publishes it

The scaffold is published through the risk peer, not the function directly. External auditors route through the risk peer; the scaffold is the risk peer's primary artifact when the auditor arrives. The function keeps it current; the risk peer presents it.

## 8. The drill log

Every exercise is logged.

### 8.1 Structure

Per drill: date, scenario, services involved, participants, scores (chapter 05 §6.4), findings, tickets opened, tickets closed from prior drills.

### 8.2 Rolling trend

The drill log is the data source for the quarterly dashboard (chapter 05 §7): detection-time trend, RPO / RTO trend, auto-halt reliability trend, score trend.

### 8.3 Retention

The log is retained for the audit horizon defined in `mod-110` chapter 05. The function does not delete historical drill logs; they are evidence of the programme's cadence.

## 9. The risk register for DR gaps

Not every drill finding gets fixed in the same quarter. The function tracks the unfixed ones.

### 9.1 Structure

Per entry: the finding, the service affected, the RPO / RTO implication, the recommended fix, the cost estimate, the owner, the acceptance decision (who, when), the review date.

### 9.2 Acceptance discipline

An entry that reads "accepted" has a named owner who accepted it. Acceptance is not "we didn't get to it"; it is an explicit leadership decision that the cost of fixing is higher than the risk of leaving. The function will not operate a tier-1 service against an accepted-by-no-one gap.

### 9.3 Resurfacing

Accepted entries resurface annually. Business context changes; a gap accepted last year may be unacceptable this year because the customer contract changed or because the regulatory posture tightened.

## 10. The annual review record

ISO 22301 and NIST SP 800-34 both require periodic review of the programme. The function's record:

- Date of review.
- Participants (function lead, service owners, risk peer, legal peer, executive sponsor).
- Changes since last review (architecture, personnel, regulation, drill findings).
- Updates to the policy (§2), the BIA, the ISCPs (§3), the RPO / RTO table (§4), the call tree (§5), the templates (§6).
- Open items carried forward.
- Sign-off.

The review record is itself an artifact, stored with the pack. An auditor reads the sequence of review records as evidence that the programme is a continuing practice.

## 11. The artifact-pack checklist

Before claiming the artifact pack is complete, the function confirms:

1. **The contingency-planning policy** is signed and current (§2).
2. **The BIA is read-from, not merely written** — every RPO / RTO target traces to a BIA entry (§4.2).
3. **Every tier-1 service has an ISCP** at command-level specificity (§3.2).
4. **The RPO / RTO table** is live, refreshed monthly by CI (§4.4).
5. **The call tree** pulls on-call identities from the live substrate and passes a semi-annual contactability check (§5.2).
6. **The six communication templates** are drafted, legally reviewed, and tested (§6).
7. **The regulator-facing summary scaffold** is current, honest, and owned by the risk peer (§7).
8. **The drill log** is being written on every exercise (§8).
9. **The risk register** has named accepters on every open gap and resurfaces annually (§9).
10. **The annual review record** is dated within the past twelve months and signed (§10).

A function whose pack passes the checklist can hand the pack to an external auditor without rewriting it. A function whose pack does not pass the checklist will not have the week an unannounced audit allocates.

## Summary

- The DR / BCP artifact pack is the function's visible capability. It is in Git, it is reviewed annually, and it survives personnel turnover.
- The pack has ten components: contingency-planning policy, BIA, per-service ISCPs, consolidated RPO / RTO table, call tree, communication templates, regulator-facing summary scaffold, drill log, risk register, and annual review record.
- The ISCP is the central artifact. The NIST SP 800-34 §4 template is AI-adapted to include stamp topology, state-class contracts, residency constraints, and the chapter 02 end-to-end sequence at command-level specificity.
- The RPO / RTO table is the cross-service view; every target traces to a BIA entry; every measured value comes from a drill; gaps surface on the quarterly review.
- Communication templates are pre-authored for status-page, user-notice, regulator-notice, executive-update, and internal-stakeholder updates. The on-call fills variables, not messages.
- The regulator-facing summary scaffold is the auditor's entry point — honest about gaps, owned by the risk peer, maintained by the function.
- A ten-item checklist is the gate from "we have artifacts" to "we have an artifact pack." The pack is the evidence the programme is a continuing practice.

Chapter 07 closes the module by naming the peer boundaries: what the function owns under this artifact pack, what multi-region architecture work escalates to the `ai-infra-architect` level, what multi-year DR strategy escalates to the `ai-infra-senior-architect` level, and what regulator-facing evidence is at legal and compliance.
