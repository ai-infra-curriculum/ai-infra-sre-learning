# mod-112-data-and-config-pipelines-as-reliability-surface: Data and Config Pipelines as Reliability Surface

**Estimated effort:** 14 hours

## What this module is

An AI serving tier reads from more than its model. It reads from the **prompt registry**, from a **runtime configuration surface**, from the **retrieval index**, from the **feature store**, and from the **eval set** the release gate holds it against. Each of these is a pipeline. Each is a reliability surface whose failure the user feels as "the product got worse" — but whose signals the function's classical SLO stack (latency, success-rate, availability) will not catch.

This module makes those surfaces first-class. It writes the reliability contract on prompts and configs as deploy artifacts (chapter 01), on retrieval freshness and recall (chapter 02), on feature and metadata stores (chapter 03), on eval-set integrity (chapter 04), and on the single pane that reads all four (chapter 05). It closes on the peer boundaries so the function's authoring work does not drift into the peers' domains (chapter 06).

The module composes with `mod-103` (SLOs), `mod-104` (observability stack), `mod-107` (vendor / provider risk, which the embedding and judge providers sit inside), `mod-108` (incident response), `mod-110` (audit retention), and `mod-111` (the progressive-delivery stack that this module's artifacts flow through).

## Learning objectives

- Treat prompts and configs as reliability-critical artifacts — prompt-template versioning, config-change rollout discipline, prompt-rollback as a first-class reliability operation; wire prompt / config changes through the same progressive-delivery stack as model-weight changes (`mod-111`).
- Own retrieval-freshness SLOs for RAG systems — index-rebuild cadence, document-staleness bounds, retrieval-recall drift as a reliability incident, embedding-index-version-mismatch detection, re-rank-latency contribution to TTFT budget.
- Own feature-store and metadata-store reliability at the ML-platform handoff — feature-drift signals as serving incidents, feature-pipeline-stall detection, cross-training-and-serving feature-parity monitoring; interface with the `ai-infra-ml-platform` peer.
- Own eval-set reliability — eval-set drift, eval-set poisoning, eval-set rot; interface with the `ai-eval-engineer` and `model-evaluation-engineer` peers.
- Instrument data and config pipelines in the same observability stack as the serving pipeline (`mod-104`) — same Prometheus / Grafana / OTel, same alert-rule discipline — so incidents surface in a single pane.
- Cite the boundaries — retrieval-pipeline authoring is at `rag-engineer`; feature-pipeline authoring is at `ai-infra-ml-platform`; eval-set construction methodology is at `model-evaluation-engineer` / `ai-eval-engineer`.

## Chapters

1. [Prompts and Configs as Reliability-Critical Artifacts](01-prompts-and-configs-as-reliability-artifacts.md) — prompt registry, config-as-CRD, progressive-delivery hook, prompt-rollback as a first-class operation.
2. [Retrieval-Freshness SLOs for RAG](02-retrieval-freshness-slos-for-rag.md) — document-staleness and recall SLOs, index-rebuild cadence, embedding-version-mismatch detection, re-rank-latency in the TTFT budget.
3. [Feature and Metadata Store Reliability](03-feature-and-metadata-store-reliability.md) — availability / freshness / completeness / parity SLOs, drift as an incident, pipeline-stall detection, the ML-platform handoff.
4. [Eval-Set Reliability](04-eval-set-reliability.md) — drift, poisoning / contamination, rot; the quarantine and demotion actions; the release-gate contract clauses.
5. [Unified Pipeline Observability in One Pane](05-unified-pipeline-observability.md) — the signal taxonomy, the Grafana layout, the alert-rule discipline, trace correlation across pipelines.
6. [Peer Boundaries for Data and Config Pipelines](06-peer-boundaries-for-data-and-config-pipelines.md) — the four peers, the shared surfaces, the anti-patterns, the function's irreducible pipeline-reliability surface.

## Structure

- `01-…md` … `06-…md`: lecture chapters.
- `exercises/`: per-exercise prompts for hands-on practice.
- `labs/`: long-form hands-on labs (scaffolded).
- `quizzes/`: knowledge checks (scaffolded).
- `resources.md`: external references.

## Exercises

1. [Prompt-template versioning and rollback as a reliability operation](exercises/exercise-01-prompt-template-versioning-and-rollback-as-a-reliability-operation.md)
2. [Retrieval-freshness SLO for RAG](exercises/exercise-02-retrieval-freshness-slo-for-rag.md)
3. [Feature drift as serving incident with ML-platform handoff](exercises/exercise-03-feature-drift-as-serving-incident-with-ml-platform-handoff.md)
4. [Eval-set drift and rot detection](exercises/exercise-04-eval-set-drift-and-rot-detection.md)
5. [Unified pipeline observability in one pane](exercises/exercise-05-unified-pipeline-observability-in-one-pane.md)
