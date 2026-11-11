### exercise-01: Tool Schema Design and Argument Validation — Solution

## 1. Solution overview

The exercise asks the applied-AI engineer to design the **tool surface** the model sees when the helpdesk-reply-copilot graduates from "retrieve-and-answer" to "retrieve, reply, and act." A tool schema is a contract between the model and the host: the model has only the schema to decide whether to call a tool, what to pass, and how to interpret a return. If the schema is loose, the model fills the gaps — with hallucinated arguments, enum values the backend has never seen, and free-text where an ID belongs. If the schema is tight but hostile ("required: user_email_or_phone_or_ticket_id, exactly one"), the model loops on validation errors instead of making progress.

This solution provides:

- A reference **tool manifest** (`tools.yaml` + generated JSON Schemas) for three tools on the helpdesk-reply-copilot surface: `search_knowledge_base`, `lookup_ticket`, and `create_followup_task`.
- A reference **argument validator** (`validate_tool_call.py`) that layers OpenAPI/JSON-Schema structural checks, Pydantic semantic checks, and ACL / principal checks — with every validation failure producing a model-readable repair hint.
- A reference **versioning and deprecation protocol** for tool schemas, including the fields every tool definition must expose (`tool_version`, `schema_version`, `deprecated_at`) and the "argument shape drift" guard CI runs on every PR.
- A reference **unit test suite** proving (a) every tool rejects extra properties, (b) every enum is enforced, (c) every ID-shaped field has a regex, and (d) every ACL-sensitive tool rejects a bare request (no `tenant_id` / `user_id`).
- A **grading rubric** for the instructor.

The lesson the exercise defends is: *the schema is the first production guardrail*. A tool whose JSON Schema allows `additionalProperties: true`, has no enums, and has a free-text ID field is one creative model call away from a prod incident. The solution encodes that discipline in artifacts a learner can inspect directly.

## 2. Worked answer or implementation

### 2a. `tools.yaml` — the tool manifest (deliverable 1)

The manifest is the source of truth. The JSON Schemas shipped to the model, the Pydantic classes used server-side, and the OpenAPI stubs for the backend are generated from it. The surface never hand-writes schemas in two places.

```yaml
# tools.yaml — helpdesk-reply-copilot tool manifest, v2
version: 2
surface_id: helpdesk-reply-copilot
owner_team: applied-ai

tools:
  search_knowledge_base:
    tool_version: 1.2.0
    schema_version: 2
    description: |
      Search the tenant's knowledge sources (helpcenter, zendesk tickets,
      salesforce knowledge) for passages that answer the user's question.
      Returns up to `k` passages with citations. Does not write.
    side_effects: read_only
    deprecated_at: null
    arguments:
      type: object
      additionalProperties: false
      required: [query]
      properties:
        query:
          type: string
          minLength: 3
          maxLength: 500
          description: The user's question, in natural language.
        source:
          type: string
          enum: [helpcenter, zendesk_tickets, salesforce_knowledge, all]
          default: all
          description: Restrict to a single source; default searches all three.
        k:
          type: integer
          minimum: 1
          maximum: 10
          default: 5
    returns:
      type: object
      required: [passages, index_freshness]
      properties:
        passages: { type: array, items: { $ref: "#/components/schemas/Passage" } }
        index_freshness:
          type: object
          additionalProperties:
            type: string
            enum: [healthy, degraded]

  lookup_ticket:
    tool_version: 1.0.1
    schema_version: 2
    description: |
      Fetch a single Zendesk ticket by ID, scoped to the current tenant.
      Returns the ticket subject, body, status, and last 5 comments.
      Does not write.
    side_effects: read_only
    deprecated_at: null
    arguments:
      type: object
      additionalProperties: false
      required: [ticket_id]
      properties:
        ticket_id:
          type: string
          pattern: "^ZD-[0-9]{6,10}$"
          description: Zendesk ticket ID in the form ZD-123456.
        include_private_comments:
          type: boolean
          default: false
          description: |
            Only effective if the calling principal has role=agent on this ticket.
            Non-agents who set this to true will receive a 403 and a model-repair hint.
    returns: { $ref: "#/components/schemas/Ticket" }

  create_followup_task:
    tool_version: 0.9.0
    schema_version: 2
    description: |
      Create a follow-up task for the current CS agent to work later.
      Requires an explicit user confirmation in the surface before invocation
      (see [[exercise-02]]). Writes to the CS queue.
    side_effects: write
    deprecated_at: null
    requires_user_confirmation: true
    arguments:
      type: object
      additionalProperties: false
      required: [ticket_id, summary, due_at]
      properties:
        ticket_id:
          type: string
          pattern: "^ZD-[0-9]{6,10}$"
        summary:
          type: string
          minLength: 10
          maxLength: 280
          description: One-sentence task description shown in the agent's queue.
        due_at:
          type: string
          format: date-time
          description: RFC 3339 timestamp, UTC. Must be in the future.
        priority:
          type: string
          enum: [low, normal, high]
          default: normal
    returns:
      type: object
      required: [task_id, created_at]
      properties:
        task_id: { type: string, pattern: "^TASK-[A-Z0-9]{8}$" }
        created_at: { type: string, format: date-time }
```

