### exercise-01: App-Altitude Latency Report — TTFT / TPOT — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to replace a single "response time" scalar with the **five-number latency vocabulary** (TTFT, TPOT, total, retry, tool round-trip) on a streaming chat surface, drive a representative traffic sample through the instrumented path, and write a **per-surface latency spec** with green / yellow / red budgets that a peer can sign off on. The deliverable is not a chart — it is the artefact that ends every future debate about whether "the surface is fast enough."

The failure mode the exercise is written to catch is the one from the problem statement: a dashboard that aggregates total response time while a provider region quietly tails out, retries hide inside the average, and user-perceived latency rots for hours before an on-call is paged. That failure is architectural, not alerting: the shape of what the dashboard measures is wrong, so a better threshold on the wrong metric cannot save the surface. The fix is to measure the five numbers separately — because they correspond to different user-visible failures and to different levers you can pull.

This solution provides:

- A **measurement vocabulary** — the five latency numbers, their exact definitions, and the span attribute keys they map to (OpenTelemetry GenAI semantic conventions where applicable).
- A **reference instrumentation module** — Python using `time.perf_counter()` around a streaming SSE client, emitting one OTel span per call with the required fields.
- A **traffic-driver sketch** — a repeatable 200–500 request sample with mixed prompt lengths and at least one tool-invoking flow.
- A **worked latency-spec example** for a reference surface (`helpdesk-reply-copilot`) with real-shaped numbers, defensible thresholds, and the retry-and-tool separation report.
- A **grading rubric** for the instructor.

The lesson the exercise defends: *a surface without TTFT / TPOT separation cannot own its latency*. Once the five numbers land on one span, every downstream exercise in this module — the SLI in [[exercise-05]], the cache-delta story in [[exercise-02]], the CPSO work in [[exercise-04]] — has a queryable foundation to run against.

## 2. Worked answer or implementation

### 2a. Measurement vocabulary (deliverable 1 — instrumentation)

Each LLM call emits one span. No sub-spans per SSE chunk. The fields below are the floor; add more only if downstream analysis requires it.

| Attribute | Definition | Unit / type |
|---|---|---|
| `gen_ai.system` | Provider identifier (`openai`, `anthropic`, `bedrock`, …) | string |
| `gen_ai.request.model` | Model requested at call site (e.g. `claude-sonnet-4-6`) | string |
| `gen_ai.response.model` | Model actually served (may differ from requested after fallback) | string |
| `gen_ai.usage.input_tokens` | Prompt tokens on the served response | integer |
| `gen_ai.usage.output_tokens` | Completion tokens | integer |
| `app.latency.ttft_ms` | `t_first_content_delta - t_handed_to_transport` | integer ms |
| `app.latency.tpot_ms` | `(total_body_ms - ttft_ms) / max(1, n_output_tokens)` | integer ms |
| `app.latency.total_ms` | `t_stream_closed - t_handed_to_transport` on the successful attempt | integer ms |
| `app.latency.retry_ms` | Sum of wall-clock across any failed attempts before the successful one; `0` on first-attempt success | integer ms |
| `app.retry.attempt_number` | Ordinal of the attempt that returned `total_ms` (1 on first-attempt success) | integer |
| `app.tool.name` | Logical tool name, present only when a tool was invoked in this turn | string, optional |
| `app.tool.round_trip_ms` | `t_tool_resumed_LLM - t_tool_call_fired` for the slowest tool invocation in this turn | integer ms, optional |
| `app.request_id` | Correlation id across tenant → gateway → provider | string |
| `app.tenant_id` | Opaque per-tenant identifier (hashed if PII-sensitive) | string |
| `app.feature` | Feature registry key (see [[exercise-04]]) | string |

`gen_ai.*` attribute keys come from the OpenTelemetry GenAI semantic-conventions working group; the `app.*` keys are local to this surface. The split matters: the `gen_ai.*` keys are portable across any OTel-conformant backend; the `app.*` keys encode surface-specific decisions that this exercise is pinning.

Measurement points, worded unambiguously:

