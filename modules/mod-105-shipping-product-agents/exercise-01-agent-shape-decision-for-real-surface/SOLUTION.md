### exercise-01: Agent Shape Decision for a Real Product Surface — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to decide the **agent shape** that ships on the helpdesk-reply-copilot surface. "Agent shape" here is the question *before* framework choice from [[exercise-04]] (mod-104): given this surface, this user, and this blast radius, is the feature a single-turn tool-calling LLM, a bounded multi-step agent, a long-horizon planner, or a chain of narrowly scoped agents? The wrong shape — picking a free-roaming planner for a 15-second reply drafter, or a single-turn call for a research task that needs 20 steps — is the most expensive mistake on the surface, because once the shape is set the entire control-flow, memory, HITL, and eval stack is built around it.

The output is a **decision memo** backed by a one-page capability-and-blast-radius diagnosis for the surface, a scored evaluation across four candidate shapes, and the pre-committed signals that would reverse the choice. The exercise is passed by writing a memo that **picks one shape**, defends it against the three alternatives, and names the operating envelope (steps, time, tool classes, and user patience budget) the shape must fit inside.

This solution provides:

- A **capability-and-blast-radius diagnosis** for the helpdesk-reply-copilot surface — the one page that pins what the agent must do and what happens when it does the wrong thing.
- A **four-shape shortlist** (single-turn, bounded tool-loop, planner-with-executor, chain-of-narrow-agents) and a **scored rubric** across seven dimensions.
- A reference **decision memo** picking the bounded tool-loop for this surface, with the migration triggers that would promote or demote the shape.
- A **grading rubric** for the instructor.

The lesson the exercise defends is: *shape is a budget decision, not a technology decision*. Every extra degree of freedom the shape grants the model has to be paid for in HITL cost, eval cost, incident-response cost, and token cost. The memo defends the tightest shape that can still do the job, and names the next shape up and the signal that would trigger a move.

## 2. Worked answer or implementation

### 2a. Capability-and-blast-radius diagnosis (deliverable 1)

Before any shape is considered, the surface is diagnosed on one page. The instructor should be able to tell from this page alone whether the surface is a reply drafter, a research tool, or an autonomous worker.

```markdown
# helpdesk-reply-copilot — surface diagnosis (v1)

## What the surface is
An in-product assistant that drafts a reply to a customer support ticket.
The CS agent is the primary user; the surface is embedded inside the
existing Zendesk-style reply box.

## The user's job in the moment
Triage a ticket, read a draft reply, edit it, send it. Target wall-clock
for the draft: < 15 s from "open ticket" to "reply visible in box."

## What actions must the agent be able to take?
1. Retrieve relevant passages from the tenant's knowledge base.
2. Fetch ticket metadata and the last N comments.
3. Draft a reply citing those passages.
4. Optionally create a follow-up task (write) — explicit user confirm.

The surface *does not* send the reply; the CS agent clicks send. Writes
are limited to the `create_followup_task` tool and never fire without
an explicit confirm. See [[exercise-03]].

## Blast radius if the agent is wrong
- **Worst, high-likelihood:** drafts a wrong answer, agent edits / rejects.
  Cost: ~30 s of wasted human time; no customer-visible harm.
- **Worst, low-likelihood:** drafts confidently-wrong answer, agent misses
  it on review, customer sees it. Reputational + incident ticket.
- **Bounded by design:** no auto-send, no refund issuance, no billing
  action, no DM to the customer.

## What the surface must *never* do
- Send a reply without an agent click.
- Create a follow-up task without the confirm step.
- Issue refunds, cancel subscriptions, change passwords (no tool).
- Call a tool after the budget trips (see [[exercise-05]] mod-104).

## Non-functional envelope
- p50 draft latency ≤ 4 s; p95 ≤ 10 s.
- Per-request token budget ≤ 12 k input + 2 k output.
- Per-request tool-call budget ≤ 6 calls.
- Per-tenant cost ceiling tracked daily; see mod-109.
```

The diagnosis is the contract the shape has to fit inside. Skipping it is the single most common failure mode on this exercise; shapes get picked on vibes, then the eval suite from [[exercise-04]] discovers the shape cannot hit p95.

### 2b. The four-shape shortlist (deliverable 2)

