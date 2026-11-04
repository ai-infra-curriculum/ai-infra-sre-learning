# Mapping the Vendor-Dependency Surface

## Why this chapter exists

A model-backed service rarely has one upstream. In production, a single user turn fans out to an identity provider, a vector database, a hosted embedding API, a hosted chat model, sometimes a second hosted model for a judge pass, a search API, a code-execution sandbox, a payment or policy gate, and more. Of these, the **model providers** — OpenAI, Anthropic, Google via Vertex AI / AI Studio, AWS Bedrock, Azure OpenAI / Azure AI Foundry, and whichever open-weights serving companies (Together, Fireworks, DeepInfra, Groq, Baseten, Modal) you depend on — are the upstreams whose outages, deprecations, and silent quality regressions most directly show up as *your* SLO violation, because the user cannot tell the difference between "the vendor returned garbage" and "your product returned garbage."

The first discipline of vendor-risk engineering is therefore simple and unglamorous: **know what you depend on**. Not "we use OpenAI" — that sentence is too coarse to survive an incident. The AI SRE needs a surface map that names, for every production code path, the specific model + endpoint + region + API-version + account tier + residency commitment, and the SLA the vendor publishes vs. the SLO you commit to. Without that map, every outage reduces to "something is wrong with the AI," every renewal is negotiated blind, and every residency audit is a surprise.

This chapter builds the map, and the vocabulary to maintain it.

## 1. The six axes of a dependency

Every vendor route in the fleet has six axes an AI SRE must be able to answer in a sentence each. The map's row shape is these six axes; everything downstream of the map — the fallback contract (chapter 02), the status-page ingestion (chapter 04), the SLA/SLO math (chapter 05) — consumes one or more of them.

### 1.1 The route

Which service inside your fleet calls the vendor, which endpoint, with which model id. "Service `summariser` → `POST https://api.openai.com/v1/chat/completions` → `gpt-4o-2024-11-20`" is the right grain. Service name is ingestion-ready; endpoint is the SLA scope on the vendor's status page; model id is the deprecation unit and the quality unit.

Model ids are not interchangeable even across minor updates. OpenAI's `gpt-4o-2024-08-06` and `gpt-4o-2024-11-20` are different models under one display name; Anthropic's `claude-3-5-sonnet-20240620` and `claude-3-5-sonnet-20241022` likewise. A dashboard that aggregates by display-name hides a release-engineering surface that the vendor manages on your behalf. The map pins the specific dated snapshot, and chapter 06 discusses when to pin and when to track.

### 1.2 The region

Which vendor region serves the route. For the hosted providers this is often:

- **OpenAI.** Historically a single global API; some endpoints are backed in specific regions (EU residency, data-zones for Zero Data Retention customers). The map records the **data zone** rather than a region for OpenAI in most accounts.
- **Anthropic.** Direct Anthropic API is global by default; enterprise customers may have EU or US residency commitments. Anthropic on Bedrock and on Vertex is region-pinned.
- **Google / Vertex AI.** Explicit GCP regions (`us-central1`, `europe-west4`, etc.). Quota, latency, and model availability are per-region.
- **AWS Bedrock.** Explicit AWS regions. Model availability varies by region; new model IDs light up in `us-east-1` or `us-west-2` first and propagate.
- **Azure OpenAI / Azure AI Foundry.** Explicit Azure regions, with a separate *Global Standard* deployment type that pools capacity across regions and a *Data Zone Standard* that pools within a declared data boundary (EU or US at the time of writing — refresh from docs).
- **Open-weights hosters (Together, Fireworks, Groq, DeepInfra, Modal, Baseten).** Typically a single global surface with best-effort regional routing. If residency matters, verify per-provider.

Region is also the unit of status-page scoping for the cloud-provider upstreams; AWS and GCP incidents are typically regional, not global.

### 1.3 The API version and model version

