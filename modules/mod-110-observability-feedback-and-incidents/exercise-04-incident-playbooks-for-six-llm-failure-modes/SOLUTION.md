### exercise-04: Incident Playbooks for the Six LLM Failure Modes — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to produce **six concrete incident playbooks**, one per LLM failure mode, each written in the second person for an on-call at 3 AM. Each playbook names its detection signal (a query against the span contract from [[exercise-01]]), its containment lever, its investigation steps, its communication script, its remediation path, and the postmortem trigger threshold. The deliverables are the six docs plus an index. They are not alerts (those live with the SLO in mod-109 [[exercise-05]]) and not root-cause analyses (those are the postmortem's job).

The failure modes covered are the six that show up most often in production LLM features and that each require a distinct response:

1. **Hallucination outbreak** — factual drift on a concrete surface claim.
2. **Prompt-injection breakthrough** — a user input steers the model past guardrails.
3. **Tool misuse** — the model calls a tool with malformed arguments or the wrong tool at scale.
4. **Latency SLO burn** — the mod-109 [[exercise-05]] fast-burn alert fired.
5. **Cost spike** — mod-109 [[exercise-04]] CPSO breaches the band or absolute spend crosses the alarm threshold.
6. **PII / sensitive-data leak** — the scrubber in [[exercise-02]] fired a rule that should not have been needed in-model, or a prompt or completion escaped redaction.

The lesson the exercise defends: *an LLM incident is not an API-outage incident*. A traditional runbook ("page the service owner, restart the pod, roll back the deploy") assumes the failure is deterministic and binary. LLM failures are distributional, often user-input-triggered, and often invisible to health checks. The playbook structure below — detection signal on the span stream, named containment lever, written communication — is what makes the response legible to a human whose instinct is "restart the pod."

## 2. Worked answer or implementation

### 2a. Playbook structure (deliverable 0)

Every playbook has the same seven-section shape so a reader does not relearn where to look across an incident:

```markdown
# PB-<id>: <failure mode name>

## 1. What this looks like
## 2. Detection signal (query)
## 3. Containment (first 15 minutes)
## 4. Investigation (first 2 hours)
## 5. Communication
## 6. Remediation
## 7. Postmortem trigger
```

The common shape also lets the status page in [[exercise-05]] pick a playbook by signal and render the first 15-minute section inline next to the alert.

### 2b. PB-01 — Hallucination outbreak

```markdown
# PB-01: Hallucination outbreak

## 1. What this looks like
Users report the model stated a wrong fact that affects downstream action
(a wrong price, a wrong customer id, a wrong policy citation). The surface
has no API error; the completion is well-formed and high-confidence.
Signal is almost always in the feedback stream from [[exercise-03]] before
it is in the span stream.

## 2. Detection signal (query)
Primary: 7-day Argilla confirmed-hallucination rate > 2× control band for
the feature.
Secondary (span-side): elevated rate of `app.outcome="success"` paired with
`correction_submitted` feedback on the same `request_id`.
Automated: an Argilla-ingest side job raises a ticket when confirmed
hallucination for a feature in the last 24h exceeds the 12-week band.

## 3. Containment (first 15 minutes)
- Confirm the pattern is real: open `/d/feedback-helpdesk-reply` and check
  the confirmed-hallucination panel. If the spike is driven by a single
  prompt template or a single tenant, contain by scope — do not degrade
  the whole feature.
- If the surface claims specific IDs (ticket ids, order ids, customer
  ids), **enable the ID-validation guardrail** from mod-107 for this
  surface:
    - config: `guardrails.id_validator.enabled: true`
    - expected effect: calls containing an unresolvable id are rewritten
      to "I don't have that record" rather than proceeding with the
      fabricated id.
- If the feature has a retrieval step (mod-103), **tighten `top_k`** or
  enable a citation-required post-filter. This is a capability loss, not a
  full disable.

## 4. Investigation (first 2 hours)
1. Pull the 50 most recent confirmed-hallucination feedback rows from
   Argilla. Group by `failure_mode == "hallucination"` and by served model
   (`gen_ai.response.model`).
2. Is the pattern correlated with a specific `gen_ai.response.model`?
   A silent fallback to a less-grounded model (mod-108) is a common cause.
3. Is it correlated with a recent prompt-template deploy? Cross-check the
   release markers in the status page.
4. Is it correlated with a retrieval index refresh? A stale index with a
   partial document rebuild matches this signature.

## 5. Communication
Internal (first 30 min):
   Channel: #incident-live
   Subject: "Hallucination outbreak on <feature> — containment in progress"
   Fields: scope (feature + tenants affected), containment taken, next
   action, ETA for next update (30 min default).
External (customer-facing):
   Customer communications are PM-authored and legal-reviewed. Draft a
   two-paragraph note (what is affected, what we are doing); do not send
   until PM + legal have signed. Status page component ([[exercise-05]])
   goes to "degraded" when containment is in place.

## 6. Remediation
- If model fallback is the cause, un-do the fallback (mod-108) and keep
  primary provider enforced until a root-cause review.
- If prompt-template deploy is the cause, roll back and reproduce in staging.
- If retrieval is the cause, re-run the index build and verify the
  mod-106 eval pack covers the missing docs; a confirmed row from
  [[exercise-03]] should become an eval case before merge.

## 7. Postmortem trigger
- Any confirmed-hallucination incident that touched a payment, legal, or
  safety surface.
- Any incident where > 100 end users were materially misinformed (defined
  as a customer-support ticket opened from the thumbs-down).
```

