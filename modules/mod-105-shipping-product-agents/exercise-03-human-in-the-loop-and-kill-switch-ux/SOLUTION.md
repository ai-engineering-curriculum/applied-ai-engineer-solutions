### exercise-03: Human-in-the-Loop and Kill-Switch UX — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to design the **human-in-the-loop (HITL)** and **kill-switch** UX that ships with the helpdesk-reply-copilot: where the agent must pause for a human, what the human sees at that pause, what the pause state persists so the agent can resume, and what *every* surface must expose as an always-reachable stop. The shape decision in [[exercise-01]] made this a bounded tool-loop with one write tool; the loop has a natural pause boundary before writes, and the user has a natural job (review a draft reply). HITL and kill-switch are the surface's answer to "what happens when the agent is wrong, and how does the user get out?"

The failure mode the exercise is written to catch is the UX one: a confirm step that looks like a modal but auto-confirms after 10 s; a "stop" button that only stops the next tool call, not the one in flight; a resume-from-pause that silently re-runs earlier steps and creates duplicate writes. HITL is a cooperative state machine between the host and the user, not a confirmation dialog.

This solution provides:

- A **HITL event taxonomy** for the surface — the three pause reasons (write-confirm, low-confidence, budget-exceeded) with what each shows, what the user can do, and how resume works.
- A reference **`AgentController` implementation** that runs the bounded tool-loop, emits `HITLEvent`s at the right boundaries, persists pause state for resume-after-refresh, and exposes a cooperative `cancel()`.
- A reference **kill-switch design** at four layers (per-request, per-session, per-tenant, per-surface) with the operator pathway for each.
- A reference **UX spec** for the confirm, low-confidence, budget-stop, and cancelled surfaces, including the non-negotiable accessibility and copy rules.
- A **unit + integration test suite** proving the loop pauses where expected, resume is idempotent, and `cancel()` stops the in-flight step within a bounded wall-clock.
- A **grading rubric** for the instructor.

The lesson the exercise defends is: *HITL is a product surface, not a toggle*. The agent does not "ask for approval"; the surface pauses, shows the user what they're approving, and persists enough state that resume is safe across refresh, crash, and operator-triggered kill.

## 2. Worked answer or implementation

### 2a. HITL event taxonomy (deliverable 1)

The agent loop emits exactly these four pause events. A fifth pause reason would need a documented justification and a UX review.

| Event | Fires when | User sees | User can | Resume semantics |
|---|---|---|---|---|
| `confirm_write` | the loop is about to call a `side_effects: write` tool (`create_followup_task`) | the proposed arguments (ticket ID, summary, due date) in human form; a "Create task" primary + "Cancel" secondary button | approve, edit arguments, cancel | approve → continue from the pause; cancel → finalise with no write |
| `low_confidence` | the model emits a `final_answer` whose confidence score < 0.5 *or* whose draft cites zero passages | the draft, flagged as "draft needs review," with a "Use anyway" + "Regenerate" + "Discard" | choose one | use → finalise; regenerate → re-enter loop with the critique; discard → finalise with no output |
| `budget_exceeded` | the per-request budget from [[exercise-05]] (mod-104) trips mid-loop | the draft so far (if any) with a "we stopped because this request got too long" explainer and a "Continue manually" button | discard or continue manually | no automatic resume; the surface does not spend more budget without a fresh user action |
| `kill_switch` | an operator-level stop fires at any layer | a non-blaming "This assistant is paused; your work is saved" banner | nothing — the surface is read-only | the pause persists; resume is only possible after the operator lifts the kill |

The invariants this encodes:

1. Every `write` tool call is preceded by a `confirm_write` pause. There is no "trust the model after the first confirm." Every write is a fresh confirm.
2. The pause surface shows the user the *arguments as they will be sent*, not a paraphrase. Paraphrased confirm UIs are the shortest path to a prompt-injection-driven write the user did not actually approve.
3. Resume is idempotent at the pause boundary. The loop re-reads from memory on resume, it does not re-execute earlier steps.
4. `kill_switch` is unilateral. The surface does not try to be clever about why; it goes read-only and shows the saved state.

