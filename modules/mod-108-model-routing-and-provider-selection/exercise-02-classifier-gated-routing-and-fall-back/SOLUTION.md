### exercise-02: Classifier-Gated Routing and Fallback — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to build **classifier-gated routing** and a **fallback chain** for the helpdesk-reply-copilot surface: a cheap upstream classifier decides which model tier handles a request, and a named fallback chain decides what happens when the chosen tier errors, times out, or returns an unusable response. The two are joined deliberately — routing picks the first choice, fallback is the plan when that choice fails.

The failure mode the exercise is written to catch is the single-model, single-provider surface that lives for months and then goes down on a Friday afternoon when its one provider has a region incident. "We'll just add a fallback when we need it" is the sentence that writes the postmortem. The partner failure is the routing table that is ten lines of if/else in the request path, with no classifier, no measurement, and no answer to "why did this request go to the expensive model?"

This solution provides:

- A **routing model** — the decision variables (request shape, modality, length, estimated difficulty, tenant class, regulatory region) and the taxonomy of routes they feed.
- A **classifier design** — what a classifier-gated router is, how to train/pick it, how to measure it, and when to retire it in favor of a cheaper heuristic.
- A **fallback chain** — the chain shape (per-route, bounded, named), the failure classes it must cover, and the degrade-vs-refuse rules that bound it.
- A **reference implementation sketch** (Python) hooking into the thin gateway from [[exercise-01]], with trace hooks a production system actually needs.
- A **validation and measurement protocol** — what to report, at what cadence, and which numbers block a routing-table change.
- A **grading rubric** for the instructor.

The lesson the exercise defends is: *routing is a continuous, measured decision, not a one-time switch.* A routing table without measurement is a dead table; a classifier without a disablement plan is a classifier that will not be touched when it regresses.

## 2. Worked answer or implementation

### 2a. Routing model (deliverable 1)

Routing on this surface has three inputs and one output.

**Inputs.**

| # | Variable | Source | Why it matters |
|---|---|---|---|
| V1 | `route` | call site (e.g. `reply.compose`, `reply.summarise`, `ticket.triage`) | different routes have different capability and latency needs |
| V2 | `modality` | request shape (text, image attached, audio clip) | not every provider speaks each modality (see [[exercise-04]]) |
| V3 | `estimated_difficulty` | classifier output in `{easy, medium, hard}` | drives cheap vs capable tier selection |
| V4 | `tenant_class` | tenant metadata (e.g. `enterprise`, `smb`, `internal`) | some tenants have model restrictions (data residency, approved-models list) |
| V5 | `region_pin` | tenant metadata | regulatory region constrains the provider set |
| V6 | `recent_failure_signal` | gateway state (provider error rate in last 60s) | hot-swap away from a degraded provider |

**Output.** A `RouteDecision`:

```python
@dataclass(frozen=True)
class RouteDecision:
    primary: Target               # provider + model + region
    fallback: list[Target]        # ordered, finite (≤ 3)
    degrade_mode: str             # "allow_shorter_reply" | "refuse_and_hitl" | "none"
    rationale_tag: str            # "classifier:hard", "pin:eu", "modality:vision", ...
```

The rationale tag is required — every routed request writes its rationale to the trace. A route without a rationale tag is a route that cannot be measured.

### 2b. The routing table (deliverable 2)

The routing table is a committed artefact, not a runtime structure.

