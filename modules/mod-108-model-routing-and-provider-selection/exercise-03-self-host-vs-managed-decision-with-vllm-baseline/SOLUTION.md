### exercise-03: Self-Host vs Managed Decision with a vLLM Baseline — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to decide whether a given route on the helpdesk-reply-copilot surface should run on a **managed provider** (API from a third party) or on a **self-hosted model** (vLLM-served open-weights model), and to produce a **vLLM baseline** that makes the decision defensible instead of a vibe. The decision is per-route, not per-surface: `reply.summarise` is not automatically the same answer as `reply.compose` or `ticket.triage`.

The failure mode the exercise is written to catch is one of two stories, each costly. Story one: the team migrates a hot route to self-host because "it will be cheaper," discovers the operational cost six months later, and quietly rolls back on a Friday. Story two: the team never considers self-host, pays the managed premium on routes whose capability needs are trivial, and the `mod-109` cost SLOs are met by capping volume. The baseline is the artefact that lets a reviewer see which story this surface is in before it ships the change.

This solution provides:

- A **decision model** — the set of variables that select for one shape vs the other, in the order they bind.
- A **vLLM baseline recipe** — what to serve, what to measure, how to measure, and what to publish so a managed route and a self-hosted route can be compared on the same terms.
- A **total-cost-of-ownership (TCO) frame** — the cost columns both options must be scored against (not only inference $/1M tokens).
- A **decision note template** — the artefact that records the decision, the owner, the trigger, and the sign-off.
- A **grading rubric** for the instructor.

The lesson the exercise defends is: *self-host is an operational commitment, not a cost trick.* A vLLM endpoint is just another `Target` in [[exercise-02]]'s routing table; the question is whether the team's on-call, GPU capacity planning, and model-ops maturity make that `Target` cheaper over a two-year window than a managed line item.

## 2. Worked answer or implementation

### 2a. Decision model (deliverable 1)

The decision binds in the following order. An earlier `exclude` is a decision; the later variables do not get consulted.

| # | Variable | What makes it `exclude` self-host | What makes it `exclude` managed |
|---|---|---|---|
| D1 | Data residency / tenant class | provider offers residency; team cannot stand up a GPU cluster in that region | tenant class forbids tenant data leaving our VPC (regulated verticals, EU residency on a non-EU provider) |
| D2 | Capability requirement | best open-weights model for this route trails the managed flagship on the surface's judge | route is trivial (summarise, classify) and open-weights are at parity |
| D3 | Expected volume | volume is low (< a small daily p95); utilisation of a GPU would sit near idle | volume is high enough that managed $/1M tokens dominates TCO after infra |
| D4 | On-call / ops capacity | team has no GPU on-call rotation or model-ops practice | team has mature on-call for GPU services and model rollouts |
| D5 | Latency profile | required p95 cannot be met behind our region's GPU capacity with headroom | managed p95 includes a round-trip the surface cannot absorb |
| D6 | Model-swap cadence | we want new flagship models within days of release; open-weights ecosystem will trail | we pin models for stability; provider snapshot upgrades are a liability |
| D7 | Content-policy fit | managed provider refuses legitimate content for this surface | open-weights path has no managed moderation — must bring our own, see mod-107 |
| D8 | Regulated workload HITL rate | HITL rate is high enough that model cost is noise | HITL rate is low; model cost is a real knob |

Decisions are per route, not per surface. A surface may have a managed `reply.compose`, a self-hosted `ticket.triage`, and a managed `reply.summarise` simultaneously — the gateway from [[exercise-01]] does not care.

### 2b. vLLM baseline recipe (deliverable 2)

The baseline exists so that `managed-A` and `self-host-vLLM-B` can be compared **on the same workload with the same evaluation**. The recipe:

1. **Pick the candidate model.** An open-weights model whose published capability covers the route's known shape (context length, tool use if applicable, multilinguality if applicable). The candidate is pinned by hash, not by tag.
2. **Pick the serving binary.** vLLM is the baseline because its paged-attention and continuous-batching behaviour is the open-source reference for throughput-oriented serving; the baseline numbers it produces are the floor another runtime has to beat to replace it. See the vLLM project for the official feature documentation and the model families it supports.
3. **Pick the hardware configuration.** One SKU per candidate. "A node with N × G GPU, M GiB RAM, local NVMe for weights" — the one the team will actually buy or rent. The baseline on borrowed hardware is a draft.
4. **Freeze the workload.** A trace-based replay of ~24 hours of surface traffic for the route, with PII scrubbed per mod-107 [[exercise-03]]. The replay is version-controlled; a workload change invalidates the baseline.
5. **Freeze the evaluation.** `mod-106` offline eval suite points at the vLLM endpoint; the judge and the judge model are the same as the ones used for the managed baseline. "Same judge, same inputs, different model" is the only way the comparison is fair.
6. **Measure the following and publish them in a baseline sheet:**

| Metric | How | Why this matters for the comparison |
|---|---|---|
| Throughput (requests/sec per replica) at the pinned context length distribution | load test with the replay at increasing concurrency until p99 budget is violated | TCO depends on this |
| p50 / p95 / p99 latency | load test at the sustained throughput | SLOs come from mod-109 |
| Time-to-first-token p95 (for streaming routes) | load test | user-perceived latency |
| GPU utilisation at sustained throughput | runtime metric | validates the SKU choice |
| Cold start time (replica add / replace) | drill — kill a pod, time return-to-ready | affects autoscaling and incident recovery |
| Weight-load time (first boot) | drill — launch into an empty cache | affects deploy cadence |
| Model-swap time (hot replace of weights) | drill — swap to a new pinned model | affects the model-swap cadence row in D6 |
| Judge quality (`mod-106` suite win-rate vs managed baseline) | run the suite | dominates D2 |
| Tail-latency behaviour under rate shock | inject 2× sudden load | affects fallback decisions (see [[exercise-02]] F1/F2) |
| Failure modes | drill: OOM, GPU reset, weight corruption | shapes the runbook |
| Observable per-request cost | derived from utilisation + SKU unit cost | fed into TCO table |

Each number is signed by the engineer who ran it, dated, and tied to the pinned model hash and serving binary version. Any number without a hash is not a baseline.

### 2c. TCO frame (deliverable 3)

**Managed option cost columns (per year):**

