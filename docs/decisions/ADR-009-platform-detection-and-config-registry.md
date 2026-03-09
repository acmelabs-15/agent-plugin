---
title: ADR-009 Platform Detection and Config Registry
type: decision
permalink: decisions/adr-009-platform-detection-and-config-registry-1
tags:
- architecture
- decision
- platform-detection
- config-registry
- mcp-namespacing
- cross-platform
- data-driven
---

# ADR-009 Platform Detection and Config Registry

## Status

**Accepted**

**Date**: 2026-03-07
**Authors**: Peter Kloss
**Consulted**: Architect agent, Analyst agent
**Informed**: All project contributors

## Context and Problem Statement

`@acmelabs-15/agent-plugin` targets 4 AI coding platforms (ADR-002). Each platform has different binary names, config directory paths, MCP config file locations, and content directory structures. Three interconnected problems must be solved:

1. **Platform detection**: How does the CLI determine which of the 4 platforms are installed on the user's machine? Binary-only checks via `which`/`where` miss GUI editors (Cursor) that users install without adding shell commands to PATH. Config-directory-only checks may false-positive from leftover directories of uninstalled software.

2. **MCP key namespacing**: When writing MCP server entries to platform config files, how are plugin-provided server names disambiguated from user-defined servers and from other plugins' servers? The shared `mcpServers` JSON object in each platform's config file has no built-in scoping mechanism.

3. **Platform-specific mapping**: Where does the knowledge of per-platform binary names, config paths, MCP file locations, content directories, and root key differences live? Embedding this in code couples platform knowledge to runtime logic. Embedding it in `plugin.json` forces plugin authors to understand platform internals.

These problems are tightly coupled: detection determines which platforms to configure, the config registry determines where to write, and MCP namespacing determines how to write entries without collisions.

## Decision Drivers

- Reliable detection across CLI-first tools (Claude Code) and GUI editors (Cursor, Kiro IDE) where binaries may not be in PATH
- Detection must complete within a bounded time (3 seconds total for all 4 platforms) to avoid blocking the install flow
- Plugin authors must not need platform-specific knowledge; `plugin.json` remains platform-agnostic (ADR-001)
- MCP server keys must not collide with user-defined servers or other plugins in the shared JSON object
- Adding support for a new platform (platform 5+) should require zero code changes for standard-format platforms
- Consistency with existing namespacing conventions (ADR-003 uses colon for component identifiers; colon now used for both component identifiers and MCP keys)

## Considered Options

### For Detection

- **Option A: Dual detection** (binary check + config directory check, both signals)
- **Option B: Binary-only detection** (check PATH for platform binary)
- **Option C: Directory-only detection** (check for config directory existence)
- **Option D: User-specified platforms** (prompt user to declare installed platforms)

### For MCP Key Namespacing

- **Option E: Slash separator** (`plugin-name/server-name`)
- **Option F: Double-dash separator** (`agentplugin--plugin-name--server-name`)
- **Option G: Colon separator** (`plugin-name:server-name`) -- chosen

### For Platform Config Registry

- **Option H: Static JSON data file** (`platforms.config.json` at agent-plugin project root)
- **Option I: Code-based adapters** (per-platform TypeScript modules with hardcoded paths)
- **Option J: Platform block in plugin.json** (plugin authors declare per-platform paths)

## Decision Outcome

Three decisions, one per problem.

### Decision 1: Dual Platform Detection (Binary + Config Directory)

**Chosen option: Option A (dual detection)**, because neither signal alone is reliable across all 4 platforms.

Detection uses two signals for each platform:

1. **Binary check**: `which` (macOS/Linux) or `where` (Windows) to find the platform's CLI binary in PATH
2. **Config directory check**: `fs.existsSync()` on the platform's known config directory path

All 4 platform checks run in parallel using `Promise.allSettled` with `Bun.spawn({ timeout: 3000 })` for binary checks. A platform is considered detected if either signal is positive. The detection result records which signal(s) matched (`binary`, `directory`, or `both`) for diagnostic purposes.

