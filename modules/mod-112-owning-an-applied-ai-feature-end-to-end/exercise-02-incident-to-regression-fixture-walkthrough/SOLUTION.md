### exercise-02: Incident-to-Regression-Fixture Walkthrough — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to walk through the lifecycle of a production incident on an LLM feature, starting from the on-call page and ending with a **regression fixture** checked into the repo that would have *caught* the incident in offline eval. The mechanics live in mod-106 [[exercise-02]] (eval set construction) and mod-110 [[exercise-04]] (incident response); this exercise forces the applied engineer to *link them together* under their own ownership.

The loop this solution specifies is deliberate:

1. **Incident detected** — an anti-metric from the [[exercise-01]] charter fires, or a user complaint is triaged to this feature.
2. **Triage** — get to a working theory and a reversible containment lever within minutes.
3. **Containment** — flip the smallest mod-111 [[exercise-01]] lever that stops user harm.
4. **Minimal reproduction** — strip the incident down to the smallest trace, prompt, and retrieved context that reproduces the failure.
5. **Regression fixture** — convert the reproduction into a check-in-able test with an input, an expected-shape assertion, and a label that lets it be slotted into the mod-106 eval set.
6. **Fix** — change the prompt / tool / guardrail; verify the fixture now passes.
7. **Ship via the shadow → canary regime** — the fix rides the same mod-111 [[exercise-02]] gates as any new variant.
8. **Post-incident review** — the fixture, the fix PR, and the charter re-ratification (if the incident triggered one) all link to a single incident record.

The failure modes this exercise defends against are specific to the handoff between incident response and offline eval:

1. **Fix without a fixture.** The team ships a prompt change, the incident closes, nobody wrote a test. Six weeks later, a prompt refactor regresses the exact same behaviour; the team does not notice until the anti-metric fires in prod a second time.
2. **Fixture that passes trivially.** The team captures "the prompt that broke" without *isolating the failure mode* — the fixture includes side-channels (a specific user ID, a specific timestamp) that make it pass even if the bug comes back.
3. **Fixture that includes PII.** The reproduction captures a real user's prompt and context, including PII that then enters the test fixtures directory and the mod-106 eval set. The incident has just become a data-handling incident.
4. **Fixture that is not graded.** The fixture is added to the eval set but no judge or assertion scores it. The suite reports "passed" because it has no opinion on the output.
5. **Containment skipped in favour of "root cause first."** The on-call tries to understand the bug before stopping the bleeding. Users continue to see the failure for the duration of the investigation.
6. **No link from the eval row back to the incident.** Six months later, someone asks "why is this unusual example in the eval set?" and the answer is lost. The eval row needs an `incident_id` field.

This solution provides:

- A **runnable reproduction** harness (`scripts/reproduce_incident.py`) that replays a captured prompt + context.
- A **regression-fixture file format** with the minimum required fields and a graded assertion.
- A **worked example** — a fictional-but-realistic structured-output incident, narrated end-to-end from page to post-incident review.
- A **grading rubric** scoring the quality of the fixture, not only its existence.
- An **anti-pattern list** for the common fixture-quality failures.

## 2. Worked answer or implementation

### 2a. The flow — eight steps, each with an owner

| Step | Owner | Artefact produced | Gate to next step |
|---|---|---|---|
| 1. Detect | On-call applied-AI engineer | Incident record (`INC-<id>`), page acknowledged | Feature triaged to this team |
| 2. Triage | On-call | Working theory + severity | Severity set to P0/P1/P2 |
| 3. Contain | On-call | Flag flip + comms | User-facing impact stopped (observed on dashboards) |
| 4. Reproduce | On-call + applied engineer | `reproductions/<INC-id>/case-<n>.json` | Failure reproduces offline, consistently |
| 5. Fixture | Applied engineer | `evals/fixtures/<INC-id>-<slug>.yaml` + assertion | Fixture runs red before the fix, green after |
| 6. Fix | Applied engineer | PR with prompt / guardrail / code change | Fixture runs green; broader eval set does not regress |
| 7. Ship | Applied engineer | mod-111 [[exercise-02]] rollout record | Shadow → canary → full |
| 8. Review | PM + applied engineer | Post-incident review doc + charter re-ratification decision | Charter either ratifies or triggers re-ratification |

