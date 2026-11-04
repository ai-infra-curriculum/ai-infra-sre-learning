# mod-111-release-engineering-and-progressive-delivery: Release Engineering and Progressive Delivery for Model Updates

> Scaffolded by `aicg org execute-plan`. Lecture chapters and exercise content are authored on subsequent autonomous cycles.

**Estimated effort:** 16 hours

## Learning objectives

- Deploy canary rollouts for model updates — Flagger / Argo Rollouts / Istio-based traffic splitting, canary-weight ramp, response-diffing with semantic-equivalence tolerance, SLO-based auto-rollback — adapted for the fact that two model versions never produce byte-identical responses
- Deploy shadow traffic for model changes — mirror production traffic to a candidate deploy without client-visible responses, compare on latency / quality / cost, feed back into the release gate
- Deploy blue-green for large models — dual-load constraints when the model does not fit on one node, atomic routing cutover, model-version header for client observability, drain-and-swap discipline
- Wire the release gate to eval + reliability signals jointly — the ai-eval-engineer peer's eval gate as a quality predicate, the SRE SLO stack as the reliability predicate, both must pass to promote; author the contract explicitly so no promotion bypasses either
- Measure release reliability with DORA four-keys adapted for AI — deployment frequency per fleet, lead time for a model-config change, change-failure rate per rollout, MTTR when a model release triggers a rollback; set targets that match team maturity
- Pair progressive delivery with the chaos + load-test programs — pre-promotion load test at canary scale, chaos probe of the new deploy before weight ramp, SLO-burn-as-circuit-breaker during ramp
- Cite the boundaries — engine-side drain + in-flight swap primitives are at ai-infra-inference-engineer peer; CI/CD pipeline wiring is at ai-infra-mlops peer (level 25); eval-gate methodology is at ai-eval-engineer peer (level 30); this module owns the reliability contract that stitches them

## Structure

- `01-…md` … `0N-…md`: lecture chapters.
- `exercises/`: per-exercise prompts.
- `labs/`: long-form hands-on labs.
- `quizzes/`: knowledge checks.
- `resources.md`: external references.
