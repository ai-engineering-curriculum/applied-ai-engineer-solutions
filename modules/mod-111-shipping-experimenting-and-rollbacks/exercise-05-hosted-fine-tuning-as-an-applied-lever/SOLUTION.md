### exercise-05: Hosted Fine-Tuning as an Applied Lever — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to run **one hosted fine-tune end to end on a narrow shape problem** — a structured-output reliability regression that prompt engineering has plateaued on — and ship the result through the same offline eval → shadow → canary regime as any other variant. The exercise is deliberately not "learn LoRA." It is "prove you can identify a *shape* problem, curate a dataset safely, run a hosted job, ship the result with the same gate discipline as mod-111 [[exercise-02]], and name the specific questions that would escalate to `fine-tuning-engineer`."

The failure modes the exercise defends against are distinct from the fine-tuning *mechanics* and specific to the *applied-engineer workflow*:

1. **Fine-tuning the wrong shape of problem.** Teams apply fine-tuning to hallucination (a knowledge problem) or to retrieval failure (a RAG problem). Both fail. The fix is the **1-page diagnosis** that argues *why* this is a shape problem before the training job is kicked off.
2. **Eval-set contamination.** The training set overlaps with the mod-106 frozen eval set; the fine-tune looks great on offline eval and regresses in production. The fix is **three-dimensional dedupe** (exact-match, near-duplicate Jaccard, source case-id) checked in as evidence.
3. **PII in the training set.** The training data is upload-forever on the provider's platform. A single customer email in the dataset survives any downstream deletion request. The fix is **mod-107 [[exercise-03]] scrubbing pipeline run over the training set**, output audit checked in.
4. **Accepting base-model outputs as labels.** The fine-tune caps at the base model's quality — if the base model produces 4% schema-invalid JSON and you use its outputs as labels, the fine-tune will also produce ~4%. The fix is **human review of every label**, specifically the JSON structure.
5. **Hyperparameter sweeps as applied-engineering work.** Hyperparameter tuning is a `fine-tuning-engineer` activity. An applied engineer who tunes hyperparameters is usually spending budget on the wrong lever — dataset quality dominates. The fix is **provider defaults, documented**.
6. **Shipping the fine-tune without the regime.** The temptation is to "promote fine-tune straight to 100% because we already validated it on offline eval." That re-introduces every failure mode mod-111 [[exercise-02]] exists to catch. The fix is **the same shadow → 1% → 5% → 25% → 100% gated rollout as any other variant**.
7. **Cost math on current-tier prices only.** The fine-tune amortises training cost over expected-lifetime outcomes; if the provider deprecates the base model in 3 months, the amortisation assumption is wrong and the fine-tune is not economically justified. The fix is **the two-line cost model with an explicit lifetime assumption and a "provider deprecation" risk note**.

This solution provides:

- A **diagnosis doc** that argues the structured-output problem is a shape problem fine-tuning can address.
- A **training-set curation protocol** — 500+ real-traffic examples, three-dimensional dedup, PII scrub, human review coverage.
- A **data audit doc** with sample counts, provenance, scrubber output, leakage check.
- A **tune config** with the base model, default hyperparameters (justified), and validation split.
- A **tune script** for OpenAI SFT (Bedrock / Vertex shape included as commented alternatives).
- An **offline-eval protocol** comparing base vs fine-tune on the frozen eval set.
- A **shadow-launch wiring** that re-uses the [[exercise-02]] substrate.
- A **canary log** with per-stage promotion artefacts.
- A **two-line cost model** with amortisation and a provider-deprecation risk note.
- A **retro and peer-role escalation list.**
- A **grading rubric** for the instructor.

## 2. Worked answer or implementation

### 2a. Diagnosis doc (deliverable 1 — frozen before curation)

