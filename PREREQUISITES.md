# Prerequisites — AI Reliability Engineer (AI SRE) track

**Role level:** 35 (AI-reliability specialist, AI Infrastructure family)
**Track:** `ai-infra-sre-learning`

This track is a level-35 reliability specialty. It assumes the learner arrives with classical SRE literacy, the K8s / cloud / observability base muscle-memoried, MLOps automation familiarity in hand, and enough transformer-model literacy to on-call for a serving fleet without a hand-held primer. Fundamentals are **not** re-taught — the modules step directly onto SLO engineering for model-backed services, LLM + GPU observability, non-deterministic failure diagnosis, capacity planning for AI workloads, vendor risk, chaos engineering, release reliability, and DR for AI.

## Assumed lower-level curriculum

Complete or be practically fluent in the equivalent of these lower-level tracks before starting:

- **[`ai-infra-junior-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning) — level 10** — engineering-craft prerequisites (Linux, Git, Python packaging, HTTP APIs, networking primitives, Docker basics, unit testing, structured logging). If you cannot ship a small Python service and instrument it, start here.
- **[`ai-infra-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-engineer-learning) — level 20** — generalist AI infrastructure: Kubernetes (deployments, services, ingress, HPA, PodDisruptionBudget, probes, graceful shutdown), Terraform (cloud provider modules, state), AWS / GCP / Azure base (VPC / IAM / managed services), container runtime, CI/CD (GitHub Actions / GitLab CI), observability base (Prometheus / Grafana / OpenTelemetry at generalist depth), cost reporting primitives. If you cannot stand up a stateless web service on Kubernetes with Terraform, point Grafana at its metrics, and author an HPA on a custom metric, start here.
- **[`ai-infra-mlops-learning`](https://github.com/ai-infra-curriculum/ai-infra-mlops-learning) — level 25** — MLOps automation (model registry, CI/CD for ML, deployment pipelines, dataset / feature-store wiring, experiment tracking). This track plugs reliability gates into those pipelines — you do not need to author the pipeline from scratch, but you must be able to read, modify, and operate one.

## Assumed working knowledge

Even without holding the equivalent role formally, you should be practically fluent in:

- **Classical SRE literacy.** You have read (or will read before mod-102) the [Google SRE Book](https://sre.google/sre-book/table-of-contents/) and [SRE Workbook](https://sre.google/workbook/table-of-contents/). You can author an SLO, interpret an error-budget burn graph, run a blameless post-mortem, and shape an on-call rotation without hand-holding. This track re-positions the vocabulary against AI workloads but does not re-teach it.
- **Python.** Packaging (pyproject.toml, uv / poetry, virtualenvs), unit testing (pytest), typed code (mypy / pyright), FastAPI / async request handling, CI (GitHub Actions / GitLab CI).
- **Kubernetes at operator depth.** Deployments, services, ingress, ConfigMap / Secret, HPA / VPA / KEDA, StatefulSet, PodDisruptionBudget, taints / tolerations, resource requests / limits, probes, headless services, service-mesh basics. GPU-specific extensions (NVIDIA GPU Operator, DCGM-exporter, Node Problem Detector, DRA, MIG, MPS) are covered here in mod-104 and mod-106.
- **Linux + networking.** `/proc`, `strace`, `tcpdump`, iptables / nftables at reading depth, DNS, TLS termination, L4 vs. L7 load balancing. GPU driver / NUMA / PCIe / NVLink / RDMA topology is covered here at triage depth.
- **PyTorch + Hugging Face Transformers.** Enough to load a model, run inference, understand `.safetensors` layout, swap in a LoRA adapter, read `transformers` modeling files for a given architecture at reading-depth. Kernel-authoring depth lives at the `ai-infra-performance` peer; engine-internal depth lives at the `ai-infra-inference-engineer` peer.
- **HTTP APIs + LLM providers.** Calling OpenAI / Anthropic / Google / Bedrock / Azure OpenAI / open-source (vLLM / SGLang / TRT-LLM) inference APIs from Python. Comfortable with streaming (SSE / WebSocket), tool calling, structured output, rate-limit / retry discipline at the client level.
- **Observability base.** Prometheus metrics, PromQL at a working level, authoring Grafana dashboards, reading OpenTelemetry traces. Inference-specific instrumentation (TTFT / TPOT / ITL decomposition, DCGM metrics, `gen_ai.*` semantic conventions, drift signals) is taught here in mod-104.
- **Transformer-model literacy.** Attention (multi-head, multi-query, grouped-query), KV cache concept, token-level autoregressive decoding, prefix / prompt / completion phases, tokenizer basics. Deep architecture authoring is **not** required.
- **Load testing.** k6 / Locust / wrk / hey; interpreting P50 / P95 / P99 / P99.9 tails. Explicit instrumentation of GenAI workload shapes (prompt-length distribution, prefix-cache-hit-rate target) is taught here in mod-106.
- **Incident-command vocabulary.** You know what IC / OL / SME roles are, what a bridge call is, what a blameless post-mortem looks like. AI-flavored incident command is taught here in mod-108.
- **Reading the primary literature.** Comfortable opening a Google SRE Book chapter, a Principles of Chaos Engineering doc, a NIST SP 800-34 Rev 1 section, or a public frontier-lab post-mortem and turning method → runnable reliability pattern within a working day.

## Not assumed, taught here

You do **not** need prior experience with:

- The AI-specific failure-class catalog — silent drift, hallucination spike, toxic blast, prompt-cache cliff, retrieval-freshness regression, agent-tool-call cascade, judge-model drift, provider-degraded mode. Covered in mod-105 and threaded through mod-108.
- Per-token-granularity latency SLO authoring (TTFT / TPOT / ITL / E2E / goodput). Covered in mod-103.
- Quality SLOs composed with latency and cost SLOs into a single error-budget stack. Covered in mod-103.
- OpenTelemetry `gen_ai.*` semantic conventions, Langfuse / Arize Phoenix / LangSmith trace platforms, drift via Arize / WhyLabs / Evidently. Covered in mod-104.
- DCGM-exporter deployment, XID-code triage, GPU-aware alert authoring. Covered in mod-104 and mod-109.
- Queueing-theoretic capacity planning adapted for prefill-bound vs. decode-bound workloads, cross-generation fleet mix, model-provider quota pools. Covered in mod-106.
- Provider-dependency risk, LiteLLM / Portkey-shape fallback, status ingestion into the incident pipeline, SLA-vs-SLO reconciliation. Covered in mod-107.
- AI-specific runbooks (top twenty), AI-flavored post-mortem template, regression-fixture back-feed discipline. Covered in mod-108.
- Chaos Mesh / LitmusChaos / Gremlin / AWS FIS for AI workloads — GPU eviction, provider timeout injection, prefix-cache poisoning, agent-cascade injection. Covered in mod-109.
- Reliability knock-on of security / guardrail controls, joint post-mortems with the security peer. Covered in mod-110.
- Canary with response-diffing + semantic-equivalence tolerance, shadow / blue-green / SLO-based auto-rollback for model updates, DORA four-keys for model releases. Covered in mod-111.
- Prompt-template rollback as a reliability operation, retrieval-freshness SLOs, feature-drift as serving incident, eval-set rot. Covered in mod-112.
- NIST SP 800-34 applied to AI, stateful-agent state reconstitution, data-residency-respecting failover, DR rehearsal shape. Covered in mod-113.
- Reliability program ritual, build-vs-buy RFC for the SRE-tooling stack, cross-peer interface contracts, career portfolio. Covered in mod-114.

## Recommended reading before starting

- [Google SRE Book](https://sre.google/sre-book/table-of-contents/) and [SRE Workbook](https://sre.google/workbook/table-of-contents/) — especially the SLO / error-budget / post-mortem / on-call chapters.
- [Principles of Chaos Engineering](https://principlesofchaos.org/) — short, read once before mod-109.
- [NIST AI RMF Generative AI Profile (AI 600-1)](https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile) — the GenAI risk taxonomy we use to name non-deterministic failure classes.
- [OpenTelemetry Semantic Conventions for Generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — the attribute surface we instrument against in mod-104.
- [NVIDIA DCGM user guide](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/) — enough to know what the metrics mean before mod-104.
- A recent public frontier-lab post-mortem or reliability write-up (OpenAI / Anthropic / Google DeepMind / Microsoft AI engineering blog) — read as an on-call engineer would, tracing each incident to the primitive that would have caught it.
- [The Tail at Scale (Dean & Barroso)](https://research.google/pubs/pub40801/) — short paper that underwrites the per-token-latency SLO framing.
- A recent public vLLM / SGLang / TensorRT-LLM release note — operator-eye view, so you can bring engine-knob vocabulary into mod-102.

## Not required, useful to have

- Prior work operating a managed inference provider (Fireworks / Together / Baseten / Modal / Replicate / Anyscale / Groq / Cerebras) — a working feel is useful for mod-107's vendor-eval framing.
- Prior work with a service mesh (Istio / Linkerd) — the progressive-delivery patterns in mod-111 compose cleanly with a mesh.
- Prior work with a cost-observability platform (Kubecost / OpenCost / Vantage) — useful for mod-103's cost-SLO module.
- Prior work with LLM-eval tooling (promptfoo / Inspect / Ragas / DeepEval / LangChain Evaluators) — mod-112 and mod-111 touch the handoff to the `ai-eval-engineer` peer.
- Certifications: Kubernetes CKA / CKAD, NVIDIA DLI GPU courses, (ISC)² CISSP for BCP / incident-response credibility, cloud-provider reliability specialty certs are treated as strong signals in enterprise hiring but not required.

## Not in scope for this track (linked out)

- **Inference-engine source-level internals and authoring** — vLLM / SGLang / TRT-LLM / Dynamo scheduler loop, KV allocator, model-runner, worker-executor layering. Owned by [`ai-infra-inference-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-inference-engineer-learning) (peer, level 35). This track teaches engine-knob fluency at on-call depth; source-level engine authoring is the peer's primary deliverable.
- **GPU / CUDA / kernel-authoring / CUTLASS / CuTe DSL / Nsight-Compute-to-SASS depth** — owned by [`ai-infra-performance-learning`](https://github.com/ai-infra-curriculum/ai-infra-performance-learning) (peer, level 35). This track reads GPU telemetry at triage level and files joint tickets when a kernel-level regression is suspected.
- **Inference-edge adversarial defence** — prompt-injection at the API, PII / jailbreak classifier sidecars, model-extraction query-rate control, model-weight supply-chain signing. Owned by [`ai-infra-security-learning`](https://github.com/ai-infra-curriculum/ai-infra-security-learning) (peer, level 35). This track owns the reliability knock-on of defensive controls only.
- **Self-service ML platform + model registry depth** — owned by [`ai-infra-ml-platform-learning`](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning) (peer, level 30). This track consumes the artifact-handoff contract.
- **Application-layer eval release-gate depth** — owned by [`ai-eval-engineer-learning`](https://github.com/ai-engineering-curriculum/ai-eval-engineer-learning) (peer, level 30, AI Engineering family). This track consumes the eval-gate signal and wires it into SLO-based auto-rollback.
- **Benchmark / statistical / judge-vs-human methodology** — owned by [`model-evaluation-engineer-learning`](https://github.com/ml-engineering-curriculum/model-evaluation-engineer-learning) (peer, level 30, ML Engineering family).
- **Post-training / fine-tuning stack depth** — owned by [`fine-tuning-engineer-learning`](https://github.com/ml-engineering-curriculum/fine-tuning-engineer-learning) (peer, level 30).
- **MLOps automation (CI/CD pipeline authoring)** — owned by [`ai-infra-mlops-learning`](https://github.com/ai-infra-curriculum/ai-infra-mlops-learning) (peer, level 25). This track plugs reliability gates into those pipelines.
- **General K8s / Terraform / cloud base** — owned by [`ai-infra-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-engineer-learning) (level 20). This track covers only GPU-specific reliability extensions.
- **Cross-stack reliability architecture at team-of-teams scope** — owned by [`ai-infra-architect-learning`](https://github.com/ai-infra-curriculum/ai-infra-architect-learning) (next-up, level 40). This track feeds platform RFCs upward.
- **People management for an AI SRE / production-engineering team** — owned by [`ai-infra-team-lead-learning`](https://github.com/ai-infra-curriculum/ai-infra-team-lead-learning) (next-up, level 40). This track stops at technical-leadership scope.
- **AI risk-management framework, harm-model engineering, regulator-facing risk register** — owned by [`ai-risk-engineer-learning`](https://github.com/ai-governance-curriculum/ai-risk-engineer-learning) (peer, level 25, AI Governance family). This track feeds AI-reliability-incident telemetry into that track's risk register.
- **Prompt engineering, agent design, product analytics** — owned by the AI Engineering family (`llm-application-developer`, `rag-engineer`, `applied-ai-engineer` at level 25).
- **Legal / compliance / privacy** — licensing, data residency, regulator sign-off. Out of scope; routed to counsel.
- **Classical-stack SRE for non-AI systems** — if the organisation also runs non-AI services, their reliability engineering is the general SRE team's responsibility; this role covers only the AI surface.
- **SecOps / DFIR / IR evidence handling** — this role contributes AI-specific telemetry to those teams but is not the system of record for security-incident evidence.