Steps 3 and 4 are allowed to run in parallel. Step 5 cannot start until Step 4 reproduces offline — a "fixture" that cannot be run outside production is not a regression test.

### 2b. Worked example — the walkthrough

> The example below uses a **fictional** feature (`helpdesk.reply_assistant` from [[exercise-01]]) and a **fabricated incident ID** (`INC-2026-11-15-helpdesk-currency`) written in the voice of a post-incident review. No real customer, operator, or outage is being referenced. Treat every name, metric value, and timestamp as illustrative pedagogy, not factual claim.

#### Step 1 — detect

At `14:47:02 UTC` on `2026-11-15`, the mod-110 [[exercise-02]] feature-health dashboard page fires on the anti-metric `grounding_violation_rate_1h > 5%` (charter kill criterion K1 from [[exercise-01]]). Observed rate: `7.3%` over the last hour. Normal baseline is `< 0.5%`.

Incident record opened: `INC-2026-11-15-helpdesk-currency` (title filled in during Step 2 once the theory firms up).

#### Step 2 — triage

The on-call engineer pulls the trace UI (mod-110 [[exercise-01]]) filtered on
`app.feature = "helpdesk.reply" AND app.grounding.violation = true AND time > 13:40 UTC`.

Looking at 20 sample spans:

- All 20 contain the user asking a question involving a specific **currency** (EUR, GBP, JPY).
- The retrieved context (per `app.retrieval.doc_ids` span attribute, mod-110 [[exercise-01]]) in each case contains USD amounts.
- The LLM response reports amounts in the *user-requested currency* — a number that does not appear in the context.

Working theory, written in the incident doc at `14:51`:

> *When the user asks about a billing amount in a non-USD currency, the model performs an unsupervised currency conversion rather than refusing. The retrieved documents only contain USD values; the conversion is a hallucination.*

Severity: **P1**. User-visible, high-trust surface, financial content. Not P0 — no action-taking, no legal-reporting event.

#### Step 3 — contain

The smallest lever: the mod-111 [[exercise-01]] `support_agent.prompt_id` flag. The variant currently in production is `support_answer_v15_concise` (shipped yesterday). The LKG (`support_answer_v14`) does not have this behaviour in offline eval — but offline eval also did not probe the currency case, which is the finding this incident surfaces.

Containment lever pulled at `14:53:11`:

- `support_agent.prompt_id` targeting changed: non-internal traffic → `v14`.
- `support_agent.cohort` left at previous value; v14 is the safe default.
- Observed TTE (mod-111 [[exercise-01]] 2f pattern): 2 seconds; next 5 minutes of traffic all land on `v14`.
- Anti-metric `grounding_violation_rate_1h` returns to `< 0.5%` within 50 minutes (window-cycling).

Containment is complete. The user-facing bleeding is stopped. Root-cause work continues in parallel.

#### Step 4 — reproduce

The on-call captures three failing spans from Step 2, using the mod-110 [[exercise-04]] trace-to-reproduction script. The reproduction file is checked into the incident folder, scrubbed by the mod-107 [[exercise-03]] PII pipeline:

```json
// reproductions/INC-2026-11-15-helpdesk-currency/case-01.json
{
  "incident_id": "INC-2026-11-15-helpdesk-currency",
  "capture_source": "trace span <scrubbed-span-id>",
  "captured_at": "2026-11-15T14:12:08Z",
  "prompt_id": "support_answer_v15_concise",
  "prompt_template_hash": "a3f1b2...<redacted-sha256>",
  "model_id": "gpt-4o-2024-08-06",
  "evaluation_context": {
    "surface": "helpdesk.web",
    "locale": "en",
    "internal": false
  },
  "user_message": "What was the total of my last invoice in euros?",
  "retrieved_context": [
    {
      "doc_id": "invoice-<scrubbed>",
      "text": "Invoice <scrubbed>. Issued 2026-10-14. Total: $142.00 USD."
    }
  ],
  "observed_response": "Your last invoice total was approximately €131.60 EUR."
}
```

