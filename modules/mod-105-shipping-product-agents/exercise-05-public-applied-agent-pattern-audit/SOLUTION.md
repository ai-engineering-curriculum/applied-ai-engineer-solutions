### exercise-05: Public Applied Agent Pattern Audit — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to audit **public writeups of product-embedded agents** — engineering blogs, conference talks, open-source agent frameworks' own reference apps, published design docs — and extract the *patterns* that recur across them. The output is not a literature review. It is a pattern catalogue the team can use when reviewing the next agent design: for each pattern, when it fits, when it is a red flag, and how the four surface decisions (shape, memory, HITL, eval) from this module apply the pattern or refuse it.

The failure mode the exercise is written to catch is the magpie one: a team reads ten blog posts, copies five decisions out of context, and ships a design whose pieces do not cohere. Public writeups are a source of hypotheses, not a source of truth; the audit's job is to sift them against the standards (NIST AI RMF, OWASP ML-Sec, MITRE ATLAS) and against the team's own surface decisions.

This solution provides:

- A reference **audit protocol** — what counts as a source, how patterns are extracted, how claims are verified, how primary-source evidence is cited.
- A reference **pattern catalogue** of eight recurring patterns across published product-agent writeups, grouped by the four surfaces from this module (shape, memory, HITL, eval).
- A reference **"red flag" catalogue** of five patterns that appear in writeups but should not propagate — and why each fails against the module's decisions or the standards.
- A reference **source-quality rubric** for ranking the public writeups — primary (team shipped the surface), secondary (someone wrote about shipping), tertiary (vendor marketing).
- A **grading rubric** for the instructor.
- The audit is deliberately written **without fabricating** specific case studies or metrics. Every claim-shaped paragraph that would otherwise need a source (a company, a number, an incident) is marked `<!-- citation-pending: ... -->` so the auto-merge block flags it for the instructor or reviewer to fill with a verified citation.

The lesson the exercise defends is: *the field has conventions, and they are worth knowing*. Public writeups converge on a small number of patterns — bounded tool loops, pre-write confirms, scratchpad memory, trajectory evals — because those patterns survive first-contact with shipped products. The audit is how the team learns the conventions without building each one from scratch.

## 2. Worked answer or implementation

### 2a. Audit protocol (deliverable 1)

```markdown
# Audit protocol — public product-agent patterns

## Scope
- Writeups of **agents embedded in shipped products** (not research
  benchmarks, not demos without users, not toys).
- Writeups whose authors either shipped the surface or document a
  shipped surface with named technical specifics.
- Dated 2023 or later; older writeups predate current tool-calling
  interfaces and are tagged "historical."

## Source-quality tiers
- **Tier 1 — primary**: the team that built the surface wrote the
  piece. Named engineers; code or architecture diagrams; concrete
  numbers (step budgets, latency, incident categories).
- **Tier 2 — secondary**: a journalist or analyst interviewed the
  team. Quotes present; specifics verifiable against a Tier 1 source.
- **Tier 3 — tertiary**: vendor marketing, conference keynote without
  a Tier 1 writeup, blog-of-a-blog summaries. *Used only as leads to
  find Tier 1 sources.*

## Inclusion criteria
Each pattern in the catalogue is sourced from **at least two** Tier 1
or Tier 2 writeups, or one Tier 1 writeup + a standard
(NIST AI RMF / OWASP ML-Sec / MITRE ATLAS) that names the same control.
A pattern sourced only from Tier 3 is excluded.

## Extraction procedure
For each writeup:
1. Record the claim (what pattern is present), the artefact (code
   excerpt, architecture diagram, number), and the source tier.
2. Note the surface (shape / memory / HITL / eval) the claim lands in.
3. Attempt a second-source confirmation before promoting to the
   catalogue.
4. If a claim cannot be verified, do not include it. If it is
   interesting enough to flag for research, write
   `<!-- citation-pending: <claim> -->` and leave it out of the
   catalogue body.

## What is explicitly out of scope
- Comparative performance numbers across named products (benchmark
  wars). The audit is pattern-focused, not vendor-focused.
- Pricing, cost per request, or model-provider selection
  ([[mod-108]] owns those).
- Any claim about non-public internal incidents or unshipped
  prototypes.
```

