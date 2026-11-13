### exercise-03: Eval-Gated CI With Promptfoo and Braintrust — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to wire the offline suite from [[exercise-01]] and the regression fixtures from [[exercise-02]] into the **CI gate** so a PR that would regress the surface cannot merge. The deliverable is not "run the eval in CI" — that is one shell command. The deliverable is the **gate design**: which bars block the merge, which block only the author, which are advisory; where results are persisted; how flakes are handled; how the second eval platform cross-checks the first.

This solution uses both tools deliberately. Promptfoo is the local, YAML-first, CI-runnable eval engine; it is the tool that owns the per-PR fast lane. Braintrust is the server-side experiment store; it is the tool that owns the long-run history, the dataset UI, and the human review of flagged rows. Each platform's role is explicit so neither becomes a "run everywhere" tool whose failures nobody knows how to debug.

This solution provides:

- A reference **`promptfooconfig.yaml`** that runs the suite in CI against the same fixtures the offline runner from [[exercise-01]] uses.
- A reference **Braintrust integration** that uploads each CI run as an experiment, tags it with the PR + commit SHA, and links back from the GitHub status check.
- A reference **`.github/workflows/evals.yml`** showing the three CI lanes (fast, full, nightly) with the exact pass bars each one enforces.
- A **flake policy** (retry rules, quarantine, the budget for retries).
- A **two-engine cross-check** that ensures the promptfoo gate and the offline runner from [[exercise-01]] agree on verdicts; disagreement is itself a CI failure.
- A **grading rubric** for the instructor.

The lesson the exercise defends is: *a CI gate is a contract between the merge button and the suite*. A gate that retries on red until it goes green is a gate the team learns to ignore. A gate whose fast lane disagrees with the nightly lane is a gate that produces incidents. The design makes both failures visible as CI failures in their own right.

## 2. Worked answer or implementation

### 2a. The three lanes (deliverable 1 — decision rationale)

| Lane | Trigger | Scope | Pass bar enforcement | Latency budget |
|---|---|---|---|---|
| **Fast** | Every PR push | Shape-B + Shape-D rows from [[mod-105 exercise-04]] + **all** regression rows from [[exercise-02]] + a 10-row sample of Shape-A | Hard gate: any miss blocks merge. | ≤ 5 min |
| **Full** | PR ready-for-review + before merge | All rows + global, slice, and no-regression bars from [[exercise-01]]'s `thresholds.yaml` | Hard gate: any miss blocks merge. | ≤ 20 min |
| **Nightly** | Scheduled 03:00 UTC on `main` | Full suite + extended set (a) adversarial suite from [[mod-107]], (b) persona slices from [[exercise-05]] | Reports via Braintrust; a miss opens an issue, does not retroactively block prior merges. | ≤ 60 min |

The pattern: **fast catches the cheap regressions quickly; full catches everything before merge; nightly catches the slow, expensive, or environmental regressions on a cadence and feeds the issue tracker**. The fast lane exists because developers won't wait 20 minutes for the full lane on every push; the full lane exists because the fast lane sampling will miss things.

### 2b. `promptfooconfig.yaml` (deliverable 2)

```yaml
# promptfooconfig.yaml — root config; lane selection via --tag.
description: helpdesk-reply-copilot eval gate
prompts:
  - file://surfaces/helpdesk-reply-copilot/prompts/draft_reply.jinja
providers:
  - id: production-controller
    # Delegates to our AgentController so promptfoo exercises the real
    # loop, not a parallel one. See exercise-01 §2d; the provider is a
    # thin shim around AgentController.from_defaults().
    config:
      path: ./eval/provider_agent_controller.py
tests:
  - file://evals/rows/task/*.yaml
  - file://evals/rows/refusal/*.yaml
  - file://evals/rows/format/*.yaml
  - file://evals/rows/regression/*.yaml
defaultTest:
  assert:
    # Format validity is a global invariant — apply to every row.
    - type: is-valid-json-schema
      value: file://surfaces/helpdesk-reply-copilot/schemas/draft_reply.json
      weight: 0.1
# Tag rows for lane selection. Promptfoo reads these via `--tag`.
tags:
  fast:
    include:
      - shape:B
      - shape:D
      - requires_refusal:true
      - source.kind:trace_snapshot   # all regression rows
      - sample:fast                   # the 10-row A-sample, pre-pinned
  full:
    include: ["*"]
sharing:
  # Upload every fast+full run to Braintrust as an experiment.
  braintrust:
    project: helpdesk-reply-copilot
    tags_from_env: ["GITHUB_SHA", "GITHUB_PR_NUMBER", "LANE"]
outputPath: .aicg/promptfoo-results.json
```