| # | Shape | One-line | Fits when… |
|---|---|---|---|
| A | **Single-turn with tools** | One LLM call, optionally emits tool calls that the host executes in parallel, then one more LLM call to compose the final reply. No multi-step reasoning in the model. | The work is bounded and the model does not need to react to one tool's result before deciding to call another. |
| B | **Bounded tool-loop agent** | The model emits tool calls in a loop; each result feeds back; the loop terminates on a hard step budget *or* on a `final_answer` signal. No sub-plans, no delegation. | The work needs at most ~5–10 steps, the model benefits from reacting to retrieval results, and every tool call is cheap to validate. |
| C | **Planner + executor** | A planner LLM produces an explicit plan (an ordered list of steps with inputs); an executor runs the plan; a critic checks the output against the plan. | The work is long (20+ steps), reproducibility matters, and a plan is auditable on its own. |
| D | **Chain of narrow agents** | A router routes the request to one of several narrow, single-purpose agents, each with its own tool subset and system prompt. | The work naturally partitions and the cost of a wrong route is low. |

A fifth option — "**no agent, extend the prompt**" — is always on the table but is answered by [[exercise-01]] (mod-101): if the feature does not need a tool, this module is the wrong module. The decision memo must confirm that at least one tool call is required; otherwise the feature is prompt-engineering, not agent work.

### 2c. The pre-committed rubric (deliverable 3)

Score each shape 1–5 on each dimension; a ≤ 2 on any ⚠ dimension is a disqualifier.

| # | Dimension | What to check | ⚠ |
|---|---|---|---|
| 1 | **Capability fit** | Can the shape actually do the job as described in the diagnosis? | ⚠ |
| 2 | **Latency fit** | Can p95 beat the envelope? Extra model turns are the single largest latency driver. | ⚠ |
| 3 | **Blast-radius fit** | Does the shape give the model only as many degrees of freedom as the blast radius tolerates? | ⚠ |
| 4 | **HITL fit** | Can the shape insert the confirm step from [[exercise-03]] at the right point without re-running earlier steps? | ⚠ |
| 5 | **Eval fit** | Can the eval suite from [[exercise-04]] score the shape's behaviour step-by-step and end-to-end? | |
| 6 | **Memory fit** | Does the shape have a natural place for the memory system from [[exercise-02]] to attach? | |
| 7 | **Migration cost** | How much of the surface has to change if we move to the next shape up? | |

The rubric is written *before* a shape is picked so shapes are not fitted to flatter the designer's preference.

### 2d. Scored comparison for the helpdesk-reply-copilot surface

| Dimension | A: Single-turn | B: Bounded loop | C: Planner+executor | D: Chain of narrow |
|---|---|---|---|---|
| Capability fit ⚠ | 3 — can retrieve-then-draft, but cannot react to "no passages found" with a rewrite | 5 — can rewrite query, re-search, then draft | 5 | 4 |
| Latency fit ⚠ | 5 — two sequential LLM calls | 4 — up to ~6 LLM calls worst case | 2 — planner + executor + critic ≈ 3–5× turns, p95 blows the envelope | 3 — router + chosen agent, adds a turn |
| Blast-radius fit ⚠ | 5 | 4 | 2 — more degrees of freedom than the surface needs, with no proportional benefit | 3 |
| HITL fit ⚠ | 3 — confirm step is awkward because tool-calls are parallel in one turn | 5 — pause the loop before any write tool | 4 | 4 |
| Eval fit | 4 | 5 — one trace per step, maps naturally to the eval trajectories | 5 | 4 |
| Memory fit | 3 | 4 — scratchpad per-request + user-pinned preferences | 5 | 4 |
| Migration cost | — | +1 model turn from A | +2 roles from B | +1 router from B |
| **Total (of 30)** | **23** | **27** | **23** | **22** |

The bounded tool-loop wins. Single-turn is close on latency but fails the HITL dimension and cannot recover from an empty retrieval. Planner-executor is over-shaped for a 15-second reply draft and misses the p95 envelope. Chain-of-narrow is defensible for a product that handles billing *and* technical support *and* account questions on one surface, but this surface is scoped to one job.

### 2e. Reference decision memo (deliverable 4)