**Why dual**: Binary-only misses GUI editors (Cursor) where users install the desktop app but never run "Install shell command" from the Command Palette. Directory-only may false-positive from config directories left behind by uninstalled software. Both signals together provide high confidence. ANALYSIS-026 documents per-platform binary names and config paths across all 3 operating systems.

**Parallel execution**: `Promise.allSettled` ensures one platform timing out (e.g., network-mounted home directory slowing `which`) does not block detection of the other 3. The 3-second timeout per binary check is generous; `which`/`where` completes in under 50ms on typical systems.

**Platform override flag**: The `--platform` flag (ADR-007 Decision 8, `--platforms` for install) allows users to explicitly specify target platforms, bypassing auto-detection entirely. This handles edge cases where detection fails (e.g., non-standard install locations, containerized environments) and enables CI pipelines to declare platforms declaratively. When `--platform` is provided, the Detect phase is skipped and the specified platforms are used directly.

**Platform detection summary** (from ANALYSIS-026):

| Platform | Binary Name | Config Dir (macOS) | Config Dir (Linux) |
|---|---|---|---|
| Claude Code | `claude` | `~/.claude/` | `~/.claude/` |
| Cursor | `cursor` | `~/Library/Application Support/Cursor/` | `~/.config/Cursor/` |
| Copilot CLI | `copilot` (or `gh copilot` extension) | `~/.copilot/` | `~/.copilot/` |
| Kiro | `kiro-cli` | `~/.kiro/` | `~/.kiro/` |

### Decision 2: MCP Key Namespacing with Colon Separator

**Chosen option: Option G (colon separator)**, because it matches Claude Code's internal convention, avoids JSON Pointer escaping conflicts, and aligns with ADR-003's existing colon convention for component identifiers.

When writing MCP server entries to platform config files, keys use the format `plugin-name:server-name`. Example: a plugin named `db-tools` providing a server named `postgres` writes the key `db-tools:postgres` into the platform's MCP config JSON.

The MCP server object itself (command, args, env) is standard across all 4 supported platforms. Only the container file path differs per platform. All 4 platforms use the `mcpServers` root key. The namespaced key lives inside this root key.

