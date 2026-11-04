# Incident Command for AI Systems

## Why this chapter exists

The classical incident-command vocabulary — Incident Commander (IC), Operations Lead (OL), Subject-Matter Expert (SME), Comms Lead, Scribe — was authored for telecommunications emergencies and adopted into tech by Google's SRE book and FEMA's Incident Command System (ICS). It works. The AI SRE inherits it almost unchanged. What does change, and what this chapter is about, is the **composition of the SME bench** and the **cadence of stakeholder communication** when the incident is quality-shaped, safety-shaped, or vendor-shaped rather than crash-shaped.

In a classical SRE incident, the on-call engineer is often enough to drive the IC role solo. The service is down, the error budget is burning, the fix is in a known subsystem, and the handoff is to the engineering team that owns the subsystem. In an AI-flavoured incident, the SME bench is wide — a hallucination-rate spike pulls in the eval engineer, the AI engineer who owns the prompt, the retrieval engineer, the inference engineer who owns the router, possibly the vendor-risk engineer, and almost certainly the safety reviewer. The IC's job is to **keep the many SMEs working on parallel threads without colliding**, to **maintain a single source of truth about the state of the incident**, and to **brief the stakeholders at a cadence that matches the incident's blast radius**, which in AI often means legal, PR, and executive stakeholders far earlier than a classical incident of equivalent user-impact.

This chapter codifies the command shape, the role interactions, bridge-call and incident-channel discipline, and the stakeholder-communication cadence the AI SRE is expected to operate. It is not a reinvention of ICS — it is the AI-specific adaptation of it.

## 1. The roles

The role list below is the vocabulary used throughout this module. Teams that already have equivalents under different names should map them explicitly in their IR documentation. What matters is that the *functions* exist and are named at the start of the incident, not that the labels match verbatim.

### 1.1 Incident Commander (IC)

The one person who is decisively in charge of the incident. The IC does not debug, does not type commands, does not write prose beyond the incident timeline. The IC:

- **Maintains the timeline**, the single source of truth for what happened and what was tried.
- **Assigns tasks** to named owners with explicit success criteria and deadlines.
- **Decides severity** and escalates / de-escalates based on the facts the OL / SMEs surface.
- **Approves mitigations** that have blast-radius above a trivial threshold (runbooks label this per chapter 01 §1.4).
- **Calls the executive escalation**, loops in legal and PR when triggers fire (section 5 and chapter 06).
- **Declares resolved**, declares post-mortem required, hands off the retro owner.

The IC is rotated. The AI SRE function owns a weekly IC rotation independent of the primary on-call rotation; the person paged first is usually the on-call engineer, and if they decide the incident needs an IC separate from them, they page the IC rotation.

### 1.2 Operations Lead (OL)

The person driving hands-on mitigation on the primary thread. The OL executes the runbook's confirm checklist, runs the mitigations, and reports state to the IC. In a small incident the OL and the IC are the same person; in any multi-SME incident the roles split, because the IC cannot both drive the room and drive a terminal.

For AI incidents the OL is usually the on-call engineer on the owning peer rotation — `ai-infra-inference-engineer` for a serving regression, `ai-eval-engineer` for a judge-drift incident, `AI Engineering` for a prompt-regression incident. The AI SRE's on-call engineer is the OL in the first thirty minutes and then may hand off to a more specific owner once the class is confirmed.

### 1.3 Subject-Matter Experts (SMEs)

The named engineers who own a specific root-cause surface. They do not drive the incident; they consult. In an AI incident the SME bench commonly includes:

- **Serving / inference** — router, scheduler, KV-cache, structured-output, streaming gateway.
- **Model platform** — registry, model-weights artifacts, deployment / rollout.
- **Performance / kernel** — GPU driver, NCCL, numeric-format, fabric, topology.
- **AI Engineering** — prompt, agent scaffolding, tool schema, feature flags.
- **Eval engineer** — judge model, canary eval set, calibration set.
- **Retrieval / pipeline** — index, embedding model, chunking, ingest.
- **Vendor risk** — provider status, fallback contract, SLA interpretation.
- **Safety / policy** — content taxonomy, filter configuration, safety-review decision.