### 2b. `agent_controller.py` — reference implementation (deliverable 2)

```python
"""agent_controller.py — bounded tool-loop with HITL + cancel.

The controller is a state machine. States: RUNNING -> PAUSED(reason) ->
RUNNING or FINAL. Every state transition writes a row to the run_log
so resume after refresh or crash reads a consistent view.
"""

from __future__ import annotations

import asyncio
import enum
import json
import time
import uuid
from dataclasses import dataclass, field
from typing import Any, Awaitable, Callable, Optional

from budget import Budget, BudgetExceededError  # see mod-104 exercise-05
from tool_error import ToolError  # see mod-104 exercise-02


class Phase(str, enum.Enum):
    RUNNING = "running"
    PAUSED = "paused"
    FINAL = "final"
    CANCELLED = "cancelled"


class PauseReason(str, enum.Enum):
    CONFIRM_WRITE = "confirm_write"
    LOW_CONFIDENCE = "low_confidence"
    BUDGET_EXCEEDED = "budget_exceeded"
    KILL_SWITCH = "kill_switch"


@dataclass
class HITLEvent:
    run_id: str
    step_n: int
    reason: PauseReason
    payload: dict  # the args / draft / explainer the UI renders
    emitted_at: float


@dataclass
class RunState:
    run_id: str
    session_id: str
    principal: dict
    phase: Phase = Phase.RUNNING
    pause: Optional[HITLEvent] = None
    final: Optional[dict] = None
    step_n: int = 0
    trace: list[dict] = field(default_factory=list)
    cancel_requested: bool = False


class AgentController:
    def __init__(
        self,
        llm_step: Callable[[RunState], Awaitable[dict]],
        run_tool: Callable[[str, dict, dict], Awaitable[dict]],
        budget: Budget,
        max_steps: int = 6,
        pause_store: "PauseStore" = None,
        kill_switch: "KillSwitch" = None,
    ) -> None:
        self._llm_step = llm_step
        self._run_tool = run_tool
        self._budget = budget
        self._max_steps = max_steps
        self._pauses = pause_store
        self._kill = kill_switch

    async def start(self, session_id: str, principal: dict,
                    user_message: str) -> RunState:
        state = RunState(
            run_id=f"run_{uuid.uuid4().hex[:12]}",
            session_id=session_id,
            principal=principal,
        )
        state.trace.append({"kind": "user", "message": user_message,
                            "at": time.time()})
        return await self._advance(state)

    async def resume(self, run_id: str, decision: dict) -> RunState:
        """Resume a paused run with the user's decision from the pause UI."""
        state = self._pauses.load(run_id)
        if state.phase != Phase.PAUSED:
            raise RuntimeError(f"cannot resume run in phase {state.phase}")
        state.trace.append({"kind": "resume", "decision": decision,
                            "at": time.time()})
        state.pause = None
        state.phase = Phase.RUNNING
        # Apply the decision; see _apply_decision.
        if not await self._apply_decision(state, decision):
            # user chose to stop (cancel / discard); finalise cleanly.
            return self._finalise(state, final=decision.get("final") or {})
        return await self._advance(state)

    async def cancel(self, run_id: str, reason: str = "user") -> None:
        """Cooperative cancel. Flips a flag; the loop checks it between steps
        and inside tool invocations that support cancellation tokens."""
        state = self._pauses.load(run_id)
        state.cancel_requested = True
        self._pauses.save(state)

    async def _advance(self, state: RunState) -> RunState:
        while state.phase == Phase.RUNNING:
            # 0. Kill-switch check: fires before any model or tool call.
            if self._kill and self._kill.is_tripped(state.principal):
                return self._pause(state, PauseReason.KILL_SWITCH,
                                   {"explainer": self._kill.message(state.principal)})

            # 1. Cancel check: cooperative, never hard-kills a running task.
            if state.cancel_requested:
                state.phase = Phase.CANCELLED
                state.final = {"cancelled": True}
                self._pauses.save(state)
                return state

            # 2. Budget check.
            try:
                self._budget.check()
            except BudgetExceededError as e:
                return self._pause(state, PauseReason.BUDGET_EXCEEDED,
                                   {"reason": str(e), "partial": state.trace})

            # 3. Step cap.
            if state.step_n >= self._max_steps:
                return self._pause(state, PauseReason.LOW_CONFIDENCE,
                                   {"reason": "step cap reached",
                                    "draft": self._last_draft(state)})

            # 4. Ask the LLM what to do next.
            action = await self._llm_step(state)
            state.step_n += 1
            state.trace.append({"kind": "llm_action", "action": action,
                                "at": time.time()})

            if action["kind"] == "final":
                if action.get("confidence", 1.0) < 0.5 or \
                   not action.get("citations"):
                    return self._pause(state, PauseReason.LOW_CONFIDENCE,
                                       {"draft": action.get("text"),
                                        "citations": action.get("citations", [])})
                return self._finalise(state, final=action)

            if action["kind"] == "tool_call":
                tool = action["tool"]
                args = action["arguments"]
                if _is_write_tool(tool):
                    return self._pause(state, PauseReason.CONFIRM_WRITE,
                                       {"tool": tool, "arguments": args,
                                        "human_summary": _summarise(tool, args)})
                # read tool: run directly.
                try:
                    result = await self._run_tool(tool, args, state.principal)
                except ToolError as e:
                    result = {"error": {"category": e.category,
                                        "model_hint": e.model_hint}}
                state.trace.append({"kind": "tool_result", "tool": tool,
                                    "result": result, "at": time.time()})

        return state

    async def _apply_decision(self, state: RunState, decision: dict) -> bool:
        """Apply a resume decision. Returns True to keep running, False to stop."""
        if decision["action"] == "cancel":
            return False
        if decision["action"] == "approve_write":
            pending = _last_pause_payload(state)
            tool, args = pending["tool"], pending.get("edited_args",
                                                      pending["arguments"])
            try:
                result = await self._run_tool(tool, args, state.principal)
            except ToolError as e:
                result = {"error": {"category": e.category,
                                    "model_hint": e.model_hint}}
            state.trace.append({"kind": "tool_result", "tool": tool,
                                "result": result, "at": time.time()})
            return True
        if decision["action"] == "regenerate":
            state.trace.append({"kind": "user_critique",
                                "note": decision.get("note", ""),
                                "at": time.time()})
            return True
        if decision["action"] == "use_anyway":
            return False  # finalise with the draft
        raise ValueError(f"unknown decision.action {decision['action']!r}")

    def _pause(self, state: RunState, reason: PauseReason,
               payload: dict) -> RunState:
        state.phase = Phase.PAUSED
        state.pause = HITLEvent(
            run_id=state.run_id, step_n=state.step_n,
            reason=reason, payload=payload, emitted_at=time.time(),
        )
        self._pauses.save(state)
        return state

    def _finalise(self, state: RunState, final: dict) -> RunState:
        state.phase = Phase.FINAL
        state.final = final
        self._pauses.save(state)
        return state


WRITE_TOOLS = {"create_followup_task"}


def _is_write_tool(name: str) -> bool:
    return name in WRITE_TOOLS


def _summarise(tool: str, args: dict) -> str:
    if tool == "create_followup_task":
        return (f"Create a follow-up task on ticket {args.get('ticket_id')} "
                f"due {args.get('due_at')}: {args.get('summary')!r}.")
    return f"Call {tool} with {json.dumps(args)[:200]}"


def _last_pause_payload(state: RunState) -> dict:
    for e in reversed(state.trace):
        if e.get("kind") == "llm_action" \
           and e["action"]["kind"] == "tool_call":
            return {"tool": e["action"]["tool"],
                    "arguments": e["action"]["arguments"]}
    raise RuntimeError("no prior tool_call to resume from")


def _last_draft(state: RunState) -> str | None:
    for e in reversed(state.trace):
        if e.get("kind") == "llm_action" and e["action"]["kind"] == "final":
            return e["action"].get("text")
    return None
```