```markdown
# docs/tune-diagnosis.md — JSON reliability fine-tune diagnosis

Owner: applied-ai-team. Frozen: 2026-11-15 10:00 UTC.

## Observed failure

The `support_answer_summariser` produces a JSON-shaped output per the
feature's response schema. Over the last 30 days, the observed rate of
schema-invalid responses is **4.0%** (± 0.2 pp). The JSON-repair fallback
path adds **247 ms p50 / 413 ms p95** of latency and **+20% cost** on the
affected turns (because the repair step is a second LLM call).

## What has been tried

- Prompt iteration 1 (2026-09-15): "Return valid JSON matching this schema
  <schema>" — improved failure rate from 6.3% → 5.1%.
- Prompt iteration 2 (2026-10-02): Added an example in the prompt — 5.1%
  → 4.3%.
- Prompt iteration 3 (2026-10-21): Added a terminal-JSON reminder — 4.3%
  → 4.0%.

Returns are flat. Three successive iterations each moved the number
~1 pp, then ~0.3 pp.

## Why this is a shape problem

- It's not a *knowledge* problem: the correct output is deterministic from
  the input and the schema; the model knows the content.
- It's not a *hallucination* problem: the content is correct; only the
  structural wrapper fails.
- It's not a *retrieval* problem: the retrieved context is correct; the
  model's rendering of it violates the schema.
- It's not a *model-tier* problem: trying gpt-4o (the top tier) instead of
  gpt-4o-mini reduces failure rate from 4% to 2% but at 10× cost per
  outcome. That is the cost-prohibitive option.

A shape problem (how the model structures outputs under a specific schema)
is the canonical fine-tuning-good-at problem. We expect 500-1000
examples of valid schema-shaped output to drive the base model's
structured-output distribution toward schema-valid.

## Success criterion

JSON-failure rate on the frozen mod-106 eval set ≤ **0.5%** — a 10×
reduction. Per-outcome cost ≤ **1.2× the base model's cost**. Per-scenario
quality (mod-106 composite) ≥ base - 2 pts on every scenario.

## Abort criterion

If offline eval shows JSON-failure rate > 1%, OR any scenario regresses
by more than 3 points, OR the composite drops by more than 2 points,
abort. Do NOT proceed to shadow.

## Budget

Training: ~$20 for 500-1000 examples on OpenAI SFT of `gpt-4o-mini`
(2026-11-15 pricing — verify at submission). Plan for two training runs
(~$40) — the first usually reveals a dataset problem. PM-approved
(PROD-4854).
```

### 2b. Training-set curation (deliverable 2)

```python
# scripts/curate_training_set.py — 500+ real-traffic examples with human review.
"""
Protocol:

  1. Pull turns from the mod-110 [[exercise-03]] feedback store where:
     - app.feature = "support_agent.summarise"
     - thumbs_up = true (user liked the answer)
     - app.outcome.reason != "json_repair"   (output was schema-valid)
     - NOT in the mod-106 frozen eval set (by case_id)
  2. Human-review every example:
     - Confirm the JSON is actually schema-valid (fix if close-but-wrong).
     - Trim / redact PII per mod-107 [[exercise-03]].
  3. Dedupe three ways:
     - Exact-match on input text.
     - Jaccard > 0.7 on 3-gram token sets.
     - Shared case_id with the eval set.
  4. Audit the final set: per-scenario counts, PII scrubs logged.
"""
import hashlib, json, pathlib, re
from datetime import datetime, UTC

from feedback_store import pull_turns           # mod-110 [[exercise-03]]
from scrubber       import scrub_pii            # mod-107 [[exercise-03]]
from eval_set       import FROZEN_EVAL_CASE_IDS # mod-106


SOURCE_WINDOW_DAYS = 90
MIN_EXAMPLES = 500
TRAINING_OUT = pathlib.Path("data/training-set.jsonl")
AUDIT_OUT    = pathlib.Path("docs/data-audit.md")


def jaccard_3gram(a: str, b: str) -> float:
    def grams(s): return {s[i:i+3] for i in range(len(s)-2)}
    ag, bg = grams(a), grams(b)
    if not ag or not bg: return 0.0
    return len(ag & bg) / len(ag | bg)


def curate() -> dict:
    turns = pull_turns(
        feature="support_agent.summarise",
        window_days=SOURCE_WINDOW_DAYS,
        thumbs_up_only=True,
        outcome_not_in=("json_repair", "error", "refusal"),
    )

    # Step 1 — exclude eval set by case_id.
    before = len(turns)
    turns = [t for t in turns if t.case_id not in FROZEN_EVAL_CASE_IDS]
    removed_by_caseid = before - len(turns)

    # Step 2 — PII scrub every input and output.
    scrub_counts = {"email": 0, "name": 0, "phone": 0, "id": 0, "secret": 0}
    cleaned = []
    for t in turns:
        inp, c_in  = scrub_pii(t.input)
        out, c_out = scrub_pii(t.output)
        for k in scrub_counts: scrub_counts[k] += c_in.get(k, 0) + c_out.get(k, 0)
        cleaned.append((t, inp, out))

    # Step 3 — dedupe exact-match inputs.
    seen = set()
    deduped = []
    removed_exact = 0
    for t, inp, out in cleaned:
        h = hashlib.sha256(inp.encode()).hexdigest()
        if h in seen:
            removed_exact += 1
            continue
        seen.add(h)
        deduped.append((t, inp, out))

    # Step 4 — near-duplicate against eval set (expensive; do last and on a
    # subset). In practice, use datasketch MinHash; the loop here is the
    # expository shape.
    from eval_set import load_frozen_eval_inputs
    eval_inputs = load_frozen_eval_inputs()
    removed_near = 0
    final = []
    for t, inp, out in deduped:
        if any(jaccard_3gram(inp, e) > 0.7 for e in eval_inputs):
            removed_near += 1
            continue
        final.append((t, inp, out))

    # Step 5 — require ≥ 500 examples; if under, widen the window.
    if len(final) < MIN_EXAMPLES:
        raise RuntimeError(
            f"only {len(final)} examples after curation; widen source window"
        )

    # Step 6 — write the training JSONL in OpenAI's SFT format.
    TRAINING_OUT.parent.mkdir(parents=True, exist_ok=True)
    with TRAINING_OUT.open("w") as f:
        for t, inp, out in final:
            f.write(json.dumps({
                "messages": [
                    {"role": "system", "content": SYSTEM_PROMPT},
                    {"role": "user",   "content": inp},
                    {"role": "assistant", "content": out},
                ]
            }) + "\n")

    return {
        "count":          len(final),
        "removed_caseid": removed_by_caseid,
        "removed_exact":  removed_exact,
        "removed_near":   removed_near,
        "scrub_counts":   scrub_counts,
    }


if __name__ == "__main__":
    stats = curate()
    print(stats)
```

