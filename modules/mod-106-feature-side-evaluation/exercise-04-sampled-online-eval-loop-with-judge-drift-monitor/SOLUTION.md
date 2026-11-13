### exercise-04: Sampled Online Eval Loop With a Judge-Drift Monitor — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to add an **online eval loop** to the production surface — the sampled, judge-scored stream that reports whether the shipped feature is behaving as the offline suite from [[exercise-01]] said it would — and to pair it with a **judge-drift monitor** that catches the case where the judge, not the feature, is what has changed. Offline evals answer "would this PR regress the fixture set." Online evals answer "is the thing actually in prod still as good as we thought when we shipped it." The two are distinct; the online loop cannot be skipped by a strong offline suite, and the offline suite cannot be replaced by live judging.

The failure mode the exercise is written to catch is twofold: (a) teams ship with no online signal and rely on user complaints as the regression detector, which costs incidents; (b) teams stand up a judge-LLM loop, watch the score drop, blame the model, and roll back — when in fact the judge has moved and the model is fine. A drift monitor is the control that makes the online signal trustworthy.

This solution provides:

- A **sampling strategy** that selects which prod requests get judged and at what rate, stratified so each slice has enough signal.
- A reference **`online_eval_worker.py`** that reads a sample of prod traces, runs the judge, writes verdicts to a time-series store, and emits alerts on threshold crossings.
- A reference **`judge_drift_monitor.py`** that runs a **frozen calibration set** through the current judge on a cadence and alerts when the judge's agreement with its pinned baseline drifts.
- A **budget model** for how many judge calls per day/week this costs and where the budget goes when it tightens.
- A **grading rubric** for the instructor.

The lesson the exercise defends is: *an online eval is a second signal you must be able to explain*. When the number moves, the team must be able to say "the model moved" or "the judge moved" or "the surface moved" with evidence. Without the drift monitor, every regression is ambiguous and every rollback is gambling.

## 2. Worked answer or implementation

### 2a. Sampling strategy (deliverable 1 — decision rationale)

Not every prod request gets judged. The design must specify the sampling rule and the per-slice floor.

```yaml
# online/sampling.yaml
schema_version: 1
sampling_target_qps: 0.2             # ~17k judged rows / day at full load
stratification:
  # Each slice has its own rate; the per-slice floor ensures signal for
  # thin slices even when they're a small fraction of traffic.
  by_shape:
    task:        { rate: 0.01, floor_per_day: 500 }
    refusal:     { rate: 0.05, floor_per_day: 200 }
    format:      { rate: 0.02, floor_per_day: 100 }
  by_tenant_class:
    enterprise:  { rate: 0.02, floor_per_day: 300 }
    smb:         { rate: 0.005, floor_per_day: 200 }
  oversample_on:
    - { trigger: "user_rating:thumbs_down", rate: 1.00 }   # every red row
    - { trigger: "agent.pause_reason:*",     rate: 0.20 }  # pauses matter
    - { trigger: "model_version:just_rolled_out", rate: 0.10, ttl_hours: 72 }
retention_days: 90
```

Design points:

- Every slice has **both** a rate and a floor. A rate alone starves thin slices; a floor alone can over-sample fat slices. The two together give usable signal per slice within the budget.
- `oversample_on` is the mechanism that puts judge spend where it matters: every thumbs-down is judged (cheap; the rate is bounded by user volume), every pause is judged at 20%, and newly-rolled-out model versions are judged at 10% for 72 hours.
- `retention_days: 90` matches the trace retention in [[exercise-02]] so a thumbs-down → fixture conversion path has the trace available.

### 2b. `online_eval_worker.py` — the sampled judge loop (deliverable 2)

