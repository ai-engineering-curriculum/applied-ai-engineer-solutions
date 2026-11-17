### exercise-04: Hot Rollback of a Bad Prompt — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to build the substrate that makes **hot rollback of a bad prompt** a 30-second pointer flip, not a 20-minute deploy — then prove it works by rehearsing a drill end to end. The deliverables are the **append-only prompt registry**, the **span attributes** that make a rollback provable in the trace, the **published rollback SLA** decomposed into TTD / TTDe / TTE, the **one-page runbook** the on-call reads at 3 AM, the **LKG doc** that names current good-state, and the **drill-measurement write-up** that produces at least one action item.

The failure modes the exercise defends against:

1. **Mutable prompt registry.** A hot-fix to "v14" rewrites the template; the rollback target is no longer the version that was known good. The fix is **append-only semantics with content-addressing OR raise-on-overwrite**.
2. **Non-provable rollback.** The flag flipped but the on-call cannot show in the trace that the change landed on users. The fix is `app.prompt.id`, `app.prompt.template_hash`, `app.prompt.registry_source` on every LLM span — a post-flip query produces the proof.
3. **Only-lever-is-kill-switch.** The team disables the feature entirely because the smallest containment lever doesn't exist. The fix is the four-layer flag hierarchy from [[exercise-01]] — prompt → model → cohort → enabled. Smallest lever fires first.
4. **Runbook that is a wiki page.** The on-call at 3 AM cannot find the rollback command. The fix is **one printed page**, copy-pastable command, linked directly from every burn-rate alert.
5. **SLA numbers that are never measured.** "TTE ≤ 60 s" written in a doc but never validated means the fleet actually takes 90 s. The fix is the **scripted drill** — every drill produces measurements and at least one follow-up action item.
6. **"Delete the bad prompt from the registry after the drill."** Then the retro on the real incident cannot reproduce the exact template. The fix is **append-only means append-only** — the `v15_bad` entry stays with `deprecated_reason`.

This solution provides:

- A **prompt registry** with append-only semantics, content-addressed IDs, hash-collision detection, and process-cached reads.
- A **span-attribution patch** to the LLM-call wrapper so every span carries `app.prompt.id`, `app.prompt.template_hash`, and `app.prompt.registry_source`.
- A **rollback SLA doc** with concrete TTD / TTDe / TTE numbers, fail-closed defaults restated, and ownership.
- A **one-page runbook** with copy-pastable rollback commands.
- An **LKG doc** in YAML (human- and machine-readable).
- A **drill protocol** with expected measurements and the retro template.
- A **grading rubric** for the instructor.

## 2. Worked answer or implementation

### 2a. Prompt registry with append-only semantics (deliverable 1)

