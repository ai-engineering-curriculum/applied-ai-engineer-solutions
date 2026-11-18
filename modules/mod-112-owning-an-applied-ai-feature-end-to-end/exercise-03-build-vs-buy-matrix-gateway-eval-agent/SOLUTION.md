### exercise-03: Build-vs-Buy Matrix — Gateway, Eval Platform, Agent SDK — Solution

## 1. Solution overview

The capstone module forces a decision the applied-AI engineer has deferred for most of the curriculum: for the **three load-bearing pieces of substrate under every feature** — the LLM gateway (mod-108), the eval platform (mod-106), and the agent SDK / orchestration layer (mod-104) — what does the team build in-house and what does it buy? This exercise asks the student to produce a defensible **build-vs-buy matrix** for each of the three, with the decision framed against the feature charter from [[exercise-01]] (not against a greenfield team or a greenfield wallet).

The exercise is not a shopping list. The deliverable is a decision rubric *and* the three decisions that come out of that rubric when the rubric is applied with this team's constraints. A valid submission can conclude "we build all three," "we buy all three," or any mix — what matters is whether the reasoning is explicit, falsifiable, and tied to upstream artefacts.

The failure modes this solution defends against are concrete:

1. **Decision by vibes.** "We'll just build it" or "we'll just use LangChain" with no criteria; the first moment the choice is under pressure the team re-litigates it.
2. **Decision by sticker price.** Buy decision pinned to monthly cost without integration cost, lock-in cost, or deprecation cost.
3. **Confusing "open source" with "free."** An OSS component the team runs in-house is a *build* on the operations axis — somebody oncalls it, patches CVEs, and absorbs breaking releases.
4. **Confusing "managed" with "no-lock-in."** A managed service's data model, feature API, and pricing are all lock-in surfaces; the matrix must name them.
5. **No exit criteria.** The team buys the component, then never defines what would prompt switching. Lock-in becomes permanent by default.
6. **Build decision without an owner.** Building substrate requires a person or team willing to carry it; without a named owner, "build" is a wish, not a plan.
7. **One matrix for all three substrates.** The three are different; a single matrix produces mushy answers. Each substrate gets its own set of criteria with weights tuned for that substrate.
8. **Decision made without the charter's kill criteria in hand.** If the charter from [[exercise-01]] names a 1-hour grounding-violation kill criterion, the eval platform must be able to run a candidate variant inside that window. "Can this tool meet the kill-criteria SLA?" is a required column.

This solution provides:

- A **shared decision rubric** — seven criteria that every build-vs-buy question is scored against.
- A **per-substrate matrix** for the three substrates, with the criteria weights tuned per substrate.
- A **worked decision** for each substrate, in the voice of an applied-AI engineer explaining the call to a staff-level reviewer.
- A **set of exit criteria** — the conditions that would re-open each decision.
- A **grading rubric** scoring whether the student's matrix is defensible.
- A **common-mistakes list** covering the frequent reasoning gaps.

## 2. Worked answer or implementation

### 2a. Shared decision rubric — the seven criteria

Every build-vs-buy decision in this exercise is scored against these seven criteria. Each is scored for *both* the build and the buy option on a 1–5 scale; the differential (build − buy) is what drives the decision. Weights are tuned per substrate (see 2b–2d).

| # | Criterion | What "build" wins on | What "buy" wins on |
|---|---|---|---|
| C1 | **Fit to charter** — does the option support the specific requirements in the feature charter (latency SLO, kill-criteria window, locale set)? | Fit is exact by construction. | Fit is approximate; the team adapts the feature to the vendor's shape. |
| C2 | **Integration cost** — engineering weeks to get to first integration, plus integrations needed with existing modules (108 gateway, 110 traces, 111 flags). | Full control of the integration surface; no adapter boilerplate. | Vendor SDK does the heavy lifting; integration is often a few days. |
| C3 | **Total 12-month cost** — vendor fees + engineering time + oncall rotation cost + dependency-tracking cost. | No vendor fees; engineering time can be amortised across features. | Known monthly spend; engineering time is low upfront. |
| C4 | **Lock-in surface** — how hard is it to move off? Named: data model, API surface, pricing changes, deprecations. | None; the team owns the codebase. | Vendor data model, vendor API, vendor pricing tier, vendor deprecation calendar. |
| C5 | **Operational burden** — who oncalls it? How frequent are CVEs, breaking releases, provider outages? | High; the team is on the hook for every failure. | Low; vendor's SRE is on the hook until an outage. |
| C6 | **Governance fit** — does the option make the charter's GOVERN requirements (NIST AI RMF) easier or harder to satisfy? In particular: can the governance frame see the lever? | Full visibility; every lever is in-tree. | Depends on the vendor; some expose governance tooling, some do not. |
| C7 | **Velocity floor** — how much time does this option free up for *feature* work vs. *substrate* work? | Low; building substrate consumes feature-engineering capacity. | High; feature team can focus on feature shape, retrieval, prompts. |

