# Load Testing as Capacity Verification

## Why this chapter exists

A capacity forecast is a prediction, an SLO is a promise, and an autoscaler is a bet that the forecast and the SLO will agree in production. Load testing is the single mechanism that closes the loop between the three — the thing you do to **verify** that the fleet behaves the way the forecast says it will and that the autoscaler holds the SLO under real pressure. Without periodic load testing, the forecast is faith and the autoscaler is an untested PID loop.

Load testing an LLM fleet is not the same as load testing a classical web service. The traffic generator has to speak the engine's API shape (chat completions, streaming, tool-use), produce realistic token distributions (not uniform 100-token prompts and 100-token completions), simulate prefix-cache behaviour correctly (not generate unique prompts every time and accidentally test cache-miss throughput), and measure the per-token latency signals the SLO cares about (TTFT and TPOT, not just end-to-end latency). The tooling has caught up — k6 has LLM-shaped extensions, Locust has user-class patterns that fit, open-source LLM-specific benchmarks (LLMPerf, GenAI-Perf, vLLM's own `benchmark_serving.py`) exist and are usable. The discipline is what this chapter builds.

## 1. What load testing is for

Three distinct uses for load testing on an LLM fleet, each with different setup and different acceptance criteria:

### 1.1 Capacity verification

The quarterly or pre-launch check that the fleet can serve the forecast's peak. The test replays a traffic shape matching the production forecast, ramps to peak, confirms SLOs hold, and records the headroom. The output is a go/no-go decision and an updated throughput curve for the forecast.

### 1.2 Pareto-curve exploration

The characterization of the fleet's throughput/latency trade-off. Push the fleet past the operating point until the SLO visibly breaks, plot `(throughput vs. TTFT p95)` and `(throughput vs. TPOT p95)`, and identify the knee of each curve. The output is a mapping from "sustained request rate" to "latency distribution" that the forecast consumes.

### 1.3 Chaos-adjacent capacity verification

Running capacity load tests while injecting failures — a node down, a replica slow, a provider returning 429s. Confirms the fleet degrades gracefully under capacity pressure, not just that it holds at steady state. Overlaps with `mod-109`; cross-reference section 7 below.

The three uses share tooling but need different test designs. Mixing them in one test produces unclear results.

## 2. Traffic-shape fidelity

The single most common load-testing mistake on LLM fleets is **unrealistic prompt and completion shapes**. A test that fires 1000-token prompts generating 500-token completions, drawn from random tokens, uniformly at a target rate, tells you almost nothing about how the production fleet will behave. The reasons:

- **Random tokens defeat prefix cache.** A production workload with 65% prefix-cache hit rate shows a different throughput than the same arithmetic workload with 0% hit rate. Your random-prompt test measures the 0% case.
- **Uniform distribution misses the tail.** Production has a long tail in prompt and completion length; the tail drives p99 behaviour. A uniform test misses it.
- **Constant inter-arrival time misses burstiness.** Production is bursty; a uniform-rate test misses HOL-blocking scenarios and cold-start behaviour.

### 2.1 Capturing a production traffic shape

The AI SRE pulls three artifacts from production to build a realistic test:

- **Prompt and completion length distributions** per workload class. Histograms, not means. From the `mod-104` metrics pipeline.
- **Prefix-cache shape.** The cacheable-prefix fraction of prompts (system message, tool descriptions, repeated retrieved documents). Reconstruct a test that pre-loads cacheable prefixes before measurement.
- **Inter-arrival pattern.** Poisson for uniform workloads, Markov-modulated Poisson for bursty workloads with on/off states, and time-of-day scaling for workloads with daily cycles. For a one-hour capacity test, a ramp-plus-sustain pattern is usually enough; for a full-day test, replay the actual arrival pattern.

Then build the test from these artifacts. The generator samples prompt length from the production histogram, generates a prompt that shares a prefix with the pool of cacheable prefixes at the production hit rate, and fires at the target rate with the production arrival pattern.

### 2.2 The "replay from production" shortcut

For some workloads, a better shortcut is **replay-from-production**: record a sample of production requests (with PII scrubbed, per the data-handling policy — see `mod-110`), and replay them against a staging fleet. The traffic shape is by construction realistic. Costs: a replay pipeline that handles PII correctly, a staging fleet large enough to be representative, and a separate judge-quality pipeline to evaluate the responses (replay does not know the right answer).

