### exercise-02: Prompt-Cache-Driven System Prompt Design — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to treat **provider prompt caching** as a first-class property of the surface's system prompt, not an afterthought. That means auditing the current prompt's byte-identical prefix, reordering it so stable content sits above volatile content, placing explicit cache breakpoints where the provider supports them, measuring the cache-hit ratio before and after, and committing a **cache-posture doc** that pins the release cadence and alert threshold.

The failure mode the exercise is written to catch is the common one: the prompt caches nothing because a timestamp template or a user-ID insertion sits above the tool schemas. Nobody notices until the bill arrives; nobody can rediscover it later because the prompt has drifted through ten "small" edits that each silently invalidated the prefix. The cache-posture doc exists so that the next prompt edit knows what it is paying for and the next model swap knows what it is giving up.

This solution provides:

- A **cache-audit playbook** — how to dump the exact JSON payload the SDK sends and classify every section as stable or volatile.
- A **provider-by-provider cache mechanic reference** — explicit breakpoints (Anthropic, Bedrock), automatic prefix matching (OpenAI, Gemini automatic), explicit context cache (Gemini). Verify the current field names against provider docs before shipping.
- A **worked redesign** of a representative helpdesk-reply-copilot prompt with before / after cache-shape tables.
- A **measurement harness** — a small driver that warms the cache, runs a sampled workload, reads the provider's cache-read field, and computes the hit ratio, write ratio, and TTFT delta.
- A **cache-posture doc template** filled in for the worked example.
- A **grading rubric** for the instructor.

The lesson the exercise defends: *the cache is a shape constraint on the prompt, not a feature you turn on.* Teams that treat the cache as "add a flag" ship prompts whose first cache-invalidating edit vaporises the savings they justified the launch with. Treating it as a shape means every prompt change goes through the posture doc.

## 2. Worked answer or implementation

### 2a. Cache-audit of the current prompt (deliverable 1)

The audit is done on the **byte stream the SDK actually sends**, not on the mental model of the prompt. Dump the request body:

```python
# dump_payload.py
import json
import logging

logging.basicConfig(level=logging.DEBUG)   # most SDKs log the body at DEBUG
# ... run one request through the current prompt build ...
# Capture via a middleware layer or by printing the payload before send:
payload = build_request(ctx)
with open("payload_before.json", "w") as f:
    json.dump(payload, f, indent=2, sort_keys=False)  # sort_keys=False keeps actual order
```

Then classify every section.

```markdown
# docs/cache-audit-before.md

| Section                       | Byte offset | Volatile? | Reason                                   |
|-------------------------------|-------------|-----------|------------------------------------------|
| Persona + rules header        | 0           | no        | Static; edited rarely (~quarterly).      |
| `Current time: {{now}}` line  | ~480        | **yes**   | Changes every request → cache killer.    |
| Tool schemas (JSON)           | ~560        | no*       | Static, but key order comes from `dict`  |
|                               |             |           | insertion — not guaranteed deterministic |
|                               |             |           | across SDK versions. *conditional.       |
| Static few-shot (3 examples)  | ~3200       | no        | Static.                                  |
| `User ID: {{user_id}}` line   | ~4700       | **yes**   | Changes per user → cache killer.         |
| Conversation history          | ~4760       | partial   | Stable from oldest-turn outward; new     |
|                               |             |           | turns append at the bottom.              |
| Current user turn             | end         | yes       | Always changes; correct placement.       |

## Cache killers (minimum 2 required)
1. `Current time` line at ~480 bytes invalidates everything that follows.
2. `User ID` line at ~4700 bytes invalidates the few-shot block plus history.

## Secondary risk
Tool-schema JSON is "logically static" but produced by `json.dumps(schema)`
with Python-dict insertion order. On SDK/environment change the key order
can shift → byte-non-identical → cache invalidation on redeploy.
```

Two cache killers is the audit floor. In practice almost every un-audited prompt has three to five, because "put the timestamp first so the model always knows the date" is common first-draft advice.

### 2b. Provider cache mechanic reference

Verify current field names and limits against the provider's current docs before deployment — these mechanics are evolving. The table below is the shape that stays stable.

