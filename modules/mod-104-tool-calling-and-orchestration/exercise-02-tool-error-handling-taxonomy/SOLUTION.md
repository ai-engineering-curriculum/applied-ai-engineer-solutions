### exercise-02: Tool Error Handling Taxonomy — Solution

## 1. Solution overview

The exercise asks the engineer to design the **error surface** for the tool layer on the helpdesk-reply-copilot. A tool error is never "just an exception" — depending on what broke, the surface wants one of four responses: *let the model fix it*, *retry the same call*, *stop the loop and tell the user*, or *trip the circuit for the whole session*. If the surface collapses all errors into a single "tool failed" string, the model will try the same bad call three more times, the user watches the latency bar grind, and the on-call sees a wall of `ToolError: unknown` in the logs.

The solution is a four-category taxonomy, a deterministic error envelope that every tool emits, a routing policy the orchestrator applies, and a counter set that makes the health of each category observable.

This solution provides:

- A **taxonomy** of four tool-error categories with concrete examples drawn from the three tools from [[exercise-01]]: `model_fixable`, `retryable`, `permanent`, and `systemic`.
- A reference **`ToolError` envelope** (Pydantic + JSON Schema) that every tool returns, with the fields required to make the routing decision deterministic.
- A reference **`route_tool_error.py`** implementing the policy: *fix* → re-prompt with the model-hint and advance the turn counter; *retry* → exponential backoff with a bounded retry count; *permanent* → stop the loop, surface a user-facing "we couldn't do this" message; *systemic* → trip the session circuit, degrade the surface, open an incident.
- A **counter set** (`metrics.py`) that increments by `(tool, category, reason_code)` so the readout shows which tool is drifting into which failure mode.
- A **worked trace** for each category showing the raw backend failure → tool-side wrapping → envelope on the wire → orchestrator decision → user-visible outcome.
- A grading rubric.

The lesson is that *error semantics are a product decision*, not a library default. A 429 from a downstream read is retryable; a 429 from the write path when the user is already past the per-request budget is systemic. The taxonomy is the place that decision lives.

## 2. Worked answer or implementation

### 2a. The four-category taxonomy (deliverable 1)

| Category | Who fixes it | Orchestrator action | Example (on the helpdesk surface) |
|---|---|---|---|
| **model_fixable** | The model. | Re-prompt with the `model_hint`; advance the tool-call counter from [[exercise-05]]; cap at `MAX_REPAIR_ATTEMPTS=2` per tool per turn. | `lookup_ticket({"ticket_id": "the last one"})` — pattern violation. |
| **retryable** | The host, via retry. | Exponential backoff (250 ms, 1 s, 4 s) with jitter; cap at 3 attempts; carry the `retry_after_ms` hint if the backend provided one; the model never sees transient retries. | `search_knowledge_base` → vector DB 503; Zendesk 429 with `Retry-After: 2`. |
| **permanent** | Nobody can fix it in this turn. | Stop the tool loop; return a user-facing message from the envelope's `user_facing_message`; do NOT re-prompt the model with a fix hint; log a `permanent` counter. | `lookup_ticket("ZD-123456")` → 404. The ticket does not exist. |
| **systemic** | The on-call. | Trip the session circuit for this tool class (reads vs. writes); fall back to the read-only surface; page the on-call if the rate crosses the threshold; refuse further calls in this session with a precise reason. | Zendesk API has returned 5xx > 20 % for the last 60 s; auth token revoked mid-session; per-request budget exhausted ([[exercise-05]]). |

Explicit non-category: an *unclassified* error is a bug. Every error path in every tool wrapper has to land in one of the four buckets; the CI test in §2e enforces that the raw exception types all map.

### 2b. The `ToolError` envelope (deliverable 2)