Replay works well for traffic-shape fidelity; it works poorly for peak-load exploration because the replayed rate is bounded by the production rate. Use replay for *shape*, synthetic for *volume*, and combine them for most capacity tests.

## 3. The k6 pattern

k6 is a load-testing tool from Grafana Labs with native support for scripted traffic, including streaming HTTP responses and the metrics the LLM-serving domain cares about. The AI SRE uses k6 for most new load tests because the scripting model fits LLM scenarios and the metrics export integrates cleanly with the existing observability stack.

### 3.1 A minimal k6 chat-streaming test

A sketch of what a k6 test for a chat endpoint looks like (illustrative, not a drop-in script):

```javascript
import http from 'k6/http';
import { Trend, Counter } from 'k6/metrics';
import { sleep } from 'k6';

const ttft = new Trend('llm_ttft_ms', true);
const tpot = new Trend('llm_tpot_ms', true);
const totalTokens = new Counter('llm_output_tokens');

export const options = {
  stages: [
    { duration: '2m', target: 50 },    // ramp
    { duration: '10m', target: 200 },  // ramp to target rate
    { duration: '20m', target: 200 },  // sustain
    { duration: '5m', target: 0 },     // ramp down
  ],
  thresholds: {
    'llm_ttft_ms': ['p(95)<2000'],
    'llm_tpot_ms': ['p(95)<60'],
    'http_req_failed': ['rate<0.01'],
  },
};

export default function () {
  // sample from production-shape distributions
  const prompt = samplePrompt();
  const maxTokens = sampleCompletionLength();
  const body = JSON.stringify({
    model: 'chat-model-v1',
    messages: [
      { role: 'system', content: systemPrompt },  // cacheable prefix
      { role: 'user', content: prompt },
    ],
    stream: true,
    max_tokens: maxTokens,
  });

  const start = Date.now();
  const res = http.post('http://gateway.example/v1/chat/completions', body, {
    headers: { 'Content-Type': 'application/json' },
    responseType: 'none',
  });
  // parse streaming response, record TTFT on first token, TPOT across tokens
  parseStream(res, start, ttft, tpot, totalTokens);
}
```

Key points:

- **Streaming is parsed per-token.** TTFT is the wall-clock from request start to first data byte; TPOT is the inter-token delta averaged across the completion. Both are first-class LLM-SLO signals (see `mod-103` chapter 01).
- **Stage shape is a ramp-plus-sustain.** A short ramp warms the fleet; a long sustain produces the steady-state measurements. Hard ramps can trip autoscalers (which may be the point) but are not the right shape for a steady-state capacity verification.
- **Thresholds encode the SLOs.** If a threshold is violated, k6 exits non-zero. The capacity verification's go/no-go gate is k6's exit code.

### 3.2 Caveats

- **k6 does not natively understand LLM API shape.** You write the parser. Open-source helpers exist; verify the current state and vet one before reinventing. LLMPerf, GenAI-Perf, and vLLM's `benchmark_serving.py` package the parsing for common APIs.
- **k6's VU model maps to concurrent users, not concurrent requests.** For LLM streaming, a VU is tied up for the full response duration. If your target is 200 concurrent requests and each request takes 10 seconds, you need 200 VUs to hold the load. For target RPS calculations, convert via Little's Law: `VUs ≈ RPS × response_time`.
- **k6's output to Prometheus / Grafana is first-class.** Ship the per-stream metrics into the same observability stack the SLOs use. The test's dashboard should look like the production dashboard, because the same queries answer both.

## 4. The Locust pattern

Locust is a Python load-testing framework with a similar role to k6. The AI SRE's reason to prefer Locust over k6 is usually language (Python ecosystem integration — numpy for sampling from production distributions, Pandas for post-test analysis). The pattern is similar:

```python
from locust import FastHttpUser, task, between
import time, json, random

class ChatUser(FastHttpUser):
    wait_time = between(0.5, 2.0)

    @task
    def chat(self):
        prompt = sample_prompt_from_production_dist()
        max_tokens = sample_completion_length()
        body = json.dumps({
            "model": "chat-model-v1",
            "messages": [
                {"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user", "content": prompt},
            ],
            "stream": True,
            "max_tokens": max_tokens,
        })
        start = time.monotonic()
        first_token_ts = None
        inter_token_deltas = []
        prev = None
        with self.client.post(
            "/v1/chat/completions",
            data=body,
            headers={"Content-Type": "application/json"},
            stream=True,
            catch_response=True,
        ) as resp:
            for line in resp.iter_lines():
                if not line:
                    continue
                now = time.monotonic()
                if first_token_ts is None:
                    first_token_ts = now
                    ttft_ms = (first_token_ts - start) * 1000
                    self.environment.events.request.fire(
                        request_type="LLM", name="chat-ttft",
                        response_time=ttft_ms, response_length=0,
                    )
                else:
                    inter_token_deltas.append((now - prev) * 1000)
                prev = now
        if inter_token_deltas:
            tpot_ms = sum(inter_token_deltas) / len(inter_token_deltas)
            self.environment.events.request.fire(
                request_type="LLM", name="chat-tpot",
                response_time=tpot_ms, response_length=0,
            )
```

The event model is coarser than k6's but adequate for most purposes. Post-test analysis in a Jupyter notebook against Locust's CSV export or the fired-events stream is the natural shape.

## 5. Pareto-curve exploration — the deliberate overload

Capacity verification confirms the fleet holds the SLO at the forecast's peak. Pareto exploration **finds where the SLO breaks**, so you know how much margin you have and what the break looks like.

The pattern:

- Start at 0.5× forecast peak; ramp slowly (e.g., 1 minute per step) through 0.7×, 0.9×, 1.0×, 1.2×, 1.5×, 2.0× of forecast peak.
- At each step, sustain for enough time to reach steady state (5–10 minutes is usually enough once autoscaler stabilisation windows elapse).
- Record TTFT p50, p95, p99; TPOT p50, p95, p99; queue depth per replica; replica count; error rate; prefix-cache hit rate.
- Plot the curves. The knee is where TTFT starts moving faster than linearly with load.

Two useful shapes the Pareto curve reveals:

- **Soft knee.** TTFT rises gradually past the design point; the fleet degrades gracefully. The autoscaler catches up before the SLO is seriously violated.
- **Hard knee.** TTFT jumps sharply at a specific load; the fleet fails catastrophically beyond the design point. Usually caused by hitting a KV-cache limit, a queue-admission cap, or a provider rate limit downstream.

A hard knee is a red flag. The autoscaler must engage well before it; the forecast's headroom must keep the operating point well below it. Chapter 01 section 3.1 is why: the queueing-theory superlinear-wait curve is what the hard knee looks like in practice.

### 5.1 The Pareto curve is the autoscaler's calibration input

Section 3.2 of chapter 05 said the HPA target comes from a measurement: "at queue depth 4, TTFT p95 is 1.4s." That measurement is produced during the Pareto curve exploration. The capacity verification and Pareto exploration are the *only* place those numbers come from; otherwise the autoscaler is tuned by vibes.

## 6. Headroom confirmation and the go/no-go gate

Once the Pareto curve exists, the capacity verification becomes concrete:

- **Design point.** The forecast's projected peak. The fleet must hold the SLO here with the autoscaler at its steady-state sizing.
- **Headroom point.** The forecast's projected peak × the headroom multiplier (e.g., 1.5× from chapter 02 section 4). The fleet must hold the SLO here after the autoscaler has scaled up.
- **Break point.** The Pareto curve's knee. The fleet must have headroom to the break point large enough that a forecast-miss does not put the production fleet in the break region.

The go/no-go decision is binary:

- **Go** if all three hold.
- **No-go** if any fails. Fix is one of: more capacity, tighter autoscaler, revised forecast, revised SLO.

Record the three points in the capacity-verification artifact. The production fleet's operating point should sit well below all three.

## 7. Wiring load tests into the SLO stack and the chaos program

### 7.1 SLO integration

Load tests produce the same metrics the SLO queries read. The pattern:

- The test's Prometheus data is tagged with a `test_run_id` label and left in a separate datasource (or a separate retention class) from production metrics. Not mixed into the SLO's compliance window.
- The SLO's burn-rate alerts are temporarily muted for the test environment during the test run, not for production.
- Post-test, the test's metrics are *compared* against the SLO's thresholds to produce the headroom report. The report is what gets committed to the capacity-review artifact.

