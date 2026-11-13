### exercise-03: PII and Secret Scrubbing on Traces — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to design the **PII and secret scrubber** that sits between the running agent and every trace sink (observability backend, eval data lake, replay harness, incident archive) on the helpdesk-reply-copilot surface. "Traces" here is everything with wall-clock fan-out: spans, tool-call payloads, LLM prompt/completion pairs, moderation decisions, memory reads, HITL confirms, and the incident bundle captured by [[mod-110]].

The failure mode the exercise is written to catch is treating the scrubber as a vendor feature. "The observability provider redacts credit cards" is not a PII policy; it is a vendor claim. A scrubber the team does not own cannot be tuned per tenant, cannot be audited when an incident needs it, and cannot be measured for false-negative leaks. The scrubber is a first-party component with its own policy, its own tests, and its own report.

This solution provides:

- A **data-class inventory** — the concrete fields that may appear in traces on this surface, each labelled with a class (identifier, credential, content, metadata).
- A **scrub policy** mapping each class to one of {drop, hash, mask, allow} with the review cadence and the tenant override rules.
- A **reference scrubber implementation** (Python) with a declarative rule set, deterministic token-level masking, and a trace-time enforcement point.
- A **test suite** proving (a) known leaks are blocked, (b) tenants cannot opt out of mandatory classes, (c) redaction is deterministic across replays, (d) throughput meets the trace sink's budget.
- An **FN leak-hunt protocol** for the periodic adversarial check of the scrubber.
- A **grading rubric** for the instructor.

The lesson the exercise defends is: *the scrubber runs on everything or it runs on nothing.* A scrubber that is applied only to the "sensitive" trace path is a configuration bug waiting on wall-clock; the policy is a trace-time invariant.

## 2. Worked answer or implementation

### 2a. Data-class inventory (deliverable 1)

```markdown
# helpdesk-reply-copilot — trace data-class inventory (v1)

| Class | Example fields | Where it appears in traces |
|---|---|---|
| `identifier.direct` | email, phone, full name, gov-id | user_text; KB passages; CRM tool output; HITL confirm payload |
| `identifier.quasi` | tenant_id, user_id, session_id | span attributes; memory keys; log context |
| `identifier.device` | IP, device_id, user agent | request envelope; audit log |
| `credential.live` | API keys, bearer tokens, OAuth tokens | tool-call inputs; retry logs; error strings; memory writes |
| `credential.secret_hint` | CSRF tokens, nonces | HTTP spans; auth middleware spans |
| `credential.password` | plaintext passwords | should never be in traces; checked anyway |
| `content.ticket` | the customer-authored ticket body | input to the agent; attached to spans |
| `content.reply` | the drafted reply | output span; moderator span |
| `content.kb` | retrieved knowledge-base passages | retrieval span; model prompt |
| `metadata.timing` | latencies, token counts | always allowed |
| `metadata.verdict` | moderator verdict, HITL outcome | always allowed |
| `metadata.model` | model name, version, provider | always allowed |
```

Any field in a trace must be assignable to one class. An unclassifiable field cannot be emitted until the inventory is updated.

### 2b. Scrub policy (deliverable 2)

```yaml
# scrub_policy.yaml (excerpt — committed; a change is a diff review)
default_policy:
  identifier.direct:   {action: mask,  reversible: tenant_kms}
  identifier.quasi:    {action: hash,  salt: per_tenant_rotating}
  identifier.device:   {action: hash,  salt: global_rotating}
  credential.live:     {action: drop}          # never stored, not even masked
  credential.secret_hint: {action: drop}
  credential.password: {action: drop, alarm: pager}
  content.ticket:      {action: mask_content, policy: pii_within}
  content.reply:       {action: mask_content, policy: pii_within}
  content.kb:          {action: allow}         # tenant-authored, non-user PII expected
  metadata.timing:     {action: allow}
  metadata.verdict:    {action: allow}
  metadata.model:      {action: allow}

tenant_overrides:
  allowed_tightening:
    - content.reply.action: drop         # tenant may drop content entirely
    - content.ticket.action: drop
  forbidden_looseness:
    - credential.live                    # cannot be relaxed by tenant
    - credential.password
    - identifier.direct                  # must remain at least masked
```

Three rules this policy encodes:

1. Credentials are **dropped**, never masked. A mask of `sk-live-****` tells an attacker what shape the key has and preserves enough to narrow a search; drop is the only correct operation.
2. Identifiers of the direct kind are **masked, not hashed**, when the trace needs to preserve the anchor for debugging. Quasi-identifiers are **hashed with a per-tenant rotating salt** so that joining across tenants is not possible even with the hashes.
3. Tenants may *tighten* but never *loosen* the policy. A tenant who wants "no content in traces at all" can choose it; a tenant who wants "PII passed through for easier debugging" cannot.