```python
"""tool_error.py — the on-the-wire shape of every tool failure."""

from __future__ import annotations

from datetime import datetime, timezone
from typing import Literal, Optional

from pydantic import BaseModel, Field, field_validator


Category = Literal["model_fixable", "retryable", "permanent", "systemic"]


class ToolError(BaseModel):
    # Classification
    category: Category
    # A short stable string — joins the tool name with a reason;
    # the readout counts by (tool, reason_code).
    reason_code: str = Field(pattern=r"^[a-z0-9_]+\.[a-z0-9_]+$")

    # Operator-facing message; may contain IDs and backend details.
    # Logged; never shown to the end user or the model.
    message: str

    # Shown to the model on a model_fixable re-prompt.
    # None on retryable/permanent/systemic (the model does not see those).
    model_hint: Optional[str] = None

    # Shown to the end user when category=permanent or when
    # systemic has tripped the circuit. None on model_fixable/retryable.
    user_facing_message: Optional[str] = None

    # Retry hint from the backend, if any.
    retry_after_ms: Optional[int] = Field(default=None, ge=0, le=60_000)

    # Context for the readout and traces.
    tool: str
    occurred_at: datetime
    request_id: str

    @field_validator("occurred_at")
    @classmethod
    def _tz_aware(cls, v: datetime) -> datetime:
        if v.tzinfo is None:
            raise ValueError("occurred_at must be timezone-aware")
        return v.astimezone(timezone.utc)

    @field_validator("model_hint")
    @classmethod
    def _hint_only_for_fixable(cls, v, info):
        # Pydantic v2: `info.data` holds already-validated fields.
        cat = info.data.get("category")
        if cat == "model_fixable" and not v:
            raise ValueError("model_fixable errors require model_hint")
        if cat != "model_fixable" and v is not None:
            raise ValueError(
                "model_hint is only set on model_fixable errors; "
                "other categories must not leak to the model."
            )
        return v

    @field_validator("user_facing_message")
    @classmethod
    def _user_msg_required_for_permanent(cls, v, info):
        cat = info.data.get("category")
        if cat == "permanent" and not v:
            raise ValueError("permanent errors require user_facing_message")
        return v
```

Design choices worth making explicit:

- `model_hint` and `user_facing_message` are *mutually exclusive* in practice. The envelope enforces it in the validators. This is the "which audience is this error for?" decision — the taxonomy is exactly the answer.
- `reason_code` is stable. It joins `tool` (`lookup_ticket`) with a short reason (`not_found`), giving the readout a grouping key that survives message wording changes.
- `retry_after_ms` is bounded to 60 s. A tool that asks for a longer back-off is systemic, not retryable.
- The envelope is one shape across all tools. A single error envelope makes the orchestrator's switch statement finite; a per-tool error shape pushes the switch into every call site.

### 2c. Wrapping raw backend failures into the envelope

