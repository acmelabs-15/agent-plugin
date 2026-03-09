---
title: ADR-013 npm-Package Distribution and Revised Command Tree
type: decision
permalink: decisions/adr-013-npm-package-distribution-and-revised-command-tree-1
tags:
- distribution
- npm
- commands
- architecture
- simplification
---

# ADR-013 npm-Package Distribution and Revised Command Tree

---
status: "superseded"
date: "2026-03-09"
decision-makers: "Peter Kloss, Agent Plugin Core Team"
consulted: "Architect agent, Analyst agent, Independent Thinker agent, High-Level Advisor agent"
informed: "All project contributors"
---

## Status

**Superseded** by [[ADR-014 Explicit Installation Model and Content Features]] (2026-03-09)

Originally accepted (2026-03-09, Round 2: 5 Accept + 1 D&C). Superseded the same day after reviewing the Vercel Skills CLI approach. ADR-014 replaces the postinstall auto-wiring model with explicit `agent-plugin add/remove/update` commands, a custom lockfile (`.agent-lock.json`), and a features model for per-component selection. This ADR remains as a historical record.

## Context and Problem Statement

During Group 7 ideation, the team reviewed ANALYSIS-033 (Consumer and Author Commands) and ANALYSIS-034 (Skill Versioning Models Comparison). ANALYSIS-033 identified 12 conflicts and 11 gaps between the design spec and existing ADRs. ANALYSIS-034 compared three production systems: TanStack Intent, Vercel Skills CLI, and Claude Code Plugins.

The original architecture (ADRs 001-012) designed custom source resolution (ADR-008), a custom lockfile (plugin-lock.json in ADR-003 Decision 3), a 6-phase installation lifecycle (ADR-010), and consumer commands (add/remove/upgrade). This created significant implementation surface area: ~2000-3000 lines of TypeScript for install/uninstall/upgrade alone (ADR-010 NEG-002), plus a custom source resolver handling npm registry, GitHub releases, GitLab, and local paths (ADR-008).

The team decided to adopt the TanStack Intent model: plugins ship as standard npm packages, bun handles all package management, and agent-plugin becomes purely a "wiring tool" that bridges npm packages to AI platform configs. This eliminates entire subsystems while preserving the core value proposition.

How should `@acmelabs-15/agent-plugin` distribute plugins, manage versions, and structure its command tree given that bun can handle all package management concerns?

## Decision Drivers

- Simplification: reduce implementation surface area by eliminating subsystems that duplicate package manager functionality
- Leverage existing tooling: bun already handles version resolution, lockfiles, source resolution, integrity verification, and upgrades
- Developer familiarity: npm package workflow is the most widely understood distribution model in the JavaScript ecosystem
- Cross-platform wiring: the core value proposition is bridging plugin content to 7 AI platform configs, not reimplementing npm
- ANALYSIS-033 findings: 12 conflicts between spec and ADRs indicate over-engineering in the consumer command layer
- ANALYSIS-034 findings: TanStack Intent demonstrates that a "wiring tool" model works in production for AI agent skills
- ANALYSIS-034 recommendation override: ANALYSIS-034 explicitly recommended AGAINST the TanStack model due to its tight coupling of skill versions to library versions. This recommendation was overridden because ANALYSIS-033 revealed 12 command conflicts in the status quo that the TanStack model resolves. The conflict elimination outweighs the versioning coupling concern for a greenfield CLI tool where plugins are authored by package maintainers.

## Considered Options

- Option A: Keep custom source resolution and consumer commands (status quo from ADRs 001-012)
- Option B: Adopt npm-package distribution with bun as package manager, agent-plugin as wiring tool (chosen)
- Option C: Hybrid model with custom lockfile but npm-only sources

## Pros and Cons of Rejected Options

### Option A: Status Quo (Custom Commands from ADRs 001-012)

- Good, because it provides full control over UX and custom upgrade logic
- Bad, because it duplicates Bun's package management capabilities
- Bad, because ANALYSIS-033 identified 12 command conflicts in this approach
- Bad, because it carries ongoing maintenance burden of the source resolution pipeline