Two axes that look like one. **API version** is the wire contract (`anthropic-version: 2023-06-01`, Azure OpenAI's `api-version=2024-10-21`, Google's SDK major version). **Model version** is the inference artifact (`gpt-4o-2024-11-20`). The vendor deprecates each on its own schedule; the map records each explicitly.

API version drift is a breaking-change surface (chapter 06 §3). Model version drift is a quality surface (chapter 06 §2 and `mod-105` chapter 03). Confusing them in the map confuses them in incident triage.

### 1.4 The account tier and quota profile

Which account, which organisation, which project / workspace, which usage tier, which reservation. The quota numbers in play — RPM, TPM input, TPM output, concurrent requests, batch quota, context-window ceiling — live here. Full quota planning is `mod-106` chapter 04's job; the vendor-dependency map stores only the current ceilings and the reservation IDs so the vendor-risk conversation can reference the numbers without duplicating the forecast.

### 1.5 The SLA the vendor publishes

The vendor's contractual uptime commitment, named explicitly:

- **OpenAI.** No public uptime SLA on the standard API at the time of writing; the Scale Tier product and enterprise contracts carry a latency and availability SLA negotiated per account. The map should record "no public SLA — standard API" or the contract's committed figure for enterprise accounts. <!-- needs-research: confirm latest published OpenAI SLA language for Scale Tier and Business/Enterprise, see openai.com/api/pricing and MSA addenda -->
- **Anthropic.** Enterprise contracts carry an availability commitment. The public API does not publish a self-service SLA; the public commitment lives in the status page's historical uptime and in the MSA. <!-- needs-research: confirm Anthropic's current published SLA posture for API and Priority Tier -->
- **Google Cloud (Vertex AI).** Google publishes a Vertex AI Generative AI SLA at `cloud.google.com/vertex-ai/sla` covering monthly uptime percentage per API. The number is model-family-scoped and changes; refresh the map from the page.
- **AWS Bedrock.** AWS publishes the Amazon Bedrock Service Level Agreement at `aws.amazon.com/bedrock/sla/` (verify URL and current percentage at time of forecast). Typical AWS-service SLA shape: monthly uptime percentage with service credits.
- **Azure OpenAI / Azure AI Foundry.** Microsoft publishes the Azure SLA for Azure OpenAI Service / Azure AI (see `azure.microsoft.com/en-us/support/legal/sla/`). The commitment differs between Standard, Provisioned Throughput Units (PTU), and Global Standard deployments.
- **Open-weights hosters.** Usually no self-service SLA on free / pay-as-you-go tiers; enterprise contracts carry one. The map records "best effort — no SLA" for consumer tiers and the contract figure for committed tiers.

The SLA is *the ceiling* on what your downstream can promise without redundancy. Chapter 05 does the math.

### 1.6 The compliance / residency surface

The constraint that often forces vendor selection independently of price and quality:

- **Data residency.** GDPR, EU AI Act data-handling, Swiss FADP, UK DPA, customer-contract "data must not leave Germany." The map records the declared data zone of the route and the compliance obligation it satisfies.
- **Zero Data Retention (ZDR) / no-training commitments.** OpenAI ZDR, Anthropic's enterprise no-training commitment, Google's "customer data will not be used to improve foundation models" statement, AWS Bedrock's data-isolation clauses, Azure's "your data is not used to train models" statement. The map records which the route is covered by.
- **HIPAA / PCI / SOC 2 / ISO 27001 / FedRAMP.** The certification the route runs under. HIPAA requires a BAA in place with each covered upstream. FedRAMP High forces GovCloud / Azure Government routing.
- **Export control.** Some vendors restrict model availability by country of origin. The map records any affected country list for the route.

A workload that must stay within EU residency cannot fall through to a non-EU region silently; chapter 02 §5 makes this a first-class concern of the fallback contract.

## 2. The surface-map artifact

The map is a structured document, not prose. A table, a YAML file, a Terraform-emitted inventory — the format is whatever your platform reads. The columns are the six axes above plus two more for operational use:

- **Criticality.** Tier-1 (user-facing, SLO-committed), Tier-2 (background / batch, soft SLO), Tier-3 (internal / dev-only, no SLO). The tier drives the fallback contract's effort budget (chapter 02).
- **Fallback plan reference.** A pointer to the fallback contract entry that owns this route's degradation path (chapter 02 §2).

A minimal map row:

```yaml
- route_id: summariser-gpt4o-prod
  service: summariser
  endpoint: https://api.openai.com/v1/chat/completions
  vendor: openai
  model: gpt-4o-2024-11-20
  api_version: v1
  region_or_zone: data_zone_us
  account: org-acme-prod / project-summariser
  tier: usage-tier-4
  quota_rpm: 10000
  quota_tpm_in: 2_000_000
  quota_tpm_out: 400_000
  reservation: scale-tier-committed-2026-Q4
  vendor_sla: "no public SLA on standard API; Scale Tier addendum §3 — 99.9% monthly"
  our_slo: "99.5% availability, p95 TTFT ≤ 2.0s"
  residency: us_only_contract_cust_X
  zdr: true
  compliance: [soc2, hipaa_baa_on_file]
  criticality: tier-1
  fallback_plan: fb-summariser-primary
```

The map is version-controlled in the same repo as the service's infrastructure code. It is updated by the service team, reviewed by the AI SRE function on change, and audited quarterly against reality (section 4).

### 2.1 Why a flat table is not enough

The map is relational. A route depends on a model id; a model id has a vendor-side deprecation date; a reservation has a renewal date and an owner; a residency commitment has a customer-contract clause. A flat spreadsheet degrades as the fleet grows; after a certain number of routes (usually around 20–30), the AI SRE function should move the map to a structured store — a git-backed YAML with validation, a CMDB-style inventory, or (increasingly common) an entry per route in Backstage's Service Catalogue with a custom plugin for the vendor fields.

The specific tool is a function of your platform. The invariant is that the map is machine-readable: chapter 04's status-page ingestor reads it to know which upstreams to subscribe to, chapter 05's SLA math reads it to compose the dependency chain, and chapter 06's version-lock audit reads it to know which model ids to verify against the vendors' deprecation calendars.

## 3. The dependency shapes beyond "we call the API"

Most dependency maps underestimate the surface because they only record the direct hosted-model call. The SRE's map must record the second-order dependencies too, because each is its own 429, deprecation, or residency event.

### 3.1 Embeddings and reranking as separate dependencies

A RAG pipeline's call to OpenAI's `text-embedding-3-large` is a separate route from the same pipeline's call to `gpt-4o`; they have their own quota pools, their own deprecation schedule, and their own SLA. Treating "we use OpenAI" as one dependency hides the embeddings surface, which is often the first to 429 under load because its RPM ceiling is lower relative to the workload's call-rate.

Reranker providers (Cohere Rerank, Voyage, Jina) add another route with its own SLA. The same logic applies.

### 3.2 Moderation and safety as dependencies

OpenAI's Moderation API, Anthropic's prompt-based safety filters, Azure Content Filters, AWS Guardrails for Bedrock, third-party classifiers (Lakera, Protect AI, Prompt Armor). Each is a vendor route; some gate traffic (a failed moderation call fails the user turn); some are observational. Gate-ing routes are first-class dependencies; observational routes are soft dependencies that still deserve a row in the map.

### 3.3 Image / audio / video generation as dependencies

`gpt-image-1`, `dall-e-3`, Stability's APIs, Runway, ElevenLabs, OpenAI TTS, Deepgram, AssemblyAI, Google Imagen. Each is its own quota pool and its own SLA surface. Multimodal fleets are a dependency matrix, not a dependency list.

### 3.4 Tool-calling target APIs as dependencies

