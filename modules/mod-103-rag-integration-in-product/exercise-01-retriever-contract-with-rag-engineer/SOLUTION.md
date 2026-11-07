### exercise-01: Retriever Contract With the Rag Engineer — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to write — and wire into code — the contract between a product surface and the `rag-engineer` peer track's retriever. The contract is the interface the two teams agree to defend across eval regressions, incident retros, and schema changes. It has three load-bearing parts: **per-source freshness SLOs**, a **coverage manifest with named exclusions**, and an **ACL enforcement pattern** that fails closed. The deliverables wrap it in a thin typed client and three surface gates so the contract has runtime teeth, not just prose.

This solution provides:

- A reference `retriever-contract.yaml` for the helpdesk-reply-copilot surface across the three named sources (help center, Zendesk tickets, Salesforce knowledge).
- A reference `retriever_client.py` that requires `tenant_id`, `user_id`, `surface_id`; rejects unknown filter keys; returns `index_freshness` and `versions` on every call; and looks up score buckets by `surface_id`.
- A reference test suite showing the three surface gates (freshness, coverage, ACL fail-closed) firing on both the healthy and degraded paths.
- A `review.md` memo defending the retriever-enforced ACL pattern, naming the surface-side anti-patterns the contract forecloses, and specifying the change-management protocol.
- A grading rubric a reviewer uses to mark a learner submission in under 15 minutes.

Learners do not have to match the reference artifacts line for line — the exercise is about the *discipline*, not a single defensible answer. The acceptance checklist in §3 is the contract the solution scores against.

## 2. Worked answer or implementation

### 2a. `retriever-contract.yaml` (deliverable 1)

```yaml
version: 1
surface_id: helpdesk-reply-copilot
owner_team: applied-ai
peer_team: rag-engineer
changelog:
  - version: 1
    date: 2026-11-06
    author: applied-ai
    summary: Initial contract — ships alongside the first two enterprise tenants.

sources:
  helpcenter.acme.com:
    description: Public help articles, crawler-indexed.
    freshness:
      owner: rag
      p95_seconds: 3600       # 1h end-to-end from publish → searchable
      p99_seconds: 10800      # 3h
      measurement_clock: source_published_at_to_index_visible_at
      on_degraded:
        owner: applied-ai
        behaviour: Mark passages from this source with stale_source_warning in the evidence panel; do not refuse the answer.
    coverage:
      owner: rag
      inclusion_rule: All published articles with status=live and locale=en-US.
      known_exclusions:
        - drafts
        - archived articles
        - non-en-US locales
      target_coverage: 0.98
      on_degraded:
        owner: applied-ai
        behaviour: low_coverage_warning flag in response; evidence panel names the degraded source.

  zendesk.tickets:
    description: Per-tenant solved/closed tickets, ingested via webhook.
    freshness:
      owner: rag
      p95_seconds: 60         # 60s webhook → searchable
      p99_seconds: 300
      measurement_clock: zendesk_resolved_at_to_index_visible_at
      on_degraded:
        owner: applied-ai
        behaviour: stale_source_warning as above.
    coverage:
      owner: rag
      inclusion_rule: tenant's tickets with status in (solved, closed) and macro_quality >= 0.5.
      known_exclusions:
        - open tickets
        - spam-flagged tickets
        - tickets tagged internal-only
      target_coverage: 0.95
      on_degraded:
        owner: applied-ai
        behaviour: low_coverage_warning; surface the per-tenant stat in the Why-this-answer trace.

  salesforce.knowledge:
    description: Per-tenant internal CS-authored articles.
    freshness:
      owner: rag
      p95_seconds: 600        # 10 min
      p99_seconds: 1800
      measurement_clock: salesforce_last_modified_to_index_visible_at
      on_degraded:
        owner: applied-ai
        behaviour: stale_source_warning.
    coverage:
      owner: rag
      inclusion_rule: KnowledgeArticle objects with PublishStatus=Online and VisibleInApp=true.
      known_exclusions:
        - drafts
        - archived
        - VisibleInApp=false (even if Online)
      target_coverage: 0.97
      on_degraded:
        owner: applied-ai
        behaviour: low_coverage_warning; surface the exclusion list to CS in the readout.

acl:
  pattern: retriever_enforced
  rationale: |
    Two enterprise tenants are onboarding under DPAs that require fail-closed
    enforcement with ≤ 60s revocation propagation. A post-retrieval surface filter
    opens an existence oracle (the retriever would still find the passage and the
    filter would silently drop it, leaking a timing side-channel). Physical
    isolation is operationally infeasible across three per-tenant sources on a
    single Qdrant cluster. Retriever-enforced ACL is the only pattern that lets
    the surface pass through an auditable principal and have the retriever return
    zero passages — not "passages the surface then hides" — for denied principals.
  revocation_propagation:
    owner: rag
    p95_seconds: 45
    p99_seconds: 60
    measurement_clock: idp_revocation_event_to_retriever_denies_principal
    on_degraded:
      owner: applied-ai
      behaviour: no_evidence branch — surface refuses to proceed to generation and returns an "ACL check unavailable" reason; logs an incident ticket.
  principals:
    required_fields: [tenant_id, user_id, surface_id]
    failure_mode: fail_closed
    audit_log: Every retrieve call logs (tenant_id, user_id, surface_id, request_id, decision).

client_contract:
  required_request_fields: [tenant_id, user_id, surface_id, query]
  required_response_fields: [passages, index_freshness, versions, request_id]
  versions_fields: [embedding, index, reranker, contract_version]
  filter_schema_version: 1
  score_thresholds:
    owned_by: rag
    surface_lookup: SURFACE_SCORE_BUCKETS[surface_id]
    hardcoded_in_surface: forbidden
```

