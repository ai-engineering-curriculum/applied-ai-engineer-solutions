# exercise-03: Provider Tier Trade-off Note — Solution

## 1. Solution overview

The exercise asks the engineer to write a one-page **trade-off note** choosing a model provider tier for the feature. "Tier" here is the level of model + hosting + contractual guarantee, not a specific product name — the note has to survive model refreshes without being rewritten.

The target shape of the note is a decision memo a product lead and a security reviewer can both read in five minutes and either sign off or push back on. The solution provides:

- A **tier taxonomy** that is model-name-free, so the rubric doesn't rot.
- A **seven-axis scoring frame** that forces the learner to be explicit about cost, latency, quality, controllability, safety, compliance, and provider risk.
- A **worked trade-off note** for the "Inbox Assistant" spec continued from [[exercise-01]] and [[exercise-02]].
- A **grading rubric** and the hard rules a passing note must obey.

Because model and price details change weekly, this solution deliberately does **not** list named models or current prices. Learners who copy a price into their note must label it with a date and a source; otherwise it must be `<!-- research-pending: current price for tier X as of <date> -->`.

## 2. Worked answer or implementation

### 2a. Tier taxonomy (model-name-free)

| Tier | Shorthand | What it is | Typical use |
|---|---|---|---|
| T0 | Frontier hosted | The provider's most capable general model, hosted by them, closed weights. | Hardest reasoning / long-context tasks, expensive to run. |
| T1 | Mid hosted | A cheaper/faster hosted model from a major provider, same API surface as T0. | Most product features; the default. |
| T2 | Small hosted | A small hosted model, often same provider, optimized for latency and cost. | Classification, routing, extraction, cheap first-pass. |
| T3 | Open-weights self-hosted | An open-weights model run on the team's own infra. | Data-sovereignty needs, offline eval, fallback. |
| T4 | Open-weights managed | An open-weights model run by a managed inference provider. | T3 benefits without owning GPUs. |
| T5 | Fine-tuned (any of the above) | Any tier above, with a task-specific fine-tune. | Steady-state task with a stable eval set; usually upgraded *to* from T1/T2 after the feature matures. |

A real decision picks **two** tiers: a **primary** and a **fallback/escape-hatch**. A note that picks only one loses points — nothing downgrades gracefully under provider incidents.

### 2b. The seven-axis scoring frame

Each candidate tier gets scored 1–5 on every axis, with a one-line rationale. The axes are:

1. **Quality on the task** — pass rate on the feature's offline eval (from [[exercise-02]]), *not* on a public leaderboard.
2. **Latency fit** — realistic p50/p95 time-to-first-token and time-to-final-token at the feature's expected context length.
3. **Cost at expected volume** — per-successful-interaction cost at the feature's actual prompt/response shape, dated.
4. **Controllability** — does the tier support the primitives the shape needs: structured outputs, tool/function calling, streaming, logprobs, system prompts, caching?
5. **Safety posture** — provider-side safety filtering, abuse monitoring, published policy on prompt-injection defense, red-team disclosures. Cross-reference to OWASP ML Top 10 and MITRE ATLAS.
6. **Compliance / data posture** — data retention defaults, training-on-customer-data defaults, region/hosting options, SOC 2 / ISO 27001 / HIPAA / BAA availability, EU AI Act stance. `<!-- research-pending: -->` anything that can't be verified from the provider's own documentation on the day of writing.
7. **Provider risk** — concentration (how many critical features already depend on this provider), portability (how similar the API is to alternatives), and roadmap signals (deprecations, rate-limit history).

A tier scoring ≤ 2 on any axis the spec cares about is disqualified for that role. "Cares about" must be justified from the SLOs produced in [[exercise-02]].

### 2c. Worked trade-off note — "Inbox Assistant"

> **Feature:** Inbox Assistant (continued from [[exercise-01]] and [[exercise-02]])
> **Shape:** single-turn structured output of `{email_id, summary, draft_reply}` per unread email.
> **SLOs that bind this decision:** p95 render ≤ 3 s; cost ≤ $0.02 / session; flag rate ≤ 0.2%; must not send email on its own (count-zero invariant).

**Candidate tiers:** T0 (frontier hosted), T1 (mid hosted), T2 (small hosted). T3–T5 are out of scope for v1: no data-sovereignty requirement in the spec and no stable eval set yet to justify a fine-tune.

**Scoring:**

| Axis | T0 Frontier | T1 Mid | T2 Small |
|---|---|---|---|
| 1. Quality on the task | 5 — surplus capacity for a simple draft | 4 — meets eval bar (from [[exercise-02]] offline set, ≥ 80% target) | 2 — borderline on nuanced tone |
| 2. Latency fit | 3 — longer first-token; risks 3 s p95 at 20 emails | 4 — comfortably fits with batching | 5 — fastest |
| 3. Cost at expected volume | 2 — blows $0.02 session cap at 20 drafts | 4 — fits cap with margin | 5 — well under cap |
| 4. Controllability | 5 — structured output + caching | 5 — same surface | 4 — may lack some features; verify before commit |
| 5. Safety posture | 5 — most mature filtering | 5 — same provider, same filtering | 4 — fewer published red-team results |
| 6. Compliance / data posture | identical across tiers for same provider | identical | identical |
| 7. Provider risk | concentrated; API portable to peers; medium | same | same |

