### exercise-02: Sampling and PII Scrubbing Policy — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to write a **sampling and PII-scrubbing policy** for the span stream produced by [[exercise-01]], and to implement it as an OpenTelemetry `SpanProcessor` that runs **before export**. The deliverable is a policy doc, a processor implementation, and a validation fixture — not an alerting change, not a backend swap. Those are in [[exercise-05]] and are outside this exercise's scope.

The failure modes the exercise is written to catch are three, and they are all common:

1. **Keep everything.** Full-fidelity tracing of a 50 QPS LLM feature with 2 KB prompts emits ~8 GB/day per surface. The cost model breaks and the team's first optimisation is "reduce retention," which eliminates the data the on-call needs during the incident they are, right now, investigating.
2. **Sample uniformly at 1%.** Errors and slow calls — the ones the on-call needs — show up in the 99% the sampler dropped. The surface looks healthy because the health signal is in the discarded 99%.
3. **Scrub after export.** Prompts carrying emails, phone numbers, and account ids land in the vendor backend. The scrubber runs on the backend side, five minutes later. The breach has already happened; the scrubber only hides it from the analyst.

The fix is a policy that treats sampling and scrubbing as two separate questions, answered in order:

- **Sampling**: head-based for volume control, tail-based for error / slow-call retention, with explicit "always keep" rules.
- **Scrubbing**: runs on the span **before** it leaves the process; keyed off a per-span-kind allowlist, not a per-field denylist.

This solution provides:

- A **policy document** with named sampling tiers, retention budget per tier, and an explicit "always keep" list.
- A **PII scrubber reference implementation** as an OTel span processor, with ordered rules (structured fields → regex → NER fallback) and a `scrub.*` attribute set that records what was scrubbed without recording what the original value was.
- A **validation fixture** — labelled test prompts the scrubber must redact, plus negatives it must not touch.
- A **governance section** on the three questions the policy must answer before every provider swap, backend swap, or new feature launch.
- A **grading rubric** for the instructor.

The lesson the exercise defends: *sampling and scrubbing are policy, not configuration.* A one-line `sampler = TraceIdRatio(0.01)` is a decision with major compliance, debuggability, and cost consequences. The policy doc is where that decision is defensible under audit; the processor is where it is enforced; the fixture is where it is tested.

## 2. Worked answer or implementation

### 2a. Sampling policy (deliverable 1)

The policy separates **volume control** (head-based) from **signal retention** (tail-based). Both run; the second overrides the first.

```markdown
# docs/trace-sampling-policy.md

## Scope
All spans produced by the instrumentation module in [[exercise-01]] for the
following features: helpdesk.reply, search.compose, pricing.summarise.

## Tiered sampling

| Tier | Rule | Target rate | Enforced by |
|------|------|-------------|-------------|
| T0  | `app.outcome in ("error", "refusal")`  | 100% | Tail-sampler always-keep |
| T0  | `app.latency.total_ms > 5000`          | 100% | Tail-sampler always-keep |
| T0  | `app.retry.attempt_number > 1`         | 100% | Tail-sampler always-keep |
| T0  | `gen_ai.response.finish_reason == "content_filter"` | 100% | Tail-sampler always-keep |
| T0  | request header `x-trace-forced: 1`     | 100% | Head-sampler override |
| T1  | `app.feature == "helpdesk.reply"` and `app.outcome == "success"` | 10% | Head-sampler ratio |
| T2  | everything else                        | 2%  | Head-sampler ratio |

## Why these tiers
- T0 keeps every signal the incident playbook in [[exercise-04]] needs.
- T1 at 10% is enough for dashboarding percentiles with tight-enough CIs on
  the primary surface at 50 QPS (~4.3M spans/month at 10%, 430k at 2%).
- T2 at 2% covers lower-QPS surfaces; raise if the feature volume climbs
  above 20 QPS sustained.

## Retention budget (span count, 28-day window)
  Target:                 ≤ 20M spans/month across all T0 + T1 + T2.
  Alarm:                  > 25M spans/month triggers policy review.
  Measurement:            backend "ingested spans per feature" panel.

## Always-keep overrides (compliance / debugging)
- Requests from allow-listed debug tenants: 100%.
- Requests with `x-trace-forced: 1` from an authenticated internal caller: 100%.

## Review cadence
Quarterly, by role: owner = applied-AI platform. Trigger an earlier review if:
- Monthly span count > 25M for two consecutive weeks.
- A new feature joins the policy's Scope list.
- A P1/P2 incident could not be investigated because a span was sampled out.
```

