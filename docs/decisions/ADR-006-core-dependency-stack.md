---
title: ADR-006 Core Dependency Stack
type: decision
permalink: decisions/adr-006-core-dependency-stack-1
status: accepted
decision-makers: Peter Kloss
consulted: Architect, Critic, Independent-thinker, Security, Analyst, High-level-advisor agents
informed: All project contributors
tags:
- decision
- dependencies
- cli
- mcp
- architecture
- gunshi
- zod
- chokidar
- yaml
---

# ADR-006 Core Dependency Stack

## Status

**Accepted**

**Date**: 2026-03-07
**Authors**: Peter Kloss
**Consulted**: Analyst team (dependency research across ANALYSIS-017 through ANALYSIS-021)
**Informed**: All project contributors

## Context and Problem Statement

`@acmelabs-15/agent-plugin` is a cross-platform CLI with an embedded MCP server for managing AI agent plugins across 7 platforms. ADR-005 established Bun as the runtime. The design specification proposed a dependency list, but each library requires validation against 4 criteria: Bun compatibility, security posture, maintenance health, and actual necessity for a CLI tool (not a browser application).

Which libraries should constitute the core dependency stack, and which proposed dependencies should be removed?

## Decision Drivers

- Bun runtime compatibility (ADR-005): every dependency must work on Bun without polyfills or patches
- Minimal dependency count: fewer dependencies reduce supply chain attack surface and maintenance burden
- Security-first: reject libraries with known CVEs or unmaintained transitive dependencies
- Single-maintainer risk: prefer libraries with community backing or viable migration paths
- CLI-appropriate: reject browser-oriented libraries that add weight without CLI benefit

## Considered Options (Per Decision)

This is a multi-decision ADR covering 9 dependency choices. Each decision documents the chosen option and rejected alternatives inline.

## Decision Outcome

**Note on ADR-003 Dependencies**: Four dependencies (deepmerge, shell-quote, validator, atomically) were selected in ADR-003. This ADR does not re-evaluate their selection. Their governance is under ADR-003. Known risk: deepmerge v4.3.1 has a 3-year release gap (ADR-003 NEG-006); deepmerge-ts is the documented migration path.

### Decision 1: CLI Framework -- gunshi

**Chosen**: gunshi v0.29.2 (pin exact version, pre-1.0)
**Fallback**: citty (similar API, 830x more downloads)

Why gunshi:

- Built-in lazy loading via `lazy()`. Critical for 20+ command CLI where loading all command modules at startup adds measurable latency.
- Explicit Bun runtime support (documented, `bun add gunshi`)
- Full TypeScript type inference on args, flags, handlers
- Created by kazupon (Vue.js core team, vue-i18n author)
- @gunshi/plugin-completion provides shell completions (bash/zsh/fish/powershell)

Rejected alternatives:

| Alternative | Rejection Reason |
|-------------|-----------------|
| commander | No built-in lazy loading. 315M downloads but would need manual implementation for every command. |
| citty | No explicit Bun support claim. No documentation site. Otherwise strongest alternative. |
| oclif | Bun compatibility issues (ts-node dependency). 17+ dependencies. Over-engineered for 20-command CLI. |

**Version pinning rationale**: gunshi is pre-1.0 (v0.29.2). Semver allows breaking changes in 0.x releases. Pin exact version in package.json. Review each minor bump before upgrading.

### Decision 2: Interactive Prompts -- @clack/prompts

**Chosen**: @clack/prompts v1.1.0

Why:

- Built-in wizard flows via `group()` for multi-step interactions (plugin scaffolding, migration wizards, collection chooser)
- Spinners, select pickers, confirm dialogs, text inputs
- Visual framing with intro/outro
- Active development (v1.1.0 March 2026)
- Terminal colors via node:util styleText (built-in, zero added bytes)

