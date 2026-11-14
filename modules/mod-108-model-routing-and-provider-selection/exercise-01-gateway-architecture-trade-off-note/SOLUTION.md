### exercise-01: Gateway Architecture Trade-Off Note — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to produce a **trade-off note** that settles how the helpdesk-reply-copilot surface talks to its model and tool providers: inline via vendor SDKs, through an in-house "thin" gateway, through an in-house "full" gateway, or through a vendor LLM gateway. The deliverable is not a vendor comparison — it is a written decision, an owner, a review trigger, and the two to three signatures that will carry it through the first provider swap.

The failure mode the exercise is written to catch is the gateway decision that is never written down. The team installs a vendor SDK on day one, then wraps it on day thirty, then rewrites the wrapper on day ninety, and no document records who decided which of the three shapes is the surface's gateway. The trade-off note exists so that routing, caching, retries, PII scrub placement, and observability all hang off one pillar and the pillar has a name.

This solution provides:

- A **gateway responsibility checklist** — the set of concerns a gateway may be asked to own, so an option can be scored against the concerns actually reachable on this surface.
- An **option matrix** — four shapes (inline SDK, thin gateway, full gateway, vendor gateway) scored against the concerns.
- A **decision template** — the trade-off note itself, with recommendation, rejected options, residual risks, triggers for re-decision, and sign-off owners.
- A **grading rubric** for the instructor.

The lesson the exercise defends is: *a gateway is not a library choice, it is an architecture pillar.* Once routing, fallback, and observability depend on it, the shape of the gateway constrains the shape of every downstream exercise in this module.

## 2. Worked answer or implementation

### 2a. Gateway responsibility checklist (deliverable 1)

The checklist enumerates *every* concern a gateway may own, before an option is scored. Each row names the concern and the exercise or module where it is enforced; a concern can only be scored "in scope" if it maps to a surface requirement.

| # | Concern | Why the gateway may own it | Referenced from |
|---|---|---|---|
| R1 | Model routing (pick a model per request) | cost/latency/capability trade-off per route | [[exercise-02]] |
| R2 | Provider fallback (switch when provider fails) | provider incidents must not stall the surface | [[exercise-02]], mod-110 |
| R3 | Retry + backoff with jitter | transient 5xx and rate-limit storms | [[exercise-02]] |
| R4 | Rate limiting (per-tenant, per-route) | cost control; noisy neighbor | mod-109 |
| R5 | Prompt cache / response cache | cost and latency | mod-109, [[exercise-02]] |
| R6 | Structured-output validation | pydantic-style schema at the edge | mod-102 |
| R7 | Tool-call shape normalisation | different providers have different function-call envelopes | [[exercise-04]], [[exercise-05]] |
| R8 | Streaming normalisation | server-sent event formats differ by provider | [[exercise-04]] |
| R9 | PII / secret scrub on prompts | privacy before egress | mod-107 [[exercise-03]] |
| R10 | PII / secret scrub on traces | privacy on observability | mod-107 [[exercise-03]] |
| R11 | Tracing + span attribution | per-model, per-route, per-tenant metrics | mod-110 |
| R12 | Cost accounting per route | reconcile invoices against observed traffic | mod-109 |
| R13 | Cross-provider contract tests | the gateway is where normalisation is verified | [[exercise-05]] |
| R14 | Model-version pinning / canarying | provider silently upgrades a snapshot | [[exercise-02]], mod-111 |
| R15 | Multi-modal adapters (vision/audio/realtime) | not every provider speaks the same modality | [[exercise-04]] |
| R16 | Guardrail hooks (pre- and post-call) | moderators, injection defence, tool gates | mod-107 |
| R17 | Secret / credential custody | rotation, scoping, audit | platform |
| R18 | Request-ID propagation | correlate across tenant → gateway → provider | mod-110 |

A gateway option is only credible for this surface if it addresses R1, R2, R7, R9–R12, R14, R16–R18. The others are flexible.

### 2b. Option matrix (deliverable 2)