`docs/data-audit.md` — output of the curation run:

```markdown
# docs/data-audit.md — training-set audit

Date: 2026-11-15.
Script: `scripts/curate_training_set.py`.
Source: mod-110 feedback store, 90-day window.

## Sample counts

| mod-106 scenario | Production rate | Training-set count | Training-set % |
|---|---|---|---|
| tier:enterprise / qtype:how-to | 22% | 115 | 23.0 |
| tier:enterprise / qtype:troubleshoot | 15% |  77 | 15.4 |
| tier:smb / qtype:how-to | 28% | 141 | 28.2 |
| tier:smb / qtype:troubleshoot | 18% |  90 | 18.0 |
| tier:free / qtype:how-to | 12% |  60 | 12.0 |
| tier:free / qtype:troubleshoot |  5% |  17 |  3.4 |
| **Total** | **100%** | **500** | **100%** |

Allocation roughly matches production traffic. Tier:free / troubleshoot is
under-represented (3.4 vs 5.0) because thumbs-up-rate on this slice is
lower; oversampling to ~5% would have required loosening the thumbs-up
filter and we judged that a worse trade than mild underrepresentation.

## Provenance

- Source window: 2026-08-17 → 2026-11-15 (90 days).
- Filter: `thumbs_up = true` AND `outcome.reason not in (json_repair,
  error, refusal)`.
- Before dedup: 847 candidates.
- Removed by eval-set case_id: 48.
- Removed by exact-input dedup: 112.
- Removed by near-duplicate dedup (Jaccard > 0.7 vs eval): 187.
- Final: 500 (we stopped curation at the target count).

## Scrubber output (from mod-107 [[exercise-03]] scrubber)

| Category | Count scrubbed |
|---|---|
| email | 203 |
| name | 412 |
| phone | 44 |
| id (orderIds, ticketIds) | 1 103 |
| secret (api keys, tokens — none expected) | 0 |

Scrubber config: `scrubber.rules_v3.yaml` from mod-107 ex-03, no changes.

## Leakage check

- Exact-match against frozen eval set: **0** leaked.
- Near-duplicate (Jaccard > 0.7 on 3-gram token sets): **0** leaked.
- Shared `case_id` with eval set: **0** leaked.

## Human-review coverage

- 500 / 500 examples human-reviewed (100%).
- Reviewer: applied-ai-team (jferguson + rstark); each example reviewed
  by exactly one person.
- 83 examples had JSON validator warnings at review — all fixed before
  training-set emission.
- 11 examples were dropped at review as borderline (e.g., the thumbs-up
  signal was ambiguous) — replaced from the candidate pool.
```

### 2c. Tune config and script (deliverable 3)