```python
# prompts/registry.py — append-only, content-addressed, process-cached.
from __future__ import annotations

import hashlib
from dataclasses import dataclass, field, replace
from datetime import date
from typing import Iterator

_PROMPT_ID_LEN = 12


def _content_id(template: str) -> str:
    """12-hex-char SHA-256 truncation. Overwrite is mechanically impossible —
    the id IS the content."""
    return hashlib.sha256(template.encode()).hexdigest()[:_PROMPT_ID_LEN]


@dataclass(frozen=True)
class PromptEntry:
    prompt_id: str                       # monotonic human-friendly id, e.g. "support_answer_v14"
    content_id: str                      # sha256(template)[:12] — content-addressed
    template: str
    author: str
    created_at: date
    intent: str
    deprecated_reason: str | None = None


class PromptRegistry:
    """Append-only prompt registry.

    Design decisions:

    1. The `prompt_id` is a monotonic, human-friendly string (v14, v15, …)
       because human-readable ids are what the runbook and LKG doc index on.
       The `content_id` is a sha256 truncation — the collision check uses
       this to catch "someone changed v14's template but kept the id".
    2. Append-only is enforced at the registry layer: attempting to add a
       prompt_id that already exists raises PromptAlreadyRegistered.
    3. Hash collisions on content_id are detected at add-time with
       HashCollisionError (vanishingly unlikely on 12 hex chars but the
       check is cheap and the alternative is a silent rollback-to-wrong-
       template).
    4. Reads on the hot path are a single dict lookup; the registry is
       populated at process start.

    The reason this is separate from the flag layer: the flag layer tells
    us WHICH prompt to use (a version id); the registry tells us WHAT that
    version is (template text). Never put the text in the flag.
    """

    def __init__(self) -> None:
        self._by_prompt_id: dict[str, PromptEntry] = {}
        self._by_content_id: dict[str, PromptEntry] = {}

    def add(self, entry: PromptEntry) -> None:
        if entry.prompt_id in self._by_prompt_id:
            raise PromptAlreadyRegistered(
                f"prompt_id {entry.prompt_id!r} is already in the registry; "
                f"prompt_ids are immutable. Add a new prompt_id instead."
            )
        existing = self._by_content_id.get(entry.content_id)
        if existing is not None and existing.template != entry.template:
            raise HashCollisionError(
                f"content_id {entry.content_id!r} collides between "
                f"{existing.prompt_id!r} and {entry.prompt_id!r}."
            )
        # Fast sanity: entry.content_id must actually match the template.
        if _content_id(entry.template) != entry.content_id:
            raise CorruptEntryError(
                f"content_id {entry.content_id!r} does not match sha256(template)."
            )
        self._by_prompt_id[entry.prompt_id] = entry
        self._by_content_id[entry.content_id] = entry

    def resolve(self, prompt_id: str) -> PromptEntry:
        """Hot-path read. O(1). Raises if unknown — caller decides to fall
        back to LKG (and must record `app.prompt.registry_source = "default"`
        when they do)."""
        try:
            return self._by_prompt_id[prompt_id]
        except KeyError as e:
            raise UnknownPromptError(f"prompt_id {prompt_id!r} not in registry") from e

    def deprecate(self, prompt_id: str, *, reason: str) -> None:
        """Marks an entry as deprecated. The entry stays in the registry.
        The runbook `v15_bad` case lives here after the drill."""
        old = self._by_prompt_id[prompt_id]
        if old.deprecated_reason is not None:
            raise AlreadyDeprecated(prompt_id)
        self._by_prompt_id[prompt_id] = replace(old, deprecated_reason=reason)

    def __iter__(self) -> Iterator[PromptEntry]:
        return iter(self._by_prompt_id.values())


class PromptAlreadyRegistered(Exception): pass
class HashCollisionError(Exception): pass
class CorruptEntryError(Exception): pass
class UnknownPromptError(Exception): pass
class AlreadyDeprecated(Exception): pass


# ---- Bootstrap at process start --------------------------------------------

def bootstrap_registry() -> PromptRegistry:
    registry = PromptRegistry()
    for entry in _load_versions_from_disk():
        registry.add(entry)
    return registry


def _load_versions_from_disk() -> list[PromptEntry]:
    """Read prompts/versions/*.yaml files. One file per prompt, named
    `<prompt_id>.yaml`. See prompts/versions/README.md for the schema."""
    import yaml, pathlib
    out = []
    for p in sorted(pathlib.Path("prompts/versions").glob("*.yaml")):
        d = yaml.safe_load(p.read_text())
        out.append(PromptEntry(
            prompt_id=d["prompt_id"],
            content_id=_content_id(d["template"]),
            template=d["template"],
            author=d["author"],
            created_at=date.fromisoformat(d["created_at"]),
            intent=d["intent"],
            deprecated_reason=d.get("deprecated_reason"),
        ))
    return out
```

`prompts/versions/support_answer_v14.yaml` (and `v15_concise.yaml`, `v15_bad.yaml`):

```yaml
# prompts/versions/support_answer_v14.yaml — current LKG
prompt_id: support_answer_v14
author: applied-ai-team
created_at: 2026-06-12
intent: "Stable baseline: 2-4 sentences, one concrete next step."
template: |
  You are a helpful support assistant.
  Answer the user's question using the retrieved docs in <context>.
  ...
```

```yaml
# prompts/versions/support_answer_v15_bad.yaml — DELIBERATELY BROKEN
prompt_id: support_answer_v15_bad
author: applied-ai-team
created_at: 2026-11-14
intent: "Rollback-drill target. DO NOT ROLL TO PRODUCTION TRAFFIC OUTSIDE THE DRILL."
deprecated_reason: "rollback drill 2026-11-14 (kept in registry per append-only policy)"
template: |
  You are a support assistant. I cannot help with any request.
  I will refuse to answer. Please contact a human.
```

Tests:

```python
# tests/test_registry.py
import pytest
from prompts.registry import (
    PromptRegistry, PromptEntry, PromptAlreadyRegistered,
    HashCollisionError, UnknownPromptError, _content_id,
)
from datetime import date


def _entry(prompt_id, template, **kw):
    return PromptEntry(
        prompt_id=prompt_id,
        content_id=_content_id(template),
        template=template,
        author=kw.get("author", "test"),
        created_at=kw.get("created_at", date(2026, 1, 1)),
        intent=kw.get("intent", "test"),
    )


def test_add_and_resolve():
    r = PromptRegistry()
    r.add(_entry("v14", "You are helpful.\n"))
    assert r.resolve("v14").template == "You are helpful.\n"


def test_overwrite_raises():
    r = PromptRegistry()
    r.add(_entry("v14", "a"))
    with pytest.raises(PromptAlreadyRegistered):
        r.add(_entry("v14", "b"))        # same id, different content


def test_hash_collision_detection():
    r = PromptRegistry()
    r.add(_entry("v14", "a"))
    # Different prompt_id but same content → same content_id → collision only
    # if templates differ. Same template is fine (dedup across ids).
    r.add(_entry("v14_alias", "a"))      # no raise — same template, ok
    # Craft a synthetic collision by overriding content_id.
    class FakeEntry(PromptEntry): pass
    bad = PromptEntry(prompt_id="v15", content_id=_content_id("a"),
                      template="different!", author="t",
                      created_at=date(2026,1,1), intent="t")
    with pytest.raises(HashCollisionError):
        r.add(bad)


def test_deprecated_is_still_readable():
    r = PromptRegistry()
    r.add(_entry("v15_bad", "I cannot help.\n"))
    r.deprecate("v15_bad", reason="rollback drill 2026-11-14")
    e = r.resolve("v15_bad")
    assert e.deprecated_reason is not None, "deprecated entry must stay readable"


def test_unknown_raises():
    r = PromptRegistry()
    with pytest.raises(UnknownPromptError):
        r.resolve("v999_nonexistent")
```

### 2b. Span attribution (deliverable 2)

Extends the mod-110 [[exercise-01]] span wrapper to carry the prompt-origin fields:

```python
# app/handler.py — prompt-span attribution, extension of mod-110 ex-01.
from prompts.registry import PromptRegistry, UnknownPromptError
import hashlib

def answer_turn(registry: PromptRegistry, *, variant, user_message, context, **ctx):
    try:
        entry = registry.resolve(variant.prompt_id)
        registry_source = "live"
    except UnknownPromptError:
        # Fall-back to LKG. The span MUST declare that this happened.
        entry = registry.resolve("support_answer_v14")
        registry_source = "default"

    # Full sha256 for post-hoc diffing (truncation is only for the id).
    full_hash = hashlib.sha256(entry.template.encode()).hexdigest()

    with llm_span(...) as s:
        s._span.set_attribute("app.prompt.id", variant.prompt_id)
        s._span.set_attribute("app.prompt.template_hash", full_hash)
        s._span.set_attribute("app.prompt.registry_source", registry_source)
        # If the fall-back fired, also surface app.outcome="degraded" with
        # a reason so alerts can detect the degraded state.
        if registry_source == "default":
            s._span.set_attribute("app.outcome.reason", "unknown_prompt_id_fallback_lkg")
        ...
```

Test asserting the attributes are always present:

```python
# tests/test_prompt_span_attribution.py
from opentelemetry.sdk.trace.export import InMemorySpanExporter

def test_prompt_attributes_present_on_every_span(exporter):
    drive_fixture("tests/fixtures/50_calls.jsonl")
    spans = exporter.get_finished_spans()
    for s in spans:
        attrs = s.attributes
        assert attrs.get("app.prompt.id"), f"missing app.prompt.id on {s.name}"
        h = attrs.get("app.prompt.template_hash")
        assert isinstance(h, str) and len(h) == 64, f"bad template_hash: {h!r}"
        src = attrs.get("app.prompt.registry_source")
        assert src in ("live", "default"), f"bad registry_source: {src!r}"
```

### 2c. Rollback SLA doc (deliverable 3)