Score each option against the responsibilities above. Each cell is one of:

- `direct` — the option owns the concern without extra work.
- `code` — the team must write code in the option to own the concern.
- `limited` — the option supports the concern but with a material caveat.
- `no` — the option cannot own the concern; another layer must.

| Concern | Inline vendor SDK | Thin internal gateway | Full internal gateway | Vendor LLM gateway |
|---|---|---|---|---|
| R1 model routing | `no` (per-call hard-code) | `code` | `direct` | `direct` |
| R2 provider fallback | `no` | `code` | `direct` | `direct` |
| R3 retry+backoff | `limited` (per SDK) | `direct` | `direct` | `direct` |
| R4 rate limiting | `no` | `code` | `direct` | `limited` (tenant mapping) |
| R5 caching | `no` | `limited` (opt-in) | `direct` | `limited` (shared cache keys) |
| R6 schema validation | `limited` (SDK-only) | `code` | `direct` | `limited` |
| R7 tool-call normalisation | `no` (surface written per-provider) | `code` | `direct` | `direct` |
| R8 streaming normalisation | `no` | `code` | `direct` | `direct` |
| R9 scrub on prompts | `no` | `direct` | `direct` | `limited` (egress is already outside) |
| R10 scrub on traces | `code` (SDK hook) | `direct` | `direct` | `limited` (depends on vendor) |
| R11 tracing | `limited` (per SDK) | `direct` | `direct` | `limited` (format maps) |
| R12 cost accounting | `no` | `direct` | `direct` | `direct` (billing feed) |
| R13 contract tests | `no` (surface-coupled) | `direct` | `direct` | `direct` (gateway is the SUT) |
| R14 version pinning | `limited` (SDK option) | `direct` | `direct` | `direct` |
| R15 multi-modal | `no` (per modality) | `limited` (per adapter) | `direct` | `limited` (modality coverage differs) |
| R16 guardrail hooks | `no` | `direct` | `direct` | `limited` (depends on vendor plugins) |
| R17 credential custody | `no` (secrets in app) | `direct` | `direct` | `direct` (also a new third party) |
| R18 request-ID propagation | `code` | `direct` | `direct` | `direct` |

This is a surface-specific scoring, not a vendor leaderboard. The point is that an inline-SDK architecture pushes nine concerns onto the surface, and a vendor gateway pushes four concerns onto a third party whose roadmap is not ours.

### 2c. The trade-off note (deliverable 3)

The note is the artefact a reviewer reads. It is short and signed. A worked example for the helpdesk-reply-copilot surface:

```markdown
# Gateway trade-off note — helpdesk-reply-copilot — v1 — 2026-11-13

## Decision
The surface will talk to model providers through a **thin internal gateway**
("helpdesk-llm-gw") rather than inline SDK calls or a vendor LLM gateway.
The gateway owns R2, R3, R7–R14, R16–R18 from the responsibility checklist.
Routing (R1) and caching (R5) are in-scope but arrive incrementally; the
first routing rule in production is a static primary + fallback pair per
route defined in [[exercise-02]].

## Options considered
### Option A — inline vendor SDK (rejected)
- Pros: fastest day-one; vendor SDKs provide typed clients and streaming.
- Cons: every concern R7, R9–R12, R14, R16 lands in surface code; swapping
  providers is a cross-surface diff, not a config edit; the contract-tests
  exercise ([[exercise-05]]) has no single SUT to target.
- Why rejected: this surface serves two providers already and a multimodal
  modality is on the roadmap; concentrating normalisation in surface code
  means the next change to the tool-call envelope will touch every reply
  path.

### Option B — thin internal gateway (CHOSEN)
- Pros: owns the concerns the surface cannot carry; keeps provider SDKs as
  the thin edge, so provider-side deprecations are absorbed in one place;
  the gateway is the SUT for [[exercise-05]]; routing and fallback rules
  from [[exercise-02]] live in a config file next to the code.
- Cons: real on-call surface — the gateway is a hop and its latency budget
  must be enforced (see mod-109); the team owns incident response for it.
- Why chosen: the concerns the gateway must own are concrete today, and
  the team has the bandwidth to own one more service. A thin shape avoids
  rebuilding features that provider SDKs already carry.

### Option C — full internal gateway with caching, prompt registry, policy engine (rejected for now)
- Pros: eventual answer for R4, R5, R16 at scale.
- Cons: a prompt registry and a policy engine are not day-one concerns;
  building them before the routing rules stabilise is premature.
- Why rejected: deferred to a follow-up note when R4 and R5 become
  pressing. The thin shape must leave hooks for this (see "future
  extensions" below).

### Option D — vendor LLM gateway (rejected for now)
- Pros: fastest day-one coverage of R1–R3, R7, R8, R12, R14.
- Cons: introduces a third-party that becomes a second provider to govern
  under mod-107's framework (control owner, data residency, scrub
  policy); multi-modal coverage (R15) depends on the vendor; vendor
  outages add a correlated failure mode to the fallback design.
- Why rejected: on this surface the tenant data classification (see mod-107
  [[exercise-03]]) makes an additional data-plane hop a legal review, not
  just an architecture review. The note commits to a quarterly re-read.

## Scope boundary
- The gateway does **not** own guardrails themselves, only the hooks.
  Moderators ([[exercise-01]] mod-107), injection defence ([[exercise-02]]
  mod-107), PII scrub ([[exercise-03]] mod-107) are mounted at the
  gateway; they are not written there.
- The gateway does **not** own the retrieval index (mod-103) or the
  tool catalog (mod-104). It owns the call to a tool-executing provider,
  not the tool implementations.

## Residual risks (named)
- `RR-01` A thin gateway with no caching means mod-109 cost SLOs will be
  met by model-tier downshift, not cache. Owner: platform.
- `RR-02` A single internal gateway is a single point of failure;
  deployment is multi-AZ but the egress path to each provider is still
  one service. Owner: SRE. Drill: quarterly.
- `RR-03` The gateway is new code; contract tests ([[exercise-05]]) must
  be in CI before the first production route moves off inline SDK.

## Triggers for re-decision
- A new modality reaches 10 % of surface traffic.
- A new provider is added for any reason (regulatory, cost, capability).
- Any routing rule from [[exercise-02]] becomes impossible to express
  without a prompt registry.
- Any gateway incident whose postmortem recommends re-architecture.
- Quarterly review.

## Sign-off
- Surface tech lead ................ (initials, date)
- Platform tech lead .............. (initials, date)
- Security owner (mod-107 crosswalk) (initials, date)
```

### 2d. Reference "thin gateway" sketch

The note commits to a shape; the following sketch anchors the shape so the next exercise can route against it. It is not the full implementation — it is the smallest surface area that proves the concerns are addressable here.

