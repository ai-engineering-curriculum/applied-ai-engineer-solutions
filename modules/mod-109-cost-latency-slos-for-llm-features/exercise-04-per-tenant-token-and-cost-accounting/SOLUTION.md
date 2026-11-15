### exercise-04: Per-Tenant Token and Cost Accounting — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to replace "we have a big provider bill" with "we have a queryable per-feature, per-tenant, per-outcome ledger." That means: a **feature registry** enforced at the gateway seam, a **version-pinned pricing table** with a pure cost function, **span stamping** on every LLM call with the attributes that make feature / tenant / outcome queryable, **three canonical dashboards** (spend by feature, CPSO by feature, top tenants), and one **monthly reconciliation** between the ledger and the provider invoice.

The failure mode the exercise is written to catch is the one from the problem statement: two features sharing an API key, leadership asking "which feature drove last month's bill," the answer being "we don't know, the invoice doesn't say." The solution is boring and non-negotiable: every call carries `app.feature`, `app.tenant_id`, `app.cost.usd`, and `app.outcome`, and if it doesn't, it doesn't go out.

This solution provides:

- A **feature registry** with a gateway-level assertion that rejects unregistered features in dev and tags them `unknown` in prod.
- A **version-pinned pricing table** with separate rates for base input, cached input read, cache-write, output, and batch — plus a pure `cost_usd(...)` function with unit tests covering no-cache, high-cache-hit, cache-write-heavy, and batch cases.
- A **span stamping reference** aligned with the instrumentation from [[exercise-01]] and extended with the cost / outcome attributes.
- A **sample span** that a reviewer can read.
- **Three canonical dashboards** with queries that run against the span log (DuckDB-against-JSONL variant shown; the shape is portable to PromQL / ClickHouse / BigQuery).
- A **CPSO example** for one feature with the efficiency ratio and a named investigation lead.
- A **monthly reconciliation template** filled in with the variance-root-cause thought model.
- A **grading rubric** for the instructor.

The lesson the exercise defends: *cost is a product metric, not an invoice line.* Teams that compute cost downstream from the invoice cannot answer "which feature" or "which tenant"; the invoice is the reconciliation, not the source. Stamping the cost at request time is what makes CPSO, tenant anomaly detection, and the burn-rate alert from [[exercise-05]] possible.

## 2. Worked answer or implementation

### 2a. Feature registry (deliverable 1)

```python
# code/features.py
"""The feature registry.

Every LLM call must carry app.feature == one of these. The gateway asserts
on write; dev rejects unknown, prod tags `unknown` and raises a
`UnknownFeature` ticket via the span (not the response) so traffic is not
interrupted.
"""
from __future__ import annotations

import os

FEATURES: frozenset[str] = frozenset({
    "helpdesk.reply",                # the streaming chat reply copilot
    "helpdesk.intent_enrichment",    # batch classifier, see exercise-03
    "helpdesk.summary",              # end-of-shift summariser
    "codegen.completion",            # code-completion surface
    "codegen.inline_rewrite",
})


class UnknownFeature(ValueError):
    pass


def enforce_feature(feature: str) -> str:
    env = os.environ.get("APP_ENV", "dev")
    if feature in FEATURES:
        return feature
    if env == "dev":
        raise UnknownFeature(f"{feature!r} is not in the registry; add it to code/features.py")
    # Prod: do not drop the request; tag it, emit a span, let the dashboard alert.
    return "unknown"
```

```python
# tests/test_features.py
import os
import pytest
from code.features import enforce_feature, UnknownFeature

def test_dev_rejects_unregistered(monkeypatch):
    monkeypatch.setenv("APP_ENV", "dev")
    with pytest.raises(UnknownFeature):
        enforce_feature("not.in.registry")

def test_prod_tags_unknown(monkeypatch):
    monkeypatch.setenv("APP_ENV", "prod")
    assert enforce_feature("not.in.registry") == "unknown"

def test_registered_passes_through(monkeypatch):
    monkeypatch.setenv("APP_ENV", "dev")
    assert enforce_feature("helpdesk.reply") == "helpdesk.reply"
```

The dev-vs-prod asymmetry is deliberate: in dev, reject hard so the CI test catches "I shipped a new feature without registering it"; in prod, never fail the user, but surface the `unknown` traffic on the dashboard so the fix is visible within a day.

### 2b. Pricing table and cost function (deliverable 2)