- **Handed to transport**: the moment immediately before the SDK's `send` / `create_stream` call. Not before the SDK is constructed, not before the request body is serialised — those are too early. Serialisation time is small for a chat payload and belongs on the client-side budget, not the LLM span.
- **First content delta**: the first SSE chunk whose structured content is **non-empty**. Providers emit keep-alive events, role-only deltas, and empty-choice frames; those are not TTFT. If the provider streams a role marker before any text, measure to the first chunk with actual tokens.
- **Stream closed**: either the `[DONE]` sentinel, the SDK's final yield, or the provider's end-of-stream event — whichever arrives last. On a cancelled stream, record `app.outcome="cancelled"` and set `total_ms` to the time of cancellation; do not treat cancellation as a timeout unless the gateway cancelled, not the caller.
- **Retry latency**: wall-clock elapsed across the failed attempts' request + backoff + reconnect, **not** the number of retries. A single 2-second backoff that recovered on the second attempt is `retry_ms=~2000`, `attempt_number=2`.
- **Tool round-trip**: the time from the moment the LLM response says "call `tool_x`" to the moment the surface's subsequent LLM call carrying the tool result is handed to transport. This is **not** tool execution time — it includes queueing, validation, auth, and any surface code between the tool-call parse and the follow-up LLM request.

### 2b. Reference instrumentation

```python
"""instrumentation.py — one-span-per-LLM-call for a streaming chat surface.

Clocks use time.perf_counter() (monotonic). The gateway owns the timer; the
SDK never does — provider-reported latencies describe the provider's path,
not the user's.
"""
from __future__ import annotations

import json
import time
from contextlib import contextmanager
from dataclasses import dataclass, field

from opentelemetry import trace

tracer = trace.get_tracer("helpdesk-reply-copilot")


@dataclass
class LatencyRecord:
    t_sent: float
    t_first_content: float | None = None
    t_closed: float | None = None
    retry_ms: int = 0
    attempt_number: int = 1
    tool_name: str | None = None
    tool_round_trip_ms: int | None = None
    input_tokens: int = 0
    output_tokens: int = 0
    finish_reason: str | None = None


@contextmanager
def latency_span(*, system: str, request_model: str, tenant_id: str, feature: str, request_id: str):
    rec = LatencyRecord(t_sent=time.perf_counter())
    with tracer.start_as_current_span("llm.chat") as span:
        span.set_attribute("gen_ai.system", system)
        span.set_attribute("gen_ai.request.model", request_model)
        span.set_attribute("app.tenant_id", tenant_id)
        span.set_attribute("app.feature", feature)
        span.set_attribute("app.request_id", request_id)
        try:
            yield rec
        finally:
            ttft_ms = int((rec.t_first_content - rec.t_sent) * 1000) if rec.t_first_content else -1
            total_ms = int((rec.t_closed - rec.t_sent) * 1000) if rec.t_closed else -1
            body_ms = total_ms - ttft_ms if ttft_ms >= 0 and total_ms >= 0 else -1
            tpot_ms = int(body_ms / max(1, rec.output_tokens)) if body_ms > 0 else -1
            span.set_attribute("app.latency.ttft_ms", ttft_ms)
            span.set_attribute("app.latency.tpot_ms", tpot_ms)
            span.set_attribute("app.latency.total_ms", total_ms)
            span.set_attribute("app.latency.retry_ms", rec.retry_ms)
            span.set_attribute("app.retry.attempt_number", rec.attempt_number)
            if rec.tool_name:
                span.set_attribute("app.tool.name", rec.tool_name)
                span.set_attribute("app.tool.round_trip_ms", rec.tool_round_trip_ms or -1)
            span.set_attribute("gen_ai.usage.input_tokens", rec.input_tokens)
            span.set_attribute("gen_ai.usage.output_tokens", rec.output_tokens)
            if rec.finish_reason:
                span.set_attribute("gen_ai.response.finish_reason", rec.finish_reason)


def iter_stream_with_timing(stream, rec: LatencyRecord):
    """Wrap an SSE iterator and stamp t_first_content / t_closed / usage onto `rec`."""
    for event in stream:
        delta = _content_delta(event)  # provider-specific, returns None or a non-empty string
        if delta and rec.t_first_content is None:
            rec.t_first_content = time.perf_counter()
        yield event
        if _is_final(event):
            rec.t_closed = time.perf_counter()
            rec.output_tokens = _usage_output(event)
            rec.input_tokens = _usage_input(event)
            rec.finish_reason = _finish_reason(event)
```

### 2c. Traffic-driver sketch (deliverable 2)

