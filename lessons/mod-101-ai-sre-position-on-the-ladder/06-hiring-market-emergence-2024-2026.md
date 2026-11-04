# The Hiring Market, 2024–2026 — "AI SRE", "LLM SRE", "AI Production Engineer"

## Why this chapter exists

Role definitions are not just internal conventions; they are also hiring-market signals. If the industry's hiring managers converge on a role name, a scope, and a level band, that convergence is evidence that the role is real — distinct from, not a rebrand of, the roles next to it. If the industry does not converge, that is also evidence: the scope is still fragmenting out of nearby roles, and candidates applying for one role title will find themselves interviewing for several very different jobs under the same label.

This chapter is a short field guide to the 2024–2026 emergence of the AI-reliability specialty in the hiring market. It is deliberately conservative: we only claim things we can source to visible public artifacts (named company titles, public job-posting conventions, published engineering blogs), and we flag with `<!-- needs-research -->` anywhere the evidence would require live postings this bootstrap did not sample.

The practical takeaway is in the last section: as a candidate or a hiring manager, read every "AI SRE" posting against the five-surface scope in chapter 01 to see which of the five the role actually covers — because at this writing, the posting title alone does not tell you.

## The three title variants

Across the market, three titles converge on roughly the scope of this curriculum. They are not identical in connotation.

- **"AI SRE"** — the most common title at companies with a strong existing SRE culture. Emphasises SLO / incident-response / post-mortem discipline, applied to AI workloads. Used by companies whose internal SRE identity is established enough that *"AI"* is the modifier.
- **"LLM SRE"** — a narrower variant. Emphasises the LLM-serving surface specifically (less emphasis on classical ML training reliability). Seen at companies whose AI footprint is dominated by LLM serving rather than classical ML / recommenders.
- **"AI Production Engineer" / "ML Production Engineer"** — the Meta / Facebook-lineage phrasing, where "Production Engineer" has historically meant the same seat SRE occupies at Google-lineage orgs. Emphasises the production operations of AI systems; the scope is broadly the same.

Fourth title you will also see, with more variance:

- **"AI Infrastructure Engineer"** — sometimes used as an umbrella including the AI SRE scope, sometimes used for the inference-engineer scope, sometimes used for the platform / MLOps scope. This is the most-overloaded of the titles and the one you should read the posting *content* for, not the title.

## What the market signal says the role covers

Without a live-posting sample this cycle, the themes below are sourced from authoritative references rather than posting frequency — and are the themes that this curriculum's module plan takes as its spec. The 2026-11 bootstrap could not run live `WebSearch` / `WebFetch` (permission-gated); the autonomous research loop is expected to backfill concrete frequency data on its next pass. Treat the ordering below as *assertion of what the authoritative sources describe as the coherent scope*, not as empirical frequency.

- **SLO engineering for model-backed services** at per-token granularity and with quality + cost SLOs alongside latency.
- **Observability** for LLM request paths (OpenTelemetry `gen_ai.*`) and GPU workloads (DCGM, GPU Operator, Node Problem Detector).
- **Incident response for non-deterministic failures** (the AI-specific failure classes from `mod-105`).
- **Capacity planning for GPU fleets and model-provider quotas.**
- **Vendor dependency management** — OpenAI / Anthropic / Google / Bedrock / Azure OpenAI with fallback contract, status ingestion, SLA-vs-SLO reconciliation.
- **Chaos engineering** targeted at AI workloads.
- **Release engineering** for model updates, with canary + shadow + SLO-based auto-rollback.
- **DR / BCP / multi-region** for AI serving.

The scope is coherent — the five surfaces from chapter 01, further decomposed. Hiring managers who draft postings against only one or two of these surfaces (for example, a posting that reads as pure MLOps CI/CD plus "and you will set up the Grafana dashboards") are describing a *subset* of the role, not a different role.

<!-- needs-research: live posting sample across Greenhouse / Lever / Ashby / Workday for "AI SRE", "LLM SRE", "AI Production Engineer", "Production Engineer - ML/AI", "AI Reliability Engineer" with filter for in-window (last 90 days) to produce an empirical frequency table per theme. -->

## Where the role is still fragmenting

The scope above is coherent as a *spec*, but at any given company the role is often *not yet* fragmented out of adjacent roles. Four fragmentation patterns to watch for in a job posting:

### 1. AI SRE still inside the MLOps role

At companies whose AI footprint is small or recent, the AI-reliability work has not yet been separated from MLOps. One posting covers both the CI/CD pipeline authoring (level-25 MLOps scope) *and* the SLO / incident / capacity / vendor scope (level-35 AI SRE scope). The role is workable; just notice that you will do two jobs.

**Posting signals.** Title is "MLOps Engineer" or "Senior MLOps Engineer"; scope includes SLOs, on-call, incident response, GPU capacity planning. The MLOps content from `ai-infra-mlops-learning` is explicitly listed alongside the content from this track.

### 2. AI SRE still inside the generalist Production Engineering role

