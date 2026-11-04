# Agent Tool-Call Cascades — Recursion, Rate-Limits, and Mis-Plumbed Schemas

## Why this chapter exists

Agent surfaces — product surfaces where the model is allowed to call tools, read results, and decide what to call next — have a reliability-failure shape that classical SRE has no vocabulary for and most LLM-SRE playbooks underweight. The failure mode is not that the output is wrong; the output is often correct, eventually. The failure mode is that the agent consumed 200 tool calls, 400 LLM invocations, 7 minutes of wall clock, and $43 in provider tokens to answer a question that should have taken 3 calls and 11 seconds.

The signals are cost, latency, and the shape of the trace tree — not the correctness of the terminal answer. The on-call who only watches quality SLIs will see no alert. The on-call who watches the cost SLI from `mod-103` chapter 04 will see the blow-up but will not know where to route it. This chapter authors the specific signals (tool-call-depth histogram, tool-call-error rate, provider-cost-per-session) and the three canonical root causes (recursive loop, rate-limit-on-nested-tool, mis-plumbed schema) that together make up class 2.7.

The chapter assumes an agent is in the stack. Not every AI SRE operates an agent surface; those who do will encounter 2.7. Those who do not should skim this chapter against the day one is added to their stack.

## 1. The shape of an agent surface, briefly

An agent, in the sense this chapter uses the term, is a loop:

1. The model receives a user input, a system message, and a set of **tool descriptors** (JSON schemas describing available functions).
2. The model generates a response that either contains a terminal answer *or* a tool call — a structured output with a tool name and arguments.
3. If a tool call: the agent framework executes the tool, captures the result, appends it to the conversation, and loops back to step 2 with the tool result in context.
4. If a terminal answer: the agent ends.

Widely-used agent frameworks include LangChain, LangGraph, LlamaIndex agents, OpenAI's Assistants API, Anthropic's tool-use pattern, Pydantic AI, AutoGen, CrewAI, and numerous in-house implementations. The failure modes this chapter treats are framework-agnostic.

Tools are typically backed by:

- HTTP APIs (REST, GraphQL).
- Database queries (SQL, NoSQL, vector).
- Search (internal search, web search).
- Code execution (sandboxed Python, SQL, JS).
- Other LLMs (meta-call surfaces — expensive).
- Internal services (your own microservices).

A well-behaved agent runs a loop of 1–10 tool calls typically, converges on a terminal answer, and exits. A pathological agent runs 50 or 200 or 1000 before hitting a depth cap or a wall-clock cap. The cost of pathology scales roughly linearly with the pathological depth, and compounds across concurrent sessions.

## 2. The three canonical root causes

### 2.1 Recursive tool loop

The agent repeatedly calls the same tool (or a small set of tools) with slightly different arguments, reading the result, deciding to call again, looping.

Concrete patterns the on-call will see:

- The agent is given a `search_documents` tool. It searches with one query, gets middling results, decides the results aren't good enough, searches again with a rephrased query, gets middling results, searches again, ad infinitum. Depth cap is the only exit.
- The agent is given `list_items` and `get_item_detail` tools. It lists 50 items, then gets each one, then decides it needs the list again with a filter, loops.
- The agent is debugging — it was given a `run_code` tool. The code fails; the agent asks the model to fix it; the fix fails too; the fix-the-fix loop runs until the depth cap.

