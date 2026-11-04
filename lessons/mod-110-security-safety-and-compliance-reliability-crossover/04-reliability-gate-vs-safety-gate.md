# Reconciling the Reliability Gate with the Safety Gate

## Why this chapter exists

A model-backed product, in the frontier-lab operational idiom, passes through at least two gates on its way from training run to users: a **safety gate** (does this model meet the organisation's safety commitments?) and a **reliability gate** (does this service meet its SLO contract and operational posture?). On paper the two gates are adjacent and composable. In practice, they are run by different functions, measure different things, read different signals, and occasionally produce contradictory verdicts.

The AI SRE's role is to **operate the reliability gate as a peer to — not inside of — the safety gate**, and to reconcile the two explicitly where they overlap or diverge. The failure modes this chapter prevents:

- The reliability gate rubber-stamps a release because serving SLOs are green, while the safety gate is still in a tier that requires release-time constraints the SRE function has no visibility into.
- The safety gate blocks a release and the AI SRE function treats the block as a safety team problem, missing that reliability obligations — availability SLOs, provider SLAs, customer commitments — continue to accrue during the block.
- The safety team and the SRE function disagree on whether a release can proceed, and no mechanism exists to mediate — so the question climbs to the executive layer, where it is resolved by the louder voice.

Three organisations have published versions of a safety-tiering / release-gating framework that the AI SRE should be able to read and compose with: Anthropic's Responsible Scaling Policy (RSP), OpenAI's Preparedness Framework, and Google DeepMind's Frontier Safety Framework. Each is specific to the organisation; each represents an industry pattern the role will encounter. This chapter uses the three as case studies, then writes the reconciliation discipline that applies regardless of which framework the organisation operates under.

The exercise for this chapter (exercise 04) produces the reconciliation document a function signs with its safety / policy peer; chapter 05 governs the handling of any data the two gates exchange; chapter 06 draws the ownership lines explicitly.

## 1. The two gates, defined

### 1.1 The reliability gate (AI SRE)

The reliability gate is a release-blocking check at the service / product boundary. Its question: "does this change, applied to this fleet, preserve the service's SLO contract, the fallback contract, and the operational posture the function committed to?" Its signals come from:

- `mod-103`'s SLO stack (burn-rate alarms, error-budget state).
- `mod-104`'s observability (serving-health signals, judge-pipeline quality signals, cost signals).
- `mod-107`'s fallback contract (fallback-hit-rate, MTTD, MTTR under injected chaos).
- `mod-109`'s continuous chaos program (the chapter 04 contract suite's verdict).
- `mod-111`'s progressive-delivery metrics (canary vs stable comparison).

Its verdict is a ternary: pass, warn, block. Its owner is the AI SRE function lead.

### 1.2 The safety gate (safety / policy / responsible-AI function)

The safety gate is a release-blocking check at the model / capability boundary. Its question: "does this model, at this capability level, meet the organisation's safety commitments — including the commitments made under a responsible-scaling-style framework?" Its signals come from:

- Pre-release evaluations on the organisation's safety eval suite.
- Red-team findings against the model (methodology owned by `ai-infra-security` or a dedicated frontier-red-team function).
- Policy-review findings (is this capability release consistent with policy commitments?).
- Capability-threshold assessments (does this model meet or cross a published threshold?).

Its verdict is also a ternary: pass, pass-with-conditions, block. Its owner is a safety / policy function lead, typically distinct from the AI SRE function lead.

## 2. The three public frameworks as case studies

Each of the three is summarised at the level needed to reason about reliability interaction. Depth of interpretation belongs to the safety / policy peer; the AI SRE needs to recognise the vocabulary and know where its surface meets.

### 2.1 Anthropic's Responsible Scaling Policy