Properties worth naming explicitly:

- `additionalProperties: false` on **every** tool. Without it, the model is free to hallucinate `{"cc_user": "boss@..."}` on `create_followup_task`, the backend silently drops it, and the model believes it worked.
- Every ID field has a `pattern`. The model will cheerfully pass `"the ticket we were just discussing"` if the field is `type: string` without a regex.
- Every string field has `maxLength`. Unbounded string fields are an injection surface and a token-bloat risk.
- Enums instead of free-text wherever the backend has a closed set.
- `side_effects` is tool-level metadata, not argument-level. The surface uses it to decide whether to show a confirm step (`requires_user_confirmation: true`) and whether to include the tool in a read-only "draft" mode.
- `tool_version` is semver on the backend contract; `schema_version` is the version of this manifest format. They move independently.

### 2b. `validate_tool_call.py` — the layered validator (deliverable 2)

```python
"""validate_tool_call.py — layered validation for tool calls.

Order of checks (short-circuit on failure):
  1. Structural: argument object matches the JSON Schema in tools.yaml.
  2. Semantic: Pydantic model enforces cross-field invariants
     (e.g., due_at must be in the future, include_private_comments
     only allowed for role=agent).
  3. Principal: calling user is allowed to invoke this tool on these
     arguments (ACL, tenant scoping).
Every failure returns a ToolValidationError whose .model_hint is
written for the model to self-correct on re-prompt.
"""

from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Any

import jsonschema
from pydantic import BaseModel, Field, field_validator, model_validator


class ToolValidationError(Exception):
    """Raised at the validator boundary. .model_hint is shown to the model."""

    def __init__(self, message: str, model_hint: str, category: str) -> None:
        super().__init__(message)
        self.model_hint = model_hint
        self.category = category  # structural | semantic | principal


# ---- semantic layer (Pydantic) ----------------------------------------------

class SearchKnowledgeBaseArgs(BaseModel):
    query: str = Field(min_length=3, max_length=500)
    source: str = Field(default="all")
    k: int = Field(default=5, ge=1, le=10)

    @field_validator("source")
    @classmethod
    def _source_in_enum(cls, v: str) -> str:
        allowed = {"helpcenter", "zendesk_tickets", "salesforce_knowledge", "all"}
        if v not in allowed:
            raise ValueError(f"source must be one of {sorted(allowed)}, got {v!r}")
        return v


class LookupTicketArgs(BaseModel):
    ticket_id: str = Field(pattern=r"^ZD-[0-9]{6,10}$")
    include_private_comments: bool = False


class CreateFollowupTaskArgs(BaseModel):
    ticket_id: str = Field(pattern=r"^ZD-[0-9]{6,10}$")
    summary: str = Field(min_length=10, max_length=280)
    due_at: datetime
    priority: str = "normal"

    @model_validator(mode="after")
    def _due_in_future(self) -> "CreateFollowupTaskArgs":
        now = datetime.now(tz=timezone.utc)
        if self.due_at.tzinfo is None:
            raise ValueError("due_at must be timezone-aware (UTC)")
        if self.due_at <= now:
            raise ValueError("due_at must be in the future")
        return self

    @field_validator("priority")
    @classmethod
    def _priority_in_enum(cls, v: str) -> str:
        if v not in {"low", "normal", "high"}:
            raise ValueError("priority must be low | normal | high")
        return v


ARG_MODELS: dict[str, type[BaseModel]] = {
    "search_knowledge_base": SearchKnowledgeBaseArgs,
    "lookup_ticket": LookupTicketArgs,
    "create_followup_task": CreateFollowupTaskArgs,
}


# ---- principal layer --------------------------------------------------------

@dataclass(frozen=True)
class Principal:
    tenant_id: str
    user_id: str
    surface_id: str
    roles: frozenset[str]  # e.g., {"agent"}, {"viewer"}


def _check_principal(
    tool_name: str, args: BaseModel, principal: Principal
) -> None:
    if not principal.tenant_id or not principal.user_id:
        raise ToolValidationError(
            "missing principal",
            model_hint="This tool requires an authenticated session. "
                       "Ask the user to sign in and try again.",
            category="principal",
        )
    if tool_name == "lookup_ticket" and isinstance(args, LookupTicketArgs):
        if args.include_private_comments and "agent" not in principal.roles:
            raise ToolValidationError(
                "include_private_comments requires role=agent",
                model_hint="Call this tool again with include_private_comments=false. "
                           "The current user is not an agent on this ticket.",
                category="principal",
            )
    if tool_name == "create_followup_task" and "agent" not in principal.roles:
        raise ToolValidationError(
            "only agents may create follow-up tasks",
            model_hint="Do not call create_followup_task. The current user cannot "
                       "create tasks; offer to draft the summary in the reply instead.",
            category="principal",
        )


# ---- public entrypoint ------------------------------------------------------

def validate_tool_call(
    tool_name: str,
    arguments: dict[str, Any],
    schema: dict[str, Any],
    principal: Principal,
) -> BaseModel:
    """Validate a model-emitted tool call. Returns the parsed Pydantic args."""
    if tool_name not in ARG_MODELS:
        raise ToolValidationError(
            f"unknown tool {tool_name!r}",
            model_hint=f"Use only the tools in the available-tools list. "
                       f"Allowed: {sorted(ARG_MODELS)}.",
            category="structural",
        )

    # 1. Structural
    try:
        jsonschema.validate(arguments, schema)
    except jsonschema.ValidationError as e:
        raise ToolValidationError(
            f"schema violation on {'/'.join(map(str, e.absolute_path))}: {e.message}",
            model_hint=_hint_from_jsonschema_error(tool_name, e),
            category="structural",
        ) from e

    # 2. Semantic
    try:
        parsed = ARG_MODELS[tool_name](**arguments)
    except Exception as e:
        raise ToolValidationError(
            f"semantic validation failed: {e}",
            model_hint=_hint_from_pydantic_error(tool_name, e),
            category="semantic",
        ) from e

    # 3. Principal
    _check_principal(tool_name, parsed, principal)

    return parsed


def _hint_from_jsonschema_error(tool: str, e: jsonschema.ValidationError) -> str:
    # Keep hints short, imperative, and tool-specific. The model has
    # one more shot; make it count.
    path = ".".join(map(str, e.absolute_path)) or "<root>"
    if e.validator == "required":
        return f"Add the missing field(s) {e.message} and re-call {tool}."
    if e.validator == "additionalProperties":
        return f"Remove the extra field(s) from arguments; only the fields "\
               f"in the schema are allowed for {tool}."
    if e.validator == "enum":
        return f"Field {path!r} must be one of {e.schema.get('enum')}."
    if e.validator == "pattern":
        return f"Field {path!r} must match the pattern "\
               f"{e.schema.get('pattern')!r}."
    if e.validator in {"minLength", "maxLength", "minimum", "maximum"}:
        return f"Field {path!r} violates bound {e.validator}={e.validator_value}."
    return f"Fix field {path!r}: {e.message}"


def _hint_from_pydantic_error(tool: str, e: Exception) -> str:
    # In production this reaches into Pydantic's error list; shown abridged.
    return f"Fix the arguments to {tool} and re-call. Error: {e}"
```

