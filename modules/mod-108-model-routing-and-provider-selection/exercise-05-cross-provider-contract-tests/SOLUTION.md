### exercise-05: Cross-Provider Contract Tests — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to write **contract tests** that pin the request / response shape and semantics the helpdesk-reply-copilot surface relies on, and to run those tests against every provider the thin gateway from [[exercise-01]] can route to. "Contract" here is the small, provider-neutral surface the gateway promises: shape of the request, shape of the response, the names and types of every field the surface consumes, the error classes the gateway maps providers' errors onto, the normalised tool-call envelope, the streaming event order, and the modality shapes from [[exercise-04]]. A new provider is only in production when its contract tests pass.

The failure mode the exercise is written to catch is the silent provider drift. On Monday the gateway normalises a provider's `function_call` envelope one way; on Wednesday the provider ships an "improved" response field; nothing fails until the surface quietly starts returning empty tool invocations and someone notices via a tenant ticket. Contract tests exist because provider release notes do not — or do, in the past tense. The contract test is the boundary where normalisation is enforced and where a regression blocks a route from going live.

This solution provides:

- A **contract specification** — the exact fields and semantics the surface consumes, authored once, independent of any provider.
- A **test taxonomy** — the dimensions a contract suite must cover (chat, tool call, structured output, streaming, errors, modalities), each with a concrete test shape.
- A **reference implementation** — a `pytest`-shape suite that targets the gateway (not individual provider SDKs), runs against a replay-recordable mock and against live providers, and gates provider promotion.
- A **drift and promotion workflow** — how a new provider is added, how a provider version bump is handled, and what a contract-test failure in CI means for production routing.
- A **grading rubric** for the instructor.

The lesson the exercise defends is: *the gateway's contract is the only real portability story.* If a surface's handlers unpack provider-shaped payloads directly, the surface is married to the provider; contract tests at the gateway are what let a routing-table change from [[exercise-02]] be a config edit instead of a surface refactor.

## 2. Worked answer or implementation

### 2a. Contract specification (deliverable 1)

The specification is written first and refers to no provider by name. Its only consumer is the gateway; its only writers are the surface team.

```yaml
# gateway_contract.yaml — helpdesk-reply-copilot — v1.2
# Provider-neutral request / response the gateway exposes to the surface.

chat:
  request:
    required: [route, model, messages]
    optional: [tools, response_format, tenant_id, request_id, trace_id, region_pin, modality]
    messages:
      each:
        role: ["system", "user", "assistant", "tool"]
        content:
          one_of:
            - type: "text"; required: [text]
            - type: "image"; required: [image_ref]     # provider-neutral image reference
            - type: "audio"; required: [audio_ref]
  response:
    required: [model_served, provider_served, usage, output]
    output:
      content:
        each:
          type: ["text", "tool_call"]
          tool_call:
            required: [id, name, arguments_json]       # arguments are JSON-valid at this boundary
    usage:
      required: [prompt_tokens, completion_tokens, modality_units]
    error:
      required: [class, retryable, provider_error_code]
      class: ["transient_5xx", "rate_limited", "unavailable",
              "content_refusal", "schema_violation", "tool_shape_mismatch",
              "contract_regression", "invalid_request"]

streaming:
  event_order: ["start", "delta*", "tool_call?", "finish"]
  delta:
    required_fields: ["seq", "text|tool_call_fragment"]
    seq: "monotonic non-decreasing"
  finish:
    required: ["finish_reason", "usage"]

structured_output:
  modes: ["json_schema", "none"]
  guarantee: "if mode == json_schema, output.content[0].text parses against the submitted schema"

tools:
  call_envelope:
    required: [id, name, arguments_json]
    arguments_json: "valid JSON per the tool's declared schema"
  result_envelope:
    required: [tool_call_id, content]   # the surface writes this back; gateway accepts in next turn

modalities:
  vision_in:
    image_ref:
      one_of:
        - {type: "url", required: [url]}
        - {type: "inline", required: [mime, data_b64]}
  audio_in:
    audio_ref: {type: "object_ref", required: [uri, mime, duration_s]}
  audio_out:
    stream: "chunked audio; mime declared on first event"
  realtime:
    see: "gateway_realtime_contract.yaml (separate spec)"
```

