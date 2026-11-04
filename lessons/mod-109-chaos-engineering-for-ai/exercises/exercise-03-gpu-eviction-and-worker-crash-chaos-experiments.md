# Exercise 03 — GPU Eviction and Worker-Crash Chaos Experiments

**Estimated effort:** 3 hours · **Output:** two authored chaos experiments (`chaos-gpu-eviction-midstream`, `chaos-worker-crash-under-stream`), a dry-run log for each, one real (or shadow-traffic) run log for each, and fixture-ledger entries describing the findings

## Objective

Author two of the chapter 03 catalogue entries — `chaos-gpu-eviction-midstream` (chapter 03 §2) and `chaos-worker-crash-under-stream` (chapter 03 §7) — in environment-specific detail, dry-run each on the substrate from exercise 02, run each in a controlled setting (staging or a shadow-traffic slice), and feed the results into the fixture ledger (`mod-108` chapter 05) so the chaos program's first two serving-path experiments are real artifacts.

These two experiments exercise the serving-path's response to a mid-stream disruption — one scheduler-initiated (eviction), one process-initiated (crash). Together they validate the streaming-stall runbook (`rb-streaming-stall`), the preemption-spike runbook (`rb-preemption-spike`), and the retry-idempotency story from `mod-107` chapter 02 §5. Running them is how the function proves the serving layer handles its hardest in-flight failure mode.

## Prerequisites

- Chapter 01 (steady state), chapter 02 (operator install), chapter 03 (catalogue entries) of this module.
- Exercise 02 complete (working chaos operator, Prometheus auto-halt, `rb-chaos-experiment-active` stub).
- A model-backed serving workload or a realistic mock (a sample LLM-serving workload like `vllm` on a non-production node, a `litellm` proxy with a local provider, or a dedicated streaming-mock service). The workload must support **real streaming responses** — the crash experiment's hypothesis is invisible without them.
- A load generator (Locust, k6, or a small Python script) capable of running streaming requests for the duration of the chaos window.
- The runbooks `rb-streaming-stall`, `rb-preemption-spike`, `rb-ttft-regression` from `mod-108` chapter 01 §3 (or at least references to their detection sections) for the handoff mapping.

## Problem statement

Work in six parts.

### Part 1 — Author `chaos-gpu-eviction-midstream`

Produce the experiment specification in your chaos-program repo under `chaos/experiments/chaos-gpu-eviction-midstream.yaml` (CRD) plus `chaos/experiments/chaos-gpu-eviction-midstream.md` (documentation). The spec follows chapter 03 §1's shape:

- **Name** (fixed).
- **Production fault model.** The specific preemption class on the specific cluster — a batch job being admitted, a Spot reclamation, a priority-class-higher pod landing.
- **Hypothesis** in the three families (serving, quality, cost) — pin specific metrics and thresholds from the fleet's `mod-104` stack.
- **Mechanism.** The CRD. Use Chaos Mesh `PodChaos` with `action: pod-kill` and a label selector `app=serving, gpu.pool=<your-pool>, chaos.target=true`; or compose with a `kubectl cordon / drain` wrapper if you want to exercise the scheduler path rather than the pod-kill path.
- **Blast-radius controls.** Cohort (one replica of one deployment), fraction (1 of N replicas), time (10-minute window), auto-halt (two Prometheus alerts: SLO burn on TTFT, streaming-stall rate on the target route).
- **Runbook ties.** `rb-preemption-spike`, `rb-streaming-stall`, `rb-ttft-regression` under mitigation failure.
- **Expected observations.** Pod-restart event, stream-close counter spike, router behaviour, dashboards.
- **Fixture back-feed hooks.** Which fixture classes the result will feed.

### Part 2 — Author `chaos-worker-crash-under-stream`

Same shape, in `chaos/experiments/chaos-worker-crash-under-stream.{yaml,md}`. Key differences:

- Mechanism uses `PodChaos` with `action: container-kill` targeting the worker container specifically (not the sidecar). If the serving deployment is a single-container pod, use `pod-kill` with a shorter duration (1 second so the pod comes back fast).
- The hypothesis checks **terminal-event discipline** on streaming: the client should see `stream_terminated_by_server` (or the equivalent in the serving gateway's protocol), not an abrupt TCP close.
- The retry-idempotency check is central: does the retried request double-bill? The fixture back-feed should name a specific per-request cost attribution metric or a trace span the function will depend on.

### Part 3 — Dry-run each

Run each experiment with `dry-run: true` (Chaos Mesh) or with the experiment pointed at a single scratch pod that is not receiving real traffic (LitmusChaos). The purpose is:

- Verify the CRD parses and the selectors resolve to the intended pod(s) only.
- Verify the auto-halt alarms are visible to the operator (the dry-run should still check the stop-condition machinery).
- Verify the operator's RBAC permits the action.

Record the dry-run output and any corrections.

### Part 4 — Run each (controlled)

Run each experiment in a controlled setting. Preferred environments, in order of preference:

1. **Staging cluster with shadow-replayed production traffic.** Closest to production fidelity.
2. **Staging cluster with synthetic load.** Load generator sending streaming requests.
3. **Local development cluster with synthetic load.** Last resort; still produces meaningful signals if the workload is realistic enough.

For each run:

- Confirm the blast radius matches the spec.
- Start the load generator on the target route.
- Trigger the experiment (`kubectl apply`).
- Watch the chaos event stream, the auto-halt alarm state, and the key dashboards.
- Record the observations.
- Stop the experiment at the time cap (or earlier if auto-halt fires).

Produce a one-paragraph summary per experiment: did the hypothesis hold? Did auto-halt need to fire? What did the on-call dashboard show?

### Part 5 — Interpret the findings

For each experiment, identify at least one finding. A finding is any result that would change behaviour: a new alert, a canary, an eval case, a load shape, a chaos amendment, or a runbook update (the six fixture classes from `mod-108` chapter 05 §1).

Representative findings this exercise typically produces:

- Pod kill's recovery time is above the hypothesis budget — the PDB is too loose or the scheduler's re-place cadence is slow. Fixture: an alert on PDB / priority-class state for serving deployments.
- Streaming client sees a raw TCP close, not a terminal event — the gateway is not forwarding the server-side terminal. Fixture: a protocol-conformance test in CI.
- Retried request double-bills — per-request cost attribution is off. Fixture: a span-tagging change in the inference trace.
- No finding — the hypothesis holds exactly. Fixture: a chaos-amendment to add the experiment to the daily cadence (exercise 06).

### Part 6 — Ledger the findings

For each finding, write a fixture-ledger entry following the `mod-108` chapter 05 §3 schema. The entry includes the owner, the state (Candidate on first write), the cross-link to the chaos run, and the one-line description.

## Requirements

1. **Both experiments authored with the chapter 03 §1 shape** — all eight sections present, environment-specific content.
2. **Both experiments version-controlled in a chaos-program repo** with the CRD and the Markdown doc.
3. **Each experiment dry-run at least once**, with the dry-run output recorded.
4. **Each experiment run at least once in a controlled environment**, with observations recorded.
5. **At least one finding per experiment**, ledgered with the fixture-ledger schema.
6. **The runbook ties are explicit** — the chaos doc cites specific `mod-108` chapter 01 runbooks.
7. **The auto-halt is wired** to at least two independent alarms per experiment (chapter 05 §2).
8. **No side-effecting agent tools are enabled during the runs.** The no-chaos list from chapter 03 §11 applies.

## Starter guidance

- **Start with `chaos-gpu-eviction-midstream`.** The failure is easier to observe (the pod is just gone); the hypothesis is simpler (did the scheduler re-place, did the router drain the connection cleanly).
- **For `chaos-worker-crash-under-stream`, instrument the client.** The experiment's hypothesis hinges on what the *client* sees. A load generator that logs the exact bytes received around the kill is more informative than a generator that only counts requests.
- **Pin the hypothesis before the run.** Decide what "success" means before running, not after. Post-hoc re-interpretation of "success" is a failure mode this exercise is calibrated against.
- **Record the RPS and the request shape.** Streaming experiments depend on request characteristics; a reproducible finding requires reproducible inputs.
- **If a finding is "no change needed," that is still a finding.** Write it up; add the experiment to the daily cadence as the next step.

## Acceptance criteria

- Two experiment artifacts in the chaos-program repo: CRD + doc for each.
- Dry-run logs for each.
- Run logs for each with timings and dashboard observations.
- At least one fixture-ledger entry per experiment, following the `mod-108` chapter 05 §3 schema.
- Both experiments passed their own blast-radius policy (no cohort or fraction violation; auto-halt wired and visible).
- The chaos program lead (or the exercise reviewer) signs off that the two experiments are promotable to the daily cadence (exercise 06's input).

## Stretch goals

- **Compose a workflow that chains the two.** Chaos Mesh's `Workflow` CRD can run eviction first, verify recovery, then crash a different worker. The chained scenario is closer to a real preemption-storm.
- **Author the KV-pressure variant.** `chaos-kv-pressure-graceful-preemption` (chapter 03 §8) uses a `StressChaos` memory-hog on the serving container as an alternative to the pod-kill. Produce the spec and dry-run; a full run is a stretch target.
- **Walk the result against a real incident.** Pick a past incident (from the function's history or a dissected public post-mortem) where mid-stream preemption or worker crash was involved; show that one of your experiments would have reproduced the symptom.
- **Chaos-verify the runbook.** With a volunteer on-call, inject the fault at a random time inside a 60-minute window and measure the time from alert-fire to mitigation. Compare against the runbook's `mod-108` chapter 01 §1.5 handoff expectations.
- **Wire the experiments into CI.** Add a reduced variant of `chaos-gpu-eviction-midstream` as a pre-promotion smoke check for serving-image PRs (chapter 05 §1.1).