Running the reproduction harness:

```bash
$ python scripts/reproduce_incident.py \
    --case reproductions/INC-2026-11-15-helpdesk-currency/case-01.json \
    --runs 10

case-01: 10/10 runs produced a non-USD currency amount not present in context.
         Example output: "Your last invoice total was approximately €131.60 EUR."
         Grounding violation: TRUE (amount not in retrieved_context).
Status:  REPRODUCES (10/10)
```

Reproduction harness in full:

```python
# scripts/reproduce_incident.py
import argparse, json, re, sys
from pathlib import Path

from app.handler import answer_turn_offline
from prompts.registry import resolve as resolve_prompt
from app.grounding import contains_number_not_in_context


def reproduce_case(path: Path, runs: int) -> dict:
    case = json.loads(path.read_text())
    entry = resolve_prompt(case["prompt_id"])
    results = []
    for _ in range(runs):
        completion = answer_turn_offline(
            model_id=case["model_id"],
            template=entry.template,
            evaluation_context=case["evaluation_context"],
            user_message=case["user_message"],
            retrieved_context=case["retrieved_context"],
        )
        violates = contains_number_not_in_context(completion, case["retrieved_context"])
        results.append({"completion": completion, "violates_grounding": violates})
    violated = sum(1 for r in results if r["violates_grounding"])
    return {
        "case_id": path.stem,
        "runs": runs,
        "violated": violated,
        "reproduces": violated >= max(1, runs // 2),
        "samples": results[:3],
    }


def main() -> int:
    ap = argparse.ArgumentParser()
    ap.add_argument("--case", required=True, type=Path)
    ap.add_argument("--runs", type=int, default=10)
    args = ap.parse_args()
    r = reproduce_case(args.case, args.runs)
    print(f"{r['case_id']}: {r['violated']}/{r['runs']} runs violated grounding.")
    print(f"Status:  {'REPRODUCES' if r['reproduces'] else 'DOES NOT REPRODUCE'} "
          f"({r['violated']}/{r['runs']})")
    return 0 if r["reproduces"] else 1


if __name__ == "__main__":
    sys.exit(main())
```

The offline reproduction (`answer_turn_offline`) is a thin wrapper that bypasses the retrieval stage and injects the `retrieved_context` from the case file directly — isolating the LLM call from the retriever, which is the correct scope for *this* failure (the retriever returned the right documents; the LLM invented the conversion).

#### Step 5 — regression fixture

The fixture is the artefact that gets checked into `evals/fixtures/` and slotted into the mod-106 [[exercise-02]] eval set. One YAML file per fixture:

```yaml
# evals/fixtures/INC-2026-11-15-helpdesk-currency-eur.yaml
id: helpdesk-currency-eur
incident_id: INC-2026-11-15-helpdesk-currency
category: grounding_violation
severity: P1
created_at: 2026-11-15
owner: applied-ai-team

inputs:
  user_message: "What was the total of my last invoice in euros?"
  retrieved_context:
    - doc_id: "invoice-fixture-01"
      text: "Invoice fixture-01. Issued 2026-10-14. Total: $142.00 USD."

expected_behaviour:
  # The graded assertion. "What a correct answer looks like" — not the one
  # output string, but the shape every correct response must have.
  must_contain_any_of:
    - "USD"
    - "dollars"
  must_not_contain_any_of:
    - "€"
    - "EUR"
    - "euros"
    - "approximately"       # conversion-language tells
    - "roughly"
  grounding:
    # Every numeric claim in the response must appear verbatim (or as a
    # normalised form) in retrieved_context.
    mode: "no_numbers_outside_context"
    context_numbers: ["142.00", "142"]

judge:
  kind: "deterministic + llm"
  deterministic_checks:
    - "must_contain_any_of"
    - "must_not_contain_any_of"
    - "grounding"
  llm_judge:
    # Secondary LLM-judged dimension in case the deterministic checks miss
    # a paraphrase of the failure mode.
    model: "claude-sonnet-4-6"
    rubric: >
      Does the assistant's response invent a currency conversion, implicit or
      explicit, that is not derivable from the retrieved context? If yes,
      score 0. If the response either stays in USD, refuses the conversion
      with a clear "I can't convert currencies," or asks the user a
      clarifying question, score 1.

runs:
  samples: 5         # Hallucination is probabilistic; one run is false comfort.
  pass_rule:
    deterministic_all: true
    llm_judge_min_mean: 0.9

tags:
  - regression
  - grounding
  - currency
```

