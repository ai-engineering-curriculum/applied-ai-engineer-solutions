### exercise-02: Memory System with Retention and PII Policy — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to design the **memory system** for the helpdesk-reply-copilot: what the agent remembers across turns in a session, what it remembers across sessions for a user, what gets written to a tenant-scoped shared store, and what is forbidden to be stored at all. "Memory" on a product agent is a database — with keys, indexes, retention, eviction, auth, and a PII policy — not a vector-store add-on. The shape decision from [[exercise-01]] determined this surface uses a bounded tool-loop; the memory system attaches at three well-defined points.

The failure mode the exercise is written to catch is the invisible kind: an agent that quietly remembers a customer's email across tenants, an "episodic" store that grows unbounded and becomes the next week's postmortem, or a "user preference" field that gets written on every turn and silently becomes PII. A memory subsystem without retention and PII rules is a data-loss incident waiting on wall-clock.

This solution provides:

- A **memory-scope taxonomy** (scratchpad / session / user-pinned / tenant-shared) with explicit keys, writers, readers, and expiry.
- A reference **`MemoryStore` implementation** (Python, SQLite-backed) with scope-typed APIs, TTL-driven eviction, and a write-time PII classifier.
- A **retention schedule** that names TTLs per scope, the review cadence, and the right-to-erasure path.
- A **PII policy** aligned to a widely-used taxonomy (OWASP ML-Sec and NIST AI RMF measurement guidance), with the allow-/deny-list of fields and the redaction rules.
- A **unit test suite** proving (a) each scope obeys its TTL, (b) PII writes are refused or redacted, (c) tenant isolation holds under forged principals, and (d) right-to-erasure wipes a user's data across scopes within one call.
- A **grading rubric** for the instructor.

The lesson the exercise defends is: *memory is a privileged write path*. The retrieval side is a search problem; the write side is a data-governance problem. The system is designed from the write path and the retention policy outward.

## 2. Worked answer or implementation

### 2a. Memory-scope taxonomy (deliverable 1)

The surface has exactly four memory scopes. A fifth would need a documented justification and a retention review.

| Scope | Key shape | Writer | Reader | Lifetime | Example |
|---|---|---|---|---|---|
| `scratchpad` | `(session_id, step_n)` | the agent loop (host, not model) | the current loop only | **request-scoped**; discarded on loop end | tool-call results used by the next step |
| `session` | `(session_id)` | the agent loop | the current session (across turns) | **24 h** or on explicit end | "the user is drafting a reply about ticket ZD-123456" |
| `user_pinned` | `(tenant_id, user_id, key)` | the user via an explicit UI action | the user's own sessions | **365 d** or until user removes | tone preference: "match the customer's register" |
| `tenant_shared` | `(tenant_id, key)` | tenant admins via an explicit UI action | any session in the tenant | **until an admin removes** | "we always sign replies 'The Acme Support Team'" |

The two rules this taxonomy encodes:

1. The model never writes to memory directly. It can *propose* a write (by emitting a tool call whose target is the memory tool), which the host validates, shows the user a confirm step if required, and then writes. Models cannot escalate memory privileges.
2. There is no "global" scope. Nothing crosses tenants. "This is useful to show every tenant" is a product request, not a memory scope; it belongs in a config repository, not a memory store.

### 2b. `memory_store.py` — reference implementation (deliverable 2)

