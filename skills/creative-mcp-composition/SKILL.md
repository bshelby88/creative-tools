---
name: creative-mcp-composition
description: "Compose multiple creative MCP servers using @openrouter/mcp patterns for multi-domain creative workflows."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [mcp, composition, creative, multi-server, openrouter, elicitation, caching]
---

# Creative MCP Composition

Compose multiple creative MCP servers into a single agent workflow using patterns from `@openrouter/mcp`.

## Architecture

```
Hermes Agent
  ├── @openrouter/mcp (composition layer)
  │   ├── Design MCP (figma, canva, sketch)
  │   ├── Generation MCP (comfy-cloud, hugging_face, gamma)
  │   ├── Asset MCP (cloudinary, miro, notion)
  │   └── Runtime MCP (touchdesigner-mcp, unreal-engine)
  └── Creative Skills
      ├── baoyu-infographic
      ├── design-md
      ├── sketch
      └── touchdesigner-mcp
```

## Multi-Server Composition Pattern

Spread MCP tools from multiple creative domains into a single agent call:

```typescript
import { createMCPTools } from '@openrouter/mcp';

const design = await createMCPTools({
  url: 'https://mcp.figma.com/mcp',
  auth: { kind: 'oauth', provider: figmaProvider },
  toolNamePrefix: 'design_',
});

const generation = await createMCPTools({
  url: 'https://mcp.comfy.com/mcp',
  auth: { kind: 'bearer', token: process.env.COMFY_API_KEY },
  toolNamePrefix: 'gen_',
});

const assets = await createMCPTools({
  url: 'https://mcp.cloudinary.com/mcp',
  auth: { kind: 'bearer', token: process.env.CLOUDINARY_URL },
  toolNamePrefix: 'asset_',
});

// Single agent call with all creative tools
const result = callModel(client, {
  model: 'anthropic/claude-opus-4-8',
  input: 'Design and generate a cyberpunk infographic, then upload to Cloudinary',
  tools: [...design.tools, ...generation.tools, ...assets.tools],
});
```

## Caching & Rehydration

For cold-start creative deployments (serverless, edge):

```typescript
import { createMCPTools, InMemoryMCPCacheStore, rehydrateMCPTools } from '@openrouter/mcp';

const store = new InMemoryMCPCacheStore();

const mcp = await createMCPTools({
  url: 'https://mcp.comfy.com/mcp',
  auth: { kind: 'bearer', token: apiKey },
  cache: { store, key: 'mcp:creative:comfy' },
  staleness: { maxAgeMs: 60 * 60 * 1000 }, // 1 hour cache
  cacheCredentials: true,
});

// Serialize for persistence
const snapshot = await mcp.serialize();
await mcp.close();

// Rehydrate without network round-trip
const mcp2 = await rehydrateMCPTools({ snapshot, auth: { kind: 'bearer', token: apiKey } });
```

## Elicitation for Creative Decisions

Handle structured creative input (e.g., "choose between 3 designs"):

```typescript
const mcp = await createMCPTools({
  url: 'https://mcp.figma.com/mcp',
  auth: { kind: 'oauth', provider: figmaProvider },
  onElicitation: async ({ message, requestedSchema }) => {
    // Present creative options to the user
    const answer = await promptCreativeChoice(message, requestedSchema);
    return { action: 'accept', content: answer };
  },
});
```

This is especially useful for:
- **Design variants**: Present 3 layout options, user picks one
- **Color/themes**: Elicit brand preferences
- **Aspect ratios**: Let user choose between landscape/portrait/square
- **Style selection**: Present style gallery for infographic generation

## Progress Streaming

For long creative operations (rendering, video generation):

```typescript
for await (const event of result.getToolStream()) {
  if (event.type === 'tool.progress' && event.source === 'mcp') {
    console.log(`[${event.toolName}] ${event.progress}`);
  }
}
```

## Hermes Native Implementation

In Hermes, use `hermes config set` for each server, then compose at runtime:

```bash
# All creative servers are configured via hermes config
# Hermes automatically discovers and loads them
hermes mcp list  # Verify all creative servers
```

The `onElicitation` pattern maps to Hermes' `clarify` tool for creative decisions.
The `toolNamePrefix` pattern maps to Hermes' `source` discriminant (client vs mcp).
The `MCPCacheStore` pattern maps to Hermes' persistent state/sessions.

## References

- `references/creative-mcp-catalog.md` — Full server catalog with URLs, auth patterns, and configuration commands

## Creative Server Inventory

| Server | Domain | Transport | Auth | Purpose |
|--------|--------|-----------|------|---------|
| comfy-cloud | AI Generation | HTTP | Bearer | Image/video/audio/3D generation |
| figma | Design | HTTP | OAuth | Design collaboration, Code Connect |
| canva | Design | HTTP | API Key | Design management, templates |
| cloudinary | Assets | HTTP | API Key | Media upload, transform, search |
| gamma | Presentations | HTTP | API Key | AI presentations/docs/sites |
| hugging_face | AI Models | HTTP | API Key | Models, datasets, Spaces |
| miro | Visual Collab | HTTP | OAuth | Boards, diagrams, frames |
| unreal-engine | 3D/Realtime | Stdio | None | UE5 editor control |
| notion | Documentation | HTTP | OAuth | Pages, databases |
| webflow | CMS | HTTP | OAuth | Sites, CMS collections |
| touchdesigner-mcp | Creative Coding | Streamable HTTP | None | TouchDesigner control |