### 2c. PB-02 — Prompt-injection breakthrough

```markdown
# PB-02: Prompt-injection breakthrough

## 1. What this looks like
A user input causes the model to exfiltrate system-prompt content, follow
attacker-supplied instructions, call a tool it should not, or ignore
guardrail rules. Often surfaces as "the model refused policy X on turn 2"
or as an unusual tool call on a session with a tool never otherwise used.

## 2. Detection signal (query)
Primary: elevated rate of `gen_ai.response.finish_reason="content_filter"`
on a *single* `openinference.user.id` or `app.tenant_id`.
Secondary: unusual `app.tool.name` distribution (tools with < 0.1% daily
calls suddenly > 5%).
Argilla side: `signal=report_safety` cluster from multiple users against
the same surface within a day.
Mod-107 side: guardrail audit log entries show an unusual pattern of
rejected-then-retry traffic.

## 3. Containment (first 15 minutes)
- Identify the suspected inputs. Pull the 20 most recent spans that match
  the signal (hashed `openinference.user.id`, feature). Preserve the
  prompt / completion span events before anything purges.
- **Enable a per-user rate limit** on the suspect hashed ids: the limit is
  configured on the gateway (mod-108 exercise-01 substrate), not on the
  feature.
- If the injection targets a specific tool, **disable the tool for this
  feature** (`TOOLS_ENABLED` config) until the injection pattern is
  understood. Capability loss is documented in the comms.

## 4. Investigation (first 2 hours)
1. Reproduce the attack in staging with the captured prompt. Confirm the
   guardrail miss.
2. Classify against MITRE ATLAS: this is Evasion / Initial Access /
   Privilege Escalation depending on effect. Record the ATLAS tactic id
   in the incident doc.
3. Check whether the input shape is covered by the mod-107 guardrail
   suite. A miss is a guardrail test gap; a hit that still breached is a
   guardrail-strength problem.
4. Pull the feedback rows from [[exercise-03]] that overlap in time;
   confirm the user-facing signal and update their severity.

## 5. Communication
Internal (first 15 min):
   Channel: #incident-live, cross-posted to #security
   Fields: scope, suspect pattern (redacted), containment, security
   involvement y/n.
External: security may require coordinated disclosure. PM + security +
legal gate the customer-facing wording. Do not post the injection payload
to a public channel.

## 6. Remediation
- Add the captured prompt to the mod-107 guardrail test suite as a
  regression.
- Tighten the system prompt or add a pre-filter; mod-107 chapter 3
  substrate.
- If a tool was involved, add schema-side validation to the tool call
  path.

## 7. Postmortem trigger
- Any confirmed breach that caused unauthorised tool invocation.
- Any breach that exfiltrated system-prompt content or prior-turn
  context from another tenant.
- Any pattern that was known to the mod-107 suite but not caught.
```

### 2d. PB-03 — Tool misuse