Anthropic has published a Responsible Scaling Policy (RSP) that commits the organisation to a tiered framework in which AI models are evaluated against capability thresholds (termed "AI Safety Levels", or ASLs in the original document), with safeguards and deployment commitments at each level. The policy has been revised publicly over multiple versions; the stable idea across versions is that models above a given capability threshold require correspondingly stricter deployment conditions, and that a model is not deployed until the organisation can meet the committed conditions. <!-- needs-research: cite the current published RSP version (number, date, URL) at the time of reading; Anthropic maintains the document on anthropic.com and has updated it several times since the initial 2023 publication -->

The reliability-gate adjacency: **deployment commitments at higher capability levels may include operational measures the AI SRE operates** — monitoring at specific sensitivities, incident-response SLAs, rollback timelines, specific redundancy or isolation requirements for the inference stack. The RSP commits the organisation; the AI SRE operates the operational half of the commitment.

A reliability-gate verdict that preserves SLOs but does not evidence the RSP-committed operational measures is a verdict that fails the joint gate even if neither function sees it alone.

### 2.2 OpenAI's Preparedness Framework

OpenAI's Preparedness Framework is the organisation's public approach to tracking and governing risks from frontier AI capability. The framework tiers capability risk (low / medium / high / critical-style tiering, with specific category labels for cybersecurity, biosecurity, AI self-improvement, and others), mandates evaluations before release, and conditions deployment on the post-mitigation residual risk tier. <!-- needs-research: cite the current published Preparedness Framework version and URL at the time of reading; the framework has been revised publicly since its 2023 publication -->

The reliability-gate adjacency: the framework explicitly treats **operational mitigations** as part of a model's deployment posture. Some mitigations are classifiers (owned by security / safety); some are monitoring (owned by the AI SRE); some are operational response capabilities (the ability to rapidly disable, roll back, or constrain a capability in production). The AI SRE owns the operational-response capability's reliability — the kill switch's availability, the fallback path's testedness, the dashboard's freshness.

A deployment the Preparedness Framework classifies as acceptable only with specific operational mitigations is a deployment the reliability gate must evidence those mitigations for. The gates do not compose by addition; they compose by shared evidence.

### 2.3 Google DeepMind's Frontier Safety Framework

Google DeepMind has published a Frontier Safety Framework that likewise tiers capability (named Critical Capability Levels, or CCLs, in the original document), with corresponding mitigations at each. The framework is specific about the categories of capability it tracks and about the required mitigations per level. <!-- needs-research: cite the current published Frontier Safety Framework version, URL, and effective date at the time of reading; DeepMind has revised the document publicly since its 2024 publication -->

The reliability-gate adjacency mirrors the RSP and Preparedness Framework. Mitigations at higher CCLs include operational commitments — monitoring, response, isolation — that the AI SRE function implements. The safety-gate verdict is conditional on those mitigations being in operational effect, not merely documented.

### 2.4 The common shape

Across the three, the common structure the AI SRE should recognise:

- **Capability tiers / levels.** A model is placed on a scale.
- **Mitigations per tier.** The higher the tier, the stricter the mitigations.
- **Operational mitigations as a category.** Monitoring, incident response, rollback capability, redundancy, isolation.
- **Release conditional on mitigations in effect.** Documentation is not enough; the operational mitigations must be demonstrably running.
- **Public commitment.** The frameworks are published; the organisation is held to them externally.

The AI SRE's integration point is the "operational mitigations" category. The safety gate asks whether the mitigation exists; the reliability gate asks whether it is operational.

## 3. Where the gates overlap

The gates overlap wherever the safety commitment reduces to an operational property the AI SRE's observability stack measures. A representative mapping — specific signals vary by framework and by product, but the *shape* is stable:

| Safety commitment (illustrative) | Operational property | AI SRE signal |
|---|---|---|
| "The model can be rapidly disabled if a capability threshold is crossed in production" | Kill-switch availability | Kill-switch heartbeat / dry-run success rate (owned by AI SRE) |
| "Monitoring must detect a safety-relevant deviation within N minutes" | MTTD on the safety classifier's alarm | `mod-108` chapter 05 fixture ledger + `mod-104` alarm freshness |
| "Rollback of a model release is possible within M minutes of a decision to roll back" | Rollback-drill verdict | `mod-111`'s rollback SLO + `mod-109` chaos run of the rollback path |
| "Fallback to a lower-capability model is available for high-risk routes" | Fallback availability | `mod-107` fallback contract verdict |
| "The deployment is isolated such that a capability leak cannot compromise peer services" | Blast-radius containment | Chaos game-day verdict + platform-isolation SLO |
| "Incident response SLA for safety-relevant incidents is K hours" | IC readiness, runbook coverage | `mod-108` chapter 01 inventory + `mod-108` chapter 02 IC posture |

Each row is a joint artifact. Chapter 02 of this module's joint post-mortem shape applies when a row's property regresses in production; chapter 03's compliance telemetry applies when the regression is also a reportable event.

## 4. Where the gates intentionally diverge

The gates also diverge, by design. A reliability-gate verdict can be green while the safety gate is amber or red; the reliability gate must not read the safety gate's amber as a reason to turn amber itself.

Three patterns of divergence the function should recognise and refuse to collapse:

### 4.1 Pre-deployment capability evaluation

Before a model release reaches the reliability gate, it has been evaluated by the safety gate's pre-release eval suite. The AI SRE function has **no role** in that evaluation. The reliability gate cannot pass a release the safety gate has not cleared; the reliability gate also cannot clear a release the safety gate is still evaluating — but the reliability gate does not re-run the eval. Divergence is correct: the two gates ask different questions.

### 4.2 Capability-threshold response

When the safety function raises the capability tier of a deployed model (new evidence has changed the assessment), new mitigations may be required. Some mitigations are AI SRE work (new monitoring sensitivity, new rollback SLA); some are safety / security work (new classifier, new red-team cadence). The reliability gate cannot make itself the owner of the capability-threshold decision; it reads the decision, implements the AI SRE-shaped mitigations, and verifies them.

### 4.3 Reliability regression without safety implication

Conversely, a reliability regression that does not affect safety posture is an SRE problem, not a safety problem. An elevated TTFT, a slow-burn SLO, a cost regression — these are AI SRE surfaces. The safety gate does not bend to accommodate them; the reliability gate does not pretend to need safety input on them. Each function owns its own decision in its own lane.

A function that collapses the two — treating every safety discussion as a joint discussion, or every reliability discussion as a joint discussion — produces a bottleneck at the joint cadence and dilutes the accountability of both functions.

## 5. The reconciliation discipline

The AI SRE and the safety / policy peer commit to a small set of practices that reconcile the gates without conflating them.

### 5.1 The joint release checklist

Before a release that meets the safety gate's criteria for "joint review" (specifics vary by framework; the organisation's RSP / Preparedness / Frontier Safety Framework-equivalent specifies the threshold), the two functions produce a joint release checklist. The checklist has three columns:

- **Operational mitigation** (from the framework's requirements for this capability tier).
- **Signal / artifact that evidences the mitigation is in effect** (an SRE-owned dashboard, alarm, drill result, chaos-run verdict).
- **Current status** (in effect / partially in effect / not in effect, with a link to the evidencing artifact).

The checklist is signed jointly. A release that is "not in effect" on a mandatory mitigation does not proceed, regardless of serving SLO status.

### 5.2 The capability-tier change protocol

When the safety function changes a model's capability tier, the AI SRE function receives a **change notice** with the new mitigation requirements. The notice specifies:

- Which mitigations are new or changed.
- Which are AI SRE-owned vs safety-owned vs joint.
- The target effective date.
- The expected SRE deliverable (new monitor, new runbook, new drill schedule, new fallback contract clause).

The AI SRE function returns, within a committed window, an acknowledgement with the operational implementation plan and the date the mitigations are expected in effect. The returned plan is tracked in the joint quarterly cadence.

### 5.3 The pre-committed decision tree

Mirroring the `mod-108` chapter 02 pre-committed decision tree pattern: the two functions pre-commit how specific incidents interact with the gate:

- **A safety-classifier false negative is detected.** The safety function's call on tier re-assessment; the AI SRE adds the detection to the dashboard; the release in question is paused if the detection escalates.
- **A reliability regression correlates with a safety-telemetry shift.** Joint retro (chapter 02 of this module, Shape C); both functions investigate before concluding; neither disowns.
- **The kill switch's drill latency drifts past its SLO.** AI SRE emergency; safety function is informed because the mitigation is no longer in effect; a release in the current window is paused until the kill switch meets its SLO again.
- **A new capability is discovered in an already-deployed model.** Safety function re-evaluates tier; AI SRE prepares the operational mitigations the new tier may require; the release-gate posture may change.

The decision tree is signed, reviewed quarterly, and lives in the joint runbook register.

### 5.4 The shared evidence surface

Both functions read the same evidence for the mitigations they jointly care about. Three practices support this:

- **A joint dashboard** that renders each operational mitigation as a signal with pass / warn / fail state. Mandatory for mitigations in the checklist above.
- **Dashboard-as-code and audit** per chapter 03 of this module; the joint dashboard is part of the compliance-telemetry contract.
- **Scheduled joint review** — a weekly or bi-weekly standing meeting where the two functions walk the dashboard. New releases, open mitigations, drift signals, and upcoming capability evaluations are the agenda.

### 5.5 Escalation path

When the two functions cannot agree:

- **Defer-to-stricter-gate by default.** If one function says block and the other says pass, the release does not proceed on the current cycle.
- **Named escalation.** If the block persists past a pre-committed window, the two function leads escalate jointly to a named cross-functional lead (CTO, head of research, equivalent). Escalation is co-authored, not one-sided.
- **No executive override without a committed mitigation.** The escalation can commit the organisation to mitigating action; it does not override the stricter gate without that action.

The escalation path is public inside the organisation; its existence forestalls the "louder voice wins" failure mode.

## 6. Gate-reconciliation under the frameworks

Three worked examples illustrate how the practices in §5 compose with the §2 frameworks.

### 6.1 Example — RSP-style release of a new model

- The safety function evaluates the model against the organisation's RSP equivalent. The verdict: eligible for deployment at the current ASL-equivalent if the operational mitigations for that ASL are in effect.
- The AI SRE function receives the capability-tier change notice (§5.2). For this ASL, the mitigations include "safety-relevant incident MTTD under 15 minutes on the user-facing route" and "kill switch verified operational via drill monthly."
- The AI SRE function already has `rb-toxic-output-blast` wired for MTTD; the drill is scheduled and the last drill was successful. The §5.1 checklist is filled, signed, committed.
- Deployment proceeds. Both functions monitor the joint dashboard from §5.4.
- A month later a new red-team finding re-classifies the model's capability. The AI SRE function receives a new change notice (§5.2) with tightened requirements; a chaos game-day is scheduled to verify a tighter MTTD; the mitigation list is updated.

### 6.2 Example — Preparedness-Framework-style post-mitigation release

- The pre-mitigation assessment places the model's capability in a "high" category for cybersecurity (illustrative). The framework requires specific post-mitigation signals before release.
- The safety function authors the mitigation plan with security; the AI SRE reviews the operational feasibility of the response-time commitments.
- The release proceeds under the post-mitigation plan. The AI SRE's joint dashboard surfaces the signals that evidence each mitigation's operational effect.
- A chaos run (chapter 03 of `mod-109`) later reveals the rollback path has a 22-minute P95, exceeding the committed 15-minute rollback SLA. The §5.3 decision tree fires: the release posture is paused until the rollback path is fixed and re-drilled.

### 6.3 Example — Frontier-Safety-Framework-style tier change

- A new evaluation suggests the deployed model crosses a CCL-equivalent threshold previously below the deployed level.
- The safety function raises the tier. The AI SRE receives the change notice. Required new mitigations: tighter isolation of the inference stack, additional monitoring on specific capability indicators.
- The AI SRE returns a plan: platform-isolation SLO (joint with `ai-infra-platform`), new monitors (joint with `ai-eval-engineer` for the capability-indicator signals).
- Until the mitigations are in effect, the model's deployment to additional capacity is paused; existing capacity is retained with existing posture until the capacity-expansion plan clears. The two gates, reconciled.

## 7. The common failure modes

The pattern of practices in §5 is simple on paper; the failure modes are specific:

- **Treating the safety-gate verdict as input rather than peer.** The AI SRE "reads" the safety verdict and translates it into an SRE decision. The result: the two gates run sequentially, safety blocks are late-breaking, operational mitigations are retrofitted under pressure. Correct: the gates run concurrently, with shared artifacts.
- **Treating operational mitigations as documentation.** The organisation records "we have monitoring for X." No one checks whether the monitor fires when X happens. The chaos program from `mod-109` chapter 03 is the discipline that prevents this; the joint dashboard is its render.
- **Collapsing joint reviews into the AI SRE's cadence.** The safety function has its own cadence (pre-release evals, red-team findings, policy committee). The joint cadence is additive, not a replacement.
- **Forgetting the public commitment.** The RSP / Preparedness Framework / Frontier Safety Framework are public. An operational mitigation the organisation has publicly committed to and does not have in effect is an external reputational surface in addition to a compliance surface. The AI SRE's attestation discipline from chapter 03 of this module applies.
- **Blanket deferring to safety on all AI-shaped incidents.** Not every AI incident is a safety incident. A TTFT regression is an SRE incident; a kernel-level GPU driver bug is an SRE / platform incident; neither is a safety incident. Defer where the taxonomy from `mod-108` chapter 03 §3 and chapter 06 says defer; do not defer by habit.

## 8. The AI SRE's gate-reconciliation worksheet

Use this at the time the joint checklist is first authored and at each major release or capability-tier change:

1. **Name the framework** the organisation operates under. If none is public, name the internal equivalent the safety function uses.
2. **Enumerate the current capability tier** of each production model, as the safety function has classified it.
3. **List the operational mitigations the framework requires** at each tier. Identify which are AI SRE-owned, which are joint, which are safety / security-owned.
4. **Author the joint release checklist** (§5.1) with the three-column shape. Sign.
5. **Author the change-notice protocol** (§5.2) and commit to the response window.
6. **Fill the pre-committed decision tree** (§5.3). Walk each entry with the safety peer; iterate until both sign.
7. **Deploy the joint dashboard** (§5.4). Confirm every mitigation on the checklist has a corresponding signal.
8. **Schedule the standing joint review.** Attendance is non-optional on both sides.
9. **Walk the escalation path** (§5.5) and validate both functions' leaders agree.
10. **Attest** jointly with the safety peer that §§1–9 are operational.

A function with this worksheet in effect can honestly answer, on request, whether the service's safety commitments and reliability commitments are reconciled. A function without it cannot.

## Summary

- **The reliability gate and the safety gate are peer gates, not nested ones.** Each asks a different question; each has an owner; each produces a verdict.
- **Three public frameworks** — Anthropic's RSP, OpenAI's Preparedness Framework, Google DeepMind's Frontier Safety Framework — provide the vocabulary of capability tiers, mitigations, and release conditioning that organisations apply to their own practice.
- **The gates overlap at operational mitigations** — monitoring, response, rollback, isolation, incident-response SLAs. These mitigations are the AI SRE's work the safety function depends on.
- **The gates intentionally diverge** on pre-deployment evaluation, capability-threshold response, and purely reliability-shaped incidents. Collapsing the divergence is a mistake.
- **Reconciliation is a documented practice**: joint release checklist, capability-tier change protocol, pre-committed decision tree, shared evidence surface, published escalation path.
- **The anti-patterns are preventable** with the worksheet and with the public commitment reminder that an operational mitigation is a documented claim the organisation stands behind externally.

Chapter 05 governs the handling of any data the two gates exchange — classifier decision logs, red-team findings, prompt / completion content that lands in a safety-adjacent incident — so the exchange itself is not a new surface.
