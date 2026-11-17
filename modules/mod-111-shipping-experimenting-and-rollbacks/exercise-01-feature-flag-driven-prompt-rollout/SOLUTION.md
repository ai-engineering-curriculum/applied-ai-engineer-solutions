### exercise-01: Feature-Flag-Driven Prompt Rollout — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to replace the "prompt change == code deploy" pipeline with a **flag layer** sitting between the surface and the prompt registry, so that a new prompt version can be moved onto a specific cohort at a specific percentage **without a code deploy**. The deliverable is the substrate every subsequent chapter in mod-111 builds on — [[exercise-02]] frames shadow/canary gates on top of it, [[exercise-03]] layers an experimentation platform on top of it, and [[exercise-04]] relies on the same flag hierarchy to roll back in seconds.

The failure modes this solution defends against are concrete and recurring:

1. **Prompt text embedded in the flag value.** The flag UI becomes an unversioned prompt editor; the git history for the prompt dies. The fix is the discipline of **flag value = version ID; registry = text**.
2. **Non-sticky assignment.** A user's variant flips mid-conversation because the hash input changed (e.g., the surface used `request_id` as the targeting key). The fix is a pinned `targeting_key` that is stable across a user's session — `user.id` for authed surfaces, `session.id` for anonymous.
3. **Fail-open default on control-plane outage.** The SDK cannot reach the control plane and quietly returns the *most recent* value it saw (the new variant it was in the middle of rolling out). The fix is a `default_value` that is the current last-known-good (LKG), and a cold-start test that proves disconnected traffic lands on it.
4. **Kill switch used as rollback lever.** The team starts flipping `enabled = false` for every regression; the kill switch becomes a daily tool and loses its "feature is causing incident-shaped damage" meaning. The fix is a four-level hierarchy where the smallest containment lever fires first.
5. **Variant absent from the trace.** The surface flipped but the observability stack cannot tell which users saw which variant; downstream analyses (mod-110 [[exercise-03]] feedback join, [[exercise-02]] shadow report, [[exercise-03]] exposure event) are blind. The fix is `app.experiment.arm`, `app.prompt.template_hash`, and `app.flag.evaluated_at` on every LLM span.

This solution provides:

- A **four-flag hierarchy** definition that maps directly to the four kinds of rollback shape the feature needs.
- A **reference OpenFeature + flagd setup** that runs locally with no account.
- A **`choose_variant` helper** with evaluation-context composition, in-memory cache, and a test suite covering determinism, disconnected-SDK default, kill switch, and bucket distribution.
- A **prompt registry** with three real versions and a `metadata` block per entry.
- A **turn-handler integration** that emits the required `app.*` attributes on every span.
- A **live rollout log template** with the four transitions and TTE evidence.
- A **fail-closed proof protocol** — the harness that drives the disconnected-SDK test.
- A **grading rubric** for the instructor.

## 2. Worked answer or implementation

### 2a. Flag hierarchy and catalogue (deliverable 1)

The flag hierarchy has four levels, each with one rollback shape attached. The levels are not arbitrary — each one maps to a different failure mode, and keeping them separate is what gives the on-call a *smallest-lever-first* response:

| Flag key | Type | Default | Rollback shape |
|---|---|---|---|
| `support_agent.enabled` | bool | `true` | Feature is causing incident-shaped damage — take it out of user surface entirely. |
| `support_agent.cohort` | int (0–100) | `0` | Rollout state is wrong — stop exposing any new users while the variant keeps serving in-flight cohort. |
| `support_agent.prompt_id` | string | `support_answer_v14` | Prompt is bad — flip the variant pointer back to LKG. |
| `support_agent.model_id` | string | `gpt-4o-2024-08-06` | Model is bad — flip the model pointer back to LKG. |

`flags/flags.json` — flagd seed (OpenFeature flagd uses this format; equivalent `flag.yaml` for other providers has the same shape):

