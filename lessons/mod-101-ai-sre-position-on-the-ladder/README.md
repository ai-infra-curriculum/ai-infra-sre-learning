# mod-101 — The AI SRE Position on the Infra Ladder

> First module of the AI Reliability Engineer (AI SRE) track. Role scope, peer boundaries, Dickerson-hierarchy placement, classical-SRE comparison, reliability-charter authoring, and the 2024–2026 hiring-market context — the frame everything downstream hangs on.

**Estimated effort:** 10 hours
**Position in curriculum:** before `mod-102` (SRE foundations applied to AI).

## Learning objectives

- Diagram the role's exact scope on the level ladder — what this role owns vs. `ai-infra-inference-engineer` (peer, level 35, serving-system authoring), `ai-infra-performance` (peer, level 35, kernel-authoring craft), `ai-infra-security` (peer, level 35, adversarial defence), `ai-infra-ml-platform` (peer, level 30, training-serving handoff), `ai-eval-engineer` (peer, level 30, application eval release-gate), `ai-infra-mlops` (peer, level 25, CI/CD), and the consumer tracks in the AI Engineering family (`llm-application-developer`, `rag-engineer`, `applied-ai-engineer` at level 25).
- Place AI SRE on Dickerson's hierarchy — monitoring at the base, then incident response, post-mortem culture, testing + release, capacity, product dev, UX at the top — and use this ordering to justify the module sequence of this curriculum.
- Position this role against classical SRE (as taught in the *Google SRE Book*) — what carries over verbatim (error budgets, blameless post-mortems, SLO methodology, on-call rotation design) and what has to be re-invented for AI workloads (SLOs at per-token granularity, non-deterministic failure classes, model-provider dependency risk, prompt-and-config as reliability surface).
- Author a "what this platform does and does not do" reliability charter — the artifact that routes an incoming reliability concern to the right owner.
- Position the role against the 2024–2026 emergence of "AI SRE" / "LLM SRE" / "AI Production Engineer" as a hiring title — what the market signal says the role covers today and where it is still fragmenting out of MLOps / production-engineering generalist titles.

## Chapters

Read in order. Each chapter is self-contained, 15–25 minutes.

1. [What AI SRE owns — the reliability contract](01-what-ai-sre-owns-and-the-reliability-contract.md) — the one-sentence charter, the five owned surfaces, the four things you do not own, and why the role sits at level 35.
2. [The peer deferral map — who you hand it to](02-the-peer-deferral-map.md) — the horizontal routing table against each peer (inference-engineer, performance, security, ml-platform, eval, mlops, consumers) and the three-question routing rule.
3. [Dickerson's hierarchy applied to AI reliability](03-dickerson-hierarchy-applied.md) — each pyramid layer walked for AI content, and the justification for the module sequence of this curriculum.
4. [Classical SRE vs. AI SRE — side by side](04-classical-sre-vs-ai-sre.md) — the two-column comparison, what carries verbatim, what has to be re-authored, and the lateral-hire framing.
5. [Authoring the reliability charter](05-authoring-the-reliability-charter.md) — the seven-section structure of the capstone artifact, with worked examples and common failure modes.
6. [The hiring market, 2024–2026](06-hiring-market-emergence-2024-2026.md) — the AI SRE / LLM SRE / AI Production Engineer title variants, four fragmentation patterns, and how to read (or write) a posting.

## Exercises

Each exercise applies the chapter content to a concrete team context you pick in exercise 01 and carry through. Target: 2 hours per exercise.

- [Exercise 01 — Role scope and deferral contract](exercises/exercise-01-role-scope-and-deferral-contract.md) — produce the Section-5-of-the-charter routing table; stress with the ten-concern test.
- [Exercise 02 — Classical SRE vs AI SRE side by side](exercises/exercise-02-classical-sre-vs-ai-sre-side-by-side.md) — produce the team-specific version of the chapter-04 comparison + the lateral-transfer checklist.
- [Exercise 03 — Reliability charter authoring](exercises/exercise-03-reliability-charter-authoring-for-an-ai-team.md) — the capstone artifact: the full seven-section charter for your team.
- [Exercise 04 — Dickerson's hierarchy applied to AI workloads](exercises/exercise-04-dickerson-hierarchy-applied-to-ai-workloads.md) — per-layer audit + two-quarter investment plan for a specific platform.

## Resources

External references cited across the chapters and exercises are collected in [`resources.md`](resources.md).

## Structure

```
mod-101-ai-sre-position-on-the-ladder/
├── README.md                                 ← this file
├── 01-what-ai-sre-owns-and-the-reliability-contract.md
├── 02-the-peer-deferral-map.md
├── 03-dickerson-hierarchy-applied.md
├── 04-classical-sre-vs-ai-sre.md
├── 05-authoring-the-reliability-charter.md
├── 06-hiring-market-emergence-2024-2026.md
├── exercises/
│   ├── exercise-01-role-scope-and-deferral-contract.md
│   ├── exercise-02-classical-sre-vs-ai-sre-side-by-side.md
│   ├── exercise-03-reliability-charter-authoring-for-an-ai-team.md
│   └── exercise-04-dickerson-hierarchy-applied-to-ai-workloads.md
├── labs/                                     ← reserved; no hands-on lab in this module
├── quizzes/                                  ← reserved; no quiz in this module
└── resources.md
```

## What this module does *not* cover

- **SLO numbers and SLI construction.** This module names the practice; `mod-103` authors the SLOs.
- **Observability instrumentation.** `mod-104`.
- **Any AI-specific failure class in diagnostic depth.** `mod-105`.
- **Vendor / provider risk mapping.** `mod-107`.
- **Release engineering.** `mod-111`.
- **DR / BCP.** `mod-113`.

This module is the *frame*. The rest of the curriculum fills it in.
