# Dickerson's Hierarchy, Applied to AI Reliability

## Why this chapter exists

Classical SRE has a well-known picture — a pyramid, drawn by Mikey Dickerson on a whiteboard at Google and formalised in the SRE Book — that orders the reliability practices by how foundational each one is. You cannot do capacity planning until you have monitoring and incident response working, because you do not even know what you are planning for. You cannot do product development on top of a reliability platform you cannot observe in production.

The pyramid still holds for AI systems. The *content* of each layer changes — "monitoring" means something more than Prometheus scrape when the thing being served is a token stream from a 70B model — but the ordering is intact. This chapter walks the pyramid layer by layer, says what each layer means for AI workloads, and uses the ordering to justify the sequence of the remaining modules in this curriculum. If you were ever going to argue with the module order (*"why are we doing SLO engineering before chaos engineering?"*), this is the chapter that answers you.

## The pyramid

From the *SRE Book*, Part III — Practices. Bottom to top:

```
                          ┌──────────────┐
                          │     UX       │   ← 7. User experience
                          ├──────────────┤
                          │   Product    │   ← 6. Product development
                          ├──────────────┤
                          │   Capacity   │   ← 5. Capacity planning
                          ├──────────────┤
                          │Testing+Release│  ← 4. Testing and release procedures
                          ├──────────────┤
                          │ Post-mortem /│   ← 3. Post-mortem / root-cause culture
                          │ root-cause   │
                          ├──────────────┤
                          │   Incident   │   ← 2. Incident response
                          │   response   │
                          ├──────────────┤
                          │  Monitoring  │   ← 1. Monitoring (foundation)
                          └──────────────┘
```

The rule the pyramid encodes: **you cannot skip layers.** Investing in layer *n* while layer *n − 1* is broken wastes the investment. Doing chaos engineering on a platform without a working monitoring layer produces a lot of noise and very few lessons; doing product-dev work on a platform without an error-budget discipline produces a lot of releases and a lot of unplanned on-call pages.

The *AI* content of each layer follows.

## Layer 1 — Monitoring

**Classical content.** CPU, memory, disk, request-rate, latency, error-rate. Four golden signals (latency, traffic, errors, saturation). Structured logs, traces, metrics.

**AI content.** Everything above, plus:

- **Request-path telemetry at per-token granularity.** TTFT (time-to-first-token), TPOT (time-per-output-token), ITL (inter-token-latency), goodput (tokens-per-second net of failed streams). The OpenTelemetry `gen_ai.*` semantic conventions are the attribute surface to instrument against.
- **GPU telemetry.** DCGM metrics (SM activity, HBM pressure, ECC errors, thermal, XID codes), driver / NUMA / PCIe / NVLink topology at reading depth, Node Problem Detector signals, MIG / MPS partitioning state.
- **Quality signals as first-class metrics.** Hallucination-rate proxies, tool-call success rate, structured-output conformance, refusal rate — surfaced on the same dashboards as latency, not hidden in a separate analytics tool.
- **Drift signals.** Input-distribution drift, output-distribution drift, retrieval-freshness lag, judge-model agreement — wired in as SRE alerts, not as data-science curiosities.

The AI-specific layer-1 work is substantial enough that an entire module of this curriculum (`mod-104`) is dedicated to it, with the `gen_ai.*` conventions as backbone.

## Layer 2 — Incident response

**Classical content.** On-call rotations, paging policy, incident command (IC / OL / SME), bridge calls, runbooks, war-room etiquette, blameless forensics.

**AI content.** The vocabulary carries over verbatim. The *failure classes* do not.

- **Non-deterministic failure classes.** Silent output drift (the model is up, requests complete, outputs are subtly worse), hallucination spike, toxic blast, prompt-cache cliff (hit-rate collapses, TTFT doubles with no deploy), retrieval-freshness regression, agent-tool-call cascade, judge-model drift, provider-degraded mode. None of these look like a failing health check.
- **Detection discipline.** For the classes above, the classical "scrape a metric, alert on threshold" pattern often does not fire — the metric that would detect them has to be *constructed* (a drift statistic, a judge pass rate, a semantic-diff score), and the alert has to tolerate real base-rate noise.
- **AI-flavored runbooks.** The top-twenty incident classes each need a runbook with AI-specific triage steps, the specific dashboards that confirm the class, and the escalation / routing decisions (back to which peer).

