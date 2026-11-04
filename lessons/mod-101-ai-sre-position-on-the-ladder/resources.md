# Resources — mod-101: The AI SRE Position on the Infra Ladder

References grounded in authoritative public sources. Items that would require live posting / incident evidence not sampled in the 2026-11 bootstrap are flagged `needs-research` so the autonomous research loop can backfill.

## Classical SRE canon

Read before — or alongside — this module. The left column of chapter 04 (what carries verbatim) is sourced from these.

- *Site Reliability Engineering* (the "SRE Book"), edited by Betsy Beyer, Chris Jones, Jennifer Petoff, Niall Richard Murphy. Online at <https://sre.google/sre-book/table-of-contents/>. Especially Part III (Practices), which is the source of Dickerson's hierarchy.
- *The Site Reliability Workbook*, edited by Betsy Beyer et al. Online at <https://sre.google/workbook/table-of-contents/>. Especially the Implementing SLOs chapter and the on-call chapters.
- *Building Secure and Reliable Systems*, Heather Adkins et al. Online at <https://sre.google/books/building-secure-reliable-systems/>. Referenced for the reliability × security crossover (mod-110 downstream of this module).
- The blameless post-mortem framing: Google SRE Book, Chapter 15 (Postmortem Culture) — <https://sre.google/sre-book/postmortem-culture/>.
- Dean & Barroso, *The Tail at Scale*, Communications of the ACM, 2013 — <https://research.google/pubs/pub40801/>. The paper that underwrites per-token-latency SLO framing.
- Harchol-Balter, *Performance Modeling and Design of Computer Systems* — <https://www.cs.cmu.edu/~harchol/PerformanceModeling/book.html>. The queueing-theory reference for capacity planning (mod-106 downstream).

## Dickerson's hierarchy

- Primary citation: *SRE Book*, Part III — Practices, intro section. The pyramid is presented as the ordering of the subsequent Practices chapters.
- Mikey Dickerson's original framing is attributed in the SRE Book; the public talk history is covered in the book's acknowledgements.

## AI-specific reliability references cited in the chapters

- OpenTelemetry Semantic Conventions for Generative AI (`gen_ai.*` attribute surface) — <https://opentelemetry.io/docs/specs/semconv/gen-ai/>. Referenced in chapters 01, 03, and chapter 06's posting-reading checklist.
- NIST AI RMF Generative AI Profile (AI 600-1) — <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>. The vocabulary for quality / drift / non-deterministic failure classes.
- NVIDIA DCGM user guide — <https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/>. GPU telemetry surface.
- NVIDIA XID error reference — <https://docs.nvidia.com/deploy/xid-errors/index.html>.
- NVIDIA GPU Operator on Kubernetes — <https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/>.
- Principles of Chaos Engineering — <https://principlesofchaos.org/>.
- NIST SP 800-34 Rev 1 (Contingency Planning Guide for Federal Information Systems) — <https://csrc.nist.gov/publications/detail/sp/800-34/rev-1/final>. The DR / BCP methodology referenced for mod-113 downstream.
- DORA Four Keys — <https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance>.
- OpenAI status page — <https://status.openai.com/>.
- Anthropic status page — <https://status.anthropic.com/>.
- Google Cloud / Vertex status — <https://status.cloud.google.com/>.
- AWS Service Health (including Bedrock) — <https://health.aws.amazon.com/>.
- Azure status — <https://status.azure.com/>.

## Inference-engine operator references (for peer-boundary context)

Operator-depth references only; source-level depth lives with the `ai-infra-inference-engineer` peer.

- vLLM — <https://docs.vllm.ai/>.
- SGLang — <https://docs.sglang.ai/>.
- NVIDIA TensorRT-LLM — <https://nvidia.github.io/TensorRT-LLM/>.
- NVIDIA Triton Inference Server — <https://docs.nvidia.com/deeplearning/triton-inference-server/>.
- NVIDIA Dynamo — <https://docs.nvidia.com/dynamo/>.

## LLM observability + drift platforms (for peer-boundary context)