| Provider | Mechanic | Who places breakpoints | Minimum prefix | TTL | Response field (verify) |
|---|---|---|---|---|---|
| OpenAI | Automatic prefix matching | Provider | Provider-defined | Provider-managed | `usage.prompt_tokens_details.cached_tokens` <!-- verify-before-publishing: confirm current field path in latest OpenAI docs --> |
| Anthropic | Explicit `cache_control` markers | Client (you) | Provider-defined | Standard ~5 min; extended beta available at higher write cost <!-- verify-before-publishing: confirm current TTL options and pricing multipliers --> | `usage.cache_read_input_tokens`, `usage.cache_creation_input_tokens` <!-- verify-before-publishing: confirm current field names --> |
| Bedrock | Explicit cache points (provider-specific) | Client | Provider-defined | Provider-managed | Provider-specific <!-- verify-before-publishing: current Bedrock cache fields per model vendor --> |
| Gemini | Both: implicit prefix caching and explicit `CachedContent` API | Hybrid | Provider-defined | Explicit TTL on `CachedContent` | `usage_metadata.cached_content_token_count` <!-- verify-before-publishing: confirm current response shape --> |

What stays true across providers:

- **Prefix-sensitive**: caches operate on a byte-identical prefix, so the cache-shape question is always "what is the longest prefix that is byte-identical across requests?"
- **Write vs read**: explicit-breakpoint providers charge extra for cache writes; the read / write ratio must exceed some floor for caching to pay.
- **TTL is finite**: if the surface's QPS is lower than `1 / TTL`, the cache expires between requests and caches nothing.

### 2c. Redesigned prompt (deliverable 2)

The redesign enforces a strict stable → volatile order and moves logging-only variables off the prompt.

```python
# prompt.py — redesigned.
# Layout (top = stable, bottom = volatile):
#   1. Persona + rules (versioned, byte-stable)
#   2. Tool schemas (JSON with sort_keys=True)
#   3. Static few-shot block (byte-stable)
#   4. [cache_control breakpoint on explicit-breakpoint providers]
#   5. Rolling conversation history (older turns first; stable prefix grows)
#   6. [optional rolling breakpoint just before current user turn]
#   7. Current user turn
#
# Deleted from the system prompt (moved to observability span):
#   - Current timestamp (was at offset ~480)
#   - User ID (was at offset ~4700)

def build_messages(ctx) -> list[dict]:
    system_blocks = [
        {"type": "text", "text": PERSONA_AND_RULES_V3},                     # (1) stable
        {"type": "text", "text": json.dumps(TOOL_SCHEMAS, sort_keys=True)}, # (2) stable + deterministic
        {"type": "text", "text": STATIC_FEWSHOT_V2,
         "cache_control": {"type": "ephemeral"}},                           # (4) explicit breakpoint
    ]
    history_blocks = [_turn(t) for t in ctx.history]
    if history_blocks:
        history_blocks[-1]["cache_control"] = {"type": "ephemeral"}         # (6) rolling breakpoint
    user_turn = _turn(ctx.current_turn)                                     # (7) volatile
    return [{"role": "system", "content": system_blocks}] + history_blocks + [user_turn]
```

Key design choices:

- **`sort_keys=True` on tool-schema JSON.** Without this, `{"name": ..., "parameters": ...}` and `{"parameters": ..., "name": ...}` serialise to different byte strings and the cache misses on every request.
- **Breakpoints at the end of each stable block.** The cache reads up to the last successful breakpoint, so placing the breakpoint after the static few-shot gets you the longest prefix the surface controls.
- **Rolling breakpoint on conversation history.** For multi-turn chats, placing a breakpoint just before the current turn means an N-turn conversation caches N-1 turns of prefix on the next turn.
- **Timestamp and user_id removed.** They were on the prompt for logging and personalisation; both move to `app.feature` / `app.request_id` / `app.tenant_id` on the observability span (from [[exercise-01]]).

For OpenAI / Gemini-automatic providers, the same ordering *is* the design — delete the `cache_control` markers and rely on the provider's automatic prefix match. Nothing else changes; this is the point.

### 2d. Measurement harness (deliverable 3)

```python
# measure_cache.py — reads cache-hit fields from the provider response.

PROVIDER_CACHE_FIELDS = {
    "openai":     ("usage.prompt_tokens_details.cached_tokens", None),
    "anthropic":  ("usage.cache_read_input_tokens",
                   "usage.cache_creation_input_tokens"),
    "gemini":     ("usage_metadata.cached_content_token_count", None),
    # Bedrock varies by model vendor; verify fields and populate per route.
}

def record_cache(resp, provider, span):
    read_path, write_path = PROVIDER_CACHE_FIELDS[provider]
    cached = _dig(resp, read_path) or 0
    writes = _dig(resp, write_path) or 0 if write_path else 0
    span.set_attribute("app.cache.read_tokens", cached)
    span.set_attribute("app.cache.write_tokens", writes)
    input_tokens = resp["usage"]["input_tokens"]
    span.set_attribute("app.cache.hit_ratio", cached / max(1, input_tokens))

def run_sample(prompt_builder, n_warmup=25, n_measure=120, qps=1.0):
    for i in range(n_warmup):
        call_surface(prompt_builder(f"warmup-{i}"))
        time.sleep(1.0 / qps)
    stats = []
    for i in range(n_measure):
        resp, latency = call_surface(prompt_builder(f"measure-{i}"))
        stats.append(_extract_stats(resp, latency))
        time.sleep(1.0 / qps)
    return stats
```

