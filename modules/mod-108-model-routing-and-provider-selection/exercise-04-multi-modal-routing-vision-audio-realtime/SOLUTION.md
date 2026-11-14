### exercise-04: Multi-Modal Routing — Vision, Audio, Realtime — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to extend routing on the helpdesk-reply-copilot surface beyond text-in / text-out. The three modalities named in the title are **vision** (an image is part of the request — a screenshot, a product photo, a scanned receipt), **audio** (speech-to-text on an uploaded voicemail; text-to-speech on a reply), and **realtime** (low-latency bidirectional speech or screen-share sessions). The deliverable is a routing extension that keeps the shape of [[exercise-01]] and [[exercise-02]] while answering the questions those extensions do not answer: which provider serves which modality, how the request/response shape differs, where fallback becomes degrade, and what the guardrails from mod-107 look like when the content is not text.

The failure mode the exercise is written to catch is "vision is just an extra field." It is not. A modality change is a capability change, a latency class change, a cost class change, a safety-tooling change, and sometimes a residency change. The surface that bolts a vision feature onto an existing text route tends to ship with no modality-aware fallback, moderation that cannot score an image, and an SLO budget that was written when the only input was 300 tokens of ticket text.

This solution provides:

- A **modality model** — what the surface's modalities are, what the per-modality capability tiers are, and what the per-modality fallback / degrade rules look like.
- A **routing extension** — how [[exercise-02]]'s routing table grows to cover modalities, including explicit cross-modality fallbacks (e.g. vision → "transcribe and treat as text").
- A **realtime-session design** — session lifecycle, idempotency, barge-in, and when the realtime path is not the right answer.
- A **guardrails-at-modality** map — which mod-107 controls run on which modality and which have to be reconsidered.
- A **grading rubric** for the instructor.

The lesson the exercise defends is: *routing by modality is a different policy, not a bigger table.* The variables change, the invariants change (moderation, residency, scrub), and the degrade paths are modality-specific.

## 2. Worked answer or implementation

### 2a. Modality model (deliverable 1)

For each modality the surface supports, name what the modality means **on this surface**, not in the abstract.

| Modality | On this surface | Request shape | Response shape |
|---|---|---|---|
| `text` | ticket body + draft reply | messages[] text parts | message text |
| `vision-in` | screenshot or photo attached to a ticket | messages[] with image parts (URL or base64) | message text (grounded to the image) |
| `audio-in` | uploaded voicemail; CS agent recording | audio file (WAV / Opus) in object storage reference | transcript + optional structured summary |
| `audio-out` | TTS of the final reply, for accessibility | text | audio stream + duration metadata |
| `realtime` | live voice call with a human agent, assisted by the model | bidirectional WS; audio frames in, audio/text frames out | partial transcripts + model turns + end-of-session summary |

**Capabilities and tiers.**

| Modality | Cheap tier | Mid tier | Flagship tier |
|---|---|---|---|
| `vision-in` | small multi-modal classifier (does the image contain X) | multi-modal chat model (describe, extract) | multi-modal chat model (reason over image + text, cite evidence) |
| `audio-in` | ASR with word timestamps, generic language model | ASR with diarisation + language ID | high-accuracy ASR with domain lexicon, long-form support |
| `audio-out` | neural TTS, single voice | neural TTS, voice choice + prosody controls | high-fidelity TTS (surface may not need this) |
| `realtime` | n/a — realtime below a quality floor is worse than async | realtime API with partial transcripts | realtime API with tool use, barge-in, server VAD |

A tier that is empty ("n/a") is a legitimate answer. A tiered modality the surface does not need yet gets a `not supported` row, not a stub.

### 2b. Routing-table extension (deliverable 2)

The routing table from [[exercise-02]] grows two decision variables:

- `V2 = modality` (already present in 2a of [[exercise-02]], now used in anger).
- `V7 = modality-provider affinity` — new. Not every provider offers every modality; this binds first.

The `modality-provider affinity` table is a static map, versioned with the routing table:

```yaml
# modality_affinity.yaml — helpdesk-reply-copilot — v1.1
providers:
  provider-a:
    modalities: [text, vision-in]
  provider-b:
    modalities: [text, vision-in, audio-in, audio-out]
  provider-c:
    modalities: [text, audio-in]
  provider-rt-a:
    modalities: [realtime]
    note: "bidirectional voice; separate SLO, separate rotation"
  self-host-vllm-m:
    modalities: [text]
    note: "self-host text only in this baseline; see [[exercise-03]]"
```

The affinity map is the invariant the router consults before anything else. A fallback that disregards it is a routing bug.

Route extensions in `routing_table.yaml`:

```yaml
# excerpt — adds vision and audio routes
routes:
  ticket.ingest.vision:
    modality: vision-in
    rules:
      - when: "V3 == 'easy'"
        primary: {provider: provider-a, model: model-mm-s-2026-09, region: us-east}
        fallback:
          - {provider: provider-b, model: model-mm-s-2026-10, region: us-east}
          - transform: "image_to_text"       # cross-modality fallback (see 2c)
            then: {provider: provider-a, model: model-m-2026-09, region: us-east}
        degrade_mode: allow_text_only_reply
        rationale_tag: "vision:easy"
      - when: "V3 == 'hard'"
        primary: {provider: provider-a, model: model-mm-l-2026-09, region: us-east}
        fallback:
          - {provider: provider-b, model: model-mm-l-2026-10, region: us-east}
        degrade_mode: refuse_and_hitl
        rationale_tag: "vision:hard"
  ticket.ingest.audio:
    modality: audio-in
    rules:
      - when: "always"
        primary: {provider: provider-c, model: asr-m-2026-09, region: us-east}
        fallback:
          - {provider: provider-b, model: asr-m-2026-10, region: us-east}
        degrade_mode: refuse_with_upload_affordance
        rationale_tag: "asr:default"
  reply.compose.tts:
    modality: audio-out
    rules:
      - when: "always"
        primary: {provider: provider-b, model: tts-s-2026-09, region: us-east}
        fallback: []
        degrade_mode: skip_tts_return_text
        rationale_tag: "tts:default"
  support.call.live:
    modality: realtime
    rules:
      - when: "always"
        primary: {provider: provider-rt-a, model: rt-m-2026-09, region: us-east}
        fallback: []
        degrade_mode: fail_to_async_handoff
        rationale_tag: "rt:default"
```

The novelty vs [[exercise-02]] is two: **cross-modality fallback** (an explicit `transform` step) and **degrade modes that are modality-specific** (`allow_text_only_reply`, `skip_tts_return_text`, `fail_to_async_handoff`).

### 2c. Cross-modality fallback (deliverable 3)

Fallback within a modality is the first move. When no provider in the affinity map is available for the modality, the router may fall across modalities — but only in a direction the surface has decided is correct.

**Allowed cross-modality fallbacks on this surface:**

- `vision-in → text`: via `image_to_text` transform (either OCR for scans / receipts, or a short caption from a smaller multi-modal classifier). The caller is a text model with the transformed text in place of the image reference. The reply must note the image was not seen by the responding model.
- `audio-in → text`: via a cheaper ASR when the primary is unavailable. The downstream text route is the fallback destination.
- `realtime → async`: not a graceful degrade — the session ends and the user is told to leave a voicemail. The surface is honest about this.

**Fallback moves that are not allowed:**

- `text → vision-in` ("ask the user for an image") — the router does not change the user's input. Product does.
- `audio-out → text` silently — if TTS fails, the surface returns the text with a visible note; it does not pretend the audio existed.
- `vision-in → refusal` to mask a provider outage — the user would blame the content, not the outage.

The router emits a distinct rationale tag on cross-modality fallback (`xmod:vision_to_text`) so [[exercise-02]]'s per-rationale report sees it as a separate class.

### 2d. Realtime session design (deliverable 4)

Realtime has a different lifecycle from one-shot. The design is minimal but concrete:

```
session.connect
  └─ auth + tenant + region check
  └─ guardrails bind (moderation, injection defence, scrub) to the session, not per-turn
turn.in_audio   ─┐
                ├─> server VAD → partial transcript → model turn (may call tools)
turn.out_audio  ─┘                                               └─> compensator mounted
barge_in → model turn is cancelled; current audio out is stopped; VAD resumes
session.heartbeat (every N seconds; cost accounting)
session.close
  └─ final transcript + summary + receipts
```

**Idempotency.** A session identifier is minted on connect and carried on every tool call. Any tool invoked from the session is called with the session ID so a replay after a disconnect is distinguishable from a fresh call.

**Guardrails.** Moderation runs on the running transcript, not just per-turn, because realtime inputs are often fragmented. A realtime session is scored under the same mod-107 rules, with a different cadence (streaming windows, not request/response units).

**When realtime is the wrong answer.** If the surface has an available async path that gives acceptable UX, realtime should be the exception. Realtime adds correlated failure (network drops cascade to session loss), a separate on-call rotation, and a cost class with no retry economics. The decision note from [[exercise-03]] is written per route; a route whose UX does not benefit from < 500 ms turn-around is better served by the async path.

### 2e. Guardrails-at-modality map (deliverable 5)

How the mod-107 controls move when the content is not text:

| mod-107 control | `vision-in` | `audio-in` | `audio-out` | `realtime` |
|---|---|---|---|---|
| H107-C01 moderation | image classifier + OCR → text moderator | ASR → text moderator; moderation on the transcript, not the waveform | TTS output is ours; moderate the text that feeds it | streaming moderation against the rolling transcript |
| H107-C03 injection defence | image text / QR codes / alt text can carry injections — treat as T3 content | audio with embedded instructions ("go to this URL and read it aloud") — treat as T3 | n/a (we author the text) | treat live user speech as T3 if it can be echoed to tools |
| H107-C06 trace scrub | avoid storing the raw image; store a reference and a derived caption | avoid storing the raw audio beyond retention policy; store the transcript scrubbed | TTS audio is a derivative of the reply; retention per policy | sessions are high-volume; retention and sampling decisions are explicit |
| H107-C08 pre-call guards | rate guard per image count; max-size guard; MIME guard | rate guard per minute of audio; max-length guard | per-reply emission cap; concurrent-TTS cap | per-tenant session cap; per-session duration cap |
| H107-C09 destructive-action gate | image content can request a destructive tool via visible text — gate in full | audio transcript can request destructive tool — gate on the transcript | n/a | realtime tool use is the hottest surface for this — gate in full and HITL |
| H107-C11 memory policy | no raw image stored in memory | no raw audio stored in memory | n/a | session-scoped memory only; nothing crosses the session boundary |
| H107-C13 kill switch | per-modality flag so a single modality can be disabled | per-modality flag | per-modality flag | per-route flag; disables the session entry |

This map is not new policy; it is the **statement of where the existing policy applies**. The mapping runbook (mod-107 [[exercise-05]]) is updated to add the modality column when this exercise lands.

### 2f. Reference implementation sketch

Mounted at the thin gateway from [[exercise-01]]:

```python
"""modality_router.py — modality-aware routing.

Extends the router from [[exercise-02]] with:
  * affinity check against `modality_affinity.yaml`
  * cross-modality transforms (vision -> text etc.)
  * per-modality degrade modes
"""
from __future__ import annotations
from dataclasses import dataclass
from typing import Callable

@dataclass(frozen=True)
class ModalityCall:
    route: str
    modality: str                     # "text" | "vision-in" | "audio-in" | "audio-out" | "realtime"
    payload_ref: str                  # object-storage reference where applicable
    tenant_id: str
    region_pin: str | None
    request_id: str

Transform = Callable[[ModalityCall], ModalityCall]

@dataclass
class ModalityRouter:
    base_router: object                        # the router from [[exercise-02]]
    affinity: dict[str, list[str]]             # provider -> [modalities]
    transforms: dict[str, Transform]           # "image_to_text" -> fn

    def decide(self, call: ModalityCall, env: dict):
        # 1. Affinity narrow: drop targets whose provider does not serve the modality.
        decision = self.base_router.decide(call, env)
        decision = self._filter_by_affinity(decision, call.modality)
        # 2. Cross-modality fallback (if the rule asks for a transform step).
        decision = self._expand_transforms(decision, call)
        return decision
```