Do not run capacity load tests against production. Run them against staging environments that mirror production shape (same model, same engine, same quantisation, same tensor-parallel degree, same hardware generation). The AI SRE's job during a production load test — which happens, but rarely, and only after staging is exhausted — is to scope blast radius: canary a subset of the fleet, shed load if SLO starts to burn, hold the chaos-program safety rails from `mod-109`.

### 7.2 Chaos program integration

`mod-109` covers chaos engineering for AI. The chapter-6-plus-chaos combination is a specific pattern:

- Run a capacity test at the design point.
- Inject a failure (one replica slow, one AZ unavailable, provider returning 429s, prefix-cache invalidated).
- Confirm the SLO still holds, or if it does not, measure the degradation.

The combined test confirms the fleet's **degraded mode** is sized correctly, not just the happy-path. A fleet that holds at peak only when every replica is healthy and the provider is clean is a fleet that will burn its error budget the first time real conditions bite.

## 8. Cadence and ownership

Capacity verification is a quarterly ritual, tied to the forecast cycle (chapter 02 section 6). Minor cadence suggestions:

- **Quarterly full test.** Replay production shape, ramp to peak, run Pareto exploration, produce the artifact.
- **Monthly smoke test.** A 15-minute test at 70% peak, confirming nothing has silently regressed (model promotion, kernel update, cluster change). Catches regressions between quarters.
- **Pre-launch test.** Before a product launch that changes forecast inputs, run the capacity test with the new shape.
- **Post-incident test.** If an incident revealed a capacity miss, re-run the capacity test with the incident's traffic shape before closing the retrospective.

Ownership: the AI SRE team owns the harness, the production-shape sampling, and the go/no-go gate. The product teams own the forecast input (what traffic they are sending next quarter). The engine peer and the performance peer own the engine-side configuration being tested; they are co-signers on the artifact.

## 9. The capacity-verification artifact

Every quarterly verification produces a one-page artifact:

- **Date, model, engine version, hardware generation, fleet size.**
- **Forecast input.** The projected peak and headroom that the test is verifying.
- **Test shape.** Prompt / completion distributions used; prefix-cache profile; inter-arrival pattern.
- **Results at design point.** TTFT p50 / p95 / p99, TPOT p50 / p95 / p99, queue depth, replicas, error rate.
- **Results at headroom point.** Same metrics; confirmation of SLO hold.
- **Pareto curve.** Table or plot of load-vs-TTFT.
- **Knee location.** Where TTFT breaks; how much headroom to the knee.
- **Go/no-go decision.** With the deltas that would need remediation on no-go.
- **Autoscaler-tuning inputs.** Updated HPA target values derived from the measurement.
- **Known deviations from production.** Where the staging fleet differs and the uncertainty that adds to the result.

The artifact feeds the forecast (updates throughput curves), the autoscaler plan (updates target values), and the SLO review (confirms headroom to SLO). The three artifacts stay in sync, revision by revision.

## Summary

- **Load testing closes the loop between forecast, SLO, and autoscaler.** Without it, the forecast is faith; with it, the forecast is calibrated.
- **Traffic-shape fidelity is the first requirement.** Prompt and completion distributions from production, prefix-cache profile reconstructed, inter-arrival pattern matched. Random prompts measure cache-miss throughput, which is not the number you care about.
- **k6 and Locust are both fine.** k6 if you want a scripting-first harness with Grafana-native metrics; Locust if you want Python integration with production's data tools. Both need streaming-aware TTFT / TPOT parsing. LLM-specific helpers (LLMPerf, GenAI-Perf, vLLM's benchmark harness) exist and are worth starting from.
- **Pareto exploration finds the knee.** Capacity verification confirms the design point; Pareto exploration finds where the SLO breaks. Both are required; neither alone is enough.
- **The go/no-go gate is explicit.** Design point, headroom point, break point. All three must hold.
- **SLO integration: tag test metrics, mute test-environment alerts, write a headroom report.** Do not pollute the production SLO's compliance window.
- **Chaos integration: capacity + injected failure.** Confirm the degraded-mode capacity, not just the happy-path.
- **Quarterly full, monthly smoke, pre-launch, post-incident.** The cadence matches the forecast's cadence.
- **The artifact is one page and feeds three others.** Forecast, autoscaler plan, SLO review all consume it.

The next chapter closes the module by naming the peer boundaries explicitly — where the AI SRE's capacity-planning role stops and the inference-engineer / performance / base-K8s peers' depth begins.