Run this twice, once against the **control** (unmodified prompt) and once against the **redesign**, at a QPS high enough to keep the cache warm within the provider's TTL. Record:

- `cache_hit_ratio_p50 = p50(stats.hit_ratio)` — the typical case.
- `cache_write_ratio = sum(writes) / (sum(writes) + sum(reads))` — on explicit-breakpoint providers.
- `ttft_delta_p95 = p95(hit_ttft) - p95(miss_ttft)` — join to the exercise-01 instrumentation.

Plausible-shape before / after for the worked example (your numbers will differ by provider and QPS):

```
Control (original prompt):
  cache_hit_ratio_p50:  0.02        (near zero — timestamp kills prefix)
  cache_write_ratio:    0.90        (writes dominate; reads near zero)
  ttft_p95_hit:         n/a
  ttft_p95_miss:        1180 ms

Redesign:
  cache_hit_ratio_p50:  0.74        (persona + tools + few-shot cached)
  cache_write_ratio:    0.08        (writes decay as reads accumulate)
  ttft_p95_hit:         620 ms
  ttft_p95_miss:        1140 ms
  ttft_delta_p95:       520 ms      (hit is ~45% faster)
```

A 3× hit-ratio uplift is the usual shape of a working redesign on an explicit-breakpoint provider at this traffic rate; automatic-prefix providers often show less dramatic a jump because the pre-redesign prompt may have been accidentally cacheable for short bursts.

### 2e. Cache-posture doc (deliverable 4)

```markdown
# Cache posture — helpdesk-reply-copilot — v1 — 2026-11-14

## Content
Prompt source: `src/prompt.py` at commit `a1b2c3d` (PERSONA_V3 + STATIC_FEWSHOT_V2).

## Cache shape
| Section                 | Byte offset (approx) | Breakpoint? | Volatile? |
|-------------------------|----------------------|-------------|-----------|
| Persona + rules         | 0                    | no          | no        |
| Tool schemas (sorted)   | ~560                 | no          | no        |
| Static few-shot         | ~3200                | YES (end)   | no        |
| Rolling history         | ~4800+               | YES (end of history, rolling) | partial |
| Current user turn       | end                  | no          | yes       |

## Providers wired
- Anthropic: 2 `cache_control: ephemeral` markers; TTL standard (~5 min).
  Extended TTL beta considered; write-cost multiplier does not pay at
  current QPS. Verify current pricing before re-evaluating.
- OpenAI: automatic prefix matching; ordering above also governs.
- Gemini: not wired for this surface (single-provider for now).

## Metrics (measured 2026-11-14, sample size 120 post-warmup)
- Cache-hit ratio (p50): control 0.02 → redesign 0.74.
- Cache-write ratio: 0.08 (reads/writes ≈ 11:1 — healthy).
- TTFT delta (miss − hit) p95: 520 ms (hit is faster).
- Effective input-cost delta (per 1k input tokens, Anthropic pricing as of
  measurement date): ~71% lower on cache-hit path. <!-- verify-before-publishing:
  recompute exact pricing multiplier against current Anthropic rate card -->

## Release cadence
- Cache-preserving edits (append-only below the last breakpoint): weekly.
- Cache-invalidating edits (persona, rules, tool-schema shape): quarterly
  or on security review, with 48h advance notice in #helpdesk-platform so
  the ratio dip is expected on dashboards.

## Alert
- Cache-hit ratio (p50 over 30 min) < 0.40 → ticket to helpdesk-platform.
- Cache-hit ratio drop > 0.20 in 24h vs trailing 7-day median → ticket.

## Cache-preserving edit — worked example
Add a new few-shot example demonstrating the "refund with partial return"
intent. The edit MUST be appended *after* the current breakpoint, not
inserted into the existing few-shot block:

```python
STATIC_FEWSHOT_V2 = EXISTING_EXAMPLES                       # breakpoint ends here
FEWSHOT_APPEND_PACK = "## Refund partial return\n…"         # new block below
system_blocks = [
    {"type": "text", "text": PERSONA_AND_RULES_V3},
    {"type": "text", "text": json.dumps(TOOL_SCHEMAS, sort_keys=True)},
    {"type": "text", "text": STATIC_FEWSHOT_V2,
     "cache_control": {"type": "ephemeral"}},
    {"type": "text", "text": FEWSHOT_APPEND_PACK},          # outside the cache
]
```
Expected ratio dip: none on the cached prefix; the new block is just
~400 fresh input tokens that pass through uncached until a second
breakpoint is added in the next cadence window.

## Owner and revisit
Owner: helpdesk-platform. Reviewed: 2026-11-14. Next review: 2027-02-01.
```

