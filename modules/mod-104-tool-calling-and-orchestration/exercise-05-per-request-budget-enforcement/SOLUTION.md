### exercise-05: Per-Request Budget Enforcement — Solution

## 1. Solution overview

The exercise asks the engineer to put a hard **per-request budget** on the helpdesk-reply-copilot's agent loop — the one wired up in [[exercise-04]] over the MCP tools from [[exercise-03]]. Without a budget, a bad model turn can rack up 12 tool calls, 60 s of wall-clock, and $0.80 of API cost on a single user question. A budget is a *product* control: beyond its four dimensions (tokens, tool calls, wall-clock, dollars), it also has a graceful-degradation plan, a user-visible trip message, and an observable "budget-tripped" readout so the team knows when the cap is doing work.

This solution provides:

- A reference **`RequestBudget`** dataclass covering the four dimensions, with per-surface defaults that match the SLOs from [[exercise-02]] (mod-101).
- A reference **`BudgetMeter`** that charges each resource as it is consumed, exposes a `remaining()` view, and raises `BudgetExceeded` with a `dimension` (`tokens | tool_calls | wall_clock | dollars`) the moment any axis trips.
- A reference **`enforce_budget`** context manager / decorator that wraps the agent loop in [[exercise-04]] and interrupts on trip.
- A **graceful-degradation ladder**: before trip, the surface drops optional work (reranker, extra retrieval `k`, speculative tool calls); on trip, it finalises with whatever draft exists and emits a user-visible "we stopped early" message.
- A **counter and alert set** so the daily readout shows which dimension is tripping on which surface.
- A grading rubric.

The lesson is that a budget is not a rate limit. A rate limit protects the backend; a budget protects the user experience and the per-user cost line. The four dimensions exist because any one of them alone is a bad proxy for the others (tokens ≠ cost on a tool-heavy loop; wall-clock ≠ tokens on a streaming model).

## 2. Worked answer or implementation

### 2a. Why four dimensions (deliverable 1 — decision rationale)

Each dimension protects a distinct property:

| Dimension | What it protects | Why the others don't cover it |
|---|---|---|
| **tokens** | Per-turn context stability; prompt-cache hit rates (mod-102 ex-01). | A loop can burn tokens on long tool outputs that cost little dollar but blow the context budget. |
| **tool_calls** | Agent-loop termination; each tool call is a model turn + a backend hit. | Tokens alone don't catch "10 cheap tool calls that each return a tiny result" — tool_calls does. |
| **wall_clock** | The user's p95 SLO. | Tokens and tool_calls both say nothing about retriever latency spikes. |
| **dollars** | The per-user unit-economics line from the spec (`≤ $0.02 / session` in mod-101 ex-01). | Tokens and dollars correlate but diverge sharply when a tool call hits a per-call-priced service. |

Default caps for the helpdesk-reply-copilot surface:

- `tokens_in + tokens_out ≤ 20_000` across the request.
- `tool_calls ≤ 6` across the request.
- `wall_clock_ms ≤ 15_000` — well under the SLO because the surface adds ~500 ms of render time.
- `dollars ≤ 0.08` — four times the mod-101 session target because one session is many requests.

These numbers are surface-specific and reviewed every month alongside the SLO pack.

### 2b. `budget.py` — the meter and the exception (deliverable 2)

