### exercise-01: OTel Gen-AI + OpenInference Instrumentation for a Feature — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to instrument a production LLM feature so that **every call emits exactly one span** that is legible to two audiences at once: an OpenTelemetry-native backend (Grafana Tempo, Honeycomb, Datadog APM) using the **OpenTelemetry Generative-AI semantic conventions**, and an LLM-aware frontend (Phoenix, Langfuse-OTLP) using **OpenInference** conventions. The deliverable is the instrumentation module and a short attribute-contract doc — not a dashboard, not a sampler. Those come in [[exercise-02]] and [[exercise-05]].

The failure mode the exercise defends against is the common anti-pattern: one logger emits a free-text "latency: 1.2s, model: gpt-4o, cost: $0.004" line, a second logger inside the SDK emits a JSON blob with different keys, and a third sub-span is emitted per SSE chunk. Downstream, every analysis starts with a parser. The fix is one span per LLM call, with a **pinned attribute contract** that reads the same whether the consumer is a human skimming a trace, a SQL query against the span store, or a Phoenix UI.

This solution provides:

- A **two-layer attribute map** — OpenTelemetry Gen-AI semantic-convention keys on the bottom layer, OpenInference keys on the top layer, and feature-local `app.*` keys above both. Every key justified.
- A **reference instrumentation module** in Python using the OpenTelemetry API, emitting one span per call with streaming, tool, and retry handling.
- A **span-event schema** for prompt / completion / tool-call payloads, explicitly separated from span attributes so that redaction in [[exercise-02]] can target the payload without touching the metric stream.
- A **validation harness** — a replay script that drives 50 recorded calls and asserts the required attributes land.
- A **grading rubric** for the instructor.

The lesson the exercise defends: *an LLM feature without a one-span-per-call contract cannot be reasoned about operationally*. The SLI in mod-109 [[exercise-05]], the feedback join in [[exercise-03]], the per-failure-mode detection signal in [[exercise-04]], and the status page in [[exercise-05]] all read this span. If the span is wrong, everything downstream is wrong.

## 2. Worked answer or implementation

### 2a. Attribute contract (deliverable 1)

The attribute namespace is layered on purpose. The lower-case `gen_ai.*` keys are from the OpenTelemetry Gen-AI semantic-convention working group and are portable across backends. The `openinference.*` keys add surface-level grouping (session, user, tags) that make LLM-specific UIs work. The `app.*` keys are feature-local and encode decisions this team owns.

| Attribute | Layer | Required? | Definition |
|---|---|---|---|
| `gen_ai.system` | OTel | required | Provider identifier (`openai`, `anthropic`, `bedrock`, `google.vertex_ai`). |
| `gen_ai.operation.name` | OTel | required | One of `chat`, `text_completion`, `embeddings`, `tool_execution`. |
| `gen_ai.request.model` | OTel | required | Model asked for at call site (e.g. `claude-sonnet-4-6`). |
| `gen_ai.response.model` | OTel | required | Model actually served; may differ after fallback. |
| `gen_ai.request.temperature` | OTel | if set | Request temperature when non-default. |
| `gen_ai.request.max_tokens` | OTel | if set | Request output cap. |
| `gen_ai.response.id` | OTel | required | Provider response id for cross-reference. |
| `gen_ai.response.finish_reason` | OTel | required | Terminal reason: `stop`, `length`, `content_filter`, `tool_calls`. |
| `gen_ai.usage.input_tokens` | OTel | required | Prompt tokens on the served response. |
| `gen_ai.usage.output_tokens` | OTel | required | Completion tokens. |
| `openinference.span.kind` | OpenInference | required | One of `LLM`, `CHAIN`, `TOOL`, `RETRIEVER`, `EMBEDDING`, `AGENT`. |
| `openinference.session.id` | OpenInference | required | Conversation / thread id the call belongs to. |
| `openinference.user.id` | OpenInference | required-hashed | Opaque per-user identifier; hash if raw user id is PII. |
| `openinference.tag.tags` | OpenInference | optional | List of free-form tags: `["prod", "experiment-C"]`. |
| `app.feature` | local | required | Feature-registry key (`helpdesk.reply`, `search.compose`). Matches the registry in mod-109 [[exercise-04]]. |
| `app.tenant_id` | local | required | Opaque tenant id (hashed if PII-sensitive). |
| `app.request_id` | local | required | Caller-side correlation id. |
| `app.retry.attempt_number` | local | required | Ordinal of the attempt that produced the response (1 on first-try success). |
| `app.latency.ttft_ms` | local | required on streaming | Mod-109 [[exercise-01]] definition. |
| `app.latency.total_ms` | local | required | Mod-109 [[exercise-01]] definition. |
| `app.outcome` | local | required | `success`, `refusal`, `error`, `cancelled`. |
| `app.cost.usd` | local | required | Mod-109 [[exercise-04]] computed cost. |