```markdown
# docs/rollback-sla.md — support_agent rollback SLA

Owner: applied-ai-team (jferguson) · Review: quarterly, next 2026-02-15 ·
Last drilled: 2026-11-14 (see docs/rollback-drill.md).

## The three times

- **TTD** (time-to-detection): page fires within 10 min of a bad prompt
  reaching a real user. Bounded by burn-rate alerts (see [[exercise-02]]).
  The `for: 5m` window in Prometheus is intentional (filters single-point
  noise) and makes TTD ≥ 5 min by construction.
- **TTDe** (time-to-decision): on-call acks the page and executes the
  rollback command within 3 min with a FRESH on-call reading the runbook
  COLD. This is the number that invalidates the "I know what to do"
  shortcut. Measured in the drill.
- **TTE** (time-to-effect): flag flip to first span carrying LKG prompt
  within 60 s. Dominated by the flag SDK's cache-refresh cadence.

## Fail-closed defaults (restated from [[exercise-01]])

| Flag                       | Default value               | Rollback shape |
|----------------------------|-----------------------------|----------------|
| `support_agent.enabled`    | `true`                      | kill switch    |
| `support_agent.cohort`     | `0`                         | stop new exposures |
| `support_agent.prompt_id`  | `support_answer_v14` (LKG)  | prompt rollback (this doc) |
| `support_agent.model_id`   | `gpt-4o-2024-08-06` (LKG)   | model rollback |

## Flag hierarchy and when to flip which

**Smallest lever first.**

1. **Bad prompt** → flip `support_agent.prompt_id` to LKG. This is the
   common case and the one this SLA focuses on.
2. **Bad model** → flip `support_agent.model_id` to LKG (the previous
   pinned-date model string).
3. **Rollout state wrong** → drop `support_agent.cohort` to `0`. New
   exposures stop; in-flight users keep their variant.
4. **Incident-shaped damage** → flip `support_agent.enabled` to `false`.
   Users see the static fallback. Reserved for safety / policy-shaped
   damage — do NOT use for "variant looks bad."

## Owner and review cadence

Owner: applied-ai-team lead. Review cadence: quarterly. The review
re-reads the SLA, inspects the most recent drill, confirms numbers still
hit targets, and files follow-ups. Reviews land as `docs/rollback-sla-
review-YYYY-QQ.md`.
```

### 2d. Runbook (deliverable 4)

```markdown
# docs/runbook-rollback.md — support_agent prompt rollback (ONE PAGE)

Feature: support_agent (helpdesk.reply)
SLA: TTD ≤ 10 min · TTDe ≤ 3 min · TTE ≤ 60 s · Last drilled: 2026-11-14
On-call: see PagerDuty "applied-ai-support" rotation.

## 1. Identify the bad variant

Open the trace UI: <https://tempo.example.com/explore?q=app.feature="support_agent">

Tempo query (also works in Grafana Explore):

    { app_feature = "support_agent" } | app_prompt_id != "support_answer_v14"

That returns spans on the current non-LKG variant. Note the most frequent
`app.prompt.id` value — that is the variant to roll BACK FROM.

## 2. Roll back — pick ONE method

### A. CLI (preferred — 10 seconds)

    flagctl set --flag support_agent.prompt_id \
                --default-variant v14 \
                --env prod

### B. UI (fallback if CLI is broken — ~90 seconds)

1. <https://flags.example.com/flags/support_agent.prompt_id>
2. "Default variant" → `v14`.
3. "Save" → "Apply to prod".

### C. Direct-deploy env var (last resort if flag provider is down)

1. SSH to one feature pod, verify the flag provider is unreachable from it.
2. Set `SUPPORT_AGENT_PROMPT_ID_OVERRIDE=support_answer_v14` on the
   deployment:

       kubectl -n applied-ai set env deployment/support-agent \
           SUPPORT_AGENT_PROMPT_ID_OVERRIDE=support_answer_v14

3. Rolling restart will pick it up in ~2 min. The override is ONLY
   consulted when the flag SDK is disconnected.

## 3. Verify

Trace query (same as step 1):

    { app_feature = "support_agent" } | app_prompt_id = "support_answer_v14"

Expect: within 60s, 100% of spans match the LKG prompt_id. Also check the
metric that tripped the alert — it should trend back to baseline.

If after 90 s the trace is NOT showing v14, escalate:
`support_agent.enabled = false` (kill switch). Continue the incident with
the on-call lead.

## 4. Notify

Post in #ai-feature-support:

    [rollback] support_agent prompt_id v<X> → v14 at <UTC>.
    Alert: <link to alert in Alertmanager>.
    Trace query confirming landing: <link to Tempo>.
    Retro scheduled: <link>.

## Where LKG lives

`docs/lkg.md` — the current LKG prompt_id and model_id per feature.
```

