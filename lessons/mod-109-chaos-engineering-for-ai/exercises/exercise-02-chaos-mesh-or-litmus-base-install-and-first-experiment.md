# Exercise 02 — Chaos Mesh (or LitmusChaos) Base Install and First Experiment

**Estimated effort:** 3 hours · **Output:** a working chaos operator on a Kubernetes cluster, one green pod-kill experiment, one `rb-chaos-experiment-active` runbook stub, and a short install note

## Objective

Install one of the two CNCF chaos operators (Chaos Mesh or LitmusChaos) on a Kubernetes cluster you control, author and run a single `PodChaos` / `pod-delete` experiment against a sample serving replica, wire the experiment to a Prometheus alert as an auto-halt signal (chapter 05 §2.1), and write the on-call runbook stub `rb-chaos-experiment-active` that tells the on-call how to recognise and abort an experiment.

The point is **substrate competence**: at the end of the exercise, the function can execute any chapter 03 experiment against this cluster because the mechanics — operator install, label targeting, dry-run, auto-halt wiring, event stream — are in hand. The experiment itself is intentionally small; the surrounding machinery is where the work is.

## Prerequisites

- Chapter 02 of this module.
- A Kubernetes cluster you can install into — a local kind / minikube / k3d cluster is acceptable; a non-production cluster with real serving-like replicas is better. A production cluster is not required and should not be used for this exercise.
- Prometheus + Grafana installed (`kube-prometheus-stack` chart is the fastest path if not already present).
- A sample serving workload — the illustrative RAG-chat product from the exercise 01 suggestion works, or the `kubernetes-sigs/kube-prometheus-stack` sample workloads.
- Admin kubectl on the cluster (operator install requires it).

## Problem statement

Work in six parts.

### Part 1 — Install the operator

Pick Chaos Mesh *or* LitmusChaos and install it following the official docs. Record:

- The exact version installed.
- The Helm chart or manifest URLs used.
- The namespace the operator landed in.
- The ServiceAccount created and the RBAC scope (which namespaces it can act on).

Lock the install down: the operator's RBAC must be scoped to the chaos namespace + target namespaces, not cluster-wide read-write. If the default install is cluster-wide, open an issue in your install note describing the tightening to be done before any experiment runs in production.

### Part 2 — Deploy the sample workload

Deploy a minimal serving-replica workload labelled `app=sample-serving, chaos.target=false` by default. The label switch is the blast-radius mechanism — only replicas with `chaos.target=true` are in the experiment's selector.

A deployment of 3 replicas is sufficient. If your environment has a real inference pod you can use, prefer it; otherwise, an `nginx` deployment with a readiness probe is enough to see the pod-kill-and-recover cycle.

### Part 3 — Author the experiment

Author one experiment:

- **Chaos Mesh route.** `PodChaos` CRD with `action: pod-kill`, `mode: fixed`, `value: "1"`, selector on `app=sample-serving, chaos.target=true`, `duration: "30s"`.
- **LitmusChaos route.** A `ChaosEngine` running the `pod-delete` experiment from hub.litmuschaos.io, scoped to the same label, with `TOTAL_CHAOS_DURATION: 30` and `CHAOS_INTERVAL: 10`.

Commit the CRD to a git repo under `chaos/experiments/exercise-02-pod-kill.yaml`. Dashboards are for inspection; the CRD is the source of truth.

### Part 4 — Wire the auto-halt

Install or re-use a Prometheus alert that reads one of the sample workload's steady-state metrics (if no AI metric is handy, a simple `up{job="sample-serving"} < 0.9` alert works; for a richer workload, use an RPS or latency SLO burn). Wire the alert as a stop condition:

- **Chaos Mesh.** Use the `Workflow`'s conditional branches, or a `StatusCheck` CRD if your version supports it, or a side-channel controller that watches Alertmanager and deletes the active chaos CR.
- **LitmusChaos.** The `probes` field supports Prometheus probes that fail the experiment when the probe condition is violated.

Dry-run the halt once by manually forcing the alert (temporarily tightening the threshold).

### Part 5 — Run the experiment

Set `chaos.target=true` on one replica (`kubectl label pod <pod-name> chaos.target=true --overwrite`). Trigger the experiment (`kubectl apply -f chaos/experiments/exercise-02-pod-kill.yaml`). Observe:

