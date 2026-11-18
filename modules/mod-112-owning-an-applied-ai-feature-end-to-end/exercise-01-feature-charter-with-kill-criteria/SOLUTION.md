### exercise-01: Feature Charter With Kill Criteria — Solution

## 1. Solution overview

The capstone module asks the applied-AI engineer to *own* a feature end-to-end. Ownership starts before a single prompt is written, with a **feature charter** that the team can point at six months later and still agree on. The charter is the document that turns every subsequent artefact — the prompt registry from mod-102, the evals from mod-106, the guardrails from mod-107, the rollouts from mod-111 [[exercise-01]], the dashboards from mod-110 [[exercise-02]] — into decisions with a reason, not accumulated momentum.

This exercise is *not* a product-spec rewrite. The product spec says "what the feature should do." The charter says "under what conditions this feature should exist." The difference matters most when the feature is *working* but producing outcomes the team did not intend — the condition that NIST AI RMF labels as a GOVERN failure, where the system is operating correctly but the governance frame is stale.

The failure modes this solution defends against are concrete and recurring:

1. **A charter with success metrics but no kill criteria.** The team can see when the feature is winning but has no agreed-upon trigger for when to shut it off. Every regression becomes a renegotiation.
2. **Kill criteria stated as vibes.** "If it starts hallucinating badly" is unactionable. Kill criteria are numeric thresholds on metrics that already exist in production telemetry — otherwise, the on-call has no reading at 03:00 that fires the kill.
3. **Kill criteria that conflate "turn it off" with "roll it back."** The two levers are different: rollback restores a prior variant (mod-111 [[exercise-04]]), kill removes the surface (mod-111 [[exercise-01]], `support_agent.enabled = false`). Conflating them costs the team the four-level hierarchy and the "smallest lever first" discipline.
4. **In-scope / out-of-scope omitted.** Without an explicit list of what the feature is *not* for, scope creep lands on the applied engineer as "well, it already does X, can it also do Y?" The charter is where the team records the "no."
5. **No success *and* unintended-consequence metric.** A feature can hit its intended metric and still cause harm (e.g., a summarisation feature increases dashboard clicks but silently increases user-side correction edits). The charter must name an **anti-metric** alongside each success metric.
6. **Charter owned by nobody.** The charter is drafted, reviewed, filed. Nothing re-reads it until the feature is in trouble. The fix is a **review cadence** with a named owner and a **re-ratify condition** tied to meaningful change.
7. **Mapping to the AI governance frame deferred.** The team fills in the NIST AI RMF alignment "later." Later never arrives; the first audit finds the gap. The fix is a short, concrete mapping section embedded in the charter.

This solution provides:

- A **worked charter** for a reference feature (helpdesk reply assistant) with every section filled with real-shape content tied to upstream module artefacts.
- A **kill-criteria design pattern** — the structure of a usable kill criterion (metric, window, threshold, action, authority).
- A **decision rationale** explaining why each section exists and what failure mode it blocks.
- A **grading rubric** the instructor uses to score a student's charter.
- A **common-mistakes list** the instructor uses to pre-empt the frequent rewrites.

## 2. Worked answer or implementation

### 2a. The charter template (what the student must produce)

The charter is a single markdown file, `docs/features/<feature-id>/charter.md`, with these sections in this order:

```markdown
# Feature charter — <feature-id>

Owner: <applied-AI-engineer name + team>
Reviewers: <PM, design, safety, legal, data (as applicable)>
Status: DRAFT | RATIFIED | SUNSET
Ratified at: <YYYY-MM-DD>   Review cadence: <quarterly | monthly | per-incident>
Re-ratify triggers: <list — see 2c>

## 1. Problem statement
- Who the user is.
- What they are trying to do.
- What is broken today that this feature addresses.
- Why an LLM is the right tool (vs. search, deterministic code, or a human).

## 2. Scope — in / out
- IN: concrete examples of what the feature does.
- OUT: concrete examples of what the feature does NOT do, and the reason.

## 3. Success metrics (feature-level, numeric, measurable in prod)
- Primary: <metric name, source telemetry, target, window>
- Secondary: <same shape>
- Anti-metrics: <same shape — what we must NOT regress>

## 4. Kill criteria (numeric triggers for taking the feature off the surface)
Each row: Metric, Window, Threshold, Action, Authority.

## 5. Dependencies
- Upstream teams / services this feature depends on.
- Downstream surfaces this feature exposes to.

## 6. Risk & governance mapping
- NIST AI RMF function alignment (MAP / MEASURE / MANAGE / GOVERN).
- OWASP ML Top 10 risks considered relevant to this feature.
- MITRE ATLAS techniques considered relevant to this feature.

## 7. Review cadence & re-ratify triggers
- When this charter MUST be re-ratified (list of triggers).

## 8. Open questions
- Known-unknowns the team has NOT resolved yet.
```

### 2b. Worked charter — reference feature: `helpdesk.reply_assistant`