```python
# driver.py — drives 200+ requests through the instrumented surface.
# Keep cost trivial: cheap model, bounded output tokens.

MIX = [
    ("short-chat", "what's the status of my order?",                        40),
    ("medium-chat", "summarise the last three messages in this thread…",   120),
    ("long-context", LONG_TICKET_BODY,                                      400),
    ("tool-flow", "look up order ORD-1024 and quote shipping",              80),  # fires a tool
]

def run_sample(n_per_bucket=60, run_id="2026-11-14-A"):
    for label, prompt, max_out in MIX:
        for i in range(n_per_bucket):
            request_id = f"{run_id}-{label}-{i:04d}"
            call_the_surface(prompt=prompt, max_out=max_out,
                             tenant_id=rotate_tenants(i),
                             feature="helpdesk.reply",
                             request_id=request_id)
```

Mixing prompt lengths is required: a sample that is 100% short chat under-reports input-token-sensitive tail variance, and a 100% long-context sample over-reports it. Rotate across ~5 tenants so the run touches rate-limit paths incidentally. Split the sample across two windows a few hours apart — a single-window sample buys you one slice of provider tail behaviour; two windows catch the "quiet afternoon vs busy morning" delta that's responsible for most week-over-week regressions.

### 2d. Worked latency spec for a reference surface

The numbers below are plausible-shape example output for the `helpdesk-reply-copilot` reference surface used across this module. They are included to show the *form* of a defensible spec. A submission against a different surface will produce different numbers; the shape is what the rubric scores.

```markdown
# Latency spec — helpdesk-reply-copilot — 2026-11-14

## Surface shape
A streaming chat surface: an agent types a question, the copilot streams a
suggested reply. "The user is waiting" means TTFT until the first token
appears in the compose box; TPOT until the reply finishes streaming.
Tool calls (order lookup, CRM search) are allowed mid-stream.

## Budgets (rolling 5-minute p95, prod)
| Metric                       | Green  | Yellow      | Red    |
| ---------------------------- | ------ | ----------- | ------ |
| TTFT p95                     | < 900ms  | 900–1500ms  | > 1500ms |
| TPOT p95                     | < 40ms   | 40–80ms     | > 80ms   |
| Total latency p95            | < 4500ms | 4500–7000ms | > 7000ms |
| Retry latency p95            | 0ms     | 0–800ms     | > 800ms  |
| Tool-call round-trip p95     | < 600ms  | 600–1200ms  | > 1200ms |
| Tool-call round-trip p99     | < 1500ms | 1500–3000ms | > 3000ms |

## Excluded from these budgets
- Warm-up traffic (< 60s since pod start)
- Requests with input_tokens > 60k (long-context explicit opt-in flow)
- Known-tenant backfill jobs tagged `app.feature=helpdesk.backfill`

## Where the numbers come from
- OTel span `llm.chat`, retention 30d, dashboard: /d/helpdesk-reply-latency
- Query: `histogram_quantile(0.95, sum(rate(gen_ai_latency_ms_bucket{feature="helpdesk.reply"}[5m])) by (le, metric))`

## Owner and revisit
Owner: helpdesk-platform. Reviewed: 2026-11-14. Next review: 2027-02-01.
```

**How each threshold is defensible:**

- **TTFT red at 1500 ms**: the copilot is embedded in an agent's compose box; usability research across similar surfaces shows typing hesitation on the agent side begins around 1.5–2 s, so TTFT > 1.5 s is a UX break, not a latency ratio.
- **TPOT red at 80 ms/token**: average English reading speed caps useful streaming at roughly 25–40 tokens/second (≈ 25–40 ms/token); doubling that is where an agent starts reading faster than the copilot streams.
- **Total red at 7 s**: a 200-token reply at TPOT 80 ms plus TTFT 1.5 s would yield ~17.5 s; the 7 s ceiling encodes "the reply must fit the agent's attention window, which is shorter than the raw math."
- **Retry red at 800 ms**: a healthy retry budget is a single sub-second backoff; retry_ms > 800 ms at p95 means retries have stopped being rare and are now dragging the distribution.
- **Tool round-trip red at 1.2 s (p95) / 3 s (p99)**: the surface tools are auth + lookup + JSON back; anything at the second-plus mark at p95 means the tool, not the model, is the latency source.

### 2e. Retry-and-tool report (deliverable 4)