```python
"""online_eval_worker.py — sample prod traces; judge; write verdicts."""

from __future__ import annotations

import asyncio
import dataclasses
import logging
import pathlib
import random
import time
from typing import AsyncIterator

import yaml

from judge_llm import JudgeLLM
from metrics import counter, gauge, histogram
from trace_store import TraceStore

log = logging.getLogger(__name__)


@dataclasses.dataclass
class SamplingPolicy:
    rate_by_shape: dict[str, float]
    floor_by_shape: dict[str, int]
    rate_by_tenant_class: dict[str, float]
    oversample_rules: list[dict]

    @classmethod
    def load(cls, path: str) -> "SamplingPolicy":
        cfg = yaml.safe_load(pathlib.Path(path).read_text())
        strat = cfg["stratification"]
        return cls(
            rate_by_shape={k: v["rate"]
                           for k, v in strat["by_shape"].items()},
            floor_by_shape={k: v["floor_per_day"]
                            for k, v in strat["by_shape"].items()},
            rate_by_tenant_class={k: v["rate"]
                                  for k, v in strat["by_tenant_class"].items()},
            oversample_rules=strat.get("oversample_on", []),
        )

    def should_sample(self, trace: dict, daily_seen: dict[str, int]) -> bool:
        shape = trace["meta"]["shape"]
        tenant_class = trace["principal"].get("tenant_class", "smb")
        for rule in self.oversample_rules:
            if _trigger_matches(rule["trigger"], trace):
                return random.random() < rule["rate"]
        base_rate = max(
            self.rate_by_shape.get(shape, 0.0),
            self.rate_by_tenant_class.get(tenant_class, 0.0),
        )
        # Floor logic: if we're under the per-day floor for this shape,
        # bias toward sampling until the floor is reached.
        floor = self.floor_by_shape.get(shape, 0)
        if daily_seen.get(shape, 0) < floor:
            base_rate = max(base_rate, 0.5)
        return random.random() < base_rate


async def run(policy_path: str, judge_lock_path: str) -> None:
    policy = SamplingPolicy.load(policy_path)
    judge = JudgeLLM.from_lock(judge_lock_path)
    daily_seen: dict[str, int] = {}
    async for trace in _trace_tail():
        try:
            if not policy.should_sample(trace, daily_seen):
                continue
            daily_seen[trace["meta"]["shape"]] = \
                daily_seen.get(trace["meta"]["shape"], 0) + 1
            await _judge_one(trace, judge)
        except Exception:
            counter("online_eval.errors", 1)
            log.exception("judge failed on trace %s", trace.get("trace_id"))


async def _judge_one(trace: dict, judge: JudgeLLM) -> None:
    started = time.monotonic()
    rubric = _rubric_for(trace["meta"]["shape"])
    score, note = await judge.score_async(
        rubric=rubric,
        input_text=trace["input"]["user_message"],
        output_text=trace["model_output"]["draft"],
    )
    _emit_verdict(trace, score, note)
    histogram("online_eval.latency_ms",
              (time.monotonic() - started) * 1000,
              tags={"shape": trace["meta"]["shape"]})
    counter("online_eval.judged", 1,
            tags={"shape": trace["meta"]["shape"]})


def _emit_verdict(trace: dict, score: float, note: str) -> None:
    # Write to the time-series store the dashboards + alerts read.
    gauge("online_eval.score", score, tags={
        "shape": trace["meta"]["shape"],
        "tenant_class": trace["principal"].get("tenant_class", "smb"),
        "model_version": trace["model_output"]["model_version"],
        "prompt_version": trace["model_output"]["prompt_version"],
    })
    # Red verdicts → persistent record for review + fixture-candidate queue.
    if score < _red_bar(trace["meta"]["shape"]):
        TraceStore.flag_for_review(
            trace_id=trace["trace_id"],
            reason="online_judge_below_bar",
            score=score,
            note=note,
        )


def _rubric_for(shape: str) -> str:
    # Mirrors the offline rubric (same wording) so offline and online judges
    # are comparable. Any divergence here must land in both files via a
    # dedicated PR (see judge_drift_monitor.py §2c).
    return (pathlib.Path("evals/rubrics") / f"{shape}.rubric.md").read_text()


def _red_bar(shape: str) -> float:
    bars = yaml.safe_load(
        pathlib.Path("evals/thresholds.yaml").read_text())
    return bars["global"]["task_rubric_mean"]["min"]


def _trigger_matches(trigger: str, trace: dict) -> bool:
    key, want = trigger.split(":", 1)
    v = _path_get(trace, key)
    return v == want if want != "*" else v is not None


def _path_get(d: dict, path: str):
    cur = d
    for part in path.split("."):
        if isinstance(cur, dict) and part in cur:
            cur = cur[part]
        else:
            return None
    return cur


async def _trace_tail() -> AsyncIterator[dict]:
    async for row in TraceStore.tail("surface=helpdesk-reply-copilot"):
        yield row


if __name__ == "__main__":
    asyncio.run(run("online/sampling.yaml", "evals/judge.lock"))
```

Design points:

- The rubric file is **shared** with the offline suite. A rubric divergence between offline and online is a bug source, not a feature.
- The `score` gauge is tagged with `model_version` and `prompt_version` so a dashboard can slice the regression along either axis — the question "did the model move or did the prompt move" is an attribute query, not a code change.
- Red rows are flagged for review in the same `TraceStore` from [[exercise-02]]; the fixture-conversion CLI picks these up. The online loop therefore grows the regression set as the surface runs.
- The worker **doesn't** write verdicts for rows above the bar as separate records — those land in the aggregate metric only, to keep cost bounded. Red rows get the detailed record.

### 2c. Judge-drift monitor (deliverable 3)

The monitor's job is to catch the case where the judge has moved while the surface has not.

```python
"""judge_drift_monitor.py — a cron job; detects judge drift."""

from __future__ import annotations

import asyncio
import json
import pathlib
import statistics
import time

import yaml

from judge_llm import JudgeLLM
from metrics import alert, gauge

CALIBRATION_SET = "evals/calibration/judge_calibration_set.jsonl"
BASELINE = "evals/calibration/judge_baseline.json"
TOLERANCE = {"mean_abs_delta": 0.05, "max_abs_delta": 0.20}


async def run() -> None:
    judge = JudgeLLM.from_lock("evals/judge.lock")
    baseline = json.loads(pathlib.Path(BASELINE).read_text())
    rows = [json.loads(line)
            for line in pathlib.Path(CALIBRATION_SET).read_text().splitlines()]

    scored: dict[str, float] = {}
    for row in rows:
        s, _ = await judge.score_async(
            rubric=row["rubric"],
            input_text=row["input_text"],
            output_text=row["output_text"],
        )
        # Average of three runs per row (same rule as offline judge_lock).
        runs = [s]
        for _ in range(2):
            runs.append((await judge.score_async(
                rubric=row["rubric"],
                input_text=row["input_text"],
                output_text=row["output_text"],
            ))[0])
        scored[row["id"]] = statistics.mean(runs)

    deltas = {rid: scored[rid] - baseline["scores"][rid]
              for rid in scored}
    mean_abs = statistics.mean(abs(d) for d in deltas.values())
    max_abs = max(abs(d) for d in deltas.values())

    gauge("judge_drift.mean_abs_delta", mean_abs,
          tags={"judge_model": baseline["judge_model"]})
    gauge("judge_drift.max_abs_delta", max_abs,
          tags={"judge_model": baseline["judge_model"]})

    if mean_abs > TOLERANCE["mean_abs_delta"] \
       or max_abs > TOLERANCE["max_abs_delta"]:
        alert(
            name="judge_drift_tripped",
            severity="page",
            body=(
                f"Judge drift detected against {baseline['baseline_at']}:\n"
                f"  mean_abs_delta = {mean_abs:.3f} (tol {TOLERANCE['mean_abs_delta']})\n"
                f"  max_abs_delta  = {max_abs:.3f} (tol {TOLERANCE['max_abs_delta']})\n"
                f"Top movers: " +
                ", ".join(f"{rid}:{d:+.2f}"
                          for rid, d in sorted(deltas.items(),
                                               key=lambda x: -abs(x[1]))[:5])
            ),
            runbook="runbooks/judge_drift.md",
        )

    # Persist today's run so operators can compare over time.
    pathlib.Path(f"logs/judge_drift/{int(time.time())}.json").write_text(
        json.dumps({"scored": scored, "deltas": deltas,
                    "mean_abs": mean_abs, "max_abs": max_abs}, indent=2))


if __name__ == "__main__":
    asyncio.run(run())
```

The calibration set:

- 40–60 rows **hand-labeled** by a human reviewer. Each row has an `(input, output, reference_score)` tuple, where `reference_score` is the score a human calibrator gave. The baseline is the judge's mean score (over three runs per row) against this set at the moment the judge was pinned.
- The calibration set is **frozen**. It does not grow — adding rows invalidates the baseline. The set is reviewed on the same cadence as the `judge.lock` (quarterly).
- Rows span the four scoring regimes (clearly-passing, clearly-failing, borderline-high, borderline-low) so the monitor detects both systematic bias (mean drift) and borderline-row flipping (max drift).

The monitor runs:

- **Nightly** on a GitHub Actions schedule, writing to the alerting system.
- **On every proposed judge-model change** as a mandatory PR step. A judge upgrade whose drift exceeds tolerance against the current baseline requires either a recalibration of the offline bars under the threshold-change protocol in [[exercise-01]] or a rollback.
- The monitor **does not** alert on single-row flips below the max tolerance — flips are noise. It alerts on aggregate drift.