```yaml
# routing_table.yaml — helpdesk-reply-copilot — v1.4
routes:
  reply.compose:
    tenant_class: any
    modality: text
    rules:
      - when: "V3 == 'easy'"
        primary: {provider: provider-a, model: model-s-2026-09, region: us-east}
        fallback:
          - {provider: provider-b, model: model-s-2026-10, region: us-east}
          - {provider: provider-a, model: model-m-2026-09, region: us-east}
        degrade_mode: allow_shorter_reply
        rationale_tag: "classifier:easy"
      - when: "V3 == 'medium'"
        primary: {provider: provider-a, model: model-m-2026-09, region: us-east}
        fallback:
          - {provider: provider-b, model: model-m-2026-10, region: us-east}
          - {provider: provider-a, model: model-l-2026-09, region: us-east}
        degrade_mode: allow_shorter_reply
        rationale_tag: "classifier:medium"
      - when: "V3 == 'hard'"
        primary: {provider: provider-a, model: model-l-2026-09, region: us-east}
        fallback:
          - {provider: provider-b, model: model-l-2026-10, region: us-east}
        degrade_mode: refuse_and_hitl
        rationale_tag: "classifier:hard"
  reply.summarise:
    modality: text
    rules:
      - when: "always"
        primary: {provider: provider-a, model: model-s-2026-09, region: us-east}
        fallback:
          - {provider: provider-b, model: model-s-2026-10, region: us-east}
        degrade_mode: allow_shorter_reply
        rationale_tag: "route:summarise-static"
  ticket.triage:
    modality: text
    rules:
      - when: "V5 == 'eu'"
        primary: {provider: provider-b, model: model-m-2026-10, region: eu-central}
        fallback:
          - {provider: provider-c, model: model-m-2026-10, region: eu-central}
        degrade_mode: none
        rationale_tag: "pin:eu"
      - when: "always"
        primary: {provider: provider-a, model: model-m-2026-09, region: us-east}
        fallback:
          - {provider: provider-b, model: model-m-2026-10, region: us-east}
        degrade_mode: allow_shorter_reply
        rationale_tag: "route:triage-default"
```

Rules are evaluated top-to-bottom; the first matching rule wins. Each rule names its rationale tag; the rationale is what `mod-110` groups by when it reports cost and latency.

### 2c. The difficulty classifier (deliverable 3)

The classifier returns `V3` ∈ `{easy, medium, hard}`. Its purpose is not perfect judgement — it is to send the enough-of-the-easy-ones to the cheap tier so the SLOs in `mod-109` are met without hand-tuning.

**Shape choices, in order of increasing cost to maintain:**

1. **Heuristic.** Token-count buckets + presence of certain markers (`code`, `table`, `multi-turn`). No training, no inference cost, no drift. The default starting point.
2. **Small discriminative model.** DistilBERT-class or similar, trained on labeled examples. Adds a classifier-inference hop (≈ 10–40 ms on CPU) and must be versioned.
3. **LLM classifier.** A cheap LLM called with a structured-output schema. Flexible on new categories, but adds latency, cost, and a self-reference failure mode (the classifier is a model that can also go down).

The choice is logged as part of the decision note from [[exercise-01]]. On this surface the recommendation is **(1) heuristic first, (2) small discriminative second**; LLM classifier is only correct when the taxonomy changes more than monthly.

**Measurements the classifier must publish.**

| Metric | Why | Pre-committed target (illustrative placeholder) |
|---|---|---|
| per-class precision + recall | the classifier is a predictor; without precision/recall it is an unmeasured heuristic | illustrative starting floor for a new helpdesk surface: `easy` precision ≥ 0.90 (mis-routing to the cheap tier is the expensive failure), `hard` recall ≥ 0.85 (missing a hard request on the cheap tier is the quality failure); the surface owner re-commits these from offline eval before each routing-table change and never copies them from another surface |
| per-class share of traffic | drift detection | compare against a committed baseline distribution |
| per-class downstream cost (model + tokens) | validates the routing promise | report per-class cost per 1k requests |
| per-class downstream quality (judge win-rate or label) | fairness to tenants on the cheap tier | integrate with `mod-106` evaluation |
| latency of the classifier itself | the classifier must not dominate the budget it saves | a classifier that adds > 20 % of the saved latency is not worth it on this route |

**Retirement plan.** If a heuristic covers the three classes with the same downstream cost and quality as the model, the classifier is retired. Classifiers are not kept for sentimental reasons.

### 2d. The fallback chain (deliverable 4)

**Rules the chain must obey.**