The rationale tag survives into the gateway trace; the per-rationale cost + latency report in [[exercise-02]] now has rows for `vision:easy`, `asr:default`, `tts:default`, `rt:default`, `xmod:vision_to_text`, so the surface can see what each modality costs and where it fails.

### 2g. Implementation notes

- **Affinity before anything else.** A route that lists a text-only model as a fallback for `vision-in` is a routing bug; affinity catches it at config-load time, not at request time.
- **Cross-modality fallback is lossy, and should be labelled.** The reply must note that the image was not seen, or the summary was produced from an inferior transcript. "Degrade silently" is a UX failure.
- **Realtime is a separate on-call.** The rotation and the observability for realtime are not the same as for the async path. Teams underestimate this.
- **Audio is cheap to send, expensive to store.** Retention policies on `audio-in` are a data-class decision; the trace scrub from mod-107 [[exercise-03]] is the enforcement point.
- **Prompt injection via OCR and QR codes is real.** Image content is adversarial content. Treat any text extracted from an image under the same T3 provenance rules as web-fetched text.
- **Realtime tool-use is the hottest destructive-action surface.** A barge-in that interrupts a tool call mid-flight must behave deterministically; design for the case where the user says "cancel that" after the tool has already executed.
- **Model-version pinning applies per modality.** A vision model's snapshot upgrade can regress differently from the text model's upgrade; the version-pin alert (`R14` from [[exercise-01]]) is per modality.
- **Contract tests cover modality shapes.** [[exercise-05]] contract tests include a `vision-in` message with URL and base64 variants, an `audio-in` shape, and a realtime handshake smoke test. Modality shapes drift silently without them.
- **Not every modality needs a tier for every route.** The empty cells in the capability table are decisions. Pretending they are "to be filled" grows the system invisibly.

## 3. Validation steps

1. The modality model lists each modality the surface supports, with request / response shape and cheap/mid/flagship tier choices. A modality not in the table is not served.
2. The provider-modality affinity map is a committed artefact; the router consults it before any other rule.
3. The routing table has at least one route per supported modality, with a rationale tag and a degrade mode. No `fallback: []` is acceptable only when the route's product shape allows total unavailability with an affordance.
4. At least one cross-modality fallback is defined with an explicit transform step, a user-visible note on the degraded reply, and a distinct rationale tag.
5. The realtime design covers session lifecycle, idempotency, barge-in, and an explicit "when realtime is wrong" line.
6. The guardrails-at-modality map covers the mod-107 controls on each modality; blanks say "n/a" with a reason.
7. The surface's trace schema has per-modality fields (image-count, audio-duration, session-duration) so cost accounting is modality-aware.
8. Contract tests include a vision shape, an audio shape, and a realtime smoke test ([[exercise-05]]). A modality not tested is a modality not supported.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Modality model with request/response shapes and tiers | 1 | Empty tiers labelled `not supported`. |
| Provider-modality affinity map is versioned | 1 | Separate file from routing table. |
| Routes per modality with rationale tag + degrade mode | 1 | Degrade modes are modality-specific. |
| Cross-modality fallback with transform + visible note | 1 | ≥ 1 allowed cross-modality path documented. |
| Non-allowed cross-modality paths are called out | 1 | Explicit list (text→vision, silent tts drop, etc.). |
| Realtime design: lifecycle, idempotency, barge-in | 1 | All three covered. |
| Explicit "realtime is wrong when…" | 1 | A line named; not every route is realtime. |
| Guardrails-at-modality map | 1 | Table includes all mod-107 controls. |
| Trace schema is modality-aware | 1 | Per-modality cost signals emitted. |
| Contract tests include all supported modalities | 1 | Referenced from [[exercise-05]]. |

