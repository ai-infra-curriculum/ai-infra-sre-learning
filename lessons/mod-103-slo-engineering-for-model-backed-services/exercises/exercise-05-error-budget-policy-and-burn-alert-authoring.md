# Exercise 05 — Error-Budget Policy and Burn-Alert Authoring

**Estimated effort:** 3 hours · **Output:** one Markdown file (~2000 words) — the full error-budget policy document plus the burn-alert configuration

## Objective

Produce the complete error-budget policy document from chapter 05 section 9 for the SLO stack you have been building in exercises 01–04. Then produce the Prometheus-shaped burn-rate alert configuration per SLI from chapter 06 section 5. The result is the operable artifact the on-call rotation reads during an incident — not a design discussion, not a conceptual essay.

Chapter 05 is the prerequisite. If you have not read it, do so before starting.

## Prerequisites

- Exercises 01, 02, 03, and 04 completed. (Your SLO stack must be complete and composed.)
- Chapters 05 and 06 of this module.
- Working knowledge of multi-window / multi-burn-rate alerting ([SRE Workbook, Chapter 5](https://sre.google/workbook/alerting-on-slos/)).

## Problem statement

You have a composed SLO stack across latency, quality, streaming-availability, and cost. The on-call rotation from `mod-102` chapter 03 exists but has no written policy against your specific SLIs. Author the policy and the alert config.

Author a Markdown file with these two parts.

## Part A — the policy document

The eight-section policy from chapter 05 section 9. Treat each section as a 2–4 paragraph artifact the primary on-call reads at 03:00.

### Section 1 — SLO surface

Compact restatement of the stack from exercises 01–04. No new SLIs; refer to those documents for detail. A single table is sufficient — the column slugs are the parse contract so a downstream review tool can diff the row set against your exercise-01/02/03/04 outputs:

| Column | Content |
|---|---|
| `sli` | SLI name, exactly as written in exercises 01–04 |
| `dimension` | `latency` / `quality` / `availability` / `cost` |
| `target` | SLO percentage (and p-level for latency) |
| `window` | Rolling window (28d default per chapter 06 §2) |
| `burn_alert_severity` | `page` / `ticket` (cost SLIs are `ticket`-only per chapter 04) |

### Section 2 — Freeze-targeting map

The per-SLI freeze map from chapter 05 §2. One row per SLI. Columns: `sli`, `change_types_frozen_on_burn`, `authorizes_exceptions` (usually the incident commander).

The `change_types_frozen_on_burn` column draws from this controlled vocabulary (reproduced from chapter 05 §2 — pick the applicable subset per SLI):

```
- serving-fleet-infra       (manifest / Helm / engine-version / HPA policy changes)
- engine-config             (batch size, scheduling, prefix-cache knobs)
- new-model-rollout         (any model-weight or model-id change on the surface)
- prompt-template-change    (system or template-level prompt edits)
- retrieval-index-pin       (index version / embedder change)
- provider-route            (fallback or primary-provider routing)
- fallback-path-change      (graceful-degradation path updates)
- guardrail-sidecar         (PII filter / policy engine changes)
- tool-schema-change        (tool-calling surface schema edits)
- constrained-decoding      (structured-output configuration)
- ui-contract               (format-drift-adjacent frontend changes)
- feature-deploy-on-surface (classical SRE feature deploy against the burning surface)
- does-not-freeze           (used for cost SLIs — negotiation trigger, not a freeze)
```

Address every SLI in your stack, including the cost SLIs (whose row is `does-not-freeze`, with the chapter 04 §5 negotiation path named in the authorization column).

### Section 3 — Allocation rule

Canary / feature / refactor split for the quarter, keyed to the three categories named in chapter 05 §3. Justify any deviation from the 40 / 40 / 20 default. The chapter's named phase templates (reproduce or document the per-team deviation):

| Phase | Canary | Feature | Refactor |
|---|---|---|---|
| Steady-state (default) | 40% | 40% | 20% |
| Early-stage rollout | 60% | 20% | 20% |
| Reliability pressure | 20% | 30% | 50% |

Write the split *per overall budget*, not per SLI. Per chapter 05 §3, pre-allocating budget per SLI ("quality budget for canaries, latency budget for feature deploys") is a mistake — the budgets burn in correlated ways and the pre-allocation becomes an accounting fiction.

### Section 4 — Burn bands

The four-band table from chapter 05 §2. Columns: `remaining_budget_band`, `canary_allowed`, `feature_deploy_allowed`, `refactor_allowed`, `additional_required_action`.

The four bands are fixed by chapter 05 §2 — reproduce the band thresholds exactly so a policy-as-code manifest (stretch goal) can enum against them:

```
- band-1: remaining ≥ 80%
- band-2: 50% ≤ remaining < 80%
- band-3: 25% ≤ remaining < 50%
- band-4: remaining < 25%
```

Be operable. "Canary allocation reduces by half" is vague. "At most one canary deploy per week; no new model rollouts; no new prompt templates; existing canaries continue" is operable. The chapter 05 §2 reference rendering (reproduce the *shape*, replace the specifics with your team's numbers):

| Band | Canary | Feature deploy | Refactor |
|---|---|---|---|
| `band-1` | normal cadence | normal cadence | normal cadence |
| `band-2` | reduced — one canary per week, no new model rollouts | unchanged | unchanged |
| `band-3` | halted (except rollbacks) | halted on this surface | doubled — this is where the fix lives |
| `band-4` | halted | halted | reliability sprint; SLO review called early |

Bands apply *per SLI* — do not collapse the stack into an aggregate-budget rule.

### Section 5 — Correlated-burn rule

Explicit rule per chapter 05 §4 case 3. Required pieces:

- **The correlation window** — 2 hours by default; justify a different window if you pick one. The chapter notes some teams prefer 1 hour (tighter) or 4 hours (wider); do not leave the window as an implicit judgment call.
- **The correlation threshold** — chapter 05 §4 case 3: "two or more SLIs simultaneously under 50% budget remaining *and* time-correlated within the window." State this threshold in your rule; do not substitute your own.
- **The escalation path** — the IC is paged *before* single-SLO responses are applied (chapter 05 §4 case 3). Name the IC rotation and the paging mechanism; name the eval-peer co-investigation hook for cases where quality is one of the correlated dimensions.
- **The handoff** — from the single-SLI runbooks (`mod-108` top-twenty set) to the correlated-burn runbook. Name the runbook URL, even if a placeholder.

### Section 6 — Eval-peer coordination rule

The two rules from chapter 05 §5, parameterized for your workload. Reproduce both as named sub-rules so the on-call can scan them:

1. **Rule 1 — a gate-passing release that burns the production SLO can be rolled back by on-call without re-invoking the gate.** Name the rollback mechanism (point at `mod-112`) and the artifact set the on-call files back to the eval peer (the burning traffic slice, the judge calls on it, the pass-rate differential vs the gate).
2. **Rule 2 — a gate-failing release cannot ship by spending budget.** Name the exception path in Section 7 as the sole channel for gate-failing-release ships (e.g., security hotfixes).

Name the eval peer's channel (if one exists on your team) or mark the dependency as "to be stood up."

### Section 7 — Exception process

The five-element exception process from chapter 05 §7. The five elements are fixed by the chapter; reproduce them as named sub-headings with your team's values filled in:

| Element | Chapter 05 §7 default | Fill in |
|---|---|---|
| `who_grants` | IC for in-incident; SRE lead for planned | your rotation / role |
| `duration` | Time-boxed, 24–48 hours typical, auto-expires | your value |
| `trigger` | Named scenario (provider outage, security hotfix, cost burn requiring capacity add) | your scenarios |
| `logged_in` | Incident channel or SLO review log | your channel / log |
| `reviewed_at` | Next SLO review; absorb or mark one-off | your cadence |

An exception that cannot be named as a scenario before it is invoked is a policy gap, not an exception.

### Section 8 — Review cadence

Monthly SLO review, quarterly allocation refresh, annual policy audit (chapter 06 §6). Name attendees from the controlled vocabulary in chapter 06 §6 (one role per attendee slot — rotate-named humans do not belong in the policy):

```
- primary-on-call-for-surface
- secondary-on-call-for-surface
- sre-lead
- business-sponsor-for-surface
- eval-peer                            (required when any quality SLI is on the agenda)
- capacity-planning-owner              (required when cost or latency SLIs are on the agenda)
```

The `business-sponsor-for-surface` attendance is load-bearing per chapter 06 §6 — without the sponsor, the review drifts into an internal SRE conversation and the SLOs decouple from the business contract. State the sponsor explicitly.

## Part B — the burn-rate alert configuration

Prometheus-shaped alert rules, one YAML block per SLI. Per SLI, three rules: fast-burn (page), medium-burn (page), slow-burn (ticket). Cost SLIs omit fast-burn per chapter 04.

The three-rule contract (reproduced from [SRE Workbook, Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) §"Multiwindow, Multi-Burn-Rate Alerts" — the long/short windows and multipliers are the parse contract, not a suggestion):

| Rule | Budget burn | Long window | Short window | Burn-rate multiplier | Severity |
|---|---|---|---|---|---|
| `fast-burn` | 2% of 28d budget in 1h | 1h | 5m | `14.4 × (1 - target)` | `page` |
| `medium-burn` | 5% of 28d budget in 6h | 6h | 30m | `6 × (1 - target)` | `page` |
| `slow-burn` | 10% of 28d budget in 3d | 3d | 6h | `1 × (1 - target)` | `ticket` |

Both the long and short windows must breach the multiplier before the rule fires — the two-window guard from chapter 06 §5 is non-negotiable. Single-window burn-rate alerts are noisy.

For each rule, include:

- `alert` — the alert name, kebab-case, SLI-prefixed (e.g., `ChatTTFTCacheMissFastBurn`).
- `expr` — the PromQL expression with the two-window guard (`and` between the long- and short-window burn-rate checks, following the chapter 06 §5 example shape).
- The threshold multiplier *computed against the specific SLO target* — a fast-burn threshold of `14.4 × (1 - 0.999) = 0.0144` on a 99.9% SLO becomes `14.4 × (1 - 0.97) = 0.432` on a 97% SLO (chapter 02 §4 and starter guidance below).
- `labels` drawn from the chapter 06 §5 controlled vocabulary (every label is required unless marked optional):
  - `severity` — `page` or `ticket`.
  - `slo` — the SLO slug, matching the Section 1 table.
  - `surface` — product surface the alert fires against.
  - `runbook` — URL (placeholder path acceptable; see Requirements below).
  - `co_owner` — optional; required for quality SLIs (`ai-eval-engineer` channel per chapter 07) and recommended for cost SLIs.
- `annotations.summary` — one-line description for the on-call.

The YAML does not have to be deployable against your real cluster; it has to be *readable and correct* for the SLI targets you set in exercises 01–04. A reviewer should be able to drop the rule into a Prometheus config and it should parse.

At minimum, produce rules for five SLIs covering at least three dimensions (e.g., TTFT-cache-miss, hallucination, partial-success, cost-per-1M-output-tokens, structured-output-conformance). If you want to cover the full stack, do so.

## Requirements

- **Policy document has all eight sections.** No TBD, no placeholder.
- **Section 1 table uses the five column slugs given** (`sli`, `dimension`, `target`, `window`, `burn_alert_severity`).
- **Freeze-targeting map addresses every SLI** in the stack, including cost (`does-not-freeze`). The `change_types_frozen_on_burn` entries draw from the 13-slug controlled vocabulary in Section 2.
- **Allocation rule names one of the three phase templates** (steady-state / early-stage rollout / reliability pressure) or documents a per-team deviation with reason.
- **Burn-band table uses the four fixed bands** (`band-1` through `band-4`) with the exact thresholds given. Each row names specific numbers (canary cadence, deploy count, authority) — no handwaves.
- **Correlated-burn rule names the correlation window, the correlation threshold** (two-or-more SLIs under 50% budget within window), the IC rotation, and the handoff runbook URL.
- **Eval-peer coordination rule reproduces both numbered sub-rules** with the rollback mechanism named for Rule 1 and the exception-process pointer for Rule 2. If no peer exists, the dependency is flagged.
- **Exception process names all five elements** (`who_grants`, `duration`, `trigger`, `logged_in`, `reviewed_at`).
- **Review cadence names attendees from the six-slug controlled vocabulary** in Section 8, including the `business-sponsor-for-surface`.
- **Alert YAML parses** (even if not deployed). Each rule uses the two-window guard pattern. Thresholds are computed per SLO using the multiplier table from Part B, not copy-pasted.
- **Every alert has a runbook URL** (can be a placeholder path — e.g., `runbooks/ttft-cache-miss-burn.md` — the point is the alert is never fired without one).
- **Every alert carries the five-label set** (`severity`, `slo`, `surface`, `runbook`, plus `co_owner` where applicable).
- **Three-minute decision ladder visible.** A new primary on-call reading the policy for the first time should resolve action for the five-step ladder (SLI → band → correlation? → dimension → permitted actions) within three minutes.

## Starter guidance

- Chapter 05 section 8 walks a worked example end-to-end. Use it as the shape; do not copy the numbers.
- The hardest section to write is section 5 (correlated-burn). Spend extra time there. The correlation-window number is a design decision — too tight (30 minutes) and you miss slow-correlated incidents; too loose (8 hours) and you treat unrelated incidents as correlated. Justify your choice.
- For the allocation rule, be realistic. A team shipping its first SLO stack is probably not in equilibrium; the first quarter's split may be 20 / 20 / 60 (heavy reliability investment) until the stack is stable. Note that you expect to re-set in the second quarter review.
- For the exception process, look at how your organization handles emergency-change authority outside the SRE team. Rhyme with that; a standalone escape valve that nobody has seen before will not survive its first use.
- Burn-rate alert thresholds are the most common bug in operable policies. Compute them fresh per SLO. A fast-burn threshold of `14.4 × (1 - 0.999) = 0.0144` on a 99.9% SLO becomes `14.4 × (1 - 0.97) = 0.432` on a 97% SLO — a 30× difference. Copy-pasting one threshold across the stack is the fastest way to a broken alerting system.
- For the alert YAML, model on the chapter 06 section 5 example. The two-window guard pattern (`and` between the long and short window) is non-negotiable; single-window burn-rate alerts are noisy.

## Acceptance criteria

- Markdown file is self-contained and ≤ 2500 words.
- Part A has all eight sections populated, each using the parse-contract column slugs / sub-headings given above.
- Part B has three alert rules per SLI for at least five SLIs (cost SLIs omit `fast-burn`).
- Every alert rule has a threshold value computed against its specific SLO (not a generic number) using the `14.4` / `6` / `1` multipliers from Part B.
- Every alert rule includes the long/short two-window guard pattern.
- Every alert rule carries the `severity` / `slo` / `surface` / `runbook` labels, plus `co_owner` for quality SLIs.
- A primary on-call can resolve a hypothetical burn (pick any SLI, any band) in three minutes against the policy without asking.
- Correlated-burn rule names the correlation window, the two-or-more-SLIs-under-50% threshold, and routes to the IC.
- Cost SLIs appear in the freeze map as explicit `does-not-freeze` entries with the chapter 04 §5 negotiation path named, not omitted.
- Exception process addresses all five fixed elements (`who_grants`, `duration`, `trigger`, `logged_in`, `reviewed_at`).
- Review-cadence attendee list includes the `business-sponsor-for-surface` slot, filled.

## Stretch goals

- **Policy-as-code manifest.** Encode the burn-band table from section 4 as a machine-readable YAML or JSON manifest your deploy tooling could consume to auto-gate rollouts at specific bands. Chapter 05 did not require this; the exercise rewards it.
- **Policy walk-through with three hypothetical incidents.** Append a short appendix: resolve three scenarios under the policy — a cost-only burn, a correlated latency+quality burn traced to a provider, and a hallucination-only burn traced to a prompt regression. Each scenario should resolve to a clear action within the three-minute ladder.
- **Peer review simulation.** Have the policy reviewed by someone who does not share your mental model — a product engineer, a classical SRE, the eval peer — and capture one surprising objection in the appendix with your revision.
- **Policy legibility test.** Time how long it takes a reader to find the response to "hallucination rate burning at 40% budget, no other SLIs burning." If longer than three minutes, restructure. A tested policy is better than a comprehensive one.