This layer is covered in `mod-105` (failure-class catalog + detection) and `mod-108` (IR process + runbooks + post-mortem template).

## Layer 3 — Post-mortem / root-cause culture

**Classical content.** Blameless post-mortems, the five-whys, remediation-item follow-through, trend analysis across incidents.

**AI content.** Same discipline, additional root-cause categories:

- **Prompt-and-config as reliability surface.** A silent output degradation may be caused not by a code change but by a prompt-template update, a RAG index rebuild, a tool-definition tweak, or a model-provider's unannounced minor upgrade. The post-mortem has to trace to the *config* dimension, not just the code dimension.
- **Regression-fixture back-feed.** Every AI-specific incident with a quality dimension should result in an input added to a regression harness that would have caught it. Otherwise the same incident recurs when the model, prompt, or retrieval changes next month.
- **Shared post-mortems across peers.** Security-fail-open → availability loss, kernel regression → latency SLO breach, pipeline misconfiguration → silent drift. The classical post-mortem template needs an "owning peers" field so the action items route correctly.

Covered depthwise in `mod-108`.

## Layer 4 — Testing and release procedures

**Classical content.** Pre-production testing, canary deploys, feature flags, progressive delivery, blue-green, SLO-based auto-rollback.

**AI content.** Everything above, plus:

- **Canary with response-diffing.** A code-diff canary compares error rates; an AI canary has to compare *outputs* on a shared stimulus set. Exact-match is wrong (outputs are stochastic); semantic-equivalence tolerance with a configurable threshold is the pattern.
- **Shadow traffic for model changes.** The new model receives duplicated real traffic and its outputs are scored, without any user ever seeing them.
- **Joint eval + SLO gate.** A release that passes classical SLOs but fails the eval gate should not promote; a release that passes eval but is slower than the SLO should not promote either. Both signals wire into one automated gate.
- **DORA four-keys for model releases.** Deploy frequency, lead time, change-failure rate, MTTR — applied to *model* releases (not just code releases), which is a different cadence and a different change-surface.

Covered in `mod-111`, with the data / config / prompt pipelines treated as a release surface in `mod-112`.

## Layer 5 — Capacity planning

**Classical content.** Headroom, utilisation targets, forecasting, procurement, queueing theory at a working level.

**AI content.** Same discipline, qualitatively different parameters:

- **Prefill-vs-decode queueing.** AI inference has two distinct phases with different compute / memory shapes; headroom has to be computed per phase, not per request.
- **Cross-generation fleet mix.** H100 / H200 / B200 / GB200 / GB300 / MI300 inventory with different throughput and cost; the planning problem is a fleet-mix optimisation, not a single-SKU question.
- **Model-provider quotas as capacity.** For any flow that calls OpenAI / Anthropic / Google / Bedrock / Azure, the provider's RPM / TPM / per-model quota is a capacity surface you have to plan against — and a dependency you cannot scale on demand.
- **Autoscaling on inference-specific metrics.** HPA on queue depth or TTFT-P99, not on CPU. Karpenter GPU bin-packing, not generic node-count scaling.

Covered in `mod-106`, with vendor-side capacity surfaced in `mod-107`.

## Layer 6 — Product development

**Classical content.** The reliability team contributes to the roadmap — platform features that reduce toil, improve visibility, or lift the error budget. SRE has a seat at design review (non-abstract design review, in SRE Workbook language).

**AI content.** The reliability team contributes to roadmap decisions that are *fundamentally AI-shaped*:

- Which models the platform supports (and which it does not), driven by operational cost and risk.
- Which inference engines the platform runs, with what knob envelopes — the operator-side view, not the author-side view.
- Which vendor providers are approved, with what fallback contract.
- Which safety / guardrail controls are inside the reliability budget vs. negotiated case-by-case.

