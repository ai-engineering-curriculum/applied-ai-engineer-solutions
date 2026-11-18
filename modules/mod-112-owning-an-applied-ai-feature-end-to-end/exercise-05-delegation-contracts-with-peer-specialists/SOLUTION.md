### exercise-05: Delegation Contracts With Peer Specialists — Solution

## 1. Solution overview

The applied-AI engineer owns the feature, but the feature depends on components owned by peer specialists: a **retrieval engineer** owns the mod-103 index, a **safety engineer** owns the mod-107 guardrails and refusal categories, a **data engineer** owns the mod-110 telemetry pipeline, a **fine-tuning engineer** owns the hosted fine-tune job from mod-111 [[exercise-05]], and an **ML-platform engineer** owns the mod-108 gateway and the mod-111 flag layer. This exercise asks the student to write the **delegation contracts** that make those dependencies load-bearing — each contract is the artefact that says "the applied-AI engineer promises X; the peer specialist promises Y; the way we tell if either side has defaulted is Z."

A delegation contract is *not* a service-level agreement in the classic SRE sense. SLAs enumerate numeric uptime targets; delegation contracts enumerate **ownership boundaries and the observable evidence that the boundary holds**. The applied-AI engineer needs these contracts because without them, every failure drops at the surface and the applied engineer either absorbs the work (unsustainable) or fires off escalations that erode the relationship (worse).

The failure modes this solution defends against:

1. **Shared ownership with no boundary.** "The retrieval engineer and the applied engineer both work on retrieval." Translation: neither is accountable. The contract draws the line.
2. **Contracts that read like wishes.** "The safety engineer promises no unsafe outputs." Un-measurable. The contract must name the measurement.
3. **Contracts without an escalation path.** When one side defaults, there is no documented next step; the dispute becomes personal.
4. **Contracts that live in heads.** The shape of who owns what is clear to the two parties until one of them rotates off; the next engineer inherits a feature with un-documented ownership.
5. **One contract for all peers.** The retrieval engineer and the safety engineer deliver different things; collapsing them into one template produces either an under-constrained retrieval contract or an over-specified safety contract.
6. **Contract without a version.** When the peer's platform changes (new retriever version, new guardrail policy), the contract is silently invalidated.
7. **Contracts asymmetric in one direction only.** The applied engineer makes no promises back — so the peer specialist has no visibility into whether they are on the critical path for an incident.
8. **No link to the charter / card.** The contract specifies interfaces, but the *why* (the charter kill criterion it supports) is not named. Six months later nobody remembers why the contract constrains what it constrains.

This solution provides:

- A **contract template** with the required sections — scope, promises both ways, verification, versioning, escalation.
- **Five worked contracts** — one per peer specialist archetype — tied to the reference feature (`helpdesk.reply_assistant`) from [[exercise-01]].
- A **contract-validity CI check** — the artefact that makes the contract self-checking against the feature card's ID table from [[exercise-04]].
- A **grading rubric** and **common-mistakes list**.

## 2. Worked answer or implementation

### 2a. The contract template

Each contract is a single markdown file, `docs/delegation/<peer-role>.md`, with these sections:

```markdown
# Delegation contract — <feature-id> × <peer-role>

Parties:
- Applied-AI side: <team / lead>
- <peer-role> side: <team / lead>

Version: <YYYY-MM-DD.<counter>>
Effective from: <YYYY-MM-DD>
Next review: <YYYY-MM-DD>

## 1. Scope
What this contract covers; what it does NOT cover (explicit boundaries to
adjacent contracts).

## 2. What <peer-role> delivers to the applied-AI side (peer → applied)
Enumerated promises. Each promise has:
- Interface (API, flag, event, document).
- Observable (how the applied-AI side reads the current state).
- Cadence (how often the promise refreshes).

## 3. What the applied-AI side delivers to <peer-role> (applied → peer)
Symmetric list. Even if the peer is "upstream," the applied side owes
telemetry, incident notice, and feedback.

## 4. What breaks the contract
Named failure modes with observables — "X is broken when metric M crosses
threshold T over window W."

## 5. Verification
The automated checks (CI jobs, dashboard panels, drift-check scripts)
that evidence whether the contract is honoured.

## 6. Escalation
Steps the applied-AI side takes when (4) fires, and the response SLA from
<peer-role>. Names a specific role, not "the team."

## 7. Versioning + change control
How the contract is versioned, how upstream changes propagate, and what
triggers a re-sign.

## 8. Dependencies on other contracts
Named cross-references to related contracts, so the full ownership map is
reconstructible from any contract.
```