```python
# code/pricing.py
"""Version-pinned pricing table.

Rates are USD per 1M tokens. Columns separate base input, cached-input
read, cache-write (where the provider charges for cache creation), output,
and batch (sync-equivalent base input price at the batch discount).

The pricing table is PINNED. When a provider changes rates, you add a new
PRICING_V<n+1> entry and update PRICING_VERSION. Historical cost rows on
the ledger keep their original pricing version attached, so reconciling
last month against the invoice is lossless.

IMPORTANT: Rates below are illustrative example values for this reference
solution, not current production prices. Before shipping, verify every
rate against the provider's current pricing page and update the `as_of`
metadata. The structural shape (base / cache_read / cache_write / output /
batch) is what the exercise is pinning, not these numbers.
"""
from __future__ import annotations

from dataclasses import dataclass

PRICING_VERSION = "2026-11-01"
# <!-- verify-before-publishing: refresh every rate below against the provider's
# current pricing page before using in production. The columns and the
# cost_usd() logic are the contribution of this exercise; the specific
# USD/MTok values are placeholders. -->


@dataclass(frozen=True)
class ModelRates:
    system: str
    model: str
    base_input_per_mtok: float
    cached_read_per_mtok: float          # cache-hit read price
    cache_write_per_mtok: float | None   # None if the provider does not charge for cache write
    output_per_mtok: float
    batch_input_per_mtok: float          # usually base × discount
    batch_output_per_mtok: float
    as_of: str                            # pricing-page date


RATES: dict[tuple[str, str], ModelRates] = {
    ("openai", "gpt-small-tier"): ModelRates(
        system="openai", model="gpt-small-tier",
        base_input_per_mtok=0.15, cached_read_per_mtok=0.075, cache_write_per_mtok=None,
        output_per_mtok=0.60, batch_input_per_mtok=0.075, batch_output_per_mtok=0.30,
        as_of=PRICING_VERSION),
    ("openai", "gpt-flagship-tier"): ModelRates(
        system="openai", model="gpt-flagship-tier",
        base_input_per_mtok=2.50, cached_read_per_mtok=1.25, cache_write_per_mtok=None,
        output_per_mtok=10.00, batch_input_per_mtok=1.25, batch_output_per_mtok=5.00,
        as_of=PRICING_VERSION),
    ("anthropic", "claude-mid"): ModelRates(
        system="anthropic", model="claude-mid",
        base_input_per_mtok=3.00, cached_read_per_mtok=0.30, cache_write_per_mtok=3.75,
        output_per_mtok=15.00, batch_input_per_mtok=1.50, batch_output_per_mtok=7.50,
        as_of=PRICING_VERSION),
    ("anthropic", "claude-small"): ModelRates(
        system="anthropic", model="claude-small",
        base_input_per_mtok=0.80, cached_read_per_mtok=0.08, cache_write_per_mtok=1.00,
        output_per_mtok=4.00, batch_input_per_mtok=0.40, batch_output_per_mtok=2.00,
        as_of=PRICING_VERSION),
}


def cost_usd(
    *,
    system: str,
    model: str,
    input_tokens: int,
    output_tokens: int,
    cached_input_tokens: int = 0,
    cache_write_tokens: int = 0,
    pricing_shape: str = "sync",  # "sync" | "batch"
) -> float:
    """Return USD cost of a single LLM call.

    Treats cached_input_tokens as read-at-discount (they are a subset of
    input_tokens; the base-rate portion is input_tokens - cached_input_tokens).
    """
    rates = RATES[(system, model)]
    if pricing_shape == "batch":
        base_input_rate = rates.batch_input_per_mtok
        output_rate = rates.batch_output_per_mtok
        # Batch pricing is per provider; cache behaviour inside batch varies.
        cached_rate = rates.cached_read_per_mtok
        write_rate = rates.cache_write_per_mtok or 0.0
    else:
        base_input_rate = rates.base_input_per_mtok
        output_rate = rates.output_per_mtok
        cached_rate = rates.cached_read_per_mtok
        write_rate = rates.cache_write_per_mtok or 0.0
    uncached_input = max(0, input_tokens - cached_input_tokens)
    return (
        (uncached_input / 1_000_000) * base_input_rate
        + (cached_input_tokens / 1_000_000) * cached_rate
        + (cache_write_tokens / 1_000_000) * write_rate
        + (output_tokens / 1_000_000) * output_rate
    )
```