This file is the source of truth. The suite below is generated from it where practical (e.g. error-class table) and hand-authored where generation would obscure semantics.

### 2b. Test taxonomy (deliverable 2)

A contract suite covers the following dimensions. Each dimension has a minimum test cardinality; omitting a dimension fails review.

| # | Dimension | What it pins | Minimum cardinality |
|---|---|---|---|
| T1 | Basic chat | request/response shape; `usage` populated | 2 |
| T2 | Multi-turn with tool call | `tool_call` envelope normalised; `arguments_json` valid JSON | 2 per registered tool |
| T3 | Structured output | JSON-schema mode returns parseable text | 1 per declared schema |
| T4 | Streaming | event order; `seq` monotonic; `finish` always arrives | 2 |
| T5 | Error classes | each `class` enumerated in the spec is producible and mapped | 1 per class |
| T6 | Modality shapes | `image_ref` URL + inline; `audio_ref`; one realtime smoke | 1 per modality the provider claims |
| T7 | Model-version pin | request specifying `model: X@sha` returns `model_served == X@sha` | 1 |
| T8 | Determinism posture | same `(messages, temperature=0, seed=S)` returns identical structure (not identical text unless provider guarantees it) | 1 |
| T9 | Timeout and cancellation | cancelled stream closes with `finish_reason="cancelled"` | 1 |
| T10 | Region pin | `region_pin=eu` served from EU provider endpoint (where provider supports it) | 1 per supported pin |

T5 is the most-missed dimension in the wild. The suite must be able to *induce* each error class deterministically so that the fallback rules in [[exercise-02]] can be tested against them.

### 2c. Reference implementation (deliverable 3)

The suite is a `pytest` package. The system under test is the gateway, not the provider SDK; the provider is parameterised.

```python
# tests/contract/test_gateway_contract.py
"""Gateway contract tests.

Run against each provider in the registered set. The gateway is the SUT;
the provider is a parametrized backend.

Modes:
  * `replay`: deterministic, uses recorded fixtures. Runs in CI on every
    commit and gates merges.
  * `live-canary`: runs against live providers in a scheduled job
    (nightly) and on new-provider promotion. Required to merge a routing
    change that adds or promotes a provider.
"""
from __future__ import annotations

import json
import pytest

from gateway_contract import load_spec, normalise_error
from llm_gw import Gateway, Call

SPEC = load_spec("gateway_contract.yaml")

# Providers registered with the gateway, loaded from modality_affinity.yaml
PROVIDERS = pytest.fixture_params_from_registry()

@pytest.mark.parametrize("provider", PROVIDERS)
class TestChatContract:
    def test_basic_chat_shape(self, gateway: Gateway, provider: str):
        call = _basic_text_call(route="reply.compose", provider=provider)
        resp = gateway.chat(call)
        _assert_matches(resp, SPEC["chat"]["response"])
        assert resp["usage"]["prompt_tokens"] >= 1
        assert resp["usage"]["completion_tokens"] >= 1
        assert resp["model_served"] == call.model
        assert resp["provider_served"] == provider

    @pytest.mark.parametrize("tool_name", ["lookup_order", "open_crm_task"])
    def test_tool_call_envelope(self, gateway, provider, tool_name):
        call = _tool_call_setup(provider, tool_name)
        resp = gateway.chat(call)
        tc = _first_tool_call(resp)
        assert tc["name"] == tool_name
        # The strong assertion: arguments_json is JSON-valid at this
        # boundary. Providers that return function args as a non-string
        # or as invalid JSON must be normalised by the gateway.
        parsed = json.loads(tc["arguments_json"])
        assert isinstance(parsed, dict)

    def test_structured_output_parses(self, gateway, provider):
        schema = _declared_schema("reply_summary_v2")
        call = _structured_output_call(provider, schema)
        resp = gateway.chat(call)
        text = _first_text(resp)
        parsed = json.loads(text)
        _assert_matches(parsed, schema)

    @pytest.mark.timeout(15)
    def test_streaming_contract(self, gateway, provider):
        call = _basic_text_call(provider=provider, stream=True)
        events = list(gateway.chat_stream(call))
        _assert_event_order(events, SPEC["streaming"]["event_order"])
        _assert_seq_monotonic(events)
        assert events[-1]["type"] == "finish"
        assert events[-1]["finish_reason"] in {"stop", "tool_use", "length"}

    @pytest.mark.parametrize("err_class", SPEC["chat"]["response"]["error"]["class"])
    def test_error_class_mapped(self, gateway, provider, err_class):
        """For each declared error class, induce it and check the mapping.

        Inducers are provider-specific but test against the normalised
        shape; a provider that cannot induce a class (e.g. no region
        pin available) is marked with `pytest.skip` and the skip is
        reported in the promotion-readiness summary.
        """
        inducer = _error_inducer(provider, err_class)
        if inducer is None:
            pytest.skip(f"{provider} cannot induce {err_class}")
        with inducer():
            try:
                gateway.chat(_basic_text_call(provider=provider))
            except Exception as exc:
                mapped = normalise_error(exc)
                assert mapped["class"] == err_class
                assert isinstance(mapped["retryable"], bool)
```