Rules for the contract:

- **No free-text in attributes.** Multi-KB prompt text does not go on a span attribute; backends truncate attributes silently and the data becomes unqueryable. Put payloads on **span events** (2c).
- **`gen_ai.*` keys are reserved for the working-group's definitions.** Do not invent `gen_ai.feature` or `gen_ai.tenant_id` — they live under `app.*`.
- **Hash user and tenant ids at the boundary, not at export.** If a hash happens inside the span processor, the raw id has already crossed a trust boundary. Hash before `set_attribute`.

### 2b. Reference instrumentation

```python
"""instrumentation.py — one-span-per-LLM-call with OTel Gen-AI + OpenInference.

Run against opentelemetry-api / opentelemetry-sdk. No vendor-specific SDKs
are imported at the instrumentation layer: the consumer passes in the
provider SDK and this module wraps the call.
"""
from __future__ import annotations

import hashlib
import json
import time
from contextlib import contextmanager
from dataclasses import dataclass, field
from typing import Any, Iterator

from opentelemetry import trace
from opentelemetry.trace import SpanKind, Status, StatusCode

tracer = trace.get_tracer("app.llm", schema_url="https://opentelemetry.io/schemas/1.27.0")


def _hash_id(raw: str, *, salt: str) -> str:
    """Stable, non-reversible hash for user / tenant ids. Salt is per-env."""
    return hashlib.sha256(f"{salt}:{raw}".encode()).hexdigest()[:24]


@dataclass
class LLMCallContext:
    system: str                  # "openai", "anthropic", ...
    operation: str               # "chat", "embeddings", ...
    request_model: str
    feature: str
    tenant_id_raw: str
    session_id: str
    user_id_raw: str
    request_id: str
    tags: list[str] = field(default_factory=list)
    temperature: float | None = None
    max_tokens: int | None = None


@contextmanager
def llm_span(ctx: LLMCallContext, *, hash_salt: str) -> Iterator["LLMSpan"]:
    with tracer.start_as_current_span(
        name=f"{ctx.operation} {ctx.request_model}",
        kind=SpanKind.CLIENT,
    ) as span:
        # OTel Gen-AI layer
        span.set_attribute("gen_ai.system", ctx.system)
        span.set_attribute("gen_ai.operation.name", ctx.operation)
        span.set_attribute("gen_ai.request.model", ctx.request_model)
        if ctx.temperature is not None:
            span.set_attribute("gen_ai.request.temperature", ctx.temperature)
        if ctx.max_tokens is not None:
            span.set_attribute("gen_ai.request.max_tokens", ctx.max_tokens)

        # OpenInference layer
        span.set_attribute("openinference.span.kind", "LLM")
        span.set_attribute("openinference.session.id", ctx.session_id)
        span.set_attribute("openinference.user.id", _hash_id(ctx.user_id_raw, salt=hash_salt))
        if ctx.tags:
            span.set_attribute("openinference.tag.tags", ctx.tags)

        # Local layer
        span.set_attribute("app.feature", ctx.feature)
        span.set_attribute("app.tenant_id", _hash_id(ctx.tenant_id_raw, salt=hash_salt))
        span.set_attribute("app.request_id", ctx.request_id)

        helper = LLMSpan(span, t_sent=time.perf_counter())
        try:
            yield helper
        except Exception as exc:
            span.set_attribute("app.outcome", "error")
            span.set_attribute("app.error.class", exc.__class__.__name__)
            span.set_status(Status(StatusCode.ERROR, str(exc)[:200]))
            raise
        finally:
            helper.stamp_final()


class LLMSpan:
    """Thin helper the caller stamps as data arrives from the provider."""

    def __init__(self, span, *, t_sent: float):
        self._span = span
        self.t_sent = t_sent
        self.t_first_content: float | None = None
        self.t_closed: float | None = None
        self.retry_ms = 0
        self.attempt_number = 1
        self.outcome = "success"
        self.cost_usd: float | None = None
        self.response_id: str | None = None
        self.response_model: str | None = None
        self.input_tokens = 0
        self.output_tokens = 0
        self.finish_reason: str | None = None

    def mark_first_content(self) -> None:
        if self.t_first_content is None:
            self.t_first_content = time.perf_counter()

    def mark_closed(self) -> None:
        self.t_closed = time.perf_counter()

    def record_prompt_event(self, messages: list[dict[str, Any]]) -> None:
        """Payloads go on span events, not attributes. See 2c for shape."""
        self._span.add_event(
            name="gen_ai.content.prompt",
            attributes={"gen_ai.prompt.json": json.dumps(messages)[:32_768]},
        )

    def record_completion_event(self, completion: str, *, tool_calls: list[dict] | None = None) -> None:
        attrs: dict[str, Any] = {"gen_ai.completion.text": completion[:32_768]}
        if tool_calls:
            attrs["gen_ai.completion.tool_calls.json"] = json.dumps(tool_calls)[:16_384]
        self._span.add_event(name="gen_ai.content.completion", attributes=attrs)

    def stamp_final(self) -> None:
        now = time.perf_counter()
        total_ms = int((self.t_closed or now) - self.t_sent) * 1000 \
                 if (self.t_closed or now) >= self.t_sent else -1
        total_ms = int(((self.t_closed or now) - self.t_sent) * 1000)
        ttft_ms = int((self.t_first_content - self.t_sent) * 1000) if self.t_first_content else -1
        self._span.set_attribute("app.latency.total_ms", total_ms)
        self._span.set_attribute("app.latency.ttft_ms", ttft_ms)
        self._span.set_attribute("app.retry.attempt_number", self.attempt_number)
        self._span.set_attribute("app.outcome", self.outcome)
        if self.response_id:
            self._span.set_attribute("gen_ai.response.id", self.response_id)
        if self.response_model:
            self._span.set_attribute("gen_ai.response.model", self.response_model)
        if self.finish_reason:
            self._span.set_attribute("gen_ai.response.finish_reason", self.finish_reason)
        self._span.set_attribute("gen_ai.usage.input_tokens", self.input_tokens)
        self._span.set_attribute("gen_ai.usage.output_tokens", self.output_tokens)
        if self.cost_usd is not None:
            self._span.set_attribute("app.cost.usd", self.cost_usd)
```