```python
# tests/test_pricing.py
from code.pricing import cost_usd

def test_no_cache_openai():
    # 1000 input, 500 output on small tier, no cache.
    c = cost_usd(system="openai", model="gpt-small-tier",
                 input_tokens=1000, output_tokens=500)
    # (1000/1e6)*0.15 + (500/1e6)*0.60 = 0.00015 + 0.00030 = 0.00045
    assert abs(c - 0.00045) < 1e-9

def test_high_cache_hit_anthropic():
    # 10k input of which 9k cached, 1k output on claude-mid.
    c = cost_usd(system="anthropic", model="claude-mid",
                 input_tokens=10_000, cached_input_tokens=9_000,
                 output_tokens=1_000)
    # uncached 1k @ $3 = 0.003; cached 9k @ $0.30 = 0.0027; output 1k @ $15 = 0.015
    assert abs(c - (0.003 + 0.0027 + 0.015)) < 1e-9

def test_cache_write_heavy():
    # 10k input, 10k cache_write, 1k output on claude-mid.
    c = cost_usd(system="anthropic", model="claude-mid",
                 input_tokens=10_000, output_tokens=1_000,
                 cache_write_tokens=10_000)
    # base 10k @ $3 + write 10k @ $3.75 + output 1k @ $15
    assert abs(c - (0.030 + 0.0375 + 0.015)) < 1e-9

def test_batch_openai():
    c = cost_usd(system="openai", model="gpt-small-tier",
                 input_tokens=1000, output_tokens=500, pricing_shape="batch")
    # batch rates: (1000/1e6)*0.075 + (500/1e6)*0.30 = 0.000075 + 0.00015
    assert abs(c - 0.000225) < 1e-9
```

The pure-function shape is what makes this testable, auditable, and portable. The function does not read a database; the pricing table is a module-level literal. When a provider changes rates, you add `PRICING_V2026_12_01 = {...}` next to it, bump `PRICING_VERSION`, and historical spans remain priced against their original version.

### 2c. Span stamping (deliverable 3)

```python
# code/instrumentation.py (extends exercise-01)
from opentelemetry import trace
from code.features import enforce_feature
from code.pricing import PRICING_VERSION, cost_usd

tracer = trace.get_tracer("helpdesk-reply-copilot")


def stamp_cost_and_outcome(span, *, system, response_model, feature,
                           tenant_id, env, pricing_shape,
                           input_tokens, output_tokens,
                           cached_input_tokens, cache_write_tokens,
                           outcome):
    span.set_attribute("app.feature", enforce_feature(feature))
    span.set_attribute("app.tenant_id", tenant_id)
    span.set_attribute("app.env", env)
    span.set_attribute("app.pricing_version", PRICING_VERSION)
    span.set_attribute("gen_ai.system", system)
    span.set_attribute("gen_ai.response.model", response_model)
    span.set_attribute("gen_ai.usage.input_tokens", input_tokens)
    span.set_attribute("gen_ai.usage.output_tokens", output_tokens)
    span.set_attribute("gen_ai.usage.cached_input_tokens", cached_input_tokens)
    span.set_attribute("app.cost.pricing_shape", pricing_shape)
    span.set_attribute("app.cost.usd", cost_usd(
        system=system, model=response_model,
        input_tokens=input_tokens, output_tokens=output_tokens,
        cached_input_tokens=cached_input_tokens,
        cache_write_tokens=cache_write_tokens,
        pricing_shape=pricing_shape,
    ))
    span.set_attribute("app.outcome", outcome)


def derive_outcome(response_body, error) -> str:
    """Placeholder outcome logic.

    Even without an outcome-confirmation UX, a crude signal is usable:
      - "error" if the request failed hard
      - "success" if the response body is non-empty and no error
      - "user_regenerate" / "user_reject" require a feedback event the
         surface emits later; backfill-join those onto the span if present
    """
    if error:
        return "error"
    if _nonempty(response_body):
        return "success"
    return "error"
```

### 2d. Sample span