```python
"""llm_gw.py — thin internal LLM gateway.

Responsibilities (checklist IDs from section 2a):
  R2 provider fallback, R3 retry+backoff, R7 tool-call normalisation,
  R8 streaming normalisation, R9/R10 scrub hooks, R11 tracing, R12 cost
  accounting, R13 contract-test SUT, R14 version pinning, R16 guardrail
  hooks, R17 credential custody, R18 request-id propagation.
Deferred:
  R1 routing (see [[exercise-02]]); R4 rate limiting (mod-109);
  R5 caching (mod-109); R15 multi-modal (see [[exercise-04]]).
"""
from __future__ import annotations

from dataclasses import dataclass
from typing import Callable, Iterable, Protocol

@dataclass(frozen=True)
class Call:
    route: str                 # logical route id, e.g. "reply.compose"
    model: str                 # pinned model id, e.g. "provider-a:model-x-2026-09"
    messages: list[dict]
    tools: list[dict] | None
    tenant_id: str
    request_id: str            # R18

class Hook(Protocol):
    name: str                  # "scrub.prompt", "moderation.pre", etc.
    def __call__(self, call: Call) -> Call: ...

class Provider(Protocol):
    name: str                  # "provider-a", "provider-b"
    def chat(self, call: Call) -> dict: ...

@dataclass
class Gateway:
    providers: dict[str, Provider]
    primary: dict[str, str]           # route -> provider name   (R1 today: static)
    fallback: dict[str, list[str]]    # route -> fallback chain (R2)
    pre_hooks: list[Hook]             # R9, R16
    post_hooks: list[Hook]            # R10, R16
    tracer: Callable[[Call, dict], None]  # R11, R12

    def chat(self, call: Call) -> dict:
        for hook in self.pre_hooks:
            call = hook(call)
        last_err: Exception | None = None
        for provider_name in [self.primary[call.route], *self.fallback.get(call.route, [])]:
            provider = self.providers[provider_name]
            try:
                raw = _retry(provider.chat, call)      # R3
                normalised = _normalise(raw)           # R7, R8
                self.tracer(call, normalised)          # R11, R12
                for hook in self.post_hooks:
                    normalised = hook(normalised)
                return normalised
            except _FatalProviderError as exc:
                last_err = exc
                continue
        raise _GatewayExhausted(call, last_err)
```

The sketch omits streaming and tool-call envelope details on purpose — those belong to [[exercise-05]]'s contract tests, which assert the normalisation shape.

### 2e. Implementation notes

- **A gateway is a dependency, not a convenience.** Every concern the gateway owns is a concern the surface *stops* owning. The checklist in 2a is the thing that makes "stops owning" legible to a reviewer; without it, the gateway grows incidentally.
- **Thin before full.** Caching, policy engines, and prompt registries belong to a full gateway. Building them before routing and fallback stabilise is a common cause of rework; the trade-off note names the trigger that promotes thin to full.
- **Latency budget belongs in mod-109.** The gateway adds a hop. The note points to mod-109 for the number; it does not claim a budget on its own.
- **Vendor gateways are second providers.** A vendor gateway governs under the same mod-107 crosswalk as the model provider — owner, data class, scrub policy. The rejection reasoning in 2c makes that explicit so a later "can we add Portkey next quarter?" decision has prior art.
- **The gateway is where contract tests run.** The sign-off commits to this: the contract tests in [[exercise-05]] target the gateway, not individual provider SDKs.
- **The note is version-controlled.** A new shape (e.g. adding R4 rate limiting) is a new revision with the previous revision linked, not an edit.

## 3. Validation steps

1. The responsibility checklist names every concern that the surface will need the gateway to carry, with a reference to the module or exercise that enforces it. Missing R2, R7, R9, R11, R14, or R16 is a hard failure.
2. The option matrix scores at least three options against every responsibility in the checklist; cells are one of the four allowed values.
3. The trade-off note states a single decision and names the rejected options with reasons. "We'll decide later" is not a decision.
4. Residual risks are explicit, numbered, and each has an owner.
5. Triggers for re-decision are enumerated. A note without triggers is a note that will be re-written on vibes.
6. Sign-off owners are named by role. Three signatures from different functions (surface, platform, security) is the minimum.
7. The note references downstream exercises ([[exercise-02]], [[exercise-04]], [[exercise-05]]) so those exercises have a defined target.
8. If the chosen option is "vendor gateway," the note includes the mod-107 crosswalk entry showing the vendor as a governed third party (NIST AI RMF GOVERN function, OWASP ML Top 10 ML06 supply-chain concern).

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Responsibility checklist is complete and traced | 1 | ≥ 15 concerns, each mapped to a module/exercise. |
| Four options scored, no blanks | 1 | Inline SDK + thin + full + vendor options; all R# scored. |
| Decision is singular and named | 1 | One chosen option; rejected options listed with reasons. |
| Residual risks are owned | 1 | ≥ 2 named risks, each with an owner. |
| Triggers for re-decision enumerated | 1 | Quarterly + ≥ 3 events. |
| Sign-off from three functions | 1 | Surface + platform + security, by role not by team. |
| Downstream exercises referenced | 1 | Note points to [[exercise-02]], [[exercise-04]], [[exercise-05]]. |
| Governance framing when vendor is chosen | 1 | If vendor option chosen, mod-107 crosswalk reference present; if rejected, the rejection cites the same framing. |
| Scope boundary is explicit | 1 | Note says what the gateway does **not** own. |
| Not a vendor comparison | 1 | Reviewer cannot find a "the X vendor is better" claim; the note is about the shape, not the brand. |

