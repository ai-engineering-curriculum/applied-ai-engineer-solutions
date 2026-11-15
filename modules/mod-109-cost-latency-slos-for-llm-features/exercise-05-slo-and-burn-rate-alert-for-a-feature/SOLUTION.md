### exercise-05: SLO and Burn-Rate Alert for a Feature — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to convert a green-by-eye latency dashboard into a **written SLO with multi-window multi-burn-rate alerts**, name the **levers** (retry budget, hedge, provider fallback, graceful degrade) that fire at each burn level, and produce a **runbook** that a 3 AM on-call can read without prior context. The deliverable is the artefact and the alert rules — not an incident response. The artefact exists so that when the alert does fire, the response is legible.

The failure mode the exercise is written to catch is the one from the problem statement: a p95 dashboard that stays green all quarter while users see degraded latency for hours before someone pages. A static p95 threshold alert cannot distinguish "the budget is being burned fast" from "a few p95 samples blipped and settled" — it fires on the wrong signal entirely. The fix is the SRE-standard multi-window multi-burn-rate pattern: a short, aggressive burn alert pages for fast exhaustion, a longer, gentler burn alert tickets for slow exhaustion, and both are grounded in the **error budget as a quantity**, not the SLO as a percentage.

This solution provides:

- A **worked SLI expression** over the span log from [[exercise-01]] and [[exercise-04]], with each clause justified.
- An **SLO target + rolling window** with the error budget computed from a real 28-day traffic query.
- **Burn-rate alert rules** in Prometheus `alerts.yaml` shape (portable to Datadog / New Relic / Honeycomb), with the 14.4×-over-1h-and-6×-over-6h pattern explained.
- A **levers table** naming exactly which lever fires at which burn level, with the cost or capability tradeoff for each.
- A **runbook stub** that reads in the second person, for a groggy on-call.
- An **MLPerf sanity-check paragraph** contrasting the feature's observed TPOT p95 against a reference serving benchmark.
- A **grading rubric** for the instructor.

The lesson the exercise defends: *a p95 dashboard is not an SLO*. An SLO is a budget with a burn rate; the burn rate is what pages someone before the quarter's budget is gone. The levers are what the on-call pulls, which means on-call must know them before the alert fires — not after.

## 2. Worked answer or implementation

### 2a. SLI expression (deliverable 1)

The SLI is a boolean over one span. Each clause is justified in the doc so a future reader does not reopen the decision.

```python
# For readability; the real artefact is docs/slo-spec.md and the query below.
def SLI_good(span) -> bool:
    return (
        span.app.latency.ttft_ms < 1500                            # (A)
        and span.gen_ai.response.finish_reason in {"stop", "tool_calls"}  # (B)
        and span.app.structural_validity == "pass"                 # (C)
        and span.app.retry.attempt_number <= 2                     # (D)
    )
```

Clause-by-clause justification (written into `docs/slo-spec.md`):

- **(A) `ttft_ms < 1500`** — Matches the "red" ceiling on the latency spec from [[exercise-01]]: 1500 ms is where the agent's compose-box UX breaks. The SLI threshold must match the spec's red boundary; inventing a new number here is how two teams end up arguing over "what good looks like."
- **(B) `finish_reason in {"stop", "tool_calls"}`** — Streams that end on `length` (truncation) or `content_filter` (moderator refusal) are not user-success outcomes, even if they are fast. `"tool_calls"` is kept because an agent-mediated tool invocation is a legitimate completion state for this surface.
- **(C) `structural_validity = "pass"`** — If the response is a JSON tool call, it must parse against the declared schema (mod-102 substrate). A fast structurally-invalid response is not a good response.
- **(D) `attempt_number <= 2`** — A single retry is acceptable; two or more is a signal the primary provider is unhealthy for that call and the surface paid real latency for the recovery. The budget absorbs occasional two-attempt wins; the SLO fails them so the on-call can see the retry-rate trend.

**Named exclusions** (from the SLO, not from the dashboard):

