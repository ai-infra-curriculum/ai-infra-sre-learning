# mod-107-vendor-and-model-provider-risk: Vendor and Model-Provider Dependency Risk

**Estimated effort:** 14 hours

A model-backed service's reliability is the composition of its own reliability and the reliability of every hosted dependency it calls. This module teaches the AI SRE to **name**, **plan around**, and **auto-react to** those dependencies: the hosted-model routes, their published SLAs, the fallback contract the router implements, the status-page signals that light the incident pipeline, the error-budget math that governs how strict the downstream SLO can be, and the vendor-eval scorecard that drives the choice of vendor set. The role's boundary is explicit: contract negotiation, security review, and the cost-aware router implementation each belong to a named peer — this module's output is the specification those peers consume.

## Learning objectives

- Model the vendor-dependency surface — which OpenAI / Anthropic / Google / Bedrock / Azure OpenAI / open-source-provider routes you depend on, the SLA each publishes vs. the SLO you commit to, the data-residency / compliance / rate-limit / token-ceiling constraints each imposes
- Design a provider-fallback contract — LiteLLM-shape router, Portkey-shape gateway, model-equivalence table across providers, when 'fallback to cheaper model' is acceptable vs. when it violates the quality SLO, idempotent retry semantics and streamed-response resumption across provider swap
- Operate against real public incidents — reference the OpenAI 2023 November DDoS multi-day degradation and subsequent frontier-lab outages, the Google Cloud regional incidents, the AWS Bedrock throttling incidents — as canonical case studies for 'provider is not always available'
- Automate provider-status ingestion — pull status.openai.com / status.anthropic.com / status.cloud.google.com / status.azure.com / health.aws.amazon.com into the incident-response pipeline; auto-open an incident when a depended-on provider reports degraded
- Reconcile SLA vs. SLO — your downstream SLO cannot be stricter than the composed SLA of your upstreams unless you add redundancy; explicitly budget 'provider degradation' as an error-budget consumer and reason about reserved-budget discipline
- Build vendor eval discipline — when to depend on one provider, when to multi-source by default, when to self-host (hand-off to ai-infra-inference-engineer peer's build-vs-buy module), what exit cost looks like, how to version-lock a provider API against breaking changes
- Cite the boundaries — the cost-aware router implementation sits with ai-infra-inference-engineer peer; the legal / contractual side of SLA sits with legal / vendor-management out of scope

## Chapter index

- [`01-vendor-dependency-surface.md`](./01-vendor-dependency-surface.md) — The six axes of a vendor dependency (route, region/zone, API/model version, account tier, SLA, residency); the surface-map artifact; second-order dependencies (embeddings, moderation, tool APIs, co-tenancy); keeping the map honest in CI and quarterly audit.
- [`02-provider-fallback-contract.md`](./02-provider-fallback-contract.md) — The health signals, equivalence table, policy, and semantics of a per-route fallback contract; LiteLLM and Portkey as reference router shapes; idempotency, streaming resumption, tool-use multi-turn, context-window mismatch, trace continuity; testing with synthetic drills and game days.
- [`03-public-incident-case-studies.md`](./03-public-incident-case-studies.md) — OpenAI 2023-11 DDoS, Anthropic overload and launch-day capacity events, Google Cloud regional incidents, AWS Bedrock throttling, Azure OpenAI regional and deployment-type patterns, open-weight hoster shape, model-deprecation events; the pattern-level lessons (correlated outages, long tails, status-page lag, provider-specific error taxonomies).
- [`04-provider-status-ingestion.md`](./04-provider-status-ingestion.md) — Scraping Atlassian Statuspage (`/api/v2/summary.json`) for OpenAI/Anthropic/open-weight hosters; Personalized Service Health (GCP), Azure Service Health, AWS Health; normalising to one event shape; enriching with the dependency map; wiring into incident-management auto-create, router breaker signal, and dashboard; the operational pitfalls (lag, over-broad scopes, status-page outages).
- [`05-sla-vs-slo-reconciliation.md`](./05-sla-vs-slo-reconciliation.md) — The series-composition rule (`∏ a_i`); why real dependencies are not independent (shared cloud, shared CDN, shared model) and the correlation correction; adding redundancy to raise the ceiling; reserved-budget discipline for provider degradation; hard-no-op cases as explicit budget line items; the error-budget table artifact.
- [`06-vendor-eval-discipline.md`](./06-vendor-eval-discipline.md) — Single-source vs. multi-source vs. self-host decision frame; version-locking API and model (OpenAI, Anthropic `anthropic-version`, Azure `api-version`, model-id deprecation calendars); exit cost as a first-class metric; the per-vendor per-route scorecard artifact; annual, quarterly, and event-driven review cadence.
- [`07-peer-boundaries.md`](./07-peer-boundaries.md) — Where vendor-risk engineering stops: the `ai-infra-inference-engineer` peer owns the router implementation and cost-aware routing; legal/vendor-management owns the contract and SLA negotiation; security owns the vendor security review; FinOps owns cost allocation; the `ai-infra-pipeline` peer owns vendor-side artifacts (fine-tunes, vector stores). Shared-artifact ownership and the AI SRE's irreducible composition role.

## Structure

- `01-…md` … `07-…md`: lecture chapters.
- `exercises/`: per-exercise prompts (surface map, fallback contract, incident case study, status ingestor, SLA-SLO reconciliation, vendor scorecard).
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.

## How this module composes with the track

- **Input from `mod-101`.** The AI SRE's position-on-the-ladder vocabulary — the generalist-with-routing-competence framing — is the premise under which chapter 07 draws the peer boundaries.
- **Input from `mod-103`.** The SLI / SLO authoring conventions; chapter 05's reconciliation math extends that vocabulary with upstream-SLA composition.
- **Input from `mod-104`.** The observability stack produces the signals (per-provider error-rate, latency, request-id correlation, egress metrics for map reconciliation) the router's breaker and the status ingestor consume.
- **Input from `mod-105`.** Provider-degraded-mode detection (`mod-105` chapter 07) is the quality-regression signal that chapter 02's breaker reads; silent-output-drift canary evals (`mod-105` chapter 03) are the equivalence-table and version-lock sentinels.
- **Input from `mod-106`.** Quota planning (`mod-106` chapter 04) is the reservation / burst / fall-through shape that the dependency map records; capacity forecast is the input to the equivalence-table sizing.
- **Output to `mod-108`.** Incident response for AI systems consumes the fallback contract's named policies and the status-ingestor's events as first-class inputs to the incident lifecycle.
- **Output to `mod-109`.** Chaos engineering uses the fallback contract and the dependency map as the menu for provider-failure injections; the game-day drill lives in `mod-109`.
- **Output to `mod-113`.** DR / BCP for AI consumes the cross-vendor and cross-region redundancy decisions this module specifies.