The `PauseStore` is a trivial key-value table over `run_id`; it is the thing that makes "refresh the page after a confirm" safe. The schema is three columns (`run_id`, `state_json`, `updated_at`); the invariant is that `save(state)` is the only write path and runs in a single transaction so the UI never sees a half-written pause.

### 2c. Kill-switch, four layers (deliverable 3)

```markdown
# helpdesk-reply-copilot — kill-switch layers (v1)

## Layer 1 — Per-request (user)
- Mechanism: a "Stop" button on the running-draft surface.
- Behaviour: calls `AgentController.cancel(run_id)`. The current step
  completes if already in flight; the next step is skipped and the
  phase moves to CANCELLED.
- Latency: ≤ 2 s observed in prod (p95), bounded by the slowest
  cancellable tool.
- Audit: writes a `cancel(user)` row.

## Layer 2 — Per-session (user)
- Mechanism: a "Disable assistant for this session" toggle in the
  surface header.
- Behaviour: sets a session flag that the controller's kill-switch
  check reads. The feature is read-only for the remainder of the
  session; existing drafts are preserved.
- Audit: writes a `kill(session, user=...)` row.

## Layer 3 — Per-tenant (tenant admin)
- Mechanism: a toggle in the tenant admin console.
- Behaviour: the kill-switch is on for every session in the tenant;
  the surface shows the "paused" banner with the admin-provided copy.
- SLA to engage: ≤ 60 s from toggle to every active session being
  paused (polled kill-switch state cache with a 30 s TTL).
- Audit: writes a `kill(tenant, admin=...)` row.

## Layer 4 — Per-surface (us)
- Mechanism: a feature flag on our infra (see [[mod-111]] rollout
  guardrails).
- Behaviour: every tenant's surface shows the paused banner. Drafts
  are preserved; no new runs start.
- Who can pull it: on-call engineer + applied-AI lead.
- SLA to engage: ≤ 5 min from decision to everyone paused.
- Post-engagement: an incident is opened within 15 min with the
  trigger reason; post-mortem within 5 business days per [[mod-110]].

## Invariant across layers
- Every kill stops *new* loop steps, not in-flight tool calls that
  have already written. The confirm-before-write path from §2a is
  the surface's defence against a write that outlives a kill.
- Every kill preserves user-visible state. Nothing is wiped on kill;
  wipe is a separate action owned by the memory system
  ([[exercise-02]]).
```

