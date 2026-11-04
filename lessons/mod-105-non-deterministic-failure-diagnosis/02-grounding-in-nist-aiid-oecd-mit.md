# Grounding the Catalog — NIST AI RMF, AIID, OECD, and MIT AI Risk Repository

## Why this chapter exists

Chapter 01 built a working ten-class taxonomy pragmatically — chosen for mutual distinguishability, routability, and operational speed. That taxonomy is load-bearing inside the on-call rotation. It is not, by itself, load-bearing in front of a regulator, an auditor, a compliance officer, a legal counsel reviewing an incident, or a sponsor asking "is this class of failure you keep talking about *a real class* or is it just how your team happens to label things?"

The answer has to be: both. The taxonomy is pragmatic *and* it maps, class by class, into vocabulary that external authoritative sources already use. This chapter draws the mapping. The four sources it uses:

- The **NIST AI Risk Management Framework (AI RMF 1.0)** and specifically the **Generative AI Profile (NIST AI 600-1)** — the U.S. standards body's canonical risk taxonomy for generative AI.
- The **AI Incident Database (AIID)** — the longest-running curated catalog of real-world AI harms, with class / type schemas.
- The **OECD.AI Incidents Monitor (AIM)** — the OECD's global incident tracker, which uses OECD-defined AI principles and hazard categories.
- The **MIT AI Risk Repository** — a cross-source meta-taxonomy that consolidates risks from 40+ frameworks into a unified schema.

Nothing in this chapter is a replacement for reading the primary sources; the chapter is an operational cross-walk. The on-call does not read any of these during an incident. The retrospective author, the compliance officer, the auditor, and the sponsor do.

## 1. NIST AI RMF and the Generative AI Profile (AI 600-1)

The NIST AI RMF is the United States' voluntary framework for managing AI risks, structured around four core functions: **Govern**, **Map**, **Measure**, **Manage**. The framework lives at <https://www.nist.gov/itl/ai-risk-management-framework> and the base document (AI 100-1) was published in January 2023.

Of operational interest here is the **Generative AI Profile**, published as **NIST AI 600-1** ("Artificial Intelligence Risk Management Framework: Generative Artificial Intelligence Profile") in July 2024. The Profile identifies risk categories specific to generative AI; the categorization is the vocabulary a regulator or compliance officer will expect to see mapped to incidents. The companion knowledge base lives at the AI Risk Management Framework Center: <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>.

The Generative AI Profile's risk categories include, among others:

