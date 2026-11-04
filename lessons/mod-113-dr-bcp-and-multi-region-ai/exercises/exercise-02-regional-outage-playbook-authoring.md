# Exercise 02 — Regional Outage Playbook Authoring

**Estimated effort:** 3 hours · **Output:** the `rb-regional-failover` runbook for the student's chosen service at command-level specificity, four impairment-class decision branches, a measured weight-propagation RTO, a warm-standby SLI set with alerts, and a one-page walkthrough of the end-to-end failover sequence.

## Objective

Author a regional-outage playbook that an on-call engineer can execute without improvisation. The exercise's point is specificity: every substrate chosen, every command named, every threshold set, every alert wired. A playbook that says "failover to the secondary" is not a playbook; a playbook that lists the actual DNS record, the actual registry channel, the actual CLI invocation, and the actual dashboard URL is.

## Prerequisites

- Chapter 02 of this module.
- Exercise 01 completed; the service's characterisation and ISCP skeleton are in Git and will be referenced here.
- Admin access to the service's production (or a parallel staging environment) sufficient to read dashboards, write runbooks, and configure DNS / LB / registry / autoscaler primitives. A student without production access works in staging; the runbook is still command-specific.
- Access to the vendor documentation for the primary cloud substrate (AWS, Azure, or Google Cloud) and the chosen model provider(s).

## Problem statement

Work in six parts.

### Part 1 — Impairment-class taxonomy

Write the four impairment classes from chapter 02 §1 as they apply to the service:

- **Full regional loss.** Which signals indicate this; which runbook branch handles it.
- **Partial regional impairment.** Which specific regional-component failures are plausible for this service (AZ loss, specific endpoint degradation, autoscaler stall) and which branch handles each.
- **Degraded-200 regional case.** Which quality signals detect this; which branch handles it.
- **Control-plane-only outage.** Which signals distinguish it from a data-plane failure; which branch handles it (often hold-and-observe).

Each class gets a named branch in the runbook. The runbook's first step is "identify the impairment class."

### Part 2 — Endpoint-failover substrate choice

Choose one of the three substrates from chapter 02 §2 (DNS failover, BGP Anycast, global load balancer) and write a 300–500 word justification tied to:

- The service's latency / RPO / RTO targets.
- Streaming vs. request-response shape of the service's workload.
- The team's existing substrate investment.
- The cutover semantics (statistical vs. explicit vs. server-initiated).

Then produce the concrete configuration:

- The exact DNS record, Anycast IP, or global-LB endpoint.
- The health-check configuration (interval, path, success criteria).
- The failover action — the specific command or console operation that invokes the switch.

### Part 3 — Weight propagation RTO measurement

Measure the actual weight-propagation RTO for the service (chapter 02 §3.3):

- Pick a small, non-production weight artifact (or use a synthetic large object).
- Promote it in the primary region.
- Measure the time to resolvability in the secondary region.
- Repeat at least five times; record the samples; compute the p50, p95, p99.
- Verify the content hash matches on both sides.

Document the results. If the measured p99 exceeds the service's weight-propagation target from Exercise 01's design-time checklist, state the gap and the remediation (pre-staged promotion, increased bandwidth, upgraded replication tier).

### Part 4 — Warm-standby SLI set

Implement the four warm-standby SLIs from chapter 02 §4.4:

- `warm_standby_ready{region}`
- `warm_standby_capacity_ratio{region}`
- `warm_standby_quality_delta{region}`
- `warm_standby_weight_lag_seconds{region}`

For each:

- Define the measurement — a PromQL expression if Prometheus-backed.
- Set the alert threshold against the service's ISCP targets.
- Point the alert at the runbook branch that handles the deviation.
- Emit the metric; confirm it appears on the dashboard.

A synthetic probe that writes to the primary and reads from the secondary is a reasonable way to measure `warm_standby_quality_delta` for the exercise.

### Part 5 — Provider-regional affinity

Extend the service's model-provider integration to tag every inference request with `provider_region` (chapter 02 §5.2). Then:

- Wire a region-scoped circuit breaker (`mod-107` chapter 02 §2.1 extended) keyed on `(provider, region)`.
- Author the preferred-region list per route, consistent with the service's residency constraint (preview chapter 04; residency-sensitive students pass the list through the chapter 04 exercise).
- Document the breaker thresholds and the state transitions.

### Part 6 — End-to-end sequence authoring

Assemble the runbook. Follow the chapter 02 §6 sequence:

1. **Detect.** The named alerts (from Part 1 and Part 4).
2. **Decide.** The decision tree between hold, partial-shift, full-failover, with the input signals named.
3. **Execute.** The ordered command sequence for each branch.
4. **Preserve state.** A one-paragraph pointer to exercise 03's state-reconstitution plan.
5. **Communicate.** Pointer to the chapter 06 §6 templates (filled in by exercise 06).
6. **Recover.** The fail-back sequence, in order.
7. **Post-mortem.** The handoff to `mod-108` chapter 03.

The runbook is a Markdown file in Git with a stable path; it is linked from the service's ISCP (exercise 01).

## Requirements

1. **The playbook is a single Markdown file** named `rb-regional-failover-<service>.md` and committed under `runbooks/` in the service's repo.
2. **Every command is literal.** A reader who has never seen the service runs the exact command; it executes.
3. **Every dashboard is a URL.** Every alert is a named alert with a URL to its definition.
4. **The weight-propagation RTO is a measured number**, not a vendor-documented SLO. The vendor SLO is cited as context; the service's target is validated against measurement.
5. **The warm-standby SLI set is live.** Dashboards show the metrics; alerts route to the on-call paging substrate.
6. **The impairment-class branching is exhaustive.** A reader confronting a scenario not matched to a branch falls into a default branch that explicitly covers "unclassified; escalate."
7. **The runbook is linked from the ISCP** (exercise 01's §4 recovery section).
8. **The runbook's length is bounded** — a reader should execute it in 15 minutes, not read it for 45. Terse command lists, not prose.

## Starter guidance

- Reuse any existing runbook template on the team; do not invent a new structure. Consistency with the rest of the runbook library is more valuable than cleverness.
- The decision tree in Part 6 is where the specificity matters. "Decide based on the SLI" is not a decision tree; "if `svc_slo_burn_rate > 2.0` for 10 minutes AND `warm_standby_ready=true`, go to branch B" is a decision tree.
- Weight-propagation measurement should be a one-off script; commit it alongside the runbook so the measurement can be re-run on future vendor changes.
- For the provider-region affinity, start by adding a label to existing metrics; the breaker can follow once the attribution is reliable.

## Acceptance criteria

- `rb-regional-failover-<service>.md` in Git with the eight-point structure (detect, decide, execute × 4 branches, preserve state, communicate, recover, post-mortem).
- Four named branches for the impairment classes.
- A documented weight-propagation RTO with p50 / p95 / p99.
- Four live warm-standby SLIs with alerts routed to the paging substrate.
- A provider-regional affinity label on every inference request and a region-scoped breaker in the router.
- The runbook linked from the service's ISCP.
- A short rollout note (≤ 400 words) documenting what did not work on the first attempt and how it was fixed.

## Stretch goals

- **Terraform / IaC manifests.** Commit the IaC that provisions the DNS record, the global-LB configuration, and the health checks; the runbook references the module version.
- **Dry-run drill.** Run the runbook end-to-end in a staging environment; measure the actual steps, the actual time per step, and the actual decision latency; document what the drill found.
- **Second substrate.** Author a parallel runbook for a second substrate (if the primary is DNS, author the Anycast version); compare in a 500-word analysis.
- **Cost model.** Produce a one-page cost model for the warm-standby capacity (chapter 02 §4.2) across the three sizing bands and justify the chosen band against the service's ISCP targets.
- **Fail-back runbook.** Author `rb-regional-fail-back-<service>.md` as a separate runbook; many teams under-document this and discover its gaps in anger.
