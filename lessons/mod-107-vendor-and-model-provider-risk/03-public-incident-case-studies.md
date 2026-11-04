# Public Incident Case Studies: Provider Is Not Always Available

## Why this chapter exists

A fallback contract (chapter 02) is only as useful as the imagined failure modes it plans for. Imagination alone is not enough; the public record is full of real, documented provider outages that eclipse what most teams simulate. An AI SRE who has walked through two or three canonical incidents builds the right calibration for *how bad it can get*, *how long it can last*, and *how the vendor communicated during it* — three variables that shape everything from breaker thresholds to the tier-1 escalation ladder.

This chapter is a case-study chapter. Each section is a public incident, grounded in the vendor's own post-incident report or status-page history, summarised for operational use. The purpose is not voyeurism; it is **calibration of the planning assumptions**. Each case closes with the fallback-contract clauses that would have made the on-call night less painful.

A note on specificity: incident write-ups change their framing over time, and some post-mortems were summarised on status pages and later archived. Where I reference a specific duration, cause, or dollar figure, I pin it to the primary source. Where I cannot verify a specific claim, I mark it with `<!-- needs-research -->` so a future editor replaces the placeholder rather than propagating uncertain detail.

## 1. OpenAI, November 2023 — the multi-day DDoS and post-DevDay degradation

### 1.1 What happened

On 2023-11-08, immediately following the November 6 DevDay announcement that introduced GPTs and the Assistants API, OpenAI experienced a multi-day period of elevated error rates and intermittent outages on ChatGPT and the API. OpenAI's public status entries during the week attributed the degradation to **"abnormal traffic patterns consistent with a DDoS attack"** (see status.openai.com historical incidents for the week of 2023-11-08). The degradation affected multiple products: ChatGPT (both free and Plus), the API, and developer tools built on the API. Full recovery took several days of successive partial mitigations.

<!-- needs-research: confirm exact start/end times and the precise mitigation language from status.openai.com history; this write-up uses the public timeline as reported but a direct link to the archived status posts is preferred. -->

### 1.2 What it looked like from the caller's side

- Sustained 5xx and timeouts on `POST /v1/chat/completions` and `POST /v1/completions`.
- Intermittent connection resets during streaming.
- Latency inflation on successful responses (p95 TTFT increased by multiples).
- The status page lagged the on-the-ground experience by minutes in some early phases of the incident, which is typical for evolving events — the operator must detect first, the status page tells the story second.

### 1.3 Why it matters for calibration

Three planning assumptions that this incident invalidated for many teams:

1. **"Outages are short."** Fallback contracts that only reason about 60–90-minute events are undersized. A multi-day degradation is the realistic upper bound for a top-tier provider, and your contract must survive it — which means cost of the secondary at full load for multiple days is not a theoretical concern.
2. **"The provider will warn us first."** The status page is a trailing indicator. The breaker (chapter 02 §2) must be the first-order signal.
3. **"The surface is one endpoint."** The 2023-11 event affected multiple surfaces simultaneously (ChatGPT, API, developer tools). A fallback strategy that only covered `chat/completions` but left embeddings, moderation, or image endpoints with no plan was still partially broken.

### 1.4 Fallback-contract clauses this event justifies

- **Multi-day secondary capacity.** Your secondary (chapter 02 §3) must have the quota headroom to carry 100% of primary's load for multiple days without 429ing itself. This is an ongoing spend decision, not an emergency one.
- **Per-endpoint breaker.** Breakers are per (route, upstream), not per vendor. The vendor being broadly degraded is not sufficient reason to open every route; conversely, the vendor being broadly green does not mean one specific endpoint is healthy.
- **Independent status detection.** Don't wait for the status page. The breaker must fire on observed call behaviour.

## 2. Anthropic — overload and degraded-mode windows

### 2.1 What happened

Anthropic's API has had multiple documented incidents of elevated `overloaded_error` responses during traffic surges — most notably in the periods around major model launches (Claude 3 family in March 2024, Claude 3.5 Sonnet in June 2024, Claude 3.5 Sonnet "new" in October 2024, and subsequent refreshes). The pattern across these events, visible on status.anthropic.com, is a sudden sustained window of elevated `overloaded_error` (HTTP 529) or elevated latency that correlates with the launch demand curve, lasting hours to days with intermittent recoveries.