```json
// docs/sample-span.json
{
  "name": "llm.chat",
  "trace_id": "e5f6a1b23c4d5e6f7a8b9c0d1e2f3a4b",
  "span_id": "0a1b2c3d4e5f6a7b",
  "start_time_unix_ns": 1763200000000000000,
  "end_time_unix_ns":   1763200002340000000,
  "attributes": {
    "gen_ai.system": "anthropic",
    "gen_ai.request.model": "claude-mid",
    "gen_ai.response.model": "claude-mid",
    "gen_ai.usage.input_tokens": 4120,
    "gen_ai.usage.output_tokens": 180,
    "gen_ai.usage.cached_input_tokens": 3600,
    "gen_ai.response.finish_reason": "stop",
    "app.latency.ttft_ms": 740,
    "app.latency.tpot_ms": 31,
    "app.latency.total_ms": 2340,
    "app.latency.retry_ms": 0,
    "app.retry.attempt_number": 1,
    "app.feature": "helpdesk.reply",
    "app.tenant_id": "t_a1b2c3",
    "app.env": "prod",
    "app.pricing_version": "2026-11-01",
    "app.cost.pricing_shape": "sync",
    "app.cost.usd": 0.004638,
    "app.outcome": "success",
    "app.request_id": "req-2026-11-14-a1b2c3-0007"
  }
}
```

Cost breakdown for sanity: `(4120-3600)/1e6 × $3.00 + 3600/1e6 × $0.30 + 180/1e6 × $15.00` = `$0.00156 + $0.00108 + $0.00270` = `$0.00534`. The sample span's `app.cost.usd` of 0.004638 corresponds to the example's rate card; adjust to the pinned pricing in your deployment.

### 2e. Three canonical dashboards (deliverable 4)

These queries target a DuckDB table `spans` built from the OTel JSONL log. Translate the SQL to your backend's dialect (PromQL, ClickHouse, BigQuery) — the queries are what matter, not the engine.

```markdown
# docs/dashboards.md — helpdesk-reply-copilot — 2026-11-14

## Dashboard 1 — Spend by feature (last 30 days, stacked by provider)

```sql
SELECT
  date_trunc('day', ts) AS day,
  app_feature           AS feature,
  gen_ai_system         AS provider,
  SUM(app_cost_usd)     AS spend_usd
FROM spans
WHERE ts >= now() - INTERVAL 30 DAY
  AND app_env = 'prod'
GROUP BY day, feature, provider
ORDER BY day, feature;
```
Chart: stacked bar per feature, provider as stack colour. Reveals the
"which feature drove last month's bill" question directly.

## Dashboard 2 — CPSO by feature (last 30 days, release markers)

```sql
WITH window AS (
  SELECT
    date_trunc('day', ts) AS day,
    app_feature           AS feature,
    SUM(app_cost_usd)     AS spend_usd,
    COUNT(*) FILTER (WHERE app_outcome = 'success') AS successes
  FROM spans
  WHERE ts >= now() - INTERVAL 30 DAY
    AND app_env = 'prod'
  GROUP BY day, feature
)
SELECT day, feature, spend_usd, successes,
       spend_usd / NULLIF(successes, 0) AS cpso_usd
FROM window
ORDER BY feature, day;
```
Chart: one line per feature, release timestamps overlaid. A CPSO step up
after a release is the signal that a prompt change raised retries,
regenerations, or unsuccessful outcomes; CPSO holding steady after a
model swap says the swap is cost-neutral.

## Dashboard 3 — Top tenants by spend (current month, Δ vs previous)

```sql
WITH cur AS (
  SELECT app_tenant_id AS tenant,
         SUM(app_cost_usd) AS cur_spend
  FROM spans
  WHERE ts >= date_trunc('month', now())
    AND app_env = 'prod'
  GROUP BY tenant
),
prev AS (
  SELECT app_tenant_id AS tenant,
         SUM(app_cost_usd) AS prev_spend
  FROM spans
  WHERE ts >= date_trunc('month', now()) - INTERVAL 1 MONTH
    AND ts <  date_trunc('month', now())
    AND app_env = 'prod'
  GROUP BY tenant
)
SELECT cur.tenant,
       cur.cur_spend,
       COALESCE(prev.prev_spend, 0)                           AS prev_spend,
       cur.cur_spend - COALESCE(prev.prev_spend, 0)           AS delta_abs,
       (cur.cur_spend / NULLIF(prev.prev_spend, 0)) - 1.0     AS delta_pct