### Option C: Hybrid (Custom Lockfile with npm-Only Sources)

- Good, because it offers a gradual migration path from the current architecture
- Bad, because two installation paths create user confusion
- Bad, because it still maintains custom resolution code
- Bad, because there is no clear heuristic for when to use which installation path

## Decision Outcome

Chosen option: "Option B: npm-package distribution with bun as package manager", because it eliminates 3 of 5 CLI command groups (add/remove/upgrade), the custom lockfile (plugin-lock.json), and the entire source resolution pipeline (ADR-008), while preserving the core wiring value proposition. Six interconnected decisions implement this model.

### Decision 1: npm-Package Distribution Model

Plugins are standard npm packages that contain a `plugin.json` manifest alongside their `package.json`. Bun handles ALL package management: version resolution, lockfile (bun.lockb), source resolution (npm registry, `github:owner/repo`, local `file:` paths), integrity verification, and upgrades.

agent-plugin is a wiring tool. It discovers `plugin.json` files in `node_modules` and installs their declared content to AI platform configurations.

This follows the TanStack Intent model (ANALYSIS-034) where skills ship inside npm packages and the package manager controls versioning. The library maintainer owns both the code and the plugin content.

**Contrast with rejected models**:

- Vercel Skills (git clone + custom lockfile): Requires custom source resolution, SHA-based version tracking, and a custom lockfile. This is the complexity ADR-008 and ADR-003 Decision 3 built.
- Claude Code Plugins (marketplace + plugin.json version): Requires marketplace infrastructure, 6 source types, auto-update mechanisms. Over-engineered for a CLI tool.
- TanStack Intent (npm packages + scan node_modules): Minimal custom infrastructure. Package manager handles everything except wiring. This is what we adopt.

**Source types supported** (via bun, not agent-plugin):

| Source | Example | Notes |
|---|---|---|
| npm registry | `bun add @scope/plugin-name` | Default, most common |
| GitHub shorthand | `bun add github:owner/repo` | Bun-native support |
| Local path | `bun add file:../my-plugin` | Development workflow |
| Git URL | `bun add git+https://github.com/owner/repo.git` | Bun-native support |

**What agent-plugin no longer handles**: Version resolution, integrity verification, source URL parsing, tarball download, archive extraction, registry API calls, GitHub API calls, GitLab API calls. Bun handles all of these.