1. **Named per route.** The fallback chain is route-specific, not global. Falling back from a summarisation model to a long-context flagship is a cost mistake.
2. **Bounded.** No more than three hops (primary + two). A chain that cycles through five providers has stopped being a fallback and has become an availability experiment in the request path.
3. **Fail-open only where the surface allows it.** For `reply.compose`, fail-open means "hand off to a human" (HITL); never "return an unmoderated stub." For `ticket.triage`, fail-open means "triage with the heuristic baseline" — i.e. the surface can stand on its own without the agent.
4. **Each hop is a different failure class.** A chain where all three hops share a region is a chain that fails on a region outage.
5. **A hop is tried only when the previous hop's failure class says it should be.** A 400 (bad request / content policy) is not a fallback trigger; it is a surface bug. A 429 or 5xx is.

**Failure classes and their handling.**

| Class | Example | Action |
|---|---|---|
| F1 transient 5xx / timeout | provider returns 503 or times out | next hop |
| F2 429 rate-limited by provider | provider throttles our key | next hop after jittered backoff; emit cost signal |
| F3 provider unavailable (DNS, TLS, circuit-open) | provider SDK cannot reach host | next hop; open circuit for 60s |
| F4 content refusal / safety response | provider refuses to answer | **not** a fallback; route to HITL — the next provider would refuse too, on this input |
| F5 schema-violation / malformed structured output | provider returned shape we cannot parse | retry once; if still bad, next hop |
| F6 tool-call shape mismatch | provider returns function call we cannot bind | surface bug — do not fall back; log and HITL |
| F7 contract regression (unexpected token in output) | provider silently changes tokenizer / format | emit `R14` version-pin alert; fall back |

Rule-of-thumb: **transient and capacity failures fall back; semantic failures do not.** Falling back on a content refusal just moves the bill to a provider that will refuse the same text, with higher latency.

### 2e. Reference implementation sketch

Mounted on the thin gateway from [[exercise-01]]:

```python
"""router.py — classifier-gated routing with bounded fallback.

Mounted as the policy that fills `Gateway.primary` + `Gateway.fallback`
on a per-request basis; the gateway's existing retry loop handles the
per-hop retries.
"""
from __future__ import annotations

from dataclasses import dataclass
from typing import Callable

@dataclass(frozen=True)
class Target:
    provider: str
    model: str
    region: str

@dataclass(frozen=True)
class RouteDecision:
    primary: Target
    fallback: list[Target]
    degrade_mode: str
    rationale_tag: str

class Classifier:
    name: str
    version: str
    def classify(self, call) -> str: ...     # returns "easy" | "medium" | "hard"

def decide(call, table: dict, classifier: Classifier | None, env: dict) -> RouteDecision:
    route = table["routes"][call.route]
    ctx = {
        "V1": call.route,
        "V2": call.modality,
        "V3": classifier.classify(call) if classifier else "medium",
        "V4": env.get("tenant_class"),
        "V5": env.get("region_pin"),
        "V6": env.get("recent_failure_signal"),
    }
    for rule in route["rules"]:
        if _match(rule["when"], ctx):
            return RouteDecision(
                primary=_target(rule["primary"]),
                fallback=[_target(t) for t in rule["fallback"]],
                degrade_mode=rule["degrade_mode"],
                rationale_tag=rule["rationale_tag"],
            )
    raise RuntimeError(f"no rule matched for {call.route}")

# Failure-class → action map; the gateway loop consults this before
# attempting the next hop.
FALLBACK_TRIGGERS = {
    "F1": "next", "F2": "next_backoff", "F3": "next_open_circuit",
    "F4": "hitl", "F5": "retry_then_next", "F6": "hitl", "F7": "next_alert",
}
```

The classifier is wrapped in a feature flag; disabling it falls through to `V3 == 'medium'`, which is the safe default route. This is the "retirement plan" — a classifier regression does not stop the surface.

### 2f. Validation and measurement protocol (deliverable 5)

What the routing + fallback system must publish, where, and on what cadence.

