### exercise-03: User Feedback Triage with Argilla — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to turn the surface's **thumbs-down button, free-text report, and correction form** into a structured feedback queue that can be triaged by a human in a tool built for that purpose — in this case, **Argilla**. The deliverable is a feedback dataset schema, an ingestion pipeline that joins user feedback to the spans from [[exercise-01]], a triage workflow with labels and SLAs, and a conversion path from reviewed feedback to evaluation cases (which belong to mod-106's eval loop). The deliverable is not a sentiment model, not a dashboard, not an auto-fix system.

The failure modes the exercise is written to catch are three:

1. **Feedback lands in a spreadsheet.** Google Sheets collects thumbs-down events; nobody reviews it; two months later a pattern analysis shows the same complaint 400 times in a row and nobody knew.
2. **Feedback is reviewed, but review state is lost.** An engineer reads 20 rows, labels them in a comment column, and moves on; the next engineer starts from row 1 again.
3. **Reviewed feedback never becomes a test.** Twenty hours of triage produces a Slack summary and no artefact; the next time the same bug ships, there is no regression test that would have caught it.

The fix is a pipeline with explicit phases: **ingest → join → queue → triage → convert**. Argilla is the triage UI and the storage for review state; it is not the eval harness or the production database. The conversion step at the end is what makes the triage work pay back — a reviewed, labelled feedback row becomes an eval case or a guardrail test in mod-106/107.

This solution provides:

- A **feedback schema** on the surface side (what the thumbs-down button submits) with explicit tie-points back to the span contract in [[exercise-01]].
- An **Argilla dataset definition** (fields, questions, metadata, vectors optional) using the `argilla` Python SDK.
- An **ingestion pipeline** that hydrates feedback rows from spans and loads them into the Argilla workspace.
- A **triage workflow** — labels, priority tiers, SLA, and reviewer routing.
- A **conversion path** that exports confirmed-bug feedback rows into mod-106 eval case fixtures.
- A **grading rubric** for the instructor.

The lesson the exercise defends: *user feedback is a data pipeline, not a form submission*. A feedback button that drops into a shared drive is a visible good intention with no operational effect. The structured pipeline through Argilla is what makes the thumbs-down into a signal that closes the loop with evaluation (mod-106) and incident response ([[exercise-04]]).

## 2. Worked answer or implementation

### 2a. Feedback schema on the surface (deliverable 1)

The thumbs-down UI produces one JSON blob. Fields the surface sets at submission time:

```json
{
  "feedback_id": "fb_01HZX8W3QG6YYE1TDC9JRV64EM",
  "submitted_at": "2026-11-14T15:04:11Z",
  "surface": "helpdesk-reply-copilot",
  "feature": "helpdesk.reply",
  "session_id": "sess_7f2e8d",
  "request_id": "req_9d3a01",
  "signal": "thumbs_down",
  "severity_user_reported": "medium",
  "free_text": "It made up a ticket ID.",
  "correction": null,
  "client_build": "web-2026.11.07"
}
```

Tie-points:

- `session_id` → `openinference.session.id` on the span.
- `request_id` → `app.request_id` on the span.
- `feature` → `app.feature` on the span.
- `submitted_at` is produced **by the surface**, not the user's clock. The span's `t_closed` is close but not identical — use `submitted_at` for ingest-side ordering; use the span's timestamp for the correlation to the LLM call.

Rules:

- **No PII in `free_text` at rest.** The surface runs the same scrubber from [[exercise-02]] on the submission path before persisting; `scrub.*` metadata tags the row.
- **Correction is structured where possible.** When the surface offers an inline edit ("reply with the fix"), the corrected text is stored in `correction`. A free-text complaint without a correction stays in `free_text`.
- **`signal` is a short enum, not a sentence.** `thumbs_down`, `thumbs_up`, `report_hallucination`, `report_safety`, `correction_submitted`. Free-text goes in `free_text`.

### 2b. Argilla dataset definition (deliverable 2)