- Inference $ (negotiated unit rate × projected tokens, with scaling for growth and for seasonal spikes).
- Volume commits or discounts (only if committed).
- Overage posture (what happens on 2× volume: hard throttle, overage pricing, provider ceiling).
- Governance overhead (vendor review, DPA re-read, residency controls).
- Incident posture (we can't ship a fix; we open a ticket).

**Self-host option cost columns (per year):**

- GPU capacity (SKU count × unit cost), including the headroom needed to serve under the measured utilisation at peak.
- Platform overhead (orchestration, observability, networking — if those don't already exist for this team, that is the cost).
- Model-ops effort (one engineer-quarter per model swap is a reasonable starting assumption; measure yours).
- On-call cost (new rotation if there isn't one; expanded rotation if there is).
- Model-licensing cost (open-weights models have licenses; "open" doesn't mean "any use at no cost").
- Capacity-planning risk (we buy for peak; off-peak is wasted; a provider doesn't charge for our idle).
- Weight-storage and distribution.
- Compliance evidence (self-hosting makes some evidence easier and some harder; do not assume one direction).

**Comparison rule.** The TCO is over a two-year window. The "we'll just move it back" sentence is cheap; the move back is six engineer-weeks in the typical case. The comparison window must be long enough that the move-back cost is included if the decision is wrong.

### 2d. Decision note (deliverable 4)

```markdown
# Self-host-vs-managed decision — helpdesk-reply-copilot — reply.compose — v1 — 2026-11-13

## Decision
`reply.compose` runs on **managed provider-A, model `model-m-2026-09`**
as primary; **managed provider-B, model `model-m-2026-10`** as the first
fallback. Self-host on vLLM is **rejected for this route for now** and
re-reviewed under the triggers below.

Separately, `ticket.triage` runs on **self-hosted vLLM, pinned model
`org-open-m-2026-07@<sha>`** on `g-sku-m × 2` (one replica per AZ);
managed `provider-a/model-m-2026-09` is the first fallback. See the
companion note for that route.

## Decision model trace (section 2a)
- D1 residency: `reply.compose` tenants are 97 % in `V5=us-east`; both
  managed providers offer the region. No exclusion.
- D2 capability: our offline suite scores the candidate open-weights
  model behind managed-A on this route (the delta is mostly on
  policy-nuance replies; see [[exercise-02]] `classifier:hard` bucket).
  Managed wins on capability.
- D3 volume: `reply.compose` sees sustained traffic that would use ~70 %
  of a `g-sku-l` node on vLLM, but peak would require two nodes. Volume
  is sufficient to consider self-host.
- D4 ops: team has no GPU on-call today. New-rotation cost is explicit
  in the TCO; the decision does not pretend ops is free.
- D5 latency: vLLM baseline hit the p95 budget with 20 % headroom; not
  excluding.
- D6 swap cadence: `reply.compose` has benefited from a managed flagship
  improvement twice this year; self-host cadence would have lagged.
  This is the single largest factor against self-host for this route.
- D7 content-policy fit: managed is a better fit here — we have been
  able to stay within provider policy, and bring-your-own moderator
  (mod-107) runs upstream anyway.

## TCO table (illustrative values; fill with your numbers before signing)
| Column | Managed path | Self-host path |
|---|---|---|
| Inference / GPU capex | $X_managed/year | $Y_gpu/year |
| Platform overhead | included | $Z_platform/year |
| Model-ops effort | low | 1 eng-qtr per swap, planned 2x/year |
| On-call cost | existing | new rotation, $W_oncall/year |
| Model-swap latency (D6) | days | weeks-to-months |
| Governance overhead | DPA already reviewed | self-host inherits mod-107 compliance in-house |
| Move-back cost if wrong | low | ~6 eng-weeks |

Values with `$X_managed` style are placeholders — the note is not signed
until the numbers are substituted and reviewed.

## Residual risks (named)
- `RR-04` Managed provider snapshot change could regress quality
  silently. Mitigation: `R14` version pin ([[exercise-01]]) + drift
  report ([[exercise-02]]).
- `RR-05` Dependency on one provider for `reply.compose`. Mitigation:
  fallback provider measured on the same `mod-106` suite.
- `RR-06` Self-host decision for `ticket.triage` is coupled to one GPU
  SKU; a supply constraint for that SKU is a re-decision trigger.

## Triggers for re-decision
- Managed $/1M token price change > 20 %.
- Open-weights candidate reaches parity with managed on the `mod-106`
  suite for this route (judge-score delta < 2 %).
- A new residency requirement for a tenant class that managed cannot
  meet.
- vLLM runtime adds a capability (e.g. on-the-fly LoRA) that changes
  the swap-cadence calculation (D6).
- Any incident whose postmortem recommends reconsideration.
- Quarterly re-read.

## Sign-off
- Route owner ...................... (initials, date)
- Platform / GPU ops lead .......... (initials, date)
- Finance partner (TCO numbers) .... (initials, date)
- Security / compliance (mod-107) .. (initials, date)
```

The note is per-route. A surface with three routes may have three notes. Every note points at the vLLM baseline sheet it was decided against.

### 2e. Serving sketch

The thin gateway from [[exercise-01]] does not care whether the `Target` is managed or self-hosted:

```python
"""vllm_provider.py — vLLM endpoint as a Provider in the thin gateway.

The gateway treats this as just another provider; the fallback rules
from [[exercise-02]] operate unchanged. The point of this adapter is to
normalise the request / response shape (Provider protocol) and emit the
same tracing fields a managed provider does.
"""
from __future__ import annotations

import httpx
from dataclasses import dataclass

@dataclass
class VLLMProvider:
    name: str              # e.g. "self-host-vllm-m"
    base_url: str          # internal URL to the vLLM endpoint
    model_hash: str        # pinned weight hash; emitted on every trace
    client: httpx.Client

    def chat(self, call) -> dict:
        req = _to_openai_compatible(call, self.model_hash)
        # vLLM exposes an OpenAI-compatible chat/completions API; use it
        # as the contract so [[exercise-05]] contract tests target the
        # same shape across providers. See vLLM OpenAI-compatible
        # server documentation.
        resp = self.client.post(f"{self.base_url}/v1/chat/completions", json=req, timeout=30)
        resp.raise_for_status()
        return _normalise(resp.json())
```

The pinned `model_hash` appears in every span (R11, R12 from [[exercise-01]]) so every provider-swap decision is traceable. A vLLM endpoint serving a non-pinned model is not a baseline — it is an experiment, and [[exercise-05]] contract tests will fail against it.

### 2f. Implementation notes

- **The baseline is pinned, the production endpoint is pinned, the two match.** A baseline run against model `A@sha_x` and a production endpoint quietly serving `A@sha_y` means the baseline is not describing production. Hash-pin both sides.
- **Replay traffic is the only honest workload.** Synthetic load tests are fine for capacity sizing; they are not fine for the capability comparison. Replay the surface's own traffic (scrubbed).
- **Judge quality dominates every other number on the sheet.** A self-host route that is 30 % cheaper at 5 % worse on the judge may still be the wrong call — the cost is paid by the human reading the reply.
- **Cold-start matters on autoscale.** A route whose fallback requires a vLLM replica to warm up has an SLO problem; the baseline must include cold-start drills.
- **Model-swap cadence is the sleeper variable.** Teams underestimate D6 until the second time a managed model-family release lands in a Tuesday meeting. If the route benefits from flagship upgrades, managed often wins even on raw cost.
- **Licences are not noise.** Several widely used open-weights families have use restrictions; read the licence. "Open" is not "unlicensed."
- **Managed is not free of operational risk.** Snapshot upgrades, content-policy drift, and vendor incidents are the managed path's analogue of GPU on-call. Both cost rows have a "surprises" line.
- **Decision is per route, but the baseline is often shared.** A single vLLM baseline can serve as evidence for several route decisions on the same open-weights model. Sign each route's note separately, but point them at the same sheet.
- **Residency trumps TCO.** If D1 says managed is excluded, the TCO comparison is academic; do not use TCO to re-argue a residency fact.

## 3. Validation steps

1. The decision model lists D1–D8 (or an equivalent set), each with a direction and an owner. A model that cannot articulate what would flip the decision is not a model; it is a preference.
2. The vLLM baseline sheet has a pinned model hash, a pinned serving binary version, and a declared SKU. Missing any of the three is a hard failure.
3. The workload used for the baseline is the surface's replayed traffic (scrubbed), not synthetic prompts.
4. The evaluation is `mod-106` offline suite — the same suite that measures the managed candidate. Different suites across the two options is a hard failure.
5. The baseline measures judge quality, throughput, p95/p99, cold-start, weight-load, and swap time. Numbers missing from this list are explicitly marked `not measured yet` rather than omitted.
6. The TCO frame has both option columns populated for every row; "included" without a number is fine only for the managed side where the number is contractual.
7. The decision note is per route, references the baseline sheet, names residual risks, lists triggers, and has three sign-off owners from different functions (route, platform, finance; security in parallel).
8. The vLLM endpoint plugs into the thin gateway as a `Provider` with the same contract-test coverage ([[exercise-05]]) as any managed provider.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Decision model with ≥ 8 variables and directions | 1 | Includes residency, swap cadence, HITL rate. |
| Per-route decisions, not per-surface | 1 | Note is written for one route; surfaces with multiple routes have multiple notes or an explicit "same as" link. |
| Baseline is pinned (hash + binary version + SKU) | 1 | No tag-based "latest" anywhere. |
| Workload is scrubbed replay, not synthetic | 1 | Workload file is version-controlled and PII-scrubbed. |
| Same `mod-106` evaluation on both options | 1 | Explicit; judge model named. |
| Baseline sheet covers all measurement rows | 1 | Missing rows marked `not measured yet`. |
| TCO over a two-year window including move-back cost | 1 | Row for move-back; row for governance; row for on-call. |
| Decision note per route with triggers | 1 | ≥ 3 triggers plus quarterly cadence. |
| Three sign-offs from different functions | 1 | Route + platform + finance or security. |
| vLLM endpoint plugs into [[exercise-01]] gateway | 1 | Not a one-off path; contract tests cover it. |

A submission scoring < 6 is rejected. Hard failures: (a) unpinned baseline, (b) different evaluation across options, (c) decision note without triggers, (d) TCO ignoring on-call or move-back cost.

## 5. Common mistakes

- **Comparing $/1M tokens and calling it a decision.** The unit price is one row of a ten-row table. A decision made from that row alone loses to D6 within a year on most hot routes.
- **Benchmarking on borrowed hardware.** A baseline on an un-representative SKU is not a baseline; it is a lie told with enthusiasm. The SKU in the baseline is the SKU production will run.
- **Different evaluation suites.** Comparing managed via vendor eval scores and self-host via `mod-106` suite is a category error. The evaluation must be the same artefact.
- **No judge-quality measurement.** Latency and cost without a quality number cannot be compared; a 50 % cheaper endpoint at 20 % worse quality is not "50 % cheaper."
- **Ignoring on-call.** Self-host without a named GPU on-call rotation is a plan with a missing section. The cost of standing up that rotation is in the TCO table.
- **No move-back budget.** Decisions get reversed. The TCO window must include the cost of reversal.
- **One decision for the whole surface.** `reply.compose` and `ticket.triage` do not have the same answer; writing a surface-wide decision overshoots on at least one of them.
- **Unpinned baseline.** A `model: open-m-latest` tag on the serving endpoint drifts; the baseline "worked yesterday" and silently regresses. Pin the hash.
- **"Open-weights model X is at parity" with no sheet.** Parity is a measured claim. The sheet is non-optional.
- **Letting the vLLM endpoint bypass the gateway.** A self-hosted endpoint reached directly by the surface bypasses tracing, scrub, moderation, and version-pinning. The endpoint is just another provider; the gateway is still the mount point.

## 6. References

- vLLM project documentation — <https://docs.vllm.ai/> — the authoritative reference for the baseline serving runtime: the OpenAI-compatible server contract (used in the `VLLMProvider` sketch), the paged-attention / continuous-batching behaviour that defines the throughput baseline, and the supported-model and quantisation docs referenced by the baseline recipe. The baseline binds the recipe to a specific vLLM version so a runtime bump is a new baseline.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML06 (AI Supply Chain) and ML10 (Model Poisoning) frame the self-host decision as a supply-chain shift: self-host moves weight custody and integrity verification in-house, which must be documented (hash pinning, provenance of weights, update path) in the baseline.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the baseline recipe implements MEASURE (replay workload, pinned evaluation, drill-based latency and cold-start numbers). The decision note implements GOVERN (roles, policies, triggers for re-decision) and MANAGE (move-back budget, named incident posture). TCO columns correspond to the RMF's resources dimension.
- MITRE ATLAS — <https://atlas.mitre.org/> — a self-host decision changes the ATLAS exposure profile: Reconnaissance and ML Model Access techniques that targeted the managed provider's API now target our own endpoint; the cross-reference to mod-107 [[exercise-05]] ensures the mapping runbook captures the move, and the routing fallback rules from [[exercise-02]] continue to enforce the defensive layers.

Related exercises: [[exercise-01]] (the gateway shape that lets managed and self-hosted endpoints be swapped without surface edits); [[exercise-02]] (vLLM is a `Target` whose primary/fallback rules live in the routing table); [[exercise-04]] (multi-modal routing — if the self-host candidate does not cover the modality, that is an automatic D2 exclusion); [[exercise-05]] (contract tests the vLLM endpoint must pass before it serves production traffic); mod-106 (offline evaluation suite consulted by the baseline); mod-109 (cost and latency SLOs the TCO is defending); mod-110 (observability of per-model cost and quality signals); mod-111 (the deployment of a managed-to-self-host or self-host-to-managed move); mod-107 [[exercise-05]] (mapping runbook updated when the self-host decision changes the supply chain).