The provider shim keeps promptfoo honest about running the production controller:

```python
# eval/provider_agent_controller.py
from agent_controller import AgentController


class AgentControllerProvider:
    """Promptfoo custom provider → production AgentController."""

    id = "production-controller"

    def __init__(self, config):
        self._ctl = AgentController.from_defaults()

    def call_api(self, prompt, context, options):
        row = context["vars"]            # promptfoo passes YAML vars here
        state = self._ctl.run_once_sync(row["fixture"])
        return {
            "output": state.final_text,
            "tokenUsage": {
                "total": state.usage.tokens_in + state.usage.tokens_out,
                "prompt": state.usage.tokens_in,
                "completion": state.usage.tokens_out,
            },
            # Non-standard but useful: ship trace side-channels through
            # the result so custom assertions can read them.
            "metadata": {
                "tool_log": state.tool_log,
                "pause_events": state.pause_events,
                "citations": state.citations,
            },
        }
```

### 2c. Braintrust integration (deliverable 3)

```python
# eval/braintrust_upload.py — called from CI after every run.
import json
import os
import pathlib
import sys

import braintrust


def upload(results_path: str, lane: str) -> None:
    results = json.loads(pathlib.Path(results_path).read_text())
    project = braintrust.init(project="helpdesk-reply-copilot")
    exp = project.create_experiment(
        name=f"pr-{os.environ.get('GITHUB_PR_NUMBER', 'local')}-{lane}",
        metadata={
            "commit": os.environ["GITHUB_SHA"],
            "pr": os.environ.get("GITHUB_PR_NUMBER", ""),
            "lane": lane,
            "thresholds_registered_at": results.get(
                "thresholds_registered_at", ""),
            "baseline_commit": results.get("baseline_commit", ""),
        },
    )
    for row in results["results"]:
        exp.log(
            input=row["vars"].get("fixture", {}).get("user_message", ""),
            output=row.get("response", {}).get("output", ""),
            expected=row["vars"].get("scoring", {}).get("rubric", []),
            scores={a["type"]: a["score"]
                    for a in row.get("gradingResult", {}).get("componentResults", [])
                    if "score" in a},
            metadata={
                "row_id": row["vars"].get("id"),
                "shape": row["vars"].get("shape", ""),
                "stratum": row["vars"].get("stratum", ""),
                "source_incident": row["vars"].get("source", {}).get("incident"),
            },
            tags=[f"lane:{lane}", f"shape:{row['vars'].get('shape', '')}"],
        )
    print(f"braintrust exp url: {exp.url()}", file=sys.stderr)


if __name__ == "__main__":
    upload(sys.argv[1], sys.argv[2])
```

The upload step is the one that makes the results reviewable six months later. The `metadata` block carries the thresholds' registration timestamp + baseline commit from [[exercise-01]] so a Braintrust experiment UI shows which bar version the run was graded against. A bar change proposal later references specific experiment URLs, not just numbers.

### 2d. GitHub Actions workflow (deliverable 4)

