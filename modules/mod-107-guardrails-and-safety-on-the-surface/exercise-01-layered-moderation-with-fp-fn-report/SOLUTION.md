### exercise-01: Layered Moderation With FP/FN Report — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to build a **layered moderation pipeline** for the helpdesk-reply-copilot surface and to produce a **false-positive / false-negative report** that is specific enough to be defended in a design review. "Layered" means at least three distinct stages, each with a different mechanism and a different failure mode, composed so a single-layer miss does not become a surface-wide miss. The FP/FN report is the artefact that proves the layers were measured, not merely stacked.

The failure the exercise is written to catch is the single-layer moderation design: one classifier, one threshold, and a vague "we'll tune it in prod" promise. A single layer always picks the wrong operating point for one of the two populations it serves (the flagged-and-wrong user or the unflagged-and-harmed user); layering lets each stage hold the operating point it is actually good at.

This solution provides:

- A **threat and taxonomy map** for the surface — what categories the moderator covers, who the affected populations are, and what the cost of each error type is.
- A **three-layer pipeline design** (static deny/allow rules → category classifier → LLM judge for hard cases) with explicit inputs, outputs, and the hand-off contract between layers.
- A **reference implementation sketch** (Python) showing the composition, the threshold wiring, and the trace hooks the FP/FN report consumes.
- A **FP/FN report template** that reports per-layer and end-to-end rates, broken down by category and population slice, with named operating points and the review cadence.
- A **grading rubric** for the instructor.

The lesson the exercise defends is: *moderation is a calibration problem, not a model problem*. A moderator that is not measured per category, per population, and per layer is not calibrated; it has an opinion. The FP/FN report converts that opinion into a defendable operating point.

## 2. Worked answer or implementation

### 2a. Threat and taxonomy map (deliverable 1)

```markdown
# helpdesk-reply-copilot — moderation taxonomy (v1)

## What the moderator must decide
For each (user message, draft reply) pair, the moderator returns one of:
- `allow` — pass through to the CS agent.
- `soft_block` — show the draft to the CS agent with a banner.
- `hard_block` — do not show the draft; log, offer rewrite affordance.

## Categories covered (surface-specific)
| Category | Example | Who gets harmed if missed | Who gets harmed if over-flagged |
|---|---|---|---|
| `harassment` | insult of the customer | customer; brand | CS agent delayed on a benign reply |
| `self_harm_adjacent` | user mentions suicide in ticket | customer safety | legitimate welfare reply gets blocked |
| `legal_advice` | reply promises a refund we cannot honor | business/legal | reply is slow when customer needs facts |
| `policy_contradiction` | reply contradicts tenant policy | tenant trust | legitimate exception path blocked |
| `sensitive_data_leak` | reply quotes another customer's data | privacy incident | reply drops useful specifics |

## Categories explicitly out of scope on this surface
- Open-domain hate speech on arbitrary input. The surface only reads ticket text and draft reply; content moderation of the broader web is not this surface's job.
- Medical / legal / financial *advice quality*. We detect the shape (claims a refund, cites statute) and route to human; quality judgement is not ours.

## Populations
- `tenant` — each tenant may tune `soft_block` thresholds within a published range; thresholds for `hard_block` are platform-set.
- `locale` — moderation must perform within 20% of English performance on the supported locales (see 2d).
- `role` — CS agent vs tenant admin; thresholds do not vary by role on this surface.
```

The taxonomy is not the OWASP or NIST list copied over; it is the surface-specific derivation that cross-references those lists. See section 6.

### 2b. Three-layer pipeline design (deliverable 2)

Each layer is named by (a) mechanism, (b) what it is good at, (c) what it is bad at, (d) the hand-off to the next layer.

| # | Layer | Mechanism | Strong at | Weak at | Hands off |
|---|---|---|---|---|---|
| 1 | **Deterministic rules** | regex / keyword / allow-deny lists for tenant-specific terms; hard-block list; URL allowlist | tenant-specific promises, forbidden product names, known exfil patterns | anything paraphrased or novel | (text, meta) → layer 2 unless hard-block |
| 2 | **Fast category classifier** | small moderation model or hosted provider API per category; returns probabilities | common category coverage, low latency, consistent calibration | surface-specific contexts, policy contradiction that requires tenant policy context | (text, meta, probs) → layer 3 only if uncertain or policy-sensitive |
| 3 | **LLM judge for hard cases** | structured-output LLM prompted with tenant policy excerpt + category rubric | surface-aware, can cite the policy clause breached | latency, cost, variance; only invoked for borderline or policy-contradiction cases | (text, meta, probs, judge verdict) → router |