```markdown
# Decision: agent shape for helpdesk-reply-copilot (v1)

## The call
Ship **Shape B: bounded tool-loop** with a hard step cap of 6 tool calls
and a hard wall-clock cap of 10 s. One LLM, three tools from
[[exercise-01]] (mod-104), the confirm-before-write step from
[[exercise-03]], the budget guard from [[exercise-05]] (mod-104).

## Why this shape
1. Capability. The surface needs to react to retrieval results — if the
   first search returns nothing useful, the model must be able to
   rewrite the query before drafting. Shape A cannot; Shape B can.
2. Blast radius. The only write tool requires an explicit user confirm;
   the shape does not need delegation or a planner to be safe.
3. Latency. Six-step worst case keeps p95 under 10 s on the model we're
   using; the planner-executor shape would miss it.
4. HITL. The loop pauses naturally at the write-tool boundary — no
   re-planning, no re-drafting on confirm.

## Why not the alternatives
- **Shape A (single-turn):** dies on "no passages" and on tool error
  recovery. The surface would learn this the hard way in week two.
- **Shape C (planner + executor):** over-shaped for a 15-s draft. The
  planner adds a turn with no proportional capability gain; the critic
  adds a second. The memo acknowledges that if this surface grows to
  include multi-ticket "weekly report" workflows, C becomes viable.
- **Shape D (chain of narrow agents):** the surface has one job; a
  router is premature. Revisit when the surface's scope doubles.

## Operating envelope (what the shape must not exceed)
- ≤ 6 tool calls per request.
- ≤ 10 s wall-clock per request.
- ≤ 12 k input tokens / ≤ 2 k output tokens per request.
- 0 writes without an explicit confirm step.

## What would trigger a shape change
- **To Shape C:** the surface starts handling "write me a weekly
  escalation summary of the last 50 tickets." The step budget crosses
  15 in the 95th percentile.
- **To Shape D:** the surface adds a billing-action tool that materially
  differs from reply-drafting (different tools, different HITL rules).
- **To Shape A:** we observe that in prod 98% of successful draft
  trajectories use ≤ 2 tool calls, and the loop machinery is pure
  overhead. (Tracked quarterly from the eval harness.)

## What this choice commits us to
- Writing a shape-specific test row in the eval suite for every tool
  (see [[exercise-04]]).
- Emitting a trace entry per step so the memory system from
  [[exercise-02]] can scope session memory to a single loop.
- The framework choice from [[exercise-04]] (mod-104) is *downstream*
  of this decision — LangGraph and Agents SDK both express Shape B.
```

### 2f. Implementation notes

- The memo is the artefact that survives this exercise. Code written here is scaffolding; the decision memo is the product. The instructor should grade on the memo's defence first and the scoring table second.
- Capability fit is the only dimension that gets a 1. A shape that cannot do the job is disqualified before scoring other dimensions. The scoring is a way of finding the tightest adequate shape, not of polling.
- The operating envelope is written as non-negotiables *before* the shape is picked, not derived from the shape. A shape that cannot fit inside it fails dimension 2.
- Memory and eval fit are not tie-breakers. A shape that wins capability, latency, blast-radius, and HITL wins. The lower-weight dimensions decide close calls.
- The migration triggers (section "What would trigger a shape change") are the most under-written part of a typical submission. A memo without them turns the shape decision into a one-way door; adding them turns it into a reversible call with a named signal.
- The shape decision decides the shape of [[exercise-02]] (memory) and [[exercise-03]] (HITL) on this surface. Downstream exercises should cite this memo, not re-litigate shape.

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. The diagnosis page exists, fits on one page, and names (a) the user's job in the moment, (b) the actions the agent takes, (c) the worst and typical blast-radius cases, (d) what the agent must never do, (e) the non-functional envelope in numbers.
2. The shortlist contains at least three shapes including one the author ultimately rejects as "over-shaped" and one as "under-shaped." A one-shape shortlist is an opinion, not a decision.
3. The rubric is written before the comparison table and names which dimensions are disqualifier-eligible (⚠).
4. The scored comparison fills every cell and gives at least one shape a ≤ 2 in some ⚠ dimension (otherwise the rubric is not discriminating).
5. The decision memo picks exactly one shape, defends it against each alternative *by dimension*, and names at least two signals that would reverse the choice.
6. The operating envelope has four numbered caps (steps, latency, tokens, writes). "Reasonable" is not a cap.
7. The memo references the downstream exercises ([[exercise-02]], [[exercise-03]], [[exercise-04]] mod-105; [[exercise-05]] mod-104) whose shape depends on this one.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Diagnosis page is one page and names user-job + envelope | 1 | Numbered envelope; no handwaving. |
| At least three shapes considered incl. a rejected over-shape | 1 | The rejected shapes are defended, not strawmanned. |
| Rubric is pre-committed and names ⚠ dimensions | 1 | Rubric appears before the comparison table in the memo. |
| Scored table fills every cell | 1 | No blanks; no "N/A." |
| Blast-radius is reasoned from product consequences | 1 | Named failure modes with cost of each; not a vibe. |
| Latency numbers come from a model/envelope, not a guess | 1 | p50 / p95 cited with a source (envelope or measurement). |
| Pick is defended dimension-by-dimension | 1 | Each alternative is killed on a specific dimension score. |
| Operating envelope has numeric caps | 1 | ≥ 4 numeric caps (steps, latency, tokens, writes). |
| Migration triggers named for ≥ 2 directions | 1 | At least one promotion signal + one demotion signal, each measurable. |
| Downstream exercises cross-referenced | 1 | Explicit links to memory, HITL, eval choices gated by this decision. |