**Modality tests** (abbreviated):

```python
@pytest.mark.parametrize("provider", VISION_PROVIDERS)
def test_vision_url_shape(gateway, provider):
    call = _vision_call(provider, image_ref={"type": "url", "url": _TEST_IMG_URL})
    resp = gateway.chat(call)
    _assert_matches(resp, SPEC["chat"]["response"])
    assert _first_text(resp).strip() != ""

@pytest.mark.parametrize("provider", VISION_PROVIDERS)
def test_vision_inline_shape(gateway, provider):
    call = _vision_call(provider, image_ref={"type": "inline", "mime": "image/png",
                                             "data_b64": _TEST_IMG_B64})
    resp = gateway.chat(call)
    _assert_matches(resp, SPEC["chat"]["response"])

@pytest.mark.parametrize("provider", ASR_PROVIDERS)
def test_audio_in_shape(gateway, provider):
    call = _audio_call(provider, audio_ref={"type": "object_ref",
                                            "uri": _TEST_AUDIO_URI,
                                            "mime": "audio/wav",
                                            "duration_s": 3.2})
    resp = gateway.chat(call)
    transcript = _first_text(resp)
    assert transcript.strip() != ""
```

**Version-pin and region-pin tests:**

```python
def test_version_pin_echoed(gateway, provider, pinned_model):
    call = _basic_text_call(provider=provider, model=pinned_model)
    resp = gateway.chat(call)
    # The gateway must refuse to serve a provider's silent snapshot upgrade.
    assert resp["model_served"] == pinned_model

def test_region_pin_served_from_region(gateway, provider, pin):
    if pin not in _supported_pins(provider):
        pytest.skip()
    call = _basic_text_call(provider=provider, region_pin=pin)
    resp = gateway.chat(call)
    assert resp["region_served"] == pin
```

### 2d. Replay fixtures and live-canary

**Replay mode.** Each provider has a `fixtures/<provider>/` directory of recorded exchanges (request → response). The replay runner matches by request hash. Replay is the mode that runs in every CI job; it is deterministic and does not consume provider quota.

- Fixtures are re-recorded on `pytest -m record` runs, which require a live-credential environment and a reviewer. A re-record is a PR.
- A test that cannot be served by a fixture fails CI; it does not fall through to live.

**Live-canary mode.** A scheduled job (nightly) and the "new provider promotion" workflow run against live providers. The canary suite is a superset of the merge suite in that it exercises the error-class inducers that fixtures cannot faithfully replay (rate-limit, region unavailable, actual timeout).

A provider enters production routing only after a live-canary run with zero contract failures and all declared modalities covered.

### 2e. Promotion and drift workflow (deliverable 4)

**Adding a new provider:**

1. Author a `providers/<name>.py` adapter and declare its modalities in `modality_affinity.yaml`.
2. Record replay fixtures for every applicable test.
3. Author error-class inducers for the classes this provider can produce.
4. Open a PR that includes: adapter + fixtures + inducer registrations.
5. CI runs the replay suite; must be green.
6. Live-canary is invoked by a reviewer; a report is attached to the PR.
7. Promotion to routing table ([[exercise-02]]) requires: green canary + mod-107 crosswalk entry + sign-off (platform + security).

**Provider version bump (model snapshot):**