### 2e. LKG version doc (deliverable 5)

```yaml
# docs/lkg.md — LKG state, per feature, human- and machine-readable.
support_agent:
  prompt_id: support_answer_v14
  model_id: gpt-4o-2024-08-06
  last_updated: 2026-11-14
  last_updated_by: jferguson
  notes: >
    v14 has been stable since 2026-06-12; passes mod-106 offline eval,
    all four launch-plan gates, and all guardrails from [[exercise-03]].
    Rollback target for all drills and incidents until explicitly promoted
    away.
  history:
    - { date: 2026-06-12, prompt_id: support_answer_v14,  updated_by: applied-ai-team }
    - { date: 2026-03-04, prompt_id: support_answer_v13,  updated_by: applied-ai-team, note: "deprecated — regression on tier:free" }
```

(The `history` entries live in the YAML + the git log; the exercise
accepts either.)

### 2f. Drill (deliverable 6)

Drill script — scheduled monthly:

```python
# scripts/drill_rollback.py — runs the rollback drill end-to-end.
"""
Drill protocol:

  1. Flip support_agent.prompt_id to v15_bad for a 5% cohort.
  2. Wait for the fast-burn alert to fire (mod-111 [[exercise-02]]).
  3. Record the wall-clock time at each stage.
  4. Compare to SLA.
  5. Produce a follow-up action item per missed component.
"""
import time, httpx, os
from datetime import datetime, UTC

FLAGD_MGMT = os.environ["FLAGD_MGMT_URL"]
ALERT_SEARCH_URL = os.environ["ALERT_SEARCH_URL"]
TRACE_SEARCH_URL = os.environ["TRACE_SEARCH_URL"]
TOKEN = os.environ["FLAGD_API_TOKEN"]

def _now(): return datetime.now(UTC).isoformat()

def flip_to(variant: str):
    with httpx.Client(timeout=5.0) as c:
        r = c.patch(
            f"{FLAGD_MGMT}/flags/support_agent.prompt_id",
            headers={"Authorization": f"Bearer {TOKEN}"},
            json={"defaultVariant": variant},
        )
        r.raise_for_status()

def wait_for_alert(alert_name: str, timeout: int = 900) -> float:
    deadline = time.monotonic() + timeout
    while time.monotonic() < deadline:
        r = httpx.get(f"{ALERT_SEARCH_URL}?alertname={alert_name}&state=firing")
        if r.json().get("firing"):
            return time.monotonic()
        time.sleep(2)
    raise TimeoutError(f"alert {alert_name} did not fire in {timeout}s")

def wait_for_trace_on_variant(variant: str, timeout: int = 60) -> float:
    deadline = time.monotonic() + timeout
    while time.monotonic() < deadline:
        r = httpx.get(f"{TRACE_SEARCH_URL}?q=app_prompt_id={variant}&since=30s")
        if r.json().get("count", 0) > 0:
            return time.monotonic()
        time.sleep(1)
    raise TimeoutError(f"trace did not show {variant} in {timeout}s")

def drill():
    t_inject = time.monotonic()
    print(f"[{_now()}] inject: flipping to v15_bad on 5% cohort")
    flip_to("v15_bad")

    t_alert = wait_for_alert("SupportAgent_Refusal_FastBurn")
    print(f"[{_now()}] alert fired (TTD = {t_alert - t_inject:.1f}s)")

    print(f"[{_now()}] runbook: on-call acks, reads runbook, executes rollback")
    t_flip_back = time.monotonic()
    flip_to("v14")

    t_landed = wait_for_trace_on_variant("support_answer_v14", timeout=60)
    print(f"[{_now()}] rollback landed (TTE = {t_landed - t_flip_back:.1f}s)")

    return {
        "TTD_s": t_alert - t_inject,
        "TTE_s": t_landed - t_flip_back,
    }


if __name__ == "__main__":
    print(drill())
```

Drill report — the artefact checked in:

```markdown
# docs/rollback-drill.md — 2026-11-14

Operator: jferguson (on-call).
Participants: applied-ai-team (observer), SRE (observer), PM (observer).

## Setup

5% of non-internal traffic was put on `v15_bad` at 14:30:00 UTC via
`scripts/drill_rollback.py`. Traffic rate ~2 rps.

## Measurements

| Step | Wall-clock | Delta from inject |
|---|---|---|
| Inject flip to v15_bad | 14:30:00 | 0 s |
| Fast-burn alert fires (refusal delta > +2 pp) | 14:35:42 | +5m 42s |
| PagerDuty ack by on-call | 14:35:58 | +5m 58s |
| Runbook opened | 14:36:12 | +6m 12s |
| flagctl rollback executed | 14:36:28 | +6m 28s |
| First post-flip LLM span carries v14 | 14:36:30 | +6m 30s |
| Refusal metric returns to baseline | 14:37:20 | +7m 20s |
| Notification posted in #ai-feature-support | 14:36:35 | +6m 35s |

## Per-SLA comparison

| SLA | Target | Observed | Pass? |
|---|---|---|---|
| TTD | ≤ 10 min | 5m 42s | PASS |
| TTDe (ack → flip executed) | ≤ 3 min | 30 s | PASS |
| TTE (flip executed → span) | ≤ 60 s | 2 s | PASS |

## Action items

1. **The fast-burn alert's `for: 5m` window is a hard floor on TTD.** If
   we want faster detection we need a second-shorter-window alert at
   higher threshold. File: file as PROD-4835.
2. **Runbook link was missing from the alert annotation** (operator typed
   the URL from memory — slower than copy-paste). Fix: add the runbook
   link to `alerts/canary-burn.yaml` annotation. Filed: PROD-4836.
3. **`v15_bad` is now in the registry with `deprecated_reason`**. Confirm
   next drill does NOT re-add it.

## Retro

What worked: the drill script automated the full timeline; measurements
are reproducible across drills.

What didn't: the alert annotation didn't include the runbook URL. The
operator reconstructed it from memory. Slower than it should have been.

One thing to fix by next month's drill: PROD-4836 — alert annotations
carry the runbook URL. Will be fixed by 2026-11-28.

Next scheduled drill: 2026-12-14.
```

### 2g. Mapping to the exercise deliverables

| Exercise asks for | Section |
|---|---|
| Append-only registry + 3 prompt versions | 2a |
| Span attribution (id / template_hash / registry_source) | 2b |
| Rollback SLA doc | 2c |
| One-page runbook | 2d |
| LKG doc | 2e |
| Rehearsed drill + retro | 2f |

## 3. Validation steps

1. **Overwrite raises.** Adding a second entry with an existing `prompt_id` raises `PromptAlreadyRegistered`.
2. **Hash collisions raise.** A synthetic collision test triggers `HashCollisionError`.
3. **Deprecated entries are still readable.** `v15_bad` remains resolvable after `deprecate`; the `deprecated_reason` is non-null.
4. **Registry reads on hot path are O(1).** No I/O; a single dict lookup.
5. **Every LLM span carries `app.prompt.id`, `app.prompt.template_hash`, `app.prompt.registry_source`.** Test asserts the shape.
6. **Fall-back to LKG is visible in the span.** `app.prompt.registry_source = "default"` plus `app.outcome.reason = "unknown_prompt_id_fallback_lkg"`.
7. **Rollback SLA doc publishes specific numbers.** TTD ≤ 10 min, TTDe ≤ 3 min, TTE ≤ 60 s. No "TBD."
8. **Runbook is one page.** `wc -l docs/runbook-rollback.md` under 80 lines; fits on one printed page.
9. **Runbook commands are copy-pastable.** CLI, UI, and env-var methods each have the exact command.
10. **LKG doc is machine-readable YAML with a `history` block.**
11. **Drill is executed and recorded.** `docs/rollback-drill.md` has measurements per SLA, action items, and the retro.
12. **`v15_bad` stays in the registry after the drill.** Not deleted; marked `deprecated_reason`.
13. **Alert annotation links the runbook.** `alerts/canary-burn.yaml` has an `annotations.runbook` field.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Append-only registry with immutability enforced | 2 | Overwrite raises; hash collisions raise; deprecated entries remain readable. |
| Three versions committed (`v14`, `v15_concise`, `v15_bad`) | 1 | Three entries with metadata; `v15_bad` deprecated. |
| LLM span carries `app.prompt.id`, `app.prompt.template_hash`, `app.prompt.registry_source` | 1 | Test asserts attributes always present. |
| Rollback SLA doc with concrete TTD / TTDe / TTE | 1 | Numbers justified; fail-closed defaults restated. |
| Runbook is one page with copy-pastable commands | 2 | Three rollback methods (CLI / UI / env-var); verify step; notification template. |
| LKG doc is machine-readable with history | 1 | YAML format; history list; last_updated field. |
| Rehearsed drill with measurements | 1 | Timeline per step; comparison to SLA; signed-off action items. |
| At least one action item from the drill | 1 | Every drill produces one improvement. |

