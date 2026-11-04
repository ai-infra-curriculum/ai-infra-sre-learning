# Instrumenting the LLM Request Path with OpenTelemetry `gen_ai.*`

## Why this chapter exists

The SLO stack from `mod-103` named five latency SLIs, a quality dimension, an availability triad for streaming, and a cost surface. Every one of those SLIs is a number computed from *events*, and events are only useful if the schema that describes them is shared across the gateway, the engine, the trace store, the metric store, the bill-of-materials system, and the on-call. The schema that unifies them is the OpenTelemetry Semantic Conventions for Generative AI — `gen_ai.*` — authored by the OpenTelemetry Working Group and adopted by vendors from Anthropic / OpenAI SDKs through LLM orchestration frameworks to the serving engines.

This chapter is the authoring reference for *what* to put on an LLM request span, *where* to set each attribute, *how* to handle prompt / completion capture when the content itself is PII, and *how* the resulting trace composes with the metric and cost signals so a single `trace_id` lets an on-call pivot from "the SLO is burning" to the individual request that burned it.

The canonical reference is the [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — read the spec itself alongside this chapter. The spec is still at **Experimental / Development** status in the OTel stability matrix at the time of writing, which has operational consequences spelled out in section 7.

## 1. Why a shared schema matters before you instrument anything

The non-obvious trap when instrumenting an LLM surface is that the four observability silos — traces, metrics, logs, and bills — have each historically evolved their own naming. The engine team emits `vllm:*` metrics. The application team emits `llm_requests_total` from the SDK. The gateway emits `http_server_*` in `http.*` naming. The finance pipeline queries a CSV from the model provider. When the SLO burns, the on-call opens four dashboards and four schemas, and the join key is "the engineer's memory of what the gateway called the request."

The `gen_ai.*` conventions fix this by giving a single vocabulary that lives on:

- the OTel **span** for the request on both the application side and the engine side
- the OTel **metric** emitted from the engine's exporter (once the engine adopts the convention — see `mod-102`)
- the OTel **log record** emitted around the request on both sides
- the span and metric *attributes*, which are the slice keys the SLO's PromQL recipes consume

A single `trace_id` that carries from the client SDK through the gateway through the engine through the GPU-side span gives the on-call a rope to pull during an incident. The SLO query from `mod-103` is sliced on `gen_ai.request.model`; the cost accounting from `mod-103` chapter 04 is aggregated on `gen_ai.usage.*`; the quality overlay from the eval peer is tagged on the same span. One schema, one join key.

This chapter writes the schema for an LLM request path. Chapter 02 writes the Prometheus recipes that consume it. Chapter 06 writes the log discipline that correlates with it.

## 2. The anatomy of a well-instrumented request span

A request to an LLM endpoint typically fans into several nested spans — the client SDK span, the gateway's routing span, the engine-side inference span, and sometimes a KV-cache / prefix-cache span. The OTel convention is clear that the *outermost* model-invocation span is the one that carries the `gen_ai.*` attributes; nested spans inherit the trace but carry their own operational attributes.

A minimal span for a chat completion looks roughly like this (in OTel SDK pseudocode):

```python
with tracer.start_as_current_span(
    name="chat gpt-4o-mini",                          # convention: "<operation> <model>"
    kind=SpanKind.CLIENT,
    attributes={
        "gen_ai.operation.name": "chat",              # chat | text_completion | embeddings | ...
        "gen_ai.system": "openai",                    # provider ID (openai, anthropic, vllm, ...)
        "gen_ai.request.model": "gpt-4o-mini",
        "gen_ai.request.max_tokens": 512,
        "gen_ai.request.temperature": 0.2,
        "gen_ai.request.top_p": 1.0,
    },
) as span:
    response = client.chat.completions.create(...)

    span.set_attribute("gen_ai.response.id",            response.id)
    span.set_attribute("gen_ai.response.model",         response.model)
    span.set_attribute("gen_ai.response.finish_reasons",
                      [choice.finish_reason for choice in response.choices])
    span.set_attribute("gen_ai.usage.input_tokens",     response.usage.prompt_tokens)
    span.set_attribute("gen_ai.usage.output_tokens",    response.usage.completion_tokens)
```

Three things to notice about that shape before moving on.

- **Span name is `"<operation> <model>"`** — e.g., `"chat gpt-4o-mini"`, `"embeddings text-embedding-3-small"`. This is what the OTel convention recommends so dashboards that group by span name land on a readable label.
- **The kind is `CLIENT` on the application side** (your gateway or SDK calling the engine or provider). On the *engine* side, it is `SERVER` on the inference span. Same trace, two spans, two kinds.
- **The `gen_ai.response.*` attributes are set after the call returns.** For streaming responses, these are set at end-of-stream; the span stays open for the duration of the stream. See section 4.

## 3. The attribute surface — every attribute you should set, grouped

The `gen_ai.*` namespace has grown to a substantial set. The practical SRE question is not "which attributes exist" but "which attributes do I need to compute each SLI from `mod-103`." The subset below is the working minimum.

### Request attributes — invariant per request

| Attribute | Required? | Purpose |
|---|---|---|
| `gen_ai.operation.name` | Required | Operation class — `chat`, `text_completion`, `embeddings`, `generate_content`, etc. Lets dashboards group by workload type. |
| `gen_ai.system` | Required | The provider / system — `openai`, `anthropic`, `vllm`, `sglang`, `vertex_ai`, `bedrock`. Lets vendor-risk dashboards (see `mod-107`) slice provider-side regressions out. |
| `gen_ai.request.model` | Required | The model id the client *asked for*. This is the SLO slice key. |
| `gen_ai.request.max_tokens` | Recommended | Caller's ceiling on output length. Correlates with E2E latency distribution. |
| `gen_ai.request.temperature` | Recommended | Sampling temperature. Correlates with cache-hit rate and with some quality metrics. |
| `gen_ai.request.top_p` / `top_k` | Recommended if set | Sampling parameters, same reason. |
| `gen_ai.request.seed` | Recommended if set | Deterministic sampling seed, for reproducibility during incident replay. |
| `gen_ai.request.stop_sequences` | Recommended if set | Explicit stop tokens — relevant to debugging "the model never stopped" cases. |

### Response attributes — set on completion

| Attribute | Required? | Purpose |
|---|---|---|
| `gen_ai.response.id` | Recommended | The provider's response id. Critical for provider-side support tickets. |
| `gen_ai.response.model` | Required | The model id the provider *actually served*. May differ from `gen_ai.request.model` when the provider aliases (e.g., `gpt-4o` → `gpt-4o-2024-08-06`). SLO measurement should slice on the *served* model, not the asked model, or both; be explicit. |
| `gen_ai.response.finish_reasons` | Recommended | Array per choice — `stop`, `length`, `tool_calls`, `content_filter`, `error`. The tail of `length` finishes is the capped-response SLI from `mod-103` chapter 02. The `content_filter` count is the refusal/safety SLI. |

### Usage attributes — the cost + capacity surface

| Attribute | Required? | Purpose |
|---|---|---|
| `gen_ai.usage.input_tokens` | Recommended | Prompt token count. Feeds cost-per-million-input-tokens SLI. |
| `gen_ai.usage.output_tokens` | Recommended | Completion token count. Feeds cost-per-million-output-tokens SLI and TPOT measurement. |

Older spec revisions used `gen_ai.usage.prompt_tokens` / `gen_ai.usage.completion_tokens`; these are aliased in some SDKs but the current convention is `input_tokens` / `output_tokens`. If your SDK is older, map to the current names at the exporter rather than letting both live in the metric surface.

### Server-side timing attributes — the TTFT / TPOT numbers

These are set by the server-side (engine or gateway) span, not the client span:

| Attribute | Purpose |
|---|---|
| `gen_ai.server.time_to_first_token` | TTFT for this request, in seconds. The number the TTFT SLI percentile aggregates over. |
| `gen_ai.server.time_per_output_token` | TPOT for this request (average inter-token gap). Feeds TPOT SLI. |
| `gen_ai.server.request.duration` | E2E for this request on the server side. |

Client-side TTFT is also useful (it includes network + gateway time); when you measure it, set it on the client span. Chapter 02 writes the recipe.

### Tool-call attributes — only on tool-calling surfaces

| Attribute | Purpose |
|---|---|
| `gen_ai.tool.name` | The tool the model invoked. |
| `gen_ai.tool.call.id` | Provider-assigned tool call id. |
| `gen_ai.tool.type` | Tool type (function, retrieval, …). |

Each tool invocation typically becomes a child span of the request span, with its own `gen_ai.*` attributes plus the `function.*` / `http.*` attributes for the actual invocation. The structured-output-conformance SLI from `mod-103` chapter 02 is computed by comparing the tool invocation span's arguments against the declared schema.

### Prompt / completion content — the sensitive attributes

These are the two attributes that need the most authoring care, because capturing them is the difference between an observable surface and a GDPR / HIPAA incident.

The current OTel conventions capture prompt and response content via **span events** named `gen_ai.content.prompt` and `gen_ai.content.completion`, each carrying a `content` attribute. (Older drafts used span attributes `gen_ai.prompt` / `gen_ai.completion`; the span-event form is the direction the convention moved to allow structured multi-turn content.)

Section 5 is entirely about the PII controls around this. Do not emit these fields in production without reading that section.

## 4. Streaming and the span lifecycle

Request spans for streaming endpoints have a lifecycle quirk that trips up every first-time instrumentation. The call returns an iterator on the first chunk, and the span cannot be closed until the stream ends — but the OTel SDK will happily close a span whose context manager has exited, meaning the naive `with tracer.start_as_current_span(...): yield stream` loses attributes that are only known at end-of-stream (output-token count, final finish reason, TPOT).

Three options, in order of robustness:

- **Wrap the stream iterator** so the span's close is deferred until the underlying iterator raises `StopIteration` (or `StopAsyncIteration`). This is what most instrumentation libraries do internally. The span stays open for the duration of the stream; `gen_ai.server.time_to_first_token` is set when the first chunk arrives; `gen_ai.usage.output_tokens` and `gen_ai.response.finish_reasons` are set when the stream closes.
- **Emit an initial span + a terminal span.** Cheaper to implement; costs you the ability to see TTFT and total duration on the same span. Not recommended.
- **Set `gen_ai.*.first_chunk.timestamp` as a span event.** The TTFT attribute is derived from (first-chunk-event-time − span-start-time). Some trace stores compute this automatically from a well-named event.

Also — if the client cancels the stream mid-way, the span's `finish_reasons` should record that (the OTel convention uses `error` or allows a system-specific value), and the on-call should be able to slice "abandoned-vs-completed" cleanly. This is the valid-event filter for the E2E SLI from `mod-103` chapter 01 section 5.

## 5. Prompt and completion capture — the PII story

Capturing the prompt and the completion gives you the most forensically useful signal in the stack. It also means your trace store now contains everything your users typed and everything your model said — including PII, PHI, prompt injections, and anything the sales team pasted in while testing. The default must be *off*, and the opt-in must be explicit and audited.

### The three modes you have to pick between

- **Off.** The default. `gen_ai.content.prompt` and `gen_ai.content.completion` events are *not* emitted. Trace still carries all metadata; content is absent. SLOs are fully computable; forensic replay is not.
- **Redacted.** Content is emitted but passed through a redaction filter first — NER-based (spaCy / Presidio) for named entities, regex for structured identifiers (email, phone, credit card, SSN), domain-specific redactors for anything your product handles. The span event carries the redacted form, plus an attribute flagging that redaction was applied (`gen_ai.content.prompt.redacted = true`) and ideally a hash of the un-redacted form so forensic matching against a user's complaint is possible without storing the content.
- **Sampled-and-raw.** Content is emitted raw but only for a small configured fraction of requests, and only from a specific set of environments (never production, or production with explicit opt-in flags). Sampling fraction is published and auditable. This is the mode most useful for eval-overlay (chapter 04) and adversarial-input forensics; it is also the mode where an audit will go hardest.

The right policy almost always combines these: *sampled-and-raw in staging, redacted in production, off for jurisdictions where redaction is insufficient under local law.* The specific policy lives in the security-safety-compliance work from `mod-110`; this chapter wires the instrumentation side.

### The redaction pipeline, in practice

A reasonable redaction pipeline for prompt / completion capture:

1. **Structured identifiers.** Regex-based detection of emails, phone numbers, credit card PANs (Luhn-validated), SSN / tax ID patterns for jurisdictions in scope, API keys / tokens (project-specific regex), URLs with secret-looking query parameters. Replace with `[REDACTED:<type>]`. This step is fast and low-FP; run it unconditionally.
2. **Named-entity recognition.** Pass remaining content through a NER model (e.g., [Microsoft Presidio](https://microsoft.github.io/presidio/), spaCy with domain-specific entity sets) to detect person names, locations, organization names, dates of birth, etc. Replace with `[REDACTED:<entity_type>]`. This step has non-trivial cost and non-trivial false-positive rate; evaluate on your corpus.
3. **Domain-specific redaction.** Your product has entity types the generic NER does not know — customer IDs, internal case numbers, medical identifiers, educational record IDs. Add regex / dictionary pass per entity type.
4. **Hash before store.** Compute SHA-256 of the original content (post-encoding normalization) and store alongside the redacted form. Enables "user X complained about response Y; find the trace" without storing the content.
5. **Emit the redacted event.** Set the `gen_ai.content.prompt.redacted` boolean and the hash. If any step above *failed open* (regex raised, NER timed out), attribute the event accordingly — you want to be able to query "how many spans had redaction skipped and why."

The tracer SDK should run the redaction *before* the attribute is set on the span. Do not rely on the trace exporter or the trace store to redact; the span exists in in-process memory between SDK and exporter, and a crash or heap-dump exposes it.

### The audit log

Every change to the sampling fraction, every change to the redaction policy, every per-environment override — these are audit events. The audit log lives wherever your security team already stores production policy changes; the trace store references it by policy version. When a trace is viewed by a human, the view event itself is auditable. This is the compliance surface from `mod-110`; from the observability side, your job is to make it enforceable in the instrumentation, not to design the audit infra.

### The DLP sanity check

Run a Data Loss Prevention scan over a sample of your trace store, monthly. Cloud providers have off-the-shelf DLP APIs (GCP DLP, AWS Macie, Azure Purview). The check is: does your redacted trace store still contain PII? If it does, your pipeline has a gap, and you want to know about it from an automated scan, not from a user data-subject-access request.

## 6. The trace that overlays latency, cost, and quality

The payoff of all this instrumentation is that a single `trace_id` composes with the SLO stack from `mod-103`.

- **Latency overlay.** TTFT, TPOT, ITL, E2E — each is computed from the span's timing attributes. The engine-side span carries `gen_ai.server.time_to_first_token` and friends; the client span computes its own TTFT from (first-chunk-timestamp − span-start-time). The on-call looking at "why is this TTFT 2s" opens the trace, sees that the engine-side TTFT is 1.8s (so the problem is the engine, not the gateway), sees the queue depth attribute on the engine span was 42 (so the problem is admission, not the model), and routes the incident accordingly.
- **Cost overlay.** The `gen_ai.usage.input_tokens` and `gen_ai.usage.output_tokens` attributes are the inputs to the cost SLI from `mod-103` chapter 04. On a provider-based model, you also carry the vendor price per 1M tokens; on a self-hosted model, you carry the inferred cost per token from the fleet-utilization math in `mod-106`. Multiply at query time in Grafana; the cost column on the per-trace view falls out.
- **Quality overlay.** The eval peer's judge pipeline emits a `PASS` / `FAIL` / `UNKNOWN` verdict per response. Carry that on the span via a convention like `ai.eval.pass_rate` or a product-specific attribute namespace — see `mod-103` chapter 07 on the peer boundary. The on-call looking at "the quality SLI burned" opens the trace, sees the failing judge verdict, sees the specific prompt/completion content (if sampled-and-raw was on in staging, or the hash matches in production), and either escalates to the eval peer or narrows the regression to a traffic shift.

### The trace-first incident workflow

The combination of these three overlays is the trace-first incident workflow:

1. SLO fires (metric-store side).
2. On-call opens the burn panel, pivots to "sample traces that contributed to the burn," which is a Loki / Tempo / trace-store query by `gen_ai.request.model` and the time window.
3. On the handful of returned traces, the on-call sees the full span tree — gateway, engine, GPU, retrieval — with latency, cost, and quality overlaid.
4. The cause is three levels deep in a single view: "the retrieval step on this surface is 800ms slow because the vector store has a bad shard."

Without the single-schema instrumentation, step 2 is "go ask the engine team if they have a dashboard." With it, step 2 is a two-click pivot. The time-to-mitigation delta is measured in minutes, every on-call shift.

## 7. The convention is still experimental — what that costs you

The OTel Generative AI semantic conventions are at **Experimental / Development** status at the time of writing. The practical consequences:

- **Attribute names may change between SDK versions.** The `gen_ai.usage.prompt_tokens` → `gen_ai.usage.input_tokens` rename is a recent example. Pin SDK versions; pin convention version in your instrumentation review; plan a "rename sweep" as a yearly cost.
- **Not every attribute is uniformly supported across SDKs.** Instrumentation libraries from one vendor may emit attributes the other vendor's SDK does not. Normalize at a gateway-side exporter rather than hoping the two emitters align.
- **Prompt / completion content capture is actively evolving.** Whether content lives as a span attribute, a span event, or an OTel log record has moved in revisions. Codify your chosen representation in the SLI spec from `mod-103` chapter 06.
- **The exporter contract is stable, the attribute set is not.** The OTLP wire format and the SDK API are stable; the exact attribute-name surface of `gen_ai.*` is not. Treat it the way you treat a GA API with an "may-break" minor-version attribute list — upgrade on cadence, run the convention-compatibility check, and don't build downstream SLOs that *depend on* a particular pre-release attribute without a mapping layer.

Track the convention's status via the [OTel semconv GitHub repository](https://github.com/open-telemetry/semantic-conventions) — the `gen-ai` directory has the active issues and the spec stability signals.

## 8. One worked example — a chat endpoint, end-to-end

Pulling the whole chapter together, here is the end-to-end shape of a single chat completion with a tool call, as spans:

```
Trace: 4e5b3a9c... (chat, gpt-4o-mini, trace started 12:03:04.212)
│
├── [CLIENT] chat gpt-4o-mini                                    [1.42s]
│   gen_ai.operation.name       = "chat"
│   gen_ai.system               = "openai"
│   gen_ai.request.model        = "gpt-4o-mini"
│   gen_ai.request.max_tokens   = 512
│   gen_ai.request.temperature  = 0.2
│   gen_ai.response.id          = "chatcmpl-9jQ..."
│   gen_ai.response.model       = "gpt-4o-mini-2024-07-18"
│   gen_ai.response.finish_reasons = ["tool_calls"]
│   gen_ai.usage.input_tokens   = 412
│   gen_ai.usage.output_tokens  = 87
│   Events:
│     * gen_ai.content.prompt       (content=REDACTED, hash=sha256:..., redacted=true)
│     * gen_ai.content.completion   (content=REDACTED, hash=sha256:..., redacted=true)
│
├── [SERVER] chat gpt-4o-mini  (engine-side span, parented through context propagation)
│   gen_ai.server.time_to_first_token  = 0.218
│   gen_ai.server.time_per_output_token = 0.011
│   gen_ai.server.request.duration     = 1.38
│   # engine-authored attributes from mod-102:
│   vllm.cache.prefix_hit_rate = 0.73
│   vllm.queue.depth_on_admission = 8
│
└── [INTERNAL] tool call  get_weather
    gen_ai.tool.name     = "get_weather"
    gen_ai.tool.call.id  = "call_abc..."
    http.response.status_code = 200
    duration             = 0.19s
```

The on-call reading this trace sees: a 1.4-second chat request; the first token arrived at 218ms (TTFT bucket: "fine for chat"); 87 output tokens at 11ms/token (TPOT bucket: "fine"); content is redacted but hash-matchable; the prefix-cache hit rate was 73% on this request; queue depth at admission was 8; one tool call fired and returned 200. One screen, one workflow, no cross-team DMs.

Every exercise in this module composes against this shape. If this schema is not in your instrumentation, chapters 02–06 have nothing to query against.

## Summary

- **One schema across the silos.** `gen_ai.*` lives on spans, metrics, and logs — join key for SLO, cost, and quality is `trace_id` plus the shared attribute surface.
- **Minimum attribute surface.** Operation + system + request-model + response-model + finish-reasons + token usage + server timings. Set tool attributes on tool-calling surfaces. Everything else is optional, usefully optional.
- **Streaming is a lifecycle problem.** The span stays open until the stream closes; TTFT is set on first chunk, token count on last. Wrap the iterator; don't use the naive context manager.
- **Prompt and completion capture is the PII story.** Off by default. Redacted in production. Sampled-and-raw in staging with audit. DLP scan monthly to verify the pipeline.
- **The payoff is the trace-first incident workflow.** Latency + cost + quality overlaid; two-click pivot from "SLO burned" to "this trace shows the cause."
- **The convention is experimental.** Pin versions, plan a rename sweep yearly, normalize at the gateway rather than trusting every SDK.

The next chapter lifts the attributes into the metric layer — Prometheus histograms over TTFT / TPOT / ITL, cache-hit and preemption rates, and PromQL recipes that feed the SLO stack directly.