```yaml
# .github/workflows/evals.yml
name: evals
on:
  pull_request:
    paths:
      - "surfaces/helpdesk-reply-copilot/**"
      - "evals/**"
      - "agent_controller/**"
  push:
    branches: [main]
  schedule:
    - cron: "0 3 * * *"
concurrency:
  group: evals-${{ github.ref }}
  cancel-in-progress: true
jobs:
  fast:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    timeout-minutes: 8
    env:
      LANE: fast
      BRAINTRUST_API_KEY: ${{ secrets.BRAINTRUST_API_KEY }}
      GITHUB_SHA: ${{ github.sha }}
      GITHUB_PR_NUMBER: ${{ github.event.pull_request.number }}
    steps:
      - uses: actions/checkout@v4
      - run: pip install -e . promptfoo braintrust
      - name: promptfoo (fast lane)
        run: promptfoo eval -c promptfooconfig.yaml --tag fast \
             --output .aicg/promptfoo-fast.json --no-progress-bar
      - name: Two-engine cross-check
        run: python -m eval.cross_check \
             .aicg/promptfoo-fast.json \
             --against offline-runner
      - name: Upload to Braintrust
        if: always()
        run: python -m eval.braintrust_upload \
             .aicg/promptfoo-fast.json fast
      - uses: actions/upload-artifact@v4
        with: { name: fast-results, path: .aicg/promptfoo-fast.json }

  full:
    if: github.event_name == 'pull_request' &&
        contains(github.event.pull_request.labels.*.name, 'ready-for-review')
    needs: [fast]
    runs-on: ubuntu-latest
    timeout-minutes: 25
    env: *fast-env       # same env block via YAML anchor
    steps:
      - uses: actions/checkout@v4
      - run: pip install -e . promptfoo braintrust
      - name: promptfoo (full lane)
        run: promptfoo eval -c promptfooconfig.yaml --tag full \
             --output .aicg/promptfoo-full.json --no-progress-bar
      - name: Enforce pre-registered thresholds
        run: python -m offline_eval.enforce_bars \
             .aicg/promptfoo-full.json evals/thresholds.yaml
      - name: Upload to Braintrust
        if: always()
        run: python -m eval.braintrust_upload \
             .aicg/promptfoo-full.json full

  nightly:
    if: github.event_name == 'schedule'
    runs-on: ubuntu-latest
    timeout-minutes: 90
    env: *fast-env
    steps:
      - uses: actions/checkout@v4
      - run: pip install -e . promptfoo braintrust
      - name: promptfoo (nightly, full + extended)
        run: promptfoo eval -c promptfooconfig.yaml \
             --tag full --tag extended \
             --output .aicg/promptfoo-nightly.json
      - name: File issue on any miss
        if: failure()
        run: python -m eval.open_issue \
             .aicg/promptfoo-nightly.json
      - name: Upload to Braintrust
        if: always()
        run: python -m eval.braintrust_upload \
             .aicg/promptfoo-nightly.json nightly
```

Design points:

- The `fast` job is a **required check** on the branch-protection rule for `main`. The `full` job is required when the `ready-for-review` label is applied. These are set in the repo's branch-protection config (reviewed on the same cadence as the threshold file).
- `concurrency.cancel-in-progress: true` kills stale runs on a push so authors don't wait on obsolete suites.
- The `full` job explicitly re-runs `enforce_bars` against `thresholds.yaml` even though promptfoo computed per-row verdicts. The reason: promptfoo's per-row assertions don't know about slice bars or no-regression bars. The runner is the source of truth on aggregate bars.
- Nightly failure opens an issue (via `eval.open_issue`); nightly misses do **not** retroactively block merges that already landed. The point is a rapid signal and an owner, not a retroactive gate.

### 2e. The two-engine cross-check (deliverable 5)

The fast lane runs against promptfoo; [[exercise-01]]'s `offline_eval.py` runs against the same fixtures on a parallel path. If the two engines disagree on any row's verdict, that is a CI failure in its own right.

```python
# eval/cross_check.py
import argparse
import json
import pathlib
import sys


def main(promptfoo_path: str, against: str) -> int:
    pf = json.loads(pathlib.Path(promptfoo_path).read_text())
    other = _run_other_engine(against)             # e.g., offline_eval.py
    pf_by_id = {r["vars"]["id"]: r["success"] for r in pf["results"]}
    other_by_id = {r["row_id"]: (r["verdict"] == "pass")
                   for r in other["rows"]}
    all_ids = sorted(set(pf_by_id) | set(other_by_id))
    disagreements: list[str] = []
    for row_id in all_ids:
        a, b = pf_by_id.get(row_id), other_by_id.get(row_id)
        if a is None or b is None:
            disagreements.append(f"{row_id}: present on only one engine")
            continue
        if a != b:
            disagreements.append(
                f"{row_id}: promptfoo={a} vs offline_runner={b}"
            )
    if disagreements:
        for d in disagreements:
            print(f"[CROSS-CHECK FAIL] {d}", file=sys.stderr)
        return 2
    return 0


if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("promptfoo_path")
    ap.add_argument("--against", default="offline-runner")
    args = ap.parse_args()
    raise SystemExit(main(args.promptfoo_path, args.against))
```

