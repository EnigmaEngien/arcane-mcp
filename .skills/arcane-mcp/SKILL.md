---
name: arcane-mcp
description: >
  MCP server that exposes the Arcane Docker management API to AI agents.
  127 tools across 14 modules — containers, images, volumes, networks,
  Compose projects, registries, vulnerability scanning, webhooks, system,
  updater, agents/environments, ports, activities/events, and the safety layer.
  Requires Arcane v3+. Built with FastMCP (Python), managed with UV.
  Triggers: working with Docker via Arcane API, managing containers/images/volumes,
  deploying Compose projects, interacting with the arcane-mcp MCP server.
---

# Arcane MCP — Docker Management for AI Agents

A Python MCP server that exposes the [Arcane](https://getarcane.app) Docker management API over the
Model Context Protocol. Any MCP client that can launch a stdio server can use it — OpenCode,
Claude Code, Claude Desktop, Cursor, VS Code.

## Requirements

**Arcane v3.0.0 or newer** (developed against v3.1.1). The v2 API was dropped in `arcane-mcp` 1.0.0:
v3 renamed `/api/environments` to `/api/agents`, moved resource actions onto query parameters, and
paginates every list. On Arcane 2.x, pin `arcane-mcp==0.4.0` — the last release that speaks v2.

## Stack

- **MCP Framework:** FastMCP ≥3.3.1
- **HTTP Client:** httpx (async, lazy client via `client.py`)
- **Env Loading:** python-dotenv (loaded at import)
- **Package Manager:** UV
- **Python:** ≥3.11

## Architecture

```
AI Agent → Arcane MCP (FastMCP) → Arcane API → Docker
```

Each tool category is its own file under `tools/`, exporting `register(mcp: FastMCP)`. `server.py`
imports and calls every `register()`. The HTTP client is lazy — built on first tool call, not at
import — and reads `ARCANE_BASE_URL` / `ARCANE_API_KEY`.

### v3 API facts that shape the code

- **Base path is `/api/agents/{env_id}`.** Build routes with `client.env_path(env_id, suffix)`, or
  `api_collection()` for the env-less collection. Never hand-write the prefix.
- **Actions are query parameters**, not sub-paths: `POST /containers/{id}?action=start`,
  `POST /projects/{id}?action=down|restart|archive|unarchive|updateservices`,
  `POST /system/containers?action=startall|startstopped|stopall`. Project up/redeploy/pull is
  `POST /projects/{id}/deploy?action=...` with a `ProjectDeployOptions` body.
- **Lists paginate at 20.** Every list tool must use `client.paginate_all(url, agent_token=...)`,
  which walks `pagination.hasNextPage`. Without it an agent silently sees the first 20 of 35.
- **Project create is multipart only**: parts `project` (JSON, compose text inside `composeContent`)
  and `manifest` (`{"fileChanges": []}`). A JSON body returns 422.
- **Several action endpoints stream NDJSON** (`application/x-json-stream`), not JSON. Use
  `client.parse_stream_response(resp)` — `resp.json()` raises `Extra data: line 2 column 1`.
- Confirmation tokens carry an `encoding` (`json` / `multipart` / `stream`) so `confirm_operation`
  replays the body correctly.
- The authority for any contract question is the server's own spec: `GET /api/openapi.json`.

## Tool Categories (127 tools, 14 modules)

| Module | Tools | Description |
|--------|-------|-------------|
| **Containers** | 18 | Lifecycle: list, inspect, create, start, stop, restart, kill, pause, unpause, remove, redeploy, commit, update, auto-update, counts. `exec` / `logs` / `stats` return notices — v3 has no endpoint. |
| **Images** | 19 | Pull, remove, tag, prune, build, search, upload, history, export, attestations, vulnerability scan, update checks (`/images/updates/*`) |
| **Volumes** | 18 | Create, inspect, remove, prune, backup/restore, workspace browse and file read. Directory create / upload / delete need the workspace manifest and are not supported |
| **Networks** | 9 | Create, inspect, remove, connect/disconnect (`?action=connect|disconnect`), prune, topology, counts |
| **Projects** | 17 | Multipart deploy, PATCH update, plain DELETE teardown, down/restart/archive/unarchive/update-services, deploy (up/redeploy/pull), build, compose, workspace file, runtime, counts |
| **System** | 9 | Docker info/version, prune, **HEAD** health, upgrade jobs, bulk container start/stop |
| **Environments** | 5 | List, get, create, update (PATCH), remove — "agents" in v3, tool names unchanged |
| **Registries** | 6 | List, create, get, update (PATCH), delete, test — global at `/api/registries` |
| **Webhooks** | 5 | List, create, update, delete. `trigger_webhook` returns a notice — v3 removed the trigger route |
| **Activities** | 7 | List/cancel activities, events, bulk events, delete event, clear history |
| **Vulnerabilities** | 6 | Summary, list all, ignore/unignore (`/vulnerabilities/ignored`), scanner status |
| **Updater** | 3 | Run, status, history |
| **Ports** | 1 | List all port mappings |
| **Operations / Safety** | 4 | confirm_operation, read_audit_log, get_operation_status, classify_tool |

## Key Conventions

- **`env_id: str = "0"`** — targets a specific Arcane agent (local Docker is `"0"`, remote agents UUIDs)
- **`agent_token: str | None = None`** — optional remote-agent auth, sent as `X-Arcane-Agent-Token`;
  pass it to `paginate_all` too or paginated lists lose it
- **Error handling** — always returns a dict, never raises: `{"error", "status_code", "detail"}`
- **Destructive safety (three layers):**
  1. **Confirmation tokens** — destructive tools return a short-lived `confirmation_token` instead
     of acting. The agent must call `confirm_operation(token, operation_hash)` as a separate call.
     Tokens live in memory, expire in 120 s, and are bound to the operation hash (action, target,
     body, params) — changing an argument invalidates them.
  2. **Dry-run mode** — prune tools default to `dry_run=True`.
  3. **Audit log** — every confirmed operation writes three events to `~/.arcane-mcp-audit.log`
     (or `$ARCANE_MCP_AUDIT_LOG`); read it with `read_audit_log()`.

## Quick Commands

```bash
uv sync                                       # Install deps
uv run python -m arcane_mcp.server            # Run the MCP server (stdio)
uv run pytest -q                              # Test suite (181 tests)
uv run ruff check src/ tests/                 # Lint
uv build                                      # Build wheel
```

## Configuration

```env
ARCANE_BASE_URL=https://arcane.example.com:3552
ARCANE_API_KEY=arc_your_key_here
```

Optional: `ARCANE_HTTP_TIMEOUT` (read timeout in seconds, default 120 — httpx's 5 s default kills slow
or remote calls mid-operation), `ARCANE_MCP_AUDIT_LOG`, `ARCANE_API_VERSION` (diagnostic override for
`detect_api_version()`, which routes no longer use).

Pass the variables in the **client's** config — a `.env` only counts if the client happens to launch
the server from that directory.

OpenCode (flat under `mcp`; the `mcp.servers` nesting in OpenCode's docs is not read by the shipping
v2.0.26 build — `opencode mcp list` reports none configured):

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "arcane-mcp": {
      "type": "local",
      "command": ["arcane-mcp"],
      "environment": {
        "ARCANE_BASE_URL": "https://arcane.example.com:3552",
        "ARCANE_API_KEY": "{env:ARCANE_API_KEY}"
      }
    }
  }
}
```

Claude Code, Claude Desktop, Cursor and VS Code each have a verified snippet in the README.

## Source

- Repo: `https://github.com/MikeCase/arcane-mcp`
- Landing page: `https://arcanemcp.splaq.us` (source `fj.splaq.us/splaq/arcane-mcp-site`; increasing
  the tool count or version means updating the badge there too)
- `fj.splaq.us/splaq/arcane-mcp` is a stale copy (June 2026) — not a source of truth

## Related Skills

- **arcane** (`~/.agents/skills/arcane/`) — the Arcane application itself (UI, SSO, GitOps). Use that
  skill for questions about the Arcane app; this skill covers the MCP server only.