```python
"""budget.py — per-request budget with four dimensions."""

from __future__ import annotations

import time
from dataclasses import dataclass, field
from typing import Literal

Dimension = Literal["tokens", "tool_calls", "wall_clock", "dollars"]


@dataclass(frozen=True)
class RequestBudget:
    tokens: int
    tool_calls: int
    wall_clock_ms: int
    dollars: float

    @staticmethod
    def for_surface(surface_id: str) -> "RequestBudget":
        if surface_id == "helpdesk-reply-copilot":
            return RequestBudget(
                tokens=20_000,
                tool_calls=6,
                wall_clock_ms=15_000,
                dollars=0.08,
            )
        raise ValueError(f"no budget defaults for {surface_id!r}")


class BudgetExceeded(Exception):
    def __init__(self, dimension: Dimension, used: float, limit: float) -> None:
        super().__init__(f"budget exceeded on {dimension}: {used} > {limit}")
        self.dimension: Dimension = dimension
        self.used = used
        self.limit = limit


@dataclass
class BudgetMeter:
    budget: RequestBudget
    request_id: str
    started_monotonic_ms: float = field(default=0.0)
    tokens_used: int = 0
    tool_calls_used: int = 0
    dollars_used: float = 0.0

    def start(self) -> None:
        self.started_monotonic_ms = time.monotonic() * 1000

    def wall_clock_ms_used(self) -> float:
        return time.monotonic() * 1000 - self.started_monotonic_ms

    def remaining(self) -> dict[Dimension, float]:
        return {
            "tokens": self.budget.tokens - self.tokens_used,
            "tool_calls": self.budget.tool_calls - self.tool_calls_used,
            "wall_clock": self.budget.wall_clock_ms - self.wall_clock_ms_used(),
            "dollars": self.budget.dollars - self.dollars_used,
        }

    def charge_tokens(self, n_in: int, n_out: int) -> None:
        self.tokens_used += n_in + n_out
        self._check("tokens", self.tokens_used, self.budget.tokens)

    def charge_tool_call(self) -> None:
        self.tool_calls_used += 1
        self._check("tool_calls", self.tool_calls_used, self.budget.tool_calls)

    def charge_dollars(self, amount: float) -> None:
        self.dollars_used += amount
        self._check("dollars", self.dollars_used, self.budget.dollars)

    def check_wall_clock(self) -> None:
        used = self.wall_clock_ms_used()
        self._check("wall_clock", used, self.budget.wall_clock_ms)

    def _check(self, dim: Dimension, used: float, limit: float) -> None:
        if used > limit:
            raise BudgetExceeded(dim, used, limit)

    def snapshot(self) -> dict:
        """What the readout logs on trip and on normal completion."""
        return {
            "request_id": self.request_id,
            "tokens_used": self.tokens_used,
            "tool_calls_used": self.tool_calls_used,
            "wall_clock_ms_used": self.wall_clock_ms_used(),
            "dollars_used": self.dollars_used,
            "budget": self.budget.__dict__,
        }
```

Design choices:

- `BudgetExceeded.dimension` is enum-typed. The orchestrator branches on it to pick the degradation strategy and the user-visible message.
- The meter owns the clock so tests can inject a fake `monotonic` and the readout is deterministic.
- `check_wall_clock` is explicit, not implicit on every charge. The orchestrator calls it on edges in the LangGraph implementation from [[exercise-04]] and before every model turn.

### 2c. `enforce_budget.py` — wiring into the agent loop (deliverable 3)

```python
"""enforce_budget.py — context manager + decorator that wraps the agent loop.

Integration points:
  - Before every model call: check wall_clock; charge tokens after.
  - Before every tool call: charge tool_calls; wrap call in asyncio.wait_for
                             with remaining wall_clock.
  - On BudgetExceeded: run the degradation ladder and finalize.
"""

from __future__ import annotations

import asyncio
import contextlib
from typing import AsyncIterator

from budget import BudgetMeter, BudgetExceeded, RequestBudget
from tool_error import ToolError


@contextlib.asynccontextmanager
async def enforce_budget(
    budget: RequestBudget, request_id: str
) -> AsyncIterator[BudgetMeter]:
    meter = BudgetMeter(budget=budget, request_id=request_id)
    meter.start()
    try:
        yield meter
    finally:
        _emit_budget_snapshot(meter)


async def model_call_with_meter(meter: BudgetMeter, llm, prompt: str, **kw) -> dict:
    meter.check_wall_clock()
    # The provider's pricing table joins (model, cache_hit) to $/token.
    # price_for is injected for tests; production reads the rate card.
    out = await llm.complete(prompt, **kw)
    meter.charge_tokens(out["usage"]["prompt_tokens"], out["usage"]["completion_tokens"])
    meter.charge_dollars(
        out["usage"]["prompt_tokens"] * out["usage"]["prompt_rate"]
        + out["usage"]["completion_tokens"] * out["usage"]["completion_rate"]
    )
    return out


async def tool_call_with_meter(
    meter: BudgetMeter, mcp, name: str, arguments: dict, tool_dollar_cost: float = 0.0
) -> dict:
    meter.charge_tool_call()  # before the call so the cap is not off-by-one
    remaining_ms = meter.remaining()["wall_clock"]
    if remaining_ms <= 0:
        raise BudgetExceeded("wall_clock", -remaining_ms, 0)
    try:
        result = await asyncio.wait_for(
            mcp.call(name, arguments), timeout=remaining_ms / 1000
        )
    except asyncio.TimeoutError as e:
        raise BudgetExceeded(
            "wall_clock", meter.wall_clock_ms_used(), meter.budget.wall_clock_ms
        ) from e
    meter.charge_dollars(tool_dollar_cost)
    return result


def _emit_budget_snapshot(meter: BudgetMeter) -> None:
    from metrics import TOOL_ERROR_COUNTERS
    snap = meter.snapshot()
    # One row per request; grouped by request_id. Also increments trip counters
    # externally on BudgetExceeded (see metrics.py).
    import logging; logging.getLogger("budget").info("budget_snapshot %s", snap)
```

