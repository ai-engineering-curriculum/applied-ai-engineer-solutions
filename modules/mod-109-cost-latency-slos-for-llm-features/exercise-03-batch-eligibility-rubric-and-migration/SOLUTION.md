### exercise-03: Batch Eligibility Rubric and Migration — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to apply the **three-question batch eligibility rubric** (freshness SLA, structural fit, economics) to one real workload, stage a **dual-run migration** from the sync path to a provider batch API, and produce the artefacts that make the migration reviewable: the rubric doc with peer sign-off, the dual-run diff report, the batch runner with observability, and a demonstrated fall-through path for late or failed batches.

The failure mode the exercise is written to catch is the batch migration that ships without a Q1 answer. A team moves an hourly cron off sync onto a batch endpoint because "it's cheaper," then discovers six weeks later that the product's downstream taxonomy actually needed updates within fifteen minutes and the batch ran for eight hours on its slowest day. The rubric catches that before the migration ships; the dual-run proves the drift is acceptable; the fall-through makes the migration safe when the provider's SLA tail misbehaves.

This solution provides:

- A **worked rubric** filled in for the reference workload (`helpdesk-reply-copilot` background intent-classifier enrichment).
- A **batch JSONL builder** that namespaces `custom_id`, matches sync-path request bytes exactly, and is idempotent on re-run.
- A **submit / poll / reconcile runner** with line-level partial-failure handling, span attribution matching [[exercise-01]] / [[exercise-04]], and a deadline shorter than the workload's freshness SLA so the fall-through can fire.
- A **dual-run diff template** separating structural and semantic drift.
- A **fall-through test** showing that a late batch re-issues its affected records against sync and the ledger reflects the extra cost.
- A **grading rubric** for the instructor.

The lesson the exercise defends: *batch is not a cost switch.* It is a workload-shape commitment: lines must be independent, the freshness SLA must be known and loose, and the economics must survive a migration-cost amortisation window. The rubric exists so that these three decisions are written down before code is cut.

## 2. Worked answer or implementation

### 2a. Rubric doc (deliverable 1)

```markdown
# Batch eligibility — helpdesk-reply-copilot: ticket-intent-enrichment — 2026-11-14

## Workload
Every hour, a scheduled job reads the last hour of created tickets,
calls an LLM to classify each ticket into (routing_queue, severity,
product_area), and writes the result back as metadata on the ticket.
Consumer: the triage dashboard reads these fields when building the
agent's work queue the next morning.

## Q1 — Freshness SLA
Maximum staleness: 4 hours (NEW — was "we don't know" before this exercise).
Source: product decision 2026-11-10 (ticket PROD-2211; signed by Avery,
product lead for triage). Reasoning: the triage dashboard is read at
start-of-shift; a new ticket must carry enrichment metadata by the
first shift read after creation. Night tickets can wait until morning.
Batch-eligible on freshness? YES — 4h >> typical batch completion time
(we will set the fall-through deadline at 3h to leave a one-hour recovery
window).

## Q2 — Structural fit
Independent at the line level? YES. Each classification is `(ticket →
labels)`; no cross-ticket context.
Any cross-request state? None. Tool catalog is a static schema, not a
per-row lookup.
Batch-eligible on structure? YES.

## Q3 — Economics
Current monthly sync cost:        $780
  (50k requests/day × 30 days × $0.00052/request, measured on exercise-04
  ledger, sample 2026-10 full-month)
Projected monthly batch cost:     $390
  (50% batch discount assumed; verify against provider pricing page before
  rollout) <!-- verify-before-publishing: confirm current OpenAI batch discount
  and that cache-across-batch behaviour matches the pricing page -->
Projected monthly savings:        $390
Migration cost estimate:          ~0.75 engineer-week (design + build +
                                   dual-run + review).
Payback horizon:                  ~2 months.

## Decision
SHIP BATCH. Dual-run for two weeks before full cutover.
Fall-through: any batch whose status is still not `completed` 3h after
submit is cancelled; the affected `custom_id`s are re-issued against the
sync endpoint; cost delta is logged and alerted at 10% monthly.

## Peer sign-off
- Product: Avery (triage)                 ✓ 2026-11-10 (PROD-2211)
- Data owner: Jordan (ticket-metadata DB) ✓ 2026-11-13
- Finance / cost owner: Sam (platform FP&A) ✓ 2026-11-13
```