### 2b. The pattern catalogue (deliverable 2)

Eight patterns, grouped by surface. Each is stated as an observable design choice, then evaluated against this module's reference surface.

#### Shape patterns

**P1. Bounded tool-loop, step-capped.** Product agents converge on a bounded tool-loop — the model emits tool calls, each result feeds back, the loop terminates on a step budget or a `final_answer`. The step budget is small (single-digit on consumer surfaces, low-double-digit on research surfaces). The alternative shapes from [[exercise-01]] (free-planner-with-executor, multi-agent delegation) appear mostly in research writeups, rarely in product writeups.

- Fits when: the work needs to react to tool results but is bounded in scope and latency.
- Red flag when: the step cap is absent or "when the model decides it is done."
- Standards alignment: NIST AI RMF MEASURE — measurable, bounded behaviour is a precondition for incident-reproducibility.
- This module's answer: Shape B in [[exercise-01]] is this pattern.

<!-- citation-pending: cite two or more Tier 1 primary writeups that explicitly describe a bounded tool-loop with a numeric step cap, including the cap value. -->

**P2. A planner only when the horizon demands it.** Published writeups of products that adopted an explicit planner-executor shape name a reason — multi-hour tasks, many-step workflows where the plan itself is the user-visible artefact (research agents, operator agents). Short surfaces (reply drafters, code-completion sidecars) rarely use this shape in published writeups.

- Fits when: the plan is a product artefact (shown to the user, reviewable); step counts exceed what a loop can reasonably budget.
- Red flag when: a planner is added "for sophistication" on a surface whose latency envelope cannot absorb planner + executor turns.
- This module's answer: Shape C in [[exercise-01]], not picked for the helpdesk surface.

<!-- citation-pending: cite one or more Tier 1 writeups for long-horizon product agents that explicitly justify adopting a planner-executor shape. -->

#### Memory patterns

**P3. Scoped memory writes, not a single bucket.** Product writeups that discuss memory distinguish between session-scoped scratchpad, user-pinned preferences, and tenant-shared policy. Writeups that report incidents around memory often trace them to a single untyped store that mixed scopes.

- Fits when: the surface has more than one kind of fact to remember.
- Red flag when: a "memory" is one keyspace with no owner per key.
- Standards alignment: NIST AI RMF MANAGE (ownership + review); OWASP ML-Sec (sensitive-data exposure).
- This module's answer: the four-scope taxonomy in [[exercise-02]] is this pattern.

**P4. Write-time PII classification.** Published designs that take PII seriously classify at write time, not at read time. Writeups that describe incidents often trace them to read-time scrubbing that could not retroactively clean backups.

- Fits when: user content flows into memory at all.
- Red flag when: PII policy is documented but not enforced by code on the write path.
- Standards alignment: OWASP ML-Sec sensitive-data guidance; MITRE ATLAS techniques around cross-session data leakage.
- This module's answer: `pii_classifier.py` in [[exercise-02]] is this pattern.

#### HITL patterns

**P5. Pre-write confirm with raw arguments.** Product writeups of write-capable agents converge on a pre-write confirm UI that shows the arguments *as they will be sent*, not a paraphrase. Writeups that describe prompt-injection-driven incidents often trace them to paraphrased confirms that let the user approve one thing while a different thing was sent.

- Fits when: the agent can perform a write of any kind.
- Red flag when: the confirm is a modal with "OK/Cancel" and no arguments, or has an auto-confirm timer.
- Standards alignment: OWASP ML-Sec input-manipulation; MITRE ATLAS prompt-injection techniques.
- This module's answer: the `confirm_write` pause in [[exercise-03]] is this pattern.