Usage at the call site:

```python
# call_site.py — the one place the surface calls an LLM.
from instrumentation import LLMCallContext, llm_span

def answer(session_id: str, user_id: str, tenant_id: str, prompt: str) -> str:
    ctx = LLMCallContext(
        system="anthropic",
        operation="chat",
        request_model="claude-sonnet-4-6",
        feature="helpdesk.reply",
        tenant_id_raw=tenant_id,
        session_id=session_id,
        user_id_raw=user_id,
        request_id=new_request_id(),
        tags=["prod", "stream-on"],
        temperature=0.2,
        max_tokens=800,
    )
    with llm_span(ctx, hash_salt=HASH_SALT) as s:
        s.record_prompt_event(build_messages(prompt))
        completion = ""
        for event in call_provider_stream(ctx, prompt):
            if event.delta:
                s.mark_first_content()
                completion += event.delta
            if event.final:
                s.mark_closed()
                s.response_id = event.response_id
                s.response_model = event.model
                s.input_tokens = event.usage.input_tokens
                s.output_tokens = event.usage.output_tokens
                s.finish_reason = event.finish_reason
                s.cost_usd = price_fn(event.model, event.usage)
        s.record_completion_event(completion)
        return completion
```

### 2c. Span-event schema for payloads (deliverable 2)