The root cause is often prompt-side (the system message's termination condition is weak), model-side (the model's reasoning isn't converging), or environment-side (the tools return results the model cannot use productively).

### 2.2 Rate-limit on a nested tool

A tool the agent calls is upstream-rate-limited. The tool call returns an error (429 or a tool-specific "retry later"). The agent interprets the error as "try again" and retries. The retry also 429s. The agent interprets again as "try again." Loop.

The failure is partly agent-side (the agent does not know how to interpret 429 as permanent-for-the-window), partly framework-side (the framework may auto-retry transparently to the agent), partly schema-side (the tool's error contract does not distinguish transient from permanent).

### 2.3 Mis-plumbed tool schema

The tool's JSON schema has changed, but the tool descriptor the agent sees has not. The agent generates arguments that satisfy the *old* schema but violate the *new* one; the tool errors; the agent retries; same arguments; same error.

Alternative shape: the schema is correct but ambiguous. Two valid arg sets are both accepted but only one works; the agent randomly picks and sometimes loops on the wrong one.

Alternative shape: the model's structured-output conformance is low. The engine or provider's structured-output guarantee is weaker than the tool expects, and about 5% of tool calls fail JSON parsing. The structured-output-conformance SLI from `mod-103` chapter 02 is the signal.

## 3. The signals that make a cascade visible

### 3.1 Tool-call-depth histogram

The single most important signal. For each agent session, count the depth of the tool-call tree (or the length of the tool-call sequence, for linear agents). Emit a histogram.

In healthy operation, depth is strongly right-concentrated — 70–90% of sessions in depth 1–3, a tail in 4–10, rare sessions in depth > 10. The histogram shape is characteristic of the surface.

When a cascade starts, the histogram shifts rightward. A bin that was previously empty (depth 20, 30, 50) becomes populated. Alert on:

- A statistically significant shift in the histogram p95 or p99.
- Appearance of counts in high-depth bins that are normally zero.
- An increase in the "hit the depth cap" count — if the agent has a hard cap at depth 40 and more sessions are hitting it, cascades are happening.

### 3.2 Tool-call error rate

Per tool, per session, the error rate on tool calls. In a cascade, the error rate on the looping tool rises sharply — the agent is calling the failing tool over and over.

Instrument:

- Per-tool error counter (OTel attribute `gen_ai.tool.name` on the tool-call span, plus a result status).
- Per-session error rate (errors / total tool calls).
- Per-tool-per-error-class breakdown (429s, schema-validation errors, timeout errors).

Alert on:

- Per-tool error rate above baseline for a sustained window.
- A spike in session-level tool-call-error rate above threshold.
- A specific tool reaching a fixed per-session error count (e.g., same tool errored 10 times in one session).

### 3.3 Provider-cost per session

For each agent session, aggregate the provider-token cost (input tokens + output tokens, priced per the cost SLI from `mod-103` chapter 04). Emit a per-session cost histogram.

In healthy operation, cost-per-session has a characteristic right-tail — most sessions are cheap, a few are expensive. A cascade makes the tail rise sharply — the p99 cost-per-session doubles.

Alert on:

- Per-session cost above a per-surface hard cap (e.g., "no single session should cost more than $N"; the alert indicates a cap was hit and the session was truncated).
- p99 of per-session cost shifted substantially.
- Daily spend on the agent surface outside the normal envelope.

### 3.4 Wall-clock per session

An agent that is looping is also slow. Session wall-clock histogram shifts right. For user-facing agent surfaces, wall-clock per session is the SLI; a shift is the alert.

### 3.5 Trace-shape signal

In the LLM-trace platform (`mod-104` chapter 04), a cascading session's trace is visually distinctive — a deep or wide tree instead of the expected shallow one. For surfaces where trace-shape visualisation is an operator tool, make the shape itself a reviewable surface. Don't rely on it as an alerting signal; do use it as a triage tool once alerted.

## 4. Joint signals and recognition

A tool-cascade alert should fire on a *joint* condition:

- Tool-call-depth histogram p99 shifted upward *or* depth-cap hits increased, *and*
- Per-session cost or wall-clock shifted upward on the same surface.

Single-signal alerts are noisier: depth shifts for legitimate workload shifts; cost shifts for traffic-volume reasons; wall-clock shifts for any slowness. The joint shifts together only on an actual cascade.

A per-tool error spike on its own is also an alert; it does not require the joint. A tool suddenly 429ing is actionable immediately, before it cascades.

## 5. Diagnosis in the first fifteen minutes

Once a cascade alert fires:

1. **Confirm the joint signal.** Depth histogram shifted; cost or wall-clock shifted; the same surface.
2. **Open a sampled trace.** Pull a sampled high-depth trace from the trace store. Look at the tree. Classify:
   - Same tool called N times with similar args → recursive loop (2.1).
   - Same tool called N times all erroring with 429-shaped errors → rate-limit on nested tool (2.2).
   - Same tool called N times all erroring with schema-validation errors → mis-plumbed schema (2.3).
   - Chain of different tools each erroring → structured-output conformance issue or environment-side error pattern.
3. **Check for a recent deploy on the affected surface.** A prompt change, a tool-descriptor change, a model change, an engine-version change, a framework-version change.
4. **Check per-tool error rates by error class.** Identify the tool and error class if not already clear from the trace.
5. **Check upstream status pages.** For 2.2, the nested tool may be upstream-degraded.
6. **Measure blast radius.** How much of the surface's traffic is cascading; cost rate; wall-clock p99; user-visible impact.

By minute 15: sub-class classification + reversible-lever decision.

## 6. Mitigation and route

### 6.1 Short-term SRE levers

- **Lower the depth cap.** Most agent frameworks expose a max-tool-calls-per-session cap. Lowering it bounds blast radius at the cost of some legitimate sessions terminating early. Reversible.
- **Short-circuit the problem tool.** Temporarily disable the tool at the gateway or framework level; let the agent either succeed without it or fall back.
- **Route around.** If the surface has an alternative model or framework path, shift traffic.
- **Throttle the surface.** Rate-limit sessions on the affected surface to cap the aggregate cost.
- **Rollback the deploy** that introduced the regression — the last prompt / tool-descriptor / framework / model change.

### 6.2 Routing

- **Recursive loop (2.1).** Primary owner: application / prompt layer (the prompt is not producing convergence) and the agent framework owner. Model layer may contribute (model is poor at the reasoning the agent requires).
- **Rate-limit on nested tool (2.2).** Primary owner: application / tool-integration layer (the tool client is not handling 429s correctly) and the nested-tool's owner (may be internal or external). Upstream capacity story if external.
- **Mis-plumbed schema (2.3).** Primary owner: application layer (tool descriptor drift) and tool owner (schema evolved). The inference-engine peer owns structured-output conformance; if the engine's conformance is below spec, it's the engine peer.

### 6.3 Hand-off ticket

The hand-off to the peer should include:

- Sub-class classification.
- Sampled trace IDs (3–5 representative cascading traces).
- Depth-histogram diff (before vs. during).
- Per-tool error-rate breakdown.
- Blast radius: traffic affected, cost impact, wall-clock impact.
- SRE mitigation in place (depth cap, kill-switch, rollback).
- Owning peer's likely investigation scope.

## 7. Specific traps in agent observability

### 7.1 Child spans must have parents

Agent frameworks that emit OTel spans sometimes produce orphaned spans — tool-call spans with no parent (or with a wrong parent). Ensure the agent framework propagates the parent span ID into each tool-call span. Without it, trace trees are flat and the depth histogram is uncomputable from spans.

### 7.2 The tool-call span is not the LLM-call span

Each iteration of the loop generates two spans: one LLM call (the model decides what to do) and one tool call (the tool executes). The tool-call-depth metric should count the *tool* spans, not the *LLM* spans. Both counts are useful; they are not the same.

### 7.3 Session IDs are the join key

Agent observability joins spans by session ID, not by trace ID. One user session typically spans multiple trace trees (each loop iteration may start a fresh trace in some frameworks). The session ID must be on every span as an attribute, or the per-session aggregation breaks.

### 7.4 Nested tools can themselves be agents

A tool that calls another LLM (summariser, classifier, planner) opens a nested agent surface. Observability must propagate through; cost and depth include the nested cost and depth. Otherwise "cost per session" under-reports.

### 7.5 Streaming agents

Agents that stream intermediate tool-call reasoning to the user (visible-thinking surfaces) have the same shape but add the complication of partial results. Observability should distinguish "this tool call ran to completion" from "this tool call was still running when the user cancelled."

## 8. The provider-token-explosion failure

A specific failure shape worth naming: agents with large contexts and small models can produce surprising cost blow-ups. Each iteration of the loop re-reads the full conversation into context. If the conversation accumulates tool results, the input-token count grows linearly per iteration, and cost per iteration grows *linearly* in iteration-count. A 40-iteration cascade can cost 40× more than a 10-iteration cascade, not because the per-call cost went up but because the per-call input context grew.

Operational implication: alerting on per-iteration cost is not enough; alert on per-session cost, which captures the compounding.

## 9. Interaction with other failure classes

### 9.1 Cascades induce prompt-cache cliffs

A cascading agent re-does prefill on each iteration against a different context. The prefix cache may not help; TTFT per iteration regresses. From the on-call's view, both class 2.5 and class 2.7 signals fire. The 2.7 depth signal is usually the leading indicator; the 2.5 cache signal is downstream.

### 9.2 Cascades can mask or produce hallucinations

An agent that tries 30 search queries and finally produces an answer is likely hallucinating — none of the searches gave it the answer and it is extrapolating. The hallucination SLI (2.2) may fire along with the cascade alert.

### 9.3 Cascades can indicate provider-degraded mode

An agent that is converging well on a stable provider may start cascading if the provider's underlying model degrades (2.9) — the model's reasoning falls off a cliff, cannot converge, loops. Cross-reference 2.7 alerts with 2.9 signals on the same model slice.

## 10. The capacity question

Agent surfaces have a capacity-planning question classical SRE does not: a surge of pathological cascades can monopolise provider-token budget or GPU capacity and starve the rest of the fleet. `mod-106` on capacity planning treats this; here the implication is:

- **Per-session cost caps are a reliability lever, not a cost lever.** They prevent one pathological session from starving all others.
- **Per-surface concurrency caps** bound the aggregate cost blow-up.
- **Admission control** on agent surfaces is harder than on chat surfaces because session cost is unknown at admission time.

Instrument so that during a cascade incident, the on-call can see both the per-session cost distribution *and* the aggregate rate the surface is burning provider budget.

## 11. Testing and chaos — the agent program

Agent surfaces are fertile ground for chaos engineering (`mod-109`). The relevant patterns:

- **Fault-injection on tools.** Have a tool deliberately 429 or 500 for a fraction of calls; measure whether the agent retries cleanly or cascades.
- **Schema-drift injection.** Change a tool's schema mid-run; measure whether the descriptor-sync pipeline catches it or the agent cascades.
- **Prompt-regression tests on convergence.** A convergence-eval set of sessions where the agent should finish in ≤ N steps; a scheduled run that measures convergence rate. Signal: convergence dropped.

Each is a scheduled job writing to the agent-observability dashboard. The chaos program's role is to make 2.7 visible before it hits production traffic.

## Summary

- **Agent cascade (class 2.7) is a cost-and-latency failure, not a correctness failure.** Users don't see wrong answers; they see slow, expensive, sometimes-truncated answers.
- **Three sub-classes.** Recursive tool loop (2.1), rate-limit on nested tool (2.2), mis-plumbed tool schema (2.3).
- **Primary signals.** Tool-call-depth histogram (the leading indicator), per-tool error rate, per-session cost, per-session wall-clock.
- **Joint alert.** Depth shift + cost or wall-clock shift on the same surface.
- **First-fifteen-minutes checklist.** Confirm joint signal; sample a cascading trace; classify sub-class; check recent deploys; check upstream status; measure blast radius.
- **SRE mitigations.** Lower depth cap, short-circuit tool, route-around, throttle, rollback. Reversible levers; depth / fix belongs to peers.
- **Route-to.** Application / framework layer (recursion, schema drift), tool owner (rate-limit, schema evolution), model layer or provider (degraded reasoning).
- **Observability traps.** Child-span parenting, session IDs as join key, nested agent surfaces, streaming visible-thinking agents, token-explosion as compounding.
- **Capacity interaction.** Per-session caps and per-surface concurrency caps are reliability levers.

The next chapter covers the two remaining classes — judge-model drift and provider-degraded mode — where the signal is uncertain by its nature: the measurement itself is suspect, or the provider is a black box that is degrading without changing its contract.