The companion test runner, which the mod-106 suite wires in:

```python
# evals/runner/regression.py (sketch — extends mod-106 [[exercise-02]] runner)
import yaml
from pathlib import Path

from evals.judges import deterministic_check, llm_judge
from app.handler import answer_turn_offline


def run_fixture(fixture_path: Path) -> dict:
    spec = yaml.safe_load(fixture_path.read_text())
    out = {"id": spec["id"], "incident_id": spec["incident_id"], "runs": []}
    for _ in range(spec["runs"]["samples"]):
        completion = answer_turn_offline(
            # Variant under test is the prompt being evaluated by the suite,
            # not the fixture's original prompt_id — the fixture tests the
            # *currently proposed* prompt against the known-bad case.
            ...
            user_message=spec["inputs"]["user_message"],
            retrieved_context=spec["inputs"]["retrieved_context"],
        )
        det = deterministic_check(completion, spec["expected_behaviour"])
        judge = llm_judge(completion, spec["judge"]["llm_judge"])
        out["runs"].append({"completion": completion, "det": det, "judge": judge})
    out["passed"] = all(r["det"]["ok"] for r in out["runs"]) and \
                    (sum(r["judge"]["score"] for r in out["runs"]) /
                     len(out["runs"])) >= spec["runs"]["pass_rule"]["llm_judge_min_mean"]
    return out
```

The fixture is checked in as part of the fix PR. The eval runner picks it up automatically because `evals/fixtures/*.yaml` is a globbed input to the mod-106 suite; no eval-suite registration code is required.

#### Step 6 — fix

The prompt change that corrects the behaviour is a *guardrail addition*, not a rewrite. Minimal diff against the template in mod-111 [[exercise-01]] 2c:

```diff
# prompts/registry.py — support_answer_v15_concise
 "You are a helpful support assistant.\n"
 "Answer the user's question using the retrieved docs in <context>.\n"
 "Respond in one or two sentences. Include exactly one concrete next step.\n"
 "If the docs do not answer the question, say so in one sentence.\n"
+"\n"
+"Rules for numeric content:\n"
+"- Every number in your answer (amount, date, invoice ID) must appear\n"
+"  verbatim in the <context> block. Do not convert between units or\n"
+"  currencies. If the user asks for a number in a form not in the\n"
+"  context (e.g. a different currency), say you do not have that\n"
+"  information and offer the raw figure that is in the context.\n"
 "\n"
 "<context>\n{context}\n</context>\n"
 "\n"
 "User: {user_message}\n"
```

The prompt version is bumped to `support_answer_v15_concise_r2` so the rollback lever stays sharp (mod-102 [[exercise-05]] prompt versioning).

Running the fixture against the patched prompt:

```bash
$ pytest evals/runner/regression.py::test_fixture[INC-2026-11-15-helpdesk-currency-eur]
PASSED — 5/5 deterministic checks OK; llm_judge mean 0.98.
```

Running the full mod-106 eval suite:

```
Before patch: 324/330 passing. After patch: 325/330 passing.
Delta: +1 (the new currency fixture). No regressions on the pre-existing 324.
```

If any pre-existing fixture had regressed, the fix would not merge — the patch is weighed against the existing eval surface, not only the new fixture.

#### Step 7 — ship via the shadow → canary regime

