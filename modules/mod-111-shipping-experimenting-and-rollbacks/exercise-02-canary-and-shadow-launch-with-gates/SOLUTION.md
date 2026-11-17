### exercise-02: Canary and Shadow Launch with Pre-Registered Gates — Solution

## 1. Solution overview

The exercise takes the flag layer from [[exercise-01]] and layers on **gated rollout**: a shadow launch of one prompt version and a canary rollout of another, each governed by a written launch-plan doc with four gate families (quality / safety / cost / latency) frozen *before* the flag flip. The deliverable is not "flip a flag then check dashboards." It is the **launch-plan doc**, the **per-stage promotion artefacts**, and the **auto-rollback rule** that make the launch decision reproducible six weeks from now when someone is auditing why `v15_concise` went to 100% on 2026-11-14.

The failure modes the exercise defends against are specific and cheap to describe (and expensive to live through):

1. **Aggregate-looks-fine, segment-is-broken.** A composite score averaged over all tenants hides a 6-pp regression that hits a 10%-of-traffic segment. The fix is **per-scenario / per-segment floors** in the launch plan.
2. **Plan drift during observation.** Mid-canary, someone moves a threshold because "the number was too strict." The plan stops being a precommitment and becomes a rationalisation. The fix is **freeze on commit and tag**; any later change is a `plan-revision-NN` note, not a silent edit.
3. **Shadow in-line with the request path.** A shadow that back-pressures the user response is worse than no shadow (adds latency, doubles cost, destabilises the primary variant). The fix is **async off the critical path with a bounded queue and drop-on-overflow**.
4. **Shadow spans conflated with primary.** The CPSO ledger counts shadow cost against the primary feature; the quality judge counts shadow responses in the aggregate. The fix is **distinct `app.feature` + `app.shadow.*` attributes** on shadow spans.
5. **Rehearsal-free auto-rollback.** A fast-burn alert rule that has never fired will not fire correctly when it matters. The fix is **a scripted drill that fires the rule synthetically and records the measured time-to-effect.**
6. **"Promotion decision" as a Slack message.** Six weeks later no one can reconstruct whether 25% → 100% was promoted on the primary metric or on cost. The fix is the **per-stage promotion artefact** — the same template filled in for each stage.

This solution provides:

- A **launch-plan doc** with four gate families, sample-size math, auto-rollback rules, and a rollback protocol — frozen before flag flip.
- A **shadow-launch wiring pattern** with async off-critical-path execution, bounded queue, drop-on-overflow counter, and the `app.shadow.*` attribute contract.
- A **shadow-report doc** that performs per-scenario breakdown (the artefact that catches the enterprise-tier failure mode).
- A **canary-log template** with the per-stage promotion block.
- A **fast-burn / slow-burn alert pair** (Prometheus + Alertmanager shape) with an auto-rollback action hitting the flag provider's API.
- A **rollback drill protocol** and the expected measurements.
- A **grading rubric** for the instructor.

## 2. Worked answer or implementation

### 2a. Launch-plan doc (deliverable 1 — frozen before flag flip)

