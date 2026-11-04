# Automating Provider-Status Ingestion into the Incident Pipeline

## Why this chapter exists

A vendor's status page is a public signal with a well-known operational flaw: nobody is watching it at 03:17 local time when a region goes red. The on-call sees the symptom — elevated error rate, p95 regression — and only after they open the laptop and the bookmarks do they discover that the status page already said so. Those lost minutes are exactly the ones where a fallback contract (chapter 02) is supposed to earn its keep by auto-switching before the user turn fails; a human-polled status page leaves the auto-switch unverified and the pager noisier than it needs to be.

The remedy is to treat the status page like any other upstream data feed: scrape or subscribe to it, parse it into a structured event, enrich it with the service-map (chapter 01), and push the enriched event into the same incident-response pipeline that handles your own alerts. This chapter builds that pipeline for the five first-party status surfaces (OpenAI, Anthropic, Google Cloud, Azure, AWS) plus the open-weights hoster shape, and discusses the second-order complications — status-page lag, overly broad incident scopes, and the authoritative-signal problem.

## 1. The status pages and what they publish

Each provider publishes status via a public page powered by one of a small set of status-page tooling vendors. The parse strategy follows from which tooling they use.

### 1.1 OpenAI — status.openai.com

- **Hosting.** Atlassian Statuspage.
- **Public endpoints.** Statuspage exposes JSON at known paths: `https://status.openai.com/api/v2/summary.json`, `.../status.json`, `.../incidents.json`, `.../components.json`, `.../scheduled-maintenances.json`. These return structured data with components (`chat`, `API`, `ChatGPT`, etc.), their current `status` (`operational`, `degraded_performance`, `partial_outage`, `major_outage`), and active incidents with updates.
- **Webhooks.** Statuspage supports subscriber-webhook integrations (`/subscribers.json` / Atlassian's webhooks product). You can register a webhook URL that receives POSTs on incident create / update / resolve events, so you do not have to poll.
- **Granularity.** Per-component (e.g., "API", "ChatGPT", "API Realtime") and per-incident-update.

<!-- needs-research: confirm Statuspage subscriber-webhook availability on OpenAI's specific tenant; this depends on the vendor's plan. The JSON endpoints are reliably public. -->

### 1.2 Anthropic — status.anthropic.com

- **Hosting.** Atlassian Statuspage.
- **Public endpoints.** Same JSON API shape as OpenAI (`/api/v2/summary.json`, `.../incidents.json`, `.../components.json`). Components typically include `Claude.ai`, `console.anthropic.com`, `api.anthropic.com` (and sub-components for model families).
- **Webhooks.** Same Statuspage subscriber mechanism.
- **Granularity.** Per-component and per-incident-update.

### 1.3 Google Cloud — status.cloud.google.com

- **Hosting.** Custom Google infrastructure.
- **Public endpoints.** A public JSON feed at `https://status.cloud.google.com/incidents.json` and a structured API that filters by service and region. See the Google Cloud documentation for the current feed format. Vertex AI and related generative-AI surfaces appear as named services.
- **Granularity.** Per-service, per-region, per-location group.
- **Subscription.** No native webhook equivalent of Statuspage; subscribe by polling the JSON feed or by using the Personalized Service Health in Cloud Monitoring (see `cloud.google.com/service-health`), which emits events into Google Cloud operations and can be forwarded with log sinks.

### 1.4 Azure — status.azure.com

- **Hosting.** Custom Microsoft infrastructure.
- **Public endpoints.** The public page summarises status; the machine-readable surface is **Azure Service Health** inside each Azure subscription, which emits events into Azure Monitor. See `learn.microsoft.com/en-us/azure/service-health/`.
- **Granularity.** Per-service, per-region, per-subscription (Service Health events can be scoped to your account's affected resources).
- **Subscription.** Azure Service Health supports alert rules on service issues, planned maintenance, and health advisories, which can be routed to webhooks, email, SMS, ITSM connectors, logic apps, etc.

### 1.5 AWS — health.aws.amazon.com

- **Hosting.** Custom AWS infrastructure.
- **Public endpoints.** AWS Service Health Dashboard at `https://health.aws.amazon.com/health/status`. The machine-readable surface is **AWS Health** (formerly Personal Health Dashboard), accessible via the `AWSHealth` API for events scoped to your account.
- **Public RSS.** AWS publishes RSS feeds for service events; see `health.aws.amazon.com/health/status` for the current feed URLs. RSS remains the easiest public-side scrape.
- **Granularity.** Per-service, per-region, per-account-affected.
- **Subscription.** AWS Health events can be captured via EventBridge rules and routed to Lambda, SNS, Step Functions, etc. Bedrock-specific events appear in the Health API when they affect your account.

### 1.6 Open-weight hosters

- **Together.** `status.together.ai` (Statuspage).
- **Fireworks.** `status.fireworks.ai` (Statuspage).
- **Groq.** `groqstatus.com` (Statuspage).
- **Baseten / Modal / DeepInfra / Replicate.** Various, mostly Statuspage. Verify each provider's URL in the map.

Statuspage-hosted providers are consistent in API: `/api/v2/*.json` endpoints and the subscriber webhook. A single parser handles them all.

## 2. Ingestion architecture

The ingestor is a small, boring service. Its job is to produce a normalised **provider-health event** on an internal bus. Downstream consumers (incident-management system, router's health signal, dashboard) subscribe to the normalised event; they do not scrape the status pages themselves.

### 2.1 Event shape

```yaml
provider_health_event:
  event_id: ph-20261104-OPENAI-chat-degraded-03
  provider: openai
  component: api.chat.completions
  region: null               # OpenAI is global except where otherwise noted
  severity: degraded_performance   # one of: operational, degraded_performance, partial_outage, major_outage, under_maintenance
  started_at: 2026-11-04T07:12:03Z
  updated_at: 2026-11-04T07:18:51Z
  source_url: https://status.openai.com/incidents/abc123
  source_last_update: "We are observing elevated error rates on chat.completions ..."
  routes_affected:           # enriched from the dependency map (chapter 01)
    - summariser-gpt4o-prod
    - classifier-gpt4o-prod
  criticality: tier-1
```

Normalising per-provider severity into a single enum is the primary parser's job. Statuspage's `operational / degraded_performance / partial_outage / major_outage / under_maintenance` is already a good target vocabulary; map Google / Azure / AWS states onto it.

### 2.2 Pull vs. push

- **Pull.** Poll the JSON / RSS endpoints every N seconds. Simple, robust, slightly laggy. Reasonable default N = 60 for general status, N = 30 if you want to lead the on-call.
- **Push.** Use provider-native subscription (Statuspage subscriber webhook; Azure Service Health alerts; AWS Health via EventBridge; GCP Personalized Service Health via Pub/Sub). Lower lag, tighter coupling to vendor-side correctness, requires a reachable endpoint.
- **Both.** Push is primary; pull is a fallback so you catch events during a webhook outage or a brief subscriber-list drift. The pull can also be slower (every 5 min) when used as a fallback.

### 2.3 Deduplication

Status-page events update multiple times during an incident. The ingestor deduplicates on `(provider, incident_id)` and only emits a new event when the severity transitions or the component set changes. Otherwise the pipeline receives one event per vendor-side text update and the on-call gets paged twice for the same incident at 07:12 and 07:14.

### 2.4 Enrichment

Before emission, the ingestor cross-references the dependency map (chapter 01) to compute `routes_affected` and `criticality`. This is the step that turns "there is an incident on the OpenAI chat API" into "there is an incident on the OpenAI chat API, which affects routes `summariser-gpt4o-prod` and `classifier-gpt4o-prod`, both tier-1."

The enrichment is what makes the pipeline worth building. A status feed that is not cross-referenced against the service map is at best interesting; cross-referenced, it is actionable.

## 3. Wiring into the incident pipeline

### 3.1 Severity → action mapping

A reasonable baseline:

- `operational`. Clear any prior degradation flag; no action.
- `degraded_performance` on a tier-1 route. Flag the breaker signal "provider.degraded = true" (read by the router per chapter 02 §2.3). Page the on-call at the awareness level ("inform, do not act").
- `partial_outage` / `major_outage` on a tier-1 route. **Auto-open an incident** in your incident-management system (PagerDuty / Opsgenie / FireHydrant / Incident.io / Rootly) with full context, page the on-call at the response level, and set the breaker's status-page signal.
- `under_maintenance`. Record; raise only if the service team did not pre-stage a maintenance plan.

### 3.2 Incident creation payload

The auto-created incident carries:

- The normalised event (§2.1).
- A link to the vendor's status page entry.
- The list of affected internal routes.
- The on-call fallback-contract reference per route.
- A preformatted first comment with the runbook pointer and the vendor's most recent status-page text.

### 3.3 Router coupling

The router consumes the normalised event via a lightweight sidecar or a short-polled cache. A pseudocode fragment for the breaker's status-signal read:

```python
def is_primary_degraded(route, now):
    health = redis.get(f"provider_health:{route.primary.provider}:{route.primary.component}")
    if not health:
        return False
    event = json.loads(health)
    if event["severity"] in ("partial_outage", "major_outage"):
        return True
    if event["severity"] == "degraded_performance" and route.criticality == "tier-1":
        return True
    return False
```

The router's breaker (chapter 02 §2) combines this signal with the locally observed error-rate signal. Status-page alone usually does not switch the router by itself — the combination is what trips the automatic fallback.

### 3.4 Dashboard surface

A single dashboard widget shows the current status of each depended-on provider (from the ingestor), with timestamps of last update and a list of currently affected routes. Observability links the widget to the service's own error-rate panels so the on-call sees "vendor says degraded; our route is also showing elevated 529" side by side. `mod-104` chapter 06 has the broader observability pattern; this chapter adds the provider-health column.

## 4. Operational pitfalls

### 4.1 Status-page lag

Status pages are not real-time. Even push-based subscriptions see a lag from the operator's keystroke to your webhook: minutes in the normal case, 10–30 minutes during fast-moving events. The ingestor must *not* be the sole trigger for a fallback. Chapter 02 §2 forbids this; chapter 04's job is to add the status-page signal alongside the breaker, not to replace it.

### 4.2 Over-broad incident scopes

Vendors sometimes publish "elevated errors on the API" when only a specific model family is affected. The ingestor cannot disambiguate; the router's local breaker on the specific route can. Treat status-page events as *necessary-for-awareness, not sufficient-for-action*.

### 4.3 Retroactive status updates

Occasionally a vendor opens an incident, updates it to "resolved," then re-opens it hours later with a connected post-mortem. Your ingestor deduplicates on `(provider, incident_id)` to handle this; keep incident history keyed by the vendor's incident_id, not by a hash of the current state.

### 4.4 Out-of-band provider communication

For enterprise accounts, some providers communicate via account manager, Slack Connect, or email in parallel with (or before) the public status page. If your team receives such comms, the pipeline needs an **account-manager-event ingress** — a webhook or a parsed email channel — that produces the same normalised event. The downstream path does not care how the event was discovered.

### 4.5 Rate limits on the status API itself

Statuspage's public JSON endpoints have no documented rate limit, but polling every 10 seconds from a large deployment is poor practice. One central ingestor polls at the chosen cadence and fans out; individual services do not call status pages directly.

### 4.6 Status-page outages

Status pages can themselves be unavailable. If the ingestor's poll fails for a sustained window, do not infer vendor degradation; the status page being 503 is not the same as the API being 503. The breaker continues to work from observed call behaviour; the pipeline raises a secondary alert for "cannot reach provider status endpoint for > 15 minutes."

## 5. Minimal reference implementation

A pragmatic reference: a small Python service (one container, one deployment) that holds the following structure.

```python
# status_ingestor.py (reference skeleton; drop into a FastAPI or a worker)
from dataclasses import dataclass
from typing import List, Optional
import httpx, time

STATUSPAGE_FEEDS = {
    "openai":    "https://status.openai.com/api/v2/summary.json",
    "anthropic": "https://status.anthropic.com/api/v2/summary.json",
    "together":  "https://status.together.ai/api/v2/summary.json",
    "fireworks": "https://status.fireworks.ai/api/v2/summary.json",
    "groq":      "https://groqstatus.com/api/v2/summary.json",
}

# Google / Azure / AWS have their own shapes handled separately.

@dataclass
class ProviderHealthEvent:
    provider: str
    component: str
    region: Optional[str]
    severity: str
    started_at: str
    updated_at: str
    source_url: str
    source_text: str

def parse_statuspage_summary(provider: str, body: dict) -> List[ProviderHealthEvent]:
    events = []
    for comp in body.get("components", []):
        if comp["status"] != "operational":
            events.append(ProviderHealthEvent(
                provider=provider,
                component=comp["name"],
                region=None,
                severity=comp["status"],
                started_at=comp["updated_at"],
                updated_at=comp["updated_at"],
                source_url=body.get("page", {}).get("url", ""),
                source_text=comp.get("description") or "",
            ))
    return events

def poll_once():
    all_events = []
    for provider, url in STATUSPAGE_FEEDS.items():
        try:
            r = httpx.get(url, timeout=10.0)
            r.raise_for_status()
            all_events.extend(parse_statuspage_summary(provider, r.json()))
        except Exception as exc:
            emit_meta_event(provider, exc)
    for ev in all_events:
        if is_new_or_changed(ev):
            publish(enrich_with_service_map(ev))

if __name__ == "__main__":
    while True:
        poll_once()
        time.sleep(60)
```

Google Cloud, Azure, and AWS parsers each have their own shape; the overall architecture is identical.

The `enrich_with_service_map` step is where the dependency map (chapter 01) joins the event. `publish` is Kafka / Pub/Sub / SNS / Redis Streams / whatever your platform uses. `is_new_or_changed` is the dedup logic. `emit_meta_event` records the ingestor's own health.

A service of this shape fits in one small container, has a very small on-call surface, and pays for itself the first time it auto-opens an incident at 03:17 local time.

## 6. Testing the ingestor

The ingestor is a boring service; it is also on the critical path for incident awareness. Three test levels:

- **Unit tests against recorded feeds.** Snapshot real status-page JSON (Statuspage's summary.json is small and public-domain), including an active-incident state and a resolved-incident state. Assert the parser produces the expected normalised event.
- **Integration tests end-to-end.** Spin up the ingestor against a mock vendor feed, assert the normalised event lands on the bus, assert dedup fires on a repeat, assert enrichment fires with a stubbed service map.
- **Chaos tests.** Fail the vendor feed, assert the meta-event is emitted after N consecutive failures and that the pipeline continues to work for the other feeds.

`mod-109` (chaos) houses the game-day coverage; chapter 04's tests are the always-on protection.

## 7. The AI SRE's ingestor worksheet

1. **Enumerate the status feeds.** One per vendor in the dependency map (chapter 01). Record the URL, auth requirement, feed format, and poll cadence.
2. **Decide pull / push / both.** Default both. Push via native subscription (Statuspage webhook, Azure Service Health alerts, AWS EventBridge, GCP Personalized Service Health). Pull as a fallback.
3. **Normalise.** One event shape, five-ish severity levels, one mapping function per vendor.
4. **Enrich.** Cross-reference the map; emit `routes_affected` and `criticality`.
5. **Route the enriched event.** Incident-management auto-create for tier-1 partial/major outages; breaker signal for all `degraded_performance+`; dashboard for everyone.
6. **Dedup.** On `(provider, incident_id)`; emit only on severity / component-set transitions.
7. **Monitor the ingestor.** Meta-alert on no-poll-success-for-N-minutes per provider.

## Summary

- **The status page is a public data feed.** Scrape it, parse it, normalise it, enrich it, route it. A human polling bookmarks at 03:17 is a planning failure.
- **Statuspage-hosted vendors expose a standard JSON API.** `/api/v2/summary.json` and friends. OpenAI, Anthropic, and most open-weight hosters use the same shape; one parser suits all.
- **Google, Azure, and AWS each have native health services.** Personalized Service Health (GCP), Azure Service Health, AWS Health (via EventBridge, RSS) — each is lower-lag than scraping the public page.
- **Enrichment against the dependency map is what makes the event actionable.** A raw "OpenAI chat API is degraded" is interesting; "OpenAI chat API is degraded and affects these three tier-1 routes" is operational.
- **Status-page lag is real.** The ingestor's event augments the breaker signal; it does not replace it. Chapter 02's breaker is the primary auto-switch signal.
- **The ingestor is on the critical path.** Monitor it, test it, feed it into chaos drills. A silent ingestor is worse than no ingestor — it tells the on-call "no incident" when there is one.

Chapter 05 turns the dependency-map SLAs and the ingestor's event stream into the error-budget math that governs how strict your downstream SLO can be.
