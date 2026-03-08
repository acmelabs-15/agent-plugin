---
title: ANALYSIS-019-mcp-framework-and-file-watching
type: note
permalink: analysis/analysis-019-mcp-framework-and-file-watching
tags:
- analysis
- mcp
- fastmcp
- file-watching
- chokidar
- infrastructure
- agent-plugin
- bun
---

# ANALYSIS-019 MCP Server Framework and File Watching

## 1. Objective and Scope

**Objective**: Evaluate MCP server framework options (fastmcp vs @modelcontextprotocol/sdk) and file watching libraries (watcher, chokidar, @parcel/watcher, Bun built-in) for the @acmelabs-15/agent-plugin project.

**Scope**: API ergonomics, boilerplate reduction, transport support, Bun compatibility, bundle size, maintenance backing, cross-platform reliability, and recommendation for each category. Excludes runtime selection (covered in ANALYSIS-016).

## 2. Context

The @acmelabs-15/agent-plugin project embeds an MCP server exposing 14+ tools for AI assistants to manage plugins programmatically. The same core logic is shared between CLI and MCP server. The runtime is Bun (per ADR decision). File watching is needed only for the `dev` command (author workflow, not consumer-facing).

The MCP protocol reached specification version 2025-11-25. The ecosystem has stabilized with formal governance, a specification enhancement proposal (SEP) process, and the MCP Registry for server discovery. OpenAI adopted MCP in March 2025. The protocol is production-ready.

## 3. Approach

**Methodology**: Web research across npm registry API, GitHub API (via gh CLI), official documentation, community comparisons, and framework comparison sites.
**Tools Used**: WebSearch (20 queries), WebFetch (8 page analyses), npm registry API (10 queries), GitHub API (5 queries), Brain memory search (2 queries).
**Limitations**: npm.org blocks direct WebFetch (403). chokidar v5 Bun compatibility not verified firsthand. fastmcp Bun testing not verified firsthand. Bundle size data from npm registry metadata (unpacked size), not bundlephobia tree-shaken size.
**Data date**: 2026-03-07

## 4. Data and Analysis

### Part 1: MCP Server Framework

#### 4.1 fastmcp (TypeScript, by punkpeye)

| Metric | Value | Source | Confidence |
|--------|-------|--------|------------|
| GitHub stars | 2,980 | github.com/punkpeye/fastmcp (2026-03-07) | High |
| Forks | 253 | github.com/punkpeye/fastmcp (2026-03-07) | High |
| Open issues | 37 | github.com/punkpeye/fastmcp (2026-03-07) | High |
| Weekly downloads | 276,728 | npm API (week ending 2026-03-06) | High |
| Latest version | 3.34.0 | npm registry (published 2026-03-06) | High |
| Release cadence | 4 releases in 5 weeks (Jan-Mar 2026) | npm registry | High |
| License | MIT | npm registry | High |
| Dependencies | 14 direct (includes @modelcontextprotocol/sdk ^1.24.3) | package.json | High |
| npm dependents | 729 packages | npm | High |
| Maintainer | Frank Fiegel / punkpeye (glama.ai) | npm | High |
| Bun compatibility | Listed alongside Node.js and Express | GitHub README | Medium |
| Module type | ESM with CJS fallback (dual export) | package.json exports field | High |
| Created | 2024-12-27 | GitHub | High |

Key dependency: fastmcp wraps @modelcontextprotocol/sdk internally. It is not a standalone protocol implementation. It adds ergonomic API sugar on top of the official SDK.

**Tool definition (fastmcp)**:

```typescript
server.addTool({
  name: "add",
  description: "Add two numbers",
  parameters: z.object({ a: z.number(), b: z.number() }),
  execute: async (args) => String(args.a + args.b),
});
```

**Features beyond the official SDK**: Built-in authentication, session management (stateful and stateless modes), CORS (default on), progress notifications, streaming output, typed server events, prompt argument auto-completion, sampling, custom HTTP routes alongside MCP endpoints, edge deployment (EdgeFastMCP for Cloudflare Workers/Deno Deploy), OAuth discovery endpoints, Standard Schema support (Zod, ArkType, Valibot via xsschema), image and audio content handling, health checks, configurable ping behavior, roots management, CLI testing/debugging tools, Fuse.js-based fuzzy search.