```markdown
# Feature charter — helpdesk.reply_assistant

Owner: Applied-AI team (lead: @a-eng-1)
Reviewers: PM-Support (@pm-s), Design (@d-1), Safety-Eng (@sec-1), Legal (@leg-1)
Status: RATIFIED
Ratified at: 2026-11-10
Review cadence: Quarterly (next: 2027-02-10) + per-incident
Re-ratify triggers: (a) any P0/P1 incident on this feature, (b) a change to the
  underlying base model's provider or model-ID, (c) addition of a new user
  cohort (locale, surface, authentication tier), (d) a change to the primary
  success metric definition.

## 1. Problem statement

- **User:** authenticated customers of our billing-support product, on web and
  in the native mobile app.
- **Job to be done:** get a concise, correct answer to a billing question
  (invoice totals, past charges, refund status) without waiting for a human
  support agent.
- **Status quo:** median first-response-time for billing tickets is multiple
  hours; 68% of billing questions are "status-of-X" lookups that do not need
  human judgement. Users leave the product while waiting.
- **Why an LLM:** the questions are highly variable in surface form but the
  underlying lookup is deterministic. Classic search exposes raw records; a
  deterministic UI would require pre-building hundreds of answer templates and
  would still fail on phrasing drift. An LLM over a RAG index of
  per-user billing documents converts the records into a conversational answer
  and absorbs phrasing variance.
- **Why *this* LLM lever:** the feature is read-only; no actions are taken on
  behalf of the user. Tool-calling is restricted to retrieval. This is the
  **lowest-blast-radius LLM shape** (mod-101 [[exercise-01]]) — correct for an
  early capstone deployment.

## 2. Scope — in / out

### IN
- Answer questions about the authenticated user's own billing data (invoices,
  payments, refunds, subscription state).
- Cite the specific invoice or payment record used to form the answer.
- Say "I don't have that information" when the answer is not derivable from
  the user's billing records in the last 24 months.

### OUT — and why
- **Taking action on the user's behalf** (issuing refunds, cancelling
  subscriptions): out. Tool-call blast radius is too wide for an initial
  deployment; action-taking is tracked separately as `helpdesk.actions` with
  its own charter.
- **Questions about other users' data**: out. Retrieval is scoped to the
  authenticated user's tenant; the system prompt forbids cross-user answers.
- **General billing policy questions** ("how does your refund policy work?"):
  out of *this* feature — handled by the existing docs-search product.
  Routing to docs-search is explicit; the LLM returns a static "see our
  policy" pointer rather than inventing policy text.
- **Legal advice / tax advice**: out. The system prompt refuses these
  categories (mod-107 [[exercise-02]] guardrail), and this charter records the
  category list as a governance decision — the guardrail is traceable back to
  a documented scoping decision, not an engineer's instinct.

## 3. Success metrics (numeric, measurable in prod)

| Role | Metric | Source | Target | Window |
|---|---|---|---|---|
| Primary | Containment rate = (turns resolved without human handoff) / (turns) | mod-110 [[exercise-02]] metrics, `app.feature="helpdesk.reply"` | ≥ 55% in 28-day trailing | 28d trailing, segmented by locale |
| Secondary | User thumbs-up rate | mod-110 [[exercise-03]] feedback join | ≥ 60% of turns with feedback | 28d trailing |
| Secondary | p95 end-to-end latency | mod-109 [[exercise-02]] SLO board | ≤ 4.0 s | 14d trailing |
| **Anti-metric** | Human-side correction rate: % of handed-off tickets where the human agent *contradicts* the AI reply | mod-110 [[exercise-03]] + support-agent tooling | ≤ 5% | 28d trailing |
| **Anti-metric** | Grounding-violation rate: % of replies that assert a specific number (amount, date, invoice ID) absent from the retrieved context | mod-107 [[exercise-03]] + mod-103 [[exercise-02]] citation audit | ≤ 1% | 7d trailing |

The anti-metrics exist because the primary metric is "containment," and the
easiest way to game it is to confidently hallucinate — containment rises
while trust drops. The anti-metrics give the team a reading on *how*
containment was earned.

## 4. Kill criteria

Each row follows this shape:

> **If** `<metric>` crosses `<threshold>` over `<window>` **then** `<action>`
> under the authority of `<role>`.

| # | Metric | Window | Threshold | Action | Authority |
|---|---|---|---|---|---|
| K1 | Grounding-violation rate (mod-107 [[exercise-03]]) | 1h rolling | > 5% | **Kill**: set `helpdesk_reply.enabled = false`; static fallback. | On-call applied-AI engineer; no further approval. |
| K2 | p95 latency (mod-109 [[exercise-02]]) | 1h rolling | > 15 s | **Kill**: set `helpdesk_reply.enabled = false`. | On-call applied-AI engineer. |
| K3 | Human-side correction rate (anti-metric) | 7d trailing | > 15% | **Scope down**: route out billing-dispute topic (narrow the "IN" list) + review within 72h. | PM + applied-AI lead. |
| K4 | Thumbs-up rate | 28d trailing | < 40% | **Scope review**: re-evaluate whether the feature meets its problem statement; possible SUNSET. | PM + applied-AI lead + director. |
| K5 | Safety-eval regression (mod-106 [[exercise-04]] safety suite) | pre-release | any new failure above policy threshold | **Block release**: candidate variant cannot leave shadow. | Safety-Eng. |
| K6 | Legal-risk trigger: a regulator inquiry or formal complaint attributed to AI-generated replies | per-incident | 1 | **Kill**: set `helpdesk_reply.enabled = false`; incident mode. | Legal + director. |
| K7 | Base-model deprecation announced by provider (mod-108 [[exercise-04]]) | per-announcement | announcement date | **Re-ratify charter**: evaluate replacement model against current frozen eval and re-decide feature viability. | Applied-AI lead. |

Note the distinction, which the exercise's rubric scores explicitly:

- **K1, K2, K6 are "kill"** — remove the feature from the user surface entirely.
- **K3, K4 are "scope-down" / "review"** — the feature stays on but its
  footprint narrows.
- **K5 is a pre-release gate** — never ships.
- **K7 is a re-ratification trigger** — the charter itself goes back through
  review because an upstream assumption changed.

The authority column names *who* is allowed to pull the lever. Nothing in
the kill-criteria list reads "escalate to the team"; every row names a
specific role that can act. This is the governance property NIST AI RMF
MANAGE requires: the lever exists, and there is a named human who can pull it.

## 5. Dependencies

- **Upstream:** billing-service API (source of record for invoices /
  payments), mod-103 [[exercise-01]] retriever, mod-108 [[exercise-02]] model
  gateway, mod-107 [[exercise-03]] PII scrubber, mod-111 [[exercise-01]] flag
  layer, mod-110 [[exercise-01]] tracing.
- **Downstream surfaces:** web helpdesk widget, native-mobile helpdesk tab.
- **Shared ownership:** mod-108 [[exercise-04]] rollback rehearsal is shared
  with the ML-platform team; the applied-AI team runs the feature-side gate,
  the ML-platform team runs the gateway-side fallback.

## 6. Risk & governance mapping

### NIST AI RMF alignment

- **MAP (1.0, 3.0):** Section 1 (problem statement) and Section 2 (in / out
  scope) satisfy MAP by establishing context and purpose. The explicit OUT
  list is the artefact that satisfies MAP-3.4 (characterise AI technology
  benefits vs. non-AI alternatives).
- **MEASURE (2.0, 3.0):** Section 3 (success + anti-metrics, tied to
  production telemetry that already exists per mod-110) satisfies the
  "metrics tracked across the operating envelope" requirement.
- **MANAGE (2.0, 4.0):** Section 4 (kill criteria with named authorities) is
  the direct MANAGE artefact. The levers map to mod-111's four-flag hierarchy.
- **GOVERN (1.0, 5.0):** Section 7 (review cadence) + the Status / Ratified-at
  / Re-ratify triggers header make the charter *itself* a governable
  artefact.

### OWASP Machine Learning Security Top 10 — considered relevant

- **ML01 (Input Manipulation)**: present. Mitigation lives in mod-107
  [[exercise-01]] (adversarial prompt handling).
- **ML02 (Data Poisoning)**: considered low-relevance — retrieval index is
  derived from internal billing records, no third-party data path.
- **ML09 (Output Integrity)**: present. Mitigation is the grounding-violation
  anti-metric (Section 3) + K1 kill criterion.

OWASP items considered and classified as out-of-scope for this feature are
listed explicitly so the next reviewer sees the reasoning, not the omission.

### MITRE ATLAS — considered relevant

- **AML.T0051 (LLM Prompt Injection)**: input layer handled by mod-107
  [[exercise-01]]; retrieved-context layer handled by mod-103 [[exercise-04]]
  query-rewriting filter.
- **AML.T0053 (LLM Plugin Compromise)**: not applicable; feature uses no
  write-side tools, only retrieval.

## 7. Review cadence & re-ratify triggers

The charter is re-ratified quarterly on a fixed date. Outside that cadence,
the following events force an immediate re-ratification:

- Any P0 or P1 incident on this feature.
- A change in the underlying base model (provider-side or our own routing
  decision in mod-108 [[exercise-02]]).
- A new user cohort added (new locale, new surface, new authentication tier).
- A change to the primary success metric's definition or source telemetry.
- A finding from the mod-106 [[exercise-04]] safety suite that moves the
  feature's safety posture.
- A regulator inquiry or formal complaint (K6).
- Base-model deprecation announcement (K7).

## 8. Open questions

- The containment target (55%) is set from the pre-launch estimate; it may
  move after the first 28d of real traffic. Re-visit at the first quarterly
  review.
- Locale support beyond `en` and `es` is not scoped yet; adding `ja` or `fr`
  would require re-ratification (new cohort).
- The anti-metric for "human-side correction rate" relies on a human agent
  annotation that is not yet standardised. Work tracked in mod-110
  [[exercise-03]] feedback-schema evolution.
```