<!-- needs-research: cite specific post-incident entries from status.anthropic.com for 2024-03, 2024-06, 2024-10 Claude launches and subsequent 2025–2026 launch-period degradations; refresh dates to the specific incidents from the archived status page before publishing operationally. -->

### 2.2 What it looked like from the caller's side

- Elevated HTTP 529 `overloaded_error` responses — a provider-specific signal distinct from a quota 429. Documented by Anthropic at `docs.anthropic.com` under error codes.
- Latency inflation, especially for long-prompt / long-output requests which the overloaded scheduler deprioritises.
- Spiky recoveries: periods of green intermixed with new red windows.

### 2.3 Why it matters for calibration

- **Provider-specific error codes are first-class.** A retry policy written against a generic 5xx misses 529. The policy must include Anthropic's `overloaded_error` as a retryable+breaker-eligible signal; likewise OpenAI's `server_error` and `rate_limit_exceeded`, Google's `RESOURCE_EXHAUSTED` and `UNAVAILABLE`, Bedrock's `ThrottlingException` and `ServiceQuotaExceededException`.
- **Launch day is capacity risk.** When a provider releases a new model, the preceding generation's capacity is often rebalanced toward the new one. Fleets that depend on the preceding model should reassess their quota headroom in the weeks after a major launch.

### 2.4 Fallback-contract clauses this event justifies

- **Provider-specific error taxonomy.** The router maps 529 / `overloaded_error` to a distinct breaker category (not quota, not infrastructure) and treats it as "provider capacity is tight — back off or fall through."
- **Retry-After respect.** When the provider returns a `retry-after` header on 429 / 529, honour it. Spamming retries into an overloaded provider makes the incident worse.

## 3. Google Cloud / Vertex AI regional incidents

### 3.1 Regional-incident shape

Google Cloud has a documented history of regional incidents — see the public incident log at `status.cloud.google.com` — that affect specific regions (e.g., `us-central1`, `europe-west1`) while other regions remain healthy. Historical examples include the 2019-06-02 global networking event that affected Google Cloud, G Suite, and YouTube, and multiple subsequent regional storage, networking, and service-specific incidents.

For Vertex AI specifically, regional-incident patterns include degraded online-prediction endpoints in a single region while cross-region traffic continues, and model-availability lag where a new model id is available in `us-central1` first and in other regions later.

<!-- needs-research: cite specific Vertex AI regional incidents from status.cloud.google.com; refresh for the most relevant recent event before publication. -->

### 3.2 Why it matters for calibration

- **Regional ≠ global.** A Vertex AI regional incident is not a Google Cloud global incident. Your fleet's blast radius depends on how many of your routes pin to the affected region.
- **Cross-region failover is a first-party pattern.** Vertex AI supports endpoint deployments in multiple regions; the fallback contract for a Vertex-AI-primary route may be "route to another Vertex AI region" before it is "route to a different vendor."

### 3.3 Fallback-contract clauses this event justifies

- **Region is a dependency axis** (chapter 01 §1.2). Record it; plan cross-region fallbacks where quota allows.
- **Watch the regional status feed, not only the global one.** `status.cloud.google.com` supports per-region and per-service filtering (chapter 04 §3).

## 4. AWS Bedrock — throttling and model-availability events

### 4.1 Throttling pattern

Bedrock's primary rate-limit error is `ThrottlingException` with associated service-quota codes. A Bedrock caller encountering sustained `ThrottlingException` against a model id has typically either exceeded the per-region, per-model on-demand quota or triggered the per-account `ServiceQuotaExceededException`. AWS documents both at `docs.aws.amazon.com/bedrock/latest/userguide/quotas.html`.

Bedrock quota is per-region, per-model-id. Cross-region traffic is explicit via **cross-region inference profiles** (CRIS), which pool capacity across a defined set of regions within a data zone (e.g., US CRIS pools across us-east-1, us-west-2, and specific others — refresh the current set from the Bedrock docs). CRIS is the vendor-native first-step fallback inside Bedrock.