The patched prompt variant `support_answer_v15_concise_r2` enters the mod-111 [[exercise-02]] shadow track. Shadow passes (no deltas on existing fixtures, delta of +1 on the new fixture). Canary opens at 5%. Anti-metric `grounding_violation_rate_1h` remains at `< 0.5%` on both the canary and control arms. Canary expands to 25%, then 100%, over the 48 hours the charter's review cadence specifies.

#### Step 8 — post-incident review

The post-incident review doc is one page, templated:

```markdown
# Post-incident review — INC-2026-11-15-helpdesk-currency

Severity: P1
User-visible duration: 14:47 (detect) — 14:53 (contain) = 6 minutes.
Root-cause fix shipped: 2026-11-17 (shadow + canary + full).

## What happened
A prompt variant shipped the previous day (`support_answer_v15_concise`) did
not refuse unsupervised currency conversion; when users asked billing
questions in non-USD currencies the model generated numeric answers that
were not in the retrieved context. The grounding-violation anti-metric fired
(charter kill criterion K1).

## What we did
- Contained with the mod-111 [[exercise-01]] prompt-ID flag (2 s TTE).
- Reproduced offline in the trace-replay harness (10/10 reproductions).
- Captured regression fixture `INC-2026-11-15-helpdesk-currency-eur` in the
  mod-106 eval set.
- Shipped a prompt-level guardrail (`r2` suffix) through the full shadow →
  canary → 100% rollout.

## What this changes
- Charter re-ratification: NOT triggered (no cohort / model / regulator
  event). The charter's K1 fired exactly as designed.
- Eval set: +1 fixture in the grounding category. The suite now probes the
  currency case for every future variant.
- Prompt review protocol: added a line to the mod-102 [[exercise-05]] prompt
  review checklist — "any numeric-content rule missing? grounding rule
  missing?" This is the delta that would have caught the variant in review.

## Links
- Incident record: INC-2026-11-15-helpdesk-currency
- Fix PR: #4321
- Fixture: evals/fixtures/INC-2026-11-15-helpdesk-currency-eur.yaml
- Rollout record: mod-111/exercise-02 shadow+canary report
- Charter: docs/features/helpdesk.reply_assistant/charter.md (unchanged)
```

### 2c. Fixture format — minimum required fields

| Field | Why it is required |
|---|---|
| `id` | Human-readable slug; used in test names and dashboards. |
| `incident_id` | Back-pointer to the incident record. Without it, the fixture loses its reason to exist as it ages. |
| `category` | One of `grounding_violation`, `safety`, `tool_error`, `latency_outlier`, `structured_output`, ... — the mod-106 suite groups by this field. |
| `severity` | The incident's severity. Fixtures born from P0/P1 incidents are "blocker" fixtures — regression here blocks release. |
| `created_at` + `owner` | Attribution; who to ask when the fixture is confusing six months later. |
| `inputs` | Minimum reproduction inputs: the user message and the retrieved context. PII-scrubbed. |
| `expected_behaviour` | A *shape* assertion, not a string match. Lists what must appear and what must NOT appear, plus any grounding constraint. |
| `judge` | Both deterministic and LLM-judge (where applicable); the pass rule is explicit. |
| `runs.samples` | Number of samples; hallucination fixtures must run ≥ 5 times to detect probabilistic reappearance. |
| `tags` | Searchable; "regression" and "grounding" let the suite slice. |

### 2d. Isolation checks — "does the fixture catch the bug and only the bug?"

Before the fixture merges, the applied engineer runs these two cross-checks:

- **Red-then-green:** run the fixture against the *pre-fix* prompt. It must fail. Then run against the post-fix prompt. It must pass. A fixture that is green against both is testing nothing; a fixture that is red against both means the fix is incomplete.
- **Side-channel strip:** change one irrelevant detail (user ID, timestamp, surface) and re-run. The result must not change. If it does, the fixture is pinned to a side-channel; strip the pin.

### 2e. PII handling in fixtures

The PII pipeline from mod-107 [[exercise-03]] runs over every captured reproduction *before* it is written to `reproductions/`. Fields replaced:

- User identifiers (`user_id`, `session_id`, `tenant_id`) → fixture-local placeholders (`user-fixture`).
- Document identifiers (`doc_id`, invoice numbers, order IDs) → `*-fixture-NN`.
- Free-text PII (names, addresses, email) → placeholder strings.
- Monetary amounts, dates, currencies that are *essential to reproducing the failure* are kept, but only if they are not themselves user-identifying.

The fixture-level audit (reviewer runs `scripts/scrub_audit.py` on the fixture file) is a required step; merging a fixture without the audit log is a hard failure.

## 3. Validation steps

1. **Reproduction before fixture.** `scripts/reproduce_incident.py --case <path>` returns `REPRODUCES` on the pre-fix state and `DOES NOT REPRODUCE` on the post-fix state. Both runs are logged.
2. **Fixture red-then-green.** `pytest evals/runner/regression.py::test_fixture[<id>]` fails on pre-fix, passes on post-fix. The fixture's own status transition is visible in the PR CI logs.
3. **Full eval suite regression check.** The full mod-106 suite shows a delta of exactly `+1` (the new fixture) with zero pre-existing fixtures regressing. A regression on a pre-existing fixture blocks the merge.
4. **PII scrub audit.** The reviewer runs the scrub audit script on the fixture inputs; no hits on name / address / email / raw ID patterns.
5. **Fixture sampling count.** `runs.samples ≥ 5` for any hallucination / grounding fixture. One-shot deterministic-only fixtures (e.g. tool-schema regression) may run with `samples: 1` only if the failure mode is fully deterministic.
6. **Deterministic + judge coverage.** At least one deterministic check *and* at least one LLM-judged check on fixtures where the failure is a semantic shape (not a format violation). Deterministic-only is insufficient; judge-only is too noisy.
7. **Back-pointer check.** `incident_id` field is populated and resolves to a real incident record; reviewer opens the incident record as part of fixture review.
8. **Shadow / canary record attached.** For P0 / P1 incidents, the PR links to the mod-111 [[exercise-02]] shadow-and-canary run record. Shipping the fix without the rollout record is a hard failure for severe incidents.
9. **Post-incident review doc exists.** One page, templated, with the "what this changes" section specifying whether charter re-ratification was triggered.
10. **Eval suite runtime budget check.** A fixture that extends the suite's runtime beyond its budget (mod-106 [[exercise-02]] budget) forces either a split-suite decision or a `tags: [slow]` segregation. Unbounded fixture growth is a known failure mode.

## 4. Rubric or review checklist

Instructor rubric (10 points total). A submission scoring < 7 is rejected; hard failures are automatic.

| Criterion | Points | What earns the points |
|---|---|---|
| Containment step pulled a flag (not a deploy) and TTE is documented | 1 | Flag lever named, timestamped, TTE measured against the mod-111 [[exercise-01]] 2f pattern. |
| Reproduction harness runs offline, deterministically reproduces the failure | 1 | 10-run harness with ≥ half of runs reproducing; the script is checked in. |
| Fixture has all nine required fields (see 2c) | 1 | No empty cells; `incident_id` and `owner` present. |
| Fixture isolates the failure mode (red-then-green + side-channel strip) | 2 | Both checks are documented in the PR; a fixture that is red against the fix is a hard failure. |
| Fixture includes both deterministic and LLM-judge assertions (for semantic failures) | 1 | `expected_behaviour` lists must_contain / must_not_contain; `judge` has an LLM rubric. |
| Fixture samples ≥ 5 for probabilistic failure modes | 1 | `runs.samples ≥ 5`; justified if less. |
| PII scrub audit log is attached | 1 | `scripts/scrub_audit.py` output on the fixture inputs; zero hits on identifier patterns. |
| Fix PR passes full eval suite with no pre-existing regressions | 1 | +1 delta; zero regressions. |
| Fix shipped through shadow → canary (for P0/P1) | 1 | Rollout record attached to the PR. |

**Hard failures (automatic rejection):**

- Fixture containing real user PII (any name, address, email, raw tenant ID, or raw document ID matching production patterns).
- Fixture that passes trivially on the pre-fix prompt (no red-then-green).
- "Fix" shipped by a code deploy without a flag flip as the containment step.
- No post-incident review doc.
- No back-pointer `incident_id` on the fixture.