The IC is responsible for knowing *that each of these functions exists* and paging them by function, not by name. "Who is the eval SME right now?" is a question the IC answers from the on-call rotation, not from memory.

### 1.4 Communications Lead (Comms Lead)

The person who writes the stakeholder updates. Internal updates (Slack channels, exec DMs) and external updates (status page, customer communications) are both this person's output. The Comms Lead does not drive mitigation and does not typically own the technical timeline — the IC is the source, the Comms Lead is the clean prose.

For AI incidents whose externally visible surface is a safety or quality claim ("the model said X about Y"), the Comms Lead works jointly with legal and PR earlier than in a classical incident. This chapter's section 5 and chapter 06 cover the cadence.

### 1.5 Scribe

The person who keeps the timeline in human-readable form, usually in the incident channel's pinned message or a shared document. The scribe's product is the artifact the post-mortem (chapter 03) opens with. In small incidents the IC scribes. In sustained incidents, a dedicated scribe is paged and stays until the incident resolves.

### 1.6 Executive / Legal / PR liaison

In AI-safety-adjacent incidents (`rb-toxic-output-blast`, model-weight corruption serving at scale, a safety-filter regression, a vendor outage with contractual implications) the IC pages an executive liaison and a legal liaison explicitly. These liaisons do not run the incident. They:

- Decide what can be said externally and when.
- Decide whether the incident crosses a formal-notification threshold (chapter 06).
- Decide whether to involve government / regulator relations.

The AI SRE's job is to brief them accurately and let them make the call. Do not draft a public statement; do not call a regulator. Hand off to the people who have the authority.

## 2. The incident channel

A single Slack / Teams / Discord channel is the incident's operational surface. The channel is the source of truth in flight; the final document is the post-mortem.

### 2.1 Channel hygiene

- **One channel per incident.** `#inc-2026-11-04-ttft-summariser` or whatever naming convention the org uses. Never re-use a channel across incidents.
- **Named roles pinned in channel topic.** IC, OL, Comms, Scribe, peer SMEs on-call. Change the topic as roles change hands.
- **Timeline pinned.** The shared document / message with the running timeline is pinned so late joiners find it.
- **Threads per workstream.** The main channel is for IC decisions, timeline updates, and all-hands context. Debugging conversation happens in threads so the main channel stays scannable. The IC bans "let me DM you about this" — the record is the channel.
- **No social chatter.** The channel is operational. Memes and apologies are not banned; they go in a different channel.

### 2.2 The posting rhythm

The IC enforces a cadence of state-of-the-incident posts in the channel. Every ~15 minutes in the active mitigation phase, every ~30 minutes once the incident is contained, the IC posts a short "state" message:

- **Severity.** Current.
- **Symptom.** Current.
- **Scope.** Current.
- **Mitigations in flight.** With named owners and ETAs.
- **Next decision point.** What the IC is waiting on.

The cadence is a hard rule. Even if the answer is "no change, still investigating," the IC posts it. Silence in the channel is the single most common failure mode of long incidents, because stakeholders page the IC out of band to ask the questions the IC should have answered proactively.

### 2.3 Bridge-call hygiene

Live-voice bridges (Zoom, Google Meet, Teams) accelerate SME-SME coordination and slow IC note-taking. The AI SRE function's doctrine is explicit:

- **Default to channel-first.** If the problem is debuggable in text, stay in text — the record is better.
- **Bridge for synchronisation, not for debugging.** When a mitigation requires three SMEs to coordinate a sequence of actions in under a minute, the bridge exists for that call. When the mitigation is one SME pasting commands, the bridge is overhead.
- **Mute discipline.** On a bridge the IC calls "speaking" and names who has the floor. More-than-one-voice is an incident in itself.
- **Scribe writes the bridge back to the channel.** Every decision made on the bridge is re-posted to the channel within two minutes. The channel is the canonical record; the bridge is a tool.
- **Record the bridge** with the participants' consent and the privacy / retention policy. Post-mortems benefit; privacy and legal requirements govern.