### 2f. Implementation notes

- **Sort tool-schema JSON keys.** The single most common invisible cache killer across teams is `json.dumps(schema)` with non-deterministic key order. `sort_keys=True` fixes it; test that two serialisations are byte-identical before trusting the cache metrics.
- **Timestamps and user IDs belong on the span, not the prompt.** The model rarely needs them; the observability stack always does. Moving them is a two-line change that unlocks the cache.
- **Write-side cost is real.** On Anthropic, cache_creation_input_tokens is more expensive than base input per token. If the surface's QPS is low enough that writes dominate, caching loses money. Confirm reads ≥ 5 × writes before declaring victory. Numbers vary by provider pricing; recompute before rollout.
- **TTL × QPS is the caching-feasibility inequality.** If `1 / QPS > TTL`, the cache expires between requests and the ratio stays near zero. Either raise the sample rate (co-locate traffic), synthesise keep-alives, or use a provider's extended-TTL option — knowing that the extended TTL costs more at write time.
- **Automatic-prefix providers are not "free caching."** Ordering still matters: the provider's automatic prefix match still fails at the first differing byte. The redesign's ordering is correct for both automatic and explicit cases.
- **Dashboards need release markers.** A 20% ratio drop after a release is expected if the release was cache-invalidating; without release markers on the chart, future-you reads the drop as an incident.
- **Measure TTFT delta, not just token count.** The cost-saving story is important; the latency-saving story is often more visible to users. Join the cache metrics to the TTFT instrumentation from [[exercise-01]].

### 2g. Mapping to the exercise deliverables

| Exercise asks for | Section of this solution |
|---|---|
| Cache audit (≥ 2 killers named) | 2a |
| Redesigned prompt (stable → volatile) | 2c |
| Measurement run + ratio computation | 2d |
| `docs/cache-posture.md` | 2e |
| Cache-preserving edit example | 2e (bottom) |

## 3. Validation steps

1. **Byte-identical prefix check.** Serialise the redesigned payload for two consecutive requests with different user turns; the first N bytes up to the current user turn must match exactly. A `diff` on the two payloads shows no differences above the final volatile section. Failure means a hidden volatile insertion somewhere in the stable blocks.
2. **Deterministic tool-schema check.** `json.dumps(TOOL_SCHEMAS, sort_keys=True)` must be byte-identical across two Python processes. Add a unit test that asserts this.
3. **Cache-hit field present.** On every response, the configured cache-read field (per provider) is non-null. A `None` here usually means the provider field path has changed or the request shape did not pass the provider's minimum-prefix threshold.
4. **Hit ratio uplift.** `redesign.hit_ratio_p50 > 3 × control.hit_ratio_p50`. The exact multiple is provider-dependent, but a non-significant uplift means the redesign did not work — do not publish the posture doc.
5. **Read / write ratio floor.** On explicit-breakpoint providers, `reads / writes ≥ 5`. Below this, caching is a cost regression masquerading as a win.
6. **TTFT delta present.** Join on `app.cache.hit_ratio > 0` versus `== 0`; the p95 TTFT on the hit slice must be measurably lower. If not, the "hit" is a false positive (the provider counted a short prefix and the rest was fresh).
7. **Release cadence named.** The posture doc commits to a cadence; without it, prompt edits will drift the cache away silently.
8. **Alert threshold named.** `< 0.40` is a sample; whatever the floor, it must be a number, not an adjective.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Audit identifies ≥ 2 cache killers in the original prompt | 1 | Named with byte offset and reason. |
| Redesign places stable content above volatile content | 1 | Tool schemas and few-shot above history / current turn. |
| Explicit breakpoints used on breakpoint-supporting providers | 1 | `cache_control` placed at the end of stable blocks. |
| Tool-schema JSON serialised deterministically | 1 | `sort_keys=True` or equivalent; test asserts byte-identity. |
| Timestamps / user IDs moved off the prompt | 1 | Removed from prompt, present on observability span instead. |
| Measurement run uses provider cache-hit fields | 1 | Correct response-field path per provider. |
| Hit-ratio uplift meaningfully larger than control | 1 | ≥ 3× on explicit-breakpoint providers; non-trivial on automatic. |
| TTFT delta reported | 1 | p95 hit vs p95 miss both shown, delta named. |
| Cache-posture doc complete | 1 | Shape table + providers + metrics + cadence + alert. |
| Cache-preserving edit example present | 1 | Edit is append-only below the last breakpoint and explained. |

