### exercise-05: Cross-Persona Feature Dashboards — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to design the **dashboards** for the helpdesk-reply-copilot feature — not one dashboard, but a family of them, one per stakeholder persona. The same eval + online data from [[exercise-01]] through [[exercise-04]] is pivoted differently for the applied-AI oncall, the product manager, the executive sponsor, and (where applicable) the customer-support ops lead. Each persona has distinct decisions to make and distinct patience for complexity. A single "dashboard for everyone" either overloads the exec or starves the oncall.

The failure mode the exercise is written to catch is the one where a team stands up a Braintrust board or a Grafana panel, shows it in standup, and then nobody can answer the question each persona actually brings: *is this feature on track against its SLOs, who is failing, what are we about to do about it, and what is the rolling cost*. A dashboard that doesn't answer a named question for a named person is wall art.

This solution provides:

- A **persona list** with the decisions each persona makes and the questions each one brings to a dashboard check-in.
- **Four dashboard specs**, one per persona, each with a tile list, data source, refresh cadence, and bottom-3 "what's broken / what to do" callout.
- A **data-contract document** that tells the dashboards where each number comes from (offline scorecard, online worker gauges, Braintrust experiments, incident tracker) — so a dashboard tile with no contract is a bug.
- A **review cadence** for each persona and the human responsible for refreshing the dashboard definitions when the feature changes.
- A **grading rubric** for the instructor.

The lesson is that a dashboard is an artefact of a decision, not a decoration. The exec cares about SLO conformance + spend + the bottom three incidents; the applied-AI oncall cares about which row ids have gone red in the last 24h; the PM cares about which personas / tenants are getting a good or bad experience; the ops lead cares about escalations into their queue. Four distinct pivots of the same data, each with a named question and a named owner.

## 2. Worked answer or implementation

### 2a. Persona list (deliverable 1 — decision rationale)

| Persona | Named decision they make | Question they bring to the dashboard | Latency tolerance |
|---|---|---|---|
| **Applied-AI oncall** | "Do I need to page / roll back / open a fixture PR today?" | "Which rows regressed; what is the current eval verdict; what did the drift monitor say?" | Minutes. |
| **Product manager** | "Is the feature serving its intended persona, and do I need to steer the next sprint?" | "How does the feature score **by user persona**, by tenant, by scenario? Where are the thumbs-downs clustered? What are the top-3 complaints?" | Hours to a day. |
| **Executive sponsor** | "Is this feature on track against the business SLOs, and how much does it cost?" | "Does the feature meet its SLO this quarter; what are the trailing 90 days of incidents; what is the rolling cost; is the trajectory green or red?" | Weekly. |
| **Customer-support ops lead** | "Is my team being asked to escalate at the expected rate; are there surprises in the pause reasons?" | "What is the escalation rate; what is the pause-reason mix; what fraction of sessions ended in a human takeover?" | Daily. |

Each persona gets its **own** dashboard. The specs below are the per-persona pivots; the underlying data is shared.

### 2b. Dashboard specs (deliverable 2)

#### 2b.i Applied-AI oncall dashboard

Named question: *What's broken right now, and what do I do about it?*

Refresh: 60 s. Tile count: 6. Owner: applied-AI oncall rotation.

| Tile | Metric | Source | Threshold / annotation |
|---|---|---|---|
| 1. Red rows — last 1 h | Count of rows where `online_eval.score` < pre-registered bar, grouped by `shape`. | Online worker from [[exercise-04]]. | Red if > 0 in the Shape-D (safety) bucket; yellow if > 10 anywhere. |
| 2. Judge drift — current | `judge_drift.mean_abs_delta` + `judge_drift.max_abs_delta`. | Drift monitor from [[exercise-04]]. | Red if above the tolerance from `judge.lock`. |
| 3. Fast-lane CI pass rate — last 24 h | Fraction of fast-lane CI runs green on `main`. | GitHub Actions + the promptfoo result artefacts from [[exercise-03]]. | Red if < 95%. |
| 4. No-regression row verdicts | Per-row pass/fail for every row in `thresholds.no_regression_rows`. | Nightly full-lane scorecard. | Red if any row fails. |
| 5. Fixture-review queue depth | Count of traces flagged `online_judge_below_bar` awaiting triage. | `TraceStore.flag_for_review` queue from [[exercise-04]]. | Red if > 20; link to the triage CLI from [[exercise-02]]. |
| 6. Top-5 red rows + "what to do" | For each: row id, incident link, oncall action (one of: open fixture PR, re-run, escalate). | Composite of 1 + 5. | Each row is actionable within 30 min. |

