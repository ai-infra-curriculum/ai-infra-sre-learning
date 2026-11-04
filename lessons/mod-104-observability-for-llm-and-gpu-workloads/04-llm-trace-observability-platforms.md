# LLM-Trace Observability Platforms — Langfuse, Phoenix, LangSmith

## Why this chapter exists

Chapter 01 produced well-shaped OpenTelemetry spans; chapter 02 gave them a metric layer; chapter 03 added the GPU signal. With just those three, you have a classical observability stack rebased for LLMs: Prometheus + Tempo (or Jaeger) + DCGM + Grafana. For many fleets that is enough — the SLO stack queries cleanly, the trace store answers the "show me the request that burned" question, and incident flow is coherent.

For other fleets it is not enough. The questions that an OTel-only stack does not answer well:

- "Of the sampled prompts whose trace we have, which ones triggered the hallucination judge to fire? Show me the response content next to the retrieved context."
- "Compare the prompt + response distribution before and after the 09:00 release — is the completion distribution noticeably different?"
- "Which of the last 10k production prompts are candidates for the next eval set? Who labelled the shortlist?"
- "On a prompt-engineering change proposed by product, what is the pass-rate delta against the sampled production set? Can I see every disagreement between the two versions side by side?"

These are *application-side* questions — about prompt and completion content, about evaluator verdicts, about human-in-the-loop labelling, about prompt / model / retrieval versioning across time. They sit above the OTel span surface and below the eval peer's offline pipelines. A dedicated LLM-trace observability platform is the layer that connects them.

This chapter surveys the three platforms most commonly reached for — **Langfuse** (open-source + commercial), **Arize Phoenix** (open-source), and **LangSmith** (commercial, from LangChain) — and gives you a decision frame for when to adopt a dedicated platform and when to stay with OTel-only. It is deliberately not a vendor comparison matrix; the three platforms have real feature overlap, each borrows OTel as a wire format, and the specific feature parity drifts. The goal is the shape of the decision, not a product endorsement.

## 1. What a dedicated LLM-trace platform is for

The common superpower across Langfuse, Phoenix, and LangSmith — the thing OTel-only doesn't give you — is a trace view that is *content-centric*, not *span-topology-centric*.

An OTel UI (Jaeger, Tempo, Grafana Traces) renders a span tree with attributes. You see "this span took 1.4s, had attribute `gen_ai.request.model=gpt-4o-mini`, child span took 180ms." You can expand attributes. What you cannot do well is browse 500 production conversations and read the actual chat content.

An LLM-trace UI, by contrast, renders:

- **Conversation view.** For a chat-style trace, the prompt / completion content as a readable conversation, including the multi-turn structure.
- **Tool-call and retrieval detail.** Which tools fired, which documents came back from retrieval, what the tool's return value was. All inline.
- **Score overlay.** Judge / eval scores shown per-turn — hallucination, toxicity, format conformance, user feedback — as a right-hand sidebar with the pass/fail and the rationale.
- **Prompt and model version.** Which prompt template version, which model id, which retrieval index version generated this response.
- **Datasets and annotation.** "Add this trace to the eval set," "label this one as a known failure," "promote this to a regression test."
- **Cross-run comparison.** Two versions of the same prompt against the same inputs, diffed side by side.

Underneath, most platforms accept OpenTelemetry spans as input — they *extend* the OTel data model, they do not replace it. The platform adds application-level affordances on top of the trace surface the AI SRE already emits. In other words, you can run the OTel pipeline *and* the dedicated platform; the dedicated platform is a specialized consumer.

### Why the SRE should care (even though this is application-shaped)

Four operational reasons the AI SRE is a stakeholder here even though the eval peer or the application team is the primary user.