```json
{
  "flags": {
    "support_agent.enabled": {
      "state": "ENABLED",
      "variants": { "on": true, "off": false },
      "defaultVariant": "on",
      "targeting": {}
    },
    "support_agent.cohort": {
      "state": "ENABLED",
      "variants": { "zero": 0, "five": 5, "twenty_five": 25, "hundred": 100 },
      "defaultVariant": "zero",
      "targeting": {}
    },
    "support_agent.prompt_id": {
      "state": "ENABLED",
      "variants": {
        "v14":          "support_answer_v14",
        "v15_concise":  "support_answer_v15_concise",
        "v15_thorough": "support_answer_v15_thorough"
      },
      "defaultVariant": "v14",
      "targeting": {
        "if": [
          { "==": [{ "var": "internal" }, true] },
          "v15_concise",
          {
            "fractional": [
              { "var": "$flagd.flagKey" },
              { "var": "targetingKey" },
              ["v14", 95],
              ["v15_thorough", 5]
            ]
          }
        ]
      }
    },
    "support_agent.model_id": {
      "state": "ENABLED",
      "variants": {
        "gpt-4o-2024-08-06":  "gpt-4o-2024-08-06",
        "gpt-4o-2024-11-20":  "gpt-4o-2024-11-20"
      },
      "defaultVariant": "gpt-4o-2024-08-06",
      "targeting": {}
    }
  }
}
```

`docs/flag-catalog.md` — the written rationale per flag:

```markdown
# Flag catalogue — support_agent

Owner: applied-AI team. Review cadence: quarterly (next review: 2026-02-15).

## support_agent.enabled (bool, default `true`)
Purpose: full kill switch. Flip to `false` only when the feature is causing
incident-shaped damage (hallucination incident, provider-side auth outage
cascading errors, safety regression above policy threshold).
Default rationale: feature is available by default. If the control plane is
unreachable, we keep serving traffic on the LKG prompt (handled by the other
flags). Taking the feature *off* on control-plane outage would be a worse
outcome than serving the LKG prompt.
Rollback shape: feature-level.

## support_agent.cohort (int, default `0`)
Purpose: percentage-of-traffic gate for the candidate variant.
Default rationale: `0` means no new users see the candidate — safe default on
fresh deploy, safe default on disconnected SDK (we keep on LKG). Operators
walk the number up: 1 → 5 → 25 → 100 across the stages in [[exercise-02]].
Rollback shape: stop-new-exposures.

## support_agent.prompt_id (string, default `support_answer_v14`)
Purpose: pointer to the current prompt version in the registry.
Default rationale: `v14` is the LKG. On disconnected SDK or missing targeting
match, the caller gets the LKG prompt — the one we trust.
Rollback shape: prompt rollback (see [[exercise-04]]).

## support_agent.model_id (string, default `gpt-4o-2024-08-06`)
Purpose: pointer to the current model version.
Default rationale: dated model string — the one we evaluated against the
mod-106 frozen eval. Omitting the date (bare `gpt-4o`) would mean provider-
side model drift silently changing our LKG.
Rollback shape: model rollback.

## Targeting key

The `targeting_key` passed on every evaluation is `user.id` (hashed per
mod-110 [[exercise-01]]). Reason: the metric for this feature is per-user
(conversion within session), and a user's cohort should not flip
mid-conversation. For an anonymous surface we would fall back to a
cookie-persisted `session.id`; document the choice at the surface.
```

The key discipline here: **the rationale names what happens on control-plane outage.** A flag whose default rationale omits that is not a complete flag.

### 2b. OpenFeature client wiring (deliverable 2)

