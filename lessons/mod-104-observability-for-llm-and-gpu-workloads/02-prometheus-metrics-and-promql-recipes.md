# Prometheus Metrics and PromQL Recipes for LLM Serving

## Why this chapter exists

Chapter 01 instrumented the LLM request path with OpenTelemetry so each request lands in the trace store as a single, consistently named span. Spans are the right surface for forensic replay — "show me the trace that caused this burn" — but they are not the right surface for aggregate SLO measurement. A million spans per hour are too many to scan at query time; the SLO query needs a pre-aggregated signal.

That pre-aggregated signal is the metric layer, and for LLM serving the right store today is Prometheus (or any OTLP-compatible metric backend consuming the same schema — Mimir, Cortex, Thanos, VictoriaMetrics). Every number in the SLO stack from `mod-103` — p95 TTFT per model, cache-hit rate, admission-reject rate, cost per 1M output tokens, availability on the streaming triad — compiles down to a PromQL expression against a well-shaped histogram or counter.

This chapter is the authoring reference for *which* metrics the fleet emits, *how they are shaped* so PromQL recipes against them are honest, and *the recipes themselves* for the SLIs from `mod-103`. The engine-exposed metrics are instrumented by the inference-engine team (vLLM, SGLang, TRT-LLM, Triton — see `mod-102` chapters 04–07); your work is to shape the gateway-side and consumption-side of the surface, to normalize provider-emitted metrics into the same schema, and to write the recipes.