```python
# datasets/define_feedback_dataset.py
# Uses the current argilla Python SDK (v2 generation) with Settings / Fields / Questions.
# Verify the SDK API against the current argilla docs before running.
import argilla as rg

client = rg.Argilla(api_url="https://argilla.internal", api_key="…")

settings = rg.Settings(
    guidelines=(
        "Triage user feedback for the helpdesk-reply-copilot surface.\n"
        "Read the prompt, the model's response, and the free-text / correction.\n"
        "Confirm or reject the signal; label the failure mode; tag downstream "
        "actions (eval-case, guardrail, incident)."
    ),
    fields=[
        rg.TextField(name="prompt",      title="User prompt",           use_markdown=True),
        rg.TextField(name="response",    title="Model response",        use_markdown=True),
        rg.TextField(name="correction",  title="User correction",       required=False),
        rg.TextField(name="free_text",   title="User free-text",        required=False),
    ],
    questions=[
        rg.LabelQuestion(
            name="confirmed",
            title="Signal confirmed?",
            labels=["confirmed", "rejected", "duplicate", "cannot-tell"],
            required=True,
        ),
        rg.LabelQuestion(
            name="failure_mode",
            title="Failure mode (if confirmed)",
            labels=[
                "hallucination",
                "wrong-tool-call",
                "unhelpful",
                "policy-refusal",
                "latency",
                "format",
                "safety",
                "other",
            ],
            required=False,
        ),
        rg.MultiLabelQuestion(
            name="downstream_actions",
            title="Downstream actions",
            labels=["eval-case", "guardrail-test", "incident", "noop"],
            required=False,
        ),
        rg.TextQuestion(
            name="reviewer_notes",
            title="Reviewer notes",
            required=False,
        ),
    ],
    metadata=[
        rg.TermsMetadataProperty(name="feature",  title="Feature",  values=["helpdesk.reply","search.compose","pricing.summarise"]),
        rg.TermsMetadataProperty(name="signal",   title="Signal",   values=["thumbs_down","thumbs_up","report_hallucination","report_safety","correction_submitted"]),
        rg.IntegerMetadataProperty(name="latency_total_ms", title="Total latency (ms)"),
        rg.FloatMetadataProperty(name="cost_usd",            title="Cost (USD)"),
        rg.TermsMetadataProperty(name="model",   title="Served model"),
        rg.TermsMetadataProperty(name="outcome", title="Outcome", values=["success","refusal","error","cancelled"]),
    ],
    # Suggestions let the ingester pre-populate an LLM-produced guess for the
    # label; the reviewer confirms or overrides. This is where suggestions
    # should live, not in the "confirmed" field itself.
    allow_extra_metadata=False,
)

dataset = rg.Dataset(
    name="helpdesk_reply_feedback",
    workspace="applied-ai-feedback",
    settings=settings,
    client=client,
)
dataset.create()
# <!-- verify-before-publishing: cross-check Field/Question/Metadata class names
#      against the current argilla Python SDK docs (https://docs.argilla.io/).
#      v2 API uses rg.Settings; earlier versions use FeedbackDataset.
# -->
```

### 2c. Ingestion pipeline (deliverable 3)

The pipeline runs continuously (or as a short-period job) and ingests feedback rows into Argilla with the span-derived context pre-populated. Suggestions (LLM-produced guesses for `failure_mode`) are optional; they speed triage, they do not replace it.