#### 4.2 @modelcontextprotocol/sdk (Official Anthropic SDK)

| Metric | Value | Source | Confidence |
|--------|-------|--------|------------|
| GitHub stars | 11,782 | github.com/modelcontextprotocol/typescript-sdk (2026-03-07) | High |
| Forks | 1,663 | github.com/modelcontextprotocol/typescript-sdk (2026-03-07) | High |
| Open issues | 338 | github.com/modelcontextprotocol/typescript-sdk (2026-03-07) | High |
| Weekly downloads | 22,792,807 | npm API (week ending 2026-03-06) | High |
| Latest stable version | v1.27.1 | npm registry | High |
| v2 status | Pre-alpha, targeting Q1 2026 stable | ts.sdk.modelcontextprotocol.io | High |
| License | MIT | npm registry | High |
| Direct dependencies | 17 (including express, hono, cors, jose, ajv) | package.json | High |
| File count | 677 files in published package | npm registry | High |
| Peer dependency | zod (v3.25+ or v4) | npm registry | High |
| Bun compatibility | Officially listed as supported runtime | GitHub README | High |
| Maintenance | Active, 6 maintainers (Anthropic employees) | npm registry | High |
| Maintainers | jspahrsummers, pcarleton, fweinberger, thedsp, ashwin-ant, ochafik | npm | High |
| Created | 2024-09-24 | GitHub | High |
| Module type | ESM | package.json | High |

**Tool definition (official SDK)**:

```typescript
server.tool(
  "add",
  "Add two numbers",
  { a: z.number(), b: z.number() },
  async ({ a, b }) => ({
    content: [{ type: "text", text: String(a + b) }],
  })
);
```

Alternative verbose form: `server.registerTool("add", { title: "...", description: "...", inputSchema: z.object({...}) }, handler)`.

**Transport support**: stdio, Streamable HTTP, SSE. Framework middleware: Express, Hono, Node.js HTTP.

**Bun performance advantage**: Benchmarked at 13.4x faster startup (95ms vs 1,270ms Node.js), 61% less idle memory (18.7 MB vs 48.2 MB), 19% faster single operation, 39% faster concurrent operations (community benchmark by gorosun).

**v2 migration risk**: v1.x will receive bug fixes and security updates for 6+ months after v2 ships. v2 will "heavily change how the transport layer works" per maintainers.

#### 4.3 Other Frameworks Considered

| Framework | Type | Weekly Downloads | Notable Feature | Why Not |
|-----------|------|-----------------|-----------------|---------|
| mcp-framework | TypeScript (v0.2.18) | 58,420 | Convention-over-config, CLI scaffolding, auto-discovery | Extra abstraction; still v0.x; bundles 12 runtime deps |
| EasyMCP | TypeScript | Low | Express-like API, decorators | Experimental/beta, lacks SSE, missing sampling/notification |
| Cloudflare agents | TypeScript | N/A | Stateful edge agents, Durable Objects, actor model | Coupled to Cloudflare platform; not suitable for local CLI |

#### 4.4 Framework Comparison

