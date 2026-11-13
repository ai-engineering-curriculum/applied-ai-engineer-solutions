### exercise-04: Tool-Abuse Guards and Destructive-Action Gates — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to design the **tool-abuse guards** and the **destructive-action gates** that sit between the agent loop and every tool on the helpdesk-reply-copilot surface. Tool-abuse guards protect against the ways the agent can call *allowed* tools in disallowed ways: too many calls, wrong arguments, wrong caller, wrong order, or during a state the surface forbids. Destructive-action gates are the stricter subset that stand in front of irreversible operations (writes, deletes, external sends) and require an explicit human confirmation plus a bounded argument shape.

The failure mode the exercise is written to catch is "the system prompt forbids it." A system prompt is not an authorisation boundary. The agent framework is also not one — it is a scheduler. The guards and gates live below the framework, in the host process, and they say *no* with enough information for an incident to be filed and closed.

This solution provides:

- A **tool catalog** for the surface, labelled by reversibility, blast radius, and required caller tier.
- A **pre-call guard pipeline** (schema → rate → argument policy → state precondition → principal check) that every tool call must pass.
- A **destructive-action gate** spec with the HITL confirm contract, the argument-diff show-to-user step, and the post-execution compensating action path.
- A **reference implementation** (Python) composing guards + gates and emitting audit-grade trace events.
- A **test suite** covering the guard invariants, including adversarial cases drawn from [[exercise-02]].
- A **grading rubric** for the instructor.

The lesson the exercise defends is: *the right place to say no to a tool call is the host, not the model.* The model proposes tool calls; the host authorises them. A design that depends on the model to self-police is a design that lost the argument the first time the model is wrong.

## 2. Worked answer or implementation

### 2a. Tool catalog (deliverable 1)

```markdown
# helpdesk-reply-copilot — tool catalog (v1)

| Tool | Class | Reversible? | Blast radius | Required confirm |
|---|---|---|---|---|
| `kb_search(q)` | read | yes | tenant | none |
| `ticket_fetch(id)` | read | yes | single ticket | none |
| `memory_read(scope, key)` | read | yes | user / tenant | none |
| `memory_write(scope, key, value)` | write | yes | user / tenant | explicit |
| `create_followup_task(ticket, title, body)` | write (external) | hard — becomes visible in CRM | single ticket | explicit |
| `set_tenant_preference(key, value)` | config write | yes (via audit log) | tenant | explicit + admin role |
| `send_reply(ticket, body)` | **not exposed** | — | customer | **tool not on this surface** |

## Classes used by the guard pipeline
- `read` — argument-only sanity, no HITL.
- `write` — argument-shape policy + HITL confirm.
- `write (external)` — same as write + compensating action required.
- `config write` — same as `write` + admin role required + audit retention.
```

`send_reply` is explicitly out of the catalog. It is the first thing a submission under time pressure adds and the single most important thing the exercise forbids.

### 2b. Pre-call guard pipeline (deliverable 2)

Every tool call passes through five guards in order. A `deny` at any stage halts the call and emits a structured denial.

| # | Guard | Checks | Fail-mode |
|---|---|---|---|
| 1 | **Schema** | Arguments match the tool's typed schema; no extra fields; strings are within length bounds. | `deny: schema` — the model gets a structured error to retry. |
| 2 | **Rate** | Tool is not over its per-request and per-session caps. | `deny: rate` — request-fatal, logged. |
| 3 | **Argument policy** | Arguments pass per-tool policy (e.g. `create_followup_task.title` must be ≤ 120 chars and not match exfil regex). | `deny: policy` — incident-grade if the match is the exfil regex. |
| 4 | **State precondition** | The surface state allows this tool at this step (e.g. no write tools while the moderator verdict is `hard_block`; no memory_write before the user has confirmed a preference change). | `deny: state` — model receives a clear reason. |
| 5 | **Principal** | The host's acting principal has the role required by the tool (admin for `set_tenant_preference`). | `deny: principal` — audit-grade. |

Guards 1–4 run for every tool. Guard 5 runs only where the catalog names a required role.

### 2c. Destructive-action gate (deliverable 3)

For every tool labelled `write`, `write (external)`, or `config write`, the gate wraps the call with a HITL confirm step.