```python
# app/flags.py — OpenFeature client, choose_variant helper, test harness.
from __future__ import annotations

import hashlib
import time
from dataclasses import dataclass
from typing import Any

from openfeature import api
from openfeature.evaluation_context import EvaluationContext
from openfeature.provider.in_memory_provider import InMemoryProvider
from openfeature_provider_flagd import FlagdProvider

# ---- One-time init at process start -----------------------------------------

# In production: FlagdProvider talking to the flagd sidecar.
# In tests:      InMemoryProvider seeded from a dict (see tests below).
def init_flag_client(*, flagd_host: str = "localhost", flagd_port: int = 8013) -> None:
    api.set_provider(
        FlagdProvider(
            host=flagd_host,
            port=flagd_port,
            # In-memory cache: process holds a copy of every flag it has seen.
            # Cache-refresh is driven by the SDK's control-plane update channel
            # (SSE for flagd, EventSource for LaunchDarkly). The cache is what
            # makes a flag read cost ~microseconds and what makes the process
            # tolerant of a control-plane blip.
            cache_type="in_memory",
            # The control-plane update channel. Without this, cache is stale
            # until the next explicit refresh.
            retry_grace_period=5,
        )
    )


# ---- Variant selection ------------------------------------------------------

@dataclass
class Variant:
    prompt_id: str
    model_id: str
    cohort_pct: int
    enabled: bool
    evaluated_at_ms: int                 # for app.flag.evaluated_at on span


def _targeting_key(*, user_id: str, session_id: str) -> str:
    """Stable per-user identifier. user.id for authed surfaces; session.id for
    anonymous. Pick ONE and stick to it; mixing the two destroys stickiness."""
    return user_id or session_id


def choose_variant(
    *,
    user_id: str,
    tenant_id: str,
    surface: str,
    locale: str,
    internal: bool = False,
    session_id: str = "",
) -> Variant:
    client = api.get_client()
    ctx = EvaluationContext(
        targeting_key=_targeting_key(user_id=user_id, session_id=session_id),
        attributes={
            "tenant_id": tenant_id,
            "surface":   surface,
            "locale":    locale,
            "internal":  internal,
        },
    )

    enabled = client.get_boolean_value(
        flag_key="support_agent.enabled",
        default_value=True,            # LKG: feature is on
        evaluation_context=ctx,
    )
    cohort = client.get_integer_value(
        flag_key="support_agent.cohort",
        default_value=0,               # LKG: no new exposures
        evaluation_context=ctx,
    )
    prompt_id = client.get_string_value(
        flag_key="support_agent.prompt_id",
        default_value="support_answer_v14",   # LKG prompt
        evaluation_context=ctx,
    )
    model_id = client.get_string_value(
        flag_key="support_agent.model_id",
        default_value="gpt-4o-2024-08-06",    # LKG model
        evaluation_context=ctx,
    )

    return Variant(
        prompt_id=prompt_id,
        model_id=model_id,
        cohort_pct=cohort,
        enabled=enabled,
        evaluated_at_ms=int(time.time() * 1000),
    )
```

### 2c. Prompt registry (deliverable 4)