```markdown
# Launch Plan — support_answer_v15_concise

- Variant IDs: control = `support_answer_v14`; candidate = `support_answer_v15_concise`.
- Launch owner: applied-ai-team (jferguson on-call this rotation).
- Tracking issue: PROD-4821.
- Plan freeze: 2026-11-10 09:00 UTC. **This doc is immutable after freeze.**
  Any threshold change requires a `plan-revision-NN.md` with a timestamp
  and a rationale; the original doc is preserved in git history.

## Sequencing

- Shadow at 10% for 3 days. (Low-rate feature: reduce to 36h; see sample-size math.)
- Canary 1% for 24h.
- Canary 5% for 72h.
- Canary 25% for 7 days.
- GA at 100%.

Reasoning for the sequence: the enterprise-tier regression that motivated
this exercise (a 6-pp drop on 10% of traffic that was invisible in the
aggregate) tells us the smallest cohort that reveals a segment-specific
problem is roughly the segment size. 1% and 5% stages confirm the aggregate
is fine; 25% is where a segment-only regression is statistically detectable
with per-segment gates.

## Gate families

### Quality
- Offline eval composite ≥ v14 - 2 pts.
- Per-scenario floor ≥ v14 - 5 pts on every scenario in the mod-106 taxonomy.
- Shadow judge win-rate ≥ 55% (fine-tune vs base on paired turns).
- Thumbs-down delta ≤ +0.5 pp (per mod-110 [[exercise-03]]).

### Safety
- Refusal-rate delta ≤ +1.0 pp.
- Moderation false-positive delta ≤ +0.5 pp.
- Injection-eval failures: zero *new* cases on the frozen set (mod-107 [[exercise-03]]).

### Cost
- CPSO delta ≤ +10%.
- Absolute daily spend on canary cohort ≤ $75 (= $0.012 avg cost-per-turn × ~6000 turns/day at 5% × 24h at 500 rpm).

### Latency
- TTFT p95 delta ≤ +150 ms.
- Total p95 delta ≤ +300 ms.
- Retry-rate delta ≤ +0.5 pp.

## Sample-size math

Rate-metric heuristic: `n ≈ 16 × p × (1 - p) / Δ²` per arm.

| Metric | Baseline p | Smallest Δ that matters | n per arm |
|---|---|---|---|
| Thumbs-down | 0.03 | 0.5 pp | 18 625 |
| Refusal     | 0.02 | 1.0 pp | 3 136 |
| Moderation FP | 0.005 | 0.5 pp | 3 184 |
| Retry rate  | 0.04 | 0.5 pp | 24 576 |

Observation-window check: at a steady 2 rps the feature emits ~172 800
turns/day. At 1% canary that is 1 728/day; at 5% canary 8 640/day; at 25%
canary 43 200/day. Thumbs-down and retry rate need the 25% stage to land
inside window; refusal / moderation-FP are decidable at 1% within 24h.
This is why the 25% stage is the first with a full set of decidable gates.

## Auto-rollback rule (hard vs soft)

- **Hard gates — auto-rollback** (flip `support_agent.prompt_id` back to `v14`):
  - Refusal-rate delta > +2.0 pp over 5 min.
  - Moderation-FP delta > +2.0 pp over 5 min.
  - New injection-eval failure on the fast-eval subset.

- **Soft gates — page only** (notify on-call, let them decide):
  - CPSO delta > +15% over 30 min.
  - TTFT p95 delta > +300 ms over 30 min.
  - Thumbs-down delta > +1.0 pp over 2h.

Burn-rate semantics: a fast-burn is (short-window p99.9 vs long-window
baseline) > 10× budget consumption (per mod-109 [[exercise-05]]). The
numbers above are the equivalent delta-over-window formulations for metrics
that are not naturally "budget-shaped."

## Rollback protocol

1. The alert fires and routes to the on-call pager (PagerDuty).
2. On-call acks within 5 min (TTD measurement boundary).
3. On-call runs the [[exercise-04]] runbook: flip `support_agent.prompt_id` to
   `support_answer_v14` via the flag-provider API.
4. Verify in the trace UI that spans within 60 seconds of the flip carry the
   old `prompt_id`. Target TTE ≤ 60 s ([[exercise-04]] SLA).
5. Post a brief notification in #ai-feature-support with the flag diff and
   the metric that tripped.
6. Schedule a retro within 24h.
```

### 2b. Shadow-launch wiring (deliverable 2)