**What agent-plugin loses**: GitLab URL shorthand support (bun does not support `gitlab:` prefix). Bare `owner/repo` shorthand without `github:` prefix (ADR-008 Decision 2's auto-detection). These are acceptable tradeoffs given the complexity reduction.

### Decision 2: Revised Command Tree

Consumer commands (`add`, `remove`, `upgrade`) are eliminated. Bun handles these operations directly. The `dev` command is eliminated because lifecycle hooks (Decision 3) replace its file-watching behavior. The `publish` command remains excluded (per ADR-002). The `complete` command is removed (shell completions are not an MVP requirement).

**Revised command tree**:

```text
agent-plugin
  Lifecycle Management
    init                    Wire lifecycle hooks into package.json (Husky model)
    deinit                  Remove lifecycle hooks from package.json
    install                 Scan node_modules, diff vs wired state, reconcile platforms
    list                    Show discovered and wired plugins

  Plugin Authoring
    create                  Scaffold a new plugin project
    validate                Check plugin.json integrity and content drift
    build                   Build plugin for distribution

  MCP Server
    mcp serve               Start embedded MCP tool server

  Content-Type Groups (ADR-012)
    skill
      create                Scaffold a new skill
      remove                Remove a skill from plugin
      list                  List skills in plugin
      eval                  Evaluate skill quality (creator skill)
      improve               Improve skill based on eval (creator skill)
    agent
      create                Scaffold a new agent
      remove                Remove an agent from plugin
      list                  List agents in plugin
      eval                  Evaluate agent quality (creator skill)
      improve               Improve agent based on eval (creator skill)
    mcp
      create                Scaffold a new MCP server
      create-tool           Add a tool to existing MCP server
      remove                Remove an MCP server from plugin
      remove-tool           Remove a tool from MCP server
      list                  List MCP servers in plugin
      eval                  Evaluate MCP server quality (creator skill)
      improve               Improve MCP server based on eval (creator skill)
    command
      create                Scaffold a new command
      remove                Remove a command from plugin
      list                  List commands in plugin
    hook
      create                Scaffold a new hook
      remove                Remove a hook from plugin
      list                  List hooks in plugin
    instruction
      create                Scaffold a new instruction
      remove                Remove an instruction from plugin
      list                  List instructions in plugin
      eval                  Evaluate instruction quality (creator skill)
      improve               Improve instruction based on eval (creator skill)
```

**Removed commands and rationale**:

| Command | Why Removed |
|---|---|
| `add` | `bun add @scope/plugin` handles installation. `postinstall` hook triggers `agent-plugin install` for wiring. |
| `remove` | `bun remove @scope/plugin` handles uninstallation. `postinstall` hook triggers `agent-plugin install` which detects removed packages and unwires. |
| `upgrade` | `bun update @scope/plugin` handles version updates. `postinstall` hook triggers rewiring. |
| `dev` | Lifecycle hooks (`postinstall`) replace the file-watching workflow. Authors use `bun add file:../my-plugin` for local development. |
| `publish` | Rejected by ADR-002. Authors use `bun publish` directly. |
| `complete` | Shell completions deferred. Not MVP. |

**Local development workflow**: Plugin authors developing locally use `bun link` to symlink their plugin into `node_modules`, then run `agent-plugin install` manually to test wiring changes. No built-in watch mode is provided. This is standard npm/bun workflow.

### Decision 3: Husky-Style Lifecycle Hook Init

`agent-plugin init` adds `"postinstall": "agent-plugin install"` to the project's `package.json` scripts section. This is inspired by the Husky model where a tool wires itself into npm lifecycle hooks. Note: modern Husky (v5+) uses the `prepare` hook, not `postinstall`, because Husky only needs to set up git hooks in a local dev context. We use `postinstall` instead because we need to rewire platform configs whenever dependencies change, and Bun fires `postinstall` on ALL package operations (add, remove, update, install).

**Smart merge behavior**:

- If `postinstall` does not exist: creates `"postinstall": "agent-plugin install"`
- If `postinstall` exists: chains without breaking the original, e.g., `"postinstall": "existing-command && agent-plugin install"`
- `agent-plugin deinit` reverses this cleanly: removes `agent-plugin install` from the chain. If it was the only command, removes the `postinstall` key entirely. If it was chained, removes only the `&& agent-plugin install` portion.

**Why postinstall covers add/remove/update**: Bun triggers `postinstall` for all dependency changes. When a user runs `bun add @scope/new-plugin`, `postinstall` fires and `agent-plugin install` detects the new plugin in `node_modules` and wires it. When a user runs `bun remove @scope/old-plugin`, `postinstall` fires and `agent-plugin install` detects the missing plugin and unwires it. When a user runs `bun update @scope/plugin`, `postinstall` fires and `agent-plugin install` detects the changed content and reconciles.

**Empirical verification**: During adr-review, a concern was raised that `bun remove` might not trigger `postinstall`. Empirical testing on Bun v1.3.8 confirmed that ALL package operations (add, remove, update, install) trigger `postinstall` scripts defined in the project's own `package.json`. This was validated with a test project containing a `postinstall` script and observing execution across all four operations.

**Bun trust requirement**: Bun requires the `--trust` flag to run lifecycle scripts from dependencies. For project-level postinstall scripts (defined in the project's own `package.json`), this restriction does not apply. The project's own postinstall runs without `--trust`. If agent-plugin is a dependency of another project and that project wants agent-plugin's postinstall to run, the consuming project must use `bun install --trust @acmelabs-15/agent-plugin` or add it to the `trustedDependencies` array in their `package.json`.

### Decision 4: install as Core Wiring Command

`agent-plugin install` is the single command that bridges npm packages to AI platform configurations. It is idempotent and safe to run multiple times.

**Behavior**:

1. **Scan**: Walk `node_modules` for packages containing a `plugin.json` manifest.
2. **Diff**: Compare discovered plugins against what is currently wired to AI platform configs. Determine what needs to be added, removed, or updated.
3. **Reconcile**: Apply the diff:
   - Add: Wire new plugin content (skills, agents, hooks, instructions, MCP entries) to detected platform configs using the existing platform adapter layer (ADR-009).
   - Remove: Unwire content from plugins no longer present in `node_modules`.
   - Update: Replace changed content for plugins whose `plugin.json` or content files differ from the wired state.

**Scope handling**:

- Project scope (default): Scans the project's `node_modules` and writes to project-level platform config files.
- Global scope (`--global`): Scans globally installed packages and writes to user-level platform config files.

**State derivation**: No separate state store is needed. The wired state is derived by scanning platform config files for entries that match the namespacing convention (ADR-003 Decision 1: `plugin-name:component-name`). The discovered state comes from scanning `node_modules`. The diff between these two states drives reconciliation.

**Hook merge strategy (full recompute)**: ADR-003's overlay/recompute pattern originally stored per-plugin hook contributions in `plugin-lock.json`. With the lockfile eliminated (Decision 6), hook merging uses a stateless full-recompute approach:

- Each plugin's `plugin.json` in `node_modules` declares its hook contributions (e.g., `"hooks": ["hooks/pre-commit.js"]`).
- On `agent-plugin install`, scan all `plugin.json` files from discovered plugins, compute the merged hook state from scratch.
- On remove (plugin absent from `node_modules` after `bun remove`), its hooks are naturally absent from the recompute. No explicit "undo" logic needed.
- `node_modules` IS the state. No separate tracking file, overlay, or contribution log is required.
- Performance: full recompute completes under 20ms for 10 plugins (ANALYSIS-035 benchmark). This makes the stateless approach viable even at scale.

This replaces ADR-003's overlay/recompute pattern that depended on `plugin-lock.json` to track per-plugin contributions.

**Security constraints**:

- **Direct dependencies only**: Scanning is restricted to packages listed in the project's `package.json` `dependencies` and `devDependencies` fields. Transitive dependencies (dependencies of dependencies) are NOT scanned for `plugin.json`. This prevents supply chain attacks where a transitive dependency injects unexpected plugin content.
- **Wiring summary output**: Every `agent-plugin install` invocation outputs a human-readable summary of wiring changes: plugins added, plugins removed, plugins updated, and a count of content items affected per plugin. This provides visibility into what changed and supports auditability.
- **Path validation (CWE-22 prevention)**: All file paths declared in `plugin.json` content arrays (`skills`, `agents`, `hooks`, `instructions`, `mcp`) are validated before use. The following are rejected with an error:
  - Path traversal sequences (`../` or `..\\`)
  - Absolute paths (starting with `/` or drive letter)
  - Symlinks that resolve outside the plugin's package directory
  - This prevents a malicious `plugin.json` from declaring content paths that read or write files outside the plugin's own `node_modules` subdirectory.

**Linker compatibility**: The scanner enumerates packages from `package.json` `dependencies` and `devDependencies` (per the direct-deps-only security constraint above) and resolves each to its location in `node_modules`, following symlinks. This dependency-list-driven approach (not directory-walk-driven) naturally handles both flat `node_modules` and Bun's isolated linker (pnpm-style with `.bun/` store and symlinks).

**Platform detection**: Uses the existing platform detection mechanism from ADR-010 Decision 3 Phase 1 (binary existence check + config directory check via `platforms.config.json`).

### Decision 5: plugin.json Simplified

The `version` field is REMOVED from the required fields in `plugin.json`. The `package.json` `version` field is authoritative. Plugin authors should not maintain version numbers in two places.

**Minimum required fields** (amended from ADR-001 Decision 4):

```json
{
  "name": "@scope/plugin-name",
  "description": "What this plugin does"
}
```

Two fields are required: `name` and `description`. The `version` field was previously required (ADR-001 Decision 4 specified `name`, `version`, `description`). Since plugins are now npm packages, `package.json` version is the single source of truth for versioning.

**plugin.json `name` vs package.json `name`**: These serve different purposes and may differ. `package.json` `name` is the npm package identifier (how you `bun add` it). `plugin.json` `name` is the plugin identity used for namespacing, display in `agent-plugin list`, and component prefixing (e.g., `plugin-name:skill-name`). The `name` field remains required in `plugin.json` as the plugin's display/identity name, intentionally independent of `package.json` `name`.

**plugin.json is a content declaration manifest**. Its primary purpose is declaring what content the plugin provides:

```json
{
  "name": "@scope/my-plugin",
  "description": "Code review skills for TypeScript projects",
  "installMode": "bundle",
  "skills": ["skills/review", "skills/lint"],
  "agents": ["agents/reviewer.md"],
  "hooks": ["hooks/pre-commit.js"],
  "instructions": ["instructions/coding-standards.md"],
  "mcp": "mcp/server.json"
}
```

**Separate project config eliminated**: The `.acmelabz/project.json` file referenced in the design spec (ANALYSIS-033 gap G-004) is not needed. `plugin.json` + `package.json` cover all required metadata. `plugin.json` declares content. `package.json` declares version, dependencies, author, license, repository, and build scripts. No third file is necessary.

**This amends ADR-001**: Decision 4 minimum required fields change from `{name, version, description}` to `{name, description}`. All other ADR-001 decisions remain unchanged.

### Decision 6: Lockfile Strategy

The custom `plugin-lock.json` is eliminated. `bun.lockb` is the single lockfile for the project.

**What bun.lockb provides** (that plugin-lock.json was designed to provide):

| Concern | plugin-lock.json (ADR-003 Decision 3) | bun.lockb |
|---|---|---|
| Version pinning | Custom per-plugin entries | Standard npm lockfile semantics |
| Reproducibility | Custom lockfileVersion field | Binary lockfile with deterministic resolution |
| Integrity | SHA-256 per file (IMP-008) | Built-in integrity verification |
| Source tracking | Custom source field per plugin | Standard resolution metadata |
| Dependency chain | Custom installedDeps array | Standard dependency tree |

**What is no longer tracked**: Installed system dependencies (`installedDeps` array from ADR-010 Decision 2) are no longer tracked in a lockfile because agent-plugin no longer installs system dependencies. Platform CLI installation is out of scope for a wiring tool. If a platform CLI is missing, `agent-plugin install` warns and skips that platform.

**Platform wiring state**: Not tracked in a lockfile. Wiring state is derived at runtime by scanning platform config files for namespaced entries (Decision 4). This makes the system stateless from agent-plugin's perspective. The source of truth is: (1) `node_modules` for what should be wired, and (2) platform config files for what is currently wired.

**This amends ADR-003**: Decision 3 (custom lockfile with atomic writes and lockfileVersion migration pipeline) is superseded. The lockfile concept, atomic write utilities, and migration pipeline are no longer needed.

## Consequences

### Positive

- **POS-001**: Eliminates ~2000-3000 lines of custom installation, uninstallation, and upgrade code (ADR-010 NEG-002 estimate). The 6-phase install lifecycle reduces to a single scan-diff-reconcile loop.
- **POS-002**: Eliminates custom source resolution entirely (ADR-008 Decisions 1-3). No tarball downloading, archive extraction, GitHub API calls, or URL parsing needed.
- **POS-003**: Eliminates custom lockfile management (ADR-003 Decision 3). No lockfileVersion migration pipeline, no atomic write utilities for plugin-lock.json, no custom state tracking.
- **POS-004**: Developers use a workflow they already know: `bun add` to install, `bun remove` to uninstall, `bun update` to upgrade. Zero new commands to learn for package management.
- **POS-005**: Security concerns from ADR-008 (zip slip CWE-22 CVSS 8.6, integrity verification CWE-494 CVSS 7.5) are delegated to bun, which handles them as part of standard package installation.
- **POS-006**: The `postinstall` hook model (Decision 3) provides automatic wiring without requiring users to remember to run `agent-plugin install` after every dependency change.
- **POS-007**: Stateless wiring (Decision 4) means no state file can become stale or corrupt. Truth is always derived from `node_modules` + platform configs.

### Negative

- **NEG-001**: Loses GitLab URL shorthand support. Bun does not support `gitlab:` prefix. GitLab-hosted plugins must be referenced via full git URL (`bun add git+https://gitlab.com/owner/repo.git`). ADR-008 Decision 1 supported this via custom source resolution.
- **NEG-002**: Loses bare `owner/repo` shorthand without the `github:` prefix. ADR-008 Decision 2 implemented auto-detection of GitHub repositories from bare owner/repo strings. Users must now write `bun add github:owner/repo`.
- **NEG-003**: Full dependency on bun's package management. If bun has a bug in source resolution or lockfile handling, agent-plugin cannot work around it. However, this is the same dependency model as any npm-based project.
- **NEG-004**: Loses ability to install system dependencies (platform CLIs) that ADR-010 Decision 2 provided. Users must install platform CLIs manually. This is mitigated by clear error messages when a platform is detected but its CLI is missing.
- **NEG-005**: Loses per-component install tracking in a lockfile. ADR-010 Phase 6 recorded SHA-256 hashes of installed files. The new model derives state at runtime, which means no historical audit trail of what was installed when.

### Confirmation

Implementation compliance will be verified through:

- **Code review**: PR reviewers confirm `agent-plugin install` scans only direct dependencies, outputs wiring summaries, and validates paths per CWE-22 constraints.
- **Integration tests**: Test suite covers postinstall trigger for add/remove/update operations, full recompute correctness when plugins are added and removed, and path traversal rejection.
- **Architecture audit**: Post-implementation review confirms no custom lockfile, no custom source resolution, and no consumer commands exist in the codebase.
- **ADR cross-reference**: Verify that ADR-008 and ADR-010 are marked as superseded, and ADR-001 and ADR-003 amendments are reflected in those documents.

## Implementation Notes

- **IMP-001**: Amend ADR-001 Decision 4 to change minimum required fields from `{name, version, description}` to `{name, description}`. Update Zod validation schema accordingly (ADR-007 Decision 8).
- **IMP-002**: Amend ADR-003 Decision 3 to mark the custom lockfile (`plugin-lock.json`) as superseded. Remove lockfileVersion migration pipeline from implementation scope.
- **IMP-003**: Mark ADR-008 (Source Resolution and Package Validation) as superseded by ADR-013. All source resolution is handled by bun.
- **IMP-004**: Mark ADR-010 (Installation Lifecycle) as superseded by ADR-013. The 6-phase install flow, system dependency installation, and interactive upgrade flow are replaced by `bun add/remove/update` + `agent-plugin install`.
- **IMP-005**: The `agent-plugin install` command reuses the platform detection logic from ADR-010 Phase 1 and the platform adapter layer from ADR-009. These subsystems remain valid.
- **IMP-006**: The smart merge behavior for `init` (Decision 3) must handle edge cases: multiple `&&` chains, `||` operators, semicolon-separated commands, and shell quoting. Test with common postinstall patterns.
- **IMP-007**: The scan-diff-reconcile loop (Decision 4) must handle the case where a plugin's `plugin.json` is malformed. Skip the plugin with a warning rather than failing the entire install.
- **IMP-008**: ADR-010 Decision 2 (system dependency installation) is no longer implemented. `agent-plugin install` should detect missing platform CLIs and print actionable messages (e.g., "Claude Code CLI not found. Install with: brew install claude") but not offer to install them.

## Reversibility Assessment

- [x] **Rollback capability**: All decisions can be rolled back without data loss. Re-introducing consumer commands, custom lockfile, or source resolution is additive work, not destructive.
- [x] **Vendor lock-in**: Bun dependency is medium lock-in. Alternative: migrate to npm or pnpm as the expected package manager. The wiring logic is package-manager-agnostic (it reads node_modules, which all package managers populate). Only the postinstall hook trigger and bun.lockb format are bun-specific.
- [x] **Exit strategy**: If bun proves problematic, the `agent-plugin install` command works with any package manager that populates node_modules. Only the `init` command (which writes to package.json scripts) and documentation need updating.
- [x] **Legacy impact**: No existing users. Greenfield project.
- [x] **Data migration**: No custom lockfile to migrate. No state files to convert.

## Vendor Lock-in Assessment

**Dependency**: Bun (package manager and runtime)
**Lock-in Level**: Medium

### Lock-in Indicators

- [x] Proprietary APIs without standards-based alternatives: bun.lockb is a binary format specific to bun. However, node_modules layout follows npm conventions.
- [ ] Data formats that require conversion to export: node_modules is standard. plugin.json and package.json are portable.
- [ ] Licensing terms that restrict migration: Bun uses MIT license.
- [x] Integration depth that increases switching cost: postinstall hook trigger behavior and `--trust` flag are bun-specific.
- [ ] Team training investment: Bun CLI is nearly identical to npm CLI.

### Exit Strategy

**Trigger conditions**: Bun stability issues, Bun project abandonment, or need to support npm/pnpm as primary package managers.
**Migration path**: Replace `bun add/remove/update` in documentation with `npm install/uninstall/update`. Replace `bun.lockb` references with `package-lock.json`. Update `init` command to handle npm's postinstall behavior (no `--trust` flag needed). The `agent-plugin install` command works unchanged because it reads `node_modules`, which all package managers populate.
**Estimated effort**: 1-2 days of documentation and `init` command updates. Zero changes to core wiring logic.
**Data export**: All data is in standard formats (package.json, plugin.json, node_modules).

### Accepted Trade-offs

Bun was already selected as runtime in ADR-005. This ADR deepens that dependency by relying on bun for package management (not just runtime). The trade-off is acceptable because: (1) bun's package manager is the fastest in the ecosystem, (2) bun.lockb format is an implementation detail that does not affect agent-plugin's core logic, and (3) the wiring layer reads node_modules which is a cross-manager standard.

## ADRs Affected

| ADR | Relationship | What Changes |
|---|---|---|
| ADR-001 | Amended | Decision 4: minimum required fields change from `{name, version, description}` to `{name, description}`. No separate project config file. |
| ADR-003 | Amended | Decision 3: custom lockfile (plugin-lock.json) superseded by bun.lockb. lockfileVersion migration pipeline eliminated. |
| ADR-008 | Superseded | All 3 decisions (source type detection, GitHub auto-detection, archive extraction) superseded. Bun handles source resolution. |
| ADR-010 | Superseded | All 4 decisions (install scope, dependency installation, 6-phase lifecycle, interactive upgrade) superseded. Bun + agent-plugin install replaces entire lifecycle. |
| ADR-002 | Unchanged | No publish command. Distribution via npm. Aligned. |
| ADR-007 | Partially affected | Command tree (Decision 1) changes. Consumer commands removed. Core architecture decisions unchanged. |
| ADR-009 | Unchanged | Platform detection and config registry remain valid. |
| ADR-011 | Unchanged | MCP tool auto-generation remains valid. |
| ADR-012 | Unchanged | Content-type command groups remain valid. |

## References

- [[ANALYSIS-033 Consumer and Author Commands]] — 12 conflicts, 11 gaps between spec and ADRs
- [[ANALYSIS-034 Skill Versioning Models Comparison]] — TanStack Intent, Vercel Skills, Claude Code Plugins
- TanStack Intent model — skills ship inside npm packages, package manager controls versioning
- Husky git hooks model — tool wires itself into package.json scripts
- [[ADR-005 Runtime and Distribution Strategy]] — Bun as runtime
- [[ANALYSIS-035 Hook Merging Strategies Without Custom Lockfile]] — under 20ms for 10 plugins

## More Information

The TanStack Intent model was selected over the Vercel Skills model and Claude Code Plugins model because it provides the best complexity-to-value ratio. ANALYSIS-034 documented three philosophies:

1. **TanStack Intent**: "Skills are a feature of the library." Skills version with the npm package. No separate lifecycle. Minimal custom infrastructure.
2. **Vercel Skills**: "Skills are documents in git repos." Content-addressable versioning via tree SHAs. Custom lockfile. Custom source resolution.
3. **Claude Code Plugins**: "Plugins are packages distributed through catalogs." Full marketplace infrastructure. 6 source types. Auto-update mechanisms.

agent-plugin's previous architecture (ADRs 001-012) was closest to the Vercel Skills model: custom source resolution, custom lockfile, custom lifecycle management. This ADR moves to the TanStack Intent model: npm packages, standard lockfile, wiring-only CLI.

The team reached this decision during Group 7 ideation after reviewing ANALYSIS-033's finding that the consumer command layer had 12 conflicts with existing ADRs and ANALYSIS-034's evidence that TanStack Intent achieves the same user outcomes by eliminating 3 of 5 CLI command groups, the custom lockfile, and the entire source resolution pipeline.

## Observations

- [decision] Adopt TanStack Intent model: plugins ship as npm packages, bun handles all package management, agent-plugin is a wiring tool only #distribution #architecture #simplification
- [decision] Consumer commands (add, remove, upgrade) eliminated. Bun handles these operations. Postinstall hook triggers agent-plugin install for automatic wiring. #commands #simplification
- [decision] Husky-style init: agent-plugin init adds postinstall hook to package.json with smart merge for existing scripts #lifecycle #init
- [decision] install command is scan-diff-reconcile with full recompute: scan direct-dependency plugin.json files, compute merged state from scratch, diff against wired platform configs, reconcile differences. Includes CWE-22 path validation and wiring summary output. #installation #wiring #security
- [decision] plugin.json version field removed from required fields. package.json version is authoritative. Minimum required: name, description. Amends ADR-001 Decision 4. #manifest #simplification
- [decision] Custom plugin-lock.json eliminated. bun.lockb is the single lockfile. Wiring state derived at runtime from node_modules + platform configs. Amends ADR-003 Decision 3. #lockfile #simplification
- [fact] ADR-008 (Source Resolution) superseded: bun handles all source resolution including npm, github:, git+https://, and file: paths #supersedes
- [fact] ADR-010 (Installation Lifecycle) superseded: 6-phase install flow replaced by bun add + agent-plugin install wiring loop #supersedes
- [insight] ANALYSIS-033 identified 12 conflicts between spec and ADRs in the consumer command layer, validating the decision to eliminate that layer entirely #evidence
- [insight] ANALYSIS-034 compared 3 production systems and found TanStack Intent achieves equivalent outcomes with minimal custom infrastructure. ANALYSIS-034 recommended against TanStack model, but this was overridden because ANALYSIS-033's 12 command conflicts validated the simplification. #evidence
- [fact] Bun v1.3.8 empirically verified: postinstall fires for add, remove, update, and install operations. Validated during adr-review. #verification
- [risk] GitLab URL shorthand lost (bun has no gitlab: prefix). Users must use full git URL. #tradeoff
- [risk] Full dependency on bun package management. Mitigated by node_modules being a cross-manager standard. #vendor-dependency

## Relations

- supersedes [[ADR-008 Source Resolution and Package Validation]]
- supersedes [[ADR-010 Installation Lifecycle]]
- amends [[ADR-001 Plugin Format and Manifest]]
- amends [[ADR-003 Conflict Resolution and Namespacing]]
- depends_on [[ADR-005 Runtime and Distribution Strategy]]
- relates_to [[ADR-007 CLI Architecture and Interaction Model]]
- relates_to [[ADR-009 Platform Detection and Config Registry]]
- relates_to [[ADR-012 Scaffolding and Content Management]]
- relates_to [[ANALYSIS-033 Consumer and Author Commands]]
- relates_to [[ANALYSIS-034 Skill Versioning Models Comparison]]
- relates_to [[ANALYSIS-035 Full Recompute Performance Benchmark]]