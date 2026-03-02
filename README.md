# Needle AI Plugins

AI provider plugins for [Needle Engine](https://needle.tools) — a web-first 3D engine built on Three.js.

## Providers

### Claude Code

The Claude Code plugin provides:

- **MCP server** — documentation search (`needle_search`) and user/project tools via `npx needle-cloud mcp`
- **Needle Engine skill** — component lifecycle, serialization, input, physics, networking, WebXR, deployment, and progressive loading reference

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
    └── SKILL.md                 # Needle Engine skill
```

## Links

- [Needle Engine Docs](https://engine.needle.tools/docs/)
- [Needle Cloud](https://cloud.needle.tools)
- [needle-cloud on npm](https://www.npmjs.com/package/needle-cloud)
- [Claude Code Plugin Docs](https://code.claude.com/docs/en/plugins)