```python
# jobs/ingest_feedback_to_argilla.py
from typing import Iterable
import argilla as rg

def hydrate(feedback: dict, span: dict | None) -> rg.Record:
    """Build an Argilla record from a feedback blob + its correlated span."""
    attrs = (span or {}).get("attributes", {})
    events = (span or {}).get("events", [])
    prompt_text = next((e["attributes"].get("gen_ai.prompt.json","") for e in events
                        if e["name"] == "gen_ai.content.prompt"), "")
    response_text = next((e["attributes"].get("gen_ai.completion.text","") for e in events
                          if e["name"] == "gen_ai.content.completion"), "")

    suggestions = []
    guess = classify_failure_mode(feedback["free_text"], response_text)   # cheap heuristic/LLM
    if guess:
        suggestions.append(rg.Suggestion(
            question_name="failure_mode", value=guess, agent="heuristic-v1"
        ))

    return rg.Record(
        id=feedback["feedback_id"],                      # idempotent re-ingest
        fields={
            "prompt":     prompt_text,
            "response":   response_text,
            "correction": feedback.get("correction") or "",
            "free_text":  feedback.get("free_text") or "",
        },
        metadata={
            "feature":           feedback["feature"],
            "signal":            feedback["signal"],
            "latency_total_ms":  int(attrs.get("app.latency.total_ms", 0)),
            "cost_usd":          float(attrs.get("app.cost.usd", 0.0)),
            "model":             attrs.get("gen_ai.response.model", "unknown"),
            "outcome":           attrs.get("app.outcome", "unknown"),
        },
        suggestions=suggestions,
    )


def run_batch(batch: Iterable[dict], ds: rg.Dataset, trace_store):
    records = []
    for fb in batch:
        span = trace_store.lookup(request_id=fb["request_id"],
                                  session_id=fb["session_id"])
        records.append(hydrate(fb, span))
    ds.records.log(records)   # idempotent on record.id
```

Operational notes:

- **Join on `request_id` first, `session_id` second.** A session with multiple turns has multiple spans; the request id is the single-call key.
- **Idempotent on `feedback_id`.** Argilla's record `id` is the dedup key; re-ingesting the same feedback blob updates the record rather than duplicating it.
- **Backfill is the same code.** A new failure-mode label added to the dataset does not require a new ingester — the records already carry the context.
- **Suggestions are produced by cheap classifiers.** The reviewer's job is to confirm, not re-read every row from zero. A heuristic that is 60% right is already a win.

### 2d. Triage workflow and SLAs (deliverable 4)

```markdown
# docs/feedback-triage-workflow.md

## Who triages
Primary: applied-AI engineer on rotation (weekly).
Backup: feature PM for ambiguous cases.
Secondary (auto): LLM suggestion agent writes `suggestions`; it does not set
`confirmed` or `downstream_actions`.

## Priority tiers (set at ingest based on signal)
- P1 (24h SLA): signal == "report_safety" OR failure_mode suggestion == "safety"
- P2 (3d SLA): signal == "report_hallucination" OR correction present
- P3 (7d SLA): thumbs_down with free_text only
- P4 (best effort): thumbs_up (used to calibrate positive-signal baseline)

## Review flow
1. Open the Argilla dataset; filter `confirmed is null` and sort by priority.
2. Read prompt + response + correction + free_text.
3. Set `confirmed` (confirmed / rejected / duplicate / cannot-tell).
4. If `confirmed`, set `failure_mode` and `downstream_actions`.
5. If `downstream_actions` contains "incident", page the on-call via the
   runbook in [[exercise-05]] — do not skip; the Argilla record alone is
   not an alert.
6. If "eval-case", add a note identifying the mod-106 eval suite to
   contribute to; the conversion step in 2e picks it up.

## SLA enforcement
A P1 record older than 24h fires an alert from the backend scheduled job
(see 2e). P2 older than 3d fires a slack message to the rotation owner.

## Volume signal
If weekly P2 volume for a feature exceeds its control-chart band for two
consecutive weeks, a PM review is scheduled. The band is computed over the
last 12 weeks of data and recorded in the policy doc.
```

### 2e. Conversion to eval cases (deliverable 5)

Reviewed-and-confirmed rows with `downstream_actions` containing `eval-case` are exported to the mod-106 eval suite as fixtures.