- Moderator refusals on policy-legitimate content. The surface's safety stack (mod-107) owns these; counting them against the latency SLO conflates two signals.
- Opt-in long-context requests (`input_tokens > 60k`). The spec in [[exercise-01]] already excludes these from the p95; the SLO inherits the exclusion.
- Warm-up traffic (< 60 s since pod start).

```sql
-- SLI materialisation query (DuckDB dialect; translate to your backend).
SELECT
  ts,
  app_feature,
  CASE
    WHEN app_latency_ttft_ms < 1500
     AND gen_ai_response_finish_reason IN ('stop', 'tool_calls')
     AND app_structural_validity = 'pass'
     AND app_retry_attempt_number <= 2
    THEN 1 ELSE 0
  END AS sli_good
FROM spans
WHERE app_feature = 'helpdesk.reply'
  AND app_env = 'prod'
  AND gen_ai_usage_input_tokens <= 60000
  AND (ts - pod_start_ts) > INTERVAL 60 SECOND
  AND NOT (app_outcome = 'content_refusal' AND app_refusal_was_legitimate = TRUE);
```

### 2b. SLO target and error budget (deliverable 2)

```markdown
## SLO
Target:   99.0% of valid requests meet SLI_good over a rolling 28-day window.
Window:   28 days.
Rationale: 99.0% is honest for a first LLM-feature SLO. 99.5% is a stretch
           for a provider-dependent surface; 99.9% is aspirational and
           would be eaten by provider tail variance before we could pull
           any lever. 28 days matches the on-call rotation cadence.

Error budget (as a quantity, from a real 28-day query):

  Measured over 2026-10-17 through 2026-11-13:
    valid requests:        1,284,900
    bad requests (SLI=0):      9,420     (observed: 99.267% — above target)

  Budget for the next 28 days:
    (1 − 0.99) × 1,284,900 ≈ 12,849 bad requests allowed per window.

Note: the budget is a REQUEST QUANTITY, not a percentage. The burn-rate
math below consumes requests, not percentage points.
```

```sql
-- Query used to measure volume for the error-budget size.
SELECT COUNT(*) AS valid_requests,
       SUM(1 - sli_good) AS bad_requests,
       1.0 * SUM(sli_good) / COUNT(*) AS observed_sli
FROM sli_materialised
WHERE ts >= now() - INTERVAL 28 DAY;
```

### 2c. Burn-rate alerts (deliverable 3)