The scoring protocol: for each option, score 1 (worst) to 5 (best) on each criterion. Compute weighted sum per option. The deltas, not the raw scores, drive the decision — a close race (|delta| ≤ 3) means the team should also run the **tie-breaker checks** (2e).

### 2b. Substrate 1 — LLM Gateway

The LLM gateway is the component in front of the provider APIs: request normalisation, model-ID routing, rate limiting, fallback, retries, cost accounting, trace emission. Scope definition is from mod-108 [[exercise-02]].

**Weights for the gateway:**

| Criterion | Weight | Why this weight |
|---|---|---|
| C1 Fit to charter | 2 | Gateway fit is table-stakes; every candidate hits the basics. |
| C2 Integration cost | 2 | One-time cost; doesn't dominate. |
| C3 Total 12-month cost | 2 | Gateway fees are real but rarely the biggest line on an AI budget. |
| C4 Lock-in surface | 4 | High — the gateway sits in the critical path of every call. Switching is expensive. |
| C5 Operational burden | 5 | **Dominant.** The gateway's uptime is the *feature's* uptime. Running a gateway requires 24/7 oncall. |
| C6 Governance fit | 3 | Mod-108 [[exercise-02]] requires per-call trace attributes; this is a governance surface. |
| C7 Velocity floor | 3 | Building a gateway is a multi-quarter effort; the velocity cost is significant but not catastrophic. |

**Options considered:**

1. **BUILD** — in-house Python / Go service in front of raw provider SDKs.
2. **BUY (OSS self-hosted)** — an OSS LLM gateway run by the team (the team oncalls it, upgrades it, runs the control plane).
3. **BUY (managed)** — a managed gateway service, provider-hosted or third-party-hosted.

**Scoring sketch (illustrative; weights applied):**

| Criterion (weight) | BUILD | BUY (OSS self-hosted) | BUY (managed) |
|---|---|---|---|
| C1 Fit (2) | 5 | 4 | 3 |
| C2 Integration (2) | 2 | 4 | 4 |
| C3 12-mo cost (2) | 3 | 3 | 2 |
| C4 Lock-in (4) | 5 | 4 | 2 |
| C5 Ops burden (5) | 2 | 3 | 5 |
| C6 Governance (3) | 5 | 4 | 3 |
| C7 Velocity (3) | 2 | 3 | 4 |
| **Weighted sum** | **61** | **65** | **64** |

Weighted sums are close (|delta| ≤ 4). The tie-breaker (2e) applies.

**Worked decision — BUY (OSS self-hosted) for a team of applied engineers without a dedicated ML-platform team:**

> Our charter in [[exercise-01]] requires a 1-hour kill-criteria window and p95 ≤ 4 s. Both options can meet those. The dominating criteria are C4 (lock-in) and C5 (ops burden). Managed scores highest on ops burden but takes the biggest C4 hit — the vendor's data model, API, and pricing calendar all become dependencies. OSS self-hosted splits the difference: we own the code, we can swap the backend provider, we patch on our schedule, but the day-to-day feature-request pipe is upstream's problem. Build loses on C2 and C7 — our team is 4 applied engineers; a full in-house gateway would consume one engineer-year that we can't spare while [[exercise-01]] features are in flight. We revisit if (a) we hit a feature that OSS can't express, (b) upstream pace drops below our needs, or (c) we grow a dedicated platform team.