The `on_degraded` rows name a specific surface behaviour, not a generic banner. The ACL pattern meets the 60s propagation number with a 15s safety margin (p95) and names fail-closed explicitly.

### 2b. `retriever_client.py` (deliverable 2)

```python
"""retriever_client.py — thin typed client for the helpdesk-reply-copilot surface.

Enforces the contract in retriever-contract.yaml at the client boundary. The
retriever itself validates too; this file is the surface's early-catch layer.
"""

from __future__ import annotations

from dataclasses import dataclass, field
from typing import Any, Literal

CONTRACT_VERSION = 1

# What the surface may put in a filter. Any other key is a client-side error.
# Owned jointly; a new filter key is a breaking change (see review.md).
FILTER_SCHEMA: dict[str, type] = {
    "source": str,          # one of the named sources in the contract
    "doc_ids": list,        # explicit allowlist, for pinning
    "updated_after": str,   # ISO-8601 timestamp
    "locale": str,
}

# Score buckets are published by rag, keyed by surface_id. The surface MUST
# NOT hard-code numeric thresholds.
SURFACE_SCORE_BUCKETS: dict[str, dict[str, float]] = {
    "helpdesk-reply-copilot": {"low": 0.35, "medium": 0.55, "high": 0.72},
}


class ContractViolation(Exception):
    """Raised at the client boundary when the surface breaks the contract."""


@dataclass(frozen=True)
class RetrieveRequest:
    tenant_id: str
    user_id: str
    surface_id: str
    query: str
    k: int = 8
    filters: dict[str, Any] = field(default_factory=dict)

    def __post_init__(self) -> None:
        for name in ("tenant_id", "user_id", "surface_id", "query"):
            if not getattr(self, name):
                raise ContractViolation(f"{name} is required")
        for key in self.filters:
            if key not in FILTER_SCHEMA:
                raise ContractViolation(
                    f"unknown filter key {key!r}; "
                    f"allowed: {sorted(FILTER_SCHEMA)} (schema v1)"
                )
            expected = FILTER_SCHEMA[key]
            if not isinstance(self.filters[key], expected):
                raise ContractViolation(
                    f"filter {key!r} must be {expected.__name__}"
                )


@dataclass(frozen=True)
class Passage:
    passage_id: str
    doc_id: str
    source: str
    text: str
    score: float
    updated_at: str


@dataclass(frozen=True)
class RetrieveResponse:
    passages: list[Passage]
    index_freshness: dict[str, Literal["healthy", "degraded"]]
    coverage: dict[str, dict[str, float]]  # per-source {"actual": .., "target": ..}
    versions: dict[str, str]  # embedding, index, reranker, contract_version
    request_id: str


class RetrieverClient:
    """Mock retriever. Replace `_call_backend` with the real HTTP/gRPC call."""

    def __init__(self, backend=None) -> None:
        self._backend = backend  # injected for tests

    def retrieve(self, request: RetrieveRequest) -> RetrieveResponse:
        response = self._call_backend(request)
        # Defensive: the retriever is contractually obliged to include these;
        # the client surfaces a loud failure if they are missing.
        for field_name in ("passages", "index_freshness", "versions", "request_id"):
            if field_name not in response:
                raise ContractViolation(
                    f"retriever response missing required field {field_name!r}"
                )
        if str(response["versions"].get("contract_version")) != str(CONTRACT_VERSION):
            raise ContractViolation(
                f"contract version mismatch: client={CONTRACT_VERSION} "
                f"retriever={response['versions'].get('contract_version')}"
            )
        return RetrieveResponse(**response)

    def _call_backend(self, request: RetrieveRequest) -> dict[str, Any]:
        if self._backend is None:
            raise RuntimeError("no backend configured; inject one for tests")
        return self._backend(request)


def score_bucket(surface_id: str, score: float) -> Literal["low", "medium", "high"]:
    """Mediated score lookup. The surface never hard-codes a number."""
    buckets = SURFACE_SCORE_BUCKETS[surface_id]
    if score >= buckets["high"]:
        return "high"
    if score >= buckets["medium"]:
        return "medium"
    return "low"
```