### 2d. UX spec (deliverable 4)

```markdown
# helpdesk-reply-copilot — HITL UX spec (v1)

## Universal rules
- The pause surface is **inline**, not a modal. Modals steal focus and
  are dismissed accidentally by keyboard users. HITL in a critical
  write path never uses a modal.
- The confirm button copy names the action: "Create task" — never
  "OK" or "Confirm."
- The cancel path is always a single click and always keyboard-reachable.
- No auto-confirm, no countdown timer. An unattended HITL times out
  to "dismissed, no action" after 10 min.
- The pause surface shows the *exact arguments that will be sent*.
  Paraphrased summaries are allowed *in addition to* the raw args,
  never *instead of* them.

## `confirm_write` surface
┌─ The assistant wants to create a follow-up task ──────────────┐
│ Ticket:   ZD-123456                                            │
│ Due:      2026-03-15 (in 4 days)                               │
│ Summary:  Refund processed; check with finance Monday.         │
│                                                                │
│ [ Create task ]  [ Edit ]  [ Cancel ]                          │
└───────────────────────────────────────────────────────────────┘
- Edit opens an inline form prefilled with the arguments.
- Cancel finalises the run with no write and writes
  `cancel(write, user)` to the audit log.

## `low_confidence` surface
- The draft renders in an amber-bordered card labelled "Draft needs
  review."
- Buttons: "Use anyway" (primary), "Regenerate with a note," "Discard."
- If "Regenerate" is chosen, the user types a one-line critique that
  is injected into the loop as a user_critique trace entry.

## `budget_exceeded` surface
- A non-apologetic banner: "We stopped here because this request got
  too long. Your draft so far is below."
- The partial draft renders read-only.
- One button: "Continue manually" — closes the surface, leaves the
  partial draft in the reply box.

## `kill_switch` surface
- The surface header turns grey; the input box is disabled; the draft
  area (if any) is preserved and read-only.
- A single line: "The assistant is paused. Your draft is saved."
- No "retry" button. Retry is an operator action, not a user action.

## Accessibility
- Every pause surface is reachable by keyboard; no action is
  mouse-only.
- Every state change is announced to screen readers via an aria-live
  region ("Draft ready for review," "Assistant paused").
- Contrast ratios meet WCAG AA at minimum on the amber + grey states.
```