**P6. Four-layer kill switch.** Published designs describe a kill switch at multiple layers — the user's "stop," the user's "disable for session," a tenant-admin lever, and an operator feature flag. Each layer has a named SLA to engage and a named owner.

- Fits when: the surface is user-facing and writeable.
- Red flag when: there is one "disable the feature" switch and no distinction between a user-level and operator-level stop.
- Standards alignment: NIST AI RMF MANAGE (human-overridable controls, named owners, response times).
- This module's answer: the four-layer design in [[exercise-03]] is this pattern.

#### Eval patterns

**P7. Per-shape pass bars with a 100% safety bar.** Product writeups about agent evals converge on per-shape scoring (task-completion, tool-use, trajectory, safety), with the safety bar treated as a hard gate at 100%. Aggregate single-number scores appear mostly in research writeups.

- Fits when: the surface has distinguishable success modes the team can score separately.
- Red flag when: the merge gate is a single aggregate pass rate.
- Standards alignment: NIST AI RMF MEASURE (per-property scoring, no amortisation of safety failures).
- This module's answer: the §2e pass-bar table in [[exercise-04]] is this pattern.

**P8. Judge-model pinning + averaging.** Writeups that take judge-LLM evals seriously pin the judge model with a version lock and average the score across N (typically 3) runs. Writeups that describe "we had a regression we couldn't reproduce" often trace it to an unpinned judge.

- Fits when: any scorer is itself an LLM.
- Red flag when: the judge is "the current best model" with no pin.
- Standards alignment: NIST AI RMF MANAGE (controls on model change); the same change-management discipline a model-routing exercise would apply.
- This module's answer: `judge.lock` + three-run averaging in [[exercise-04]] is this pattern.

### 2c. Red-flag catalogue (deliverable 3)

Patterns that appear in writeups but should not propagate:

**R1. "Give the agent all the tools."** Writeups that show a 40-tool manifest usually either (a) document a research-only surface, or (b) describe an incident after the fact. A tool count ≫ the shape's step cap means the model picks randomly; see [[exercise-01]] (mod-104).

**R2. "Memory as a vector DB, we're done."** Vector memory is one read-path implementation of one scope; it does not replace the taxonomy, the retention schedule, or the PII classifier from [[exercise-02]].

**R3. "The model knows when to ask for help."** Writeups that rely on the model's self-report of uncertainty often have an incident later. HITL is a product-surface state machine, not an emergent behaviour; see [[exercise-03]].

**R4. Auto-confirm with a countdown.** This is the single most common HITL anti-pattern across public writeups. It undoes the confirm. See [[exercise-03]] §2d.

**R5. Aggregate benchmark scores as the merge gate.** Writeups that report "our agent gets 73% on X" without per-shape or per-safety breakdowns are not reporting the shape of their failures. See [[exercise-04]].

Each R-row should be instanceable from at least one real writeup when the student does the audit; this solution leaves the specific sources unresolved rather than invent them:

<!-- citation-pending: cite one or more Tier 1 or Tier 2 writeups that exhibit R1 (40-tool manifest on a shipped surface). -->
<!-- citation-pending: cite one or more Tier 1 or Tier 2 writeups that exhibit R2 (vector-DB-as-memory with no retention story). -->
<!-- citation-pending: cite one or more Tier 1 or Tier 2 writeups that exhibit R3 (model self-report as HITL trigger). -->
<!-- citation-pending: cite one or more Tier 1 or Tier 2 writeups that exhibit R4 (auto-confirm countdown on a write path). -->
<!-- citation-pending: cite one or more Tier 1 or Tier 2 writeups that report an aggregate benchmark score as the sole merge gate. -->

### 2d. Source-quality rubric (deliverable 4)