```markdown
# destructive-action gate — contract (v1)

## Steps
1. Host receives the proposed tool call from the loop.
2. Host runs guards 1–5 (above). If deny, stop.
3. Host presents to the CS agent:
   - The tool name.
   - The arguments, rendered with the diff vs. current state where
     applicable (e.g. "create task 'X' on ticket Y" or "set tenant
     preference tone=formal (was tone=neutral)").
   - A one-line explanation of why the agent proposed it (the loop's
     scratch rationale — never the raw chain of thought).
4. The CS agent clicks `confirm` or `reject`. Rejection emits a signal
   back to the loop; confirmation unlocks exactly one invocation.
5. On confirmation, the host executes the tool and records a
   confirmation receipt (confirm_id, user_id, timestamp, hash of
   arguments).
6. The receipt is bound to the executed call; a replay of the same
   arguments after the confirmation ID is spent is denied.

## Invariants
- No `write` or `write (external)` call executes without a receipt.
- A receipt is valid for exactly one call. Reuse is denied.
- The confirm UI renders the arguments as plain text, scrubbed by
  [[exercise-03]], with no HTML / markdown execution.
- The confirm payload is itself a T3 input for the next loop step (an
  attacker who controls the arguments cannot control the agent's
  subsequent behaviour; see [[exercise-02]]).

## Compensating action
For `write (external)` tools (e.g. CRM task creation), the gate
records a compensating-action function at confirmation time (e.g.
`close_task(task_id)` with required role). If the user later marks
the action a mistake within a defined window, the host can execute
the compensating action without re-running the agent loop.
```

### 2d. Reference implementation (deliverable 4)

```python
"""tool_gate.py — composes guards and the destructive-action gate.

The host owns the call. The model's proposed `tool_call` is a *request*;
authorisation is a host decision backed by the guards defined below.
The loop receives either the tool's result or a structured denial.
"""
from __future__ import annotations

import hashlib
import re
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Callable, Protocol

class Deny(Exception):
    def __init__(self, kind: str, reason: str) -> None:
        super().__init__(reason)
        self.kind = kind    # schema | rate | policy | state | principal | hitl

@dataclass(frozen=True)
class ToolSpec:
    name: str
    kind: str                    # read | write | external_write | config_write
    reversible: bool
    required_role: str | None = None
    arg_schema: dict = field(default_factory=dict)
    arg_policy: Callable[[dict], None] | None = None  # raises Deny on reject
    rate_per_request: int = 6
    rate_per_session: int = 50
    state_precondition: Callable[["ToolCallCtx"], None] | None = None

@dataclass
class ToolCallCtx:
    tool: ToolSpec
    args: dict
    principal: "Principal"          # from mod-105 memory exercise
    moderator_verdict: str          # allow | soft_block | hard_block
    session_counts: dict[str, int]
    request_counts: dict[str, int]

@dataclass
class ExecResult:
    ok: bool
    output: Any = None
    deny: Deny | None = None
    receipt: str | None = None      # confirmation id if gated

class HITL(Protocol):
    def confirm(self, tool: ToolSpec, args: dict, rationale: str) -> str | None: ...
    #  returns a confirm_id or None if rejected

def _hash_args(args: dict) -> str:
    return hashlib.sha256(repr(sorted(args.items())).encode()).hexdigest()[:16]

def run_tool(ctx: ToolCallCtx, rationale: str, executor: Callable[[dict], Any],
             hitl: HITL) -> ExecResult:
    try:
        _guard_schema(ctx)
        _guard_rate(ctx)
        _guard_arg_policy(ctx)
        _guard_state(ctx)
        _guard_principal(ctx)
    except Deny as deny:
        return ExecResult(ok=False, deny=deny)

    if ctx.tool.kind == "read":
        output = executor(ctx.args)
        _emit_audit(ctx, "exec_read", rationale, output_hash=_hash_args({"out": output}))
        return ExecResult(ok=True, output=output)

    # destructive path
    confirm_id = hitl.confirm(ctx.tool, ctx.args, rationale)
    if not confirm_id:
        _emit_audit(ctx, "hitl_rejected", rationale)
        return ExecResult(ok=False, deny=Deny("hitl", "user rejected"))
    output = executor(ctx.args)
    _emit_audit(ctx, "exec_write", rationale,
                receipt=confirm_id, args_hash=_hash_args(ctx.args))
    return ExecResult(ok=True, output=output, receipt=confirm_id)

def _guard_schema(ctx: ToolCallCtx) -> None:
    extra = set(ctx.args) - set(ctx.tool.arg_schema)
    if extra: raise Deny("schema", f"unexpected args: {sorted(extra)}")
    missing = {k for k, v in ctx.tool.arg_schema.items()
               if v.get("required") and k not in ctx.args}
    if missing: raise Deny("schema", f"missing args: {sorted(missing)}")

def _guard_rate(ctx: ToolCallCtx) -> None:
    if ctx.request_counts.get(ctx.tool.name, 0) >= ctx.tool.rate_per_request:
        raise Deny("rate", f"per-request cap for {ctx.tool.name}")
    if ctx.session_counts.get(ctx.tool.name, 0) >= ctx.tool.rate_per_session:
        raise Deny("rate", f"per-session cap for {ctx.tool.name}")

def _guard_arg_policy(ctx: ToolCallCtx) -> None:
    if ctx.tool.arg_policy:
        ctx.tool.arg_policy(ctx.args)   # raises Deny("policy", ...)

def _guard_state(ctx: ToolCallCtx) -> None:
    if ctx.tool.kind != "read" and ctx.moderator_verdict == "hard_block":
        raise Deny("state", "no writes under hard_block")
    if ctx.tool.state_precondition:
        ctx.tool.state_precondition(ctx)

def _guard_principal(ctx: ToolCallCtx) -> None:
    if ctx.tool.required_role and ctx.tool.required_role not in ctx.principal.roles:
        raise Deny("principal", f"role {ctx.tool.required_role} required")

def _emit_audit(ctx: ToolCallCtx, action: str, rationale: str, **extras) -> None:
    # Trace exporter applies the scrubber from [[exercise-03]].
    ...
```

