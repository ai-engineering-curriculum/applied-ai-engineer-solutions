# exercise-02: SLO Mapping from Spec to Metrics — Solution

## 1. Solution overview

The exercise asks the engineer to translate a product spec into a set of **Service Level Objectives (SLOs)** and the **measurable indicators (SLIs)** that back them. SLOs for an LLM feature are not just latency and availability — they also need to cover **answer quality**, **unsafe output rate**, and **cost per successful interaction**, because any of those going wrong is user-visible even when the service is technically "up."

The deliverable is a mapping table from each spec clause to:

- an **SLI** (a concrete, loggable measurement),
- an **SLO** (a target value / threshold on that SLI over a window),
- an **error budget** policy (what happens when the SLO is burned),
- the **alert / page condition** derived from burn rate.

The NIST AI RMF `MEASURE` function is the frame for "what should we measure"; the SRE community's SLI/SLO/error-budget terminology is the frame for "how we commit to it." The worked example uses the same "Inbox Assistant" spec referenced from [[exercise-01]] so learners can see the chain Spec → Shape → SLOs.

## 2. Worked answer or implementation

### 2a. The four SLO categories every LLM feature needs

Learners must produce one SLO per category below, or explicitly justify why the category does not apply.

| Category | Why it matters | Default SLI shape |
|---|---|---|
| **Latency** | User-visible responsiveness. | p50 / p95 / p99 of time-to-first-token *and* time-to-final-token, measured end-to-end (client-observed when possible). |
| **Quality** | The feature can be "up" and still wrong. | Task-specific pass rate against an offline eval set + online proxy (thumbs/implicit signal). Must specify sample size. |
| **Safety** | Blast-radius control for the chosen shape. | Rate of outputs that trip the safety filter or that users flag, over N interactions. Must specify both false-positive and false-negative handling. |
| **Cost** | A runaway feature kills the budget before latency does. | Cost per successful interaction (not per request — successful = did not retry, did not get flagged). |

Availability (plain uptime) is still required, but for LLM features it is usually inherited from the provider's SLA and the feature's own infra — the four categories above are what the feature team *owns*.

### 2b. The mapping template (what the learner submits)

For each spec clause, the learner fills one row. One spec clause can map to several rows; some rows may map to no spec clause (floor SLOs) and must be flagged as such.

```
| Spec clause (quoted)   | Category (L/Q/S/C) | SLI (concrete signal)   | SLO (target, window)            | Error budget action                         | Alert / page       |
```

The SLO must name:

1. the **signal source** (where the metric comes from — gateway log, eval harness, user-feedback event),
2. the **aggregation window** (e.g., 30-day rolling),
3. the **target** (e.g., 99% of requests),
4. the **exclusions** (what you deliberately do not count — e.g., provider 5xx that failed over automatically).

### 2c. Worked example — "Inbox Assistant" spec

> **Spec (local exercise context, from [[exercise-01]]):** The feature summarizes the last 20 unread emails for the user, then drafts a one-paragraph reply to each. Panel must render first useful content within **p95 3 s**. Must not send email on its own. Target cost: **≤ $0.02 per user-session**. Drafts must not contain content that the user flags as "inappropriate" more than **1 in 500** sessions.

Spec → SLO mapping:

| Spec clause | Cat | SLI | SLO | Error budget action | Alert |
|---|---|---|---|---|---|
| "panel renders first useful content within p95 3 s" | L | Client-observed time from "open panel" event to first non-skeleton render; percentile computed over all opens in the window; excludes sessions where the user closed the panel in < 500 ms. | **p95 ≤ 3.0 s**, 28-day rolling, ≥ 95% of calendar days must also meet it. | If 28-day budget is 50% burned at mid-window, freeze latency-regressing merges; 90% burned → rollback last prompt / model change. | Burn-rate alert: 2% of the monthly budget burned in 1 h (page) or 5% burned in 6 h (ticket). |
| "target cost ≤ $0.02 per user-session" | C | `sum(token_cost + tool_cost) / count(successful_sessions)` from the gateway, where successful = at least one draft rendered and the session wasn't retried end-to-end. | **p50 cost per successful session ≤ $0.015**, **p95 ≤ $0.025**, 7-day rolling. | Budget burn pauses prompt-length experiments; two consecutive weeks over → re-open [[exercise-03]] provider-tier decision. | Ticket if p95 > $0.025 for 3 days running; no page — cost is a slow signal. |
| "must not send email on its own" | S | Count of `send_email` tool invocations attributable to the LLM path. Policy: must be zero, by shape (see [[exercise-01]]). | **0 autonomous sends** ever. This is a hard invariant, not a percentile. | Any non-zero reading is P1 incident + rollback. | Immediate page on count > 0. |
| "drafts flagged 'inappropriate' ≤ 1 / 500 sessions" | S | Per-session flag rate from the feedback event stream; denominator = sessions with ≥ 1 draft rendered. | **≤ 0.2% of sessions** with a flag, 28-day rolling. | 50% burned → enable stricter safety prompt + audit sample of 100 flagged sessions; 90% burned → pause new rollouts. | 4% burn in 1 h pages; 10% in 24 h tickets. |
| *(floor SLO, no explicit spec clause)* Quality: "drafts the user actually sends without edits" | Q | Per-draft offline eval pass rate on 200-example golden set + online send-without-edit rate from the UI. | **Offline pass ≥ 80%** on golden set per release; **online unedited-send rate ≥ 40%**, 14-day rolling, with a floor not a ceiling. | Offline regression blocks release; online drop > 10 pp triggers model/prompt review. | Offline eval failure in CI blocks merge; online drop tickets. |

