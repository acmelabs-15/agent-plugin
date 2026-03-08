---
title: ANALYSIS-032 MCP SDK Bun Runtime Compatibility
type: analysis
permalink: analysis/analysis-032-mcp-sdk-bun-runtime-compatibility-1
tags:
- mcp-sdk
- bun
- runtime-compatibility
- zod
- transport
---

# ANALYSIS-032 MCP SDK Bun Runtime Compatibility

## Objective

Verify whether `@modelcontextprotocol/sdk` runs 100% on Bun runtime with zero Node.js dependency.

## Verdict

**Compatible.** The MCP TypeScript SDK works on Bun without Node.js. All Node.js built-in imports (`node:process`, `node:stream`, `node:child_process`, `Buffer`) are within Bun's compatibility layer. Zero native/binary dependencies exist.

## SDK Versions Assessed

| Version | Package | Status |
|---------|---------|--------|
| v1.27.1 | `@modelcontextprotocol/sdk` | Current stable |
| v2.0.0-alpha | `@modelcontextprotocol/server` + `client` + `core` | Development on main |

## Node.js Module Usage and Bun Support

| Module | Used By | Bun Status |
|--------|---------|------------|
| `node:process` | StdioServerTransport | [PASS] Fully supported |
| `node:stream` | StdioServerTransport (Readable, Writable) | [PASS] Fully implemented |
| `node:child_process` | StdioClientTransport only | [PASS] Minor gaps (proc.gid/uid) not used by SDK |
| `node:stream/web` | StreamableHTTPClientTransport (type-only) | [PASS] No runtime effect |
| `Buffer` global | ReadBuffer (core shared stdio) | [PASS] Fully implemented |

## Transport Compatibility

| Transport | Node.js APIs | Bun Verdict |
|-----------|-------------|-------------|
| StdioServerTransport | `node:process`, `node:stream` | [PASS] Use for stdio MCP servers |
| WebStandardStreamableHTTPServerTransport (v2) | Zero Node.js imports | [PASS] Recommended for HTTP servers |
| StdioClientTransport | `node:child_process`, `node:process`, `node:stream` | [PASS] Standard use cases work |
| StreamableHTTPClientTransport | `node:stream/web` (type-only) | [PASS] Web Standard APIs |
| WebSocketClientTransport (v2) | Zero Node.js imports | [PASS] Standard WebSocket |
| Hono Middleware | Zero Node.js imports | [PASS] Built for Web Standard runtimes |
| Node Middleware | `node:http` | N/A -- Do NOT use on Bun. Use Hono instead. |

## Zod v4 Compatibility

Zod v4 `toJSONSchema()` is compatible with MCP SDK tool input schemas:

- Both target JSON Schema Draft 2020-12 by default
- SDK v1.23.0+ includes `zod-compat` layer handling Zod v4 natively
- SDK peer dependency: `"zod": "^3.25 || ^4.0"`
- All known breaking issues resolved by December 2025

### Minimum Version Matrix

| Package | Minimum | Recommended |
|---------|---------|-------------|
| `@modelcontextprotocol/sdk` | 1.23.0 | 1.27.1 |
| `zod` | 4.1.13 | 4.3.6 |

## Resolved GitHub Issues

| Issue | Description | Status |
|-------|-------------|--------|
| #91 | Bun StdioServerTransport | [PASS] Fixed |
| #201 | SSEServerTransport on Bun | [PASS] Fixed (Bun 1.2.6) |
| #348 | Buffer import for Deno/Bun | [PASS] Merged |
| #709 | Bun compilation failure | [PASS] Import path issue |
| #1269 | Non-Node environment support | v2 architecture is the solution |

## Dependencies (All Pure JS)

zod, ajv, ajv-formats, cross-spawn, eventsource, eventsource-parser, jose, pkce-challenge, hono. Zero C++ addons, zero node-gyp builds.

## Recommendations for agent-plugin

1. Use `StdioServerTransport` for plugin MCP servers (standard stdio communication)
2. Use `Bun.serve()` + Hono middleware for any HTTP-based MCP servers
3. Do NOT import `@modelcontextprotocol/node` -- use `@modelcontextprotocol/hono` instead
4. Pin SDK >= 1.23.0 and Zod >= 4.1.13 in package.json
5. The HTML eval viewer should use `Bun.serve()` directly (no Express, no Node http)

## Observations

- [fact] MCP TypeScript SDK works on Bun with zero Node.js runtime dependency #bun #mcp-sdk
- [fact] All Node.js built-in imports used by SDK are within Bun's compatibility layer #runtime-compatibility
- [fact] Zod v4 toJSONSchema() compatible with MCP SDK tool input schemas, both target JSON Schema Draft 2020-12 #zod #json-schema
- [fact] SDK minimum versions for Zod v4 support: SDK >= 1.23.0, Zod >= 4.1.13 #version-requirements
- [fact] Zero native/binary dependencies in MCP SDK -- all pure JavaScript #dependencies
- [decision] Use StdioServerTransport for plugin MCP servers and Bun.serve() + Hono for HTTP #transport-selection
- [constraint] Do NOT use @modelcontextprotocol/node package -- use @modelcontextprotocol/hono instead #bun-only
- [insight] SDK v2 adds explicit multi-runtime support via export condition shims and Web Standard transports #future-compat

## Relations

- relates_to [[ADR-006 Core Dependency Stack]]
- relates_to [[ADR-012 Scaffolding and Content Management]]
- relates_to [[ANALYSIS-019 MCP Framework and File Watching]]
- relates_to [[ADR-005 Runtime and Distribution Strategy]]
- relates_to [[ADR-011 Auto-Generated CLI from MCP Tools]]