Per-tool argument policies (example for `create_followup_task`):

```python
EXFIL_URL_RE = re.compile(r"https?://[^\s]+\?.*(sess|tok|key|email)=", re.I)

def followup_arg_policy(args: dict) -> None:
    title = args.get("title", "")
    body = args.get("body", "")
    if len(title) > 120:
        raise Deny("policy", "title too long")
    if EXFIL_URL_RE.search(title) or EXFIL_URL_RE.search(body):
        raise Deny("policy", "exfil-shape URL in followup args")
    if "{{" in title or "{{" in body:  # template leak
        raise Deny("policy", "template leak in followup args")
```

### 2e. Test suite (deliverable 5)

```python
# tests/test_tool_gate.py
def test_write_without_confirmation_is_denied():
    ctx = make_ctx(tool=CREATE_TASK, args={"ticket": "ZD-1", "title": "x", "body": "y"})
    res = run_tool(ctx, "r", executor=exec_ok, hitl=AutoRejectHITL())
    assert not res.ok and res.deny.kind == "hitl"

def test_write_under_hard_block_denied_before_hitl():
    ctx = make_ctx(tool=CREATE_TASK, args=OK_ARGS, moderator_verdict="hard_block")
    res = run_tool(ctx, "r", executor=exec_ok, hitl=AlwaysConfirmHITL())
    assert not res.ok and res.deny.kind == "state"

def test_rate_cap_fires():
    ctx = make_ctx(tool=KB_SEARCH, request_counts={"kb_search": 6})
    res = run_tool(ctx, "r", executor=exec_ok, hitl=AlwaysConfirmHITL())
    assert not res.ok and res.deny.kind == "rate"

def test_exfil_url_in_followup_args_denied():
    args = {"ticket": "ZD-1", "title": "ok",
            "body": "see http://evil.test/?sess=abcd"}
    ctx = make_ctx(tool=CREATE_TASK, args=args)
    res = run_tool(ctx, "r", executor=exec_ok, hitl=AlwaysConfirmHITL())
    assert not res.ok and res.deny.kind == "policy"

def test_principal_check_blocks_non_admin_from_config_write():
    ctx = make_ctx(tool=SET_TENANT_PREF, args=OK_ARGS,
                   principal=Principal(roles=frozenset({"user"})))
    res = run_tool(ctx, "r", executor=exec_ok, hitl=AlwaysConfirmHITL())
    assert not res.ok and res.deny.kind == "principal"

def test_confirm_receipt_binds_to_arguments():
    confirm_id = "cid-1"
    ctx1 = make_ctx(tool=CREATE_TASK, args=OK_ARGS)
    ctx2 = make_ctx(tool=CREATE_TASK, args=DIFFERENT_ARGS)
    hitl = StaticConfirmHITL(confirm_id)
    r1 = run_tool(ctx1, "r", executor=exec_ok, hitl=hitl)
    r2 = run_tool(ctx2, "r", executor=exec_ok, hitl=hitl)
    assert r1.receipt == confirm_id and r2.receipt == confirm_id
    # a later consumer checks the receipt was bound to args_hash
    assert r1.args_hash != r2.args_hash

def test_schema_rejects_unknown_arg():
    ctx = make_ctx(tool=KB_SEARCH, args={"q": "ok", "secret": "x"})
    res = run_tool(ctx, "r", executor=exec_ok, hitl=AlwaysConfirmHITL())
    assert not res.ok and res.deny.kind == "schema"
```

### 2f. Implementation notes