Wrapping the LangGraph flow from [[exercise-04]]:

```python
# In the LangGraph implementation, every node that charges a resource
# takes `meter` from the graph context and calls `model_call_with_meter`
# or `tool_call_with_meter`. The graph has a top-level `on_edge` hook
# that calls meter.check_wall_clock() between nodes.
```

### 2d. Degradation ladder (deliverable 4)

```python
"""degrade.py — the pre-trip and on-trip degradation strategy."""

from __future__ import annotations

from budget import BudgetMeter, BudgetExceeded, Dimension

# Pre-trip: when < 30 % remaining on any dimension, drop optional work.
DROP_AT_FRACTION = 0.30


def pre_trip_actions(meter: BudgetMeter) -> dict:
    """Returns a dict of flags the orchestrator honours before the next step."""
    rem = meter.remaining()
    flags = {
        "drop_reranker": False,
        "lower_retrieval_k": False,
        "skip_followup_task_proposal": False,
        "stop_after_draft": False,
    }
    if rem["tokens"] / meter.budget.tokens < DROP_AT_FRACTION:
        flags["drop_reranker"] = True
        flags["lower_retrieval_k"] = True
    if rem["tool_calls"] / meter.budget.tool_calls < DROP_AT_FRACTION:
        flags["skip_followup_task_proposal"] = True
    if rem["wall_clock"] / meter.budget.wall_clock_ms < DROP_AT_FRACTION:
        flags["stop_after_draft"] = True
    if rem["dollars"] / meter.budget.dollars < DROP_AT_FRACTION:
        flags["drop_reranker"] = True
        flags["skip_followup_task_proposal"] = True
    return flags


ON_TRIP_MESSAGES: dict[Dimension, str] = {
    "tokens": (
        "I ran out of room to think about this. Here's what I had so far — "
        "a colleague will need to pick up the rest."
    ),
    "tool_calls": (
        "I had to stop after a few lookups. Here's a partial draft — "
        "add anything I missed."
    ),
    "wall_clock": (
        "This took longer than I'm allowed. Here's a partial draft; "
        "please check it before sending."
    ),
    "dollars": (
        "I had to stop early. Here's what I drafted — please review before sending."
    ),
}


def on_trip_finalise(err: BudgetExceeded, partial_draft: str | None) -> dict:
    """Construct the final response envelope after a trip."""
    return {
        "reply": partial_draft or "",
        "truncated": True,
        "trip_dimension": err.dimension,
        "user_facing_message": ON_TRIP_MESSAGES[err.dimension],
    }
```

Key properties:

- The pre-trip ladder *degrades inside the request* before we trip. It drops optional work in a defined order; it never silently drops required steps (the final `draft` is always attempted).
- The on-trip messages name the dimension to the reader without saying "token budget" — the user does not care which axis tripped; they care that something stopped short.
- `trip_dimension` is on the envelope for the readout to count, not for the user to see.

### 2e. Metrics and alerts (deliverable 5)

```python
"""metrics_budget.py — budget counters and derived rates."""

from collections import Counter

BUDGET_COUNTERS: Counter = Counter()
BUDGET_TRIP_COUNTERS: Counter = Counter()


def record_snapshot(snapshot: dict) -> None:
    # Per-request histograms; one increment per completion.
    bucket_tokens = _bucket(snapshot["tokens_used"], [1000, 4000, 8000, 16000])
    BUDGET_COUNTERS[f"budget.tokens.hist.{bucket_tokens}"] += 1


def record_trip(dimension: str, surface_id: str) -> None:
    BUDGET_TRIP_COUNTERS[f"budget.trip.{surface_id}.{dimension}"] += 1


def _bucket(x: float, edges: list[float]) -> str:
    for e in edges:
        if x <= e:
            return f"le_{int(e)}"
    return "overflow"
```

Alerts:

- **Pageable**: `rate(budget.trip.*.wall_clock) > 2 % of requests over 5 min` → the retriever or a downstream tool is slow; join with the tool-error systemic alert from [[exercise-02]].
- **Pageable**: `rate(budget.trip.*.dollars) > 1 % of requests over 5 min` → either a prompt or a model switch is tanking cost control.
- **Non-pageable**: any `budget.trip.*.tokens` or `budget.trip.*.tool_calls` trending up over a week → the prompt or the agent policy is starting to run long; investigate.

