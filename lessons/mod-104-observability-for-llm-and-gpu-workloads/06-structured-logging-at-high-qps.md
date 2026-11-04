# Structured Logging at High QPS — Sampling, Redaction, Correlation, Cost

## Why this chapter exists

The trace stack from chapter 01 answers "show me the span for request X." The metric stack from chapter 02 answers "what percent of requests breached the TTFT SLO in the last hour?" Neither answers "show me every log line emitted while processing request X across the gateway, engine, and GPU-side event bus." That is the logging layer's job.

For an inference fleet, the logging layer is both the most familiar observability silo (every engineer has written a log statement) and the most routinely misused. Common failure modes:

- Logging raw prompt and completion content in cleartext, at full volume, into a shared log store — which is a compliance incident waiting to be filed under its own CVE number.
- Writing logs with no `trace_id`, so there is no way to correlate a log line to the trace or metric that burned.
- Logging at unlimited volume because "logs are cheap," and later discovering the monthly spend is competing with the GPU bill.
- Logging structured data as a free-form string (`logger.info(f"request {req.id} took {took}s")`) so query-side extraction is a regex nightmare.
- Logging the same information at all three layers (gateway, engine, GPU-event bus) with three different naming conventions.

This chapter is the authoring reference for structured logging on an LLM fleet — the schema, the redaction discipline, the trace-ID correlation, the sampling strategy, the cost model, and the integration with Grafana Loki, Cloud Logging (GCP), or CloudWatch Logs. The goal is logs that compose with the trace and metric stacks, are safe at high QPS, and do not blow the budget.