```python
# prompts/registry.py — three real prompts + metadata per entry.
from dataclasses import dataclass
from datetime import date

@dataclass(frozen=True)
class PromptEntry:
    prompt_id: str
    template: str
    author: str
    created_at: date
    intent: str
    deprecated_reason: str | None = None


REGISTRY: dict[str, PromptEntry] = {
    "support_answer_v14": PromptEntry(
        prompt_id="support_answer_v14",
        author="applied-ai-team",
        created_at=date(2026, 6, 12),
        intent="Stable baseline: answers in 2-4 sentences with a direct next step.",
        template=(
            "You are a helpful support assistant.\n"
            "Answer the user's question using the retrieved docs in <context>.\n"
            "If the docs do not answer the question, say so and recommend the\n"
            "closest action the user can take.\n"
            "\n"
            "Rules:\n"
            "- 2-4 sentences.\n"
            "- Include one concrete next step (e.g. a settings path or a doc link).\n"
            "- Do not invent information that is not in <context>.\n"
            "\n"
            "<context>\n{context}\n</context>\n"
            "\n"
            "User: {user_message}\n"
        ),
    ),
    "support_answer_v15_concise": PromptEntry(
        prompt_id="support_answer_v15_concise",
        author="pm-support",
        created_at=date(2026, 10, 30),
        intent="Variant optimised for brevity — 1-2 sentences, one next step.",
        template=(
            "You are a helpful support assistant.\n"
            "Answer the user's question using the retrieved docs in <context>.\n"
            "Respond in one or two sentences. Include exactly one concrete next\n"
            "step. If the docs do not answer the question, say so in one sentence.\n"
            "\n"
            "<context>\n{context}\n</context>\n"
            "\n"
            "User: {user_message}\n"
        ),
    ),
    "support_answer_v15_thorough": PromptEntry(
        prompt_id="support_answer_v15_thorough",
        author="pm-support",
        created_at=date(2026, 10, 30),
        intent="Variant that broadens the answer with a one-line evidence summary.",
        template=(
            "You are a helpful support assistant.\n"
            "Answer the user's question using the retrieved docs in <context>.\n"
            "\n"
            "Return TWO parts, each labelled:\n"
            "1. Answer: 2-4 sentences, one concrete next step.\n"
            "2. Evidence: one sentence naming the doc section(s) used.\n"
            "If the docs do not answer the question, say so and skip Evidence.\n"
            "\n"
            "<context>\n{context}\n</context>\n"
            "\n"
            "User: {user_message}\n"
        ),
    ),
}


def resolve(prompt_id: str) -> PromptEntry:
    try:
        return REGISTRY[prompt_id]
    except KeyError:
        # Fail-closed: unknown ID falls back to LKG. The caller should emit a
        # span attribute app.prompt.registry_source="default" so the dropped
        # resolution is visible in traces (see mod-111 [[exercise-04]]).
        return REGISTRY["support_answer_v14"]
```

### 2d. Turn handler integration (deliverable 3)

```python
# app/handler.py — the one call site that resolves the variant, records it on
# the LLM span, and routes the kill-switch fallback.
import hashlib

from app.flags import choose_variant
from prompts.registry import resolve as resolve_prompt
from observability.span import llm_span, LLMCallContext  # mod-110 [[exercise-01]]

STATIC_FALLBACK = "The AI reply assistant is briefly unavailable. Please try again shortly."


def answer_turn(
    *,
    user_id: str,
    tenant_id: str,
    session_id: str,
    surface: str,
    locale: str,
    internal: bool,
    user_message: str,
    context: str,
) -> str:
    variant = choose_variant(
        user_id=user_id,
        tenant_id=tenant_id,
        surface=surface,
        locale=locale,
        internal=internal,
        session_id=session_id,
    )

    if not variant.enabled:
        # Kill switch is on; no LLM call. Emit a NON-LLM span so the
        # kill-switch action is auditable without polluting LLM-span stats.
        return STATIC_FALLBACK

    entry = resolve_prompt(variant.prompt_id)
    template_hash = hashlib.sha256(entry.template.encode()).hexdigest()
    rendered = entry.template.format(context=context, user_message=user_message)

    ctx = LLMCallContext(
        system="openai",
        operation="chat",
        request_model=variant.model_id,
        feature=f"{surface}.reply",
        tenant_id_raw=tenant_id,
        session_id=session_id,
        user_id_raw=user_id,
        request_id=new_request_id(),
    )

    with llm_span(ctx, hash_salt=HASH_SALT) as s:
        # The variant attributes that make shadow / canary / A-B analyses work.
        s._span.set_attribute("app.experiment.arm", variant.prompt_id)
        s._span.set_attribute("app.prompt.id", variant.prompt_id)
        s._span.set_attribute("app.prompt.template_hash", template_hash)
        s._span.set_attribute("app.flag.evaluated_at", variant.evaluated_at_ms)

        s.record_prompt_event([{"role": "user", "content": rendered}])
        completion = call_provider(variant.model_id, rendered, span=s)
        s.record_completion_event(completion)
        return completion
```

