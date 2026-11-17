### exercise-05: The Per-Feature Status Page and On-Call Runbook — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to produce two artefacts that **close the loop** on the module's instrumentation, policies, feedback flow, and playbooks: a **per-feature status page** (the public truth about what is up and what is degraded) and the **on-call runbook** that sits behind it (the private guide a tired engineer reads when the status page goes yellow). Together they take the data in [[exercise-01]], the controls from [[exercise-02]], the user-reported signals from [[exercise-03]], and the responses from [[exercise-04]] and render them as a single legible surface for both customers and operators.

The failure modes the exercise defends against are specific and common:

1. **The status page is PR, not engineering.** A marketing page that always says "all systems operational" while the on-call is working an incident destroys trust the first time a customer calls out the mismatch.
2. **The runbook is written for the author.** A bulleted list of dashboards the author remembers reads as gibberish to the on-call who came on-shift twenty minutes ago.
3. **The two are not coupled.** Status says "degraded"; runbook says nothing about what degraded means for *this* feature; the escalation path is a Slack channel search.

The fix is a pair of artefacts that share a schema. The status page component states what the user sees; the runbook names the levers; both link to the same playbook (from [[exercise-04]]) and the same SLO (from mod-109 [[exercise-05]]).

This solution provides:

- A **status page component schema** — per-feature state, its signals, its allowed transitions.
- A **worked status page** for the reference surface (`helpdesk.reply`) with real-shaped signals wired to [[exercise-01]] spans and [[exercise-03]] feedback.
- A **runbook template** with first-15-minutes, dashboards, levers, escalations, and resolve criteria.
- A **worked runbook** for the same feature.
- A **publication pipeline** — how the status page updates automatically from signals vs manually from the on-call.
- A **grading rubric** for the instructor.

The lesson the exercise defends: *an LLM feature cannot be on-called without a status page.* "Monitoring" (dashboards) tells engineers what is happening; a status page tells customers what is happening; a runbook tells engineers what to do about it. The three are different artefacts; teams that collapse them produce confusion in every incident.

## 2. Worked answer or implementation

### 2a. Status page component schema (deliverable 1)

```markdown
## Component (per-feature)

Name:          applied-AI feature name (matches app.feature from [[exercise-01]]).
State:         one of {operational, degraded, partial-outage, major-outage}.
State source:  {auto, manual}. Auto for SLO-driven; manual for playbook-driven.
Signals:       which measurements feed State.
Capability:    short sentence a non-engineer reads about what the user sees.
Playbook:      link to the incident playbook (from [[exercise-04]]) active, if any.
Message:       short customer-facing note. Default wording per playbook; filled by on-call.
Owner:         role (not person) responsible for the component.
History:       last 30 days of state transitions visible to customers.
```

State semantics, pinned:

- **operational**: SLO ([mod-109 exercise-05]) is green AND no manual override.
- **degraded**: user-visible capability loss OR mod-109 slow-burn alert active.
- **partial-outage**: a subset of users cannot complete the feature's golden path.
- **major-outage**: the feature is unavailable to most users.

The state is enum, not free text. Transitions require a reason captured in the history row.

### 2b. Worked status page (deliverable 2)

```markdown
# Status (customer-facing)

## helpdesk.reply (AI Reply Assistant)
State:      operational
Capability: "AI-drafted reply suggestions in Help Desk."
Signals:
 - SLO:                 mod-109 [[exercise-05]] SLI_good rolling 1h.
 - Hallucination rate:  confirmed-hallucination rate from [[exercise-03]].
 - Latency SLO burn:    mod-109 [[exercise-05]] fast-burn / slow-burn.
 - Scrubber activity:   scrub.rules_count rate on completions.
History (last 30 days):
  2026-11-10 14:22 → degraded (provider latency; PB-04)   recovered 15:04
  2026-10-28 09:11 → degraded (prompt rollback; PB-01)    recovered 11:40

## search.compose (AI Search Composer)
State:      operational
Capability: "Natural-language search over your workspace."
Signals:    SLO; confirmed-hallucination; latency; scrubber.
History: (empty for last 30 days)

## pricing.summarise (AI Pricing Summary)
State:      operational
Capability: "AI-summarised pricing on quote pages."
Signals:    SLO; confirmed-hallucination; latency; scrubber.
History: (empty for last 30 days)
```

Rules for the customer-facing page:

- **No raw metrics.** Customers do not read p95. They read a state and a sentence.
- **History is visible.** 30 days of past transitions, including the recovery timestamp. A page that shows only the current state reads as curated marketing.
- **Capability sentence matches the UX.** If users see "AI-drafted reply suggestions," that is what the page says — not "helpdesk-reply-copilot."