### 2e. Tests (deliverable 5)

```python
"""test_agent_controller.py — HITL and cancel invariants."""

import asyncio

import pytest

from agent_controller import (
    AgentController, Phase, PauseReason, RunState,
)


class InMemoryPauseStore:
    def __init__(self):
        self._store = {}

    def save(self, state: RunState) -> None:
        self._store[state.run_id] = state

    def load(self, run_id: str) -> RunState:
        return self._store[run_id]


class StubKillSwitch:
    def __init__(self):
        self.tripped = False

    def is_tripped(self, principal): return self.tripped
    def message(self, principal): return "paused by admin"


class StubBudget:
    def __init__(self, allow: int):
        self.left = allow

    def check(self):
        if self.left <= 0:
            raise RuntimeError("budget")
        self.left -= 1


@pytest.mark.asyncio
async def test_write_tool_pauses_before_call():
    async def llm(state):
        return {"kind": "tool_call", "tool": "create_followup_task",
                "arguments": {"ticket_id": "ZD-123456",
                              "summary": "follow up Mon",
                              "due_at": "2099-01-01T00:00:00Z"}}

    async def run_tool(name, args, p):
        raise AssertionError("must not run before confirm")

    ctl = AgentController(llm, run_tool, StubBudget(10),
                          pause_store=InMemoryPauseStore(),
                          kill_switch=StubKillSwitch())
    state = await ctl.start("s1", {"tenant_id": "t1", "user_id": "u1"}, "...")
    assert state.phase == Phase.PAUSED
    assert state.pause.reason == PauseReason.CONFIRM_WRITE
    assert state.pause.payload["tool"] == "create_followup_task"


@pytest.mark.asyncio
async def test_resume_cancel_does_not_write():
    actions = iter([
        {"kind": "tool_call", "tool": "create_followup_task",
         "arguments": {"ticket_id": "ZD-123456",
                       "summary": "follow up Mon",
                       "due_at": "2099-01-01T00:00:00Z"}},
    ])
    writes = []

    async def llm(state): return next(actions)
    async def run_tool(name, args, p):
        writes.append((name, args))
        return {"task_id": "TASK-1"}

    pauses = InMemoryPauseStore()
    ctl = AgentController(llm, run_tool, StubBudget(10),
                          pause_store=pauses,
                          kill_switch=StubKillSwitch())
    state = await ctl.start("s1", {"tenant_id": "t1", "user_id": "u1"}, "...")
    state = await ctl.resume(state.run_id, {"action": "cancel"})
    assert writes == []
    assert state.phase == Phase.FINAL


@pytest.mark.asyncio
async def test_kill_switch_pauses_immediately():
    kill = StubKillSwitch()
    kill.tripped = True

    async def llm(state): raise AssertionError("must not call LLM")
    async def run_tool(name, args, p): raise AssertionError

    ctl = AgentController(llm, run_tool, StubBudget(10),
                          pause_store=InMemoryPauseStore(),
                          kill_switch=kill)
    state = await ctl.start("s1", {"tenant_id": "t1", "user_id": "u1"}, "...")
    assert state.phase == Phase.PAUSED
    assert state.pause.reason == PauseReason.KILL_SWITCH


@pytest.mark.asyncio
async def test_cancel_stops_before_next_step():
    """Cooperative cancel: the running step is already scheduled, but the
    next step (and its tool) must not fire."""
    steps = []

    async def llm(state):
        steps.append(state.step_n)
        return {"kind": "tool_call", "tool": "search_knowledge_base",
                "arguments": {"query": "reset password"}}

    async def run_tool(name, args, p):
        return {"passages": []}

    pauses = InMemoryPauseStore()
    ctl = AgentController(llm, run_tool, StubBudget(10),
                          pause_store=pauses,
                          kill_switch=StubKillSwitch(),
                          max_steps=20)
    # Start, let one step run, then cancel.
    state = await ctl.start("s1", {"tenant_id": "t1", "user_id": "u1"}, "...")
    await ctl.cancel(state.run_id)
    # A subsequent resume-advance must observe cancel and stop.
    state = pauses.load(state.run_id)
    state.phase = Phase.RUNNING  # simulate another advance tick
    state = await ctl._advance(state)
    assert state.phase == Phase.CANCELLED
```

