# The SLO Lifecycle, Wired to Prometheus, Grafana, and OpenTelemetry

## Why this chapter exists

Chapters 01–04 defined the SLI surface and chapter 05 defined the policy that operates against it. Neither chapter said *how an SLI comes into existence*, *how it is kept accurate*, or *how it is retired* when the service it protects exits scope. That is the SLO lifecycle, and if it is not run, the SLO document drifts out of sync with production in weeks: SLIs reference pipelines that have been refactored, burn-rate alerts fire on long-decommissioned signals, cost targets reflect pricing from the last contract. A policy written against a stale SLO is worse than no policy, because it provides false confidence.

This chapter authors the six-stage lifecycle — define, baseline, instrument, alert, review, retire — and wires each stage to a concrete artifact in the Prometheus / Grafana / OpenTelemetry stack. The deeper observability treatment (semantic conventions, exemplars, trace propagation) belongs to `mod-104`; this chapter uses the stack *at authoring depth*, naming where each stage's artifact lives.

The classical reference for SLO lifecycle is [SRE Workbook, Chapter 2 — Implementing SLOs](https://sre.google/workbook/implementing-slos/); for alerting on SLOs, [SRE Workbook, Chapter 5](https://sre.google/workbook/alerting-on-slos/). The OpenTelemetry Semantic Conventions for Generative AI live at <https://opentelemetry.io/docs/specs/semconv/gen-ai/>.

## 1. The six stages

1. **Define.** Write the SLI definition — ratio of good events to valid events, valid-event filter, window. Author the SLO target and the burn-rate alert thresholds.
2. **Baseline.** Measure the current production SLI distribution over at least two full business cycles (14 days, 28 days) *before* committing to a target. Set the target against baseline, not against intuition.
3. **Instrument.** Wire the SLI into Prometheus (or equivalent) with clean metric names and dimensions. Emit the supporting trace events through OpenTelemetry. Build the Grafana panel.
4. **Alert.** Configure the multi-window / multi-burn-rate alerts. Route to the on-call and the ticket queue per chapter 05.
5. **Review.** Monthly SLO review consumes the alert history, the burn-rate trajectory, and the exception log. Decides whether SLI, target, or alert config needs adjustment.
6. **Retire.** When the service exits scope — decommissioned, replaced, merged into another — archive the SLO, remove the alerts, delete the panels, and record the retirement.

Each stage below is addressed with the artifact it produces.

## 2. Define

The artifact: the **SLO definition document**. One per SLI. Can be a single file holding the whole stack from chapters 01–04, or one file per SLI; pick a convention and keep it.

### What the definition document must contain

- **SLI name** — human-readable (e.g., "TTFT, cache-miss slice, chat surface, p95").
- **Metric identity** — the Prometheus series and OTel attributes that the SLI will be computed from. Full name and label keys.
- **Good-event predicate** — in operational prose. "A good TTFT event is a chat request where (a) the prompt did not hit the prefix cache in any layer and (b) the first-token-delta from accepted-timestamp was ≤ *T* ms."
- **Valid-event filter** — the denominator. "A valid TTFT event is a chat request where (a) prefix-cache miss was recorded, (b) the request completed (reached step 4 from chapter 03), and (c) client did not disconnect before first-token."
- **Window** — 28 rolling days is the default across this module.
- **Target** — the SLO percentage. Must be a number that was baselined (see section 3), not a guess.
- **Burn-rate alert thresholds** — the multi-window / multi-burn-rate table recomputed for *this* target per chapter 02 section 4.
- **Signal source and version** — for quality SLIs, the eval-peer-owned pipeline and its commit SHA. For streaming SLIs, the engine version and the gateway version. The point is *reproducibility*: another engineer must be able to recompute the SLI from the definition.
- **Owner** — the on-call team the SLI pages. Usually AI SRE; quality SLIs are co-owned with the eval peer.

### Keep it under one page per SLI

A two-page SLI definition is a signal of either accumulated ambiguity ("we had to say this twice because the first time was unclear") or accumulated special cases. Both are symptoms; fix them by tightening the valid-event filter until the SLI is one page.

## 3. Baseline

The artifact: the **baseline report**. One per SLI, one-time, re-run on material change.

### Why baseline before targeting

An SLO target picked without a baseline is an SLO target picked against the SRE's intuition — and intuition on tail-latency, judge pass-rate, and token-cost is empirically bad. The usual failure modes:

- **Target set too tight.** The SLO burns from day one. The on-call rotation is flooded with pages, the error-budget policy from chapter 05 becomes a permanent freeze, and the team learns to ignore the alerts.
- **Target set too loose.** The SLO never burns, which feels fine until an incident does serious user harm *and* the SLO reports clean. The SLO lost its diagnostic value.

Both failures come from the same root: the target was not grounded in production data.

### How to baseline

- **Measure the raw SLI over at least 14 days of representative traffic.** Longer is better; 28 days covers a full business cycle. If traffic is seasonal (weekly / monthly cycles), span at least two cycles.
- **Compute the percentiles of interest.** For a latency SLI, p50 / p95 / p99 / p99.9. For a quality SLI, the pass-rate distribution and the slice-level breakdowns. For a cost SLI, the per-1M-token rate over the window, with worst-day and best-day noted.
- **Document the distribution shape.** A p99 TTFT of 2s may be driven by a long tail of 32k-prompt requests; a p99 of 2s may also be driven by a 10% rate of 5s stalls. Same number, different pathology. The baseline report notes which it is.
- **Pick the target based on baseline + headroom.** A reasonable rule: target = current baseline * (1 - headroom factor), where the headroom factor encodes "how much can we burn before the user notices." For latency, 10–20% is typical; for quality, often less (quality SLOs tend to track close to baseline because the business model does not tolerate much regression).

### The baseline report template

- **SLI name.**
- **Baseline window** (start / end timestamps).
- **Traffic profile** — RPS, prompt-length distribution, output-length distribution, workload-class mix.
- **Measured distribution** — the percentiles and the distribution-shape commentary.
- **Proposed target** — the SLO number with the headroom-factor justification.
- **Risks** — known traffic shifts (upcoming launches, retiring surfaces) that could invalidate the baseline within the window.
- **Signed by** — the SLO author, the business sponsor who agrees to the target.

Baselines are re-run on material change: model upgrade, provider swap, major traffic-pattern shift. The review cadence (section 6) is where "material change" is adjudicated.

## 4. Instrument

The artifact: the **metric and trace schema** backing each SLI, plus the Grafana panels. Three levels — the raw signal, the ratio, the alert.

### Prometheus layer — the raw counters

Author two counters per SLI (classical SRE pattern): `*_total` for valid events, `*_bad_total` for bad events. Example for TTFT cache-miss:

```
# HELP chat_ttft_valid_total Count of valid TTFT measurement events (cache-miss slice).
# TYPE chat_ttft_valid_total counter
chat_ttft_valid_total{surface="chat",model="llama-3-70b",cache_slice="miss"} 123456

# HELP chat_ttft_bad_total Count of valid TTFT events whose TTFT exceeded the SLO threshold.
# TYPE chat_ttft_bad_total counter
chat_ttft_bad_total{surface="chat",model="llama-3-70b",cache_slice="miss"} 234
```

With the two counters, the 28-day SLI is a trivial PromQL expression:

```
# TTFT SLI — fraction of valid events that were good (not bad)
1 - (
  increase(chat_ttft_bad_total[28d])
  / increase(chat_ttft_valid_total[28d])
)
```

The classical ratio metric pattern applies without modification. A recording rule precomputes this at a rolling interval so Grafana does not evaluate the 28-day range on every panel load.

### OpenTelemetry layer — the span attributes

Every request emits a span (or spans) with attributes that let you *join* the SLI counter back to the per-request detail when debugging. Follow the [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) for the canonical attribute names:

- `gen_ai.request.model` — model identifier.
- `gen_ai.system` — provider / platform name.
- `gen_ai.response.finish_reason` — stop reason (used by partial-success SLI).
- `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens` — token counts for cost SLIs.
- `gen_ai.server.time_to_first_token` — TTFT.

Extend with SLI-specific attributes:

- `slo.cache_hit` — boolean for the TTFT cache slice.
- `slo.served_by_fallback` — for the graceful-degradation SLI.
- `slo.stream_completed_cleanly` — for partial-success SLI.
- `slo.max_inter_token_gap_ms` — for streaming-stall SLI.
- `slo.structured_output_valid` — for structured-output-conformance SLI.

Record the full attribute set on every request. When the Prometheus counter burns and the on-call needs to drill into which requests contributed, the trace exemplars become the bridge.

### Grafana layer — the dashboard contract

A single standardized layout per surface:

- **Header panel.** Current SLI, SLO target, budget remaining (28d), burn rate (1h / 6h / 3d). One panel per SLI, no exceptions.
- **Trajectory panel.** Rolling SLI over the window, with the SLO target line, so the burn trajectory is visible.
- **Burn-rate panel.** The multi-window burn rates, color-coded against the alert thresholds.
- **Breakout panels.** Per-model, per-provider, per-cache-slice — whatever dimensions the SLI is bucketed on.
- **Trace drill-down.** Link to the tracing backend, filtered by the SLI's bad-event predicate, so the on-call can jump from "the SLI is burning" to "here are 100 requests that burned it" in one click.

Keep the dashboard layout the same across SLIs. The on-call primary should be able to open any SLI's dashboard and know where to look without re-orienting.

## 5. Alert

The artifact: the **burn-rate alert rules** per SLI.

The multi-window / multi-burn-rate pattern from [SRE Workbook, Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) is the canonical reference. The implementation in Prometheus:

```yaml
# Fast burn — 2% of budget in 1 hour
- alert: ChatTTFTFastBurn
  expr: |
    (
      sum(rate(chat_ttft_bad_total[1h]))
      / sum(rate(chat_ttft_valid_total[1h]))
    ) > (14.4 * (1 - 0.99))
    and
    (
      sum(rate(chat_ttft_bad_total[5m]))
      / sum(rate(chat_ttft_valid_total[5m]))
    ) > (14.4 * (1 - 0.99))
  labels:
    severity: page
    slo: chat_ttft_cache_miss_p95
  annotations:
    summary: "TTFT fast-burn — 2% of 28d budget in 1h"
    runbook: "https://..."
```

Two things to note:

- **The two-window guard.** Fire only if both the 1h and 5m windows are burning. Classical two-window rule — suppresses single-sample noise. [SRE Workbook, Chapter 5](https://sre.google/workbook/alerting-on-slos/) is the reference.
- **The threshold multiplier.** `14.4 * (1 - target_SLO)` computes the burn-rate threshold against the specific target. Different target, different threshold — as noted in chapter 02 section 4.

The alert rule set per SLI typically contains three rules: fast-burn (page), medium-burn (page), slow-burn (ticket). Cost SLIs typically omit the fast-burn per chapter 04.

### Labels on alerts

The on-call rotation routing from `mod-102` chapter 03 depends on the alert carrying enough label metadata to route itself:

- `severity: page | ticket`.
- `slo: <slo-name>` (matches the SLO document).
- `surface: <product-surface>`.
- `runbook: <url>`.
- `co_owner: <peer-channel>` (e.g., `ai-eval-engineer` for quality SLIs).

Every alert that fires without a runbook URL is a reliability bug — fix at the alert-rule level, not in the on-call's head.

## 6. Review

The artifact: the **monthly SLO review log**. Standing meeting, standing agenda, written minutes.

### The standing agenda

1. **SLI health over the last 28 days.** One slide per SLI: current value, target, budget remaining, trajectory.
2. **Burn events.** Every SLI that was in burn-rate alert during the window. Review the incident, the response, the time-to-mitigation.
3. **Exceptions.** Every exception granted under the policy (chapter 05 section 7). Decide: absorb into policy, or mark one-off.
4. **Target adjustments.** Any SLI where the baseline has materially changed (traffic shift, model upgrade, provider change) and the target needs refreshing.
5. **New SLIs.** Any product surface that has grown enough reliability concern to warrant its own SLI.
6. **Retiring SLIs.** Services exiting scope (see section 7).
7. **Allocation refresh** (quarterly only, at the first review of the quarter).

### Attendees

- Primary / secondary on-call for the surface.
- SRE lead.
- Business sponsor for the surface.
- Eval peer for quality SLIs.
- Capacity-planning owner for cost / latency SLIs (consumers of the SLO-driven capacity sizing from `mod-106`).

The sponsor's attendance is load-bearing. Without the sponsor, the review becomes an internal SRE conversation and the SLOs drift away from the business contract. The sponsor's job is to say "the number you are proposing is / is not acceptable against what we promise the user."

### Minutes

Short, in a durable location. Minimum content per meeting:

- Attendees.
- Any target changes made this meeting, with justification.
- Any SLIs added or retired.
- Any exceptions reviewed and their resolution.
- Action items with owners and dates.

The minutes are the policy's *audit trail*. If an SLO is ever challenged ("why is the TTFT target 800 ms and not 500 ms?"), the review log answers it.

## 7. Retire

The artifact: the **SLO retirement record**.

### When to retire

- The service the SLO protects is decommissioned.
- The service was merged into another; the surviving service has its own SLO.
- The SLI was replaced by a better one (e.g., the hallucination SLI moved from a rule-based filter to a judge pipeline; the old SLI retires when the new one is in place).

### What retirement requires

- **Alert rules removed.** Prometheus / Alertmanager configs updated. Silence does not retire — delete the rules.
- **Grafana panels archived.** Keep the historical panels readable; remove them from active dashboards.
- **Runbook archived.** The runbook URL that the alert pointed to is marked retired.
- **Baselines archived.** The historical baseline reports stay available for context; they are not deleted.
- **Retirement record.** One paragraph in the SLO review log: the SLO name, the retirement date, the reason, the replacement (if any).

Retirement discipline matters. A retired-but-not-removed alert is worse than noise — on-call pages fire on dead signals, the team learns to ignore them, and legitimate alerts become background noise.

## 8. The lifecycle artifact set, summarized

| Stage | Artifact | Where it lives |
|---|---|---|
| Define | SLO definition document | Documentation repo, one file per SLI |
| Baseline | Baseline report | Documentation repo, one per SLI, timestamped |
| Instrument | Prometheus counters, OTel attributes, Grafana panels | Metrics / dashboards repo |
| Alert | Prometheus alert rules, Alertmanager routing | Monitoring repo |
| Review | Monthly SLO review minutes | Documentation repo, append-only |
| Retire | Retirement record | Appended to the SLO definition; alerts and panels removed |

Six stages, six artifacts, each addressable by URL. If any stage lacks its artifact, the SLO is *in-flight* — not operable, not countable toward the error-budget stack. The exercise in section 9 asks you to walk a sample SLO through all six.

## 9. Walking a single SLI end-to-end

To make the lifecycle tangible, trace the TTFT cache-miss SLI from chapter 01 through all six stages.

**1. Define.** `chat_ttft_cache_miss_p95` — valid events are chat requests with cache-miss; good events are those with TTFT ≤ *T* ms. 28d window. Target *to be set after baseline*. Owner: AI SRE.

**2. Baseline.** Run for 28 days against production traffic. Measured p95 = 820 ms, p99 = 1.4 s, p50 = 310 ms. Distribution has a long tail driven by 8k-prompt requests. Proposed target: p95 ≤ 1000 ms (20% headroom over baseline). Signed off by the chat product sponsor.

**3. Instrument.** Prometheus counters `chat_ttft_valid_total` and `chat_ttft_bad_total` with labels `{surface, model, cache_slice}`. OTel attributes `gen_ai.server.time_to_first_token` and `slo.cache_hit=false` on every request span. Grafana panel in the standard layout.

**4. Alert.** Three Prometheus alert rules — fast-burn (page), medium-burn (page), slow-burn (ticket). Labels include `slo`, `surface`, `runbook`, `severity`. Routing configured in Alertmanager to the chat-surface on-call.

**5. Review.** First month after launch: SLI value 97.8%, target 99%. The gap is a real burn; the on-call investigates. Root cause turns out to be a prefix-cache-eviction bug on high-fanout prompts. Fix lands, SLI returns to 99.3% in month two. The review log records the burn, the incident, and the fix.

**6. Retire.** Twelve months later, the chat surface is replaced by a unified chat-and-RAG surface. The TTFT cache-miss SLI retires; the replacement SLI on the new surface is already in place (went through its own lifecycle). Retirement record references the new SLI; alerts and panels for the old one removed.

Six stages, one SLI, a full year. Multiply by the 10–20 SLIs on a real stack and the lifecycle is a non-trivial operational load — which is why running the review cadence matters.

## Summary

- **Six stages** — define, baseline, instrument, alert, review, retire. Each stage produces a concrete artifact; a stage without an artifact is a stage that did not happen.
- **Define** — SLI definition document, one per SLI, under one page, with explicit good-event / valid-event predicates and signal-source versioning.
- **Baseline** — measure production over at least two business cycles before setting the target. Avoid the "set too tight" and "set too loose" failure modes.
- **Instrument** — Prometheus `_total` and `_bad_total` counters for the ratio, OpenTelemetry Generative AI semantic conventions for the per-request span attributes, Grafana panels in a standardized per-surface layout.
- **Alert** — multi-window / multi-burn-rate rules in Prometheus, per-SLI threshold recomputation, labels for routing, runbook URL on every alert.
- **Review** — monthly cadence, sponsor present, agenda covers burn events, exceptions, target adjustments, new SLIs, retirements, quarterly allocation refresh.
- **Retire** — when service exits scope: remove alerts, archive panels, record the retirement. Discipline against noise.

The next chapter closes the module with the explicit peer boundary to `ai-eval-engineer` — what the SLO contract is, and what the eval peer owns underneath it.