A submission scoring < 6 is rejected. Hard failures: (a) no residual risks named, (b) no sign-off owners, (c) note is a vendor leaderboard.

## 5. Common mistakes

- **Deciding by vendor features.** A trade-off note that evaluates specific vendor gateways against each other picks a vendor before the surface has stated what it needs. The shape of the gateway comes first; the vendor comes last.
- **No scope boundary.** If the note doesn't say what the gateway does *not* own, the gateway will absorb retrieval, memory, tool implementations, and guardrail content within six months. The scope boundary is the thing that makes "mount a hook" an option instead of "write the moderator here."
- **Treating "inline SDK" as a non-option.** For small surfaces inline SDK is the right answer. Rejecting it requires argument, not reflex.
- **Pretending a vendor gateway is not a third party.** A vendor LLM gateway handles tenant data on egress. It is a governed third party under mod-107, and the note must say so if it is adopted.
- **Over-scoping the gateway.** Caching, prompt registries, and policy engines are full-gateway concerns. Shipping them in the thin version is scope creep that will not survive the first incident review.
- **No measurement.** The gateway must report per-route, per-provider, per-model, per-tenant volume and cost. If the note does not commit to R11 and R12, the provider-swap decision in [[exercise-02]] cannot be made.
- **"We'll just use `litellm`."** That is a tool, not a decision. The note must still say what responsibilities land on `litellm` and which land on the surface; the four-column matrix does not get to collapse because a dependency is chosen.
- **No triggers.** A note without triggers is a document that will be re-opened at the next outage with no prior framing; the trigger list is how the team avoids re-litigating.
- **Single-sign-off.** One signature on an architecture pillar is a political liability. Three signatures from different functions is the floor.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML06 (AI Supply Chain) is the item this note engages most directly: a gateway is itself a supply-chain choice, and a vendor gateway adds a third party to the supply chain that must be governed. The responsibility checklist's R14 (version pinning) and R17 (credential custody) are the controls that answer ML06 on this surface.
- MITRE ATLAS — <https://atlas.mitre.org/> — the gateway is the enforcement point for several ATLAS tactics observed on this surface: pre-call hooks mount the moderator (mod-107 [[exercise-01]]) and injection defence (mod-107 [[exercise-02]]) against Initial Access via prompt injection; post-call hooks mount the output sanitiser against Defence Evasion; R11 tracing provides the attribution a response to Collection / Exfiltration requires.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the trade-off note is the GOVERN-function artefact for the routing/provider subsystem: it records roles (sign-off owners), policies (scope boundary), and an accountability path (triggers for re-decision). R11 and R12 implement the MEASURE function at the gateway level; R2 and R14 implement MANAGE.
- vLLM project documentation — <https://docs.vllm.ai/> — referenced here because [[exercise-03]] will anchor the self-host option on a vLLM baseline; the responsibility checklist's R14 and R17 apply equally when the "provider" is a self-hosted vLLM endpoint, so the same gateway shape covers both cases.

Related exercises: [[exercise-02]] (the routing + fallback rules this gateway will enforce); [[exercise-03]] (the self-host decision whose chosen baseline plugs into this gateway as "just another provider"); [[exercise-04]] (multi-modal adapters mounted at R15); [[exercise-05]] (contract tests whose SUT is this gateway); mod-107 [[exercise-05]] (mapping runbook, which lists the gateway as the enforcement point for several controls); mod-109 (cost and latency SLOs the gateway's measurements feed).