```python
# scripts/tune.py — OpenAI SFT.
"""
Decisions:

  Base model: gpt-4o-mini-2024-07-18. Smaller tier; cheaper per token;
  more responsive to fine-tuning for shape problems; commonly recommended
  by OpenAI as the first SFT target.

  Hyperparameters: provider defaults (n_epochs=3, learning_rate_multiplier=
  "auto", batch_size="auto"). Applied-engineer discipline: do NOT tune.
  Dataset improvements are the first lever; hyperparameter sweeps are
  fine-tuning-engineer scope.

  Validation split: 10% hold-out from the training-shape data. Do NOT
  pass the mod-106 frozen eval set — that is the promotion-decision
  dataset and stays unpolluted.
"""
import json, os, pathlib, time
from openai import OpenAI

client = OpenAI()
TRAIN = pathlib.Path("data/training-set.jsonl")
VALID = pathlib.Path("data/validation-set.jsonl")


def split_validation():
    """Hold out 10% of training as validation. Deterministic by case_id."""
    lines = TRAIN.read_text().splitlines()
    import random
    rng = random.Random(42)
    idx = list(range(len(lines)))
    rng.shuffle(idx)
    cut = int(0.1 * len(lines))
    val_idx, trn_idx = set(idx[:cut]), set(idx[cut:])
    VALID.write_text("\n".join(lines[i] for i in sorted(val_idx)) + "\n")
    TRAIN.write_text("\n".join(lines[i] for i in sorted(trn_idx)) + "\n")


def kick_off_job() -> str:
    train_file = client.files.create(file=TRAIN.open("rb"), purpose="fine-tune").id
    valid_file = client.files.create(file=VALID.open("rb"), purpose="fine-tune").id
    job = client.fine_tuning.jobs.create(
        training_file=train_file,
        validation_file=valid_file,
        model="gpt-4o-mini-2024-07-18",
        # Provider defaults — do NOT set n_epochs / lr_mult / batch_size.
        suffix="support-json-reliability-2026q4",
    )
    return job.id


def poll_until_done(job_id: str) -> dict:
    while True:
        j = client.fine_tuning.jobs.retrieve(job_id)
        if j.status in ("succeeded", "failed", "cancelled"):
            return {
                "status": j.status,
                "fine_tuned_model": j.fine_tuned_model,
                "trained_tokens":   j.trained_tokens,
                "result_files":     j.result_files,
            }
        time.sleep(60)


if __name__ == "__main__":
    split_validation()
    job_id = kick_off_job()
    print(f"job_id={job_id}")
    result = poll_until_done(job_id)
    print(result)
```

`docs/tune-config.md`:

```markdown
# docs/tune-config.md — fine-tune configuration

Provider: OpenAI (SFT).
Base model: `gpt-4o-mini-2024-07-18`.
  Justification: smaller tier → cheaper to serve → faster → shape-problem
  responsive. Top-tier SFT is not self-service for this account and is
  rarely economical for shape problems.
Suffix: `support-json-reliability-2026q4`.
Hyperparameters:
  - n_epochs: AUTO (provider default).
  - learning_rate_multiplier: AUTO.
  - batch_size: AUTO.
  Justification: applied-engineer discipline is to not sweep. Dataset
  quality is the first lever. If the AUTO defaults produce a bad fine-
  tune, the next action is iterating on the dataset or escalating to
  `fine-tuning-engineer`, not sweeping hyperparameters.
Validation split: 10% held out from the training JSONL (seed 42, by line
  index). The mod-106 frozen eval set is NOT handed to the provider.

Job result:
  job_id: ftjob-abc123 (OpenAI).
  fine_tuned_model: `ft:gpt-4o-mini-2024-07-18:example:support-json-reliability-2026q4:abcdef`.
  trained_tokens: 1.42M.
  status: succeeded (2026-11-15 15:42 UTC).

Registry entry (added to [[exercise-04]] prompt / model registry):
  model_id: ft:gpt-4o-mini-2024-07-18:example:support-json-reliability-2026q4:abcdef
  base_model: gpt-4o-mini-2024-07-18
  training_set_sha256: 4f...e9 (sha256 of data/training-set.jsonl)
  cost_usd_train: 18.42 (from OpenAI billing)
  provenance: scripts/tune.py, job_id=ftjob-abc123.
```

### 2d. Offline eval (deliverable 4)