```python
"""memory_store.py — product-agent memory with scopes, TTL, PII guard.

The write path is privileged: every write goes through `write()`, which
(a) checks the principal is allowed to write to the scope, (b) runs the
PII classifier, (c) refuses or redacts, (d) sets a TTL, (e) records an
audit log row. Reads are scoped by the principal; a session_id or
user_id from a different tenant never resolves.
"""

from __future__ import annotations

import json
import sqlite3
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from enum import Enum
from typing import Any, Iterable, Optional

from pii_classifier import PIIClassifier, PIIVerdict


class Scope(str, Enum):
    SCRATCHPAD = "scratchpad"
    SESSION = "session"
    USER_PINNED = "user_pinned"
    TENANT_SHARED = "tenant_shared"


SCOPE_TTL: dict[Scope, Optional[timedelta]] = {
    Scope.SCRATCHPAD: timedelta(minutes=10),   # hard cap; usually cleared on loop end
    Scope.SESSION: timedelta(hours=24),
    Scope.USER_PINNED: timedelta(days=365),
    Scope.TENANT_SHARED: None,                 # manual removal only
}


@dataclass(frozen=True)
class Principal:
    tenant_id: str
    user_id: str
    session_id: str
    roles: frozenset[str]  # {"user"} | {"user","tenant_admin"}


class MemoryError(Exception):
    """Raised at the memory boundary; carries `category` for the taxonomy."""

    def __init__(self, msg: str, category: str) -> None:
        super().__init__(msg)
        self.category = category  # principal | pii | scope | not_found


class MemoryStore:
    def __init__(self, db_path: str, pii: PIIClassifier) -> None:
        self._conn = sqlite3.connect(db_path, isolation_level=None)
        self._pii = pii
        self._conn.executescript(
            """
            CREATE TABLE IF NOT EXISTS memory (
              scope TEXT NOT NULL,
              tenant_id TEXT NOT NULL,
              user_id TEXT,
              session_id TEXT,
              key TEXT NOT NULL,
              value_json TEXT NOT NULL,
              created_at TEXT NOT NULL,
              expires_at TEXT,
              written_by TEXT NOT NULL,
              PRIMARY KEY (scope, tenant_id, user_id, session_id, key)
            );
            CREATE INDEX IF NOT EXISTS memory_expiry
              ON memory (expires_at) WHERE expires_at IS NOT NULL;
            CREATE TABLE IF NOT EXISTS memory_audit (
              id INTEGER PRIMARY KEY AUTOINCREMENT,
              at TEXT NOT NULL,
              action TEXT NOT NULL,  -- write | redact | refuse | read | erase | evict
              scope TEXT, tenant_id TEXT, user_id TEXT, session_id TEXT,
              key TEXT, principal_user TEXT, category TEXT, note TEXT
            );
            """
        )

    # ---- write path --------------------------------------------------------

    def write(
        self,
        scope: Scope,
        key: str,
        value: Any,
        principal: Principal,
        *,
        requires_user_confirmation: bool = False,
    ) -> None:
        self._check_writer(scope, principal, requires_user_confirmation)

        verdict = self._pii.classify(value)
        if verdict.action == "refuse":
            self._audit("refuse", scope, principal, key,
                        category="pii", note=verdict.reason)
            raise MemoryError(f"refused PII write: {verdict.reason}", "pii")
        if verdict.action == "redact":
            value = verdict.redacted_value
            self._audit("redact", scope, principal, key,
                        category="pii", note=verdict.reason)

        ttl = SCOPE_TTL[scope]
        now = datetime.now(tz=timezone.utc)
        expires_at = (now + ttl).isoformat() if ttl else None

        self._conn.execute(
            """INSERT OR REPLACE INTO memory
               (scope, tenant_id, user_id, session_id, key,
                value_json, created_at, expires_at, written_by)
               VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)""",
            (
                scope.value,
                principal.tenant_id,
                principal.user_id if scope != Scope.TENANT_SHARED else None,
                principal.session_id if scope in (Scope.SCRATCHPAD, Scope.SESSION) else None,
                key,
                json.dumps(value),
                now.isoformat(),
                expires_at,
                principal.user_id,
            ),
        )
        self._audit("write", scope, principal, key)

    # ---- read path ---------------------------------------------------------

    def read(self, scope: Scope, key: str, principal: Principal) -> Any:
        self._evict_expired()
        row = self._conn.execute(
            """SELECT value_json FROM memory
               WHERE scope = ? AND tenant_id = ? AND key = ?
                 AND (user_id = ? OR user_id IS NULL)
                 AND (session_id = ? OR session_id IS NULL)""",
            (scope.value, principal.tenant_id, key,
             principal.user_id, principal.session_id),
        ).fetchone()
        if row is None:
            raise MemoryError(f"no {scope.value} value at {key!r}", "not_found")
        self._audit("read", scope, principal, key)
        return json.loads(row[0])

    def list_user_pinned(self, principal: Principal) -> list[tuple[str, Any]]:
        self._evict_expired()
        rows = self._conn.execute(
            """SELECT key, value_json FROM memory
               WHERE scope = ? AND tenant_id = ? AND user_id = ?""",
            (Scope.USER_PINNED.value, principal.tenant_id, principal.user_id),
        ).fetchall()
        return [(k, json.loads(v)) for k, v in rows]

    # ---- right-to-erasure --------------------------------------------------

    def erase_user(self, tenant_id: str, user_id: str) -> int:
        """Wipe every memory row for one user across all scopes. Returns count."""
        cur = self._conn.execute(
            """DELETE FROM memory
               WHERE tenant_id = ? AND user_id = ?""",
            (tenant_id, user_id),
        )
        count = cur.rowcount
        self._conn.execute(
            """INSERT INTO memory_audit (at, action, tenant_id, user_id, note)
               VALUES (?, 'erase', ?, ?, ?)""",
            (datetime.now(tz=timezone.utc).isoformat(),
             tenant_id, user_id, f"rows_deleted={count}"),
        )
        return count

    # ---- eviction ----------------------------------------------------------

    def _evict_expired(self) -> None:
        now = datetime.now(tz=timezone.utc).isoformat()
        cur = self._conn.execute(
            "DELETE FROM memory WHERE expires_at IS NOT NULL AND expires_at <= ?",
            (now,),
        )
        if cur.rowcount:
            self._conn.execute(
                """INSERT INTO memory_audit (at, action, note)
                   VALUES (?, 'evict', ?)""",
                (now, f"rows_evicted={cur.rowcount}"),
            )

    # ---- principal + scope checks -----------------------------------------

    def _check_writer(
        self, scope: Scope, p: Principal, requires_user_confirmation: bool
    ) -> None:
        if not p.tenant_id or not p.user_id or not p.session_id:
            raise MemoryError("missing principal", "principal")
        if scope == Scope.USER_PINNED and not requires_user_confirmation:
            raise MemoryError(
                "user_pinned writes require an explicit user confirm",
                "scope",
            )
        if scope == Scope.TENANT_SHARED and "tenant_admin" not in p.roles:
            raise MemoryError(
                "tenant_shared writes require role=tenant_admin",
                "principal",
            )
        if scope == Scope.SCRATCHPAD and not requires_user_confirmation:
            # scratchpad is host-written; the "requires_user_confirmation" flag
            # is being abused here as "is host context," not a UX element.
            # Real impl: separate caller-identity check.
            pass

    # ---- audit -------------------------------------------------------------

    def _audit(
        self, action: str, scope: Scope, p: Principal, key: str,
        *, category: str | None = None, note: str | None = None,
    ) -> None:
        self._conn.execute(
            """INSERT INTO memory_audit (at, action, scope, tenant_id,
               user_id, session_id, key, principal_user, category, note)
               VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)""",
            (
                datetime.now(tz=timezone.utc).isoformat(),
                action, scope.value, p.tenant_id,
                p.user_id, p.session_id, key, p.user_id, category, note,
            ),
        )
```