An agentic system that calls a tool API (Google Search, Serper, Wolfram, Twilio, Stripe, a user-owned function) depends on each of those as much as it depends on the model provider. The map's scope is at least the AI vendor dependencies; whether it extends to all tool targets is an org-level decision, but the vendor-risk function should at minimum know the critical-path tools and their SLAs.

### 3.5 Shared-provider co-tenancy as an invisible dependency

If your organisation has multiple products that share an OpenAI organisation, a single Anthropic workspace, or a single AWS Bedrock account, your traffic competes with your siblings' traffic for the shared quota pool. A sibling's spike is an incident on your route. The map's "account" and "project / workspace" fields are the dependency boundary for this; `mod-106` chapter 04 §3.1 sketches the gateway-side shaping that keeps a sibling from starving you.

## 4. Keeping the map honest

A dependency map is only useful if it matches reality. The three forces that drift it are:

- **Silent model-id change.** A team updates `gpt-4o` to the next-dated snapshot without updating the map. The quality and latency change but nothing flags it. Mitigation: the map is enforced in CI against the source — a test asserts that the config's model id matches the map's.
- **New route added without the map update.** A team ships a new call to an embedding endpoint and the map does not know. Mitigation: egress-side instrumentation (chapter 04) discovers routes the map does not have and raises an alert.
- **SLA / quota drift the vendor made.** The vendor silently raised a tier ceiling, deprecated a product, or published a new SLA percentage. Mitigation: a quarterly audit cycle reads each vendor's latest docs and updates the map.

A reasonable cadence is:

- **Continuous.** CI blocks on map/config drift.
- **Weekly.** Egress-metric reconciliation (compare observed vendor hosts against map's `endpoint` set).
- **Quarterly.** Full audit — read every vendor's current SLA, quota, deprecation calendar, and residency docs; update the map; cross-sign with the service teams that own each tier-1 route.

## 5. Reading the map in an incident

The map pays for itself in two moments. First, when a vendor status page goes red: the on-call pivots on "vendor = X, affected routes" to find every service that touches that upstream. Second, when a customer asks "is our data covered by your SOC 2?" and the answer requires enumerating every upstream the customer's data traverses. In both moments, the map is the artifact that makes the question tractable in minutes instead of days.

## 6. The AI SRE's surface-map worksheet

1. **Enumerate services.** List every production service that calls a hosted model, embedding, moderation, rerank, image, audio, or tool API.
2. **Fill the six axes per route.** Route, region/zone, API + model version, account + quota profile, vendor SLA, compliance/residency.
3. **Pin the criticality.** Tier-1 / tier-2 / tier-3.
4. **Reference the fallback plan.** For every tier-1 route, the fallback-plan entry (chapter 02) must exist and be linked.
5. **Version the artifact.** The map lives in git; the service owner is the first CODEOWNER; the AI SRE function is the second.
6. **Wire CI.** A test asserts code/config/map agreement.
7. **Schedule the audit.** Quarterly cadence on the calendar with named owners per vendor.

## Summary

- **"We use OpenAI" is not a dependency statement.** The unit of dependency is the (service, endpoint, model id, API version, region/zone, account tier, SLA, residency) tuple. The map is a first-class artifact of the AI SRE function.
- **SLAs are per product, not per vendor.** OpenAI Scale Tier is a different SLA than the standard API, Azure Global Standard is a different SLA than PTU, Vertex AI Gemini is a different SLA than Vertex AI's embedding endpoints. The map carries the specific product's current language.
- **Residency and ZDR are mandatory columns.** A map without them cannot answer the compliance-audit question or the fallback-is-legal question (chapter 02).
- **Second-order dependencies count.** Embeddings, moderation, image, audio, rerank, and tool-call APIs are routes with their own quotas and their own SLAs. Treating them as "the same vendor" hides the surface that fails first.
- **The map must be maintained.** Enforce in CI, reconcile with egress metrics, audit quarterly. A stale map is worse than no map because it lies about what will survive.

Chapter 02 takes this map and turns it into a fallback contract — the explicit, codified plan for what happens when a route in the map fails.
