# Joint Post-Mortems with the Security Peer

## Why this chapter exists

Three classes of incident on a model-backed service require a post-mortem that neither the AI SRE alone nor the `ai-infra-security` peer alone can write:

1. **A defensive-control change caused a reliability regression.** The security peer shipped a new classifier, bumped a rate-shaper threshold, enabled a policy rule — and p99 regressed, or a route started intermittently 504-ing, or the fallback contract from `mod-107` stopped holding.
2. **A reliability remediation weakened the defensive posture.** The AI SRE disabled a sidecar to recover a serving regression, raised a wall-clock timeout past the window where an injection-suppression heuristic worked, or widened a fallback path in a way that defeats a data-residency rule.
3. **A shared root cause touched both domains.** A retrieval index was poisoned by a malicious upload *and* produced quality regressions on unrelated queries; a provider's silent model swap *and* their changed classifier behaviour both landed in the same outage; a prompt-template change introduced a prompt-leakage vector *and* an inference cost blow-up at the same time.

For each shape, the single-function post-mortem — the shape chapter 03 of `mod-108` writes — produces a document that is either technically correct and politically unreadable, or politically careful and technically incomplete. A joint document avoids both failure modes by committing, out of incident, to a shape where each function's facts live in their function's voice and the ownership of each action item is unambiguous.

This chapter writes that shape. It does not replace the post-mortem template in `mod-108` chapter 03; it extends it. The exercise for this chapter (exercise 02) produces the actual template a function can file into its post-mortem tooling.

## 1. The three incident shapes, named

Before the template there is vocabulary. Each of the three shapes produces different retrospectives, action-item classes, and leadership readouts, so the function labels them explicitly and chooses the shape in the first fifteen minutes of the retro.

### 1.1 Shape A — "defensive change caused reliability regression"

The archetype: `ai-infra-security` deployed version N+1 of the prompt-injection classifier. It passes its functional eval and its precision/recall bar. In production the classifier's p99 is 400 ms where N's p99 was 80 ms — the model path on cold-cache tokens is slower. The route's TTFT p99 crosses its SLO. The on-call AI SRE revert-boxes the classifier to N. Incident resolves.

Features of Shape A:

- **The change was visible to one function but not the other.** The security deploy was tracked in a security-owned CI, the SLO stack never saw a canary stage, the AI SRE function learned about the change from the regression, not the deploy.
- **The fix was a revert, not a redesign.** The classifier is good; its budget was wrong.
- **The missing artifact is the contract from chapter 01 §2.** Had the per-hop budget been signed, the deploy would have failed the §2.3 fast-burn alarm in canary.
- **The blame is structural, not individual.** Neither function's leader pushed the other out of bounds; both operated without the contract that would have caught it.

### 1.2 Shape B — "reliability remediation weakened defensive posture"

The archetype: during a model-provider degraded event (chapter 04 of `mod-107`), the on-call AI SRE routes traffic to the secondary provider. The secondary provider is contracted but was not in the week's defensive test pass; its response format differs in a way the output moderation classifier does not handle, and its rate-limit behaviour is more permissive. During the degraded window, 1.2% of requests passed outputs that would have been blocked on the primary. Incident is resolved on reliability axis within the fallback contract's SLO. The security peer discovers the moderation gap two days later from an audit of the classifier's decision log.

Features of Shape B:

- **The reliability remediation worked** — the fallback contract held, serving SLOs were honoured.
- **The defensive gap was invisible on the AI SRE's dashboard** — moderation precision on a fallback provider was not a reliability SLI.
- **The fix is cross-cutting.** Either the moderation stack must be provider-aware, or the fallback contract must include a defensive-parity clause, or the data-residency / class rules must gate fallback eligibility.
- **The blame is a missing boundary.** The AI SRE's authority to trigger the fallback did not include authority over defensive posture, but no one had explicitly named that gap.

### 1.3 Shape C — "shared root cause"