The tiers are stated as a decision, not a derivation. The 10%/2% numbers are defensible because they are computed against a stated retention budget and the tail-sampler guarantees signal survival — not because 10% "feels right."

### 2b. Scrubbing policy (deliverable 2)

The scrubbing policy is an **allowlist on attribute keys** and a **denylist on event payload patterns**. Two lists, two directions:

```markdown
## Attribute allowlist (anything not on this list is dropped)
The contract from [[exercise-01]] is the floor. Specifically:
  gen_ai.system
  gen_ai.operation.name
  gen_ai.request.model
  gen_ai.request.temperature
  gen_ai.request.max_tokens
  gen_ai.response.model
  gen_ai.response.id
  gen_ai.response.finish_reason
  gen_ai.usage.input_tokens
  gen_ai.usage.output_tokens
  openinference.span.kind
  openinference.session.id
  openinference.user.id
  openinference.tag.tags
  app.feature
  app.tenant_id
  app.request_id
  app.retry.attempt_number
  app.latency.*
  app.outcome
  app.error.class
  app.cost.usd
  scrub.*               (set by this processor; see 2d)

Every other attribute key is dropped. New keys require a policy update.

## Event payload denylist (regex + NER)
Runs on gen_ai.content.prompt and gen_ai.content.completion.

Ordered rules (first match wins):
  1. Email:         [A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}             (i-flag)
  2. Phone (NA):    \b(?:\+?1[\s.-]?)?\(?\d{3}\)?[\s.-]?\d{3}[\s.-]?\d{4}\b
  3. Phone (E.164): \+\d{8,15}\b
  4. Credit card:   \b(?:\d[ -]*?){13,19}\b   (post-filter: Luhn-valid only)
  5. SSN (US):      \b\d{3}-\d{2}-\d{4}\b
  6. IBAN:          \b[A-Z]{2}\d{2}[A-Z0-9]{11,30}\b
  7. IP (v4+v6):    (standard IANA pattern; drop unless in debug tier)
  8. UUID:          8-4-4-4-12 hex  (keep; it is the correlation id shape)
  9. NER fallback:  spaCy small model, labels: PERSON, GPE, ORG, DATE, MONEY
                    Only PERSON is masked by default. GPE/ORG/DATE/MONEY
                    are tagged in scrub.labels but not redacted.

Replacement token: "[REDACTED:<rule>]" where <rule> is the rule name above.
```

The `scrub.*` attributes record what was scrubbed, not what was redacted:

```
scrub.rules_matched   = ["email", "phone-na"]   # names only, not values
scrub.rules_count     = 2
scrub.payload_bytes_before = 4192
scrub.payload_bytes_after  = 4164
scrub.ner_fallback_triggered = false
```

Those fields are what [[exercise-04]] playbooks read when a PII-leak alert fires: the on-call can tell what *shape* of data was present without re-reading the data itself.

### 2c. Reference implementation — tail-head sampler

Head-based ratios run on span start (no tail context yet). Tail-based rules need the completed span. The pattern is two cooperating processors: a head sampler decides the initial sampling bit, then a tail processor flips the bit back to "keep" if any T0 rule matches at end of span.