AI incidents often draw many voices onto the bridge — safety, legal, PR, executive — because the questions they have are not the questions the engineers can answer in a thread. The bridge-call hygiene above is more important, not less, when the audience is non-technical.

## 3. The lifecycle

The ICS-style lifecycle below is the canonical shape. Each phase names the role whose decision gates the next transition.

```
Detect ─▶ Declare ─▶ Investigate ─▶ Mitigate ─▶ Stabilise ─▶ Resolve ─▶ Retrospect
                                                              │
                                                              └─▶ Fix (owned by peer, tracked by AI SRE)
```

### 3.1 Detect → Declare

**Detection** is the SLO burn-rate alert, the runbook's primary signal, a customer-support flag, a vendor status event, or a user-visible outage report. The on-call opens the runbook from chapter 01, runs the confirm checklist, and decides whether to **declare** an incident.

Declare the incident when any of:

- An SLO burn alert has fired at the 1-h or faster window.
- A runbook's severity ladder specifies SEV-2 or SEV-1 by default.
- User impact is confirmed on a committed-SLO route.
- A safety-adjacent signal has fired at any scale.
- A vendor outage has been declared on a route in the dependency map.

The declaration is a channel post: `/inc declare sev2 rb-ttft-regression summariser` or whatever tooling the org uses. The channel opens, the roles page, the clock starts.

### 3.2 Investigate

The IC assigns the OL to run the confirm checklist from the runbook. SMEs are paged if the checklist points to their surface. The channel posting rhythm begins.

An AI-specific trap in the investigate phase: the apparent symptom and the real cause often sit in different runbooks. "TTFT regressed" (`rb-ttft-regression`) frequently has "prompt template changed and invalidated the cache" (`rb-cache-hit-cliff`) as the real cause. The confirm checklist exists to route to the correct runbook within the first ten minutes. The IC should expect to switch runbooks once, and the OL should be comfortable with it.

### 3.3 Mitigate

Mitigations are picked from the runbook's menu (chapter 01 §1.4). The IC approves blast-radius-above-trivial mitigations. Each mitigation is a scribed line in the timeline: the time, the mitigation, the owner, the expected effect, the measured effect after the expected time-to-effect window elapses.

AI-specific trap: a mitigation that *looks* reversible (route to the fallback provider) may not be, because the downstream prompt caches, embeddings, and evals are built against the primary. "Open the breaker" is reversible; "re-train the embedding index because we fell through to a different embedding provider" is not. The mitigation menu in each runbook flags reversibility explicitly for this reason.

### 3.4 Stabilise

Stable is not resolved. The symptom has stopped escalating; the mitigation is holding; the SMEs have a path forward. In this phase the IC:

- Drops the posting cadence from 15 min to 30 min.
- Declares a stable update for stakeholders.
- Starts the post-mortem scaffolding (chapter 03) with the owner assigned.

In many AI incidents stable is the achievable state for hours or days while the root-cause fix is non-trivial (re-embedding an index, waiting for a vendor to deploy a fix, cutting a new model version). The IC role may be handed off during this phase to the next rotation — explicitly, with a formal handoff that mirrors the channel's posting shape.

### 3.5 Resolve

Resolved means the user-visible impact is gone and the mitigation is no longer required — either the root cause is fixed or the mitigation has become the steady state (e.g., permanent rate cap). Resolved is a post to the channel, an update to the status page, and the formal hand-off of the post-mortem owner and due date.

### 3.6 Retrospect

Chapter 03 of this module. The retro opens no later than two business days after resolve.

### 3.7 Fix

