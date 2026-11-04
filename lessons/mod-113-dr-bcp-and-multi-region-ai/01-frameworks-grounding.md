# Frameworks Grounding: NIST SP 800-34 and the Three Cloud Reliability Guides

## Why this chapter exists

Disaster Recovery and Business Continuity Planning are mature disciplines. There is a NIST publication that names the steps (SP 800-34 Rev 1), a set of ISO standards that formalise the management-system shape (ISO 22301), and three primary cloud-provider reliability guides that each encode a different vendor's operational vocabulary for the same problem. The temptation on an AI platform team is to skip them — "our workloads are different" — and reinvent the practice from first principles, usually under incident pressure, usually incompletely. The temptation on a classical-trained SRE moving into AI is the opposite — adopt the frameworks verbatim and miss the places the AI workload breaks the frameworks' assumptions.

This chapter fixes both failure modes. It grounds the module in the four primary references, in their own vocabulary, with the correct citations. It then names, per framework, the specific AI-workload assumptions that *do not* survive the translation: a model-weight artifact store is not a database, a cold-start-sensitive large model is not a stateless web service, conversation state is not session state, a model-provider dependency is not a database dependency, and a quality SLO cannot be restored by restoring a snapshot. The downstream chapters write the AI-adapted discipline; this chapter writes the frame under which they do so.

