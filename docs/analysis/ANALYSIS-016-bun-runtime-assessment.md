---
title: ANALYSIS-016 Bun Runtime Assessment
type: note
permalink: analysis/analysis-016-bun-runtime-assessment
tags:
- analysis
- bun
- runtime
- cli
- cross-platform
- typescript
- agent-plugin
---

# ANALYSIS-016 Bun Runtime Assessment

## 1. Objective and Scope

**Objective**: Evaluate Bun as the runtime for `@acmelabs-15/agent-plugin`, a cross-platform CLI tool that manages AI agent plugins across 7 coding platforms.

**Scope**: Bun maturity, performance (especially CLI startup), cross-platform support, native APIs, npm compatibility, distribution strategies, community adoption, and risks compared to Node.js and Deno. Excludes framework-level decisions (gunshi, etc.) except for Bun compatibility.

## 2. Context

The design spec proposes Bun as the runtime. The tool is a CLI + embedded MCP server published as `@acmelabs-15/agent-plugin` on npm. It uses TypeScript strict mode with dependencies: gunshi (CLI framework), @clack/prompts, zod v4, deepmerge, gray-matter. It must work on Windows, macOS, and Linux.

In December 2025, Anthropic acquired Oven (the company behind Bun). Claude Code ships as a Bun-compiled binary to millions of developers. This acquisition fundamentally changes the risk calculus for Bun adoption.

## 3. Approach

**Methodology**: Web research across Bun documentation, benchmark articles, GitHub issues, npm registry, community discussions, and real-world migration case studies.
**Tools Used**: WebSearch (12 queries), WebFetch (3 page analyses), Brain memory search (2 queries), codebase file reading.
**Limitations**: Could not run benchmarks directly. Some benchmark numbers are from third-party articles and may vary by hardware. Bun version numbering has reached 1.3.x (no 2.0 release as of March 2026).

## 4. Data and Analysis

### 4.1 Bun Maturity (March 2026)

| Metric | Value | Source |
|--------|-------|--------|
| Current version | 1.3.x (Bun 1.3 "biggest release yet") | bun.com/blog |
| Owner | Anthropic (acquired Oven, Dec 2025) | anthropic.com/news |
| License | MIT (confirmed remaining open source) | bun.com/blog/bun-joins-anthropic |
| Production user | Claude Code ($1B ARR, millions of users) | Anthropic announcement |
| Node.js test suite pass rate (fs) | 92% | bun.com/docs/runtime/nodejs-compat |
| Node.js test suite pass rate (path, os, events, querystring) | 100% each | bun.com/docs/runtime/nodejs-compat |

### 4.2 CLI Startup Performance

| Runtime | Cold Start Time | Source |
|---------|----------------|--------|
| Bun 1.3 | 8-15ms | Multiple benchmark articles (dev.to, Medium, 2025-2026) |
| Node.js 24 | 40-120ms | Same sources |
| Deno 2.x | 40-60ms | dev.to runtime comparisons |

Bun starts 4-8x faster than Node.js for CLI invocations. For a CLI tool invoked frequently, this difference is perceptible. A 100ms+ startup adds latency that users feel on every command.

The Tigris CLI case study (real-world migration) measured 0.064s for Node.js vs 0.104s for the Bun compiled binary on Apple M4 Max. The compiled binary was slightly slower than Node.js in startup, but the team deemed the 40ms difference negligible. This contradicts generic benchmark claims and suggests compiled binaries have different startup characteristics than `bun run`.

### 4.3 Bun vs Node.js for CLI Tools

| Factor | Bun | Node.js |
|--------|-----|---------|
| TypeScript execution | Native, no build step | Requires tsc/tsx/esbuild |
| Package installation | 2-3x faster than npm | npm/pnpm/yarn mature |
| Startup time | 8-15ms | 40-120ms |
| Node API compat | >95% | 100% (native) |
| Ecosystem maturity | Growing, Anthropic-backed | 15+ years, battle-tested |
| Binary compilation | Built-in (bun build --compile) | Requires pkg/nexe/sea |
| Test runner | Built-in (bun test) | Jest/Vitest/node --test |
| Native TypeScript | Yes (JSC-based) | Yes (--strip-types in Node 22+, experimental) |

### 4.4 Bun vs Deno 2.x