At companies with a strong classical production-engineering / SRE culture whose AI footprint is a recent addition, the AI-reliability work is bolted onto the generalist seat. One engineer covers the AI serving platform *and* the classical web / database / data-pipeline platform. The AI-specific depth is thin; you are expected to add it.

**Posting signals.** Title is "Production Engineer" or "Site Reliability Engineer" (no AI modifier); scope mentions AI / LLM workloads as a bullet alongside bullets for the rest of the company's stack. Expect to retrofit the content of this curriculum onto a role that was not originally scoped for it.

### 3. AI SRE as "inference-engineer with reliability bullets"

At companies whose AI-infrastructure identity is strong and whose SRE identity is weak, the AI-reliability work is bolted onto the inference-engineer role. One engineer builds and operates the serving system. This conflicts with the peer-boundary in chapter 02 and tends to under-serve the SRE side (SLO / post-mortem / chaos discipline), because the engine author is typically closer to engineering the engine than to reliability engineering.

**Posting signals.** Title is "Inference Engineer" or "LLM Infrastructure Engineer"; scope emphasises engine source-code depth (vLLM / SGLang / TRT-LLM at author depth) *and* SLO / incident response. One or the other side usually gets short-changed in practice.

### 4. AI SRE as "AI Platform Engineer"

At companies in early build-out, one platform-engineering role covers everything AI-infrastructure-adjacent — model registry, pipeline, serving, SLOs, incident, vendor. All five of chapter 01's surfaces plus half of the four peers'. This is the classical "founding platform engineer" shape, and it is a career-making or career-breaking seat depending on how the company then matures.

**Posting signals.** Title is "AI Platform Engineer" or "ML Platform Engineer"; scope is comprehensive; team size is small (one-digit); the company is <200 total engineering headcount.

## How to read a posting (for candidates)

Ignore the title. Read the *scope* bullets against the five surfaces from chapter 01:

1. **SLO engineering for model-backed services.** Is it mentioned? At what depth? If the posting mentions "SLOs" but no AI-specific SLI (TTFT, TPOT, quality rate, cost per 1M tokens), the SLO work is probably classical.
2. **Observability for LLM and GPU.** Does it reference OpenTelemetry `gen_ai.*`, LLM-trace platforms, DCGM, GPU Operator? If only Prometheus / Grafana / Datadog are listed, the observability surface is classical.
3. **Non-deterministic incident response.** Any mention of silent drift, hallucination, quality regression, prompt-cache cliff, provider-degraded? If not, incident response is probably classical-only.
4. **Capacity + vendor risk.** GPU fleet planning, model-provider quotas, fallback / LiteLLM / Portkey? If not, the role is probably under-scoped on the vendor surface.
5. **Chaos / release / DR for AI.** Any mention of chaos experiments targeted at AI workloads, canary with response-diffing, DR for stateful AI? If not, these surfaces may not be in scope at that company yet.

A role with all five is a fully-scoped AI SRE seat. A role with three or four is one of the fragmentation patterns above — survivable, usually workable, but read carefully.

## How to write a posting (for hiring managers)

The symmetric version of the above. If you are hiring into this role:

- Pick a title your company's internal convention supports — "AI SRE", "AI Production Engineer", or "AI Reliability Engineer" — and stick to it across the team.
- Scope the posting against the five surfaces explicitly. Do not pattern-match on classical SRE postings; the AI-specific SLIs, the failure-class catalog, the vendor-dependency surface, and the quality-SLO wiring are the tells that make the right candidates self-identify.
- Level the role at a mid-to-staff band. Entry-level candidates will struggle with the breadth; principal candidates will be bored of the operational depth. The 2024–2026 market has converged on a specialist mid-to-staff band for this role.
- Name the peer surface explicitly — which peers the role defers to (inference-engineer, performance, security, ml-platform, eval, mlops), so candidates see the ownership rule is understood.
- Expect a long time-to-fill. The specialty is young; the candidate pool is still small; the people who are strong on *both* classical SRE and AI-specific depth are not yet abundant.

## A note on the role name used in this curriculum

This track uses **AI Reliability Engineer (AI SRE)** as its canonical name because it is the most-neutral phrasing: it does not commit to either the Google-lineage "SRE" or the Meta-lineage "Production Engineer" convention, and it makes the *reliability* scope explicit.

All three market titles in this chapter — AI SRE, LLM SRE, AI Production Engineer — map onto the scope this curriculum teaches. The content is the same; the badge on your business card may differ.

## Summary

- Three titles — "AI SRE", "LLM SRE", "AI Production Engineer" — converge on the same scope. "AI Infrastructure Engineer" is sometimes a synonym, sometimes a different role; read the content, not the title.
- The scope is the five surfaces from chapter 01, further decomposed into the fourteen modules of this curriculum.
- At some companies the role has not yet fragmented out of MLOps, generalist production engineering, inference engineering, or AI platform engineering. Four fragmentation patterns are listed above with posting signals.
- Candidates: read scope bullets against the five surfaces. Hiring managers: pick a title, scope against the five surfaces, level mid-to-staff.
- The autonomous research loop is expected to backfill empirical posting-frequency evidence; this chapter's unquantified claims are marked `needs-research` and will be tightened when that data lands.
