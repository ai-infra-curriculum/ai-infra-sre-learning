# Exercise 03 — On-Call Rotation Design for an AI SRE Team

**Estimated effort:** 2 hours · **Output:** one Markdown file (~1500 words) + a hand-off template

## Objective

Design the on-call rotation that enforces the error-budget policy from exercise 02. The output is two artifacts:

1. A **rotation-design document** — shape, roles, escalation ladder, on-call-health metric plan, paging policy — for the team you have been carrying through this module.
2. A **hand-off checklist template** your team will use at every rotation hand-off.

Both artifacts should be operable by a team member reading them cold.

## Prerequisites

- Chapter 03 of this module.
- Exercises 01 and 02 (same workload + same policy).

## Problem statement

You have the SLI stack (exercise 01) and the policy that governs budget spend (exercise 02). You have not yet designed the rotation that executes the policy in real time. This exercise produces that design.

Work against a concrete team size — pick one of the three canonical cases:

- **Case A — small team (4–6 engineers).** Single-region default.
- **Case B — medium team (7–12 engineers).** Single-region with mature pager discipline, or early FTS if overnight volume justifies.
- **Case C — large team (≥ 13 engineers).** FTS-eligible across two or three regions.

Pick the case that matches your workload; if your workload is small (which it probably is), do not fight the math and pick Case A.

## Requirements

### Artifact 1 — Rotation-design document

#### Section 1 — Shape

Name your team size + region strategy. Justify in two sentences against chapter 03 section 1:

- If single-region, how do you cap overnight pages?
- If FTS, how many engineers per region, what is the hand-off cadence?

Explicitly address the "did you tune the pager before splitting?" test from the chapter.

#### Section 2 — Three roles

Per chapter 03 section 2, write:

- **Primary** — pager-holder, response-time SLO (MTTA numbers), triage boundary (when does a page become an incident?). The chapter's defaults are MTTA ≤ 5 min business hours / ≤ 15 min overnight, and "declare incident if the page is not resolved within 30 min of triage" — reproduce these numbers or state your deviation.
- **Secondary** — backup response-time SLO, when do they engage, how do they coordinate with the primary during an incident. The chapter's defaults are the 10-min pager fan-out window from primary if unacknowledged, plus the rolling-incident-shadow function — reproduce or state your deviation.
- **Incident commander** — rotation shape (shared with other teams? dedicated?), when is the IC paged, what is their authority in the three-way spend decision from exercise 02.

For each role, write a half-paragraph job description your rotation member will read on their first shift.

#### Section 3 — Escalation ladder (model-aware)

A table with eight rows, one per symptom class from chapter 03 section 5. Columns: `symptom` → `SRE-side triage steps` → `when to escalate` → `which peer SME to page` → `how to page them`.

The eight symptom classes to cover (reproduced from chapter 03 §5 as the parse contract — the table must contain one row per symptom):

```
1. classical-infra           (CPU / memory / network / scheduler / 5xx)
2. gpu-fault                 (XID, DCGM alarm)
3. serving-engine-misbehave  (vLLM OOM, SGLang scheduler thrash, TRT-LLM engine crash)
4. kernel-regression         (new engine release with slow FlashAttention path)
5. quality-regression        (hallucination spike, judge-pass-rate burn)
6. model-artifact-handoff    (promotion from training to serving)
7. provider-degradation      (vendor outage or degraded mode)
8. guardrail-misbehave       (PII filter / policy engine misclassification)
```

The peer-SME column draws from this controlled vocabulary (reproduced from chapter 03 §5 and the `mod-101` deferral map — pick exactly one per row):

```
- platform-sre              (cloud / scheduler / network infra)
- ai-infra-performance      (GPU faults, kernel regressions)
- ai-infra-inference-engineer (serving-engine knob fluency)
- ai-eval-engineer          (quality-signal methodology)
- ai-infra-ml-platform      (artifact promotion, registry handoff)
- ai-infra-security         (guardrails, PII, policy)
- vendor-support            (provider-side degradation)
```

Fill in the actual names / channels / rotations for your team — "Slack `#ai-infra-engine-oncall`" or "PagerDuty rotation `inference-engine-sme`," not "the inference-engine peer." If you do not know the real channel, use a placeholder `TBD-<peer>` and note it as a follow-up to resolve.

#### Section 4 — On-call-health metrics

A dashboard spec table with one row per metric from chapter 03 section 4. Required columns: `metric` → `source` → `target` → `review trigger` → `owner`.

The four metrics to cover (parse contract — one row per metric, exact slugs):

```
- page-volume        (pages/week, stratified business-hours vs overnight, and by alert source)
- actionable-rate    (fraction of pages that produced a meaningful response)
- sleep-disruption   (overnight pages per week per rotation member, with post-shift self-report)
- cognitive-load     (post-shift 1-5 exhaustion self-report)
```

The chapter's default targets and review triggers (reproduce or document your deviation):

| Metric | Default target | Default review trigger |
|---|---|---|
| `page-volume` | ≤ 2 interrupts per shift | Any shift with > 2 interrupts sustained over two weeks |
| `actionable-rate` | ≥ 70% actionable | < 50% actionable over a 4-week window → pager-tuning sprint |
| `sleep-disruption` | ≤ 2 overnight pages / week / primary | > 2 for two consecutive weeks → overnight-page audit |
| `cognitive-load` | Modal response 1–3 | Pattern of 4s and 5s over two consecutive shifts → reshape |