Four observations the solution makes explicit:

1. The **safety invariant** ("0 autonomous sends") is not a percentile — it is a count-zero invariant. Learners who turn it into "99.99%" lose points.
2. The **cost SLO** is per **successful** session, not per request. Retries and failures must not appear to make unit economics better.
3. The **quality SLO** has both an offline and an online SLI. Offline guards releases; online guards reality.
4. **Exclusions are named.** The latency SLO excludes sessions where the user closed the panel in < 500 ms (noise), and the cost SLO excludes failed sessions (otherwise retries flatter the denominator).

### 2d. Error budget policy template

```
Window: 28-day rolling (default) or 7-day for cost
Budget calculation: (1 - SLO_target) * eligible_events_in_window

Automated actions at burn thresholds:
  25% burned → notify owning team channel (no action)
  50% burned → freeze risky merges in the feature's directory
  75% burned → disable the most recent experiment flag
  100% burned → rollback to last known-good configuration

Manual override:
  Owning engineer can suspend an action with a written justification in the incident channel.
  Suspensions expire in 24 h.
```

## 3. Validation steps

A submitted mapping is valid when all of the following hold:

1. Every spec clause is covered by at least one row, or explicitly marked "non-SLO" with a reason (e.g., "this clause is a UX rule, not a measurable guarantee").
2. All four categories (Latency / Quality / Safety / Cost) appear — or any missing category has a one-line justification.
3. Every SLI names its signal source and exclusions.
4. Every SLO has a target **and** a window **and** an aggregation (percentile or rate).
5. At least one SLO is modeled as a hard invariant (count-zero) if the spec has a "must not" clause.
6. Every SLO has an error-budget action and an alert rule — no SLO without a consequence.
7. The alert rule uses **burn rate** (budget burned per time) rather than raw threshold crossing, except for hard invariants.
8. The mapping names which SLO is **leading** (changes first on a regression — usually latency or online quality) and which is **lagging** (changes last — usually cost).

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Spec-clause coverage | 2 | Every clause is mapped, or an explicit "non-SLO" note is given. |
| Category coverage | 1 | All four categories present or justified. |
| SLI concreteness | 2 | Every SLI names signal source, aggregation, and exclusions; nothing reads "latency is good." |
| SLO completeness | 1 | Every SLO has target + window + percentile. |
| Hard invariants recognised | 1 | "Must not" clauses become count-zero invariants, not percentiles. |
| Error-budget action | 1 | Every SLO has a burn consequence; actions escalate. |
| Burn-rate alerts | 1 | Alerts are burn-rate based, except invariants (which page immediately). |
| Leading / lagging SLO called out | 1 | The memo identifies which SLO regresses first. |

Submissions scoring < 6 go back.

## 5. Common mistakes

- **Turning a "must not" clause into a percentile.** "Must not send email on its own" is a count-zero invariant. 99.99% is still ~44 wrong sends per 500k sessions — not what the spec says.
- **Measuring cost per request instead of per successful interaction.** Retries and failed requests should not flatter the denominator.
- **No exclusions.** Every real SLI has exclusions (bot traffic, sub-500 ms bounces, provider failover). If exclusions are missing, the number is noise.
- **Raw-threshold alerts instead of burn-rate alerts.** A single bad minute shouldn't page if the 28-day budget is fine. Burn rate solves this.
- **Floor quality SLO measured only offline.** Offline evals guard releases; they don't catch distribution shift. The online proxy is not optional.
- **Mixing availability and quality into one SLO.** "Was up and good" is actually two SLOs. Keep them separate so the error-budget actions can differ.
- **Latency SLO measured server-side only.** The spec is about user-visible render, not server response. Client-observed when the data pipeline allows.

## 6. References

- NIST AI Risk Management Framework (AI RMF 1.0) — <https://www.nist.gov/itl/ai-risk-management-framework> — the `MEASURE` function provides the structure for category coverage; the `MANAGE` function provides the error-budget / action framing.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — used to identify which safety SLOs are non-negotiable (e.g., prompt injection and model DoS risks).
- MITRE ATLAS — <https://atlas.mitre.org/> — adversary techniques inform the safety SLIs (e.g., abnormal token-growth signals that may indicate prompt-injection-driven cost attacks).

Related exercises: [[exercise-01]] produced the feature shape these SLOs guard; [[exercise-03]] feeds the cost SLO; [[exercise-04]] cross-checks whether all ISO/IEC 25059 quality dimensions in scope have a matching SLO.