1. The gateway treats `model: X@sha_old` and `model: X@sha_new` as distinct targets.
2. The new snapshot runs the full contract suite in replay first (fixtures re-recorded); any failure blocks.
3. mod-106 offline evaluation re-runs on the new snapshot; regression beyond the committed tolerance blocks.
4. Routing-table promotion is a PR that moves the pin; it cannot be a runtime edit.

**Contract-test failure in CI:**

- Any contract test failure blocks merge for the branch that changed the gateway or an adapter.
- A nightly canary failure opens a routing-table restriction: the failing provider is pulled out of primary slots for affected routes until the next green canary.
- A contract failure on a provider in production: the fallback chain from [[exercise-02]] routes around it; the mapping runbook (mod-107 [[exercise-05]]) is updated with the finding and the owner.

### 2f. Implementation notes

- **The SUT is the gateway, not the SDK.** The contract tests are not provider SDK tests. The gateway's normalisation is the thing under test; a passing test against the raw SDK but a failing gateway path is still a bug in production.
- **JSON validity of `arguments_json` is the single highest-value assertion.** Tool-call argument fields drift silently between providers (dict vs stringified JSON vs partial-JSON streaming fragments). The contract pins it as a JSON-valid string at the gateway boundary.
- **Inducers are test infrastructure.** An error-class without an inducer is a class that the fallback rules in [[exercise-02]] cannot be exercised against. Build the inducer; without it the fallback is a letter of intent.
- **Replay first, canary second.** Running every test against live providers in CI burns budget and misses the determinism you want in a merge gate. Replay for merges; canary for promotion and nightly.
- **Record fixtures through the gateway.** Fixtures capture gateway-normalised responses alongside raw provider responses, so a re-record also audits the normaliser.
- **Regions, pins, and residency are contract.** A `region_pin=eu` that a provider cannot honour should fail a contract test on this surface, not quietly land in `us-east`.
- **Streaming contract is where realtime bugs hide.** Out-of-order deltas or missing `finish` events escape unit tests and show up as user-visible stall. Monotonic `seq` is a cheap assertion with high value.
- **Determinism posture is a declaration, not a demand.** Providers rarely guarantee bit-identical outputs. The test asserts *structure* is identical; textual identity is only asserted when a provider's docs promise it.
- **Error inducer catalogue is versioned.** Providers change how they signal rate limits. The inducer registration must be reviewed like any other contract change.
- **Contract failures route to the mapping runbook.** A failed contract test is a mod-107 [[exercise-05]] entry — the runbook names which control depended on the shape that drifted.

## 3. Validation steps

1. The contract specification is a single committed file and does not name providers. If a provider name appears in the spec, the spec is wrong.
2. Each of T1–T10 dimensions has at least the minimum cardinality of tests, parameterised by provider.
3. Error-class inducers exist for every class in the spec, per provider; providers with gaps are marked `skip` with a reason and surfaced in the promotion-readiness summary.
4. Tool-call `arguments_json` is asserted JSON-valid at the gateway boundary across all tool tests.
5. Streaming tests assert event order and `seq` monotonicity, and assert a `finish` event always arrives (including cancellation path).
6. Modality tests exist for every modality declared in `modality_affinity.yaml` for each provider that claims it ([[exercise-04]]).
7. Version-pin test asserts `model_served` equals the pin (no silent upgrades). Region-pin test asserts `region_served`.
8. Replay fixtures are version-controlled; a `pytest -m record` run produces a reviewable PR.
9. Live-canary is defined and runs on both a schedule and on new-provider promotion. Promotion requires a green canary.
10. Contract failures are wired to the fallback rules in [[exercise-02]] and to the mapping runbook in mod-107 [[exercise-05]].

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Provider-neutral spec | 1 | No provider name in the YAML. |
| All 10 dimensions covered (T1–T10) | 1 | Each at minimum cardinality. |
| Tool-call JSON validity asserted | 1 | Across every registered tool. |
| Streaming event order + seq monotonic + finish always | 1 | Tested including cancellation. |
| Error classes mapped with inducers | 1 | Every class has at least one provider that induces it. |
| Modality tests per declared affinity | 1 | Vision URL + inline, audio, realtime smoke. |
| Version-pin and region-pin asserted | 1 | `model_served` / `region_served` fields pinned. |
| Replay fixtures reviewable; record workflow defined | 1 | Re-record is a PR. |
| Live-canary defined; gates promotion | 1 | Green canary required for routing promotion. |
| Wired to [[exercise-02]] fallback and mod-107 [[exercise-05]] runbook | 1 | A contract failure has a documented downstream effect. |