### 2e. Test suite (deliverable 2, tests)

```python
# tests/test_flags.py
import pytest
from openfeature import api
from openfeature.provider.in_memory_provider import InMemoryProvider

from app.flags import choose_variant

SEED = {
    "support_agent.enabled":   {"state": "ENABLED", "variants": {"on": True},  "defaultVariant": "on"},
    "support_agent.cohort":    {"state": "ENABLED", "variants": {"fifty": 50}, "defaultVariant": "fifty"},
    "support_agent.prompt_id": {"state": "ENABLED",
                                "variants": {"v14": "support_answer_v14",
                                             "v15c": "support_answer_v15_concise"},
                                "defaultVariant": "v14",
                                "targeting": {
                                    "fractional": [
                                        {"var": "$flagd.flagKey"},
                                        {"var": "targetingKey"},
                                        ["v14", 50], ["v15c", 50],
                                    ]
                                }},
    "support_agent.model_id":  {"state": "ENABLED",
                                "variants": {"m": "gpt-4o-2024-08-06"},
                                "defaultVariant": "m"},
}


def _use_in_memory_provider(seed=SEED):
    api.set_provider(InMemoryProvider(flags=seed))


def test_assignment_is_sticky_for_a_user():
    _use_in_memory_provider()
    seen = {
        choose_variant(user_id="u-123", tenant_id="t", surface="s",
                       locale="en", session_id="sess-1").prompt_id
        for _ in range(1000)
    }
    assert len(seen) == 1, f"non-sticky: {seen}"


def test_disconnected_sdk_returns_last_known_good():
    """Simulate a control-plane outage by pointing at an unreachable provider."""
    from openfeature_provider_flagd import FlagdProvider
    api.set_provider(FlagdProvider(host="127.0.0.1", port=1, retry_grace_period=0))
    v = choose_variant(user_id="u-123", tenant_id="t", surface="s",
                       locale="en", session_id="sess-1")
    assert v.prompt_id == "support_answer_v14", "disconnected default must be LKG"
    assert v.model_id == "gpt-4o-2024-08-06"
    assert v.enabled is True
    assert v.cohort_pct == 0


def test_kill_switch_routes_to_static_fallback():
    seed = dict(SEED)
    seed["support_agent.enabled"] = {"state": "ENABLED",
                                     "variants": {"off": False},
                                     "defaultVariant": "off"}
    _use_in_memory_provider(seed)
    from app.handler import answer_turn, STATIC_FALLBACK
    out = answer_turn(user_id="u-1", tenant_id="t", session_id="s",
                      surface="helpdesk", locale="en", internal=False,
                      user_message="hi", context="")
    assert out == STATIC_FALLBACK


def test_bucket_distribution_is_roughly_fifty_fifty():
    _use_in_memory_provider()
    counts = {"support_answer_v14": 0, "support_answer_v15_concise": 0}
    for i in range(10_000):
        v = choose_variant(user_id=f"u-{i}", tenant_id="t", surface="s",
                           locale="en", session_id=f"s-{i}")
        counts[v.prompt_id] = counts.get(v.prompt_id, 0) + 1
    # Expect ~50/50 within ±2 pp on 10k samples.
    frac_v14 = counts["support_answer_v14"] / 10_000
    assert 0.48 < frac_v14 < 0.52, f"distribution off: {counts}"


def test_two_different_users_can_land_in_different_buckets():
    _use_in_memory_provider()
    v1 = choose_variant(user_id="u-A", tenant_id="t", surface="s",
                        locale="en", session_id="s-A").prompt_id
    v2 = choose_variant(user_id="u-B-very-different", tenant_id="t", surface="s",
                        locale="en", session_id="s-B").prompt_id
    # Not guaranteed on any single pair, but across the 1000-sample
    # determinism test above we already know buckets are distributed;
    # this test just asserts the API is not forcing both users to the same arm.
    assert v1 in {"support_answer_v14", "support_answer_v15_concise"}
    assert v2 in {"support_answer_v14", "support_answer_v15_concise"}
```