```markdown
# Rubric for a source entering the catalogue

## Must have
- Date ≥ 2023 (or tagged "historical" with an explicit note on which
  parts predate current tool-calling conventions).
- Primary-source signals: named engineering authors, architecture
  diagrams or code fragments, named tool / step budgets.
- At least one verifiable technical claim (not just vibes).

## Nice to have
- A follow-up post with a retrospective (shipping is not landing).
- A link to a public repo or runnable demo.
- A mapping to one of NIST AI RMF, OWASP ML-Sec, or MITRE ATLAS
  controls (often absent; when present, strong signal).

## Immediate disqualifiers
- No author attribution.
- "Our proprietary agent framework" with no technical content.
- Numbers quoted without methodology.
- A marketing page masquerading as a design writeup.

## How to weight a disputed claim
- When two Tier 1 writeups disagree (e.g., one caps at 6 tool calls,
  another at 20), record both, note the surface differences, and
  leave the pattern's "fits when" clause broad enough to cover both.
  Disagreement between credible primary sources is a signal that the
  right answer depends on the surface.
```

### 2e. The audit's commitments to the team (deliverable 5)

```markdown
# What the audit obligates the team to do

- When a new public writeup lands that names a pattern already in our
  catalogue: add it to the sources, update the "red flag when" or
  "fits when" clause if the new source shifts the boundary, and
  review in the next weekly sync.
- When a new public writeup lands that names a pattern NOT in our
  catalogue: tag it `needs-review`; the applied-AI lead owns a
  decision within two weeks.
- When our own shipped surface hits an incident whose root cause is
  a pattern we did not have: add a pattern (or a red flag) to the
  catalogue; the postmortem from [[mod-110]] is the source.
- Quarterly: re-read the catalogue end-to-end. Mark patterns that
  are no longer load-bearing as `deprecated`, with a reason.

## What the audit does NOT do
- It does not pick a framework ([[exercise-04]] mod-104 does).
- It does not pick a model ([[mod-108]] does).
- It does not approve a surface for launch ([[mod-111]] does).
- It does not replace a security review ([[mod-107]]).
```

### 2f. Implementation notes