A submission scoring < 6 is rejected. Hard failures: (a) no byte-identical-prefix check, (b) no measured hit-ratio delta, (c) no alert threshold.

## 5. Common mistakes

- **"We'll add `cache_control` and be done."** The marker alone caches nothing if the bytes above it are volatile. Order first, then mark.
- **Non-deterministic JSON serialisation.** `json.dumps({…})` without `sort_keys=True` produces insertion-ordered output. A refactor that touches schema-construction order silently breaks the cache.
- **Timestamp as "grounding."** Models do not need the current time for 99% of chat surfaces; the model's cutoff plus the user turn is enough. If the model really needs the time, put it in the user-turn area, not the system prompt.
- **Measuring only hit ratio.** Reads / writes ratio is the economics story; TTFT delta is the user-experience story. A hit ratio of 0.60 with writes dominating is a worse deal than a hit ratio of 0.40 with 10× reads.
- **Chasing hit ratio by moving content around weekly.** Every move invalidates the prefix. The release cadence in the posture doc is not red tape — it is the thing that lets the cache actually warm.
- **Ignoring the write-cost beta.** On Anthropic's extended-TTL beta, writes cost more per token. If you opt in without recomputing payback, the surface loses money quietly. Recompute before rollout.
- **Automatic-prefix complacency.** Teams on OpenAI / Gemini-automatic sometimes assume caching "just works" — then wonder why the ratio is 0.1. Ordering matters everywhere; the only difference is where the breakpoints are placed.
- **No alert.** A posture without an alert is a doc that notices cache regressions by the monthly invoice. The hit-ratio alert catches silent regressions in hours, not weeks.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML06 (AI Supply Chain) applies to prompt caching as a vendor-dependency decision: a cache-posture doc that pins provider mechanics and TTLs is what lets a provider migration (mod-108 [[exercise-02]]) carry its cost story. The alert threshold is the operational control that catches a provider-side cache change.
- MITRE ATLAS — <https://atlas.mitre.org/> — the cache-shape audit is also a surface-area audit for prompt-injection defence: moving volatile user-controlled content to the bottom of the payload means a cached, trusted prefix is not interleaved with attacker-shaped content. The adversarial-prefix tactics ATLAS catalogues are easier to reason about when the prompt is split into stable (trusted) and volatile (user-controlled) regions.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the cache-posture doc is the GOVERN artefact for prompt-cache decisions (release cadence, owner, revisit date) and the MEASURE artefact for the hit-ratio and TTFT-delta metrics. The alert belongs to the MANAGE function.
- Provider prompt-caching documentation — verify current field names, TTLs, and pricing before publishing the posture doc:
  - Anthropic prompt caching: <https://docs.anthropic.com/> <!-- verify-before-publishing: confirm current `cache_control` schema, extended-TTL beta status, and per-token write-cost multiplier -->
  - OpenAI prompt caching: <https://platform.openai.com/docs/> <!-- verify-before-publishing: confirm current `cached_tokens` field path and automatic-prefix minimum -->
  - Google Gemini context cache: <https://ai.google.dev/> <!-- verify-before-publishing: confirm current `CachedContent` API shape, TTL field, and pricing -->
  - Amazon Bedrock prompt caching: <https://docs.aws.amazon.com/bedrock/> <!-- verify-before-publishing: confirm current cache mechanics per model vendor -->

Related exercises: [[exercise-01]] (the TTFT instrumentation this exercise consumes); [[exercise-04]] (per-tenant cost accounting reads `app.cache.read_tokens` for the cache-aware cost function); [[exercise-05]] (the hit-ratio drop alert feeds the slow-burn runbook); mod-102 (structured prompts) is where the prompt-version discipline lives; mod-108 [[exercise-01]] (gateway shape) is where the cache metrics are stamped onto the span.