The multi-window multi-burn-rate pattern is the SRE-standard shape (originating in Google's SRE Workbook, Chapter 5, "Alerting on SLOs"). Use the same numbers; do not invent new ones.

- **14.4× burn rate** over 1 hour → fast-burn page. At 14.4× you consume the entire 28-day budget in `28 / 14.4 ≈ 1.94 days` if sustained, which is why 14.4 corresponds to "2-day burn-through."
- **6× burn rate** over 6 hours → slow-burn ticket. At 6× you consume the budget in `28 / 6 ≈ 4.67 days` — "5-day burn-through."

The two windows together are what prevent false-positive-on-blip and false-negative-on-sustained-but-moderate.

```yaml
# alerts/slo-burn.yaml
# Burn-rate alerts for the helpdesk.reply feature SLO.
# SLO target: 99.0% of valid requests meet SLI_good over 28 days.
#
# Pattern: multi-window multi-burn-rate (see Google SRE Workbook, Ch. 5,
# "Alerting on SLOs"). 14.4× over 1h burns the 28-day budget in ~2 days;
# 6× over 6h burns it in ~5 days.
#
# "Burn rate" = (1 - observed_sli_over_window) / (1 - slo_target)
# At 14.4×, 1 - sli = 14.4 × 0.01 = 0.144 over 1h.
# At 6×,    1 - sli = 6    × 0.01 = 0.06  over 6h.

groups:
  - name: helpdesk-reply-slo
    rules:
      - alert: HelpdeskReplySLOFastBurn
        expr: |
          (
            1 - (
              sum(rate(sli_good_total{feature="helpdesk.reply"}[1h]))
              /
              sum(rate(sli_total{feature="helpdesk.reply"}[1h]))
            )
          ) > (14.4 * 0.01)
        for: 5m
        labels:
          severity: page
          slo: helpdesk-reply-ttft
        annotations:
          summary: "helpdesk.reply SLO fast burn (14.4× over 1h)"
          runbook: "docs/runbook.md#fast-burn"
          lever: "retry, hedge if armed, fallback on sustained"

      - alert: HelpdeskReplySLOSlowBurn
        expr: |
          (
            1 - (
              sum(rate(sli_good_total{feature="helpdesk.reply"}[6h]))
              /
              sum(rate(sli_total{feature="helpdesk.reply"}[6h]))
            )
          ) > (6 * 0.01)
        for: 15m
        labels:
          severity: ticket
          slo: helpdesk-reply-ttft
        annotations:
          summary: "helpdesk.reply SLO slow burn (6× over 6h)"
          runbook: "docs/runbook.md#slow-burn"
          lever: "degrade (disable optional tool), then investigate"
```

The `for:` holds (5 min on the fast alert, 15 min on the slow) are what prevent single-sample noise from paging. They are the second half of the "multi-window" story; without them the fast alert fires on every 1-hour blip.

### 2d. Levers table (deliverable 4)

```markdown
## Levers (what fires at which burn)

| Lever | Fires when | Cost / capability tradeoff |
|---|---|---|
| **Retry** | Transient 5xx / 429 on primary provider, any time. Always armed. | +5% retry budget on the window; TTFT delta is the retry latency (see exercise-01 report, ~470 ms p95 uplift). |
| **Hedge** | Fast-burn alert fires AND no first token in `T_hedge = 900 ms`. | +20–40% primary-side cost on hedged calls (both provider requests pay through TTFT). Cross-provider cost model (stretch goal): at current 15 QPS, steady-state hedge cost delta is ~$0.12/hour. Disable outside fast-burn. |
| **Fallback** | 10 consecutive primary 5xx OR primary error rate > 50% over 2 min. Mod-108 chapter 02 substrate. | Fallback model is `claude-small` (reference). CPSO delta: approximately +12% on fallback traffic because the fallback model has lower cache-hit ratio; see exercise-04 ledger for current number. Cycles back to primary after 10 min healthy. |
| **Degrade** | Slow-burn alert fires. Product sign-off required for first-time-this-quarter. | Disable the optional `crm.search` tool (adds ~1050 ms to p95 tool-round-trip; see exercise-01 report). Feature capability loss: agent sees fewer historical tickets on hover. Expected TTFT p95 improvement: ~500 ms. Product sign-off: Avery (triage). |
```

Every lever names a tradeoff. The runbook relies on the on-call knowing what each lever costs — "pull the hedge" is a confident instruction only if "the hedge costs 30% more on that path" is already in the doc.

### 2e. Runbook stub (deliverable 5)

```markdown
# docs/runbook.md — helpdesk-reply SLO runbook

## You are on-call. You just got paged. Read this.

The alert is one of two. The label `slo=helpdesk-reply-ttft` confirms you
are in the right runbook.

## FAST BURN (14.4× over 1h) — PAGE

### What this means
In the last hour, more than 14.4% of valid helpdesk.reply requests failed
the SLI. If this holds, we burn the entire 28-day error budget (~12,800
bad requests) in less than 2 days. The dashboard looks worse than the
quarterly chart has ever looked.

### First 90 seconds
1. Open: dashboard `/d/helpdesk-reply-slo` (links from the alert).
2. Confirm the alert is real by cross-checking:
   - Is `app.retry.attempt_number > 1` rate climbing? (It usually is.)
   - Is any single provider showing elevated 5xx?

### Lever 1 — Check retries are firing (always)
Retries should already be absorbing transient 5xx. If not firing, the
primary gateway's retry config is stuck → page platform.

### Lever 2 — Enable hedge (default off)
Edit `routing.yaml` on primary: `hedge_enabled: true` for route
`helpdesk.reply`. Expected effect: TTFT p95 drops ~15–25%. Cost impact
is on the span (`app.cost.*`); monitor CPSO on dashboard 2.

### Lever 3 — Flip primary provider (mod-108 chapter 02)
If a specific provider is unhealthy, flip `primary: anthropic` ↔
`primary: openai` on the route. The thin gateway (mod-108 exercise-01)
absorbs the provider-side difference.

### Escalate when
After 15 minutes, if the burn rate is still > 14.4×, escalate to:
- helpdesk-platform: tech lead, phone tree
- platform-sre: on-call rotation

### Resolve
Mark the alert resolved only when the 1-hour burn rate has stayed
< 2× for 30 minutes. Write a short postmortem (mod-110 chapter 05
substrate).

## SLOW BURN (6× over 6h) — TICKET

### What this means
The SLO has been breached at a rate that eats the budget in ~5 days. Not
an immediate page; a ticket. Work the ticket by end of shift.

### First checks
1. Open: dashboard `/d/helpdesk-reply-slo`.
2. Which clause of the SLI is failing? (Common: `attempt_number > 2`
   or `structural_validity = pass`.)
3. Correlate with any release markers in the last 24h on dashboards 1
   and 2 of the ledger (mod-109 exercise-04).

### Lever 1 — Degrade (requires product sign-off)
Disable the `crm.search` tool for this surface. Config:
`src/surface.py`, `TOOLS_ENABLED`. Expected TTFT p95 improvement: ~500 ms.
Product sign-off: Avery (triage) — reach by DM; sign-off valid for 24h.

### Lever 2 — Rollback recent release
If the step up correlates with a release marker, roll back. Owner:
helpdesk-platform. Rollback procedure: `scripts/deploy rollback <sha>`.

### Escalate when
After 2 hours of work on the ticket without improvement, convert the
ticket to a page.

## Dashboards
- `/d/helpdesk-reply-slo` — SLI burn and budget consumption.
- `/d/helpdesk-reply-latency` — per-component latency (mod-109 exercise-01).
- `/d/cost-by-feature` — spend + CPSO (mod-109 exercise-04).
```

### 2f. MLPerf sanity check (deliverable 6)

```markdown
## Reference-frame sanity check

Our measured TPOT p95 on `helpdesk.reply` is **31 ms / output token**
(from the exercise-01 instrumentation, 7-day window).

MLPerf Inference publishes model-and-hardware-specific tokens-per-second
numbers for a small set of reference models and accelerators. A direct
one-to-one comparison against `claude-mid` is not possible (vendor-managed
provider; MLPerf covers open-weight models on known hardware). The closest
comparable is a MLPerf Inference submission for a similarly-sized decoder
model on a current-generation accelerator, which lands in the 40–80 output
tokens/second range at single-stream latency-constrained scenarios —
equivalent to ~12–25 ms / output token on the server-side alone. Our
observed 31 ms / token is ~1.3–2.6× slower than that reference, which is
the expected shape: our number includes gateway hop + SSE framing + retry
tax + normalisation, and MLPerf's is model-and-hardware only. The order of
magnitude agrees, so the gating factor is not the model — it is the stack.

Cite: MLPerf Inference results, <https://mlcommons.org/benchmarks/inference-datacenter/>.
<!-- verify-before-publishing: before publication, replace the 40–80 tok/s range
with an exact value from the most recent MLPerf Inference Datacenter
submission for a comparable model/hardware pair, and cite the submission
URL and date. -->
```

The point of the sanity check is not to set the SLO — it is to rule out the hypothesis "the SLO is unachievable because the model cannot run that fast on any stack." If the production number is dramatically slower than the MLPerf reference (say, 10× worse), that is a signal the constraint is in the gateway, the retries, the tool-path, or the SSE encoding — not the model.

### 2g. Implementation notes and gotchas

- **The SLI threshold must match the "red" line on the latency spec.** If [[exercise-01]] says red is 1500 ms and the SLI says 2000 ms, the two documents will drift and the on-call will not know which is authoritative.
- **Multi-window is the point.** A single-window burn alert fires too much on short windows and too late on long windows. 1h + 6h is not arbitrary; it is the pair that minimises both failure modes.
- **`for: 5m` on the fast-burn alert.** Without the hold, a single noisy 5-minute window pages. The hold is what turns "a p95 outlier" into "a real burn."
- **99.0%, not 99.9%.** An LLM-feature SLO at 99.9% requires 99.99% provider uptime after retries; that is a provider property, not a surface property. Pick 99.0% and move to 99.5% after a quarter of green.
- **The levers must be built before the alert fires.** This exercise names which lever to pull; mod-108 and chapter 05 are where the retry / hedge / fallback / degrade paths are implemented. An SLO with no built levers is a document that will be read as a eulogy at the incident review.
- **Budget as a quantity, not a percentage.** "99%" is hard for on-call to reason about; "12,849 bad requests this window" is a number they can watch decrement.
- **Exclusions are named explicitly.** Moderator refusals, opt-in long-context, warm-up. Every hidden exclusion is a future debate; writing them makes the SLO defensible.
- **MLPerf is a reference frame, not a target.** The sanity check is "is our number in the right order of magnitude?", not "set the SLO equal to MLPerf's throughput."
- **Runbook in the second person.** "You are on-call. You just got paged." reads to a tired human; "the runbook consists of the following procedures" does not.
- **Owner and revisit date.** SLOs are revisited quarterly; without a revisit date, they decay. The owner is a role, not a person, so the doc survives a team re-org.

### 2h. Mapping to the exercise deliverables

| Exercise asks for | Section of this solution |
|---|---|
| SLI expression + justification | 2a |
| SLO target + window + error-budget quantity | 2b |
| Burn-rate alerts (fast + slow) | 2c |
| Levers table | 2d |
| Runbook stub | 2e |
| MLPerf / provider sanity check | 2f |

## 3. Validation steps

1. **SLI clauses map to real span fields.** Every clause reads an attribute that exists on the spans from [[exercise-01]] and [[exercise-04]]; query returns non-empty.
2. **Threshold matches the latency spec.** The SLI's TTFT threshold equals [[exercise-01]]'s "red" boundary. A mismatch is a hard failure.
3. **Error budget is a quantity, not a percentage.** Doc says "N bad requests per window," computed from a real 28-day query.
4. **Fast-burn alert fires on 14.4×.** Synthetic test: produce a 15-minute window where 15% of requests fail the SLI; the alert fires after the `for:` hold.
5. **Slow-burn alert does not fire on a 15-minute 15% blip.** Same synthetic test; the 6h window rolls through the blip; the slow-burn alert does not fire.
6. **Every lever names a tradeoff.** No row in the levers table reads "no cost" — the point of naming the lever is making the tradeoff explicit.
7. **Runbook is readable in under 90 seconds.** Measure on a non-expert reviewer. A runbook that takes longer loses to time pressure during a real incident.
8. **MLPerf citation is specific.** A link, a submission or dataset identifier, and a date.
9. **Owner and revisit date present.** Owner by role; next-review ≤ one quarter out.
10. **Alert rules are in a committed file.** `alerts/slo-burn.yaml` lives in the repo; no "configured directly in Grafana UI" implementations.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| SLI is a boolean over one span | 1 | ≥ 3 clauses, each justified; reads only span attributes. |
| SLI threshold matches the latency spec | 1 | Same number as exercise-01's red boundary. |
| SLO target is defensible | 1 | 99.0% or 99.5%; not 99.9% without strong reason. |
| Error budget is a quantity | 1 | Computed from a real 28-day query; stated in requests. |
| Fast-burn and slow-burn both defined | 1 | 14.4× / 1h / 5m hold and 6× / 6h / 15m hold. |
| Burn thresholds explained in-comment | 1 | Comment ties 14.4× to ~2-day and 6× to ~5-day burn-through. |
| Levers named with tradeoffs | 1 | Retry + hedge + fallback + degrade, each with cost or capability cost. |
| Runbook is written for an on-call at 3 AM | 1 | Second person; first-90-seconds section; dashboard + lever + escalation. |
| MLPerf (or provider serving) sanity check present | 1 | Specific citation; order-of-magnitude comparison explicit. |
| Owner + revisit date | 1 | Owner by role; next-review date named. |

A submission scoring < 6 is rejected. Hard failures: (a) SLI threshold invented independent of the latency spec, (b) only one burn window, (c) levers table without tradeoffs.

## 5. Common mistakes

- **Picking 99.9%.** For a first LLM-feature SLO, this is aspirational — provider tail variance eats the budget before any lever can fire.
- **Single-window burn alert.** 1 hour alone is false-positive-prone; 6 hours alone is too-late. The pair is the industry standard for a reason.
- **SLI that contradicts the latency spec.** Two sources of truth for "what good looks like" will drift; the SLI must read from the spec.
- **No exclusions.** Every SLO has edge cases that shouldn't count (warm-up, opt-in long-context, legitimate moderator refusals). Hiding them makes the budget noisier and the alert less trustworthy.
- **Error budget as a percentage.** "99%" is right but unactionable; "12,849 bad requests" is the number on-call watches.
- **Levers without tradeoffs.** "Enable hedge" with no cost attached is an instruction nobody wants to give at 3 AM. Name the cost.
- **Runbook written for the author.** A runbook is for someone less familiar than you, at a worse time than now, with less context than you have. Second person; literal paths; explicit dashboards.
- **MLPerf as SLO ceiling.** MLPerf is a reference frame for whether the model can physically run that fast. Setting the SLO to MLPerf's throughput is picking an unreachable target; the SLO belongs to the surface, not the model.
- **No `for:` hold on alerts.** Pages for every 5-minute blip erode trust in the alerting path faster than any other single bug.
- **Owner by name.** Reorganisations happen; an owner by role (`helpdesk-platform`) survives, an owner by name (`Jamie`) doesn't.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML04 (Model Denial-of-Service) is the top-of-mind risk a latency SLO defends against: an attacker who drives up tool round-trip time, or a tenant whose integration retries at 10× the median, consumes error budget without producing outcomes, and the burn-rate alert is the operational surface for that failure. The degrade lever is also the control that keeps the surface usable under adversarial load.
- MITRE ATLAS — <https://atlas.mitre.org/> — the SLI clauses engage ATLAS tactics directly: structural-validity failures at a sustained rate are a Collection / Model Evasion signal; retry-attempt distributions that shift are an Impact signal; the fast-burn alert is the detection point for both. The runbook's "lever → escalate" path is the response-plan artefact for Resource Hijacking consequences.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the SLO + burn-rate + levers + runbook stack is a worked example of the MEASURE → MANAGE loop: measurement produces the SLI stream, the SLO is the governance commitment, burn-rate alerts are the operational trigger, and the levers are the pre-authorised management actions. The quarterly revisit is a GOVERN-function accountability control.
- Google SRE Workbook, Chapter 5, "Alerting on SLOs" — <https://sre.google/workbook/alerting-on-slos/> — the multi-window multi-burn-rate pattern (the specific 14.4× / 1h and 6× / 6h numbers) originates here; verify the current chapter's wording before citing in the posture doc.
- MLPerf Inference — <https://mlcommons.org/benchmarks/inference-datacenter/> — the reference-frame for the sanity-check paragraph. Submissions are published per round; pick the most recent round with a comparable model/hardware pair. <!-- verify-before-publishing: cite the exact submission URL, round date, and tokens-per-second result used in the sanity-check paragraph above. -->

Related exercises: [[exercise-01]] (the TTFT and retry attributes the SLI reads); [[exercise-02]] (hit-ratio drop is an input to the slow-burn investigation); [[exercise-03]] (batch-path line-error ratio is a parallel SLO on the batch route); [[exercise-04]] (`app.outcome` and `app.cost.usd` are read by the degrade lever's CPSO delta); mod-108 [[exercise-01]] (the gateway is where the retry / hedge / fallback levers live); mod-108 [[exercise-02]] (fallback rules); mod-110 (incident response and postmortems that follow a fired alert).
