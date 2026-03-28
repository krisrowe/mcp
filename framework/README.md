# Solution Architecture

How we build MCP-powered tools — the frameworks we use, the patterns
we follow, and why.

---

## Foundations

### Third-party

| Component | Role | Link |
|-----------|------|------|
| **MCP Python SDK** (`mcp`) | The official SDK for building MCP servers and clients. Provides `FastMCP` for decorator-based tool definition and `ClientSession` for calling other servers. | [github.com/modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk) |
| **Model Context Protocol** | The open protocol that defines how AI agents discover and call tools. All MCP servers speak this protocol over stdio or HTTP. | [modelcontextprotocol.io](https://modelcontextprotocol.io/) |
| **Click** | CLI framework. Thin CLI wrappers use Click for argument parsing and terminal formatting. | [click.palletsprojects.com](https://click.palletsprojects.com/) |
| **agentskills.io** | Open standard for AI agent skills (SKILL.md format). Used by Claude Code, Gemini CLI, VS Code/Copilot, Cursor, and 30+ other tools. | [agentskills.io](https://agentskills.io/specification) |

### First-party

| Component | Role | Link |
|-----------|------|------|
| **mcp-app** | Config-driven framework for deploying MCP servers as multi-user HTTP services. Adds JWT auth, per-user data isolation, and admin endpoints. Used for cloud-deployed solutions. | [github.com/krisrowe/mcp-app](https://github.com/krisrowe/mcp-app) |
| **gapp** | Deployment lifecycle for Python MCP servers on Google Cloud Run. Handles Terraform, secrets, credential mediation, CI/CD. | [github.com/krisrowe/gapp](https://github.com/krisrowe/gapp) |
| **app-user** | Auth library for multi-user apps. ASGI middleware for JWT validation, user management, per-user data stores. | [github.com/krisrowe/app-user](https://github.com/krisrowe/app-user) |
| **aicfg** | Cross-platform skill and configuration management for Claude Code and Gemini CLI. Marketplace registration, skill installation, MCP server management. | [github.com/krisrowe/aicfg](https://github.com/krisrowe/aicfg) |

### How they fit together

```
┌─────────────────────────────────────────────────┐
│                  Solution Repo                   │
│                                                  │
│   sdk/    ← business logic (importable)          │
│   mcp/    ← thin MCP wrapper (FastMCP)           │
│   cli/    ← thin CLI wrapper (Click)             │
│                                                  │
├──────────────┬──────────────────────────────────-┤
│  Local mode  │  Cloud mode                       │
│  stdio       │  mcp-app (HTTP + auth)            │
│  pipx install│  gapp deploy (Cloud Run)           │
│              │  app-user (JWT + user data)        │
└──────────────┴───────────────────────────────────┘
```

**Local-only tools** (dotfiles-manager, aicfg, bills-agent): The SDK,
MCP server, and CLI are the complete architecture. Install via `pipx`,
register the MCP server locally via stdio transport. No cloud
infrastructure needed.

**Cloud/multi-user tools** (food-agent, monarch-access, gworkspace-access):
Same SDK+MCP+CLI core, plus **mcp-app** for HTTP transport and JWT auth,
**app-user** for per-user data isolation, and **gapp** for deployment to
Cloud Run. The SDK layer is identical to local tools — the cloud stack
wraps it, just as CLI and MCP wrap it.

We use the official **MCP Python SDK** (`from mcp.server.fastmcp import
FastMCP`) for all servers. See [docs/mcp-framework.md](../docs/mcp-framework.md)
for a detailed comparison with the standalone `fastmcp` package and
why we chose the official SDK.

---

## The SDK-First Pattern

### Why

Every solution repo separates business logic from interface layers:

```
my-app/
  sdk/       ← all logic: importable, testable, no I/O concerns
  cli/       ← thin wrapper: calls SDK, formats for terminal
  mcp/       ← thin wrapper: calls SDK, exposes as MCP tool schema
```

**The rule:** If you're writing logic in a CLI command or MCP tool,
stop and move it to SDK.

This exists for three reasons:

1. **Consistency.** The same logic drives the interactive CLI, the AI
   agent interface (MCP), and any direct Python import. No behavior
   divergence between interfaces.

2. **Testability.** SDK functions take arguments and return dicts. They
   don't print, don't format, don't depend on transport. Unit tests
   call them directly — no subprocess spawning, no MCP client setup.

3. **Future clients.** The SDK is structured for programmatic
   consumption whether or not external consumers exist today. Any repo
   can be `pip install`ed and its SDK imported by another package. Some
   already are (gworkspace-access's SDK is imported by
   agentic-consult). The architecture is ready for this whether it
   happens in a week or never.

### Why "SDK"

"SDK" is an architectural posture, not an adoption metric. The term is
justified by how the code is structured, not by how many consumers
exist:

- Functions take arguments and return structured data (dicts)
- No CLI or transport concerns mixed into the logic
- Importable as a Python package
- Designed so that any future client can consume it without refactoring

Alternatives considered:

- **`core/`** — common in Django, Home Assistant, Celery. Implies
  "internal to this app," which undersells the intent for repos where
  the SDK is or will be consumed cross-repo.
- **`lib/`** — common in Rust and Rails, uncommon in Python where it
  often implies vendored dependencies.

We chose `sdk/` for consistency across the ecosystem and because it
accurately describes the design intent.

### Details and constraints

**SDK functions return dicts, not formatted strings.** Every SDK
function returns a dict with a `success` key and relevant data. The CLI
layer formats for the terminal; the MCP layer returns as-is.

```python
# sdk/sync.py
def track(path: str) -> dict:
    # ... business logic ...
    return {"success": True, "path": str(rel_path), "committed": committed}
```

**CLI is one-liner calls to SDK plus formatting:**

```python
# cli/main.py
@main.command()
@click.argument("path")
def track(path):
    result = sync.track(path)
    if result["success"]:
        click.echo(f"Tracked: {result['path']}")
```

**MCP is one-liner calls to SDK:**

```python
# mcp/server.py
@mcp.tool()
def dot_track(path: str) -> dict:
    """Start tracking a file."""
    return sync.track(path)
```

**No business logic in CLI or MCP.** If a function does something
beyond calling an SDK function and formatting the result, it's in the
wrong layer.

**SDK modules can import each other.** `sdk/sync.py` may call
`sdk/repo.py`. The SDK is a cohesive package, not isolated functions.

**Tests target the SDK directly.** Unit tests call SDK functions, not
CLI subprocesses or MCP tool invocations. See
[develop-unit-tests](https://github.com/krisrowe/skills/blob/main/coding/develop-unit-tests/SKILL.md)
for the testing philosophy.

### Examples

The most mature repos following this pattern, suitable as references:

| Repo | SDK package | Notes |
|------|------------|-------|
| [gworkspace-access](https://github.com/krisrowe/gworkspace-access) | `gwsa.sdk` | Cross-repo SDK consumer exists. Mail, docs, drive, chat, calendar modules. Cloud-deployed via gapp. |
| [dotfiles-manager](https://github.com/krisrowe/dotfiles-manager) | `dotgit.sdk` | Clean local-only example. Sync, stores, exclude, remote modules. Bare git repo orchestration. |
| [aicfg](https://github.com/krisrowe/aicfg) | `aicfg.sdk` | Skills marketplace, MCP server management, settings. Local-only. Comprehensive test suite. |
| [food-agent](https://github.com/krisrowe/food-agent) | `food_agent.sdk` | Cloud-deployed via mcp-app + gapp. Multi-user with per-user data isolation. |

---

## Solution Repo Conventions

### Documentation

Every solution repo should have:

- **README.md** — what it is, how to install, how to use. User-facing.
- **CONTRIBUTING.md** — architecture, design principles, constraints,
  testing, how to add features. Contributor-facing (human and agent).
- **CLAUDE.md** — imports README.md and CONTRIBUTING.md via `@` syntax
  so Claude Code loads them as context.
- **.gemini/settings.json** — lists README.md and CONTRIBUTING.md as
  context files so Gemini CLI loads them.

See
[setup-agent-context](https://github.com/krisrowe/skills/blob/main/coding/setup-agent-context/SKILL.md)
for a skill that configures this automatically.

### Packaging

Solution repos are installable Python packages with entry points in
`pyproject.toml`:

```toml
[project.scripts]
my-app = "my_app.cli.main:main"        # CLI entry point
my-app-mcp = "my_app.mcp.server:main"  # MCP server entry point
```

Install with `pipx install` for CLI use. Register MCP server with
`claude mcp add` or `aicfg mcp add` for agent use.