- **Per-rationale-tag report** (daily): request count, cost, latency p50/p95, judge-score mean (from `mod-106`), fallback-hop distribution. Grouped by tenant_class.
- **Classifier drift report** (weekly): per-class share of traffic vs baseline; alert if any class moves > 20 % of its baseline share.
- **Fallback health report** (real-time dashboard, reviewed weekly): % of requests served by primary vs each fallback hop, per route. A route where primary serves < 95 % of traffic over a week is a routing-table review trigger.
- **Routing-table change PR gate**: every change re-runs the offline evaluation (`mod-106` offline suite) and the cost projection. A PR without both attached is rejected.
- **Quarterly re-reads** of OWASP ML Top 10, MITRE ATLAS, and NIST AI RMF against the routing stack (see mod-107 [[exercise-05]]). Provider swaps are re-scored in that review.

### 2g. Implementation notes

- **Static routes are routing too.** `reply.summarise`'s rule is `always → cheap model`. It is still a route decision, with a rationale tag; the dashboard groups it the same way. "No classifier" is a legitimate answer for a route; "no measurement" is not.
- **Content refusals are not transient.** The most common wrong fallback is to retry a refusal on a second provider. Both will refuse. The chain becomes an expensive way to arrive at the same answer. Route to HITL.
- **Per-hop bounded.** Three hops is a budget, not a floor. A long chain hides the real problem, which is that primary is wrong for this route.
- **Region pins are first-class.** A tenant in `V5=eu` must never be routed to a US-only provider as a fallback. The `region_pin` check is a route-level invariant; the test suite asserts it.
- **The classifier can be two layers.** On this surface a token-count bucket decides `easy` vs `medium/hard`; a small model decides within `medium/hard`. The report attributes both decisions.
- **Dry runs matter.** A proposed routing-table change should be replayed over a sample of production traffic (traces only, no re-calls) before it is deployed. `mod-106` offline suite is the evaluation harness; `mod-111` handles the deployment itself.
- **A classifier that always says "medium" is a bug, not a baseline.** If the classifier collapses to a single class in production, the routing-table change-detector should fire within a day; the report in 2f is the mechanism.
- **A fallback that is never exercised is a fallback that is untested.** A failure-injection drill against each route is scheduled quarterly; the test asserts the surface still answers, even if via HITL.

## 3. Validation steps

1. The routing model lists at least six decision variables with their sources. Missing V5 (region pin) or V6 (recent failure signal) is a hard failure — the first is a legal risk, the second is why fallback can help.
2. The routing table is a committed artefact (YAML or equivalent). Every rule names a rationale tag. A rule without a rationale tag is not reviewable.
3. Each route has a named fallback chain of ≤ 3 hops, each hop a different failure class (ideally different region or different provider).
4. Failure classes are enumerated and bound to actions. Content refusal (F4) is **not** a fallback trigger; a solution that falls back on refusals is rejected.
5. The classifier has a disablement plan (feature flag to a safe default class) and a retirement plan (criteria under which the heuristic is restored).
6. Measurement is defined: per-rationale-tag report, classifier drift report, fallback health report. A solution without measurement cannot defend any of its thresholds.
7. The routing-table change workflow is defined: PR → offline eval + cost projection → review.
8. The routing stack is mounted on the thin gateway from [[exercise-01]] (or whichever shape the trade-off note settled on). It does not re-implement the gateway's retry loop.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Routing model with ≥ 6 variables and their sources | 1 | Includes region pin and recent-failure signal. |
| Routing table is committed artefact with rationale tags | 1 | YAML (or equivalent); every rule tagged. |
| Classifier design covers choice, measurement, disablement, retirement | 1 | All four addressed; heuristic-first unless justified. |
| Fallback chain ≤ 3 hops, per-route, named | 1 | Not a global chain. |
| Failure classes enumerated with per-class actions | 1 | Includes content refusal routed to HITL, not next provider. |
| Degrade modes defined per route | 1 | `allow_shorter_reply` / `refuse_and_hitl` / `none`. |
| Measurement: per-rationale-tag + drift + fallback health | 1 | Three reports, with cadences. |
| Routing-table change workflow gated on offline eval + cost | 1 | PR gate named. |
| Failure-injection drill scheduled | 1 | Quarterly or better; test asserts surface answers. |
| Mounted on gateway from [[exercise-01]] | 1 | Does not re-implement retry / trace layer. |