This is the *leadership* dimension covered in `mod-114`.

## Layer 7 — User experience

**Classical content.** The reliability contract exists to serve the user; a platform with nominally-passing SLOs but a bad user experience has the wrong SLOs.

**AI content.** User experience of AI systems has reliability dimensions that do not exist in classical UX:

- **Perceived latency with streaming.** TTFT (time-to-first-token) often matters more than total latency — a slow-to-start but fast-to-stream response feels responsive; a fast-to-start response that pauses mid-stream feels broken.
- **Perceived quality degradation.** Users notice quality cliffs (the model got "worse today") even when metrics don't; the SLO stack has to include a quality dimension that tracks what users perceive.
- **Graceful degradation under provider-degraded mode.** When your upstream provider is in degraded mode, the user-visible behaviour (slow but correct, fast but reduced, fast but refused) is a reliability *decision*, not a product afterthought.

The UX layer informs the SLO design in `mod-103` — what goes into the error-budget stack is ultimately what users would complain about.

## The module sequence, justified

Reading the module list against the pyramid:

| Module | Layer | Why in this position |
|---|---|---|
| `mod-101` (this one) | — | Role scope. Done before any technical layer. |
| `mod-102` SRE foundations applied to AI | — | Vocabulary + engine literacy. Lets you *name* the surfaces the pyramid layers will instrument. |
| `mod-103` SLO engineering | 1 → 4 | SLO authoring is the artifact that monitoring (1), incident response (2), and release gates (4) all reference. |
| `mod-104` Observability | 1 | Monitoring layer, deepwise. |
| `mod-105` Non-deterministic failure diagnosis | 2 | Incident response, failure-class catalog. |
| `mod-106` Capacity planning | 5 | Only sensible after layers 1–4 exist, so you have signals to plan against. |
| `mod-107` Vendor risk | 5 | Capacity has a vendor-provider dimension; treated in its own module because the dependency shape is different. |
| `mod-108` Incident response | 2 → 3 | Full IR process + post-mortem culture, after the failure-class content in mod-105. |
| `mod-109` Chaos engineering | 1 → 4 | Validates layers 1–4; pointless before them. |
| `mod-110` Reliability × security / safety / compliance | 1 → 4 | The reliability knock-on of adjacent controls; depends on existing layers. |
| `mod-111` Release engineering | 4 | Testing / release layer, deepwise. |
| `mod-112` Data / config pipelines | 4 | Same layer, different surface (prompt / retrieval / eval-set). |
| `mod-113` DR / BCP / multi-region | 2 → 4 | Reaches across incident response, release, and capacity; depends on all three. |
| `mod-114` Cross-functional leadership | 6 | Product-dev layer; the program-ritual module. |

The three capstone projects in the paired repo span layers 1–5 (`project-101`), 1–3 + chaos (`project-102`), and 4–6 (`project-103`).

## Summary

- Dickerson's hierarchy — monitoring, incident response, post-mortem, testing+release, capacity, product-dev, UX — carries from classical SRE to AI SRE without reordering.
- The *content* of every layer picks up AI-specific surfaces: per-token latency SLIs, GPU telemetry, non-deterministic failure classes, canary-with-response-diffing, prefill-vs-decode capacity, model-provider quotas, quality-in-the-UX-layer.
- The curriculum's module sequence tracks the pyramid: monitoring-and-SLOs first, then incident classes, then capacity, then testing / release / chaos / DR, then leadership. You can read a given module's prerequisites straight off the pyramid.

## References

- *SRE Book*, Part III — Practices. The Dickerson pyramid is the backbone of the Practices chapters.
- *SRE Workbook*, especially Chapter 5 (Alerting on SLOs) and the on-call chapters.
- OpenTelemetry Semantic Conventions for Generative AI (`gen_ai.*`).
- NIST AI RMF Generative AI Profile (AI 600-1) — the vocabulary for the quality / drift / failure-class layer.
- *The Tail at Scale*, Dean & Barroso — underwrites the per-token-latency framing.