The pipeline has an explicit **operating-point configuration** per category:

```yaml
# moderation_policy.yaml (excerpt)
categories:
  harassment:
    layer1_hard_block_terms: [<tenant_hard_block_list.yaml>]
    layer2_threshold_soft: 0.30
    layer2_threshold_hard: 0.80
    layer3_invoked_when: "0.30 <= p < 0.80"
  policy_contradiction:
    layer1_hard_block_terms: []
    layer2_threshold_soft: null   # layer 2 does not score this category
    layer2_threshold_hard: null
    layer3_invoked_when: "always"  # judgement requires tenant policy
```

The yaml is version-controlled; a threshold change is a diff review, not a runtime tweak.

### 2c. Reference implementation sketch (deliverable 3)

```python
"""moderation_pipeline.py — three-layer moderator with trace hooks.

Each layer emits a span to the trace bus so the FP/FN report can
attribute a verdict to the layer that produced it. The final verdict is
the strongest refusal any layer emitted, not a softmax over layers.
"""
from __future__ import annotations

from dataclasses import dataclass, field
from enum import Enum
from typing import Protocol

class Verdict(str, Enum):
    ALLOW = "allow"
    SOFT_BLOCK = "soft_block"
    HARD_BLOCK = "hard_block"

@dataclass(frozen=True)
class ModerationInput:
    user_text: str
    draft_reply: str
    tenant_id: str
    locale: str

@dataclass
class LayerOutput:
    layer: str                      # "rules" | "classifier" | "judge"
    verdict: Verdict
    categories: dict[str, float]    # per-category score; empty for rules
    reasons: list[str] = field(default_factory=list)
    skipped: bool = False

@dataclass
class ModerationResult:
    final: Verdict
    layers: list[LayerOutput]
    # populated by the router below; the FP/FN report keys on these fields
    category_hit: str | None = None
    policy_clause: str | None = None

class Layer(Protocol):
    name: str
    def __call__(self, inp: ModerationInput) -> LayerOutput: ...

def moderate(inp: ModerationInput, layers: list[Layer], policy: dict) -> ModerationResult:
    outs: list[LayerOutput] = []
    strongest: Verdict = Verdict.ALLOW
    hit_cat: str | None = None
    clause: str | None = None
    for layer in layers:
        out = layer(inp)
        outs.append(out)
        if out.skipped:
            continue
        if _rank(out.verdict) > _rank(strongest):
            strongest = out.verdict
            # category attribution: first non-skipped layer that raised the bar wins
            hit_cat = _dominant_category(out, policy)
            clause = _cited_clause(out)
        # fast-exit on hard_block; judge is expensive
        if strongest == Verdict.HARD_BLOCK:
            break
    return ModerationResult(final=strongest, layers=outs,
                            category_hit=hit_cat, policy_clause=clause)

def _rank(v: Verdict) -> int:
    return {Verdict.ALLOW: 0, Verdict.SOFT_BLOCK: 1, Verdict.HARD_BLOCK: 2}[v]
```

Hand-off is encoded by each layer's `skipped` / verdict semantics — the pipeline does not combine probabilities across layers because the layers do not share calibration. Each layer is scored on its own operating point; the pipeline composes verdicts, not scores.

### 2d. The FP/FN report (deliverable 4)

The report is produced against a labeled **holdout set** maintained alongside the moderator; the set is versioned and cannot be edited by the on-call engineer without a reviewer. Minimum content:

```markdown
# Moderation FP/FN Report — helpdesk-reply-copilot — v1.3 — 2026-11-10

## Holdout
- 2,000 (user_text, draft_reply) pairs, labeled by two annotators with a
  third-adjudicator tiebreaker; inter-annotator agreement κ = 0.78.
- Stratified by category, locale, and tenant archetype (internal / SMB
  / enterprise). See `holdout_manifest.yaml` for sampling fractions.

## Operating points (committed before measurement)
- `hard_block` end-to-end FP target ≤ 0.5 % per category; FN ≤ 2 %.
- `soft_block` end-to-end FP target ≤ 5 %; FN ≤ 10 %.
- No single-slice (category × locale) may exceed 2× the overall rate.

## Per-layer coverage
| Layer | Fires on | Inputs scored | Produces hard_block | Produces soft_block |
|---|---|---|---|---|
| 1 — rules | all | 2000 | 23 | 58 |
| 2 — classifier | post-rules | 1919 | 61 | 198 |
| 3 — judge | borderline / policy | 341 | 14 | 73 |

## End-to-end confusion (holdout, counts and rates)
| Category | TP | FP | TN | FN | Precision | Recall |
|---|---|---|---|---|---|---|
| harassment | 71 | 4 | 418 | 7 | 0.95 | 0.91 |
| self_harm_adjacent | 22 | 1 | 475 | 2 | 0.96 | 0.92 |
| legal_advice | 44 | 9 | 440 | 7 | 0.83 | 0.86 |
| policy_contradiction | 28 | 11 | 436 | 25 | 0.72 | 0.53 |
| sensitive_data_leak | 19 | 2 | 477 | 2 | 0.90 | 0.90 |

## Layer attribution (where the final verdict came from)
| Category | Layer 1 | Layer 2 | Layer 3 |
|---|---|---|---|
| harassment | 18 % | 71 % | 11 % |
| self_harm_adjacent | 5 %  | 83 % | 12 % |
| legal_advice | 38 % | 24 % | 38 % |
| policy_contradiction | 0 %  | 8 %  | 92 % |
| sensitive_data_leak | 42 % | 48 % | 10 % |

## Slice breakdowns (abbreviated)
- Locale `pt-BR`: `harassment` recall 0.77 (English 0.91) — **fails 20 % threshold**;
  action: add 80 Portuguese-labeled examples by 2026-12-01 and re-run.
- Tenant archetype `enterprise`: `policy_contradiction` FP 0.08 (overall 0.02) —
  within policy; attributed to tenant policies that heavily reference internal
  product names; no action.

## Known residual risks
- Layer 1 depends on the tenant hard-block list being up to date; owner-of-list
  review cadence: 30 days; last review 2026-10-30.
- Layer 3 LLM judge has model-version drift risk; see mod-110 drift monitor.

## Review cadence
- Holdout refresh: quarterly or on any new category added.
- Thresholds: reviewed when any slice crosses the 2× guardrail.
- Operating-point change: diff review + report re-run; no runtime edits.
```

The report lives next to the policy file; a PR that moves a threshold cannot merge without re-running the report and committing it.

### 2e. Implementation notes

- **Why three layers and not two.** Two is adequate on the median case; the third layer exists specifically to handle policy-contradiction, which cannot be classified without the tenant policy excerpt. Dropping layer 3 moves the policy-contradiction FN rate from 0.53 recall to ~0.20 in the reference data; the rate is defended in the report, not asserted in the design.
- **Why verdicts compose and scores do not.** Each layer is calibrated separately, often on different populations; averaging probabilities across layers is a common silent regression. Composing verdicts (take the strongest) keeps each layer's operating point intact and attributable.
- **Why the judge is last.** Judges are the most expensive layer. Placing them last is a cost choice *and* a failure-mode choice: a judge mis-fire can only *tighten* a verdict the earlier layers didn't catch, never loosen one.
- **Why the policy is YAML, not code.** Thresholds are reviewed by non-engineers (policy, legal). YAML gives them a diff they can read. A code-only policy cannot be reviewed by the roles that own the operating points.
- **What the model never sees.** The user's PII (see [[exercise-03]]) is scrubbed before layer 2 and layer 3; layer 1 may see raw text because it is deterministic and local. A moderator is not a safe PII channel.
- **How this layer stack fits the next exercise.** [[exercise-02]] runs the prompt-injection evaluator *after* the moderator; the moderator does not try to detect injection by itself.

## 3. Validation steps

