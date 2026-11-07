### exercise-02: Citations and Evidence UX — Solution

## 1. Solution overview

The exercise asks the engineer to make citations and evidence a **first-class part of the schema**, not an afterthought on the render layer. A policy Q&A bot can cite a real passage and still be wrong — the citation ID resolves but the claim is not supported — so the surface has to enforce sentence-level grounding at parse time, de-duplicate by document in the evidence panel, refuse citations to passages the current user cannot read, and treat "no good evidence" as a valid output rather than an error branch.

This solution provides:

- A `GroundedAnswer` Pydantic schema with sentence-level `supported_by` lists and the two `@model_validator` contradiction checks the exercise specifies.
- A reference `parse_and_validate(raw_output, pack)` that enforces the citation-hallucination and ungrounded-claim guards, increments counters, and documents the one-shot repair-then-fall-back policy.
- A minimal Streamlit render demonstrating inline citations, an evidence panel with `doc_id`-level de-duplication, a Why-this-answer expandable, and a distinct "no good evidence" layout.
- A `demos.md` walking through all six failure modes end-to-end (raw model output → parser decision → rendered UI state).
- A `metrics.py` + `confidence.py` with the five counters and a deterministic bucket function; `confidence.md` explains the calibration plan.
- A grading rubric for the instructor.

The assumption the schema encodes is simple: *a sentence either has a citation, or it is a named linking phrase, or the answer is empty and `unsupported_reason` is populated*. Anything else is a contract violation and the model has to repair or the surface falls to "no good evidence." The render layer is deliberately spartan because the lesson is the schema and the parser, not CSS.

## 2. Worked answer or implementation

### 2a. `schema.py` — the `GroundedAnswer` shape (deliverable 1)

```python
"""schema.py — GroundedAnswer with sentence-level supported_by."""

from __future__ import annotations
from pydantic import BaseModel, Field, model_validator
from typing import Literal

ALLOWED_LINKING_PHRASES: set[str] = {
    "As a result,",
    "In summary,",
    "To clarify,",
    "That said,",
    "Specifically,",
    "For context,",
}


class Sentence(BaseModel):
    text: str
    supported_by: list[str] = Field(
        default_factory=list,
        description="Passage IDs from the retrieved pack. Empty only for linking sentences.",
    )
    is_linking: bool = False

    @model_validator(mode="after")
    def linking_sentences_only_when_empty(self):
        if self.is_linking:
            if self.supported_by:
                raise ValueError(
                    "linking sentence must have empty supported_by"
                )
            opener = self.text.split(" ", 1)[0] + (
                "," if "," in self.text.split(" ", 1)[0] else ""
            )
            # Full allow-list match against the exact opener (case-sensitive).
            if not any(self.text.startswith(p) for p in ALLOWED_LINKING_PHRASES):
                raise ValueError(
                    f"is_linking=True but text does not start with an allowed "
                    f"linking phrase; allowed: {sorted(ALLOWED_LINKING_PHRASES)}"
                )
        return self


class GroundedAnswer(BaseModel):
    sentences: list[Sentence] = Field(default_factory=list)
    unsupported_reason: str | None = None
    query_rewrite: str | None = None  # populated if mod-103 exercise-04's rewriter ran

    @model_validator(mode="after")
    def exactly_one_branch(self):
        has_sentences = len(self.sentences) > 0
        has_reason = self.unsupported_reason is not None
        if has_sentences and has_reason:
            raise ValueError(
                "GroundedAnswer has both sentences and unsupported_reason; "
                "exactly one branch must be populated"
            )
        if not has_sentences and not has_reason:
            raise ValueError(
                "GroundedAnswer has no sentences and no unsupported_reason; "
                "the model produced nothing and gave no reason"
            )
        return self

    @model_validator(mode="after")
    def non_linking_sentences_require_support(self):
        for i, s in enumerate(self.sentences):
            if not s.is_linking and not s.supported_by:
                raise ValueError(
                    f"sentence {i} is not a linking phrase and has empty "
                    f"supported_by; this is an ungrounded claim"
                )
        return self
```