### 2c. Kill-criteria design pattern

Every kill criterion is a tuple:

1. **Metric** — must be measurable in production telemetry *that already exists*. If the criterion requires a dashboard that is not yet built, the first corrective task is "build the dashboard" — the kill criterion is unarmed until then.
2. **Window** — the rolling window the metric is aggregated over. Kills on short windows (minutes to hours) imply high-confidence, low-noise metrics. Reviews on long windows (days to weeks) implicitly accept noise.
3. **Threshold** — a numeric cut-off. Qualifiers like "high" or "sudden" are disallowed; they do not fire alerts.
4. **Action** — one of: **KILL** (feature off), **SCOPE-DOWN** (narrower IN list), **REVIEW** (scheduled re-ratification), **BLOCK-RELEASE** (pre-shipment gate). These map 1:1 to the four containment levers the applied engineer already owns via mod-111 [[exercise-01]].
5. **Authority** — a named role with the standing to pull the lever. "The team" is not an authority. Un-named authority on a kill criterion means the kill never fires.

A kill criterion that is missing any of these five fields is not ratified.

### 2d. Re-ratify triggers — what forces the charter back through review

The re-ratify triggers list is the charter's defence against staleness. Treat it as a list of *upstream-contract changes* that break charter assumptions:

- A model swap changes the quality baseline.
- A new cohort changes the "who" in Section 1.
- A metric-definition change silently changes every threshold in Section 4.
- A safety finding changes the governance risk posture.
- A regulator event changes the legal frame.