```markdown
# Retry and tool separation report — 2026-11-14 — run 2026-11-14-A/B

## Scope
- 480 requests across two windows (morning + evening).
- Four prompt-length buckets (short / medium / long-context / tool-flow).
- Two tenants rotating; same provider; same model pin.

## Retries
- 27 / 480 requests retried at least once (5.6%).
- p95 TTFT, first-attempt only: 870 ms
- p95 TTFT, all requests:       1340 ms
- Delta: 470 ms, ~54% uplift. Retries explain more than half of the
  gap between the "typical" TTFT the dashboard shows and the p95 budget.

## Tool round-trip fraction
- 120 tool-invoking turns in the sample.
- On the p95 tool turn: total_ms = 5200 ms, tool_round_trip_ms = 1050 ms.
- Tool round-trip is ~20% of the user-perceived latency on the slow turn.
- The slowest tool_round_trip_ms samples cluster on `crm.search`; model
  wait time on that path is dominated by the tool, not the LLM.

## Conclusion
Retries are the biggest single hidden source of user-perceived latency
on this surface. The tool-round-trip signal is strong enough that the
`crm.search` tool should be considered for a prewarm path (chapter 02's
cache-shape work does not help this).
```

### 2f. Implementation notes and gotchas

- **`time.perf_counter()`, not `time.time()`.** Wall-clock skew across a request boundary (NTP slew, container pause) can produce negative TTFT or impossibly fast TPOT; `perf_counter()` is monotonic and avoids both.
- **Compute p95 with a real percentile estimator.** `sorted(xs)[int(0.95 * len(xs))]` is a nearest-rank estimate that systematically underestimates; use `numpy.percentile(xs, 95, method="linear")` (NumPy renamed `interpolation=` → `method=` in 1.22) or your backend's native histogram quantile. For streaming ingestion, a HDR-histogram or the OTel `ExplicitBucketHistogram` with a reasonable bucket set is standard.
- **Measure from the gateway, not the SDK.** The provider's own `timings` metadata describes the provider side of the wire; the user's clock starts when the gateway sends and ends when the user sees content.
- **One span, fixed attribute set.** Sub-spans per SSE chunk explode cardinality on the backend without adding analytic power; the `t_first_content_delta` timestamp captured during streaming is sufficient.
- **Tool round-trip is not tool execution.** It includes everything between the parse and the follow-up LLM call, so surface-side bugs (e.g. a slow permission check) show up on it. That is by design.
- **Retry-latency separation matters more than any other decomposition.** A single retry inside a p95 is a ~50% TTFT uplift on this exercise's sample; teams that collapse "response time" into a scalar watch the retry rate silently eat their budget.
- **Keep the sample cheap.** A few hundred requests on a small model is enough to stabilise p95; chasing p99 with the paid API is false precision. Use the sample size the surface can rerun weekly.

### 2g. Mapping to the exercise deliverables

| Exercise asks for | Section of this solution |
|---|---|
| Instrumentation (five numbers, one span) | 2a + 2b |
| Traffic sample (200–500 reqs, mixed shapes) | 2c |
| `docs/latency-spec.md` | 2d |
| `docs/retry-and-tool-report.md` | 2e |
| Starter guidance highlights | 2f |

## 3. Validation steps

1. **Span emission check.** Run the driver with `n_per_bucket=5` against a local collector; confirm one span per LLM call with every required attribute populated. A missing field is a hard failure, not a warning.
2. **TTFT monotonicity check.** `app.latency.ttft_ms <= app.latency.total_ms` on every span; violations indicate a clock bug (`time.time()` used somewhere) or a transport-wrap mismatch.
3. **Percentile estimator check.** Compute p95 TTFT both by `numpy.percentile(..., method="linear")` and by `sorted[int(0.95*n)]`; if the delta exceeds 5% the sample is too small to publish and the spec is premature.
4. **Retry-separation check.** Filter to `attempt_number==1`; verify p95 TTFT on that slice is strictly lower than on the whole population, and report both numbers. If the first-attempt p95 is not lower, the surface is not actually retrying (and some bug is hiding retries elsewhere).
5. **Tool-span integrity.** For every span with `app.tool.name` set, `app.tool.round_trip_ms` must be a positive integer. Negative or missing means the surface started a tool call it never finished — investigate before publishing.
6. **Threshold defensibility.** Every yellow / red cell has a one-sentence justification tied to surface UX, not aesthetics.
7. **Exclusion list is finite and named.** "We exclude backfill traffic" is fine; "we exclude outliers" is not — outliers are the tail you built the spec to catch.
8. **Review cadence present.** Owner by role (not by name), revisit date within one quarter.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Instrumentation emits one span per call | 1 | Five numbers on every span; `gen_ai.*` and `app.*` namespaces respected. |
| TTFT definition used correctly | 1 | Measured to first **non-empty** content delta; role-only chunks skipped. |
| Retry latency isolated from first-attempt latency | 1 | First-attempt filter present and reported separately. |
| Tool round-trip measured as surface-to-surface | 1 | Includes queue + validation + auth, not just tool execution. |
| Traffic sample covers ≥ 3 prompt shapes | 1 | Short + medium + long; at least one tool-invoking flow. |
| Percentile computed with a real estimator | 1 | `numpy.percentile(... method="linear")` or histogram-quantile; no nearest-rank. |
| Latency spec table complete | 1 | All six rows filled with real numbers and three-column budgets. |
| Thresholds defensibly justified | 1 | Each yellow/red cell has a one-sentence UX justification. |
| Retry-and-tool report shows first-attempt vs whole-population delta | 1 | Both p95 TTFTs reported, delta named. |
| Owner and revisit date present | 1 | Owner by role; next-review date ≤ one quarter out. |