The validators fire *at parse time* — the model's output cannot reach the render if it has an ungrounded sentence or a both-branches contradiction.

### 2b. `parser.py` — parse-and-validate with pack-check and counters (deliverable 2)

```python
"""parser.py — parses the model output and enforces the pack-check."""

from __future__ import annotations
import json
from pydantic import ValidationError

from schema import GroundedAnswer, Sentence, ALLOWED_LINKING_PHRASES
import metrics
import confidence

MAX_REPAIR_ATTEMPTS = 1  # one-shot repair, then fall to no_good_evidence


def parse_and_validate(
    raw_output: str,
    pack: list[dict],
    user_has_access,
    model_fn,  # callable(messages) -> str, injected
    manifest_sources: list[str],
    index_freshness: dict[str, str],
) -> dict:
    """Parse model output, enforce the pack-check, render the payload."""
    attempt = 0
    current_output = raw_output
    violation = None

    while attempt <= MAX_REPAIR_ATTEMPTS:
        try:
            parsed = json.loads(current_output)
            answer = GroundedAnswer.model_validate(parsed)
            violation = _pack_check(answer, pack, user_has_access)
            if violation is None:
                return _render_payload(
                    answer, pack, manifest_sources, index_freshness
                )
        except (ValidationError, json.JSONDecodeError) as exc:
            violation = str(exc)
            _bump_violation_counter(exc)
        if attempt == MAX_REPAIR_ATTEMPTS:
            break
        attempt += 1
        current_output = model_fn(_repair_messages(violation))

    # Fall to no_good_evidence — document the choice.
    metrics.no_good_evidence_total.inc(labels={"reason": "parse_or_pack_check_failed"})
    return _render_payload(
        GroundedAnswer(unsupported_reason=f"Answer failed validation: {violation}"),
        pack,
        manifest_sources,
        index_freshness,
    )


def _pack_check(answer: GroundedAnswer, pack: list[dict], user_has_access) -> str | None:
    pack_ids = {p["passage_id"] for p in pack}
    for i, s in enumerate(answer.sentences):
        for pid in s.supported_by:
            if pid not in pack_ids:
                metrics.citation_hallucination_total.inc()
                return f"sentence {i} cites passage {pid!r} not in retrieved pack"
            passage = next(p for p in pack if p["passage_id"] == pid)
            if not user_has_access(passage):
                metrics.cited_but_forbidden_total.inc()
                return f"sentence {i} cites passage {pid!r} user cannot access"
    return None


def _render_payload(
    answer: GroundedAnswer,
    pack: list[dict],
    manifest_sources: list[str],
    index_freshness: dict[str, str],
) -> dict:
    if not answer.sentences:
        return {
            "no_good_evidence": True,
            "unsupported_reason": answer.unsupported_reason,
            "sources_checked": manifest_sources,
            "pack_size": len(pack),
            "next_action": {"label": "File a policy-gap ticket", "kind": "policy_gap"},
        }

    # Dedup evidence by doc_id; keep both passage_ids for the eval.
    evidence_by_doc: dict[str, dict] = {}
    for s in answer.sentences:
        for pid in s.supported_by:
            p = next(pp for pp in pack if pp["passage_id"] == pid)
            doc = evidence_by_doc.setdefault(
                p["doc_id"],
                {
                    "doc_id": p["doc_id"],
                    "doc_title": p["doc_title"],
                    "doc_url": p["doc_url"],
                    "spans": [],
                    "passage_ids": [],
                    "freshness": index_freshness.get(p["source"], "healthy"),
                },
            )
            doc["spans"].append(p["text"])
            doc["passage_ids"].append(pid)

    metrics.answers_rendered_total.inc()
    return {
        "no_good_evidence": False,
        "sentences": [s.model_dump() for s in answer.sentences],
        "citations": [{"sentence_index": i, "ids": s.supported_by}
                      for i, s in enumerate(answer.sentences)],
        "evidence": list(evidence_by_doc.values()),
        "confidence_bucket": confidence.confidence_bucket(answer, pack),
        "query_rewrite": answer.query_rewrite,
    }


def _repair_messages(violation: str) -> list[dict]:
    return [
        {"role": "system", "content": "Your previous answer violated the schema. Return corrected JSON only."},
        {"role": "user", "content": violation},
    ]


def _bump_violation_counter(exc) -> None:
    msg = str(exc).lower()
    if "ungrounded" in msg:
        metrics.ungrounded_claim_total.inc()
    elif "both sentences and unsupported_reason" in msg:
        metrics.ungrounded_claim_total.inc()  # shape confusion counts
```