A disagreement is almost always one of three things, all of which are bugs worth fixing before the suite is trusted:

1. A row ID that landed in one engine's input and not the other (fixture layout drift).
2. A scorer that evaluates "contains_citation" loosely in one engine and strictly in another (scorer-parity drift).
3. A promptfoo assertion missing from the YAML (so the runner's extra bar passes silently). Fix: add the assertion.

The cross-check does not run on the full lane because the full lane *is* the offline runner (the `enforce_bars` step above). The cross-check is a fast-lane check only; its job is to keep the fast lane honest.

### 2f. Flake policy (deliverable 6)

Retries are a weapon; they should be used sparingly:

- A row may be retried **at most once** per CI run on a verdict mismatch against the pre-registered threshold. The retry runs with a different provider-side nonce (`PROMPTFOO_NONCE=${{ github.sha }}-retry`) to force a non-cached call where applicable.
- If both runs disagree, the row's final verdict is the **worse** of the two (safe-by-default).
- If both runs fail, the PR is blocked. If both succeed, the PR proceeds.
- A row that trips the retry path in three consecutive main-branch runs is tagged `flake_candidate: true` in Braintrust and the oncall applied-AI engineer owns either stabilising the row or moving it to `quarantine: true`.
- Judge-LLM rows are special: the `runs_per_scorer: 3` averaging from [[exercise-01]]'s `judge.lock` runs inside each attempt, so the retry already has within-attempt averaging. If the attempt-level verdicts still disagree, the row is a candidate for a *different-model judge* to break the tie — not a fourth same-judge run.
- Retries are **never** silent. Every retried row is logged with `attempt: 2` to the Braintrust experiment; a dashboard tracks retry volume per PR. A PR whose retry count exceeds 10% of rows is flagged for review — flakes shouldn't become the dominant cost of CI.

### 2g. Implementation notes

- **The fast lane's sample is pinned, not random.** The 10-row A-sample is listed by ID in `evals/samples/fast.yaml`. A random sample would mean each PR's verdict depends on which rows got picked, which defeats the point of the gate.
- **Both tools run against the same fixture library.** Promptfoo reads the YAML rows directly; the offline runner reads the same files. A fork between tool inputs is a second-order scorer drift source and should be forbidden.
- **The production controller is the SUT.** The promptfoo provider is a thin shim around `AgentController.from_defaults()`. A promptfoo "prompt test" that bypasses the controller tests the model, not the surface.
- **Braintrust owns the UI, promptfoo owns the gate.** The team reviews flagged rows in Braintrust; the merge button cares about the promptfoo exit code. Mixing the responsibilities (e.g., "look at Braintrust before merging") is an invitation for the gate to be bypassed.
- **CI secrets are scoped.** The Braintrust API key is a repo secret; the promptfoo config does not write to external services other than Braintrust. The provider shim does not include real customer credentials — the fixtures use synthetic principals (from [[exercise-02]]'s redaction pipeline).
- **The fast lane is truly fast.** If the fast lane creeps above 5 min, the first move is to shrink the A-sample, not to increase the timeout. The gate loses value as developers learn to tab away from it.
- **The nightly's extended set must not drift into the merge gate.** It is tempting to fold nightly bars into the full lane when they look stable. Resist: nightly bars exist because they are expensive, environmental, or flaky; folding them in produces a gate that intermittently fails for reasons unrelated to the PR.

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. `promptfooconfig.yaml` exists, declares the production controller as its provider, and tags rows for fast/full lane selection.
2. The three CI lanes (fast, full, nightly) exist with distinct triggers, distinct timeouts, and distinct enforcement rules.
3. The fast lane is a required check in the branch-protection rule; the full lane is required for `ready-for-review` PRs.
4. The cross-check step compares promptfoo results against the offline runner and exits non-zero on any disagreement.
5. Every CI run uploads to Braintrust with `commit`, `pr`, `lane`, and `baseline_commit` metadata; the Braintrust experiment URL is printed to the GitHub check summary.
6. The retry policy is at most one retry per row per run; a worse-of-two verdict is final; retry counts are visible in Braintrust.
7. The full lane re-runs `enforce_bars` against `evals/thresholds.yaml` so slice and no-regression bars are enforced.
8. Nightly failures open an issue; nightly bars do not retroactively block prior merges.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Three CI lanes with distinct triggers and scope | 1 | Fast, full, nightly all defined; fast ≤ 5 min target. |
| Promptfoo provider is a shim around the production controller | 1 | `provider_agent_controller.py` wraps `AgentController.from_defaults()`. |
| Braintrust upload includes PR, commit, lane, baseline_commit | 1 | All four tags present on every experiment. |
| Fast lane is a required branch-protection check | 1 | Documented in `evals/CI.md`; branch protection snippet. |
| Full lane re-runs `enforce_bars` against `thresholds.yaml` | 1 | Slice + no-regression bars enforced in full lane. |
| Two-engine cross-check between promptfoo and offline runner | 1 | Disagreement exits non-zero; three disagreement reasons listed. |
| Flake policy capped at one retry, worse-of-two final | 1 | Retry logic documented; retry counts visible in Braintrust. |
| Nightly failures open issues, not retroactively block | 1 | `eval.open_issue` step exists; no retroactive block. |
| Fast-lane sample is pinned by ID, not random | 1 | `evals/samples/fast.yaml` exists; reviewed quarterly. |
| Secrets scoped and no customer data in fixtures | 1 | Synthetic principals; Braintrust key is a repo secret. |

A submission scoring < 6 is rejected. The hard failures are (a) no production-controller provider (eval runs a parallel loop), (b) no `enforce_bars` in the full lane, (c) unlimited retries, (d) nightly bars conflated with gate bars, (e) random fast-lane sampling.

## 5. Common mistakes

- **Running promptfoo prompt-only.** Testing the raw prompt bypasses the controller, the retriever, and the tool loop. The provider shim is what makes promptfoo exercise the surface.
- **Relying on promptfoo assertions alone for aggregate bars.** Promptfoo scores per-row; it does not know about slice means or no-regression bars. The full lane must re-enforce bars via the offline runner.
- **Treating Braintrust as a second gate.** Mixing merge decisions between the YAML gate and the UI gate means neither is authoritative. Promptfoo gates; Braintrust reviews.
- **Unlimited retries.** Three-retry CI turns red PRs green by attrition. Cap at one; take the worse of two; escalate persistent flakes.
- **Folding nightly bars into full.** The gate then fails intermittently for environmental reasons; the team learns to re-run until it passes.
- **Random fast-lane sampling.** A PR's verdict depends on the sample, not the change. Pin the sample; review quarterly.
- **No cross-check between engines.** Promptfoo and the runner drift in scorer semantics; incidents slip through because each engine reports a different verdict on the same row.
- **Secrets in fixtures.** A redaction miss in a fixture is a leak with a long retention. Use synthetic principals end-to-end.
- **Fast lane slower than 5 min.** Developers context-switch away; the gate becomes advisory in practice. Shrink the sample before extending the timeout.
- **Braintrust upload gated on success.** Uploading only on green runs means the Braintrust history is biased toward passes; the UI loses its diagnostic value. Upload always.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the three-lane gate, the required-check branch-protection rule, and the Braintrust history link together implement the RMF's **MEASURE** function requirement that measurement results be auditable and the **GOVERN** function's separation of duties: the engine that gates the merge and the engine that reviews the runs are distinct.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the refusal rows and adversarial rows in the gate are the surface's answer to the input-manipulation items in the OWASP top 10; the 100% bar on those rows is encoded in the full lane's `enforce_bars` step.
- MITRE ATLAS — <https://atlas.mitre.org/> — new ATLAS technique entries that apply to the surface's threat model should land as new rows in the regression set from [[exercise-02]], which the fast lane runs on every PR.

Related exercises: [[exercise-01]] (pre-registered thresholds the full lane enforces); [[exercise-02]] (regression fixtures run in the fast lane); [[exercise-04]] (nightly feeds judge-drift signals back into the gate); [[exercise-05]] (persona dashboards read Braintrust experiment history).