The Q1 answer being new is the exercise's point. Before the rubric, the hourly cron existed because "someone set it that way." Writing the SLA is often the most valuable deliverable here.

### 2b. Batch JSONL builder (deliverable 2)

```python
# code/build_batch.py
"""Build a provider-shape batch input file from a workload fixture.

Requirements:
  - custom_id is namespaced `<batch_seq>-<row_id>` — collisions across days
    cannot happen.
  - Each line's `body` is byte-identical to what the sync path sends for
    the same row, so downstream diff is meaningful.
  - Idempotent: re-running with the same inputs produces the same bytes.
"""
from __future__ import annotations

import hashlib
import json
from pathlib import Path


def batch_seq(now_hour: str) -> str:
    # e.g. "2026-11-14T15" → stable per-hour sequence id.
    return hashlib.sha1(now_hour.encode()).hexdigest()[:10]


def make_line(seq: str, row: dict) -> dict:
    """Return a single JSONL line for the batch endpoint.

    The `body` field is produced by the same function the sync path uses,
    so the two paths emit byte-identical request bodies for the same row.
    """
    from code.sync_path import build_sync_request  # single source of truth

    return {
        "custom_id": f"{seq}-{row['id']}",
        "method": "POST",
        "url": "/v1/chat/completions",     # verify against provider's batch endpoint
        "body": build_sync_request(row),
    }


def build(input_path: Path, output_path: Path, now_hour: str) -> dict:
    seq = batch_seq(now_hour)
    out_lines = []
    seen = set()
    with input_path.open() as f:
        for line in f:
            row = json.loads(line)
            if row["id"] in seen:          # idempotency on re-run
                continue
            seen.add(row["id"])
            out_lines.append(make_line(seq, row))
    # Deterministic ordering for byte-stable output file.
    out_lines.sort(key=lambda d: d["custom_id"])
    with output_path.open("w") as f:
        for ln in out_lines:
            f.write(json.dumps(ln, sort_keys=True) + "\n")
    return {"seq": seq, "line_count": len(out_lines), "output": str(output_path)}
```

Key points:

- **`build_sync_request(row)` is the single source of truth.** Importing it from the sync path (instead of rewriting the body construction) is what makes the dual-run diff meaningful. Any schema drift between the two paths becomes compile-time visible.
- **`sort_keys=True` on the inner body.** Same reason as [[exercise-02]]: byte-identical payloads across sync and batch.
- **Deterministic line order.** Sorting by `custom_id` means the output file hashes stably across re-runs, which matters for CI-shape reproducibility.

### 2c. Submit / poll / reconcile runner (deliverable 3)