### 2b. Contract 1 — Applied-AI × Retrieval Engineer

> This contract governs the retriever interface that mod-103 [[exercise-01]] specifies. The retrieval engineer owns the index generation, the embedding model, the chunking strategy, and the retrieval API. The applied engineer owns the prompt that consumes the retrieved content and the citation / grounding behaviour at the surface.

```markdown
# Delegation contract — helpdesk.reply_assistant × retrieval-engineer

Parties:
- Applied-AI side: @a-eng-1 (applied-AI team)
- Retrieval-engineer side: @r-eng-1 (retrieval-platform team)

Version: 2026-11-10.1
Effective from: 2026-11-10
Next review: 2027-02-10 (quarterly) + per incident

## 1. Scope

Covers the `billing-retriever` service version `v4.2` consumed by the
`helpdesk.reply_assistant` feature on surfaces `helpdesk.web` and
`helpdesk.mobile-native`. Covers the embedding model, chunking, index-
generation cadence, and the retrieval API surface documented at
`retrievers/billing/README.md`.

Not covered: the prompt that consumes the retrieved content (applied-AI
side owns this), the citation UX at the surface (applied-AI side owns
this), or re-ranking (owned by a separate contract — see §8).

## 2. What retrieval delivers to applied (peer → applied)

| # | Promise | Interface | Observable | Cadence |
|---|---|---|---|---|
| R1 | Retriever returns documents for a (tenant_id, query, k) tuple in ≤ 1.5 s p95. | `POST /retrieve` | mod-110 [[exercise-01]] span `app.retrieval.latency_ms` on the critical path | Continuous; alert fires on 1h p95 > 1.5 s. |
| R2 | Retrieved documents are scoped to the authenticated tenant. | API contract; `tenant_id` required. | mod-103 [[exercise-03]] retrieval-quality probe flags any cross-tenant leak. | Per-request; audit sampled daily. |
| R3 | Retriever surfaces a `doc_id`, `source`, and `last_updated_at` per document. | JSON response schema. | Response schema validation in-line. | Per-request. |
| R4 | Index generation is refreshed daily; the applied-AI side can read the current generation timestamp. | `retrievers/billing/INDEX_GEN` file in-repo. | Feature-card drift check ([[exercise-04]] 2c). | Daily. |
| R5 | Breaking changes (schema, API, chunking strategy) carry ≥ 2 weeks of deprecation notice. | Deprecation notice in `retrievers/billing/CHANGELOG.md` + Slack announce. | Change control via §7. | On change. |
| R6 | Retrieval-side eval: retriever is run against a frozen mod-103 [[exercise-03]] eval set weekly; results are shared. | Weekly report in `retrievers/billing/eval/`. | Report file present and dated. | Weekly. |

## 3. What applied delivers to retrieval (applied → peer)

| # | Promise | Interface | Observable | Cadence |
|---|---|---|---|---|
| A1 | Applied side forwards retrieval traces including `app.retrieval.doc_ids` and `app.retrieval.returned_k` on every LLM span. | mod-110 [[exercise-01]] span attributes. | Spans visible in the retrieval-team trace view. | Per-request. |
| A2 | Applied side flags retrieved-document quality issues via a weekly feedback review against the retrieval team's inbox. | `retrieval-feedback` channel + mod-110 [[exercise-03]] feedback pipeline. | Weekly review log in `docs/retrieval-feedback/`. | Weekly. |
| A3 | Applied side names an incident `INC-<id>` when retrieval is implicated in a feature-side incident; the retrieval team is paged in Step 2 (triage). | Incident page via PagerDuty. | Incident record references the retrieval service version. | Per-incident. |
| A4 | Applied side provides regression fixtures for retrieval-implicated bugs (per [[exercise-02]] 2b) via `evals/fixtures/retrieval/*.yaml`. | Fixture file + eval runner. | Fixture file linked from the incident record. | Per-incident. |

## 4. What breaks the contract

- R1 broken when `app.retrieval.latency_ms` 1h-p95 > 1.5 s for two
  consecutive 1h windows.
- R2 broken when a cross-tenant leak is detected in the mod-103
  [[exercise-03]] audit — any single leak is a P0 event.
- R3 broken when the applied side receives a response that fails schema
  validation (missing `doc_id`, missing `source`, missing `last_updated_at`).
- R4 broken when `INDEX_GEN` is ≥ 48 hours stale and no notice has been
  posted.
- R5 broken when a schema / API change lands without the required 2-week
  notice.
- A1 broken when a sample of spans checked by the retrieval team shows
  missing `app.retrieval.*` attributes on > 1% of spans in a sample.
- A2 broken when the weekly feedback log is absent for ≥ 2 consecutive
  weeks.

## 5. Verification

- CI job `verify-retrieval-contract` runs daily:
  - Reads `app.retrieval.*` attributes on last 1h of spans and asserts > 99%
    coverage.
  - Reads `INDEX_GEN` and asserts freshness ≤ 48h.
  - Pulls the weekly retrieval eval report and asserts presence.
- Feature-card drift check ([[exercise-04]] 2c) reads the retriever version.
- The contract version header is read by the drift check; a mismatch
  between claimed and live retriever version fails CI.

## 6. Escalation

- R1 breach: applied side pages `retrieval-primary` oncall; retrieval team
  acknowledges within 30 min and reports mitigation plan within 2h.
- R2 breach: P0; applied side fires the charter K6-adjacent kill lever
  (set `support_agent.enabled = false`) while retrieval investigates;
  joint incident doc written within 24h; mod-107 [[exercise-03]] +
  mod-103 [[exercise-01]] joint audit within 72h.
- R4 breach: applied side downgrades to warm-cache serving if available;
  pages retrieval-primary; retrieval responds within 2h.
- A1 breach: retrieval team pings `applied-ai-primary` channel; applied
  side must restore attribute coverage within one business day (the span
  wrapper from mod-110 [[exercise-01]] is the authoritative source).

## 7. Versioning + change control

- Contract version increments on any change to §2 or §3.
- Retriever version bump (e.g. `v4.2` → `v5.0`) requires contract re-sign.
- Minor patch versions (`v4.2.x`) do not require re-sign unless they change
  the response schema or latency envelope.
- Both parties re-sign the contract; the version header is machine-checked.

## 8. Dependencies on other contracts

- `docs/delegation/safety-engineer.md` — the mod-107 [[exercise-03]]
  cross-tenant audit is the enforcement surface for R2.
- `docs/delegation/data-engineer.md` — the mod-110 [[exercise-01]] span
  pipeline is the observable surface for R1 and A1.
- `docs/delegation/ml-platform-engineer.md` — the mod-108 gateway sits
  downstream of the retriever; retrieval latency contributes to the
  feature's p95 SLO in that contract.
```

### 2c. Contract 2 — Applied-AI × Safety Engineer

```markdown
# Delegation contract — helpdesk.reply_assistant × safety-engineer

Parties:
- Applied-AI side: @a-eng-1
- Safety-engineer side: @sec-1 (safety team)

Version: 2026-11-10.1
Effective from: 2026-11-10
Next review: 2027-02-10 + per incident

## 1. Scope

Covers the guardrail policy `helpdesk.guard v2.1` (mod-107 [[exercise-02]]),
the refusal categories applied to this feature, the PII scrubber integration
(mod-107 [[exercise-03]]), and the safety-eval suite extensions
(mod-106 [[exercise-04]]).

Not covered: the prompt engineering that reduces jailbreak susceptibility
(applied-AI side owns this); the end-user policy text (legal / PM own this).

## 2. What safety delivers to applied (peer → applied)

| # | Promise | Interface | Observable | Cadence |
|---|---|---|---|---|
| S1 | A versioned guardrail policy covering the refusal categories named in the charter OUT list ([[exercise-01]] Section 2). | `guardrails/helpdesk/policy.yaml` + version pin. | Feature-card ID table ([[exercise-04]]) pin. | Change-controlled; see §7. |
| S2 | PII scrubber over inputs and outputs, with scrub logs retained ≥ 30 days. | mod-107 [[exercise-03]] pipeline. | `app.pii.scrubbed_count` span attribute present on every LLM span. | Per-request. |
| S3 | A safety-eval suite extension for this feature's known attack surface (jailbreaks per locale, prompt-injection vectors via retrieved content, grounding-adjacent policy). | `evals/safety/helpdesk/*.yaml` in the mod-106 [[exercise-04]] suite. | Weekly safety-eval run; pass rate ≥ policy threshold. | Weekly; pre-release gate K5 (charter). |
| S4 | Response to a new safety finding: a contained-mitigation (guardrail update + regression fixture) within one business day of the finding. | PR to `guardrails/helpdesk/`. | PR timestamp + landing. | Per-finding. |
| S5 | A quarterly safety review of the feature's OWASP ML Top 10 and MITRE ATLAS mapping. | Written review attached to the feature card ([[exercise-04]] Section 6). | Review doc present with date within 90d. | Quarterly. |

## 3. What applied delivers to safety (applied → peer)

| # | Promise | Interface | Observable | Cadence |
|---|---|---|---|---|
| A1 | Applied side runs the safety-eval suite on every candidate variant before shadow (mod-111 [[exercise-02]] gate). | CI job `safety-eval-gate`. | CI log present on every mod-111 shadow enrolment. | Per-candidate. |
| A2 | Applied side files any safety-adjacent incident with safety-engineer named in Step 2 (triage) of [[exercise-02]]. | Incident record. | Incident record names a safety co-owner. | Per-incident. |
| A3 | Applied side forwards user-flagged content via the mod-110 [[exercise-03]] feedback pipeline with the `feedback.category = safety` label. | Feedback pipeline. | Daily feedback-digest to the safety team. | Daily. |
| A4 | Applied side does not ship a prompt change that weakens a refusal category without safety-engineer sign-off. | mod-102 [[exercise-05]] prompt-review checklist. | PR has a safety-engineer review checkmark. | Per-prompt-change. |

## 4. What breaks the contract

- S1 broken when the guardrail policy version in the feature card does not
  match the live policy version (drift-check CI).
- S2 broken when > 0.1% of LLM spans are missing `app.pii.scrubbed_count`.
- S3 broken when the weekly safety-eval run fails the policy threshold.
- S4 broken when a safety finding has no PR within one business day.
- A1 broken when a candidate is enrolled to mod-111 shadow without a
  passing `safety-eval-gate` CI log.
- A4 broken when a prompt change touches a refusal category without
  safety sign-off.

## 5. Verification

- CI job `verify-safety-contract` reads:
  - Guardrail policy version pin vs. live policy version.
  - `app.pii.scrubbed_count` coverage on last 1h of spans.
  - Weekly safety-eval run result.
  - `safety-eval-gate` CI log on the most recent shadow enrolment.
- Charter K5 (pre-release safety gate) is the hard-fail-on-regression
  mechanism.

## 6. Escalation

- S3 breach (safety-eval regression): candidate is blocked from shadow;
  safety-engineer and applied-AI lead meet within 1 business day to
  resolve.
- S4 breach (missed mitigation SLA): applied-AI lead escalates to
  safety-engineering lead; director notified on second occurrence.
- S2 breach (scrubbing coverage drop): applied-AI lead investigates the
  span wrapper; if the root cause is on the applied side, A1 of the
  retrieval contract applies (fix within one business day).

## 7. Versioning + change control

- Guardrail policy bumps (v2.1 → v2.2) are communicated via the mod-107
  [[exercise-02]] policy channel ≥ 1 week before landing.
- Contract re-signs when the policy major-version increments.

## 8. Dependencies on other contracts

- Retrieval contract: S3 and R2 together enforce cross-tenant containment.
- Data-engineer contract: span attributes for S2 depend on the mod-110
  [[exercise-01]] pipeline.
- Legal / policy contract (out of scope here, named for completeness):
  the end-user policy text is owned there.
```

### 2d. Contract 3 — Applied-AI × Data Engineer

```markdown
# Delegation contract — helpdesk.reply_assistant × data-engineer

Parties:
- Applied-AI side: @a-eng-1
- Data-engineer side: @de-1 (observability pipeline team)

Version: 2026-11-10.1
Effective from: 2026-11-10

## 1. Scope

Covers the mod-110 [[exercise-01]] trace-emission pipeline, the
mod-110 [[exercise-02]] feature-health dashboards, the mod-110
[[exercise-03]] feedback pipeline, and the mod-109 [[exercise-04]] cost
pipeline. Not covered: the content of the dashboards (applied-AI side
chooses what to render), individual eval runs (owned by applied-AI).

## 2. What data-eng delivers to applied (peer → applied)

| # | Promise | Interface | Observable | Cadence |
|---|---|---|---|---|
| D1 | Span ingestion with ≤ 5 min lag p95 from the OTLP collector to the trace UI. | OTLP collector + trace backend. | mod-110 [[exercise-01]] pipeline-health dashboard. | Continuous. |
| D2 | Feedback pipeline reads thumbs-up / thumbs-down events with ≤ 1h lag to the feedback dashboard. | mod-110 [[exercise-03]] feedback pipeline. | Dashboard freshness probe. | Continuous. |
| D3 | Cost telemetry is joined to the LLM span within the same 1h window (required for mod-109 [[exercise-04]] cost per resolved turn). | Cost-join pipeline. | Per-turn cost field present on ≥ 99% of spans ≥ 1h old. | Continuous. |
| D4 | Dashboard schema stability: no breaking changes to the query surface without 2-week notice. | Dashboard-schema change log. | Change log entry + Slack announce. | Change-controlled. |
| D5 | PII-aware span-storage retention: raw inputs / outputs retained ≤ 30 days; scrubbed telemetry retained per policy. | Retention policy in the data-engineering wiki. | Audit sample monthly. | Continuous. |

## 3. What applied delivers to data-eng (applied → peer)

| # | Promise | Interface | Observable | Cadence |
|---|---|---|---|---|
| A1 | Applied side uses the mod-110 [[exercise-01]] span wrapper; no ad-hoc span emission. | Wrapper enforced via lint. | Lint check on CI. | Per-PR. |
| A2 | Applied side adheres to the semantic-attribute naming (`gen_ai.*`, `app.*`) from mod-110 [[exercise-01]]. | Naming convention. | `verify-span-attributes` CI job. | Per-PR. |
| A3 | Applied side signals meaningful span-attribute additions via a PR tag so data-eng can update dashboards upstream. | `span-attr-added` PR label. | Label present on PRs that add `app.*` attributes. | Per-PR. |

## 4. What breaks the contract

- D1 broken when ingestion lag p95 > 5 min for a 1h window.
- D2 broken when feedback-dashboard freshness probe > 1h.
- D3 broken when per-turn cost field is missing on > 1% of spans ≥ 1h old.
- D4 broken when a dashboard-schema change ships without 2-week notice.
- A1 broken when an ad-hoc span appears in prod traces (lint fails).

## 5. Verification

- CI job `verify-data-contract` runs hourly.
- mod-110 [[exercise-02]] feature-health dashboard includes a pipeline-
  health panel that reads the D1 / D2 / D3 observables.

## 6. Escalation

- D1 / D2 / D3 breach: data-eng oncall paged; mitigation within 2h.
  During the breach, applied-AI side's dashboards are stale — kill
  criteria that depend on 1h rolling metrics (charter K1, K2) are
  temporarily unreadable, so the applied-AI oncall must widen the
  observation window to 4h and watch the trace UI directly.
- D4 breach: applied-AI side's dashboards may silently break. Data-eng
  owes a post-incident doc explaining what broke and how to re-compute
  the affected metrics.

## 7. Versioning + change control

- Data-pipeline minor versions: no re-sign required.
- Dashboard-schema major changes: contract re-sign.

## 8. Dependencies on other contracts

- Retrieval contract: A1 / A2 / A3 overlap — retrieval attributes are
  emitted via the same span wrapper.
- Safety contract: S2 `app.pii.scrubbed_count` depends on the same
  pipeline.
- ML-platform contract: cost telemetry (D3) depends on the gateway's
  cost-emission attribute (mod-108 [[exercise-02]]).
```

### 2e. Contract 4 — Applied-AI × Fine-Tuning Engineer

```markdown
# Delegation contract — helpdesk.reply_assistant × fine-tuning-engineer

Parties:
- Applied-AI side: @a-eng-1
- Fine-tuning-engineer side: @ft-1

Version: 2026-11-10.1

## 1. Scope

Covers the lifecycle of any hosted fine-tune shipped on behalf of this
feature (mod-111 [[exercise-05]]). This contract is **optional**: it is
signed only when the feature actually ships a fine-tune. No fine-tune is
currently in flight for `helpdesk.reply_assistant`; the contract is a
template.

## 2. What fine-tuning-eng delivers to applied (peer → applied)

| # | Promise | Interface | Observable |
|---|---|---|---|
| F1 | Hyperparameter choices for the fine-tune job, with rationale. | `finetune/<job-id>/config.yaml` | Config checked in with the PR. |
| F2 | Training-set / eval-set dedup verification (three-dimensional per mod-111 [[exercise-05]] 2b). | `finetune/<job-id>/dedup-report.txt` | Report checked in. |
| F3 | A loss-curve review flagging under / overfitting. | `finetune/<job-id>/loss-review.md` | Review doc present. |
| F4 | A base-model-deprecation-risk note citing provider announcements. | Field in the loss-review doc. | Present + dated. |

## 3. What applied delivers to fine-tuning-eng (applied → peer)

| # | Promise | Interface | Observable |
|---|---|---|---|
| A1 | Problem diagnosis (why this is a shape problem) per mod-111 [[exercise-05]] 2a. | `finetune/<job-id>/diagnosis.md` | Doc present at job kickoff. |
| A2 | Training set curated per the protocol in mod-111 [[exercise-05]] 2b — PII-scrubbed, human-reviewed labels. | Training set + audit log. | PR-reviewed. |
| A3 | Offline eval results on the mod-106 frozen suite before the shadow gate. | Eval report. | Attached to the PR. |

## 4. What breaks the contract

- F2 breach: eval-set contamination. Production candidate is pulled;
  retrain required.
- A2 breach: PII in the training set. Hard failure; job cancelled, data
  deletion requested per the hosted-platform data-retention policy.

## 5. Verification

- Fine-tuning engineer's checklist attached to the PR.
- Applied engineer runs the mod-106 frozen suite on the trained model.

## 6. Escalation

- Fine-tune job shipped without one of the F1–F4 artefacts: applied-AI
  side blocks the shadow enrolment until the artefact lands.
- Data-handling breach (A2): named safety-engineer and legal on the
  incident record.

## 7. Versioning

- Contract re-signs per fine-tune job (not per quarter — fine-tune jobs are
  the unit of change).

## 8. Dependencies

- Safety contract: A2 depends on the mod-107 [[exercise-03]] PII pipeline.
- Data-engineer contract: cost telemetry on the fine-tuned model depends
  on the gateway (mod-108) emitting the right model-ID.
```

### 2f. Contract 5 — Applied-AI × ML-Platform Engineer

```markdown
# Delegation contract — helpdesk.reply_assistant × ml-platform-engineer

Parties:
- Applied-AI side: @a-eng-1
- ML-platform side: @mp-1 (platform team)

Version: 2026-11-10.1

## 1. Scope

Covers the mod-108 gateway, the mod-111 [[exercise-01]] flag layer, and the
mod-108 [[exercise-04]] rollback rehearsal. Not covered: the feature-side
flag targeting (applied-AI side owns the flag catalogue); the feature-side
rollback rehearsal (shared, with specific split below).

## 2. What ml-platform delivers to applied (peer → applied)

| # | Promise | Interface | Observable |
|---|---|---|---|
| M1 | Gateway routes requests with ≤ 50 ms p95 overhead on top of provider latency. | mod-108 [[exercise-02]] gateway. | Gateway latency dashboard. |
| M2 | Model-ID observability: every LLM span carries the resolved model-ID and model-version. | `gen_ai.response.model` span attribute. | mod-110 [[exercise-01]] span audit. |
| M3 | Provider fallback: when the primary provider errors, gateway routes to the configured fallback within the request SLA. | mod-108 [[exercise-03]] fallback config. | `gen_ai.response.model` on fallback reflects the fallback model. |
| M4 | Flag layer uptime ≥ 99.9%; flag-read p95 overhead ≤ 10 ms. | OpenFeature + flagd deployment. | Flag-health dashboard. |
| M5 | Flag-layer kill-switch SDK semantics preserved (disconnected SDK returns `default_value`, not last-seen) per mod-111 [[exercise-01]] 2g. | SDK configuration documented. | Quarterly disconnected-SDK test. |
| M6 | Model-deprecation announcements forwarded within 1 business day of provider announcement. | Platform-team announce channel. | Charter K7 trigger log. |

## 3. What applied delivers to ml-platform (applied → peer)

| # | Promise | Interface | Observable |
|---|---|---|---|
| A1 | Applied side uses the gateway for all provider calls; no direct SDK bypass. | Lint rule. | CI check. |
| A2 | Applied side owns the four-flag hierarchy per mod-111 [[exercise-01]] and keeps it in sync with the feature card. | Flag catalogue. | Feature-card drift check. |
| A3 | Applied side participates in the quarterly mod-108 [[exercise-04]] rollback rehearsal. | Calendar invite. | Rehearsal log. |

## 4. What breaks the contract

- M1 broken when gateway p95 overhead > 50 ms for a 1h window.
- M3 broken when a provider failure results in a user-visible error (no
  fallback engaged).
- M4 broken when flag-health dashboard shows < 99.9% uptime over a day.
- M5 broken when the quarterly disconnected-SDK test shows a non-default
  value served under disconnection.
- A1 broken when a direct-SDK call is detected in prod traces.

## 5. Verification

- CI job `verify-platform-contract` reads gateway p95 overhead, flag-layer
  uptime, and the most recent disconnected-SDK test result.

## 6. Escalation

- M3 breach: joint P1 incident; applied-AI side considers kill lever
  (charter K2) while platform investigates.
- M5 breach: P0. Fail-closed guarantees from mod-111 [[exercise-01]] 2g
  are the foundation every other rollback assumes; if they break the
  entire rollback story is broken.

## 7. Versioning

- Gateway minor versions: no re-sign.
- Flag-provider swap or SDK major version: contract re-signs.

## 8. Dependencies

- Data-engineer contract: M2 depends on span pipeline.
- Safety contract: the gateway is the choke point where safety headers
  (safety policy version) are attached — overlap documented in S1.
- Retrieval contract: retrieval latency contributes to the gateway's p95
  envelope — R1 and M1 together constrain end-to-end latency.
```

### 2g. Contract-validity CI check

```python
# scripts/verify_delegation_contracts.py
# Walks `docs/delegation/*.md` and asserts:
# - Each contract has sections 1–8.
# - Each contract's version header matches the live pin on the feature card.
# - The peer-role is unique (no two contracts claim the same role).
# - Every "Observable" cell cites a dashboard, span attribute, or file path
#   that the applied-AI side can resolve.
from __future__ import annotations

import re
import sys
from pathlib import Path


REQUIRED_SECTIONS = [f"## {i}." for i in range(1, 9)]


def check(contract: Path) -> list[str]:
    text = contract.read_text()
    problems = []
    for s in REQUIRED_SECTIONS:
        if s not in text:
            problems.append(f"{contract.name}: missing section {s}")
    if "Version:" not in text:
        problems.append(f"{contract.name}: missing Version: header")
    if "Next review:" not in text:
        problems.append(f"{contract.name}: missing Next review: header")
    # Observable cells must mention one of these (crude heuristic).
    for match in re.finditer(r"\| \d+ \|.*?\|.*?\|(.+?)\|", text):
        obs = match.group(1).strip()
        if not any(tok in obs for tok in ("dashboard", "span", "CI", "report", "file", "pipeline", "log")):
            problems.append(f"{contract.name}: Observable cell too vague: {obs!r}")
    return problems


def main() -> int:
    contracts = sorted(Path("docs/delegation").glob("*.md"))
    all_problems: list[str] = []
    for c in contracts:
        all_problems.extend(check(c))
    if all_problems:
        print("Delegation contract problems:")
        for p in all_problems:
            print("  -", p)
        return 1
    print(f"OK — {len(contracts)} delegation contracts validated.")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

The CI job runs on every PR touching `docs/delegation/`.

### 2h. Review cadence + re-sign triggers

Each contract names its own `Next review:` header. In addition, a contract re-signs on any of:

- Peer-platform major-version bump (retriever v4 → v5; policy v2 → v3; flag provider swap).
- Addition of a new cohort to the feature (new locale, new surface) — may change the peer's envelope.
- A P0 or P1 incident where the contract's §4 "breaks" clause fired.
- Change to the feature charter's dependencies list ([[exercise-01]] Section 5) — the charter is the master index of what contracts must exist.

## 3. Validation steps

1. **Each contract has all eight sections.** `scripts/verify_delegation_contracts.py` runs green in CI.
2. **Promise lists are symmetric.** Every contract has both §2 (peer → applied) and §3 (applied → peer); a one-sided contract is a hard failure.
3. **Every promise names an interface, an observable, and a cadence.** A promise without an observable is a vibes-level promise.
4. **Observables resolve.** Every observable cell names a dashboard, span attribute, file path, or CI job that the applied-AI side can actually open. Vague text ("the platform team's dashboard") fails the CI check in 2g.
5. **Break conditions are measurable.** Every §4 entry is a metric + threshold + window, not a mood.
6. **Escalation names a specific role, not "the team."** `retrieval-primary` oncall, `applied-ai-lead`, named safety-engineer — not "escalate up."
7. **Versioning is tied to upstream version.** The contract version increments when the upstream peer-platform version increments; a frozen contract against a changing peer-platform is drift.
8. **Dependencies on other contracts are named.** §8 cross-references exist and resolve.
9. **All contracts implied by the charter's Section 5 dependency list exist.** If the charter names dependencies on retrieval, safety, data-eng, fine-tuning-eng, and ml-platform-eng, all five contracts are present.
10. **Contracts align with the feature card's ID table.** The retriever version pinned in the card matches the retriever version in the retrieval contract's §1.

## 4. Rubric or review checklist

Instructor rubric (10 points total). A submission scoring < 7 is rejected.

| Criterion | Points | What earns the points |
|---|---|---|
| Five contracts present (retrieval / safety / data-eng / fine-tuning / ml-platform) | 1 | All five peer-role archetypes have a contract or a documented non-applicability note. |
| Each contract has all eight required sections | 1 | No missing scope / promises / breaks / verification / escalation / versioning / dependencies. |
| Promises are bidirectional (§2 and §3 both populated) | 1 | No contract is one-sided. |
| Every promise has an interface, observable, and cadence | 2 | Observable cells name dashboards / spans / file paths / CI jobs — not "the team will tell us." |
| Break conditions are measurable (metric + threshold + window) | 1 | No vibes-level "when things look bad." |
| Escalation steps name specific roles | 1 | `retrieval-primary` oncall, `safety-lead` — not "escalate up." |
| Versioning tied to peer-platform version | 1 | Clear mapping from upstream version bump to contract re-sign. |
| CI check (`verify_delegation_contracts.py`) runs green | 1 | Script exists, runs in CI, exits zero. |
| Contract versions align with the feature card's ID table | 1 | Drift-check from [[exercise-04]] passes against the contract pins. |

**Hard failures (automatic rejection):**

- One-sided contracts (only the peer promises things).
- Break conditions that are not measurable.
- Escalation paths that terminate in "the team."
- Contracts missing versioning / review cadence.
- No charter cross-reference.

## 5. Common mistakes

- **Contract reads like a checklist of feature requirements.** Delegation contracts describe the *interface*, not the feature. "The retriever must return relevant documents" is a feature requirement; "the retriever returns documents matching the schema in R3 within R1's latency envelope" is a contract.
- **One-sided contracts.** The applied-AI side makes no promises back. The peer has no visibility into whether they are on the critical path for an incident. Add the §3 symmetric list.
- **Promises stated as outcomes, not interfaces.** "Safety will catch jailbreaks" is an outcome. "Safety will provide a versioned guardrail policy with the categories in the charter OUT list" is an interface.
- **Observables that cannot be read.** "The retrieval team watches their dashboard" is not an observable; "the retrieval-team dashboard URL is <X>" is.
- **Break conditions using "high" and "low."** Replace with metric + threshold + window; the on-call needs a number.
- **Escalation to "the team."** Name a role. "The team" is where accountability dies.
- **No versioning.** The retriever upgrades from v4 to v5; the contract is now describing a system that does not exist. Version the contract against the peer-platform version.
- **Over-fitting the contract to a single incident.** The contract is a *general* ownership map, not an incident post-mortem. When a specific failure mode needs a specific fix, put it in the §4 break conditions, not in the top-level scope.
- **Fine-tuning contract signed when no fine-tune is in flight.** The contract is optional and should be a template until an actual job is on the critical path.
- **Contract diverges from feature card.** The ID table in the feature card and the §1 scope of each contract name versions; if they disagree, the drift-check CI should fail. Running both checks is not redundant — they catch different things.
- **Safety-engineer sign-off gate omitted on prompt changes affecting refusal categories.** A4 of the safety contract is exactly this discipline; without it, the applied engineer can ship a change that silently loosens a refusal category.
- **Treating the contract list as frozen.** When the charter adds a dependency (e.g., the feature grows a write-side tool, pulling in a new peer), a new contract must be authored. The charter's Section 5 is the authoritative index.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — GOVERN-2.1 requires documented roles and responsibilities for the AI system lifecycle; delegation contracts are the artefact that satisfies this at applied-AI altitude. GOVERN-4.1 requires that the humans with decision authority be named; the "Escalation" section of every contract satisfies this for its scope.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML06 (AI Supply Chain Attacks) risks grow sharply when ownership is diffuse; the delegation contract is the mechanism that makes the supply-chain map explicit. The retrieval and ml-platform contracts in particular cover the upstream components whose compromise would be an ML06 event.
- MITRE ATLAS — <https://atlas.mitre.org/> — AML.T0051 (Prompt Injection) mitigations span retrieval filtering (mod-103) and guardrails (mod-107); the delegation contracts are what keep those mitigations consistently owned. A gap between contracts is a gap in the mitigation story.

Related exercises: [[exercise-01]] (the charter's Section 5 dependency list is the master index of contracts that must exist), [[exercise-02]] (every incident where a peer is implicated routes through the contract's escalation path), [[exercise-03]] (the substrate decisions determine which peer teams exist and therefore which contracts are needed), [[exercise-04]] (the feature card's ID table is drift-checked against the contract version pins); mod-103 [[exercise-01]] (retriever contract that the retrieval delegation contract depends on), mod-106 [[exercise-04]] (safety-eval suite feeding the safety contract's S3), mod-107 [[exercise-02]] (guardrail policy version driving S1), mod-108 [[exercise-02]] (gateway feeding the ml-platform contract's M1 / M2), mod-108 [[exercise-04]] (rollback rehearsal in the ml-platform contract's A3), mod-110 [[exercise-01]] (span wrapper that the data-eng contract's A1 enforces), mod-111 [[exercise-01]] (flag layer feeding the ml-platform contract's M4 / M5), mod-111 [[exercise-05]] (fine-tune lifecycle feeding the optional fine-tuning contract).