The three-layer order is deliberate. Structural errors are the cheapest to produce and the easiest for the model to repair. Semantic errors need a Pydantic class because JSON Schema cannot express "due_at must be in the future." Principal errors must fire *last*, because a schema-violating call does not deserve a 403 — it deserves a schema correction.

### 2c. Versioning and deprecation protocol (deliverable 3)

```markdown
# Tool Schema Change Protocol

## Fields every tool exposes

- `tool_version`: semver of the backend contract. Bump major on breaking changes.
- `schema_version`: version of the manifest format itself. The surface pins it.
- `deprecated_at`: null or RFC 3339 timestamp. If non-null, the surface still
  exposes the tool but logs a deprecation counter on every call.

## Patch (no version bump)

- Add an optional field with a default.
- Add a value to a non-closed enum (one not marked `closed: true`).
- Tighten `maxLength` where no existing caller approaches the limit
  (requires a 7-day log review on the raw argument distribution).

## Breaking (major bump + deprecation window)

- Remove or rename a field.
- Change a field's type.
- Add a required field.
- Add a value to an enum marked `closed: true`.
- Change `additionalProperties` from true to false (we ship it as false
  from day one; see §2a).

Breaking changes land under a new tool name: `lookup_ticket_v2`. The old
tool stays in the manifest with `deprecated_at` set and a 30-day
deprecation window. The model sees both; the system prompt instructs
it to prefer the newer version. After the window, the old tool is
removed and the surface rejects calls to it with a `migrated_to` hint.

## CI guard: "argument shape drift"

A CI job diffs `tools.yaml` against main and fails on any breaking
change that does not also add a new tool entry. Learners are expected
to show this CI config in the submission.
```

