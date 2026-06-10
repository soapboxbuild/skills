---
name: soapbox-plugin-design
description: Use when building a plugin, tool, or data integration that needs to work across Soapbox's three AI platforms: Claude Code (MCP client), opencode (MCP client), and Paperclip (plugin-sdk). Covers the two-package architecture, MCP server format, config registration, and Paperclip plugin wrapper pattern.
---

# Soapbox Cross-Platform Plugin Design

## Overview

Soapbox runs three AI platforms with different plugin interfaces. A plugin that works across all three requires **two packages with shared business logic**:

1. **MCP server** — consumed by Claude Code and opencode (identical protocol, different config files)
2. **Paperclip plugin** — wraps the same logic using `@paperclipai/plugin-sdk`

Paperclip agents cannot consume external MCP servers; they only call tools registered via the plugin SDK's `ctx.tools.register()`. This is why two packages are needed.

---

## Package 1: MCP Server (Claude Code + opencode)

### Tech stack
- `@modelcontextprotocol/sdk` — the official MCP SDK
- `zod` — for input schemas
- Transport: `stdio` (most compatible) or HTTP

### Minimal server template

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({ name: "plugin-name", version: "1.0.0" });

server.tool(
  "tool_name",
  "Description of what this tool does and WHEN to call it",
  {
    param: z.string().describe("Description of param"),
  },
  async ({ param }) => {
    const result = yourBusinessLogic(param);
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

### Claude Code config (`~/.claude.json` → `mcpServers`)

```json
{
  "mcpServers": {
    "my-plugin": {
      "type": "stdio",
      "command": "node",
      "args": ["/path/to/dist/server.js"]
    }
  }
}
```

Or for npm-published package:
```json
{
  "mcpServers": {
    "my-plugin": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@soapbox/plugin-name"]
    }
  }
}
```

HTTP transport (for deployed servers):
```json
{
  "mcpServers": {
    "my-plugin": {
      "type": "http",
      "url": "https://your-plugin.soapbox.build/mcp",
      "headers": { "Authorization": "Bearer <token>" }
    }
  }
}
```

### opencode config (`opencode.json` → `"mcp"`)

```json
{
  "mcp": {
    "my-plugin": {
      "type": "local",
      "command": ["node", "/path/to/dist/server.js"]
    }
  }
}
```

The MCP server package is **identical** for both platforms. Only the config file format differs.

---

## Package 2: Paperclip Plugin

### Location
`packages/plugins/plugin-<name>/` in the Paperclip monorepo.

### Follows the `paperclip-email` pattern:
- `src/worker.ts` — calls `definePlugin` + `runWorker`
- `src/logic.ts` — pure business logic (shared with MCP server via extracted module)
- `package.json` with `@paperclipai/plugin-sdk` dependency
- `vitest.config.ts` for tests

### Minimal plugin template

```typescript
import { definePlugin, runWorker } from "@paperclipai/plugin-sdk";

const plugin = definePlugin({
  async setup(ctx) {
    ctx.tools.register("tool_name", {
      displayName: "Tool Name",
      description: "Description of what this tool does",
      parametersSchema: {
        type: "object",
        properties: {
          param: { type: "string", description: "Description" },
        },
        required: ["param"],
      }
    }, async (params) => {
      const { param } = params as { param: string };
      const result = yourBusinessLogic(param);
      return { content: JSON.stringify(result), data: result };
    });
  },
});

export default plugin;
runWorker(plugin, import.meta.url);
```

### Installation (Paperclip Railway volume)
Build `dist/`, copy to `~/.paperclip/plugins/<plugin-name>/`, install via Paperclip plugins API.

---

## Shared Business Logic

Extract data loading, lookup, and computation into a **third package or shared module** that both the MCP server and Paperclip plugin import:

```
packages/
  plugin-crrem-core/       ← pure TypeScript, no platform deps
    src/data/*.json        ← bundled datasets
    src/index.ts           ← exported functions
  plugin-crrem-mcp/        ← wraps core with MCP SDK
  plugin-crrem-paperclip/  ← wraps core with plugin-sdk
```

This keeps platform coupling minimal and business logic testable in isolation.

---

## Quick Reference

| Concern | Claude Code | opencode | Paperclip |
|---|---|---|---|
| Plugin protocol | MCP stdio/HTTP | MCP stdio | `@paperclipai/plugin-sdk` |
| Config location | `~/.claude.json` mcpServers | `opencode.json` mcp | `~/.paperclip/plugins/` |
| Tool definition | `server.tool(name, desc, zodSchema, fn)` | same | `ctx.tools.register(name, schema, fn)` |
| Can consume external MCP? | ✅ yes | ✅ yes | ❌ no |
| UI slots | ❌ | ❌ | ✅ via manifest |
| Webhooks/scheduled jobs | ❌ | ❌ | ✅ via ctx |

## Reference files
- MCP server example: `/home/claude/openwork/packages/openwork-ui-mcp/index.mjs`
- Paperclip plugin example: `/home/claude/paperclip/packages/plugins/paperclip-email/`
- Paperclip SDK docs: `/home/claude/paperclip/packages/plugins/sdk/README.md`
- Paperclip plugin spec: `/home/claude/paperclip/doc/plugins/PLUGIN_SPEC.md`