```python
# jobs/export_confirmed_to_evals.py
import argilla as rg
import json
from pathlib import Path

def export_eval_cases(ds: rg.Dataset, out_dir: Path) -> int:
    out_dir.mkdir(parents=True, exist_ok=True)
    n = 0
    # Argilla v2 query API; verify against current SDK docs.
    for rec in ds.records(query=rg.Query(
            # confirmed == confirmed AND downstream_actions contains eval-case
        )).to_list():
        if not _is_eval_case(rec):
            continue
        case = {
            "id": rec.id,
            "feature": rec.metadata["feature"],
            "prompt": rec.fields["prompt"],
            "expected_behavior": rec.fields["correction"] or "see reviewer notes",
            "failure_mode": _value(rec.responses, "failure_mode"),
            "notes": _value(rec.responses, "reviewer_notes"),
            "source": "user_feedback",
            "created_from_feedback_at": _iso_now_from_rec(rec),
        }
        (out_dir / f"{rec.id}.json").write_text(json.dumps(case, indent=2))
        n += 1
    return n


def _is_eval_case(rec) -> bool:
    responses = rec.responses or {}
    confirmed = _value(responses, "confirmed")
    actions = _value(responses, "downstream_actions") or []
    return confirmed == "confirmed" and "eval-case" in actions


def _value(responses, name):
    # Argilla returns a response list keyed by user; use the first final value.
    # Verify shape against current SDK.
    r = (responses or {}).get(name) or []
    return r[0].value if r else None
```

The exported JSON lands in `evals/cases/<feature>/` and is picked up by the mod-106 eval harness's file loader. The eval harness re-runs the case against the current model and flags regressions — closing the loop.

### 2f. Governance (deliverable 6)

```markdown
## Review cadence
- Weekly: rotation owner closes the P1/P2 queue to zero before end of shift.
- Monthly: PM + applied-AI engineer review the failure-mode distribution
  and open a work item for any mode that exceeded its band.
- Quarterly: review the schema; add / retire labels based on observed
  misclassifications and new feature coverage.

## Ownership
Owner: applied-AI platform (role, not person).
Argilla workspace admin: platform-AI tech lead.

## Data retention
Argilla records retain 180 days by default. Confirmed records tagged
"eval-case" or "guardrail-test" are exported to the eval repo and retained
indefinitely as test fixtures.
```

### 2g. Mapping to the exercise deliverables

| Exercise asks for | Section of this solution |
|---|---|
| Surface-side feedback schema | 2a |
| Argilla dataset definition | 2b |
| Ingestion pipeline (span ↔ feedback join) | 2c |
| Triage workflow + SLAs | 2d |
| Conversion to eval cases | 2e |
| Governance / review cadence | 2f |

## 3. Validation steps

1. **Every feedback row joins to a span.** Run a day's ingest; assert ≥ 98% of records have non-empty `prompt`/`response` fields. Failures mean the trace was sampled out (see [[exercise-02]]) or the request id did not propagate.
2. **Idempotency.** Re-ingest the same feedback blob; record count unchanged.
3. **PII scrubbed before Argilla sees it.** Inject an email in `free_text`; assert the Argilla record stores the `[REDACTED:email]` form and `scrub.*` on the source span matches.
4. **Suggestions do not pre-set `confirmed`.** Assert the `confirmed` question is `null` on fresh ingest even when a `failure_mode` suggestion is present.
5. **P1 SLA alert fires.** Mark a record as P1 with a 25h-old `submitted_at`; assert the SLA job emits an alert.
6. **Conversion produces runnable eval cases.** Confirm and tag a record with `eval-case`; run the conversion; assert a file exists in `evals/cases/<feature>/` with the fields in 2e, and the mod-106 harness loads it without error.
7. **Backfill works on a schema change.** Add a new `failure_mode` label; existing records show the new label as an option; previously-labelled records are unchanged.
8. **Reviewer attribution.** Argilla records store per-user response history; the attribution survives through the eval-case export's `notes` field.
9. **Volume control chart is defensible.** Weekly P2 band is computed from the last 12 weeks; document shows the numbers, not a hand-wave.
10. **Owner + review cadence documented.** The workflow doc names the role owner and next-review date.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Feedback schema with span tie-points | 1 | `request_id`, `session_id`, `feature` present and documented. |
| Argilla dataset: fields / questions / metadata | 2 | Prompt + response + correction + free_text as fields; `confirmed` + `failure_mode` + `downstream_actions` as questions. |
| Suggestions produced by a cheap classifier | 1 | `Suggestion` objects attached at ingest; reviewer confirms, does not read cold. |
| Ingestion joins feedback to spans | 1 | Join on `request_id` first; fallback to `session_id`. |
| PII scrubbed before Argilla sees it | 1 | Same scrubber as [[exercise-02]] applied on the submission path. |
| Triage workflow with priority tiers + SLA | 1 | P1 safety / P2 hallucination / P3 free-text; SLA alert on breach. |
| Conversion path to eval cases | 1 | Exports JSON that the mod-106 harness loads. |
| Governance and ownership documented | 1 | Owner by role; review cadence; retention policy. |
| Validation harness exists | 1 | At least the join-rate and idempotency checks are automated. |