```python
"""tool_errors_wrap.py — map raw backend exceptions to ToolError.

Each tool wrapper has a mapping table. Unknown exceptions raise
UnclassifiedToolError, which CI treats as a test failure (every error
path must be in the taxonomy).
"""

from __future__ import annotations

import httpx
import jsonschema

from tool_error import ToolError
from validate_tool_call import ToolValidationError


class UnclassifiedToolError(Exception):
    """Raised when a raw backend error has no taxonomy mapping. CI-fatal."""


def _now():
    # Injected in tests; keeps the envelope deterministic for golden-file assertions.
    from datetime import datetime, timezone
    return datetime.now(timezone.utc)


def wrap_lookup_ticket_failure(exc: Exception, request_id: str) -> ToolError:
    now = _now()
    if isinstance(exc, ToolValidationError):
        # Structural / semantic / principal errors from [[exercise-01]]
        # are model_fixable (structural, semantic) or permanent (principal).
        if exc.category == "principal":
            return ToolError(
                category="permanent",
                reason_code="lookup_ticket.forbidden",
                message=str(exc),
                user_facing_message="You don't have access to this ticket.",
                tool="lookup_ticket",
                occurred_at=now,
                request_id=request_id,
            )
        return ToolError(
            category="model_fixable",
            reason_code=f"lookup_ticket.{exc.category}_arg",
            message=str(exc),
            model_hint=exc.model_hint,
            tool="lookup_ticket",
            occurred_at=now,
            request_id=request_id,
        )
    if isinstance(exc, httpx.HTTPStatusError):
        s = exc.response.status_code
        if s == 404:
            return ToolError(
                category="permanent",
                reason_code="lookup_ticket.not_found",
                message=f"zendesk 404 for {exc.request.url}",
                user_facing_message="That ticket wasn't found.",
                tool="lookup_ticket",
                occurred_at=now,
                request_id=request_id,
            )
        if s == 401 or s == 403:
            return ToolError(
                category="systemic",
                reason_code="lookup_ticket.auth",
                message=f"zendesk {s}; auth token may be revoked",
                user_facing_message=(
                    "We've lost access to Zendesk — try again in a minute."
                ),
                tool="lookup_ticket",
                occurred_at=now,
                request_id=request_id,
            )
        if s == 429:
            retry_after = int(float(exc.response.headers.get("retry-after", 1)) * 1000)
            return ToolError(
                category="retryable",
                reason_code="lookup_ticket.rate_limited",
                message=f"zendesk 429; retry-after {retry_after}ms",
                retry_after_ms=min(retry_after, 60_000),
                tool="lookup_ticket",
                occurred_at=now,
                request_id=request_id,
            )
        if 500 <= s < 600:
            return ToolError(
                category="retryable",
                reason_code="lookup_ticket.upstream_5xx",
                message=f"zendesk {s}",
                tool="lookup_ticket",
                occurred_at=now,
                request_id=request_id,
            )
    if isinstance(exc, (httpx.ConnectError, httpx.ReadTimeout)):
        return ToolError(
            category="retryable",
            reason_code="lookup_ticket.network",
            message=f"network: {type(exc).__name__}",
            tool="lookup_ticket",
            occurred_at=now,
            request_id=request_id,
        )
    raise UnclassifiedToolError(f"no taxonomy mapping for {type(exc).__name__}") from exc
```

### 2d. `route_tool_error.py` — the orchestrator's policy (deliverable 3)

