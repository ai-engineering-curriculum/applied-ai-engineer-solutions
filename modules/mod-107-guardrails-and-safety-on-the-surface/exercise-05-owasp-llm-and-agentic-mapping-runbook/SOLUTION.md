### exercise-05: OWASP LLM and Agentic Mapping Runbook — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to produce a **mapping runbook** that places the helpdesk-reply-copilot's concrete controls — moderator, injection defence, PII/secret scrub, tool guards and gates, memory policy, HITL, incident bundle — against external threat and risk frameworks so that the surface can pass a security review without inventing the taxonomy on the day of the review. The relevant frameworks are OWASP Machine Learning Security Top 10, MITRE ATLAS, and NIST AI Risk Management Framework; the runbook maps the surface's controls to each, in both directions, and reports coverage gaps with owners.

The failure mode the exercise is written to catch is "we are OWASP-compliant" told to a stakeholder in a room. OWASP is not a certification, ATLAS is not a checklist, and the AI RMF is a set of functions, not controls. The runbook is the artefact that lets a reviewer *verify* which control answers which concern and, more importantly, which concerns no control yet answers.

This solution provides:

- A **control inventory** for the surface — every guardrail from mod-107, each with a stable ID, owner, enforcement point, and measurement.
- A **threat/risk-framework crosswalk** for OWASP ML Top 10, MITRE ATLAS tactics/techniques, and NIST AI RMF functions, each mapped to controls (and to `uncovered` where applicable).
- A **gap report template** that produces actionable tickets with owners and a target date.
- A **review cadence and re-mapping procedure** triggered by surface, framework, or model changes.
- A **grading rubric** for the instructor.

The lesson the exercise defends is: *framework coverage is a build artefact, not a slide.* The runbook is version-controlled, referenced from every exercise's deliverables, and is the thing a security reviewer reads first.

## 2. Worked answer or implementation

### 2a. Control inventory (deliverable 1)

```markdown
# helpdesk-reply-copilot — control inventory (v1)

Each control has a stable ID (`H107-C##`), an owner, the exercise that
defined it, an enforcement point in code, and the measurement that
tells us whether it is operating.

| ID | Control | Defined by | Owner | Enforcement point | Measurement |
|---|---|---|---|---|---|
| H107-C01 | Layered moderation (rules → classifier → judge) | [[exercise-01]] | moderation team | `moderation_pipeline.moderate` | FP/FN report per category/slice |
| H107-C02 | Moderation policy YAML (per-tenant thresholds) | [[exercise-01]] | policy / legal | `moderation_policy.yaml` | diff-reviewed on every change |
| H107-C03 | Prompt-injection defence stack (4 layers) | [[exercise-02]] | security | `agent_host` + `tool_gate` | AgentDojo-shape report (utility + security + adaptive) |
| H107-C04 | Provenance tagging of every T3 source | [[exercise-02]] | agent host | `host.wrap_input` | span assertion in trace tests |
| H107-C05 | Per-source tool-exposure restriction | [[exercise-02]] | agent host | `tool_gate.state_precondition` | unit tests + injection harness |
| H107-C06 | PII/secret scrubber at trace exporter | [[exercise-03]] | observability | `exporter.export_spans` | leak-hunt protocol monthly |
| H107-C07 | Scrub policy version-controlled | [[exercise-03]] | privacy | `scrub_policy.yaml` | diff-reviewed; tenant looseness forbidden |
| H107-C08 | Pre-call guard pipeline (5 guards) | [[exercise-04]] | agent host | `tool_gate.run_tool` | unit tests + audit dashboard |
| H107-C09 | Destructive-action gate (HITL + receipt) | [[exercise-04]] | agent host | `tool_gate.run_tool` destructive path | audit: 100 % write-tool calls receipted |
| H107-C10 | Compensating actions for external writes | [[exercise-04]] | product | `compensators.*` | drill: quarterly rollback rehearsal |
| H107-C11 | Memory scope + retention + tenant isolation | [[exercise-02]] (mod-105) | storage | `MemoryStore.write/read` | test suite: cross-tenant, TTL, erasure |
| H107-C12 | HITL confirm UX | [[exercise-03]] (mod-105) | product | `confirm_ui` | UX test; no HTML rendered |
| H107-C13 | Kill switch | [[exercise-03]] (mod-105) | on-call | `feature_flag.kill_copilot` | drill: monthly |
| H107-C14 | Model-provider contract (base-model refusal) | upstream | model | provider SLA | periodic re-measurement on model-swap |