### 2c. `pii_classifier.py` — write-time redaction (deliverable 3)

```python
"""pii_classifier.py — write-time PII classifier.

The classifier runs on *every* value written to memory. The output is a
verdict: allow, redact (with a cleaned value), or refuse. The field
lists and regexes below are a reference starting point; in a shipped
surface, they are owned by the privacy team and versioned.
"""

from __future__ import annotations

import re
from dataclasses import dataclass
from typing import Any, Literal


EMAIL = re.compile(r"[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}", re.I)
PHONE = re.compile(r"\+?\d[\d\-\s().]{8,}\d")
# 13–19 digit credit-card-like numbers; a final Luhn check would run in prod.
CARD  = re.compile(r"(?:\d[ -]?){13,19}")
# Simple US SSN pattern; real surfaces run a locale-aware library.
SSN   = re.compile(r"\b\d{3}-\d{2}-\d{4}\b")

# Keys that are always refused if present at the top level of a write.
DENY_KEYS = {"password", "secret", "api_key", "access_token", "session_token"}

# Keys we explicitly tolerate even if they look like PII (names in a
# "signature" preference, for example).
ALLOW_KEYS = {"preferred_signature"}


@dataclass(frozen=True)
class PIIVerdict:
    action: Literal["allow", "redact", "refuse"]
    reason: str
    redacted_value: Any | None = None


class PIIClassifier:
    def classify(self, value: Any) -> PIIVerdict:
        if isinstance(value, dict):
            for k in value:
                if k.lower() in DENY_KEYS:
                    return PIIVerdict("refuse", f"deny-listed key {k!r}")
            redacted = {k: self._scrub(v, allow=(k in ALLOW_KEYS))
                        for k, v in value.items()}
            if redacted != value:
                return PIIVerdict("redact", "scrubbed PII patterns", redacted)
            return PIIVerdict("allow", "clean")
        if isinstance(value, str):
            cleaned = self._scrub(value, allow=False)
            if cleaned != value:
                return PIIVerdict("redact", "scrubbed PII patterns", cleaned)
            return PIIVerdict("allow", "clean")
        return PIIVerdict("allow", "non-sensitive type")

    def _scrub(self, v: Any, *, allow: bool) -> Any:
        if not isinstance(v, str) or allow:
            return v
        s = v
        s = CARD.sub("[REDACTED_CARD]", s)
        s = SSN.sub("[REDACTED_SSN]", s)
        s = EMAIL.sub("[REDACTED_EMAIL]", s)
        s = PHONE.sub("[REDACTED_PHONE]", s)
        return s
```

