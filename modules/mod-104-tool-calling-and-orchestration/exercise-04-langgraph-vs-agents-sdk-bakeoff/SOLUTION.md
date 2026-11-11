### exercise-04: LangGraph vs. Agents SDK Bakeoff — Solution

## 1. Solution overview

The exercise asks the engineer to decide which orchestration framework — **LangGraph** or **OpenAI Agents SDK** — the helpdesk-reply-copilot will adopt for the multi-tool agent loop that uses the MCP server from [[exercise-03]]. The decision is not a framework survey. It is a scored bakeoff on **the specific work this surface does**: three tools, a confirm step before writes, a hard per-request budget ([[exercise-05]]), the error taxonomy from [[exercise-02]], and a two-month-in evals harness from [[exercise-03]] (mod-103) that already runs nightly.

The output is a decision memo backed by two parallel reference implementations of the same reply-drafting flow — one in each framework — scored against a fixed rubric. The exercise is passed by writing a memo that **picks one**, kills the other for this surface, and names the signal that would reverse the decision.

This solution provides:

- A **shortlist** of the two frameworks plus one honest alternative (a hand-rolled loop) and the pre-committed rubric they are scored against.
- A reference **LangGraph implementation** of the "search → draft → confirm → act" flow.
- A reference **Agents SDK implementation** of the same flow.
- A scored comparison table across the seven rubric dimensions.
- A `decision.md` memo with the ship/kill verdict, the migration-cost estimate, and the condition under which we'd revisit.
- A grading rubric for the instructor.

The lesson is that orchestration-framework choice is a trade between **control** (what you can wire up yourself) and **convention** (what you get for free). For a shipped surface with a strict budget and a battle-tested evals harness, the pivotal axes are observability granularity, the fit of the framework's state model to the error taxonomy, and the lock-in cost of migrating off. The memo defends a choice against those, not against Twitter hype.

> **Version-pinning note:** the LangGraph and OpenAI Agents SDK API surfaces evolve. Both reference implementations below name symbols from the versions pinned in each exercise's `requirements.txt`; when adopting this memo, re-verify the exact symbol names (graph builders, tool decorators, run/stream entrypoints) against the pinned release notes before copying code, and record the versions used in the decision memo so a future upgrade surfaces as a tracked migration rather than a silent breakage.

## 2. Worked answer or implementation

### 2a. The scoring rubric (pre-committed) (deliverable 1)

Score each framework 1–5 on each dimension. A ≤ 2 on any dimension with the ⚠ marker is a disqualifier.

| # | Dimension | What to check | ⚠ |
|---|---|---|---|
| 1 | **Control flow fit** | Can the framework express "search → draft → (user confirms) → act," including pausing mid-loop for a UI confirm step without re-running earlier steps? | ⚠ |
| 2 | **State model fit** | Does state live in a place the surface can serialize, snapshot, and replay for incident reproduction? | ⚠ |
| 3 | **Error-taxonomy fit** | Can the framework wire up the four-category routing from [[exercise-02]] without wrapping every node in a try/except? | |
| 4 | **Budget enforcement fit** | Can the framework interrupt mid-loop when the budget from [[exercise-05]] trips, and surface a user-visible "we stopped" message? | ⚠ |
| 5 | **Observability granularity** | Does every step emit a trace entry the readout from mod-103 can join on `request_id`? | |
| 6 | **Dev velocity** | How much code is required to add a fourth tool next sprint? To change the model-provider? | |
| 7 | **Lock-in / portability** | How much work to port off if we hit a blocker? What is vendor-coupled vs. framework-coupled? | |

The rubric is written *before* the implementations so the implementations are not fitted to flatter one framework.

### 2b. Shortlist considered

1. **LangGraph** (graph-centric; explicit node + edge DSL; checkpoint-able state).
2. **OpenAI Agents SDK** (agent-centric; tool decorators + handoffs; opinionated trace model).
3. A **hand-rolled loop** (no framework; raw provider SDK + our own state machine). The honest baseline — a "do nothing" option.

A single-vendor framework is a legitimate choice only if the surface commits to that vendor for the life of the feature. We serve the retriever-contract's multi-provider aspiration from [[exercise-01]] (mod-103), so "model-provider portability" is a hard dimension, not a nice-to-have.

### 2c. LangGraph reference implementation (deliverable 2)