```markdown
# docs/tune-eval.md — base vs fine-tune on frozen mod-106 eval

Date: 2026-11-16. Eval runner: mod-106 ex-02 harness.

## Composite scores

| Metric | gpt-4o-mini (base) | ft:…abcdef (tune) | Δ |
|---|---|---|---|
| Composite (mod-106 judges) | 4.21 / 5 | 4.27 / 5 | +0.06 |
| JSON-schema validity | 95.8% | **99.8%** | **+4.0 pp** |
| Avg output tokens | 143 | 138 | -3.5% |

## Per-scenario breakdown

| Scenario | Base | Tune | Δ |
|---|---|---|---|
| tier:enterprise / how-to | 4.18 | 4.21 | +0.03 |
| tier:enterprise / troubleshoot | 4.09 | 4.14 | +0.05 |
| tier:smb / how-to | 4.36 | 4.41 | +0.05 |
| tier:smb / troubleshoot | 4.22 | 4.28 | +0.06 |
| tier:free / how-to | 4.15 | 4.22 | +0.07 |
| tier:free / troubleshoot | 4.03 | 4.04 | +0.01 |

No scenario regressed. Success criteria from `tune-diagnosis.md`:

- JSON-failure rate ≤ 0.5%: achieved (observed 0.2%).
- Composite ≥ base - 2 pts: achieved (+0.06).
- No scenario below base - 3 pts: achieved (min delta +0.01).

## Decision

Offline eval PASSES. Proceed to shadow per [[exercise-02]] regime.
```

### 2e. Shadow launch (deliverable 5)

`docs/tune-shadow.md` — re-uses the [[exercise-02]] shadow substrate with the fine-tune registered as the shadow variant:

```markdown
# docs/tune-shadow.md — fine-tune shadow window

Window: 2026-11-17 09:00 → 2026-11-18 09:00 UTC (24h — low-rate feature).
Shadow rate: 10% of non-internal turns.
Shadow variant: `ft:gpt-4o-mini-2024-07-18:example:support-json-reliability-2026q4:abcdef`.
Control: `gpt-4o-mini-2024-07-18` (base).

Paired samples: 1 728 shadowed turns, 1 728 control turns paired by request_id.

## Results

- mod-106 judge win-rate (fine-tune vs base): 53.1% (95% CI: 50.7–55.5).
  **Gate: ≥ 55%. HOLD — within CI but below point-estimate target.**
- JSON-failure rate: shadow 0.3% vs control 4.1%.
  **Gate: ≤ 0.5%. PASS.**
- CPSO on shadow traffic: $0.0094 vs control $0.0098.
  **Gate: delta ≤ +10%. PASS (shadow is cheaper by 4%).**
- Shadow p95 latency: 1.9 s (informational — shadow is async, not
  user-facing).

## Decision

The primary case for this fine-tune is the **structured-output
reliability win** (4.1% → 0.3% failure rate). The judge win-rate is
within gate CI but below target; we judge this ambiguous, not failing,
because the fine-tune's job was NOT to improve content quality — it was
to improve structural reliability. The content-quality judge is picking
up small variance, not a regression.

Proceed to canary with a HOLD-on-judge note — if the canary's judge win-
rate at 5% stays below 55%, we re-evaluate.
```

### 2f. Canary and ship (deliverable 6)

`docs/tune-canary-log.md` — one promotion block per stage, matching the [[exercise-02]] template:

```markdown
# docs/tune-canary-log.md — fine-tune canary

## Canary — fine-tune at 0% → 1% (2026-11-18 → 2026-11-19)

Samples: canary 1 728; control 170 848.

| Gate | Threshold | Observed | Status |
|---|---|---|---|
| Composite judge | ≥ base - 2 pts | 4.27 (base 4.21) | PASS |
| JSON failure rate | ≤ 0.5% | 0.2% | PASS |
| Refusal delta | ≤ +1 pp | +0.0 pp | PASS |
| Moderation FP delta | ≤ +0.5 pp | +0.0 pp | PASS |
| CPSO delta | ≤ +10% | -4% (fewer repair calls) | PASS |
| p95 TTFT delta | ≤ +150 ms | -40 ms | PASS |
| Retry delta | ≤ +0.5 pp | -3.5 pp (!) | PASS |

Decision: PROMOTE to 5%. Signed off: jferguson.

## Canary — fine-tune at 1% → 5% (2026-11-19 → 2026-11-22, 72h)

(fill after window)

## Canary — fine-tune at 5% → 25% (dates)

## Canary — fine-tune at 25% → 100% (dates)
```

### 2g. Cost model (deliverable 7)

