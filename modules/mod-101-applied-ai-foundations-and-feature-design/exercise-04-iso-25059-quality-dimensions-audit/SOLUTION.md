# exercise-04: ISO/IEC 25059 Quality Dimensions Audit — Solution

## 1. Solution overview

The exercise asks the engineer to audit an LLM feature against the quality model for AI systems defined in **ISO/IEC 25059:2023** — part of the SQuaRE family of standards (ISO/IEC 25000 series). ISO/IEC 25059 extends the general software product quality model in **ISO/IEC 25010** with AI-specific considerations.

The audit produces:

- A **coverage matrix** mapping each applicable quality (sub-)characteristic to one of {covered, partial, not covered, N/A} with evidence.
- A **gap list** of any sub-characteristics marked "partial" or "not covered" with an owner and a mitigation.
- A **tie-back** of each in-scope sub-characteristic to a specific SLO from [[exercise-02]], a specific control, or a specific test — audits that don't tie back are wall art.

Because ISO/IEC 25059 is a paid standard and its sub-characteristic list evolves across editions, the solution gives the **audit skeleton and method** that works regardless of the exact extension list, and explicitly requires the learner to confirm the current sub-characteristic list against the official standard. Any claim about a specific ISO/IEC 25059 sub-characteristic that cannot be verified from the standard itself must be wrapped in `<!-- research-pending: -->`.

## 2. Worked answer or implementation

### 2a. Scope and inputs

Before the audit starts, record:

- **Feature under audit**: name + one-line description.
- **Feature shape**: output of [[exercise-01]].
- **SLOs**: output of [[exercise-02]].
- **Provider tier**: output of [[exercise-03]].
- **Standard edition used**: `ISO/IEC 25059:2023` + any later amendments; the exact edition is cited so the audit is reproducible.

### 2b. Audit skeleton — base ISO/IEC 25010 characteristics

The base model from ISO/IEC 25010 (2011 edition, updated 2023) defines eight top-level product-quality characteristics. ISO/IEC 25059 **adds** AI-specific sub-characteristics on top of these, and may elevate a few to new top-level items. The audit goes through each 25010 top-level item first and then through the 25059 additions; if a characteristic does not apply to this feature, mark it N/A with a one-line reason.

| 25010 top-level characteristic | Audit questions for an LLM feature | Where evidence usually lives |
|---|---|---|
| Functional suitability | Does the feature do what the spec says on realistic inputs? Precision / recall / task pass rate? | Offline eval set + quality SLO from [[exercise-02]]. |
| Performance efficiency | Latency, throughput, resource use under expected load. | Latency SLO + load tests. |
| Compatibility | Does it coexist with the surrounding product (session state, other features, provider APIs)? | Integration tests, API contracts. |
| Interaction capability (formerly Usability) | Can users accomplish the task with few errors? Learnability, accessibility. | UX research, task-completion metrics. |
| Reliability | Behaviour under partial failure: provider outage, timeout, malformed response. | Fallback path (from [[exercise-03]]), circuit breakers, error-budget policy. |
| Security | Confidentiality / integrity / authenticity, prompt-injection surface, abuse controls. | Threat model (OWASP ML Top 10 + MITRE ATLAS mapping); safety SLOs from [[exercise-02]]. |
| Maintainability | Modularity, reusability, testability of prompts, tools, and routing. | Code review, prompt-versioning scheme. |
| Portability | Can the feature move to another provider / tier? | Fallback tier from [[exercise-03]]; abstracted provider interface. |

### 2c. Audit skeleton — ISO/IEC 25059 AI additions

ISO/IEC 25059 extends 25010 with sub-characteristics that address concerns specific to AI systems (adaptability of behaviour, transparency to users and auditors, intervenability, etc.). **The exact set of sub-characteristics and their names must be read from ISO/IEC 25059:2023 directly**; the following is the audit scaffold, not an authoritative restatement:

| AI-specific concern | Audit questions | Where evidence usually lives |
|---|---|---|
| Functional adaptability / adaptation behaviour | Does the feature's behaviour change with new data, retraining, or context, and is that intended? How is drift detected? | Online quality SLO; drift monitors; retraining policy. |
| User controllability | Can the user override, correct, or stop the feature? For an agentic shape, can they interrupt? | UI design of confirm steps, cancel controls, edit affordances. |
| Transparency | Do users and operators know this is AI-generated, what it used as input, and how confident it is? | UX disclosure, logprobs / confidence surface, citations if RAG. |
| Robustness | Behaviour under distribution shift and adversarial input (prompt injection, out-of-distribution queries). | Red-team suite, OWASP ML Top 10 + MITRE ATLAS mitigations. |
| Intervenability | Can a human reviewer or auditor step into the loop at defined points? | Reviewer tooling; trace retention policy. |
| Information quality of inputs/outputs | Do we know the source, recency, and licensing of training/retrieval data? | Data catalogue; retrieval index freshness metric. |
| Societal and ethical considerations | Is there a documented analysis of fairness, inclusion, and impact? | AI RMF `MAP` function outputs; impact assessment. |

> <!-- research-pending: confirm the exact sub-characteristic names and definitions used in ISO/IEC 25059:2023; names above are the audit scaffold and may not match the standard's wording. -->

### 2d. Coverage matrix template (what the learner submits)

```
| Characteristic | Sub-characteristic | Applies? | Status  | Evidence (link / ID)        | Tie-back (SLO / control / test) | Owner | Mitigation if gap |
|----------------|--------------------|----------|---------|------------------------------|---------------------------------|-------|-------------------|
| Reliability    | Fault tolerance    | Yes      | covered | runbook-44, chaos-test-7     | Latency SLO + fallback tier     | SRE   | —                 |
| Security       | Prompt-injection resistance | Yes | partial | threat-model-v2 line 31 | Safety SLO (flag rate)          | AppSec| add input filter (Q2) |
| ...            | ...                | ...      | ...     | ...                          | ...                             | ...   | ...               |
```

`Status` is one of {covered, partial, not covered, N/A}. Every row with `partial` or `not covered` must have an `Owner` and `Mitigation`. `N/A` must have a reason in the Evidence column (e.g., "feature is read-only, no write-side intervenability needed").

### 2e. Worked example — "Inbox Assistant" (partial table)

Context: feature shape = single-turn structured output (from [[exercise-01]]); primary tier T1 + fallback T2 (from [[exercise-03]]); SLOs set in [[exercise-02]].

| Characteristic | Sub-characteristic | Applies? | Status | Evidence | Tie-back | Owner | Mitigation |
|---|---|---|---|---|---|---|---|
| Functional suitability | Functional correctness | Yes | covered | Offline eval `inbox-golden-v3`, 200 examples, pass ≥ 80% | Quality SLO in [[exercise-02]] | Feature eng | — |
| Performance efficiency | Time behaviour | Yes | covered | Load test LT-12, client-observed p95 | Latency SLO ≤ 3 s | SRE | — |
| Reliability | Fault tolerance | Yes | covered | Fallback to T2 on T1 5xx | Error-budget policy ([[exercise-02]]) | SRE | — |
| Security | Data integrity | Yes | covered | Email read via scoped OAuth, no write scope | Hard invariant: 0 autonomous sends | AppSec | — |
| Security | Prompt-injection resistance | Yes | partial | Content-based filter on draft output; no structural defence on email body | Flag-rate SLO, OWASP ML Top 10 (LLM01 injection) | AppSec | Add input-side canonicalization; ATLAS-mapped red-team suite — target Q1 2027 |
| ISO 25059 extension: Transparency | User-facing AI disclosure | Yes | covered | "Drafted by AI — review before sending" banner on every draft | Interaction capability audit | Design | — |
| ISO 25059 extension: User controllability | Override / stop | Yes | covered | Per-draft edit + reject; feature-level toggle in settings | Spec clause: "do not send on its own" | Feature eng | — |
| ISO 25059 extension: Robustness | Out-of-distribution handling | Yes | partial | No explicit OOD detector; relies on general safety filter | MITRE ATLAS (abuse categories) | AppSec | Add OOD monitor on input length and language; target Q2 2027 |
| ISO 25059 extension: Intervenability | Audit review of flagged sessions | Yes | partial | Flagged sessions stored 30 days; no scheduled review cadence | Safety SLO burn action | Ops | Add weekly audit sample of 50 flagged sessions |
| Portability | Replaceability | Yes | covered | T2 fallback uses same request shape | Provider decision in [[exercise-03]] | Feature eng | — |
| Maintainability | Modifiability | Yes | covered | Prompt versioned; rollback path documented | Release runbook | Feature eng | — |
| ISO 25059 extension: Information quality | Training / retrieval data provenance | N/A | — | Feature does not use a custom retrieval index; uses in-context email body only | — | — | — |