### 2f. Test suite (deliverable 6)

```python
"""test_budget.py — unit tests for the meter and the degradation ladder."""

import asyncio
import pytest

from budget import BudgetExceeded, BudgetMeter, RequestBudget
from degrade import pre_trip_actions, on_trip_finalise


@pytest.fixture
def small_budget():
    return RequestBudget(tokens=1_000, tool_calls=2, wall_clock_ms=500, dollars=0.01)


def test_tokens_trip(small_budget):
    m = BudgetMeter(budget=small_budget, request_id="r")
    m.start()
    with pytest.raises(BudgetExceeded) as exc:
        m.charge_tokens(900, 200)
    assert exc.value.dimension == "tokens"


def test_tool_calls_trip(small_budget):
    m = BudgetMeter(budget=small_budget, request_id="r")
    m.start()
    m.charge_tool_call()
    m.charge_tool_call()
    with pytest.raises(BudgetExceeded) as exc:
        m.charge_tool_call()
    assert exc.value.dimension == "tool_calls"


def test_pre_trip_drops_optional(small_budget):
    m = BudgetMeter(budget=small_budget, request_id="r")
    m.start()
    m.charge_tokens(800, 0)  # 80 % used; above the 70 % threshold -> drop reranker
    flags = pre_trip_actions(m)
    assert flags["drop_reranker"]
    assert flags["lower_retrieval_k"]


def test_on_trip_user_message_is_dimension_specific(small_budget):
    exc = BudgetExceeded("wall_clock", 600, 500)
    out = on_trip_finalise(exc, partial_draft="hello")
    assert out["truncated"]
    assert out["trip_dimension"] == "wall_clock"
    assert "longer than" in out["user_facing_message"]


@pytest.mark.asyncio
async def test_wall_clock_trip_on_slow_tool(monkeypatch):
    from enforce_budget import tool_call_with_meter

    class FakeMCP:
        async def call(self, name, args):
            await asyncio.sleep(0.2)  # longer than the remaining budget

    budget = RequestBudget(tokens=10_000, tool_calls=10, wall_clock_ms=50, dollars=1.0)
    m = BudgetMeter(budget=budget, request_id="r")
    m.start()
    with pytest.raises(BudgetExceeded) as exc:
        await tool_call_with_meter(m, FakeMCP(), "search_knowledge_base", {"query": "x"})
    assert exc.value.dimension == "wall_clock"
```

### 2g. Implementation notes