Controls are additive: a given threat may be covered by more than one.
A threat covered by **zero** controls is a `gap`, listed below.
```

### 2b. OWASP ML Top 10 crosswalk (deliverable 2a)

The OWASP ML Top 10 is an ML-system taxonomy. The map here names which surface controls answer each item and names the *residual* risk where a full answer requires upstream model work.

| OWASP ML Item | How it shows up on this surface | Covering controls | Residual |
|---|---|---|---|
| ML01 Input manipulation (adversarial inputs) | adversarial user_text / ticket body; adversarial KB passages | H107-C01, H107-C03, H107-C04, H107-C05 | provider base-model robustness (H107-C14) |
| ML02 Data poisoning | tenant admin seeding the KB; user seeding memory | H107-C01 (policy_contradiction), H107-C11 (memory policy), H107-C06 (scrub audit) | poisoning of training data: upstream — not this surface |
| ML03 Model inversion attack | trace-based inference about training data | H107-C06 (scrubber on traces) | base-model inversion upstream |
| ML04 Membership inference attack | distinguishing training examples | H107-C06, H107-C11 | base-model upstream |
| ML05 Model theft | agent cloning via logs | H107-C06 (scrub model names, outputs on traces are not training data), H107-C14 (provider contract) | provider API-side protections |
| ML06 AI supply chain | model upgrades, tool-chain pins | H107-C03 harness is re-run on model swap (see mod-108); scrub policy diff-reviewed (H107-C07) | third-party agent framework CVEs: tracked in platform SCA |
| ML07 Transfer learning attack | fine-tunes used on this surface | n/a on this surface (no fine-tune); if introduced, owned by mod-108 | — |
| ML08 Model skewing | agent drift via feedback loops | H107-C01 (moderator FP/FN report cadence); mod-106 online-eval drift monitor | — |
| ML09 Output integrity attack | hidden instructions in output; stripping/replacing tokens | H107-C03 layer 4 output sanitiser; H107-C09 receipt binding | — |
| ML10 Model poisoning | corruption of model weights/artefacts | n/a — upstream | provider contract H107-C14 |

### 2c. MITRE ATLAS crosswalk (deliverable 2b)

ATLAS is a tactics/techniques framework for adversarial ML. The crosswalk picks the techniques reachable on this surface (so is not the complete ATLAS table) and names the covering controls.

| Tactic | Technique (ATLAS-ID shape) | Reachable on this surface? | Covering controls | Residual |
|---|---|---|---|---|
| Reconnaissance | search of public materials for model/tool identity | yes | H107-C06 (do not emit model IDs in user-visible traces) | — |
| Resource Development | adversarial examples prepared offline | yes | H107-C03 harness includes offline-prepared attacks | — |
| Initial Access | indirect prompt injection via KB / ticket / tool output | yes | H107-C03, H107-C04, H107-C05, H107-C09 | — |
| ML Model Access | API-abuse to probe model | yes | H107-C08 rate guard; H107-C14 provider throttles | — |
| Execution | tool misuse via agent | yes | H107-C08, H107-C09 | — |
| Persistence | plant content in memory / KB for later run | yes | H107-C11 memory policy; H107-C01 moderator KB sampling | moderator FN on KB (see FP/FN report) |
| Defence Evasion | hidden channel in output; zero-width chars; prompt decorators | yes | H107-C03 layer 4 output sanitiser | adaptive attacks reduce efficacy — tracked in harness |
| Collection | exfil of user data in tool outputs | yes | H107-C09 (arg policy), H107-C06 | — |
| Exfiltration | exfil URL in reply; CRM task with payload | yes | H107-C09 arg policy; moderator on `sensitive_data_leak` | — |
| Impact | wrong reply reaches customer | yes | H107-C01, H107-C12 (HITL), H107-C13 (kill switch) | — |

Any ATLAS technique the crosswalk marks "reachable" with no covering control is a `gap`.

### 2d. NIST AI RMF crosswalk (deliverable 2c)

The AI RMF is a set of functions — GOVERN, MAP, MEASURE, MANAGE. The crosswalk maps the surface's artefacts to each function so a reviewer can see the whole governance loop.

| Function | What RMF asks for | Surface artefacts that satisfy it |
|---|---|---|
| GOVERN | Roles, accountability, policies | Control-owner column in H107-C## table; policy files diff-reviewed; incident-response path (mod-110) |
| MAP | Context, intended use, risk categorisation | [[exercise-01]] (mod-105) diagnosis page; threat model in [[exercise-02]]; data-class inventory in [[exercise-03]] |
| MEASURE | Pre-registered, measurable targets and ongoing evaluation | FP/FN report, AgentDojo-shape report, leak-hunt report, mod-106 eval suite, mod-110 drift monitor |
| MANAGE | Operating response: HITL, kill switches, compensators, review cadence | H107-C09, H107-C10, H107-C12, H107-C13; review cadences across controls |

### 2e. Gap report template (deliverable 3)

```markdown
# Gap report — helpdesk-reply-copilot — v1 — 2026-11-11