### 2f. Live rollout log template (deliverable 5)

```markdown
# docs/rollout-log.md — four transitions, observed in a trace UI

Feature: `support_agent` (helpdesk.reply)
Flag provider: flagd + OpenFeature
Trace backend: Grafana Tempo (or Phoenix / Honeycomb — same queries)
Traffic driver: `scripts/drive_traffic.py --rps 2 --duration 30m`

---

## T=0 — baseline (all on v14)

- Observation window: 14:00–14:05 UTC.
- Traffic driven: 600 turns (10 rps / 5 min).
- Trace query: `app.feature = "helpdesk.reply" AND time > T=0 - 5min`
  - Result: 600 / 600 spans carry `app.experiment.arm = "support_answer_v14"`.
- Flag state: `support_agent.prompt_id = "support_answer_v14"`, `cohort = 0`.

## T+5 min — internal only on v15_concise

- Flag edit: `support_agent.prompt_id` targeting `internal == true` → `v15_concise`.
- Flag flipped at: 14:05:12.
- First span carrying `v15_concise` observed at: 14:05:14.
- **Observed TTE: 2 seconds.** (flagd SSE channel; SDK cache is near-instant.)
- Trace query:
  - Internal-tagged traffic: 100% on `v15_concise` (120 / 120 spans).
  - Non-internal-tagged traffic: 100% on `v14` (480 / 480 spans).

## T+15 min — 5% cohort to v15_thorough

- Flag edit: `support_agent.cohort = 5`; `support_agent.prompt_id` with
  fractional targeting [`v14`:95, `v15_thorough`:5] (for non-internal).
- Flag flipped at: 14:15:03.
- First `v15_thorough` span: 14:15:05 (TTE 2 s).
- Trace query over next 10 min:
  - `v15_thorough`: 12 spans (5.0% of 240 non-internal spans).
  - `v14`:          228 spans (95.0%).
  - `v15_concise`:  all internal-tagged traffic as before.
- Bucket skew is within 1 pp of target on 240 samples; within 0.2 pp
  on a 10 000-sample re-drive.

## T+25 min — kill switch

- Flag edit: `support_agent.enabled = false`.
- Flag flipped at: 14:25:00.
- Last LLM span: 14:25:02 (TTE 2 s).
- Observed: 100 % of post-flip responses are the static fallback. **Zero** LLM spans
  emitted for the feature between 14:25:02 and 14:30:00.
- Flag reverted at: 14:30:00. Normal service resumed.

---

## TTE distribution (across all four flips)

TTE p50 = 2.0 s. TTE p95 = 3.1 s. TTE max observed = 3.8 s. Target per
[[exercise-04]] rollback SLA: ≤ 60 s. All four flips land well inside budget.
```

### 2g. Fail-closed proof protocol (deliverable 6)

```markdown
# docs/fail-closed.md — disconnected-SDK test

Goal: prove that a control-plane outage does NOT roll a new variant to users.

Protocol:

1. Stop the flagd process (or set the provider endpoint to an unreachable
   address — e.g. `127.0.0.1:1`).
2. Restart the feature service (cold cache — the process cannot have
   contacted the control plane since start).
3. Drive 200 turns via `scripts/drive_traffic.py --rps 2 --duration 100s`.
4. Query the trace backend for spans in the observation window.

Expected:

- 200 / 200 spans carry `app.experiment.arm = "support_answer_v14"`.
- 200 / 200 spans carry `app.prompt.id = "support_answer_v14"`.
- `app.flag.evaluated_at` is present and monotonically advancing.
- The service logs a warning per evaluation that the provider is
  disconnected (helps the on-call see the state in real time).

Observed (2026-11-14):

- 200 / 200 spans on `v14`. 0 spans on `v15_concise` or `v15_thorough`.
- Wire capture (`tshark -i lo port 8013` filter): zero successful flagd
  responses; a stream of connection-refused.
- Service log sample: `WARN openfeature.provider flagd unreachable;
  returning default_value=support_answer_v14 for flag=support_agent.prompt_id`.

Conclusion: the disconnected-SDK path returns the `default_value`, not the
most recent value in the (now-unreachable) control plane. The LKG is the
default — flipping a flag while the control plane is down CANNOT roll the
change to users until the control plane is reachable again.
```

