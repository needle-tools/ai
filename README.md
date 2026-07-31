# Needle AI Plugins

AI provider plugins for [Needle Engine](https://needle.tools) — a web-first 3D engine built on Three.js.

## Agent Skill

Install the Needle Engine skill for any supported AI coding agent:

```bash
npx skills add needle-tools/ai
```

This works with Claude Code, Cursor, GitHub Copilot, Codex, Gemini CLI, Windsurf, and [40+ other agents](https://github.com/vercel-labs/skills).

> **Already using `@needle-tools/engine`?** The Vite plugin auto-installs the skill for detected agents — no manual setup needed.

## Providers

### Claude Code

The Claude Code plugin provides:

- **MCP server** — documentation search (`needle_search`) and user/project tools via `npx needle-cloud mcp`
- **Needle Engine skill** — component lifecycle, serialization, input, physics, networking, WebXR, deployment, and progressive loading reference

Agents without the MCP server can search the same corpus over HTTP — no key required:

```bash
curl -s "https://search.needle.tools/api/semantic-search?q=how+to+add+a+rigidbody&limit=5"
```

See [`references/mcp.md`](providers/claude/plugin/skills/needle-engine/references/mcp.md) for the full MCP tool inventory and search API reference.

#### Install

```
/plugin install needle-engine
```

Or load locally for development:

```bash
claude --plugin-dir ./providers/claude/plugin
```

#### Structure

```
providers/claude/plugin/
├── .claude-plugin/plugin.json   # Plugin metadata
├── .mcp.json                    # MCP server config (npx needle-cloud mcp)
└── skills/needle-engine/
    ├── SKILL.md                 # Needle Engine skill
    ├── references/              # Loaded on demand (api, physics, networking, xr, mcp, ...)
    ├── scripts/lookup-api.mjs   # Search the installed package's .d.ts files
    └── templates/               # Component starting points
```

## Links

- [Needle Engine Docs](https://engine.needle.tools/docs/)
- [Needle Cloud](https://cloud.needle.tools)
- [needle-cloud on npm](https://www.npmjs.com/package/needle-cloud)
- [Claude Code Plugin Docs](https://code.claude.com/docs/en/plugins)