```python
"""route_tool_error.py — single policy for every ToolError."""

from __future__ import annotations

import asyncio
import random
from dataclasses import dataclass, field
from typing import Callable, Optional

from tool_error import ToolError


MAX_REPAIR_ATTEMPTS = 2  # per tool per model turn
MAX_RETRYABLE_ATTEMPTS = 3  # host-side; model is unaware
RETRY_BACKOFF_MS = (250, 1_000, 4_000)
SYSTEMIC_CIRCUIT_WINDOW_S = 60
SYSTEMIC_CIRCUIT_THRESHOLD = 3  # trips after N systemic errors in the window


@dataclass
class SessionState:
    # tool -> repair attempts this turn
    repair_attempts: dict[str, int] = field(default_factory=dict)
    # tool_class ("read" | "write") -> circuit tripped?
    circuit_tripped: dict[str, bool] = field(default_factory=dict)
    # recent systemic events for the circuit breaker
    recent_systemic: list[float] = field(default_factory=list)


@dataclass
class RouteDecision:
    action: str  # "reprompt" | "retry" | "stop" | "abort_session"
    model_hint: Optional[str] = None
    user_facing_message: Optional[str] = None
    retry_after_ms: int = 0
    counter: Optional[str] = None  # emit this in metrics


def route(
    err: ToolError, state: SessionState, now_s: float, tool_class: str
) -> RouteDecision:
    counter = f"tool_error.{err.tool}.{err.reason_code}"
    if err.category == "model_fixable":
        state.repair_attempts[err.tool] = state.repair_attempts.get(err.tool, 0) + 1
        if state.repair_attempts[err.tool] > MAX_REPAIR_ATTEMPTS:
            return RouteDecision(
                action="stop",
                user_facing_message=(
                    "I couldn't complete that step after a few tries. "
                    "A human will need to pick it up."
                ),
                counter=f"{counter}.repair_cap",
            )
        return RouteDecision(
            action="reprompt", model_hint=err.model_hint, counter=counter,
        )
    if err.category == "retryable":
        return RouteDecision(
            action="retry",
            retry_after_ms=err.retry_after_ms or 0,
            counter=counter,
        )
    if err.category == "permanent":
        return RouteDecision(
            action="stop",
            user_facing_message=err.user_facing_message,
            counter=counter,
        )
    # systemic
    state.recent_systemic = [
        t for t in state.recent_systemic if now_s - t <= SYSTEMIC_CIRCUIT_WINDOW_S
    ]
    state.recent_systemic.append(now_s)
    if len(state.recent_systemic) >= SYSTEMIC_CIRCUIT_THRESHOLD:
        state.circuit_tripped[tool_class] = True
        return RouteDecision(
            action="abort_session",
            user_facing_message=(
                err.user_facing_message
                or "We're having trouble with that backend right now. "
                   "Please try again in a few minutes."
            ),
            counter=f"{counter}.circuit_tripped",
        )
    return RouteDecision(
        action="stop",
        user_facing_message=err.user_facing_message,
        counter=counter,
    )


async def execute_with_retry(
    call: Callable[[], "object"],
    tool: str,
    tool_class: str,
    wrap_failure: Callable[[Exception, str], ToolError],
    state: SessionState,
    request_id: str,
) -> tuple[Optional[object], Optional[RouteDecision]]:
    """Returns (result, None) on success or (None, decision) on terminal failure."""
    for attempt in range(MAX_RETRYABLE_ATTEMPTS + 1):
        try:
            return await call(), None
        except Exception as exc:  # noqa: BLE001
            err = wrap_failure(exc, request_id)
            decision = route(err, state, _monotonic_s(), tool_class)
            if decision.action != "retry":
                return None, decision
            if attempt == MAX_RETRYABLE_ATTEMPTS:
                return None, RouteDecision(
                    action="stop",
                    user_facing_message=(
                        "That step kept failing. We'll try again later."
                    ),
                    counter=f"tool_error.{tool}.retry_exhausted",
                )
            backoff = max(decision.retry_after_ms, RETRY_BACKOFF_MS[attempt])
            await asyncio.sleep((backoff + random.uniform(0, 50)) / 1000)


def _monotonic_s() -> float:
    # Injected in tests so the circuit window is deterministic.
    import time
    return time.monotonic()
```

The policy enforces three guardrails:

- **The model never sees a retry.** Retryable failures resolve inside the host; if they exhaust, the surface shows a user-facing message. Telling the model "we got a 503 three times" is tokens spent to re-learn what the host already knows.
- **Repair attempts are capped per tool per turn.** A model that cannot repair `lookup_ticket` in two tries is in a loop; stop and tell the user.
- **Systemic failures trip a session-scoped circuit**, not a global one. A global circuit would take down healthy tenants on a single bad one.

### 2e. Observability (deliverable 4)

```python
"""metrics.py — counters for the tool error taxonomy."""

from collections import Counter

TOOL_ERROR_COUNTERS: Counter = Counter()


def record(decision_counter: str) -> None:
    if decision_counter:
        TOOL_ERROR_COUNTERS[decision_counter] += 1


# Readout shape (daily):
#   tool_error.lookup_ticket.not_found            102   (permanent — expected on 404s)
#   tool_error.lookup_ticket.rate_limited          44   (retryable — zero if backend is healthy)
#   tool_error.search_knowledge_base.model_fixable 18   (model — watch; should trend down)
#   tool_error.create_followup_task.auth            3   (systemic — pageable)
#   tool_error.create_followup_task.auth.circuit_tripped 1
```

The counters are joined by `.` so a Grafana group-by on a prefix (`tool_error.lookup_ticket.*`) gives a per-tool failure breakdown. The two most useful alerts:

- `sum(tool_error.*.systemic) by (tool) > 3 in 60s` → page the on-call.
- `rate(tool_error.*.model_fixable) by (tool)` trending up over a week → the tool description or arg schema needs tightening (loops back to [[exercise-01]]).

### 2f. Worked traces (deliverable 5)

**Trace A — `model_fixable`**