> <!-- research-pending: validate that the audit column headings `Functional correctness`, `Time behaviour`, `Modifiability`, `Replaceability`, `Fault tolerance`, `Data integrity` match ISO/IEC 25010's current sub-characteristic names in the edition cited by the learner. -->

## 3. Validation steps

A submission is valid when:

1. The audited feature's shape, SLOs, and provider tier from [[exercise-01]]–[[exercise-03]] are referenced; the audit is not written in a vacuum.
2. Every top-level characteristic from ISO/IEC 25010 appears in the matrix or is marked N/A with a reason.
3. Every ISO/IEC 25059 AI-specific sub-characteristic that the learner has verified from the standard appears in the matrix, and any scaffold item they cannot verify is flagged `<!-- research-pending: -->`.
4. No row has status `covered` without a concrete evidence link (test ID, runbook, SLO, doc link).
5. Every `partial` or `not covered` row has an owner **and** a dated mitigation.
6. Security rows cross-reference OWASP ML Top 10 categories and, where relevant, MITRE ATLAS techniques.
7. At least one row ties to each SLO from [[exercise-02]] — i.e., every SLO is traceable to a quality characteristic, not just dangling.
8. The standard edition is cited (e.g., `ISO/IEC 25059:2023`).

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Inputs referenced | 1 | Shape + SLOs + provider tier from [[exercise-01]]–[[exercise-03]] are called out. |
| 25010 coverage | 2 | All eight top-level characteristics present or N/A with reason. |
| 25059 extensions | 2 | AI-specific sub-characteristics appear and are either verified (cited) or `<!-- research-pending: -->`. |
| Evidence specificity | 2 | Every `covered` row has a concrete artifact link / ID. |
| Gap ownership | 1 | Every `partial` / `not covered` row has an owner and a dated mitigation. |
| Safety cross-references | 1 | Security rows name OWASP ML Top 10 categories; adversarial rows name MITRE ATLAS techniques where relevant. |
| SLO tie-back | 1 | Every SLO from [[exercise-02]] is covered by at least one audit row. |

Scoring < 6 → back.

## 5. Common mistakes

- **Citing ISO/IEC 25059 sub-characteristics from memory.** Standards evolve across editions; wording matters. Use the actual document or flag `<!-- research-pending: -->`.
- **Skipping 25010 to jump to the AI-specific additions.** 25059 extends 25010 — the base characteristics still apply.
- **"Covered" rows with no evidence link.** That isn't a covered row; that is an assertion.
- **Security row that is not mapped to OWASP ML Top 10 or MITRE ATLAS.** The audit loses its cross-referenceability.
- **Gap rows without an owner.** An unowned gap is permanent.
- **Collapsing transparency into "we log it."** Transparency in 25059 is user-facing, not operator-facing; the audit must distinguish them.
- **Marking something N/A to avoid work.** N/A requires a defensible reason; otherwise it is a failed audit, not a passed one.

## 6. References

- ISO/IEC 25059:2023 — Software engineering — Systems and software Quality Requirements and Evaluation (SQuaRE) — Quality model for AI systems. The authoritative source for the AI-specific sub-characteristic list; purchase/access via the ISO catalogue. Learners must cite the specific edition they used.
- ISO/IEC 25010 — the base software product quality model that 25059 extends.
- NIST AI Risk Management Framework (AI RMF 1.0) — <https://www.nist.gov/itl/ai-risk-management-framework> — maps naturally onto the audit's governance, mapping, measurement, and management rows (GOVERN / MAP / MEASURE / MANAGE).
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — supplies the risk vocabulary for Security rows.
- MITRE ATLAS — <https://atlas.mitre.org/> — adversary techniques matrix; use for Robustness and Security rows involving adversarial input.

Related exercises: [[exercise-01]] (shape dictates which characteristics are in scope), [[exercise-02]] (SLOs must trace back into the audit), [[exercise-03]] (tier decision backs Portability / Replaceability rows).