### 2c. Reference scrubber implementation (deliverable 3)

```python
"""scrubber.py — trace-time scrubber for the reply-copilot.

Ordering: span-level rules run first (keys like "user.email" are masked
by key); then content rules run on free-text fields; then an opt-in LLM
tagger handles stubborn free-text classes. The LLM tagger is never the
first line — it is the backup for known-hard content.

Enforcement point: this scrubber is called inside the trace exporter.
No span leaves the process without passing through it.
"""
from __future__ import annotations

import hashlib
import hmac
import re
from dataclasses import dataclass, field
from enum import Enum
from typing import Callable

class Action(str, Enum):
    DROP = "drop"
    MASK = "mask"          # preserve shape, remove content
    HASH = "hash"          # deterministic one-way with rotating salt
    MASK_CONTENT = "mask_content"  # mask free-text per pii policy
    ALLOW = "allow"

@dataclass(frozen=True)
class FieldRule:
    key_matcher: Callable[[str], bool]
    data_class: str
    action: Action
    reversible_kms: str | None = None

# Example: by-key rules compiled from the policy
FIELD_RULES: list[FieldRule] = [
    FieldRule(lambda k: k.endswith(".email"), "identifier.direct", Action.MASK),
    FieldRule(lambda k: k in {"user_id", "tenant_id", "session_id"},
              "identifier.quasi", Action.HASH),
    FieldRule(lambda k: "authorization" in k.lower() or "api_key" in k.lower(),
              "credential.live", Action.DROP),
    FieldRule(lambda k: k.endswith(".password"), "credential.password",
              Action.DROP),
]

# Content-regex rules for free-text fields (ordered: credentials first)
CONTENT_REGEXES: list[tuple[str, str, Action]] = [
    # live credentials: drop whole field if any match (fail closed)
    (r"\b(sk-[A-Za-z0-9]{20,}|xoxb-[A-Za-z0-9-]{20,})\b",
     "credential.live", Action.DROP),
    (r"\b(AKIA[0-9A-Z]{16}|ghp_[A-Za-z0-9]{36})\b",
     "credential.live", Action.DROP),
    # emails / phones in content: mask in place
    (r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b",
     "identifier.direct", Action.MASK),
    (r"\b(\+?\d[\d\s().-]{7,}\d)\b",
     "identifier.direct", Action.MASK),
]

@dataclass
class ScrubReport:
    actions: dict[str, int] = field(default_factory=dict)
    dropped_fields: int = 0
    masked_tokens: int = 0

def _hash(value: str, salt: bytes) -> str:
    return hmac.new(salt, value.encode(), hashlib.sha256).hexdigest()[:16]

def _mask(value: str) -> str:
    return re.sub(r"[A-Za-z0-9]", "x", value)

def scrub_span(span: dict, salt: bytes, report: ScrubReport) -> dict:
    out = {}
    for key, value in span.items():
        if isinstance(value, dict):
            out[key] = scrub_span(value, salt, report)
            continue
        if not isinstance(value, str):
            out[key] = value
            continue
        rule = next((r for r in FIELD_RULES if r.key_matcher(key)), None)
        if rule and rule.action == Action.DROP:
            report.dropped_fields += 1
            continue
        if rule and rule.action == Action.HASH:
            out[key] = _hash(value, salt)
            continue
        if rule and rule.action == Action.MASK:
            out[key] = _mask(value)
            continue
        # content path: regex rules
        scrubbed, drop_field = _apply_content_rules(value, report)
        if drop_field:
            report.dropped_fields += 1
            continue
        out[key] = scrubbed
    return out

def _apply_content_rules(value: str, report: ScrubReport) -> tuple[str, bool]:
    scrubbed = value
    for pattern, cls, action in CONTENT_REGEXES:
        hits = re.findall(pattern, scrubbed)
        if not hits:
            continue
        report.actions[cls] = report.actions.get(cls, 0) + len(hits)
        if action == Action.DROP:
            return "", True          # fail closed: drop the whole field
        if action == Action.MASK:
            scrubbed = re.sub(pattern, lambda m: _mask(m.group(0)), scrubbed)
            report.masked_tokens += len(hits)
    return scrubbed, False
```

Enforcement rule: **the trace exporter does not know how to emit a span without calling `scrub_span`.** The scrubber is a hard dependency of the exporter; disabling it is a build-time change.

### 2d. Test suite (deliverable 4)

Minimum tests, each is a committed pytest case:

```python
# tests/test_scrubber.py
def test_live_key_dropped_not_masked():
    inp = {"message": "please use sk-live-abcdef0123456789abcdef0123"}
    out = scrub_span(inp, salt=b"s", report=ScrubReport())
    assert "sk-live" not in out["message"]
    assert "sk-" not in out["message"]       # masked would preserve "sk-"

def test_email_masked_not_hashed_in_free_text():
    inp = {"reply": "contact alice@acme.com for details"}
    out = scrub_span(inp, salt=b"s", report=ScrubReport())
    assert "alice@acme.com" not in out["reply"]
    assert "@" in out["reply"]               # mask preserves shape

def test_tenant_id_hashes_deterministically_within_salt():
    out1 = scrub_span({"tenant_id": "acme"}, salt=b"s1", report=ScrubReport())
    out2 = scrub_span({"tenant_id": "acme"}, salt=b"s1", report=ScrubReport())
    assert out1["tenant_id"] == out2["tenant_id"]

def test_salt_rotation_changes_hash():
    out1 = scrub_span({"tenant_id": "acme"}, salt=b"s1", report=ScrubReport())
    out2 = scrub_span({"tenant_id": "acme"}, salt=b"s2", report=ScrubReport())
    assert out1["tenant_id"] != out2["tenant_id"]

def test_tenant_cannot_loosen_credential_policy():
    # overlaid policy that attempts to allow credential.live
    with pytest.raises(PolicyError):
        load_policy({"credential.live": {"action": "allow"}}, tenant=True)

def test_nested_spans_are_scrubbed_recursively():
    inp = {"root": {"inner": {"authorization": "Bearer sk-live-xxx"}}}
    out = scrub_span(inp, salt=b"s", report=ScrubReport())
    assert "authorization" not in out["root"]["inner"]

def test_throughput_budget(benchmark):
    # scrub must handle 10k spans/sec on the exporter thread
    span = make_rich_span()
    benchmark(scrub_span, span, salt=b"s", report=ScrubReport())
    assert benchmark.stats["mean"] < 0.0001  # <0.1 ms / span
```

### 2e. FN leak-hunt protocol (deliverable 5)

A periodic adversarial sweep against the scrubber's traces. Monthly minimum, immediately after any policy diff or scrubber change. The sweep is a reusable notebook; its findings are tracked as tickets.

```markdown
# Scrubber leak-hunt — protocol (v1)

## Input
- A random 24h slice of production traces (post-scrub), sampled to 50k spans.
- The current scrub policy and any pending diffs.
- The known-leak regex library (in-repo).

## Sweeps
1. **Regex library**: run the entire known-leak regex library against the slice.
   Any hit is an incident-grade bug.
2. **Shape heuristics**: hunt for high-entropy strings that look like tokens
   (length ≥ 32, alphabet ≥ 48). Any that survive the scrubber get a human review.
3. **Semantic probe**: run a classifier on a 1 % sub-sample looking for
   tokens-in-free-text (names, phone numbers) that were not masked. Review FPs
   with the scrubber owner.
4. **Replay round-trip**: pick 100 traces and attempt to use them to
   reconstruct the user who authored the ticket. If any identifier joins
   back to a user, the scrub is insufficient.

## Output
- A short report with each hit, the regex / sweep that caught it, the policy
  gap, and a PR link that closes it.
- Ticket owners default to the trace surface that emitted the leak (CRM span,
  retry logger, etc.), not the scrubber itself — the scrubber is a filter;
  the source should not have been emitted at all.
```

### 2f. Implementation notes

- **Why drop, not mask, for credentials.** A masked credential tells an attacker (a) it was a credential and (b) roughly its shape; and masks occasionally preserve enough high-information characters to be searchable. Credentials are the one class where losing debugging is worth it.
- **Why hash for quasi-identifiers.** A tenant_id or user_id needs to join across spans within a session for a developer to debug anything. The hash preserves that join locally and breaks it cross-tenant. The salt rotates so an attacker who gains a day of traces cannot correlate across months.
- **Why by-key rules run before content regex.** By-key rules are faster, make the common path cheap, and cover the structured cases without false positives. Content regex is the fallback for free-text fields.
- **Why the exporter owns the enforcement point.** Every trace sink (OTel, vendor APM, eval lake, incident bundle) terminates at the exporter; placing enforcement there is the only way to guarantee "every trace is scrubbed." A scrubber in the agent is bypassable by any code path that logs.
- **Why the policy is versioned and diff-reviewed.** Policy changes are legal/compliance events, not operational toggles.
- **Why the scrubber is cheap first, LLM second.** A per-span LLM scrubber is either too expensive or too slow. The LLM tagger is reserved for the hard known-residual categories where a regex cannot classify.
- **Why we test fail-closed on credentials.** The test asserts that a matched credential drops the *whole field*, not the matched substring. A credential surrounded by free text is almost certainly leaking adjacent context that is also sensitive.