A submission scoring < 6 is rejected. Hard failures: (a) no span join (free-floating thumbs-down), (b) no conversion to eval cases, (c) PII leaks into Argilla.

## 5. Common mistakes

- **Feedback with no span join.** The thumbs-down is a signal only if you can see what the model actually said. Request id is the join key; without it, triage becomes guesswork.
- **Suggestions used as ground truth.** A classifier guess goes on `Suggestion`, not on the `confirmed` field. Reviewers confirm; otherwise the dataset is a classifier-on-itself loop.
- **Free-text is the only field.** A free-text complaint is the user's description of the failure; the schema also needs the model's response and the user's correction. Argilla field design enforces this.
- **Priority by signal count.** Two safety reports are P1; 200 thumbs-down are P3. Priority is on the signal type, not aggregate volume.
- **No conversion step.** Triage pays back when a confirmed row becomes a test. Without the export, the triage is read-and-forget.
- **PII lands in Argilla.** Argilla is an internal tool; it is still storage. The scrubber from [[exercise-02]] runs on the submission path, not inside Argilla.
- **No backfill plan on schema change.** Adding a new failure-mode label without updating the dataset settings means the new label is not selectable for old rows that may need it.
- **Owner by name.** Reorgs happen; owner is a role (`applied-AI platform`) not a person.
- **Retention is implicit.** Argilla default retention and eval-repo retention are different; the governance doc states both.
- **The reviewer is the model.** Using an LLM to set `confirmed` closes the feedback loop into a model-on-model echo chamber. Suggestions are advisory; confirmation is human.

## 6. References

- Argilla documentation — <https://docs.argilla.io/> — authoritative source for the dataset model (`Settings`, `Field`, `Question`, `Suggestion`, `Record`) used in 2b, 2c, and 2e. Argilla v2 (the generation this solution targets) consolidated feedback datasets under the unified dataset model; earlier versions used `FeedbackDataset`. <!-- verify-before-publishing: pin to the Argilla SDK version in use (e.g. argilla >= 2.0.0) and confirm Field / Question / Query class names against the current docs. -->
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the MEASURE → MANAGE loop closes only when user-observed failures feed back into evaluations and controls. The feedback-to-eval conversion in 2e is the artefact that closes that loop for an LLM feature.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML09 (Output Integrity) depends on being able to re-read the actual model output when a user reports it was wrong. The ingestion join in 2c is what makes that evidence trail usable. ML02 (Data Poisoning) and ML10 (Model Poisoning) show up as patterns in confirmed feedback that cluster around an input-side manipulation; the failure-mode label distribution is where those patterns become visible.
- MITRE ATLAS — <https://atlas.mitre.org/> — Collection and Exfiltration tactics against feedback-driven loops exploit the suggestion → confirmation path. The workflow in 2d is explicit that an LLM suggestion does not set `confirmed`; reviewer confirmation is a Response control that blocks the Collection path from closing via the dataset.

Related exercises: [[exercise-01]] (spans the feedback joins to); [[exercise-02]] (scrubs the submission on its way in); [[exercise-04]] (safety-labelled feedback can escalate to an incident playbook); [[exercise-05]] (status page per-feature "thumbs-down rate" panel reads the feedback metric); mod-106 (the eval suite the confirmed rows become fixtures for); mod-107 (the guardrail tests that `guardrail-test`-tagged rows feed).