```markdown
# PB-03: Tool misuse

## 1. What this looks like
The model calls a tool with malformed arguments (JSON schema violation),
calls a tool it should not (wrong tool for the task), or calls a tool
with the right shape but values that violate business logic (negative
quantity, impossible id). Most commonly fires as an elevated
`app.structural_validity="fail"` rate on tool-call spans.

## 2. Detection signal (query)
Primary: for spans where `gen_ai.response.finish_reason="tool_calls"`:
  `rate(app.structural_validity="fail") > 2 × 7d baseline` for 10 min.
Secondary: tool child spans whose `app.outcome="error"` rate > 2 × 7d
baseline for 30 min.
Business-logic side: mod-105 agent-level audit log for a surge of
"tool executed but transaction rolled back" events.

## 3. Containment (first 15 minutes)
- If the signal is structural: the gateway (mod-108 exercise-01) already
  rejects malformed calls. Confirm the rejection rate on the gateway's
  own dashboard; if the model is retrying blindly, the retry budget on
  the gateway blows up — cap it at 1 for tool_calls.
- If the signal is wrong-tool-at-scale: **narrow the tool allowlist for
  this feature** (`TOOLS_ENABLED` in config). Capability loss documented.
- Capture at least 20 example tool-call spans with their arguments for
  the investigation.

## 4. Investigation (first 2 hours)
1. Group failing tool calls by tool name and by served model. A specific
   model-and-tool pair is the common cause after a model swap.
2. Check whether the tool's JSON schema recently changed. A schema that
   drifted from the model's training distribution produces structural
   failures within hours.
3. Pull the feedback rows with `failure_mode="wrong-tool-call"` from
   [[exercise-03]] and compare against the automated failures.

## 5. Communication
Internal: #incident-live, 15-min cadence. Scope = features touching the
tool.
External: if the tool is user-facing (e.g. "book a meeting", "create a
ticket"), the status page component goes to "degraded" when capability
loss is active. PM drafts the customer-facing wording.

## 6. Remediation
- If a schema drift, pin the tool schema at the LLM-facing boundary and
  version it; the LLM sees v1 while the backend accepts v2.
- If a model swap, roll back or constrain the tool-call shape in the
  system prompt.
- Add the failing call shape to the mod-104 tool-call regression suite.

## 7. Postmortem trigger
- Any tool-misuse incident that touched a write path (billing, inventory,
  identity).
- Any incident with > 1% affected sessions in a 1-hour window.
```

### 2e. PB-04 — Latency SLO burn

```markdown
# PB-04: Latency SLO burn

## 1. What this looks like
The fast-burn alert from mod-109 [[exercise-05]] fired. The surface may
still *work*; it works *slow enough that users notice*.

## 2. Detection signal (query)
Primary: `HelpdeskReplySLOFastBurn` or `HelpdeskReplySLOSlowBurn` alert
from the mod-109 alert rules.
Secondary: `app.latency.ttft_ms` p95 on the per-feature latency panel
above the red boundary for > 5 min.
Correlated: `app.retry.attempt_number > 1` rate climbing.

## 3. Containment (first 15 minutes)
This playbook defers to the mod-109 [[exercise-05]] runbook for the
specific lever sequence (retry → hedge → fallback → degrade). Three
local-to-this-playbook additions:
- Confirm the lever chosen matches the burn window. Fast-burn favours
  hedge / fallback; slow-burn favours degrade.
- If a provider swap is used (mod-108 exercise-02 substrate), write the
  swap in the incident doc so the fallback-cost cost delta is attributable
  in the ledger from mod-109 [[exercise-04]].

## 4. Investigation (first 2 hours)
1. Is TTFT, body throughput (TPOT), or retry the dominant contributor?
   Open the latency panel from mod-109 [[exercise-01]].
2. Which provider is tailing out? Group p95 by `gen_ai.response.model`.
3. Cross-check with the provider's status page. If it is a known
   provider incident, document the provider's incident id in the doc.

## 5. Communication
Internal: #incident-live, 15-min cadence. Primary channel for latency
incidents is also #platform-sre.
External: status page component to "degraded" during fast-burn, "under
investigation" during slow-burn. PM reviews the customer note; latency
alone usually does not need a legal review.

## 6. Remediation
- Keep the chosen lever engaged through the burn window plus 30 min
  after recovery.
- Run a mod-109 [[exercise-05]] post-review of the SLO itself. Repeated
  burns on the same feature are a signal the SLO is set too tight or the
  provider mix is wrong.

## 7. Postmortem trigger
- Any fast-burn that exhausted > 25% of the quarter's error budget.
- Any slow-burn that persisted > 48 hours without recovery.
```

