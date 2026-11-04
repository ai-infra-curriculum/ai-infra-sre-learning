# Exercise 01 — Guardrail Sidecar Latency Budget Authoring

**Estimated effort:** 2 hours · **Output:** a signed per-hop latency-budget contract for a chosen route on a chosen fleet, plus the SLI / alarm set that evidences it in operation

## Objective

Produce the function's **guardrail latency-budget contract** — the signed artifact that commits each defensive control on a route to a specific per-hop latency allocation, names the SLIs that evidence the allocation in production, wires the SLO burn-rate alarms that fire when the allocation is breached, and commits the negotiation discipline for future changes. The contract is the thing a joint post-mortem (exercise 02) references by name when Shape A fires; its absence is a predictable failure mode of first-generation AI SRE functions.

A good deliverable is a document a would-be deploy pipeline can read once and refuse to promote a change that exceeds its budget. A bad deliverable is a dashboard screenshot with no commitments.

## Prerequisites

- Chapter 01 of this module.
- `mod-103` (SLI / SLO vocabulary) — the TTFT / TPOT SLOs the per-hop budgets decompose from.
- `mod-104` (observability stack) — the labels, histograms, and recording rules the SLIs will land in.
- `mod-108` chapter 01 (runbook library) for the runbook shape.
- Access to or a plausible representation of a model-backed product with at least one route, a working defensive stack (even a stub), and an observability stack.

## Problem statement

Work in four parts.

### Part 1 — Describe the route and its serving SLO

In 200–300 words:

- Name the route (e.g., `chat`, `summariser`, `rag-search`, `structured-extraction`).
- Name the TTFT, TPOT, availability, and (where applicable) streaming-stall SLOs in force for the route.
- List the defensive controls on the route's request and response paths, in order. For each, name the owner (`ai-infra-security`, platform, policy, retrieval auth, etc.).
- Link the dashboard that currently renders end-to-end latency for the route, and note whether per-hop histograms exist today. If they do not, state that explicitly as a gap the contract will close.

### Part 2 — Draft the per-hop allocation table

Produce the allocation table from chapter 01 §2.1, filled for this route. One row per hop. Columns:

- Hop name.
- p50 budget.
- p95 budget.
- p99 budget.
- Owner.
- Current measurement (p50 / p95 / p99 from the past 30 days; "unknown" is a valid value that flags a §2.2 gap).
- Fail-behaviour (fail-open, fail-closed, fail-quiet, retry-with-backoff) and the justification.

The sum of the p95 column must be less than the route's p95 serving SLO. If it is not, the exercise forces one of two choices: the budget is reshaped, or the SLO is renegotiated — both of which are tracked explicitly in the gap list.

### Part 3 — Author the SLI set and the burn-rate alarms

For each hop in the table, write:

- The Prometheus / equivalent histogram name, label set, and recording rule.
- The burn-rate alarm expressions (fast-burn and slow-burn), referencing the hop's p95 and p99 budget as the SLO target.
- The pager routing — which function's on-call is paged, and whether the other function is informed.

A minimum of three hops have working SLIs and alarms at the end of the exercise (even if authored as PRs rather than merged); the rest are listed as gaps.

### Part 4 — Author the negotiation discipline

Write the function's commitment to the §2.4 negotiation discipline:

- How a security peer proposes a new or changed defensive control (template, required fields).
- How the AI SRE reviews and allocates budget from the SLO.
- How the control is piloted (canary stage, experiment from chapter 03 of `mod-109`).
- How changes that move the latency distribution are tracked (git-committed contract revision, joint sign).
- The gap list — artifacts the contract depends on that are not yet in place, each with an owning peer and a blocker.

## Requirements

1. **The route is named, not generic.** A reader should know which routes, which providers, which observability stack.
2. **The allocation table is complete.** Every hop on the request / response path has a row. Hops with "unknown" measurement are flagged as §2.2 gaps.
3. **The sum of the p95 column respects the serving SLO.** If it cannot, the gap is named and the resolution (reshape or renegotiate) is committed to.
4. **Every hop has a fail-behaviour.** No hop reads "TBD" on fail-behaviour.
5. **At least three hops have authored SLIs and alarms.** The rest are in the gap list with a scheduled date.
6. **The negotiation discipline is a concrete workflow**, not an aspiration. The template exists; the pilot path references a specific chaos experiment; the sign-off is a named role, not "security."
7. **The document is signed.** AI SRE function lead and `ai-infra-security` lead (names or role-handles, not "TBD"). The sign is itself a Shape A prevention.
8. **No content from the attack catalogue.** The exercise owns the latency budget; it does not author classifier content.

## Starter guidance

- Start with the route description and the serving SLO. Imprecision there cascades.
- Map the request path physically — walk from edge ingress to model call and back. Draw it on paper; the drawing is the input to the table.
- For hops whose current latency is unknown, do not guess. Mark "unknown" and prioritise the instrumentation as the first gap.
- Fail-behaviour is the most underdiscussed field. For each hop, ask: when this hop is slow / down / errored, what should the service do? Pre-commit the answer out of incident.
- Alarms must be scoped so that both fast-burn and slow-burn produce distinct pages. A hop that only has a fast-burn alarm misses week-long drift; a hop that only has a slow-burn alarm misses a one-hour outage.
- The negotiation template is the primary preventive artifact against Shape A incidents. It is worth spending time on even if it looks like paperwork.
- The gap list is the document's honesty. A contract with no gap list has not been read critically.

## Acceptance criteria

- A single document (≤ 3000 words) titled `guardrail-budget-<route-slug>.md` in the function's documentation repo or the exercise workspace.
- The allocation table has one row per hop, with all seven columns filled.
- The sum of the p95 column is less than the route's p95 serving SLO, or the gap is explicitly named.
- Every hop has a declared fail-behaviour.
- At least three hops have working SLIs and burn-rate alarms (PR links are acceptable if not yet merged).
- The negotiation discipline is a concrete template, pilot path, and sign-off roster.
- Signed by the AI SRE function lead and the `ai-infra-security` lead.
- The gap list has at least three entries, each with blocker and owning peer.

## Stretch goals

- **Extend to a second route** so the contract covers the function's representative routes, not just one.
- **Draft a chaos experiment** (per `mod-109` chapter 03) that verifies the budget holds under load, and link it in the contract.
- **Add a cost-budget column.** Some controls (LLM-as-a-classifier, external DLP services) have a per-request cost; adding the cost dimension prevents a Shape A cost blow-up.
- **Translate the contract to a release-gate check.** `mod-111`'s release engineering reads this check at promotion; author the first version of the gate expression.
- **Dry-run a Shape A incident** against the contract — walk a realistic defensive-change scenario and confirm the contract catches the regression. The exercise is informal; the finding is a fixture.