A submission scoring < 6 is rejected. Hard failures: (a) tests target the provider SDK instead of the gateway, (b) no error inducers (fallback untestable), (c) spec is provider-named, (d) no live-canary on promotion, (e) streaming contract missing.

## 5. Common mistakes

- **Testing the SDK, not the gateway.** The SDK is the vendor's responsibility; the gateway's normalisation is the surface's responsibility. Tests must speak to the gateway's API.
- **`arguments_json` as a dict.** The surface's turn-handler expects a string in the normalised envelope. A dict in one provider's response and a string in another is the exact failure mode contract tests exist for.
- **No streaming cancellation test.** Cancellation is the path that leaks sessions; a `finish` event missing on cancel is a slow leak.
- **No inducer for rate-limit.** `rate_limited` without an inducer means the fallback rule in [[exercise-02]] for `F2` is a comment, not a verified behaviour.
- **Replay-only.** A green replay suite with no canary is a suite that cannot tell when a live provider changes behaviour. Canary catches drift.
- **Canary-only.** A canary-only suite consumes quota on every commit, is flaky, and discourages running tests before push. Replay first.
- **Provider name in the spec.** `response.openai_choice.content` is a spec that forgot it has multiple backends. The spec names fields, not providers.
- **Not re-running on version bump.** A provider snapshot upgrade is a new target. The suite runs again or the pin does not move.
- **No link to the routing table.** A failing contract test on a provider still in `primary` for a route is a known regression the routing table should route around. The link from contract failure → routing restriction is the operational payoff of the whole exercise.
- **Treating "not applicable" as a free pass.** A provider that cannot induce a given error class is a provider with less observable behaviour; the skip is tracked in the promotion-readiness summary, not silently accepted.
- **No modality fixtures.** Vision and audio tests use representative media that must be scrubbed, licensed, and committed. "We'll test it once in prod" is a risk the exercise exists to remove.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML06 (AI Supply Chain) is the item this exercise most directly operationalises: contract tests on every provider, with a replay + canary cadence, convert "we trust our provider" into a verifiable claim; the version-pin and region-pin assertions address supply-chain drift. ML09 (Output Integrity) applies to the structured-output and tool-call tests, which enforce that a provider's output matches the declared shape before it reaches surface handlers.
- MITRE ATLAS — <https://atlas.mitre.org/> — the error-class inducer catalogue is the artefact an ATLAS-informed review relies on to test the surface's response to attacker-caused and provider-caused unavailability. Streaming cancellation and tool-call normalisation are both exfiltration-adjacent tests: a malformed tool call that bypassed the gateway's normalisation would be the Collection-tactic entry point. Contract tests prevent that bypass.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the promotion workflow (PR + canary + mod-106 regression check + sign-off) implements GOVERN (documented decisions, named accountability) and MANAGE (change gated on evidence). The contract suite itself implements MEASURE at the provider-boundary: continuous, pre-committed, documented.
- vLLM project documentation — <https://docs.vllm.ai/> — the vLLM OpenAI-compatible server contract is one of the backends under test; the self-hosted route from [[exercise-03]] is promoted via the same canary workflow, so the baseline there is not just a performance measurement but also an accumulation of replay fixtures that enter this suite.

Related exercises: [[exercise-01]] (the gateway is the SUT; the spec is the gateway's public contract); [[exercise-02]] (error-class inducers produce the signals the fallback rules route against; contract failures restrict primary-slot assignment); [[exercise-03]] (the vLLM endpoint is promoted via this suite before serving production traffic); [[exercise-04]] (modality tests cover every modality declared in the affinity map); mod-106 (offline evaluation runs alongside contract tests on provider version bumps); mod-107 [[exercise-05]] (contract failures are logged against the mapping runbook; drift is a mapping event); mod-110 (per-provider drift and canary results feed the dashboards); mod-111 (the deployment workflow that requires green contract + green canary before a routing-table change ships).