### 2c. Signal wiring (deliverable 3)

```yaml
# status-page/signals.yaml
# How the automatic status engine moves a component between states.

helpdesk.reply:
  signals:
    slo_fast_burn:
      source: alert.HelpdeskReplySLOFastBurn      # from mod-109 [[exercise-05]]
      effect: degraded
      playbook: PB-04
      message_default: |
        We are seeing elevated response times on AI-drafted replies.
        You can continue using Help Desk; AI suggestions may be slower
        than usual while we investigate.
    slo_slow_burn:
      source: alert.HelpdeskReplySLOSlowBurn
      effect: degraded
      playbook: PB-04
      message_default: |
        We are seeing elevated response times on AI-drafted replies.
    scrub_rate:
      source: metric.scrub_rules_count_rate_on_completion[10m]
      threshold: "> 2 * baseline_7d"
      effect: degraded
      playbook: PB-06
      message_default: |
        AI-drafted replies are temporarily operating in safe mode while we
        review a privacy indicator. Service is available; suggestions may
        be more conservative than usual.
    hallucination_spike:
      source: argilla.confirmed_hallucination_rate_1h
      threshold: "> 2 * baseline_12w"
      effect: degraded
      playbook: PB-01
      message_default: |
        We are reviewing AI-drafted replies for accuracy. You may see
        fewer AI suggestions while we investigate.

  manual_overrides:
    - partial-outage        # on-call can set with reason
    - major-outage          # requires PM sign-off
```

The signals are read by a small watcher process that owns the status page's state field. The wiring makes the page **auto-drive to `degraded`** on any of the known failure modes; moving to `operational` requires both the signal to clear AND a manual "resolved" by the on-call (the double-trigger prevents flapping).

### 2d. Worked runbook (deliverable 4)

```markdown
# docs/runbook-helpdesk-reply.md — Helpdesk-reply on-call runbook

## You are on-call. You just got paged. Read this.

The alert that paged you carries a `playbook` label. If it does not, something
is wrong upstream; follow the generic PB-00 "unknown alert" path (last
section).

## First 90 seconds (any alert)

1. Open the status page at `/status`. Find the `helpdesk.reply` component.
   If its state is still `operational`, the auto-driver may not have run
   yet. Set state to `degraded` manually with the alert id as the reason.
2. Open the alert; note the `playbook` label (PB-01..PB-06).
3. Click through to the playbook from the index in
   `docs/playbooks/README.md`.

## Dashboards (in this order)

1. `/d/helpdesk-reply-slo`          — mod-109 [[exercise-05]] SLO panel.
2. `/d/helpdesk-reply-latency`      — mod-109 [[exercise-01]] TTFT / TPOT / retry.
3. `/d/helpdesk-reply-cost`         — mod-109 [[exercise-04]] cost + CPSO.
4. `/d/helpdesk-reply-feedback`     — [[exercise-03]] confirmed-signal rates.
5. `/d/helpdesk-reply-tool`         — tool-call structural validity +
                                      tool round-trip p95.

## Playbook routing (first-pass triage in < 2 min)

| If you see                                                         | Go to |
|--------------------------------------------------------------------|-------|
| p95 TTFT above red / slow-burn or fast-burn SLO alert              | [[exercise-04]] PB-04 |
| `app.structural_validity=fail` spike on tool_calls                 | [[exercise-04]] PB-03 |
| `scrub.rules_count` spike on completion events                     | [[exercise-04]] PB-06 |
| Argilla confirmed-hallucination panel outside control band         | [[exercise-04]] PB-01 |
| Daily cost or CPSO above alarm (mod-109 [[exercise-04]])           | [[exercise-04]] PB-05 |
| `content_filter` rate spike on a single user hash OR safety reports| [[exercise-04]] PB-02 |

## Levers (what you can pull, and the cost)

| Lever | Config location | Cost / capability tradeoff | Who signs off |
|---|---|---|---|
| Enable hedge | `routing.yaml: helpdesk.reply.hedge_enabled` | +20–40% primary-side cost on hedged calls; see mod-109 [[exercise-05]]. | On-call (temporary). |
| Flip primary provider | `routing.yaml: helpdesk.reply.primary` | Different CPSO; see mod-108 [[exercise-02]] cost table. | On-call (temporary). |
| Cap `max_tokens` | `routing.yaml: helpdesk.reply.max_tokens` | Shorter replies. | On-call for emergency; PM for > 1 day. |
| Downshift model | `routing.yaml: helpdesk.reply.fallback_model` | Lower quality. | PM for anything beyond 2h. |
| Narrow TOOLS_ENABLED | `src/surface.py: TOOLS_ENABLED` | Capability loss (named tools disabled). | PM for user-visible tools. |
| Per-user rate limit | gateway config (mod-108 [[exercise-01]]) | Target user cannot complete. | On-call (minutes); PM beyond 1h. |
| Safe-mode | `src/surface.py: SAFE_MODE=true` | Conservative completions; refuses on any ID emission. | On-call; mandatory for PB-06. |
| Disable retrieval | `src/surface.py: RAG_ENABLED=false` | Replies lose grounding. | On-call for PB-06; PM otherwise. |

## Escalation

- Minute 0:  You (on-call).
- Minute 15: Rotation owner (if alert not acknowledged).
- Minute 30: Tech lead (helpdesk-platform).
- Minute 45: Director (helpdesk-platform) + platform-SRE on-call.
- Any time: Legal + security for PB-02 or PB-06 (immediately on confirmation).

## Communication

- Open an incident channel: `#inc-helpdesk-reply-<yyyymmdd-hhmm>`.
- Post the first status at minute 5. Template:
    "helpdesk.reply degraded. Playbook: <id>. Containment in progress.
     Status page updated. Next update in 15 min."