Primary references for this chapter: [NIST SP 800-34 Rev 1](https://csrc.nist.gov/pubs/sp/800/34/r1/upd1/final) (*Contingency Planning Guide for Federal Information Systems*), [ISO 22301:2019](https://www.iso.org/standard/75106.html) (*Societal security — Business continuity management systems*), [AWS Well-Architected Framework — Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html), [Microsoft Azure Well-Architected Framework — Reliability](https://learn.microsoft.com/en-us/azure/well-architected/reliability/), and the [Google Cloud Architecture Framework — Reliability](https://cloud.google.com/architecture/framework/reliability) pillar together with the [Google Cloud disaster recovery planning guide](https://cloud.google.com/architecture/dr-scenarios-planning-guide).

## 1. Vocabulary

The frameworks disagree on surface-level word choice. The function fixes a single vocabulary for the module and keeps the frameworks' native terms only when citing them.

- **Disruption.** Any event that takes the service below its SLO for longer than the alert window. A disruption may or may not be a *disaster* in the NIST sense; the function treats "disruption" as the alert class and "disaster" as the escalation class.
- **Disaster.** A disruption whose recovery requires cross-region or cross-provider action. This is the class NIST SP 800-34 is written for.
- **RPO (Recovery Point Objective).** The maximum tolerable data loss, measured as the time between the last recoverable state and the disruption. A 5-minute RPO means losing up to 5 minutes of state is acceptable; a 0 RPO requires synchronous replication.
- **RTO (Recovery Time Objective).** The maximum tolerable outage duration, measured from disruption declared to service restored. A 15-minute RTO on a critical route is a strong commitment; an 8-hour RTO on a batch route is routine.
- **BCP (Business Continuity Plan).** The organisational artifact that defines how the business keeps operating through a disruption — people, communications, decision rights, regulator notice. NIST calls this a "BCP"; ISO 22301 formalises the management-system shape.
- **DR (Disaster Recovery) Plan.** The IT-focused subset of the BCP. NIST calls this a "Disaster Recovery Plan (DRP)" in SP 800-34 §3.6, distinct from the "Information System Contingency Plan (ISCP)" in §3.5. The function writes the DR plan; legal and risk peers own the BCP envelope.
- **BIA (Business Impact Analysis).** NIST SP 800-34 §3.2's step 2. The artifact that measures the business cost of a disruption per mission-critical process; the function reads the BIA to size RPO / RTO targets and challenges them where they are inconsistent with the serving architecture.
- **Multi-region.** Serving from more than one geographic region, with a defined policy for which region handles which traffic. The function uses "multi-region" for the capability and "failover" for the operation that invokes it.

The frameworks' native terms map onto this vocabulary cleanly. NIST's "contingency planning" is the full lifecycle; AWS's "disaster recovery strategies" is the subset of that lifecycle that produces the DR plan; Azure's "mission-critical workload" guidance and Google Cloud's "DR planning guide" are their vendor-specific instantiations of the same subset.

## 2. NIST SP 800-34 Rev 1: the seven-step frame

NIST SP 800-34 Rev 1 (November 2010, update 1 May 2011) codifies the US federal practice for information-system contingency planning. The publication is written for federal systems under FISMA, but the practice is portable: the seven-step process, the three-tier plan taxonomy, and the artifact set show up in every mature DR programme, federal or not. The function reads it cover to cover once and returns to the step list by name.

### 2.1 The seven-step process

SP 800-34 §3 names seven progressive steps. The function treats the first four as design-time, the fifth as artifact-authoring, and the last two as operational:

1. **Develop the contingency planning policy.** A signed policy statement authorising the programme, naming roles and responsibilities, and setting scope. On an AI platform team this is one or two pages; the function drafts it, the engineering leadership and the compliance peer sign.
2. **Conduct the business impact analysis (BIA).** Identify mission-critical processes, map them to systems and dependencies, assign RPO and RTO per process. The BIA is the input that sizes the whole programme; a DR plan whose RPO / RTO were not read from a BIA is a plan the business did not actually buy.
3. **Identify preventive controls.** Controls that reduce the probability or impact of disruption *before* it happens — redundant power, cross-AZ deployments, backups, change-control gates, rate-limiting at the edge. Under the AWS / Azure / Google guides most of these are design choices; §4 below writes the AI-specific extensions.
4. **Create contingency strategies.** The recovery architectures — backup / restore, pilot light, warm standby, hot standby — chosen per workload based on the BIA's RPO / RTO targets. AWS's "four DR strategies" (§4.1) is the clearest exposition of the choice space.
5. **Develop the information system contingency plan (ISCP).** The written DR plan. SP 800-34 §4 gives a template; chapter 06 of this module authors an AI-adapted version.
6. **Ensure plan testing, training, and exercises.** Tabletop exercises, functional drills, full-scale failovers. SP 800-34 §3.5 is prescriptive on cadence; chapter 05 of this module adapts the cadence for AI workloads and ties it to the chaos programme from `mod-109`.
7. **Ensure plan maintenance.** Review cadence (at least annually), update triggers (architecture change, personnel change, post-incident), configuration-management ties. The function treats the DR plan the same way it treats the top-20 runbook library from `mod-108` chapter 01: a living artifact under review, not a shelved document.

### 2.2 The three-tier plan taxonomy

SP 800-34 §3.5–3.6 distinguishes several plan types and makes clear they are *complementary*, not substitutes. The three that matter most for an AI SRE function:

- **Information System Contingency Plan (ISCP).** System-level recovery for a specific service. SP 800-34 §4 is its template. Every tier-1 AI service in the function's charter gets one.
- **Disaster Recovery Plan (DRP).** IT-wide recovery for major, long-duration disruptions, usually invoking cross-region or cross-site action. The function co-authors this with the broader platform SRE team.
- **Business Continuity Plan (BCP).** Organisational continuity — legal, communications, HR, customer-facing operations. The function contributes the technical annex and the RPO / RTO table; the legal / risk / executive team owns the plan proper.

The function must know which tier a given decision lives at. An RPO / RTO change is an ISCP question. A region-switch rehearsal cadence is a DRP question. A "do we notify regulators during a four-hour outage" question is a BCP question. Confusing the three produces DR plans that over-promise on authority the function does not hold and under-deliver on mechanics it does.

### 2.3 Where NIST SP 800-34 does not quite fit the AI workload

SP 800-34 Rev 1 is implicitly written for IT systems with a conventional database tier, a conventional application tier, and conventional backup / restore semantics. Four assumptions break on AI workloads and chapter 02–05 adapt each:

- **"Back up the data"** — assumes the data is a few-hundred-gigabyte database, not a multi-terabyte, cross-region-slow, pinned-version model-weight artifact (chapter 02 §3).
- **"Restore the service"** — assumes restore time is dominated by data restore, not by cold-start of a 70B-parameter model whose shard-loading time exceeds the RTO target (chapter 02 §4).
- **"Reconstitute state"** — assumes state is transactional records, not conversation history / KV state / tool-call state / trace store, all of which the AI workload treats as reliability-critical (chapter 03).
- **"Alternate processing site"** — assumes any alternate site is interchangeable, not that the EU alternate site may not legally serve EU-resident traffic that the US primary was serving in violation of policy (chapter 04).

The function cites SP 800-34 for the frame and writes the AI-adapted mechanics in the subsequent chapters.

## 3. The three cloud-provider reliability guides

Each major cloud vendor publishes a "reliability" chapter of its well-architected framework. They are not substitutes for NIST SP 800-34 — the NIST document is prescriptive about the *process*, the vendor guides are prescriptive about the *mechanics on that vendor's substrate*. The function reads all three, picks the vendor-relevant one as the implementation reference, and keeps the other two as a hedge against vendor lock-in on the programme's own vocabulary.

### 3.1 AWS Well-Architected Reliability Pillar

The AWS [Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html) is organised around five design principles — automatically recover from failure, test recovery procedures, scale horizontally, stop guessing capacity, manage change in automation — and a set of "best practices" grouped by foundations, workload architecture, change management, and failure management. The parts that matter for DR:

- **The four DR strategies.** The [AWS Disaster Recovery of Workloads on AWS](https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/disaster-recovery-options-in-the-cloud.html) whitepaper names four strategies in RPO / RTO / cost order: **backup and restore** (highest RPO / RTO, lowest cost), **pilot light** (core infrastructure always warm, minimal capacity), **warm standby** (scaled-down functional copy always running), and **multi-site active-active** (traffic actively served from multiple regions, lowest RPO / RTO, highest cost). Chapter 02 adapts the four strategies to AI-specific workload classes.
- **Shared responsibility for resilience.** AWS's [Shared Responsibility Model for Resilience](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/shared-responsibility-model-for-resiliency.html) distinguishes resilience "of the cloud" (AWS's responsibility — hardware, hypervisor, regional isolation) from resilience "in the cloud" (the customer's responsibility — application architecture, cross-AZ deployment, cross-region replication). The function names, per dependency, which side of the line each failure sits on. A model-provider outage (`mod-107` chapter 03) is a customer-side concern; an EC2 host failure is an AWS-side concern mediated by the customer's Auto Scaling configuration.
- **Multi-Region patterns.** The [AWS Multi-Region fundamentals](https://docs.aws.amazon.com/whitepapers/latest/aws-multi-region-fundamentals/aws-multi-region-fundamentals.html) whitepaper documents Route 53 failover, cross-region replication for S3, Aurora Global Database, and the DynamoDB global-table pattern. Chapter 02 maps these onto the AI-specific state-replication problem.

### 3.2 Azure Well-Architected Framework — Reliability

Microsoft's [Reliability pillar](https://learn.microsoft.com/en-us/azure/well-architected/reliability/) is organised around a set of **design principles** (design for business requirements, design for failure, observe application health, respond to failure and malfunction, implement self-healing and self-preservation) followed by a **checklist** whose items map onto RE:01 … RE:10 design-review identifiers. The parts that matter:

- **The reliability design review checklist.** The Azure docs ([RE:01 – RE:10](https://learn.microsoft.com/en-us/azure/well-architected/reliability/checklist)) give a structured audit list that fits inside the NIST SP 800-34 "preventive controls" step. The function uses the checklist as the ISCP preventive-controls annex for Azure-hosted workloads.
- **Mission-critical workload guidance.** The [Mission-critical workload architecture](https://learn.microsoft.com/en-us/azure/well-architected/mission-critical/) sub-pillar is more opinionated than the general Azure reliability docs and names specific patterns — regional stamps, deployment stamps, zonal isolation — that map directly onto the AI serving tier's stamping needs (chapter 02 §2).
- **Azure Site Recovery and backup patterns.** The [Business continuity and disaster recovery documentation](https://learn.microsoft.com/en-us/azure/reliability/) gives the Azure-native primitives (Azure Backup, Azure Site Recovery, geo-redundant storage classes). Chapter 02 maps these onto the model-weight replication problem.

### 3.3 Google Cloud Reliability Guide

Google Cloud's reliability content spans two documents: the [Architecture Framework — Reliability](https://cloud.google.com/architecture/framework/reliability) pillar and the [Disaster recovery planning guide](https://cloud.google.com/architecture/dr-scenarios-planning-guide). The parts that matter:

- **The DR scenario tiers.** The [DR building blocks](https://cloud.google.com/architecture/dr-scenarios-building-blocks) and the [DR scenarios for data](https://cloud.google.com/architecture/dr-scenarios-for-data) / [applications](https://cloud.google.com/architecture/dr-scenarios-for-applications) guides name the same four strategies as AWS, in Google's own vocabulary: **cold** (backup / restore), **warm** (pilot light), **hot** (warm standby), and **multi-site** (active-active). The equivalence is close enough that chapter 02 uses the AWS vocabulary with a single parenthetical mapping note.
- **Error budgets and graceful degradation.** The Google reliability pillar is the only one of the three that writes error-budget practice (`mod-103`) into the reliability frame itself. The function reads this as a reminder that DR / BCP sits *inside* the SLO discipline; a failover that preserves the SLO is a non-event, a failover that burns the quarterly budget is a reportable disruption.
- **Spanner and multi-region databases.** The [Spanner](https://cloud.google.com/spanner/docs) and [Cloud SQL high-availability](https://cloud.google.com/sql/docs/postgres/high-availability) documentation gives the globally-consistent and regionally-replicated database patterns. Chapter 03's state-reconstitution discussion cites these as reference points for the state-store choice.

### 3.4 Reading the three in parallel

The three guides agree on the four DR strategies (backup / restore, pilot light, warm standby, multi-site active-active), they agree on the shared-responsibility split between vendor and customer resilience, and they agree that DR is a design question upstream of the operational drill cadence. They disagree on vocabulary (pilot light vs. warm vs. hot) and on substrate (Route 53 vs. Traffic Manager vs. Cloud Load Balancing), and each is strongest on its own primitives. The function reads the vendor-relevant guide as the implementation reference and the other two as a sanity check against vendor lock-in on the programme vocabulary.

## 4. The AI-specific extensions the frameworks assume but do not write

NIST SP 800-34 and the three cloud guides give a complete frame; they stop short of five AI-specific adaptations the function must author. Each is a downstream chapter in this module.

### 4.1 Model-weight artifacts are not database rows

A 70B-parameter FP16 model weighs ~140 GB; a 405B-parameter model weighs ~810 GB. Replicating weights cross-region is a large-object replication problem, dominated by bandwidth and S3-class cross-region replication latency, not by transactional-replication semantics. The AWS Reliability Pillar's "cross-region replication" vocabulary applies, but the sizing — "replication lag budget" and "weight-propagation RTO" as a first-class planning metric — does not. Chapter 02 §3 writes it.

### 4.2 Cold-start is a first-class RTO cost

A serving tier whose model takes 8 minutes to load a shard-split 70B model has an *irreducible* 8-minute RTO on any cold-start path, regardless of how fast the control-plane recovery is. Warm standby is not a cost optimisation for AI serving; it is a correctness condition for any RTO tighter than the cold-start duration. The frameworks name warm standby as a generic strategy; chapter 02 §4 writes why it is often not optional for AI.

### 4.3 State reconstitution includes conversation and tool-call state

Classical DR literature treats "state" as transactional database rows. On an agent system, "state" includes the conversation history, the KV cache state that produced the next-token distribution, the in-flight tool-call stack, and the trace-store entries the next request will depend on. Each has distinct RPO sensitivity — the conversation history must survive a disruption, the KV cache need not. Chapter 03 writes the taxonomy and the per-class replication strategy.

### 4.4 Data residency constrains failover direction

NIST SP 800-34 §3.4 names "alternate processing sites" generically; the AWS multi-region guides implicitly assume any region may serve any traffic. GDPR, the EU AI Act, and the region-of-residence commitments teams make under customer contracts forbid the obvious failover topology: EU traffic may not silently failover to US infrastructure. Chapter 04 writes the residency-aware failover design.

### 4.5 Model-provider outages are an irreducible dependency

A team whose primary inference substrate is a hosted provider (Anthropic, OpenAI, AWS Bedrock, Google Vertex, Azure AI Foundry) inherits the provider's regional-outage posture. The function's DR plan must include *provider-level* failover, not only cloud-region-level failover. `mod-107` chapter 02 writes the general fallback contract; chapter 02 §5 of this module writes the DR-specific extension — the cross-region, cross-provider, residency-respecting playbook.

## 5. The design-time checklist

Before writing any of the AI-specific chapters, the function completes a framework-level pass against each workload in the charter. Produce, in Git:

1. A **classification tag** — tier-1 (customer-visible, strict RPO / RTO), tier-2 (internal, tolerant RPO / RTO), tier-3 (batch, best-effort) — assigned per service, read from the BIA.
2. A **shared-responsibility note** citing the vendor's own shared-responsibility documentation (AWS, Azure, or Google), identifying which failure classes sit on the vendor and which on the function.
3. The **chosen DR strategy** per service, named in AWS vocabulary (backup / restore, pilot light, warm standby, multi-site active-active) with a one-line justification against the service's RPO / RTO.
4. A list of the **AI-specific extensions** from §4 that apply, with a pointer to the chapter and section where the mechanics are written.
5. The **framework citations** backing each choice — a reader must be able to follow a link from the design note to NIST SP 800-34 §X, AWS Reliability Pillar §Y, or the equivalent Azure / Google reference.

A DR programme that fails to produce this pass is a programme that borrowed vocabulary from the frameworks without binding the vocabulary to the function's specific workloads. The next chapter — regional-outage playbook authoring — assumes the pass is done.

## Summary

- Four primary references anchor this module: NIST SP 800-34 Rev 1 for the process, and the AWS / Azure / Google Cloud reliability guides for the substrate-specific mechanics. The function reads all four and cites them by section.
- NIST SP 800-34's seven-step contingency-planning process and three-tier plan taxonomy (ISCP, DRP, BCP) give the shape of the function's DR programme. The function owns the ISCP, co-authors the DRP, and contributes the technical annex to the BCP.
- The three cloud reliability pillars agree on four DR strategies (backup / restore, pilot light, warm standby, multi-site active-active) and disagree on vocabulary. The function picks the vendor-relevant guide as the implementation reference and standardises the module on AWS vocabulary with mapping notes.
- Five AI-specific extensions — model-weight replication cost, cold-start-dominated RTO, agent state reconstitution, data residency in failover, model-provider dependency — are assumed by the frameworks but not written by them. Chapters 02–05 fill them in.
- A design-time checklist — classification, shared-responsibility note, chosen strategy, AI extensions, framework citations — is the function's gate from "read the frameworks" to "write the AI-specific DR mechanics."

Chapter 02 writes the first of those extensions: the regional-outage playbook for AI workloads, from DNS / Anycast / global-LB failover through to cross-region replication of the model-weight artifact store.
