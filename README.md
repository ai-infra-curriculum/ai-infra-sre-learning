# AI Infrastructure · AI Reliability Engineer (AI SRE) — Learning Repository

<!-- aicg:site-banner -->
> 🎓 Part of the free, open-source **AI Career Curriculum** ecosystem — [Infrastructure](https://github.com/ai-infra-curriculum) · [ML Engineering](https://github.com/ml-engineering-curriculum) · [AI Engineering](https://github.com/ai-engineering-curriculum) · [Governance](https://github.com/ai-governance-curriculum) · [Startup](https://github.com/ai-startup-curriculum) · [Workforce](https://github.com/ai-workforce-curriculum).
<!-- /aicg:site-banner -->

Owns the operational reliability contract of production AI / ML / LLM / agent systems: SLOs for model-backed services at per-token granularity, observability for LLM request paths and GPU workloads, incident response for non-deterministic failures, capacity planning for GPU fleets and model-provider quotas, dependency risk on OpenAI / Anthropic / Google / Bedrock / Azure OpenAI, chaos engineering for AI workloads, release engineering for model updates, DR / BCP / multi-region for serving fleets, and cross-functional reliability program leadership at team scope.

**Role level:** 35 (AI-reliability specialist, AI Infrastructure family)
**Planned scope:** 14 modules (216 h) + 3 projects (120 h) = 336 h — see [`CURRICULUM.md`](CURRICULUM.md)
**Research evidence:** [`JOB_REQUIREMENTS.md`](JOB_REQUIREMENTS.md) · [`.aicg/job-requirements.json`](.aicg/job-requirements.json)
**Module + project plan:** [`.aicg/curriculum-plan.json`](.aicg/curriculum-plan.json)

**Peer boundaries at the same level (all level 35, AI Infrastructure family):**
- [`ai-infra-inference-engineer-learning`](https://github.com/ai-infra-curriculum/ai-infra-inference-engineer-learning) owns the serving system itself (vLLM / SGLang / TensorRT-LLM / Dynamo at source-level depth).
- [`ai-infra-performance-learning`](https://github.com/ai-infra-curriculum/ai-infra-performance-learning) owns the kernel-authoring / GPU-optimisation craft.
- [`ai-infra-security-learning`](https://github.com/ai-infra-curriculum/ai-infra-security-learning) owns inference-edge adversarial defence.
- **This track** owns the reliability contract around the stack — SLO, incident, chaos, capacity, vendor risk.

> **Status**: scaffolded by `aicg org bootstrap-role`. `JOB_REQUIREMENTS.md`, `.aicg/job-requirements.json`, and `.aicg/curriculum-plan.json` are authored in scaffold-mode (2026-11-04 bootstrap session — WebSearch / WebFetch were permission-gated, live posting evidence deferred to the next autonomous research cycle). Module and project content are not yet authored. Run `aicg org research` and `aicg org daily` to drive the autonomous fill-in loop.

## Layout

```
ai-infra-sre-learning/
├── lessons/mod-XXX-*/        modules with lectures, exercises, labs, quizzes
├── projects/project-XXX-*/   multi-module capstones
├── CURRICULUM.md             role-level coverage map
├── PREREQUISITES.md          assumed entry skills
├── JOB_REQUIREMENTS.md       readable requirements summary
├── VERSIONS.md               release history
└── README.md                 this file
```

## Paired Solutions Repo

[`ai-infra-sre-solutions`](https://github.com/AI-Infra-Curriculum/ai-infra-sre-solutions) carries the reference implementations.

---

<!-- aicg:maintained-by -->
Maintained by [VeriSwarm.ai](https://veriswarm.ai)