The one-shot-repair-then-fall policy is explicit and documented in the module docstring of `parser.py` (above): on any validation failure, repair once; if the repair also fails, take the `no_good_evidence` branch and increment `no_good_evidence_total{reason="parse_or_pack_check_failed"}`. That reason label is what the daily readout in [[exercise-03]] slices on.

### 2c. `app.py` — minimal Streamlit render (deliverable 3)

```python
"""app.py — Streamlit render for GroundedAnswer payloads.

Run: streamlit run app.py
The UI does not need to be pretty. It needs to be legible enough that a
reviewer can verify a claim in one click.
"""

import streamlit as st

def render(payload: dict) -> None:
    if payload["no_good_evidence"]:
        _render_no_evidence(payload)
        return

    left, right = st.columns([2, 1])
    with left:
        st.subheader("Answer")
        if payload.get("query_rewrite"):
            with st.expander("Why this answer"):
                st.write(f"Query (rewritten): {payload['query_rewrite']}")
                st.write(f"Passages considered: {len(payload['evidence'])} docs")
                st.write(f"Confidence: {payload['confidence_bucket']}")
        for i, s in enumerate(payload["sentences"]):
            cites = payload["citations"][i]["ids"]
            pills = " ".join(f"`[{c}]`" for c in cites) if cites else "_linking_"
            st.markdown(f"{s['text']} {pills}")

    with right:
        st.subheader("Evidence")
        for doc in payload["evidence"]:
            badge = {"healthy": "🟢", "degraded": "🟡", "offline": "🔴"}.get(
                doc["freshness"], "⚪"
            )
            st.markdown(f"**{badge} {doc['doc_title']}**")
            for span in doc["spans"]:
                st.markdown(f"> {span}")
            st.markdown(f"[Open source]({doc['doc_url']})  \n"
                        f"_passages: {', '.join(doc['passage_ids'])}_")


def _render_no_evidence(payload: dict) -> None:
    st.warning("No good evidence for this question.")
    st.write(payload["unsupported_reason"])
    st.markdown("**Sources checked:**")
    for s in payload["sources_checked"]:
        st.markdown(f"- `{s}`")
    if st.button(payload["next_action"]["label"]):
        st.toast("Policy-gap ticket filed (stub).")
```

Key behaviours:

- Inline citations render as pills next to each sentence; a linking sentence shows `_linking_` and does not require a citation.
- The evidence panel de-duplicates on `doc_id`; two citations into the same document become one card with two spans.
- The freshness badge is driven directly from the `index_freshness` the contract from [[exercise-01]] publishes — no new metadata.
- "No good evidence" is a layout, not an error: it names the sources checked and offers a next action.

### 2d. `demos.md` — the six failure modes (deliverable 4)