### 2d. Retention schedule (deliverable 4)

```markdown
# helpdesk-reply-copilot — memory retention schedule (v1)

| Scope | TTL | Eviction cadence | Backup retention | Review owner |
|---|---|---|---|---|
| scratchpad     | 10 min hard cap; cleared on loop end | per request    | not backed up          | applied-AI |
| session        | 24 h                                 | nightly job    | 7 d incremental        | applied-AI + privacy |
| user_pinned    | 365 d since last write               | nightly job    | 30 d incremental       | privacy |
| tenant_shared  | until admin removes                  | on removal     | 30 d incremental       | privacy |

## Right to erasure
A `DELETE /me/memory` request clears user_pinned + session + scratchpad
for the calling user across the tenant in a single transaction and
writes one `erase` audit row. SLA: within 30 d per GDPR Art. 17 norms;
operationally, within 24 h.

## Review cadence
- Monthly: privacy team reviews the DENY_KEYS and ALLOW_KEYS lists.
- Quarterly: applied-AI + privacy review the TTLs against incident
  history and churn (if a scope's TTL is never hit in practice, shorten).
- Each schema change to the `memory` table: privacy sign-off before
  merge; the migration ships a backfill of expires_at.
```

### 2e. Unit tests (deliverable 5)

```python
"""test_memory_store.py — proves the four invariants of the memory system."""

import time
from datetime import datetime, timedelta, timezone

import pytest

from memory_store import MemoryStore, Scope, MemoryError, Principal
from pii_classifier import PIIClassifier


@pytest.fixture
def store(tmp_path):
    return MemoryStore(str(tmp_path / "mem.db"), PIIClassifier())


USER = Principal(tenant_id="t_1", user_id="u_1",
                 session_id="s_1", roles=frozenset({"user"}))
ADMIN = Principal(tenant_id="t_1", user_id="u_admin",
                  session_id="s_2", roles=frozenset({"user", "tenant_admin"}))
OTHER_TENANT = Principal(tenant_id="t_2", user_id="u_x",
                         session_id="s_3", roles=frozenset({"user"}))


class TestScopeTTLs:
    def test_session_expires(self, store, monkeypatch):
        store.write(Scope.SESSION, "ticket", "ZD-123", USER,
                    requires_user_confirmation=False)
        # fast-forward beyond the 24h TTL by stubbing datetime.now
        future = datetime.now(tz=timezone.utc) + timedelta(hours=25)
        monkeypatch.setattr(
            "memory_store.datetime",
            type("D", (), {"now": classmethod(lambda cls, tz=None: future)}),
        )
        with pytest.raises(MemoryError) as exc:
            store.read(Scope.SESSION, "ticket", USER)
        assert exc.value.category == "not_found"


class TestPIIGuard:
    def test_refuses_api_key(self, store):
        with pytest.raises(MemoryError) as exc:
            store.write(Scope.SESSION, "prefs",
                        {"api_key": "sk-xxx"}, USER)
        assert exc.value.category == "pii"

    def test_redacts_email_in_string(self, store):
        store.write(Scope.SESSION, "note",
                    "please contact alice@acme.com", USER)
        assert store.read(Scope.SESSION, "note", USER) \
            == "please contact [REDACTED_EMAIL]"


class TestTenantIsolation:
    def test_other_tenant_cannot_read(self, store):
        store.write(Scope.SESSION, "ticket", "ZD-123", USER)
        with pytest.raises(MemoryError) as exc:
            store.read(Scope.SESSION, "ticket", OTHER_TENANT)
        assert exc.value.category == "not_found"

    def test_tenant_shared_requires_admin(self, store):
        with pytest.raises(MemoryError) as exc:
            store.write(Scope.TENANT_SHARED, "signature",
                        "The Acme Team", USER,
                        requires_user_confirmation=True)
        assert exc.value.category == "principal"
        # Admin can write
        store.write(Scope.TENANT_SHARED, "signature",
                    "The Acme Team", ADMIN,
                    requires_user_confirmation=True)
        assert store.read(Scope.TENANT_SHARED, "signature", USER) \
            == "The Acme Team"


class TestErasure:
    def test_erase_user_wipes_scopes(self, store):
        store.write(Scope.SESSION, "ticket", "ZD-123", USER)
        store.write(Scope.USER_PINNED, "tone", "formal", USER,
                    requires_user_confirmation=True)
        count = store.erase_user("t_1", "u_1")
        assert count == 2
        with pytest.raises(MemoryError):
            store.read(Scope.USER_PINNED, "tone", USER)


class TestUserPinnedRequiresConfirm:
    def test_refuses_without_confirm(self, store):
        with pytest.raises(MemoryError) as exc:
            store.write(Scope.USER_PINNED, "tone", "formal", USER,
                        requires_user_confirmation=False)
        assert exc.value.category == "scope"
```

