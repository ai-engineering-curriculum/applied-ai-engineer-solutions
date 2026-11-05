# exercise-01: Feature Shape Decision Rubric — Solution

## 1. Solution overview

The exercise asks the engineer to look at a product spec and decide the **shape** of the LLM feature before anything else. "Shape" means the user-visible interaction model and the system-level control flow that follow from it: is this a single-shot classifier, a streaming chat turn, a tool-calling agent loop, or an offline batch job? That choice sets almost every downstream constraint — latency budget, eval harness, failure mode, cost ceiling, rollout gate — so it deserves a written rubric, not a gut call.

This solution gives:

- A fixed **decision rubric** with five dimensions the reviewer scores every candidate shape against.
- A **worked example** applying the rubric to a sample spec drawn from the exercise brief (a "summarize-and-reply" inbox assistant).
- A **grading rubric** the instructor uses to mark a learner submission.

The rubric is deliberately shape-agnostic. It does not prescribe "always use tool calling" — it forces the learner to make the trade-off explicit.

## 2. Worked answer or implementation

### 2a. The five-dimension decision rubric

For a given candidate shape (e.g., *single-turn structured output*, *streaming chat*, *tool-using agent*, *async batch*), score each dimension 1–5 and record the rationale. A dimension ≤ 2 is a hard disqualifier unless the spec explicitly accepts it.

| # | Dimension | What to check | Disqualifier signal |
|---|---|---|---|
| 1 | **User task fit** | Does the shape match the user's actual job-to-be-done in the spec (interactive vs. fire-and-forget, one answer vs. iterative refinement)? | User needs to interrupt / correct mid-task, but the shape is non-streaming batch. |
| 2 | **Latency & presence** | Can the shape hit the spec's p50/p95 budget end-to-end (model + retrieval + tools + render)? | p95 latency target < model's realistic first-token time. |
| 3 | **Control & reversibility** | How reversible is a wrong model output in this shape? Can the user see it before it takes effect? | Shape auto-executes side effects (sends mail, posts PR) with no confirm step, in a domain where errors are costly. |
| 4 | **Blast radius if wrong** | What is the worst-case cost of a bad output in this shape? Consider safety (OWASP ML Top 10 relevance), legal, and reputational blast radius. | Autonomous agent shape in a domain where a hallucinated action violates a regulatory duty. |
| 5 | **Eval & observability fit** | Can you build a per-turn eval and an incident-grade trace for this shape with current tooling? | Multi-step agent loop with no way to attribute a bad final answer to a specific step. |

A candidate shape is "Recommended" only if every dimension is ≥ 3 and no spec-level disqualifier fires. Ties break toward the **simpler** shape (single-turn > streaming > tool-use > agent loop), because simpler shapes have smaller blast radius, cheaper evals, and faster rollback.

### 2b. Candidate shapes considered

The rubric is applied to a fixed shortlist so learners do not debate taxonomy:

1. **Single-turn structured output** — one request, one validated JSON response. No streaming.
2. **Single-turn streamed chat** — one request, tokens streamed to the UI; no tools.
3. **Multi-turn chat with memory** — session with prior turns, no tools.
4. **Tool-calling single-shot** — one model call that may emit one or more tool calls, executed, then a final response.
5. **Tool-calling agent loop** — the model can iterate tool-call → observation → tool-call until a stop condition.
6. **Async batch / offline job** — results delivered later (email, notification, webhook).

### 2c. Worked example — "Inbox Assistant" spec

> **Spec (local exercise context):** The feature summarizes the last 20 unread emails for the user, then drafts a one-paragraph reply to each. The user reviews each draft in a side panel and sends it manually. p95 time-to-first-useful-render budget: 3 s. The feature must not send email on its own. Target cost: ≤ $0.02 per user-session.

Candidate scoring:

| Shape | 1. Task fit | 2. Latency | 3. Control | 4. Blast radius | 5. Eval fit | Verdict |
|---|---|---|---|---|---|---|
| 1. Single-turn structured output | 4 — matches "list of {summary, draft} objects" | 3 — one call, tight; depends on provider | 5 — user reviews before send | 5 — no side effects | 5 — JSON schema → easy per-field eval | **Recommended** |
| 2. Single-turn streamed chat | 2 — no structure for 20 items | 5 | 4 | 4 | 2 — hard to eval free text at scale | Disqualified on 1, 5 |
| 3. Multi-turn chat with memory | 2 — not conversational; refinement is per-draft, not per-session | 3 | 4 | 4 | 3 | Disqualified on 1 |
| 4. Tool-calling single-shot | 3 — reads inbox via tool, drafts reply | 3 | 4 | 4 — tool read-only | 4 | Viable, but strictly more complex than shape 1 if emails are passed in-context |
| 5. Tool-calling agent loop | 2 — no iterative decision-making in the spec | 2 — unpredictable latency | 2 — loop could re-plan | 3 — must hard-gate `send_email` | 2 — attribution across steps is painful | Disqualified on 1, 2, 3, 5 |
| 6. Async batch | 2 — user is waiting in a panel | 1 — violates 3 s render target | 5 | 5 | 4 | Disqualified on 1, 2 |