```python
# app/shadow.py — fire the shadow variant asynchronously off the request path.
import asyncio
import logging
import time
from dataclasses import dataclass

from observability.span import llm_span, LLMCallContext

log = logging.getLogger("shadow")

# Bounded queue; drop-on-overflow with a counter the caller can scrape.
_shadow_queue: asyncio.Queue = asyncio.Queue(maxsize=256)
shadow_drops = 0


@dataclass
class ShadowRequest:
    control_variant: str         # e.g. "support_answer_v14"
    shadow_variant: str          # e.g. "support_answer_v15_concise"
    model_id: str
    feature: str                 # e.g. "support_agent"
    tenant_id: str
    user_id: str
    session_id: str
    request_id: str              # correlation back to the primary span
    prompt_rendered: str


def enqueue_shadow(req: ShadowRequest) -> None:
    """Non-blocking enqueue. On overflow, drop and increment the counter.
    Called from the request path; must not raise, must not block."""
    global shadow_drops
    try:
        _shadow_queue.put_nowait(req)
    except asyncio.QueueFull:
        shadow_drops += 1
        log.warning("shadow queue full; dropping shadow for request_id=%s", req.request_id)


async def shadow_worker() -> None:
    """Long-running worker. Reads the queue and fires the shadow call.
    One or more workers can run; concurrency is bounded by queue+worker count."""
    while True:
        req = await _shadow_queue.get()
        try:
            await _fire_shadow(req)
        except Exception:
            # Errors in the shadow NEVER affect the user. Just log.
            log.exception("shadow call failed; swallowing")
        finally:
            _shadow_queue.task_done()


async def _fire_shadow(req: ShadowRequest) -> None:
    ctx = LLMCallContext(
        system="openai",
        operation="chat",
        request_model=req.model_id,
        # IMPORTANT: a distinct app.feature so the CPSO ledger does not
        # double-count shadow spend against the primary feature.
        feature=f"{req.feature}.shadow",
        tenant_id_raw=req.tenant_id,
        session_id=req.session_id,
        user_id_raw=req.user_id,
        request_id=req.request_id,
    )
    with llm_span(ctx, hash_salt=HASH_SALT) as s:
        s._span.set_attribute("app.shadow.of", req.control_variant)
        s._span.set_attribute("app.shadow.variant", req.shadow_variant)
        s._span.set_attribute("app.experiment.arm", req.shadow_variant)
        completion = await call_provider_async(req.model_id, req.prompt_rendered, span=s)
        s.record_completion_event(completion)
```

Primary turn-handler integration — the shadow is enqueued *after* the
response has been returned to the user:

```python
# app/handler.py — hooking the shadow into the turn handler.
import random

from app.flags import choose_variant
from app.shadow import enqueue_shadow, ShadowRequest

def answer_turn(**kw) -> str:
    v = choose_variant(**{k: kw[k] for k in ("user_id","tenant_id","session_id",
                                             "surface","locale","internal")})
    ...
    completion = _run_primary(v, user_message=kw["user_message"], context=kw["context"])

    # Shadow: only if the shadow flag is on, and only every N-th turn.
    if _shadow_enabled() and random.random() < _shadow_rate():
        enqueue_shadow(ShadowRequest(
            control_variant=v.prompt_id,
            shadow_variant=_shadow_variant_id(),
            model_id=v.model_id,
            feature="support_agent",
            tenant_id=kw["tenant_id"],
            user_id=kw["user_id"],
            session_id=kw["session_id"],
            request_id=current_request_id(),
            prompt_rendered=_render(_shadow_variant_id(), kw),
        ))

    return completion
```

Test proving the shadow does not block the primary path:

```python
# tests/test_shadow_isolation.py
import asyncio
import time
import pytest

from app.shadow import _shadow_queue, enqueue_shadow, ShadowRequest, shadow_worker


@pytest.mark.asyncio
async def test_shadow_does_not_block_primary_even_on_10s_shadow_latency(monkeypatch):
    slept_for = []

    async def slow_provider(*a, **kw):
        slept_for.append(time.perf_counter())
        await asyncio.sleep(10)
        return "shadow-response"

    monkeypatch.setattr("app.shadow.call_provider_async", slow_provider)
    worker = asyncio.create_task(shadow_worker())

    t0 = time.perf_counter()
    enqueue_shadow(ShadowRequest(
        control_variant="v14", shadow_variant="v15c", model_id="gpt-4o-mini",
        feature="support_agent", tenant_id="t", user_id="u", session_id="s",
        request_id="req", prompt_rendered="x",
    ))
    t_enqueue = time.perf_counter() - t0
    assert t_enqueue < 0.001, f"enqueue blocked for {t_enqueue:.3f}s"

    await asyncio.sleep(0.1)   # let the worker start the slow call
    assert len(slept_for) == 1, "shadow should be in-flight by now"

    worker.cancel()


def test_shadow_drops_on_queue_full(monkeypatch):
    for _ in range(300):          # queue maxsize is 256
        enqueue_shadow(ShadowRequest(
            control_variant="v14", shadow_variant="v15c", model_id="gpt-4o-mini",
            feature="support_agent", tenant_id="t", user_id="u", session_id="s",
            request_id="req", prompt_rendered="x",
        ))
    from app import shadow
    assert shadow.shadow_drops >= 44, f"drops counter should have advanced: got {shadow.shadow_drops}"
```