### 2f. Implementation notes

- The three-scope-plus-scratchpad taxonomy is the smallest one that covers the surface. Adding a "team" scope between `user_pinned` and `tenant_shared` is a product decision that takes a retention review; do not add it preemptively.
- The classifier is a write-time filter, not a read-time one. Reads are faster than writes on this workload; if PII slips past write-time classification, cleaning it at read-time is firefighting, not policy. The retention schedule's monthly review is how the deny- and allow-lists stay current.
- The model has no direct write access. The host decides when a memory write happens; the model can request one by emitting the `pin_preference` or `remember_signature` tool call (see [[exercise-01]] mod-104), which goes through the HITL confirm step from [[exercise-03]]. Models cannot elevate memory scope.
- `erase_user` is a single transaction across scopes. A partial erase is a privacy incident; the test for it is non-negotiable.
- The `expires_at` index exists so the nightly eviction scan is O(expired-rows), not O(all-rows). A memory store that scans the whole table every night becomes a p99 latency incident at scale.
- Audit is on by default. Every write, redact, refuse, read, erase, and evict writes a row. The audit table is itself subject to retention — typically 90 d live, 2 y cold — set by compliance, not applied-AI.
- Tenant isolation is enforced by the SQL `WHERE` clauses, not by the application layer. A memory system that depends on "the application remembers to pass tenant_id" is one forgotten join from a cross-tenant leak.
- The eval harness in [[exercise-04]] runs a smoke scenario where a session writes PII and asserts the store refuses or redacts. If the eval fails, the deny-list regressed; this is the first-line signal that the policy is drifting from the implementation.

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. The taxonomy names exactly four scopes with an explicit TTL and writer/reader rule for each. A fifth scope must come with a documented justification.
2. The `MemoryStore` interface is scope-typed: `write(scope, key, value, principal, ...)` — not an untyped `set(key, value)`. Scope is the first-class argument, not a key prefix convention.
3. The PII classifier runs on every `write` and emits one of `{allow, redact, refuse}`. A write that produces `refuse` raises; a write that produces `redact` stores the cleaned value and writes an audit row.
4. Tenant isolation is enforced in SQL, not in Python. A test proves a principal from tenant B cannot read tenant A's rows.
5. `erase_user(tenant_id, user_id)` wipes every row across scopes in a single transaction and writes an `erase` audit row. A test proves it covers at least session + user_pinned.
6. Scratchpad has a hard TTL (≤ 1 hour); it is a safety net for the "my loop crashed without clearing" case, not a design feature.
7. The retention schedule names TTLs, eviction cadence, backup retention, and the review owner for each scope.
8. The eviction query uses the `expires_at` index; `EXPLAIN QUERY PLAN` shows an indexed scan, not a seq scan.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Four-scope taxonomy with TTL + writer per scope | 1 | Table, not prose. |
| Scope-typed `write` / `read` APIs | 1 | `scope` is an enum argument. |
| Write-time PII classifier with allow/redact/refuse | 1 | Verdict type with an explicit action. |
| Deny-list for secret-shaped keys | 1 | `password`, `api_key`, `access_token` at minimum. |
| Tenant isolation enforced in SQL | 1 | Query has `tenant_id = ?`; test proves cross-tenant deny. |
| `erase_user` single-transaction + audit | 1 | One call wipes ≥ 2 scopes + audit row. |
| Retention schedule names owner + cadence | 1 | Privacy + applied-AI roles listed by scope. |
| Audit on every write / read / erase | 1 | `memory_audit` table populated by every path. |
| Unit tests cover TTL, PII, tenant, erase, confirm | 1 | Five distinct test classes, each with ≥ 1 assertion that would fail under a regression. |
| Eviction is index-driven | 1 | `expires_at` index exists; nightly job uses it. |