**Decision:** **T1 primary, T2 fallback.** Not T0: cost axis (2) disqualifies it against the $0.02 session cap. Not T2 as primary: quality axis (2) is below the offline-eval floor. T2 is kept as the fallback for incident mode — if the T1 provider degrades, routing to T2 preserves the latency and cost SLOs at the cost of a temporary quality dip, which the error-budget policy from [[exercise-02]] explicitly accepts.

**Why this survives a model refresh:** the decision is tied to **axes**, not to model names. When the provider releases a new T1 model, we re-run the offline eval and the cost calculation; if both still pass, the decision holds. If T2 later closes the quality gap, we re-score and may promote it to primary.

**Open items:**

- `<!-- research-pending: current per-million-token price for the chosen T1 and T2 models from the provider's published pricing page on the day of merge -->`
- Confirm whether the chosen T2 supports the structured-output primitive the shape depends on; if not, treat the fallback as a shape downgrade ([[exercise-01]] shape 4), not just a model swap.

### 2d. The hard rules a passing note must obey

1. **No bare model names without a date.** Model identifiers rot — tier names don't.
2. **Two tiers, not one.** A primary and a fallback, or a defended "no fallback needed" argument.
3. **Scoring rationale uses this feature's eval and this feature's SLOs** — not leaderboard scores and not generic benchmarks.
4. **Compliance claims cite the provider's own current documentation.** Anything un-citable is tagged `<!-- research-pending: -->`.
5. **Provider risk is named.** The note identifies the concentration cost of adding another feature on this provider.
6. **The note names the re-decision trigger** — the event that reopens this decision (quality regression, price change > X%, provider deprecation).

## 3. Validation steps

A submission is valid when:

1. The feature's SLOs (from [[exercise-02]]) are quoted or summarized so the trade-off is anchored to real constraints, not vibes.
2. At least three tiers appear in the scoring table.
3. Every cell has a short rationale, not a bare number.
4. The chosen primary and fallback are named, and the fallback's role (degraded-quality mode, full-replacement, offline-only) is explicit.
5. Any price or capability claim is either cited to a provider doc with a date or wrapped in `<!-- research-pending: -->`.
6. A re-decision trigger is written down.
7. The provider-risk row explicitly counts how many other features already depend on the chosen provider (concentration is a real cost).

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Anchored to feature SLOs | 2 | The decision references this feature's eval + latency + cost SLOs, not generic benchmarks. |
| Primary + fallback | 2 | Two tiers named, with the fallback's role defined. |
| Scoring rationale | 2 | Every cell has one line of reasoning tied to the feature. |
| Compliance / data posture | 1 | Retention, training-on-data, region, and relevant certifications called out or `<!-- research-pending: -->`. |
| Safety posture | 1 | Cross-reference to OWASP ML Top 10 or MITRE ATLAS when the shape has non-trivial blast radius. |
| Provider concentration | 1 | Note counts how many other features depend on this provider. |
| Re-decision trigger | 1 | Named event reopens the decision. |

Scoring < 6 → back.

## 5. Common mistakes

- **Pinning the decision to a specific model name.** Six months later the model is deprecated and the note is dead. Pin to tiers; re-score models.
- **Picking frontier "to be safe."** Frontier is sometimes the wrong safe choice: it blows cost SLOs and adds latency the feature can't absorb.
- **No fallback.** Any single-provider feature will see degraded mode eventually. The fallback can be a smaller model, a non-LLM path, or a degraded UX, but it must exist.
- **Leaderboard scores as evidence.** Public evals do not predict this feature's eval. Use the offline eval from [[exercise-02]].
- **Ignoring provider concentration.** The fifth feature on the same provider has very different risk from the first.
- **Compliance by assumption.** "I think they're SOC 2" is not a citation. Either link the provider's doc or `<!-- research-pending: -->`.
- **Confusing training-on-customer-data default with opt-out availability.** Both matter; the note has to name both.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — informs axis 5 (safety posture); learners should map tier capabilities to OWASP risk categories relevant to the shape.
- MITRE ATLAS — <https://atlas.mitre.org/> — adversary techniques matrix; useful for scoring which abuse categories each tier publishes mitigations for.
- NIST AI Risk Management Framework (AI RMF 1.0) — <https://www.nist.gov/itl/ai-risk-management-framework> — the `GOVERN` function is the frame for the provider-risk row (third-party risk + roles & responsibilities).

Related exercises: [[exercise-01]] (the feature shape dictates which tier capabilities are non-negotiable); [[exercise-02]] (the SLOs are the budget this decision must fit); [[exercise-04]] (the ISO/IEC 25059 audit cross-checks whether provider posture maps to all in-scope quality sub-characteristics).