**Decision:** Shape 1 — single-turn structured output — with a JSON schema of `{email_id, summary, draft_reply}` objects. The `send` action lives outside the LLM path entirely and is a plain product button. If retrieval of email bodies turns out to be too big for one context window, fall back to shape 4 (tool-calling single-shot with a `get_email_body(id)` tool), but *only then* — do not pre-optimize.

**Why simpler wins here:** shape 1 is reversible (every draft is user-reviewed), auditable (one call = one trace), cheap to eval (per-field metrics), and the spec's "do not send on its own" requirement is satisfied by design, not by prompt discipline. Promoting to a tool-using agent loop would add attribution cost and blast radius without closing any gap in the spec.

### 2d. Decision rationale template (what the learner submits)

The rubric produces a short memo with this skeleton — this is the artifact the learner submits:

```
Feature: <name>
Spec summary: <3-line summary of the product spec>

Shortlisted shapes considered: <list from 2b>

Scored comparison: <the table from 2c>

Chosen shape: <name>
Why this shape wins: <2-4 bullets tying to spec constraints>
Why the next-best shape loses: <1-2 bullets>
Hard disqualifiers that fired for the rejected shapes: <list>
Rollback / escape hatch if this shape turns out wrong: <what you switch to, and the signal that triggers the switch>
```

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. The product spec is quoted or summarized at the top in ≤ 5 lines so a reviewer can score it independently.
2. At least **four** of the six canonical shapes from 2b appear in the comparison table (not just the chosen one and a strawman).
3. Every dimension score in the table has a one-line rationale — no bare numbers.
4. Every rejected shape has at least one dimension at ≤ 2 **or** an explicit "viable but dominated by simpler option" verdict.
5. The chosen shape's latency row and blast-radius row each cite a specific number or constraint from the spec, not a generic claim.
6. The rollback row names a concrete next shape and a concrete trigger signal (e.g., "switch to shape 4 if any email body exceeds 8k tokens on 5% of sessions").
7. If the chosen shape has side effects, there is one sentence identifying the OWASP ML Top 10 risk category that applies and how the shape mitigates it, or a `<!-- research-pending: -->` tag if the mapping is unclear.

Step 7 is the smallest-possible connection to the course's security baseline; see references.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Spec restated faithfully | 1 | Reviewer can score the rubric without reading the original spec. |
| Shortlist is complete | 1 | ≥ 4 canonical shapes compared; no missing shape that the spec obviously suggests. |
| Every score has a rationale | 2 | No bare numbers; rationale references a spec constraint or a known shape property. |
| Disqualifiers are called out explicitly | 1 | Rejected shapes have a named blocking dimension, not just a low total. |
| Chosen shape ties to spec numbers | 2 | Latency, cost, and blast-radius rows cite the spec, not generic defaults. |
| Simpler-wins tie-break applied | 1 | If two shapes tie, the memo justifies the simpler pick or explains why it is dominated. |
| Rollback trigger is concrete | 1 | Named next shape + named signal (metric, threshold, or event). |
| Safety / risk hook present | 1 | One sentence mapping the chosen shape to OWASP ML Top 10 or NIST AI RMF function (GOVERN / MAP / MEASURE / MANAGE), or an explicit `<!-- research-pending: -->`. |

A submission scoring < 6 is rejected and sent back for a second pass.

## 5. Common mistakes

- **Jumping straight to "agent loop"** because it feels most impressive. The rubric exists to catch this: agent loops score poorly on control, blast radius, and eval fit for most product specs.
- **Ignoring the latency budget end-to-end.** Learners score "model latency" instead of "model + retrieval + tool + render." The spec's budget is for the user-visible path.
- **Treating "streaming" as a shape by itself.** Streaming is a presentation choice layered on top of a shape. The rubric separates them: a streamed shape still has to pass control and eval dimensions.
- **Disqualifying a shape without naming the blocking dimension.** Reviewers can't audit a verdict of "no."
- **No rollback row.** The chosen shape must come with the next-simpler shape queued as an escape hatch and a signal that triggers the fall-back.
- **Over-fitting to one provider.** The rubric is provider-agnostic. Learners should not pick a shape because "model X is good at it today" — provider choice is [[exercise-03]].

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — used in the safety-hook row of the instructor rubric.
- NIST AI Risk Management Framework (AI RMF 1.0) — <https://www.nist.gov/itl/ai-risk-management-framework> — the GOVERN / MAP / MEASURE / MANAGE functions give the vocabulary for the risk-hook sentence.
- MITRE ATLAS — <https://atlas.mitre.org/> — adversary techniques matrix; learners pointing at agent-loop shapes should name the ATLAS techniques their shape enables (e.g., tool misuse) and how the shape mitigates them.

Related exercises: [[exercise-02]] turns the chosen shape into SLOs; [[exercise-03]] layers the provider decision on top; [[exercise-04]] audits the chosen shape against ISO/IEC 25059 quality dimensions.
