### exercise-03: MCP Server and Client Integration — Solution

## 1. Solution overview

The exercise asks the engineer to expose the helpdesk-reply-copilot's tool layer through the **Model Context Protocol** (MCP) — not as an end in itself, but because the same tool set needs to be reachable from (a) the surface's own in-process orchestrator, (b) a desktop IDE assistant another team is shipping, and (c) an evals harness that drives the tool loop offline. Those three consumers should not each re-implement auth, pagination, and schema discovery. MCP is the integration contract they share.

The output is a small MCP server (`helpdesk-tools`) that wraps the same three tools from [[exercise-01]] plus a thin resource endpoint, a reference MCP client wired into the surface's orchestrator, and the operational contract — capabilities, auth, transport, lifecycle, logging — that defends the integration across the three consumers.

This solution provides:

- A reference **`helpdesk-tools` MCP server** (Python, stdio + HTTP+SSE transports) exposing `search_knowledge_base`, `lookup_ticket`, `create_followup_task`, and a read-only `principal` resource.
- A reference **MCP client** wrapper the surface's orchestrator uses, which discovers tools at startup, caches the schema, enforces the principal header on every call, and maps protocol errors into the taxonomy from [[exercise-02]].
- An **operational contract** naming the server's capability set, auth scheme, logging surface, session lifecycle, and the "what MCP does *not* do for us" boundary.
- A **worked integration test** that drives the server over stdio, then over HTTP+SSE, with the same assertion set — this is the test that catches transport-coupled bugs.
- A grading rubric.

The lesson is that MCP is a **transport and discovery contract**, not a replacement for the schema and taxonomy work done in [[exercise-01]] and [[exercise-02]]. The server does *not* re-define argument schemas (they ship from the manifest in [[exercise-01]]); it does *not* invent an error shape (every tool error is wrapped by the taxonomy in [[exercise-02]]). MCP's job is exactly: announce tools, carry the call, carry the result, and handle the session plumbing.

> **Spec-pinning note:** the MCP protocol method names used throughout (`tools/list`, `tools/call`, `resources/read`, `initialize`) follow the canonical shapes documented at https://modelcontextprotocol.io. Before shipping, pin the server and client to a specific MCP spec version in `requirements.txt` and record that version in the server's `initialize` response so transport-coupled bugs surface as version mismatches rather than silent protocol drift.

## 2. Worked answer or implementation

### 2a. Why MCP for this surface (deliverable 1 — decision rationale)

The surface has three consumers of the same tool set:

1. The surface's own orchestrator (in-process today, but splitting out to a sidecar next quarter).
2. A desktop IDE assistant another team is shipping that wants read-only access to the same knowledge-base search.
3. The evals harness that re-plays gold-set queries nightly and needs to drive the same tool loop deterministically.

Three options were considered:

| Option | Fit | Why rejected / accepted |
|---|---|---|
| Hand-rolled HTTP JSON APIs per consumer | Low | Three auth schemes, three schema formats, three pagination patterns. Each new consumer is a port. |
| A single OpenAPI backend + per-consumer client libs | Medium | OpenAPI solves schema discovery but not tool-centric semantics (tool metadata, dynamic listing, streamed results). We'd still invent a wrapper per consumer. |
| **MCP server** | **High** | MCP is purpose-built for exposing tools to LLM-driven clients. One `tools/list` + `tools/call` contract; tool schemas are discoverable; both stdio (desktop) and HTTP+SSE (surface, evals) transports are supported. |

The **decision rule**: MCP is justified when ≥ 2 consumers would otherwise duplicate the tool-call plumbing. For a single-consumer surface, a direct in-process call is simpler. We have three consumers, so MCP wins.

The server owns:

- Tool discovery (`tools/list`).
- Tool execution (`tools/call`).
- A `principal` resource clients may read but not mutate.
- Transport-level auth (a `MCP-Principal` header on HTTP; environment variables on stdio).

The server does **not** own:

- Argument schemas (shipped from `tools.yaml` in [[exercise-01]]; the server re-exports).
- Error semantics (every tool failure is a `ToolError` envelope from [[exercise-02]]; the server surfaces it as an MCP `isError: true` result).
- Per-request budget enforcement (that lives in the surface's orchestrator; see [[exercise-05]]).

### 2b. `helpdesk_tools_server.py` — the MCP server (deliverable 2)

The reference server uses the Python MCP SDK's `Server` and `stdio_server` entry points; the HTTP+SSE transport is set up alongside. Learner submissions that use a different SDK or a different language are acceptable as long as the capabilities match.

```python
"""helpdesk_tools_server.py — MCP server exposing the three helpdesk tools.

Transports: stdio (default; desktop IDE) and HTTP+SSE (surface, evals).
Capabilities: tools, resources (read-only principal).
"""

from __future__ import annotations

import json
import logging
import os
import pathlib
from typing import Any

import yaml

# The MCP SDK API shape below matches the "official MCP Python SDK" at the
# time of writing; pin the version in requirements.txt. If the SDK renames
# a decorator, the surface will see an import error at start-up, not at
# first call.
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.server.sse import sse_server
from mcp import types

from tool_error import ToolError
from tool_errors_wrap import (
    wrap_lookup_ticket_failure,
    wrap_search_kb_failure,
    wrap_create_followup_task_failure,
)
from validate_tool_call import Principal, validate_tool_call


log = logging.getLogger("helpdesk-tools")

MANIFEST = yaml.safe_load(pathlib.Path("tools.yaml").read_text())
TOOL_SCHEMAS: dict[str, dict] = {
    name: spec["arguments"] for name, spec in MANIFEST["tools"].items()
}
TOOL_META: dict[str, dict] = MANIFEST["tools"]

server = Server("helpdesk-tools")


# ---- tools/list -------------------------------------------------------------

@server.list_tools()
async def list_tools() -> list[types.Tool]:
    """Return the tool catalog. Model-facing description is here."""
    tools: list[types.Tool] = []
    for name, spec in TOOL_META.items():
        if spec.get("deprecated_at"):
            # Deprecated tools stay listed with a prefix so the client's
            # system prompt can nudge the model to the replacement. See
            # the versioning protocol in [[exercise-01]].
            name_out = name
            description = f"[DEPRECATED {spec['deprecated_at']}] {spec['description']}"
        else:
            name_out = name
            description = spec["description"]
        tools.append(
            types.Tool(
                name=name_out,
                description=description,
                inputSchema=spec["arguments"],
            )
        )
    return tools


# ---- tools/call -------------------------------------------------------------

@server.call_tool()
async def call_tool(name: str, arguments: dict[str, Any]) -> list[types.TextContent]:
    principal = _principal_from_env_or_header()
    request_id = _new_request_id()
    try:
        parsed = validate_tool_call(
            name, arguments, TOOL_SCHEMAS[name], principal
        )
    except Exception as exc:  # ToolValidationError or others
        err = _wrap(name, exc, request_id)
        return [_text_error(err)]

    try:
        if name == "search_knowledge_base":
            result = await _search_kb(parsed, principal)
        elif name == "lookup_ticket":
            result = await _lookup_ticket(parsed, principal)
        elif name == "create_followup_task":
            result = await _create_followup_task(parsed, principal)
        else:
            err = ToolError(
                category="permanent",
                reason_code=f"{name}.not_implemented",
                message=f"tool {name!r} not implemented on this server",
                user_facing_message="That action isn't supported here.",
                tool=name, occurred_at=_now(), request_id=request_id,
            )
            return [_text_error(err)]
    except Exception as exc:  # noqa: BLE001
        err = _wrap(name, exc, request_id)
        return [_text_error(err)]

    return [types.TextContent(type="text", text=json.dumps(result))]


# ---- resources -------------------------------------------------------------

@server.list_resources()
async def list_resources() -> list[types.Resource]:
    # One read-only resource: the current principal. Useful for the IDE
    # client to display "signed in as …" and for evals to assert that a
    # fixture principal is in use.
    return [
        types.Resource(
            uri="principal://current",
            name="Current principal",
            description="Read-only view of the calling principal (tenant_id, user_id, roles).",
            mimeType="application/json",
        )
    ]


@server.read_resource()
async def read_resource(uri: str) -> str:
    if uri != "principal://current":
        raise ValueError(f"unknown resource {uri}")
    p = _principal_from_env_or_header()
    return json.dumps({
        "tenant_id": p.tenant_id, "user_id": p.user_id,
        "surface_id": p.surface_id, "roles": sorted(p.roles),
    })


# ---- helpers ---------------------------------------------------------------

def _wrap(name: str, exc: Exception, request_id: str) -> ToolError:
    mapping = {
        "search_knowledge_base": wrap_search_kb_failure,
        "lookup_ticket": wrap_lookup_ticket_failure,
        "create_followup_task": wrap_create_followup_task_failure,
    }
    return mapping[name](exc, request_id)


def _text_error(err: ToolError) -> types.TextContent:
    """Encode a ToolError as MCP TextContent flagged isError=True.

    The MCP protocol lets tool results indicate an error via a result-level
    flag; the client in §2c reads that flag before parsing JSON.
    """
    return types.TextContent(
        type="text",
        text=err.model_dump_json(),
        annotations={"isError": True},  # client-side convention
    )


def _principal_from_env_or_header() -> Principal:
    # stdio transport: principal comes from env vars (set by parent process).
    # HTTP transport: principal comes from a signed MCP-Principal header
    #                (opaque JWT-ish; verified by the transport wrapper).
    tenant = os.environ.get("MCP_PRINCIPAL_TENANT", "")
    user = os.environ.get("MCP_PRINCIPAL_USER", "")
    surface = os.environ.get("MCP_PRINCIPAL_SURFACE", "")
    roles = frozenset(os.environ.get("MCP_PRINCIPAL_ROLES", "").split(","))
    if not (tenant and user and surface):
        # Fall closed: the server exposes nothing until the transport
        # layer has populated the principal.
        raise PermissionError("missing MCP principal")
    return Principal(tenant_id=tenant, user_id=user, surface_id=surface, roles=roles)


def _new_request_id() -> str:
    import uuid
    return f"req_{uuid.uuid4().hex[:12]}"


def _now():
    from datetime import datetime, timezone
    return datetime.now(timezone.utc)


# ---- actual tool bodies (thin; delegate to the shared backend client) -------

async def _search_kb(args, principal: Principal) -> dict:
    # Calls the retriever contract from mod-103 exercise-01.
    ...


async def _lookup_ticket(args, principal: Principal) -> dict: ...
async def _create_followup_task(args, principal: Principal) -> dict: ...


# ---- entry points ----------------------------------------------------------

def run_stdio():
    import asyncio
    async def _main():
        async with stdio_server() as (reader, writer):
            await server.run(reader, writer, server.create_initialization_options())
    asyncio.run(_main())


def run_http(host: str = "127.0.0.1", port: int = 8402):
    import asyncio
    async def _main():
        async with sse_server(host=host, port=port) as (reader, writer):
            await server.run(reader, writer, server.create_initialization_options())
    asyncio.run(_main())
```

Key points:

- Tool schemas are read from `tools.yaml`; the server is the delivery mechanism, not a second source of truth.
- Principal is sourced from the **transport** (env on stdio, signed header on HTTP+SSE) and validated *before* the tool runs. The MCP protocol itself does not define auth; the surface is responsible for the header, same as any other API.
- Tool errors are packaged as the taxonomy envelope ([[exercise-02]]) and flagged `isError: true` so the client can route them without JSON heuristics.
- Deprecated tools stay listed with a `[DEPRECATED …]` prefix so the model still knows about them during the 30-day window from [[exercise-01]]'s change-management protocol.

### 2c. `helpdesk_tools_client.py` — the MCP client (deliverable 3)

```python
"""helpdesk_tools_client.py — MCP client for the surface orchestrator.

Responsibilities:
  - Spawn / connect to the server (stdio in-process, HTTP for sidecar).
  - Discover tools at startup; cache the schema; refuse unknown tools.
  - Enforce principal on every call (fail-closed if missing).
  - Map MCP protocol errors into the taxonomy from [[exercise-02]].
"""

from __future__ import annotations

import asyncio
import json
import os
from dataclasses import dataclass
from typing import Any, Optional

from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from mcp.client.sse import sse_client
from mcp import types

from tool_error import ToolError
from validate_tool_call import Principal


@dataclass
class MCPConfig:
    transport: str  # "stdio" | "http"
    command: Optional[list[str]] = None  # stdio
    url: Optional[str] = None            # http
    principal: Principal = None


class HelpdeskToolsClient:
    def __init__(self, cfg: MCPConfig) -> None:
        self._cfg = cfg
        self._session: ClientSession | None = None
        self._schemas: dict[str, dict] = {}

    async def connect(self) -> None:
        if self._cfg.transport == "stdio":
            params = StdioServerParameters(
                command=self._cfg.command[0],
                args=self._cfg.command[1:],
                env=self._principal_env(),
            )
            self._ctx = stdio_client(params)
        elif self._cfg.transport == "http":
            headers = self._principal_headers()
            self._ctx = sse_client(self._cfg.url, headers=headers)
        else:
            raise ValueError(f"unsupported transport {self._cfg.transport}")
        self._reader, self._writer = await self._ctx.__aenter__()
        self._session = ClientSession(self._reader, self._writer)
        await self._session.initialize()
        tools = (await self._session.list_tools()).tools
        self._schemas = {t.name: t.inputSchema for t in tools}

    async def close(self) -> None:
        if self._session:
            await self._session.close()
        await self._ctx.__aexit__(None, None, None)

    async def call(self, name: str, arguments: dict[str, Any]) -> Any:
        if not self._session:
            raise RuntimeError("client not connected")
        if name not in self._schemas:
            # Refuse to call a tool the server never announced. This closes
            # the "typosquatted tool" surface.
            raise ValueError(f"unknown tool {name!r} (not in server catalog)")
        res = await self._session.call_tool(name, arguments)
        # Each content entry is either a normal result or an isError envelope.
        for content in res.content:
            if getattr(content, "annotations", None) and content.annotations.get("isError"):
                err = ToolError.model_validate_json(content.text)
                raise ToolInvocationError(err)
        # Success path.
        return json.loads(res.content[0].text)

    def _principal_env(self) -> dict[str, str]:
        p = self._cfg.principal
        return {
            "MCP_PRINCIPAL_TENANT": p.tenant_id,
            "MCP_PRINCIPAL_USER": p.user_id,
            "MCP_PRINCIPAL_SURFACE": p.surface_id,
            "MCP_PRINCIPAL_ROLES": ",".join(sorted(p.roles)),
        }

    def _principal_headers(self) -> dict[str, str]:
        # In production this mints a short-lived signed token server-side;
        # the server verifies the signature and extracts the principal.
        p = self._cfg.principal
        return {
            "MCP-Principal-Tenant": p.tenant_id,
            "MCP-Principal-User": p.user_id,
            "MCP-Principal-Surface": p.surface_id,
            "MCP-Principal-Roles": ",".join(sorted(p.roles)),
            "Authorization": f"Bearer {os.environ['HELPDESK_MCP_TOKEN']}",
        }


class ToolInvocationError(Exception):
    def __init__(self, err: ToolError) -> None:
        super().__init__(err.message)
        self.err = err
```

The client's two guardrails:

- **Server-announced tools only**: a call to a name that was not in `tools/list` is refused client-side. This blocks a "the model emitted a tool name and the client dutifully forwarded it" mis-execution.
- **Typed errors**: the client turns `isError` MCP content into a `ToolInvocationError(ToolError)`. The orchestrator hands the envelope to `route(...)` from [[exercise-02]] without re-parsing.

### 2d. Operational contract (deliverable 4)

```markdown
# helpdesk-tools MCP server — operational contract v1

## Capabilities

- tools (read + one write, as listed in tools.yaml v2)
- resources (read-only: principal://current)
- logging (notifications emitted at WARN+ for server-internal events)

Not supported (today):
- prompts (no prompt templates distributed through the server)
- sampling (clients do not delegate model sampling to this server)

## Transports

- stdio: used by the desktop IDE assistant; principal in env vars.
- HTTP + SSE: used by the surface's sidecar and the evals harness;
  principal in signed headers; server runs behind the surface's auth
  proxy, same posture as any internal service.

## Session lifecycle

- `initialize` is called once per session; the client caches the
  tool catalog.
- Tool catalog hot-reloads are **not** supported on an open session.
  A `tools.yaml` change requires a server restart; clients re-list on
  reconnect. This keeps the catalog deterministic for the agent loop.
- Sessions time out after 30 min idle; clients reconnect on demand.

## Auth

- MCP itself does not define auth; the transport does.
- stdio: trusts the parent process to set `MCP_PRINCIPAL_*` env vars;
  server fails-closed if any required var is missing.
- HTTP: `Authorization: Bearer <short-lived token>` and
  `MCP-Principal-*` headers. The reverse proxy verifies the bearer
  and sets a trusted header `X-Auth-Verified: 1`; the server
  rejects any request without it.

## Logging

- Every tool call logs `(request_id, tool, category, reason_code?, latency_ms)`.
- Per-call payloads are never logged. Payloads are PII-adjacent on this
  surface (ticket bodies, user questions).
- Deprecation warnings log at WARN with the tool name and `deprecated_at`.

## What this server does NOT do

- Rate limit. (The surface's budget enforcement in [[exercise-05]] is
  the per-request authority.)
- Retry. (The surface's `execute_with_retry` in [[exercise-02]] is the
  policy; the server surfaces raw outcomes.)
- Cache tool results. (Cache lives in the retriever, not the MCP layer.)
```

### 2e. Integration test — same assertions over both transports (deliverable 5)

```python
"""test_mcp_integration.py — same tests, two transports."""

import pytest

from helpdesk_tools_client import HelpdeskToolsClient, MCPConfig, ToolInvocationError
from validate_tool_call import Principal

AGENT = Principal(
    tenant_id="t_1", user_id="u_1", surface_id="helpdesk-reply-copilot",
    roles=frozenset({"agent"}),
)


@pytest.fixture(params=["stdio", "http"])
async def client(request):
    if request.param == "stdio":
        cfg = MCPConfig(
            transport="stdio",
            command=["python", "-m", "helpdesk_tools_server"],
            principal=AGENT,
        )
    else:
        cfg = MCPConfig(
            transport="http",
            url="http://127.0.0.1:8402/sse",
            principal=AGENT,
        )
    c = HelpdeskToolsClient(cfg)
    await c.connect()
    yield c
    await c.close()


@pytest.mark.asyncio
async def test_lists_three_tools(client):
    assert set(client._schemas.keys()) == {
        "search_knowledge_base", "lookup_ticket", "create_followup_task",
    }


@pytest.mark.asyncio
async def test_lookup_known_ticket(client):
    out = await client.call("lookup_ticket", {"ticket_id": "ZD-100001"})
    assert out["ticket_id"] == "ZD-100001"


@pytest.mark.asyncio
async def test_not_found_is_permanent(client):
    with pytest.raises(ToolInvocationError) as exc:
        await client.call("lookup_ticket", {"ticket_id": "ZD-999999"})
    assert exc.value.err.category == "permanent"
    assert exc.value.err.reason_code == "lookup_ticket.not_found"


@pytest.mark.asyncio
async def test_bad_arg_is_model_fixable(client):
    with pytest.raises(ToolInvocationError) as exc:
        await client.call("lookup_ticket", {"ticket_id": "nope"})
    assert exc.value.err.category == "model_fixable"
    assert "pattern" in (exc.value.err.model_hint or "").lower()


@pytest.mark.asyncio
async def test_client_refuses_unknown_tool(client):
    with pytest.raises(ValueError, match="unknown tool"):
        await client.call("delete_user", {"user_id": "u_1"})
```

The same test file exercises both transports via a parametrised fixture. That is the whole point of putting the tools behind MCP — the stdio and HTTP paths must look identical to the orchestrator.

### 2f. Implementation notes

- The server imports `tools.yaml` at startup. Adding a tool means editing the manifest and restarting. Hot-reload of a tool catalog on an active session is deliberately not supported because it would let a long-running agent see a different catalog mid-loop.
- The `principal://current` resource is the only resource the server exposes. Resources are the right shape for *static context*; tools are the right shape for *actions*. Learners sometimes model `search_knowledge_base` as a resource — that is wrong; it takes arguments and is not static.
- MCP does not define a per-method budget or a wall-clock timeout; the client wraps `call_tool` with a `asyncio.wait_for` fed by the budget from [[exercise-05]].
- The deprecation prefix in the tool description (`[DEPRECATED …]`) is the model's cue to prefer the replacement. The system prompt from [[exercise-01]]'s change-management protocol should explicitly instruct "prefer the latest version if both are listed."
- A failing principal check is category `permanent`, not `systemic`. A missing principal is a configuration error; a revoked token mid-session is systemic. Match the taxonomy carefully.

## 3. Validation steps

A completed submission is self-validating when all of the following hold:

1. The server is reachable over at least two transports (stdio and HTTP+SSE) and the same integration test file runs green against both.
2. `tools/list` returns the three tools with `inputSchema` equal to the manifest in [[exercise-01]]; no handwritten schema in server code.
3. `tools/call` returns successes as normal content and errors as content annotated `isError` with a parseable `ToolError` envelope ([[exercise-02]]).
4. The server reads principal from the transport (env on stdio, signed headers on HTTP) and fails closed if missing.
5. The client refuses to call a tool not announced by `tools/list` (test exists).
6. The client surfaces errors as `ToolInvocationError(ToolError)`; it does not re-parse strings.
7. The `principal://current` resource is read-only and returns the same principal the server will enforce for the session.
8. The operational contract names what MCP does **not** do: rate limit, retry, cache. These responsibilities live in [[exercise-02]] (retry) and [[exercise-05]] (budget).
9. Tool catalog is immutable per session; a restart is required to pick up a `tools.yaml` change.
10. Deprecated tools appear with a `[DEPRECATED]` description prefix during the 30-day window from [[exercise-01]].

## 4. Rubric or review checklist

Instructor rubric (10 points total):

| Criterion | Points | What earns the points |
|---|---|---|
| Decision rationale justifies MCP over OpenAPI | 1 | Lists ≥ 2 consumers; MCP is justified on consumer count, not hype. |
| Server lists tools from the shared manifest | 1 | No duplicate schema; a YAML edit changes the server. |
| Two transports work with the same tests | 2 | Parametrised fixture; green on both; stdio and HTTP+SSE both covered. |
| Principal sourced from transport, fail-closed | 1 | Missing principal → `PermissionError` before tool runs. |
| Errors wrapped as `ToolError`, flagged `isError` | 1 | Client raises `ToolInvocationError(ToolError)`; no string parsing. |
| Client rejects unannounced tool names | 1 | Explicit test. |
| One read-only resource | 1 | `principal://current`; read-only; shape matches `Principal`. |
| Op contract names non-MCP responsibilities | 1 | Rate limit, retry, cache are all elsewhere. |
| Deprecation handling | 1 | Description prefix + log counter; aligned with [[exercise-01]]. |

A submission scoring < 6 is rejected. The hard failures are (a) handwritten schemas on the server, (b) principal check after the tool runs, (c) same tests do not pass on both transports.

## 5. Common mistakes

- **Treating MCP as a schema source.** The server must read schemas from the manifest in [[exercise-01]]. If someone edits `tools.yaml` and the server's description still says the old thing, the surface and MCP consumers drift.
- **Omitting auth because "MCP doesn't define it."** MCP does not define auth, so you must. On stdio it is env vars from a trusted parent; on HTTP it is a reverse proxy + bearer token + signed principal headers. Fail closed.
- **Treating the MCP server as an authority on rate limiting, retry, or caching.** It is not — the surface-side layers own those. Putting a retry loop inside the server hides retryable failures from the budget counter in [[exercise-05]].
- **Hot-reloading the tool catalog.** Tempting; wrong. An agent loop that sees a different catalog mid-turn can no longer be debugged with the trace.
- **Surfacing MCP protocol errors as prose to the model.** The model does not need to know a `JSON-RPC` parse error happened. Convert everything into the taxonomy and let `route(...)` decide.
- **One test per transport, not one test matrix across transports.** You want the exact same assertion set to pass on stdio and HTTP. Otherwise one transport ships a regression.
- **Modeling `search_knowledge_base` as a resource.** Resources are static; tools take arguments. This is the canonical MCP confusion; keep tools as tools.
- **Letting the server mint principals.** The server trusts the transport. If the server starts fabricating "principals" from body fields, the entire ACL story from [[exercise-01]] (mod-103) is defeated.

## 6. References

- Model Context Protocol — <https://modelcontextprotocol.io/> — the canonical MCP specification; cite the exact spec version pinned in `requirements.txt` on the server.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — the server-is-not-an-authority stance is the surface answer to ML05 (model inversion) / ML07 (transfer learning attack): an MCP server with its own principal minting is an exfiltration surface across tenants.
- MITRE ATLAS — <https://atlas.mitre.org/> — transports + auth + fail-closed principal mapping blunt the ATLAS "Execution" and "Collection" techniques that target tool backends.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — the operational contract is a GOVERN artefact (what the server does, what it doesn't) and the integration test matrix is a MEASURE artefact (coverage across transports).

Related exercises: [[exercise-01]] (mod-104) is the source of truth for schemas; [[exercise-02]] (mod-104) owns the error envelope flagged `isError` on the wire; [[exercise-05]] (mod-104) wraps client `call_tool` with the per-request budget; the retriever contract in [[exercise-01]] (mod-103) is what `search_knowledge_base` delegates to inside the server body.