| Criterion | @modelcontextprotocol/sdk | fastmcp |
|-----------|--------------------------|---------|
| **Weekly downloads** | 22.8M | 277K |
| **GitHub stars** | 11,782 | 2,980 |
| **Boilerplate** | Moderate (server.tool() is concise) | Less verbose (single-object addTool) |
| **Tool definition** | `server.tool(name, desc, schema, handler)` | `server.addTool({ name, params, execute })` |
| **Transport** | stdio, Streamable HTTP, SSE | stdio, HTTP streaming, SSE |
| **Session management** | Manual | Built-in (stateful + stateless modes) |
| **Authentication** | Manual (jose included for JWT) | Built-in (authenticate callback) |
| **Error handling** | Manual | Built-in patterns |
| **Bun compatibility** | Officially supported (documented) | Listed but unverified |
| **Direct dependencies** | 17 | 14 (+ SDK as transitive dep) |
| **Effective dep tree** | 17 deps | 17 + 14 deps (SDK included) |
| **Backing** | Anthropic (6 maintainers) | Individual maintainer (punkpeye/glama.ai) |
| **Protocol updates** | First to implement spec changes | Lags behind official SDK releases |
| **v2 migration** | Direct upgrade path | Must wait for punkpeye to update dependency |
| **Custom HTTP routes** | Use framework adapter (Express, Hono) | Built-in alongside MCP endpoints |
| **Edge deployment** | Not built-in | Built-in (EdgeFastMCP) |
| **Schema validation** | Zod only | Zod + ArkType + Valibot (Standard Schema) |
| **Module format** | ESM | ESM + CJS dual |

### Part 2: File Watching

#### 4.5 chokidar

| Metric | Value | Source | Confidence |
|--------|-------|--------|------------|
| Weekly downloads | 126,250,283 | npm API (week ending 2026-03-06) | High |
| GitHub stars | 11,949 | github.com/paulmillr/chokidar (2026-03-07) | High |
| Forks | 609 | GitHub (2026-03-07) | High |
| Open issues | 34 | GitHub (2026-03-07) | High |
| Latest version | v5.0.0 (ESM-only, Nov 2025) | npm registry | High |
| Min Node.js | v20.19.0 | package.json engines field | High |
| Dependencies | 1 (readdirp v5) | package.json | High |
| License | MIT | npm registry | High |
| Module type | ESM-only (v5) | package.json `"type": "module"` | High |
| Language | TypeScript | GitHub | High |
| Bun test script | `"test:bun1": "bun index.test.js"` present in package.json | npm registry | High |
| Created | 2012-04-20 | GitHub | High |

The presence of `"test:bun1": "bun index.test.js"` in chokidar v5's package.json scripts indicates the maintainer explicitly tests under Bun. This is a strong signal that Bun compatibility is intentional. The v5 rewrite eliminated all native dependencies (fsevents no longer bundled), removing the primary historical barrier to Bun compatibility.

#### 4.6 watcher (by Fabio Spampinato)

| Metric | Value | Source | Confidence |
|--------|-------|--------|------------|
| Weekly downloads | 23,584 | npm API (week ending 2026-03-06) | High |
| GitHub stars | 139 | github.com/fabiospampinato/watcher (2026-03-07) | High |
| Forks | 17 | GitHub (2026-03-07) | High |
| Open issues | 15 | GitHub (2026-03-07) | High |
| Latest version | 2.3.1 | npm registry | High |
| Last published | 2024-04-06 (23 months ago) | npm registry | High |
| Last GitHub update | 2026-02-12 | GitHub (2026-03-07) | High |
| License | MIT | npm registry | High |
| Dependencies | 3 (dettle, stubborn-fs, tiny-readdir) | package.json | High |
| Native dependencies | None (pure JavaScript) | GitHub README | High |
| Rename detection | Yes (unique feature) | GitHub README | High |
| Module type | ESM-only | package.json `"type": "module"` | High |
| Bun compatibility | Not mentioned; likely compatible (pure JS, no native deps) | GitHub README | Medium |
| Platform support | macOS (FSEvents), Windows, Linux (native recursive on Node 20.13+/22.1+) | GitHub README | High |

Pure JavaScript with zero native dependencies is the strongest Bun compatibility signal. The 23,584 weekly downloads and February 2026 GitHub activity indicate the project is maintained, though npm publishing has stalled since April 2024.

#### 4.7 @parcel/watcher