- The audit is a *reading practice*, not a one-shot deliverable. The catalogue is a living document; the grading rubric below measures the student's discipline in the extraction, not the completeness of a point-in-time survey.
- Every pattern is grounded in **at least two** named sources, or **one named source + a standards control**. A single-source pattern is marked as a hypothesis. The rule is self-imposed because the field moves fast; a single writeup can name a local maximum that does not generalise.
- The audit is deliberately conservative about naming vendors or companies. Patterns are the artefact; the sources back them up. The catalogue should read like it will still be useful in six months, not like a snapshot of this quarter's engineering-blog highlights.
- The `citation-pending` markers are not sloppiness; they are the auto-merge block. A reviewer who sees them knows the student has flagged the places where the solution text would otherwise have made up a case study. In a real submission, each marker is replaced with a concrete citation after the student does the actual reading. This solution ships them unresolved rather than fabricate.
- The catalogue's job is to accelerate review of the next design, not to tell the team what to build. Patterns are described as design choices; the "fits when" / "red flag when" lines are the operational guidance.
- The audit's output feeds directly into the review process for every subsequent agent-shaped feature at the company. It is a repeated-use artefact, not a term paper.

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. The audit protocol defines the three source tiers and the inclusion criterion (≥ 2 Tier 1/2 or 1 Tier 1 + standards control).
2. The pattern catalogue groups patterns by the four surfaces from this module (shape / memory / HITL / eval). Each pattern names "fits when," "red flag when," and a standards-alignment line.
3. Every pattern cites at least two real sources, or one source + a standards control. Patterns backed only by Tier 3 sources are excluded. (In this reference solution the specific citations are left as `citation-pending` placeholders; a real submission fills them.)
4. The red-flag catalogue names at least five anti-patterns with a concrete reason each (standards misalignment, incident mode, or contradiction with this module's decisions).
5. The source-quality rubric names explicit disqualifiers.
6. Each pattern is tied to a specific prior exercise in this module (or named as "not covered elsewhere").
7. The "what the audit obligates the team to do" section names the review cadence and the hand-off to [[mod-110]] / [[mod-111]].
8. Any factual claim not derivable from the cited primary sources or the named standards is either removed or flagged with `<!-- citation-pending: ... -->`.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Three-tier source rubric with explicit disqualifiers | 1 | Table; marketing pages excluded. |
| Inclusion rule ≥ 2 Tier 1/2 or 1 Tier 1 + standard | 1 | Rule stated, applied per-pattern. |
| Catalogue grouped by the four module surfaces | 1 | Shape / memory / HITL / eval sections. |
| Each pattern has "fits when" + "red flag when" | 1 | Operational guidance, not description. |
| Standards cross-reference on every pattern | 1 | NIST AI RMF / OWASP ML-Sec / MITRE ATLAS named. |
| ≥ 5 red-flag anti-patterns with reasons | 1 | Each reason ties to a standard or this module. |
| Every pattern tied to a prior exercise in the module | 1 | Explicit `[[exercise-N]]` links. |
| Citation-pending markers used honestly | 1 | Fabricated sources replaced with `citation-pending`. |
| Audit protocol names scope + out-of-scope | 1 | What this audit does not do is named. |
| Team obligations + review cadence | 1 | Hand-off to [[mod-110]] / [[mod-111]] named. |

A submission scoring < 6 is rejected. The hard failures are (a) fabricated case studies or metrics, (b) Tier 3-only patterns, (c) a catalogue of descriptions with no "fits when / red flag when" guidance, (d) no standards cross-reference.

## 5. Common mistakes

- **Treating the audit as a literature review.** The output is a pattern catalogue with operational guidance, not a book report.
- **Fabricating specifics.** "Company X caps at 7 tool calls" is a claim; if you cannot cite it, do not write it. Use `citation-pending`.
- **Including Tier 3 marketing as evidence.** Vendor blog posts are a lead, not a source. Walk back to the Tier 1 writeup or exclude the pattern.
- **Patterns without "red flag when."** A pattern that does not say when it is wrong is a recommendation, not an audit finding.
- **No standards cross-reference.** The whole point of grounding in NIST AI RMF / OWASP ML-Sec / MITRE ATLAS is to pull the field's conventions onto an auditable control surface. Without the mapping, the catalogue is one more opinion.
- **Patterns detached from the module's surfaces.** Every pattern must land in one of shape / memory / HITL / eval (or be explicitly noted as orthogonal). A floating pattern cannot be reviewed against the surface.
- **"Permanent" catalogue, no review cadence.** The field moves; the catalogue decays. Quarterly review is the floor.
- **Confusing the audit with a security review.** OWASP ML-Sec and MITRE ATLAS are referenced for alignment; the audit does not replace the security review from [[mod-107]].
- **Writing the audit once and treating it as finished.** It is a repeated-use artefact. The "team obligations" section is the mechanism for keeping it alive.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the audit's inclusion rule, the standards cross-reference per pattern, and the quarterly review cadence map to the RMF's GOVERN (policies and reviews) and MANAGE (owned, dated controls) functions. The RMF is the baseline the catalogue is scored against when a pattern's standards-alignment line is written.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the red-flag catalogue's anti-patterns (R2 memory-as-vector-DB, R4 auto-confirm, R5 aggregate-safety) each land on OWASP top-10 items; the catalogue uses OWASP as a shared vocabulary with the security team.
- MITRE ATLAS — <https://atlas.mitre.org/> — the patterns around HITL pre-write confirm (P5), four-layer kill switch (P6), and write-time PII classification (P4) are the surface's implementation of controls against ATLAS techniques; ATLAS is the catalogue's bridge to the threat-model side of review.

Related exercises: [[exercise-01]] (Shape P1/P2 patterns map here), [[exercise-02]] (Memory P3/P4 patterns map here), [[exercise-03]] (HITL P5/P6 patterns map here), [[exercise-04]] (Eval P7/P8 patterns map here); downstream [[mod-107]] (security review inherits the OWASP + ATLAS cross-reference); [[mod-110]] (incident postmortems feed new patterns and red flags back to the catalogue); [[mod-111]] (rollout guardrails use the catalogue as the design-review checklist).
