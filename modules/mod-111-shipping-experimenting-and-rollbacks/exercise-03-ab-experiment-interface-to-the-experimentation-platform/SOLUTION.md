### exercise-03: A/B Experiment Interface to the Experimentation Platform — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to wire the feature into an experimentation platform (GrowthBook, Statsig, Eppo, LaunchDarkly Experimentation, or equivalent) as an **interface exercise** — assignment call, exposure event, metric emission, guardrail registration. The deliverable is the *seam*, not the statistical analysis. What the applied engineer owns is that the interface is clean enough for a peer role (`senior-ml-engineer`, `model-evaluation-engineer`) to trust the platform's output when they run the analysis.

The failure modes the exercise defends against are specific to the *interface*, not the statistics:

1. **Assignment key does not match the metric's unit of analysis.** The metric is per-user (conversion within a session), but the SDK is assigning per-session. The experiment reports a false positive because the unit mismatch inflates the effective sample size. Fix: `assignment_key = user.id` and document *why* in `experiment-spec.md`.
2. **Exposure fires on the wrong boundary.** Firing on flag lookup inflates the denominator with users who never saw the variant (SRM); firing after a reload-triggered flag refresh double-counts. Fix: exposure fires **after the user receives the response**, exactly once per rendered response.
3. **Metrics derived from spans instead of emitted as events.** Span-derived metrics couple the experiment's primary metric to the tracing config — a sampler change in mod-110 [[exercise-02]] silently halves the metric. Fix: dedicated event emission to the platform's ingest endpoint.
4. **No guardrail metrics registered.** The team ships the "better conversion" variant and discovers a 40% cost regression two weeks later. The guardrails are the mechanism that forbids shipping-against-the-interests-not-measured-by-the-primary-metric.
5. **SRM read as a statistical result instead of as an interface bug.** The platform's SRM alarm fires with a 52/48 split; the team spends two days arguing about whether it's a real effect. Fix: in > 90% of SRM alarms the cause is an asymmetric exposure log. First look is at the exposure code, not the statistical model.
6. **Blurred boundary between what the applied engineer owns and what a peer role owns.** CUPED, sequential testing, HTE, delta-method on ratio metrics are peer-role territory. The applied engineer runs the interface and reads the platform's output; the peer role picks the method when the delta is non-obvious.

This solution provides:

- An **assignment SDK call** against GrowthBook (the reference free path), structured so swapping to Statsig or Eppo is one initialiser.
- An **exposure logger** that fires after the response is rendered, once per rendered response, and exactly zero times on error.
- A **per-turn metric emitter** that pushes primary + six secondary metrics + session roll-up to the platform's ingest endpoint.
- A **guardrail-metric registration** config (GrowthBook YAML export).
- An **SRM-drill protocol** that deliberately mis-wires exposure, observes the platform's SRM alarm, and documents the debugging.
- An **experiment-spec doc** with the mandatory fields.
- A **peer-role boundary note** naming the four families of work that live one seniority tier deeper.
- A **grading rubric** for the instructor.

## 2. Worked answer or implementation

### 2a. Assignment call (deliverable 1)

```python
# app/experiment.py — assignment via the experimentation platform SDK.
from __future__ import annotations

from dataclasses import dataclass

from growthbook import GrowthBook              # reference free path
# For Statsig: from statsig import statsig
# For Eppo:    from eppo_client import init, get_instance

@dataclass
class Assignment:
    experiment_id: str          # "v15_concise_vs_v14_2026q4"
    variant: str                # "control" | "treatment" (or a label the platform returns)
    assignment_id: str          # the platform's assignment token; correlates exposure to assignment


class ExperimentClient:
    """Thin wrapper over the platform SDK. One instance per process."""

    def __init__(self, *, api_host: str, client_key: str):
        self._gb = GrowthBook(
            api_host=api_host,
            client_key=client_key,
            # In-memory cache + control-plane update channel. GrowthBook calls
            # these "features" internally; the shape is identical to
            # [[exercise-01]]'s flag client.
            cache_ttl=60,
            enabled=True,
        )
        self._gb.load_features()

    def get_assignment(
        self,
        *,
        user_id: str,
        tenant_id: str,
        surface: str,
        locale: str,
    ) -> Assignment:
        """Return the variant for this user on the configured experiment.

        The assignment_key is user_id because the primary metric is a per-user
        metric (conversion-within-session). If the primary metric were a
        per-tenant metric (e.g. seat utilisation), we would key on tenant_id
        and document that explicitly in experiment-spec.md.
        """
        # Attributes the experiment's targeting rules reference.
        self._gb.set_attributes({
            "id":        user_id,       # GrowthBook's hashAttribute default
            "tenantId":  tenant_id,
            "surface":   surface,
            "locale":    locale,
        })
        exp = self._gb.get_experiment_result("v15_concise_vs_v14_2026q4")
        return Assignment(
            experiment_id="v15_concise_vs_v14_2026q4",
            variant=exp.variation_id,   # "0" = control, "1" = treatment, mapped below
            assignment_id=exp.key,      # platform's token for the result
        )
```