### 2d. The runbook (deliverable 4)

```markdown
# runbooks/judge_drift.md

## Who gets paged
- Primary: applied-AI oncall.
- Secondary: data-science eval owner.

## First 15 minutes
1. Pull the latest judge-drift report from `logs/judge_drift/`.
   Confirm the delta is not a one-off (look at the previous 3 runs).
2. Compare the provider's model changelog against the judge's pinned
   version. If the provider silently shipped a new checkpoint, the
   expected drift explanation is "the judge moved."
3. Pull the online eval score dashboard. If the online score dropped
   at the same hour as the drift detection, the two are correlated —
   the online drop is probably the judge, not the surface.
4. If the online score did not drop but drift did, the judge has moved
   and is masking a drop that would otherwise show up — more urgent.

## Decision tree
- **Judge moved, no online drop** → the pin is wrong; re-pin to the
  old checkpoint if still available; otherwise, enter the recalibration
  protocol (below).
- **Judge moved, online drop** → disambiguate: run the current surface
  version against the old pin; if it passes, the regression is in
  the judge, not the surface; if it fails, the regression is in both.
- **No judge move, drift tripped by max delta** → investigate the
  specific borderline rows; usually a judge-prompt sensitivity issue.
  File a judge-rubric improvement PR.

## Recalibration protocol
When the judge must move:
1. Capture the new judge's scores against the frozen calibration set.
2. Score the offline fixture set under both judges; generate a
   comparison report.
3. File a thresholds-change PR under [[exercise-01]]'s protocol with
   the new judge in `judge.lock`, the new baseline in
   `judge_baseline.json`, and the recalibrated bars in `thresholds.yaml`.
4. Merge only after a second owner signs off and the full offline
   lane from [[exercise-03]] passes on the new judge.
```

### 2e. Budget model (deliverable 5)

```markdown
# Online eval cost — weekly budget

## Volume
- Sampled rows/day: ~17,000 (Shape A 10k @ 1%, B 500 @ 5%, etc.)
- Drift monitor: 60 rows × 3 runs × 2 runs/day = 360 judge calls/day.
- Oversample volume (thumbs-downs, pauses, rollouts): ~500/day.
- Total: ~18k judge calls/day.

## Cost (indicative)
- At ~$0.001 / judge call, ~$18/day, ~$550/mo. Subject to the judge
  provider's price list; re-estimate on each `judge.lock` review.

## When the budget tightens
1. Cut `by_shape.task.rate` first (10% rate drop → ~1,000 fewer calls/day).
2. Keep `by_shape.refusal.rate` and `oversample_on.thumbs_down`
   untouched — they produce the fixture-conversion signal.
3. Do not cut the drift monitor. 360 calls/day is negligible and the
   monitor's value is explanatory, not throughput-dependent.
```

### 2f. Implementation notes