Primary references: the [Grafana Loki documentation](https://grafana.com/docs/loki/latest/), [Google Cloud Logging best practices](https://cloud.google.com/logging/docs/view/logs-explorer-interface), [OpenTelemetry Logs data model](https://opentelemetry.io/docs/specs/otel/logs/data-model/), and the [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) for the security side.

## 1. The three roles of logs on an inference fleet

Before writing a schema, be clear about what the log is for. There are three distinct uses, each with its own retention, redaction, and sampling profile.

### Role 1 — Operational logs (the "what happened" record)

Per-request records of the operational lifecycle: request accepted, admission decision, routed to pool, upstream call started, upstream response received, response sent to client. The bread-and-butter log role.

- **Retention.** Days to weeks (14–30 is typical).
- **Redaction.** No raw prompt or completion content; metadata only.
- **Sampling.** Full-rate typically; high-QPS fleets may sample at the 10% level at this role but it is uncommon.
- **Correlation.** Must carry `trace_id`, `span_id`, request id, model id.

### Role 2 — Error and diagnostic logs (the "what went wrong" record)

Stack traces, retry details, provider error responses, admission-reject reasons, deserialization failures. The "this is why the request failed" record.

- **Retention.** Weeks to months (30–90 days typical).
- **Redaction.** Raw content may leak in the provider error response — scrub at log emission.
- **Sampling.** Full-rate always. Dropping an error log is worse than dropping a success log.
- **Correlation.** Trace id critical.

### Role 3 — Audit logs (the "who did what and when" record)

Content-access events (someone read a stored prompt), configuration changes (SLO threshold modified, redaction policy changed), production-data exports, admin actions.

- **Retention.** Months to years, often driven by compliance regime.
- **Redaction.** No PII in the audit record itself; the record references the actor and the action, not the data operated on.
- **Sampling.** None; every event.
- **Correlation.** Actor id, action id, resource id, trace id if request-scoped.

Operational logs go into Loki / Cloud Logging at high volume; error logs go into the same store but are more selectively retained; audit logs should live in a separate append-only store with stricter access control. Using one store for all three is a common shortcut and a common audit finding.

## 2. The log schema — what every line should carry

Structured logs are JSON (or OTel log records, which encode to JSON on the wire). Every log line on the request path should carry the following fields; name conventions align with OpenTelemetry's [log data model](https://opentelemetry.io/docs/specs/otel/logs/data-model/) and the `gen_ai.*` semantic conventions from chapter 01.

### Top-level fields

| Field | Example | Required? |
|---|---|---|
| `timestamp` | `2026-11-04T09:12:34.123Z` (ISO 8601, UTC, ms precision) | Yes |
| `severity` | `DEBUG` / `INFO` / `WARN` / `ERROR` / `CRITICAL` (OTel severity text) | Yes |
| `severity_number` | OTel severity number (1–24) | Recommended |
| `service_name` | `llm-gateway` / `vllm-worker` / `retrieval-service` | Yes |
| `service_version` | `v1.42.0` or a git SHA | Yes |
| `host` / `pod` / `region` | Infrastructure attributes | Yes |
| `message` | The human-readable message, short | Yes |
| `event` | Machine-readable event name (`request.accepted`, `upstream.called`, `response.sent`) | Yes — enables structured queries |

### Correlation fields

| Field | Example |
|---|---|
| `trace_id` | OpenTelemetry trace id (32 hex chars) |
| `span_id` | OpenTelemetry span id (16 hex chars) |
| `request_id` | Gateway-assigned request id |
| `tenant_id` | Hashed or pseudonymous tenant id |

The `trace_id` and `span_id` are how the log line joins to the trace stack from chapter 01. Any log line without them is operationally orphaned — you see it, you cannot pivot from it. Enforce at the emitter.

### LLM-specific fields (per the `gen_ai.*` conventions)

| Field | Example |
|---|---|
| `gen_ai.operation.name` | `chat` |
| `gen_ai.system` | `openai` |
| `gen_ai.request.model` | `gpt-4o-mini` |
| `gen_ai.response.model` | `gpt-4o-mini-2024-07-18` |
| `gen_ai.response.finish_reasons` | `["stop"]` |
| `gen_ai.usage.input_tokens` | `412` |
| `gen_ai.usage.output_tokens` | `87` |
| `gen_ai.server.time_to_first_token` | `0.218` |
| `surface` | `chat` |

### Content fields (per the redaction rules in section 3)

| Field | Example | Notes |
|---|---|---|
| `prompt.content` | — | **Default: omitted.** See section 3. |
| `completion.content` | — | **Default: omitted.** See section 3. |
| `prompt.content.hash` | `sha256:ab...` | Hash of the unredacted content, if content capture is off. Enables forensic matching without storing content. |
| `prompt.content.redacted` | `true` / `false` | Whether the stored content passed the redaction pipeline. |
| `prompt.content.redaction_policy_version` | `v1.3.0` | For audit. |

### Example log line — an accepted chat request

```json
{
  "timestamp": "2026-11-04T09:12:34.218Z",
  "severity": "INFO",
  "severity_number": 9,
  "service_name": "llm-gateway",
  "service_version": "v2.14.3-ab83fe1",
  "host": "gw-us-west-2-c-7",
  "region": "us-west-2",
  "event": "request.accepted",
  "message": "chat request accepted for gpt-4o-mini",
  "trace_id": "4e5b3a9c8f2d1e7b6c4a0f8e9d3b5a2c",
  "span_id": "a1b2c3d4e5f60718",
  "request_id": "req_01HQZABC...",
  "tenant_id": "sha256:7f9b...",
  "gen_ai.operation.name": "chat",
  "gen_ai.system": "openai",
  "gen_ai.request.model": "gpt-4o-mini",
  "surface": "chat",
  "prompt.content.hash": "sha256:abcde1234567...",
  "prompt.content.redacted": false
}
```

Every log line follows this shape. A reader with the schema in mind can scan a hundred lines and immediately see what shifted.

## 3. Prompt and completion content — the redaction discipline

Chapter 01 covered the OTel-side policy for prompt / completion capture. The log layer is the other side of the same policy, and the stakes are the same: the log store is a content store if logs carry content, and the log store is routinely more accessible to engineers than the trace store.

The default on operational and error logs is: **content is not emitted.** Logs carry metadata (token counts, finish reasons, timings) — never raw text.

### When content *is* emitted, the pipeline

Match the chapter 01 redaction pipeline:

1. **Structured identifier regex** — emails, phone numbers, Luhn-validated PANs, SSNs for in-scope jurisdictions, URLs with secret-looking tokens. Replace with `[REDACTED:<type>]`.
2. **NER-based PII detection** — [Presidio](https://microsoft.github.io/presidio/), spaCy with domain entity sets, or a cloud DLP API. Replace detected entities with `[REDACTED:<entity>]`.
3. **Domain-specific redaction** — your product's identifier types.
4. **Hash and store** — SHA-256 of the unredacted form, emitted as a hash field.
5. **Emit flag** — `prompt.content.redacted=true` and the policy version.

If content capture is **off**, still emit the hash (`prompt.content.hash`) so that when a user reports a problem with "my chat from this morning," the on-call can match by hash without storing content. This is the compliance-friendly default.

### Redaction has to happen at the emitter

A common footgun: emit the raw log, redact at the aggregator (Fluent Bit / Vector / OTel Collector). In this architecture, the raw content exists in the host's log files, in the kernel's buffer, and in the network-layer transport until the aggregator sees it. A crash, a heap dump, or a side-channel log tap exposes it.

Correct: redact *before* the log line leaves the emitting process. The aggregator's job is routing, not security.

### Severity override during incidents

An exception the compliance team usually supports: during a declared incident, an incident-commander-authorized flag can enable raw-content capture *for a specified time window and a specified narrowed surface*. The flag is an audited event (logged separately in the audit store), the window auto-expires, and the captured content has shorter retention. This is the mechanism for "we need to debug this one bad response" without permanently loosening the policy.

Codify it in the policy document; do not leave it informal.

## 4. Sampling at high QPS — the economic forcing function

A medium-scale inference fleet produces roughly 1 log line per request (role 1) × 5–10 lines per request lifecycle × N requests per second. At 1000 QPS, that is ~10k log lines per second, which at steady-state fill rates lands in the terabyte-per-day range. The log-store bill is non-trivial; at scale it can rival the GPU bill.

Three sampling strategies compose:

### Strategy A — head-based sampling (sample before emission)

Decide at request admission time: this request's logs will be emitted, or they won't. Set a `sample_decision` flag on the trace context; every log line in the request's lifecycle honours it.

- **Pros.** Deterministic; the entire request's log history is present or absent as a unit. Cheap at the emitter.
- **Cons.** You lose signal on the sampled-out requests. You don't see their errors because errors are sampled out alongside successes.

Mitigation: head-based sampling for *success* role-1 logs; always emit role-2 (error) and role-3 (audit) regardless of sample decision. The decision table in your emitter looks like:

```
if severity >= ERROR: emit
elif event in AUDIT_EVENTS: emit
elif sample_decision: emit
else: drop
```

### Strategy B — tail-based sampling (sample after observing)

Keep the entire request's logs in a short-term buffer; at request end, decide to keep or drop based on what happened. Keep all errors, all slow requests (above a percentile threshold), a configurable fraction of baseline.

- **Pros.** Captures the interesting tail — errors, slow requests, long-completion requests — at near-100%. Drops the bulk of uninteresting baseline.
- **Cons.** Buffer cost per emitter; more complex; the decision is made post-hoc.

Tail-based sampling is the right shape for a trace store (Tempo / Jaeger / platform-side) and is increasingly the right shape for a log store at scale. The [OpenTelemetry tail-sampling processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor) is the collector-side implementation.

### Strategy C — differential rate by event type

Not all events are equal. `request.accepted` at 100% is wasted; `request.rejected_admission` at 100% is a feature. Author a sampling rate per `event` name:

| Event | Rate |
|---|---|
| `request.accepted` | 1% (or head-sampled 10%) |
| `request.rejected_admission` | 100% |
| `upstream.called` | 1% |
| `upstream.5xx` | 100% |
| `upstream.timeout` | 100% |
| `response.sent` | 1% |
| `response.stream_abandoned` | 100% |
| `safety.content_filter_hit` | 100% |

A table like this in the emitter configuration is the honest shape. The high-rate events are the interesting ones, which is also the ones the on-call pulls up during an incident.

### Sampling is not a cost solution by itself

If you are sampling at 1% and your log store is still over budget, the issue is not the sample rate but the per-line size — carrying full retrieval context in every log is the usual culprit. Audit log line sizes before raising sample rates; a log with 10kB of embedded retrieval context at 1% sample is more expensive than a 500-byte log at 100%.

## 5. Trace-ID correlation — making the stacks join

The payoff of the log schema from section 2 is the trace-ID correlation. Three concrete workflows the correlation enables:

### Workflow A — "logs for this trace"

An on-call looking at a trace in Tempo clicks a "logs for this trace" link; the Loki query `{service_name=~"llm-.*"} |= "trace_id=4e5b3a..."` returns every log line emitted during the request. Grafana's "Traces to logs" feature does this natively when the trace store and log store are in the same Grafana. Configure the datasource mapping once; it works everywhere.

### Workflow B — "traces for these logs"

Flip side: looking at a batch of error logs, click one to jump to its trace. The error log carries `trace_id`; the trace store opens to the specific trace. Diagnostic time drops from minutes to seconds.

### Workflow C — "metric → logs"

An SLO alert fires. The alert's annotation carries a query template for logs that contributed. The on-call runs the templated query and lands on the error logs from the window. This is the "burn-rate → root cause" bridge.

In Grafana, all three are configured through the datasource `exemplars` and `derivedFields` feature. OTel-SDK-emitted logs usually have the trace context propagated automatically if the logging library has the OTel integration (`opentelemetry-instrumentation-logging` for Python, `otelbridge` for Go, etc.). Do not rely on application code to set `trace_id` by hand — the instrumentation library should do it.

## 6. Loki and Cloud Logging — the storage integrations

The two most common log stores for an inference fleet in 2026 are Grafana Loki (self-hosted or Grafana Cloud) and the cloud-provider-native logging services (GCP Cloud Logging, AWS CloudWatch Logs, Azure Monitor Logs). Each has operational quirks worth naming.

### Grafana Loki — the operator's view

- **Index model.** Loki indexes only on a bounded label set; the log body is not indexed. This makes label discipline critical — the same discipline from chapter 02 metrics. High-cardinality labels (user id, request id, trace id) *must not* be Loki labels; they live in the log body and are matched via LogQL body matchers (`|=`, `|~`, `| json | ...`).
- **Labels you want.** `service_name`, `severity`, `region`, `tenant_id` bucketed, `surface`. **Not labels.** `trace_id`, `request_id`, `user_id`, `prompt_hash`.
- **Retention.** Configured per-tenant; rotate at the operational / error / audit role boundaries from section 1.
- **Cost model.** Ingest bytes + storage-years. Trimming log line size and sampling both have direct cost impact.
- **Query cost.** LogQL queries over large windows with body matchers scan a lot of data. Add a label match to narrow first; `trace_id` queries should always carry a `service_name` label match.

### Cloud Logging / CloudWatch / Azure Monitor — the operator's view

- **Index model.** Full-text indexing is usually enabled; queries are faster at the cost of ingest-side indexing work and higher per-byte cost.
- **Labels.** Cloud-logging platforms are more generous about free-form labels and often have higher caps on cardinality; you can carry `trace_id` as a label if the platform supports it. Still don't; the operational model generalizes less across stores.
- **Retention.** Configured per log bucket; comparable to Loki.
- **Cost model.** Ingest + storage + queries. Often bills queries separately; interactive dashboards over cloud logging can be surprisingly expensive.
- **Export surfaces.** All three have exports to GCS / S3 / BigQuery / Azure Storage. Audit-retention logs often flow to an archive bucket for compliance.

### Multi-store architectures

Large fleets often run two log stores: Loki for the high-volume operational logs (fast, cheap-per-byte) and a cloud provider's audit-logging surface for the compliance tier (slower, more durable, more auditable). The AI SRE does not need to choose one; the choice is role-appropriate.

## 7. Cost-per-log accounting

The log-store bill is one of the few observability bills that can be itemized cleanly per surface and per role. Treat it the same way you treat the cost SLI from `mod-103` chapter 04:

- **Emit a counter.** `logs_emitted_bytes_total{service_name, severity, surface, event}` — a counter of the total bytes of log data emitted, labeled on your log labels.
- **Divide by the request counter.** Compute `log_bytes_per_request` as a recording rule. Alert on `log_bytes_per_request > 50 KiB` sustained — that is a signal that someone added a log statement with embedded context and did not realize the cost.
- **Attribute by surface.** Grafana panel: cost per 1k requests, per surface, per log role. The finance conversation can look at it.
- **Budget alerts.** When monthly spend projects over budget (based on the first N days of the month), alert. The finance team's SLO can drive this, same as the request-cost SLO.

A reasonable target for a mature inference fleet is < 2% of the GPU spend going to log storage. Observability-of-the-observability is a subtle discipline; the metric on the log pipeline is how you keep it honest.

## 8. The log-pipeline reliability story

The log pipeline is itself a production dependency. If the collector / aggregator goes down, you lose the log signal; if it's slow, log lag means the on-call reads stale state during an incident. Treat with the following SLIs:

- **Log ingest lag.** End-to-end from emitter to queryable in the log store. SLO: p99 < 60s.
- **Log drop rate.** Emitted vs. ingested. SLO: < 0.1% per role.
- **Collector availability.** Uptime of the aggregator layer. SLO: better than the services it serves.
- **Log store query latency.** p95 of standard dashboard queries. SLO: < 5s.

These are fleet-internal SLIs — they do not sit on the product-facing SLO stack, but they do roll up to the SRE team's platform SLOs.

## 9. Common failure modes and the fixes

Enumerated for the sake of a review checklist:

- **Unstructured log lines.** `logger.info(f"...")` with embedded fields. Fix: switch the emitter to a structured logging library (`structlog` in Python, `slog` in Go, OTel-native loggers everywhere). Log the string message as `message` and the fields as top-level JSON.
- **Content leaking in error paths.** Provider error responses often echo the original prompt. The redactor runs on the request-side log but misses the error-side log. Fix: redact every log line that touches any response body, not just happy-path logs.
- **Missing `trace_id` on async worker logs.** The main request handler propagates the trace context; the background worker it dispatches to doesn't. Fix: context-propagation review for every `asyncio.create_task` / goroutine spawn / Celery task boundary. OTel's context propagation hooks are the standard pattern.
- **Logs with 100kB bodies from embedded retrieval chunks.** The log statement included the full retrieval context for debugging and then went to prod. Fix: a log-line-size guardrail at the emitter (warn if line > 10kB; drop if > 100kB with an "oversize_dropped" counter increment). Catches these at review time rather than at bill time.
- **Audit logs in the same store as operational logs.** Audit retention is years; operational retention is days. Mixing them means either paying years-retention on everything or losing audit records at rotation. Fix: dedicated audit-log store with its own access control and retention policy.
- **Logs that re-emit the SLO metric.** A log line per request includes `latency_ms` and someone built a dashboard aggregating logs for latency. The metric store is authoritative; the log store is per-event. Fix: use the metric store for aggregate dashboards; use logs only for the per-event forensic path.
- **Fluent Bit / Vector / Collector crash during traffic surge.** The aggregator's resource limits were set for baseline load; a 5× burst causes the collector to OOM. Fix: autoscale the collector, set an on-disk buffer so brief surges don't drop logs, alert on collector-side back-pressure.

## 10. One worked example — reading the trace-log-metric join

Concrete workflow. An on-call sees the SLO burn alert: "TTFT p95 breached on `chat` surface over the last 15 minutes."

1. Open Grafana SLO panel. TTFT p95 jumped from 480ms to 1800ms at `09:42`.
2. Open Dashboard B (engine state) from chapter 02 section 5. Queue depth climbing; cache-hit rate down from 68% to 31%.
3. Pivot to logs: query Loki with `{service_name="vllm-worker", surface="chat"} | json | severity="WARN"` over the 09:40–09:50 window. See a run of `event="cache_eviction"` logs with `reason="memory_pressure"`.
4. Click one of the warn logs; Grafana opens the trace for that log line. See the full request lifecycle; see the specific prompt that triggered the eviction (hash matches an unusual long prompt from a specific tenant).
5. Query logs again, filtered by the tenant hash from the trace: `{service_name="llm-gateway"} | json | tenant_id="sha256:7f9b..."` over the last 2 hours. Tenant has 3× its normal traffic; all long prompts.
6. Pivot to DCGM dashboard from chapter 03 — HBM pressure up but not pathological. The eviction is engine-level, not hardware-level.
7. Decision: admission-control rate-limit this tenant's prompts above a length threshold; coordinate with product; document in incident post-mortem.

Elapsed time: ten minutes, from page to mitigation. The join between the trace / metric / log stacks is what makes that time possible; without it, the same incident is a half-day engineering escalation.

## Summary

- **Three log roles.** Operational, error, audit. Each with its own retention, redaction, sampling, and store.
- **Structured schema.** JSON (or OTel log records) with top-level fields, correlation fields (`trace_id` / `span_id`), `gen_ai.*` fields for LLM metadata, content fields only under the redaction policy.
- **Redaction at emission.** No raw content by default; when enabled, the redaction pipeline runs before the log line leaves the process. Hash-only is the compliance-friendly default.
- **Three sampling strategies.** Head-based (decide early), tail-based (decide late, keep errors + slow), differential per event type. Always emit errors and audits regardless of sample decision.
- **Correlation workflows.** Logs-for-trace, traces-for-logs, metric-to-logs. Grafana's datasource mapping + OTel context propagation + the `trace_id` in every line.
- **Loki vs. cloud logging** — Loki indexes labels only; cloud stores index body too. Label discipline matters as much as on metrics.
- **Cost-per-log accounting.** Emit bytes counter; recording rule for bytes/request; budget alerts. Target < 2% of GPU spend.
- **Log pipeline is a production dependency** with its own SLIs — ingest lag, drop rate, collector availability, query latency.
- **Common failure modes** — unstructured lines, leaking in error paths, missing `trace_id` in async workers, oversized lines, mixed-role stores, aggregator crashes under surge.

The final chapter maps the peer boundaries — engine metric ownership (inference-engineer), kernel-level tracing (performance peer), application-side eval overlay (eval peer) — so the observability work routes clearly across the role ladder.