Design points:

- Every red tile links to the **action**, not just the number. Tile 6 is the critical one: it names the next move per row.
- The oncall dashboard has **no quarter-long trends**. A 1 h window and a 24 h window are enough; the exec dashboard owns the quarter view.
- Tile 4 is why pre-registered thresholds matter: the oncall doesn't need to argue about bars at 3 AM; the bar is in `thresholds.yaml`.

#### 2b.ii Product-manager dashboard

Named question: *Which personas / tenants are getting a good or bad experience, and where should the next sprint focus?*

Refresh: 1 h. Tile count: 7. Owner: PM.

| Tile | Metric | Source | Threshold / annotation |
|---|---|---|---|
| 1. Task-rubric mean by user-persona slice | Rolling-7-day mean, grouped by `user_persona` tag on the request. | Online worker. | Annotated vs. global bar. |
| 2. Task-rubric mean by tenant class | Enterprise vs. SMB; p50 + p10. | Online worker. | Enterprise bar tighter than SMB. |
| 3. Thumbs-down heatmap | Thumbs-down rate by (user_persona × scenario), last 30 days. | Feedback store. | Hot cells highlighted. |
| 4. Top-3 complaints | Clustered user comments on thumbs-down, with a sample quote per cluster. | Feedback store + clustering job. | Refreshed daily. |
| 5. Prompt-version adoption | Fraction of traffic on each live prompt version. | Online worker (`prompt_version` tag). | Annotated when > 5% is on a non-canonical version. |
| 6. Fixture coverage by persona | For each `user_persona`, count of offline rows tagged with that persona. | Offline fixture set. | Any persona with < 5 rows is yellow. |
| 7. Change log (last 30 d) | Feature-impacting changes: prompt edits, model version swaps, threshold changes. | Repo + deploy log. | Annotated on all time-series tiles. |

Design points:

- Tile 7 (change log) is the diagnostic bridge: when a line moves on tile 1 or 3, the PM can correlate to a specific change.
- Tile 6 makes "we have no coverage for persona X" visible to the PM who owns the backlog. Without it, the dashboard's slices lie by omission.
- The PM dashboard does not include dollar cost or incident counts — those belong on the exec dashboard. Keep the PM view focused on experience + coverage.

#### 2b.iii Executive dashboard

Named question: *Is this feature on track against the business SLOs, how much does it cost, and what's the trajectory?*

Refresh: 24 h. Tile count: 5. Owner: engineering director; reviewed with sponsor weekly.

| Tile | Metric | Source | Threshold / annotation |
|---|---|---|---|
| 1. SLO conformance (trailing 90 d) | Fraction of days where all pre-registered global bars held online. | Online worker + `thresholds.yaml`. | ≥ 95%. |
| 2. Incident count by severity (trailing 90 d) | SEV-1/2/3 counts. | Incident tracker. | Target: ≤ 1 SEV-1 / quarter. |
| 3. Rolling monthly cost | $ per month on inference + eval + judge + drift monitor. | Cost system. | Target annotated; trend highlighted. |
| 4. Rolling monthly spend per served request | Cost / served request, with p50 + p95. | Composite. | Target annotated. |
| 5. Trajectory summary (bottom-3) | Three bullets: largest regressions, biggest wins, one risk or ask. | Hand-authored by applied-AI lead. | Updated weekly, before sponsor review. |

Design points:

- The exec dashboard is intentionally small. Five tiles. Nothing else.
- Tile 5 is hand-authored. A dashboard without a written summary of the trajectory is a graph without a conclusion. The applied-AI lead writes it; the engineering director approves it; the sponsor reads it in the weekly review.
- Trailing-90-day windows prevent the exec from chasing a one-day blip.

#### 2b.iv Customer-support ops lead dashboard

Named question: *Is my team being asked to escalate at the expected rate, and are pauses behaving the way I expect?*

Refresh: 1 h. Tile count: 5. Owner: support ops lead; reviewed with applied-AI oncall weekly.

| Tile | Metric | Source | Threshold / annotation |
|---|---|---|---|
| 1. Escalation rate (sessions → human takeover) | Last 7 days, per shift. | Online worker + ticket system. | Annotated vs. monthly baseline. |
| 2. Pause-reason mix | Distribution of pause reasons (e.g., `confirm_write`, `low_confidence`, `policy_block`). | Online worker (`pause_reason` tag). | Annotated when a reason's share moves > 10 pp. |
| 3. Human-override rate by pause reason | Of pauses that reached a human, what fraction was approved / denied / modified. | Pause-store audit log. | Target: low denial rate; high denial means over-pausing. |
| 4. Top-5 scenarios driving escalation | Scenario clusters with the highest escalation rate. | Feedback + ticket system. | Clickable to a sample of linked tickets. |
| 5. Shift-level load projection | Expected escalations for the next 24 h based on the last 7 d. | Online worker + staffing model. | Red if > 1.5× the staffed capacity. |