```python
"""graph_langgraph.py — helpdesk reply-drafting flow as a LangGraph.

Nodes:
  classify_intent  → whether to retrieve, write, or just chat
  search           → calls search_knowledge_base over MCP
  draft            → model proposes a reply (and optional follow-up task)
  await_confirm    → surface shows the draft; loop pauses here
  act              → if the user confirms, calls create_followup_task over MCP
  finalize         → emits the final response envelope
"""

from __future__ import annotations

from typing import Literal, Optional

from pydantic import BaseModel
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver

from helpdesk_tools_client import HelpdeskToolsClient, ToolInvocationError
from tool_error import ToolError
from route_tool_error import SessionState, route


class GraphState(BaseModel):
    user_message: str
    principal: dict
    passages: list[dict] = []
    draft_reply: Optional[str] = None
    followup: Optional[dict] = None  # proposed task; may be None
    confirmed: Optional[bool] = None
    final: Optional[dict] = None
    errors: list[ToolError] = []
    session: dict = {}  # SessionState serialised (reconstructed on resume)


async def classify_intent(state: GraphState) -> GraphState:
    # ... calls the classifier from [[exercise-04]] (mod-103); may short-circuit to chat.
    return state

async def search(state: GraphState, *, mcp: HelpdeskToolsClient) -> GraphState:
    try:
        resp = await mcp.call("search_knowledge_base", {"query": state.user_message, "k": 5})
    except ToolInvocationError as e:
        state.errors.append(e.err)
    else:
        state.passages = resp["passages"]
    return state

async def draft(state: GraphState, *, llm) -> GraphState:
    # Single-turn structured output with the GroundedAnswer schema from
    # [[exercise-02]] (mod-103). Returns draft + optional follow-up proposal.
    out = await llm.complete_with_schema(_prompt(state), schema=DRAFT_SCHEMA)
    state.draft_reply = out["reply"]
    state.followup = out.get("followup")
    return state

async def await_confirm(state: GraphState) -> GraphState:
    # Terminal node for this turn. The graph CHECKPOINTS here; the
    # surface serialises `state` and waits on the user. Resume is
    # resume(graph, thread_id, {"confirmed": True/False}).
    return state

async def act(state: GraphState, *, mcp: HelpdeskToolsClient) -> GraphState:
    if not state.followup or not state.confirmed:
        return state
    try:
        task = await mcp.call("create_followup_task", state.followup)
    except ToolInvocationError as e:
        state.errors.append(e.err)
    else:
        state.followup = {**state.followup, "task_id": task["task_id"]}
    return state

async def finalize(state: GraphState) -> GraphState:
    state.final = {
        "reply": state.draft_reply,
        "task": state.followup,
        "errors": [e.model_dump() for e in state.errors],
    }
    return state


def build_graph(mcp: HelpdeskToolsClient, llm) -> StateGraph:
    g = StateGraph(GraphState)
    g.add_node("classify", classify_intent)
    g.add_node("search", search.__get__(None).__wrapped__)  # DI omitted for brevity
    g.add_node("draft", draft)
    g.add_node("await_confirm", await_confirm)
    g.add_node("act", act)
    g.add_node("finalize", finalize)

    def route_after_classify(s: GraphState) -> str:
        return "search" if s.user_message else END

    g.add_edge(START, "classify")
    g.add_conditional_edges("classify", route_after_classify, {"search": "search", END: END})
    g.add_edge("search", "draft")
    g.add_edge("draft", "await_confirm")

    def route_after_confirm(s: GraphState) -> str:
        if s.confirmed and s.followup:
            return "act"
        return "finalize"

    g.add_conditional_edges("await_confirm", route_after_confirm, {"act": "act", "finalize": "finalize"})
    g.add_edge("act", "finalize")
    g.add_edge("finalize", END)
    return g.compile(checkpointer=MemorySaver())
```

### 2d. Agents SDK reference implementation (deliverable 3)