- **Sampling is stratified, not uniform.** A uniform sample starves refusal and format rows — those are the shapes with the tightest bars and the most expensive misses in prod. Stratified sampling keeps the thin slices observable.
- **Red rows feed back to the fixture library.** The online worker flags red rows via the same `TraceStore.flag_for_review` the on-call uses for incident capture in [[exercise-02]]. The regression suite therefore grows from the online signal, not only from paging incidents.
- **The drift monitor runs on a frozen calibration set.** A moving calibration set is not a monitor. The set is small (40–60 rows), hand-labeled, and reviewed quarterly.
- **Judge pin = judge model + judge prompt + runs averaged.** Drift can come from the model, the prompt, or the averaging rule. The monitor's baseline file captures all three. A judge-prompt edit is a baseline-invalidating change and must go through the recalibration protocol.
- **Correlated drift and online-score drop is the diagnostic moment.** The runbook's decision tree is where the dual-signal pays off: with both signals, the team can say "judge" or "surface" with confidence; without the drift signal, every regression is ambiguous.
- **Oversample on thumbs-down.** Thumbs-downs are the leading indicator; sampling them at 100% costs essentially nothing (users thumbs-down < 2% of the time) and feeds the fixture-conversion pipeline.
- **The online loop is additive to the offline suite, not a replacement.** Online scores are about *was* — the thing that just happened in prod. Offline scores are about *would* — the thing that would happen if we shipped this. Both signals are needed.
- **Alert fatigue is the enemy.** The drift monitor alerts on aggregate thresholds (`mean_abs_delta`, `max_abs_delta`). Per-row flips below the max tolerance are logged and visible in the dashboard but do not page. Over-alerting is how teams learn to ignore the signal.

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. `online/sampling.yaml` exists with per-shape rate + floor, per-tenant-class rate, and oversample rules.
2. The worker samples real prod traces and writes scores to a time-series store tagged by shape, tenant class, model version, and prompt version.
3. Red rows flow to `TraceStore.flag_for_review` for fixture-candidate triage.
4. `evals/calibration/judge_calibration_set.jsonl` contains 40–60 hand-labeled rows across the four scoring regimes (clearly-pass, clearly-fail, borderline-high, borderline-low).
5. The drift monitor computes `mean_abs_delta` + `max_abs_delta` against a frozen baseline and alerts on tolerance breaches.
6. A runbook exists naming primary + secondary oncall, the first-15-minutes checklist, and the recalibration protocol.
7. The budget doc names weekly call volume, dollar cost, and the tightening order.
8. Judge-prompt edits are treated as baseline-invalidating; the linter rejects a prompt edit without a baseline refresh.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Stratified sampling with per-slice rate + floor | 1 | Rate and floor both present for ≥ 2 axes. |
| Oversample rules on thumbs-down + new-rollout | 1 | 100% on thumbs-down; TTL'd rate on new model versions. |
| Online worker writes score tagged by model/prompt version | 1 | Enables slicing without a code change. |
| Red rows flow back to the fixture-review queue | 1 | `TraceStore.flag_for_review` or equivalent. |
| Frozen calibration set with four scoring regimes | 1 | 40–60 rows; regimes documented; frozen. |
| Drift monitor with mean + max deltas + alert tolerances | 1 | Both metrics; alert on breach of either. |
| Runbook with decision tree + recalibration protocol | 1 | Judge-vs-surface disambiguation written down. |
| Budget doc with volume + cost + tightening order | 1 | Named numbers; drift monitor is uncut. |
| Judge-prompt edits force baseline refresh | 1 | Linter + required PR step. |
| Alerting is aggregate, not per-row | 1 | Per-row flips are logged, not paged. |

A submission scoring < 6 is rejected. The hard failures are (a) uniform sampling with no floor, (b) moving calibration set (no baseline), (c) drift monitor that pages per-row, (d) no runbook + no recalibration protocol.

## 5. Common mistakes

- **Uniform sampling.** Head rows swamp the budget; refusal and format slices have no signal. Stratified-with-floor is the fix.
- **Judging every request.** Cost explodes; the judge becomes a bottleneck. Sample deliberately; oversample on signal-rich triggers.
- **Growing calibration set.** New rows change the baseline each time; drift becomes undetectable. Freeze the set; review quarterly.
- **No separate drift monitor.** Every online regression is ambiguous: model or judge? Stand up the monitor from day one.
- **Per-row alerting.** Alert fatigue; the team mutes the channel. Aggregate deltas with named tolerances are the alert surface.
- **Blaming the model on drift.** The runbook decision tree exists to prevent the rollback-first reflex. Check the judge before touching the surface.
- **Online worker computes a different rubric than offline.** Divergence silently moves the comparison baseline. Share the rubric files.
- **No fixture feedback loop.** Red online rows should queue for review; otherwise the online signal is a dashboard without a fix path.
- **Scaling down the drift monitor under budget pressure.** The monitor is 360 calls/day — 2% of the sampled budget. Cutting it saves nothing and loses the explanatory signal.
- **Judge-prompt edits treated as "just a change."** Any prompt change moves the baseline; the linter + required PR step ensures recalibration happens with the edit.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the sampled online loop + stratification + drift monitor implement the RMF's **MEASURE** function requirement that production systems emit observable evidence of continued conformance, and the **MANAGE** function's incident-response expectation that drifts produce triageable, owner-routable alerts.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — oversampling on refusal rows and thumbs-downs is the surface's defence against the ML top-10 output-integrity items as they manifest post-deploy. The drift monitor is the control that distinguishes a safety regression in the feature from a shift in the judge.
- MITRE ATLAS — <https://atlas.mitre.org/> — online red rows that match ATLAS technique entries should be fast-pathed to the regression fixture queue from [[exercise-02]] so adversarial regressions picked up in prod feed the CI gate on the next PR.

Related exercises: [[exercise-01]] (offline thresholds and the judge pin the drift monitor compares against); [[exercise-02]] (red online rows become fixture candidates); [[exercise-03]] (the nightly lane consumes online aggregate scores for the dashboard); [[exercise-05]] (persona dashboards pivot online scores by stakeholder).
