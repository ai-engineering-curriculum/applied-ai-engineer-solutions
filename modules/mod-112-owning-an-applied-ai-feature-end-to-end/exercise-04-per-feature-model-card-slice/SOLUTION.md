### exercise-04: Per-Feature Feature-Card Slice — Solution

## 1. Solution overview

Model cards, as originally proposed, describe a *model*. At applied-AI altitude the right unit is a **feature** — the surface, prompt, retrieval index, guardrails, and policies that compose a specific user-facing behaviour on top of a model. This exercise asks the applied-AI engineer to produce a **feature-card slice** that documents one of their features for three audiences: the on-call engineer (needs the dashboard links and kill lever), the auditor (needs the governance mapping), and the next applied-AI engineer who inherits the feature (needs the operational context that the charter summarises).

The feature card is deliberately narrower than the [[exercise-01]] charter and deliberately broader than a data-sheet. The charter answers "should this feature exist?" The feature card answers "what *is* this feature, right now, and how do I operate it?"

The failure modes this solution defends against:

1. **Feature card drifted from reality.** The card describes a prompt version that shipped six months ago; the current prompt is three variants later. The fix is tying the card to *IDs that are machine-checkable* — prompt ID, model ID, retriever version — so a CI check can fail the build when the card and the code disagree.
2. **Feature card = marketing page.** The card reads like a product spec. The on-call cannot find the dashboard, the auditor cannot find the governance mapping, the next engineer cannot find the known limitations.
3. **Feature card omits known limitations.** Users and auditors need to see what the feature *does not do well*. Hiding limitations behind "we're working on it" is a pattern that erodes trust.
4. **Feature card references no upstream module artefact.** The card lists metrics, but does not say which dashboard renders them. The on-call has to go hunting.
5. **Feature card does not map to NIST AI RMF.** The regulator asks "how do you govern this?" and the card has no answer; the applied engineer has to re-assemble the mapping ad hoc.
6. **OWASP / ATLAS mapping is a check-the-box line.** "We considered OWASP" is not the mapping; a mapping names the technique and the mitigation artefact (mod-107 guardrail / mod-103 retriever filter / mod-111 kill lever).
7. **Feature card is not reviewed.** The card exists but nobody owns updates. The fix is a review cadence tied to meaningful change (same pattern as the [[exercise-01]] charter re-ratify triggers).

This solution provides:

- A **feature-card template** with required sections.
- A **worked card** for the reference feature (`helpdesk.reply_assistant`) at a specific point-in-time, tied to upstream module artefacts.
- A **drift-detection check** — a CI script that fails the build when the card asserts an ID that does not match the live code / registry.
- A **decision rationale** explaining why each section exists.
- A **grading rubric** scoring the quality of the card.
- A **common-mistakes list** covering the typical gaps.

## 2. Worked answer or implementation

### 2a. The feature-card template

A feature card is a single markdown file, `docs/features/<feature-id>/feature-card.md`, with these sections in this order:

```markdown
# Feature card — <feature-id>

Owner: <applied-AI engineer or team> (slack: <handle>)
Charter: <link to [[exercise-01]] charter>
Last reviewed: <YYYY-MM-DD>
Next scheduled review: <YYYY-MM-DD>
Automatic drift checks: PASSING | FAILING | STALE

## 1. What this feature does
- Short description (≤ 5 sentences).
- User-facing surface(s): <names>.
- Supported locales: <list>.
- User tiers / cohorts: <list>.

## 2. Intended use (and non-use)
- Intended use: <bullet list of jobs-to-be-done>.
- Out of scope: <bullet list of jobs this feature explicitly does NOT serve>.

## 3. The ID table (the drift-detection surface)
- Current prompt ID and registry version.
- Current model ID (provider + dated model string).
- Retriever version / index generation.
- Guardrail policy version.
- Flag layer version (OpenFeature / flagd / provider).

## 4. Metrics in production
- Primary metric: <name, target, current trailing value, dashboard link>.
- Secondary metric(s).
- Anti-metric(s).
- SLO on latency / cost (mod-109 [[exercise-02]]).

## 5. Known limitations
- What this feature does not do well, with evidence (the eval row or
  incident ID).

## 6. Risk & governance mapping
- NIST AI RMF function alignment (MAP / MEASURE / MANAGE / GOVERN).
- OWASP Machine Learning Security Top 10 risks considered, mitigations
  named by module artefact.
- MITRE ATLAS techniques considered, mitigations named by module
  artefact.

## 7. Operations
- Kill lever (from the charter): <flag key, how to flip, who authorises>.
- Rollback lever (smaller-than-kill): <flag key, how to flip>.
- Oncall rotation + escalation path.
- Dashboards: <trace UI, SLO board, cost board, feedback board>.
- Alerts that page on this feature.

## 8. Change log (summary only; the full log is the git history on this file)
- Last 3 ratified changes with date and one-line rationale.

## 9. Known open questions
- Carry-forward from the charter, updated per review.
```

