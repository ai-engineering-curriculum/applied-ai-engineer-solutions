### exercise-02: Trace Snapshot as Regression Fixture — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to treat production **traces as the source of fixtures** for the offline suite from [[exercise-01]]: when a user's reply is bad, the trace that produced it is captured, redacted, converted into a frozen regression row, and committed. The next deploy that would produce the same bad behaviour fails CI. The failure mode the exercise is written to catch is the one where every incident post-mortem ends with "we'll add a test," no test gets added, and the same bug ships twice.

The core idea is that a *trace* — the structured record of what the agent saw and did on a single request — already contains almost everything an eval row needs: the input, the retrieved context, the tool calls, the model output, and the surface's structured artefacts. The missing pieces are (a) a scorer that encodes what *should* have happened, (b) the redaction pipeline, and (c) the governance that keeps the fixture library from rotting.

This solution provides:

- A **trace schema** that captures enough of a request to replay it deterministically against the production controller.
- A reference **`trace_to_fixture.py`** that turns a captured trace into a `evals/rows/regression/*.yaml` row, with explicit redaction of PII and tenant-sensitive content.
- A reference **assertion library** so a fixture can encode what the correct behaviour is in terms the runner can grade (`must_cite`, `must_not_tool_call`, `must_refuse`, `must_pause_with`, `judge_llm_bar`).
- A **retention and retirement policy** for the fixture library.
- A **grading rubric** for the instructor.

The lesson is that *a regression fixture is a frozen artefact of an incident, not a general-purpose test case*. A trace-sourced row is strictly stronger than a hand-authored one on the incidents it encodes, because it captures the exact prompt, context, and tool sequence that produced the regression — the things engineers get wrong when they try to remember a bug from memory.

## 2. Worked answer or implementation

### 2a. Trace schema (deliverable 1)

A trace is the structured record the surface emits for every request. The minimum schema for replayability:

```yaml
# trace record — JSON-serialisable; one per request.
trace_id: tr_01HTZ…           # unique id
captured_at: "2026-11-02T14:22:03Z"
surface: helpdesk-reply-copilot
surface_version: "0.14.3"
agent_controller_version: "1.4.0"
judge_pin:                     # if an inline judge was called in prod
  model: "claude-eval-judge-v3"
  sha: "sha256:4b0f…"
principal:                     # authenticated identity at request time
  tenant_id: "t_acme"
  user_id: "u_alice"           # pre-redaction; the pipeline strips this
  roles: ["agent"]
input:
  user_message: "Hey, we need to refund $40 on ticket ZD-442277…"
  session_id: "s_ab12"
retrieved_context:             # deterministic re-derivable set
  retriever_version: "r_v8"
  pack:
    - { id: "KB-1021", title: "Refund policy", body: "…", score: 0.84 }
    - { id: "KB-0914", title: "Ticket conventions", body: "…", score: 0.62 }
tool_log:
  - step: 1
    tool: "lookup_ticket"
    args: { ticket_id: "ZD-442277" }
    result: { customer_id: "c_7742", status: "open", add_on: "pro_tier" }
    schema_sha: "sha256:…"
    fired: true
    after_confirm: false
  - step: 2
    tool: "create_followup_task"
    args: { ticket_id: "ZD-442277", summary: "escalate refund approval" }
    result: null
    fired: false
    after_confirm: true
    pause_reason: "confirm_write"
model_output:
  draft: |
    Thanks for reaching out — I've escalated this $40 refund to a
    manager for approval; you'll hear back within one business day.
  citations: []                 # the bug: KB-1021 not cited
outcome:
  user_rating: "thumbs_down"
  user_comment: "You didn't need to escalate; policy allows <$100."
  linked_incident: "INC-2026-09-14"
```

The invariants that make the trace replayable:

- `surface_version` + `agent_controller_version` pin the code that produced it.
- `retrieved_context.pack` is captured **as it was shown to the model** — replay does not re-run the retriever against a moving corpus.
- `tool_log` captures each tool's `schema_sha` so a replay can detect if the tool's schema has since changed (replay still runs, but the fixture is flagged `stale: true` until it's revalidated).
- `judge_pin` is recorded when an inline judge fired in prod, so the fixture can be scored against the same judge checkpoint (or explicitly against a newer one for calibration).

### 2b. `trace_to_fixture.py` — the conversion pipeline (deliverable 2)

```python
"""trace_to_fixture.py — convert a captured trace into a regression fixture.

Usage:
    python -m trace_to_fixture TRACE_PATH FIXTURE_PATH \
        --incident INC-2026-09-14 \
        --assertion "must_cite:KB-1021" \
        --assertion "must_not_tool_call:create_followup_task"
"""

from __future__ import annotations

import argparse
import hashlib
import json
import pathlib
import re
import sys
from typing import Any

import yaml

from redactor import redact_pii, redact_tenant_fields


ASSERTION_PATTERN = re.compile(
    r"^(must_cite|must_not_cite|must_refuse|must_pause_with|"
    r"must_tool_call|must_not_tool_call|judge_llm_bar):(.+)$"
)


def convert(trace_path: str, fixture_path: str, incident: str,
            assertions: list[str], notes: str) -> None:
    trace = json.loads(pathlib.Path(trace_path).read_text())

    redacted = _redact(trace)
    rubric = _build_rubric(assertions)

    fixture = {
        "id": _id_from_trace(redacted),
        "source": {
            "kind": "trace_snapshot",
            "trace_id": redacted["trace_id"],
            "captured_at": redacted["captured_at"],
            "incident": incident,
            "surface_version": redacted["surface_version"],
            "agent_controller_version": redacted["agent_controller_version"],
            "schema_shas": _tool_schema_shas(redacted),
        },
        "fixture": {
            "session_id": redacted["input"]["session_id"],
            "principal": redacted["principal"],
            "user_message": redacted["input"]["user_message"],
            "knowledge_base": _kb_from_pack(redacted),
        },
        "scoring": {
            "rubric": rubric,
            "pass_at": 1.0,          # regression rows are binary by default
        },
        "notes": notes,
        "retire_on": None,            # see §2d retention policy
    }

    pathlib.Path(fixture_path).write_text(
        "# Auto-generated from trace " + redacted["trace_id"] + "\n"
        + yaml.safe_dump(fixture, sort_keys=False)
    )


def _redact(trace: dict) -> dict:
    out = json.loads(json.dumps(trace))
    out["principal"] = redact_tenant_fields(out["principal"])
    out["input"]["user_message"] = redact_pii(out["input"]["user_message"])
    for p in out["retrieved_context"]["pack"]:
        p["body"] = redact_pii(p["body"])
    for entry in out["tool_log"]:
        entry["args"] = redact_pii(entry["args"])
        if entry.get("result"):
            entry["result"] = redact_pii(entry["result"])
    out["model_output"]["draft"] = redact_pii(out["model_output"]["draft"])
    return out


def _build_rubric(assertions: list[str]) -> list[dict]:
    rubric: list[dict] = []
    for a in assertions:
        m = ASSERTION_PATTERN.match(a)
        if not m:
            raise ValueError(f"malformed assertion {a!r}")
        kind, arg = m.group(1), m.group(2)
        rubric.append(_assertion_to_rubric(kind, arg))
    if not rubric:
        raise ValueError("at least one assertion is required")
    return rubric


def _assertion_to_rubric(kind: str, arg: str) -> dict:
    if kind == "must_cite":
        return {"kind": "contains_citation", "to": arg, "weight": 1.0}
    if kind == "must_not_cite":
        return {"kind": "not_contains_citation", "to": arg, "weight": 1.0}
    if kind == "must_refuse":
        return {"kind": "refusal_present", "reason": arg, "weight": 1.0}
    if kind == "must_pause_with":
        return {"kind": "pause_reason_matches", "want": arg, "weight": 1.0}
    if kind == "must_tool_call":
        return {"kind": "tool_matched", "tool": arg, "weight": 1.0}
    if kind == "must_not_tool_call":
        return {"kind": "no_tool_call", "tool": arg, "weight": 1.0}
    if kind == "judge_llm_bar":
        prompt, bar = arg.rsplit("@", 1)
        return {
            "kind": "judge_llm",
            "prompt": prompt,
            "weight": 1.0,
            "pass_at": float(bar),
        }
    raise ValueError(f"unknown assertion kind {kind}")


def _kb_from_pack(trace: dict) -> list[dict]:
    pack = trace["retrieved_context"]["pack"]
    return [{"id": p["id"], "title": p["title"], "body": p["body"]}
            for p in pack]


def _tool_schema_shas(trace: dict) -> dict[str, str]:
    return {e["tool"]: e.get("schema_sha", "")
            for e in trace["tool_log"] if "schema_sha" in e}


def _id_from_trace(trace: dict) -> str:
    salt = (trace["trace_id"] + trace["surface_version"]).encode()
    return "regression_" + hashlib.sha1(salt).hexdigest()[:10]


if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("trace")
    ap.add_argument("fixture")
    ap.add_argument("--incident", required=True)
    ap.add_argument("--assertion", action="append", required=True)
    ap.add_argument("--notes", default="")
    args = ap.parse_args()
    convert(args.trace, args.fixture, args.incident, args.assertion,
            args.notes)
```

The conversion is intentionally *explicit* about assertions. The CLI refuses to produce a fixture without at least one `--assertion`, because a regression fixture whose scorer is "did the output change" is a change-detector, not a regression test. The engineer writing the row must name the invariant the fix enforces.

### 2c. The resulting fixture

Given the trace in §2a and the command:

```shell
python -m trace_to_fixture traces/tr_01HTZ.json \
  evals/rows/regression/r_refund_self_serve.yaml \
  --incident INC-2026-09-14 \
  --assertion "must_cite:KB-1021" \
  --assertion "must_not_tool_call:create_followup_task" \
  --assertion "judge_llm_bar:Score 1 if the reply correctly identifies this as a self-serve sub-$100 refund and does not escalate; 0 otherwise.@1.0" \
  --notes "Refund escalation regression; fix landed in PR #1194 — do not drop below 1.0."
```

the pipeline emits:

```yaml
# Auto-generated from trace tr_01HTZ…
id: regression_a1b2c3d4e5
source:
  kind: trace_snapshot
  trace_id: tr_01HTZ…
  captured_at: "2026-11-02T14:22:03Z"
  incident: INC-2026-09-14
  surface_version: "0.14.3"
  agent_controller_version: "1.4.0"
  schema_shas:
    lookup_ticket: "sha256:…"
    create_followup_task: "sha256:…"
fixture:
  session_id: s_ab12
  principal: { tenant_id: t_redacted, user_id: u_redacted, roles: [agent] }
  user_message: |
    Hey, we need to refund $40 on ticket ZD-REDACTED…
  knowledge_base:
    - { id: KB-1021, title: "Refund policy", body: "…" }
    - { id: KB-0914, title: "Ticket conventions", body: "…" }
scoring:
  rubric:
    - { kind: contains_citation, to: KB-1021, weight: 1.0 }
    - { kind: no_tool_call, tool: create_followup_task, weight: 1.0 }
    - kind: judge_llm
      prompt: |
        Score 1 if the reply correctly identifies this as a self-serve
        sub-$100 refund and does not escalate; 0 otherwise.
      weight: 1.0
      pass_at: 1.0
  pass_at: 1.0
notes: >
  Refund escalation regression; fix landed in PR #1194 — do not drop below 1.0.
retire_on: null
```

The row is pin-compatible with the runner from [[exercise-01]] — it drops into `evals/rows/regression/` and the no-regression-row block in `thresholds.yaml` is updated by the fix PR.

### 2d. Retention and retirement policy (deliverable 3)

Regression fixtures accumulate. Without retention policy, the suite rots into 2,000 rows nobody understands. The rules:

1. Every fixture has a **source incident** (`source.incident`). A fixture whose incident link is dead (deleted ticket, merged into another incident) is reviewed on the next quarterly.
2. A fixture may set `retire_on:` to a future date **only if** the same invariant is covered by another non-regression row with an explicit `covers_incident: INC-…` pointer. The intent is to replace the frozen snapshot with a hand-authored row that is clearer to read.
3. `retire_on: null` means the fixture stays until explicitly retired. Retirement requires a PR that (a) deletes the fixture, (b) adds a line to `evals/CHANGES.md` naming the incident and the replacement row, and (c) is signed off by a second owner.
4. **Schema drift triggers a flag, not a deletion.** If a tool's `schema_sha` in the fixture no longer matches the current schema, the fixture is tagged `stale: true` and the runner warns. The oncall applied-AI engineer owns re-authoring the row against the new schema within 14 days.
5. **Model-family drift triggers a recalibration PR**, not a deletion. When the production model is moved (see downstream model-routing exercises), regression rows that fail on the new model are triaged: real regressions get a model-level hotfix; false alarms get the judge bar re-cut in a dedicated PR.

### 2e. Capture pipeline (deliverable 4)

The production side needs to emit traces in a form the pipeline can read. Two capture modes:

- **Always-on sampled capture.** Every request emits a trace to an append-only store at a 2% rate; low-grade rows are retained 14 days and purged. When a thumbs-down + a human-flag land on the same `trace_id`, the trace is promoted to the long-retention bucket and queued for fixture conversion.
- **On-demand capture for incidents.** When an incident pager fires, the oncall runs `agentctl trace capture --session s_ab12` which forces the surface to emit the trace for that specific session_id (if still within the sliding capture window). The trace is attached to the incident ticket.

Both modes write to the same schema. Both go through the same redactor before leaving the trusted boundary. The redaction pass is non-skippable; the CLI refuses to convert a trace that is not marked `redacted: true` in its envelope metadata.

### 2f. Implementation notes

- **A fixture is an incident, not a dataset row.** The temptation is to sample 10,000 traces and bulk-convert them into fixtures. This produces a suite that is large, redundant, slow, and uninterpretable. Convert on incident; keep each fixture's incident link visible.
- **Replay must use the trace's retrieved context verbatim.** Re-running the retriever against the live corpus at replay time means the retriever changes can silently mask or unmask the bug. The fixture freezes what the model saw.
- **Redaction is boundary-enforced.** The redactor is a library; the CLI refuses to convert a trace whose envelope is not marked redacted. The redactor is audited on the same cadence as the model-routing privacy review.
- **Assertions are the engineering, not the trace.** The trace captures what happened; the assertion captures what *should* happen. The assertion is hand-authored by the engineer who owns the fix. A fixture without an explicit assertion is a change-detector, not a regression test.
- **`schema_sha` is the staleness oracle.** When a tool's schema changes, the fixture is flagged, not silently broken. The oncall rewrites the row against the new schema within 14 days; the fix PR for the schema change is responsible for queuing the rewrite.
- **The pipeline is one-way.** Fixtures are not edited by hand; they are rebuilt from the trace + a new assertion set when the assertions need to change. Hand-edits are forbidden by the CI linter (the generated-file header `# Auto-generated from trace …` is required for any row under `evals/rows/regression/`).
- **One incident can produce more than one fixture.** A multi-turn incident often needs one fixture per turn (the first user message, the follow-up after the pause, etc.). The fixtures share an `incident` field but have distinct `trace_id`s.

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. A trace schema exists with `surface_version`, `agent_controller_version`, `retrieved_context.pack`, `tool_log[].schema_sha`, and `judge_pin` (when applicable).
2. `trace_to_fixture.py` refuses to produce a fixture without an `--incident` flag and at least one `--assertion`.
3. The conversion pipeline passes the trace through `redact_pii` and `redact_tenant_fields` and refuses traces without the `redacted: true` envelope.
4. At least one worked example row in `evals/rows/regression/` carries an auto-generated header and a `source.incident` reference.
5. The retention policy document (`evals/RETENTION.md`) names the four lifecycle states (`active`, `stale`, `retire-queued`, `retired`), the triggers between them, and the owner.
6. `schema_sha` mismatches at run time print a `stale: true` warning without failing the row — but the fixture is queued for rewrite within 14 days.
7. Hand-edits to files in `evals/rows/regression/` are rejected by a CI linter (the generated-file header is required).
8. The capture pipeline is documented: always-on-sampled + on-demand on incident, with the sampling rate and retention days named.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Trace schema is replayable | 1 | `retrieved_context.pack` + `tool_log[].schema_sha` + version pins. |
| `trace_to_fixture.py` requires `--incident` + ≥ 1 `--assertion` | 1 | CLI refuses underspecified calls. |
| Redactor runs before fixture write; envelope is checked | 1 | Non-redacted traces refused by the CLI. |
| Fixture rows carry `source.incident` and `source.schema_shas` | 1 | At least one example row demonstrates both fields. |
| Assertions map to runner rubric kinds | 1 | ≥ 5 of the 7 assertion kinds implemented. |
| Retention policy document covers lifecycle states | 1 | Four states + triggers + owner named. |
| `schema_sha` mismatch is a soft flag, not a hard fail | 1 | Runner emits `stale: true` warning; 14-day rewrite SLO. |
| CI linter rejects hand-edits to regression rows | 1 | Generated-file header required; linter blocks violations. |
| Capture pipeline specifies always-on sample + on-demand | 1 | Sampling rate + retention policy named. |
| One multi-turn incident yields more than one fixture | 1 | Example demonstrates the one-incident-many-fixtures case. |

A submission scoring < 6 is rejected. The hard failures are (a) no redaction, (b) fixtures without an assertion (change-detectors), (c) hand-editable regression rows (no linter), (d) replay re-running the live retriever.

## 5. Common mistakes

- **Treating the trace as a test case directly.** A trace captures what happened. A test captures what should have happened. The assertion is the engineering step that bridges them; without it the row is a change-detector and will fail on every refactor.
- **Bulk-converting traces to grow the suite.** The suite's value is per-incident coverage, not per-row volume. Convert on incident, one at a time.
- **Re-running the retriever at replay time.** The fixture then measures the retriever's current state, not the agent's behaviour on the captured context. Freeze the pack.
- **Skipping redaction for "internal" traces.** Internal data is still tenant data. Redaction is boundary-enforced; the pipeline must refuse to convert non-redacted traces.
- **Hand-editing regression rows.** The next engineer cannot audit the row against the original trace. The CI linter is the fix.
- **No `schema_sha`.** When the tool schema changes, the fixture runs green against the old shape silently. The `schema_sha` + staleness flag is the detector.
- **No `retire_on` policy.** The suite bloats with rows nobody owns. Retention + a second-owner-signoff retirement path keeps the suite clean.
- **Judge bar frozen against a model that has moved.** When the model family changes, the judge bar may need recalibration. Recalibrate in a dedicated PR under the threshold-change protocol from [[exercise-01]].
- **One assertion per fixture when the incident had three invariants.** A complex incident often needs `must_cite + must_not_tool_call + judge_llm_bar` to encode what "fixed" means. Under-asserting leaves the bug in the next bisect.
- **Capturing traces only on paging incidents.** Low-grade thumbs-downs are the leading indicator; the always-on sampled capture plus a thumbs-down filter catches regressions before they page.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the per-incident fixture library, the retention policy, and the auditable linkage from fixture to incident implement the RMF's **MANAGE** function requirement that incidents produce artefacts that persist beyond the incident and feed back into measurement. The schema-drift + staleness flag maps to the **MEASURE** function's documentation-currency expectation.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the redaction pipeline is the surface's answer to the output-integrity items in the OWASP top 10 as they apply to inadvertent PII or tenant-sensitive content leaking into the fixture library.
- MITRE ATLAS — <https://atlas.mitre.org/> — adversarial traces (prompt-injection, cross-tenant leak attempts) that the surface catches in prod should be converted to regression fixtures on the same cadence as benign incidents; ATLAS technique entries are the canonical vocabulary for labelling the incident.

Related exercises: [[exercise-01]] (regression rows land in the no-regression block in `thresholds.yaml`); [[exercise-03]] (the CI gate runs regression rows on every PR); [[exercise-04]] (online eval loop writes traces to the capture store); [[exercise-05]] (persona dashboards surface regression-row verdicts alongside aggregate scores).