**Why colon**: Claude Code (the primary target platform, ADR-002) already uses colon internally for plugin namespacing (`plugin:<name>:<server>` per GitHub issue #15145). Matching this convention means zero friction on the primary platform. Colon also avoids JSON Pointer (RFC 6901) escaping issues -- slash requires `~1` escaping in JSON Pointer paths, and jq requires bracket notation for slash-containing keys. ADR-003 already uses colon for component identifiers, making colon the established separator convention in this project. The MCP protocol is actively standardizing namespacing via SEP-993; colon is the lowest-regret choice given current evidence.

**Collision prevention**: The `plugin-name:` prefix ensures a plugin's MCP servers cannot collide with user-defined servers (which have no colon prefix) or with other plugins' servers (which have a different prefix). Combined with ADR-003's kebab-case name validation (`^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`), the key space is deterministic and collision-free.

### Decision 3: Platform Config Registry via platforms.config.json

**Chosen option: Option H (static JSON data file)**, because it separates platform knowledge from both plugin authoring and runtime code.

All platform-specific mapping lives in a single static JSON data file named `platforms.config.json` at the agent-plugin project root (the tool's own repository, not user projects). This file contains:

- **Config paths per OS**: macOS, Linux, and Windows paths for each platform's config directory
- **Binary names**: CLI binary names for platform detection
- **MCP config locations**: project-level and user-level MCP config file paths
- **Content directory mappings**: where each platform expects skills, agents, prompts to be placed
- **Detection methods**: binary name + config directory path per OS
- **Env var overrides**: environment variables that override default config paths (e.g., `XDG_CONFIG_HOME`)
- **Root key**: `mcpServers` (consistent across all 4 platforms)

**Plugin author experience**: Authors write platform-agnostic `plugin.json` (ADR-001). No `platforms` block, no per-platform knowledge required. The tool reads both files at install time: `plugin.json` (what the plugin provides) and `platforms.config.json` (where and how to write it).

**Adding a new platform**: Adding support for platform 5+ requires adding a JSON entry to `platforms.config.json` for standard-format platforms. No code changes, no new adapter modules, no new TypeScript files. The entry specifies binary name, config paths per OS, MCP config location, root key, env var overrides, and content directories. Platforms with non-standard MCP formats would require a format transformer in the runtime code in addition to the JSON entry.

**Claude Code user-scoped MCP config**: Claude Code supports both project-level (`.mcp.json` in project root) and user-level (`~/.claude.json`) MCP configuration. The `platforms.config.json` entry for Claude Code includes both paths, enabling installs to either scope.

**Env var overrides**: The `platforms.config.json` schema includes an `envOverrides` field per platform that lists environment variables to check for non-standard paths. Detection expands these variables before checking paths. Of the 4 supported platforms, Cursor on Linux respects `XDG_CONFIG_HOME` for its config directory. The remaining 3 platforms use fixed config paths with no env var overrides.

**Schema example** (`platforms.config.json`, showing 2 of 4 platforms):

```json
{
  "platforms": {
    "claude-code": {
      "displayName": "Claude Code",
      "binaryNames": ["claude"],
      "configDirs": {
        "macos": "~/.claude/",
        "linux": "~/.claude/",
        "windows": "%USERPROFILE%\\.claude\\"
      },
      "mcpConfig": {
        "project": ".mcp.json",
        "user": "~/.claude.json",
        "rootKey": "mcpServers"
      },
      "envOverrides": {},
      "contentDirs": {
        "skills": ".claude/skills/",
        "agents": ".claude/agents/",
        "commands": ".claude/commands/"
      }
    },
    "cursor": {
      "displayName": "Cursor",
      "binaryNames": ["cursor"],
      "configDirs": {
        "macos": "~/Library/Application Support/Cursor/",
        "linux": "~/.config/Cursor/",
        "windows": "%APPDATA%\\Cursor\\"
      },
      "mcpConfig": {
        "project": ".cursor/mcp.json",
        "user": "~/.cursor/mcp.json",
        "rootKey": "mcpServers"
      },
      "envOverrides": {},
      "contentDirs": {
        "skills": ".cursor/skills/",
        "agents": ".cursor/agents/",
        "commands": ".cursor/commands/"
      }
    }
  }
}
```

All 4 supported platforms use the standard `{ command, args, env }` MCP object format. The schema retains a `formatTransformer` field in `mcpConfig` for future platforms that may require non-standard formats; the runtime would dispatch to a named transformer function that converts the standard shape to the platform's expected format.

**Maintenance and distribution**: `platforms.config.json` is bundled inside the `@acmelabs-15/agent-plugin` npm package and ships with every release. When a platform changes its config paths (e.g., Kiro migrating from `.amazonq/` to `.kiro/`), the file is updated and published as a new package version. Users receive updates through normal `npm update` or `bun update` flows. No remote fetching or dynamic update mechanism is needed -- the file is static data that changes at the same cadence as the tool itself.

**Why not code-based adapters (Option I)**: Code-based adapters couple platform knowledge to TypeScript modules. Each new platform requires a new module, tests, and a build. ANALYSIS-029 refined ANALYSIS-027's adapter approach to this data-driven design, where the adapter logic reads from the data file rather than encoding paths in source code.

**Why not plugin.json (Option J)**: Forcing plugin authors to specify per-platform paths contradicts ADR-001's decision that all plugins are inherently cross-platform with no `platforms` field. Plugin authors should not need to know that Kiro uses `.kiro/settings/mcp.json` while Cursor uses `.cursor/mcp.json`.

## Consequences

### Positive

- **POS-001**: Dual detection achieves high reliability across all 4 platforms. GUI editors without PATH binaries are caught by directory check. Uninstalled tools with leftover directories are confirmed by binary check. ANALYSIS-026 rates overall detection reliability as "High" for all 4 platforms.
- **POS-002**: Parallel detection via `Promise.allSettled` with 3-second timeouts bounds total detection time. One slow check does not block the other 3.
- **POS-003**: Colon-separated MCP keys (`plugin-name:server-name`) match Claude Code's internal convention and ADR-003's component identifier convention. Avoids JSON Pointer (RFC 6901) escaping conflicts.
- **POS-004**: MCP key collision is impossible under the namespacing scheme. User-defined servers have no colon prefix. Plugin servers are scoped to their plugin name.
- **POS-005**: `platforms.config.json` reduces the cost of adding a standard-format platform from "write a TypeScript module with tests" to "add a JSON entry." Community contributors can add standard-format platform support without understanding runtime internals.
- **POS-006**: Plugin authors remain platform-agnostic. `plugin.json` contains zero platform-specific knowledge. The tool handles all translation.
- **POS-007**: Static JSON is parseable by any language or tool, enabling future non-TypeScript integrations or validation scripts.
- **POS-008**: Maintenance strategy is zero-overhead: `platforms.config.json` ships inside the npm package and updates arrive through normal `npm update` or `bun update` flows. No remote config fetching, no dynamic update mechanism, no separate versioning to track.

### Negative

- **NEG-001**: Dual detection runs 4 binary checks (subprocesses) plus 4 directory existence checks on every install. On typical systems this completes in under 200ms total, but on slow filesystems or network-mounted homes the 3-second timeouts could extend detection to 12 seconds (4 platforms x 3 seconds). Mitigation: `Promise.allSettled` parallelism caps this at 3 seconds for the binary checks plus negligible time for `fs.existsSync`. Additional mitigation: the `--platform` flag allows users to bypass detection entirely on slow filesystems, specifying target platforms directly.
- **NEG-002**: `platforms.config.json` is a second file that must stay synchronized with the runtime logic that reads it. Schema validation of this file is needed to catch malformed entries before runtime.
- **NEG-003**: Directory-only detection can still false-positive for platforms whose config directories persist after uninstall. The install flow should treat directory-only detection as lower confidence and optionally confirm with the user.
- **NEG-004**: Colon in MCP keys (`db-tools:postgres`) could be confused with URI scheme syntax (e.g., `http:`) in some contexts. This is cosmetic only; MCP keys are plain string identifiers in JSON objects, not URIs.
- **NEG-005**: Static JSON cannot express conditional logic (e.g., "use this path on macOS 14+ but that path on macOS 13"). If platform config paths become version-dependent, the data file format must evolve or a thin code layer must supplement it.

## Pros and Cons of the Options

### Option A: Dual Detection (Chosen)

Binary check via `which`/`where` combined with config directory existence check.

- Good, because catches GUI editors not in PATH via directory fallback
- Good, because confirms uninstalled tools with leftover directories via binary check
- Good, because parallel execution bounds total time regardless of individual failures
- Neutral, because spawns 4 subprocesses for binary checks (negligible overhead)
- Bad, because directory-only matches have lower confidence than dual matches

### Option B: Binary-Only Detection (Rejected)

Check PATH for platform binary using `which`/`where`.

- Good, because simple implementation with single signal
- Bad, because misses GUI editors (Cursor) where user has not installed shell command
- Bad, because ANALYSIS-026 rates Cursor binary-in-PATH likelihood as "Medium"

### Option C: Directory-Only Detection (Rejected)

Check for config directory existence using `fs.existsSync()`.

- Good, because no subprocess overhead
- Bad, because false-positives from leftover directories of uninstalled software
- Bad, because cannot distinguish "installed and configured" from "was installed, now removed"

### Option D: User-Specified Platforms (Rejected)

Prompt user to declare which platforms they have installed.

- Good, because zero false positives when user answers correctly
- Bad, because fails in CI pipelines, Docker builds, and MCP server contexts (no interactive terminal per ADR-002)
- Bad, because users may not know which platforms are installed (especially in managed environments)
- Bad, because answer goes stale when user installs or removes a platform

### Option E: Slash Separator (Rejected)

MCP keys formatted as `plugin-name/server-name`.

- Good, because consistent with directory-based namespacing
- Bad, because JSON Pointer (RFC 6901) requires escaping slash as `~1`
- Bad, because jq requires bracket notation for slash-containing keys
- Bad, because ADR-003 rejected slash for component namespacing (different context but precedent)

### Option F: Double-Dash Separator (Rejected)

MCP keys formatted as `agentplugin--plugin-name--server-name`.

- Good, because unambiguous (no other convention uses double-dash)
- Bad, because verbose (42 characters for a simple key vs 20 with slash)
- Bad, because hard to read in JSON config files
- Bad, because the `agentplugin` prefix is redundant since only agent-plugin writes namespaced keys

### Option G: Colon Separator (Chosen)

MCP keys formatted as `plugin-name:server-name`.

- Good, because Claude Code (primary platform) uses colon internally (`plugin:<name>:<server>`)
- Good, because consistent with ADR-003 component identifier convention
- Good, because no JSON Pointer escaping or jq bracket notation required
- Good, because MCP keys are strings and colons are valid in JSON keys
- Neutral, because MCP protocol is standardizing namespacing via SEP-993; colon is lowest-regret choice

### Option H: Static JSON Data File (Chosen)

All platform mapping in `platforms.config.json` at agent-plugin project root.

- Good, because adding a standard-format platform requires zero code changes (non-standard formats need a format transformer)
- Good, because separates platform knowledge from runtime logic and from plugin authoring
- Good, because parseable by any language or tool
- Neutral, because requires schema validation to catch malformed entries
- Bad, because cannot express conditional logic tied to platform versions

### Option I: Code-Based Adapters (Rejected)

Per-platform TypeScript modules with hardcoded paths and detection logic.

- Good, because can express conditional logic and complex detection
- Bad, because each new platform requires a new module, tests, and a build
- Bad, because platform knowledge is coupled to TypeScript; cannot be consumed by other tools
- Bad, because ANALYSIS-029 demonstrated this can be simplified to data + thin reader

### Option J: Platform Block in plugin.json (Rejected)

Plugin authors declare per-platform paths in their manifest.

- Good, because plugin author has full control over where content is written
- Bad, because contradicts ADR-001 decision that plugins are inherently cross-platform with no `platforms` field
- Bad, because forces plugin authors to track 4 platforms' path conventions
- Bad, because path changes in platform updates would require updating every published plugin

## Implementation Notes

- **IMP-001**: Binary detection uses `Bun.spawn([whichCmd, binaryName], { timeout: 3000 })` where `whichCmd` is `which` on macOS/Linux and `where` on Windows. Exit code 0 means binary found.
- **IMP-002**: Config directory detection uses `fs.existsSync(expandedPath)` where `expandedPath` resolves `~`, `$HOME`, `%USERPROFILE%`, and `%APPDATA%` per OS. Env var overrides from the `envOverrides` field in `platforms.config.json` are expanded via `process.env` lookups before path resolution (e.g., checking `XDG_CONFIG_HOME` before defaulting to `~/.config/Cursor/` on Linux).
- **IMP-003**: Detection results include a `method` field (`binary`, `directory`, or `both`) for diagnostics and confidence assessment.
- **IMP-004**: `platforms.config.json` should be validated against a JSON Schema at build time and at tool startup. Malformed entries should produce actionable error messages.
- **IMP-005**: The MCP key `plugin-name:server-name` is written into the platform's root key object. For Claude Code, this means `mcpServers["plugin-name:server-name"]`. All 4 supported platforms use the `mcpServers` root key.
- **IMP-006**: MCP server objects contain `command`, `args`, and `env` fields. These are identical across all 4 supported platforms. The `formatTransformer` field in `platforms.config.json` is reserved for future platforms with non-standard formats.
- **IMP-007**: The install flow reads `platforms.config.json` during the Detect phase (which platforms exist) and the Apply phase (where to write content).
- **IMP-008**: `platforms.config.json` is bundled in the `@acmelabs-15/agent-plugin` npm package, validated against its JSON Schema at startup, and versioned alongside the tool. Platform path changes ship as new package versions through normal npm/bun update flows.
- **IMP-009**: The `--platform` flag skips the Detect phase and uses the specified platforms directly. Combined with `--ci`, this enables fully deterministic non-interactive installs. Users can also use `--platform` when auto-detection fails due to non-standard install locations, containerized environments, or slow filesystems.

## Reversibility Assessment

- [x] **Rollback capability**: Switching detection strategy requires only changing the detection function. No data migration needed. MCP key format change requires rewriting existing entries in platform config files, which the lockfile tracks.
- [x] **Vendor lock-in**: No new vendor lock-in. `Bun.spawn` has a direct Node.js equivalent (`child_process.exec`). `fs.existsSync` is a standard Node.js API. `platforms.config.json` is plain JSON.
- [x] **Exit strategy**: `platforms.config.json` can be replaced with code-based adapters by reading the JSON file and converting entries to TypeScript objects. The data format is the superset.
- [x] **Legacy impact**: No existing installations to migrate. Greenfield project.
- [x] **Data migration**: MCP key format change (`plugin-name:server-name` to another format) requires updating entries in platform config files. The lockfile tracks all installed MCP entries, enabling automated rewriting.

## Confirmation

Implementation compliance will be verified through:

1. Integration tests running dual detection on all 4 platforms across macOS, Linux, and Windows CI runners
2. Unit tests verifying `platforms.config.json` schema validation catches missing fields and malformed entries
3. Unit tests verifying MCP key format matches `plugin-name:server-name` pattern with kebab-case validation
4. Property-based tests verifying that MCP keys from different plugins never collide
5. Acceptance test verifying that adding a JSON entry to `platforms.config.json` enables detection and installation for a new platform without code changes

## References

- [[ANALYSIS-026 Platform Detection and Mapping]] — binary names, config paths, detection strategy, parallel implementation
- [[ANALYSIS-029 Platform Config Registry]] — platforms.config.json design, data-driven approach, MCP key namespacing
- [[ADR-001 Plugin Format and Manifest]] — plugin.json is platform-agnostic, no platforms field
- [[ADR-002 Target Platforms and Audiences]] — 4 target platforms, 3 audience types
- [[ADR-003 Conflict Resolution and Namespacing]] — colon separator for component identifiers, kebab-case validation
- [[ADR-005 Runtime and Distribution Strategy]] — Bun runtime, Bun.spawn for subprocess management

## Observations

- [decision] Dual platform detection adopted: binary check via which/where AND config directory existence check, run in parallel for all 4 platforms #platform-detection #reliability
- [decision] Promise.allSettled with Bun.spawn timeout of 3000ms ensures parallel detection completes within bounded time regardless of individual failures #parallel-detection #performance
- [decision] MCP key namespacing uses colon separator (plugin-name:server-name), matching Claude Code internal convention and ADR-003 component identifier pattern #mcp #namespacing
- [decision] All platform-specific mapping consolidated into platforms.config.json as a static JSON data file at the agent-plugin project root #platform-config #data-driven
- [decision] Adding a standard-format platform requires only a JSON entry in platforms.config.json. Non-standard-format platforms would additionally require a format transformer in runtime code #extensibility #data-driven
- [fact] All 4 supported platforms use the mcpServers root key, simplifying MCP config writes #mcp #platform-consistency
- [fact] All 4 supported platforms use the standard MCP format (command, args, env). No format transformers needed for current platform set. #mcp #format-consistency
- [fact] Claude Code supports both project-level (.mcp.json) and user-level (~/.claude.json) MCP configuration #claude-code #mcp-scope
- [fact] Of 4 supported platforms, only Cursor on Linux uses env var overrides (XDG_CONFIG_HOME for config directory). The remaining 3 platforms use fixed config paths. #env-overrides #detection
- [fact] GUI editor Cursor has Medium likelihood of binary in PATH, requiring directory fallback for reliable detection #detection-reliability
- [fact] Copilot CLI users may have only the gh extension (gh copilot) without a standalone copilot binary. binaryNames field supports arrays to check both copilot and gh with subcommand verification #copilot #detection-gap
- [insight] Colon chosen over slash for MCP keys because Claude Code uses colon internally, JSON Pointer RFC 6901 requires slash escaping as ~1, and ADR-003 establishes colon as the project's identifier separator convention #namespacing #consistency
- [insight] Static JSON chosen over code-based adapters because ANALYSIS-029 demonstrated platform knowledge can be fully expressed as data, enabling community contributions without runtime code changes #architecture #simplification
- [constraint] Plugin authors write platform-agnostic plugin.json with no platforms block; agent-plugin reads both plugin.json and platforms.config.json at install time #authoring #separation-of-concerns
- [decision] Dual detection kept for v1: low implementation cost (parallel async with Promise.allSettled + fs.existsSync), works for GUI editors from day one without retrofitting. Debate P0-7 simplification to binary-only rejected. #platform-detection #p0-resolution
- [decision] --platform flag added as override: bypasses auto-detection entirely, enables CI pipelines to declare platforms declaratively, mitigates slow filesystem timeout concern (NEG-001) #platform-override #ci

## Relations

- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-002 Target Platforms and Audiences]]
- extends [[ADR-003 Conflict Resolution and Namespacing]]
- depends_on [[ADR-005 Runtime and Distribution Strategy]]
- relates_to [[ANALYSIS-026 Platform Detection and Mapping]]
- relates_to [[ANALYSIS-029-platform-config-registry]]

