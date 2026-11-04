# Data Residency in Failover

## Why this chapter exists

A clean DR plan moves traffic from the failing region to a healthy one. A legally defensible DR plan refuses to move traffic from an EU region to a US region silently, even if moving the traffic would meet the RTO, because doing so breaches the residency commitment under which the traffic was collected in the first place. The regulatory posture — GDPR for EU personal data, the forthcoming EU AI Act for high-risk AI systems, HIPAA for US health data, sector-specific regimes for finance, India's DPDP Act, Brazil's LGPD, UK GDPR — treats cross-border transfer as a controlled event with its own legal basis, not a side effect of a 500 Internal Server Error. The function's DR plan must respect this.

The chapter writes the shape of a residency-respecting failover: how to isolate per region at the design layer, when controlled cross-region bursting is acceptable, how to reason about the residency / reliability trade-off when the primary region is down and the only available secondary is out-of-residency, and where the escalation goes when the function cannot decide alone. It stays firmly on the engineering side of the line — the legal determinations (what counts as personal data, what the lawful basis is, whether a Standard Contractual Clause covers the failover) are at the legal and compliance peers. The function makes those determinations *implementable*.

Primary references: [EU General Data Protection Regulation (GDPR)](https://eur-lex.europa.eu/eli/reg/2016/679/oj) (Articles 44–50 on international transfers), [EU AI Act](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32024R1689), [EDPB Guidelines on international transfers](https://www.edpb.europa.eu/our-work-tools/general-guidance/guidelines-recommendations-best-practices_en), [US HIPAA Security Rule](https://www.hhs.gov/hipaa/for-professionals/security/laws-regulations/index.html), [Standard Contractual Clauses (EU Commission)](https://commission.europa.eu/law/law-topic/data-protection/international-dimension-data-protection/standard-contractual-clauses-scc_en), [AWS Data residency](https://aws.amazon.com/compliance/data-protection/), [Microsoft — Data residency in Azure](https://azure.microsoft.com/en-us/explore/global-infrastructure/data-residency/), and [Google Cloud — Data residency](https://cloud.google.com/architecture/framework/security/data-residency-sovereignty). Chapter 06 of `mod-110` writes the audit-retention surface that overlaps with this chapter; this chapter writes the failover-direction constraint.

## 1. The residency vocabulary

The function fixes vocabulary. Each term below has a specific legal operational meaning; collapsing them produces policy that reads reasonably and implements badly.

- **Data residency.** Where data physically sits — in which cloud region, which jurisdiction. A commitment to "EU data residency" means the storage substrate is in an EU region.
- **Data sovereignty.** The jurisdiction's legal authority over the data. A European user's data stored in an EU region operated by a US cloud vendor is subject to EU law *and* potentially subject to US law via the vendor; sovereignty is about *whose law applies*, not where the bytes sit.
- **Cross-border transfer.** Moving personal data from one jurisdiction to another. Under GDPR Chapter V, a cross-border transfer requires a legal basis (adequacy decision, SCC, Binding Corporate Rules, derogation).
- **In-scope data.** Data categories the regulation applies to. GDPR applies to personal data of EU data subjects; HIPAA applies to PHI in a covered entity's control; sector regimes apply to their sector's data. A DR plan that does not classify its data cannot reason about residency.
- **Controlled bursting.** Deliberately shifting a bounded, defined subset of traffic across regions under a pre-authorised legal basis. The opposite of "silent failover."
- **Local processing.** Processing happens in the region. A cross-region call that *reads* a model weight from a cache is not a transfer of in-scope data; a cross-region call that *sends* a user's prompt to another region is.

## 2. Per-region isolation as a design primitive

Residency-respecting DR starts well upstream of the DR event. The design-time primitive is **per-region isolation**: an EU stack and a US stack run as two distinct, parallel deployments with their own serving tier, their own state stores, and their own trace stores. The DR plan writes the rules under which they interact; the default is that they do not.

### 2.1 The regional stamp

The function's architecture follows a **regional stamp** pattern (adopted from Azure's [Mission-critical workload guidance](https://learn.microsoft.com/en-us/azure/well-architected/mission-critical/mission-critical-application-platform#deployment-stamps-pattern) and generally applicable across vendors):

- Each stamp is a complete, independent deployment of the service in one region.
- Each stamp has its own ingress, its own service tier, its own state stores, its own trace store, its own warm-standby secondary *inside the same residency boundary*.
- Stamps do not share state. A user whose traffic lands on the EU stamp has no data stored in the US stamp.
- A single global routing layer dispatches users to the stamp appropriate to their residency.

A stamp is the atomic unit of residency. A cross-stamp operation is, by construction, a cross-residency operation; the DR plan defines when (if ever) a cross-stamp operation is allowed.

### 2.2 Within-residency warm standby

The chapter 02 §4 warm-standby posture holds *inside each stamp's residency*. An EU stamp has its primary in (for example) `eu-west-1` and its warm standby in `eu-west-2` or `eu-central-1` — both EU regions. A US stamp has its primary in `us-east-1` and its warm standby in `us-west-2`. Cross-stamp failover — EU to US — is not the default DR move; within-stamp failover is.

The decision has a cost. A stamp with two EU regions available has reasonable resilience against a single-region event; a stamp with only one usable in-residency region has no in-residency failover path. The function must enumerate the in-residency region set per stamp, confirm it meets the operational needs of the BIA, and name the cases where it does not (§4 below).

### 2.3 Isolation of the shared control plane

Some components are unavoidably shared across stamps:

- **The model-weight artifact store.** Weights are not personal data; they are a model identifier. Cross-residency replication of weights is legally fine; chapter 02 §3 handles the mechanics.
- **The prompt / config registry.** The registry content is not personal data; replication is fine.
- **The release pipeline.** CI / CD runs in a single operational environment; the function pushes the artifact, not customer data.
- **The observability roll-up.** Dashboards may need to read SLI / counts across stamps. Counts are not personal data; raw user prompts are.

The function draws the line: shared control plane for artifacts, regional isolation for user data. Observability dashboards aggregate on fields that are explicitly non-PII; the trace content itself (prompts, responses) is retained in-residency and is not exported.

### 2.4 Audit boundary

The audit record (`mod-110` chapters 02 and 05) is in-scope data because it contains the model's inputs and outputs. The function keeps audit records within the residency of the originating stamp. Cross-stamp read of audit records is reserved for the legal / compliance peer, under a controlled process, not for the function's day-to-day investigation.

## 3. The residency / reliability trade-off

The fundamental tension: a stamp's primary region is down, the in-residency secondary is also down (or simply does not exist), and the only available healthy region is out-of-residency. The function cannot pretend this case does not exist, and cannot silently cross the residency boundary when it happens.

### 3.1 The decision space

The function enumerates the plausible decisions:

- **Hold.** Serve 5xx / degraded responses from the impaired primary; wait for recovery. Users in the stamp see an outage; residency is preserved. The RTO is violated.
- **Degraded mode in-residency.** A degraded serving tier in the stamp (a smaller model, a rule-based fallback, a text-only canned response) stays up without crossing residency. Users see reduced quality; residency is preserved.
- **Controlled cross-residency burst under pre-authorised terms.** A pre-approved, pre-documented legal basis allows a defined subset of traffic to flow out-of-residency for the duration of the event. Users continue to be served at full quality; residency is a documented, bounded exception.
- **Full cross-residency failover.** All traffic moves to the out-of-residency secondary. Appropriate only for stamps whose in-scope data classification explicitly permits this, or where the business decision has accepted the compliance exposure.

The default on an EU stamp carrying GDPR-scope personal data is **hold or degraded in-residency**. Cross-residency failover is the exception, not the rule.

### 3.2 Pre-authorising controlled bursting

A function that wants cross-residency as an option must prepare it *before* the incident. The preparation is cross-functional:

- **Legal determines the lawful basis.** For GDPR, this may be a Standard Contractual Clause and a documented Transfer Impact Assessment. The SCC lives in Git with the ISCP; the function references it, does not author it.
- **The data categories allowed to transfer are enumerated.** Not every category of in-scope data may be approved for transfer; the function's service may hold several. The ISCP enumerates which classes burst, which do not.
- **The maximum burst duration is defined.** "Up to 4 hours in any 30-day period"; beyond that, legal re-approval is required.
- **The notification obligation is defined.** Who is notified (DPO, customer, regulator), on what timeline, by what template.
- **The burst is logged.** Every cross-residency request is tagged `burst_event_id=<id>` in traces; the audit retention bucket for the burst is in-residency and retained per the SCC.

When a disruption triggers the burst, the function invokes it operationally; it does not negotiate the legal basis under incident pressure.

### 3.3 The decision is not the function's alone

The RTO is the function's SLO; the decision to cross residency during an outage is not a reliability decision, it is a compliance decision. The function's ISCP cites an **escalation to the data protection officer (DPO) or equivalent**, with a documented response-time target (minutes, ideally measured). The DPO's decision is logged in the incident channel alongside the technical decision.

If the DPO is unreachable within the response-time target, the function's default is the conservative decision: hold or degraded in-residency. The function does not improvise a residency exception.

### 3.4 User notice

A cross-residency burst that affects users' data is, on most interpretations, a user-notice event. The exact threshold is a legal question; the function pre-authors the notice template and the DPO signs off on when it fires. A user whose session was bursted out-of-residency should be notified with a timeline consistent with the regulation's breach-notification rules (GDPR Article 33–34: 72 hours to the supervisory authority on applicable breaches; no fixed user-notice window but "without undue delay").

## 4. Pattern catalogue for residency-constrained failover

Three common patterns and when to use them.

### 4.1 Two-in-residency with symmetric warm standby

The stamp has two usable regions within the residency boundary. Both are warm. Primary failover goes region-to-region within the stamp; cross-residency is not invoked under single-region events. This is the preferred posture; it keeps residency compliance in the default path.

- **EU stamp:** `eu-west-1` (primary), `eu-west-2` or `eu-central-1` (warm). Chapter 02 §4 sizing applies.
- **US stamp:** `us-east-1` (primary), `us-west-2` (warm).
- **UK stamp (post-Brexit, under UK GDPR):** `eu-west-2` (primary); UK has fewer regional options; pair with UK-adequate region.
- **APAC / India / Brazil / Middle East:** vendor coverage is uneven; some jurisdictions have only one in-region option. §4.3 applies.

### 4.2 One-in-residency with degraded-mode fallback

The stamp has only one usable region. A full-region impairment has no in-residency failover; the stamp holds or runs a degraded mode within the region (a degraded serving tier that does not require the impaired dependencies — a smaller model served from a different AZ, a rule-based fallback). The ISCP cites the degraded-mode behaviour; it is drilled (chapter 05 §3).

### 4.3 Single-region jurisdiction with pre-authorised burst

Some jurisdictions have one cloud-provider region. The stamp has no in-residency failover. The function and the legal peer pre-authorise a controlled burst to a specific out-of-residency region under a documented SCC or equivalent. The burst is scoped (data classes, max duration, notification) per §3.2. An outage invokes the burst; recovery closes it.

The function must document, before the incident, that this is the stamp's chosen posture. A regulated-industry audit (chapter 06 §5) asks this question explicitly.

### 4.4 Cross-stamp provider-level failover

A hosted model-provider's region is down (`mod-107` chapter 03 / chapter 02 §5 of this module). The provider's own cross-regional failover may or may not have shifted; the function's router has a provider-regional preference list. The residency constraint on the list is: the EU stamp's preferred-region list contains only EU provider regions, with cross-region burst only through the §3.2 pre-authorised path.

The function reads the provider's own residency documentation. Anthropic, OpenAI, AWS Bedrock, Azure AI Foundry, and Google Vertex each publish regional endpoints with residency commitments; the function pins only regions whose residency is aligned with the stamp and documents the pinning in the ISCP. <!-- needs-research: on first ISCP authoring, pull each contracted provider's current residency documentation and list the approved regional endpoints per stamp -->

## 5. The residency field in the DR plan

The DR plan artifact (chapter 06) carries, for each service:

- The **residency boundary** per stamp.
- The **in-residency region set** and which are warm.
- The **chosen pattern** from §4.
- The **DPO contact and escalation target** with response-time SLA.
- The **SCC or equivalent legal basis** for the controlled burst, if applicable.
- The **data categories eligible for burst**, with field-level granularity.
- The **maximum burst duration and the 30-day rolling allowance**.
- The **user-notice template and trigger condition**.
- The **post-burst teardown runbook** that terminates cross-residency traffic cleanly on primary recovery.

Each field is a concrete artifact in Git. A plan that leaves any as "TBD" is a plan with an unexercised residency path.

## 6. Implementation primitives

The residency posture is enforced in code, not only in documentation.

### 6.1 Request-level residency tag

Every request entering the stack is tagged on ingress with its residency class (`EU`, `US`, `UK`, `APAC`, etc.). The tag is derived from:

- The user's declared region (billing address, locale) as the first signal.
- The IP geolocation as the fallback.
- The authentication context (tenancy, contract terms) as the authoritative overlay.

The tag flows through the request as an immutable metadata field and is stamped on every subsequent audit log, trace, and downstream call.

### 6.2 Routing layer enforcement

The global routing layer (chapter 02 §2) inspects the residency tag before dispatching to a stamp. Rules:

- A request tagged `EU` is dispatched to the EU stamp. The routing layer refuses to dispatch it to a non-EU stamp unless the residency policy for that request explicitly permits it (§3.2 burst).
- A request tagged `US` is dispatched to the US stamp. Mirror rule.
- A request whose residency cannot be determined is handled per a declared default — often "fail safe" (drop with an error) rather than "fail convenient" (serve from the nearest region).

The routing layer's refusal rule is itself a reliability concern: a bug here that misroutes an EU request to the US stamp is a breach, not an outage. The function treats the routing layer with the same deploy discipline as the serving tier.

### 6.3 Egress controls

The service tier's egress to external dependencies (model providers, retrieval sources, tools) is itself subject to residency. The EU stamp's egress is scoped to EU-resident endpoints by default; cross-residency egress is a per-tool pre-authorised exception. The egress firewall rules live in Git with the ISCP.

### 6.4 Storage substrate scoping

The regional stamp's storage layer is region-locked. The IaC manifests explicitly specify the region, the IaC's policy-as-code layer (Open Policy Agent, Sentinel, Config) rejects any deployment that targets an out-of-residency region for an in-scope data substrate, and the deployment pipeline's admission controllers enforce the rule at apply time.

### 6.5 Audit emission scoping

Trace and audit emission is also region-scoped. A stamp in the EU emits to an EU-resident trace store; the cross-region replication (chapter 03 §5) is within-residency. The global observability roll-up reads from each stamp's in-residency store and aggregates on non-PII dimensions only.

## 7. The residency checklist

Before enabling the DR plan on a residency-regulated service, the function confirms:

1. **The regional stamp pattern is in place** per §2.1; stamps do not share state.
2. **The in-residency region set and warm-standby sizing per stamp** are documented per §2.2.
3. **The residency / reliability trade-off decision space** (§3.1) is enumerated per stamp.
4. **The cross-residency burst is pre-authorised** per §3.2 — or explicitly documented as not available, and the holdbehaviour is drilled.
5. **The DPO escalation path and response-time target** are documented and contact details are current.
6. **The user-notice template** is pre-authored and the trigger condition is agreed with legal.
7. **The request-level residency tag** is emitted on ingress and threaded through to every downstream call.
8. **The routing layer enforces the residency rule** in code, not only in documentation.
9. **Egress controls** block out-of-residency dependency calls by default.
10. **The IaC policy-as-code layer** rejects out-of-residency deployments of in-scope data substrates.
11. **The audit / trace emission is region-scoped**; observability roll-ups aggregate on non-PII dimensions only.
12. **A drill of the in-residency failover** exists per chapter 05 §3.
13. **A drill of the cross-residency burst** (where authorised) exists per chapter 05 §3.4, with legal participation in the tabletop.

A service that fails any item is a service whose first real cross-residency event will double as a compliance incident.

## Summary

- Residency is a design primitive upstream of failover. The regional-stamp pattern makes each residency an independent deployment; the DR plan writes the rules for the rare cross-stamp interactions.
- Within-residency warm standby is the default resilience pattern. Cross-residency is not the first DR move; it is the exception, pre-authorised in advance, scoped in duration and in data category.
- Three patterns cover most stamps: two-in-residency with symmetric warm standby (preferred), one-in-residency with degraded-mode fallback, and single-region jurisdictions with a pre-authorised controlled burst.
- The decision to invoke a cross-residency burst during an outage is a compliance decision, not a reliability decision. The function escalates to the DPO with a documented response-time SLA; the default on no-answer is the conservative choice.
- Enforcement lives in code: residency tags on every request, routing-layer refusal of mis-residency dispatch, egress controls, IaC admission policy, region-scoped audit emission. Documentation alone is not enforcement.
- A 13-item checklist is the gate from "residency considered" to "residency drilled." The DR drill programme (chapter 05) exercises the in-residency failover; the cross-residency burst (where authorised) is drilled separately with legal present.

Chapter 05 writes the rehearsal cadence — how often each playbook is tabled, drilled, and failed over — and how the cadence composes with the chaos programme from `mod-109` so that DR is an ongoing practice, not a shelved document.