```python
"""graph_agents_sdk.py — same flow as an OpenAI Agents SDK agent.

The Agents SDK models tools as decorated functions and handoffs as
named child agents. The confirm step is modelled as an explicit
`return_control_to_surface` tool the agent must call before acting.
"""

from __future__ import annotations

from dataclasses import dataclass, field
from typing import Optional

from agents import Agent, Runner, function_tool  # package name per docs
from helpdesk_tools_client import HelpdeskToolsClient, ToolInvocationError
from tool_error import ToolError


@dataclass
class RunContext:
    mcp: HelpdeskToolsClient
    principal: dict
    errors: list[ToolError] = field(default_factory=list)
    followup: Optional[dict] = None
    draft: Optional[str] = None
    awaiting_confirm: bool = False


@function_tool
async def search_knowledge_base(ctx: RunContext, query: str, k: int = 5) -> dict:
    try:
        return await ctx.mcp.call("search_knowledge_base", {"query": query, "k": k})
    except ToolInvocationError as e:
        ctx.errors.append(e.err)
        return {"passages": [], "error": e.err.model_dump()}


@function_tool
async def propose_followup_task(
    ctx: RunContext, ticket_id: str, summary: str, due_at: str, priority: str = "normal"
) -> dict:
    """Propose — not create — a follow-up task. The surface will show
    a confirm; the agent must stop the loop and call `return_control`."""
    ctx.followup = {
        "ticket_id": ticket_id,
        "summary": summary,
        "due_at": due_at,
        "priority": priority,
    }
    return {"proposed": True}


@function_tool
async def return_control(ctx: RunContext, draft_reply: str) -> dict:
    """The agent's exit — hands the draft back to the surface, which may
    (on user confirm) call the write tool outside the agent context."""
    ctx.draft = draft_reply
    ctx.awaiting_confirm = True
    return {"handed_back": True}


agent = Agent(
    name="helpdesk-reply-copilot",
    instructions=(
        "You help CS agents draft replies. First call search_knowledge_base. "
        "If a follow-up task would help, call propose_followup_task. "
        "Always finish by calling return_control(draft_reply=...)."
    ),
    tools=[search_knowledge_base, propose_followup_task, return_control],
)


async def run_once(ctx: RunContext, user_message: str) -> RunContext:
    await Runner.run(agent, user_message, context=ctx)
    return ctx


# The write step (`create_followup_task`) runs outside the Agents SDK
# agent, because the surface is responsible for the human-in-the-loop
# confirmation. After confirm, the surface calls the MCP client directly:
#
#   if ctx.confirmed and ctx.followup:
#       await mcp.call("create_followup_task", ctx.followup)
```

Note that in the Agents SDK implementation the confirmation break is modelled by having the agent *return control* via a tool call — the write does not happen inside the agent. In LangGraph, the break is a graph checkpoint. Both work; they differ in where the "stop" fact lives (tool contract vs. graph topology).

### 2e. Scored comparison (deliverable 4)

Scoring against the rubric in §2a. The hand-rolled baseline is scored for context.

| Dimension | LangGraph | Agents SDK | Hand-rolled | Notes |
|---|---|---|---|---|
| 1. Control flow fit ⚠ | **5** | 3 | 5 | LangGraph's checkpoint-on-node is a natural fit for "pause at confirm." The Agents SDK encodes it as a tool contract — equivalent, but if the model forgets to call `return_control`, the loop continues; we need a wrapper. |
| 2. State model fit ⚠ | **5** | 3 | 5 | LangGraph's `StateGraph[GraphState]` is explicit, snapshotable, and resumable. The Agents SDK carries state in a Python `RunContext` that is less structured; serialising mid-run is extra work. |
| 3. Error-taxonomy fit | 4 | 3 | 5 | In LangGraph we map `ToolError` → graph node branch. In the Agents SDK we catch in each `function_tool` and stash on `RunContext`. Hand-rolled wins on explicitness; LangGraph is close enough. |
| 4. Budget enforcement fit ⚠ | 4 | 3 | 5 | LangGraph exposes a step counter and `interrupt_before`/`interrupt_after`; we inject a budget check on each edge. Agents SDK requires wrapping `Runner.run` with `asyncio.wait_for` and intercepting tool calls. Hand-rolled is explicit. Both frameworks pass; Agents SDK is tighter. |
| 5. Observability granularity | 4 | **5** | 2 | Agents SDK ships a first-class trace model with run IDs that joins with the provider's traces natively. LangGraph has `get_state_history` and callbacks but we'd wire the join on `request_id` ourselves. Hand-rolled has only what we build. |
| 6. Dev velocity | 4 | **5** | 2 | Adding a tool is a one-decorator change in Agents SDK; in LangGraph it is a new node + edge + schema entry. Hand-rolled adds its own loop. |
| 7. Lock-in / portability | **4** | 2 | 5 | LangGraph is model-provider-agnostic. Agents SDK is tightly coupled to OpenAI's trace model and some provider-specific features; migrating to Anthropic/Bedrock is a rewrite of the agent definitions. Hand-rolled is maximally portable. |
| **Total** | **30** | 24 | 29 | — |