A team **with** a dedicated ML-platform team legitimately reaches a different conclusion — their C5 cost is lower (the platform team's oncall rotation absorbs it), which can tip the matrix toward BUILD. The rubric does not force a single answer; it forces the answer to name why the inputs resolved as they did.

**Exit criteria (when we re-open):**

- **BUILD entered as default:** the OSS project ships a feature the in-house gateway would need ≥ 3 engineer-months to match.
- **BUY (OSS) entered as default:** a CVE / breaking release / abandonment event on upstream raises C5 above a chosen threshold; upstream falls behind the pace of provider feature releases by ≥ 2 quarters.
- **BUY (managed) entered as default:** vendor announces a pricing change that would raise 12-month cost ≥ 50%; vendor's governance posture becomes incompatible with the charter.

### 2c. Substrate 2 — Eval Platform

The eval platform is the component that runs the mod-106 [[exercise-02]] suite, stores fixtures (including the [[exercise-02]] regression fixtures), and reports pass/fail at the gates in mod-111 [[exercise-02]]. Scope: offline eval, judge composition, storage, dashboards.

**Weights for the eval platform:**

| Criterion | Weight | Why this weight |
|---|---|---|
| C1 Fit to charter | 4 | **Dominant.** The eval platform must be able to run the exact judges, fixtures, and pass-rules the charter + mod-106 specifies. |
| C2 Integration cost | 2 | Integrations: fixture format, CI, dashboards. Significant but one-time. |
| C3 Total 12-month cost | 2 | Eval costs are LLM-call costs more than vendor fees; both options have similar underlying costs. |
| C4 Lock-in surface | 3 | Mid — fixture formats and judge configurations are portable if stored in-repo. |
| C5 Operational burden | 2 | Eval is batch; outages are tolerable. |
| C6 Governance fit | 5 | **Dominant.** The eval platform is the artefact an auditor asks about — "show me the test that gated this release." |
| C7 Velocity floor | 3 | Building an eval harness is 2–4 engineer-months; not trivial. |

**Options considered:**

1. **BUILD** — in-house eval runner (pytest extension or small internal service) running judges against a fixture directory.
2. **BUY (OSS)** — an OSS eval framework (pytest-based or standalone).
3. **BUY (managed)** — a managed evaluation / observability platform with eval surfaces.

**Scoring sketch:**

| Criterion (weight) | BUILD | BUY (OSS) | BUY (managed) |
|---|---|---|---|
| C1 Fit (4) | 5 | 4 | 3 |
| C2 Integration (2) | 3 | 4 | 3 |
| C3 12-mo cost (2) | 4 | 4 | 2 |
| C4 Lock-in (3) | 5 | 4 | 2 |
| C5 Ops (2) | 3 | 4 | 5 |
| C6 Governance (5) | 5 | 4 | 3 |
| C7 Velocity (3) | 3 | 4 | 4 |
| **Weighted sum** | **80** | **77** | **62** |

**Worked decision — BUILD, with fixtures stored in-repo in a vendor-neutral YAML format:**

> The dominating criteria are C1 and C6. The charter's kill criteria (K1 grounding-violation, K5 safety-eval regression) must be evaluable deterministically with judges we trust; the auditor's question six months from now is "show me the eval row that passed before release." The managed option would answer "log into their dashboard" — this makes the governance frame depend on a third party we do not control. The OSS option scores well and we considered it seriously; we chose BUILD narrowly because our fixture format (the YAML in [[exercise-02]] 2b) already encodes our judge composition, pass rules, and sampling policy, and wrapping an OSS runner around it is not materially less work than writing the thin runner ourselves. We would flip to OSS the moment the OSS project supports our fixture format natively (we track this as a quarterly review).

**Exit criteria:**

- **BUILD entered as default:** an OSS framework supports our full fixture format and judge composition natively; maintenance cost on the in-house runner exceeds 0.5 engineer-months per quarter.
- **BUY (OSS) entered as default:** the OSS project becomes unmaintained or incompatible with our fixture format; the project's judge-composition model diverges from ours.
- **BUY (managed) entered as default:** charter removes the "eval evidence is in-repo" governance requirement (unlikely).

### 2d. Substrate 3 — Agent SDK / Orchestration Layer

The agent SDK is the component that orchestrates tool-calling loops: tool schema definition, tool-call parsing, streaming, retries, structured outputs. Scope: mod-104 [[exercise-01]], [[exercise-02]], [[exercise-04]].

**Weights for the agent SDK:**

| Criterion | Weight | Why this weight |
|---|---|---|
| C1 Fit to charter | 3 | Tool-calling feature shape varies per feature; fit matters but isn't dominant. |
| C2 Integration cost | 3 | SDK-choice touches every feature's call-site; integration cost compounds. |
| C3 Total 12-month cost | 2 | SDK licences are usually free or trivial; costs show up in engineering time. |
| C4 Lock-in surface | 5 | **Dominant.** The agent SDK shapes feature code; switching SDKs rewrites every feature. |
| C5 Operational burden | 2 | SDKs run in-process; outages are the feature's problem, not the SDK's. |
| C6 Governance fit | 3 | Tool-call traces are a governance surface (mod-104 [[exercise-02]]). |
| C7 Velocity floor | 4 | A good SDK dramatically speeds feature authoring; a wrong SDK slows every feature down. |

**Options considered:**

1. **BUILD** — thin in-house tool-calling layer on top of provider SDKs.
2. **BUY (OSS)** — an OSS orchestration framework (there are several leading candidates; mod-104 [[exercise-04]] compares them).
3. **BUY (managed)** — a managed agent-orchestration service.

**Scoring sketch:**

| Criterion (weight) | BUILD | BUY (OSS) | BUY (managed) |
|---|---|---|---|
| C1 Fit (3) | 5 | 3 | 2 |
| C2 Integration (3) | 2 | 4 | 3 |
| C3 12-mo cost (2) | 4 | 5 | 2 |
| C4 Lock-in (5) | 5 | 3 | 1 |
| C5 Ops (2) | 4 | 4 | 4 |
| C6 Governance (3) | 4 | 3 | 2 |
| C7 Velocity (4) | 3 | 5 | 4 |
| **Weighted sum** | **84** | **79** | **56** |

**Worked decision — BUILD (thin layer), with a stated plan to re-evaluate OSS every six months:**

> The agent SDK shapes every feature's source code. C4 (lock-in) weighted 5 reflects that a wrong choice here is the most expensive wrong choice on this matrix — switching SDKs is a cross-feature rewrite. Our charter supports read-only tool calls today ([[exercise-01]] Section 2); a thin in-house layer over provider SDKs is 2–3 engineer-weeks and keeps feature source code in direct correspondence with the provider's tool-call shape. Once a feature needs richer orchestration (agent-to-agent handoff, cross-turn planning), we expect the OSS lane to be the right answer, and we have reserved re-evaluation cadence to catch that moment. The managed option is a hard no for our team: C4 at 1 means a provider-side change can break every feature in one incident window.

**Exit criteria:**

- **BUILD entered as default:** we take on an orchestration pattern (agent-to-agent, cross-turn planning) that would require rewriting the in-house layer beyond trivial scope.
- **BUY (OSS) entered as default:** the OSS layer becomes incompatible with our observability (mod-110 [[exercise-01]]) spans or governance requirements.
- **BUY (managed) entered as default:** the entire team pivots to a single managed-provider stack with first-party agent surfaces; the charter removes the multi-provider requirement.

### 2e. Tie-breaker checks

When the weighted sums are close (|delta| ≤ 3), run these three tie-breakers in order:

1. **The "page at 3 AM" check.** Which option fails most gracefully when the on-call is alone at 3 AM on a holiday? BUILD assumes the on-call can read source; BUY (managed) assumes the vendor's status page is honest; BUY (OSS) assumes the on-call can patch upstream.
2. **The "six-months-off-team" check.** Which option is most legible to the next engineer six months after the author has rotated off the team? BUILD can decay fastest without ownership; BUY (OSS) has upstream docs; BUY (managed) has vendor docs that outlive any single engineer.
3. **The "regulator tap on the shoulder" check.** Which option makes it easiest to answer "show me the test / log / policy that governs this behaviour"? BUILD keeps the artefact in-repo; BUY (managed) often puts it in a vendor dashboard that your auditor cannot access.

If the three tie-breakers do not converge, escalate the decision — this is a sign the matrix is missing a criterion or the team disagrees on a weight. Escalation is a *better* answer than picking one at random.

### 2f. The one-page decision record

Every decision is written up as a one-page record checked into `docs/substrates/<name>.md`:

```markdown
# Substrate decision — <name>
Owner: <applied-AI lead>
Decided at: <YYYY-MM-DD>
Review cadence: quarterly (next: <YYYY-MM-DD>)

## Options considered
- BUILD: <specific technology stack>
- BUY (OSS): <specific project name + version>
- BUY (managed): <specific vendor + tier>

## Weights
<table of weights, with one-sentence justification per weight>

## Scores
<option-by-criterion score table>

## Decision
<one paragraph>

## Exit criteria (what would re-open this decision)
- BUILD entered as default: <specific conditions>
- BUY (OSS) entered as default: <specific conditions>
- BUY (managed) entered as default: <specific conditions>

## Dependencies this decision creates
- Upstream contract: <mod-108 / mod-106 / mod-104 artefacts assumed stable>
- Oncall rotation impact: <hours/week on this substrate>
- Breaking-release calendar to track: <upstream project / vendor>
```

### 2g. Mapping to the exercise deliverables

| Exercise asks for | Section of this solution |
|---|---|
| Shared decision rubric | 2a |
| Gateway matrix | 2b |
| Eval platform matrix | 2c |
| Agent SDK matrix | 2d |
| Decision procedure when scores are close | 2e |
| Check-in artefact | 2f |

## 3. Validation steps

1. **Every criterion has a weight justification.** Grep each substrate's weight table — a weight without a one-sentence justification is a hard failure.
2. **Every scored cell has a reason the reviewer can challenge.** The score "5" for BUY (managed) on C5 must point to a specific vendor capability or an operational fact the reviewer can falsify.
3. **Each substrate has a different weight distribution.** If gateway, eval, and SDK all share the same weight vector, the student has defaulted to one-matrix-fits-all — a hard failure.
4. **At least one criterion is scored differently across the three options for each substrate.** If BUILD and BUY (OSS) score identically on all seven criteria for one substrate, the matrix does not discriminate and the decision is not defensible.
5. **Exit criteria exist for every option, not only the chosen one.** The student names what would re-open the decision toward each of the two un-chosen options.
6. **Exit criteria are falsifiable.** "Vendor becomes expensive" is not falsifiable; "vendor announces a pricing change raising 12-month cost ≥ 50%" is.
7. **Tie-breaker ran when needed.** Any substrate whose weighted-sum delta is ≤ 3 must show the three tie-breaker results.
8. **Decision record is checked in per 2f.** The artefact exists for all three substrates; each has an owner, a decided-at date, and a review cadence.
9. **Charter cross-reference.** Each decision record cites the [[exercise-01]] charter requirement that drives the C1 score. Decisions untethered from the charter are a hard failure.
10. **Governance mapping.** The decision records name which NIST AI RMF sub-function each substrate supports and how the chosen option satisfies it.

## 4. Rubric or review checklist

Instructor rubric (10 points total). A submission scoring < 7 is rejected.

| Criterion | Points | What earns the points |
|---|---|---|
| Seven-criterion rubric with per-criterion definitions | 1 | Each criterion is defined with "what build wins on / what buy wins on" to prevent single-sided scoring. |
| Per-substrate weight tuning with justification | 2 | Three distinct weight vectors; each weight has a one-sentence justification. |
| Scored matrix for each substrate | 2 | Every cell has a defensible score; cells differ across options. |
| Worked decision paragraph per substrate | 1 | The paragraph names the dominant criterion and the trade-off accepted. |
| Exit criteria per substrate for *every* option | 1 | Not only "when we would change our mind," but what would move the default to each of the three options. |
| Falsifiable exit criteria | 1 | Each exit criterion is a specific, measurable event (CVE, pricing change, feature parity, re-ratification). |
| Tie-breaker procedure applied where needed | 1 | The three tie-breakers in 2e are named; results are captured for any close decision. |
| Decision records checked in with owners and cadence | 1 | The one-page artefact in 2f exists per substrate. |

**Hard failures (automatic rejection):**

- Decision with no weighted-sum calculation.
- Same weight vector across all three substrates.
- Decisions untethered from the [[exercise-01]] charter (no cross-reference).
- No exit criteria for the non-chosen options.
- Scoring "build wins on everything" or "buy wins on everything" on all three substrates — a one-sided matrix is a diagnostic sign the student has not engaged with the trade-offs.

## 5. Common mistakes

- **"We'll use LangChain" / "We'll use LiteLLM" as a decision.** Naming the product is not the decision; the decision is the matrix that led to the name.
- **One matrix across all three substrates.** Gateway, eval, and SDK have different weight vectors — merging them into one matrix produces mushy answers. Score each separately.
- **Treating OSS as "free."** Self-hosted OSS has an operational burden that costs real engineering time; price it on C5 and C7.
- **Treating managed as "lock-in = 0."** Vendor data model, pricing tier, and deprecation calendar are all lock-in surfaces; price them on C4.
- **Decision without exit criteria.** A build-vs-buy call that is never re-opened is a bet, not a decision. Falsifiable exit criteria are mandatory.
- **Vague exit criteria.** "If the vendor becomes bad" is not an exit criterion; "if the vendor's p95 latency exceeds 2× our SLO for two consecutive weeks" is.
- **Decision untethered from the charter.** The charter from [[exercise-01]] is the input to the C1 score; a decision that does not cite the charter's kill criteria or SLO on the C1 cells is not grounded.
- **"Build is a learning opportunity."** Building substrate for pedagogy reasons burns the team's capacity. If learning is the goal, name it as a weighted criterion (and defend why it matters more than C7 velocity).
- **Scoring the chosen option a 5 on every criterion.** If the chosen option is never anywhere but 5, the matrix is working backward from the conclusion. Expect at least one 2 or 3 on the chosen option — that is the trade-off you accepted.
- **No owner on the decision record.** "The team" is not an owner. Name a person.
- **Missing governance (C6) column.** The governance fit of each substrate is what the auditor asks about; omitting it means the decision cannot be defended under audit.
- **Not re-scoring when a new feature changes the charter.** The matrix is a point-in-time artefact; when the charter adds a new cohort / model / regulator trigger, the matrix re-runs.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — GOVERN-1.6 requires that organisational decisions about AI systems be documented with the reasoning that produced them. The build-vs-buy decision record in 2f is the direct artefact for substrate-level decisions.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML06 (AI Supply Chain Attacks) is the primary OWASP consideration in BUY decisions (OSS and managed both introduce supply-chain surface); the "oncall for upstream CVEs" and "vendor security posture" questions in 2b–2d are the mitigation surface.
- MITRE ATLAS — <https://atlas.mitre.org/> — the AML.T0057 (LLM Jailbreak) class of techniques maps to the agent SDK substrate (2d): tool-call surfaces are jailbreak targets. The C1 score on SDK should reflect whether the option supports schema-validated tool calls (mod-104 [[exercise-01]]) by construction.

Related exercises: [[exercise-01]] (charter provides the C1 inputs for all three substrates), [[exercise-02]] (incident loop names substrate dependencies — gateway logs, eval platform fixtures, agent SDK tool-call traces), [[exercise-04]] (the feature card cites substrate decisions as one of its sections), [[exercise-05]] (delegation contracts formalise the dependency surface the substrate decisions assume); mod-104 [[exercise-04]] (LangGraph vs Agents SDK bake-off that pre-dates this matrix with technical comparisons), mod-106 [[exercise-02]] (eval set format and sampling policy driving the eval-platform C1 score), mod-108 [[exercise-02]] (gateway contract — the C1 score on gateway is driven by this), mod-110 [[exercise-01]] (trace emission as a substrate requirement — all three substrates must satisfy this), mod-111 [[exercise-01]] (flag layer providing the containment lever that the substrates must interact cleanly with).
