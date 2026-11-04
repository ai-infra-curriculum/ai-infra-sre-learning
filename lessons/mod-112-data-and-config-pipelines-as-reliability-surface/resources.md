# Resources for mod-112-data-and-config-pipelines-as-reliability-surface

Curated references for the four pipeline surfaces this module writes a reliability contract on. Prefer the primary docs first; the papers and surveys are listed to anchor the specific claims the chapters make.

## Prompts and configs (chapter 01)

### Prompt-management platforms

- [Microsoft Prompt flow](https://microsoft.github.io/promptflow/) — prompt-engineering platform with versioned prompt artifacts and evaluation integration; open-source core, Azure ML integration.
- [LangSmith prompt management](https://docs.langchain.com/langsmith/create-a-prompt) — LangChain's commercial prompt registry and evaluation surface; content-addressed prompt versions with role-based review.
- [PromptLayer](https://docs.promptlayer.com/) — prompt-management and observability layer; emphasises prompt versioning, request tagging, and A/B evaluation.
- [Agenta](https://docs.agenta.ai/) — open-source LLMOps platform with prompt versioning, evaluation, and deployment; useful as a self-hostable reference.
- [Helicone](https://docs.helicone.ai/) — LLM observability and prompt-management platform; strong on request-level tagging and cost / latency attribution.
- [Promptfoo](https://www.promptfoo.dev/docs/) — prompt- and model-evaluation CLI; useful for CI-integrated prompt regression testing.

### Feature flags and runtime configuration

- [OpenFeature specification](https://openfeature.dev/specification/) — vendor-neutral feature-flag specification. Useful when the function wants flag semantics but a vendor-portable interface.
- [LaunchDarkly feature management guide](https://docs.launchdarkly.com/) — reference for a commercial flag platform; describes staged rollouts, user-targeting rules, and audit logging patterns.
- [Flagd](https://flagd.dev/) — OpenFeature-compatible flag service maintained under CNCF. Worth reading for a self-hosted substrate.

### Progressive delivery (upstream from `mod-111`)

- [Flagger documentation](https://docs.flagger.app/) — CNCF progressive-delivery controller used in `mod-111` and extended in chapter 01 for prompt / config channels.
- [Argo Rollouts documentation](https://argo-rollouts.readthedocs.io/) — Argo's rollout controller with canary / blue-green / experiment primitives.

## Retrieval (chapter 02)

### Vector-index substrates

- [FAISS](https://github.com/facebookresearch/faiss) — Meta's library for efficient similarity search; the HNSW implementation is a de facto reference.
- [Milvus](https://milvus.io/docs) — Open-source vector database; multiple index types, cluster-aware deployment.
- [Weaviate](https://weaviate.io/developers/weaviate) — Vector-native database with modular embedders and hybrid-search primitives.
- [Qdrant](https://qdrant.tech/documentation/) — Rust-written vector DB; strong on filtered search and payload indexing.
- [Pinecone](https://docs.pinecone.io/) — Managed vector-DB service; useful as a reference for operator-free retrieval.
- [pgvector](https://github.com/pgvector/pgvector) — Postgres extension for vector similarity; useful when the function prefers familiar operational posture.
- [Elasticsearch — semantic search](https://www.elastic.co/guide/en/elasticsearch/reference/current/semantic-search.html) — Elastic's semantic-search guide; useful for hybrid BM25 + dense retrieval.

### Embedding models and rerankers

- [Cohere Rerank](https://docs.cohere.com/docs/overview-rerank) — Managed re-rank service; frequently cited re-rank-latency reference point.
- [Voyage AI documentation](https://docs.voyageai.com/) — Embedding and re-rank services; useful when comparing provider-pin options.
- [BGE / BAAI models on Hugging Face](https://huggingface.co/BAAI) — Open-source embedding and re-rank models with published benchmarks.
- [Jina AI embeddings](https://jina.ai/embeddings/) — Open-source embedding model family with multilingual coverage.

### Retrieval evaluation

- [BEIR](https://github.com/beir-cellar/beir) — Benchmark suite for retrieval; the canonical place to read about recall-at-k evaluation methodology.
- [MTEB (Massive Text Embedding Benchmark)](https://huggingface.co/spaces/mteb/leaderboard) — Public leaderboard for embedding and re-rank models; useful for comparing candidate embedding models under a pinned methodology.
- [Malkov & Yashunin, *Efficient and Robust Approximate Nearest Neighbor Search Using HNSW Graphs*](https://arxiv.org/abs/1603.09320) — Canonical HNSW paper; read for the graph properties the function monitors for degradation.

## Feature and metadata stores (chapter 03)

### Feature stores

- [Feast documentation](https://docs.feast.dev/) — Open-source feature store; good reference for feature-definition conventions and offline / online store split.
- [Tecton documentation](https://docs.tecton.ai/) — Commercial feature platform; strong on streaming features and training-serving parity tooling.
- [Vertex AI Feature Store](https://cloud.google.com/vertex-ai/docs/featurestore) — Google's managed feature store.
- [SageMaker Feature Store](https://docs.aws.amazon.com/sagemaker/latest/dg/feature-store.html) — AWS's feature store; consult for the online / offline store split and the feature-group abstraction.
- [Databricks Feature Store](https://docs.databricks.com/aws/en/machine-learning/feature-store/) — Databricks' integrated feature store; strong integration with the lakehouse model.

### ML platform rules and methodology

- Google — [*Rules of Machine Learning* (Rule #29 on training-serving skew)](https://developers.google.com/machine-learning/guides/rules-of-ml) — The canonical practitioner's rule-set; chapter 03's parity discipline inherits from Rule #29.
- [MLOps community resources](https://mlops.community/) — Practitioner community with recorded talks on feature-store operation and parity monitoring.

### Model registries and lineage

- [MLflow Model Registry](https://mlflow.org/docs/latest/model-registry.html) — Open-source model-registry reference.
- [SageMaker Model Registry](https://docs.aws.amazon.com/sagemaker/latest/dg/model-registry.html).
- [Vertex AI Model Registry](https://cloud.google.com/vertex-ai/docs/model-registry/introduction).

## Eval sets (chapter 04)

### Eval harnesses and platforms

- [OpenAI Evals](https://github.com/openai/evals) — Open-source eval harness; read for the test-case / grader abstraction.
- [Inspect AI](https://inspect.aisi.org.uk/) — UK AISI's open-source eval framework; strong on safety and capability evaluation methodology.
- [Promptfoo](https://www.promptfoo.dev/docs/) — CLI eval harness with CI integration.
- [LangSmith evaluations](https://docs.langchain.com/langsmith/evaluation-concepts) — LangChain's platform for eval runs and judge calibration.
- [Weights & Biases Weave](https://wandb.ai/site/weave/) — Weights & Biases' evaluation and tracing platform.
- [Hugging Face `lm-evaluation-harness`](https://github.com/EleutherAI/lm-evaluation-harness) — The canonical open eval harness for LLM capability evaluation.

### Judge calibration and eval methodology

- Zheng et al. — [*Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena*](https://arxiv.org/abs/2306.05685) — The canonical reference for LLM-as-judge methodology and judge-vs-human agreement.
- Liang et al. — [*HELM: Holistic Evaluation of Language Models*](https://arxiv.org/abs/2211.09110) — Stanford's holistic evaluation framework and benchmark methodology.
- [Chatbot Arena / LMSYS Leaderboard](https://lmarena.ai/) — Live human-preference ranking; reference for human-vs-judge calibration data.

### Contamination detection

- Carlini et al. — [*Extracting Training Data from Large Language Models*](https://arxiv.org/abs/2012.07805) — Foundational reference for membership inference against LLMs.
- Oren et al. — [*Proving Test Set Contamination in Black Box Language Models*](https://arxiv.org/abs/2310.17623) — Specific methodology for detecting eval contamination in black-box models. <!-- needs-research: verify this is the canonical reference the field uses; alternative references exist and the function should cite the one most consistent with current best-practice -->

## Observability (chapter 05)

### Metrics, traces, logs

- [Prometheus best practices — metric and label naming](https://prometheus.io/docs/practices/naming/) — The function's label discipline inherits directly.
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) — The ingest substrate for pipeline spans.
- [OpenTelemetry — `gen_ai.*` semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — The vocabulary `mod-104` builds on; this module's pipeline spans extend it.
- [Grafana documentation](https://grafana.com/docs/grafana/latest/) — Dashboard authoring, panel conventions, variable mechanisms.
- [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) — Alert routing and silencing primitives.
- [Google SRE Book — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) — Multiwindow multi-burn-rate reference.

### Drift tooling (upstream from `mod-104` chapter 05)

- [Evidently](https://docs.evidentlyai.com/) — Statistical test-suite catalog for drift; open-source under Apache 2.0.
- [Arize AX / Arize Phoenix](https://docs.arize.com/) — Embedding-centric drift and trace observability.
- [WhyLabs / `whylogs`](https://docs.whylabs.ai/) — Data-profiling-first drift detection.

## Peer-boundary context (chapter 06)

- `mod-101` — Position on the ladder. The specific slugs and levels for `rag-engineer`, `ai-infra-ml-platform`, `ai-eval-engineer`, and `model-evaluation-engineer` are canonical there.
- `mod-111` chapter 08 — Peer boundaries for release engineering. The `ai-infra-mlops` and `ai-infra-inference-engineer` peer descriptions are consistent across modules.
- `mod-110` chapter 05 — Retention discipline for audit records. The pipeline-audit emission in this module targets that discipline.

## Standards and governance

- [ISO/IEC 42001:2023 — AI management systems](https://www.iso.org/standard/81230.html) — AI-specific management-system standard; useful as a governance reference for pipeline-audit retention and review cadence.
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework) — The AI RMF; useful for mapping pipeline reliability work into a broader risk-management posture.