Test proving assignment stickiness:

```python
# tests/test_assignment.py
def test_assignment_is_sticky_for_a_user(experiment_client):
    assignments = {
        experiment_client.get_assignment(
            user_id="u-sticky-42", tenant_id="t", surface="helpdesk", locale="en"
        ).variant
        for _ in range(1000)
    }
    assert len(assignments) == 1, f"non-sticky: {assignments}"
```

### 2b. Exposure logging (deliverable 2)

```python
# app/exposure.py — fire exposure after the response is rendered.
import logging
import time
from dataclasses import dataclass

from growthbook import GrowthBook            # same instance as in experiment.py

log = logging.getLogger("exposure")

@dataclass
class Exposure:
    experiment_id: str
    variant: str
    user_id: str
    tenant_id: str
    surface: str
    timestamp_ms: int
    assignment_id: str


def record_exposure(
    *,
    gb: GrowthBook,
    experiment_id: str,
    variant: str,
    user_id: str,
    tenant_id: str,
    surface: str,
    assignment_id: str,
) -> None:
    """Fire exposure. Called from a `finally` block after response send,
    and ONLY when the response was successfully served."""
    event = {
        "experiment_id": experiment_id,
        "variant":       variant,
        "user_id":       user_id,
        "tenant_id":     tenant_id,
        "surface":       surface,
        "assignment_id": assignment_id,
        "timestamp_ms":  int(time.time() * 1000),
    }
    # GrowthBook's "trackingCallback" pattern — the platform ingests
    # exposures via its SDK event sink; for other platforms, replace with
    # statsig.log_event(...) or Eppo's exposure callback.
    gb.track_experiment(event)
    log.debug("exposure emitted: %s", event)
```

Turn-handler wiring — exposure fires once, after send, zero-on-error:

```python
# app/handler.py — exposure placement.
from app.experiment import ExperimentClient
from app.exposure import record_exposure

_exposure_fired_marker = "x-ai-experiment-exposure-fired"

def answer_turn(*, user_id, tenant_id, session_id, surface, locale, user_message, context):
    assignment = experiment_client.get_assignment(
        user_id=user_id, tenant_id=tenant_id, surface=surface, locale=locale
    )
    try:
        response = _run_llm_for_variant(assignment.variant, user_message, context)
        send_response_to_user(response)
    except Exception:
        # User did NOT experience the variant. Do not record exposure.
        raise
    else:
        # Idempotency guard — the turn may involve two LLM calls
        # (e.g. schema-reprompt), but the exposure is a per-rendered-response
        # event, not a per-LLM-call event.
        if not getattr(response, _exposure_fired_marker, False):
            record_exposure(
                gb=experiment_client._gb,
                experiment_id=assignment.experiment_id,
                variant=assignment.variant,
                user_id=user_id,
                tenant_id=tenant_id,
                surface=surface,
                assignment_id=assignment.assignment_id,
            )
            setattr(response, _exposure_fired_marker, True)
        return response
```

Tests:

```python
# tests/test_exposure.py
def test_exposure_fires_on_success(fake_platform, handler):
    handler.answer_turn(user_id="u-1", tenant_id="t", session_id="s",
                       surface="helpdesk", locale="en",
                       user_message="hi", context="")
    assert fake_platform.exposure_count == 1


def test_exposure_does_not_fire_on_error(fake_platform, handler, monkeypatch):
    monkeypatch.setattr(handler, "_run_llm_for_variant",
                        lambda *a, **kw: (_ for _ in ()).throw(RuntimeError("provider 500")))
    with pytest.raises(RuntimeError):
        handler.answer_turn(user_id="u-1", tenant_id="t", session_id="s",
                            surface="helpdesk", locale="en",
                            user_message="hi", context="")
    assert fake_platform.exposure_count == 0


def test_exposure_fires_exactly_once_on_multi_call_turn(fake_platform, handler_with_reprompt):
    """Turn does: LLM call -> schema validation failure -> LLM reprompt -> send."""
    handler_with_reprompt.answer_turn(user_id="u-1", tenant_id="t", session_id="s",
                                      surface="helpdesk", locale="en",
                                      user_message="hi", context="")
    assert fake_platform.exposure_count == 1
```

### 2c. Primary and secondary metric emission (deliverable 3)

```python
# app/metrics.py — emit primary + six secondary metrics per turn, and the
# session-level roll-up.
from __future__ import annotations

import time
from dataclasses import dataclass
from typing import Any

import httpx

@dataclass
class TurnMetrics:
    user_id: str
    tenant_id: str
    experiment_id: str
    variant: str
    assignment_id: str
    turn_id: str

    summary_positively_received: bool    # primary
    llm_cost_usd: float                  # secondary
    ttft_ms: int                         # secondary
    total_latency_ms: int                # secondary
    retry_count: int                     # secondary
    refused: bool                        # secondary
    moderation_blocked: bool             # secondary


_GROWTHBOOK_INGEST_URL = "https://gb.example.com/api/metrics/ingest"


async def emit_turn_metrics(m: TurnMetrics) -> None:
    """One call per turn. Pushes seven events, one per metric.

    Metrics are EMITTED, not derived from spans — span-derived metrics are a
    valid pattern for eng observability but couple the experiment result to
    the tracer config. Keep the two separate.
    """
    now = int(time.time() * 1000)
    base = {
        "user_id":       m.user_id,
        "tenant_id":     m.tenant_id,
        "experiment_id": m.experiment_id,
        "variant":       m.variant,
        "assignment_id": m.assignment_id,
        "turn_id":       m.turn_id,
        "timestamp_ms":  now,
    }
    events = [
        {**base, "metric": "summary_positively_received", "value": int(m.summary_positively_received)},
        {**base, "metric": "llm_cost_usd",                "value": m.llm_cost_usd},
        {**base, "metric": "ttft_ms",                     "value": m.ttft_ms},
        {**base, "metric": "total_latency_ms",            "value": m.total_latency_ms},
        {**base, "metric": "retry_count",                 "value": m.retry_count},
        {**base, "metric": "refused",                     "value": int(m.refused)},
        {**base, "metric": "moderation_blocked",          "value": int(m.moderation_blocked)},
    ]
    async with httpx.AsyncClient(timeout=1.0) as client:
        await client.post(_GROWTHBOOK_INGEST_URL, json={"events": events})


async def emit_session_metric(
    *, user_id: str, tenant_id: str, experiment_id: str, variant: str,
    session_id: str, session_resolved: bool,
) -> None:
    """Fired once at session close, keyed by the user's INITIAL assignment.

    This is the metric that lets the peer-role analysis compare session-level
    conversion when the primary metric is turn-level. Keyed by initial-arm
    so that late-session variant changes (shouldn't happen, but let's be
    defensive) do not flip the attribution.
    """
    await emit_turn_metrics_raw({
        "metric":        "session_resolved",
        "value":         int(session_resolved),
        "user_id":       user_id,
        "tenant_id":     tenant_id,
        "experiment_id": experiment_id,
        "variant":       variant,
        "session_id":    session_id,
        "timestamp_ms":  int(time.time() * 1000),
    })
```

The feedback-join step that produces `summary_positively_received` — a post-
hoc job reads the mod-110 [[exercise-03]] feedback store, joins to the
turn_id, and sets the boolean before emission. The exercise does not need
to implement the job; the schema shape is what the exercise asks for.

### 2d. Guardrail-metric registration (deliverable 4)

`docs/guardrails.yaml` — GrowthBook datasource-config export (equivalent
shape for Statsig / Eppo / LaunchDarkly):