Prompt and completion payloads are span **events**, not attributes. Two reasons:

1. OTel backends treat attributes as **indexable low-cardinality metadata** and events as **timestamped high-cardinality data**. A 10 KB prompt on an attribute will be truncated, re-indexed on every query, and often dropped by export limits.
2. Redaction in [[exercise-02]] has a single hook (`on_event` on the span processor) that can touch the payload without rewriting attribute paths every time the prompt shape changes.

```json
// Event: "gen_ai.content.prompt"
{
  "name": "gen_ai.content.prompt",
  "attributes": {
    "gen_ai.prompt.json": "[{\"role\":\"system\",\"content\":\"...\"},{\"role\":\"user\",\"content\":\"...\"}]"
  }
}

// Event: "gen_ai.content.completion"
{
  "name": "gen_ai.content.completion",
  "attributes": {
    "gen_ai.completion.text": "…final assistant text…",
    "gen_ai.completion.tool_calls.json": "[{\"name\":\"crm.lookup\",\"arguments\":{…}}]"
  }
}
```

Field size budget: cap each payload at 32 KiB at the instrumentation layer. The scrubber in [[exercise-02]] trusts this cap; without it, a single pathological call can blow the per-span export limit and silently drop the trace.

### 2d. Validation harness (deliverable 3)

```python
# tests/test_instrumentation.py
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import InMemorySpanExporter, SimpleSpanProcessor
from opentelemetry import trace

REQUIRED_ATTRS = {
    "gen_ai.system",
    "gen_ai.operation.name",
    "gen_ai.request.model",
    "gen_ai.response.model",
    "gen_ai.response.finish_reason",
    "gen_ai.usage.input_tokens",
    "gen_ai.usage.output_tokens",
    "openinference.span.kind",
    "openinference.session.id",
    "openinference.user.id",
    "app.feature",
    "app.tenant_id",
    "app.request_id",
    "app.retry.attempt_number",
    "app.latency.total_ms",
    "app.outcome",
}

def test_required_attributes_present():
    exporter = InMemorySpanExporter()
    provider = TracerProvider()
    provider.add_span_processor(SimpleSpanProcessor(exporter))
    trace.set_tracer_provider(provider)

    replay_recorded_fixture("tests/fixtures/50_calls.jsonl")   # drives the surface

    spans = exporter.get_finished_spans()
    assert len(spans) == 50, f"expected 50 spans, got {len(spans)}"
    for s in spans:
        missing = REQUIRED_ATTRS - set(s.attributes.keys())
        assert not missing, f"span {s.name} missing: {missing}"
        # events present on non-error spans
        if s.attributes.get("app.outcome") == "success":
            names = {e.name for e in s.events}
            assert "gen_ai.content.prompt" in names
            assert "gen_ai.content.completion" in names
        # hashed ids never look like email / uuid
        uid = s.attributes["openinference.user.id"]
        assert "@" not in uid and len(uid) == 24
```

### 2e. Implementation notes and gotchas