```python
# sampling.py
from opentelemetry.sdk.trace.sampling import (
    Sampler, SamplingResult, Decision, TraceIdRatioBased
)
from opentelemetry.trace import SpanKind
from opentelemetry.sdk.trace import ReadableSpan, SpanProcessor
from opentelemetry.sdk.trace.export import BatchSpanProcessor

FEATURE_RATIOS = {
    "helpdesk.reply":   0.10,
    "search.compose":   0.10,
    "pricing.summarise":0.10,
}
DEFAULT_RATIO = 0.02


class TieredHeadSampler(Sampler):
    """Head sampler: ratio keyed by feature; forced trace header always keeps."""

    def __init__(self):
        self._ratio_samplers = {
            k: TraceIdRatioBased(v) for k, v in FEATURE_RATIOS.items()
        }
        self._default = TraceIdRatioBased(DEFAULT_RATIO)

    def should_sample(self, parent_context, trace_id, name, kind=None,
                      attributes=None, links=None, trace_state=None):
        attrs = attributes or {}
        if attrs.get("x-trace-forced") == "1":
            return SamplingResult(Decision.RECORD_AND_SAMPLE)
        feature = attrs.get("app.feature")
        sampler = self._ratio_samplers.get(feature, self._default)
        return sampler.should_sample(
            parent_context, trace_id, name, kind=kind,
            attributes=attributes, links=links, trace_state=trace_state,
        )

    def get_description(self) -> str:
        return "TieredHeadSampler(feature-keyed)"


class TailAlwaysKeepProcessor(SpanProcessor):
    """Tail processor: forces export for T0 spans regardless of head decision."""

    def __init__(self, downstream_exporter):
        # BatchSpanProcessor already wraps this; we delegate to its batch pipe.
        self._batch = BatchSpanProcessor(downstream_exporter)

    def on_start(self, span, parent_context=None):
        pass

    def on_end(self, span: ReadableSpan) -> None:
        if self._must_keep(span) or span.context.trace_flags.sampled:
            self._batch.on_end(span)

    def shutdown(self):
        self._batch.shutdown()

    def force_flush(self, timeout_millis: int = 30_000) -> bool:
        return self._batch.force_flush(timeout_millis)

    @staticmethod
    def _must_keep(span: ReadableSpan) -> bool:
        a = span.attributes
        if a.get("app.outcome") in ("error", "refusal"):
            return True
        if int(a.get("app.latency.total_ms", 0)) > 5000:
            return True
        if int(a.get("app.retry.attempt_number", 1)) > 1:
            return True
        if a.get("gen_ai.response.finish_reason") == "content_filter":
            return True
        return False
```

Wire-up:

```python
# bootstrap.py
provider = TracerProvider(sampler=TieredHeadSampler())
provider.add_span_processor(PIIScrubProcessor())           # see 2d
provider.add_span_processor(TailAlwaysKeepProcessor(OTLPSpanExporter(...)))
trace.set_tracer_provider(provider)
```

The processor order matters: **scrub first, then tail-keep**. A scrubbed-away span that gets kept by the tail rule carries scrubbed payloads; a kept span that hits the scrubber later still exposes raw payloads if the export races the scrub.

### 2d. Reference implementation — PII scrubber