### 2d. Unit tests (deliverable 4)

```python
"""test_tool_validation.py — unit tests for validate_tool_call."""

import json
import pathlib

import pytest

from validate_tool_call import (
    Principal,
    ToolValidationError,
    validate_tool_call,
)

SCHEMAS = json.loads(pathlib.Path("generated/tool_schemas.json").read_text())

AGENT = Principal(
    tenant_id="t_1", user_id="u_1", surface_id="helpdesk-reply-copilot",
    roles=frozenset({"agent"}),
)
VIEWER = Principal(
    tenant_id="t_1", user_id="u_2", surface_id="helpdesk-reply-copilot",
    roles=frozenset({"viewer"}),
)


class TestSearchKnowledgeBase:
    def test_happy_path(self):
        parsed = validate_tool_call(
            "search_knowledge_base",
            {"query": "how do I reset my password", "k": 3},
            SCHEMAS["search_knowledge_base"],
            AGENT,
        )
        assert parsed.query.startswith("how do I")
        assert parsed.k == 3
        assert parsed.source == "all"

    def test_rejects_extra_property(self):
        with pytest.raises(ToolValidationError) as exc:
            validate_tool_call(
                "search_knowledge_base",
                {"query": "how to reset", "cc_boss": True},
                SCHEMAS["search_knowledge_base"],
                AGENT,
            )
        assert exc.value.category == "structural"
        assert "extra field" in exc.value.model_hint.lower()

    def test_rejects_bad_enum(self):
        with pytest.raises(ToolValidationError) as exc:
            validate_tool_call(
                "search_knowledge_base",
                {"query": "how to reset", "source": "slack"},
                SCHEMAS["search_knowledge_base"],
                AGENT,
            )
        assert "must be one of" in exc.value.model_hint


class TestLookupTicket:
    def test_rejects_free_text_id(self):
        with pytest.raises(ToolValidationError) as exc:
            validate_tool_call(
                "lookup_ticket",
                {"ticket_id": "the one we were just discussing"},
                SCHEMAS["lookup_ticket"],
                AGENT,
            )
        assert exc.value.category == "structural"
        assert "pattern" in exc.value.model_hint.lower()

    def test_private_comments_requires_agent(self):
        with pytest.raises(ToolValidationError) as exc:
            validate_tool_call(
                "lookup_ticket",
                {"ticket_id": "ZD-123456", "include_private_comments": True},
                SCHEMAS["lookup_ticket"],
                VIEWER,
            )
        assert exc.value.category == "principal"
        assert "include_private_comments=false" in exc.value.model_hint


class TestCreateFollowupTask:
    def test_due_in_past_is_rejected(self):
        with pytest.raises(ToolValidationError) as exc:
            validate_tool_call(
                "create_followup_task",
                {
                    "ticket_id": "ZD-999999",
                    "summary": "follow up on refund",
                    "due_at": "2020-01-01T00:00:00Z",
                },
                SCHEMAS["create_followup_task"],
                AGENT,
            )
        assert exc.value.category == "semantic"

    def test_viewer_cannot_write(self):
        with pytest.raises(ToolValidationError) as exc:
            validate_tool_call(
                "create_followup_task",
                {
                    "ticket_id": "ZD-999999",
                    "summary": "follow up on refund",
                    "due_at": "2099-01-01T00:00:00Z",
                },
                SCHEMAS["create_followup_task"],
                VIEWER,
            )
        assert exc.value.category == "principal"
```