**Risk**: Bun stdin issues documented (GitHub #4835, #3099, #7033). Mitigation: test with Bun 1.3.x during implementation. Inquirer.js as fallback if stdin issues are blocking.

Rejected alternatives:

| Alternative | Rejection Reason |
|-------------|-----------------|
| inquirer | Larger API surface. No built-in wizard grouping. |
| prompts | Unmaintained (last release 2022). |
| enquirer | Unmaintained (last release 2020). |

### Decision 3: Terminal Colors -- node:util styleText

**Chosen**: node:util styleText (built-in, zero dependencies)

Why: @clack/prompts v1.1.0 replaced picocolors with node:util styleText. Terminal colors use styleText directly. No separate color library dependency. Covers basic CLI color needs (bold, dim, red, green, yellow, cyan). No reason to add chalk, kleur, picocolors, or any other color library when the runtime provides this capability natively.

### Decision 4: Validation -- Zod v4 (full)

**Chosen**: Zod v4 full (not Zod Mini)

Why:

- Org-wide validation standard (already decided in ADR-003 sanitization pipeline)
- Bundle size irrelevant for CLI/backend (not a browser library)
- Method chaining, IntelliSense, built-in English error messages
- Used for plugin.json validation, frontmatter validation, config validation, lockfile schema
- Powers the 6-layer sanitization pipeline (ADR-003)

Rejected alternatives:

| Alternative | Rejection Reason |
|-------------|-----------------|
| Zod Mini | Subset API. Bundle savings irrelevant for CLI. Would require workarounds for features needed in sanitization pipeline. |
| AJV | JSON Schema based. Less ergonomic for TypeScript. No `.transform()` for sanitization. |
| Valibot | Smaller ecosystem. Fewer integrations. |

### Decision 5: Frontmatter Parsing -- yaml 2.x (manual parser)

**Chosen**: yaml 2.x (eemeli/yaml) with manual 5-10 line frontmatter extraction + Zod validation
**DISQUALIFIED**: gray-matter (CVE-2025-64718 in pinned js-yaml@^3.13.1, maintainer inactive 5 years)

Why yaml 2.x:

- ESM native, built-in TypeScript types
- YAML 1.2 spec (modern, vs js-yaml 3.x which uses YAML 1.1)
- Zero dependencies, 82M weekly downloads
- Zero known CVEs
- Frontmatter parsing is trivial: split on `---`, parse YAML block, validate with Zod

The manual frontmatter parser is approximately 10 lines:

1. Check if content starts with `---`
2. Find closing `---`
3. Extract YAML string between markers
4. Parse with yaml 2.x
5. Validate with Zod schema
6. Return `{ frontmatter, body }`

Rejected alternatives:

| Alternative | Rejection Reason |
|-------------|-----------------|
| gray-matter | **SECURITY DISQUALIFICATION**: CVE-2025-64718 (prototype pollution via js-yaml@^3.13.1). Unmaintained 5 years. CJS-only. 4 dependencies. |
| front-matter | Same js-yaml vulnerability. Unmaintained 6 years. |
| js-yaml | v3.x has the CVE. v4.x exists but yaml 2.x is the modern ESM replacement with YAML 1.2 support. |

### Decision 6: MCP Server Framework -- @modelcontextprotocol/sdk

**Chosen**: @modelcontextprotocol/sdk (official Anthropic SDK)
**Rejected**: fastmcp

Why official SDK:

- fastmcp wraps the official SDK (it is a transitive dependency). Using fastmcp means paying both sizes: 4.25 MB (fastmcp) + 1.31 MB (official SDK underneath).
- Official SDK explicitly documents Bun as supported. fastmcp lists Bun compatibility as "unknown".
- fastmcp value-adds (auth, sessions, CORS, HTTP routes) target remote HTTP MCP servers. This project uses stdio-based embedded server. Those features are irrelevant.
- 157 contributors + Anthropic backing vs single maintainer
- Direct upgrade path when MCP v2 spec ships

Boilerplate savings from fastmcp: approximately 30 lines across 14 tools. Not worth the overhead and risk.

### Decision 7: File Watching -- chokidar v5

**Chosen**: chokidar v5 (primary), watcher as fallback
**Scope**: Only used for the author `dev` command (live reload during plugin development)

Why chokidar v5:

- 123M weekly downloads, zero native dependencies, ESM-only
- Cross-platform reliable (FSEvents on macOS, inotify on Linux, ReadDirectoryChanges on Windows)
- Industry standard for file watching in Node.js/Bun ecosystem

Rejected alternatives:

| Alternative | Rejection Reason |
|-------------|-----------------|
| @parcel/watcher | Confirmed broken on Bun (prebuilt binary loading fails). |
| Bun fs.watch | Documented bugs with recursive watching (new files not detected). |
| watcher | 528 weekly downloads, last commit April 2024. Kept as fallback only. |

### Decision 8: Shell Completions -- @gunshi/plugin-completion

**Chosen**: @gunshi/plugin-completion (part of gunshi ecosystem)

Why: Comes with the adopted CLI framework. Uses bombshell-dev/tab internally. Supports bash, zsh, fish, powershell. No separate shell completion library needed.

### Decision 9: Dependencies Removed from Design Spec

Six dependencies proposed in the design specification were evaluated and removed:

| Dependency | Reason for Removal |
|------------|-------------------|
| drizzle-orm | JSON lockfile (ADR-003) sufficient for 5-20 plugins. bun:sqlite built-in if database ever needed. |
| @orama/orama | Array.filter() sufficient for plugin search at expected scale. No plugin manager bundles a search engine. |
| @huggingface/transformers | Over-engineering. 25-118 MB model download. AI assistants via MCP already have semantic understanding. |
| gray-matter | CVE-2025-64718 in js-yaml@^3.13.1. Replaced by yaml 2.x + manual parser. |
| micromark / remark / markdown-it | No markdown-to-HTML rendering needed. Tool extracts frontmatter and passes body as-is to platform adapters. |
| fastmcp | Wraps official SDK (double the size), Bun compatibility unknown, value-adds irrelevant for stdio server. |

**Upgrade paths**: If scale demands grow beyond current design:

- At 50+ plugins: add MiniSearch (35KB, in-memory fuzzy search)
- At 1000+ items: migrate lockfile to bun:sqlite
- These thresholds are estimates. Measure before migrating.

### Decision 10: CI Environment Detection -- ci-info

**Chosen**: ci-info (zero dependencies, 14M weekly downloads, detects 50+ CI vendors)

Why: Industry standard for CI environment detection. Used by npm, pnpm, Wrangler, and Yarn. Maintained by watson/ci-info on GitHub. Covers all major CI platforms including GitHub Actions, GitLab CI, CircleCI, Jenkins, Buildkite, Azure DevOps, and 40+ others. Returns both boolean detection and vendor name, enabling vendor-specific logging and behavior.

Rejected alternatives:

| Alternative | Rejection Reason |
|-------------|-----------------|
| Custom detection | Design spec proposed checking 9 environment variables. Misses 40+ vendors. Creates ongoing maintenance burden as CI platforms add or change env vars. |
| is-ci | Thin wrapper around ci-info that returns boolean only. We need vendor name for logging and diagnostics. Adding is-ci would pull in ci-info as transitive dependency anyway. |

## Final Dependency Summary

| Package | Purpose | Version | Weekly Downloads |
|---------|---------|---------|-----------------|
| gunshi | CLI framework with lazy loading | 0.29.2 (pin exact) | 21K |
| @gunshi/plugin-completion | Shell completions (bash/zsh/fish/powershell) | latest | -- |
| @clack/prompts | Interactive CLI prompts with wizard flows | 1.1.0 | -- |
| zod | Schema validation (6-layer pipeline) | v4 | 65M |
| yaml | YAML/frontmatter parsing (YAML 1.2) | 2.x | 82M |
| @modelcontextprotocol/sdk | MCP server (stdio, 14 tools) | >= 1.24.0 | 5.65M |
| chokidar | File watching (dev command only) | 5.x | 123M |
| deepmerge | Config/hook merging (ADR-003) | 4.x | -- |
| shell-quote | Hook command safety (ADR-003) | >= 1.7.3 | 22.7M |
| validator | String sanitization (ADR-003) | >= 13.15.22 | 15.5M |
| atomically | Atomic file writes for lockfile (ADR-003) | pin exact | -- |
| ci-info | CI environment detection (50+ vendors) | latest | 14M |

**Total**: 13 dependencies (9 new in this ADR + 4 from ADR-003).

## Consequences

### Positive

- **POS-001**: 13 total dependencies. Lean for a full-featured CLI tool with embedded MCP server. Comparable tools (oclif-based CLIs) start at 17+ dependencies for the framework alone.
- **POS-002**: 6 dependencies removed from spec. Simpler architecture, smaller install, faster startup.
- **POS-003**: All dependencies verified Bun-compatible (except @clack/prompts stdin behavior, which requires testing).
- **POS-004**: Security-first. gray-matter CVE eliminated. Official SDK preferred over single-maintainer wrapper. No libraries with known vulnerabilities.
- **POS-005**: Clear upgrade paths documented for when complexity is justified (MiniSearch at 50+ plugins, bun:sqlite at 1000+ items).

### Negative

- **NEG-001**: gunshi has 21K weekly downloads and is pre-1.0. Migration to citty may be needed if project is abandoned. Mitigation: citty has similar API; migration effort estimated at 1-2 weeks for 20+ commands (unverified -- citty uses different command definition shapes and has no equivalent shell completion plugin).
- **NEG-002**: @clack/prompts Bun stdin compatibility unverified. Bun issues #4835, #3099, #7033 document stdin edge cases. Mitigation: test during implementation sprint; Inquirer.js as fallback.
- **NEG-003**: No full-text search at launch. Users with 50+ plugins may want fuzzy search. Mitigation: MiniSearch upgrade path documented; Array.filter with includes() covers exact and substring matching.

## Dependency Security Controls

### Version Pinning Policy

All dependencies MUST use exact version pins or tight semver ranges in package.json. The "latest" specifier is prohibited for production dependencies. gunshi's exact pin (0.29.2) is the model for pre-1.0 libraries.

### Lockfile Integrity

- Use bun.lockb with integrity hashes
- CI MUST use bun install --frozen-lockfile to prevent lockfile tampering
- Lockfile is committed to version control

### CI Audit Gate

- bun audit (or equivalent) runs in CI as a blocking gate
- New CVEs in dependencies block the build until resolved or documented as accepted risk

### Dependency Governance

New runtime dependencies require:

1. Bun compatibility verification (tested, not assumed)
2. Security audit (CVE check, maintainer count, download volume)
3. Single-maintainer risk assessment (bus factor)
4. ADR-006 amendment documenting the addition and rationale

### Supply Chain Monitoring

- Enable GitHub Advisory alerts on the repository
- Consider Socket.dev or Snyk for real-time dependency monitoring
- Quarterly manual dependency audit per Confirmation item 6

## Reversibility Assessment

- [x] **Rollback capability**: Each dependency can be replaced independently. No dependency creates coupling that prevents rollback.
- [x] **Vendor lock-in**: All dependencies are MIT/ISC licensed open-source. No proprietary APIs.
- [x] **Exit strategy**: gunshi to citty (similar API, 1-2 week migration for 20+ commands). @clack/prompts to Inquirer.js. yaml to js-yaml v4. chokidar to native Bun fs.watch (when bugs are fixed). @modelcontextprotocol/sdk has no alternative (it IS the standard).
- [x] **Legacy impact**: Greenfield project. No existing code to migrate.
- [x] **Data migration**: No dependency stores persistent data. Lockfile format (ADR-003) is independent of library choices.

## Vendor Lock-in Assessment

**Dependency**: @modelcontextprotocol/sdk
**Lock-in Level**: Medium

### Lock-in Indicators

- [x] Proprietary APIs without standards-based alternatives (MCP is Anthropic-defined protocol)
- [ ] Data formats that require conversion to export
- [ ] Licensing terms that restrict migration
- [x] Integration depth that increases switching cost (14 tools implemented against SDK API)
- [ ] Team training investment

### Exit Strategy

**Trigger conditions**: MCP protocol abandoned by Anthropic, or incompatible fork emerges
**Migration path**: MCP is an open specification. Alternative SDKs could implement the same protocol. Tool definitions are declarative and portable.
**Estimated effort**: 3-5 days to adapt to alternative SDK implementing same protocol
**Data export**: No data stored in SDK. All state in lockfile (ADR-003).

### Accepted Trade-offs

MCP is the standard protocol for AI agent tool integration, backed by Anthropic and adopted by all 7 target platforms. The lock-in risk is acceptable because the protocol itself is open, and the SDK is the reference implementation.

## Confirmation

Implementation compliance will be confirmed via:

1. `package.json` audit: verify exact 13 dependencies listed, no unlisted additions
2. `bun install` succeeds without errors or fallback to Node.js polyfills
3. CLI startup benchmark: cold start under 50ms via bunx (per ADR-005 CI threshold); compiled binary cold start under 150ms
4. @clack/prompts stdin test: run interactive wizard under Bun, verify all input types work
5. Frontmatter parser test: parse 100 SKILL.md files with yaml 2.x, validate against Zod schemas
6. Quarterly dependency audit: check for new CVEs, maintenance status, Bun compatibility regressions

## Implementation Notes

- **IMP-001**: Pin gunshi to exact version `0.29.2` in package.json (no caret, no tilde). Review each minor version bump individually before upgrading.
- **IMP-002**: The frontmatter parser is a utility function, not a library. Implement in `src/utils/frontmatter.ts`. Approximately 10 lines of code.
- **IMP-003**: chokidar is a devDependency-like concern (only used for `agent-plugin dev` command). Consider lazy-importing it only when the dev command is invoked.
- **IMP-004**: @clack/prompts Bun stdin testing must happen before v1.0 release. Create a test harness that exercises: text input, select picker, confirm dialog, multi-select, and group wizard.
- **IMP-005**: If @clack/prompts fails Bun stdin testing, migration to Inquirer.js requires replacing `group()` calls with sequential prompt chains. Estimate 1-2 days.
- **IMP-006**: validator MUST be pinned to >= 13.15.22 due to CVE-2025-12758 (CVSS 7.5, Unicode length bypass in isLength()). Verify which validator functions the sanitization pipeline uses.
- **IMP-007**: @clack/prompts Bun compatibility is a BLOCKING GATE. Before any interactive flow implementation: (1) test text(), select(), confirm(), multiselect(), and group() on target Bun version, (2) all 5 must pass without stdin errors, (3) if any fail, evaluate Bun version pinning or switch to Inquirer.js with revised timeline (1-2 weeks for wizard reimplementation). No interactive code is written until this gate passes. NOTE: Initial testing on Bun 1.3.8 shows setRawMode() works correctly and non-interactive APIs pass. The EPERM regression from Bun 1.3.2 (issue #24615) appears resolved in 1.3.8.
- **IMP-008**: Verify Zod v4 satisfies @modelcontextprotocol/sdk peer dependency (zod v3.25+ or v4) without runtime conflicts before implementation.
- **IMP-009**: Hook command execution method (exec vs execFile) MUST be resolved in ADR-004. If exec() is used, shell-quote install-time AST inspection is bypassable at runtime (CWE-78).

## References

- **REF-001**: ANALYSIS-017 CLI Framework Comparison (gunshi vs commander vs citty vs oclif)
- **REF-002**: ANALYSIS-018 Interactive Prompts and Colors (@clack/prompts evaluation, node:util styleText for terminal colors)
- **REF-003**: ANALYSIS-019 MCP Framework and File Watching (@modelcontextprotocol/sdk vs fastmcp, chokidar vs alternatives)
- **REF-004**: ANALYSIS-020 Frontmatter and Markdown Processing Libraries (gray-matter CVE, yaml 2.x selection)
- **REF-005**: ANALYSIS-021 Data Storage and Search (drizzle-orm/orama removal, lockfile sufficiency)
- **REF-006**: ANALYSIS-013 Input Sanitization Patterns (Zod v4, shell-quote, validator selection)
- **REF-007**: ADR-001 Plugin Format and Manifest (plugin.json schema requiring validation)
- **REF-008**: ADR-003 Conflict Resolution and Namespacing (lockfile, sanitization pipeline, deepmerge/atomically/shell-quote/validator)
- **REF-009**: ADR-005 Bun Runtime (runtime decision driving compatibility requirements)

## Observations

- [decision] gunshi v0.29.2 chosen as CLI framework for built-in lazy loading, explicit Bun support, and TypeScript type inference; citty is documented fallback #cli #gunshi #dependency
- [decision] @clack/prompts v1.1.0 chosen for interactive CLI with built-in wizard flows via group(); terminal colors via node:util styleText (no separate color library) #cli #prompts #dependency
- [decision] yaml 2.x with manual frontmatter parser replaces gray-matter after CVE-2025-64718 security disqualification #security #frontmatter #dependency
- [decision] @modelcontextprotocol/sdk (official Anthropic) chosen over fastmcp wrapper: double-size avoided, Bun explicitly supported, stdio-only server needs no HTTP features #mcp #dependency
- [decision] chokidar v5 for file watching in dev command; @parcel/watcher broken on Bun, native Bun fs.watch has recursive watching bugs #file-watching #dependency
- [decision] 6 dependencies removed from design spec: drizzle-orm, @orama/orama, @huggingface/transformers, gray-matter, markdown renderers, fastmcp #simplification #dependency
- [risk] gunshi is pre-1.0 with 21K weekly downloads; citty migration path documented at 1-2 week effort for 20+ commands #gunshi #single-maintainer
- [risk] @clack/prompts Bun stdin compatibility unverified; Bun issues #4835, #3099, #7033 document edge cases; Inquirer.js is documented fallback #clack #bun-compat
- [fact] Final stack is 13 total dependencies (9 new + 4 from ADR-003), lean for a CLI with embedded MCP server #dependency-count
- [decision] ci-info adopted for CI detection: zero deps, 50+ vendors, used by npm/pnpm/Wrangler #ci #dependency
- [fact] picocolors no longer transitive via @clack/prompts v1.1.0 -- replaced by node:util styleText #dependency-change
- [fact] Zod v4 full chosen over Mini because bundle size is irrelevant for CLI and sanitization pipeline needs full API surface #zod #validation
- [insight] MCP SDK has medium vendor lock-in but acceptable because MCP is an open protocol adopted by all 7 target platforms #mcp #lock-in

## Relations

- depends_on [[ADR-001 Plugin Format and Manifest]]
- depends_on [[ADR-003 Conflict Resolution and Namespacing]]
- depends_on [[ADR-005 Bun Runtime]]
- relates_to [[ANALYSIS-017-cli-framework-comparison]]
- relates_to [[ANALYSIS-018-interactive-prompts-and-colors]]
- relates_to [[ANALYSIS-019-mcp-framework-and-file-watching]]
- relates_to [[ANALYSIS-020 Frontmatter and Markdown Processing Libraries]]
- relates_to [[ANALYSIS-021-data-storage-and-search]]
- relates_to [[ANALYSIS-013-input-sanitization-patterns]]
- relates_to [[ADR-002 Target Platforms and Audiences]]
- relates_to [[ADR-007 CLI Architecture and Interaction Model]]