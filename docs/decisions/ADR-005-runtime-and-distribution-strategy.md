---
title: ADR-005 Runtime and Distribution Strategy
type: decision
permalink: decisions/adr-005-runtime-and-distribution-strategy-1
status: Accepted
date: '2026-03-07'
decision-makers: Agent Plugin Core Team
consulted: Architect agent, Critic agent, Independent-thinker agent
informed: Implementer agents, QA agents
tags:
- decision
- runtime
- bun
- distribution
- architecture
---

## Status

**Accepted**

## Context

We are selecting the runtime for `@acmelabs-15/agent-plugin`, a cross-platform CLI tool with an embedded MCP server for managing AI agent plugins across 7 coding platforms. The runtime choice affects three critical dimensions:

1. **CLI startup performance**: The tool is invoked per-command. Every millisecond of cold start latency compounds across developer workflows. A plugin manager that feels sluggish on every invocation erodes trust and adoption.
2. **Development velocity**: Native TypeScript execution eliminates the build-test-debug cycle friction that slows iteration on a greenfield project.
3. **Distribution reach**: Users need both a zero-install path (bunx) and a zero-dependency path (standalone binary) to cover corporate lockdown environments and offline scenarios.

Three runtimes were evaluated in detail (ANALYSIS-016-bun-runtime-assessment):

- **Bun 1.3.x**: 8-15ms cold start via `bun run`/`bunx` (compiled binaries: 50-100ms+), native TypeScript, built-in binary compilation, built-in sqlite
- **Node.js 22.x**: 40-120ms cold start, requires tsc/esbuild build step, no native binary compilation (requires pkg or nexe)
- **Deno 2.x**: 40-60ms cold start, native TypeScript, permission-based security model, `deno compile` for binaries