```python
# scrub.py
import re
from typing import Iterable, Pattern
from opentelemetry.sdk.trace import ReadableSpan, Span, SpanProcessor

ALLOWED_ATTRIBUTE_PREFIXES = (
    "gen_ai.", "openinference.", "app.", "scrub.",
)

RULES: list[tuple[str, Pattern[str]]] = [
    ("email",      re.compile(r"[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}", re.I)),
    ("phone-na",   re.compile(r"\b(?:\+?1[\s.-]?)?\(?\d{3}\)?[\s.-]?\d{3}[\s.-]?\d{4}\b")),
    ("phone-e164", re.compile(r"\+\d{8,15}\b")),
    ("ssn-us",     re.compile(r"\b\d{3}-\d{2}-\d{4}\b")),
    ("iban",       re.compile(r"\b[A-Z]{2}\d{2}[A-Z0-9]{11,30}\b")),
    ("ipv4",       re.compile(r"\b(?:\d{1,3}\.){3}\d{1,3}\b")),
    # Credit card last — needs Luhn post-check (not shown).
    ("cc",         re.compile(r"\b(?:\d[ -]*?){13,19}\b")),
]


def scrub_text(s: str) -> tuple[str, list[str], int]:
    matched: list[str] = []
    out = s
    for name, pat in RULES:
        if pat.search(out):
            matched.append(name)
            out = pat.sub(f"[REDACTED:{name}]", out)
    return out, matched, len(s)


class PIIScrubProcessor(SpanProcessor):
    """Allowlist attributes; denylist-regex events. Runs in-process."""

    def on_start(self, span: Span, parent_context=None) -> None:
        pass

    def on_end(self, span: ReadableSpan) -> None:
        # Attribute pass: drop anything not on the allowlist.
        for key in list(span.attributes.keys()):
            if not key.startswith(ALLOWED_ATTRIBUTE_PREFIXES):
                # ReadableSpan attributes are not mutable; see note below.
                pass

        # Event pass: scrub payload strings on gen_ai.content.* events.
        rules_matched: set[str] = set()
        bytes_before = bytes_after = 0
        for ev in span.events:
            if ev.name not in ("gen_ai.content.prompt", "gen_ai.content.completion"):
                continue
            for k, v in list(ev.attributes.items()):
                if not isinstance(v, str):
                    continue
                cleaned, m, orig_len = scrub_text(v)
                if m:
                    rules_matched.update(m)
                    bytes_before += orig_len
                    bytes_after += len(cleaned)
                    ev.attributes[k] = cleaned

        if rules_matched:
            # scrub.* attributes are set via a span-mutation helper the
            # instrumentation SDK exposes for in-flight ReadableSpans.
            span._attributes["scrub.rules_matched"] = sorted(rules_matched)  # noqa
            span._attributes["scrub.rules_count"] = len(rules_matched)
            span._attributes["scrub.payload_bytes_before"] = bytes_before
            span._attributes["scrub.payload_bytes_after"] = bytes_after

    def shutdown(self):
        pass

    def force_flush(self, timeout_millis: int = 30_000) -> bool:
        return True
```

> **Note on `ReadableSpan` mutation.** Current OTel SDKs expose span attributes as immutable on `ReadableSpan`. In production, run the scrubber inside a **custom exporter wrapper** that constructs a new span-data payload with the scrubbed events, rather than mutating in place. The structure above is written for teaching; the policy test fixture in 2e asserts the output shape and tolerates either implementation. <!-- verify-before-publishing: confirm current OTel Python SDK's supported mutation path (BatchSpanProcessor hook, custom exporter wrap, or SpanProcessor on_end mutability guarantee) and update code accordingly. -->

### 2e. Validation fixture (deliverable 3)

```jsonl
// tests/fixtures/scrub_cases.jsonl
{"id": "email-1",     "in": "please contact alex@example.com",                             "expect": {"rules": ["email"],      "out_contains": "[REDACTED:email]"}}
{"id": "phone-1",     "in": "call me at (415) 555-0199",                                   "expect": {"rules": ["phone-na"],   "out_contains": "[REDACTED:phone-na]"}}
{"id": "phone-e164",  "in": "+14155550199 is my number",                                   "expect": {"rules": ["phone-e164"], "out_contains": "[REDACTED:phone-e164]"}}
{"id": "ssn-1",       "in": "my SSN is 123-45-6789",                                       "expect": {"rules": ["ssn-us"],     "out_contains": "[REDACTED:ssn-us]"}}
{"id": "iban-1",      "in": "transfer to GB29NWBK60161331926819",                          "expect": {"rules": ["iban"],       "out_contains": "[REDACTED:iban]"}}
{"id": "cc-1",        "in": "card 4242 4242 4242 4242",                                    "expect": {"rules": ["cc"],         "out_contains": "[REDACTED:cc]"}}
{"id": "uuid-keep",   "in": "trace e7b0c1f4-1234-4abc-9def-aaaaaaaaaaaa continues",        "expect": {"rules": [],             "out_contains": "e7b0c1f4-1234"}}
{"id": "neg-1",       "in": "account overview for order ORD-1024",                        "expect": {"rules": [],             "out_contains": "ORD-1024"}}
{"id": "neg-2",       "in": "temperature = 0.2, max_tokens = 800",                        "expect": {"rules": [],             "out_contains": "max_tokens"}}
```

