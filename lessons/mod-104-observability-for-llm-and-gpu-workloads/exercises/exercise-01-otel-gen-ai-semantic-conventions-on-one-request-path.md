# Exercise 01 — OTel GenAI Semantic Conventions on One Request Path

**Estimated effort:** 3 hours · **Output:** working instrumentation + a short Markdown write-up (~1000 words)

## Objective

Instrument one real LLM request path, end-to-end, with the OpenTelemetry `gen_ai.*` semantic conventions from chapter 01. Produce traces in a local OTel-compatible backend (Jaeger / Tempo / Grafana) that a reviewer can open, see the span tree, and confirm that every required attribute is present and well-shaped. Attach a short write-up that names the PII-capture mode chosen, the streaming-span lifecycle decision, and the convention-version pinning.

You pick the workload in this exercise and carry it through the module. Pick one of:

- A chat completion against a self-hosted vLLM, SGLang, or Ollama endpoint.
- A chat completion against a hosted provider (OpenAI, Anthropic, Vertex, Bedrock) using a small-cost model (`gpt-4o-mini`, `claude-3-5-haiku`, etc.).
- A RAG pipeline where retrieval + generation happen together.

The exercise is identical in shape across all three.

## Prerequisites

- Chapter 01.
- Familiarity with the [OTel Python SDK](https://opentelemetry.io/docs/languages/python/) or an equivalent language SDK.
- A local OTel trace backend you can query. [Jaeger-all-in-one](https://www.jaegertracing.io/docs/latest/getting-started/) or [Grafana Tempo + Grafana](https://grafana.com/docs/tempo/latest/) both work; Docker-Compose stacks for either are 30-minute lifts.

## Problem statement

Stand up a minimal application that:

1. Accepts a chat prompt (CLI argument, HTTP request, or notebook cell — pick what is cheapest for you).
2. Sends it to the chosen LLM endpoint.
3. Streams the response back.
4. Emits OpenTelemetry spans for the full lifecycle that conform to the GenAI semantic conventions.

Output artifacts:

- **Source code** of the instrumented application (one file is fine; two to five files is also fine).
- **A sample trace** — produce at least three runs: (a) a short successful completion, (b) a client-cancelled stream, (c) a completion that hits the max-tokens length cap. Export the trace JSON (`curl` against the backend's trace API or a Grafana export) and commit alongside the code.
- **A Markdown write-up** (~1000 words) covering the decisions enumerated below.

## Requirements

### Span structure

Produce a span tree with at least two spans:

- A **client span** — kind `CLIENT`, named per the OTel convention `"<operation> <model>"`, carrying the client-side request attributes.
- An **inner span or span event** — representing the streaming completion. Either a child span parented through context propagation (if your application has a gateway separate from the SDK call) or span events on the client span for `first_chunk_received` and `stream_closed`.

### Required attributes

On the client span, all of the following must be present (or justified as N/A in writing):

- `gen_ai.operation.name`
- `gen_ai.system`
- `gen_ai.request.model`
- `gen_ai.request.max_tokens`
- `gen_ai.request.temperature`
- `gen_ai.response.id`
- `gen_ai.response.model`
- `gen_ai.response.finish_reasons`
- `gen_ai.usage.input_tokens`
- `gen_ai.usage.output_tokens`

### Streaming lifecycle

The span must stay open for the duration of the stream. TTFT is recorded as (first-chunk-timestamp − span-start-time); either via a server-timing attribute (`gen_ai.server.time_to_first_token`) or as a span event timestamp that can be subtracted at query time. Document which you chose and why.

### Content capture mode

Pick one of the three modes from chapter 01 section 5:

- **Off** — no content attributes emitted. Still emit `prompt.content.hash` so forensic matching is possible.
- **Redacted** — content passed through a redaction pipeline. For the exercise, use a minimal regex-only pipeline (`email` + `phone number` + `URL-with-credential`); you are not expected to deploy Presidio for this exercise unless you want to.
- **Sampled-and-raw** — raw content captured at a configured fraction (e.g., 10%). Flag the sampled mode on the span.

Justify your choice in the write-up. Note: the default for anything that might ship into production is Off or Redacted.

### Convention version pinning

Record in the write-up:

- The OTel GenAI convention version you instrumented against. (As of 2026, this is the version string from the semantic-conventions repository's `gen-ai` directory.)
- The OTel SDK versions in use (Python / Go / JS / etc.).
- The expected compatibility horizon — "when does this instrumentation break?"

### Normalization

If your chosen provider SDK emits its own schema (OpenAI SDK, Anthropic SDK), explicitly translate to the `gen_ai.*` names. Do not emit both a provider name and the standard name for the same attribute — pick one shape at the emitter.

## Starter guidance

- The OTel Python SDK's [auto-instrumentation libraries](https://pypi.org/project/opentelemetry-instrumentation/) do not yet all emit the `gen_ai.*` names natively; you may need to hand-instrument or extend. This is expected; writing the instrumentation is the exercise.
- For the Jaeger / Tempo backend, use the [OTel Collector](https://opentelemetry.io/docs/collector/) between your application and the backend. This mirrors a production topology and gives you the shape of the pipeline.
- For streaming, the OpenAI Python SDK exposes the stream as a generator; wrap the generator in a context-managed iterator so the OTel span stays open until the generator's `StopIteration`. Reference: look at how [openai-python](https://github.com/openai/openai-python) handles streaming and layer your instrumentation around it.
- For the client-cancelled-stream run, break out of the stream iteration early and ensure the span records `gen_ai.response.finish_reasons = ["error"]` or an appropriate span status. The span must not be "successful" if the stream did not complete.
- The hash field (`prompt.content.hash`) is `sha256(prompt_text.encode("utf-8")).hexdigest()` — commit the hash emission even when content is off, so the write-up can speak to the forensic workflow.

## Acceptance criteria

- **Working code.** A reviewer can clone the repo, run `docker-compose up` (or equivalent), run the application against a configured endpoint, and see traces in Jaeger / Tempo / Grafana.
- **Three runs exported.** Short-success, cancelled-stream, length-cap. Each shows the expected attribute set on the span.
- **All required attributes present** on the short-success trace. Any missing attribute is explicitly N/A in the write-up with a reason.
- **Streaming lifecycle is correct.** The span's start and end bracket the stream; TTFT is recoverable from the trace.
- **Content mode is explicit.** One mode is active; the write-up defends it.
- **Convention version is pinned.** Written in the README; a reviewer knows which version of the convention the instrumentation targets.
- **Reviewer can resolve** "which attribute is `gen_ai.response.finish_reasons` on a length-cap trace and what does it mean for my TPOT SLI?" without asking.

## Stretch goals

- **Gateway + engine propagation.** Add an intermediate HTTP server (gateway) that forwards to the LLM endpoint, and propagate the OTel context through the HTTP headers. Verify the two spans (gateway and engine) share a `trace_id`.
- **Dual export.** Configure the OTel Collector to export to two backends simultaneously (e.g., Jaeger + a Langfuse / Phoenix local instance from chapter 04's exercise 04). Verify the same spans show up in both.
- **Presidio redaction.** Replace the regex redaction with a Presidio-backed NER pipeline. Measure the end-to-end latency overhead; document it. The exercise is explicitly not to tune the NER; it is to see what the latency cost of production-grade redaction looks like.
- **Multi-turn conversation.** Extend to a multi-turn chat where the second request's trace carries the first request's trace as a span link. Document the schema choice.
