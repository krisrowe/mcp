# MCP Ecosystem

A collection of frameworks, plugins, and MCP servers for building and deploying personal productivity tools powered by AI agents.

Everything here follows an **MCP-first** design — tools are self-documenting, agent-discoverable, and work with Claude, Gemini, and any MCP-compatible client. CLI is optional.

---

## Frameworks & Infrastructure

| Repo | What it does |
|------|-------------|
| [gapp](https://github.com/krisrowe/gapp) | Deploy Python MCP servers to Google Cloud Run with auth, secrets, and credential mediation. CLI + MCP + Claude plugin. |
| [app-user](https://github.com/krisrowe/app-user) | Auth and user data management for multi-user web apps. JWT auth, admin endpoints, per-user data storage. |
| [claude-plugin-creator](https://github.com/krisrowe/claude-plugin-creator) | Scaffold Claude Code plugins with self-installing dependencies, MCP server config, and hooks. |
| [claude-plugins](https://github.com/krisrowe/claude-plugins) | Plugin marketplace for Claude Code. Install community and personal plugins with `claude plugin install`. |

## MCP Servers — Productivity

| Repo | What it does |
|------|-------------|
| [food-agent](https://github.com/krisrowe/food-agent) | Food logging with nutrition tracking. Timezone-aware daily logs, food catalog, voice-friendly via MCP. |
| [notes](https://github.com/krisrowe/notes) | Note-taking backed by Google Sheets. SDK, CLI, and MCP server with optional AppSheet integration. |
| [bills-agent](https://github.com/krisrowe/bills-agent) | Bill tracking with Monarch Money cross-reference. Claude Code plugin. |
| [pay-calc](https://github.com/krisrowe/pay-calc) | Pay and tax projection tools. |

## MCP Servers — API Integrations

| Repo | What it does |
|------|-------------|
| [monarch-access](https://github.com/krisrowe/monarch-access) | Monarch Money financial data — accounts, transactions, budgets. SDK, CLI, and MCP server. |
| [ticktick-mcp](https://github.com/krisrowe/ticktick-mcp) | TickTick task management — projects, tasks, completion. MCP server. |
| [gworkspace-access](https://github.com/krisrowe/gworkspace-access) | Google Workspace — Gmail, Drive, Docs, Calendar, Chat. MCP server. |
| [dotfiles-manager](https://github.com/krisrowe/dotfiles-manager) | Dotfile management using a bare git repo backed by GitHub. CLI and MCP server. |

## MCP Servers — Developer Tools

| Repo | What it does |
|------|-------------|
| [agentic-consult](https://github.com/krisrowe/agentic-consult) | Developer workstation management — git repo status, Drive backups, Gemini API with local file context, security scanning. |

---

## Architecture

All MCP servers follow a common pattern:

```
my-app/
  sdk/          # Business logic — all behavior lives here
  mcp/          # Thin MCP layer — tool schemas, calls SDK
  cli/          # Optional thin CLI — calls SDK, formats output
```

**SDK-first**: Every feature is implemented in the SDK layer, ensuring consistency whether accessed via MCP, CLI, or direct import.

**Deployment**: Servers run locally via stdio for low-latency agent integration, or deploy to Cloud Run via [gapp](https://github.com/krisrowe/gapp) for remote/mobile access.

**Plugins**: Claude Code plugins bundle MCP servers with auto-installing dependencies via the [self-installing pattern](https://github.com/krisrowe/claude-plugin-creator). Install from the [marketplace](https://github.com/krisrowe/claude-plugins) or point Claude at any plugin repo directly.

## Framework and Design

All servers follow an **SDK-first** pattern: business logic lives in an
importable `sdk/` package, with thin MCP and CLI wrappers that call into
it. This ensures consistent behavior across all interfaces and enables
cross-repo SDK consumption (e.g., gworkspace-access's SDK is imported
by agentic-consult). Local tools run via stdio and pipx; cloud tools
add [mcp-app](https://github.com/krisrowe/mcp-app) for HTTP auth and
[gapp](https://github.com/krisrowe/gapp) for Cloud Run deployment.

See **[framework/](framework/)** for the full architecture guide —
foundations, the SDK-first pattern, constraints, examples, and solution
repo conventions.

Design references:
- [docs/mcp-framework.md](docs/mcp-framework.md) — why we use the official MCP Python SDK over standalone FastMCP
- [docs/mcp-proxy.md](docs/mcp-proxy.md) — MCP tool proxying and server-to-server orchestration