## 3. Validation steps

1. Every field class in the inventory maps to exactly one policy action.
2. The policy forbids tenants from loosening credential and direct-identifier classes; an attempt to load a loosening overlay raises.
3. `scrub_span` is called from the exporter unconditionally; a unit test asserts that the exporter without the scrubber refuses to start.
4. Tests pass for: credential drop (no shape preserved), email mask (shape preserved), hash determinism within salt, hash change across salt rotation, nested-span recursion, throughput budget.
5. The leak-hunt protocol is committed and has at least one historical run attached (run on the reference data provided).
6. The scrubber is in the exporter path for every sink listed in the inventory (observability backend, eval lake, replay, incident bundle).
7. A policy change requires a reviewed PR; runtime edits are not possible (configuration is a build artefact).
8. The report format from the leak-hunt routes tickets to the *source span* owner, not to "the scrubber team."

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Data-class inventory covers the surface | 1 | ≥ 8 field classes mapped from real trace fields. |
| Policy covers every class with an action | 1 | No `TODO`, no `unknown`. |
| Credentials drop, not mask | 1 | Shape is not preserved. |
| Quasi-IDs hashed with rotating per-tenant salt | 1 | Rotation named and measurable. |
| Scrubber enforced at the exporter boundary | 1 | Exporter refuses to start without scrubber. |
| Test suite covers drop/mask/hash/nested/throughput | 1 | All five tests present and passing. |
| Tenant override forbidden-looseness enforced | 1 | Attempt to loosen raises. |
| Leak-hunt protocol with ≥ 3 sweep methods | 1 | Includes a replay round-trip. |
| Report routes tickets to source owner | 1 | Ownership is traceable, not generic. |
| Policy is version-controlled; no runtime edits | 1 | Build-time artefact; diff review required. |

A submission scoring < 6 is rejected. Hard failures: (a) credentials masked instead of dropped, (b) scrubber not in the exporter path, (c) policy editable at runtime, (d) no leak-hunt protocol.

## 5. Common mistakes

- **Trusting the observability vendor's redaction.** Vendor redaction is best-effort, not a policy you own. The scrubber lives in your process so you can test it, audit it, and change it without a vendor ticket.
- **Masking credentials instead of dropping them.** "`sk-****`" is still a credential-shape leak. If a credential appears in a trace at all, that is the real bug; mask is a worse version of drop.
- **Hashing without salt rotation.** A static salt is a long-lived identifier; the hash becomes the user_id by another name. Rotate at a defined cadence.
- **Making the scrubber optional via a feature flag.** The one trace that matters for an incident is the one that was emitted when the flag was off.
- **Scrubbing only the "prompt" and "completion" fields.** The retry logs, the tool-call error payloads, the HITL confirm envelope, and the memory read audit log all carry PII. Enforcement is at the exporter, not at known field names.
- **Dropping free-text content too aggressively.** A scrubber that drops every reply makes incident forensics impossible. Mask within the content; drop only the whole field for credentials.
- **No leak-hunt.** The scrubber cannot be assumed to be complete without an adversarial check. The leak-hunt is the only process that catches the classes you forgot to classify.
- **Giving tenants a "disable scrub for debug" toggle.** Debug toggles become production toggles. Tenants may tighten; never loosen.
- **Assuming structured fields are safe.** `message` is free text even when named like a column. Every string field is a candidate for the content path.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the scrub policy's privacy and sensitive-information controls map to NIST AI RMF's MANAGE function requirements for data handling across the AI system lifecycle, including trace and telemetry data.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the exercise is the operational counterpart to the OWASP ML items that cover sensitive data exposure and unintended data leakage across logs and intermediate artefacts.
- MITRE ATLAS — <https://atlas.mitre.org/> — ATLAS enumerates exfiltration techniques through logs and telemetry side channels; the scrubber plus leak-hunt protocol is the defence class ATLAS expects to see in place when those techniques are in the threat model.

Related exercises: [[exercise-01]] (moderator sees scrubbed inputs on its LLM-based layers); [[exercise-02]] (prompt-injection defence uses the scrubber on T3 inputs before they reach the model); [[exercise-04]] (tool gates share the fail-closed discipline this exercise applies to credentials); [[exercise-05]] (this scrubber is one of the controls catalogued in the OWASP/ATLAS mapping runbook).