```python
# code/run_batch.py
"""Submit a batch file, poll with a deadline, reconcile results.

The deadline is the workload's freshness SLA minus a recovery window
(e.g. SLA 4h → poll for 3h → fall through at 3h → 1h recovery).

Line-level partial failures are surfaced, never silently dropped.
"""
from __future__ import annotations

import json
import time
from dataclasses import dataclass
from typing import Callable

from opentelemetry import trace

tracer = trace.get_tracer("helpdesk-reply-copilot.batch")


@dataclass
class BatchResult:
    status: str                  # "completed" | "expired" | "failed" | "cancelled"
    batch_id: str
    results_by_custom_id: dict
    line_errors: dict            # custom_id -> {"code": ..., "message": ...}
    cost_usd: float


def submit_and_poll(
    provider_client,
    jsonl_path: str,
    deadline_seconds: int,
    pricing_fn,                  # from exercise-04
    feature: str,
    fallthrough: Callable[[list[str]], None],
) -> BatchResult:
    with tracer.start_as_current_span("batch.submit_and_poll") as span:
        span.set_attribute("app.feature", feature)
        span.set_attribute("app.batch.deadline_s", deadline_seconds)
        uploaded = provider_client.files.create(file=open(jsonl_path, "rb"),
                                                purpose="batch")
        batch = provider_client.batches.create(
            input_file_id=uploaded.id,
            endpoint="/v1/chat/completions",
            completion_window="24h",  # provider contract; our deadline is tighter
        )
        span.set_attribute("app.batch.batch_id", batch.id)
        t0 = time.monotonic()
        while True:
            batch = provider_client.batches.retrieve(batch.id)
            span.set_attribute("app.batch.status", batch.status)
            if batch.status == "completed":
                break
            if batch.status in {"failed", "cancelled", "expired"}:
                # Fall-through: re-issue all submitted rows against sync.
                custom_ids = _all_custom_ids(jsonl_path)
                fallthrough(custom_ids)
                return BatchResult(batch.status, batch.id, {}, {}, 0.0)
            if time.monotonic() - t0 >= deadline_seconds:
                provider_client.batches.cancel(batch.id)
                custom_ids = _all_custom_ids(jsonl_path)
                fallthrough(custom_ids)
                return BatchResult("deadline_exceeded", batch.id, {}, {}, 0.0)
            time.sleep(30)

        # Extract results and separate line-level errors.
        output = provider_client.files.content(batch.output_file_id).text
        results, line_errors, cost = {}, {}, 0.0
        for raw in output.splitlines():
            rec = json.loads(raw)
            cid = rec["custom_id"]
            if rec.get("error"):
                line_errors[cid] = rec["error"]
                continue
            results[cid] = rec["response"]["body"]
            cost += pricing_fn(
                system="openai",  # or the provider in use
                model=rec["response"]["body"]["model"],
                input_tokens=rec["response"]["body"]["usage"]["prompt_tokens"],
                output_tokens=rec["response"]["body"]["usage"]["completion_tokens"],
                cached_input_tokens=0,
                pricing_shape="batch",
            )

        line_error_ratio = len(line_errors) / max(1, len(results) + len(line_errors))
        span.set_attribute("app.batch.line_error_ratio", line_error_ratio)
        span.set_attribute("app.cost.usd", cost)
        span.set_attribute("app.cost.pricing_shape", "batch")
        if line_errors:
            fallthrough(list(line_errors.keys()))

        return BatchResult("completed", batch.id, results, line_errors, cost)
```

Observability shape is deliberate:

- `app.batch.batch_id` — correlates the batch run across the submit, poll, and reconcile steps.
- `app.batch.status` — distinguishes `completed` from `expired` / `failed` / `cancelled` / `deadline_exceeded`.
- `app.batch.line_error_ratio` — the single metric a dashboard alerts on; above 0.5% on a mature workload is a regression.
- `app.cost.usd` and `app.cost.pricing_shape="batch"` — join to the ledger from [[exercise-04]] so the batch discount lands in reporting.

Provider-specific field names (`files.create`, `batches.create`, `completion_window`) are from the OpenAI Batch API shape; adjust if targeting Anthropic Message Batches, Bedrock, or Vertex. <!-- verify-before-publishing: confirm current endpoint names and completion-window values per provider -->

### 2d. Dual-run shadow diff (deliverable 4)

Run the same sample through both the sync endpoint (current path) and the batch endpoint (new path), then diff.

```python
# code/dual_run_diff.py
def diff(sync_results: dict, batch_results: dict) -> dict:
    structural, semantic, missing = [], [], []
    for cid, sync in sync_results.items():
        if cid not in batch_results:
            missing.append(cid); continue
        batch = batch_results[cid]
        if _top_level_shape(sync) != _top_level_shape(batch):
            structural.append({"id": cid, "sync_shape": _top_level_shape(sync),
                               "batch_shape": _top_level_shape(batch)})
            continue
        if _extract_classification(sync) != _extract_classification(batch):
            semantic.append({"id": cid,
                             "sync": _extract_classification(sync),
                             "batch": _extract_classification(batch)})
    n = len(sync_results)
    return {
        "total": n,
        "structural_diff_rate": len(structural) / max(1, n),
        "semantic_diff_rate":   len(semantic) / max(1, n),
        "missing_rate":         len(missing) / max(1, n),
        "examples": {"structural": structural[:5], "semantic": semantic[:5],
                     "missing": missing[:5]},
    }
```