```yaml
experiment: v15_concise_vs_v14_2026q4

metrics:
  primary:
    - key:        summary_positively_received_rate
      source:     summary_positively_received
      aggregation: mean_by_user
      unit:       user
      direction:  increase       # want it to go up

guardrails:
  - key:         refused_rate
    source:      refused
    aggregation: mean
    unit:        user
    direction:   decrease        # must not regress UP
    threshold:   p_value < 0.01 OR delta > 0.01
    action:      block_shipping

  - key:         moderation_blocked_rate
    source:      moderation_blocked
    aggregation: mean
    unit:        user
    direction:   decrease
    threshold:   delta > 0.005
    action:      block_shipping

  - key:         avg_llm_cost_usd
    source:      llm_cost_usd
    aggregation: mean
    unit:        turn
    direction:   decrease
    threshold:   delta > 0.0015           # +15% on a $0.01 baseline
    action:      block_shipping

  - key:         p95_total_latency_ms
    source:      total_latency_ms
    aggregation: p95
    unit:        turn
    direction:   decrease
    threshold:   delta > 300
    action:      block_shipping

  - key:         p95_retry_count
    source:      retry_count
    aggregation: p95
    unit:        turn
    direction:   decrease
    threshold:   delta > 0.5
    action:      block_shipping

  - key:         support_ticket_within_24h_rate
    source:      product.support_ticket_opened_within_24h   # product-side metric
    aggregation: mean_by_user
    unit:        user
    direction:   decrease
    threshold:   delta > 0.01
    action:      block_shipping
    note:        "Placeholder — wire to the product analytics source before experiment starts."
```

`action: block_shipping` is the platform's convention for a guardrail that, if breached at experiment-stop time, blocks the "ship treatment" button in the UI. This is the mechanism that converts "we have guardrails" into "we cannot ship against them."

### 2e. SRM-drill protocol (deliverable 5)

```markdown
# docs/srm-incident.md — Deliberate SRM Drill

## Goal
Deliberately mis-wire exposure to introduce a sample-ratio mismatch,
observe the platform's alarm, and record the debugging trail.

## The mis-wire (intentional)

In `app/handler.py`, change exposure to only fire when the LLM call did
not retry. This is a realistic bug — an engineer might argue they want
"clean" exposures that are not contaminated by retry-path edge cases.

```diff
- record_exposure(gb=..., experiment_id=..., variant=..., ...)
+ if retry_count == 0:
+     record_exposure(gb=..., experiment_id=..., variant=..., ...)
```

## Driving traffic

`scripts/drive_traffic.py --rps 2 --duration 100m --split 50/50`
produces 2000 turns on a nominal 50/50 split.

Because the two prompt variants have different retry rates
(the v15_concise prompt produces shorter outputs that fail a JSON-length
check less often, so its retry rate is lower), the exposure filter
preferentially drops control (v14) exposures — the two arms end up with
asymmetric exposure counts.

## Observed split percentages

- Platform's exposure report at 1h: control 45.2%, treatment 54.8%.
  Expected 50.0% / 50.0%.
- Platform's chi-squared p-value: < 1e-6 (fires SRM).

## Platform's SRM alert

GrowthBook UI shows a red banner on the experiment card:
"Sample Ratio Mismatch detected (chi2 p < 0.000001). Review the exposure
pipeline before interpreting this experiment."

Alert webhook (configured on the experiment) posts to #ai-eng-alerts:
"[Experiment SRM] v15_concise_vs_v14_2026q4 — control 45.2% vs expected 50%.
Investigate exposure wiring before reading results."

## Debugging trail

1. First look: the exposure code. Reviewed `app/handler.py` and `app/exposure.py`.
2. Spotted the `if retry_count == 0:` guard.
3. Confirmed hypothesis by replaying 100 turns in a dev environment and
   logging the exposure count per variant — treatment fired 100% of calls,
   control fired only 80% (20% had non-zero retry).
4. Confirmed retry-rate correlated with variant: control v14 produced
   wrong-schema outputs at 4%; v15_concise at 0.5%. That's the structural
   asymmetry.

## Fix

Remove the guard. Exposure fires on every rendered response regardless of
retry count. Add a test (one of the three in 2b) asserting exposure fires
on a multi-LLM-call turn.

## Re-launch

- Declared the current experiment window invalid at 15:30 UTC.
- Reset the experiment start timestamp in the platform UI (GrowthBook: a
  one-click action that preserves the config but clears the results).
- Began a new window with the fix in production. Confirmed exposure split
  at 1h: 50.1% / 49.9%, p = 0.78 (no SRM).

## The lesson

An SRM alarm is a **bug report on the interface**, not a statistical
result. In > 90% of real SRM alarms, the cause is asymmetric exposure.
First look is always at the exposure code.
```