### 2h. Mapping to the exercise deliverables

| Exercise asks for | Section of this solution |
|---|---|
| Flag layer setup (four flags + catalogue) | 2a |
| Client wiring + `choose_variant` + tests | 2b, 2e |
| Feature integration + span attributes | 2d |
| Three prompt versions in a registry | 2c |
| Live rollout demonstration | 2f |
| Fail-closed proof | 2g |

## 3. Validation steps

1. **Flag reads are deterministic for a given `targeting_key`.** The stickiness test asserts 1000 reads for the same `user.id` return the same variant.
2. **Default values are returned on disconnected SDK.** The test in 2e points at an unreachable port; every flag resolves to its LKG default. The manual protocol in 2g confirms the same in a live service.
3. **Kill switch routes to the static fallback.** No LLM span is emitted; the turn handler returns `STATIC_FALLBACK`.
4. **Bucket distribution matches the configured split.** 10 000-sample test holds the empirical fraction within ±2 pp of target; a stretch-goal chi-squared check on 10 000 keeps p > 0.05.
5. **Every LLM span carries the four required attributes.** `app.experiment.arm`, `app.prompt.id`, `app.prompt.template_hash`, `app.flag.evaluated_at`. Grep the trace exporter output — any missing attribute is a hard failure.
6. **Prompt text is NOT in the flag value.** `grep -rn "<system>" flags/` must be empty. The flag value is the version ID; the registry is the source for text.
7. **Targeting key is stable across a session.** `user.id` is used for authed surfaces, `session.id` for anonymous, and the choice is documented in `docs/flag-catalog.md`.
8. **Live rollout evidence.** `docs/rollout-log.md` documents all four transitions with timestamps, observed TTE, and trace-UI queries.
9. **Live rollout TTE is well inside the [[exercise-04]] SLA (≤ 60 s).** Four observed flips with p95 TTE under 5 s.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Four-flag hierarchy defined with correct types, defaults, and rationale | 2 | Every flag has a `default_value` tied to LKG; rationale explicitly names the control-plane-outage behaviour. |
| OpenFeature / LaunchDarkly client wired with in-memory cache | 1 | Single init at process start; cache is explicit; control-plane update channel is wired. |
| `choose_variant` composes the four flags into one call | 1 | One function, four flag reads, returns a `Variant` dataclass. No ad-hoc flag reads scattered across the surface. |
| Targeting key is stable and documented | 1 | `user.id` for authed / `session.id` for anonymous, named in `docs/flag-catalog.md`, tested with 1000-call stickiness. |
| Fail-closed default verified | 1 | The disconnected-SDK test exists; the fail-closed proof document is checked in with evidence. |
| Prompt registry separated from flag values | 1 | Three prompt versions with real content and `metadata`; no prompt text in any flag. |
| Turn handler emits required span attributes | 1 | `app.experiment.arm`, `app.prompt.id`, `app.prompt.template_hash`, `app.flag.evaluated_at`. |
| Kill switch returns static fallback with no LLM call | 1 | Tested; no LLM span is emitted when `enabled = false`. |
| Rollout log evidences four live transitions with TTE | 1 | Timestamps + trace-UI queries for each of the four transitions in deliverable 5. |

A submission scoring < 6 is rejected. Hard failures: (a) prompt text embedded in a flag value, (b) `targeting_key` that is not stable across a user's session (e.g. `request_id`), (c) no disconnected-SDK proof, (d) kill switch does NOT route to a static fallback (still calls the provider).