### 2b. Worked feature card — `helpdesk.reply_assistant`

```markdown
# Feature card — helpdesk.reply_assistant

Owner: Applied-AI team (lead: @a-eng-1; slack: #applied-ai-oncall)
Charter: docs/features/helpdesk.reply_assistant/charter.md ([[exercise-01]])
Last reviewed: 2026-11-10
Next scheduled review: 2026-11-24 (two-weekly during the first quarter after
  initial GA; quarterly afterwards)
Automatic drift checks: PASSING (last CI run: 2026-11-15 14:20 UTC)

## 1. What this feature does

Authenticated users in the billing-support surface can ask natural-language
questions about their own billing records (invoices, payments, refunds,
subscription state). The feature retrieves the user's billing documents from
the mod-103 [[exercise-01]] retriever, composes a 2–4 sentence answer using
the prompt in Section 3, and cites the document(s) used.

- User-facing surfaces: `helpdesk.web`, `helpdesk.mobile-native`.
- Supported locales: `en`, `es`.
- User tiers: authenticated customers only. Internal staff get the same
  surface with an additional "show context" developer panel.

## 2. Intended use (and non-use)

Intended use (mirrors charter Section 2 IN):
- Read-only question answering over the user's own billing records.
- Citing the specific record used to form the answer.
- Saying "I don't have that information" when the answer is not in records.

Not in scope (mirrors charter Section 2 OUT):
- Taking actions on the user's behalf (refunds, cancellations).
- Cross-user questions.
- Legal / tax advice.
- General billing policy — routed to docs-search product.

## 3. ID table — drift-checked

| Field | Value as of this card | Source of truth |
|---|---|---|
| Prompt ID | `support_answer_v15_concise_r2` | `prompts/registry.py:REGISTRY` |
| Prompt template SHA-256 | `<pinned SHA-256>` | computed from registry |
| Model ID | `gpt-4o-2024-08-06` | `flags/flags.json: support_agent.model_id` |
| Retriever version | `billing-retriever v4.2` | `retrievers/billing/VERSION` |
| Index generation | `2026-11-10T00:00Z` | `retrievers/billing/INDEX_GEN` |
| Guardrail policy | `helpdesk.guard v2.1` | `guardrails/helpdesk/VERSION` |
| Flag provider | `flagd v0.11` (OpenFeature 1.7) | `flags/README.md` |

The `scripts/check_feature_card_ids.py` CI job (see 2c) reads this table and
compares it against the sources of truth; disagreement fails the build.

## 4. Metrics in production

| Role | Metric | Dashboard | Target | Trailing-28d |
|---|---|---|---|---|
| Primary | Containment rate | mod-110 [[exercise-02]] dashboard `feature=helpdesk.reply` | ≥ 55% | **63%** (2026-11-15) |
| Secondary | Thumbs-up rate | mod-110 [[exercise-03]] feedback board | ≥ 60% | **67%** |
| Secondary | p95 latency | mod-109 [[exercise-02]] SLO board | ≤ 4.0 s | **3.2 s** |
| Anti-metric | Grounding-violation rate | mod-107 [[exercise-03]] + mod-110 | ≤ 1% | **0.3%** |
| Anti-metric | Human-side correction rate | support-agent tooling + mod-110 | ≤ 5% | **3.8%** |
| Cost | Cost per resolved turn | mod-109 [[exercise-04]] cost board | ≤ budget | within budget |

The values in the "Trailing-28d" column are manually pasted on each card
review; the dashboard is the live source and is linked for the on-call.

## 5. Known limitations

- **Currency conversion refused by design.** As of the fix in
  `INC-2026-11-15-helpdesk-currency` ([[exercise-02]] 2b), the model
  refuses to convert amounts between currencies. Users who expect this
  behaviour will see a refusal and a pointer to the source record. Eval
  fixture: `evals/fixtures/INC-2026-11-15-helpdesk-currency-eur.yaml`.
- **Records older than 24 months not retrieved.** Retrieval index covers
  the last 24 months per the retrieval contract in mod-103 [[exercise-01]].
  The prompt refuses to answer from memory for older records.
- **Non-English locales lag by ~1 week on new prompt variants.** Translation
  review sits in the mod-103 [[exercise-04]] gated-retrieval pipeline;
  prompt updates in `en` roll to `es` after the translation-review pass.
- **Mobile surface has no "show context" dev panel.** The surface-level UI
  parity gap is tracked in the open-questions list.

Each limitation is linked to either an incident fixture (so the regression
pathway is testable) or an evaluation case (so the limitation is monitored
as the feature evolves).

## 6. Risk & governance mapping

### NIST AI RMF alignment

| Function | Sub-function | Where this feature satisfies it |
|---|---|---|
| MAP | MAP-1.2 (purpose and context) | Charter Section 1 + feature card Section 1. |
| MAP | MAP-3.4 (AI vs non-AI alternative) | Charter Section 1 "Why an LLM" argument. |
| MEASURE | MEASURE-2.5 (metrics across operating envelope) | Feature card Section 4 (primary + secondary + anti). |
| MEASURE | MEASURE-2.11 (dashboards for operating envelope) | Feature card Section 7 (dashboard list). |
| MANAGE | MANAGE-2.3 (corrective action) | Feature card Section 7 (kill + rollback lever). |
| MANAGE | MANAGE-4.1 (verification of corrective action) | [[exercise-02]] regression fixtures slotted into mod-106. |
| GOVERN | GOVERN-1.6 (documented decisions) | Charter Section 4 + feature card Section 8 changelog. |
| GOVERN | GOVERN-5.1 (feedback channels) | mod-110 [[exercise-03]] feedback pipeline. |

### OWASP Machine Learning Security Top 10

- **ML01 (Input Manipulation Attack):** present. Mitigation: mod-107
  [[exercise-01]] adversarial-prompt handling; mod-103 [[exercise-04]]
  query-rewriting filter for retrieved content.
- **ML02 (Data Poisoning):** low relevance. Retrieval index is derived from
  internal billing records; no external data ingestion path.
- **ML04 (Membership Inference):** present at the retrieval boundary.
  Mitigation: tenant-scoped retrieval + prompt instructions to not reveal
  cross-user data (mod-103 [[exercise-01]] retriever contract).
- **ML09 (Output Integrity Attack):** present. Mitigation: grounding
  anti-metric + K1 kill criterion from the charter; regression fixture from
  [[exercise-02]].
- **ML06 (AI Supply Chain Attacks):** present via model-provider dependency.
  Mitigation: mod-108 [[exercise-02]] gateway with fallback routing; charter
  K7 re-ratification trigger on provider deprecation.

OWASP items considered and classified not applicable to this feature
(ML05, ML07, ML08, ML10) are listed here with the reason, so the next
reviewer sees the reasoning, not the omission.

### MITRE ATLAS

- **AML.T0051 (LLM Prompt Injection):** addressed at the input layer by
  mod-107 [[exercise-01]] and at the retrieved-context layer by mod-103
  [[exercise-04]]. Trace evidence: `app.prompt.injection_detected` span
  attribute (mod-107 [[exercise-02]]).
- **AML.T0053 (LLM Plugin Compromise):** not applicable. This feature uses
  no write-side tools.
- **AML.T0057 (LLM Jailbreak):** mitigation via system-prompt refusal
  categories (mod-107 [[exercise-02]]); fixtures exist in the mod-106
  safety suite for the common jailbreak patterns (mod-106 [[exercise-04]]).

## 7. Operations

### Kill lever

Flag: `support_agent.enabled` (bool). Flip to `false` in flagd / your flag
provider. Observed TTE: ≤ 5 seconds (mod-111 [[exercise-01]] evidence).
Routes all traffic to the static fallback message.

Authority to flip: the on-call applied-AI engineer; no further approval
needed within the triggers defined by charter K1, K2, K6.

### Rollback lever (smaller-than-kill)

Flag: `support_agent.prompt_id` (string). Flip to the LKG prompt ID
(`support_answer_v14`). Observed TTE: ≤ 5 seconds. Leaves the surface up;
only swaps the prompt. Use this first; use the kill lever only for
feature-level damage.

### Oncall rotation & escalation

- L1 oncall: Applied-AI team weekly rotation (PagerDuty service
  `applied-ai-primary`).
- Escalation on charter K6 (legal / regulator): page Legal within
  15 minutes; page director within 30 minutes.

### Dashboards

- Feature health: mod-110 [[exercise-02]] `feature=helpdesk.reply`.
- Traces: mod-110 [[exercise-01]] trace UI, pre-filtered.
- Feedback: mod-110 [[exercise-03]] feedback dashboard.
- Cost / latency SLO: mod-109 [[exercise-02]] SLO board.
- Shadow / canary status (during rollouts): mod-111 [[exercise-02]] report.

### Alerts that page on this feature

- `grounding_violation_rate_1h > 5%` (charter K1, P1 page).
- `p95_latency_1h > 15s` (charter K2, P1 page).
- `canary_arm_errors > 2%` during an active mod-111 [[exercise-02]]
  rollout (P2 page).

## 8. Change log (summary; full history on this file)

- **2026-11-17** — Shipped `support_answer_v15_concise_r2` (fix for
  `INC-2026-11-15-helpdesk-currency`). Shadow + canary passed; ratified.
- **2026-11-14** — Shipped `support_answer_v15_concise` (brevity variant).
- **2026-10-30** — Enabled `es` locale (new cohort; triggered charter
  re-ratification).

## 9. Known open questions (carried from charter)

- Containment target revisit at next quarterly charter review.
- Locale support for `ja` and `fr` not scoped.
- Mobile surface "show context" developer panel parity.
```