The archetype: a RAG corpus is updated nightly from a tenant-controlled upload path. A tenant ingests a document containing a chunked injection payload that targets the retrieval model's embedding and the downstream generator's instruction-following. For a two-day window, that document is retrieved for 0.3% of queries and produces either an injection-style response or a quality regression (depending on the downstream generator's acceptance of the payload). Detection comes from a mix of a judge-pipeline quality drift signal and a few user reports of "weird answers" containing hidden instructions.

Features of Shape C:

- **The root cause is upstream of both functions' control planes.** Neither the serving stack nor the defensive stack was itself broken; the ingest pipeline fed both a poisoned artifact.
- **The quality signal and the safety signal correlate.** Both functions' dashboards tilted; neither's alarms tripped alone with sufficient specificity to call it.
- **The fix is pipeline discipline.** Ingest-time validation, chunk-boundary instruction sanitisation, per-tenant corpus quarantine — none of which are classifiers in the serving path.
- **The ownership is joint by default.** Both functions wear the retrospective; neither can disown it.

A fourth shape — "reliability remediation triggered a compliance event" — is structurally similar to Shape B but with legal as the third seat; chapter 05 of this module handles its specifics.

## 2. The template, extended from `mod-108` chapter 03

The joint post-mortem reuses the §2 shape of `mod-108` chapter 03 (one incident, past tense, actor-role voice, timeline from signal to resolution) and adds the following sections. Appearing in both the SRE's and the security peer's filing systems requires one document, one URL, one history. Dual-owned documents fork; the template defends against that with explicit co-author fields.

### 2.1 Header

```
Incident: <id>
Shape: A | B | C | other
Primary surface(s): <reliability | defensive | compliance | multiple>
AI SRE co-author: <name>
Security co-author: <name>
Legal reviewer (if triggered): <name>
Status: Draft | In review | Published
Window (UTC): <start> → <stable> → <resolved>
```

The `Shape` field forces the first-fifteen-minutes classification from §1. It drives the template branches in §3.

### 2.2 Narrative (joint)

The narrative is one text, drafted by whichever function held primary incident command, reviewed line-by-line by the other. The §2.1 chapter 03 rule — observation and inference separated — applies. A line that reads "the classifier was too slow" is fused observation and inference; the joint template rewrites this as:

> At 14:03 UTC, the prompt-injection classifier emitted a p95 of 410 ms over a 30-second window (dashboard: `grafana://guardrail/hop/prompt-injection`). The signed p95 budget for this hop is 25 ms (contract: `kb://sre-security/guardrail-budget-chat-route`). The 16× budget breach is the first-stage cause of the TTFT p99 regression observed at 14:04.

Observation (metric + source) and inference (what that implies for this incident) stay separated. Both authors sign the narrative.

### 2.3 Function-specific detail blocks

Three detail blocks follow the narrative, each in its function's voice:

- **AI SRE detail — serving/latency/availability.** The reliability facts: which SLOs burned, which runbooks fired, which mitigations were tried, which failed. Written by the AI SRE co-author.
- **Security detail — defensive posture.** The defensive facts: which controls were affected, which were bypassed, which alerts fired, which did not, what the classifier's decision log shows for the window. Written by the security co-author.
- **Joint detail — shared root cause / interaction.** The third block is the one that only exists on multi-function incidents: the facts that belong to both or to the seam. For Shape A, the seam between "classifier deploy process" and "serving SLO alarm." For Shape B, the seam between "fallback contract" and "defensive-parity requirement." For Shape C, the seam between "ingest discipline" and "serving/defensive surface."

The three blocks lie side by side in the document. Neither owns the other.

### 2.4 Root cause (classified)

The root-cause classifier extends the `mod-108` chapter 03 §3 taxonomy (model-weight, kernel, scheduler, prompt / config, retrieval, provider, agent-cascade, eval-drift) with the joint categories:

- **defensive-change-untested-budget** (Shape A archetype).
- **defensive-change-untested-latency-tail** (Shape A variant where the p95 was fine but the p99 breached).
- **fallback-without-defensive-parity** (Shape B archetype).
- **fallback-timeout-defeats-defensive-heuristic** (Shape B variant).
- **ingest-poisoning-cross-surface** (Shape C archetype).
- **provider-change-cross-surface** (Shape C variant: provider's model and provider's classifier both changed).
- **shared-tool-call-path-cross-surface** (Shape C variant: agent tool behaviour defeated both a reliability cap and a defensive rule).

A root cause that cannot be classified on the list is a candidate for a new taxonomy entry, reviewed in the quarterly joint retrospective (§5).

### 2.5 Action items (ownership-tagged)

Action items in a joint post-mortem have a mandatory `Owner` and `Owning-function` field. Representative shape:

| ID | Action | Owner | Function | Due | Fixture-class |
|---|---|---|---|---|---|
| AI-001 | Author `rb-guardrail-hop-latency-breach` | A. Rao | AI SRE | 2026-11-18 | runbook |
| AI-002 | Add fast-burn alarm on classifier hop p99 | A. Rao | AI SRE | 2026-11-18 | alert |
| SEC-001 | Add canary stage to classifier deploy | K. Yoon | Security | 2026-11-25 | gate |
| SEC-002 | Add p99 panel to security operational dashboard | K. Yoon | Security | 2026-11-18 | dashboard |
| JOINT-001 | Sign per-hop budget contract for this route | A. Rao + K. Yoon | AI SRE + Security | 2026-11-18 | contract |

Three rules:

- **No action item has a joint owner on its implementation**; "jointly owned" items are decomposed into function-owned sub-items. `JOINT-001` above is a signing action, not an implementation action.
- **Every action item ties to a fixture class.** The six classes from `mod-108` chapter 05 (alert, canary eval case, eval-set entry, load-test shape, chaos experiment, runbook update) are joined by three joint-specific classes: contract, gate, dashboard.
- **Every action item has a due date measured in days / weeks**, not quarters. A post-mortem whose action items complete in Q+1 does not change the system the function operates.

### 2.6 Lessons (joint and function-specific)

Two lesson blocks, side by side:

- **Shared lessons.** What the organisation learned about the seam between the two functions. Short (3–6 bullets). These are the lessons the quarterly joint retro (§5) consumes.
- **Function-specific lessons.** Reserved for lessons that only apply to one function and that the function's own retrospective cycle will track. A joint post-mortem's shared lesson block carries more weight than its function-specific blocks; a joint post-mortem whose function-specific blocks swallow its shared block is a document the shape failed to produce.

## 3. The shape-specific branches

The joint template has three branches, one per incident shape. Each branch names the facts that must appear, the artifacts the retro must reference, and the action-item classes most likely to land.

### 3.1 Shape A branch — defensive change caused reliability regression

**Must-name facts:**

- The specific control that changed (version, deploy time, author).
- The per-hop latency contract in force before the change (§2.1 of chapter 01).
- The measured per-hop latency after the change (histogram link).
- The SLO burn trajectory and its budget at the point of mitigation.
- The release process the change followed and the gates it passed.

**Must-reference artifacts:**

- The signed budget contract (or its absence).
- The security peer's eval report for the change (link).
- The AI SRE's SLO burn dashboard for the window.

**Expected action-item classes:**

- Budget contract (if missing or stale).
- Canary gate at the defensive deploy level (per chapter 01 §2.4).
- Fast-burn alarm on the per-hop budget.
- Joint shared dashboard panel.

### 3.2 Shape B branch — reliability remediation weakened defensive posture

**Must-name facts:**

- The remediation taken (fallback engaged, timeout widened, control disabled, worker breaker tripped).
- The defensive controls affected by the remediation and the way they were affected.
- The scope of the defensive gap in terms of requests, users, data classes, durations.
- Whether a compliance trigger (chapter 03 of this module; chapter 06 of `mod-108`) fires. If yes, legal is a co-reviewer.

**Must-reference artifacts:**

- The fallback contract (`mod-107` chapter 02) for the route.
- The defensive-parity annex to the fallback contract (which may not yet exist — if so, that is the primary action item).
- The classifier / policy decision log for the window.

**Expected action-item classes:**

- Defensive-parity annex authored for the fallback contract.
- Provider-aware defensive stack (if the gap was in provider differences).
- Chaos experiment added to `mod-109`'s provider-fallback-contract suite that verifies defensive parity under the fallback (not just reliability).
- Pre-committed decision tree update for the IC runbook (chapter 02 of `mod-108`).

### 3.3 Shape C branch — shared root cause

**Must-name facts:**

- The upstream artifact that was poisoned / drifted / changed (document, model version, template, tool contract).
- The two downstream signals (one reliability-coloured, one defensive-coloured) and how long each existed before correlation.
- The detection path that eventually connected them and the time to correlation.

**Must-reference artifacts:**

- The ingest / release / provider-change audit trail.
- Both functions' detection dashboards for the window.
- Any user reports or support escalations that preceded the detection.

**Expected action-item classes:**

- Ingest discipline (validation, sanitisation, quarantine).
- Cross-surface detection (a signal that fires when a reliability tilt and a defensive tilt correlate).
- Shared dashboard panel designed to surface the correlation.
- Joint runbook (`rb-cross-surface-corpus-event`-shaped) added to the library in `mod-108` chapter 01.

## 4. The workflow around the template

Having a template is necessary but not sufficient. The function commits to a workflow that produces the joint document, on time, with both signatures.

### 4.1 Trigger and authorship

- A triggering signal from the chapter 06 of `mod-108` taxonomy is the default trigger: the "crossover" subset of that taxonomy (compliance-trigger conditions, safety-trigger conditions that overlap with reliability) nominates the incident for joint retro.
- When the incident command (chapter 02 of `mod-108`) declares the incident, the IC's first calls after the on-call rotations include a page to the security peer's on-call if any of §1's shapes seem to apply.
- Authorship is co-decided in the first hour. Default: the function with primary incident command drafts the narrative; the other co-authors inline on the shared document.

### 4.2 Drafting cadence

- **Day 0 — Timeline and narrative skeleton.** Both functions contribute their observations. Timeline is time-ordered and merged.
- **Day 1–2 — Function-specific detail blocks, root cause draft, candidate action items.**
- **Day 3 — Joint review with both function leads. Root cause classified. Action items committed.**
- **Day 5 — Legal review (if triggered). Published to the two functions' retrospective archives and (if required) to the broader org.**

The schedule intentionally overlaps with the two-day draft / five-day submission cadence from `mod-108` chapter 03 §4 so joint retros do not have a slower baseline than single-function retros.

### 4.3 Publication and distribution

- One document, one URL, one history. Both functions' archives link to the same canonical.
- The distribution list includes both function leads, both on-call rotations, the shared stakeholders (safety / policy peer per chapter 06 of this module, `ai-risk-engineer` when compliance-triggered, legal when safety-triggered).
- Action items are mirrored into each function's issue tracker but retain the single canonical ID from the post-mortem.

### 4.4 Follow-through

The function tracks action-item completion in the quarterly review (§5). The joint-retro commitments are read against the shared surfaces dashboard (chapter 01 §7.6) and the contract register (chapter 01 §2). An action item that has slipped by two review cycles is escalated to both function leads jointly — not escalated by one to the other.

## 5. The quarterly joint retrospective

Joint post-mortems compound. The quarterly joint retrospective reads the past quarter's joint documents and asks:

- **Which shape dominates?** A quarter of Shape A documents says the budget contracts are not being enforced in deploy; a quarter of Shape B says the fallback contract needs a defensive-parity rewrite; a quarter of Shape C says the ingest pipeline is the next engineering investment.
- **Which seam has been named more than once?** A recurring seam is a structural problem, not an incident. The quarterly retro commits an engineering investment proportional to the recurrence.
- **Which action items from prior joint retros have not landed?** A list, with ownership and blockers.
- **Which new root-cause categories emerged?** Candidates for the §2.4 taxonomy expansion.

The quarterly joint retro has a single-page readout to the function leaders; the readout is the input to the mod-114 cross-functional leadership cadence.

## 6. The anti-patterns

Observed on real functions and recognised early:

- **"We'll write a function-specific retro each and reference each other."** Produces two incomplete documents that never converge. The joint document exists so the shared facts are not told twice.
- **"Shape C is really a security problem; they should own it."** A shared root cause is shared. One-sided ownership of Shape C breaks the fixture back-feed loop on the AI SRE side, because the fixture that catches the next instance lives in the serving stack, not the defensive stack.
- **"Let's defer the action items until the next quarter's roadmap."** Joint-retro action items have due dates in days / weeks (§2.5). Multi-quarter deferral turns the retro into theatre.
- **Legal as a blocker on publication.** Legal reviews for publication when a trigger from chapter 03 of this module fires. For incidents without a legal trigger, publication to the two-function archives does not wait on legal — the two functions' internal distribution is not external communication.
- **Naming individuals in the narrative.** Both the `mod-108` chapter 03 §5 blameless discipline and the `ai-infra-security` team's own professional norms apply. Named individuals are co-authors and action-item owners; narrative actors are roles ("the on-call AI SRE", "the classifier deploy pipeline").

## 7. The AI SRE's joint-retro worksheet

Use this during the incident to prepare the retro and after the incident to run it:

1. **Classify the shape (A / B / C) in the first fifteen minutes.** If uncertain, mark as "other" and let the Day 1 review re-classify.
2. **Confirm the co-authors.** Named individuals from both functions on the IC bridge.
3. **Pre-commit the joint document URL.** One shared location, linked from both functions' incident channels.
4. **List the artifacts that must be preserved** before the on-call rotation ends (dashboard pins, decision-log exports, deploy audit trail, user-report threads). Chapter 05 of this module governs the handling of any content-bearing artifacts.
5. **Nominate legal review requirement** per chapter 03 of this module's triggers. If legal is reviewing, the §4.2 cadence shifts to include their review before publication.
6. **Draft the narrative first, the detail blocks second, the action items third.** Not in reverse; action items authored before the narrative tend to be optimised for closure, not for correctness.
7. **Sign on the action items.** Both function leads sign; both functions' trackers mirror them.
8. **Publish to the quarterly review's intake queue.** The quarterly review (§5) is the compounding mechanism; retros filed only to the individual-incident archive are not learning, they are paperwork.

## Summary

- **Three incident shapes require joint post-mortems** — defensive change that caused reliability regression (Shape A), reliability remediation that weakened defensive posture (Shape B), and shared root cause (Shape C). The shape is named in the first fifteen minutes.
- **The joint template extends `mod-108` chapter 03** rather than replacing it. Co-author header, joint narrative with separated observation and inference, three function-specific detail blocks, extended root-cause taxonomy, ownership-tagged action items with due dates in days / weeks, and twin lesson blocks.
- **Each shape has a template branch** — must-name facts, must-reference artifacts, expected action-item classes. The branches are what a function fills in; the base template carries the structure.
- **The workflow is co-authored, five-day-cadenced, and single-document.** One URL, one history, both functions' archives link in.
- **The quarterly joint retrospective is the compounding mechanism.** Recurring shapes become engineering investments; outstanding action items are jointly escalated.
- **The anti-patterns at the seam — dual retros, one-sided ownership of Shape C, multi-quarter deferral, individual blame — are recognised and refused** so the template does what it was authored to do.

Chapter 03 takes the telemetry the budget contracts and joint retros produce and turns it into the compliance-side deliverable the `ai-risk-engineer` peer and legal need.