```markdown
# docs/dual-run-diff.md — helpdesk-reply-copilot:intent-enrichment — 2026-11-14

## Sample
500 tickets from the last 24 hours, same input on both paths. Model
pinned to the same version; temperature 0.

## Results
- Total records: 500
- Batch line-error ratio: 0.2% (1/500) — one record hit a content-filter
  rejection; the fall-through re-issued sync and succeeded.
- Structural diff rate: 0.0% (0/500).
- Semantic diff rate: 1.4% (7/500) — all seven differed on `severity`
  between `low` and `medium`. Three look like batch-side wins, four look
  like sync-side wins. Below the triage eval-suite tolerance of 3%
  (ref: mod-106 enrichment suite v4).
- Cost delta on this 500-record sample:
  - Sync:  $0.26
  - Batch: $0.14 (46% lower — within the expected 50% band, slightly less
           because of a small cache-miss population).

## Verdict
Within tolerance. Proceed with two-week dual-run at production volume
before deprecating the sync path. Any ratio excursion > 3% blocks
cutover.
```

The split between structural and semantic drift is the point. Structural drift is a bug (shape changed between the two paths) and must be fixed before cutover. Semantic drift is expected (different runs of the same model vary), and its only question is "does it stay inside the eval tolerance from mod-106?"

### 2e. Fall-through test (deliverable 5)

```python
# tests/test_fall_through.py
def test_cancelled_batch_falls_through_to_sync(
    provider_client_stub, sync_endpoint_stub, pricing_fn, feature,
):
    """A cancelled batch re-issues its records against sync.

    Prove: no records are silently dropped; the ledger shows two cost
    rows for the affected ids (the batch attempt's $0 + the sync re-issue's
    actual cost); the batch status on the span is correct.
    """
    provider_client_stub.simulate_status("cancelled")
    captured = []

    def fallthrough(custom_ids: list[str]) -> None:
        for cid in custom_ids:
            resp = sync_endpoint_stub.call(cid)
            captured.append((cid, resp))

    result = submit_and_poll(
        provider_client_stub, "fixtures/batch_input.jsonl",
        deadline_seconds=1,                       # force fast fall-through
        pricing_fn=pricing_fn, feature=feature,
        fallthrough=fallthrough,
    )
    expected_ids = _all_custom_ids("fixtures/batch_input.jsonl")
    assert sorted(cid for cid, _ in captured) == sorted(expected_ids)
    assert result.status == "cancelled"
    # Ledger assertions (join to span log):
    spans = _query_spans(feature=feature, since=_t0())
    sync_cost = sum(s["app.cost.usd"] for s in spans
                    if s["app.cost.pricing_shape"] == "sync")
    batch_cost = sum(s["app.cost.usd"] for s in spans
                     if s["app.cost.pricing_shape"] == "batch")
    assert sync_cost > 0 and batch_cost == 0.0
```

The test's job is the invariant, not the heroics: every submitted `custom_id` must either be in `results_by_custom_id` or in the fall-through stream. A silent drop is the only hard failure.

### 2f. Implementation notes and gotchas