- **One span per call, not per chunk.** A sub-span per SSE delta turns a 1-span trace into a 400-span trace and nothing queries cleanly. TTFT is an attribute on the parent span, not a `start_as_current_span` on each delta.
- **Tool invocations are child spans.** If the surface fires `tool_x`, the tool execution gets its own span with `openinference.span.kind="TOOL"`, parented to the LLM span. The LLM span still records the final completion and the surface-level `app.latency.tool_round_trip_ms` as in mod-109 [[exercise-01]].
- **Hash user ids at the boundary.** Hashing in the processor means the raw id has already lived in-process beyond the hashing point. Hash before `set_attribute`.
- **Do not instrument inside the provider SDK.** Wrapping the SDK is tempting; the problem is every SDK upgrade re-opens the instrumentation surface. Wrap the call site instead.
- **Streaming and non-streaming share the span shape.** `ttft_ms = -1` when no stream; `t_first_content = t_closed` is wrong because it hides whether streaming was configured.
- **Finish reason is not optional.** A span that does not report finish reason cannot be excluded from the SLI's `length` / `content_filter` filter in mod-109 [[exercise-05]].
- **`openinference.span.kind` enables LLM-aware UIs.** Phoenix and Langfuse-OTLP use it to render the span as an LLM trace instead of a generic HTTP trace. Omitting it degrades UX without breaking the span.

### 2f. Mapping to the exercise deliverables

| Exercise asks for | Section of this solution |
|---|---|
| Attribute contract across OTel Gen-AI + OpenInference | 2a |
| Reference instrumentation module | 2b |
| Payload separation (events vs attributes) | 2c |
| Validation harness | 2d |
| Operational guidance for the author | 2e |

## 3. Validation steps

1. **Every required attribute lands on every span.** The harness in 2d asserts the set; a span missing any key is a hard failure.
2. **Prompts and completions are on events, not attributes.** Grep the exporter output: `gen_ai.prompt` must not appear as an attribute key on an LLM span.
3. **User and tenant ids never appear raw.** Assert the hashed id has 24 hex characters and does not contain `@` or raw tenant prefixes.
4. **Streaming TTFT is positive.** On a streaming call with ≥ 1 content delta, `app.latency.ttft_ms > 0` and `< total_ms`.
5. **Tool calls produce child spans.** When `gen_ai.response.finish_reason = "tool_calls"`, at least one child span with `openinference.span.kind = "TOOL"` exists under the LLM span.
6. **Error outcomes carry `app.error.class`.** A provider 500 produces `app.outcome = "error"` and a non-empty `app.error.class`; the span status is `ERROR`.
7. **Finish reason is always set.** No span may land without `gen_ai.response.finish_reason` on a non-error outcome.
8. **Schema URL pinned.** The tracer carries a `schema_url`; a backend upgrade that changes the convention version is detectable.
9. **Attribute values are within size budget.** Max attribute value length is bounded (OTel SDK default is 4096 unless overridden). The harness asserts no truncation occurred on the hash or latency fields.
10. **Per-feature coverage.** Every call site in the feature module is wrapped — a lint rule (`grep -n "provider_sdk.chat.completions.create" | grep -v with llm_span` returns empty).

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| One span per LLM call | 1 | No sub-spans per SSE delta; tool calls as proper child spans. |
| OTel Gen-AI attributes correct | 2 | `gen_ai.system`, `operation.name`, `request.model`, `response.model`, `response.id`, `response.finish_reason`, `usage.input_tokens`, `usage.output_tokens` all present and named per convention. |
| OpenInference attributes correct | 1 | `openinference.span.kind`, `session.id`, `user.id` present. |
| Local `app.*` attributes present | 1 | `feature`, `tenant_id`, `request_id`, `retry.attempt_number`, `latency.total_ms`, `outcome`, `cost.usd`. |
| Payloads on events, not attributes | 1 | `gen_ai.content.prompt` / `gen_ai.content.completion` events present; no payload attributes on span. |
| User + tenant ids hashed at boundary | 1 | `_hash_id` or equivalent called before `set_attribute`; salt per-env. |
| Streaming and non-streaming both handled | 1 | `ttft_ms` positive on streaming; sensible on non-streaming. |
| Error path sets `app.outcome=error` and span status ERROR | 1 | Exception path tested. |
| Validation harness asserts the contract | 1 | Automated test fails loudly on missing keys. |

A submission scoring < 6 is rejected. Hard failures: (a) prompts on attributes, (b) raw user/tenant id in attributes, (c) sub-span per SSE delta.