For each row, name the `source` (which system or survey emits the number — PagerDuty, Opsgenie, a Google Form) and the `owner` (usually the SRE team's engineering manager, not the on-call rotation itself).

#### Section 5 — Paging policy

Enumerate the five paging-policy rules from chapter 03 section 6, specialized to your SLOs from exercise 01. Reproduce the rule list verbatim as sub-headings so the policy is scannable at 03:00:

1. **Burn-rate alert thresholds** — the fast / medium / slow numbers, keyed to each SLO from exercise 01 (e.g., 14.4×/1h, 6×/6h, 1×/3d for an availability SLO; rederived per SLO per the [SRE Workbook, Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) table).
2. **Multi-window silencing** — the second-window requirement for each alert (fast-burn must *also* satisfy the medium-burn window before the page fires).
3. **Correlated-signal silencing** — the implementation of the exercise-02 reconciliation rule inside the paging layer. Name the correlation window (the chapter uses 2h; name yours), the suppression rule (which alert is silenced and which fires as the incident), and the IC role as the correlated-vs-independent arbiter.
4. **Provider-status ingestion** — which provider status pages are ingested, through what channel. Name each provider you consume (OpenAI, Anthropic, Google, AWS, Azure, or the specific subset for your workload) and the integration path (status-page webhook → paging stack, or scraped feed → correlation engine).
5. **Change-window routing** — how a declared deploy / model-promotion / prompt change window disables or re-routes the alerts on the burning surface for its duration. Name the declaration mechanism (a deploy tag, a PR label, a manual on-call command) and the default window length.

Each rule is one short paragraph; the primary on-call must be able to scan the whole policy in under three minutes.

### Artifact 2 — Hand-off checklist template

A six-section Markdown template matching chapter 03 section 3, ready to be copy-pasted by the outgoing on-call into a durable channel. The six sections (parse contract — exact order, exact slugs as the section anchors):

```
1. open-incidents
2. open-pages
3. known-bad-state
4. in-flight-changes
5. slo-budget-state
6. recent-post-incident-actions
```

Each section has a short prompt (one or two lines of italic guidance) naming the fields the outgoing on-call fills in. The suggested per-section fields are:

| Section | Suggested fields |
|---|---|
| `open-incidents` | Incident ID · current state · next action · next-action owner · expected resolution time |
| `open-pages` | Alert name · firing-since · resolved-but-not-root-caused? · notes |
| `known-bad-state` | Component · nature of the degradation · intentional vs awaiting fix · expected normalization time |
| `in-flight-changes` | Change ID · type (deploy / model promotion / prompt / provider route) · owner · window |
| `slo-budget-state` | Each SLO at < 50% budget remaining · current band from exercise 02 |
| `recent-post-incident-actions` | Fix landed · affected surface · window the fix could still produce confused behaviour |

The template is a fillable form — placeholders in `<angle brackets>`, inline guidance in short italics, no example data baked in. Target a 3-minute fill time; a template that takes 15 minutes to complete will be skipped.

## Starter guidance

- Resist the urge to design FTS before you need it. Case A is the honest answer for most Year-One AI SRE teams, and the pager-tuning work is more valuable than the rotation-splitting work at that stage.
- The escalation ladder rows are a translation of mod-101's deferral map into on-call language. If you have the mod-101 exercise 01 output, lift the peer owners directly.
- The on-call-health metrics are not vanity metrics — they drive the policy decisions on toil-reduction sprints (chapter 01 section 3). Wire them into the same SLO-review ritual as the error-budget policy.
- The hand-off checklist is the shortest artifact but the most-read one. A template that takes 15 minutes to fill in will be skipped; a template that takes 3 minutes will be adopted. Err toward brevity.

## Acceptance criteria

- Markdown file is self-contained and ≤ 2000 words total across both artifacts.
- Rotation shape named with justification against the "did you tune the pager before splitting?" test.
- Three roles each have a half-paragraph job description, and each cites the specific MTTA / fan-out / incident-declaration numbers from chapter 03 §2 (or documents the per-team deviation).
- Escalation ladder has one row per symptom from the eight-class list in Section 3, and each `which peer SME to page` entry draws from the seven-slug controlled vocabulary in the same section.
- On-call-health dashboard reproduces the four metrics (`page-volume`, `actionable-rate`, `sleep-disruption`, `cognitive-load`) with specific `source`, `target`, `review trigger`, and `owner` filled per row.
- Paging policy enumerates all five rules from chapter 03 §6 as named sub-headings, and the correlated-signal rule explicitly encodes the exercise-02 reconciliation rule (correlation window, suppression rule, IC-as-arbiter).
- Hand-off checklist template contains the six sections from Artifact 2 in the exact order and slugs given, with per-section placeholder fields — no "fill in later" placeholders in required sections.
- A new team member can read both artifacts in 15 minutes and understand how the rotation works.

## Stretch goals

- **Pager-tuning retrospective template.** Author the retrospective format your team will run when actionable-rate drops below 50%. Cite one historical example (real or hypothetical) of what the output of that retrospective looks like.
- **Compensation / sustainability section.** Add a short appendix on shift compensation, overnight-page caps, and PTO / shift-swap policy. This is the part of the rotation most likely to be omitted from the technical design and most likely to determine whether the rotation is sustainable at two years.
- **Cross-team hand-off.** If your team shares incidents with product teams (which it will), author the one-paragraph protocol for *product* ↔ *SRE* hand-off during a cross-team incident.
- **Simulation drill.** Walk two incidents from exercise 02's hypothetical-scenario appendix (if you did the stretch there) through the rotation design. Who pages whom, what does the hand-off look like at shift change if the incident is still open? Any gaps revealed should be closed by revising the design.