The fixture is the contract. New rules add rows; deprecated rules are not removed, only marked `expect.rules` empty on the row that previously exercised them, so the test suite shows the policy change.

### 2f. Governance (deliverable 4)

```markdown
## Pre-launch checklist (before any new feature joins Scope)

1. Which backend sees these spans? Is it in-region for all tenants? Is a BAA
   (or equivalent) in place if medical / financial data can appear in the
   payload events?
2. What is the per-tier retention? Confirm the 28-day target survives the
   feature's QPS at the ratio assigned in 2a.
3. Does any new call path produce attribute keys outside the allowlist? If
   yes, the allowlist is updated in the policy doc *before* code lands.

## Pre-provider-swap checklist
1. Does the new provider return fields that would introduce new attribute
   keys? Add them to the allowlist or drop them explicitly.
2. Does the new provider echo prompts into a different response field? The
   scrubber already covers gen_ai.content.*, but a provider-specific echo
   field needs the same denylist.

## Incident-driven policy change
A P1/P2 that could not be investigated because a span was sampled out
triggers two actions: raise the T1/T2 ratio for that feature for 30 days
(return to policy after retro) and add a new T0 rule if the signal pattern
is reusable.
```

### 2g. Mapping to the exercise deliverables

| Exercise asks for | Section of this solution |
|---|---|
| Sampling policy with tiers + retention budget | 2a |
| Scrubbing policy (attribute allowlist + payload denylist) | 2b |
| Sampler implementation (head + tail) | 2c |
| Scrubber implementation | 2d |
| Validation fixture | 2e |
| Governance / review cadence | 2f |

## 3. Validation steps

1. **Fixture pass rate is 100%.** `pytest tests/test_scrub.py` runs the fixture in 2e; every row's expected rules and output substring match.
2. **Negative cases are preserved.** UUIDs, order ids (`ORD-1024`), and literal `max_tokens` strings are not redacted.
3. **Attribute allowlist is enforced.** Inject a span with `app.secret="leak"` through the processor; assert the key is absent at export.
4. **Error spans always keep.** Set the head sampler to a ratio of 0.0 and emit 100 spans where 5 have `app.outcome="error"`. Assert exactly 5 reach the exporter.
5. **Slow spans always keep.** Same setup with 5 spans where `app.latency.total_ms=10000`. Assert 5 reach the exporter.
6. **Forced header overrides ratio.** With ratio 0.0, inject `x-trace-forced=1` on one span's start context; assert it reaches the exporter.
7. **Scrub metadata is correct.** For each redacted span, `scrub.rules_matched` contains the right rule names and nothing more.
8. **Backend receives no raw PII.** A canary test end-to-end: send a span with `test+e2e-canary@example.invalid` and query the backend for the literal string; expect zero results.
9. **Retention target holds.** Compute monthly span volume from the current QPS × tier ratio; assert ≤ 20M.
10. **Policy doc owner and review date present.** Both fields filled; next review ≤ one quarter out.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Head-based + tail-based sampling | 2 | Both layers exist; tail runs after head and can override. |
| T0 always-keep list correct | 1 | `error`, `refusal`, `latency > 5000ms`, `retry > 1`, `content_filter`, forced header. |
| Retention budget stated as a quantity | 1 | Span count per window, with a measurable alarm threshold. |
| Allowlist on attribute keys | 1 | New key = policy change; drop everything else. |
| Denylist on payload events, ordered rules | 1 | At minimum email + phone + SSN + credit card + IBAN. |
| Replacement preserves rule name | 1 | `[REDACTED:<rule>]` or equivalent; rule names set on `scrub.*`. |
| Scrubber runs in-process, before export | 1 | Processor registered on the TracerProvider; order documented. |
| Fixture-based validation | 1 | Positive + negative cases, red/green diff on PR. |
| Governance section present | 1 | Pre-launch and pre-provider-swap checks; review cadence. |

