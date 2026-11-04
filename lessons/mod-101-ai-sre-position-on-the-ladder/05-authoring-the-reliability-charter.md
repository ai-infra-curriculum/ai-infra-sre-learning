# Authoring the Reliability Charter

## Why this chapter exists

A reliability charter is a short, written artifact that answers one question: *when a reliability-adjacent concern arrives, who owns it?* Every engineering organisation already has an informal answer to this question, carried in the heads of the senior people on the team. The charter makes it explicit.

Three reasons you need this artifact in writing:

1. **Onboarding.** New engineers (on your team and on peer teams) cannot read senior engineers' minds. A charter they can read in ten minutes saves a week of pattern-matching.
2. **Scaling.** Informal routing works for a team of five. At twenty, with shift-left on-call and multiple product teams filing tickets, informal routing produces a long tail of concerns bouncing between queues.
3. **Blast radius.** When a serious incident spans peers, the charter is the prior document everyone refers to in the post-mortem to decide what should have happened. Without it, every cross-peer incident re-litigates ownership in the room — which is the wrong room to have that conversation in.

This chapter gives you the structure of a charter, worked examples of the three hardest sections, and the review ritual that keeps it alive. Exercise 03 has you produce one for a concrete team.

## The sections of a charter

The charter is one Markdown page (target: 500–1500 words). Any longer and nobody re-reads it; any shorter and it will not cover the real cases. The seven sections:

```
1. One-sentence mission
2. What this platform is (short noun list)
3. SLOs the platform publishes
4. What this team owns     ← the "yes" column
5. What this team does not own ← the "no" column, with the routing target for each
6. The routing rule for ambiguous cases
7. Review cadence and version history
```

A good charter is testable: pick ten plausible incoming concerns, apply the charter, and it routes each one to a clear owner without discussion. If three of the ten are ambiguous, the charter is not yet done.

## Section 1 — One-sentence mission

Example, adapted from chapter 01:

> This team owns the operational reliability contract of the \[*platform name*\] — the production AI serving stack supporting \[*customer surface*\]. We own the SLOs, the incident response, the capacity plan, and the vendor-dependency contract; we do not own the model, the product, or the content of model responses.

Rules of thumb:

- One sentence. If it takes two, the mission is probably not actually a mission yet.
- Names the platform explicitly — a charter for "the whole company's AI reliability" is not actionable.
- Includes one clear *does not own* clause, so the sentence is informative on its own.

## Section 2 — What this platform is

A short enumeration of the surfaces this reliability contract covers. Not architecture — *surface*. Eg:

- The internal LLM gateway serving teams X, Y, and Z.
- The managed inference fleet running `model-a` and `model-b` on GPUs in region A and region B.
- The retrieval pipeline that serves RAG traffic for team Y.
- The agent-orchestration layer for the \[named\] feature.

If a surface is not on the list, it is explicitly out of scope. The charter should be re-opened and re-published, not tacitly stretched, when a new surface is added.

## Section 3 — SLOs the platform publishes

A one-line summary of each SLO the platform commits to, with a pointer to the authoritative definition elsewhere (dashboard, SLO repo, or `mod-103`-shaped SLO document).

Do not restate the numbers in the charter — numbers change; charters should change infrequently. The charter commits that *there is* an SLO for this dimension; the SLO repo commits to the current target.

Example:

- TTFT P95 — see `slo/ttft`.
- TPOT P95 — see `slo/tpot`.
- Availability under streaming (good-stream-rate) — see `slo/availability-streaming`.
- Tool-call success rate — see `slo/tool-success`.
- Cost per 1M completion tokens — see `slo/cost-per-mtok`.

## Section 4 — What this team owns (the "yes" column)

A flat list, in order of operational weight, of the owned responsibilities. This is the content of chapter 01 restated in the voice of *your* platform. Each item should be answerable with a specific artifact or page.

Example items:

- **SLO authoring and lifecycle.** All SLOs on the above platform are authored, reviewed, and retired by this team. See `slo/README.md`.
- **Incident response during on-call.** 24×7, follow-the-sun, primary / secondary. Rotation page: `oncall/rotation`.
- **Runbooks.** Top-N runbooks live in `runbooks/`.
- **Capacity plan.** Monthly cycle; see `capacity/plan.md`.
- **Model-provider dependency contract.** LiteLLM-shape fallback, status ingestion, provider-health runbooks. See `vendors/README.md`.
- **Release gate for model updates.** Canary + shadow + SLO-based auto-rollback. See `release-gate/`.
- **Chaos program.** Monthly experiment cadence; see `chaos/calendar.md`.
- **DR / BCP.** NIST SP 800-34-shaped plan; annual table-top; biennial full failover. See `dr/plan.md`.

Rule of thumb: eight to twelve items is right. Fewer and you are under-specifying; more and you are absorbing work that should be at a peer.

## Section 5 — What this team does not own, with routing target

This is the section that does the most work in practice. Structure as a two-column table: *concern → routing target*, with just enough qualifier that the routing is unambiguous.

Worked example:

| Concern | Routes to | Qualifier |
|---|---|---|
| Serving-engine source-code bug | `ai-infra-inference-engineer` team | Engine repo issue, with repro conditions and the SLO it violates |
| Kernel-level performance regression | `ai-infra-performance` team | Jointly-filed with GPU telemetry attached |
| Prompt-injection classifier policy | `ai-infra-security` team | We own only the latency / cache knock-on |
| Model-weight supply-chain / signing | `ai-infra-security` team | ML-BOM + signing-verification issues |
| Model artifact handoff bug | `ai-infra-ml-platform` team | Registry format, promotion manifest, resource profile |
| Eval-set drift / rot | `ai-eval-engineer` team | We own release-gate wiring to the signal, not the signal itself |
| Judge-model agreement methodology | `model-evaluation-engineer` team | Statistical methodology is theirs |
| Pipeline authoring | `ai-infra-mlops` team | We author the *gates* in the pipeline, not the pipeline |
| Prompt engineering / product design | Product / application team | Routed through consumer-track channels |
| Business-cost model | FinOps | We own the cost SLI and the levers; they own the finance model |
| Legal / data-residency statute interpretation | Legal / counsel | We produce residency-respecting telemetry |
| Security-incident evidence of record | SecOps / DFIR | We contribute AI-specific telemetry only |

A concern that appears in Section 4's "yes" list cannot also appear here; if the same concern lands in both, the charter is contradicting itself. Review for that conflict at every revision.

## Section 6 — The routing rule for ambiguous cases

Three-question test from chapter 02, restated:

1. Is this operational? If no → design / training / product owner.
2. Does the fix live inside a peer craft? If yes → that peer, with a reproducible signal.
3. Else — it is ours.

Followed by two or three worked examples of the hardest recent cases your team has routed. Over time this grows into a short case-law section — the specific precedents that resolved the previously-ambiguous cases. That history is what makes the charter *useful* rather than aspirational.

Example case-law entry:

> **Case: slow TTFT only at certain prompt lengths (2026-07-11).** Initially triaged as a kernel regression and routed to `ai-infra-performance`. Their kernel isolation showed no regression. Returned to AI SRE and traced to a batch-scheduler config in the engine: prefill chunk size had been lowered one release prior. Root cause was ours; fix was ours (engine knob). Charter update: engine-knob regressions are owned by AI SRE, not by `ai-infra-inference-engineer`, unless the knob exposes a bug in the engine code itself.

<!-- The above case is illustrative, not a real incident. Replace with your team's actual precedents when authoring. -->

## Section 7 — Review cadence and version history

Charter should be revisited:

- **Quarterly**, as a scheduled review.
- **After every cross-peer incident**, as an action item in the post-mortem if the incident surfaced an ambiguous ownership case.
- **On platform scope change**, when a new surface is added or an old one is retired.

Version history at the bottom of the file; each revision has a date, author, and a one-line change note.

## Common failure modes

Charters fail predictably. Four to watch for:

1. **Aspirational scope.** Charter claims to own things the team does not actually staff. The queue backs up, trust decays, nobody re-reads the charter. Fix: be honest about the scope — a smaller charter that is enforced is worth more than a larger charter that is not.
2. **Ambiguous "does not own".** Section 5 says *"not our problem"* without saying *"routes to"*. Concerns bounce; nothing moves. Fix: every "does not own" line names the routing target.
3. **No case-law.** Section 6 states the three-question rule but never accumulates examples. The same ambiguous cases recur and get re-litigated. Fix: append to section 6 after every ambiguous case is resolved, with a one-paragraph precedent.
4. **Never revised.** Charter is authored once and sits for two years. The platform evolves; the charter does not. The gap becomes dangerous. Fix: quarterly review is on the calendar, not a best-effort intention.

## Related artifacts

The charter is the top-level document; a few other artifacts fit under it and should be linked from Section 4:

- **SLO definitions.** Authored in `mod-103` form; live in the SLO repo; the charter points at them.
- **Runbook index.** The top-N runbooks live in `runbooks/`; one file per class, authored per `mod-108`.
- **On-call policy.** Rotation shape, paging policy, compensation; one file in `oncall/`; owned by this team per `mod-114`.
- **Vendor-dependency map.** Provider-fallback contract and vendor-eval records per `mod-107`.
- **Release-gate definition.** The eval + SLO gate per `mod-111`.
- **Chaos calendar.** The experiment cadence per `mod-109`.
- **DR plan.** NIST SP 800-34-shaped per `mod-113`.

The charter is the entry point to all of them. Everything else is downstream.

## Summary

- A reliability charter is a one-page, seven-section artifact that makes ownership explicit, so cross-peer routing happens by document and not by debate.
- The two sections that do the most work are the *yes* list (Section 4) and the *no* list with routing targets (Section 5).
- A charter is testable: ten plausible incoming concerns should each route to one clear owner.
- Revise quarterly, after cross-peer incidents, and on platform scope change. Append case-law precedents as the ambiguous cases get resolved.
- Exercise 03 has you produce a charter for a specific team context, using this chapter as the template.