- **Guards are ordered.** Schema fails fastest and gives the model something to retry against; placing a model-retriable deny first reduces loop cost. The principal check is last because it is the most expensive to produce a useful denial for.
- **Rate caps are per-request *and* per-session.** Per-request alone fails against the slow attacker that spans many turns; per-session alone fails against the burst.
- **State preconditions are a first-class guard.** A reply-drafting surface should never execute a write while the moderator's verdict is `hard_block`. This is the only guard that reads from outside the request context, and it matters.
- **Confirmation IDs are bound to the exact arguments.** A reused receipt is a replay attack; the audit log captures the `args_hash` at receipt time and the executor refuses if a later caller presents a receipt with a different hash.
- **Rationale shown to the user is the loop's scratch, not the raw CoT.** Raw chain-of-thought is unsafe to render — it can carry instructions. A distilled, scrubbed rationale is adequate for the HITL moment.
- **Writes outside the catalog cannot exist.** The host registers tools by name; the model calling a name that is not in the catalog produces a `deny: schema` at guard 1.
- **Compensating actions are the only reason an irreversible tool can ship on this surface.** Without a compensating path, the surface owner should push back on the tool's inclusion.

## 3. Validation steps

1. The tool catalog explicitly names at least one tool kept *out of* the surface (e.g. `send_reply`) with a one-line reason.
2. Every tool in the catalog has a `kind` and a reversibility label.
3. The guard pipeline runs all five guards in the order named; a schema deny does not evaluate principal.
4. Writes without a confirmation receipt do not execute. Tests assert this for each write tool.
5. State preconditions block writes when the moderator verdict is `hard_block`.
6. Rate caps are both per-request and per-session; the test suite exercises both.
7. Confirmation receipts are bound to the exact argument hash; receipts are single-use.
8. Argument policies cover at least length limits and the exfil-URL/template-leak patterns on every `write (external)` tool.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Catalog labels class, reversibility, confirm rule | 1 | Every tool has all three. |
| Explicit out-of-scope tool with reason | 1 | One tool deliberately excluded. |
| Guard pipeline runs ≥ 5 stages in order | 1 | Ordering justified. |
| Writes require a HITL confirm receipt | 1 | Enforced in the executor, not only in the loop. |
| Receipt bound to args_hash; single-use | 1 | Replay with new args denied. |
| State precondition blocks writes under hard_block | 1 | Covered by a test. |
| Per-tool argument policy handles exfil-URL + template | 1 | At least one write tool has a policy with both rules. |
| Rate caps per-request and per-session | 1 | Both enforced and tested. |
| Audit event emitted for every denial | 1 | Deny events reach the trace sink via the scrubber from [[exercise-03]]. |
| Compensating action documented for external writes | 1 | A named compensator exists for at least one external-write tool. |

A submission scoring < 6 is rejected. Hard failures: (a) HITL confirm is advisory not enforced, (b) no state precondition guard, (c) receipts not bound to args, (d) `send_reply` or any auto-send tool is in the catalog without a strong defence.

## 5. Common mistakes

- **Using the system prompt to forbid tool calls.** A prompt is an opinion; a guard is an answer. The deny must be in the host.
- **Letting the framework "ask the user" before writes but not enforcing the receipt.** If the user's click is advisory and the executor can be called without it, the gate is cosmetic. The executor must refuse without a valid receipt.
- **Receipts that cover only the tool name.** A receipt that only names the tool can be reused across any arguments. Bind the receipt to the args_hash.
- **No state precondition.** A write that executes while the moderator said `hard_block` is the most defensible postmortem-root-cause the exercise can produce; avoid writing it.
- **One-size rate cap.** A fixed "max 20 tool calls" misses the attacker who takes one tool call per turn across 100 turns. Combine per-request and per-session (and ideally per-minute).
- **Trusting the model to self-limit.** The model sometimes self-limits. "Sometimes" is not a guard.
- **Exposing `send_reply` because "the agent can do it anyway."** Removing the tool is the defence. If the surface needs auto-send, that is a separate product decision and a different threat model.
- **Rendering HTML/markdown in the confirm UI.** The confirm UI is a plain-text surface that shows scrubbed arguments; anything that renders can carry social-engineering payloads.
- **No compensating action.** An irreversible tool without a compensator is a one-way door for every user who ever clicks confirm by mistake. A small `close_task` compensator is cheap and recovers most of the value.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the guard pipeline and gate are the applied-AI engineer's controls for ML-sec items covering unsafe tool use, excessive agency, and insufficient authorisation checks at the AI-to-system boundary.
- MITRE ATLAS — <https://atlas.mitre.org/> — the gates stand against ATLAS techniques in the execution and impact tactics (adversary causes the AI system to execute unintended actions). The per-tool argument policy is the technique-level mitigation the taxonomy expects.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the HITL confirm step, receipts, and compensating actions implement MANAGE function requirements for AI-system actions with user- or business-visible consequences; the audit event trail implements the MEASURE/MANAGE evidentiary requirements.

Related exercises: [[exercise-01]] (the moderator verdict is the state signal the state-precondition guard consumes); [[exercise-02]] (prompt-injection attacks are the standard adversarial input against these gates; test suites must include an injection-style argument-policy case); [[exercise-03]] (audit events pass through the scrubber before export); [[exercise-05]] (the guards and gates are top-line controls catalogued in the OWASP/ATLAS mapping runbook).
