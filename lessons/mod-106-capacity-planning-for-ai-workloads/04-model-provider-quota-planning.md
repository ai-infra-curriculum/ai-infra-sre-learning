# Planning Model-Provider Quota as Capacity

## Why this chapter exists

A capacity plan that only addresses your own GPU fleet misses the other half of most modern AI workloads: the **hosted model providers**. OpenAI, Anthropic, Google (Gemini via Vertex AI / AI Studio), Amazon Bedrock (serving models from Anthropic, Meta, Mistral, Cohere, Amazon's own, and others), and Azure AI Foundry (serving OpenAI and other models) are, from the AI SRE's perspective, **multi-tenant GPU fleets you do not operate**. You have no engine knobs, no autoscaler, no visibility into their batch size, and no way to add capacity except by negotiating. You have **quota** — a contract about requests per minute (RPM), tokens per minute (TPM), concurrent requests, and burst behaviour — and a tier-system that moves the quota ceiling based on spend, time-in-good-standing, and sometimes a direct negotiation.

Quota is capacity. The same AI SRE who sizes a GPU fleet to peak-plus-headroom must size the provider quota pool to peak-plus-headroom too, because a `429 rate_limited` from a provider during the peak hour is a reliability incident whose root cause is a planning failure. This chapter walks through the quota planning discipline across the five major providers at the time of writing, the common quota-pool patterns, the reservation vs. burst-credit models each provider offers, and the data-residency constraint that forces provider selection separately from quota math.

**A caveat on the specifics.** Provider quota limits, tier thresholds, and reservation product names change frequently. This chapter teaches the *planning discipline* with the current (circa late 2026) shape as the illustrative map. Before acting on any specific number, verify against the provider's documentation at the time of the forecast. The discipline is stable; the numbers are not.

## 1. Quota is a capacity contract, not a rate limit

Classical SRE treats rate limits as a defensive mechanism against abuse. Provider quota is not that — it is **the shape of the capacity you have bought**. The distinction matters for how you plan against it.

- **An abuse rate limit** has a hard ceiling you treat as "never hit." If you hit it, something is wrong upstream.
- **A capacity quota** has a ceiling you treat as "we have reserved exactly this much, and we must operate with headroom beneath it." If you hit it, you provisioned too little and the forecast failed.

Hitting a quota ceiling in production manifests as a 429 to your gateway, which manifests as a user-visible error or a long retry queue that burns your TTFT SLO. Either way, the error-budget charge belongs to the capacity-planning function, not to the provider. The provider delivered what you bought.

### 1.1 The three quota axes

For every provider, the quota model is a composition of at least three axes:

- **Requests per minute (RPM).** How many API calls in a minute. A simple counter, usually enforced in a sliding window, sometimes with per-second sub-limits.
- **Tokens per minute (TPM).** How many input + output tokens in a minute. Enforced on a token count estimated at request time (the provider counts prompt tokens plus `max_tokens`-worth of completion, often; actual token counting varies).
- **Concurrent requests.** How many requests can be in flight simultaneously. Important for long-generation workloads where a request stays open for seconds.

Hitting any one of these returns a 429. All three must be forecasted; the one that binds first is the one that will fail during the incident.

### 1.2 The implicit fourth axis: context-window share

Some providers do not advertise it, but **very long context requests implicitly consume more quota than their token count suggests**, because the provider's internal scheduler biases against them (long prefills block small requests behind them, so admission control is stricter). Observed behaviour at the edge: a 200K-token-context request may return 429 even when your TPM headroom is 80%, because the provider's own admission-control is pacing you. The AI SRE's planning posture is to treat very-long-context traffic as a separate class with its own headroom, not as a share of the aggregate quota.

## 2. Provider-by-provider map

Each major provider has a quota model that looks the same from the outside and is subtly different inside. The AI SRE needs the shape of each to plan against them.

### 2.1 OpenAI

OpenAI's quota model:

- **Usage tiers (Tier 1 through Tier 5).** Each tier has per-model RPM and TPM ceilings. Tier promotion is automatic based on cumulative spend and account age thresholds published in the OpenAI platform docs.
- **Per-organisation + per-project.** Quota is enforced at the organisation level; projects within an organisation share the pool. API keys are scoped to a project, so per-key consumption is per-project.
- **Per-model pools.** `gpt-4o`, `gpt-4o-mini`, `o1`, `o3`, `gpt-image-1`, embedding models, moderation — each has its own RPM / TPM pool. Routing traffic between models does not share the pool.
- **Batch API.** A separate asynchronous endpoint with a 24-hour SLA and discounted pricing; it has its own quota pool (typically higher TPM for the same spend tier because latency is not guaranteed).
- **Scale Tier / Priority processing.** For large customers, OpenAI offers reserved capacity tiers (named "Scale Tier" historically and rebranded occasionally) that trade a committed spend for guaranteed throughput and lower variance. The reservation is a separate pool from the on-demand quota.
- **Realtime / streaming.** The Realtime API has its own concurrent-session limit, enforced separately.

**Planning posture.** Model-mix-aware. If your workload spans `gpt-4o` and `gpt-4o-mini`, forecast each pool separately; a surge on `gpt-4o` does not fall through to `gpt-4o-mini` automatically — you have to route the fall-through yourself.

### 2.2 Anthropic

Anthropic's quota model:

- **Tiered quota.** Workspace / organisation tiers, each with per-model RPM and TPM ceilings. Promotion gates on account history and spend.
- **Per-organisation, per-workspace.** Workspaces are the quota boundary below the organisation. Keys are per-workspace.
- **Per-model.** `claude-opus-4-8`, `claude-sonnet-4-6`, `claude-haiku-4-5` (illustrative; refresh from Anthropic's docs) each have their own pools.
- **Input vs. output token distinction.** Anthropic enforces separate ITPM (input tokens per minute) and OTPM (output tokens per minute). A workload with long prompts and short completions can hit ITPM first; a workload with short prompts and long completions hits OTPM first. The AI SRE forecasts both.
- **Priority tier / provisioned throughput.** Anthropic offers reserved capacity through priority programs with committed spend in exchange for guaranteed throughput and lower variance. Separate pool from on-demand.
- **Prompt caching has quota implications.** Cached tokens are billed and counted differently; the ITPM accounting may partly credit cached reads, which changes the effective TPM at a given workload. Verify against the current docs.

**Planning posture.** Input / output asymmetry-aware. Prompt-cached workloads see different effective limits than fully un-cached ones.

### 2.3 Google (Gemini via Vertex AI and AI Studio)

Google's quota model:

- **Vertex AI vs. AI Studio.** Two surfaces. Vertex AI is the production surface with stronger quota controls, project-level quota, and integration with GCP's quota system. AI Studio is the developer surface with lighter controls. Fleets at scale run on Vertex.
- **Per-project per-region per-model quota.** GCP quota is project-scoped and regional. `us-central1` quota for `gemini-2.5-pro` is distinct from `europe-west1` quota for the same model. Cross-region routing is explicit; multi-region quota is a multiplicative plan, not an additive one.
- **Quota types.** Online prediction RPM, online prediction TPM, batch prediction tokens, embedding RPM, embedding TPM. Online and batch are separate quota pools.
- **Provisioned Throughput (PT).** Google's reserved-capacity product. You purchase units of provisioned throughput ("Generative AI throughput units" or `GSU` in the current docs; refresh for the current name) that guarantee a per-minute capacity for a specific model in a specific region. PT is a separate pool from on-demand; on-demand traffic at PT-plus throughput falls through to the on-demand pool or 429s.
- **Quota increases.** Available through the GCP console's standard quota-increase workflow, with provider approval. Usually fast for small increases, slower for large ones.

**Planning posture.** Region-aware. Multi-region fleets need per-region quota forecasts; a surge in one region does not borrow from another. Provisioned Throughput is the right default for predictable production load; on-demand is for burst.

### 2.4 Amazon Bedrock

Bedrock's quota model:

- **Per-region, per-model-id.** Quotas are enforced per AWS region and per Bedrock model ID (`anthropic.claude-sonnet-4-6:0`, `meta.llama3-1-70b-instruct-v1:0`, `amazon.nova-pro-v1:0`, etc.). Cross-region quota is separate.
- **RPM and TPM quotas.** Managed through AWS Service Quotas and increasable through standard service-quota workflow.
- **Provisioned Throughput.** Bedrock's reserved product. You buy model units (MU) of provisioned throughput for a specific model, committed for 1 or 6 months. Guaranteed per-minute throughput; higher per-token cost than on-demand but consistent availability.
- **Cross-region inference profiles.** Bedrock's feature that lets a single request be served from the nearest-available region's pool, smoothing single-region quota hotspots. Changes the quota planning math: you can plan at the cross-region-pool level rather than per-region, with higher effective ceiling at the cost of some per-region clarity.
- **On-demand vs. Marketplace models.** Different models on Bedrock may have different quota behaviour. Verify per-model-ID.

**Planning posture.** Region-plus-model-ID-aware. Cross-region inference profiles can be a legitimate capacity primitive if your data-residency contract allows it; they are not a free pass.

### 2.5 Azure AI Foundry (formerly Azure OpenAI Service)

Azure's quota model:

- **Deployment-based.** You deploy a specific model to a specific Azure resource in a specific region with an explicit capacity assignment (`tokens per minute` and / or `requests per minute`). Unlike OpenAI's shared pool, Azure makes you *allocate* the capacity to a deployment.
- **Standard vs. Provisioned Throughput Unit (PTU).** Standard deployments share capacity at the region's pool level. PTU deployments reserve a dedicated throughput block for consistent performance. PTUs are sold in discrete units with a model-specific conversion from PTU count to tokens-per-minute.
- **Quota by region by model.** Per-region per-model quota, with separate quotas for Standard and PTU deployments. Quota increases through Azure support.
- **Global vs. regional deployments.** Global Standard deployments route across regions to smooth load (an Azure feature similar in spirit to Bedrock cross-region inference); regional Standard pins to one region.
- **Data zones.** Azure's data-residency primitive — deployments can be scoped to a geographic data zone (e.g., EU) and traffic stays within it.

**Planning posture.** PTU is a contractual capacity planning primitive. For predictable production load, PTU is the right default and the forecast is denominated in PTU count, not request rate. Standard is for development, bursty workloads, and models not yet available as PTU.

### 2.6 Summary table — provider quota shape

| Provider | Scope | Reservation product | Residency primitive | Burst option |
|---|---|---|---|---|
| OpenAI | Org / project | Scale Tier (committed) | — (US-based processing by default) | Standard tier RPM / TPM |
| Anthropic | Org / workspace | Priority Tier | Workspace-level region (where offered) | Standard tier RPM / TPM / OTPM |
| Google (Vertex AI) | Project / region | Provisioned Throughput | GCP region + VPC-SC + CMEK | On-demand quota |
| AWS Bedrock | Account / region | Provisioned Throughput (MU) | AWS region + cross-region inference profiles | On-demand quota |
| Azure AI Foundry | Subscription / resource / region | PTU | Azure data zones + region | Standard deployment |

**Verify the current state against each provider's docs when you forecast.** The table above is for planning-discipline illustration, not for operational commitment.

## 3. Per-key vs. per-org quota pools

Almost every provider's quota is enforced at a boundary larger than a single API key. This has two consequences for planning.

### 3.1 The co-tenancy risk inside your own organisation

If your organisation has one AI SRE team and three product teams all consuming the same OpenAI organisation's `gpt-4o` pool, a surge on Product A's workload consumes quota that Product B needs to serve TTFT during its own peak hour. The surge is **inside your own org**; there is no provider failure. The AI SRE's forecast must cover total org-wide demand, and the enforcement of per-team fair-share must happen in your gateway, not in the provider.

Common patterns:

- **Gateway-side rate shaping.** The LLM gateway (LiteLLM, Portkey, Kong AI, a home-built equivalent) enforces per-team or per-product quota budgets within the org-level pool. The sum of the gateway budgets equals or is less than the provider's org-level quota. Budget allocation is a product-management conversation; enforcement is an SRE function.
- **Separate workspaces / projects per product.** If the provider supports per-project quota allocation (Anthropic workspaces, OpenAI projects, GCP projects, Azure resources), isolate products at the provider boundary. Each product has its own ceiling; a surge does not leak across products. Operationally cleaner but requires per-product forecasting and per-product quota ceilings that may not match the per-product spend profile.
- **Hybrid.** Separate workspaces for critical products, shared workspace for everything else with gateway-side shaping.

### 3.2 Hot keys and key rotation

Some providers enforce per-key rate limits in addition to per-org / per-project quotas. A hot key (one API key that handles 80% of traffic) can hit a per-key ceiling even if the org is well within its pool. The AI SRE's forecast must consider the key-usage distribution. The remediation is shard the traffic across more keys, or move to a surface where the provider offers higher per-key allowances for your spend tier.

Key rotation also interacts with quota: rotating a key that is 80% of your traffic to a new key can temporarily suppress quota if the provider's per-key rate-tracking has a warm-up period. Rare but real; call it out in the key-rotation runbook (`mod-110` chapter covering secret rotation).

## 4. Reservation vs. burst-credit models

Every provider has a reservation model and a burst model, with different names and different arithmetic.

### 4.1 Reservation: pay for dedicated throughput

- **OpenAI Scale Tier** — committed spend trades for guaranteed throughput and reduced variance. Typically a monthly spend commitment with a committed capacity curve.
- **Anthropic Priority Tier** — committed spend trades for guaranteed throughput, lower latency variance, and sometimes dedicated capacity.
- **Vertex AI Provisioned Throughput** — purchased in throughput units for a specific model and region. 1-week to 1-year commitments. Separate pool from on-demand.
- **Bedrock Provisioned Throughput (MU)** — purchased in model units, 1- or 6-month commitments. Guaranteed per-minute throughput.
- **Azure PTU** — purchased in PTU units for a specific deployment, with monthly or yearly commitments.

**Capacity-planning posture.** Reserved capacity is the right default for the *predictable* portion of the workload — the baseline that the forecast is confident about, with 1-week forward visibility. The baseline is the number of requests per minute the forecast says will happen whether you want them or not; reserve that and no more.

### 4.2 Burst: on-demand with a quota ceiling

- On-demand quotas on every provider cover the burst above baseline. Charges are per-token with no commitment; the ceiling is the standard RPM / TPM quota.
- **Burst sizing.** The forecast's peak minus the reserved baseline is the burst requirement. The burst must fit within the standard quota ceiling with headroom.
- **Burst-credit tiers.** Some providers quietly offer soft-burst — short periods above the nominal ceiling without 429s if your account history is clean. Do not plan against soft-burst; if it exists, it is bonus survival margin.

### 4.3 The reservation / burst mix

A production capacity plan for a hosted model workload looks like this (illustrative):

```
Workload: product-X-chat, model=claude-sonnet-4-6, region=us-east-1
  Baseline (90th-percentile hour over last 28 days)
    - 120K input tokens/min, 25K output tokens/min
    - Reserve: Anthropic Priority Tier equivalent to 150K ITPM / 30K OTPM
  Peak (99th-percentile hour over last 28 days)
    - 240K ITPM, 55K OTPM
  Burst above baseline
    - 90K ITPM, 25K OTPM
    - Falls through to standard tier quota: 1M ITPM / 300K OTPM
  Fall-through
    - If 429 persists > 30s, fail over to Bedrock cross-region inference
      profile for anthropic.claude-sonnet-4-6:0 (headroom 500K ITPM across
      pooled regions)
  Headroom
    - Reserved pool: provisioned at 1.25× baseline (chapter 02 §4 principle)
    - Burst pool: provisioned at 2× peak-above-baseline
  Risks
    - Reservation commitment is 6 months; if product growth > 2× that,
      reserve is undersized by Q2
    - Fall-through to Bedrock changes data-residency (US-based inference
      instead of US-East-only); check compliance before enabling for EU-origin
      traffic
```

Each line has a reason. The output is sized, reserved, bursted, fallen-through, and compliance-flagged.

## 5. Data-residency constrained provider choice

For many workloads, the choice of provider is constrained before the capacity math begins. Residency constraints arise from GDPR (EU personal data must stay in the EU or an approved jurisdiction), HIPAA (US health data under BAA), the EU AI Act's requirements for high-risk AI systems, UK, Swiss, and other national data protection acts, sector-specific rules (financial services, defence, public sector), and contractual requirements (customer contracts that restrict where their data can be processed).

The AI SRE's planning job is to carry the residency constraint into the quota forecast, because residency-constrained workloads cannot share pools with unconstrained workloads.

### 5.1 The residency map at the time of writing

- **Vertex AI** supports per-region deployment with VPC Service Controls and customer-managed encryption keys (CMEK). EU-only, UK-only, Japan-only processing is achievable.
- **Bedrock** supports per-region inference and data zones (`EU`, `US`) that restrict cross-region inference profile routing. Data stays within the zone.
- **Azure AI Foundry** supports regional and data-zone-scoped deployments. EU data zone, US data zone.
- **OpenAI** offers data-residency options for Enterprise customers (EU data residency program, zero-data-retention terms). Standard tier processes in US.
- **Anthropic** offers region-aware deployment for enterprise customers in specific jurisdictions where available (verify current state).

### 5.2 The planning implication

If a workload must be served from EU residency, the quota forecast must be against the EU-region pools only. On-demand EU quota is typically lower than US quota because provider capacity is concentrated in US regions; reservation (PTU, MU, PT) is often more essential in constrained regions because the baseline must be guaranteed there.

**Plan the residency constraint first. Then plan the quota. Then plan the burst.** Doing them in any other order produces a plan that cannot ship because of a compliance issue nobody surfaced until the lawyer read it.

## 6. The quota-plan artifact

A quota plan is to provider capacity what a node-pool shape (chapter 02) is to own-fleet capacity. A one-page quota plan contains:

1. **Workload identification.** Product / surface / region / model version. Decomposed per chapter 02.
2. **Forecast.** Baseline and peak ITPM / OTPM / RPM / concurrent-requests. Per workload per provider per region.
3. **Reservation.** Provider's reservation product, units, commitment term, expiry date, renewal action owner.
4. **Burst pool.** Standard tier quotas and their current ceilings. Headroom beneath each.
5. **Fall-through.** If the primary provider surge-429s, where does traffic go? The secondary provider's pool, the fall-through region, the degraded-mode path (shorter prompts, smaller model). Named options with cost and compliance annotations.
6. **Residency annotations.** Every pool tagged with its residency zone. Any fall-through crossing a zone boundary explicitly called out.
7. **Reservation calendar.** When does each reservation expire; when does the renewal decision have to be made; who owns it.
8. **Sensitivities and risks.** What changes break the plan. Common ones: a model deprecation moves the pool; a tier promotion restarts the ceiling; a residency contract tightens; a product launch invalidates the forecast.

The plan is refreshed on the same quarterly cycle as the GPU-fleet forecast (chapter 02 section 6). The same reliability discipline — forecast error tracked, actuals compared to projection, renewal lead times watched — applies.

## 7. The vendor-risk boundary

Provider quota planning is one side of a two-sided conversation with the model providers; `mod-107` covers the other side (vendor evaluation, SLA negotiation, incident handoff, deprecation calendar). The boundary:

- **This chapter (`mod-106`):** quota as capacity. How much, where, reserved or burst, with what headroom.
- **`mod-107`:** vendor as risk. Which providers to use at all, what SLAs they offer and what they do not, how to negotiate the Scale Tier / PTU contract, what the deprecation notice periods look like, how to handle a provider degrading silently (`mod-105` chapter 07).

Capacity planning and vendor risk feed each other: a provider you are over-reliant on from a capacity perspective is a provider you have vendor risk with. The quarterly capacity review and the vendor-risk review should be adjacent on the calendar.

## Summary

- **Quota is capacity, not a safety limit.** Hitting a quota ceiling during peak hour is a planning failure; the error-budget charge belongs to the capacity-planning function.
- **Plan on three axes, sometimes four.** RPM, TPM (often split into ITPM / OTPM), concurrent requests. Very-long-context traffic is implicitly a fourth axis.
- **Each provider's quota model is subtly different.** OpenAI is tier-plus-project; Anthropic is tier-plus-workspace with input/output asymmetry; Google is per-region-per-project with PT; Bedrock is per-region-per-model-ID with cross-region inference; Azure is deployment-based with PTU as the reservation. Memorise the shape; verify numbers at forecast time.
- **The per-key vs. per-org split matters.** Co-tenant surges inside your own org are a common incident shape; isolate with projects / workspaces or shape with the gateway.
- **Reservation vs. burst is a product-level decision.** Reserve the predictable baseline; burst the volatility; fall through to a secondary provider or region on quota exhaustion.
- **Residency constrains provider choice before quota math begins.** EU, HIPAA, and sector-specific contracts narrow the pool; plan residency first, quota second, burst third.
- **Produce a quota-plan artifact per workload per quarter.** Forecast, reservation, burst, fall-through, residency, calendar, risks. One page, defensible.
- **Capacity and vendor-risk are two conversations.** This chapter covers capacity; `mod-107` covers risk. Review them adjacently.

The next chapter turns from capacity planning to capacity *operation* — the autoscaling discipline that moves replicas on queue-depth and TTFT custom metrics rather than CPU or GPU-util.