### 2c. Shadow-report doc (deliverable 3)

```markdown
# Shadow Report — support_answer_v15_concise vs v14

- Window: 2026-11-10 10:00 → 2026-11-13 10:00 UTC (3 days).
- Shadow rate: 10% of non-internal turns.
- Samples: 25 870 shadowed turns; same 25 870 control turns paired by
  request_id.
- Gate reference: launch-plan freeze 2026-11-10 09:00.

## Aggregate quality

- mod-106 LLM-judge win-rate (shadow vs control): 57.3% (± 0.6 pp @ 95 CI).
  **Gate: ≥ 55%. PASS.**
- Thumbs-down proxy: shadow would have produced a 3.1% rate vs control 3.0%.
  **Gate: delta ≤ +0.5 pp. PASS.**

## Per-scenario quality

| Scenario (mod-106) | Control p | Shadow p | Δ pp | Gate |
|---|---|---|---|---|
| `tier:enterprise / qtype:how-to` | 4.8 | 3.6 | **-1.2** | (within -5 floor; PASS) |
| `tier:enterprise / qtype:troubleshoot` | 4.5 | 4.4 | -0.1 | PASS |
| `tier:smb / qtype:how-to` | 4.6 | 4.7 | +0.1 | PASS |
| `tier:smb / qtype:troubleshoot` | 4.3 | 4.4 | +0.1 | PASS |
| `tier:free / qtype:how-to` | 4.1 | 4.2 | +0.1 | PASS |

The per-scenario floor of **v14 - 5 pts** is NOT breached on any scenario.
Note the -1.2 on enterprise/how-to — within gate but worth watching; this is
the segment that the aggregate-hid-regression that motivated the exercise
would have reproduced here if we had no per-scenario gate.

## Cost

- Shadow CPSO: $0.0121 (± $0.0003) vs control $0.0117.
  **Gate: delta ≤ +10%. Observed +3.4%. PASS.**

## Latency (informational — shadow is async)

- Shadow p95 total: 2.9 s (control 2.7 s).
- Shadow's p95 is the number we use to size the canary's expected user-
  perceived p95; we do not gate shadow on latency.

## Decision

All quality, safety, and cost gates PASS. Promote to the canary stages per
the launch-plan sequencing. Flag flip scheduled for 2026-11-13 14:00 UTC.
```

### 2d. Canary log template (deliverable 4)

```markdown
# Canary Log — v15_concise

## Canary Promotion — v15_concise at 0% → 1%
Observation window: 2026-11-13 14:00 → 2026-11-14 14:00 UTC (24h).
Samples: canary 1 728 turns; control 170 848 turns.

| Gate | Threshold | Observed | Status |
|---|---|---|---|
| Offline eval (composite) | ≥ v14 - 2 pts | 4.42 vs 4.47 | PASS |
| Offline eval (worst scenario) | ≥ v14 - 5 pts | 3.6 (enterprise/how-to) | PASS |
| Thumbs-down delta | ≤ +0.5 pp | +0.2 pp | PASS |
| Refusal delta | ≤ +1.0 pp | +0.1 pp | PASS |
| Moderation FP delta | ≤ +0.5 pp | +0.0 pp | PASS |
| CPSO delta | ≤ +10% | +3.6% | PASS |
| Daily spend on canary | ≤ $75 | $21.40 | PASS |
| TTFT p95 delta | ≤ +150 ms | +60 ms | PASS |
| Retry delta | ≤ +0.5 pp | +0.0 pp | PASS |

Decision: PROMOTE to 5%. Signed off by: jferguson (on-call).

## Canary Promotion — v15_concise at 1% → 5%
Observation window: 2026-11-14 14:00 → 2026-11-17 14:00 UTC (72h).
Samples: canary 25 920; control 492 480.

| Gate | Threshold | Observed | Status |
|---|---|---|---|
| Offline eval (composite) | ≥ v14 - 2 pts | (re-run) 4.41 vs 4.47 | PASS |
| Thumbs-down delta | ≤ +0.5 pp | +0.3 pp | PASS |
| Refusal delta | ≤ +1.0 pp | +0.2 pp | PASS |
| Moderation FP delta | ≤ +0.5 pp | +0.1 pp | PASS |
| CPSO delta | ≤ +10% | +3.9% | PASS |
| Daily spend on canary | ≤ $75 | $72.10 | PASS (at limit; watch) |
| TTFT p95 delta | ≤ +150 ms | +90 ms | PASS |
| Retry delta | ≤ +0.5 pp | +0.0 pp | PASS |

Decision: PROMOTE to 25% with a note that daily spend on canary is near
threshold — 25% stage will exceed the configured $75/day; update the
threshold to $400/day in `plan-revision-01.md` before flipping, or cap the
25% stage at 20% until cost drops. Signed off by: jferguson.

## Canary Promotion — v15_concise at 5% → 25%
(fill after window)

## Canary Promotion — v15_concise at 25% → 100%
(fill after window)
```