A submission scoring < 6 is rejected. Hard failures: (a) no retry/first-attempt split, (b) no justification for yellow/red thresholds, (c) a single "response_time_ms" scalar instead of the five numbers.

## 5. Common mistakes

- **Treating the SDK's reported latency as TTFT.** The SDK's number is from the SDK's perspective; the user's clock is the gateway's. Use `perf_counter` around the SDK call.
- **Firing TTFT on the role-only chunk.** Many providers emit a frame with `{role: "assistant"}` and no content before the first token. That is not a token; the user sees nothing. TTFT is to the first non-empty content delta.
- **Collapsing retries into total latency.** A single transient 5xx with a 1.5 s backoff on an otherwise p50 request looks like a p95 outlier on `total_ms`. Separate it.
- **Sub-spans per SSE chunk.** Backends optimise for cardinality, not count; a span per chunk is thousands of spans per tenant per day with no analytic payoff beyond the first-content timestamp you already captured.
- **Nearest-rank p95.** A 200-sample sorted array indexed at `int(0.95 * 200) = 190` returns the 190th value, which systematically underestimates p95. Use a linear-interpolated percentile.
- **Running 10k requests "to get better percentiles."** For a cheap-model sample, 200–500 stabilises p95 to within a few percent; chasing p99 with the paid API is the wrong tool — a longer rolling window from production traffic is.
- **Aesthetic thresholds.** "Green < 1s because it looks nice" is not a budget; "Green < 900ms because the compose-box UX breaks around 1.5s and we need headroom for retries" is.
- **No excluded-population list.** Every surface has traffic shapes outside the SLO (warm-up, backfill, opt-in long-context). Listing them is what keeps the spec honest.
- **Clock skew via `time.time()`.** A monotonic clock is the floor; otherwise an NTP slew during the request will produce negative TTFT in the logs.
- **No owner / no revisit.** A spec without an owner is a document with no one to update it when the surface changes.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the latency decomposition this exercise produces is the measurement substrate that supports ML06 (AI Supply Chain) governance: separating provider-attributable latency (TTFT / TPOT / retry_ms) from surface-attributable latency (tool round-trip) is what lets a provider-swap decision from [[exercise-02]] of mod-108 be made with evidence rather than vibes.
- MITRE ATLAS — <https://atlas.mitre.org/> — the retry-latency signal in particular is an operational indicator ATLAS tactic categories draw on: a sustained spike in `app.latency.retry_ms` on a specific route is correlated with either provider rate-limiting (Impact / Denial of ML Service, adjacent) or an abuse pattern upstream (Resource Hijacking). Keeping the signal on the span means the surface's incident response has the data without new instrumentation.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the latency spec is a MEASURE-function artefact (MEASURE 2.3 "Monitoring of AI system performance across its deployed context") and the MANAGE-function hand-off into [[exercise-05]]'s SLO. The owner and revisit fields map to GOVERN's accountability-and-cadence requirements.
- OpenTelemetry GenAI semantic conventions — <https://opentelemetry.io/docs/specs/semconv/gen-ai/> — the `gen_ai.*` attribute keys this solution uses are the working-group's current shape; keep the `app.*` keys surface-local so downstream OTel-native consumers see conformant data. Verify the current attribute names in the OTel repository before publishing a production dashboard, since the GenAI conventions are still stabilising.

Related exercises: [[exercise-02]] (the TTFT delta between cache-hit and cache-miss consumes this instrumentation); [[exercise-04]] (adds `app.cost.*` and `app.outcome` to the same span); [[exercise-05]] (the SLI boolean reads `app.latency.ttft_ms` and `app.retry.attempt_number` directly from these spans); mod-108 [[exercise-01]] (the gateway is the enforcement point for this instrumentation); mod-110 (tracing + feedback build on this span shape).