### 2f. PB-05 — Cost spike

```markdown
# PB-05: Cost spike

## 1. What this looks like
Daily spend on `app.cost.usd` for a feature or tenant exceeded the alarm
threshold from mod-109 [[exercise-04]]. Often correlated with a loop in
a tool-calling agent, a change in prompt that blew up input tokens, or a
single tenant whose integration retries at 10× the median.

## 2. Detection signal (query)
Primary: mod-109 [[exercise-04]] daily-spend alarm:
  `sum_24h(app.cost.usd) by (app.feature) > 2 × 7d median AND > $<budget-floor>`
Secondary: per-tenant: same, grouped by `app.tenant_id` hash. A single
tenant with > 5× median spend overnight is usually a loop.
Tertiary: `app.retry.attempt_number > 1` rate climbing in parallel.

## 3. Containment (first 15 minutes)
- Identify scope: feature-wide, tenant-wide, or route-wide. The Cost per
  Successful Outcome (CPSO) panel groups by all three.
- If a single tenant: **apply the per-tenant rate limit** on the gateway
  (mod-108). The ledger notes the lost revenue for the hour.
- If a single route: **shrink `max_tokens`** on the route to the 95th
  percentile of healthy usage, or flip the route to a smaller model
  (mod-108 exercise-02 substrate).
- If a retry loop: cap `retry_attempts` to 1 for the route; the gateway
  change propagates in < 2 min.

## 4. Investigation (first 2 hours)
1. Which component of CPSO blew up: input tokens, output tokens, retries,
   or outcome rate? The CPSO panel has the breakdown.
2. If output tokens: a prompt change that removed a length constraint is
   the common cause.
3. If retries: a provider's tail latency plus a too-aggressive retry
   budget is the common cause; cross-check with PB-04.
4. If a tenant: inspect their request shape on the trace stream for
   obvious loops (same `openinference.session.id` every ~2 seconds for
   hours).

## 5. Communication
Internal: #incident-live, 30-min cadence. Finance is looped in when
spend exceeds the monthly budget alarm.
External: a per-tenant rate limit is customer-visible. CSM + PM own the
wording. Status page is usually not changed.

## 6. Remediation
- If a prompt change, roll back and add a `max_tokens` enforcement test
  to the CI suite.
- If an agent loop, add a per-session tool-call budget (mod-105
  substrate).
- If a tenant integration bug, raise a support ticket and lift the rate
  limit after the customer deploys their fix.

## 7. Postmortem trigger
- Any spike where unplanned spend exceeded the monthly budget alarm.
- Any spike caused by an internal deploy (vs. a tenant-side change).
```

### 2g. PB-06 — PII / sensitive-data leak