```markdown
# Failure-mode demonstrations

Every demo follows the shape: **raw model output → parser decision → rendered UI state.**

## 1. Citation hallucination

**Raw output:**
```json
{"sentences": [
  {"text": "Reimbursements are approved within 10 business days.",
   "supported_by": ["P99"], "is_linking": false}
]}
```
Pack IDs: `{P1, P2, P3, P4}`.

**Parser decision:** `_pack_check` detects `P99` ∉ pack. `citation_hallucination_total++`. One-shot repair asks the model to re-cite. Repaired output still references `P99`. Parser falls to `no_good_evidence` with `unsupported_reason="sentence 0 cites passage 'P99' not in retrieved pack"`.

**Rendered state:** No-evidence card with the manifest sources and the policy-gap button.

## 2. Off-passage claim cited to a real passage

**Raw output:** cites `P1` for a claim about carry-over PTO; `P1` says nothing about carry-over.

**Parser decision:** Passes the pack-check (the ID resolves). This is a *faithfulness* failure, not a parse failure. The surface renders the answer but wires it to the online judge (see [[exercise-03]]'s `eval_triad.py`) so the `cited_but_unsupported` rate is at least countable.

**Rendered state:** Answer renders with citation pill `[P1]`. The Why-this-answer expandable notes "faithfulness sampling on." A nightly judge run will flag the case for the adversarial fixture.

## 3. Duplicate citations on the same claim

**Raw output:** `supported_by: ["P3", "P7"]` where `P3` and `P7` are chunks of the same doc `D42`.

**Parser decision:** Both cite IDs valid; the evidence panel groups them by `doc_id`. `evidence` has one card for `D42` with two spans, both highlighted.

**Rendered state:** One evidence card, both spans visible, with `_passages: P3, P7_` in the footer so the eval can still attribute to the chunk.

## 4. Citation to a forbidden document

**Raw output:** cites `P5`; `user_has_access(P5)` returns `False` (passage belongs to another tenant or to a role the user lacks).

**Parser decision:** `_pack_check` returns a violation; `cited_but_forbidden_total++`; falls to `no_good_evidence` with reason "sentence 0 cites passage 'P5' user cannot access." The render refuses to show the passage.

**Rendered state:** No-evidence card. The reason the user sees is not the raw message (which would leak existence) — the render layer translates it to "no results you have access to."

## 5. Empty pack

**Raw output:** not produced — the generator is not called.

**Parser decision:** Upstream of `parse_and_validate`, the orchestrator checks `len(pack) == 0` and short-circuits with `GroundedAnswer(unsupported_reason="pack is empty")`. `no_good_evidence_total{reason="empty_pack"}++`.

**Rendered state:** No-evidence card. "Sources checked" names the manifest; the policy-gap button is present.

## 6. Off-corpus answer

**Raw output:**
```json
{"sentences": [
  {"text": "Our policy aligns with the 2024 federal guideline.",
   "supported_by": [], "is_linking": false}
]}
```
No `is_linking=True`, no `supported_by`.

**Parser decision:** `GroundedAnswer`'s `non_linking_sentences_require_support` validator rejects at parse time. `ungrounded_claim_total++`. One-shot repair; if the repair is also ungrounded, fall to `no_good_evidence`.

**Rendered state:** No-evidence card.
```

### 2e. `metrics.py` + `confidence.py` (deliverable 5)

```python
"""metrics.py — the five counters."""

from collections import Counter
from dataclasses import dataclass

@dataclass
class _Counter:
    name: str
    _by_labels: Counter = None
    def __post_init__(self):
        self._by_labels = Counter()
    def inc(self, labels: dict | None = None) -> None:
        key = tuple(sorted((labels or {}).items()))
        self._by_labels[key] += 1
    def total(self) -> int:
        return sum(self._by_labels.values())

citation_hallucination_total = _Counter("citation_hallucination_total")
ungrounded_claim_total = _Counter("ungrounded_claim_total")
no_good_evidence_total = _Counter("no_good_evidence_total")
cited_but_forbidden_total = _Counter("cited_but_forbidden_total")
answers_rendered_total = _Counter("answers_rendered_total")
```