| Gap ID | Framework / item | Residual risk | Current compensation | Owner | Target |
|---|---|---|---|---|---|
| G-01 | OWASP ML10 (model poisoning) | upstream provider corrupts a model artefact | H107-C14 provider contract; mod-110 drift monitor would catch | model team | 2026-12-31 (review) |
| G-02 | ATLAS Defence Evasion, adaptive subset of H107-C03 | adaptive attacks reduce security to 68 % on `hidden_channel × memory` | H107-C03 layer 4 output sanitiser upgrade PR; harness blocks merge | security | 2026-11-30 |
| G-03 | NIST MANAGE: compensating action rehearsal | H107-C10 compensators exist but are not drilled | monthly tabletop scheduled | SRE | 2026-12-15 |
| G-04 | OWASP ML02 (data poisoning): KB ingest from public URLs | moderator catches at draft time, not ingest time | ingest-time scan pending | KB team | 2027-01-15 |
```

Every gap has an owner, a current compensation if any, and a date. A gap without a date is a decision not to fix, and must be approved by name.

### 2f. Review cadence and re-mapping procedure (deliverable 4)

```markdown
# Mapping runbook — cadence (v1)

## When the runbook is re-run
- **Scheduled.** Quarterly. Each framework section is re-read against
  the current OWASP / ATLAS / NIST revision. A section whose source
  moved gets a diff committed.
- **Triggered.** The runbook is re-run on:
  - A new tool enters the catalog ([[exercise-04]]).
  - A new memory scope is introduced ([[exercise-02]] mod-105).
  - The surface changes shape (mod-105 exercise-01).
  - The base model is swapped (mod-108).
  - Any incident whose postmortem recommends a mapping update.

## Who signs off
- Platform security owns the crosswalks.
- Each named control owner signs off on the row(s) their control appears in.
- Legal signs off on scrub policy and moderator policy rows.
- Product owns the HITL + kill switch rows.

## Artefacts the runbook produces
- Updated crosswalk tables (above).
- A dated gap report with owners and targets.
- A signed note of the controls whose measurement failed since the
  last run and the remediation taken.