| Metric | Value | Source | Confidence |
|--------|-------|--------|------------|
| Weekly downloads | 21,235,960 | npm API (week ending 2026-03-06) | High |
| GitHub stars | 767 | github.com/parcel-bundler/watcher (2026-03-07) | High |
| Forks | 59 | GitHub (2026-03-07) | High |
| Open issues | 65 | GitHub (2026-03-07) | High |
| Latest version | 2.5.6 | npm registry | High |
| License | MIT | npm registry | High |
| Native implementation | C++ with node-gyp build, platform-specific prebuilds | npm registry | High |
| Dependencies | 4 (detect-libc, is-glob, node-addon-api, picomatch) | package.json | High |
| Platform support | macOS, Linux, Windows, FreeBSD | GitHub | High |
| Bun compatibility | Requires native N-API addon loading; Bun supports 95% of N-API but prebuilt binary loading may fail | Bun docs | Medium |
| WASM fallback | @parcel/watcher-wasm available but significantly less efficient | npm | High |

High performance but native C++ dependencies create cross-platform installation friction. Risk level too high for a CLI tool targeting broad developer audiences.

#### 4.8 Bun Built-in (node:fs.watch)

| Metric | Value | Source | Confidence |
|--------|-------|--------|------------|
| Implementation | Uses Node.js fs.watch API | Bun docs | High |
| Recursive watching | Supported via `recursive: true` option | Bun docs | High |
| Known bug | Newly created subdirectories not tracked (recursive mode) | GitHub issue #15939 | High |
| Known bug | Files created after watcher starts not detected | GitHub issue #15085 | High |
| Dependencies | None (built-in) | N/A | High |
| Cross-platform | macOS, Windows, Linux | Bun docs | High |
| Maturity | fs module at 92% Node.js test suite pass rate | Bun compat docs | High |

Two documented bugs with recursive watching make it unreliable for watching plugin source directories where files are created during development.

#### 4.9 File Watcher Comparison

| Criterion | chokidar v5 | watcher | @parcel/watcher | Bun fs.watch |
|-----------|-------------|---------|-----------------|-------------|
| **Weekly downloads** | 126.3M | 23.6K | 21.2M | Built-in |
| **GitHub stars** | 11,949 | 139 | 767 | N/A |
| **Bun compatibility** | Strong signal (bun test script) | Likely (pure JS) | Risky (native addon) | Native but buggy |
| **Native deps** | 0 (v5) | 0 | C++ prebuilds | N/A |
| **Recursive watch** | Yes | Yes (native) | Yes | Yes (buggy) |
| **Ignore patterns** | Yes (picomatch) | Yes (function/regex) | Yes (picomatch) | No |
| **Cross-platform** | Yes | Yes | Yes | Yes |
| **Maintenance** | Active (v5 Nov 2025) | GitHub active, npm stale | Active | Active (Bun team) |
| **Module format** | ESM-only | ESM-only | CJS | N/A |
| **Rename detection** | No | Yes (unique) | No | No |
| **Bundle impact** | Small (1 dep) | Small (3 small deps) | Large (native) | Zero |
| **TypeScript** | Written in TS | Types included | Types included | Built-in |

## 5. Results

### MCP Server Framework

fastmcp is a convenience wrapper around @modelcontextprotocol/sdk. Every fastmcp installation includes the full official SDK as a transitive dependency. The boilerplate reduction is modest: approximately 2-3 fewer lines per tool definition, totaling approximately 30 lines across 14 tools.

The official SDK explicitly lists Bun as a supported runtime. fastmcp's Bun compatibility is listed but unverified. The SDK's `server.tool()` shorthand narrows the API verbosity gap to near-parity with fastmcp's `server.addTool()`.