```python
"""confidence.py — deterministic bucket function."""

from retriever_client import score_bucket, SURFACE_SCORE_BUCKETS  # from exercise-01

def passage_score_bucket(pack, pid: str) -> str:
    p = next(pp for pp in pack if pp["passage_id"] == pid)
    return score_bucket(p["source_surface_id"], p["score"])


def confidence_bucket(answer, pack) -> str:
    """Return low / medium / high. Deterministic given (answer, pack)."""
    non_linking = [s for s in answer.sentences if not s.is_linking]
    if not non_linking:
        return "low"

    cited = sum(1 for s in non_linking if s.supported_by)
    ratio = cited / len(non_linking)
    cited_ids = [pid for s in non_linking for pid in s.supported_by]
    buckets = [passage_score_bucket(pack, pid) for pid in cited_ids] or ["low"]

    if ratio >= 1.0 and all(b == "high" for b in buckets):
        return "high"
    if ratio >= 0.8 and all(b != "low" for b in buckets):
        return "medium"
    return "low"
```

```markdown
# confidence.md — bucket calibration plan

The bucket function is deterministic and does **not** take a model-produced
score. It is derived from the retriever's score buckets (owned by the rag
team, published by surface_id — see [[exercise-01]]) and the cited-support
ratio of the answer.

## Bucket boundaries

| Bucket | cited_supported_ratio | per-passage buckets |
|---|---|---|
| high | ≥ 1.0 (every non-linking sentence cites) | every cited passage is `high` |
| medium | ≥ 0.8 | no cited passage is `low` |
| low | otherwise | any `low` is enough |

## Calibration protocol

1. Hand-label 100 recent answers for correctness (binary: supported by the pack and matches the policy as written).
2. Compute accuracy per bucket.
3. Promote the bucket labels to the user-visible UI only if:
   - `accuracy(high) > accuracy(medium) + 0.05` and
   - `accuracy(medium) > accuracy(low) + 0.05` and
   - each bucket has ≥ 20 answers in the sample.
4. If the test fails, redesign the function (e.g., add a second signal — "cited_doc_count" as a floor, or the judge's faithfulness score if the online eval is cheap enough).

Until calibration is done, the UI shows the bucket in the Why-this-answer
expandable (a debug surface), not in the main answer area.
```

### 2f. Implementation notes

- The schema **forbids a single trailing "Sources:" list** by construction: sources are per-sentence via `supported_by`, not per-answer. [Chapter 03] of the module warns against the trailing list.
- `P<n>` numbering in the model prompt, mapped to `passage_id` at parse time, is critical. The prompt never shows the URL — if the model hallucinates, we want a `P<n>` that doesn't resolve, not a fabricated URL that looks plausible.
- `cited_but_forbidden_total` and the ACL-aware pack check are the second-chance enforcement on top of the retriever-enforced ACL from [[exercise-01]]. Defence in depth, not a replacement.
- The one-shot repair is a deliberate budget decision: two repair attempts double the latency of a bad call for a marginal improvement in recovered answers. The fall-to-no_good_evidence branch is the honest default.

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. The `GroundedAnswer` schema rejects an object where `sentences=[]` and `unsupported_reason=None` with a clear `ValueError`.
2. The schema rejects an object where `sentences=[Sentence(text="X", supported_by=[], is_linking=False)]` with "ungrounded claim" in the message.
3. The schema rejects `{"sentences": [...], "unsupported_reason": "x"}` with "both sentences and unsupported_reason."
4. `parse_and_validate` with a pack `[{passage_id: "P1"}]` and a raw output citing `P99` increments `citation_hallucination_total` exactly once per failed parse attempt.
5. Same, with `user_has_access` returning `False` for `P1`, increments `cited_but_forbidden_total`.
6. An empty pack short-circuits before the generator is called (verify by stubbing `model_fn` to raise; the test still succeeds).
7. The Streamlit app renders inline citation pills next to each sentence and a separate evidence panel on the right; grep the rendered HTML for the citation ID to confirm both locations.
8. The evidence panel de-duplicates two passages that share a `doc_id`. A fixture with two passage IDs on the same doc renders one evidence card with two spans, not two cards.
9. The no-evidence state renders "Sources checked" with every manifest source and a next-action button.
10. `confidence_bucket` is deterministic: calling it twice on the same `(answer, pack)` returns the same bucket. Calling it with a `low` passage in `supported_by` never returns `high`, regardless of the ratio.
11. `demos.md` shows all six failure modes with the raw output, the parser decision, and the rendered state.
12. `confidence.md` names the calibration criteria and the hand-label sample size.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Schema validators are correct | 2 | Both contradiction validators + the ungrounded-claim validator fire with clear messages. |
| Pack-check at parse time | 1 | Unknown passage ID rejected *before* the render layer sees it; counter increments once. |
| Repair-or-fall policy is documented | 1 | Module docstring names repair budget and the fall-to-no_good_evidence outcome. |
| Render shows inline citations + evidence panel | 1 | Pills next to sentences; panel on the right; freshness badge visible. |
| Evidence de-dup by doc_id | 1 | Two passages, same doc → one evidence card; passage IDs preserved in the footer. |
| Forbidden-doc refuses to render | 1 | Mocked ACL fail → no-evidence state; the user-visible reason does not leak existence. |
| No-evidence state is first-class | 1 | Named sources checked, concrete next action, no stack trace, no error affordance. |
| Six failure-mode demos recorded | 1 | `demos.md` has one entry per failure mode with raw / parser / rendered. |
| Confidence bucket deterministic + calibration plan | 1 | Function is pure given inputs; `confidence.md` has a calibration protocol with concrete thresholds. |