### Amendment #1: Platform Reduction and Complete Content Directory Mapping (2026-03-09)

Per ADR-002 Amendment #1, supported platforms reduced from 7 to 4. The `platforms.config.json` schema is updated with complete content directory mappings for all 8 content types across the 4 supported platforms.

**Complete `platforms.config.json` content directory mappings**:

| Content Type | Claude Code | Cursor | GitHub Copilot | Kiro |
|---|---|---|---|---|
| Skills | `.claude/skills/` | `.cursor/skills/` | `.github/skills/` | `.kiro/skills/` |
| Agents | `.claude/agents/` (.md) | `.cursor/agents/` (.md) | `.github/agents/` (.agent.md) | `.kiro/agents/` (.json) |
| Commands | `.claude/commands/` (.md) | `.cursor/commands/` (.md) | `.github/prompts/` (.prompt.md) | `.kiro/commands/` (.md) |
| Rules | `.claude/rules/` (.md) | `.cursor/rules/` (.mdc/.md) | `.github/instructions/` (.instructions.md) | `.kiro/steering/` (.md) |
| AGENTS.md | CLAUDE.md (root) | AGENTS.md (root) | AGENTS.md (root) | AGENTS.md (root) |
| Hooks | `.claude/settings.json` | `.cursor/hooks.json` + `.cursor/hooks/` | `.github/hooks/*.json` | `.kiro/hooks/*.kiro.hook` |
| MCP | `.mcp.json` | `.cursor/mcp.json` | `~/.copilot/mcp-config.json` | `.kiro/settings/mcp.json` |

**Format differences requiring platform adapters**:
- Agents: Kiro uses JSON, Copilot uses `.agent.md` extension, Claude Code and Cursor use `.md`
- Commands: Copilot uses `.prompt.md` with different frontmatter schema
- Rules: Cursor supports MDC format with inclusion modes, Copilot uses `applyTo` globs, Kiro uses inclusion frontmatter
- Hooks: All 4 platforms use JSON but with different schemas and event names
- AGENTS.md: Claude Code uses CLAUDE.md instead

**Dropped platform entries removed**: OpenCode, Amp, Windsurf entries removed from `platforms.config.json`. The D1 detection table and D3 schema example have been updated inline to reflect only the 4 supported platforms.