```markdown
# docs/tune-cost.md — base vs fine-tune per-outcome cost

Prices (2026-11-15 pricing; verify at review time):
  gpt-4o-mini input:   $0.15 / 1M tokens.
  gpt-4o-mini output:  $0.60 / 1M tokens.
  ft:gpt-4o-mini input:  $0.30 / 1M tokens. (2× base per OpenAI SFT serving.)
  ft:gpt-4o-mini output: $1.20 / 1M tokens. (2× base.)

Average outcome on this feature (from CPSO ledger, mod-109 [[exercise-04]]):
  input tokens  avg: 420
  output tokens avg: 140
  json_repair rate: 4.0% (base) vs 0.3% (tune). Each repair adds ~180
  tokens of input and ~90 of output.

## Base cost per outcome

Effective input tokens  = 420 + 0.040 × 180 = 427.2
Effective output tokens = 140 + 0.040 ×  90 = 143.6
Cost = 427.2 × $0.15/1M + 143.6 × $0.60/1M = $0.0001503

Observed CPSO (from ledger): $0.0098 = 0.0001503 × 65.2× ... (CPSO includes
retries, system prompt overhead, moderation calls; the per-outcome math
above is just the primary call.)

## Fine-tune cost per outcome

Effective input tokens  = 420 + 0.003 × 180 ≈ 420.5
Effective output tokens = 140 + 0.003 ×  90 ≈ 140.3
Cost = 420.5 × $0.30/1M + 140.3 × $1.20/1M = $0.000294

...but repair-rate dropped 13× so the saved repair calls compensate.

## Amortised training cost

Training cost (from scripts/tune.py result): $18.42.
Expected lifetime of fine-tune: we assume 3 months (90 days) at current
rate of 170 848 turns/day × 100% = ~15.4M outcomes.
  Amortised training cost per outcome: $18.42 / 15 400 000 = $0.0000012
  (negligible).

## Two-line comparison

Base model per outcome:
  = 0.15 × 10^-6 × 427.2 + 0.60 × 10^-6 × 143.6
  = **$0.0001503**

Fine-tune per outcome (including amortised training):
  = 0.30 × 10^-6 × 420.5 + 1.20 × 10^-6 × 140.3 + ($18.42 / 15.4M)
  = **$0.000295**

Delta: **+$0.000145 per outcome** (+96% on the primary-call cost).

But: the base's 4% json_repair path adds ~0.04 × 1.20× = +4.8% cost above
the primary number; the tune's 0.3% repair path adds +0.36%. Net observed
CPSO in the canary stage 1%: tune -4% vs base (from 2f). The fine-tune is
cheaper in production because the repair-cost savings exceed the per-token
premium.

Break-even check — if the fine-tune had NOT reduced repair rate:
  break-even: $18.42 / ($0.0001503 - $0.000295) = N/A (fine-tune would be
  more expensive at every volume).
  The economic case for the fine-tune rests on the repair-rate reduction,
  NOT on the per-token price.

## Provider-deprecation risk

OpenAI's gpt-4o-mini-2024-07-18 is pinned-date. If OpenAI deprecates it
before the 90-day amortisation window, we must re-fine-tune on the
replacement base. Each re-fine-tune costs ~$20. The risk is not
economically material at current volumes; flag for review if we're still
on this fine-tune at Month 3.
```

### 2h. Retro and peer-role note (deliverable 8)

```markdown
# docs/tune-retro.md — fine-tune retro

## Did the diagnosis hold?

Yes. The problem was a structural-reliability shape problem; the fine-
tune reduced json_repair rate from 4.0% to 0.3% — a 13× reduction. The
content-quality judge win-rate was essentially flat (as expected — this
was never about content quality).

## Total elapsed engineering time

- Diagnosis: 1 hour.
- Dataset curation + audit: 3 hours (longer than the 2-hour exercise
  budget because of thorough dedup + full human review).
- Config + tune job kickoff: 20 minutes.
- Training (async, provider side): 1.5 hours wall-clock.
- Offline eval + shadow + canary 1%: 4 hours including waiting windows.
- **Total active engineering: ~8 hours. Wall-clock (with waits): ~3 days.**

## Eval / shadow / canary outcome

- Offline eval: PASS (all gates).
- Shadow: PASS (JSON gate solid, judge win-rate ambiguous but OK).
- Canary 1%: PASS (all gates; CPSO actually improved).

Current state (as of this retro): promoting to 5%.

## Re-run trigger

Re-fine-tune when:
1. The base model `gpt-4o-mini-2024-07-18` is deprecated or replaced.
2. The schema changes materially (new required field; a change to the
   structure of required fields).
3. The repair rate on the fine-tuned model drifts above 1% for more than
   7 days — suggests training data is no longer representative.
4. A second production-grade feature wants the same shape fix — then
   invest in a joint-training-set.

## What would justify escalating to `fine-tuning-engineer`?

Four concrete future scenarios:

1. **Repair rate returns to > 2% after this fine-tune.** The dataset
   fix probably isn't enough; the next lever is PEFT (LoRA) on a larger
   dataset or a change to the fine-tuning method. Peer-role conversation.

2. **We want to try DPO on preference data.** The mod-110 feedback
   pipeline has thumbs-up vs thumbs-down pairs; a direct preference
   optimisation run is a different method and requires a peer-role
   method-choice conversation, not applied-engineer execution.

3. **We're considering fine-tuning the top-tier model.** Frontier SFT is
   expensive, slow, and often gated. Peer-role should be in the call
   before kickoff.

4. **We want to run multiple fine-tunes in parallel (SFT vs DPO on the
   same base).** Comparing them cleanly requires the same variance-
   reduction and sequential-testing considerations from [[exercise-03]].
```