### 2e. Implementation notes

- The model sees only the `description`, `arguments` schema, and the tool name — nothing about `side_effects`, `requires_user_confirmation`, or `ARG_MODELS`. Those are host-side metadata. Keep the description tight: the field has a cost (tokens on every turn) and a benefit (fewer bad calls). Use it to clarify *what the tool does not do* ("Does not write") when confusion is plausible.
- The model-hint strings do not include principal identifiers (`tenant_id`, `user_id`). Those are not the model's business and may leak into training data on provider-side retention.
- `additionalProperties: false` on the arguments object is shipped to the model as part of the tool spec. Modern frontier models respect it; the validator still enforces it belt-and-braces.
- The pattern on IDs (`^ZD-[0-9]{6,10}$`) also forces the model to find a real ID from context (retrieval, a prior tool result) rather than inventing one. Learners sometimes drop the pattern "for flexibility" — flexibility is where injection and hallucinated-ID incidents live.
- The joint retriever-contract from [[exercise-01]] (mod-103) already defines `index_freshness` and `versions` shapes; the `search_knowledge_base` return shape reuses those types so the two contracts cannot drift.
- Argument validation is a layer; it is not the whole story. The error-handling taxonomy in [[exercise-02]] defines how the parsed-but-failing tool call surfaces back to the model. A rejected call is a `model_fixable` error there; a succeeding call that then hits a backend 500 is a `retryable` error.

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. `tools.yaml` defines at least three tools; every tool has `additionalProperties: false`, every ID field has a `pattern`, every string has a `maxLength`, every enum field is declared `enum`.
2. Every tool declares `side_effects` as one of `read_only | write | admin`. Every `write` tool declares `requires_user_confirmation: true`.
3. Every tool declares `tool_version`, `schema_version`, and `deprecated_at` (nullable).
4. `validate_tool_call(...)` runs the three layers in the order **structural → semantic → principal**; a learner who checks principal first has leaked tool existence to unauthorized users.
5. Every `ToolValidationError` carries a `category` ∈ `{structural, semantic, principal}` and a `model_hint` string. The hint is imperative, tool-specific, and never names a principal identifier.
6. `pytest` passes with at least: happy-path, extra-property, bad-enum, bad-pattern, cross-field semantic violation, and principal denial.
7. The CI guard script fails a PR that renames a required field without adding a `_v2` tool.
8. The `search_knowledge_base` return shape reuses the `Passage` and `index_freshness` types defined in the retriever contract from [[exercise-01]] (mod-103). A learner who redefines them in `tools.yaml` is rejected.

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Manifest covers ≥ 3 tools incl. one write | 1 | One read, one write with confirm, one ACL-sensitive. |
| `additionalProperties: false` everywhere | 1 | Including nested objects. |
| ID fields have patterns + strings have maxLength | 1 | Grep the manifest; no untyped free-text IDs. |
| Enums are closed where the backend is closed | 1 | `source`, `priority`, `side_effects` are enums. |
| Three-layer validator in correct order | 2 | Structural first; principal last. Each layer tested. |
| Model-hint strings are imperative + tool-specific | 1 | No "validation failed"; every hint names the field and the fix. |
| Semantic rules via Pydantic cross-field | 1 | At least one `model_validator` (`due_at` future). |
| Principal checks fail closed | 1 | Missing `tenant_id`/`user_id` → deny. |
| Version + deprecation protocol documented | 1 | Patch vs. breaking named; 30-day window stated. |
| CI guard against shape drift | 1 | A test or workflow that fails on a breaking rename. |

