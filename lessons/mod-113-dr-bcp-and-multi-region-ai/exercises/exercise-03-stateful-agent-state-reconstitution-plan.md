# Exercise 03 — Stateful Agent State Reconstitution Plan

**Estimated effort:** 3 hours · **Output:** the per-service state-class inventory, a measured cross-region read-after-write probe with alert wiring, the three-way tool-call recovery decision matrix implemented for every tool in the service's catalogue, a trace-store buffer sized against the ISCP RTO, and the completed per-service RPO / RTO matrix from chapter 03 §7.

## Objective

Operationalise chapter 03's state taxonomy for the student's chosen service. The exercise produces the data that the regional-outage playbook (exercise 02) assumes: the conversation store survives failover, the tool-call state resumes rather than re-issues, the trace store does not drop writes, the registry degrades to static-mode gracefully. The exercise is design + implementation; it concludes with a demonstrated cross-region write / read / recover cycle.

## Prerequisites

- Chapter 03 of this module.
- Exercises 01 and 02 completed; the ISCP and the regional-failover runbook exist.
- Read / write access to the service's state substrates — conversation store, tool-call state store, registry, trace emission pipeline.
- Access to the inventory of tools the service invokes, with their idempotency documentation (`mod-105` chapter 05).

## Problem statement

Work in six parts.

### Part 1 — State-class inventory

For the student's service, enumerate each of the six state classes from chapter 03 §1. Per class:

- Does this service have state in this class? Yes / no.
- Where does it live (specific store, specific service, specific region).
- The volume (per-request and per-tenant).
- The access shape (read-dominant, write-dominant, mixed).
- The RPO sensitivity (as classified in chapter 03) and the service-specific RPO target.
- The RTO sensitivity and the service-specific RTO target.

A class marked "no" is a design decision — stated explicitly, not ignored. ("This service has no distributed prefix cache because we have not yet justified the investment" is a valid entry.)

### Part 2 — Conversation / memory state cross-region probe

Implement a cross-region read-after-write probe for the conversation store (chapter 03 §2.3). The probe:

- Writes a well-known key with a timestamp payload to the primary.
- Reads the same key from the secondary on a tight loop (every 100 ms or finer).
- Records the time from write to first successful matching read.
- Emits the result as `conversation_store_read_after_write_seconds{src_region, dst_region}`.
- Runs continuously at low cost (once per minute is sufficient for the exercise; higher cadence is appropriate for production).

Wire alerts:

- Threshold on p99 cross-region lag.
- Threshold on read failure rate.

Point the alerts at the `rb-regional-failover` runbook branch that handles replication degradation.

### Part 3 — Session affinity verification

Confirm the service's routing-layer session affinity (chapter 03 §2.2). Pick:

- The affinity key (session cookie, user ID, device identifier).
- The sticky duration.
- The behaviour on key absence (fail-safe to residency-correct stamp, or fall through to geographic routing).

Write a 200-word statement of the policy; cite the configuration file or routing rule where the policy is enforced. If the service does not currently have affinity, produce the change proposal with the specific config diff.

### Part 4 — Tool-call recovery decision matrix

For every tool in the service's catalogue — not a representative sample, every tool — produce one row of the three-way recovery decision matrix:

| Tool | Side effects | Idempotency key present | Replay safe? | Resume safe? | Reconcile mechanism | Recovery decision |

For a tool where the recovery decision cannot be made safely (no idempotency, external side effects), either:

- Add the idempotency support upstream of this exercise's completion, or
- Explicitly mark the tool as "operator-in-the-loop on resume"; document the operator contact; name the safeguard in the ISCP (exercise 01).

The matrix is in the service's `runbooks/` or ISCP directory in Git.

### Part 5 — Trace-store buffer sizing and emission test

Size the local OTel collector buffer (chapter 03 §5.2) against the service's ISCP RTO + a margin:

- State the RTO target.
- State the collector's steady-state write rate.
- State the chosen buffer size (time or bytes) and the rationale.
- Configure the buffer via the collector's YAML (the OpenTelemetry Collector's `file_storage` extension or equivalent persistent-queue mechanism).

Then run an emission-survival test:

- In a staging environment, start the service under synthetic load.
- Block the regional ingest endpoint for a duration equal to the ISCP RTO plus the margin.
- Unblock the endpoint.
- Confirm no traces were lost — count spans emitted vs. spans ingested.

Document the test procedure, the results, and any gap between the configured buffer and the surviving duration.

### Part 6 — The per-service RPO / RTO matrix

Assemble the chapter 03 §7 matrix with the measurements and decisions from Parts 1–5:

- Columns: state class, RPO target, RTO target, substrate, reconciliation policy.
- Rows: one per in-use state class.
- An extra column: "latest measurement" with the value from the probe (Part 2) or the sizing test (Part 5), with the date of the measurement.

Commit the matrix as a Markdown table in the service's ISCP (exercise 01's §4 recovery section), replacing the skeleton the ISCP had.

## Requirements

1. **The inventory covers all six state classes**, with explicit "no" entries where applicable.
2. **The cross-region read-after-write probe is running continuously** and emits a labelled metric. Alerts route to the paging substrate.
3. **Session affinity policy is cited in code**, not described in prose.
4. **The tool-call decision matrix has one row per tool.** A tool without a safe recovery decision is explicitly flagged.
5. **The trace-emission survival test runs in staging** and passes against the ISCP RTO + margin. The procedure and results are committed.
6. **The RPO / RTO matrix is in the ISCP**, linked from the service's README, and references the measurement date so staleness is visible.
7. **The reconciliation policy for the conversation store** is documented (append-only union / primary-wins / manual) and the choice is justified against the service's write pattern.
8. **The matrix is reproducible.** A successor engineer can re-run Part 2's probe and Part 5's test with no additional context.

## Starter guidance

- The hardest part is often Part 4. Teams that adopted agent patterns without idempotency discipline have tools whose recovery decision is "we would guess." The exercise forces that out into the open; closing the gap may take a separate initiative with the tool owners.
- For Part 2, the simplest probe is a small Python or Go service running in two regions. Keep it simple; the point is the measurement, not the probe's generality.
- Part 5 is easier in staging than production. Do not block production ingest in anger to get this measurement.
- The matrix is not a one-time artifact. Re-run the measurements quarterly or on substrate change; the drill programme (exercise 05) will test the matrix's claims.

## Acceptance criteria

- A state-class inventory in Git with all six classes addressed.
- A running cross-region probe emitting labelled metrics, alerts wired.
- A session-affinity policy documented in code.
- A tool-call recovery decision matrix with 100% coverage of the service's tool catalogue.
- A trace-emission survival test passing in staging, with procedure and results committed.
- An updated RPO / RTO matrix in the ISCP, with measurement dates.
- A short write-up (≤ 500 words) summarising the measurements and the open gaps.

## Stretch goals

- **Multi-region substrate migration.** If the conversation store is single-region today, propose the migration to a multi-region substrate (DynamoDB global tables, Cosmos DB multi-write, or equivalent); produce the migration plan with cost and risk.
- **Durable-execution framework.** Prototype one long-running agent workflow on top of Temporal or AWS Step Functions (chapter 03 §4.4); demonstrate the mid-run resume after a simulated region switch.
- **Reconciliation drill.** Run the reconciliation policy against a synthetically produced split-brain conversation state; verify the policy's outcome matches the ISCP's claim.
- **Static-mode drill.** Block the registry for a defined window; verify the serving tier runs cleanly in static-mode cache; document the duration before any observable degradation.
- **Trace-store gap post-mortem.** If the Part 5 test fails on the first run (common — the collector's buffer is often undersized), write a brief post-mortem using the `mod-108` chapter 03 template.