```

### 2g. Implementation notes

- **Two directions.** The crosswalk maps *frameworks → controls* (above) and should also map *controls → frameworks* so each guardrail knows which concerns it carries. The latter is often produced by `make runbook` from the same source.
- **One control, many threats.** A control like H107-C06 (scrubber) legitimately covers several ATLAS tactics *and* several OWASP items. Double-counting is fine; avoiding gaps is the primary goal.
- **Named residuals, not hidden ones.** A residual is a risk the surface *cannot* close locally (e.g. model poisoning). Naming it in the table signals the stakeholder dependency to the reviewer.
- **Reachability matters.** ATLAS has techniques this surface does not reach (e.g. training-side attacks). The crosswalk explicitly marks them unreachable rather than leaving them out — unreachability is a claim that can be challenged.
- **Framework versions move.** Every table has a committed note of the framework revision read; a framework version bump creates its own PR.
- **The runbook is not the system prompt.** None of the above goes into the agent's context. The runbook is for humans; the agent is governed by the controls it references.

## 3. Validation steps

1. The control inventory has a stable ID for each control and names an owner per row.
2. Each control row names an enforcement point in code (a function, a file, or a feature flag) and a measurement.
3. The OWASP ML Top 10 crosswalk covers all ten items and names residuals rather than hiding them.
4. The ATLAS crosswalk marks each listed technique reachable or unreachable with one-line justification; nothing is left blank.
5. The AI RMF crosswalk covers all four functions with artefacts named per function.
6. The gap report has at least one named gap with an owner and a date; "no gaps" is unlikely on a real surface and is itself a flag for shallow mapping.
7. A cadence is declared and the triggers for re-mapping are enumerated.
8. The runbook is referenced from each exercise's SOLUTION / README in this module (cross-ref is the test of whether the runbook is actually operating).

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Control inventory with stable IDs, owners, enforcement points | 1 | ≥ 12 controls; no blanks. |
| Each control has a measurement, not just a description | 1 | Measurement is a named test, report, or cadence. |
| OWASP ML Top 10 crosswalk covers all 10 items | 1 | Every row has at least one control or a named residual. |
| MITRE ATLAS crosswalk enumerates reachable techniques | 1 | Each row marked reachable / unreachable with justification. |
| NIST AI RMF crosswalk covers all 4 functions | 1 | Artefacts named per function. |
| Residuals are explicit, not hidden | 1 | Residual column populated where applicable. |
| Gap report has owners and target dates | 1 | No undated gaps; no un-owned gaps. |
| Cadence defined with triggered re-runs | 1 | Quarterly + trigger list. |
| Sign-off owners named by role | 1 | Not "the team." |
| Runbook cross-referenced from mod-107 exercises | 1 | Each exercise links to the runbook; this file links back. |

A submission scoring < 6 is rejected. Hard failures: (a) the crosswalk hides residuals, (b) no measurement per control, (c) no cadence, (d) an incident-grade framework revision was not read against.

## 5. Common mistakes

- **Treating OWASP as a checklist.** OWASP is a taxonomy for conversation with reviewers; a control-by-control map against a concrete surface is the output, not a tick-sheet.
- **One-way mapping.** Mapping frameworks → controls without the reverse direction means controls do not know what they carry; the first framework-version bump orphans half of them.
- **Hiding residuals.** A residual is the one thing a security reviewer wants to see first; hiding it signals a runbook written to flatter the surface rather than govern it.
- **No measurement column.** A control that cannot be measured is a claim. The exercise explicitly asks for measurement per control.
- **Fresh runbook for each review.** Rewriting the runbook on the day of the review is the failure mode. The runbook is a living artefact, re-run on a cadence, with diffs in git.
- **Overclaiming coverage.** A control whose adaptive security is 68 % does not *close* the threat; it reduces it. Say so.
- **No sign-off owners.** Shared ownership is no ownership. Each row has a name.
- **No trigger list for re-mapping.** A quarterly cadence misses the "we swapped the model yesterday" case; triggers catch it.
- **Letting the runbook diverge from the exercises.** The whole point is that the runbook is the index; if each exercise's SOLUTION doesn't link back, the runbook is not operational.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the authoritative list for ML-sec items ML01–ML10 used in the OWASP crosswalk; the crosswalk cites each item and maps surface-specific controls to it with named residuals for anything upstream.
- MITRE ATLAS — <https://atlas.mitre.org/> — the authoritative tactics-and-techniques framework for adversarial ML; the ATLAS crosswalk picks the reachable techniques on the helpdesk surface and attributes them to controls, with reachability justified per row.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the AI RMF's GOVERN / MAP / MEASURE / MANAGE functions are the four-cell structure this runbook organises artefacts against; the cadence and sign-off rules implement the RMF's governance expectations.

Related exercises: [[exercise-01]] through [[exercise-04]] (each contributes controls to the inventory and is cross-referenced from the crosswalks); upstream [[exercise-01]] (mod-105) diagnosis is the "context" input to NIST MAP; mod-106 eval suite is the measurement half of NIST MEASURE; mod-110 incident path is the MANAGE half.