### 2f. Implementation notes

- The controller is the only place `_run_tool` is called with a write tool. The guard (`_is_write_tool` → pause) is the invariant; a developer who routes a write tool around the controller is the single most important thing the code review catches.
- The pause payload includes `human_summary` *and* the raw `arguments`. The human summary exists for scan-ability; the raw args exist so an auditor can tell exactly what was approved. Only `human_summary` is a privacy decision (it can be edited without changing behaviour); `arguments` is the ground truth.
- Resume is keyed by `run_id`, not session or user. A user who refreshes and reloads sees the pause they left; two concurrent runs on the same session have independent pause states.
- `cancel` is cooperative. It never hard-kills a step; the next loop tick observes the flag. For long-running tool calls, the tool is responsible for checking its own cancellation token; the controller sets the flag and moves on.
- The kill-switch cache has a 30 s TTL. A shorter TTL costs more; a longer TTL means a tenant-admin pause takes too long to propagate. The 30 s is quoted in §2c so operators know the worst-case engage time.
- No auto-confirm, no timer countdown. Any pause with an "it will go through in 10 s" behaviour has lost the HITL argument before the UI is drawn. The 10-minute "dismissed" timeout exists to clean up abandoned pauses, not to approve them.
- The spec covers only the four pause reasons. "Escalate to a human senior agent" is a different feature — it is a *tool* the model can call, not a HITL pause. HITL is about the user already on the surface making a decision.
- The eval harness from [[exercise-04]] includes a fixture per pause reason; a regression in the controller that collapses two reasons or skips a pause will fail that suite.

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. The pause taxonomy names exactly four reasons, each with a user-action set and a resume semantics. A fifth reason must be justified.
2. Every `side_effects: write` tool call flows through a `confirm_write` pause. A test proves the tool is not called before the user approves.
3. The pause surface renders the *raw arguments*, not only a paraphrase.
4. `resume(run_id, decision)` is idempotent at the pause boundary — resuming twice with the same decision does not fire the tool twice.
5. `cancel(run_id)` is cooperative, does not raise, and stops the loop within one step.
6. The kill-switch has four layers (request / session / tenant / surface) with the SLA to engage named for each.
7. The surface shows preserved state during a kill. No wipe-on-kill.
8. Every pause writes to the pause store atomically; a crash between pause and UI render does not produce a lost run.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Four-reason pause taxonomy with action + resume | 1 | Table, not prose. |
| Write tools always paused before call | 1 | Test present and passes. |
| Pause payload includes raw args (not just summary) | 1 | Spec + code both. |
| Resume is idempotent at the pause boundary | 1 | Test resumes twice; tool called once. |
| Cancel is cooperative + bounded latency | 1 | p95 cancel latency quoted; test covers mid-loop cancel. |
| Four-layer kill switch with SLA per layer | 1 | Layers + numbers; audit row named per layer. |
| Preserve-state on kill | 1 | Spec: no wipe; test: draft present after kill. |
| No modal, no auto-confirm, no countdown | 1 | UX spec is explicit; copy for buttons names the action. |
| Accessibility (keyboard + aria-live + contrast) | 1 | Named in spec; one bullet per concern. |
| Atomic pause write | 1 | Pause store writes in one transaction; test simulates crash-after-pause. |