## 5. Common mistakes

- **Fix without a fixture.** The team closes the incident, nobody files the regression test. Rule: the incident does not close until the fixture is merged.
- **Fixture asserts the exact response string.** Hallucination fixes are probabilistic; the model will produce a different-but-correct response next time and the test will fail on a change that is actually a pass. Use shape assertions: `must_contain_any_of`, grounding mode, judge rubric — not literal-string equality.
- **Fixture captures a side-channel.** The test passes because of a specific user ID or timestamp in the prompt, not because the bug is fixed. Strip side-channels during fixture authoring (2d).
- **One-shot judge on a probabilistic failure.** Running the judge once on a hallucination fixture gives false comfort. Run ≥ 5 samples and average the score; set `llm_judge_min_mean`.
- **PII in the fixture file.** The incident becomes a data-handling incident. Scrub before writing to disk; audit before merging.
- **No deterministic check.** LLM-judge-only fixtures drift as the judge model changes. Pair every LLM-judged check with a deterministic shape check (`must_not_contain`, numeric-grounding mode).
- **Fixture that extends suite runtime past budget.** The eval set grows forever if every fixture is kept in the fast suite. Tag slow fixtures (`tags: [slow]`) and segregate the suite; mod-106 [[exercise-02]] names the budget.
- **Containment skipped in favour of "understanding root cause first."** Users keep seeing the failure while the team debugs. Containment is a reversible lever; pulling it adds minutes of downtime risk and prevents hours of user harm.
- **`kill` used where `prompt_id` flip would have sufficed.** Taking the whole feature off when a smaller lever exists (prompt swap) is a smaller-lever-first violation (mod-111 [[exercise-01]] 2a).
- **No link from fixture back to incident.** Six months later the fixture looks odd ("why a currency case?") and nobody knows. Mandate `incident_id` on every regression fixture.
- **The fixture becomes the only test.** Fixtures are the floor, not the ceiling — a comprehensive eval set still needs the mod-106 [[exercise-02]] curated examples. Regression fixtures protect against *re-occurrence*; curated examples protect against *regression on behaviour we already care about*.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — MANAGE-2.3 requires continuous monitoring and corrective action for deployed AI systems; the incident-to-fixture loop is the direct operational artefact that satisfies this for feature-level LLM deployments. MANAGE-4.1 further requires that a corrective action be verifiable — the regression fixture is the verifiable artefact.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the ML09 (Output Integrity) category is the most common source of incidents at this altitude; the grounding-violation anti-metric and the currency fixture are directly aligned with ML09 mitigations.
- MITRE ATLAS — <https://atlas.mitre.org/> — AML.T0051 (LLM Prompt Injection) incidents produce fixtures with a different shape (adversarial input, specific exploit prompt) but the same lifecycle; the fixture format in 2c is category-agnostic by design.

Related exercises: [[exercise-01]] (the kill criterion K1 is what paged the on-call in Step 1), [[exercise-04]] (the feature card should list a count of merged regression fixtures as a trust signal), [[exercise-05]] (a fixture that implicates a peer specialist — e.g. a retriever bug — forks into the delegation contract); mod-102 [[exercise-05]] (prompt versioning that produces the `r2` suffix), mod-103 [[exercise-04]] (retriever-side fixtures follow the same format), mod-106 [[exercise-02]] (eval set construction — the fixture is one row), mod-106 [[exercise-04]] (safety-eval suite — a different fixture category feeding charter K5), mod-107 [[exercise-03]] (PII scrubber used in Step 4), mod-110 [[exercise-01]] (trace capture that seeds the reproduction), mod-110 [[exercise-04]] (incident response playbook — the parent process), mod-111 [[exercise-01]] (flag layer providing the containment lever in Step 3), mod-111 [[exercise-02]] (shadow + canary gates used in Step 7), mod-111 [[exercise-04]] (prompt rollback — the single-flag version of Step 3).