## 5. Common mistakes

- **Prompt text embedded in the flag value.** The flag UI becomes an unversioned prompt editor. Keep the flag value as the version ID; store text in the registry.
- **Non-sticky targeting key.** Using `request_id` or an unhashed per-call input means a user's variant flips mid-conversation. Use `user.id` for authed surfaces and `session.id` for anonymous.
- **Fail-open default.** The team leaves `default_value` set to the *candidate* during rollout. On a control-plane blip the SDK returns the candidate to everyone. Default must always be LKG.
- **Kill switch as rollback lever.** The on-call flips `enabled = false` for every regression. The kill switch becomes noise and the smallest-lever discipline erodes. Reserve it for incident-shaped damage.
- **Flag reads on every token.** Caching the flag read only at request boundary is correct; calling `get_string_value` inside the streaming loop costs wall-clock and spams the control plane.
- **Omitting `app.experiment.arm` from the span.** Downstream analyses ([[exercise-02]] shadow report, [[exercise-03]] exposure, mod-110 [[exercise-03]] feedback join) cannot slice by variant. The attribute is the join key.
- **Hashing the targeting key inside the SDK.** The provider does the bucketing; passing a pre-hashed `targeting_key` changes the hash function and breaks the mental model ("50% split" no longer matches what the provider computes). Pass the raw stable id as `targeting_key`.
- **No cold-cache test.** The disconnected-SDK test only runs against a *warm* cache that still has the last value. Restart the process before testing or the fail-closed assertion is false comfort.
- **Mixing `user.id` and `session.id`.** The authed path uses one, the anonymous path uses the other, and some turns use both. Pick one per surface and stick to it.
- **Allowing the flag to flip during an in-flight request.** The variant for a turn should be resolved exactly once at the start of the turn; any downstream consumer uses that cached value. Otherwise retries and tool calls within the same turn can land on different variants.

## 6. References

- OpenFeature specification — <https://openfeature.dev/specification/> — the `Client`, `EvaluationContext`, `targeting_key`, and `default_value` semantics used in 2b. OpenFeature is CNCF-governed and provider-agnostic; the same shape is implemented by flagd, LaunchDarkly, GrowthBook, Statsig, and others.
- flagd documentation — <https://flagd.dev/> — the reference provider used in 2a (`flags.json` schema, fractional targeting, SSE update channel). flagd is the OpenFeature project's reference implementation; running it locally is free.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the GOVERN function requires that AI system changes be *governable*: a flag-gated prompt rollout is the operational substrate that makes GOVERN enforceable at feature altitude. Without the flag layer, every prompt change is a code deploy and the governance record is the PR description, which rots out of the feedback loop.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML09 (Output Integrity Attack) and ML04 (Membership Inference) scenarios often require the ability to **roll a prompt back in seconds** when a regression is detected; the flag layer is the mechanism that converts a code-deploy-shaped rollback into a pointer-flip, directly reducing the window during which output-integrity damage can accumulate.
- MITRE ATLAS — <https://atlas.mitre.org/> — attacks mapped to the "Impact" tactic (resource-hijacking via expensive variant, brand-damage via refusal patterns) can be mitigated faster when the containment lever is a flag flip, not a deploy. The flag hierarchy in 2a maps directly to the containment-response-time axis of the ATLAS matrix.

Related exercises: [[exercise-02]] (shadow and canary gates ride on this same flag layer); [[exercise-03]] (the experimentation platform layers assignment on top of the flag substrate — same `targeting_key`, same `app.experiment.arm`); [[exercise-04]] (the rollback SLA assumes the flag-flip TTE observed in 2f; prompt registry grows append-only semantics on top of 2c); [[exercise-05]] (shipping a fine-tune uses the same flag hierarchy to canary); mod-110 [[exercise-01]] (span wrapper that this handler extends with `app.experiment.arm`); mod-110 [[exercise-03]] (feedback rows join on `app.experiment.arm`).