The peer-owned work item that lands the root-cause change. The AI SRE function does not own the fix; it does own the tracking (Jira ticket, Linear issue, whatever) so that the fix lands within the agreed window and the regression fixture (chapter 05) is produced.

## 4. Handoffs across rotations

AI incidents routinely outlive a rotation. A re-embed takes a day; a vendor fix takes a week; a prompt eval regression needs the next full workday to triage. The handoff shape:

- **The outgoing IC writes a handoff note** pinned to the channel: current state, open threads with owners, decisions still pending, next scheduled checkpoint, severity.
- **The incoming IC reads it in-channel**, asks clarifying questions in-channel, and then acknowledges the handoff in-channel. The record is on-channel so the next rotation and the post-mortem both see it.
- **Cross-timezone handoffs follow the sun.** The IC rotation is scoped so there is always a waking IC; the AI SRE function budgets for this when sizing the rotation.
- **Executive / legal / PR liaisons are not rotated through the IR channel.** They have their own rotation; they are briefed by the IC at handoff.

## 5. Stakeholder communication cadence

The externally-visible side of an AI incident has a different shape from a classical incident. Classical: status page, customer email, PR statement if pressed. AI-flavoured: all of the above *plus* legal, regulator, policy, and safety-team involvement at thresholds that trigger much earlier.

### 5.1 Internal stakeholders

- **Engineering leadership.** Briefed at declare, at stable, at resolve, and whenever severity changes.
- **Product / customer-success.** Briefed if customer-visible impact is confirmed.
- **Executive on-call.** Paged at SEV-1 by default and at SEV-2 if the incident has a safety-adjacent shape.
- **Legal / compliance.** Paged at any `rb-toxic-output-blast`, `rb-weight-corruption` (if user-visible), any provider outage exceeding the contracted SLA, or any incident with reportable-event implications (chapter 06).
- **Safety / policy.** Paged at any safety-adjacent signal, regardless of severity.
- **Security.** Paged at any `rb-eval-set-poisoning`, suspected weight tampering, or suspected provider-account compromise.

### 5.2 External stakeholders

- **Status page.** Updated at declare (if the incident is likely to be user-visible), at stable, and at resolve. The template language per runbook (chapter 01 §1.6) is the starting point; the Comms Lead adjusts.
- **Customer success / account teams.** Briefed as the status page is updated so the front-line can answer customer questions.
- **Named enterprise customers with per-contract commitments.** Briefed directly when the incident impacts their contracted SLOs, residency, or data-handling commitments.
- **Regulators / auditors.** The AI SRE does **not** contact regulators. Legal / `ai-risk-engineer` make that call with the executive liaison. The AI SRE's job is to provide the technical facts the decision needs (chapter 06).
- **Public / press.** PR owns. The engineer's job is to not pre-empt them in public posts.

### 5.3 The AI-specific communication traps

Four failure modes recur in AI incident communication. The IC owns avoiding them:

- **Naming the vendor prematurely.** Saying "OpenAI is down" in a status post before the vendor has acknowledged the event is both potentially wrong (local breaker opened on a different signal) and relationally expensive. The template language says "an upstream provider is degraded;" the vendor is named only after the vendor acknowledges.
- **Over-promising a quality fix.** "The model will stop hallucinating about X" commits the product to an uncertain outcome. The template language commits to the mitigation step ("we have reduced feature capability until the regression is diagnosed"), not to the quality outcome.
- **Under-communicating during a long tail.** When the primary mitigation holds and the root-cause fix takes days, stakeholders go quiet in the channel and loud out-of-band. The IC maintains the 30-min cadence with "still stable, no change" posts explicitly.
- **Discussing specific customer data or outputs in the open channel.** Sample traces, flagged outputs, user transcripts — all go into a controlled-access artifact store, not into the incident channel. The channel references the artifact by ID.

## 6. Safety-adjacent incidents and the pre-committed decision tree