Design points:

- Tile 5 is the one that makes the ops lead pay attention — it is the only dashboard surface that feeds the staffing model. Without it, the ops dashboard is a retrospective, not a planning tool.
- Tile 3 catches the over-pausing failure mode: a surface that pauses too much turns the "human-in-the-loop" from a safety net into a bottleneck. The denial rate on pauses is the leading indicator.

### 2c. Data-contract document (deliverable 3)

Every tile has a named data source. The contract makes this visible so a tile with no contract is a bug.

```yaml
# dashboards/data_contract.yaml
dashboards:
  oncall:
    tiles:
      1_red_rows_last_1h:
        source: "metric:online_eval.score"
        grouping: ["shape"]
        window: "1h"
        bar_source: "evals/thresholds.yaml"
      2_judge_drift:
        source: "metric:judge_drift.mean_abs_delta, metric:judge_drift.max_abs_delta"
        bar_source: "evals/judge.lock"
      3_fast_lane_pass:
        source: "github_actions:evals/fast"
        window: "24h"
      4_no_regression:
        source: "artifact:offline-scorecard (nightly)"
        extracts: "rows[].verdict where row.source.kind == 'trace_snapshot'"
      5_fixture_review_queue:
        source: "trace_store:flag_for_review"
        state: "open"
      6_top5_red_actions:
        source: "composite:1,5 + runbook:rows-triage.md"
  product:
    tiles:
      1_rubric_by_persona:
        source: "metric:online_eval.score"
        grouping: ["user_persona"]
        window: "7d"
      ...   # (abridged)
  executive:
    ...
  ops:
    ...
```

Rules:

- A dashboard tile that is not covered by the contract is a lint failure: the dashboard-definition linter diffs the tile list against the contract and refuses to publish if there is a tile without a contract entry.
- Data sources are restricted to: `metric:*`, `artifact:*`, `github_actions:*`, `trace_store:*`, `cost_system:*`, `incident_tracker:*`. New source types require a contract-schema PR.
- Every source has a window + aggregation documented — "task-rubric score" without a window is ambiguous.

### 2d. Review cadence (deliverable 4)

| Dashboard | Human owner | Review cadence | What gets reviewed |
|---|---|---|---|
| Oncall | Applied-AI oncall rotation | On shift handoff | Current tile accuracy; action links; new runbooks needed. |
| Product | PM | Weekly | Which slices show regression; whether tile 6 "coverage by persona" is pointing at a backlog gap. |
| Executive | Engineering director | Weekly before sponsor review | Tile 5 trajectory summary rewritten; SLO conformance confirmed. |
| Ops | Support ops lead + applied-AI oncall | Weekly | Escalation rate vs. staffing; pause-reason mix stability. |

Every dashboard also has a **quarterly definition review** where the tile list itself is revisited. Dashboards that haven't changed in a year are either (a) stable and still useful or (b) wall art — the quarterly review is where this is checked.

### 2e. Implementation notes