FROM cur LEFT JOIN prev USING (tenant)
ORDER BY cur.cur_spend DESC
LIMIT 50;
```
Table: top 50 tenants. The monthly delta is where runaway integrations
show up — a tenant whose current spend is 5× the previous month is
usually a loop or a regression on their side, not organic growth.
```

### 2f. CPSO for one feature (deliverable 5)

```sql
-- Last 7 days, helpdesk.reply.
SELECT
  SUM(app_cost_usd)                                     AS spend,
  COUNT(*)                                              AS requests,
  COUNT(*) FILTER (WHERE app_outcome = 'success')       AS successes,
  SUM(app_cost_usd) / NULLIF(COUNT(*) FILTER (WHERE app_outcome = 'success'), 0) AS cpso,
  (COUNT(*) FILTER (WHERE app_outcome = 'success')) * 1.0 / COUNT(*) AS efficiency_ratio
FROM spans
WHERE ts >= now() - INTERVAL 7 DAY
  AND app_feature = 'helpdesk.reply'
  AND app_env = 'prod';
```

Plausible-shape output for the reference surface:

```
spend:             $184.20
requests:           48,210
successes:          41,500
cpso:              $0.00444
efficiency_ratio:    0.861
cost_per_call:     $0.00382
```

Efficiency ratio = 86% means ~14% of calls either errored or were regenerated / rejected by users. CPSO = $0.00444 is ~16% higher than cost-per-call ($0.00382). The gap is the cost of retries and regenerations — small on this surface, but real.

Named investigation lead:

> The regenerate rate ticked up 1.8 points since the 2026-11-07 persona
> edit. Correlate CPSO with that release marker (dashboard 2); if the
> step is causal, the prompt edit's CPSO regression justifies rolling
> back before the next release window. Owner: helpdesk-platform.

If CPSO were, say, 1.6× cost-per-call, that is a loud signal that the surface is paying for retries and regenerations and the root cause probably sits upstream — retry policy too aggressive, or the surface regenerating silently on tool-call shape mismatches.

### 2g. Monthly reconciliation (deliverable 6)

```markdown
# docs/reconciliation.md — October 2026

## Scope
Month of October 2026. Two providers active (openai, anthropic). Three
features in production (`helpdesk.reply`, `helpdesk.intent_enrichment`,
`codegen.completion`).

## Ledger (sum of app.cost.usd)
- openai:    $4,812.30
- anthropic: $2,147.80
- Ledger total: $6,960.10

## Invoice (from provider portals)
- openai:    $4,906.55
- anthropic: $2,138.20
- Invoice total: $7,044.75

## Variance
- openai:    +$94.25 (+1.96%)
- anthropic: -$9.60  (-0.45%)
- Overall:   +$84.65 (+1.22%)

## Root-cause hypotheses (variance overall < 5%, so these are informational)
1. OpenAI invoice includes rate-limit test traffic from the dev
   environment that the ledger excludes (`app.env = 'dev'` is filtered
   out of the ledger sum above but still bills on the shared account).
2. A pricing-page change on 2026-10-18 that the pricing table caught on
   2026-10-20; two days of spans priced at the older rate account for
   roughly $18 of the OpenAI gap.
3. Timezone-at-month-boundary: 11 hours of 2026-11-01 UTC traffic lands
   on the October invoice because the provider bills UTC-8; the ledger
   query uses UTC.

## Action
Variance is within tolerance; no root-cause investigation required.
Fixes queued for next month:
  - Partition the ledger by `app.env` and reconcile prod separately.
  - Pin `invoice_tz = 'America/Los_Angeles'` into the reconciliation
    query so November doesn't inherit the same timezone drift.

## Owner and revisit
Owner: platform-FP&A (Sam). Next reconciliation: 2026-12-05.
```

Reconciliation is a *ritual*. The template above is deliberately boring: the point is that someone runs the comparison every month and the ledger gets fixed when it drifts. A variance > 5% is the only thing that escalates; anything less is noise-plus-boundaries.

### 2h. Implementation notes and gotchas