A submission scoring < 6 is rejected. The hard failures are (a) no operating envelope, (b) a shape picked without defending against the next-higher shape, (c) no migration triggers (one-way door).

## 5. Common mistakes

- **Picking a shape before writing the diagnosis.** The diagnosis exists so the shape has something to fit; skipping it produces an "autonomous agent" memo for a reply drafter.
- **Confusing shape with framework.** "We picked LangGraph" is not a shape decision. Shape is Shape A/B/C/D; framework is a downstream choice governed by [[exercise-04]] (mod-104).
- **Over-shaping to look sophisticated.** A planner-executor shape on a 15-s surface is a performance regression, not architecture. The simplest shape that fits is the right shape.
- **Under-shaping to look clever.** A single-turn shape that cannot react to "no passages" fails the first retrieval-miss in prod. The memo must say *how* the shape handles the known failure modes.
- **No numeric operating envelope.** "Low latency" and "small budget" lose the argument the first time an incident asks "what was the budget supposed to be?" Caps are numbers.
- **No migration triggers.** A memo without "here is what would make us change our mind" turns the decision into a one-way door. Name the signal and where it is measured.
- **Scoring every shape 4–5 across the board.** A rubric that fails to discriminate is a rubric that was fitted to the preferred shape. At least one shape should score a ⚠ disqualifier; if none do, the dimensions are not sharp enough.
- **Treating blast radius as "security review."** Blast radius is a product question answered by the surface owner. The security review in [[mod-107]] inherits this answer; it does not produce it.
- **Not naming what the agent must never do.** The "never do" list is the boundary the shape protects. Without it, the shape has no reason to be bounded.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the diagnosis + operating envelope + migration-trigger pattern maps to NIST AI RMF's MAP (context + intended use), MEASURE (envelope limits as measurable targets), and MANAGE (trigger signals that force re-evaluation). The RMF is explicit that AI-system scope must be documented before controls are chosen; this exercise is that documentation step for a product surface.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the "blast radius" and "what the agent must never do" sections are the applied-AI engineer's input to the OWASP ML-Sec review; without them, the security review has nothing to bound.
- MITRE ATLAS — <https://atlas.mitre.org/> — ATLAS maps adversary techniques against AI systems; shape choices directly constrain which techniques are reachable (a single-turn shape with no write tool has a different exposure than a planner-with-executor with delegated tool use). The shape memo is the artefact an ATLAS-informed threat model is scored against.

Related exercises: [[exercise-02]] (memory scope follows shape), [[exercise-03]] (HITL insertion points follow shape), [[exercise-04]] (eval suite is written against the shape's trajectory model); upstream [[exercise-01]] (mod-104) tool manifest and [[exercise-05]] (mod-104) budget guard are the primitives this shape is built from.
