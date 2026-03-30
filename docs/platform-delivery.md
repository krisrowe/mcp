# Platform Delivery: MCP, Plugins, Extensions, and Skills

How MCP servers, skills, and bundled packages are delivered into agent
platforms (Claude Code, Gemini CLI) — what works, what doesn't, and
where the gaps are.

**Date:** March 2026
**Status:** Reference — updated as platforms evolve

---

## What MCP Delivers

MCP (Model Context Protocol) is the open standard both platforms use
for tool integration. Here's what it provides and whether it's
sufficient for each use case.

| What MCP delivers | Enough? | Notes |
|---|---|---|
| Tools (model-invoked functions) | ✅ Yes | Core value of MCP. Agent decides when to call them. |
| Prompts (user-invoked templates) | ✅ Yes | Equivalent to user-invocable skills/slash commands. |
| Resources (on-demand data) | 🤷 Niche | Most use cases are better served by tools that return the same data. Designed more for IDE integrations than agentic workflows. |
| Server instructions (startup guidance) | ⚠️ Partial | Helps agent discover tools. Not workflow orchestration. 2KB limit in Claude. |
| **Model-invocable workflow instructions** | ❌ No | MCP cannot deliver instructions that the model auto-invokes based on context. This is the gap skills fill. |
| **Hooks / automation** | ❌ No | No MCP mechanism for "run this on session start" or "run this before/after a tool call." Platform-specific. |
| **Dependency auto-install** | ❌ No | No MCP mechanism for installing Python packages or other dependencies. Platform-specific. |

### The Key Distinction: Auto-Invocation

The critical gap in MCP is model-invocable instructions. Here's the
full picture of who triggers what:

| Mechanism | Who triggers it | What happens |
|---|---|---|
| **MCP Tool** | Model decides to call it | Runs a function, returns a result |
| **MCP Prompt** | User types a slash command | Injects a pre-written template into the conversation |
| **MCP Resource** | User or agent explicitly requests it | Returns data (like reading a file the server exposes) |
| **MCP Server Instructions** | Auto-loaded at startup | Passive guidance for tool discovery (not task instructions) |
| **Skill (user-invocable)** | User types a slash command | Injects instructions into the conversation |
| **Skill (model-invocable)** | Model decides it's relevant based on description | Injects instructions into the conversation |

MCP prompts and user-invocable skills are functionally identical from
the user's perspective — type a command, content gets injected. The
difference is where content lives (MCP server vs SKILL.md file) and
whether the model can auto-trigger it.

**Model-invocable skills are what MCP can't do.** There's no MCP
feature where a server says "here's a block of instructions — inject
these into the agent's context whenever the user's task matches this
description." Skills do this: the model reads the skill's `description`
field, decides "this is relevant right now," and loads the instructions
without the user asking.

---

## Platform Feature Comparison

### MCP Protocol Support

| Feature | What it means | Claude Code | Gemini CLI |
|---|---|---|---|
| **MCP Tools** | Functions the agent can call to do things (run queries, create files, etc.). The core MCP feature. | ✅ Full. Deferred loading — only tool names load at startup, full schemas fetched on demand. | ✅ Full. All tool schemas loaded at startup. |
| **MCP Resources** | Read-only data endpoints a server exposes (e.g., `docs://api-reference`). Data you pull in, not a function you call. | ✅ On-demand. Referenced via `@server:resource://path`. Not auto-loaded. | ✅ Auto-discovered. Show up in `/mcp` output. Referenceable in chat. |
| **MCP Prompts** | Pre-written prompt templates a server advertises. Become slash commands. User-invoked only — the agent cannot auto-trigger them. | ✅ Become `/mcp__server__prompt_name` commands. | ✅ Become slash commands. |
| **MCP Server Instructions** | Free-text string the server sends at startup. Passive guidance — tells the agent when to look for tools. | ✅ Loaded at startup. 2KB limit. | ❓ Not explicitly documented. May be supported. |
| **Sampling** | Server asks the agent's model to generate text on its behalf. Reverses the normal flow. | ❌ Not supported. | ❓ Not documented. Likely unsupported. |
| **Channels (push)** | Server pushes messages into the session without being asked. Event-driven (e.g., CI build results). | ✅ Requires `--channels` flag. | ❓ Not documented. |

### Packaging and Delivery

