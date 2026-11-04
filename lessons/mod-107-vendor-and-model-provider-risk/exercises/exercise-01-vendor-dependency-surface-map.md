# Exercise 01 — Vendor Dependency Surface Map

**Estimated effort:** 2 hours · **Output:** one structured YAML / Markdown surface map plus a short cover note

## Objective

Produce the vendor-dependency surface map artifact described in chapter 01 §2 for a real or realistic product. The map must cover **every production route that calls a hosted model, embedding, moderation, rerank, image, or audio API**, each row filled in on the six axes (route, region/zone, API/model version, account/quota, SLA, residency) plus the operational columns (criticality, fallback-plan reference). The cover note names the gaps: rows you could not fill, vendor-side information that was not available, and questions that should be answered before the map is signed off.

A good deliverable can be handed to the on-call during an incident and answered in minutes: "which of our routes touches Anthropic?" "what is the SLA on our Vertex route?" "if we fail over to Bedrock, does it violate our EU customer contract?"

## Prerequisites

- Chapter 01 of this module.
- Access to (or plausible representation of) a product's codebase, config, or architecture doc. If you do not have a real codebase, use an illustrative RAG-chat product with a reasonable fan-out (chat model, embedding, moderation, one tool API).
- Current vendor documentation open for whichever upstream appears in the fleet. SLA language and quota-tier structure drift; the map must snapshot the current state.

## Problem statement

Author the map in three parts.

### Part 1 — Enumeration

List every production route that calls a hosted provider. The enumeration must be complete; incompleteness is the single most common failure of this exercise. Sources to cross-reference:

- The service codebase (grep for provider SDKs: `openai`, `anthropic`, `google.generativeai`, `boto3 bedrock-runtime`, `azure.ai`, `litellm`, `together`, `fireworks`, `cohere`, `voyageai`, etc.).
- The service config (env vars, secrets, IaC — API keys and endpoint URLs).
- Egress / network logs if available (the vendor hostnames the fleet calls in production).
- The team's own understanding; this is often incomplete, which is why egress data matters.

State the enumeration method explicitly in the cover note so a reviewer can audit it.

### Part 2 — Map rows

For every enumerated route, produce one row of the map. Use the schema from chapter 01 §2 (reproduced below for reference):

```yaml
- route_id: <stable-id>
  service: <service-name>
  endpoint: <vendor-endpoint-URL>
  vendor: <openai | anthropic | google | bedrock | azure_openai | together | fireworks | groq | baseten | modal | other>
  model: <specific-dated-model-id>
  api_version: <wire-version>
  region_or_zone: <region-or-data-zone>
  account: <org-or-project>
  tier: <usage-tier>
  quota_rpm: <int>
  quota_tpm_in: <int>
  quota_tpm_out: <int>
  reservation: <reservation-id-and-term, or null>
  vendor_sla: <verbatim-or-paraphrased-SLA-language>
  our_slo: <availability-and-latency>
  residency: <declared-data-zone-or-contract-clause>
  zdr: <bool>
  compliance: [<soc2>, <hipaa_baa_on_file>, ...]
  criticality: <tier-1 | tier-2 | tier-3>
  fallback_plan: <plan-id-or-null-with-reason>
```

Rules:

- Pin the model to its specific dated snapshot (`gpt-4o-2024-11-20`, not `gpt-4o`).
- Record the API version separately from the model version.
- Record the SLA language **verbatim** or a close paraphrase with a link to the source; do not substitute a normalised percentage if the vendor's language has conditions.
- If any field cannot be answered, write `unknown — <reason>` rather than leaving it blank.

### Part 3 — Cover note

A short Markdown note (≤ 500 words) with four sections:

- **Enumeration method.** Which sources you cross-referenced.
- **Gaps.** Rows with `unknown` fields; whom you would ask to fill them.
- **Second-order surfaces.** Embeddings, moderation, image, audio, rerank, and tool-API rows. If any are missing, say why.
- **Co-tenancy risk.** If multiple routes share an organisation / workspace / project, call it out and name the gateway-side shaping (or its absence).

## Requirements

1. **Every production hosted-API route appears as a row.** Missing rows is a failure.
2. **Model ids are dated snapshots.** Aliases (`gpt-4o`, `claude-3-5-sonnet`) are not acceptable for tier-1 rows.
3. **SLA language is grounded.** A link to the vendor's published SLA or a quote from the signed contract.
4. **Residency is per row.** Not "the product is EU"; each row's data-zone.
5. **Criticality is tier-1 / tier-2 / tier-3**, not a vague "important."
6. **Fallback-plan reference is present or explicitly null with a reason.** Tier-1 rows with null fallback and no justification fail the exercise.
7. **Cover note names the enumeration method** so a reviewer can re-run it.

## Starter guidance

- The hardest part is enumeration. Grep the codebase once, read the deploy config once, and still assume you missed something. The follow-up is a one-week egress-log review to find the routes the grep missed.
- SLA language changes. Snapshot the vendor's current page with a date-of-check in the cover note.
- Embeddings and moderation are the most-often-missed rows. A workload with only a `chat.completions` row is almost certainly incomplete.
- A single "we use OpenAI" row is not a map. Each distinct (service, endpoint, model, region) tuple is a row.
- Co-tenancy is invisible until the sibling product spikes. If your organisation has multiple products on one provider's quota, the map should say so.

## Acceptance criteria

- YAML / Markdown surface map with ≥ one row per production hosted-API route.
- All rows have dated model ids and explicit SLA language.
- Residency and compliance labels on every row.
- Criticality assigned (tier-1 / tier-2 / tier-3).
- Fallback-plan references present for every tier-1 row, or explicit nulls with reasons.
- Cover note names the enumeration method, calls out gaps, discusses second-order surfaces, and names co-tenancy risk.
- All vendor-side numbers (SLA percentages, quota ceilings, reservation IDs) carry a date-of-check.

## Stretch goals

- **Codify the map in CI.** Write a test that asserts the codebase's model-id constants match the map. Running this on PR blocks silent drift.
- **Egress reconciliation.** Pull one week of egress metrics (which vendor hostnames the fleet called, how many calls, from which pods) and reconcile against the map. Every row in the map should see traffic; every traffic-seen hostname should be in the map. Discrepancies get a new row or a new alert.
- **Deprecation calendar overlay.** For every row, look up the vendor's current deprecation announcement and add a `deprecated_at` field. Produce a Gantt-style timeline of upcoming migrations.
- **Cross-service dependency diagram.** Build a visual (Mermaid, Graphviz, or hand-drawn) of services → vendor routes, colour-coded by criticality and residency. Useful for the auditor and the executive who asks "what breaks if OpenAI is red?"
- **Multi-environment map.** If your fleet has dev / staging / prod, reconcile the map across environments and surface any environment that calls a vendor the prod map does not have (a common source of leaked keys and surprise bills).