The SDK has 22.8M weekly downloads (82x more than fastmcp's 277K), 6 Anthropic maintainers, and a direct v2 migration path.

### File Watching

chokidar v5 eliminated all native dependencies and includes a `"test:bun1"` script in its package.json. This is the strongest Bun compatibility signal among the options. With 126M weekly downloads and 11,949 stars, it is the industry standard.

@parcel/watcher is disqualified (native C++ addon risk). Bun's built-in fs.watch is disqualified (recursive watching bugs). The watcher package (pure JS, 23.6K weekly downloads) is the fallback option.

## 6. Discussion

### MCP Framework: The Wrapper Tax

fastmcp adds a dependency layer between this project and the official MCP protocol. The tradeoff: zero dependency on an individual maintainer's release cadence versus saving 30 lines of boilerplate across 14 tools.

The official SDK's `server.tool()` shorthand (4 arguments) is nearly as concise as fastmcp's `server.addTool()` (single object). The verbosity gap has narrowed. The difference is cosmetic, not structural.

The embedded MCP server communicates via stdio (not remote HTTP). This eliminates fastmcp's main differentiators: built-in auth, session management, CORS, custom HTTP routes, and edge deployment. These features matter for remote MCP servers. They provide zero value for an embedded stdio server.

fastmcp's Bun compatibility is listed but unverified. The official SDK explicitly documents Bun support and community benchmarks confirm 13.4x faster startup under Bun. For a Bun-native project, using the framework that confirms Bun support reduces integration risk.

The SDK v2 migration path is direct: upgrade the dependency. With fastmcp, the migration path depends on punkpeye releasing a compatible version. This creates an uncontrolled dependency on a single maintainer's schedule.

Download volume tells the adoption story: 22.8M vs 277K weekly downloads. The SDK is 82x more widely used. Community support, bug reports, and ecosystem tooling all favor the official SDK.

### File Watching: chokidar v5 Changes the Calculus

chokidar v5 changes the Bun compatibility picture fundamentally:

1. Zero native dependencies (fsevents removed entirely)
2. ESM-only (aligns with Bun's module system)
3. `"test:bun1": "bun index.test.js"` in package.json (explicit Bun testing by maintainer)
4. Single dependency (readdirp v5, also pure JS)

These changes eliminate the primary historical barrier to Bun compatibility. The Bun test script in the official package.json is the strongest compatibility signal short of firsthand verification.

The watcher package remains a viable fallback. Pure-JS implementation, zero native dependencies. The 23.6K weekly downloads and February 2026 GitHub activity show ongoing maintenance, though npm publishing stalled since April 2024.

@parcel/watcher is disqualified. C++ native addon architecture creates installation friction inappropriate for a CLI tool targeting diverse developer environments.

Bun's built-in fs.watch is disqualified for recursive watching. Documented bugs with new file detection (#15085, #15939) directly conflict with the dev command's use case.

### Practical Path Forward

1. Install chokidar v5 and verify it works under Bun (estimated: 1 hour)
2. If it works: ship it. 126M weekly downloads, zero native deps, industry standard.
3. If it fails: install watcher (fabiospampinato). Pure JS, zero native deps, cross-platform.
4. Monitor Bun fs.watch bug fixes for potential future simplification to zero dependencies.

## 7. Recommendations

### MCP Server Framework

| Priority | Recommendation | Rationale | Effort |
|----------|---------------|-----------|--------|
| P0 | Use @modelcontextprotocol/sdk directly | Confirmed Bun support, no wrapper dependency, direct v2 upgrade path, Anthropic-backed, 82x more downloads | Low (14 tool definitions, ~30 extra lines vs fastmcp) |
| P1 | Use v1.x stable, plan v2 migration when stable | v1.x supported for 6+ months post-v2 launch | Deferred |
| P2 | Create thin internal abstraction over tool registration | Isolate tool definitions from SDK API changes during v1-to-v2 migration | Low |

### File Watching

| Priority | Recommendation | Rationale | Effort |
|----------|---------------|-----------|--------|
| P0 | Test chokidar v5 under Bun immediately | Industry standard, 0 native deps in v5, Bun test script in package.json | Low (1 hour test) |
| P1 | If chokidar fails: use watcher (fabiospampinato) | Pure JS, no native deps, cross-platform, ignore patterns | Low |
| P2 | If both fail: use Bun fs.watch with flat directory watching | Zero deps, but limited to single-level watching | Low |
| P2 | Monitor Bun fs.watch recursive bug fixes | Issues #15085 and #15939 may be resolved in future Bun releases | None (passive) |

## 8. Conclusion

**MCP Framework Verdict**: Use @modelcontextprotocol/sdk directly. [PASS]
**Confidence**: High
**Rationale**: The official SDK has confirmed Bun support, Anthropic backing (6 maintainers), direct protocol update path, 22.8M weekly downloads (82x more than fastmcp), and 11,782 GitHub stars. fastmcp wraps this same SDK with modest boilerplate savings (approximately 30 lines across 14 tools) that do not justify adding a single-maintainer dependency for an embedded stdio-based MCP server where auth, sessions, CORS, and HTTP routes provide zero value.

**File Watching Verdict**: Use chokidar v5, fall back to watcher. [PASS]
**Confidence**: Medium-High (Bun test script in package.json is strong signal; not firsthand verified)
**Rationale**: chokidar v5 eliminated all native dependencies, is ESM-only, and includes a Bun test script in its package.json. With 126M weekly downloads and 11,949 GitHub stars, it is the industry standard. If Bun compatibility fails, the watcher package (pure JS, zero native deps, 23.6K weekly downloads) is the immediate fallback.

### User Impact

- **What changes for you**: The MCP server uses the official SDK API directly. Tool definitions use `server.tool(name, desc, schema, handler)`. File watching in the `dev` command uses chokidar v5 (or watcher as fallback).
- **Effort required**: MCP framework choice adds approximately 30 lines vs fastmcp across 14 tools. File watching requires 1 hour of Bun compatibility testing before implementation.
- **Risk if ignored**: Using fastmcp creates a single-maintainer dependency with unverified Bun compatibility and no direct v2 migration path. Skipping file watcher testing risks broken `dev` command on one or more platforms.

## 9. Appendices

### Observations

- [fact] fastmcp v3.34.0 (published 2026-03-06) depends on @modelcontextprotocol/sdk ^1.24.3 as a transitive dependency; 276K weekly downloads, 2,980 GitHub stars #mcp #architecture
- [fact] @modelcontextprotocol/sdk v1.27.1 explicitly lists Bun as a supported runtime alongside Node.js and Deno; 22.8M weekly downloads, 11,782 GitHub stars, 6 Anthropic maintainers #bun #compatibility
- [fact] Official SDK v2 targets Q1 2026 with transport layer breaking changes; v1.x gets 6+ months of maintenance post-v2 launch #mcp #migration
- [risk] fastmcp is maintained by a single individual (punkpeye/glama.ai); protocol spec changes create a lag window where this project would be blocked #maintenance #risk
- [fact] chokidar v5.0.0 (Nov 2025) is ESM-only with 1 dependency (readdirp), 126.3M weekly downloads, 11,949 GitHub stars; package.json contains "test:bun1" script indicating explicit Bun testing #file-watching #bun
- [fact] Bun fs.watch has documented bugs: new files not detected in recursive mode (issues #15085, #15939); fs module at 92% Node.js test suite pass rate #bun #bug
- [fact] @parcel/watcher 2.5.6 uses native C++ addon via node-gyp; Bun implements 95% of N-API but prebuilt binary loading is not guaranteed #bun #compatibility
- [insight] For an embedded stdio-based MCP server, fastmcp's main differentiators (auth, sessions, CORS, HTTP routes, edge deployment) provide zero value; boilerplate difference is approximately 30 lines across 14 tools #architecture
- [fact] watcher 2.3.1 (last npm publish Apr 2024) has 23.6K weekly downloads and 139 GitHub stars; pure JavaScript with zero native deps; GitHub shows activity as recent as Feb 2026 #file-watching
- [fact] MCP protocol stabilized at spec 2025-11-25 with formal governance and SEP process; OpenAI adopted MCP in March 2025 #mcp #ecosystem
- [fact] Bun MCP server benchmarks show 13.4x faster startup (95ms vs 1,270ms), 61% less idle memory (18.7MB vs 48.2MB) compared to Node.js #bun #performance
- [decision] Recommend @modelcontextprotocol/sdk over fastmcp for embedded stdio MCP server: confirmed Bun support, no wrapper dependency, direct v2 upgrade path, 82x more weekly downloads #mcp #recommendation
- [decision] Recommend chokidar v5 for file watching with watcher as fallback: Bun test script in package.json, zero native deps, 126M weekly downloads, industry standard #file-watching #recommendation

### Relations

- relates_to [[ANALYSIS-016-bun-runtime-assessment]]
- relates_to [[ADR-001-plugin-format-and-manifest]]
- relates_to [[ADR-002-target-platforms-and-audiences]]

### Sources Consulted

- [fastmcp GitHub](https://github.com/punkpeye/fastmcp) - Stars (2,980), features, dependencies, README
- [fastmcp npm registry API](https://registry.npmjs.org/fastmcp/latest) - Version 3.34.0, deps, license
- [fastmcp npm downloads](https://api.npmjs.org/downloads/point/last-week/fastmcp) - 276,728/week
- [@modelcontextprotocol/sdk GitHub](https://github.com/modelcontextprotocol/typescript-sdk) - Stars (11,782), Bun support, v2 timeline
- [@modelcontextprotocol/sdk npm registry API](https://registry.npmjs.org/@modelcontextprotocol/sdk/latest) - Version 1.27.1, 17 deps, 6 maintainers
- [@modelcontextprotocol/sdk npm downloads](https://api.npmjs.org/downloads/point/last-week/@modelcontextprotocol/sdk) - 22,792,807/week
- [MCP Specification 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25) - Protocol status
- [MCPVerified Framework Comparison](https://mcpverified.com/server-frameworks/comparison) - Feature matrix across 4 frameworks
- [Building MCP Servers with Bun](https://dev.to/gorosun/building-high-performance-mcp-servers-with-bun-a-complete-guide-32nj) - Bun performance benchmarks
- [chokidar GitHub](https://github.com/paulmillr/chokidar) - Stars (11,949), v5 features, "test:bun1" script
- [chokidar npm registry API](https://registry.npmjs.org/chokidar/latest) - v5.0.0, 1 dep (readdirp), engines >=20.19.0
- [chokidar npm downloads](https://api.npmjs.org/downloads/point/last-week/chokidar) - 126,250,283/week
- [watcher GitHub](https://github.com/fabiospampinato/watcher) - Stars (139), features, platform support, last activity Feb 2026
- [watcher npm registry API](https://registry.npmjs.org/watcher/latest) - v2.3.1, 3 deps, last publish Apr 2024
- [watcher npm downloads](https://api.npmjs.org/downloads/point/last-week/watcher) - 23,584/week
- [@parcel/watcher GitHub](https://github.com/parcel-bundler/watcher) - Stars (767), C++ native addon
- [@parcel/watcher npm registry API](https://registry.npmjs.org/@parcel/watcher/latest) - v2.5.6, node-gyp build
- [@parcel/watcher npm downloads](https://api.npmjs.org/downloads/point/last-week/@parcel/watcher) - 21,235,960/week
- [Bun fs.watch docs](https://bun.com/docs/guides/read-file/watch) - API surface, recursive support
- [Bun Node-API docs](https://bun.com/docs/runtime/node-api) - 95% N-API implementation
- [Bun fs.watch recursive bug #15939](https://github.com/oven-sh/bun/issues/15939) - New subdirectories not tracked
- [Bun fs.watch bug #15085](https://github.com/oven-sh/bun/issues/15085) - Files created after watcher starts not detected
- [mcp-framework npm](https://registry.npmjs.org/mcp-framework/latest) - v0.2.18, 58,420 downloads/week

### Data Transparency

- **Found**: GitHub metrics for all 5 packages via API (2026-03-07), npm weekly downloads via API for all packages, dependency trees from npm registry, Bun compat status for official SDK (confirmed), chokidar Bun test script in package.json, fastmcp license (MIT), MCP spec version and governance status, Bun N-API coverage (95%), Bun MCP server benchmarks, mcp-framework metadata
- **Not Found**: chokidar v5 Bun test pass/fail results (script exists but no published results), fastmcp Bun test results (listed as compatible but no test evidence), @parcel/watcher Bun prebuilt binary loading status under current Bun version, watcher Bun compatibility test results