- Update the status page component message using the playbook's
  `message_default` from 2c; edit only with PM sign-off.

## Resolve

- Component state transitions to `operational` only when:
  1. The driving signal has cleared for ≥ 30 min AND
  2. The on-call types `/status resolved helpdesk.reply` which captures a
     reason and a resolution timestamp.
- Write a short postmortem stub in `postmortems/` within 24h for any
  incident whose playbook section 7 trigger fired.

## PB-00 — unknown alert

If an alert without a `playbook` label pages you:
1. Set the component to `degraded` with the alert id as reason.
2. Open the alert's query and identify which attribute is firing.
3. Cross-reference the attribute to one of PB-01..PB-06 using the routing
   table above.
4. If genuinely novel: page the tech lead. An alert without a playbook is
   itself a bug in the alerting system; file a post-incident item to add
   the playbook.
```

### 2e. Publication pipeline (deliverable 5)

```markdown
## How the status page updates

Auto-driver (process, runs every 30s):
1. Reads the alerts active on the alertmanager for signals wired in 2c.
2. Reads the Argilla daily metric for the confirmed-hallucination panel.
3. Reads the scrubber metric for the completion-event rate.
4. For each component: computes the next state from the signals, applies
   the default-message if transitioning *to* a worse state, and writes to
   the status store.
5. Flapping protection: a component may not transition to `operational`
   within 30 min of a bad-state entry; the manual "resolve" overrides this.

On-call override (CLI):
   `/status set helpdesk.reply degraded --playbook PB-01 --message "<text>"`
   `/status resolved helpdesk.reply --reason "signal cleared after 45m"`

Audit:
- Every transition is logged with (actor, timestamp, reason).
- The history page renders the last 30 days from the audit log.
- A transition to `major-outage` requires both on-call AND PM approvers
  (two-key enforcement in the CLI).
```

### 2f. Governance (deliverable 6)

```markdown
## Review cadence
- After every incident: runbook is reviewed for gaps within 72h.
- Monthly: on-call retro reviews status-page auto-driver false positives /
  false negatives from the past 30 days. Thresholds in 2c are adjusted,
  and the policy doc records the change.
- Quarterly: the whole runbook is re-read by a non-author (new rotation
  joiner) as a tired-human test; any section that takes > 2 min to parse
  is rewritten.

## Ownership
- Status page: applied-AI platform (role).
- Per-component message defaults: PM for that feature (role).
- Runbook: applied-AI platform (role) in partnership with helpdesk-platform.

## Change control
- Threshold changes in 2c require two reviews: the on-call rotation owner
  and the component PM.
- Adding a new component to the status page requires an owner + a signals
  wiring + at least one playbook mapping before go-live.