When any trigger fires, the charter's `Status` returns to `DRAFT` until the review completes. The feature itself can keep running; what cannot keep running is the *claim* that the charter still governs the feature.

## 3. Validation steps

A reviewer validates a submitted charter by executing these checks in order. Failing any step is a rejection.

1. **All eight sections are present.** A missing section is a hard failure. The template exists specifically to prevent omission.
2. **Problem statement names a specific user, a specific job, and why an LLM is the right lever.** "Users want AI" is not a problem statement.
3. **In / Out scope has at least three concrete items on each side, each with a *reason* for the categorisation.** "Out: legal advice" without a reason is incomplete; "Out: legal advice — the correctness bar requires licensed-counsel review, which this feature cannot provide" is complete.
4. **Primary metric, secondary metric, and *at least one* anti-metric are present and sourced from telemetry that exists in the mod-110 stack.** A metric that requires a new pipeline is deferred; the charter cannot cite a metric that is not readable on the day of ratification.
5. **Every kill criterion has all five fields.** Metric / Window / Threshold / Action / Authority. Grep the kill-criteria table — empty cells are a hard failure.
6. **At least one kill criterion names an "anti-metric" threshold, not only a success-metric threshold.** This is the test for "can you game the primary metric?"
7. **Kill and scope-down are distinguished.** At least one entry in the Section 4 table is a KILL (feature off) and at least one is SCOPE-DOWN or REVIEW. A table where every row says "kill the feature" is under-specified; a table where no row says "kill the feature" has no actual kill criterion.
8. **Dependencies are listed by *name*, not category.** "Retrieval service" is not a dependency; "mod-103 [[exercise-01]] retriever, version pinned to v4.2" is.
9. **NIST AI RMF alignment names at least three of MAP / MEASURE / MANAGE / GOVERN with the *specific sub-function* satisfied by a *specific section of this charter*.** Hand-waving at NIST without naming which section does the work is a common failure.
10. **At least one OWASP ML Top 10 entry and at least one MITRE ATLAS technique are named with mitigations that reference specific upstream module artefacts.** "We considered OWASP" without naming items and mitigations is a hard failure.
11. **Re-ratify triggers list has at least four items.** Fewer than four means the author treated this as "we'll just re-review quarterly" — exactly the governance failure the triggers list defends against.
12. **Status / Ratified-at / Review cadence are set.** A charter in `DRAFT` status cannot be used to score the feature.

## 4. Rubric or review checklist

Instructor rubric (10 points total). An AI feature cannot ship until its charter scores ≥ 7 and has zero hard failures.

| Criterion | Points | What earns the points |
|---|---|---|
| Problem statement names user, job, status quo, and justifies the LLM lever | 1 | All four sub-fields present; the "why LLM" argument is concrete. |
| In-scope and Out-of-scope lists each have ≥ 3 concrete items with reasons | 1 | The reasons tie each OUT to a specific risk, dependency, or non-goal — not "we just don't want to do that yet." |
| Success metrics are numeric, sourced from existing production telemetry | 1 | Target value, window, and `app.feature` label are all stated. |
| At least one anti-metric is defined and tied to a plausible way the primary metric could be gamed | 1 | The anti-metric should prevent a known failure (grounding violations, human-side correction). |
| Kill criteria table has ≥ 5 rows, every row has all five fields | 2 | Metric / Window / Threshold / Action / Authority — all five, every row. |
| Kill criteria include at least one KILL and at least one SCOPE-DOWN or REVIEW | 1 | The distinction between "feature off" and "feature narrowed" exists. |
| NIST AI RMF alignment names specific sub-functions and ties each to a charter section | 1 | MAP / MEASURE / MANAGE / GOVERN; at least three are mapped explicitly. |
| OWASP ML Top 10 + MITRE ATLAS mapping includes at least one applicable item each with a specific mitigation | 1 | The mitigation references an upstream module artefact or an existing team process. |
| Review cadence + re-ratify triggers list has ≥ 4 trigger types | 1 | Model change, cohort change, metric-definition change, safety finding, incident, regulator event. |