```
raw:    ToolValidationError(category="structural", model_hint="Field 'ticket_id' must match pattern '^ZD-[0-9]{6,10}$'.")
wrap:   ToolError(category=model_fixable, reason_code=lookup_ticket.structural_arg,
                  model_hint="Field 'ticket_id' must match pattern '^ZD-[0-9]{6,10}$'.")
route:  RouteDecision(action=reprompt, model_hint="Field 'ticket_id' must …", counter=tool_error.lookup_ticket.structural_arg)
turn n+1: model re-emits lookup_ticket({"ticket_id": "ZD-123456"}) → success
counter: tool_error.lookup_ticket.structural_arg += 1
```

**Trace B — `retryable`**

```
raw:    httpx.HTTPStatusError(429, headers={"retry-after": "2"})
wrap:   ToolError(category=retryable, reason_code=lookup_ticket.rate_limited, retry_after_ms=2000)
route:  RouteDecision(action=retry, retry_after_ms=2000)
host:   sleep(2000 + jitter ms) → retry → success
model:  sees a normal tool result; no awareness of the 429
counter: tool_error.lookup_ticket.rate_limited += 1
```

**Trace C — `permanent`**

```
raw:    httpx.HTTPStatusError(404)
wrap:   ToolError(category=permanent, reason_code=lookup_ticket.not_found,
                  user_facing_message="That ticket wasn't found.")
route:  RouteDecision(action=stop, user_facing_message="That ticket wasn't found.")
surface: renders the user-facing message as a conversation turn; agent loop terminates.
model:  does not re-prompt (otherwise it will try to invent a different ticket ID)
counter: tool_error.lookup_ticket.not_found += 1
```

**Trace D — `systemic`**

```
raw:    httpx.HTTPStatusError(401) — token revoked mid-session
wrap:   ToolError(category=systemic, reason_code=lookup_ticket.auth)
route:  RouteDecision(action=stop, user_facing_message="We've lost access to Zendesk — try again in a minute.")
… three systemic events for tool_class=read in 60 s …
route:  RouteDecision(action=abort_session, counter=tool_error.lookup_ticket.auth.circuit_tripped)
surface: switches to read-only mode; disables all Zendesk tools for this session; shows a banner; opens an incident ticket.
```

### 2g. Implementation notes

- A `permanent` error does *not* re-prompt the model. Doing so is the number-one way to turn a user-visible 404 into a hallucinated ticket ID that happens to exist in another tenant.
- `retry_after_ms` is strictly the backend's hint. The host applies its own backoff curve on top (the `max(...)` in `execute_with_retry`). Trusting a backend's `Retry-After: 300` would blow the per-request wall-clock budget from [[exercise-05]].
- Error envelope fields are logged with the `request_id` from the retriever contract ([[exercise-01]] mod-103). This is what makes incident traces join across the surface, tool layer, and retriever.
- `user_facing_message` strings are reviewed by the content team before they ship. They are not formatted from templates with backend identifiers; the identifier goes in `message`.
- The CI test that forbids unclassified exceptions is as important as the taxonomy itself. See §3.

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. The four categories are named exactly `model_fixable`, `retryable`, `permanent`, `systemic`. A learner who adds a fifth bucket has either missed one category (likely `systemic`) or re-invented one.
2. The `ToolError` envelope enforces: `model_hint` is set *iff* `category == "model_fixable"`; `user_facing_message` is set *iff* `category in {"permanent", "systemic"}`.
3. The `route(...)` function is pure (no I/O) and returns a `RouteDecision` whose `action` is one of `{reprompt, retry, stop, abort_session}`.
4. `execute_with_retry(...)` caps retryable at 3 attempts and never shows the retry to the model.
5. The systemic circuit trips after 3 events in 60 s on a single tool class and is session-scoped, not global.
6. Every tool has a `wrap_<tool>_failure` function with explicit mappings; any unmapped exception raises `UnclassifiedToolError`. A pytest grep-style test asserts no `wrap_*` function has a bare `except Exception: return ToolError(...)` that short-circuits taxonomy.
7. The metrics counter uses the shape `tool_error.<tool>.<reason_code>` and extends with `.circuit_tripped` / `.repair_cap` / `.retry_exhausted` as appropriate.
8. The five worked traces are present and each shows (raw → wrap → route → surface outcome → counter).
9. Alerts: there is at least one documented alert for systemic rate and one for a drifting `model_fixable` rate. The exercise answer explains why the two alerts have very different urgency.
10. No `permanent` trace re-prompts the model; no `model_fixable` trace reaches the end user.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Four categories, correctly named | 1 | Exactly four; `systemic` is distinct from `retryable` with examples. |
| Envelope enforces hint/message exclusivity | 1 | Pydantic validators reject invalid combinations. |
| Routing policy returns one of four actions | 1 | `route(...)` is pure; `execute_with_retry` is thin. |
| Model never sees retries | 1 | Retries are inside the host loop; traces show this. |
| Repair capped per tool per turn | 1 | `MAX_REPAIR_ATTEMPTS` enforced; test case exists. |
| Circuit breaker is session-scoped | 1 | Trips on (tool_class, session); documented. |
| Unclassified errors are CI-fatal | 1 | `UnclassifiedToolError` + CI test that forbids bare `except`. |
| Counter shape is query-friendly | 1 | `tool_error.<tool>.<reason_code>`; group-by works. |
| Five traces, one per category + one retry | 1 | Each trace lists raw → wrap → route → surface → counter. |
| Alerts named and tiered | 1 | Pageable systemic alert; non-pageable drift alert for `model_fixable`. |