A submission scoring < 6 is rejected. The hard failures are (a) an unbounded string on an ID field, (b) `additionalProperties: true` anywhere, (c) principal checks before structural checks.

## 5. Common mistakes

- **`additionalProperties: true` (or omitted).** JSON Schema's default is permissive; omitting the field is equivalent to letting the model invent keys. Every object type needs `additionalProperties: false` explicitly.
- **Unbounded string on an ID field.** The model will cheerfully hallucinate `"ticket_id": "the one we were just discussing"`. Pattern + maxLength are mandatory.
- **One mega-tool.** A single `helpdesk_do_thing(action, ...)` tool collapses the type system. Each distinct side effect is its own tool so the schema can enforce its own argument shape.
- **Principal checks before structural checks.** Returning 403 to a malformed call *admits the tool exists* and leaks a validation oracle. Structural first, then semantic, then principal.
- **Model-hint strings that aren't actionable.** "Invalid argument" tells the model nothing; "Field `source` must be one of [helpcenter, zendesk_tickets, salesforce_knowledge, all]" tells it exactly what to do.
- **Describing side effects in prose only.** If `create_followup_task` writes, it must be marked `side_effects: write` as structured metadata so the surface can show a confirm step without parsing the description.
- **Hand-writing schemas in two places.** The manifest, the Pydantic models, and the OpenAPI stubs drift the day someone forgets to update one. Generate from a single source.
- **Loosening a schema to "unblock the model."** If the model keeps calling with bad args, the fix is a better description or a narrower enum, not `additionalProperties: true`. The next failure mode is silent.
- **Omitting `tool_version` / `schema_version`.** The first tool-schema incident starts with "nobody could tell which version prod was running."

## 6. References

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — argument validation is the surface's answer to ML01 (input manipulation) and ML06 (output integrity): an unvalidated tool argument is an unsanitised input that reaches a privileged backend.
- MITRE ATLAS — <https://atlas.mitre.org/> — ATLAS maps adversary techniques to AI systems; a free-text ID field is an injection surface that enables several of the "LLM-Assisted Prompt Injection" and "Resource Access" techniques.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the layered validator with `category` tags is a MEASURE artefact (categorized counters feed the readout) and the version + deprecation protocol is a MANAGE artefact (controls on change).

Related exercises: [[exercise-02]] (mod-104) defines the error taxonomy that the `category` field feeds into; [[exercise-03]] (mod-104) wraps these tools in an MCP server; [[exercise-05]] (mod-104) wires per-request budget caps that count tool calls; the retriever contract in [[exercise-01]] (mod-103) owns the `Passage` and `index_freshness` types reused here.