A submission scoring < 6 is rejected. The hard failures are (a) a global/no-scope memory, (b) application-layer tenant isolation with no SQL `WHERE tenant_id = ?`, (c) partial erasure (one scope cleaned, another missed), (d) no write-time PII classifier.

## 5. Common mistakes

- **Treating memory as a vector store.** A product-agent memory is scoped, typed, tenanted, and audited. Vector search is one implementation of one scope's read path, not the design.
- **Letting the model write directly.** The model can *propose* a memory write via a tool call; the host decides whether to persist. If the model can write, the model can leak another user's data into a prompt.
- **One "memory" scope.** A single untyped store becomes everyone's junk drawer: scratchpad bleeds into user preferences, which bleeds into tenant policy, with no retention story for any of them.
- **No TTL on the fallback paths.** Scratchpad looks ephemeral because the loop clears it on exit; when the loop crashes without clearing, the 10-minute hard cap is the thing that keeps the table from growing forever.
- **PII classifier at read time.** By the time a value hits the read path it's already been stored, backed up, and exported. Classify on write.
- **Tenant isolation in Python.** `if memory["tenant"] == principal.tenant_id: return memory["value"]` is one forgotten line from a cross-tenant leak. The `WHERE tenant_id = ?` does the work the application cannot be trusted to do.
- **Partial erasure.** A right-to-erasure endpoint that clears user_pinned but forgets session memory is a worse failure than no endpoint, because it looks compliant. One transaction, every scope.
- **Audit as an afterthought.** An audit table added two months after the memory ships has no backfill; you cannot answer "did we leak this value?" for anything before the add-date.
- **Unbounded `tenant_shared`.** The one scope with no TTL is also the scope most likely to accumulate stale policy strings. Its removal path is not optional.
- **Deny-list drift.** The classifier's regexes and key lists age. A quarterly review is the floor; incidents from [[mod-110]] feed new entries.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the per-scope retention schedule, audit log, and review cadence map directly to the RMF's MEASURE (measurable retention + classification outcomes) and MANAGE (owned review cadence; incident-driven updates) functions. The RMF is explicit that AI-system data stores need governance that is reviewed on a schedule, not stood up once.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the write-time PII classifier addresses input- and training-data integrity concerns; the deny-list for secret-shaped keys is the surface's answer to the OWASP guidance on preventing sensitive-data exposure through model-adjacent stores.
- MITRE ATLAS — <https://atlas.mitre.org/> — several ATLAS techniques exploit cross-session or cross-tenant memory to exfiltrate data or escalate privileges. The tenant-isolation-in-SQL + model-cannot-write-directly design is the control mapped to those techniques.

Related exercises: [[exercise-01]] (the bounded-tool-loop shape makes `session` the natural per-loop memory scope); [[exercise-03]] (user_pinned + tenant_shared writes go through the HITL confirm path); [[exercise-04]] (the eval harness runs a PII-write smoke test against this store); upstream [[exercise-01]] (mod-104) defines the `remember_signature` / `pin_preference` tool calls whose writes land here.