## 5. Common mistakes

- **Instrumenting inside the provider SDK.** Every SDK upgrade reopens the instrumentation surface. Wrap at the call site.
- **Emitting a span per SSE chunk.** Turns the trace view into a flood and prevents straightforward queries. TTFT is an attribute on the parent, not a span.
- **Free-text prompt on an attribute.** The attribute gets silently truncated and the data is unusable at query time.
- **Raw user id on `openinference.user.id`.** Any backend operator can read trace data. Hash before set.
- **Missing finish reason.** The SLI in mod-109 [[exercise-05]] cannot exclude `length` and `content_filter` outcomes, so they land as SLO violations. Debug time measured in hours.
- **Inventing `gen_ai.*` keys.** The `gen_ai.*` namespace belongs to the convention. Feature-local keys live under `app.*`.
- **Omitting `openinference.span.kind`.** Phoenix / Langfuse-OTLP UIs degrade to generic-HTTP rendering. Fixable but ugly.
- **Hashing in the span processor.** The raw id has already crossed trust boundaries by the time the processor runs.
- **No schema URL on the tracer.** A convention-version upgrade silently relabels fields; pin the schema URL so the change is reviewable.
- **Logging in parallel.** A free-text "latency: 1.2s" log line that duplicates the span attribute guarantees the two sources drift. Delete the log; keep the span.

## 6. References

- OpenTelemetry Generative-AI semantic conventions — <https://opentelemetry.io/docs/specs/semconv/gen-ai/> — the `gen_ai.*` attribute keys, operation names, and the prompt / completion event shape used in 2a and 2c. The conventions are still stabilising; keep the tracer's `schema_url` pinned and re-verify attribute names before each production rollout. <!-- verify-before-publishing: pin to the specific schema URL and spec version in use (e.g. v1.27.0) and update the schema_url constant above to match. -->
- OpenInference Semantic Conventions — <https://github.com/Arize-ai/openinference/blob/main/spec/semantic_conventions.md> — the `openinference.*` keys (span.kind, session.id, user.id, tag.tags) used in 2a. The spec is the authoritative source for the span-kind enum. <!-- verify-before-publishing: confirm the current spec URL and list of span kinds against the openinference repository main branch. -->
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — this instrumentation is the MEASURE-function substrate for an LLM feature: without a one-span-per-call stream carrying model, outcome, retry, cost, and user-session fields, no downstream risk-management loop can close (MANAGE depends on measured signals). The attribute contract is the artefact that makes the measurement reproducible across teams.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML02 (Data Poisoning) and ML09 (Output Integrity) rely on being able to retrieve the exact prompt / completion pair for a given outcome weeks after the fact. The prompt-and-completion span events are the primary evidence trail. ML05 (Model Theft) attempts show up as anomalous `gen_ai.usage.input_tokens` on `openinference.user.id` aggregates; those queries are only possible because the ids are hashed-but-stable.
- MITRE ATLAS — <https://atlas.mitre.org/> — Collection and Discovery tactics against LLM-backed features produce detectable patterns on this span shape (unusual prompt-event content, outlier `finish_reason="content_filter"` rates on a single user hash, `app.outcome="refusal"` clusters). The span contract is the detection substrate; ATLAS tactics are the pattern library the detection engineers query against.

Related exercises: [[exercise-02]] (the sampler and scrubber read span events and act on `app.feature` / `openinference.user.id`); [[exercise-03]] (feedback rows join on `openinference.session.id` and `app.request_id`); [[exercise-04]] (playbook detection signals read `app.outcome`, `gen_ai.response.finish_reason`, `app.retry.attempt_number`, `app.cost.usd`); [[exercise-05]] (status page per `app.feature` is driven by rollups of this span); mod-109 [[exercise-01]] (the latency vocabulary this span inherits); mod-109 [[exercise-04]] (cost on `app.cost.usd`); mod-109 [[exercise-05]] (SLI reads this span directly).