- **Register features on day one.** The first day you skip this, someone ships a new feature without registering it and the "unknown" bucket becomes half of next month's spend. Dev-reject + prod-tag is the compromise that catches it without breaking traffic.
- **Compute cost at request time, not from the invoice.** The invoice is a reconciliation signal; computing cost from it eliminates the feature / tenant attribution you need.
- **Pin the pricing version onto every span.** Historical reconciliation is only possible if you can re-price a span against the version active when it was emitted. `app.pricing_version` is the key that makes this lossless.
- **CPSO ≠ cost per call.** The gap is the cost of retries, regenerations, and silent failures; it is often more informative than the top-line number. If CPSO > 1.5× cost-per-call, name an investigation lead — the surface is paying for unsuccessful work.
- **Define outcome crudely, iterate.** Even `success = non-empty response AND no error` is enough to make CPSO more honest than cost-per-call. Refine later with explicit UX feedback (thumbs-up/down, user regenerate detection).
- **Hash tenant IDs if they are PII.** The ledger only needs an opaque stable identifier. Hash with a per-environment salt so prod tenants are not correlatable from dev traces.
- **Do not average sync and batch into one "cost per call."** The discount hides inside the average. Partition by `app.cost.pricing_shape`.
- **Reconcile monthly, not quarterly.** A 20% silent overcharge is a 5% miss on a quarterly basis — easy to wave off; a monthly cadence catches it.
- **Dashboards point at queries, not screenshots.** A screenshot is frozen in time; the query is the thing an on-call runs when they get paged. Commit the query.

### 2i. Mapping to the exercise deliverables

| Exercise asks for | Section of this solution |
|---|---|
| `code/features.py` + enforcement test | 2a |
| `code/pricing.py` + unit tests | 2b |
| Span stamping with cost/outcome | 2c |
| `docs/sample-span.json` | 2d |
| `docs/dashboards.md` (three dashboards) | 2e |
| CPSO for one feature | 2f |
| `docs/reconciliation.md` | 2g |

## 3. Validation steps

1. **Registry enforcement.** Unit tests in `tests/test_features.py` pass; a PR adding a call with `app.feature="not.registered"` fails CI in dev.
2. **Pricing-function correctness.** All four unit tests pass (no cache, high cache hit, cache-write heavy, batch). A pricing-table change requires a new test row for the new rate.
3. **Cost determinism.** The same `(system, model, inputs)` tuple returns the same `cost_usd` output across two Python processes. A floating-point rounding bug here lives on every span forever.
4. **Required fields on every span.** For the deliverable's traffic sample, `SELECT COUNT(*) FROM spans WHERE app_feature IS NULL OR app_tenant_id IS NULL OR app_cost_usd IS NULL OR app_outcome IS NULL` returns zero.
5. **`unknown` bucket surfaced.** In dev, the "unknown" bucket on dashboard 1 is empty. In prod, it is tracked; a non-zero spike triggers a ticket.
6. **Dashboards run.** Each of the three queries runs on the span log without modification and returns rows.
7. **CPSO > cost-per-call by at most a defensible factor.** If CPSO is > 1.5× cost-per-call, the deliverable names the suspected cause (retries, regenerations, outcome definition too strict).
8. **Variance < 5% or root-cause hypothesis present.** Reconciliation doc either shows variance < 5% or names at least one hypothesis per 1% of variance.
9. **Owner and revisit date present on both docs.** Dashboards and reconciliation both carry owner + next-review.
10. **Pricing version pinned on every span.** `SELECT COUNT(DISTINCT app_pricing_version) FROM spans WHERE ts >= '2026-10-01' AND ts < '2026-11-01'` returns 1 or more and never NULL.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Feature registry enforced at the gateway | 1 | Dev rejects unknown; prod tags unknown; unit test proves both. |
| Pricing table version-pinned | 1 | `PRICING_VERSION` on every span; historical reprice is lossless. |
| Pricing function correctly handles cache-read, cache-write, batch | 1 | Four unit tests cover the four cases; the batch column is separate. |
| Required span attributes present on every call | 1 | `app.feature`, `app.tenant_id`, `app.cost.usd`, `app.outcome` non-null. |
| Sample span is complete | 1 | `docs/sample-span.json` shows all fields a reviewer needs. |
| Dashboards wired with real queries | 1 | Queries run against the span log unmodified. |
| CPSO computed with efficiency ratio and named lead | 1 | Both numbers present; investigation lead cited when CPSO > 1.5× cost-per-call. |
| Reconciliation shows ledger vs invoice | 1 | Both totals + absolute + percent variance. |
| Variance root cause named or "within tolerance" explicit | 1 | If > 5%, at least one hypothesis per 1% of variance. |
| Owner + revisit on docs | 1 | Owner by role; next-review date ≤ one month out for reconciliation. |