`rb-toxic-output-blast`, `rb-weight-corruption` at serving scale, `rb-hallucination-rate-spike` on a regulated-domain route, and `rb-eval-set-poisoning` are the four classes most likely to pull in legal, PR, regulator, and executive stakeholders. The AI SRE function pre-commits a decision tree so that these are not debated in the first minute of an incident:

- **Trigger: a safety classifier output exceeds threshold on a user-facing route for ≥ 5 minutes.** Page safety and legal immediately. Do not wait for the engineering diagnosis. Chapter 06 covers the formal thresholds.
- **Trigger: user-reported safety flag with an egregious single sample.** Same — page safety and legal on first report, regardless of signal aggregation.
- **Trigger: weights load without checksum verification and quality collapses.** Page security and legal.
- **Trigger: eval-set poisoning suspected.** Page security and legal.
- **Trigger: a provider outage exceeds the SLA credit threshold.** Page vendor-risk and legal.

A pre-committed decision tree is an IC-protective device. It removes the "should I escalate?" judgement call at the moment of highest cognitive load. The exercise for this chapter asks the learner to fill in their organisation's version of this tree.

## 7. Working with legal, PR, and regulator-facing teams

Three operational principles the AI SRE must internalise about the legal / PR / regulator interface:

- **Facts, not framing.** Brief with timestamps, metric names, scope, and mitigations. Do not editorialise or speculate. Legal and PR need accurate raw material; they do not need your opinion of blame.
- **Preserve artifacts.** Flagged outputs, request IDs, user interactions, decision logs — all are discoverable in a formal-notification context. Preserve to the retention policy; do not delete even after resolution until legal advises.
- **Defer formal communication.** The AI SRE does not draft the regulator submission, does not post the public statement, does not reply to press. The role's output is the technical narrative that legal and PR consume.

Chapter 06 covers the formal-reporting thresholds (EU AI Act Article 72 post-market monitoring, FDA PCCP triggered change, SR 11-7 model-risk-management notification). This chapter's scope is operational discipline during the live incident.

## 8. The AI SRE's incident-command worksheet

1. **Name the roles explicitly** at incident declare. IC, OL, SMEs, Comms, Scribe. Pin in channel topic.
2. **Open the channel** with the naming convention and the pinned timeline.
3. **Enforce the posting cadence.** 15 minutes active, 30 minutes stable. "No change" is a valid update.
4. **Use the bridge for synchronisation**, not for debugging. Scribe back to the channel within two minutes.
5. **Pre-commit the safety decision tree.** Trigger-to-page mapping that does not require judgement at 3 AM.
6. **Brief legal / PR / exec at the trigger**, not at convenience. They prefer early and accurate to late and polished.
7. **Hand off with a pinned handoff note.** Across rotations, across timezones.
8. **Declare stable before resolved.** Stabilise is the long tail; the IC role may change hands.
9. **Hand the retrospective owner an incident channel that is a usable artifact**, not a reconstruction job.

## Summary

- **The IC lifecycle is classical, the SME bench is AI-specific.** ICS roles are unchanged. The bench of SMEs the IC must coordinate is wider and more specialised than in classical SRE.
- **Channel discipline beats cleverness.** One channel, pinned roles, pinned timeline, threads for workstreams, a posting cadence the IC enforces explicitly, and a scribe for the bridge.
- **Stakeholder cadence is earlier than classical SRE.** Safety-adjacent, regulator-adjacent, and vendor-contract incidents page legal / exec / PR on trigger, not on severity threshold.
- **Pre-commit the decision tree.** Trigger-to-page mapping is authored out-of-incident and consulted in-incident.
- **The AI SRE briefs; others formally communicate.** The role's output is the technical narrative. Legal, PR, and the exec liaison own what is said and to whom.

Chapter 03 picks up at the retrospective: the blameless post-mortem adapted for AI-specific root-cause categories and the back-feed discipline that turns an incident into a durable fixture.