A submission scoring < 6 is rejected. Hard failures: (a) scrubbing after export, (b) uniform-rate sampling with no T0, (c) no attribute allowlist (denylist-only).

## 5. Common mistakes

- **Scrubbing after export.** The backend sees raw PII before the scrubber fires. Compliance posture is already broken by the time the scrubber runs.
- **Denylist-only on attribute keys.** A new call site adds `app.debug_info="session cookie"` and ships without a code review on the scrubber. Allowlist flips the default to safe.
- **Uniform 1% sampling.** Hides error and slow calls in the discarded 99%. Tail-keep is the fix.
- **Scrubbing attribute values.** The hash in `openinference.user.id` would match a 24-hex regex and get redacted. Attribute scrubbing is the wrong tool; the allowlist is.
- **Regex without Luhn on credit cards.** Matches long order numbers; false-positive rate is high. Luhn is cheap; run it.
- **Replacing with a fixed string.** `[REDACTED]` loses the rule name. The on-call in [[exercise-04]] cannot tell whether an email or an SSN was present.
- **No governance section.** Two quarters later, three features have joined Scope silently, retention has doubled, and nobody signed off. Review cadence prevents that.
- **One sampler, one processor.** Head and tail do different jobs and must be two components with a documented ordering.
- **Mutating `ReadableSpan` without checking SDK guarantees.** The reference code is written for teaching; production implementations use a custom exporter wrapper. See the note after 2d.
- **No validation fixture.** A scrubber without a fixture is one PR away from regression. The fixture is the test, the doc, and the audit evidence.

## 6. References

- OpenTelemetry Sampling specification — <https://opentelemetry.io/docs/specs/otel/trace/sdk/#sampling> — the head-based sampler interface (`Sampler.should_sample`) and the tail-sampler pattern used in 2c. The `TraceIdRatioBased` sampler is the reference ratio sampler; the tail-processor pattern is implemented in the OTel Collector's `tail_sampling` processor. <!-- verify-before-publishing: confirm current sampler API and ratio-sampler class names against the OTel Python SDK version pinned by the project. -->
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the GOVERN function requires documented data-handling policies, periodic review, and incident-driven updates. The policy doc in 2a / 2f is the GOVERN artefact; the scrubber is the MANAGE control that enforces it.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML02 (Data Poisoning) and ML03 (Model Inversion) both require evidence trails to investigate, which the sampling tiers preserve; ML08 (Model Skewing via Adversarial Input) depends on being able to re-read suspect prompt payloads, which is why T0 forces retention on error / refusal / content-filter outcomes. The scrubber is the compensating control that keeps those payloads safe to store.
- MITRE ATLAS — <https://atlas.mitre.org/> — ATLAS tactics such as Collection and Exfiltration target the trace stream directly when it carries raw prompts. The attribute allowlist eliminates the Exfiltration-via-attribute path; the payload denylist shrinks the Exfiltration-via-event surface. The `scrub.rules_matched` attribute is the detection substrate the [[exercise-04]] PII-leak playbook reads to trigger.

Related exercises: [[exercise-01]] (defines the span shape this policy operates on); [[exercise-03]] (feedback rows must reference `openinference.session.id`, which is on the allowlist); [[exercise-04]] (PII-leak and cost-spike playbooks read `scrub.*` and sampled error rates); [[exercise-05]] (status page per-feature volume panel reads the per-tier rollups); mod-109 [[exercise-04]] (cost attribution reads `app.cost.usd`, which is allow-listed above).