### 2f. Experiment spec doc (deliverable 6)

```markdown
# docs/experiment-spec.md — v15_concise_vs_v14_2026q4

## Hypothesis

Users on `v15_concise` are more likely than users on `v14` to positively
receive the response (thumbs-up or surface's equivalent conversion event)
within 30 minutes of the turn, holding guardrail metrics (refusal rate,
moderation FP rate, cost, latency, retry, 24h support-ticket rate) at
non-regressed levels.

## Primary metric

`summary_positively_received_rate` — mean of per-turn boolean, aggregated
per user, over the experiment window.

## Unit of assignment

`user_id`. The metric is per-user (a user can have multiple turns; we take
the mean over their turns). Per-session would inflate effective N and
produce false positives on a correlated-trial dataset.

## Traffic split

50 / 50. The feature has sufficient traffic that balanced power is cheap.

## Duration

14 days. Calibrated at feature's ~170k turns/day × 50% per arm × 14 days =
~1.19M exposures per arm. At baseline p=0.08 for the primary metric and
MDE of 0.5 pp, `n ≈ 16 × 0.08 × 0.92 / 0.005² = 47 104` per arm. 14 days
delivers 25× that; the excess buys variance-reduction margin and lets us
inspect per-segment effects.

## Minimum detectable effect (MDE) on the primary metric

0.5 pp on `summary_positively_received_rate` (baseline 8.0%, so Δ relative
~6%). Calculation sanity-checked with `senior-ml-engineer` (peer-role
10-minute consult on 2026-11-09).

## Guardrail metrics and thresholds

(See `docs/guardrails.yaml` for the config-as-code version.)

- `refused_rate`: block ship if delta > +1 pp or p < 0.01 and direction up.
- `moderation_blocked_rate`: block ship if delta > +0.5 pp.
- `avg_llm_cost_usd`: block ship if delta > +15%.
- `p95_total_latency_ms`: block ship if delta > +300 ms.
- `p95_retry_count`: block ship if delta > +0.5.
- `support_ticket_within_24h_rate`: block ship if delta > +1 pp.

## Stopping rules

Fixed-horizon: 14 days. **No peeking** — the primary-metric test is run at
Day 14 with a two-sided α = 0.05. Early-stopping requires a sequential
testing method; if we want it, loop in `senior-ml-engineer` (peer role) to
pick the method (Always-Valid Inference, Group-Sequential, mSPRT, etc.).

## Analysis owner

Applied engineer runs the *interface*: assignment, exposure, metric
emission, guardrail registration, SRM check. `senior-ml-engineer` runs the
analysis when the delta is non-obvious (CUPED variance reduction, HTE
analysis, delta method on the ratio metric, sequential testing).
```

### 2g. Peer-role boundary note (deliverable 7)