A submission scoring < 6 is rejected. Hard failures: (a) no affinity check (router may send vision to a text-only provider), (b) cross-modality fallback silent to user, (c) realtime without a session design, (d) guardrails map omits moderation on image / audio.

## 5. Common mistakes

- **Bolt-on vision.** Adding an image field to the existing text route is the one-week version that ships without moderation, without a sized context budget, and without a fallback. The route is a new route; the table has a new row.
- **Silent cross-modality degrade.** Vision provider is down, so the pipeline falls to a text-only model with an OCR'd description, and the reply reads as if the model saw the picture. The user takes the reply literally. Visible notes are mandatory.
- **Realtime as the default.** Realtime UX is magic; realtime infrastructure is a correlated failure ship. Default to async; make realtime a per-route decision.
- **Moderating the waveform.** Moderators do not (today) read audio directly on this surface. The ASR transcript is the moderated artefact; this means ASR is a safety-critical component.
- **Storing raw images for debugging.** Raw images in logs cross the privacy policy in a way the scrubber cannot remediate. Store derived captions or references; delete raws by retention policy.
- **Treating image text as trusted.** OCR extracted from a user-uploaded image is T3 content under the injection-defence model. A phone number in an image is not more trustworthy than a phone number in the ticket body.
- **Barge-in races with tools.** A user says "cancel that" after a destructive tool has executed. The session design must say what happens — the code cannot discover it at 2 am.
- **One SLO budget for all modalities.** Vision requests are latency-heavier and cost-heavier than text; a single budget hides this. Per-modality SLOs live in mod-109.
- **Letting a modality live outside the gateway.** A direct SDK call to a realtime provider that bypasses the gateway bypasses tracing, scrub, moderation, and cost accounting. The gateway covers realtime too.
- **No contract tests for vision / audio shapes.** Provider shape drifts are easy to miss; a URL-variant that worked yesterday returns 400 today. The contract test pins it.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — ML01 (Input Manipulation) applies per modality: adversarial images, audio with embedded instructions, and QR-code or EXIF channels are first-class ML01 vectors on this surface. The guardrails-at-modality map operationalises ML01 responses per modality, and ML06 (Supply Chain) applies to modality-specific provider pins the same way it applies to text providers.
- MITRE ATLAS — <https://atlas.mitre.org/> — ATLAS tactics that this exercise newly engages: Initial Access via indirect injection carried by image text or audio prompts; Collection / Exfiltration via realtime channels (a live session is a new egress surface); Impact via barge-in races that interrupt tool calls. The per-modality kill switch (H107-C13) and the session-scoped memory (H107-C11) are the defensive controls ATLAS review will ask for.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the modality model and affinity map implement MAP (context, intended use per modality). The per-modality cost signals and tier measurements implement MEASURE. The realtime session design (lifecycle, kill switch, async fallback) implements MANAGE. The sign-off path for a new modality (route owner + platform + security + product) implements GOVERN.
- vLLM project documentation — <https://docs.vllm.ai/> — relevant because the self-host decision from [[exercise-03]] is modality-constrained: the vLLM baseline described there may not cover vision or audio for this surface; that absence appears directly in the modality-affinity map (`self-host-vllm-m: [text]`), making the dependency explicit rather than implied.

Related exercises: [[exercise-01]] (the gateway's R15 — multi-modal adapters — are mounted here); [[exercise-02]] (routing-table extension; rationale tags flow into the per-rationale report); [[exercise-03]] (self-host is modality-constrained and those constraints appear in the affinity map); [[exercise-05]] (contract tests pin the per-modality request / response shapes); mod-107 [[exercise-01]] / [[exercise-02]] / [[exercise-03]] / [[exercise-04]] / [[exercise-05]] (moderation, injection defence, scrub, tool gate, and mapping runbook extended per modality); mod-109 (per-modality SLOs); mod-110 (per-modality incident and drift dashboards); mod-111 (shipping a new modality without destabilising existing routes).