The canonical references are the [Prometheus best practices for metric and label naming](https://prometheus.io/docs/practices/naming/), the [histogram and summary guide](https://prometheus.io/docs/practices/histograms/), and the [OpenTelemetry metric semantic conventions for GenAI](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/). The SLO-side recipes anchor to the [SRE Workbook on burn-rate alerting](https://sre.google/workbook/alerting-on-slos/).

## 1. The metric surface — what to emit and where it comes from

Three emitter positions on an LLM request path. Each emits a different slice; the three compose into the full picture.

- **Engine-side (vLLM / SGLang / TRT-LLM / Triton).** Owned by the inference-engine peer. Metrics live at the engine's `/metrics` endpoint; the AI SRE scrapes them. This is where TTFT / TPOT / ITL numbers from the server's point of view live, plus engine-internal state (cache hit rate, queue depth, preemptions). You do not author these; you *consume* them with recipes, and you own their SLI surface via the SLO document.
- **Gateway-side.** Your emitter. Metrics for client-perceived TTFT (engine TTFT + gateway + network), admission reject rate at the gateway, provider-side errors, routing decisions. This is where you normalize provider metrics (OpenAI / Anthropic / Vertex / Bedrock) into one schema.
- **Application / SDK-side.** Emitted by the application calling the gateway. Measures end-user latency including the client's network. Useful for root-cause splitting between "service is slow" and "network is slow"; not the primary SLI source for a server SLO.

In the ideal shape, all three emit metrics with the same `gen_ai.*` labels from chapter 01, and PromQL joins across them freely. In practice, you will have at least one emitter whose labels don't align — a provider SDK emitting `model=gpt-4o`, a gateway emitting `gen_ai.request.model="gpt-4o"`. Normalize at the gateway using metric relabeling in the Prometheus scrape config, or at the OTLP collector using an [attribute processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/attributesprocessor). Do not let misaligned labels reach the SLO query — the recipe silently drops data.

## 2. The core metric families — shape, labels, cardinality

There are five families that every LLM-serving SLO burns against. Author each as a Prometheus metric with a bounded label set, OTel semantic-convention-aligned naming, and histogram bucket choices that are honest about the latency distribution.

### Family 1 — latency histograms (TTFT, TPOT, ITL, E2E)

The naming convention from the [OpenTelemetry GenAI metric spec](https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-metrics/) is:

- `gen_ai.server.time_to_first_token` — histogram, seconds, server-side TTFT
- `gen_ai.server.time_per_output_token` — histogram, seconds, server-side TPOT
- `gen_ai.server.request.duration` — histogram, seconds, server-side E2E
- `gen_ai.client.operation.duration` — histogram, seconds, client-side total duration

When these map into Prometheus, the dots become underscores and `_seconds` is appended to the base metric name (Prometheus uses a unit suffix convention), so you will see metric names like `gen_ai_server_time_to_first_token_seconds` with the standard `_bucket` / `_sum` / `_count` histogram suffixes. ITL is per-token gap, which is not a single number per request — see section 3 for the "ITL as histogram-of-histograms" pattern.

**Label surface (bounded, cardinality-safe):**

| Label | Values | Why |
|---|---|---|
| `gen_ai_request_model` | Model id (`gpt-4o-mini`, `claude-3-5-sonnet-20241022`, `llama-3.1-70b-instruct`) | Primary SLO slice. |
| `gen_ai_response_model` | Served model id | For providers that alias; usually separate from request model only on vendor paths. |
| `gen_ai_operation_name` | `chat`, `text_completion`, `embeddings`, `tool_calls` | Workload class slice. |
| `gen_ai_system` | `openai`, `anthropic`, `vllm`, `sglang`, `bedrock`, … | Vendor-risk slice. |
| `gen_ai_response_finish_reason` | `stop`, `length`, `tool_calls`, `content_filter`, `error` | SLI slice for the length-cap SLI, the refusal SLI. |
| `surface` | Product surface id (your naming — `chat`, `copilot`, `summarize`) | SLO stack slice. |

**Do not** label on tenant, user, request id, prompt hash, or anything with cardinality in the millions. Prometheus will stay up with 2M series but the query plan will not be interactive, and the SLO query that runs every 15 seconds becomes unaffordable. Cardinality discipline is the single largest reliability risk your metric surface has.

### Bucket choice for latency histograms

Default buckets for latency histograms in most libraries (Prometheus client) are tuned to web-scale request latency — milliseconds to seconds — and they bunch at the sub-second end. For LLM latency, you want buckets that give the p95 / p99 estimator honest coverage over the range that will actually occur:

- **TTFT.** Range of interest is roughly 50ms to 60s (cold cache on a 32k-token prompt). Reasonable buckets: `0.05, 0.1, 0.2, 0.4, 0.8, 1.5, 3, 6, 12, 30, 60`.
- **TPOT.** Range is roughly 5ms/token (very small models on a fast GPU) to 500ms/token (large models under batch pressure). Reasonable buckets: `0.005, 0.01, 0.02, 0.04, 0.08, 0.15, 0.3, 0.6, 1.0`.
- **E2E.** Range is TTFT + output-length × TPOT; for streaming, can be tens of seconds. Buckets: `0.1, 0.5, 1, 2, 5, 10, 20, 60, 120, 300`.

The downside of coarse buckets is that the percentile estimator (`histogram_quantile`) is only as accurate as the bucket boundaries around the percentile. If your p95 sits in a bucket that spans 1.5–3s, your reported p95 could be anywhere in that range. Buy finer resolution in the band where the SLO target sits.

### Native histograms, where available

Prometheus [native histograms](https://prometheus.io/docs/concepts/metric_types/#histogram) — exponentially-bucketed, high-resolution, storage-efficient — are the longer-term direction and eliminate the "picking buckets" problem. OpenTelemetry's [exponential histogram](https://opentelemetry.io/docs/specs/otel/metrics/data-model/#exponentialhistogram) data type maps to this. When your Prometheus deployment supports them, use them; the bucket-tuning section above becomes a migration note rather than ongoing work.

### Family 2 — engine-internal counters (cache hit, preemption, admission reject, queue depth)

These are emitted by the engine and consumed by you. Expect names like (vLLM-aligned examples; verify against the specific engine's `/metrics` surface — see `mod-102`):

- `vllm:gpu_cache_usage_perc` / `vllm:cache_config_info` — KV-cache usage
- `vllm:num_requests_running` / `vllm:num_requests_waiting` / `vllm:num_requests_swapped` — queue surface
- `vllm:prompt_tokens_total` / `vllm:generation_tokens_total` — counters for cost and throughput
- `vllm:time_to_first_token_seconds` / `vllm:time_per_output_token_seconds` / `vllm:e2e_request_latency_seconds` — the engine-side histograms
- `vllm:request_prefill_time_seconds` / `vllm:request_decode_time_seconds` — prefill vs. decode split
- `vllm:request_preemptions_total` — preemption counter
- `vllm:spec_decode_*` — speculative-decode metrics, if enabled

<!-- needs-research: cross-check the exact current metric names and label sets emitted by vLLM / SGLang / TRT-LLM against the specific versions your fleet runs, since the engine surfaces evolve between releases. -->

For the SRE, two details matter beyond the names:

- **Cache hit vs. miss as a rate, not a gauge.** A cache-usage gauge tells you how full the KV cache is; the SLI the SLO stack wants is the *hit rate* — the fraction of requests whose prefix matched a cached one. Expose it as a counter pair (hit + miss), compute the ratio in PromQL.
- **Queue-depth distribution over time, not just the instantaneous value.** A gauge sampled every 15s hides per-request queueing. If the engine does not expose it as a histogram at admission, instrument it on the gateway.

### Family 3 — admission + routing counters (gateway-side)

Emitted by your gateway:

- `gateway_admissions_total{status, model, surface}` — accepted / rejected / deferred, with reason
- `gateway_routing_decisions_total{model, pool, decision}` — which pool the request was routed to
- `gateway_upstream_errors_total{model, upstream, kind}` — upstream (provider or self-hosted engine) errors, by kind

Admission reject rate feeds the streaming-availability SLI from `mod-103` chapter 03. Routing-decision counters feed capacity planning from `mod-106`. Upstream-error counters feed vendor-risk dashboards from `mod-107`.

### Family 4 — cost and usage counters

`gen_ai.client.token.usage` (or the engine-side equivalent) is a counter, split by `gen_ai.token.type` = `input` / `output`. The sum over the window multiplied by the per-token price gives the cost SLI from `mod-103` chapter 04.

If you emit cost directly as a metric (in some unit of currency per token, with a per-model label), the SLO query is simpler but the pricing table has to live in the exporter — bad, pricing changes without your emission knowing. Prefer to emit raw token counters and multiply at query time against a recording rule that reads from a pricing ConfigMap.

### Family 5 — quality counters (eval overlay)

Owned by the eval peer (see `mod-103` chapter 07), consumed by you:

- `eval_judgments_total{model, surface, dimension, verdict}` — count of judge verdicts
- `eval_judgments_pass_total` — the numerator
- `eval_judgments_total` — the denominator

The ratio over a window is the quality SLI. The label `dimension` separates hallucination / toxicity / tool-call-success / format-conformance / refusal-rate.

## 3. ITL as a histogram-of-histograms — the per-token distribution

Inter-token latency is per-token, not per-request. A naive emission scheme would have your engine emit 500 histogram observations for a 500-token response, which is both cardinality-exploding and query-hostile.

The pattern that works in practice:

- The engine emits a *per-request* histogram observation of **the p99 inter-token gap within the request** — one value per request, named `gen_ai_server_itl_within_request_p99_seconds`. The SLO query then takes histogram_quantile over this to produce "the p99 across requests of the p99 within each request" — the compound statement the SLI spec from `mod-103` chapter 01 calls for.
- The engine emits a separate *stall counter* — `gen_ai_server_stall_events_total` — incremented once per request whose max inter-token gap crossed a configured threshold (e.g., 1s). The SLO query divides this by the request count to get the stall rate — the second ITL-side SLI from `mod-103`.

The first signal gives you the shape of the per-token distribution; the second gives you the unambiguous "the stream froze for the user" count. Together they cover what the SLO stack wants without exploding cardinality.

If your engine does not emit either natively, instrument on the gateway by timing successive chunks of the streaming response. The gateway-side ITL is slightly biased upward by network jitter, which is honest for a user-facing SLI.

## 4. PromQL recipes — the SLI stack, fully written out

Each recipe below is a complete PromQL expression you could paste into a Grafana panel or a Prometheus recording rule. They assume the metric naming from this chapter. Substitute your prefix if different.

### 4.1 — TTFT percentile per model, 5-minute rolling window

```promql
histogram_quantile(
  0.95,
  sum by (gen_ai_request_model, le) (
    rate(gen_ai_server_time_to_first_token_seconds_bucket{surface="chat"}[5m])
  )
)
```

Key details:

- `rate(..._bucket[5m])` turns each bucket counter into a per-second rate; `sum by (..., le)` aggregates across replicas while preserving the `le` bucket boundary; `histogram_quantile(0.95, ...)` interpolates the p95 from the aggregated buckets.
- Aggregate *before* `histogram_quantile`, never after. Averaging percentiles across replicas is statistically meaningless and gives a number that may not exist in the data.
- The window has to be long enough to collect enough samples for the percentile to be stable. On low-QPS surfaces, prefer a 30m window for the dashboard view and a shorter window for the burn-rate alert — but make sure the alert's absolute threshold accounts for sample size.

### 4.2 — Cache-hit rate per model, 5m window

Assumes the engine emits `vllm:cache_hits_total` and `vllm:cache_misses_total` counters (adapt names to your engine):

```promql
sum by (model) (rate(vllm_cache_hits_total[5m]))
/
(
  sum by (model) (rate(vllm_cache_hits_total[5m]))
  +
  sum by (model) (rate(vllm_cache_misses_total[5m]))
)
```

Operational use: a sharp drop in cache-hit rate is a leading indicator of a traffic-mix shift (new prompt distribution) or a cache-eviction incident (memory pressure, see chapter 03). Alert on `drop > 20% from the 24h baseline` rather than an absolute threshold.

### 4.3 — Admission-reject rate (streaming availability SLI)

```promql
sum by (surface, model) (rate(gateway_admissions_total{status="rejected"}[5m]))
/
sum by (surface, model) (rate(gateway_admissions_total[5m]))
```

Feeds the availability triad from `mod-103` chapter 03. Combine with the stall-rate signal (recipe 4.6) and the partial-success rate to get the full availability number.

### 4.4 — Queue depth distribution

If the engine emits queue depth as a histogram:

```promql
histogram_quantile(
  0.99,
  sum by (model, le) (rate(vllm_queue_depth_bucket[5m]))
)
```

If it is only a gauge, use a per-scrape max instead and note the limitation on the dashboard:

```promql
max_over_time(vllm_num_requests_waiting{model="..."}[5m])
```

An important subtlety: queue depth under admission control caps out at the admission-control limit. A "queue depth = 100 and admission-reject rate climbing" picture is different from "queue depth = 100 and admission-reject rate zero" — the first is capacity-bound, the second is throughput-bound. Chart both together.

### 4.5 — Preemption rate

```promql
sum by (model) (rate(vllm_request_preemptions_total[5m]))
/
sum by (model) (rate(vllm_requests_total[5m]))
```

Preemption rate > 1% is a sign that the scheduler is making forward-progress trade-offs — either batching is too aggressive for the mixed workload, or admission control is not tight enough. Diagnostically, look at it alongside the ITL stall rate (recipe 4.6) — preemption is a source of mid-stream stalls.

### 4.6 — Streaming-stall rate

```promql
sum by (surface, model) (rate(gen_ai_server_stall_events_total[5m]))
/
sum by (surface, model) (rate(gen_ai_server_requests_total[5m]))
```

The complement (`1 - stall_rate`) is the streaming-availability SLI from `mod-103` chapter 03.

### 4.7 — Prefill-decode token mix

```promql
sum by (model) (rate(vllm_prompt_tokens_total[5m]))
/
(
  sum by (model) (rate(vllm_prompt_tokens_total[5m]))
  +
  sum by (model) (rate(vllm_generation_tokens_total[5m]))
)
```

The ratio tells you whether the fleet is spending its GPU cycles on prefill (prompt tokens, compute-bound) or decode (output tokens, memory-bound). A shift from 20% prefill to 50% prefill between traffic windows means your capacity math from `mod-106` has to be re-run — the two phases scale differently.

### 4.8 — Cost per 1M output tokens (recording-rule shape)

```promql
# Recording rule: output cost rate (dollars/second)
record: llm:cost_output_dollars_per_second
expr: |
  sum by (model) (
    rate(gen_ai_client_token_usage_total{type="output"}[5m])
  )
  * on (model) group_left
    model_pricing_per_million_output_tokens / 1e6
```

Where `model_pricing_per_million_output_tokens` is a ConfigMap-driven static metric with per-model labels. The division by `1e6` converts from "per million tokens" to "per token." Query over a 24h window and multiply by 86400 to get daily cost; cost SLI from `mod-103` chapter 04 is then a percentile over daily spend vs. forecast.

### 4.9 — Worker concurrency (saturation)

```promql
avg by (model, pool) (vllm_num_requests_running / vllm_max_num_running_requests_config)
```

The ratio of currently-running to configured-max. A value close to 1.0 sustained is saturation. Combine with queue depth (4.4) for the full saturation picture — "we are at max concurrency and the queue is 50 deep" is a different operational state than "we are at max concurrency and the queue is empty" (admission control is holding).

### 4.10 — Multi-window burn-rate alert (TTFT SLI)

Following the SRE Workbook multi-window pattern, with an SLO of p95 TTFT ≤ 800ms across 28 days:

```promql
# Fast burn: 2% of budget in 1h → alert if TTFT breach rate > 14.4× the SLO allowance
(
  sum(rate(gen_ai_server_time_to_first_token_seconds_bucket{le="0.8", surface="chat"}[1h]))
  /
  sum(rate(gen_ai_server_time_to_first_token_seconds_count{surface="chat"}[1h]))
) < (1 - 0.05 * 14.4)
```

The expression reads: "the share of requests completing TTFT within 800ms over the last 1h is below what the fast-burn-rate multiplier requires." Mirror this at the medium (6h, 6× multiplier) and slow (3d, 1× multiplier) windows. The SRE Workbook burn-rate chapter is the reference for the multiplier choices; `mod-103` chapter 05 wires them into the error-budget policy.

## 5. Dashboard panels — which recipes to lay out together

Three dashboards worth authoring explicitly. Chapter 07 goes deeper into dashboard authoring for incident readiness; the panel inventory below is the metric-side slice.

### Dashboard A — "The one SLO dashboard"

One row per SLI from `mod-103`. Each row has:

- The SLI value over the rolling window (left, big number)
- The SLO target line (horizontal line)
- The 7-day history (center, time series)
- The burn-rate panel (right, three lines for fast/medium/slow burn)

Recipes 4.1 (TTFT), the corresponding TPOT / E2E percentile recipes, 4.3 (admission reject), 4.6 (stall rate), and the quality SLI recipes feed the rows.

### Dashboard B — "The engine state dashboard"

Diagnostic view — opened when the SLO burns and the on-call needs to narrow the cause. Panels:

- Queue depth (4.4) with the admission-cap line overlaid
- Cache-hit rate (4.2) with a 24h baseline
- Preemption rate (4.5)
- Prefill-decode mix (4.7)
- Worker concurrency saturation (4.9)
- TTFT bucketed by prompt-length bucket (a variant of 4.1 with `le` on `prompt_length_bucket`)

### Dashboard C — "The cost dashboard"

Owned jointly with finance. Panels:

- Cost per 1M input tokens, per model, 24h window (variant of 4.8)
- Cost per 1M output tokens, per model, 24h window (4.8)
- Daily spend vs. forecast
- Cost per successful request (cost ÷ request count, excluding errors)
- Cost by surface

## 6. Cardinality, cost, and the "don't-do-this" list

Every one of the following has broken a Prometheus deployment somewhere; enumerate them in your metric-review checklist so none of them ships from your team.

- **High-cardinality labels.** User id, request id, prompt hash, raw prompt-length (vs. a prompt-length bucket), full temperature float (vs. a bucket), region+pool+tenant exploded. A label value with more than ~1000 distinct values is a smell; more than 10000 is a bug.
- **Unbounded `finish_reason` or `error` label.** Error strings from upstream providers are often free-form; normalize to a bounded set (`timeout`, `rate_limit`, `provider_5xx`, `provider_4xx`, `client_canceled`, `other`) at the gateway.
- **Per-replica cost accounting.** The pricing join (recipe 4.8) is per-model, not per-replica. Do not label the pricing ConfigMap with pod-level labels — the cardinality explodes once you have 500 replicas.
- **Histogram with no buckets or default buckets.** Default buckets are web-latency-shaped (5ms–10s); for LLM latency, they bunch wrong. Tune per family from section 2.
- **Averaging percentiles.** If you take an average of `histogram_quantile(...)` across pods, the result has no statistical meaning. Always `sum by (..., le)` *inside* the `histogram_quantile`.
- **Sampling at the metric emission layer.** Metrics should be exact counts of what happened; sampling belongs at the trace / log layer where cardinality is an issue. A sampled counter undermines the SLO's ability to detect low-rate events.
- **Mutating a metric's label set in flight.** If a release adds a new label, the old series and the new series are different time series in Prometheus — the recording rule and alert against the old series goes quiet while the new series slowly accumulates. Treat label-set changes as breaking changes to the SLO query; coordinate with the metric owner.

## 7. Normalizing provider metrics into the schema

One of the sharper practical problems: the provider you call (OpenAI, Anthropic, Vertex, Bedrock) emits *its own* metrics at the client SDK level in its own schema, and your fleet calls both self-hosted engines and providers in the same gateway. Three options:

- **Translate at the OTel collector.** Use the [attributes processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/attributesprocessor) and the [metricstransform processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/metricstransformprocessor) to rename the provider's metric names and labels to match the `gen_ai.*` schema. Lossless; the only cost is collector configuration maintenance.
- **Translate at Prometheus scrape time.** Use `metric_relabel_configs` to rewrite labels; use recording rules to alias metric names. Cheaper to configure; less expressive than the collector.
- **Translate at the gateway.** Emit your own gateway-side `gen_ai.*` metrics for every call, regardless of upstream. More code, cleaner schema, more memorable to debug.

Pick one — mixing them silently drifts the schema. If the gateway is the point where routing happens anyway, gateway-side normalization is usually the honest choice.

## 8. Feeding the SLO stack

Every recipe in this chapter feeds a specific SLI from `mod-103`. The one-page mapping:

| SLI from `mod-103` | Recipe | Chapter / section |
|---|---|---|
| TTFT p95 per workload class | 4.1 | `mod-103` ch 01, ch 06 |
| TPOT p95 | TPOT variant of 4.1 | `mod-103` ch 01 |
| ITL p99 within request | 4.6 (with histogram_quantile) | `mod-103` ch 01 |
| Streaming stall rate | 4.6 | `mod-103` ch 03 |
| Admission-reject availability | 4.3 | `mod-103` ch 03 |
| Capped-response (length-finish) rate | counter from `gen_ai_response_finish_reason` label | `mod-103` ch 02 |
| Refusal / content-filter rate | counter from `gen_ai_response_finish_reason` | `mod-103` ch 02 |
| Tool-call success rate | eval_judgments ratio | `mod-103` ch 02 / ch 07 |
| Cost per 1M output tokens | 4.8 | `mod-103` ch 04 |
| Cost per successful request | 4.8 / request-count | `mod-103` ch 04 |

With these ten recipes wired, the SLO stack becomes a set of recording rules that compute their SLI continuously. The burn-rate alerts from recipe 4.10 fire into Alertmanager; the Grafana dashboard from section 5 is the on-call's first panel; the trace-first workflow from chapter 01 is one click away via Loki or Tempo.

## Summary

- **Metric surface** — five families: latency histograms (TTFT / TPOT / ITL / E2E), engine-internal counters (cache / queue / preemption), gateway admission + routing counters, cost / usage counters, quality counters from the eval peer.
- **Label discipline** — slice on model / operation / system / finish-reason / surface. Never on user id, request id, prompt hash, or anything with cardinality > ~1000.
- **Histogram buckets** — tune per family. TTFT 50ms–60s, TPOT 5ms–1s/token, E2E 100ms–5m. Prefer native/exponential histograms where supported.
- **ITL is a histogram-of-histograms plus a stall counter** — one per-request p99 observation feeding `histogram_quantile`, one stall counter on top of a configured threshold.
- **Ten recipes** cover the SLO stack from `mod-103`. Author them as recording rules; wire burn-rate alerts off the recording-rule output.
- **Cardinality discipline** — do not mutate label sets in place, do not label on user id, do not sample at the metric layer, do not average percentiles. The metric surface is the SLO's load-bearing floor; a broken label plan is an outage waiting.

The next chapter drops below the engine onto the GPU itself — SM activity, HBM pressure, PCIe / NVLink saturation, XID codes, thermal throttling — and authors DCGM-aware alerts that distinguish a bad GPU from a bad workload.