1. The taxonomy names at least four surface-specific categories *and* at least one category that is explicitly out of scope with a reason.
2. Three layers are defined with distinct mechanisms. Two layers of "LLM with different prompt" is one layer in two costumes.
3. The hand-off between layers is explicit; no layer's output feeds the next as "inputs" without a named contract (verdict, probs, reasons).
4. The operating points are committed *before* the report is run. A report that tunes thresholds against its own holdout is a leakage incident; the file history should show thresholds set in an earlier commit than the report.
5. The holdout is labeled by at least two annotators with a reported κ; a single-annotator set is unusable for the FP/FN report.
6. The report reports per-category *and* per-slice rates; a single aggregate rate is a report that will hide a locale regression on day one.
7. Per-layer attribution is reported. If the report cannot say which layer produced each verdict, the stack is not instrumented and the next round of tuning is guessing.
8. The report lists at least one known residual risk and the review cadence.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Surface-specific taxonomy with populations and costs | 1 | Surface-derived, not an OWASP copy. Includes both FP and FN costs per category. |
| Explicit out-of-scope categories with reasons | 1 | At least one "not us" item with a one-line justification. |
| Three layers with distinct mechanisms | 1 | Rules vs classifier vs judge or equivalent; not three LLMs. |
| Hand-off contract named between layers | 1 | Inputs/outputs typed; verdicts compose, scores don't cross. |
| Policy is version-controlled and reviewable | 1 | YAML (or equivalent) with per-category thresholds and a reviewer named. |
| Holdout is labeled by ≥ 2 annotators with κ | 1 | κ ≥ 0.6 and labeled population documented. |
| FP/FN report has per-category and per-slice rates | 1 | ≥ 2 slice axes (e.g. locale × category) reported. |
| Per-layer attribution reported | 1 | Final-verdict source traceable to a single layer. |
| Operating points committed pre-measurement | 1 | Commit history or a signed "operating point" block present. |
| Review cadence + residual risks listed | 1 | At least one risk owned by name and one cadence named. |

A submission scoring < 6 is rejected. Hard failures: (a) one-layer stack, (b) report without per-category breakdown, (c) thresholds tuned on the holdout set.

## 5. Common mistakes

- **Stacking three LLM layers and calling it defence-in-depth.** Three LLMs with different prompts share failure modes (model-version drift, jailbreak surface). Defence-in-depth needs *different* mechanisms, not different wrappers on the same mechanism.
- **Reporting a single accuracy number.** An aggregate accuracy hides the one category that costs the business the most. Per-category plus per-slice is the floor.
- **Tuning thresholds on the holdout.** The holdout exists to measure the operating point committed in advance. Tuning against it turns the report into self-grading.
- **No per-layer attribution.** When an incident fires, you need to know *which* layer let the input through. A pipeline without per-layer traces answers "all of them" and nobody learns anything.
- **Over-flagging as the safe choice.** False positives are not free: they train the CS agents to ignore the banner. Each soft-block that is later ignored by humans is the next iteration's prompt-injection carrier.
- **Treating "hard_block" as the default for anything sensitive.** A hard-block is a UX event. If the category is "legal advice shape," a soft-block with a rewrite affordance is almost always the correct product choice; hard-block is reserved for outputs that must not reach the CS agent.
- **Letting the model write to the policy file.** Policy is human-authored. The moderator is a consumer of the policy, not an author.
- **Labeling only in English.** A moderator that is only measured on English will quietly regress on every other locale. The slice section exists so the regression is seen before it ships.
- **No review cadence.** A policy without a cadence is a policy that drifts. The cadence is the thing that keeps the operating point defensible six months out.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the surface-specific taxonomy categories (`sensitive_data_leak`, `policy_contradiction`) are the surface-level derivatives of OWASP ML categories such as ML02 (Data Poisoning) and ML07 (Transfer Learning Attack) applied to the moderator's inputs; the OWASP list is the upstream reference the taxonomy cites, not copies.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the FP/FN report's "operating points committed before measurement" discipline maps to NIST AI RMF's MEASURE function (measurable, pre-registered performance targets with documented review cadence). The per-slice breakdown implements the "measure across populations" requirement.
- MITRE ATLAS — <https://atlas.mitre.org/> — the moderator is one of the defences an ATLAS-informed threat model is scored against; the per-layer attribution section exists so an ATLAS review can identify which defence class failed when an input slipped through.

Related exercises: [[exercise-02]] (prompt-injection evaluator runs after this moderator and treats its verdict as one of its signals); [[exercise-03]] (PII scrub runs *before* the moderator's LLM-based layers); [[exercise-04]] (destructive-action gates share the "composable verdict" discipline); [[exercise-05]] (this moderator is one of the controls the OWASP/ATLAS mapping runbook catalogs).