**Hard failures (automatic rejection regardless of score):**

- Kill criteria that are not numeric (vibes-level thresholds).
- No anti-metric (primary metric can be gamed by hallucination).
- No named authority on at least one kill criterion.
- Problem statement that does not name a specific user (uses "users" generically).
- Charter in `DRAFT` status (the submission is incomplete; nothing to grade).

## 5. Common mistakes

- **Charter is a restatement of the product spec.** The product spec says what the feature does; the charter says under what conditions it should exist. If the two documents are interchangeable, the charter is not doing its job.
- **Metrics without an anti-metric.** Every primary metric can be gamed. Containment can be gamed by hallucination; thumbs-up can be gamed by sycophancy. Name the gaming path and the anti-metric that catches it.
- **Kill criteria stated without authority.** "If the error rate spikes, kill the feature" — but *who* kills it? If no role is named, the kill never fires at 03:00.
- **Kill = rollback.** Setting `enabled = false` is not the same as reverting to a prior prompt. Use mod-111 [[exercise-01]]'s four-flag hierarchy and pick the *smallest* lever. The charter should name both options when appropriate.
- **"We considered NIST AI RMF" without specifics.** A line that says "aligned with NIST AI RMF" with no sub-function mapping fails every audit. Name the function, name the sub-function, name the charter section that satisfies it.
- **OUT list reads like a wish-list.** "We won't do X yet" is not an OUT. A proper OUT names a reason ("blast radius too wide for initial deployment," "requires licensed-counsel review").
- **Dependencies listed by category, not instance.** "A retriever" is not a dependency; "mod-103 [[exercise-01]] retriever, version v4.2, pinned" is. Categories do not page; instances do.
- **Re-ratify triggers only include "quarterly."** Quarterly cadence catches drift; triggers catch change. Both are required.
- **Open questions omitted.** Every charter has open questions. A charter with an empty Open Questions section is either overconfident or dishonest.
- **Charter ratified without the safety engineer as a reviewer.** For any feature on the LLM lever, Safety-Eng is a required reviewer (mod-107 [[exercise-01]]). Missing them means Section 6 (OWASP / ATLAS mitigations) has not been audited.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the MAP / MEASURE / MANAGE / GOVERN function model is the backbone of Section 6. GOVERN in particular requires named authorities for every corrective lever — the Authority column in the kill-criteria table is the direct response.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the Section 6 OWASP mapping draws from this list. ML01 (Input Manipulation) and ML09 (Output Integrity) are the items most often applicable to feature-side LLM deployments; the charter records which items were considered, not only which were accepted.
- MITRE ATLAS — <https://atlas.mitre.org/> — the Section 6 ATLAS mapping draws from the techniques catalogue. AML.T0051 (LLM Prompt Injection) is the most common applicable technique; the mitigation-trace in the charter must name the module artefact that handles it.

Related exercises: [[exercise-02]] (the incident-to-regression-fixture walkthrough extends K1 / K2 into a concrete regression check-in for the feature), [[exercise-03]] (the build-vs-buy matrix references the charter's dependency list as its input), [[exercise-04]] (the feature card slice re-uses Sections 3 / 5 / 6 as its core), [[exercise-05]] (delegation contracts formalise the Dependencies list in Section 5); mod-101 [[exercise-01]] (feature-shape decision rubric that pre-dates the charter), mod-106 [[exercise-04]] (safety eval suite feeding K5), mod-107 [[exercise-03]] (PII scrubber and grounding-violation metric feeding K1), mod-108 [[exercise-02]] (gateway providing model-ID observability for K7), mod-109 [[exercise-02]] (SLO board feeding K2), mod-110 [[exercise-02]] (feature-health dashboard sourcing primary and anti-metrics), mod-111 [[exercise-01]] (flag hierarchy providing the kill / scope-down levers named in Section 4).