- **One dashboard per persona, four personas, four dashboards.** Collapsing them produces a multi-tab monolith that no persona reads. Separating them makes each owner responsible for their view.
- **Every tile has a named decision.** A tile that does not contribute to a decision the persona is making is removed. "Nice to know" tiles are wall art.
- **Executive dashboard has a hand-authored summary.** Numbers without a conclusion are a report. The weekly summary is the forcing function that the trajectory is being read, not just displayed.
- **Change log is a tile, not a banner.** Correlating "when the regression started" to "what shipped when" is 80% of a product-side regression discussion.
- **Ops dashboard owns the staffing signal.** Dashboards that don't feed an operational decision slide to the bottom of the stack. Tile 5 is why the ops lead opens the page.
- **Cost belongs on the exec board, not the oncall board.** Cost is a lagging signal; the oncall cares about the next 30 minutes. Keep the pivots focused.
- **Dashboards read the contract, not the raw metrics.** When a metric moves locations, only the contract changes; the dashboard definitions are stable.
- **No per-tenant dashboards unless a tenant demands SLAs.** Per-tenant views multiply the surface area of what to maintain. Build them when a specific tenant has a contract clause requiring them — not speculatively.
- **The persona list is reviewed when the feature changes.** When a new stakeholder emerges (e.g., trust & safety), add a persona and a dashboard. Do not retrofit tiles into an existing persona's board.
- **Dashboard definitions are source-controlled.** Grafana JSON, Braintrust layouts, whatever renderer — the definitions live in the repo under `dashboards/<persona>/`. A tile change that is not in a PR is not a tile change.

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. Four personas are named, each with a decision and a question.
2. Four dashboard specs exist with tile lists; each tile names its data source, aggregation, and window.
3. `dashboards/data_contract.yaml` covers every tile; the dashboard-definition linter refuses to publish uncontracted tiles.
4. Oncall dashboard includes a "what to do" tile linking each red row to a next action.
5. PM dashboard includes a fixture-coverage-by-persona tile.
6. Executive dashboard has ≤ 5 tiles and includes a hand-authored weekly trajectory summary.
7. Ops dashboard includes a staffing-projection tile keyed to the pause-reason mix.
8. Review cadences are documented per persona + a quarterly definition review.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Four distinct personas with named decisions | 1 | Each persona has a decision, a question, a latency tolerance. |
| Four dashboard specs, each with ≤ 7 tiles | 1 | No monolith; no persona's board exceeds 7 tiles. |
| Oncall board includes a "what to do" action tile | 1 | Every red row has a next action. |
| PM board includes fixture-coverage-by-persona | 1 | Gap-pointing tile is visible to the backlog owner. |
| Executive board is ≤ 5 tiles + hand-authored summary | 1 | Hand-authored weekly trajectory; no auto-generated summary. |
| Ops board includes a staffing-projection tile | 1 | Operational decision is on the surface. |
| `dashboards/data_contract.yaml` covers every tile | 1 | Linter blocks uncontracted tiles. |
| Change-log tile in PM dashboard | 1 | Correlation with shipped changes is possible. |
| Review cadences documented per persona | 1 | Weekly + quarterly definition review. |
| Dashboard definitions in source control | 1 | PR-reviewable; `dashboards/<persona>/` structure. |

A submission scoring < 6 is rejected. The hard failures are (a) a single "universal" dashboard, (b) tiles without a data-contract entry, (c) an exec dashboard without a hand-authored summary, (d) an ops dashboard with no staffing signal, (e) dashboards outside source control.

## 5. Common mistakes

- **One dashboard for everyone.** The exec reads it once and stops; the oncall can't find the red rows. Separate per persona.
- **Numbers without a decision.** A tile that does not contribute to a named decision is wall art. Remove it; repopulate from questions.
- **Exec dashboard overloaded.** Seven tiles become fifteen become thirty. Cap it; move depth to the PM or oncall board.
- **Oncall board with quarter-long trends.** The oncall doesn't need the quarter view; they need "what went red in the last hour." The exec board owns trends.
- **No change-log overlay.** Moves in the time series are uncorrelated to shipped changes; every regression is a mystery.
- **Tiles with ambiguous windows.** "Task-rubric score" without a window could be 1 h or 90 d. The contract fixes this.
- **Staffing signal missing from ops.** The ops lead has no operational reason to open the page; the dashboard becomes a retrospective.
- **Hand-authored summary skipped on the exec board.** Numbers without a conclusion are a report, not a signal. The summary is the forcing function.
- **Dashboard definitions out of source control.** A tile change that is not in a PR is not a tile change; silent drift between personas' views is a re-occurring debugging cost.
- **Per-tenant dashboards built speculatively.** The surface area explodes; nobody maintains them. Build them only on demand.
- **Judge-drift signal buried.** The oncall board must surface the drift monitor's current status; otherwise online regressions are always ambiguous about whether the judge moved.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the four persona dashboards + the data contract + the review cadence implement the RMF's **MAP** function (stakeholders and their decisions are explicit) and **MANAGE** function (continuous, auditable observation of the deployed system, with named owners per view).
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the oncall board's red-rows tile and the ops board's pause-reason mix are the surface's operational read on the safety and output-integrity posture, pivots of the same telemetry the OWASP ML top 10 recommends monitoring for.
- MITRE ATLAS — <https://atlas.mitre.org/> — ATLAS technique entries that match an incident on the exec board's tile 2 (incident count) should be labelled with the ATLAS id in the incident tracker, so the quarterly dashboard-definition review can cross-reference emerging adversarial patterns against the dashboards' coverage.

Related exercises: [[exercise-01]] (bars the oncall board reads); [[exercise-02]] (fixture queue surfaced on the oncall board); [[exercise-03]] (CI pass rate on the oncall board; experiment history behind the PM board); [[exercise-04]] (online + drift metrics behind the oncall, PM, and ops boards).