- **Q1 is the exercise's actual lesson.** Nobody tells you the freshness SLA. Writing it is the hardest and most valuable step; most teams discover that nobody knows and that the first conversation to have is a product one, not an engineering one.
- **Deadline < freshness SLA − recovery window.** If the SLA is 4 hours and the provider promises 24, polling for 23 hours leaves no time for fall-through. Poll for 3 hours; fall through at 3; leave a 1-hour sync-recovery buffer.
- **Byte-identical sync and batch request bodies.** Same prompt, same model, same `sort_keys=True`. Any divergence shows up as structural diff, and the review will reject it.
- **Namespace `custom_id`.** `<batch_seq>-<row_id>` is the minimum; the most common reconciliation bug is a day-N submission colliding with a day-N+1 submission because both used bare `row_id`.
- **Line-level errors are not batch-level errors.** A batch can be `completed` while 3% of its lines carry `error` fields. Reconcile per-line; alert on the ratio.
- **Prompt-caching inside a batch.** On some providers, cache reads apply inside batch jobs; on others, they do not. If the workload shares a long system prompt across lines, verify this on the provider's current docs before publishing economics. <!-- verify-before-publishing: confirm per-provider behaviour (OpenAI, Anthropic, Bedrock, Vertex) for prompt caching inside batch jobs -->
- **Discount is not the whole cost story.** Batch adds: migration time, dual-run monitoring, fall-through sync cost, operational complexity on failures. Economics in the rubric must cover amortised migration, not only the discount delta.
- **Dual-run cost is double for the duration.** Running two weeks of dual-run doubles the cost for that window. The payback-horizon calculation should include this; usually it still pays, but naming the number keeps the economics honest.
- **One workload at a time.** The exercise's starter guidance is right: a single well-executed migration beats three half-finished ones. The rubric is per-workload for a reason.

### 2g. Mapping to the exercise deliverables

| Exercise asks for | Section of this solution |
|---|---|
| `docs/batch-eligibility.md` | 2a |
| `code/build_batch.py` | 2b |
| `code/run_batch.py` (submit/poll/reconcile) | 2c |
| `docs/dual-run-diff.md` | 2d |
| Fall-through test | 2e |

## 3. Validation steps

1. **Q1 has a real answer or an explicit owner.** "We don't know" is only acceptable with a named owner and a due date for the answer.
2. **`build_batch.py` byte-identity check.** `md5(build_sync_request(row))` matches the `md5(body)` of the same row's batch line. Non-identity means the diff report will be noise.
3. **Idempotency check.** Running `build_batch.py` twice on the same input produces the same output file (same md5).
4. **`custom_id` collision check.** Across two consecutive runs of `build_batch.py` with different `now_hour` values, no `custom_id` from run A appears in run B.
5. **Deadline enforcement.** With `deadline_seconds=1`, the runner cancels the batch and invokes `fallthrough` within one polling interval.
6. **Line-error surfacing.** When the provider returns a line with `"error"` populated, that line is not in `results_by_custom_id` and is instead in the fall-through stream.
7. **Span attribution.** Every batch run emits a span with `app.batch.batch_id`, `app.batch.status`, `app.batch.line_error_ratio`, `app.cost.usd`, `app.cost.pricing_shape`.
8. **Dual-run diff has both structural and semantic rates.** A single "diff rate" that collapses the two is incomplete.
9. **Peer sign-off has three roles.** Product, data owner, finance. Missing any is a hard failure.
10. **Fall-through test proves no silent loss.** Every submitted `custom_id` is either in results or in fallthrough; the union equals the input set exactly.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Q1 is answered (real SLA or explicit owner) | 1 | Freshness SLA stated in time units, cited to product. |
| Q2 structural fit is reasoned, not asserted | 1 | Line-level independence argued; cross-request state named. |
| Q3 economics covers migration + operational cost | 1 | Savings − amortised migration cost ≥ 0 within a named horizon. |
| `build_batch.py` produces byte-identical request bodies | 1 | Same function constructs sync and batch bodies. |
| `custom_id` namespaced per submission | 1 | `<batch_seq>-<row_id>`; no cross-day collisions. |
| Submit/poll/reconcile handles all four outcomes | 1 | completed / expired / failed / deadline_exceeded all covered. |
| Observability spans use chapter-04 vocabulary | 1 | `app.batch.*` + `app.cost.*` fields present. |
| Dual-run diff splits structural vs semantic | 1 | Both rates reported; eval-suite tolerance named. |
| Fall-through is tested end-to-end | 1 | Test proves no silent record loss. |
| Peer sign-off from three roles | 1 | Product, data owner, finance all named (or outstanding owner tagged). |

