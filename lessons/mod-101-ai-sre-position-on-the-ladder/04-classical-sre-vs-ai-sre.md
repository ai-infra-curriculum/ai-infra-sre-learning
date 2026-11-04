# Classical SRE vs. AI SRE — What Carries, What Has To Be Re-Invented

## Why this chapter exists

A common career path into AI SRE is laterally from classical SRE — three to ten years of SRE on web services, databases, or data pipelines, then a pivot to a serving platform that runs an LLM. That transfer works, because most of classical SRE is *methodology*, and the methodology is intact. But the pivot fails when engineers assume the methodology maps verbatim, because a small but critical set of practices has to be re-authored for AI workloads, and the re-authored forms look superficially similar enough to the classical forms that the gap is easy to miss until an incident exposes it.

This chapter does the side-by-side. The left column — *what carries over verbatim* — is the reason you can be hired into this role with classical SRE credentials. The right column — *what has to be re-invented* — is the reason this role is a specialty and not an existing SRE team wearing a new jersey.

The reference for the left column is the [Google SRE Book](https://sre.google/sre-book/table-of-contents/) and the [SRE Workbook](https://sre.google/workbook/table-of-contents/). The right column is sourced from the remaining modules in this curriculum.

## Left column — what carries verbatim

These practices do not change. If you have run them before, run them the same way here.

### 1. Error budgets

The error-budget abstraction — *the amount of unreliability we can tolerate before we slow down feature work* — is identical. SLO defines "good"; error budget is the complement; policy says what happens when the budget is burned. The AI surface uses the same abstraction, just with AI-shaped SLIs populating it (per-token latency, quality rates, cost per 1M tokens). Covered in `mod-103`.

### 2. Blameless post-mortem culture

Blameless, in the SRE Book sense — focus on the system of work that produced the incident, not the individuals in that system. Human error is a symptom of system design. The ritual (timeline, contributing factors, five-whys, remediation items with named owners and dates, publish broadly) carries over. Covered in `mod-108`.

### 3. SLO methodology

The pipeline from *what users care about* → *SLI definition* → *SLO target* → *error-budget policy* → *reporting* is intact. The craft of choosing which three or four SLIs actually matter for a service, the discipline of not over-specifying, the review cadence, all transfer. The AI-specific work is which SLIs — covered in `mod-103`.

### 4. On-call rotation design

Rotation shape (follow-the-sun, primary / secondary, handoff ritual), toil-budget enforcement, paging-policy discipline (page on burn-rate, not on single-sample threshold excursion), compensation / sustainability constraints, learning-oriented on-call shadowing — identical. Covered in `mod-108` and `mod-114`.

### 5. Four golden signals

Latency, traffic, errors, saturation. Still the right four. The AI content is that each of them decomposes into AI-specific sub-signals (latency becomes TTFT / TPOT / ITL / E2E; errors includes quality failures; saturation includes KV-cache pressure) — but the top-level frame is unchanged. Covered in `mod-103` and `mod-104`.

### 6. Non-abstract large system design

The Workbook's "non-abstract" discipline — do the back-of-envelope sizing, write the design doc, pre-compute capacity, name the failure modes before launch — applies to any AI launch. Covered in `mod-106` and `mod-111`.

### 7. Toil elimination and automation

Toil is still toil: manual, repetitive, tactical, interrupt-driven, scaling-linearly-with-service-growth, lacking enduring value. The AI surface creates its own toil (manual canary analysis, manual quota requests, manual drift-chart review) that has to be automated to the same standard. Covered in `mod-114`.

### 8. Launch coordination

The launch-checklist pattern, launch-review cadence, "production-ready" definition — all carry. The AI-specific items added to the checklist are covered in `mod-111` and `mod-112`.

### 9. Chaos / disaster readiness as discipline

The Principles of Chaos Engineering and the discipline of regular failure drills carry over as a shape. The *experiments* that populate the shape are AI-specific — covered in `mod-109` and `mod-113`.

## Right column — what has to be re-invented

These are the practices where classical SRE's exact artifact does not survive the move to AI workloads. The methodology survives; the artifact does not.

### 1. SLOs at per-token granularity

**Classical artifact.** A request-latency SLO: "99.9% of requests complete in < 300 ms."

**Why it breaks for AI.** Model responses are *streams*, not single-shot replies. A user experiences three distinct latency surfaces in one logical request:

- TTFT — time-to-first-token — "did the model start responding?"
- TPOT — time-per-output-token — "is it streaming at a reasonable pace?"
- ITL — inter-token-latency — "is the stream smooth or is it stalling?"

A single E2E latency metric smears all three together and loses the ability to diagnose a regression. A fast-TTFT, slow-TPOT response feels broken differently from a slow-TTFT, fast-TPOT response.

**Re-invented artifact.** A multi-SLI latency stack: separate SLOs for TTFT, TPOT, ITL, and E2E goodput, each with its own threshold and burn-rate alert. Covered in `mod-103`.

### 2. Non-deterministic failure classes

**Classical artifact.** The service is up or down. Requests succeed or fail. Latency is in-SLO or out. Each signal is binary or scalar and crosses a threshold.

**Why it breaks for AI.** A model-backed service can be nominally up, returning 200s, meeting latency SLOs, and still be badly broken because *the content of the responses has drifted*. Silent output drift, hallucination spike, toxic blast, prompt-cache cliff, retrieval-freshness regression, agent-tool-call cascade, judge-model drift, and provider-degraded mode are each failure classes with no classical analogue. The classical monitoring layer never fires.

**Re-invented artifact.** A failure-class catalog with per-class detection signals (which are often *constructed* — a drift statistic, a judge pass rate, a semantic-diff score — not scraped), per-class runbooks, per-class escalation routing. Covered in `mod-105` and `mod-108`.

### 3. Model-provider dependency risk

**Classical artifact.** Upstream dependency = a database, a cache, a queue. You run them, you monitor them, you have a fallback or a read-replica.

**Why it breaks for AI.** For any production flow that calls OpenAI / Anthropic / Google / Bedrock / Azure OpenAI, the upstream is a *vendor* — not something you own, not something you can monitor from inside, not something you can scale on demand. It can degrade silently (returning answers that are subtly worse), hit its own capacity ceiling (your RPM / TPM / per-model quota), or go down for reasons outside the SLA's explicit coverage. The SLA the vendor publishes and the SLO your users feel are related but not the same number.

**Re-invented artifact.** A vendor-dependency surface map, a provider-fallback contract (LiteLLM / Portkey-shape routing), status-page ingestion into the incident pipeline, SLA-vs-SLO reconciliation, vendor-eval discipline. Covered in `mod-107`.

### 4. Prompt-and-config as reliability surface

**Classical artifact.** The reliability surface is *code* and *config*. Change management covers them; a rollback is a redeploy.

**Why it breaks for AI.** For an AI service, the production behaviour is determined by:

- Model artifact
- Serving-engine knobs
- Prompt template(s)
- Tool / function definitions
- Retrieval index snapshot + retrieval parameters
- Eval set + judge-model version (because the release gate depends on them)
- Guardrail-sidecar policy

Any of these can change behaviour; most of them are *not* in the code-review pipeline the classical SRE change-management process was designed around. A silent degradation can trace to a prompt-template PR merged by a product engineer, or a RAG index rebuild, or an unannounced minor update from the model provider.

**Re-invented artifact.** Expanded change-management: prompt-template versioning with rollback, retrieval-index-version pinning, eval-set lifecycle treated as a release surface, provider-model-version pinning in the fallback contract. Covered in `mod-112`.

### 5. Canary with response-diffing

**Classical artifact.** Canary deploy: 5% traffic to new, compare error rates, promote if error rate ≤ control.

**Why it breaks for AI.** The response content is stochastic. Two identical inputs produce slightly different outputs even against the *same* model build. Exact-match diffing is useless; a regression in model *quality* will not show in the error-rate signal (both canary and control return 200s); a dangerous regression can hide inside a passing classical canary.

**Re-invented artifact.** Canary with semantic-equivalence tolerance — send a stimulus set through both candidate and control, score the pairs with a judge model or a reference comparator, promote if the semantic-equivalence score is within tolerance *and* classical SLOs hold. Shadow traffic as a complement. Covered in `mod-111`.

### 6. Per-token cost as a first-class SLI

**Classical artifact.** Cost is a budget concern, tracked separately from reliability.

**Why it breaks for AI.** Token-level pricing means the cost per successful request varies with prompt length, generation length, model choice, cache-hit rate, and vendor routing decisions — all moving continuously, all under your control. A platform meeting latency and quality SLOs while silently burning the cost envelope is failing in a way the classical reliability picture does not name. Finance will notice a month later; by then you have shipped the regression.

**Re-invented artifact.** Cost-per-1M-tokens and cost-per-successful-request as first-class SLIs, with burn-rate alerts, in the same stack as latency and quality. Covered in `mod-103`.

### 7. Quality SLOs

**Classical artifact.** Correctness is binary — a request either produced the right answer or an error. You measure the error.

**Why it breaks for AI.** AI outputs are rarely binary-correct. "Correct" is a spectrum — hallucinated, partially hallucinated, correct but unhelpful, correct and useful, over-refusing, under-refusing. Users complain about quality long before classical SLOs breach.

**Re-invented artifact.** Quality SLOs built on constructed signals — hallucination-rate proxy, tool-call success rate, structured-output conformance, toxic-output rate, refusal rate, each with a judge or validator pipeline producing the signal and an SLO threshold with error budget. The eval peer owns the methodology of the signal; you own the wiring to the SLO stack. Covered in `mod-103` and the seam with `mod-112`.

## The cross-reference table

Compressed form, for the exercise:

| Classical SRE practice | AI SRE form | Changes |
|---|---|---|
| Error budget | Error budget | None — budget abstraction identical |
| Blameless post-mortem | Blameless post-mortem, with prompt/config root-cause category and cross-peer owner routing | Category additions only |
| SLO methodology | SLO methodology, SLIs are AI-specific | Methodology identical; SLIs re-authored |
| On-call rotation | On-call rotation | None (sustainability / toil / shadow pattern identical) |
| Four golden signals | Four golden signals (latency / traffic / errors / saturation), decomposed | Each signal has AI-specific sub-signals |
| Request-latency SLO | **TTFT / TPOT / ITL / E2E / goodput stack** | Single SLO → multi-SLI stack |
| Binary failure modes | **Failure-class catalog with constructed signals** | New class of failure not seen in classical |
| Upstream dependency | **Model-provider dependency with vendor-SLA / fallback / status ingestion** | New type of dependency; cannot scale on demand |
| Change management | **Expanded change management: prompt, retrieval, eval, guardrail, provider version** | New reliability surfaces not under code review |
| Canary with error-rate diff | **Canary with semantic-equivalence diff + shadow traffic** | Stochastic output invalidates exact-diff |
| Cost as budget concern | **Cost SLI in the error-budget stack** | Token-level pricing promotes cost to reliability signal |
| Binary correctness | **Quality SLOs with constructed signals** | Correctness becomes continuous |
| Chaos experiments | Chaos experiments — **AI-specific** experiment set | Shape identical; experiment set re-authored |
| DR plan | DR plan — **AI-specific**, including provider-fallback | NIST SP 800-34 base intact; AI-specific additions |

## The lateral-hire pitch

Everything in the left column is why the classical SRE → AI SRE pivot works. Everything in the right column is why it is a specialty.

If you came from classical SRE: do not feel you have to throw out your reflexes. The error-budget discipline, the blameless post-mortem, the toil accounting, the launch-review ritual — these are exactly what the AI stack needs. What you have to add is in the right column, and the rest of this curriculum walks you through each of those.

If you came from the AI / ML / inference side: do not feel you have to re-derive the SRE discipline from first principles. Read the SRE Book and SRE Workbook once end-to-end; you will find the left column is already a coherent toolkit. What you have to add is the SRE-practice side applied to the workloads you already know.

## Summary

- **Left column** (carries verbatim): error budgets, blameless post-mortems, SLO methodology, on-call design, four golden signals, non-abstract design, toil discipline, launch coordination, chaos as a shape.
- **Right column** (re-invented): per-token-granularity latency SLOs, non-deterministic failure classes, model-provider dependency risk, prompt-and-config as reliability surface, canary with response-diffing, cost as first-class SLI, quality SLOs.
- The methodology is intact; the artifacts change. Exercise 02 asks you to produce the full side-by-side for a specific team context.
