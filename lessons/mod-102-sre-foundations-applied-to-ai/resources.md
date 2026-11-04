# Resources — mod-102: SRE Foundations Applied to AI

References grounded in authoritative public sources. Items that require version-specific confirmation against the engine documentation at the time of the reader's deployment are flagged `needs-research` so the autonomous research loop can backfill.

## Classical SRE canon (chapter 01)

- *Site Reliability Engineering* (the "SRE Book"), ed. Betsy Beyer et al. — <https://sre.google/sre-book/table-of-contents/>. Especially:
  - [Chapter 3 — Embracing Risk](https://sre.google/sre-book/embracing-risk/) (error-budget abstraction, source for chapter 02).
  - [Chapter 4 — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/) (SLI / SLO vocabulary).
  - [Chapter 5 — Eliminating Toil](https://sre.google/sre-book/eliminating-toil/) (toil, toil budget).
  - [Chapter 11 — Being On-Call](https://sre.google/sre-book/being-on-call/) (rotation design, chapter 03).
  - [Chapter 27 — Launch Coordination Engineering](https://sre.google/sre-book/launch-coordination-engineering/) (LCE, chapter 01 section 5).
- *The Site Reliability Workbook*, ed. Betsy Beyer et al. — <https://sre.google/workbook/table-of-contents/>. Especially:
  - [Chapter 2 — Implementing SLOs](https://sre.google/workbook/implementing-slos/) (SLO authoring).
  - [Chapter 5 — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) (multi-window / multi-burn-rate — authoritative reference for chapter 02 and chapter 03).
  - [Chapter 8 — On-Call](https://sre.google/workbook/on-call/) (rotation health metrics, embedded vs. consulting).
  - [Chapter 12 — Introducing Non-Abstract Large System Design](https://sre.google/workbook/non-abstract-design/) (chapter 01 section 6, exercise 05).

## OpenTelemetry / observability standards (referenced across module)

- OpenTelemetry Semantic Conventions for Generative AI — <https://opentelemetry.io/docs/specs/semconv/gen-ai/>. The `gen_ai.*` attribute surface; deepened in `mod-104`.
- Prometheus metric and alerting discipline — <https://prometheus.io/docs/practices/alerting/>.

## NIST AI reliability / risk references

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>.
- NIST AI RMF Generative AI Profile (AI 600-1) — <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>. Vocabulary for quality / drift / non-deterministic failure classes used in chapter 02 "bad-event" definitions.

## vLLM (chapter 04)

- Documentation home — <https://docs.vllm.ai/>.
- Engine arguments reference — <https://docs.vllm.ai/en/latest/serving/engine_args.html>.
- OpenAI-compatible server guide — <https://docs.vllm.ai/en/latest/serving/openai_compatible_server.html>.
- Metrics reference — <https://docs.vllm.ai/en/latest/serving/metrics.html>.
- Automatic prefix caching — <https://docs.vllm.ai/en/latest/features/automatic_prefix_caching.html>.
- Chunked prefill — <https://docs.vllm.ai/en/latest/features/chunked_prefill.html>.
- Speculative decoding — <https://docs.vllm.ai/en/latest/features/spec_decode.html>.
- Original PagedAttention paper (Kwon et al., 2023 — "Efficient Memory Management for Large Language Model Serving with PagedAttention") — <https://arxiv.org/abs/2309.06180>.

<!-- needs-research: confirm each of the vLLM docs URLs above resolves against the pinned engine version at the time of reading, and note any URL migrations. -->

## SGLang (chapter 05)

- Documentation home — <https://docs.sglang.ai/>.
- Server launch and arguments — <https://docs.sglang.ai/backend/server_arguments.html>.
- RadixAttention paper (Zheng et al., 2023 — "Efficiently Programming Large Language Models using SGLang") — <https://arxiv.org/abs/2312.07104>.
- Data-parallel-attention explainer — SGLang blog and docs.

<!-- needs-research: confirm SGLang docs URL paths and the current canonical flag names for `--enable-dp-attention` and speculative-decoding flags against the pinned deployment version. -->

## TensorRT-LLM and Triton (chapter 06)

- TensorRT-LLM documentation — <https://nvidia.github.io/TensorRT-LLM/>.
- TensorRT-LLM GitHub repository — <https://github.com/NVIDIA/TensorRT-LLM>.
- `trtllm-build` reference (version-specific — see release notes) — <https://nvidia.github.io/TensorRT-LLM/commands/trtllm-build.html>.
- Triton Inference Server documentation — <https://docs.nvidia.com/deeplearning/triton-inference-server/>.
- Triton model repository layout — <https://github.com/triton-inference-server/server/blob/main/docs/user_guide/model_repository.md>.
- Triton model configuration (`config.pbtxt`) — <https://github.com/triton-inference-server/server/blob/main/docs/user_guide/model_configuration.md>.
- Triton TensorRT-LLM backend — <https://github.com/triton-inference-server/tensorrtllm_backend>.

<!-- needs-research: trtllm-build flag naming has evolved across TensorRT-LLM minor versions. Confirm the flag set in chapter 06 against the docs for the TRT-LLM version currently in production at the reader's platform. -->

## NVIDIA Dynamo (chapter 06)

- Documentation — <https://docs.nvidia.com/dynamo/>.
- NVIDIA Dynamo repository — <https://github.com/ai-dynamo/dynamo>.
- Disaggregated-serving background (papers and blogs frequently cited):
  - "Splitwise: Efficient Generative LLM Inference Using Phase Splitting" (Patel et al., 2024) — <https://arxiv.org/abs/2311.18677>.
  - "DistServe: Disaggregating Prefill and Decoding for Goodput-optimized Large Language Model Serving" (Zhong et al., 2024) — <https://arxiv.org/abs/2401.09670>.

<!-- needs-research: NVIDIA Dynamo is a relatively new project and docs URLs and feature surface have evolved rapidly; confirm the docs path and the current knob surface at reading time. -->

## GPU telemetry and fleet operations (chapter 03 escalation, exercise 05)

- NVIDIA DCGM user guide — <https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/>.
- NVIDIA XID error codes — <https://docs.nvidia.com/deploy/xid-errors/index.html>.
- NVIDIA GPU Operator on Kubernetes — <https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/>.
- Node Problem Detector — <https://github.com/kubernetes/node-problem-detector>.

## Provider status pages (chapter 03 paging policy, chapter 06 Dynamo cost comparison)

- OpenAI — <https://status.openai.com/>.
- Anthropic — <https://status.anthropic.com/>.
- Google Cloud / Vertex — <https://status.cloud.google.com/>.
- AWS Service Health (including Bedrock) — <https://health.aws.amazon.com/>.
- Azure — <https://status.azure.com/>.

## Provider-fallback / gateway (chapter 03 escalation, chapter 06 routing)

- LiteLLM — <https://docs.litellm.ai/>.
- Portkey — <https://docs.portkey.ai/>.

## Peer curricula (chapter 07 and the deferral contract)

- `ai-infra-inference-engineer-learning` — serving-system authoring (peer, level 35) — <https://github.com/ai-infra-curriculum/ai-infra-inference-engineer-learning>.
- `ai-infra-performance-learning` — kernel authoring (peer, level 35) — <https://github.com/ai-infra-curriculum/ai-infra-performance-learning>.
- `ai-infra-security-learning` — adversarial defence (peer, level 35) — <https://github.com/ai-infra-curriculum/ai-infra-security-learning>.
- `ai-infra-ml-platform-learning` — ML platform (peer, level 30) — <https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning>.
- `ai-eval-engineer-learning` — application eval release-gate (peer, level 30) — <https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning>.
- `ai-infra-mlops-learning` — CI/CD (peer, level 25) — <https://github.com/ai-infra-curriculum/ai-infra-mlops-learning>.

## Further reading referenced across the module

- Dean & Barroso, *The Tail at Scale*, Communications of the ACM, 2013 — <https://research.google/pubs/pub40801/>. Underwrites the per-token-latency-SLO framing in chapter 01.
- Harchol-Balter, *Performance Modeling and Design of Computer Systems* — <https://www.cs.cmu.edu/~harchol/PerformanceModeling/book.html>. Queueing-theory reference for the chapter 01 section 6 back-of-envelope.
- Principles of Chaos Engineering — <https://principlesofchaos.org/>.

## Needs-research (flagged for autonomous loop)

<!-- needs-research: confirm vLLM flag naming (especially speculative-decoding config and prefix-caching knobs) against the current stable docs for the version pinned in representative production platforms. Flag deprecation / renaming has happened across minor releases and the chapter 04 tables should be versioned. -->

<!-- needs-research: confirm SGLang flag naming (especially `--enable-dp-attention` and speculative-decoding flags) against the current stable docs. SGLang moves fast; chapter 05 tables should be versioned. -->

<!-- needs-research: trtllm-build flag naming against the current TensorRT-LLM stable release. Several arguments have been renamed or reorganized across versions and chapter 06 should cite the version-specific docs URL. -->

<!-- needs-research: NVIDIA Dynamo throughput-delta numbers on prefill/decode separation vs. co-located serving, pulled from the official NVIDIA Dynamo blog and whitepaper. Chapter 06 references these as motivational but does not cite a specific number. -->

<!-- needs-research: public post-mortem corpus from frontier labs (OpenAI / Anthropic / Google DeepMind / Microsoft / major model-hosting providers) 2024–2026 classified by engine-root-cause (vLLM / SGLang / TRT-LLM / Dynamo), for use as chapter 04–06 case material and exercise 04 dry-run inputs. Reuse the mod-101 version of this entry if it has already been resolved. -->