```markdown
# PB-06: PII / sensitive-data leak

## 1. What this looks like
The model's completion contained PII it should not have (another user's
data, an internal-only identifier, an unredacted account number). Can
also surface as a scrubber ([[exercise-02]]) rule firing at an elevated
rate on completions — a signal the model is producing PII in-generation,
not that the user supplied it.

## 2. Detection signal (query)
Primary: `scrub.rules_count > 0` rate on `gen_ai.content.completion`
events exceeds 2 × 7d baseline for 10 min.
Specific-rule signal: `scrub.rules_matched contains "ssn-us"` or
`"cc"` on a feature that should not be producing those categories.
Argilla side: `signal=report_safety` with `failure_mode=safety` and
reviewer-notes mentioning "other user's data."

## 3. Containment (first 15 minutes)
- Preserve evidence: snapshot the T0-retained spans (see [[exercise-02]])
  for the affected window to a secure store before any purge runs.
- **Flip the feature to safe-mode**: a mode in which the system prompt
  requires the model to refuse to emit any identifier matching the
  scrubber patterns, AND the gateway drops any completion that still
  matches after refusal. Capability loss is documented.
- If the leak looks like cross-tenant data, **disable retrieval** for
  the feature until the retrieval scope is audited.

## 4. Investigation (first 2 hours)
1. Classify: did the PII come from the user's prompt, from retrieval,
   from tool output, or from the model's parametric memory? The
   gen_ai.content.prompt event tells you the first; the retrieval span
   tells you the second; the tool span tells you the third.
2. If from retrieval: audit the retrieval scope — is a user's query
   fetching another user's documents? Cross-check mod-103 tenant-scope
   tests.
3. If from the model's memory: this is the hardest case. A model that
   emits PII it was never shown is a training-data problem; log the
   evidence and escalate to security.
4. Pull all feedback rows with `failure_mode="safety"` in the window
   and confirm whether this is one incident or several.

## 5. Communication
Internal: #incident-live AND #security AND #privacy as first action,
not after containment. Legal joins by default for any incident with a
confirmed cross-tenant leak.
External: customer notifications are legal-authored. Do not publish
scoped details until counsel signs. The status page component goes to
"degraded" or "outage" depending on scope; status-page text is reviewed
by legal.

## 6. Remediation
- If retrieval scope: fix the scope filter and add a mod-103 test that
  asserts cross-tenant isolation on a canonical adversarial query.
- If model memory: a provider conversation and (potentially) a model
  swap; mod-108 route.
- Mandatory: the leaked content type is added to the mod-107 safety
  suite as a regression; the scrubber rule in [[exercise-02]] is
  reviewed for correctness.

## 7. Postmortem trigger
- Every PII incident triggers a postmortem; no discretion.
- Cross-tenant data exposure additionally triggers a security review
  and may trigger regulatory notification per legal.
```

### 2h. Playbook index (deliverable final)

```markdown
# docs/playbooks/README.md

| id    | Failure mode                 | Primary detection | First lever |
|-------|------------------------------|-------------------|-------------|
| PB-01 | Hallucination outbreak       | [[exercise-03]] confirmed-hallucination panel | ID-validation guardrail or citation-required filter |
| PB-02 | Prompt-injection breakthrough| content_filter spike on single user / safety reports | Per-user rate limit; disable involved tool |
| PB-03 | Tool misuse                  | structural_validity=fail spike on tool_calls | Narrow TOOLS_ENABLED; cap tool-call retries |
| PB-04 | Latency SLO burn             | mod-109 SLO fast-burn / slow-burn alert | mod-109 runbook lever sequence |
| PB-05 | Cost spike                   | mod-109 cost alarm | Per-tenant rate limit; max_tokens cap; model downshift |
| PB-06 | PII / sensitive-data leak    | scrub.rules_count elevated on completions | Safe-mode + disable retrieval |
```

The index is linked from the status page in [[exercise-05]] so that any alert line in the UI is one click from its playbook.

### 2i. Mapping to the exercise deliverables

| Exercise asks for | Section of this solution |
|---|---|
| Common playbook structure | 2a |
| Six playbooks | 2b–2g |
| Playbook index | 2h |

## 3. Validation steps

1. **Every playbook has all seven sections.** `grep -c '^## '` returns 7 for each.
2. **Every detection signal is a query against known span attributes.** Cross-check every "query" clause reads attributes defined in [[exercise-01]] or [[exercise-02]] or [[exercise-03]].
3. **Every playbook names a containment lever that exists.** `TOOLS_ENABLED`, `guardrails.id_validator.enabled`, per-user rate limit, `max_tokens` cap, safe-mode — each must be a real config or a dependency the playbook names.
4. **Communication includes internal-channel + external-wording owner.** Every playbook names `#incident-live` or equivalent and names the owner of the customer-facing wording.
5. **Postmortem trigger is a threshold, not a vibe.** Each playbook's section 7 names a quantity (count affected, % of error budget, surface touched).
6. **Index links to each playbook.** Markdown check: each row's id resolves to the file.
7. **Second-person style.** Playbook body uses "you" — not "the on-call" or passive voice.
8. **Legal/security loop-in flagged where required.** PB-02 and PB-06 loop in security as first action; PB-01 and PB-06 loop in legal on comms.
9. **No playbook contradicts the mod-109 [[exercise-05]] runbook.** PB-04 explicitly defers to it for the lever sequence.
10. **Each playbook has a short name an on-call can remember.** "Hallucination outbreak," "injection breakthrough," "tool misuse," "latency burn," "cost spike," "PII leak."

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Six distinct playbooks, one per failure mode | 2 | Not a single "incident" doc with sections; six docs that stand alone. |
| Common seven-section structure | 1 | Every playbook has all seven sections in order. |
| Detection signals queryable against the span | 2 | Each primary signal names a span attribute or event path. |
| Containment levers exist and are documented | 1 | Each lever points to a real config path or a named dependency. |
| Communication plan names channels and wording owner | 1 | Internal channel, cadence, external-wording owner. |
| Postmortem triggers are quantities | 1 | A threshold a reader can check, not a feeling. |
| Legal / security loop-in flagged where required | 1 | PB-02 and PB-06 specifically. |
| Second-person style, readable in < 2 min | 1 | Tired-human test. |