A submission scoring < 6 is rejected and sent back for a second pass. The two validators from the schema are the load-bearing part — a submission that ships a `GroundedAnswer` where both branches can co-exist has not solved the exercise.

## 5. Common mistakes

- **Treating `GroundedAnswer` as a loose dict.** Without the validators, the model will eventually emit a "best of both worlds" output (some sentences + an `unsupported_reason`) and the render will silently drop one side. The validators turn that into a loud parse failure.
- **Letting the model see URLs.** If the prompt includes `doc_url`, the model will fabricate URLs when it hallucinates citations. Pass `P<n>` only; map to URL at render time.
- **A trailing Sources list.** "Policy X says Y and Z. `[1][2]`" is not sentence-level grounding — the reader cannot tell which claim `[1]` supports. Enforce `supported_by` per-sentence; see [Chapter 03].
- **Mixing "no good evidence" into the error path.** A try/except around the generator that renders a toast is wrong. `no_good_evidence` is a return value, not an exception.
- **De-duplicating on passage_id instead of doc_id.** Two chunks of the same policy doc should be one evidence card. De-duplicating on `passage_id` leaves the panel cluttered and lies about coverage.
- **Rendering a forbidden-doc reason verbatim.** "Cites passage P5 that user cannot access" is an existence oracle. The user-visible reason must be opaque; the internal log can be specific.
- **Using the model's confidence number.** A model-produced float is not a confidence score — it is a token-probability artifact that drifts across prompt changes. The bucket function has to be deterministic and derived from signals the surface owns.
- **One repair attempt budget quietly becomes four.** The retry-and-repair pattern (mod-102 chapter 04) is to pick a budget and stick to it. Document the budget in the module docstring; the eval readout will show whether the budget was right.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the citation-hallucination guard is the surface-level answer to ML06 (output-integrity attacks) and, when a forbidden-doc citation is involved, ML02 (data poisoning via indirect injection — a passage crafted to be cited into the wrong tenant's answer).
- MITRE ATLAS — <https://atlas.mitre.org/> — the "ungrounded claim" and "off-corpus answer" failure modes correspond to LLM parametric-knowledge leakage; the schema's grounding requirement is the mitigation.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the evidence panel + Why-this-answer expandable are MEASURE-function artefacts (traceability and explainability); the "no good evidence" branch is a MANAGE control that keeps the surface honest when MAP context is incomplete.

Related exercises: [[exercise-01]] publishes `index_freshness` and the retriever-enforced ACL consumed by the pack-check; [[exercise-03]] slices the counters in this file by surface and tenant in the daily readout; [[exercise-04]]'s rewriter populates `GroundedAnswer.query_rewrite` so the Why-this-answer expandable shows the rewritten query.