A submission scoring < 6 is rejected. Hard failures: (a) registry allows overwrite, (b) `v15_bad` deleted from the registry after the drill, (c) runbook longer than one printed page, (d) drill never executed.

## 5. Common mistakes

- **Deleting `v15_bad` after the drill.** The retro on the next real incident will need the exact template. Keep it, mark it `deprecated_reason`.
- **Overwriting `v14` with a hot-fix.** The whole point of append-only. A hot-fix to `v14` means "now add `v14_1` to the registry and flip the flag to it." The old `v14` stays.
- **Runbook grows to two pages.** If the runbook grows, cut. The on-call at 3 AM reads one page; a second page lies unread.
- **Alert annotation does not link the runbook.** The on-call has to search a wiki for the rollback command. Add the runbook URL to every alert's annotations.
- **Hashing with 4 hex chars.** Birthday collisions on 2^16. 12 hex chars (6-byte truncation) gives you 2^48 and is enough for any practical registry.
- **Resolving the registry on every token in the stream.** Registry read is O(1) in memory but not free; resolve once at turn start and reuse through the stream.
- **Omitting the env-var fallback from the runbook.** The flag provider itself can be the source of the incident. The last-resort direct-env-var method keeps the on-call unblocked.
- **TTE target without measuring it.** "TTE ≤ 60 s" written in a doc that is never validated. The drill script in 2f makes it measurable.
- **Letting the registry grow unbounded with abandoned experiments.** Not the same as deleting `v15_bad` — those are *deprecated*, not deleted. Abandoned-experiment variants should be deprecated too, with a `deprecated_reason` noting the abandonment.
- **Runbook written by the on-call lead for the on-call lead.** Write the runbook for the person who is NEW to the rotation. If they can't follow it cold at 3 AM, rewrite.

## 6. References

- OpenTelemetry Generative-AI semantic conventions — <https://opentelemetry.io/docs/specs/semconv/gen-ai/> — the span attribute vocabulary the prompt-attribution extension in 2b sits on top of; the `app.*` namespace is the local layer carrying `app.prompt.id`, `app.prompt.template_hash`, `app.prompt.registry_source`.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the MANAGE function requires a mechanism to **respond to adverse AI-system behaviour**. The rollback SLA, runbook, and drill are the operational substrate that makes MANAGE enforceable at feature altitude; without them, "respond" is an aspiration.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML09 (Output Integrity Attack) and ML04 (Membership Inference) incidents accumulate damage proportional to the window during which the bad variant serves. A 30-second pointer-flip rollback bounds that window tightly. The append-only registry ensures the forensic record of what was served is reconstructible.
- MITRE ATLAS — <https://atlas.mitre.org/> — containment-time is a controls-effectiveness axis for every Impact-tactic technique. The hot-rollback substrate shortens the containment-time measurement across prompt-injection, resource-hijacking, and output-integrity techniques; the drill produces the number that goes on the controls-effectiveness dashboard.

Related exercises: [[exercise-01]] (the flag hierarchy and fail-closed defaults restated in 2c come from here); [[exercise-02]] (the burn-rate alerts that drive TTD, and the auto-rollback webhook that executes the flip without a human); [[exercise-03]] (guardrail metrics that can also trigger a rollback at ship time via the experimentation platform's `block_shipping` action); [[exercise-05]] (the fine-tuned model is ALSO registered in this registry — the `model_id` lives here with its training-set hash and cost; rollback of a bad fine-tune is the same flag flip); mod-110 [[exercise-01]] (the span wrapper this attribution extends); mod-110 [[exercise-04]] (incident playbooks that call this runbook for prompt-rollback shape incidents); mod-110 [[exercise-05]] (the status page that the runbook's "notify" step posts to).