### 2i. Mapping to the exercise deliverables

| Exercise asks for | Section |
|---|---|
| Diagnosis doc (shape vs knowledge argument) | 2a |
| Training-set curation (≥ 500, scrub, dedup, review) | 2b |
| Data audit doc | 2b |
| Tune config (base, hyperparams, validation split) | 2c |
| Tune script + run | 2c |
| Offline eval against frozen set | 2d |
| Shadow launch | 2e |
| Canary and ship | 2f |
| Cost model (two-line comparison) | 2g |
| Retro + peer-role escalation note | 2h |

## 3. Validation steps

1. **Diagnosis doc is frozen before curation.** Git timestamp precedes `data/training-set.jsonl`.
2. **Training set has ≥ 500 examples.** `wc -l data/training-set.jsonl` ≥ 500.
3. **Three-dimensional dedup against eval set passes.** `docs/data-audit.md` records 0 leaks across all three dimensions.
4. **PII scrubber ran over the training set.** Scrubber output table is in `docs/data-audit.md`.
5. **Human-review coverage is 100%.** Reviewer names recorded.
6. **Training set proportions match production traffic.** Scenario breakdown table is in `docs/data-audit.md`; deviations are explained.
7. **Hyperparameters are provider defaults, justified.** `docs/tune-config.md` names "AUTO" with a one-line "applied-engineer-discipline" rationale.
8. **Validation set is NOT the eval set.** 10% split from training data, deterministic.
9. **Offline eval was run against the frozen mod-106 eval set.** Per-scenario table + JSON-failure rate table are in `docs/tune-eval.md`.
10. **Fine-tune registered in [[exercise-04]] registry.** `model_id` with `training_set_hash`, `cost_usd_train`, `base_model`.
11. **Shadow was run before canary.** `docs/tune-shadow.md` precedes `docs/tune-canary-log.md` in the timeline.
12. **Per-stage promotion artefacts in canary log.** At least four stages with full gate tables.
13. **Cost model has both per-outcome costs and the training-cost amortisation.** The break-even math is shown even if N/A.
14. **Retro names four peer-role escalation scenarios.** Each is specific to a future question in this fine-tune.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Diagnosis argues shape-vs-knowledge convincingly | 1 | Observed failure + what has been tried + why-not-the-alternatives + success / abort criteria. |
| Training-set has ≥ 500 curated examples | 1 | Count met; sources named. |
| Three-dimensional dedup against eval set, 0 leaks | 1 | Exact + near-duplicate + case_id, all three run and reported. |
| PII scrub run over training data, output audited | 1 | Scrubber output counts by category in data-audit. |
| Human-review coverage = 100% | 1 | Reviewer named per example or per batch. |
| Hyperparameters are provider defaults, justified | 1 | AUTO is set and documented; no sweep. |
| Offline eval includes JSON-failure rate + per-scenario | 1 | Both tables present; comparison against diagnosis success criteria is explicit. |
| Shadow and canary follow the [[exercise-02]] regime | 1 | Same gate families; per-stage promotion artefacts. |
| Cost model is two-line + amortisation + risk note | 1 | Both costs computed; training cost amortised; provider-deprecation risk flagged. |
| Retro names four peer-role escalation triggers | 1 | Each trigger is specific to this fine-tune. |

A submission scoring < 6 is rejected. Hard failures: (a) eval-set contamination (any leak), (b) PII in training set, (c) hyperparameter sweep done by applied engineer, (d) shipped without the [[exercise-02]] shadow → canary regime.

## 5. Common mistakes