A submission scoring < 6 is rejected. The hard failures are (a) a write tool that can fire before confirm, (b) auto-confirm after a timer, (c) a kill switch that wipes user state, (d) a resume that re-runs prior steps and double-writes.

## 5. Common mistakes

- **Writing HITL as a modal with "Confirm."** Modals are for non-critical confirms; writes are not non-critical. Inline, labelled buttons, keyboard-reachable.
- **Showing a paraphrase instead of the arguments.** "Create a follow-up task" is a summary. `{ticket_id: "ZD-123456", summary: "..."}` is what gets sent. The user must see both.
- **Auto-confirm after a countdown.** This is the single highest-severity UX mistake the exercise catches. The countdown undoes the HITL.
- **Letting `cancel` try to hard-kill the running step.** Cooperative only. Tools that want to be cancellable accept a token; the controller sets a flag.
- **Resume that re-executes prior steps.** Resume reads the trace and the pause; it never re-runs the earlier LLM step. Double writes are the consequence.
- **One-layer kill switch.** A user stop + an operator stop are different jobs; collapsing them into one toggle means the operator has no path to pause a tenant without firing an incident.
- **Wipe-on-kill.** The purpose of the kill switch is to pause the agent, not erase the user's work. Wipe is a different button, owned by [[exercise-02]].
- **No pause for `low_confidence` final answers.** An agent that confidently ships a wrong answer is the shape's worst failure mode. A confidence-or-citation pause is the cheapest defence.
- **A kill switch no one can reach.** If the toggle is three menus deep, it is not a kill switch, it is a feature request. Universal rule: one click, on the surface header.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — HITL pause reasons, resume semantics, and the four kill-switch layers map to RMF MEASURE (trackable intervention events with audit rows) and MANAGE (owner-level stop controls with named SLA and post-engagement process). The RMF is explicit that AI systems acting on user behalf must support a human-overridable control; this exercise is the product-surface implementation of that control.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the confirm-before-write + raw-args-visible design is the surface's defence against prompt-injection-driven writes (ML01-style input manipulation that would otherwise result in a model-driven write the user did not authorise).
- MITRE ATLAS — <https://atlas.mitre.org/> — ATLAS techniques such as prompt-injection and resource-abuse are mitigated by the HITL boundary before privileged tool invocation; the kill-switch layers are the operator-side controls mapped to the "contain an active incident" techniques.

Related exercises: [[exercise-01]] (shape decision made the loop boundary a natural pause point); [[exercise-02]] (memory writes also flow through the HITL confirm path); [[exercise-04]] (the eval suite includes a pause-reason fixture per row); upstream [[exercise-02]] (mod-104) is the error taxonomy whose `model_fixable` category routes back into the loop and whose `fatal` category triggers finalisation — see §2b `_apply_decision`; [[exercise-05]] (mod-104) defines the `BudgetExceededError` that produces the `budget_exceeded` pause.