- A budget is per request, not per session. Session-level caps belong in a different control (we have session-scoped rate limits elsewhere; cross-request fairness is not this layer's job).
- The `dollars` calculation requires a rate card. The rate card is versioned and injected; it is not a hard-coded constant. When the provider rolls a new pricing tier, the surface updates the rate card config, and the budget enforcement continues without a code change.
- `BudgetExceeded` on `wall_clock` is categorised as `systemic` in the taxonomy from [[exercise-02]] — but only because it indicates downstream slowness. Budget-exhausted on `tokens` or `tool_calls` is *expected under some prompt regressions* and should not page.
- `charge_tool_call` increments *before* the call. This prevents the "we survived on 6 tool calls but the 7th backend fired" race.
- Cost measurement for third-party tools needs a published per-call price or a fixed estimate. For tools the surface owns, set the price to 0.0; for tools that call a third-party (e.g., an external KB search), the MCP response can carry a `cost_usd` field and the orchestrator passes it to `charge_dollars`.
- The pre-trip ladder is deliberately conservative. In the first month of operation, learners may find the ladder drops the reranker too aggressively — in which case the DROP_AT_FRACTION threshold moves, not the ladder's order.
- Interactions with the retry policy from [[exercise-02]]: every retry eats the wall-clock budget. The host-side retries do not reset the meter. The learner must not "compensate" by raising the wall-clock cap — the cap is the product SLO.

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. `RequestBudget` has exactly four fields: `tokens`, `tool_calls`, `wall_clock_ms`, `dollars`. Three-field answers are rejected; the four dimensions each defend a different property.
2. `BudgetExceeded.dimension` is one of exactly those four strings.
3. `BudgetMeter.charge_tool_call` increments before the call (test exists).
4. `tool_call_with_meter` enforces `wall_clock` via `asyncio.wait_for(remaining_ms)` and raises `BudgetExceeded("wall_clock", ...)` on timeout, not `asyncio.TimeoutError`.
5. The pre-trip ladder has at least four flags and drops in a documented order; the final `draft` is never dropped.
6. On-trip user-visible messages are **dimension-specific**; the same message for all four axes fails.
7. Budget snapshots are logged on every request (success or trip), joined on `request_id`.
8. Alerts: at least one pageable (`wall_clock` or `dollars` trip rate) and one non-pageable (drift on `tokens` or `tool_calls`).
9. Tests cover trip on each of the four dimensions, plus the pre-trip ladder firing when < 30 % remains.
10. The budget layer wraps the agent loop from [[exercise-04]] without the agent loop seeing `BudgetExceeded` directly — the orchestrator catches it and runs `on_trip_finalise`.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Four-dimension design with rationale | 1 | Each dimension named + why the others don't cover it. |
| `BudgetMeter` + `BudgetExceeded(dimension)` | 1 | Typed dimension; enum-shaped. |
| Charge-before-call for `tool_calls` | 1 | Off-by-one race avoided. |
| Wall-clock enforced via `wait_for` with remaining budget | 1 | Not a sleep loop; not a post-hoc check. |
| Pre-trip ladder with documented order | 1 | ≥ 4 flags; required work never dropped. |
| Dimension-specific user messages on trip | 1 | Four messages; readable; no "token budget" leak. |
| Per-request snapshot logged, joined on request_id | 1 | One structured log per request. |
| Trip counters by `(surface, dimension)` | 1 | Shape supports group-by in Grafana. |
| Alerts tiered pageable vs. non-pageable | 1 | At least one of each. |
| Unit tests cover all four trip paths + ladder | 1 | Green pytest. |

A submission scoring < 6 is rejected. The hard failures are (a) only tokens enforced, (b) wall-clock enforced after the call, (c) on-trip returns an exception to the user rather than a finalised partial draft.

## 5. Common mistakes

- **Tokens-only budget.** Catches cost in cheap models, misses it in tool-heavy flows and in slow retrievers. Four dimensions exist because any single one is a bad proxy.
- **Wall-clock enforced after the call.** By the time a 10 s tool call returns, the SLO is already broken. Wall-clock must be enforced via timeout on every blocking operation.
- **Raising `BudgetExceeded` to the user.** The user gets a finalised response envelope — `truncated: True` plus whichever draft was produced — not an exception message. The exception is the host's control flow, not the UX.
- **Charge-after on tool calls.** An off-by-one race: the 7th call fires, the backend bills us, and only then does the meter say "over." Charge first, call second.
- **Omitting pre-trip degradation.** Without it, every near-miss is a hard trip; users see "we stopped" more often than they should.
- **Same message for all four trip dimensions.** The messages differ because the implications differ: wall-clock trip means the backend is slow; dollars trip means a prompt regression or a pricing change; tokens trip means context overflow; tool_calls trip means a loopy agent.
- **Host-side retries from [[exercise-02]] not charging the meter.** A retry that burns wall-clock without decrementing the budget lets a transient 503 blow the SLO.
- **Page on token trip.** Token trips are expected when a prompt grows. Alert on wall-clock and dollars; investigate tokens and tool_calls.
- **Cost rate card hard-coded.** The provider's pricing changes. Put the rate card in config, versioned.
- **Scope creep to session budgets.** A per-session budget is a different control. Don't fold it into the per-request meter; it muddies both.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the budget is the surface answer to ML04 (membership inference) / resource exhaustion paths: a hard per-request cap is the control that bounds how much a crafted prompt can extract or spend.
- MITRE ATLAS — <https://atlas.mitre.org/> — the systemic category of budget trips (wall-clock) is the mitigation for the ATLAS "Denial of ML Service" family; counters make it visible and the pageable alert makes it actionable.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the four-dimension design is a MEASURE artefact (resource consumption categorised), and the pre-trip ladder + on-trip finalisation are MANAGE artefacts (controls that bound the blast radius of a bad turn).

Related exercises: [[exercise-01]] (mod-104) is wrapped by the budget on every validated call; [[exercise-02]] (mod-104)'s retry loop charges wall-clock through this meter; [[exercise-03]] (mod-104)'s MCP client is called via `tool_call_with_meter`; [[exercise-04]] (mod-104)'s orchestrator owns the top-level `enforce_budget` context; the spec target `≤ $0.02 per session` from mod-101 [[exercise-01]] is the primary driver of the `dollars` cap.