### 4.2 Why it matters for calibration

- **Provisioned Throughput (PT) ≠ unlimited.** PT is a reservation; traffic in excess of the reservation falls through to on-demand quota or 429s. The AI SRE forecasts both pools.
- **Model availability varies by region.** A fleet assuming `anthropic.claude-sonnet-4-5:0` is present in every region is wrong; the router must know which regions have which models.

### 4.3 Fallback-contract clauses this event justifies

- **CRIS as first-step fallback.** For Bedrock-primary routes, cross-region inference is the first fallback step, not cross-vendor.
- **Separate quota breakers for PT and on-demand.** The router's health signal distinguishes "we exhausted our reservation" from "the service is unavailable," because the response differs.

## 5. Azure OpenAI Service — regional deployment and version drift

### 5.1 The deployment model

Azure OpenAI Service deployments are region-scoped and have multiple deployment types:

- **Standard (regional).** Pinned to a region.
- **Global Standard.** Pools capacity across Microsoft's global footprint.
- **Data Zone Standard.** Pools capacity within a declared data boundary (e.g., EU).
- **Provisioned Throughput Units (PTU).** Reserved capacity at the region level.

Azure OpenAI's SLA (see `azure.microsoft.com/en-us/support/legal/sla/`) differs across these deployment types.

### 5.2 Version-pinning as a reliability surface

Azure OpenAI exposes model versions via explicit deployment configuration — e.g., `gpt-4o` is deployed with a specific model version (`2024-08-06`, `2024-11-20`, etc.) tied to the deployment name. API version is separate (`api-version=2024-10-21`). The deployer controls both; this is operationally different from OpenAI's direct API where model id is a request parameter and API-version drift is handled at the SDK level. Changes to either affect the route; both belong in the dependency map (chapter 01).

Azure OpenAI has also had periods of **capacity contention** around high-demand model releases and preview-to-GA promotions; refresh specific incident details from `status.azure.com` and from Azure OpenAI service health advisories before citing them operationally. <!-- needs-research: specific Azure OpenAI capacity events from status.azure.com -->

### 5.3 Fallback-contract clauses this event justifies

- **Deployment-type-aware policy.** A PTU route that exhausts reservation should fall through to a Standard or Global Standard deployment of the same model, with the understanding that Standard's SLA is different from PTU's.
- **Deployment-level version pinning.** The map (chapter 01 §1.3) records the Azure deployment's model version; the CI drift-check asserts the deployment configuration matches the map.

## 6. Open-weight hoster incidents

### 6.1 What is distinct about the open-weight hosters

Together, Fireworks, DeepInfra, Groq, Baseten, Modal, Replicate, Lambda Labs, and the growing list of inference-as-a-service companies run on top of a fast-moving ecosystem — new model releases weekly, engine upgrades (vLLM, SGLang, TensorRT-LLM) on comparable cadence, and capacity that is often more elastic than the hyperscaler providers because the companies rent GPU capacity on demand themselves.

The public incident pattern across these providers is distinct:

- **More frequent brief outages.** Smaller operational teams, faster-moving infrastructure, and less-mature internal change-management mean small windows of incident more often than at the hyperscalers — typically single-digit minutes, occasionally longer.
- **Model-availability churn.** A model id supported today may be removed or re-deployed on different hardware tomorrow; the quality profile may change under the same id.
- **Published SLAs are enterprise-tier only.** Consumer / pay-as-you-go tiers usually carry no SLA. The map records this explicitly.

### 6.2 Why it matters for calibration

An open-weight hoster is useful in a fallback contract either as a cheaper tier-B alternate or as a third-vendor diversification against the top-three risk. It is rarely the right choice as the only provider for a tier-1 SLO-committed route, unless you have an enterprise contract with specific commitments.

## 7. Model-deprecation events

### 7.1 OpenAI, Anthropic, Google deprecation patterns

All three major frontier-lab providers have deprecated model versions on announced schedules. Public examples:

- **OpenAI.** Published deprecations at `platform.openai.com/docs/deprecations` — periodic retirement of older `gpt-3.5-turbo-*`, `gpt-4-*`, `text-davinci-*`, embedding models, and preview snapshots. OpenAI generally gives multi-month notice and (often) remains callable with an automatic redirect to the successor until the final shutdown date.
- **Anthropic.** Published deprecations at `docs.anthropic.com/en/docs/about-claude/model-deprecations`. Earlier Claude generations (Claude Instant, Claude 2.0, Claude 2.1) have been retired on announced schedules. Shape is similar: multi-month notice, optional redirect behaviour.
- **Google.** Vertex AI model lifecycle at `cloud.google.com/vertex-ai/generative-ai/docs/learn/model-versions`. GA models receive a 12-month deprecation notice by default; preview models have a different, shorter lifecycle.

### 7.2 Why it matters for calibration

A model deprecation is a scheduled outage of a specific dependency row in your map. Your route either migrates to the successor before the shutdown, or it fails in a known-time-and-date incident that was on the calendar for months. Vendor-risk engineering treats a deprecation notice as a work item with the same severity as a planned maintenance.

### 7.3 Fallback-contract clauses this event justifies

- **Deprecation calendar in the map.** For every row in the dependency map, record the current announced deprecation date (null if none). The quarterly audit refreshes.
- **Successor equivalence qualified in advance.** When a vendor announces a deprecation, your canary-eval harness (`mod-105` chapter 03) runs the successor against the workload within the deprecation notice period, and the map / equivalence table is updated.

## 8. Third-party dependency incidents inside the vendor

Even hosted-model providers depend on third parties. OpenAI has documented incidents attributed to upstream providers (CDN partner outages, auth provider degradations); Anthropic runs on AWS. The point is not to enumerate the sub-tree — it is to observe that your provider's own fallback discipline is part of what you are buying. A provider with no public post-mortem culture is a provider that has not been forced to practice. The map's SLA column is the contractual signal; the vendor's public incident history is the behavioural one.

## 9. Pattern-level lessons

Across all the case studies, four patterns recur that the AI SRE must plan against:

### 9.1 Correlated outages

The major events are rarely isolated to one surface. The 2023-11 OpenAI event hit multiple endpoints simultaneously. Launch-day Anthropic events affect multiple models. GCP regional events affect every GCP service in the region, including Vertex AI. A fallback that assumes independent-upstream failures is brittle; plan for correlated upstream degradations.

### 9.2 Long tails

Incidents are not gaussian. A median outage is minutes; the long tail is days. Your error-budget math (chapter 05) must survive the long tail, not only the median.

### 9.3 Status-page lag

The status page is a trailing indicator. Observed behaviour is the leading indicator. Chapter 04 wires both into the pipeline; the breaker reads observed; the human reads status.

### 9.4 Provider-specific error taxonomies are real

Treating all 5xx as equivalent loses information. Each provider has its own error vocabulary (`overloaded_error`, `ThrottlingException`, `RESOURCE_EXHAUSTED`, `rate_limit_exceeded`, `model_overloaded`) that encodes the actionable failure class. The router's normalisation preserves this.

## Summary

- **Real outages are longer and broader than most planning assumes.** The 2023-11 OpenAI event was multi-day; launch-period Anthropic events can last hours; regional Google / AWS / Azure events can affect every service in the region at once. The fallback contract must survive the long tail.
- **Each provider has a specific error taxonomy.** `overloaded_error` ≠ 429 ≠ 503 ≠ 529 ≠ `ThrottlingException`. The router preserves the distinction; the breaker uses it.
- **Status pages lag reality.** Observed behaviour is the leading indicator; the status page is the story. Chapter 04 consumes both; the breaker consumes the observed one.
- **Model deprecations are scheduled outages.** They sit on a calendar; failing to migrate before the shutdown is a planning failure, not a provider failure.
- **Correlated upstream failures are real.** A GCP regional event, an Anthropic launch-day capacity crunch, or an OpenAI DDoS all take multiple surfaces down together. Diversification across vendors is the only mitigation; diversification across endpoints of one vendor is not.

Chapter 04 wires the status-page ingestion into the incident pipeline so the on-call sees the signal within seconds of the vendor publishing it.