### 2e. Fast-burn / slow-burn alert pair (deliverable 5)

`alerts/canary-burn.yaml` — Prometheus Alertmanager shape:

```yaml
groups:
  - name: support_agent-canary
    rules:
      # ---- HARD GATES — auto-rollback ----
      - alert: SupportAgent_Refusal_FastBurn
        # Fast-burn: 5-minute delta of refusal rate > 2 pp between canary and control.
        expr: |
          (
            sum(rate(app_llm_refused_total{app_feature="support_agent", app_experiment_arm="support_answer_v15_concise"}[5m]))
            / sum(rate(app_llm_calls_total{app_feature="support_agent", app_experiment_arm="support_answer_v15_concise"}[5m]))
          ) - (
            sum(rate(app_llm_refused_total{app_feature="support_agent", app_experiment_arm="support_answer_v14"}[5m]))
            / sum(rate(app_llm_calls_total{app_feature="support_agent", app_experiment_arm="support_answer_v14"}[5m]))
          ) > 0.02
        for: 5m
        labels:
          severity: page
          action: auto_rollback
        annotations:
          summary: "Refusal rate on v15_concise is +2 pp over v14 for 5m — auto-rollback firing."
          runbook: "https://wiki.example.com/applied-ai/rollback-support_agent"

      - alert: SupportAgent_ModerationFP_FastBurn
        expr: |
          (mod_fp_rate{arm="support_answer_v15_concise"}
            - mod_fp_rate{arm="support_answer_v14"}) > 0.02
        for: 5m
        labels:
          severity: page
          action: auto_rollback
        annotations:
          summary: "Moderation FP delta > +2 pp — auto-rollback."

      # ---- SOFT GATES — page only ----
      - alert: SupportAgent_CPSO_SlowBurn
        # 30-minute window. CPSO regression at +15% pages the on-call.
        expr: |
          (cpso_usd{arm="support_answer_v15_concise"} / cpso_usd{arm="support_answer_v14"} - 1) > 0.15
        for: 30m
        labels: { severity: page, action: notify }
        annotations:
          summary: "v15_concise CPSO is +15% over v14 for 30m — on-call to review."

      - alert: SupportAgent_TTFT_SlowBurn
        expr: |
          histogram_quantile(0.95, sum(rate(ttft_ms_bucket{arm="support_answer_v15_concise"}[30m])) by (le))
          - histogram_quantile(0.95, sum(rate(ttft_ms_bucket{arm="support_answer_v14"}[30m])) by (le))
          > 300
        for: 30m
        labels: { severity: page, action: notify }

      - alert: SupportAgent_ThumbsDown_SlowBurn
        expr: |
          (thumbs_down_rate{arm="support_answer_v15_concise"}
            - thumbs_down_rate{arm="support_answer_v14"}) > 0.01
        for: 2h
        labels: { severity: page, action: notify }
```

The `action: auto_rollback` label is read by an Alertmanager webhook
receiver that calls the flag-provider API:

```yaml
# alertmanager.yml (fragment)
route:
  routes:
    - matchers: [ action="auto_rollback" ]
      receiver: rollback-webhook
      group_wait: 0s
      group_interval: 30s
      repeat_interval: 1h
receivers:
  - name: rollback-webhook
    webhook_configs:
      - url: http://ops-tools:8080/rollback/support_agent
        send_resolved: false
```

The receiver's implementation (small Go / Python service) calls the flag
provider's management API to set `support_agent.prompt_id = "support_answer_v14"`:

```python
# ops/rollback_webhook.py
import os, httpx
from fastapi import FastAPI, Request

app = FastAPI()
FLAGD_MGMT = os.environ["FLAGD_MGMT_URL"]
API_TOKEN = os.environ["FLAGD_API_TOKEN"]

@app.post("/rollback/support_agent")
async def rollback(req: Request):
    body = await req.json()
    async with httpx.AsyncClient() as client:
        r = await client.patch(
            f"{FLAGD_MGMT}/flags/support_agent.prompt_id",
            headers={"Authorization": f"Bearer {API_TOKEN}"},
            json={"defaultVariant": "v14"},
            timeout=5.0,
        )
        r.raise_for_status()
    return {"ok": True, "alert": body.get("alerts", [{}])[0].get("labels", {})}
```

### 2f. Rollback drill (deliverable 5 — the drill)

```markdown
# Rollback Drill — 2026-11-14

Purpose: force the fast-burn auto-rollback rule, measure TTD / TTE.

## Setup
- Canary cohort is at 5% (`support_agent.cohort=5`, `prompt_id=v15_concise`).
- Synthetic traffic injects 1 000 moderation-blocked responses into the
  canary cohort over 1 minute (via a test harness that labels spans with
  `app.outcome="refusal"` and `app.experiment.arm="support_answer_v15_concise"`).

## Timeline (wall-clock)
| Step | Time (UTC) | Delta |
|---|---|---|
| Injection start | 14:30:00 | 0 s |
| Moderation-FP delta crosses +2 pp | 14:30:42 | +42 s |
| Fast-burn alert fires | 14:35:42 | +5m 42s (5m `for` window + 42s detection) |
| Alertmanager routes to webhook | 14:35:43 | +5m 43s |
| flagd API returns 200 on `prompt_id := v14` | 14:35:44 | +5m 44s |
| First post-flip LLM span carries `prompt_id=v14` | 14:35:46 | +5m 46s |
| Moderation-FP rolls back to baseline | 14:36:40 | +6m 40s |

## Measurements

- **Observed TTD (injection → alert fires)**: 5m 42s. Within the launch-plan
  budget of 10 min.
- **TTE (alert fires → first span on v14)**: 4 s. Well under 60s SLA.
- **Fully automated**: no human in the loop between injection and rollback.

## Post-flip verification

Trace query: `app.feature="support_agent" AND time > 14:35:46`.
- 100% of spans carry `app.prompt.id="support_answer_v14"`. 0 spans on
  `v15_concise` after the flip.

## Action items from drill

1. Alertmanager `for: 5m` is intentional (avoids single-point noise),
   but means TTD is bounded below by 5 min. Document in the launch-plan.
2. The webhook service runs as a single pod; add a second replica to
   avoid a single-point failure in the rollback path.
3. The webhook has no circuit-breaker. If the flag provider is down
   when the alert fires, we need a secondary rollback target (direct
   environment-variable flip on the feature deployment). File as
   PROD-4832.
```

### 2g. Launch retro (deliverable 6)

```markdown
# Launch Retro — v15_concise

What the plan predicted: 72h at 5% delivers enough samples for the refusal
and moderation gates; thumbs-down needs 25% to be decidable. Predicted
CPSO delta under +5%; predicted TTFT delta under +100 ms.

What actually happened: refusal delta stayed at +0.1–0.2 pp across all
stages; thumbs-down delta landed at +0.3 pp (within gate). CPSO came in at
+3.6–+3.9% (half the predicted ceiling). TTFT delta +60–+90 ms.

Close-call gates:
- Daily spend on canary at 5% was at $72 against a $75 gate — one weekend
  spike away from the hard gate. The 25% stage was going to breach the
  dollar cap, which forced a mid-flight threshold-doc update
  (`plan-revision-01.md`). Next time: scale the daily-spend gate with
  cohort size, not flat.

Drill findings: TTD is bounded below by the alert's `for: 5m` window; TTE
is 4 s. Webhook service needs a second replica.

One change for next launch's plan: pre-size the daily-spend gate by cohort.
```