A submission scoring < 6 is rejected. Hard failures: (a) fewer than six distinct playbooks, (b) a detection signal that requires a human to look at a chart (not a query), (c) a containment lever that does not exist in the stack.

## 5. Common mistakes

- **One generic "LLM incident" playbook.** A hallucination is not a latency burn; the containment levers are different, the communication is different, the postmortem threshold is different.
- **Detection signals that are "the dashboard looks bad."** The dashboard is downstream of the span query. Name the query.
- **Containment levers that don't exist.** "Enable the content filter" without a config path is theatre; the first time it is used, the on-call cannot find it.
- **No legal/security loop-in on PII.** PB-06 that treats a cross-tenant leak like a code bug produces regulatory exposure. Loop in at first action.
- **Communication by Slack alone.** A status page component change (see [[exercise-05]]) is customer-facing truth; a Slack thread is institutional memory. Both run.
- **Postmortem by discretion.** "If it was serious" produces inconsistent postmortems. A threshold produces consistent ones.
- **Playbook in third person.** "The on-call should" reads as a textbook; "you" reads as a procedure.
- **No deferral to the SLO runbook.** PB-04 is the SLO incident. The runbook already exists (mod-109 [[exercise-05]]); the playbook defers to it and adds the LLM-specific bits.
- **Index missing.** A playbook library with no index is one playbook deep — the on-call has to know the right title.
- **Status-page text written during the incident.** Have default wording per playbook. Fill in the scope; do not invent the sentence from scratch at 3 AM.

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the six failure modes above map to multiple OWASP categories: PB-01 engages ML09 (Output Integrity) and ML05 (Model Theft via Response-based attacks); PB-02 engages ML08 (Model Skewing via Adversarial Input) and ML04 (Model Denial-of-Service via resource-draining prompts); PB-03 engages ML04 and ML09 on the tool path; PB-05 engages ML04 directly on cost side-channels; PB-06 engages ML02 (Data Poisoning) when the leak originates in training data and ML03 (Model Inversion) when it originates in parametric memory. The playbooks are the operational Response controls that the OWASP categorisation requires to be documented.
- MITRE ATLAS — <https://atlas.mitre.org/> — ATLAS tactics mapped to each playbook: PB-01 (Collection of adversarial outcomes), PB-02 (Initial Access → Evasion → Privilege Escalation chain), PB-03 (Impact via tool-chain abuse), PB-04 (Impact via Resource Hijacking), PB-05 (same), PB-06 (Exfiltration). Section 4 of each playbook asks the responder to classify against ATLAS tactic ids and record them in the incident doc so pattern analysis across incidents is possible.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the playbooks are the MANAGE-function artefact: measurement (span stream, feedback stream) raises a signal, the playbook is the pre-authorised management action, communication is the GOVERN-function accountability control, the postmortem is the MAP-function update loop. The structure in 2a is deliberately aligned with the NIST functions.

Related exercises: [[exercise-01]] (the span contract every detection signal queries); [[exercise-02]] (scrubber rules power PB-06's detection; T0 retention preserves evidence); [[exercise-03]] (confirmed-hallucination and safety feedback power PB-01 and PB-06); [[exercise-05]] (status page surfaces the active playbook next to its alert; runbook defers to these); mod-105 (agent-level audit log for PB-03); mod-107 (guardrails invoked across PB-01, PB-02, PB-06); mod-108 (gateway levers for PB-02, PB-03, PB-04, PB-05); mod-109 [[exercise-04]] (cost ledger for PB-05); mod-109 [[exercise-05]] (SLO runbook PB-04 defers to).