Key properties: required fields raise `ContractViolation` *before* the network hop; the filter schema is versioned; `versions.contract_version` is checked both directions; score thresholds are never numeric literals on the surface.

### 2c. The three surface gates + tests (deliverable 3)

```python
"""surface_gates.py — freshness, coverage, and ACL fail-closed guards."""

from dataclasses import dataclass
from typing import Any

@dataclass
class GateDecision:
    proceed: bool
    stale_source_warning: bool = False
    low_coverage_warning: bool = False
    degraded_sources: list[str] | None = None
    low_coverage_sources: list[str] | None = None
    no_evidence: bool = False
    reason: str | None = None


def freshness_gate(index_freshness: dict[str, str]) -> GateDecision:
    degraded = [s for s, state in index_freshness.items() if state != "healthy"]
    return GateDecision(
        proceed=True,
        stale_source_warning=bool(degraded),
        degraded_sources=degraded or None,
    )


def coverage_gate(coverage: dict[str, dict[str, float]]) -> GateDecision:
    low = [s for s, c in coverage.items() if c["actual"] < c["target"]]
    return GateDecision(
        proceed=True,
        low_coverage_warning=bool(low),
        low_coverage_sources=low or None,
    )


def acl_gate(acl_check_result: dict[str, Any] | None) -> GateDecision:
    # Fail-closed: any error / timeout / None → no_evidence, no generation.
    if acl_check_result is None or acl_check_result.get("error"):
        return GateDecision(
            proceed=False,
            no_evidence=True,
            reason="we couldn't verify permissions for this request",
        )
    if acl_check_result.get("denied"):
        return GateDecision(
            proceed=False,
            no_evidence=True,
            reason="no results you have access to",
        )
    return GateDecision(proceed=True)
```

```python
"""test_gates.py — pytest suite for the three gates."""

import pytest
from surface_gates import freshness_gate, coverage_gate, acl_gate


class TestFreshnessGate:
    def test_all_healthy(self):
        d = freshness_gate({"helpcenter": "healthy", "zendesk": "healthy"})
        assert d.proceed and not d.stale_source_warning

    def test_degraded_source_raises_warning(self):
        d = freshness_gate({"helpcenter": "healthy", "zendesk": "degraded"})
        assert d.proceed and d.stale_source_warning
        assert d.degraded_sources == ["zendesk"]


class TestCoverageGate:
    def test_above_target(self):
        d = coverage_gate({"helpcenter": {"actual": 0.99, "target": 0.98}})
        assert not d.low_coverage_warning

    def test_below_target_raises_warning(self):
        d = coverage_gate({"zendesk": {"actual": 0.80, "target": 0.95}})
        assert d.low_coverage_warning
        assert d.low_coverage_sources == ["zendesk"]


class TestACLGate:
    def test_allowed(self):
        d = acl_gate({"denied": False})
        assert d.proceed and not d.no_evidence

    def test_denied(self):
        d = acl_gate({"denied": True})
        assert not d.proceed and d.no_evidence

    def test_error_fails_closed(self):
        d = acl_gate({"error": "idp timeout"})
        assert not d.proceed and d.no_evidence
        assert "couldn't verify permissions" in d.reason

    def test_none_fails_closed(self):
        d = acl_gate(None)
        assert not d.proceed and d.no_evidence
```