A submission scoring < 6 is rejected. Hard failures: (a) no Q1 answer, (b) silent record loss in the fall-through, (c) sync and batch request bodies differ in shape.

## 5. Common mistakes

- **Shipping without Q1.** The rubric exists because this is the most-skipped step. A batch migration that discovers the workload actually had a 15-minute SLA is a rollback; a rubric that writes "we don't know" with a named owner keeps the migration honest.
- **`custom_id = row_id`.** Collisions within a submission are caught by the provider; collisions *across* submissions are what reconcile silently to the wrong row. Namespace per batch.
- **Request-body drift between sync and batch.** Rewriting the body construction for the batch path is a common bug. Import the sync path's builder; the dual-run exists precisely to catch drift, and the only drift that is not a bug is semantic drift from model stochasticity.
- **Polling until the provider contract expires.** `completion_window="24h"` is the contract; the surface's SLA might be 4 hours. Set the runner's deadline to `SLA − recovery_window`, not to the contract.
- **Silent drop on line-level errors.** The provider's batch response is `completed` even when 2% of lines have `error` fields. If the reconcile step discards errored lines, those records never get re-tried; downstream data is quietly corrupted.
- **Ignoring fall-through cost.** Fall-through re-issues at sync pricing, which is the number the economics were trying to avoid. Budget for it; alert when fall-through cost is > 10% of the projected monthly savings.
- **Treating "50% off" as the only savings story.** Discount × volume is the top line; amortised migration time, operational overhead, and fall-through costs all subtract. Payback horizon is the honest number.
- **Running three migrations in parallel.** Each migration's dual-run occupies monitoring attention; running three in parallel means noise drowns signal. One at a time.
- **No cache-across-batch check for prompt-heavy workloads.** If the workload shares a long system prompt across all lines, prompt caching might or might not apply inside the batch job. Verify on the provider's current docs; if it does not, the write-side cost penalty eats some of the discount.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the batch path is a second provider surface and is therefore in scope for ML06 (AI Supply Chain). The `app.batch.line_error_ratio` metric is the control that flags content-policy and schema regressions on the batch side before they reach downstream consumers.
- MITRE ATLAS — <https://atlas.mitre.org/> — batch jobs change the detection surface: a single flagged line in a 50k-record batch is easier to miss than a single flagged sync call. ATLAS's Collection / Exfiltration tactics are the frame for why `line_error_ratio` must be a first-class metric rather than a log-only output, and why the fall-through path is a required safety control rather than an optimisation.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the three-question rubric is a GOVERN artefact (recorded decision, named sign-off roles, scheduled revisit); the dual-run diff is the MEASURE artefact; the fall-through path is the MANAGE artefact that catches the known failure mode.
- Provider batch documentation — verify endpoints, completion windows, and discount rates before shipping:
  - OpenAI Batch API: <https://platform.openai.com/docs/> <!-- verify-before-publishing: confirm current endpoint, completion-window values, discount rate, and cache-across-batch behaviour -->
  - Anthropic Message Batches: <https://docs.anthropic.com/> <!-- verify-before-publishing: confirm current API shape, discount rate, and TTL behaviour -->
  - Amazon Bedrock Batch Inference: <https://docs.aws.amazon.com/bedrock/> <!-- verify-before-publishing: confirm current batch job mechanics per model vendor -->
  - Google Vertex Batch Prediction: <https://cloud.google.com/vertex-ai/docs> <!-- verify-before-publishing: confirm current batch mechanics and pricing -->

Related exercises: [[exercise-01]] (the span vocabulary this runner extends); [[exercise-02]] (cache-inside-batch question is answered against this provider); [[exercise-04]] (`pricing_fn` and `app.cost.*` fields are reused here); [[exercise-05]] (a sustained `line_error_ratio` spike feeds the slow-burn alert); mod-106 (eval-suite tolerance is the acceptance bound for the semantic-diff rate); mod-108 [[exercise-01]] (gateway shape treats the batch endpoint as a second route).