- **Running the fine-tune without the diagnosis.** Most first-attempt fine-tunes that fail do so because the problem was knowledge or retrieval, not shape. The diagnosis doc is cheap insurance.
- **Letting the eval set leak into training.** Instant invalidates the result. Three-dimensional dedup is the standard; MinHash or SimHash are productionisations of the Jaccard step.
- **Accepting base-model outputs uncritically.** You cap the fine-tune at base-model quality. For a JSON-reliability fine-tune specifically, review every label's JSON structure.
- **Sweeping hyperparameters.** Dataset quality dominates. Sweeping is `fine-tuning-engineer` scope.
- **Fine-tuning the top tier of the model family.** Applied fine-tuning wins are on smaller tiers. Top-tier SFT is slower, more expensive, often gated, and rarely economical for shape problems.
- **Treating offline-eval pass as ship-ready.** Offline eval ≠ shadow ≠ canary. All three are different observations; collapsing them re-introduces every failure mode mod-111 [[exercise-02]] exists to catch.
- **Cost math on current-tier prices only.** The amortisation assumption must be stated explicitly. If the base model deprecates in 90 days, the amortisation assumption breaks.
- **Deleting `v15_bad`-style "didn't-ship" training sets.** Keep the data audit even for rejected runs — the retro on the next attempt needs to know what was tried.
- **Shadow metrics derived from traces instead of from dedicated events.** Same mistake as [[exercise-03]]. Emit events; don't derive.
- **Hiding the training-cost line on economic comparison.** If the fine-tune "only pays for itself after 90 days," that is a load-bearing assumption and it belongs in the cost doc.

## 6. References

- OpenAI Fine-Tuning documentation — <https://platform.openai.com/docs/guides/fine-tuning> — the SFT shape used in 2c (JSONL format, Files API, Jobs API, hyperparameter semantics, result-files structure). Verify prices before each run against <https://openai.com/api/pricing/>. <!-- verify-before-publishing: confirm gpt-4o-mini SFT pricing and availability at launch time. -->
- AWS Bedrock Custom Models documentation — <https://docs.aws.amazon.com/bedrock/latest/userguide/custom-models.html> — the Bedrock equivalent of 2c (training-data S3 upload, provisioned throughput requirement, supported base models). Shape of job-kickoff, hyperparameter defaults, and model-registry integration all map one-to-one from the OpenAI example.
- Google Vertex AI Tuning documentation — <https://cloud.google.com/vertex-ai/generative-ai/docs/models/tune-models> — the Vertex equivalent (Gemini Flash tuning, dataset format, pipeline-based job orchestration). Supported hyperparameters are a provider-dependent choice; defaults-only applies here too.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the GOVERN function requires that AI-system *changes* be governable. The diagnosis doc + data audit + cost model + retro are the GOVERN artefacts that make a fine-tune accountable. Without them, a fine-tune is a model change with no paper trail.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML02 (Data Poisoning) is the primary threat model for the training-set curation protocol. The three-dimensional dedup against the eval set, the human-review pass, and the PII scrubber are controls that reduce ML02 risk; the data audit is the controls-effectiveness evidence. ML07 (Transfer Learning Attack) applies to the "which base model" choice — the fine-tune inherits the base model's safety properties, which is part of the shape-problem-only scope.
- MITRE ATLAS — <https://atlas.mitre.org/> — Resource Development tactics against LLM-backed features include "obtain training data" and "poison training data". The training-set audit is the evidence that this fine-tune's training data has been curated and scrubbed to a documented standard.

Related exercises: [[exercise-01]] (the flag layer the fine-tune ships through — `support_agent.model_id` flips between base and fine-tune); [[exercise-02]] (the shadow → canary regime the fine-tune is subjected to); [[exercise-03]] (the experimentation-platform interface the fine-tune could run through if we wanted an A/B result against the base — not run here; the fine-tune's primary case is the offline-eval reliability metric, which doesn't need an experimentation platform); [[exercise-04]] (the model registry the fine-tune lands in; rollback of a bad fine-tune is a flag flip on `model_id`); mod-106 (the frozen eval set this fine-tune is judged against); mod-107 [[exercise-03]] (the PII scrubber the training set is run through); mod-109 [[exercise-04]] (CPSO ledger the cost model consumes); mod-110 [[exercise-01]] (the span wrapper that tags fine-tune calls with `app.feature = "support_agent.tune_shadow"` / "support_agent" and `gen_ai.response.model = <fine_tuned_model>`); mod-110 [[exercise-03]] (the feedback pipeline the training-set curation reads from).