1. **Incident root-cause is faster with content visible.** When the hallucination SLI burns, the on-call needs to see failing prompts next to passing prompts, with the retrieval context inline. OTel trace view shows attributes; LLM-trace platform shows the actual text. Minutes-to-diagnose drops.
2. **Judge-drift detection relies on the platform's score store.** The eval peer's judge verdicts are time-series data; the LLM-trace platform usually exposes them to the metric stack. Without it, the signal goes to a bespoke pipeline that doesn't compose with the SLO stack.
3. **Prompt + model versioning is reliability metadata.** Rollback decisions from `mod-103` chapter 05 need to know "which prompt version was serving when the burn started." The trace platform stores this cleanly; OTel has to encode it as a span attribute convention you haven't standardized.
4. **The dataset pipeline for release gating lives in the platform.** The release-gate eval suite from `mod-103` chapter 07 is seeded from production samples; the platform is where those samples are browsed, filtered, and curated. If the AI SRE can't see what's in the gate, the SRE can't defend the release-gate contract.

The platform is an eval-engineer *primary* tool; it is an AI-SRE *shared* tool.

## 2. The three platforms — thumbnail each

Each of the three has a clear design center and a clear usage profile. Read the primary documentation at the links; the summaries below are orientation, not a replacement.

### Langfuse ([langfuse.com](https://langfuse.com), [GitHub](https://github.com/langfuse/langfuse))