Both ⚠-marked dimensions (1, 2, 4) are ≥ 3 for all three. No framework is disqualified. The score deltas concentrate on state fit, confirm-step fit, and portability.

### 2f. `decision.md` — the memo (deliverable 5)

```markdown
# Orchestration Framework Decision — helpdesk-reply-copilot

## Decision

**Adopt LangGraph** for the agent loop that wraps the MCP client from
[[exercise-03]]. **Do not adopt the OpenAI Agents SDK** for this surface.
Keep the hand-rolled loop code around as a fallback escape hatch.

## Why LangGraph wins here

1. The confirm step is a product-visible pause ("show the draft, wait
   for the CS agent to click Send"), not a model-driven one. LangGraph's
   checkpoint-at-node is the exact shape of that pause; the graph serialises
   state, the surface renders, and resume is a one-line resume call. In
   the Agents SDK we had to invent `return_control` as a tool and trust the
   model to call it; that's a plausible path, but it is one prompt-regression
   away from a double-act.

2. The state is explicit (`GraphState` Pydantic model). An on-call reading
   the readout sees exactly which fields are populated at each step; the
   incident-repro flow is `resume(thread_id)` with the checkpointed state.

3. Model portability. The retriever contract from [[exercise-01]] (mod-103)
   commits us to a multi-provider posture. The Agents SDK ties us to a
   single vendor's trace model and some provider-specific features; LangGraph
   is agnostic.

## Why the Agents SDK loses here (specifically)

- The Agents SDK is a *better* framework if the primary need is
  automatic provider-trace joining and fast tool addition. Our primary
  need is a product-visible confirm step with resumable state. These
  are different problems.
- Its trace model's win on observability (dimension 5) does not offset
  the state-fit loss on dimension 2.

## Why we aren't going hand-rolled

- Hand-rolled's score (29) is a point behind LangGraph's (30).
- Dev velocity is the gap. Adding tool #4 and #5 next quarter is cheaper
  in LangGraph than in hand-rolled, and we know those are coming.
- LangGraph's checkpoint + resume is non-trivial code to re-implement.

## Migration-cost estimate

- From the current hand-rolled loop to LangGraph: ~5 days for one engineer.
  The nodes are 1:1 with the current functions; the state class is 90 %
  mechanical.

## What would reverse this decision

- If the provider we pick for the primary model ships a trace model that
  the surface's readout cannot consume natively, AND LangGraph's callbacks
  require more than ~1 day of integration to produce the same join, we
  revisit the Agents SDK.
- If LangGraph introduces a hard breaking change in the state checkpointer
  that invalidates our in-flight sessions on upgrade, we revisit the
  hand-rolled option (not the Agents SDK — the same provider-lock-in issue
  stands).

## Guardrails on the choice

- We pin LangGraph to a specific version in `requirements.txt`; upgrades
  are scheduled, not opportunistic.
- The MCP client layer ([[exercise-03]]) is the ONLY surface the framework
  talks to for tools. Switching frameworks does not force a re-wire of
  the tool layer.
```

### 2g. Implementation notes