| Factor | Bun | Deno 2.x |
|--------|-----|----------|
| Startup | 8-15ms | 40-60ms |
| npm compatibility | ~95% | Full (Deno 2 removed barriers) |
| Security sandbox | None | Built-in permission system |
| Built-in tooling | Runtime + bundler + test runner + pkg mgr | Runtime + linter + formatter + test runner + REPL |
| TypeScript | Native | Native |
| Corporate backing | Anthropic ($10B+ valuation) | Deno Company |
| Ecosystem direction | Performance-first, AI tooling | Security-first, web standards |

Deno's security sandbox is irrelevant for this use case (a plugin manager needs file system access by design). Bun's faster startup gives it the edge for CLI tools.

### 4.5 Cross-Platform Compatibility

| Platform | Status | Known Issues |
|----------|--------|-------------|
| macOS (ARM64) | Stable | Code signing requires JIT entitlements |
| macOS (x64) | Stable | None documented |
| Linux (x64) | Stable | None documented |
| Linux (ARM64) | Stable | None documented |
| Windows (x64) | Stable | Minor UI bugs in `bun init` (fixed in 1.2.4) |
| Windows (ARM64) | Unstable | Fails to run, open tracking issue |

Cross-compilation supported via `--target` flag: can build Linux/Windows binaries from macOS and vice versa. The Windows ARM64 gap is a low-risk issue since most Windows development happens on x64. However, it should be monitored.

### 4.6 Bun Native APIs for CLI Tools

| API | Purpose | Stability | Advantage over Node.js |
|-----|---------|-----------|----------------------|
| Bun.file() | Lazy file reading | Stable (since 1.0) | Multiple output formats (.text(), .json(), .stream()) |
| Bun.write() | Atomic file writes | Stable | Auto-creates parent directories |
| Bun.Glob | File/string pattern matching | Stable (since 1.0.14) | 3x faster than fast-glob/micromatch |
| Bun.spawn / Bun.spawnSync | Process execution | Stable | Async streaming + sync modes |
| Bun.serve() | HTTP server | Stable | Built-in, no Express needed (for MCP server) |

These APIs are production-stable. They simplify CLI tool development by eliminating dependencies (no need for `glob`, `fast-glob`, or `express`).

### 4.7 Node.js Built-in Module Compatibility

| Module | Status | Notes |
|--------|--------|-------|
| fs | Fully implemented | 92% of Node.js test suite passes |
| path | Fully implemented | 100% test suite pass |
| crypto | Partially implemented | Missing secureHeapUsed, setEngine, setFips |
| child_process | Partially implemented | Missing proc.gid, proc.uid. IPC limitations. |
| os | Fully implemented | 100% test suite pass |
| net | Fully implemented | Complete |
| http | Fully implemented | Outgoing body buffered, not streamed |
| stream | Fully implemented | Complete |
| events | Fully implemented | 100% test suite pass |
| buffer | Fully implemented | Complete |
| url | Fully implemented | Complete |

For this project's needs (file operations, path manipulation, process spawning, HTTP serving), the required modules are fully or nearly fully implemented. The crypto gaps (secureHeapUsed, setEngine) are irrelevant for a plugin manager.

### 4.8 Key Dependency Compatibility