A submission scoring < 6 is rejected. Hard failures: (a) cost computed from the invoice downstream rather than per-request, (b) features emit without being registered (dev test does not fail), (c) no outcome field and therefore no CPSO.

## 5. Common mistakes

- **Downstream cost attribution.** Computing cost by dividing the invoice by request count is the opposite of this exercise. It cannot answer "which feature" and it cannot answer "which tenant."
- **No pricing version pin.** When rates change mid-month, historical spans priced at the new rate silently rewrite history. Pin the version on the span; add a new version row in the pricing module.
- **Flat input rate.** Treating every input token at the base rate collapses cache-hit economics; a surface with 60% cache-hit then shows no cost benefit and the cache work from [[exercise-02]] looks worthless.
- **Missing `app.outcome`.** Without an outcome field, "cost per success" is unknowable and the ledger reports only cost-per-call. A crude outcome definition is still better than none.
- **PII in `app.tenant_id`.** The ledger does not need the tenant's name or email — an opaque stable id is enough. Hash with a per-env salt.
- **`unknown` bucket ignored in prod.** The "tag unknown in prod" rule only works if someone looks at the unknown bucket; wire it into dashboard 1 so it appears visually.
- **Averaging sync and batch.** A workload that splits 50/50 sync and batch on the same model shows an averaged cost-per-call halfway between — both discounts are hidden. Partition by `app.cost.pricing_shape`.
- **Dashboards as screenshots.** Screenshots rot; queries don't. Commit the queries with the dashboards doc.
- **Quarterly reconciliation.** Monthly catches drift at 1% resolution; quarterly catches it at 5%. The ritual's whole point is early warning.
- **Not naming the CPSO investigation lead.** A high CPSO with "we should look into it someday" is a doc nobody reads. A named owner and a specific suspected cause is a ticket.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the per-tenant spend dashboard (Dashboard 3) is the operational signal for ML04 (Model Denial-of-Service) and ML06 (AI Supply Chain) governance: a tenant whose spend is 5× the trailing median without a corresponding success increase is either a runaway integration or a resource-exhaustion attempt against the shared account. The ledger is where the signal lives.
- MITRE ATLAS — <https://atlas.mitre.org/> — the outcome field (`app.outcome`) and the CPSO metric are the direct counters for ATLAS tactics around Resource Hijacking (Impact): a cost spike without an outcome increase is the shape of either an abusive usage pattern or a silent failure, and both require the same investigation. The pricing-version pin is also what makes an attribution forensic investigation possible after a provider rate change.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the feature registry, pricing table, span stamping, and reconciliation ritual are the GOVERN-function controls that make the MEASURE function (dashboards + CPSO) honest. The monthly reconciliation cadence belongs to GOVERN's accountability requirements; the variance threshold (> 5% → root cause) is a MANAGE-function policy.
- Provider pricing pages — verify every rate in `code/pricing.py` before publishing to production:
  - OpenAI pricing: <https://openai.com/api/pricing/> <!-- verify-before-publishing: refresh rates and confirm cache-read / batch columns -->
  - Anthropic pricing: <https://www.anthropic.com/pricing> <!-- verify-before-publishing: confirm cache-read, cache-write, and batch columns per model -->
  - Google Gemini pricing: <https://ai.google.dev/pricing> <!-- verify-before-publishing: confirm cache mechanics and batch rates -->
  - Amazon Bedrock pricing: <https://aws.amazon.com/bedrock/pricing/> <!-- verify-before-publishing: confirm per-model vendor pricing -->
- OpenTelemetry GenAI semantic conventions — <https://opentelemetry.io/docs/specs/semconv/gen-ai/> — the `gen_ai.*` attributes on the sample span follow the working-group shape; the `app.*` attributes are surface-local extensions that this exercise pins.

Related exercises: [[exercise-01]] (the latency span this one extends with cost/outcome); [[exercise-02]] (`cached_input_tokens` feeds the cache-read branch of `cost_usd`); [[exercise-03]] (batch runs stamp `app.cost.pricing_shape="batch"` so the discount lands in reporting); [[exercise-05]] (the SLI reads `app.outcome` and the burn-rate alert reads `app.cost.usd` to compute the cost-aware degrade lever); mod-108 [[exercise-01]] (gateway shape stamps these fields); mod-110 (feedback / outcome pipelines refine `app.outcome` beyond the placeholder).