- Langfuse — <https://langfuse.com/docs>.
- Arize Phoenix — <https://arize.com/docs/phoenix>.
- LangSmith — <https://docs.smith.langchain.com/>.
- Arize AI (drift) — <https://docs.arize.com/>.
- WhyLabs (drift) — <https://docs.whylabs.ai/>.
- Evidently AI (drift) — <https://docs.evidentlyai.com/>.

## Provider-fallback / gateway (for peer-boundary context)

- LiteLLM — <https://docs.litellm.ai/>.
- Portkey — <https://docs.portkey.ai/>.
- OpenAI Cookbook — production rate-limit patterns — <https://cookbook.openai.com/examples/how_to_handle_rate_limits>.

## Progressive delivery (for peer-boundary context)

- Flagger — <https://docs.flagger.app/>.
- Argo Rollouts — <https://argoproj.github.io/argo-rollouts/>.
- Istio — <https://istio.io/latest/docs/>.

## Peer tracks within this org (sibling curricula)

Linked out per the ownership rule in `CURRICULUM.md`.

- `ai-infra-inference-engineer-learning` — serving-system authoring (peer, level 35) — <https://github.com/ai-infra-curriculum/ai-infra-inference-engineer-learning>.
- `ai-infra-performance-learning` — kernel authoring (peer, level 35) — <https://github.com/ai-infra-curriculum/ai-infra-performance-learning>.
- `ai-infra-security-learning` — adversarial defence (peer, level 35) — <https://github.com/ai-infra-curriculum/ai-infra-security-learning>.
- `ai-infra-ml-platform-learning` — ML platform (peer, level 30) — <https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning>.
- `ai-eval-engineer-learning` — application eval release-gate (peer, level 30) — <https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning>.
- `model-evaluation-engineer-learning` — benchmark / statistical methodology (peer, level 30) — <https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning>.
- `ai-infra-mlops-learning` — CI/CD (peer, level 25) — <https://github.com/ai-infra-curriculum/ai-infra-mlops-learning>.
- `llm-application-developer-learning` — consumer track (level 25) — <https://github.com/ai-engineering-curriculum/llm-application-developer-learning>.
- `rag-engineer-learning` — consumer track (level 25) — <https://github.com/ai-engineering-curriculum/rag-engineer-learning>.
- `applied-ai-engineer-learning` — consumer track (level 25) — <https://github.com/ai-engineering-curriculum/applied-ai-engineer-learning>.

## Responsible-AI release tiering (for context on the role's upward interface)

These frameworks shape what gets released and therefore shape the reliability work. Read once; cited in chapters 05 and 06.

- Anthropic Responsible Scaling Policy — <https://www.anthropic.com/rsp>.
- OpenAI Preparedness Framework — <https://openai.com/safety/preparedness/>.
- Google DeepMind Frontier Safety Framework — <https://deepmind.google/discover/blog/introducing-the-frontier-safety-framework/>.

## Certifications referenced in chapter 06 and PREREQUISITES.md

- CNCF CKA / CKAD — <https://www.cncf.io/training/certification/>.
- NVIDIA Deep Learning Institute — <https://www.nvidia.com/en-us/training/>.
- (ISC)² CISSP (BCP / incident-response credibility) — <https://www.isc2.org/Certifications/CISSP>.
- AWS / Azure / GCP reliability-specialty certifications (specific landing pages change often).

## Needs-research (flagged for autonomous loop)

<!-- needs-research: live posting sample across Greenhouse / Lever / Ashby / Workday for the titles "AI SRE", "LLM SRE", "AI Production Engineer", "AI Reliability Engineer", and overloaded variants of "AI Infrastructure Engineer", filtered to postings in the last 90 days. Produce a per-theme empirical frequency table and backfill into chapter 06. -->

<!-- needs-research: a public post-mortem corpus from frontier labs (OpenAI / Anthropic / Google DeepMind / Microsoft AI Engineering / major model-hosting providers) for 2024–2026, classified by failure class per the chapter-05 catalog. Useful as a chapter-06 evidence base and as case-law source material for exercises 01 and 03. -->

<!-- needs-research: the reconciliation discrepancy flagged in JOB_REQUIREMENTS.md — the sibling ai-infra-inference-engineer packet lists ai-infra-sre at level 30, while this packet is authored at level 35. Confirm the level placement against live 2026 market signal and update sibling packets accordingly. -->