- The chaos operator's event stream (`kubectl get events -n chaos` or the operator's log).
- The replica's lifecycle (kill, re-schedule, ready).
- The Prometheus alert's state (does not fire if the workload is healthy; if it fires, the auto-halt should terminate the experiment).

Record the timings: kill → ready, which panels moved, whether the alert fired.

### Part 6 — Author the runbook stub

Write `runbooks/rb-chaos-experiment-active.md` with the seven-section shape from `mod-108` chapter 01 §1. The sections:

- **Header.** `rb-chaos-experiment-active`, severity SEV-3 default, peer owner "AI SRE chaos program lead."
- **Detection signals.** How the on-call knows a chaos run is active: the chaos event stream in the incident channel, the Grafana annotation that chaos auto-posts, the `chaos.active=true` label on an experiment CR.
- **Confirm checklist.** "Is this a chaos run or a real incident?" with the three questions that disambiguate.
- **Mitigation menu.** How to abort an active experiment (the `chaos abort <run-id>` or `kubectl delete chaos ...` command, with the specific CRD name and namespace for this environment).
- **Handoff.** The chaos program lead's rotation.
- **User communication.** None (chaos experiments by policy do not produce user-visible communication).
- **Fixture back-feed.** If a chaos run was aborted by the on-call, the ledger records why.

## Requirements

1. **The operator is installed and version-recorded.** A one-line `operator: chaos-mesh 2.6.3 (helm chart X.Y.Z)` is enough.
2. **RBAC is scoped, not cluster-wide.** Document deviations.
3. **The experiment CRD is in git, not just the dashboard.** The repo path and commit hash are cited.
4. **A Prometheus alert is wired as an auto-halt signal.** The alert's expression is recorded; the stop-condition mechanism is documented.
5. **The experiment ran green.** Timings recorded.
6. **The auto-halt was tested at least once (dry-run).** Record the timings of the manual-force and the chaos operator's abort response.
7. **The runbook stub exists.** Seven sections, environment-specific content.
8. **The install note is short (≤ 500 words).** Covers: version, RBAC, limitations, the specific stop-condition mechanism, the on-call-channel integration.

## Starter guidance

- Default to Chaos Mesh if the function has no prior opinion; its CRD vocabulary is richer and the dashboard is more approachable for a first install. Switch to LitmusChaos later if GitOps is a hard requirement.
- `kind create cluster` + `helm install` is the shortest path to a working operator. Reserve production for later exercises.
- Do not skip the auto-halt wiring. "I'll add auto-halt after the first experiment" is how programs ship without it.
- Keep the first experiment deliberately small. The scope is the operator, not a dramatic kill; a 30-second pod-kill on a non-critical replica is the right size.
- The `rb-chaos-experiment-active` runbook is unglamorous but load-bearing. If the on-call cannot tell a chaos run from an incident, every chaos run risks a response from the pager team.

## Acceptance criteria

- An operator installed on a non-production cluster with scoped RBAC, version documented.
- An experiment CRD committed to git with the sample workload and the `chaos.target` label scheme.
- A Prometheus alert wired as a stop condition; one dry-run of the halt completed and timed.
- Evidence (logs, screenshots, kubectl events) that the experiment ran and the replica recovered within the expected window.
- `rb-chaos-experiment-active.md` written with all seven sections populated.
- A short install note (≤ 500 words) summarising the environment, versions, and open items.

## Stretch goals

- **Second operator in a side-by-side install.** Install LitmusChaos in a different namespace and run the same pod-kill experiment. Write a 200-word compare-contrast on what each operator does better for the fleet.
- **GitOps the install.** If the environment uses Argo CD, commit the operator install and the experiment CRD under an Argo application and verify the drift-detection catches a hand-edited CRD.
- **Chaos event stream into the on-call channel.** Wire the operator's events to Slack / Teams / PagerDuty event-notification (not alert; informational) so a chaos run shows up in the channel the on-call watches.
- **A Grafana annotation for every chaos run.** Operators can post `annotations/api/annotations` to a shared dashboard; the on-call dashboard should show the chaos window as a shaded region.
- **Write the deployment pattern for a GPU host.** Add a second workload labelled as `gpu.pool=mock-gpu` (emulating a real GPU pool) and verify the operator selectors can discriminate. This is the preparatory work for exercise 03.