An additional strategic factor: Anthropic acquired Oven (Bun's parent company) in December 2025. Claude Code ships as a Bun-compiled binary to millions of developers. This tool targets AI coding platforms where Bun is becoming the de facto runtime.

## Decision Drivers

- **CLI cold start latency**: Invoked per-command; sub-20ms target via `bun run`/`bunx` for responsive developer experience (compiled binaries have a separate, higher threshold)
- **Native TypeScript execution**: Eliminate build step friction during development
- **Cross-platform binary compilation**: macOS (x64/arm64), Linux (x64/arm64), Windows (x64) without third-party packaging tools
- **Anthropic ecosystem alignment**: This tool targets AI coding platforms; Claude Code uses Bun
- **Node.js API compatibility**: Existing npm ecosystem libraries must work without modification
- **Built-in tooling**: Reduce external dependency count for test runner, bundler, package manager
- **gunshi CLI framework support**: gunshi (chosen CLI framework) explicitly supports Bun as a runtime

## Decision

We adopt **Bun 1.3.x** as the sole runtime for @acmelabs-15/agent-plugin with a **Bun-only distribution strategy**. No Node.js consumer support is provided.

### Runtime: Bun 1.3.x

Bun provides the fastest CLI cold start (8-15ms), native TypeScript execution, and a comprehensive built-in toolset (test runner, bundler, package manager, binary compiler, sqlite). Node.js API compatibility exceeds 95% for core modules.

Key runtime capabilities used by this project:

- `Bun.file()` and `Bun.write()` for filesystem operations in plugin installation
- `Bun.Glob` for component path resolution in plugin.json declarations
- `Bun.spawn` for subprocess management (MCP server lifecycle)
- `bun:sqlite` for future local registry/cache needs (zero additional dependencies)
- `bun build --compile` for standalone binary generation

### Distribution Strategy: Bun-Only

Following the Tigris CLI pattern, we distribute through two channels (both Bun-only):

**Primary: npm package**

```bash
bunx @acmelabs-15/agent-plugin install <plugin>
```

The npm package works with bunx. Bun executes TypeScript directly — no build step required.

**Secondary: Compiled standalone binaries**

```bash
# Downloaded from GitHub Releases
agent-plugin install <plugin>
```

Built via `bun build --compile` targeting:

| Platform | Architecture | Status |
|---|---|---|
| macOS | x64 | Supported |
| macOS | arm64 | Supported |
| Linux | x64 | Supported |
| Linux | arm64 | Supported |
| Windows | x64 | Supported |
| Windows | arm64 | Broken (known Bun issue) |

Compiled binaries are 50-100 MB per platform. This is acceptable for a developer tool but rules out distribution via package managers with size constraints.

## Consequences

### Positive

- **POS-001**: 4-8x faster CLI startup via `bun run`/`bunx` (8-15ms vs 40-120ms Node.js) improves developer experience on every command invocation. This advantage applies to the `bun run`/`bunx` execution path; compiled binaries start in 50-100ms+, comparable to Node.js
- **POS-002**: Native TypeScript execution eliminates the build step during development, reducing iteration cycle time
- **POS-003**: Built-in binary compilation via `bun build --compile` removes dependency on third-party packaging tools (pkg, nexe, vercel/pkg)
- **POS-004**: Anthropic ownership of Oven ensures long-term viability; Claude Code proves Bun's production readiness at scale
- **POS-005**: `bun:sqlite` available built-in if local database needs arise (plugin registry cache, installation history) with zero additional dependencies
- **POS-006**: Bun.file(), Bun.write(), Bun.Glob, Bun.spawn provide purpose-built APIs for CLI tool development patterns used throughout the codebase

### Negative

- **NEG-001**: @clack/prompts has documented Bun stdin issues (GitHub issues #4835, #3099, #7033) and a Bun 1.3.2 EPERM regression (issue #24615). Requires testing with Bun 1.3.x before committing to interactive flows. Mitigation: test early in development; fall back to Inquirer.js or custom stdin handling if unresolved
- **NEG-002**: Compiled binaries are 50-100 MB per platform. Acceptable for developer tool distribution via GitHub Releases but prevents lightweight package manager distribution
- **NEG-003**: Windows ARM64 support is broken in Bun. Low risk: Windows ARM64 represents a small fraction of developer tool installations
- **NEG-005**: Compiled binary startup (50-100ms via `bun build --compile`) is comparable to Node.js, not faster. The 4-8x startup advantage applies only to `bun run`/`bunx` execution. Users of standalone binaries do not benefit from the cold start improvement
- **NEG-006**: Anthropic could redirect Bun engineering toward Claude Code's internal needs rather than general-purpose CLI tooling. Corporate acquisition changes roadmap control from community-driven to corporate-driven

## Alternatives Considered

### Node.js 22.x

- **Description**: Industry-standard JavaScript runtime with the largest ecosystem
- **Good**: Largest ecosystem, most mature, universal corporate approval
- **Good**: LTS support with predictable release cadence
- **Bad**: 40-120ms cold start is 4-8x slower than Bun for CLI invocations
- **Bad**: Requires TypeScript build step (tsc or esbuild) for every development iteration
- **Bad**: No built-in binary compilation; requires pkg or nexe (both have maintenance concerns)
- **Bad**: No built-in test runner equivalent to bun:test (Jest/Vitest add dependencies)
- **Rejection Reason**: The cold start penalty is unacceptable for a CLI tool invoked per-command. The build step friction slows development on a greenfield project where iteration speed matters most

### Deno 2.x

- **Description**: Secure-by-default runtime with native TypeScript and web standards focus
- **Good**: Native TypeScript execution without build step
- **Good**: Built-in `deno compile` for standalone binaries
- **Good**: Permission-based security model (--allow-read, --allow-net)
- **Neutral**: npm compatibility improved in Deno 2 but still has edge cases
- **Bad**: 40-60ms cold start is 3-5x slower than Bun
- **Bad**: Permission model adds friction for a tool that inherently needs broad filesystem and network access (plugin installation, MCP server management)
- **Bad**: gunshi CLI framework explicitly supports Bun as a runtime; Deno support is secondary
- **Rejection Reason**: Slower startup than Bun with no compensating advantage. The permission model works against this tool's use case rather than for it

## Confirmation

Implementation compliance will be verified through:

- [ ] Bun 1.3.x runs all project dependencies without compatibility issues
- [ ] CLI cold start via `bun run`/`bunx` measured at less than 20ms on macOS (arm64) and Linux (x64). Compiled binary cold start measured and documented (target: under 150ms)
- [ ] @clack/prompts interactive flows (select, confirm, text input) work correctly under Bun 1.3.x
- [ ] Compiled binaries execute correctly on macOS (x64, arm64), Linux (x64, arm64), Windows (x64)
- [ ] npm package installs and runs via `bunx` without errors
- [ ] CI pipeline produces binaries for all 5 supported platform/architecture combinations

## Reversibility Assessment

- [x] **Rollback capability**: Code uses standard Node.js APIs (>95% compatible). Migration to Node.js requires adding a build step but no API rewrites
- [x] **Vendor lock-in**: Bun-specific APIs (Bun.file, Bun.Glob, Bun.spawn) are used but have direct Node.js equivalents (fs, glob, child_process). Lock-in level: Low
- [x] **Exit strategy**: Replace Bun-specific APIs with Node.js equivalents; add tsc/esbuild build step; switch binary compilation to pkg/nexe
- [x] **Legacy impact**: No existing codebase to migrate; greenfield project
- [x] **Data migration**: No persistent data formats tied to Bun runtime

## Vendor Lock-in Assessment

**Dependency**: Bun runtime (Oven, acquired by Anthropic)
**Lock-in Level**: Low

### Lock-in Indicators

- [x] Proprietary APIs without standards-based alternatives: Bun.file(), Bun.Glob, Bun.spawn have Node.js equivalents
- [ ] Data formats that require conversion to export: Not applicable
- [ ] Licensing terms that restrict migration: MIT licensed
- [x] Integration depth that increases switching cost: bun:sqlite and bun build --compile have no direct Node.js equivalents
- [ ] Team training investment: Bun API surface is small; Node.js knowledge transfers

### Exit Strategy

**Trigger conditions**: Bun development stalls, critical unfixed bugs, or Anthropic deprecates Bun
**Migration path**: Replace Bun-specific APIs with Node.js equivalents (fs, glob, child_process); add esbuild for TypeScript compilation; use pkg or nexe for binary compilation; replace bun:sqlite with better-sqlite3
**Estimated effort**: 2-4 days for API replacement, 1-2 days for build pipeline migration
**Data export**: No Bun-specific data formats used

### Accepted Trade-offs

Bun-specific APIs (Bun.file, Bun.Glob, bun:sqlite) provide measurable developer experience and performance benefits. The exit cost is low (3-6 days) because all Bun APIs have well-documented Node.js equivalents. Anthropic's ownership of both Bun and the primary target platform (Claude Code) reduces abandonment risk.

## Implementation Notes

- **IMP-001**: Pin Bun version in package.json `engines` field: `"bun": ">=1.3.0"`
- **IMP-002**: CI must test on Bun 1.3.x specifically; do not assume latest Bun compatibility
- **IMP-003**: Test @clack/prompts stdin handling in CI before implementing interactive flows (NEG-001 risk)
- **IMP-004**: Binary compilation targets should be configured in a build matrix, not hardcoded per-platform scripts
- **IMP-006**: Document Bun installation as a development prerequisite in CONTRIBUTING.md
- **IMP-007**: Compiled binaries distributed via GitHub Releases MUST include SHA-256 checksums and code signing (macOS Developer ID, Windows Authenticode). Full binary integrity strategy to be defined in release engineering. CWE-494 mitigation
- **IMP-008**: @clack/prompts Bun compatibility is a BLOCKING GATE. Before any interactive flow implementation: (1) test text, select, confirm, multi-select, and group() under Bun 1.3.x, (2) verify issue #24615 (EPERM regression) is resolved, (3) if testing fails, select Inquirer.js as replacement before proceeding. No interactive code without passing this gate
- **IMP-009**: The embedded MCP server is a long-running process where cold start latency is irrelevant. MCP server performance depends on memory footprint, stability under sustained load, and event loop behavior, not startup time. Bun's HTTP server performance (via Bun.serve()) and WebSocket handling should be benchmarked for the MCP server component separately from CLI cold start measurements
- **IMP-011**: CI pipeline MUST include a cold start benchmark that fails the build if `bun run` startup exceeds 50ms (with 20ms target). This prevents performance regression as dependencies grow. Benchmark should measure time-to-first-output of the CLI entrypoint, not a minimal script

## References

- **REF-001**: ANALYSIS-016-bun-runtime-assessment (cold start benchmarks, compatibility matrix, API surface analysis)
- **REF-002**: Anthropic acquires Oven (December 2025)
- **REF-003**: Claude Code distribution model (Bun-compiled binary)
- **REF-004**: Tigris CLI dual distribution pattern (npm + compiled binaries)
- **REF-005**: @clack/prompts Bun issues: GitHub #4835, #3099, #7033
- **REF-006**: gunshi CLI framework Bun support documentation

## Observations

- [decision] Bun 1.3.x adopted as runtime for @acmelabs-15/agent-plugin based on 4-8x cold start advantage over Node.js #runtime #bun
- [decision] Bun-only distribution strategy: npm package (bunx) as primary, compiled standalone binaries as secondary; no npx/Node.js support #distribution #bun-only
- [decision] Node.js rejected due to 40-120ms cold start and mandatory TypeScript build step #runtime #node
- [decision] Deno 2.x rejected due to 3-5x slower startup than Bun and permission model friction #runtime #deno
- [fact] Bun cold start via `bun run`/`bunx`: 8-15ms; compiled binary: 50-100ms+; Node.js: 40-120ms; Deno: 40-60ms (ANALYSIS-016 benchmarks) #performance
- [fact] Anthropic acquired Oven (Bun parent company) December 2025; Claude Code ships as Bun-compiled binary #ecosystem
- [fact] Compiled binaries are 50-100 MB per platform via bun build --compile #distribution
- [fact] Node.js API compatibility exceeds 95% for core modules in Bun 1.3.x #compatibility
- [risk] @clack/prompts has documented Bun stdin issues (GitHub #4835, #3099, #7033) requiring pre-v1.0 testing #clack #compatibility
- [risk] Windows ARM64 support broken in Bun; low impact for developer tool market #windows #compatibility
- [fact] npm distribution via bunx only; no TypeScript build step needed since Bun executes TS natively #distribution #bun-only
- [decision] Bun-only distribution: no Node.js consumer support, no npx, no abstraction layer #bun-only #distribution
- [risk] Compiled binary startup (50-100ms) does not benefit from Bun's cold start advantage #performance #binaries
- [risk] Anthropic acquisition shifts Bun roadmap control from community-driven to corporate-driven #ecosystem #vendor
- [requirement] Binary releases MUST include SHA-256 checksums and code signing (CWE-494) #security #distribution
- [requirement] @clack/prompts Bun compatibility is a blocking gate before interactive flow implementation #clack #blocking

## Relations

- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-002 Target Platforms and Audiences]]
- relates_to [[ANALYSIS-016-bun-runtime-assessment]]
- depends_on [[ADR-003 Conflict Resolution and Namespacing]]