- Both implementations use the **same MCP client** from [[exercise-03]]. The framework choice touches *orchestration*, not the tool layer. This is the whole value of the MCP boundary: the bakeoff is on 300 lines of state wiring, not on 1,500 lines of tool implementations.
- Both implementations share the same `ToolError` envelope ([[exercise-02]]) for errors. In LangGraph, errors become a graph branch; in the Agents SDK, errors become a stashed field on `RunContext`.
- Neither framework is wired to the per-request budget in this file; that is [[exercise-05]]'s responsibility and adds about 10 lines in either implementation.
- The "await_confirm" / "return_control" split is the single most instructive diff between the two frameworks. Reviewers should linger there.
- The learner's submission should include working code, not slideware. The scoring is defensible only if each row has an artefact behind it; otherwise it collapses to taste.
- If the learner chooses the Agents SDK instead, that is a valid outcome — the exercise is about the memo, not the winner. The disqualifying answer is "both are fine, we'll pick later." A bakeoff is for deciding.

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. The rubric (seven dimensions, three ⚠-marked disqualifiers) is written **before** either implementation.
2. Both implementations exist, both compile, and both run the same end-to-end integration test (search → draft → confirm → act).
3. Both use the MCP client from [[exercise-03]] — no framework-specific tool reimplementation.
4. Both propagate `ToolError` ([[exercise-02]]) into the orchestration state without a bare `except`.
5. The score table has a non-zero entry in every row and the totals are the arithmetic sum.
6. The decision memo names *why the losing framework loses specifically for this surface* — not a generic "it's less flexible."
7. The memo names a numeric **migration cost** and a **reversal condition**. A memo without one of those is a review, not a decision.
8. The chosen framework is pinned to a specific version in `requirements.txt`.
9. The memo says what MCP client / tool-contract work is reusable **if the framework changes** — LangGraph → Agents SDK should not force a rewrite of [[exercise-01]] or [[exercise-03]].
10. The honest "hand-rolled baseline" is present. A bakeoff with only the two named frameworks is incomplete; the baseline anchors the scoring.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Pre-committed scoring rubric | 1 | Written before implementations; disqualifiers marked. |
| Both implementations run end-to-end | 2 | Same input, same output assertions; one green test per framework. |
| MCP client reused in both | 1 | No per-framework tool reimplementation. |
| Hand-rolled baseline scored | 1 | Honest "no framework" anchor present. |
| Score table filled with artefact-backed rationale | 2 | Every cell has a one-line reason, tied to code. |
| Decision memo picks one | 1 | "Both are fine" is a fail. |
| Memo names reversal conditions | 1 | A numeric trigger, not vibes. |
| Migration cost estimated | 1 | Days, not "small" / "medium." |

A submission scoring < 6 is rejected. The hard failures are (a) a table without working code, (b) a decision memo that does not pick one, (c) implementations that reimplement the tool layer instead of reusing MCP.

## 5. Common mistakes

- **No pre-committed rubric.** Writing the rubric after implementation produces a rationalisation, not a decision.
- **Scoring on taste, not artefact.** "LangGraph feels cleaner" is not a cell. "Adding tool #4 required 1 new node + 1 edge + 1 schema entry in LangGraph vs. 1 decorator in Agents SDK" is a cell.
- **Comparing frameworks on features they advertise rather than on what the surface needs.** A streaming-response feature is irrelevant if the surface's p95 is dominated by retrieval latency.
- **Omitting the hand-rolled baseline.** Without it, two mediocre frameworks look great. The baseline sometimes wins; that is a valid outcome.
- **Reimplementing the tool layer per framework.** If the bakeoff is 1,500 lines per side, you're really comparing tool implementations. Reuse the MCP client.
- **Decision memo without a reversal condition.** Teams that cannot name what would reverse a decision cannot un-adopt a framework when it turns bad.
- **Choosing a framework for its ecosystem of plugins rather than for the surface's work.** Plugins are nice; *the four load-bearing dimensions here are confirm-step fit, state, budget, and portability*. Plugins do not help on any of them.
- **Picking the Agents SDK while also committing to multi-provider.** The two decisions conflict. Pick one.
- **Pin-later.** An unpinned framework is a time bomb; the version goes in `requirements.txt` the same PR as the implementation.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the confirm-step design is the surface answer to ML06 (output integrity): a resumable, inspectable orchestration lets a human catch a hallucinated write before it lands.
- MITRE ATLAS — <https://atlas.mitre.org/> — serialisable state is what makes incident reproduction tractable, blunting the "Discovery" techniques that benefit from vendor logs no one on the team can pull.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the pre-committed rubric is a MAP artefact (named decision criteria), and the migration-cost + reversal condition are MANAGE artefacts (controls on the decision itself).

Related exercises: [[exercise-01]] (mod-104) defines the tools both frameworks wrap; [[exercise-02]] (mod-104) defines the error envelope both frameworks propagate; [[exercise-03]] (mod-104) is the MCP client both frameworks share; [[exercise-05]] (mod-104) is the budget layer both frameworks must honour; the classifier and rewriter in [[exercise-04]] (mod-103) live in the `classify` node of the LangGraph implementation and in a tool in the Agents SDK one.