Each gate has a one-paragraph setup and a one-line assertion, as the exercise prescribes.

### 2d. `review.md` (deliverable 4)

```markdown
# Retriever Contract Review Memo — helpdesk-reply-copilot v1

## ACL pattern choice: retriever-enforced

We chose retriever-enforced ACL because the two enterprise DPAs require
fail-closed enforcement with ≤ 60s revocation propagation. A post-retrieval
filter would leave the retriever returning passages the surface then drops,
which opens two leaks:

1. An existence oracle via response timing — the "has results / no results"
   distinction is observable in latency even when the surface returns
   "no results."
2. A log-of-reads artefact — the retriever logs will record a hit against
   a passage the user is not permitted to see, which fails our DPA's
   "no record of denied principal reads" clause.

Physical isolation (one index per tenant) is operationally infeasible across
three per-tenant sources on a single Qdrant cluster given the sprint budget.
Retriever-enforced ACL is the only pattern that terminates the read before
observation.

## What the surface refuses to reach into

- The vector store directly. The retriever client is the only path.
- Any `_private.*` metadata fields the retriever may happen to return.
- Numeric score thresholds. The surface uses `SURFACE_SCORE_BUCKETS[surface_id]`;
  if the rag team re-trains the embedder and shifts the score distribution,
  the surface does not need a redeploy.
- Per-source behaviour, e.g., branching on `source == "zendesk.tickets"`.
  All source-specific logic lives in the manifest.

## Change-management protocol

- **Patch** (contract_version bumps minor): adding a known_exclusion, adding
  a new `on_degraded` action, documenting an incident. Surface team is
  notified; no surface change required.
- **Breaking** (contract_version bumps major): changing a freshness SLO upward,
  adding a required field, changing the ACL pattern, changing the filter
  schema, changing the shape of `versions` or `index_freshness`. Requires a
  PR with a `# breaking-change:` block, two-week deprecation window, and a
  surface-team sign-off.
- Retriever signals a change by opening a PR against this file in the
  shared `retriever-contracts` repo. The surface CI fails on any contract
  version drift until the surface pins the new version.

## One question for the retriever team, where the joint gold set is the
deciding evidence