| Feature | What it means | Claude Code | Gemini CLI |
|---|---|---|---|
| **Plugins / Extensions** | Platform-specific bundle format that packages MCP servers, skills, hooks, and commands into one installable unit. | ✅ Plugins. `claude plugin install`. Decentralized marketplace (any git repo). | ✅ Extensions. `gemini extensions install`. No marketplace — git URL or local path. |
| **Bundled MCP servers** | Plugin/extension includes an MCP server that auto-starts when installed. | ✅ Via `.mcp.json` in the plugin. | ✅ Via MCP config in the extension. |
| **Bundled Skills** | Plugin/extension includes SKILL.md files installed alongside tools. | ✅ `skills/` directory. Namespaced as `/plugin:skill-name`. | ✅ `skills/` directory. |
| **Bundled Hooks** | Plugin/extension includes automation triggers (session start, before/after tool calls). | ✅ `hooks/hooks.json`. Rich: SessionStart, PreToolUse, PostToolUse, FileChanged, etc. | ✅ `hooks/hooks.json`. Can migrate Claude hooks via `gemini hooks migrate`. |
| **Bundled Slash Commands** | Plugin/extension includes custom user-typed commands. | ✅ `commands/` directory. | ✅ `.toml` files. |
| **Bundled Context** | Plugin/extension injects system-level instructions into every session. | ✅ Implicit via CLAUDE.md. | ✅ Explicit via GEMINI.md in the extension. |
| **Bundled Subagents** | Plugin/extension defines custom agent personas with their own prompts and tool restrictions. | ✅ `agents/` directory. | ❓ Not documented. |
| **Auto-install deps** | Plugin/extension handles installing its own dependencies without a separate install step. | ✅ SessionStart hook runs `pip install -t` into persistent storage. Diff-based — only reinstalls when requirements change. | ✅ Extensions handle their own install lifecycle. |

### Standalone Capabilities

| Feature | What it means | Claude Code | Gemini CLI |
|---|---|---|---|
| **Standalone Skills** | Skills installed without a plugin/extension. Just the SKILL.md. | ✅ Copy to `~/.claude/skills/` or install via aicfg. | ✅ `gemini skills install <source>`. |
| **Standalone Hooks** | Hooks configured without a plugin/extension. | ✅ Via `settings.json`. | ✅ Via settings or `gemini hooks`. |
| **Context Injection via MCP** | Can an MCP server automatically inject information into the conversation without user/agent action? | ❌ Server instructions are closest (passive, 2KB, startup only). Resources and prompts require explicit invocation. | ❌ Same limitations. Resources discoverable but not auto-injected. |

---

## Delivery Strategy

### For MCP server projects that need to deliver tools + skills + context

A single repo can target both platforms with minimal divergence:

```
my-project/
├── src/                        # SDK + MCP server code
├── pyproject.toml              # Package definition
│
├── .claude-plugin/             # Claude plugin packaging
│   └── plugin.json
├── .mcp.json                   # MCP server config (shared concept, platform-specific paths)
├── hooks/
│   └── hooks.json              # SessionStart for auto-install + any automation
├── skills/
│   └── my-workflow/
│       └── SKILL.md            # Model-invocable workflow guidance
├── CLAUDE.md                   # Claude context injection
├── GEMINI.md                   # Gemini context injection
└── requirements.txt            # For plugin self-install
```

### Installation tiers (in order of friction)

1. **Plugin/extension install** (zero friction): `claude plugin install` or `gemini extensions install` — tools, skills, hooks, context all appear immediately.
2. **Standalone skill install** (low friction): `aicfg skills install` or `gemini skills install` — gets the workflow guidance but not the MCP tools.
3. **Manual MCP registration** (medium friction): `pipx install <pkg>` then register stdio server in platform config. Tools work but no skills, hooks, or context.
4. **Pure MCP** (universal but limited): Register the server on any MCP-compatible platform. Gets tools and prompts. No skills, hooks, or auto-install.

### What should be a skill vs what should be a tool docstring

- **Tool docstring**: How to use THIS tool. Parameters, return values, edge cases. Self-contained.
- **Skill**: How to orchestrate ACROSS tools. Workflow guidance, design philosophy, decision frameworks, conventions that span multiple tools or even multiple MCP servers. The model auto-invokes this when the task matches.

A tool docstring should never need to reference other tools or explain a multi-step workflow. That's what skills are for.

---

## Open Questions

- Could MCP be extended to support model-invocable instructions? A server could advertise "context blocks" with trigger descriptions, similar to skill frontmatter. The agent would load them based on relevance.
- Could MCP resources serve as a delivery mechanism for skills? A server could expose `skills://my-workflow` as a resource, but the agent would still need to know when to fetch it — which is the auto-invocation problem.
- Is there appetite in the MCP spec community for a `context` or `instructions` capability richer than the current server instructions (which are limited, passive, and not task-scoped)?