```

### 2g. Mapping to the exercise deliverables

| Exercise asks for | Section of this solution |
|---|---|
| Status page schema | 2a |
| Worked per-feature status page | 2b |
| Signal wiring (auto-driver) | 2c |
| On-call runbook | 2d |
| Publication pipeline | 2e |
| Governance / review cadence | 2f |

## 3. Validation steps

1. **Status page states are enum.** `grep -P '\bState: ' status.md | sort -u` yields values only from the 4-element enum.
2. **Every signal wiring points to a real alert / metric.** Cross-check 2c against alertmanager rules and the metric names in [[exercise-01]] / [[exercise-03]].
3. **Runbook first-90-seconds fits one screen.** ≤ 30 lines in the "First 90 seconds" + "Dashboards" sections combined.
4. **Playbook routing table covers all six playbooks.** Six rows, each resolving to PB-01..PB-06.
5. **Every lever names a signoff role.** No row of 2d's lever table reads "no signoff needed" — the on-call always knows who to loop in for a > 1h-duration lever.
6. **Flapping protection works.** Simulate a signal that rings on-and-off every 20 min; the status page stays `degraded` for the 30 min hold on each cycle.
7. **Manual override logs an audit row.** Set `degraded` via CLI; the audit log row has actor + timestamp + reason.
8. **Two-key enforcement on major-outage.** Attempt to set `major-outage` from one actor; the CLI refuses without the PM approver.
9. **History renders transitions and recoveries.** A full degraded-then-recovered cycle shows both rows in the 30-day history.
10. **Non-author tired-human test passes.** A new rotation joiner reads the runbook cold and reaches the right playbook within 2 minutes on each of the six scripted scenarios.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Status page schema with enum states | 1 | Four states, defined; transitions reasoned. |
| Signals wired to real sources | 2 | Each signal names an alert id or metric path from this module. |
| Playbook mapping present | 1 | Every signal points to a PB-01..PB-06. |
| Default message per playbook | 1 | Customer-facing wording pre-written; filled with scope at incident time. |
| Runbook has first-90-seconds section | 1 | Fits on one screen; readable by a tired human. |
| Lever table with signoff per lever | 1 | No "no signoff" rows; everything has a role. |
| Escalation ladder in minutes | 1 | Zero / 15 / 30 / 45; legal + security called out for PB-02 / PB-06. |
| Resolve criteria are not discretionary | 1 | Signal cleared ≥ 30 min AND explicit CLI command. |
| Flapping protection + two-key major-outage | 1 | Policy stated; enforcement mentioned. |

A submission scoring < 6 is rejected. Hard failures: (a) status page states are free text, (b) runbook never links to playbooks, (c) no history on the customer-facing page.

## 5. Common mistakes

- **Status page that only shows green.** Customers notice; trust erodes. Visible history is the fix.
- **Status page with raw percentiles.** Customers do not read p95. They read a state and a sentence.
- **Runbook author-oriented.** Second-person and one-screen-first-view are the fixes.
- **Default message invented at 3 AM.** Pre-write per playbook; fill in scope.
- **No link from status page to playbook.** A customer-visible "degraded" with no internal routing means every incident starts with a Slack search.
- **Resolve on gut feel.** A clear-and-then-manual-command double-trigger is cheap and prevents flapping.
- **Manual overrides without audit.** When something gets questioned, nobody can say when or why the state changed.
- **No governance review.** Thresholds decay; a monthly retro is what keeps them honest.
- **No two-key on major-outage.** A single typo cannot flip a customer-visible "major outage"; the two-key keeps that honest.
- **Runbook as the only source.** The status page is customer-facing truth; the runbook is internal. Collapsing them produces a page that neither customers nor on-calls can read.

## 6. References

- Google SRE Workbook, "Incident Response" and "Communicating" chapters — <https://sre.google/workbook/incident-response/> — the runbook's escalation ladder, communication cadence, incident channel pattern, and resolve criteria are the SRE-book defaults applied to an LLM feature. Verify the current chapter titles and wording before citing in the posture doc. <!-- verify-before-publishing: pin the chapter URLs and dates against the current SRE Workbook contents. -->
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the runbook is the GOVERN + MANAGE artefact that makes the module's MEASURE stream actionable; the status page is the GOVERN-function accountability surface to external stakeholders. The quarterly runbook review and the audit-logged state transitions are explicit GOVERN controls.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the lever table includes the Response controls that defend against ML04 (Denial-of-Service) via `max_tokens` cap and rate limit, ML08 (adversarial input) via per-user rate limit, ML09 (output integrity) via safe-mode and retrieval-disable. The status page's "degraded" state is itself the public-facing control that lets a surface fail *loudly*, which is a Detection control in the ATLAS response-plan sense.
- MITRE ATLAS — <https://atlas.mitre.org/> — the runbook's PB-02 and PB-06 escalations align with ATLAS Response actions (containment, eradication, recovery) in an LLM-specific context. The incident-channel naming and the two-key major-outage control are operational artefacts of the Response phase.

Related exercises: [[exercise-01]] (the span attributes the signals read); [[exercise-02]] (scrubber metrics power the privacy signal; retention guarantees evidence survives to resolution); [[exercise-03]] (confirmed-hallucination panel is a signal source); [[exercise-04]] (every playbook ends here, and the runbook defers every lever decision to the relevant playbook); mod-109 [[exercise-05]] (the SLO and its burn-rate alerts are the primary latency signal; the runbook there is the latency-specific sibling of this one); mod-108 (the lever configs the runbook names live in the gateway).