### 2h. Mapping to the exercise deliverables

| Exercise asks for | Section |
|---|---|
| Launch-plan doc (frozen, 4 gate families, sample-size math, auto-rollback) | 2a |
| Shadow-launch wiring (async, bounded queue, `app.shadow.*`) | 2b |
| Shadow report with per-scenario breakdown | 2c |
| Canary log with per-stage promotion artefacts | 2d |
| Auto-rollback alerts + flag-flip action | 2e |
| Rollback drill with measured TTD / TTE | 2f |
| Launch retro | 2g |

## 3. Validation steps

1. **Launch plan is frozen before flag flip.** `git tag plan-freeze-v15_concise` landed at 09:00 UTC; the flag didn't move until after.
2. **Four gate families each have a specific threshold.** Quality (composite, per-scenario, judge, thumbs-down); safety (refusal, mod-FP, injection-eval); cost (CPSO, absolute $); latency (TTFT, total, retry). No "TBD" or "monitor."
3. **Sample-size math is shown.** Every gate names baseline p, smallest Δ that matters, and the resulting n.
4. **Shadow is off the critical path.** The isolation test passes: 10-second shadow latency does not block a 1 ms primary path.
5. **Shadow queue is bounded with drop-on-overflow.** Counter advances; spans are not infinitely queued.
6. **Shadow spans are tagged distinctly.** `app.feature = "support_agent.shadow"` (so CPSO ledger separates); `app.shadow.of` and `app.shadow.variant` present.
7. **Shadow report has per-scenario breakdown.** The aggregate-vs-per-scenario table is the deliverable that catches the enterprise-tier regression.
8. **Per-stage promotion artefacts fit the template.** Every stage has the gate table filled in; "PROMOTE / HOLD / ROLLBACK" is explicit and signed.
9. **Fast-burn alert triggers the flag flip in a synthetic drill.** Observed TTE is well under 60 s SLA.
10. **Soft-burn alert pages but does not auto-flip.** Prometheus alert `action: notify` goes to the human pager, not the rollback webhook.
11. **Retro exists and names one change for next launch.** The feedback loop is closed.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Launch plan is committed+tagged before any flag flip | 1 | Timestamp on the freeze matches the plan header; no silent threshold edits in the git history. |
| Four gate families with quantitative thresholds | 2 | Quality, safety, cost, latency — each has at least two numeric gates; per-scenario floor is present. |
| Sample-size math shown for the key rate metrics | 1 | `n ≈ 16 × p × (1 - p) / Δ²` applied per metric; observation windows justified against traffic. |
| Shadow is async, bounded, and dropped on overflow | 1 | The isolation test passes; a drop counter is exposed. |
| Shadow spans tagged distinctly from primary | 1 | `app.shadow.*` attributes and distinct `app.feature` value. |
| Shadow report includes per-scenario breakdown | 1 | Table of per-scenario deltas, not just an aggregate judge score. |
| Per-stage promotion artefacts for ≥ 4 stages | 1 | 1%, 5%, 25%, 100% — each with the full gate table and a signed decision. |
| Auto-rollback rule triggers the flag flip | 1 | Fast-burn alert → webhook → flag-provider API → trace shows v14. |
| Rollback drill measured end-to-end | 1 | TTD, TTE recorded against SLA; at least one action item filed. |

A submission scoring < 6 is rejected. Hard failures: (a) plan edited mid-canary without a `plan-revision-NN` note, (b) shadow in-line with the request path, (c) per-scenario floors missing, (d) auto-rollback rule never rehearsed.

## 5. Common mistakes