- **Confabulation** (the Profile's term; often called "hallucination" in industry) — model producing erroneous or fabricated content.
- **Dangerous, violent, or hateful content** — production of unsafe content categories.
- **Data privacy** — including risks of training-data memorization, PII leakage, re-identification.
- **Harmful bias and homogenization** — discrimination, demographic skew in outputs.
- **Human-AI configuration** — risks from how humans interact with AI (over-reliance, under-reliance, automation bias).
- **Information integrity** — mis- and disinformation, deepfakes, impersonation.
- **Information security** — including risks related to model and system security.
- **Intellectual property** — IP leakage, infringement, training data provenance.
- **Obscene, degrading, and/or abusive content**.
- **Value chain and component integration** — risks from third-party models, data, and infrastructure.
- **Environmental impacts**.
- **CBRN information or capabilities** — chemical, biological, radiological, nuclear uplift risks.

<!-- needs-research: confirm the exact current list of NIST AI 600-1 risk categories and their canonical titles against the published Profile document; the list above is the shape reported in the published version but minor rewording across errata should be verified before citation in a regulated environment. -->

The Profile pairs each risk category with suggested actions under **Govern / Map / Measure / Manage**. For the AI SRE operating the measurement and management side, the relevant takeaway is: *your incident classes should be mappable to Profile risk categories*, because that is the vocabulary a U.S. federal risk review will use.

### 1.1 Cross-walk — chapter 01 classes to AI RMF GenAI Profile categories

| Chapter 01 class | NIST AI RMF GenAI Profile category | Notes |
|---|---|---|
| 2.1 Silent output drift | Not a direct category; rolls up under *Information integrity* or *Human-AI configuration* when the drift changes answer-correctness | Drift as an incident class is operational; NIST frames the harm it causes |
| 2.2 Hallucination spike | **Confabulation** | Direct mapping. The Profile is explicit that confabulation is the AI 600-1 term for hallucination |
| 2.3 Toxic / unsafe blast | **Dangerous, violent, or hateful content** and/or **Obscene, degrading, and/or abusive content** | Direct mapping; pick the category matching the content type |
| 2.4 Refusal spike | **Human-AI configuration** (over-refusal impairs legitimate use) | Softer mapping; refusal is NIST-adjacent, not a direct category |
| 2.5 Prompt-cache cliff | Not a NIST-named category; performance / cost / UX impact | NIST does not name infra-side failure classes directly |
| 2.6 Retrieval-freshness regression | **Information integrity** (stale/incorrect retrieval → incorrect output) | Indirect mapping via the output quality impact |
| 2.7 Agent-tool-call cascade | **Value chain and component integration** (tool composition risk) and/or **Information security** (uncontrolled tool access) | The Profile's value-chain category covers agent-tool composition |
| 2.8 Judge-model drift | Measurement-side risk; closest analogue is **Human-AI configuration** (the measurement is the human-AI contract) | NIST does not name judge drift directly; a measurement-risk mapping is defensible |
| 2.9 Provider-degraded mode | **Value chain and component integration** | Direct mapping; provider is a third-party component |
| 2.10 Policy / safety regression | Composition — whichever content-category applies, plus **Value chain** if provider-caused | |

The mapping is not always one-to-one and is not always clean. The point is that *every* class the on-call can declare in the incident channel has at least one defensible NIST GenAI-Profile category to name in the retrospective. The retrospective author translates; the on-call does not.

### 1.2 Why this matters operationally

Three operational consequences of the mapping.

- **Reports to leadership / legal use NIST vocabulary.** The incident summary that goes to the CISO, the GC, or the risk committee speaks in AI RMF terms, not in on-call slang. The retrospective template (chapter 08) carries a field for the AI RMF category.
- **Regulatory engagement is pre-wired.** Whether under the EU AI Act, NIST-aligned executive-order tracks, sector-specific guidance (HIPAA, FINRA, FDA), or voluntary frameworks, "we classified the incident as a Confabulation event per the NIST AI 600-1 taxonomy" is a stronger statement than "we called it a hallucination."
- **Metrics align with external benchmarks.** The AI RMF **Measure** function suggests metrics; aligning your SLIs (`mod-103`) to those metrics means your SLO burn rate is directly reportable upward.

## 2. The AI Incident Database (AIID)

The AI Incident Database is a community-maintained catalog of AI-related incidents, hosted by the Responsible AI Collaborative. It lives at <https://incidentdatabase.ai/>. The database curates public reports — press, academic, blog — of AI systems causing or nearly causing real-world harm. Each incident is assigned a numeric ID, a title, a narrative, source references, and a schema of entities involved.

The AIID uses two taxonomies: the **CSET AI Harm Taxonomy** (developed with Georgetown's Center for Security and Emerging Technology) and the **GMF Taxonomy** (Goals, Methods, and Failures). Both are described on the AIID website; the CSET taxonomy is particularly useful because it classifies *harm types* in a way that overlaps with operational incident classes.

### 2.1 How the AIID is useful to the on-call retrospective

The database is primarily a *case library*, not a day-of-incident tool. Operational uses:

- **Case pattern matching.** During retrospective, search AIID for historical incidents that pattern-match yours. "A recommender system silently drifted after a model update and drove a demographic bias increase" — there are cases. "A deployed LLM's refusal rate jumped after an upstream fine-tune" — there are cases.
- **External precedent for severity.** If a case in AIID with a similar shape caused regulatory action, media attention, or legal consequence, your internal severity classification should take that into account.
- **Chaos-engineering fodder.** `mod-109` uses the AIID as a prompt library for failure injection — "inject a failure of this shape and see if the fleet is ready."

### 2.2 Cross-walk — chapter 01 classes to AIID CSET harm categories

The CSET AI Harm Taxonomy classifies harms along axes: who is harmed, what kind of harm, what the AI did wrong. For the chapter 01 classes, the operational mapping:

| Chapter 01 class | AIID CSET-axis hint | AIID browse |
|---|---|---|
| 2.1 Silent output drift | "Performance degradation" or "Unexpected behavior" harm axis | Use AIID search for "model update" or "silent" + your domain |
| 2.2 Hallucination spike | "Information harm — incorrect information" or "Confabulation" | Numerous cases; "hallucination" / "fabrication" terms |
| 2.3 Toxic / unsafe blast | "Discrimination and bias," "Hate speech," "Harmful content" | Many cases; search by content type |
| 2.4 Refusal spike | "Access denial," "Service denial" | Fewer AIID entries directly; adjacent to bias-driven denial |
| 2.6 Retrieval-freshness regression | "Information harm — outdated information" | Search for RAG / retrieval cases |
| 2.7 Agent cascade | "Autonomous system failure," "Runaway cost" | Newer category; fewer historical entries, growing |
| 2.9 Provider-degraded | "Provider dependency failure" | Few directly-named cases but adjacent cases exist |

<!-- needs-research: confirm CSET AI Harm Taxonomy axis names and the AIID case taxonomy's current labels — the AIID has revised its taxonomy structure and the CSET framework has had multiple versions; exact axis names for the mapping above should be checked against the current taxonomy page before use in a regulated report. -->

### 2.3 Operational hygiene around AIID

- **Submit incidents where appropriate.** The AIID encourages submission of public-domain incidents. For incidents your organisation chooses to disclose, submission is a public-goods contribution and is encouraged in the NIST Govern function.
- **Do not cite AIID as a legal authority.** The database is community-curated; it is a research resource, not a regulatory register. Cite it as "a widely-used catalog of AI incidents."
- **Do not treat absence of an entry as absence of a class.** AIID coverage is uneven; the fact that no AIID entry matches your incident does not mean your class is unprecedented.

## 3. The OECD.AI Incidents Monitor

The OECD.AI Incidents Monitor is a tracker maintained by the OECD's AI Policy Observatory, lives at <https://oecd.ai/en/incidents>, and operationalises the OECD AI Principles (<https://oecd.ai/en/ai-principles>) by tracking real-world incidents against those principles. The AIM pulls from global media and NGO sources, categorises incidents against OECD-defined **hazard types**, and makes the dataset searchable.

The Monitor's hazard classification is explicitly international and principle-based. Its categorisations tend to be at a higher altitude than the AIID's and are especially relevant for incidents with cross-jurisdictional, policy, or international business impact.

### 3.1 Why the OECD source matters

Three operational reasons.

- **International regulators cite it.** Non-U.S. regulators and policy bodies lean heavily on OECD framings. Multi-region AI deployments (`mod-113` on DR / BCP / multi-region) will encounter OECD framings when engaging with European, Japanese, Korean, Australian, or OECD-member-country regulators.
- **Mapping to OECD AI Principles.** Each hazard maps to one or more OECD principles — **inclusive growth**, **human-centred values**, **transparency**, **robustness, security and safety**, **accountability**. Your incident retrospective that names the violated principle(s) aligns with international framings.
- **Trend signal.** The OECD aggregates volume and trend data. "Our incident is the Nth case of class X worldwide this quarter" is informative framing for sponsor and executive communication.

### 3.2 Cross-walk — chapter 01 classes to OECD AI Principles

| Chapter 01 class | OECD AI Principle(s) most directly violated |
|---|---|
| 2.1 Silent output drift | Robustness, security and safety; Transparency and explainability |
| 2.2 Hallucination spike | Transparency and explainability; Robustness |
| 2.3 Toxic / unsafe blast | Human-centred values and fairness; Robustness |
| 2.4 Refusal spike | Inclusive growth (over-refusal denies access to legitimate uses); Human-centred values |
| 2.5 Prompt-cache cliff | Not principle-aligned; infra-side |
| 2.6 Retrieval-freshness regression | Robustness; Transparency |
| 2.7 Agent cascade | Robustness; Accountability |
| 2.8 Judge-model drift | Transparency and explainability; Accountability |
| 2.9 Provider-degraded mode | Robustness; Accountability (value chain) |
| 2.10 Policy regression | Composite |

The OECD lens surfaces an on-call-visible tension: a *refusal spike* is a reliability failure that classical SRE would not name; the OECD principle "inclusive growth" names it as a *principle violation* because the service is now excluding legitimate uses. The on-call still handles the refusal spike the same way; the retrospective frames it against the principle for leadership.

## 4. The MIT AI Risk Repository

The MIT AI Risk Repository is a cross-framework meta-taxonomy that aggregates risks from ~40 source frameworks (NIST, OECD, EU AI Act drafts, academic taxonomies, industry frameworks) into two unified classifications: a **Causal Taxonomy** (what caused the risk — entity, intent, timing) and a **Domain Taxonomy** (what kind of risk — seven domains, 23 subdomains).

The Repository lives at <https://airisk.mit.edu/>. It is the single most useful cross-walk between taxonomies, because every risk entry cites which source framework(s) it derives from. For the AI SRE who is asked "how does this failure class map to the NIST and OECD taxonomies at once?" — the Repository is the pre-built answer.

### 4.1 The Domain Taxonomy and the chapter 01 classes

The MIT Domain Taxonomy's seven domains (with subdomain examples) include:

- **Discrimination and toxicity** (hate speech, unfair discrimination, misrepresentation).
- **Privacy and security** (exposing private information, security vulnerabilities).
- **Misinformation** (false or misleading information, pollution of the information ecosystem).
- **Malicious use and malicious actors** (fraud and scams, cyberoffense, disinformation campaigns).
- **Human-computer interaction** (over-reliance on AI, loss of human agency).
- **Socioeconomic and environmental** (systemic bias, labour displacement, environmental impact).
- **AI system safety, failures, and limitations** (lack of capability, robustness failures, unsafe behaviour, loss of control).

<!-- needs-research: confirm MIT AI Risk Repository domain and subdomain names against the current version of the Repository — the taxonomy has been updated across releases and the domain titles above should be verified before citation in external communication. -->

For the chapter 01 classes, the dominant MIT Domain is **AI system safety, failures, and limitations** for the drift / cliff / cascade / provider failures (2.1, 2.5, 2.6, 2.7, 2.8, 2.9), and **Discrimination and toxicity** / **Misinformation** / **Human-computer interaction** for the content-shaped failures (2.2, 2.3, 2.4).

### 4.2 Why the MIT Repository is the pragmatic go-to

- **One taxonomy that cites the others.** When a retrospective says "this incident is MIT domain 7.2 (robustness failure), which maps to NIST GenAI Profile confabulation, OECD robustness-and-safety principle, AIID CSET performance-degradation harm" — that is the full defensible mapping. The Repository provides the cross-walk.
- **Updated as frameworks evolve.** The Repository is actively maintained; it tracks framework revisions.
- **Research-grade.** The Repository is academic, peer-reviewed-adjacent, and widely cited. It survives external scrutiny.

## 5. The composite cross-walk — a single table

Pulling the four sources together, the operational composite is:

| Chapter 01 class | NIST GenAI Profile (AI 600-1) | AIID / CSET harm axis | OECD AI Principle | MIT Domain |
|---|---|---|---|---|
| 2.1 Silent output drift | Information integrity (indirect) | Performance degradation | Robustness | AI system safety / failures |
| 2.2 Hallucination spike | Confabulation | Information harm — incorrect | Transparency; Robustness | Misinformation; AI system safety |
| 2.3 Toxic / unsafe blast | Dangerous / violent / hateful; Obscene / degrading | Discrimination / hate / harmful content | Human-centred values | Discrimination and toxicity |
| 2.4 Refusal spike | Human-AI configuration | Access denial | Inclusive growth | Human-computer interaction |
| 2.5 Prompt-cache cliff | None direct | None direct | None direct | AI system safety / failures |
| 2.6 Retrieval-freshness regression | Information integrity | Information harm — outdated | Robustness; Transparency | Misinformation; AI system safety |
| 2.7 Agent cascade | Value chain and component integration | Autonomous system failure | Robustness; Accountability | AI system safety / failures |
| 2.8 Judge-model drift | Human-AI configuration (measurement) | Measurement/evaluation failure | Transparency; Accountability | AI system safety / failures |
| 2.9 Provider-degraded mode | Value chain and component integration | Provider dependency failure | Robustness; Accountability | AI system safety / failures |
| 2.10 Policy / safety regression | Composite | Composite | Composite | Composite |

This is the table you paste into your retrospective template, with the specific classes for the incident filled in.

<!-- needs-research: before shipping this table into a compliance-grade retrospective template, confirm each NIST GenAI-Profile category name and each MIT Domain name against the current versions of the respective publications. These frameworks have had revisions and errata since initial publication; the mapping *shape* is stable but the exact labels should be verified. -->

## 6. The boundary — what external taxonomies do and do not give you

The four sources are about *harm framing*, not about *operational diagnosis*. The following are things the taxonomies do *not* give you, and that this module provides instead:

- **Primary signal.** NIST does not tell you which Prometheus metric to query; MIT does not tell you which OTel attribute to slice on. Chapter 01 does. Chapters 03–08 deepen.
- **Suspect owner.** NIST does not know who your inference-engine peer is. The peer-boundary content in `mod-101`, `mod-104` chapter 07, and this module's chapter 08 does.
- **Time-to-recognition budget.** The external taxonomies do not operationalise detection timing. Your on-call rotation does.
- **The specific text to paste into an incident channel.** The on-call types "class 2.5, prompt-cache cliff, routing to engine peer"; they do not type "AI 600-1 Information integrity risk" at minute two. The retrospective does the translation.

The on-call speaks the operational taxonomy. The retrospective, the regulator-facing report, the sponsor update, and the audit response speak the external taxonomies. The incident lifecycle (built in `mod-108`) carries the translation at defined handoff points.

## 7. One worked example — retrospective passage

To make the cross-walk concrete, here is a short retrospective passage that uses the operational taxonomy and the external one together:

> **Incident Summary.** On 2026-Mon-DD at HH:MM UTC, the quality SLI for the `chat-support-v1` surface breached its 30-day budget at a 14× burn rate. On-call declared class **2.1 — silent output drift**, sub-class model-weight change, at HH:MM+7.
>
> **Classification for external framing.** Under NIST AI 600-1, this incident is a *Confabulation* risk event (the drifted model began producing factually-incorrect answers at 2.3× the baseline rate). Under the MIT AI Risk Repository, it is a Domain 7 (AI system safety, failures, and limitations) event, subdomain *robustness failure*. Under the OECD AI Principles, the primary principle violated is *Robustness, security and safety*. The AIID has no directly-matching case entry; the closest pattern is incidents `#NNN` and `#MMM` (recommender-system model-update drift cases).
>
> **Root cause.** The ML platform promoted a new model version at HH:MM-02; the version's output distribution on the production query mix had not been pre-measured against the quality SLI baseline. The release-engineering gate from `mod-111` did not include a canary-eval signal that would have caught this before promotion.
>
> **Routing.** Ownership: ML platform peer (promotion), eval peer (baseline / judge calibration), release engineering (gate).
>
> **Action items.** (1) Pre-promotion canary-eval gate wired into the registry promotion flow. (2) Judge-pipeline recalibration against the new version. (3) Retrospective action tracked in MIT Domain 7 subdomain tag.

The operational classification (class 2.1) was declared in the incident channel. The external classification was added during retrospective. Both are in the same document, visible to both the SRE team and the compliance / legal / regulatory stakeholders.

## Summary

- **Four authoritative sources.** NIST AI RMF and the GenAI Profile (AI 600-1); the AI Incident Database with CSET harm taxonomy; the OECD.AI Incidents Monitor and AI Principles; the MIT AI Risk Repository with its Causal and Domain taxonomies.
- **Each chapter-01 class maps to vocabulary in these sources.** The cross-walk table (section 5) is the composite; no class leaves the on-call unable to translate upward.
- **Operational vs. external vocabularies live side by side.** The on-call speaks the operational taxonomy; the retrospective and regulator-facing surfaces speak the external one. The lifecycle carries the translation.
- **The external sources do not replace this module.** They frame harm; this module's chapters do diagnosis, routing, and recognition-time. Both are needed.

The next chapter opens the first and most-general class — silent output drift — and builds the detection discipline (canary-set evals, output-distribution comparison, cost and latency as proxy signals).