| Dependency | Bun Compatible | Notes |
|-----------|---------------|-------|
| gunshi (CLI framework) | Yes | Explicitly supports Bun. Install via `bun add gunshi`. Active development (v0.29.2, March 2026). |
| @clack/prompts | Partial | Working demos exist (Medium, Aug 2025). Historical issues with multiple sequential text prompts (GitHub issues #4835, #3099, #7033). Latest v1.1.0 (March 2026). Needs testing. |
| zod v4 | Yes | Pure TypeScript, no native dependencies |
| deepmerge | Yes | Pure JavaScript, no native dependencies |
| gray-matter | Yes | Pure JavaScript, no native dependencies |

The @clack/prompts compatibility is the highest-risk dependency. Multiple GitHub issues document terminal interaction breaks when chaining prompts in Bun. The issues may be resolved in Bun 1.3.x, but verification is required before committing.

### 4.9 Distribution Options

**Option A: npm package with Bun shebang**

- Users install via `npm install -g @acmelabs-15/agent-plugin`
- Entry point uses `#!/usr/bin/env bun` shebang
- Requires Bun installed on user's machine
- Risk: forces users to install Bun

**Option B: npm package with Node.js shebang (develop with Bun)**

- Users install via `npm install -g @acmelabs-15/agent-plugin`
- Entry point uses `#!/usr/bin/env node` shebang
- Requires TypeScript compilation step for distribution
- Works with Node.js OR Bun (bun run respects node shebang by default)
- Risk: loses Bun-native API advantages, needs build step

**Option C: Compiled standalone binary**

- `bun build --compile` produces single executable
- No runtime dependency (no Bun, no Node.js required)
- Cross-compile targets: macOS (x64/arm64), Linux (x64/arm64), Windows (x64)
- Binary size: ~50-60 MB per platform
- Risk: large binary size, no `npm install` workflow

**Option D: Dual distribution (npm + binary)**

- npm package for `npx`/`bunx` usage (Option B)
- Compiled binaries for standalone installation (Option C)
- Maximum reach, higher maintenance cost

The Tigris CLI case study demonstrates Option D successfully: npm distribution for ecosystem integration, compiled binary for environments without Node.js.

### 4.10 Binary Size

| Target | Approximate Size |
|--------|-----------------|
| macOS ARM64 | ~51 MB |
| macOS x64 | ~51 MB |
| Linux x64 | ~57 MB |
| Windows x64 | ~100 MB |

Bun acknowledges "Bun's binary is still way too big." Open issues track size reduction. For comparison, Go CLI binaries are typically 5-15 MB. The size is acceptable for a developer tool but not ideal.

### 4.11 Community Adoption

| Project | Significance |
|---------|-------------|
| Claude Code (Anthropic) | Ships as Bun binary. $1B ARR. Millions of users. |
| Tigris CLI | Migrated from Node.js to Bun. Documented case study with benchmarks. |
| Bunli | Complete CLI development ecosystem built for Bun. |
| Multiple CLI tutorials/guides (2025-2026) | Growing body of Bun CLI development knowledge. |

Claude Code is the strongest validation signal. It proves Bun can power a cross-platform CLI tool distributed to millions of developers on macOS, Linux, and Windows.

## 5. Results

### Evidence Table

| Finding | Source | Confidence |
|---------|--------|------------|
| Bun starts 4-8x faster than Node.js for CLI invocations | Multiple benchmark articles | High |
| Compiled Bun binary startup is comparable to Node.js (not faster) | Tigris CLI case study | Medium (single case study) |
| Bun Windows ARM64 support is broken | GitHub issue tracking | High |
| @clack/prompts has documented Bun compatibility issues | GitHub issues #4835, #3099, #7033 | High (but may be fixed in 1.3.x) |
| Anthropic owns Bun and ships Claude Code on it | Anthropic announcement | High |
| Bun binary size is 50-100 MB per platform | Bun docs, Tigris case study | High |
| gunshi explicitly supports Bun | gunshi.dev documentation | High |
| Node.js API compatibility is >95% | Bun documentation | High |
| Bun native TypeScript execution eliminates build step | Bun documentation | High |

### Facts (Verified)

- Bun 1.3.x is the current stable release as of March 2026
- Anthropic acquired Oven (Bun's parent company) in December 2025
- Claude Code ships as a Bun-compiled binary to millions of developers
- Bun natively executes TypeScript without a compilation step
- `bun build --compile` produces standalone binaries for macOS, Linux, and Windows (x64 and ARM64 except Windows ARM64)
- Core Node.js modules (fs, path, os, events, stream, buffer, url, net, http) are fully implemented
- crypto and child_process are partially implemented with gaps in edge-case APIs
- gunshi v0.29.2 explicitly supports Bun as a runtime
- Bun.Glob matches strings 3x faster than fast-glob/micromatch

### Hypotheses (Unverified)

- @clack/prompts multiple-prompt chaining issues may be resolved in Bun 1.3.x (needs testing)
- The --bytecode flag may provide additional startup improvement for compiled binaries (not benchmarked for this use case)
- Dual distribution (npm + binary) is the optimal distribution strategy (needs user research)

## 6. Discussion

### Why Bun Is the Right Choice for This Project

Three factors converge to make Bun the optimal runtime:

1. **CLI startup performance**: 8-15ms cold start vs 40-120ms for Node.js. For a tool invoked per-command, this compounds into a noticeably snappier experience.

2. **Native TypeScript**: Eliminates the tsc/esbuild build step entirely during development. `bun run src/cli.ts` just works. This accelerates the development loop and simplifies CI.

3. **Anthropic ownership**: Bun is no longer a VC-funded startup. It is owned by a $10B+ AI company that ships its flagship product (Claude Code) on Bun. This is the strongest possible signal for long-term viability. The risk of Bun being abandoned is near zero.

### Why Not Node.js

Node.js remains the safest default, but "safe" is not the same as "optimal." For this project:

- Node.js requires a TypeScript build step (tsc or esbuild). Bun does not.
- Node.js startup is 4-8x slower. For a CLI tool, this matters.
- Node.js has no built-in binary compilation. Bun does.
- Node.js has no built-in test runner comparable to Bun's (node --test exists but is less mature).

The primary Node.js advantage (ecosystem maturity) is less relevant for a greenfield project with known dependencies.

### Why Not Deno

Deno 2.x is a strong runtime, but two factors disqualify it for this project:

- Startup is 3-5x slower than Bun (40-60ms vs 8-15ms).
- The permission-based security model adds friction for a tool that inherently needs broad file system and network access.
- gunshi's Bun support is explicitly documented; Deno support exists but is secondary.

### Risk: @clack/prompts Compatibility

This is the highest-risk finding. Multiple GitHub issues document problems with @clack/prompts in Bun, specifically when chaining multiple text prompts. Mitigation options:

1. Test @clack/prompts thoroughly with Bun 1.3.x before committing
2. Have a fallback plan to switch to Inquirer.js or prompts (both pure JS, Bun-compatible)
3. File upstream issues if problems persist

### Risk: Binary Size

50-100 MB binaries are large for a CLI tool. This matters for:

- Distribution bandwidth
- Installation time
- Disk space on CI runners

Mitigation: Use npm distribution as the primary channel (dependencies resolved at install time, small package size). Offer compiled binaries as an optional "zero-dependency" installation path.

### Distribution Strategy Recommendation

Use **Option D (Dual distribution)** following the Tigris CLI pattern:

1. **Primary**: npm package with Node.js shebang. Users install via `npm install -g @acmelabs-15/agent-plugin` or `npx @acmelabs-15/agent-plugin`. Works with both Node.js and Bun. Requires a TypeScript build step for distribution.

2. **Secondary**: Compiled Bun binaries for macOS, Linux, Windows. Published as GitHub releases. For users who do not have Node.js/Bun installed.

3. **Development**: Use Bun natively. `bun run src/cli.ts` with no build step. Bun-native APIs available during development, abstracted behind interfaces for Node.js compatibility at distribution time.

This hybrid approach maximizes reach while preserving Bun's development experience advantages.

**Alternative**: If the project decides Bun-only is acceptable (requiring users to install Bun), use `#!/usr/bin/env bun` shebang directly and skip the build step entirely. This is simpler but limits the user base.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|----------|---------------|-----------|--------|
| P0 | Adopt Bun as the development runtime | Native TypeScript, fast startup, built-in tooling, Anthropic backing | Low |
| P0 | Test @clack/prompts with Bun 1.3.x before committing to it | Historical compatibility issues documented in 3 GitHub issues | Low |
| P1 | Use dual distribution (npm + compiled binary) | Maximizes user reach without requiring Bun installation | Medium |
| P1 | Abstract Bun-native APIs behind interfaces | Enables Node.js fallback for npm distribution path | Medium |
| P2 | Monitor Windows ARM64 support | Currently broken, but low market share for dev tools | None |
| P2 | Evaluate @clack/prompts alternatives (Inquirer.js) as fallback | Reduces dependency risk if Bun compatibility issues persist | Low |

## 8. Conclusion

**Verdict**: Proceed
**Confidence**: High
**Rationale**: Bun is the optimal runtime for this project. Anthropic's acquisition eliminates the primary risk (abandonment). Claude Code proves Bun works for cross-platform CLI distribution at massive scale. The 4-8x startup improvement and native TypeScript execution provide concrete developer and user experience advantages over Node.js.

### User Impact

- **What changes for you**: Development uses `bun run` directly with no build step. TypeScript executes natively. Distribution uses a build step for npm compatibility. Standalone binaries available for zero-dependency installation.
- **Effort required**: Low for Bun adoption. Medium for dual distribution setup. @clack/prompts needs testing (1-2 hours).
- **Risk if ignored**: Choosing Node.js adds unnecessary build complexity, slower startup, and misses alignment with the Anthropic/Claude Code ecosystem that this tool targets.

## Observations

- [fact] Bun 1.3.x is current stable; Anthropic acquired Oven (Bun) in December 2025, ensuring long-term viability #runtime #risk-mitigation
- [fact] Bun cold start is 8-15ms vs Node.js 40-120ms, a 4-8x improvement critical for CLI UX #performance #cli
- [fact] Claude Code ships as Bun-compiled binary to millions of developers across macOS, Linux, Windows, proving cross-platform viability #production-validation
- [fact] Bun natively executes TypeScript without a build step, eliminating tsc/esbuild from the development workflow #developer-experience
- [fact] Compiled Bun binaries are 50-100 MB per platform; Bun team acknowledges size needs reduction #distribution #limitation
- [fact] Node.js API compatibility is >95% with core modules (fs, path, os, events, stream, buffer, url, net, http) fully implemented #compatibility
- [risk] @clack/prompts has documented Bun compatibility issues with multiple sequential text prompts (GitHub issues #4835, #3099, #7033) #dependency-risk
- [risk] Windows ARM64 Bun support is broken with an open tracking issue #cross-platform
- [insight] Dual distribution (npm + compiled binary) following Tigris CLI pattern maximizes user reach while preserving Bun development advantages #distribution
- [decision] Bun is recommended as the development runtime for @acmelabs-15/agent-plugin based on startup performance, native TypeScript, Anthropic backing, and Claude Code ecosystem alignment #runtime-choice

## Relations

- relates_to [[ANALYSIS-001-agent-plugin-foundation-and-vision]]
- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]

## 9. Appendices

### Sources Consulted

- Anthropic acquires Bun announcement: <https://www.anthropic.com/news/anthropic-acquires-bun-as-claude-code-reaches-usd1b-milestone>
- Bun joins Anthropic blog: <https://bun.com/blog/bun-joins-anthropic>
- Bun Node.js compatibility docs: <https://bun.com/docs/runtime/nodejs-compat>
- Bun single-file executable docs: <https://bun.com/docs/bundler/executables>
- Bun Glob docs: <https://bun.com/docs/runtime/glob>
- Tigris CLI Bun migration case study: <https://www.tigrisdata.com/blog/using-bun-and-benchmark/>
- Bun production readiness 2026: <https://dev.to/last9/is-bun-production-ready-in-2026-a-practical-assessment-181h>
- Bun vs Node.js 2026 switch analysis: <https://dev.to/alexcloudstar/bun-vs-nodejs-is-it-time-to-switch-in-2026-5821>
- Bun vs Deno vs Node.js 2026 benchmarks: <https://dev.to/jsgurujobs/bun-vs-deno-vs-nodejs-in-2026-benchmarks-code-and-real-numbers-2l9d>
- Runtime comparison guide 2026: <https://dev.to/dataformathub/nodejs-vs-deno-vs-bun-the-ultimate-runtime-guide-for-2026-di>
- Bun performance benchmarks 2025: <https://strapi.io/blog/bun-vs-nodejs-performance-comparison-guide>
- @clack/prompts Bun issues: <https://github.com/oven-sh/bun/issues/4835>, <https://github.com/oven-sh/bun/issues/3099>, <https://github.com/oven-sh/bun/issues/7033>
- @clack/prompts with Bun demo: <https://medium.com/@wangminder/a-simple-but-powerful-cli-demo-using-clack-with-ts-and-bun-cec91deeb95d>
- Gunshi documentation: <https://gunshi.dev/>
- Gunshi npm: <https://www.npmjs.com/package/gunshi>
- Windows ARM64 Bun issue: <https://github.com/jdx/mise/discussions/7155>
- Bun cross-platform apps guide: <https://blog.logrocket.com/developing-cross-platform-apps-bun/>
- Bun CLI applications guide: <https://oneuptime.com/blog/post/2026-01-31-bun-cli-applications/view>
- Bun binary size reduction request: <https://github.com/oven-sh/bun/issues/5854>

### Data Transparency

- **Found**: Bun version and release cadence. Anthropic acquisition details. Startup benchmarks from multiple sources. Node.js API compatibility matrix from official docs. Cross-platform support matrix. Binary compilation targets and sizes. Tigris CLI real-world migration data. gunshi Bun compatibility. @clack/prompts Bun issues.
- **Not Found**: Exact Bun 1.3.x release date. Whether @clack/prompts issues are resolved in Bun 1.3.x (needs hands-on testing). Bun internal roadmap for binary size reduction timeline. Windows ARM64 fix timeline. Memory usage benchmarks for compiled binaries running MCP servers.