```markdown
# docs/peer-role-boundary.md — When to loop in a peer role

Four families of experimentation work live one seniority tier deeper than
applied-AI engineer. They are peer-role territory (`senior-ml-engineer` or
`model-evaluation-engineer`). For this experiment, the specific question
that would trigger each is:

## 1. Variance reduction (CUPED)

CUPED uses a pre-experiment covariate to shrink the primary-metric
variance and narrow the confidence interval. Trigger: the Day-14 result is
statistically inconclusive (p-value in (0.05, 0.15)) and the MDE we would
need to bring the result inside α is tight on current N. The pre-treatment
`summary_positively_received_rate` over the 14 days *before* the
experiment started is a natural covariate.

Why peer role: CUPED misapplication inflates false-positive rates. The
covariate must be measured *before* assignment and uncorrelated with the
treatment. `senior-ml-engineer` picks the covariate and the regression
form.

## 2. Sequential testing (early stopping)

Any time the team asks "can we stop at Day 7 if the result is already
significant?" the answer is "not with a fixed-horizon test." A sequential
method (Always-Valid Inference, Group-Sequential, mSPRT) preserves the
type-1 error rate under peeking. Trigger: product wants to ship faster
than the fixed horizon.

Why peer role: sequential methods have method-specific assumptions and
require upfront choice (not post-hoc). Picking the wrong method is
indistinguishable from p-hacking.

## 3. Heterogeneous treatment effects (HTE / segment-level)

The aggregate primary metric moves +0.5 pp but the per-tier breakdown
shows +2 pp on enterprise and -0.5 pp on free. The ship decision depends
on whether the enterprise effect is real or multiple-testing noise.
Trigger: product asks "is the effect bigger for enterprise?"

Why peer role: multiple-comparison corrections across tenant tiers,
locales, surfaces, and the interaction terms require a method choice
(Bonferroni, Benjamini-Hochberg, Lin's adjustment, HTE forest). The
applied-AI engineer provides the data slices; the peer role picks the
method.

## 4. Delta-method on ratio metrics

The primary metric is a ratio (`positively_received / turns_served`). The
naive variance estimator ignores the correlation between numerator and
denominator. Trigger: the team debates "do we average per-user or per-turn
when turns-per-user varies by variant?" This is the ratio-metric problem.

Why peer role: the delta method is textbook statistics, not a new
invention, but applying it correctly in the platform's output requires
knowing *which* column in the result uses it. `senior-ml-engineer` reads
the platform's documentation and checks.

## Rule of thumb

Applied engineer answers questions of the shape "is the interface
correct?" Peer role answers questions of the shape "is the method
appropriate?" If the answer to the first is "yes" and the answer to the
second is "I'm not sure," the next action is a 20-minute peer-role
consult, not an all-afternoon textbook refresher.
```

### 2h. Mapping to the exercise deliverables

| Exercise asks for | Section |
|---|---|
| Assignment call with user_id key + stickiness test | 2a |
| Exposure logging + three exposure tests | 2b |
| Primary + secondary metric emission | 2c |
| Guardrail metric registration | 2d |
| SRM drill with intentional mis-wire | 2e |
| Experiment spec doc | 2f |
| Peer-role boundary note | 2g |

## 3. Validation steps

1. **Assignment is sticky.** 1000 `get_assignment` calls for the same `user_id` return the same variant.
2. **Exposure fires once per rendered response.** Three tests pass: success-fires, error-does-not-fire, multi-LLM-call-still-fires-once.
3. **Metrics emit the expected event shape.** `summary_positively_received` + six secondary metrics + session roll-up, each with `user_id`, `experiment_id`, `variant`, `timestamp_ms`.
4. **Guardrail metrics are registered in the platform.** Screenshot or `docs/guardrails.yaml` shows all six with threshold and `block_shipping` action.
5. **SRM drill fires the platform's SRM alarm.** Platform shows red banner + alert webhook posts; `docs/srm-incident.md` documents the observed split, the debug trail, and the fix.
6. **Experiment-spec doc names hypothesis, primary metric, assignment unit, split, duration, MDE, guardrails, stopping rules.** No TBDs.
7. **Peer-role boundary note names four families and the specific triggering question in this experiment.**
8. **No prompt text in a flag / platform config.** The experiment variant maps to `prompt_id`; prompt text lives in the registry from [[exercise-01]].
9. **Metrics are emitted, not derived from spans.** Grep the ingest URL calls in the code; no cron job reads traces to produce experiment metrics.
10. **The experimentation platform treats the primary metric as per-user, not per-turn.** Config shows `unit: user` for the primary metric.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Assignment uses `user_id` and stickiness test passes | 1 | 1000-call stickiness test present and passing; the `user_id` choice is documented. |
| Exposure fires after render, once, zero-on-error | 2 | All three tests pass; the `finally`-block placement is explicit. |
| Primary + six secondary + session metric all emitted | 1 | Grep shows calls for each; event shapes include `experiment_id`, `variant`, `timestamp_ms`. |
| Guardrail metrics registered in platform config-as-code | 1 | `docs/guardrails.yaml` or UI screenshot; all six present with thresholds. |
| SRM drill is executed and documented | 2 | Intentional mis-wire → platform fires SRM → debug trail → fix → re-launch, all in `docs/srm-incident.md`. |
| Experiment spec doc has all fields | 1 | Hypothesis, primary metric, assignment unit, split, duration, MDE, guardrails, stopping rules. |
| Peer-role boundary note names four families with triggers | 1 | CUPED, sequential testing, HTE, delta method — each with a specific question from this experiment. |
| Metrics are emitted, not span-derived | 1 | No cron job reads traces to produce experiment metrics. |

