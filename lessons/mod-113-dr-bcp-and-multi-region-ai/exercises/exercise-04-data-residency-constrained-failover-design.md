# Exercise 04 — Data Residency-Constrained Failover Design

**Estimated effort:** 2 hours · **Output:** the per-stamp residency posture document for an EU + US + APAC deployment of the student's chosen service, a request-level residency-tag implementation, a routing-layer enforcement rule, a pre-authorised cross-residency burst design (or a documented decision not to authorise one), and the chapter 04 §7 residency checklist completed and signed by a legal peer (or a documented plan to obtain the sign-off).

## Objective

Design the residency posture of the service against a three-region topology (EU, US, APAC) and prove that the design enforces the posture in code, not only in documentation. The exercise forces decisions the function cannot defer: which stamps exist, which regions are in each stamp, which bursts are pre-authorised with what legal basis, and what the routing-layer refusal rule is when a request's residency is unclear.

## Prerequisites

- Chapter 04 of this module.
- Exercises 01–03 completed.
- Access to the service's routing layer (service mesh, API gateway, or application-level routing) sufficient to implement a routing rule.
- A contact in legal / compliance (DPO, risk peer, or general counsel) available for a 30-minute consultation during the exercise. A student without this access produces the output in draft and flags the legal sign-off as a prerequisite before any real deployment.

## Problem statement

Work in five parts.

### Part 1 — Stamp topology

Design the stamp topology for the service:

- **Stamps.** EU stamp, US stamp, APAC stamp. For each, list the regions, the primary, and the within-residency warm standby (chapter 04 §4.1).
- **Residency boundary.** Which regulatory regime applies to each stamp (GDPR for EU, applicable US law for US, jurisdiction-specific for APAC — often multiple sub-stamps for APAC given the heterogeneous regulation).
- **Shared control plane.** What is shared across stamps (artifacts, prompt registry, config, release pipeline, observability roll-up) and what is not (user data, conversation state, trace content).
- **Residency commitments.** The service's contractual commitments per stamp — specific customer contracts, regulatory regimes, internal policies.

Document the topology in a diagram (ASCII or a tool of choice) committed to the ISCP.

### Part 2 — Request-level residency tagging

Implement the chapter 04 §6.1 residency tag:

- **Signals.** The three signals (user declaration, IP geolocation, auth-context override) in order.
- **Default.** The behaviour on ambiguity (fail-safe to a declared default, or drop with an error).
- **Propagation.** How the tag flows through the request (header, context object, trace attribute).
- **Audit.** How the tag is emitted to the audit record (`mod-110` chapter 02).

Commit the implementation diff. Add a unit test that confirms the tag is present on every served response.

### Part 3 — Routing-layer enforcement

Implement the chapter 04 §6.2 routing rule:

- **Normal routing.** Tag `X` dispatches to stamp `X`.
- **Mis-residency refusal.** A request tagged `EU` landing at the US stamp's ingress is refused; the error includes a well-defined code that the client can act on.
- **Burst routing.** A request that invokes the §3.2 pre-authorised burst (Part 4) dispatches to the out-of-residency stamp with a `burst_event_id` attribute.

Add an integration test: synthesise an EU-tagged request targeted at the US stamp's ingress; confirm the refusal. Synthesise a US-tagged request; confirm successful dispatch. Record the test results.

### Part 4 — Pre-authorised cross-residency burst design

For each stamp, decide whether a pre-authorised burst is in scope. For each stamp where it is:

- **Legal basis.** State the instrument — SCCs, BCRs, adequacy decision, derogation — that authorises the burst. The specific instrument is drafted by legal; the function cites it. Attach the instrument's reference to the ISCP.
- **Data categories eligible.** Enumerate the fields of the request / state that may burst. Fields not listed are not permitted to burst.
- **Maximum duration and 30-day allowance.** State the per-event maximum and the rolling cap.
- **Notification obligation.** User notice, DPO notice, regulator notice. Pre-written templates (reference exercise 06).
- **Teardown runbook.** Name the runbook that cleanly terminates the burst on primary recovery. If the runbook does not yet exist, write it.
- **Invocation gating.** Who authorises the burst during an incident. Default: the DPO, via a documented response-time SLA. The service IC does not unilaterally invoke.