### 2c. Drift-detection CI job (what makes this card self-correcting)

```python
# scripts/check_feature_card_ids.py
# Fails the CI build if the feature card's ID table disagrees with the live
# code / registry.
from __future__ import annotations

import hashlib
import json
import re
import subprocess
import sys
from pathlib import Path

import yaml  # or markdown table parsing — this sketch uses YAML frontmatter
              # which an in-house card tool typically produces.


CARD_PATH = Path("docs/features/helpdesk.reply_assistant/feature-card.md")
REGISTRY_PATH = Path("prompts/registry.py")
FLAGS_PATH = Path("flags/flags.json")
RETRIEVER_VERSION = Path("retrievers/billing/VERSION")
INDEX_GEN = Path("retrievers/billing/INDEX_GEN")
GUARDRAIL_VERSION = Path("guardrails/helpdesk/VERSION")


def parse_id_table(card: str) -> dict:
    # Minimal parser: pulls the ID table rows under Section 3.
    lines = card.splitlines()
    in_table = False
    out = {}
    for line in lines:
        if line.startswith("## 3."):
            in_table = True
            continue
        if in_table and line.startswith("## "):
            break
        m = re.match(r"\| (\w[^|]+) \| `?([^|`]+)`? \|", line)
        if m:
            out[m.group(1).strip()] = m.group(2).strip()
    return out


def live_prompt_sha(prompt_id: str) -> str:
    # Import the registry and hash the template. Sketch only.
    import importlib
    reg_mod = importlib.import_module("prompts.registry")
    entry = reg_mod.REGISTRY[prompt_id]
    return hashlib.sha256(entry.template.encode()).hexdigest()


def live_flag_value(flag_key: str) -> str:
    flags = json.loads(FLAGS_PATH.read_text())["flags"]
    variants = flags[flag_key]["variants"]
    default = flags[flag_key]["defaultVariant"]
    return variants[default]


def main() -> int:
    card = CARD_PATH.read_text()
    claimed = parse_id_table(card)

    checks = {
        "Prompt template SHA-256": live_prompt_sha(claimed["Prompt ID"]),
        "Model ID":                 live_flag_value("support_agent.model_id"),
        "Retriever version":        RETRIEVER_VERSION.read_text().strip(),
        "Index generation":         INDEX_GEN.read_text().strip(),
        "Guardrail policy":         GUARDRAIL_VERSION.read_text().strip(),
    }

    failed = []
    for key, live in checks.items():
        if claimed.get(key) != live:
            failed.append((key, claimed.get(key), live))
    if failed:
        print("Feature card drift detected:")
        for key, c, l in failed:
            print(f"  {key}: card says {c!r} but live is {l!r}")
        return 1
    print("Feature card ID table matches live state.")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

The CI job runs on every PR. The feature card's header line `Automatic drift checks: PASSING | FAILING | STALE` is updated by the same job.

- **PASSING** — last run passed and ran in the last 48 hours.
- **FAILING** — last run failed. The PR that caused the failure is blocked.
- **STALE** — last run ran > 48 hours ago (the card has not been exercised recently; investigate why CI is not running on the file).

The drift check is the mechanism that keeps the card from rotting. Without it, the card is documentation; with it, the card is executable governance.

### 2d. Review cadence

The feature card is reviewed:

- **Automatically** by CI on every PR touching the feature.
- **Manually** on the schedule in the card header (two-weekly during the first quarter after GA, quarterly afterwards).
- **On every event** that re-ratifies the charter ([[exercise-01]] Section 7).

The *manual* review is a 20-minute meeting that produces a dated entry in Section 8 (change log). Missed manual reviews push the card into `STALE` and block the next release — a soft gate that is easier to clear than letting the card drift unchecked.

## 3. Validation steps

1. **All nine sections present.** Grep for `## 1.` through `## 9.`; a missing section is a hard failure.
2. **Charter cross-reference exists.** The card's header links to the [[exercise-01]] charter and names the charter's last-ratified date.
3. **ID table is complete.** Every row (Prompt / Model / Retriever / Index / Guardrail / Flag provider) is populated.
4. **Drift-check script runs green.** `scripts/check_feature_card_ids.py` exits 0 in CI.
5. **Metrics rows cite dashboards.** Every metric row in Section 4 has a dashboard link; no metric is listed without a way to read it.
6. **At least three known limitations with evidence links.** A card with zero limitations is a card written by someone not actually running the feature.
7. **NIST AI RMF mapping names specific sub-functions.** At least three of MAP / MEASURE / MANAGE / GOVERN are mapped by *sub-function*, not only by top-level function.
8. **OWASP Top 10 mapping lists items considered AND items ruled out.** The ruled-out items name the reason; omission by silence is a hard failure.
9. **MITRE ATLAS mapping names at least one technique applicable to the feature.** With a mitigation that points to a specific upstream module artefact.
10. **Operations section includes kill lever + rollback lever + oncall path + dashboards + alerts.** All five fields; any missing field is a hard failure.
11. **Change log has at least three dated entries once the feature is in production.** The Section 8 summary must have real content — a card with an empty change log is a card in draft.
12. **"Automatic drift checks" header is updated by CI.** The header value must reflect the last CI run; `STALE` is a soft gate.

## 4. Rubric or review checklist

Instructor rubric (10 points total). A card scoring < 7 is rejected.

| Criterion | Points | What earns the points |
|---|---|---|
| All nine sections present with correct content shape | 1 | No section is a placeholder; every section has specific content. |
| Charter cross-reference + header fields populated | 1 | Owner, review cadence, last reviewed, next review, drift-check state. |
| ID table complete and drift-checked by CI | 2 | Script exists and runs; CI job is wired; `PASSING` header reflects truth. |
| Metrics section links dashboards (not names metrics without sources) | 1 | Every metric row has a dashboard link and a trailing value. |
| At least three known limitations with evidence links (fixture or incident) | 1 | Limitations are concrete and link to the regression fixture or incident doc. |
| NIST AI RMF mapping covers at least three functions at the sub-function level | 1 | MAP / MEASURE / MANAGE / GOVERN sub-function names map to specific sections. |
| OWASP ML Top 10 mapping names items considered *and* items ruled out | 1 | Ruled-out items have a one-line reason. |
| MITRE ATLAS mapping names at least one applicable technique with a mitigation referencing an upstream module artefact | 1 | The mitigation is traceable, not vague. |
| Operations section complete (kill + rollback + oncall + dashboards + alerts) | 1 | All five fields present with specific values. |

**Hard failures (automatic rejection):**

- Missing ID table or no drift-check CI job.
- No NIST AI RMF mapping at the sub-function level.
- No kill lever specified in Operations.
- Charter link missing.
- Metrics section lists metrics without dashboard links.

## 5. Common mistakes

- **Card drifted from code, no mechanism to catch.** Without the drift-check CI job, the card ages out silently. Treat the ID table as a tested assertion.
- **ID table has values but no sources.** "Prompt ID: support_answer_v15" without the registry path is an un-checkable claim.
- **Metrics without dashboard links.** The on-call at 03:00 cannot read a metric that has no URL. Every metric cites its dashboard.
- **"Known limitations: none."** This is almost always wrong. Every feature has known limitations; listing them is a trust signal, not a weakness admission.
- **NIST AI RMF line at the function level only.** "Aligned with GOVERN" is not the mapping; "GOVERN-1.6 satisfied by Section 8 change log" is.
- **OWASP mapping lists only applicable items.** The items considered and *ruled out* carry as much governance weight as the included items; the next reviewer needs to see the reasoning.
- **Kill lever named but not tied to the flag key.** The on-call needs to know exactly which flag flips, not "flip the kill flag."
- **Change log that references commit hashes only.** The log should read like a story: date, ratified change, one-line rationale.
- **Review cadence missing.** Without a scheduled review the card ages out; the STALE gate exists for this reason.
- **Mapping to ATLAS without a mitigation.** Naming AML.T0051 and not naming the mod-107 artefact that handles it leaves the reviewer no auditable thread.
- **Feature card described as "a nice-to-have doc."** It is the artefact that answers both "what paged me" (on-call) and "show me the governance" (auditor). Treat it as a dependency of release, not a doc deliverable.
- **Feature card merged without the Safety-Eng reviewer** (if the feature carries any ML01 / ML09 / ATLAS mitigations). Those sections are not self-grading.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the Section 6 mapping draws directly from the function / sub-function catalogue. MEASURE-2.11 ("metrics tracked across the operating envelope") and MANAGE-2.3 ("corrective actions documented") are the two sub-functions a feature card most directly satisfies at applied altitude.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the Section 6 OWASP mapping must list items considered and items ruled out; the project explicitly recommends documenting the reasoning behind the exclusions so the next review has traceability.
- MITRE ATLAS — <https://atlas.mitre.org/> — the Section 6 ATLAS mapping names techniques with mitigations traceable to module artefacts. AML.T0051 (Prompt Injection) is the most common technique applicable to feature-level LLM deployments; AML.T0053 (Plugin Compromise) becomes applicable the moment the feature adds write-side tools.

Related exercises: [[exercise-01]] (the charter is the upstream artefact; the feature card is downstream and narrower), [[exercise-02]] (every regression fixture referenced in Section 5 came from this walkthrough), [[exercise-03]] (the substrate decisions determine which flag provider / gateway / eval platform appear in the ID table), [[exercise-05]] (the dependencies listed in Section 7 are the counterparts that delegation contracts formalise); mod-102 [[exercise-05]] (prompt versioning that drives the Prompt ID / SHA-256 row), mod-103 [[exercise-01]] (retriever contract providing the Retriever version row), mod-106 [[exercise-02]] (eval set tied to the limitations in Section 5), mod-107 [[exercise-03]] (guardrail policy providing the Guardrail policy row + anti-metric sourcing), mod-108 [[exercise-02]] (gateway providing the Model ID observability), mod-109 [[exercise-02]] (SLO board providing the latency metric), mod-110 [[exercise-02]] (feature-health dashboard driving the primary metric), mod-110 [[exercise-03]] (feedback pipeline for thumbs-up metric), mod-111 [[exercise-01]] (flag layer providing the kill / rollback levers).