A submission scoring < 6 is rejected. The hard failures are (a) re-prompting the model on `permanent`, (b) showing retryable failures to the model, (c) a global circuit instead of a session-scoped one.

## 5. Common mistakes

- **Collapsing `retryable` and `systemic`.** A 503 from a healthy tenant is retryable; the *rate* of 503s across the session is systemic. The policy needs both.
- **Re-prompting the model on `permanent`.** The model will fabricate a different ticket ID that happens to exist elsewhere. Permanent means stop.
- **Leaking operator details to the user.** `user_facing_message` cannot contain IDs, stack frames, or "zendesk." It is a product sentence.
- **Leaking user-facing text to the model.** The model does not see `user_facing_message`; it sees `model_hint` only on `model_fixable`. Confusing the two makes every retry a prompt-injection surface.
- **Unbounded retry.** `Retry-After: 300` from the backend becomes a 5-minute wall-clock spike. Cap `retry_after_ms` and the number of attempts.
- **Global circuit breaker.** One bad tenant's auth revocation trips reads for everyone. Scope the circuit to the session.
- **No counter.** If you cannot see `rate(model_fixable) by tool` go up after a prompt change, you ship regressions. The counter is as important as the taxonomy.
- **Catching `Exception` and wrapping as `permanent`.** This mutes systemic failures as "user sees a 'not found' banner" while the backend is on fire. Unknown exceptions must raise `UnclassifiedToolError` so CI fails, not dropped into a soft category.
- **One retry policy for reads and writes.** Idempotent reads retry freely; writes need an idempotency key per retry to avoid duplicate `create_followup_task` rows.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the taxonomy's hint/message split is the surface answer to ML06 (output integrity): the model's error-response path is itself an input surface, and uncontrolled error strings are a prompt-injection channel.
- MITRE ATLAS — <https://atlas.mitre.org/> — the systemic category covers the ATLAS "Denial of ML Service" family (exhaustion of downstream credentials, forced error loops); the session-scoped circuit breaker is the mitigation.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the counter set is a MEASURE function; the circuit breaker is a MANAGE control. Both feed the readout templated in [[exercise-03]] (mod-103).

Related exercises: [[exercise-01]] (mod-104) emits `ToolValidationError.category` which becomes the `model_fixable` reason subcode; [[exercise-03]] (mod-104) wraps these tools in an MCP server whose transport errors feed `wrap_*_failure`; [[exercise-05]] (mod-104) enforces per-request budgets — a budget-exhausted error is `systemic`; the retriever contract in [[exercise-01]] (mod-103) owns the `request_id` these traces join on.