> The reranker rollout is proposed for next sprint. On the gold set from
> exercise-03, what is the delta in `recall@5` on the `long_tail` and
> `multi_hop` strata with and without the reranker? If the delta on
> `multi_hop` is below +0.08, we will not ship the reranker and will
> redirect the latency budget to query rewriting in exercise-04.
```

### 2e. Implementation notes

- The YAML is *the* source of truth. The client's constants (`CONTRACT_VERSION`, the required fields, `SURFACE_SCORE_BUCKETS`) should be code-generated from it, or at least linted against it in CI. A stretch-goal CI script that fails a build when the two drift is strongly recommended.
- `FILTER_SCHEMA` lives in the client because the surface needs to catch typos *before* the network hop (the Pydantic-at-the-boundary pattern from mod-102 chapter 02). The retriever still validates; the client's job is an early-catch, not a security boundary.
- The ACL gate's reason strings are user-visible in the "no good evidence" UX from [[exercise-02]]. Keep them precise — "couldn't verify permissions" is not the same as "denied."

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. `retriever-contract.yaml` names all three sources and, per source, has a freshness block (p95, p99, measurement clock, on_degraded), a coverage block (inclusion rule, known exclusions, target), and an owner tag.
2. The ACL block names a single pattern, sets `revocation_propagation.p99_seconds` ≤ 60, and names `fail_closed` explicitly.
3. Every SLO has an `on_degraded` row naming a specific surface behaviour, not a banner. Grep the file for `warning` / `warning_` / `no_evidence` and confirm a UX element is named.
4. `retriever_client.py` rejects a `RetrieveRequest(tenant_id="", ...)` at construction, before any network hop. A pytest asserting `pytest.raises(ContractViolation)` is enough.
5. The same client rejects a filter dict containing `{"custom_key": "foo"}` with a message that lists the allowed keys.
6. Every mock response returned by the test harness includes `index_freshness`, `coverage`, and `versions`. A client-side check confirms the `versions.contract_version` matches the local constant.
7. No numeric score threshold appears on the surface. `grep -rn "score *[<>]=" surface/` returns zero matches outside the `score_bucket` helper.
8. The three gate tests all pass; each has one `healthy` case and one `degraded` / failure case.
9. `review.md` names the two rejected ACL patterns with one-line justifications each (post-retrieval filter leaks timing; physical isolation infeasible) and states a change-management protocol with at least one `patch` and one `breaking` example.
10. The eval-reference question in `review.md` points at a specific gold-set stratum from [[exercise-03]] with a named threshold, not a generic "please measure."

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Contract covers all three sources | 1 | Every source has freshness, coverage, ACL referenced in the block. |
| Freshness SLOs are measurable | 1 | p95 and p99 named in seconds; measurement clock is a specific two-endpoint definition, not "end-to-end." |
| Coverage manifest names exclusions | 1 | At least one named exclusion per source; the target coverage is a number. |
| ACL pattern is defensible | 2 | One pattern named; rejected patterns named with one-line reason each; propagation SLO ≤ 60s; fail-closed written out. |
| `on_degraded` rows name UX | 1 | Every row names a specific flag / UX element / branch, not a generic banner. |
| Client enforces required principals | 1 | `tenant_id`, `user_id`, `surface_id` raise at construction, before network. |
| Filter schema version + rejection | 1 | Unknown filter key rejected with a schema-aware error; version field present. |
| Three gates have passing tests | 1 | Pytest run is green; each gate has both healthy and degraded cases. |
| Review memo + change-management | 1 | ACL rationale is specific to the DPA constraint; change-management names breaking vs. patch with examples. |

A submission scoring < 6 is rejected and sent back for a second pass. The checklist that matters most is lines 1–4 — a contract that doesn't fail closed is not a contract.

## 5. Common mistakes

- **Freshness SLO without a measurement clock.** "60s freshness" is unverifiable. The clock has to name two endpoints: publish-at and index-visible-at, with explicit tie-break for races.
- **One ACL pattern for all sources.** Two tenants may legitimately want different patterns (retriever-enforced for `zendesk.tickets`, post-retrieval for a public source). The contract should permit per-source ACL *or* justify a single pattern across all three; a hand-wave is not enough.
- **Hard-coded score thresholds on the surface.** The classic failure mode: a dev tries `if passage.score > 0.72: cite_it`, the retriever re-trains the embedder next quarter, and the surface silently filters out everything. The `score_bucket` indirection is non-negotiable.
- **Treating ACL as a surface filter.** A post-retrieval filter is not retriever-enforced ACL. The contract must require the retriever to never *return* a denied passage, not to return it and have the surface hide it. The timing side-channel in `review.md` is the discussion to surface.
- **Missing `versions` or `index_freshness`.** The client should refuse a response that lacks either. If the retriever "forgets" to include them, the surface loses the ability to render the stale-source warning and the "Why this answer" trace from [[exercise-02]].
- **No change-management clause.** Without `breaking` vs. `patch` definitions, the next reranker upgrade will silently change the score distribution and the surface will either over-cite (bucket boundaries wrong) or under-cite (bucket boundaries wrong the other way).
- **"We'll version this later."** `version: 1` with a changelog stub is the minimum. A versionless contract is one PR away from a prod incident.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the ACL fail-closed clause is the surface-level answer to ML06 (output integrity attacks) and ML02 (data poisoning), where an injected passage must never reach the generator for a denied principal.
- MITRE ATLAS — <https://atlas.mitre.org/> — retriever-enforced ACL blunts ATLAS techniques around credential access to RAG-indexed secrets and existence-oracle reconnaissance.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the contract is a MAP / MANAGE artefact in RMF terms: it records the context (MAP) and names the controls that fire on breach (MANAGE). The `on_degraded` rows are the MEASURE hook.

Related exercises: [[exercise-02]] consumes `index_freshness` for the stale-source badge and the "no good evidence" branch; [[exercise-03]] is where the joint gold set referenced in `review.md` lives; [[exercise-04]] adds a classifier that may change the effective source mix, which the contract's change-management clause has to accommodate.