- **Writing the plan after the flag flip.** The plan is a precommitment. A plan written in week 2 of a canary is a rationalisation; it does not catch the segment-only regression.
- **Aggregate-only gates.** The 6-pp enterprise-tier regression was invisible in the composite score. Per-scenario and per-segment floors are the fix.
- **Shadow in-line.** Awaiting the shadow before returning to the user doubles cost, doubles latency, and makes the shadow a dependency the user experiences. The queue + worker is non-negotiable.
- **Moving a threshold mid-canary.** If the plan says "CPSO delta ≤ +10%" and the observed delta is +12%, the correct action is rollback-or-hold, not threshold edit. If the threshold really was wrong, file `plan-revision-NN.md` with a timestamp — don't silently edit.
- **Rehearsal-free auto-rollback.** A fast-burn rule that has never been triggered is unproven. The drill in 2f catches the webhook-single-replica-problem *before* an incident does.
- **Shadow cost silently rolling into the primary feature's CPSO.** The `app.feature` must be distinct (`support_agent.shadow` vs `support_agent`) or the CPSO dashboard will show a step change when shadow traffic starts.
- **Burn-rate without a "for" window.** A naive threshold alert on a 5-second window will fire every minute. The `for: 5m` window is what filters single-point noise.
- **Treating 1% and 5% stages as "same shape, different sample size."** 1% is "aggregate not horribly broken"; 5% is "cost+latency survive at steady state"; 25% is the first stage where segment-only regressions are statistically detectable. Each stage decides something different.
- **Running shadow and canary at the same time without separation.** A shadow of `v15_concise` running during a canary of `v15_concise` means the same variant is firing both as primary and shadow — the shadow signal is noise and the CPSO is double-counted.
- **Fail-forward retrospective.** "We made it to 100% so the plan worked" is not a retro. The retro's value is the one change it identifies for next launch's plan; if there is no change, the retro was skipped.

## 6. References

- OpenTelemetry Generative-AI semantic conventions — <https://opentelemetry.io/docs/specs/semconv/gen-ai/> — the span attribute vocabulary the shadow wiring extends (`app.shadow.of`, `app.shadow.variant` are local-scope; `gen_ai.*` and `openinference.*` are untouched). The one-span-per-call discipline from mod-110 [[exercise-01]] is what makes the shadow traces queryable.
- Prometheus Alertmanager documentation — <https://prometheus.io/docs/alerting/latest/alertmanager/> — the `for:`, `labels.action`, routing-by-matcher, and webhook-receiver constructs used in 2e. The burn-rate alert shape is adapted from the SRE Workbook's multi-window multi-burn-rate chapter.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the launch-plan doc is the GOVERN-function artefact that makes an AI change *governable*: the thresholds are the policy, the freeze is the attestation, the retro closes the MAP/MEASURE/MANAGE loop. Without the plan doc, there is no auditable record of what the team agreed to accept.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the safety gate family directly serves ML09 (Output Integrity) and ML05 (Model Theft via resource-exhaustion attacks) — refusal-rate and moderation-FP regressions are the early signals. The shadow's role in detecting a regression *before* it reaches users is a controls-effectiveness measurement for ML09.
- MITRE ATLAS — <https://atlas.mitre.org/> — the auto-rollback rule is a containment control for Impact-tactic techniques (e.g. resource hijacking via an expensive variant, or a prompt that invites unsafe outputs the moderation stack catches but at a higher FP rate). Pre-registered thresholds + automated rollback shrink the window ATLAS tactics have to accumulate damage.

Related exercises: [[exercise-01]] (the flag layer the shadow and canary ride on; the `app.experiment.arm` attribute the gate queries consume); [[exercise-03]] (an experimentation-platform interface layered on the same substrate — the gate math in 2a is the "interface-side" cousin of the experimentation-platform's primary-metric math); [[exercise-04]] (the rollback runbook the on-call reads when a canary gate trips; the SLA numbers in 2f match the [[exercise-04]] rollback SLA); [[exercise-05]] (a hosted fine-tune ships through this same shadow → canary → GA regime); mod-109 [[exercise-04]] (CPSO ledger the cost gate queries); mod-109 [[exercise-05]] (burn-rate alert vocabulary 2e inherits); mod-106 (per-scenario taxonomy the per-scenario gates index into); mod-107 [[exercise-03]] (injection-eval the safety gate consults); mod-110 [[exercise-03]] (thumbs-down feedback the quality gate consumes).