- **License.** Core is MIT-licensed open source. Cloud-hosted commercial tier exists; self-hosted via Docker / Helm is a supported path.
- **Design center.** Full LLM-engineering platform — traces, prompt management, evaluations, playground, LLM-as-judge pipeline, A/B tests, dataset management. Arrives feature-complete rather than "add a plugin per capability."
- **Ingest.** Native SDK (Python, TypeScript); accepts OpenTelemetry spans via the [OpenTelemetry endpoint](https://langfuse.com/docs/opentelemetry/get-started); has integrations with LangChain, LlamaIndex, OpenAI SDK, Anthropic SDK.
- **When it earns its keep.** Teams that want one tool across application observability + prompt management + eval orchestration without stitching multiple services. The self-host story is strong and compliance-friendly.
- **Trade-off to be aware of.** It is both an observability tool and a dev-workflow tool. The dev-workflow surfaces (prompt playground, prompt versioning UI) have an opinion about how prompts are authored; if your team has already standardized a different prompt-management flow, the integration is lighter but still meaningful.

### Arize Phoenix ([phoenix.arize.com](https://phoenix.arize.com), [GitHub](https://github.com/Arize-ai/phoenix))

- **License.** Elastic License 2.0 (open-source-ish; source-available with specific restrictions). Self-host Docker / Helm; also hosted as part of the Arize AX commercial product.
- **Design center.** Research-flavoured observability — originally focused on embedding-space analysis and drift detection, expanded into LLM traces, evals, and dataset curation. Strong on the "look at embeddings across time" use case.
- **Ingest.** Native SDK; accepts OpenTelemetry spans via OTLP — Phoenix is particularly OTel-first in its data model. Integrations with the major framework SDKs.
- **When it earns its keep.** Teams that care about embedding-drift analysis, retrieval evaluation, and a notebook-driven workflow (Phoenix has a Jupyter-friendly in-memory mode). The link to Arize AX makes it a bridge to a full MLOps platform if you later go commercial.
- **Trade-off to be aware of.** The feature surface for prompt management is lighter than Langfuse's. Pair it with a separate prompt-management tool if that is a workflow you need.

### LangSmith ([smith.langchain.com](https://smith.langchain.com), [docs](https://docs.smith.langchain.com))

- **License.** Commercial only (SaaS and self-hosted enterprise). No open-source edition.
- **Design center.** The LangChain-native observability and eval platform. Tightest integration with the LangChain ecosystem; works well as a general-purpose LLM observability tool for non-LangChain stacks but was designed-from with LangChain in mind.
- **Ingest.** Native SDK; accepts OTel; special-case integration with the LangChain / LangGraph runtimes.
- **When it earns its keep.** Teams whose application is already built on LangChain / LangGraph, where the native-integration depth pays off. Also useful for teams that want a fully managed SaaS and are comfortable with the licensing.
- **Trade-off to be aware of.** No open-source core; vendor-lock is real. Data residency and self-host become a procurement discussion rather than a Dockerfile.

<!-- needs-research: feature parity across the three platforms changes often; verify current capabilities (particularly around judge-model orchestration, cost tracking, and OTel compatibility) directly from each vendor's documentation at adoption time. -->

### Honourable mentions

- **Weights & Biases Weave** — tightly integrated with the broader W&B platform; good if your ML team is already W&B-native.
- **Helicone** — proxy-style LLM observability; simple to drop in as a reverse proxy; narrower feature surface than the three above.
- **Opik (Comet)** — open-source LLM-trace tool from Comet; growing feature surface.

## 3. Dedicated platform vs. OTel-only — a decision frame

The question "do I need a dedicated LLM-trace platform?" has five axes. Each axis gives a push or a pull; two or three pulls is a signal to adopt.

### Axis 1 — content-centric workflow frequency

Does your incident, debugging, and release-gate flow regularly require *reading the conversation content*?

- **Push toward OTel-only.** You are serving stateless completions where the content is bulk data (embeddings, batch summarization) and nobody ever needs to read a specific response. Attribute-level views are enough.
- **Pull toward dedicated platform.** Chat surfaces, tool-calling workflows, agentic workflows — any case where "which specific response went wrong" is a question an on-call will ask weekly.

### Axis 2 — eval and judge pipeline maturity

How much judge / eval work is your team doing?

- **Push toward OTel-only.** No systematic eval pipeline yet; quality monitoring is ad-hoc. Adopting a platform without using the eval features is overfitting the tooling to the workflow.
- **Pull toward dedicated platform.** You have eval sets, LLM-as-judge pipelines, release-gate eval suites, human-in-the-loop labelling. The platform's dataset + annotation surface is where this work lives.

### Axis 3 — prompt management ownership

Does someone own prompt versioning as a first-class artifact?

- **Push toward OTel-only.** Prompts live in code; version-control is git; promotion is a release. No dedicated prompt-management workflow is in play.
- **Pull toward dedicated platform.** Product team wants to iterate on prompts without going through a code deploy; prompts are versioned, tagged, A/B tested. Langfuse and LangSmith both have strong prompt-management surfaces.

### Axis 4 — self-host and compliance posture

Do you need to self-host the content store?

- **Push toward OTel-only.** You have strict data-residency rules and your existing Loki / Tempo deployment is already trusted by your compliance team. Adding a new content-storing SaaS is a procurement lift.
- **Pull toward dedicated platform, but filtered.** Langfuse and Phoenix both self-host. LangSmith self-host is enterprise-tier. If the self-host story is non-negotiable, the choice narrows.

### Axis 5 — team scale and surface area

How many LLM surfaces are you operating?

- **Push toward OTel-only.** One or two surfaces, small team, incidents measured in single digits per month. The overhead of adopting a platform outpaces the payoff.
- **Pull toward dedicated platform.** Five or more product surfaces, dedicated eval team, incidents measured in dozens per month. The content-centric workflow is where incident time is spent.

### Composite outcome

- **0–1 pulls.** Stay OTel-only. Add value by investing in the OTel span quality, Loki redaction pipeline, and Grafana dashboards.
- **2–3 pulls.** Adopt a dedicated platform. The choice between the three narrows on self-host / open-source / LangChain-native preferences.
- **4–5 pulls.** Adopt now; the pain of not adopting is already showing up as incident-response latency.

Revisit the frame every six months. The pulls change as the surface grows.

## 4. Integration shape — how an SRE wires it up

Regardless of platform, the integration pattern is roughly consistent. The platform is a *consumer* of the trace stream; the AI SRE is a *producer* on the OTel side.

### Shape A — platform native SDK, parallel to OTel

Your application uses the platform's native SDK *and* the OTel SDK. The native SDK sends content-rich traces to the platform; the OTel SDK sends attribute-level spans to Tempo / Jaeger / your metrics backend.

- Pro: platform's best-case ingest; fewer translation surprises.
- Con: double-instrumentation; two SDKs to maintain; risk of drift between what each emits.

### Shape B — single OTel SDK, platform ingests via OTLP

Your application uses OTel only. An OTel exporter is configured to send spans to the platform's OTLP endpoint in addition to your main OTel backend.

- Pro: single source of truth; one SDK; the convention from chapter 01 is the schema.
- Con: platform-specific affordances (score attachment, dataset curation) may require extra attribute conventions beyond `gen_ai.*`.

Shape B is the direction the ecosystem is moving; the OTel GenAI conventions are explicitly designed to let dedicated platforms consume the same spans the SRE's main trace backend consumes. When choosing a platform, weight Shape-B maturity heavily.

### Shape C — proxy-based (Helicone-style)

A reverse proxy sits between your application and the LLM provider; the proxy observes the request and emits traces to the platform.

- Pro: zero application-side instrumentation.
- Con: only sees what goes through the proxy; agents, tool-callouts, retrieval all become additional integrations; the proxy itself is a reliability surface you now own.

Shape C makes sense for a small starting footprint or a fleet where application-side instrumentation is politically expensive. For an AI SRE operating at level 35, Shape B is usually the honest target.

## 5. What the SRE owns in a platform-adopting fleet

Once a dedicated LLM-trace platform is in the stack, the ownership split with the eval peer and the application team becomes:

- **AI SRE owns.** The OTel instrumentation that feeds the platform. The platform's deployment and reliability (if self-hosted). The retention policy on content-containing traces, aligned to the PII policy from chapter 01. The observability-side alerting on the platform's health (is ingest healthy? is the UI up?). The cost accounting on the trace volume.
- **Eval peer owns.** The judge pipelines that emit scores into the platform. The datasets and eval sets curated in the platform. The release-gate eval suites that run against the platform's dataset surface. The quality of the scores themselves.
- **Application team owns.** The prompt templates and their versioning. The dev-workflow use of the playground / preview features. The feedback loops from product analytics into the trace platform.
- **Shared.** The retention policies, the access control (who sees content), the data-subject-access-request workflow for a user's traces, the audit trail on content viewing.

The AI SRE is the natural owner of the platform's *reliability* even when the eval peer is the primary user. Treat the platform as a production dependency with its own SLOs — "trace ingest lag p99 < 30s," "search queries p95 < 2s," "UI availability monthly" — and operate it accordingly.

## 6. The traps and the costs

Adopting a dedicated platform is not free. The most common failure modes:

- **Double-schema drift.** Your OTel conventions and the platform's native conventions name the same thing differently; your dashboards show one number, the platform shows another; the SLO burn and the platform's view disagree. Normalize early; standardize on `gen_ai.*` and translate at the edge if the platform's native fields differ.
- **Content leakage via the platform.** The platform stores content. If the retention policy, access controls, and audit trail are not aligned with the policy from chapter 01, the platform becomes the easier exfiltration path. Treat the platform's content store with the same discipline as the Loki content store.
- **Over-reliance on the platform for incident response.** If the SLO alerts trigger on platform-emitted metrics and the platform goes down during the incident, you have no signal. Keep the Prometheus / Grafana / Tempo stack as the *primary* signal source; the platform is an enricher.
- **Vendor lock on the content history.** If you run on a commercial-only platform and decide to migrate, the historical trace content is in a proprietary schema. Export cadence matters; self-host is easier to migrate.
- **Cost surprises on the ingest side.** Platforms meter on trace volume, score volume, or active users. Chat-style applications with sampled-and-raw content capture can produce surprisingly large ingest. Model the cost at expected and surge QPS before committing; set a quota in the exporter so the platform does not take down your budget.
- **Prompt-management workflow collision.** The platform's prompt-management UI is a workflow choice. If product and engineering have not agreed on who owns prompts, the platform's UI becomes a political artifact. Decide ownership first; configure the platform to match.

## 7. A short worked example — picking between the three

Concrete illustrative scenario (not a real case):

A level-35 AI SRE is operating an internal LLM gateway serving three product surfaces: a chat support product, a RAG-based documentation assistant, and a code-completion feature. The eval team has four engineers, a mature judge-model pipeline, and is adding a release-gate eval suite. Product is beginning to iterate on prompts weekly. The platform runs in the company's own AWS and the security team requires self-host for anything that touches customer content. The application is pure OpenAI SDK + a few provider abstractions — no LangChain.

Walking the axes from section 3:

- Axis 1 (content workflow) — **pull**. Chat and RAG both; the on-call reads conversations weekly.
- Axis 2 (eval maturity) — **pull**. Mature pipeline; needs a dataset surface.
- Axis 3 (prompt management) — **pull**. Weekly iteration; version-control-in-code is losing.
- Axis 4 (self-host) — **pull-and-filter**. LangSmith self-host is enterprise-tier; narrows to Langfuse or Phoenix.
- Axis 5 (scale) — **pull**. Three surfaces; eval team growth.

Five pulls, with a filter at axis 4. Between Langfuse and Phoenix, the deciding factor is the prompt-management feature depth (favouring Langfuse) vs. the embedding-drift analysis depth (favouring Phoenix). Since the eval team is already building its own embedding-drift workflow, Phoenix's advantage is diluted; Langfuse's prompt-management is the active pain point. Decision: Langfuse, self-hosted, deployed under the SRE's own SRE-platform SLOs.

The pattern of the analysis is the point: walk the axes, filter on constraint (self-host), decide on marginal advantage.

## 8. What to measure after adoption

Once adopted, the AI SRE owns these ongoing measurements of the platform itself:

- **Ingest lag.** Time between span emission and span queryable in the platform. SLO: p99 < 60s.
- **Ingest drop rate.** Spans emitted vs. spans ingested. SLO: drop rate < 0.1%.
- **UI availability.** User-facing availability of the platform UI. SLO: whatever the eval team needs to run their workflow — typically monthly 99%.
- **Trace / score retention compliance.** % of traces that have not exceeded the configured retention window. Should be 100% by construction; alert on exceptions.
- **Content-view audit volume.** Count of human views of raw content, by role. Not an SLO; a security metric.
- **Cost.** Monthly spend on the platform (self-host infra or SaaS bill), divided by SLO-covered request volume. Flag unexpected growth to the cost-SLO conversation in `mod-103` chapter 04.

The platform is now a production dependency. Instrument it with the same rigor as the rest of the stack.

## Summary

- **Dedicated platforms extend OTel, they don't replace it.** The AI SRE instruments with `gen_ai.*`; the platform consumes.
- **The superpower is content-centric views.** Conversation, retrieval context, tool calls, scores, versioning, datasets — all in one UI.
- **Three primary platforms.** Langfuse (open-source, broad), Phoenix (open-source, embedding-strong), LangSmith (commercial, LangChain-native). Pick on self-host / eval-maturity / prompt-management needs.
- **Five-axis decision frame** — content workflow, eval maturity, prompt management, self-host posture, scale. Two-three pulls to adopt, four-five to adopt urgently.
- **Integration shape B** — single OTel SDK with a platform OTLP ingest — is the honest target.
- **The AI SRE owns the platform's reliability.** Ingest lag, drop rate, UI availability, retention compliance, cost. The platform is a production dependency.
- **Traps.** Double-schema drift; content leakage via the platform; over-reliance in incidents; vendor lock; cost surprises; prompt-ownership politics.

The next chapter composes the signal dimension — drift, data quality, and judge-model behavior — into the SRE's alerting surface so drift incidents page the on-call, not just the ML team.
