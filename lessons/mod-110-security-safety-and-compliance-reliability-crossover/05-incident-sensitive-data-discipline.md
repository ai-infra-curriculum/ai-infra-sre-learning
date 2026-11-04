# Incident-Sensitive Data Handling Discipline

## Why this chapter exists

Incidents on model-backed services routinely surface **content** — the prompts users sent, the completions the model produced, the retrieval chunks the model used, the tool calls the agent made. In classical SRE, the equivalent surfaces are logs: structured, pre-redacted, indexed, retained to a documented policy. In AI SRE, the surface is narrative text that may contain personally identifiable information, regulated-data category markers, intellectual property, safety-adjacent content, or outputs that themselves constitute a reportable event.

Mishandling this surface is a repeat failure mode on first-generation AI SRE functions:

- A team pastes a full prompt + completion into the open incident channel because the moderator classifier fired; the content contained PII that is now in the channel's retention.
- A debugging engineer exports a RAG-corpus slice to a personal notebook to reproduce an issue; the slice contains customer-confidential documents.
- An on-call engineer forwards a toxic-output sample to the security peer via Slack DM; chain of custody is broken and the sample now exists in two retention schedules.
- An incident channel is archived at the retention window because no one marked the content as legal-hold-retained; artifacts needed in a later compliance filing are irrecoverable.

The AI SRE function owns the **operational discipline** that prevents these outcomes without taking on the responsibilities of SecOps / IR (who own evidence handling for security incidents) or legal (who own retention and disclosure determinations). This chapter writes that discipline. The exercise for this chapter (exercise 05) produces the function's operational runsheet. Chapter 03 of this module already named the retention surface at the compliance level; this chapter governs the hour-by-hour operational practice that keeps the compliance surface honest.

## 1. The classes of incident-sensitive data

Not all AI incident data is equally sensitive. The function classifies by category up-front so the handling rules — redaction, access, retention — are applied consistently:

- **Content data.** Prompts, completions, retrieval chunks, tool-call arguments and returns, conversation history. May contain PII, regulated-data markers, customer-confidential material, safety-adjacent content.
- **Classifier decision data.** The output of guardrails, moderation classifiers, policy-engine evaluations — scores, labels, decisions. Often contain content snippets as an evidentiary field.
- **Model-internal data.** Attention maps, logit snapshots, intermediate activations when debugging requires them. Less common operationally; when present, may be IP-sensitive.
- **Provider interaction data.** Request / response payloads with the model provider. The responses may contain any of the above classes; the requests may contain customer data under a ZDR / no-training contract.
- **User-report data.** Content a user has forwarded to support alleging an incident. Legal precedent is complicated: the content is simultaneously the user's and (once reported) a record the organisation is responsible for.
- **Operational telemetry.** Metrics, traces, non-content logs. Mostly non-sensitive, but tenant identifiers, hashed user IDs, and pattern-level telemetry can become re-identifying at scale.

The §2 discipline applies uniformly; §3 scales the discipline to the class.

## 2. The five principles of handling

Five rules, pre-committed out of incident. The function's on-call runbook repeats them so a 3 AM incident does not have to re-derive them.

### 2.1 Minimum necessary content

If the question can be answered without the full content, use the minimal substitute. A classifier spike is debuggable from the classifier's decision distribution, the input length distribution, and the model / route identifier — not from the content itself. The function reaches for content only when the non-content surfaces are insufficient.

Three substitutes the discipline recommends in order of preference:

- **Hashed or tokenised references.** A `prompt_hash = sha256(...)` is enough to correlate across the pipeline without revealing content.
- **Structural features.** Length, language, cluster membership, tool-call shape, classifier scores — enough to describe the shape without reproducing the content.
- **Redacted content.** Named entities replaced with `[PERSON_1]`, numeric identifiers hashed, free-text generalised. A redacted sample is still content, but its re-identification risk is reduced.

Full content is the last resort, not the first.

### 2.2 Named storage, not ambient channels

Content, when captured, lands in a **named store** — a specific object-locked bucket, a specific controlled-access incident-artifact workspace — not in the incident channel, not in email, not in a shared drive link, not in a DM. The named store has:

- RBAC with a documented role list.
- Access audit (who opened what, when).
- Retention policy aligned to legal's determination.
- Legal-hold capability (artifacts can be pinned from deletion on legal's instruction).

The `mod-108` chapter 06 §5 preservation rules bind here; this chapter extends them with the operational discipline.

### 2.3 Chain of custody

For any artifact that may be presented externally — to a regulator, a counterparty, a safety committee — the function maintains chain of custody:

- Who captured the artifact.
- From where (dashboard, log store, trace, user report).
- When (UTC timestamp).
- How the artifact was transferred into the named store.
- Who has accessed it since, when, and for what purpose.

The chain lives in the artifact's metadata in the named store, not in an engineer's head. SecOps / IR owns the chain for security-flagged artifacts; the AI SRE function mirrors the discipline for AI-safety-adjacent artifacts so the two surfaces interoperate cleanly.

### 2.4 Legal determines retention

The function does not decide retention windows. Legal does. The AI SRE function's commitments:

- **Default retention** on incident artifacts is set to the longest plausible legal window (e.g., seven years in the US financial-services case, five years in the EU AI Act case if the Act applies — <!-- needs-research: confirm the exact retention expectations under Art. 72 for high-risk AI systems and whether the implementing act specifies a baseline retention period -->), erring long when uncertain.
- **Legal-hold override.** When legal issues a hold on specific artifacts, those are exempt from any automated deletion the storage system would otherwise perform. The function maintains a legal-hold register.
- **Deletion requires clearance.** No artifact is pruned without a legal sign-off on the pruning. The function's quarterly data-handling review (§7) is the regular channel for clearance requests.

### 2.5 Selective sharing with SecOps / IR

When an artifact needs to move to a peer function — SecOps / IR, the safety peer, `ai-risk-engineer`, legal — the function uses a **selective-sharing** pattern, not ambient forwarding:

- A specific artifact ID (not a copy) is granted to the peer's role in the named store.
- The access is time-bounded (e.g., read-only for 30 days, re-grantable on request).
- The sharing event is logged in the chain of custody.
- The peer's handling is their function's responsibility from that point; the AI SRE's role is to make the share auditable, not to govern the downstream handling.

A copy-paste share (into a different channel, a different tool, a different account) breaks the chain and creates two retention schedules. The function refuses the pattern.

## 3. Applying the discipline to each data class

### 3.1 Content data

- **At capture:** redact at the point of ingestion if possible. Hash prompts, mark detected entities, store the original behind an elevated access role.
- **During debugging:** prefer the hashed and redacted form. Elevate to the raw form only with a documented justification and under the chain-of-custody trail.
- **At rest:** named store with role-gated access, long default retention, legal-hold capability.
- **At share:** selective-sharing pattern.
- **Special subcategory — toxic content:** stored in a specifically isolated store ("safety-samples store") with a narrower access role list, because the content itself is operationally hazardous to re-expose (`mod-108` chapter 02 §5.3 is explicit about not pasting toxic samples into an open incident channel).

### 3.2 Classifier decision data

- **At capture:** decisions are telemetry; retain per `mod-104` with the content field redacted. Keep the raw-content pointer separately behind an elevated access role.
- **At share:** the decision distribution and the structured decision record are shareable widely; the raw-content field travels only under §2.5.
- **Special case:** the classifier's own false-positive / false-negative sample set is an operational artifact the security peer may need; selective-sharing applies.

### 3.3 Model-internal data

- **At capture:** rare, elevated. Requires a documented debugging purpose and the model-owner's sign-off.
- **At rest:** short retention by default (weeks), unless the debugging is tied to an incident that elevates to compliance surface.
- **At share:** read-only, with IP review if the data leaves the function's trust boundary.

### 3.4 Provider interaction data

- **At capture:** the function captures this as part of operational telemetry. Content fields are redacted or hashed at the capture layer unless an incident justifies the raw form.
- **At rest:** named store, retention aligned to the customer contract (ZDR / no-training commitments colour retention here — a ZDR contract that includes a retention cap means the function **cannot** keep raw request content beyond the cap, which the on-call must know).
- **At share with the provider:** governed by the vendor contract (`mod-107` chapter 02). The provider's incident-handling norms may require the AI SRE to share a specific request / response pair; the sharing is logged in the chain of custody.

### 3.5 User-report data

- **At capture:** the user has sent the content; the function captures it with the user's identifier attached and the report's full context (which support channel, which user, when).
- **At rest:** named store, extended retention because the report may become evidence in a later proceeding.
- **At share:** selective-sharing, with explicit note that the artifact is "user-reported" so the receiving function understands its evidentiary character.
- **Special case:** if the reported content itself is a regulated event (CSAM, credible threat, etc.), the handling is governed by the SecOps / IR and legal protocols for those events, which supersede this discipline and which the AI SRE function does not re-author.

### 3.6 Operational telemetry

- **At capture:** standard `mod-104`.
- **At rest:** retention per chapter 03 of this module (compliance signals) or per the function's operational retention policy for non-compliance signals.
- **At share:** widely shareable within the organisation; care is needed for cross-tenant correlation (per-tenant signals re-identify tenants).

## 4. The operational runsheet, in order

During an incident, the on-call executes the runsheet in order. A clean sequence prevents the ambient-channel failure mode.

1. **Classify.** Which §1 classes of data does this incident touch?
2. **Reach for substitutes first.** Does the diagnosis require content? If not, use hashes and structural features.
3. **If content is required, capture to the named store.** Not to the incident channel. The capture path is in the runbook; the engineer does not improvise.
4. **Mark the artifact ID in the incident channel.** The channel references the artifact ID (`incident-artifact-2026-11-04-chat-route-sample-a7f3`), not the content itself.
5. **Record the chain of custody.** Capture-time, capture-source, capture-engineer. The store's intake form enforces this.
6. **If sharing is needed, use selective-sharing.** Grant access; do not forward.
7. **When the incident resolves, verify the chain is closed.** All artifacts in the store, all accesses logged, no ambient copies.
8. **At the retrospective, enumerate artifacts.** The §4 artifact inventory from `mod-108` chapter 06 is filled from the named store, not reconstructed from memory.
9. **At the retrospective, hand off legal-hold decisions.** Legal confirms which artifacts are held; the function's legal-hold register is updated.
10. **If a trigger from §3 of this module's chapter 03 fires, proceed to the handoff.** Chapter 06 of `mod-108` governs the handoff mechanics; this discipline makes the handoff's evidence base complete.

## 5. The specific anti-patterns

The following patterns have broken the chain on real functions and are named so the on-call recognises them:

- **The "I'll just paste this one sample"** anti-pattern. A single sample into an incident channel is still a leak into the channel's retention; and the channel's access list is wider than the named-store's role list.
- **The "I'll screenshot this to a note"** anti-pattern. Screenshots evade the audit path. They are a copy outside the chain.
- **The "I'll forward this to security"** DM anti-pattern. A DM to the security peer is ambient forwarding. §2.5 is the correct path.
- **The "we don't need to retain this, it's noise"** anti-pattern. The function does not make retention decisions in-incident. Legal does, out-of-incident.
- **The "let's move the channel to archive"** anti-pattern. Archiving an incident channel without checking the legal-hold register deletes retained artifacts prematurely.
- **The "we can't share this with them, it's PII"** anti-pattern. SecOps / IR may be authorised to see PII in a security-flagged incident. The function does not gate downstream peers out of privacy intuition; it uses selective-sharing with the chain-of-custody audit.
- **The "we have to share this with them, they asked"** anti-pattern. Peer requests do not override the sharing rules. Requests follow §2.5's path.

A function that sees any of these patterns in a retro files a fixture (chapter 05 of `mod-108`) to prevent recurrence.

## 6. The tooling that makes the discipline tractable

The discipline above is unworkable without tooling. The function invests in a small set of capabilities:

- **A named artifact store** with object-lock, RBAC, access audit, legal-hold capability, and a programmatic API for capture.
- **A redaction library** that the capture-path uses by default. Entity-recognition-based, policy-driven, deterministic.
- **A chain-of-custody workflow** integrated into the incident tooling. Opening an artifact to view it logs the access automatically; no separate book-keeping.
- **A "safety-samples" sub-store** with tighter access control than the general incident artifact store, for toxic / sensitive content that should not be visible even to the broader IR rotation.
- **A legal-hold register** with a simple API: `hold(artifact_id, reason, duration)`. Legal operates it; engineering reads it.
- **Automated share-expiry** so the time-bounded sharing in §2.5 enforces itself.
- **Dashboards for the data-handling itself.** How many artifacts exist, how many are on legal hold, how many shares are active, how many are overdue for expiry. The dashboards are reviewed in §7.

The function does not build this tooling from scratch where existing SecOps / IR tooling applies. `mod-108` chapter 07's peer map is the guide for which tool belongs to which function; overlap is reconciled explicitly rather than by independent build.

## 7. The quarterly data-handling review

Once a quarter, the function reviews the data-handling posture:

- **Artifact inventory.** How many incident artifacts are retained? Which legal-hold windows are active or expiring?
- **Access-audit review.** Who accessed what, when, for what documented reason? Anomalies (an engineer accessing an artifact unrelated to their incident, unexpected access volume) are flagged.
- **Pruning clearance requests.** Which artifacts are candidates for deletion past their retention? Legal clears the batch.
- **Policy updates.** Did a regulator action, a customer contract change, or a legal recommendation shift the policy? If yes, the policy is updated, the runsheet is revised, and the on-call rotation is briefed.
- **Tooling health.** Named-store availability, redaction-library false-positive / false-negative rate, chain-of-custody workflow usage.
- **Runbook refresh.** Any anti-pattern observed since the last review becomes a runbook amendment.

The review has an AI SRE, a SecOps / IR, and a legal / compliance participant. Its output is a dated summary in the compliance archive.

## 8. The role boundaries, repeated

To keep the discipline aligned with the broader peer boundaries from `mod-108` chapter 07 and chapter 06 of this module:

- **SecOps / IR** owns evidence handling for security-flagged incidents. The AI SRE's discipline is designed to interoperate with SecOps / IR's chain of custody, not to replace it.
- **Legal** owns retention and disclosure. The AI SRE records what legal commits to; the function does not set the windows.
- **The safety / policy peer** owns the safety-adjacent interpretation of content. The AI SRE preserves and routes; the peer interprets.
- **`ai-risk-engineer`** consumes the chain-of-custody record as part of the compliance narrative; does not operate it.
- **The AI SRE** owns the in-incident capture discipline, the named store's operational health, the chain-of-custody workflow, the selective-sharing API, and the quarterly review's SRE-side input.

Nothing above is co-owned on its implementation; everything above is jointly consumed.

## Summary

- **AI incident data falls into six classes** — content, classifier decisions, model-internal, provider interactions, user reports, operational telemetry — each with its own sensitivity and handling.
- **Five principles bind every class** — minimum necessary content, named storage not ambient channels, chain of custody, legal-set retention, selective sharing with peers.
- **The operational runsheet** sequences the discipline so an on-call executes it reliably at 3 AM.
- **The anti-patterns at this surface are specific and preventable** with the runsheet and the tooling investment.
- **Tooling makes the discipline tractable** — named store with object-lock, redaction library, chain-of-custody workflow, safety-samples sub-store, legal-hold register, automated share-expiry, data-handling dashboards.
- **The quarterly data-handling review compounds the practice** with a mix of AI SRE, SecOps / IR, and legal / compliance participants.
- **Boundaries hold** — SecOps / IR owns security-incident evidence; legal owns retention and disclosure; the safety peer interprets content; the AI SRE operates the discipline that makes all their work possible.

Chapter 06 closes the module by drawing the peer-boundary map for the whole crossover — adversarial defence methodology at `ai-infra-security`, AI risk management framework ownership at `ai-risk-engineer`, legal / regulator-facing output at legal / compliance — so the module composes with the track without over-reaching.