A submission scoring < 6 is rejected. Hard failures: (a) falls back on content refusals, (b) no rationale tags on rules, (c) no measurement for the classifier, (d) no region-pin invariant.

## 5. Common mistakes

- **Falling back on content refusals.** The most common and most expensive mistake. A refusal is a semantic signal, not a capacity signal. Route to HITL; do not pay a second provider to tell the same user "I can't help with that."
- **Unbounded fallback chains.** The temptation is "one more provider, just in case." Every extra hop is a tail-latency time bomb.
- **Classifier calibration never measured.** If no one reports classifier precision/recall per class, the classifier is an opinion. The report in 2f is non-optional.
- **No region pin.** Tenants with `V5=eu` getting routed to `us-east` on fallback is a compliance incident. The invariant is part of the test suite, not a comment in the config.
- **Routing table in code.** Ten lines of if/else works for a week. The YAML (or equivalent) is the reviewable surface for policy, legal, and security roles; a code-only policy denies them a seat at the table.
- **Shared fallback for all routes.** A global `fallback = [modelX, modelY]` is a shape that will overpay for summarisation and under-serve the compose path. Fallbacks are per-route.
- **Fail-open to an unmoderated model.** If the primary is moderated and the fallback is not (because "we need something back"), the fallback has bypassed the moderator. Mount the moderator on the gateway, not on the primary.
- **Classifier cost that eats the saving.** A classifier that adds 150 ms to save 300 ms can be worth it; one that adds 100 ms to save 110 ms is not. Measure before you ship, not after.
- **Never drilling the fallback.** Routes where the primary is 99.99 % available are routes where the fallback is a letter of intent, not a system. Quarterly failure-injection converts the letter into a verified capability.
- **Silent provider-version drift.** `provider-a:model-x` gets a snapshot upgrade on 2026-11-30 and the rationale-tag cost report jumps. The pinning (R14 from [[exercise-01]]) plus the drift report is the thing that surfaces this before the invoice does.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML06 (AI Supply Chain) is the item engaged by every routing and fallback decision: pinning model versions (R14), gating provider swaps on offline eval (2f), and enumerating failure classes are the operational controls the OWASP guidance expects for a multi-provider supply chain.
- MITRE ATLAS — <https://atlas.mitre.org/> — the fallback chain is a defensive response to attacker-caused and provider-caused unavailability events; the F3 / F7 failure classes and the per-rationale-tag reports are what let an incident handler reason about which provider tenant traffic traversed during an incident. Content-refusal handling (F4 routed to HITL) is a defensive answer to Impact tactics that would otherwise cost twice via a naive fallback.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the measurement protocol in 2f implements MEASURE: pre-committed targets, per-population reporting, drift detection, change-gate review. The fallback failure-injection drill implements MANAGE. The routing-table PR gate implements GOVERN (accountability and documented decisions).
- vLLM project documentation — <https://docs.vllm.ai/> — referenced here because a self-hosted vLLM endpoint ([[exercise-03]]) is just another `Target` in the routing table; the same rationale tags, fallback rules, and measurement apply, which is what lets the self-host/managed decision be made without rewriting the routing stack.

Related exercises: [[exercise-01]] (the thin gateway this router mounts on, the responsibility checklist's R1/R2/R3/R14); [[exercise-03]] (self-host as a routing `Target`, measured with the same rationale-tag report); [[exercise-04]] (modality-aware rules for vision / audio / realtime routes); [[exercise-05]] (contract tests pin the request/response shapes this router depends on); mod-106 (offline evaluation suite consulted by the routing-table change gate); mod-109 (cost and latency SLOs whose "why" is these routing decisions); mod-110 (dashboards that group by rationale tag); mod-111 (deployment of a routing-table change); mod-107 [[exercise-05]] (mapping runbook listing router + fallback as controls).