For each stamp where a burst is explicitly *not* in scope:

- Document the alternative posture (hold, degraded-mode in-residency) with the drill scenario pointer (exercise 05).

### Part 5 — Residency checklist and legal sign-off

Complete the chapter 04 §7 checklist (13 items). For each item:

- **State.** In place / drafted / not started.
- **Evidence.** A Git path or URL that a reader can follow to verify.
- **Open questions.** For items not yet complete, the blocker and the target close date.

Then: schedule the 30-minute consultation with the legal peer. The consultation's output:

- Written approval of the stamp topology.
- Written approval (or documented edits) of the pre-authorised burst instrument(s).
- Written approval (or documented edits) of the user-notice template trigger condition.
- A list of residual compliance concerns not addressed by this exercise.

Attach the consultation notes to the ISCP.

## Requirements

1. **Three stamps are documented**, each with its region set, warm standby, and residency boundary.
2. **The residency tag is implemented** and emitted on every response. A unit test asserts.
3. **The routing-layer refusal rule is live** and tested with a synthesized mis-residency request.
4. **Each stamp has a declared burst posture** — either a pre-authorised burst with the legal instrument cited, or a documented "no burst" posture with the alternative behaviour.
5. **The 13-item checklist is completed** with evidence, not promises.
6. **The legal peer has reviewed the design** and the consultation notes are attached. If legal review cannot be scheduled within the exercise's window, the notes document the outreach and the planned follow-up date.
7. **The IaC admission policy (chapter 04 §6.4) is tested** — a PR attempting to deploy an in-scope data substrate to an out-of-residency region is blocked by the admission controller or policy-as-code layer.

## Starter guidance

- The APAC stamp is almost always a cluster of sub-stamps (Japan, Singapore, India, Australia) with different residency commitments. Do not model APAC as one stamp unless the service's customer base makes it one.
- The pre-authorised burst is the piece most teams skip. The exercise forces an explicit decision — authorise with legal, or explicitly do not authorise and drill the alternative.
- Part 2's unit test is easy if the tag is a context object; harder if the implementation relies on inheritance from an unspecified default. If the latter, fix the implementation.
- Part 3's enforcement must fail closed. A routing-layer bug that admits a mis-residency request is a breach; the test asserts refusal under adverse conditions, not only the happy path.
- For the legal consultation, send the stamp topology, the residency tag design, and the burst design in advance; do not use the 30 minutes to reconstruct the context.

## Acceptance criteria

- A per-stamp topology diagram and document in the ISCP.
- A residency tag implementation with a unit test and audit emission.
- A routing-layer enforcement rule with an integration test.
- A pre-authorised burst design per stamp (where applicable) with legal basis, scope, cap, notifications, and teardown.
- A completed chapter 04 §7 checklist with evidence.
- Attached legal consultation notes with approvals or documented follow-up.
- A tested IaC admission rule that blocks mis-residency deployments.

## Stretch goals

- **DPO escalation drill.** Run a 15-minute drill: simulate a cross-residency burst invocation request during an incident, time the DPO's response, document the escalation path's performance.
- **Multi-provider residency map.** For each contracted model provider, enumerate the provider's residency commitments per region; produce a map from service stamp to approved provider regional endpoints (chapter 04 §4.4).
- **Egress firewall.** Implement the chapter 04 §6.3 egress controls: block cross-residency dependency calls at the network layer; verify with an integration test.
- **User-notice template drafting.** Draft the user-notice template (chapter 06 §6.2) specific to this service's user base; circulate for legal review; include the review notes in the output.
- **Residency regression test in CI.** Add a CI job that runs the integration tests from Parts 2 and 3 on every PR; break the build on regression.