A submission scoring < 6 is rejected. Hard failures: (a) assignment key does not match the metric's unit of analysis, (b) exposure fires on flag lookup instead of after render, (c) no SRM drill executed, (d) guardrail metrics absent from the platform.

## 5. Common mistakes

- **Firing exposure on assignment call instead of on render.** Inflates the denominator with users who were assigned but never saw the variant. Breaks the SRM check on any real feature.
- **Using `session_id` as the assignment key when the metric is per-user.** Mis-matched unit of analysis. `senior-ml-engineer` will point this out in the first review; fix it before.
- **Deriving metrics from spans.** Span-derived metrics couple the experiment result to the tracer config. A sampler change in mod-110 [[exercise-02]] now silently halves the primary metric.
- **No guardrails.** Shipping the "better conversion" variant and discovering a 40% cost regression two weeks later is a classic. Guardrails are a precommitment, not a monitoring convenience.
- **Reading an SRM alarm as a statistical result.** The interface is the bug. First look at the exposure code.
- **Peeking at results before Day 14.** With a fixed-horizon test, early looks inflate the type-1 error rate. If you want early stopping, use a sequential method and pick it before the experiment starts.
- **Running the applied engineer as the primary analyst.** The applied engineer runs the interface. The peer role runs the analysis. Blurring the boundary leaves both artefacts unreviewed.
- **Testing exposure in isolation without the error path.** The error-path test is where > 90% of SRM alarms would originate. Test it first.
- **No idempotency on multi-LLM-call turns.** A schema-reprompt turn fires exposure twice — a dedupe-by-request-id discipline must run in the metric emitter too, not only in the exposure logger.
- **Secondary metric list without a guardrail registration.** Emitting `llm_cost_usd` but not registering `avg_llm_cost_usd` as a guardrail means the cost regression does not block shipping — the ship button stays green.

## 6. References

- GrowthBook documentation — <https://docs.growthbook.io/> — the assignment, exposure, metric, and guardrail concepts used throughout. The datasource config format in 2d is adapted from GrowthBook's experiment config schema.
- OpenFeature specification — <https://openfeature.dev/specification/> — the assignment client in 2a layers on top of the OpenFeature flag client from [[exercise-01]]; the `EvaluationContext` → assignment relationship is the common substrate across platforms.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the experiment-spec doc (2f) and guardrail-metric registration (2d) are the MEASURE-and-MANAGE artefacts for an AI change: the hypothesis and primary metric encode what is being measured; the guardrails encode what the team will NOT tolerate; the stopping rule encodes when the measurement ends. Without these artefacts, "we A/B tested it" is unaccountable.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML09 (Output Integrity) and ML01 (Input Manipulation) guardrails in 2d are the surface at which an experiment can be forbidden from shipping a variant that would increase attack surface. The `moderation_blocked_rate` and `refused_rate` guardrails are the controls that prevent an output-integrity regression from reaching GA.
- MITRE ATLAS — <https://atlas.mitre.org/> — the SRM-as-interface-bug framing (2e) parallels the detection-vs-response distinction in ATLAS: an alarm is a trigger to investigate, not a conclusion. Applying the same rule to experimentation alarms keeps the team from treating SRM as a p-value.

Related exercises: [[exercise-01]] (the flag layer the assignment client layers on; the `targeting_key` discipline is shared); [[exercise-02]] (the gate math in the launch plan is the interface-side cousin of the experimentation-platform's primary-metric math; the shadow report's per-scenario breakdown parallels the HTE question that triggers peer-role involvement here); [[exercise-04]] (the rollback runbook the on-call reads when a guardrail trips at ship time); mod-109 [[exercise-04]] (CPSO ledger that produces `llm_cost_usd`); mod-110 [[exercise-03]] (feedback pipeline that feeds `summary_positively_received`); mod-106 (scenario taxonomy the HTE breakdown indexes into).
