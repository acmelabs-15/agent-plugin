---
title: ADR-010 Installation Lifecycle
type: decision
status: accepted
date: 2026-03-07
decision-makers: Peter Kloss
permalink: decisions/adr-010-installation-lifecycle-1
tags:
- decision
- installation
- lifecycle
- scope
- dependencies
- upgrade
- mcp
---

# ADR-010 Installation Lifecycle

## Status

**Accepted**

**Consulted**: Architect agent, Analyst agent, Critic agent
**Informed**: All project contributors

## Context and Problem Statement

`@acmelabs-15/agent-plugin` is a cross-platform CLI with an embedded MCP server for managing AI agent plugins across 7 platforms (ADR-002). ADR-003 established always-namespace, overlay/recompute hooks, JSON lockfile with atomic writes, and hybrid D+C platform config. ADR-005 established Bun as runtime. ADR-007 defined the CLI command tree, three-tier input resolution, and @clack/prompts component mapping.

What ADR-003 and ADR-007 left unspecified:

1. **Install scope**: Should project or global be the default? What happens when no project directory is detected?
2. **System dependencies**: Should the tool auto-install platform CLIs and system tools, or only check them?
3. **Installation phases**: What is the exact sequence of operations during install, and how does the user control platform selection?
4. **Upgrade mechanics**: How do users discover and apply updates to installed plugins?

ANALYSIS-027 (installation mechanics) researched install scope patterns across npm, mise, proto, asdf, and Vercel Skills. ANALYSIS-029 (platform config registry) defined `platforms.config.json` as the static data source for platform-specific knowledge. This ADR codifies the decisions that emerged from those analyses and subsequent team discussion. CLI generation (previously Decision 5 in this ADR) has been extracted to ADR-011.

## Decision Drivers

- Per-project reproducibility: plugins installed at project scope produce identical environments across team members
- CI/non-interactive compatibility: all install operations must work without prompts when `--ci` or `--yes` flags are passed (ADR-007 Decision 3)
- User consent for system changes: installing platform CLIs or system dependencies modifies the user's system outside the project directory
- Atomicity: partial installs must never corrupt project state
- Clean uninstall: everything installed must be removable without residue

## Considered Options

### Install Scope Default

- Option A: Project scope default with `--global` flag (chosen)
- Option B: Global scope default (Homebrew model)
- Option C: Always prompt for scope

### System Dependency Policy

- Option A: Check-only, never auto-install (ANALYSIS-027 initial recommendation)
- Option B: Auto-install with explicit user confirmation (chosen)
- Option C: Auto-install silently

## Decision Outcome

Four interconnected decisions govern the installation lifecycle. Each is documented below.

### Decision 1: Install Scope -- Project Default with Interactive Fallback

**Chosen option**: Project scope by default, `--global` flag for user-wide installation.

Project scope means all plugin content writes to the current project directory. This matches npm (`npm install` = local, `npm install -g` = global), mise (`mise use` = local, `mise use -g` = global), proto (`--pin local` default), and Vercel Skills (project scope default). Per-project installation provides reproducibility: every team member gets the same plugins.

**No-project fallback**:

When no project directory is detected (no `plugin.json`, `package.json`, `.git`, or other project indicators in the directory tree) and `--global` is not passed:

- **Interactive mode**: prompt the user via `@clack/prompts` confirm: "No project directory detected. Install globally?" If confirmed, proceed with global scope. If cancelled, exit cleanly.
- **Non-interactive mode** (CI, piped input, `--ci` flag): error with message "No project directory detected. Use --global for user-wide installation." Exit code 2 (usage error, per ADR-007 Decision 10).

This follows ADR-007 Decision 4 (three-tier input resolution): interactive mode prompts, CI mode errors with flag hint.

**Scope affects 4 targets**:

| Target | Project Scope | Global Scope |
|---|---|---|
| Plugin files | Project root directories | `~/.config/agent-plugin/plugins/` |
| Lockfile | `./plugin-lock.json` | `~/.config/agent-plugin/plugin-lock.json` |
| Platform config | Project-level config files | User-level config files |
| MCP server entries (colon-namespaced keys per ADR-009, e.g., `plugin:server`) | Project-level MCP config | User-level MCP config |

### Decision 2: Dependency Installation with User Confirmation

**Chosen option**: Agent-plugin CAN install platform CLIs and system dependencies, but ONLY with explicit user confirmation via multiselect prompt.

**Key distinction**: Dependencies are defined by the agent-plugin package itself in `platforms.config.json`, NOT by plugin authors in `plugin.json`. The `systemDependencies` field does not exist in the plugin manifest schema. Plugin authors never specify install commands. The agent-plugin package knows which binaries each platform requires (e.g., "claude-code needs the `claude` binary, installable via `brew install claude`") and encodes that knowledge in its own trusted source code.

ANALYSIS-027 initially recommended check-only (never auto-install) based on supply chain security concerns (500,000+ malicious packages in 2024, CWE-78 arbitrary code execution from untrusted manifests). The debate (DEBATE-ADR-010, P0-2) raised this again with a CVSS 9.1 rating. The revised approach mitigates that concern: install commands come from the trusted agent-plugin package code, not from untrusted plugin manifests. The security model is equivalent to any npm package running postinstall scripts. Users trust the agent-plugin package when they install it.

**How it works**:

1. During the Resolve phase (Decision 3, Phase 3), the tool identifies all missing dependencies by consulting `platforms.config.json` for the selected platforms. This includes the full dependency chain: if `claude` requires `brew` and `brew` is also missing, both appear in the list.
2. Missing dependencies are presented via a `@clack/prompts` multiselect (not confirm) with all items selected by default. The prompt message reads: "These required dependencies are missing and need to be installed. Unselect any you don't want the package to install for you."
3. The user sees every missing dependency with its install method and can deselect any they want to handle manually.
4. All installed dependencies are tracked in `plugin-lock.json` for clean uninstall.

**Example install prompt**:

```text
⚠ These required dependencies are missing and need to be installed.
  Unselect any you don't want the package to install for you:

  [x] brew (Homebrew) — required to install other dependencies
  [x] claude (brew install claude)
  [x] node >= 18 (brew install node)
```

**Example uninstall prompt**:

```text
  These dependencies were installed by agent-plugin during install.
  Select which ones to uninstall:

  [x] claude (brew uninstall claude)
  [ ] node >= 18 (brew uninstall node)
```

**Why the security calculus changed**:

The CVSS 9.1 concern (P0-2) targeted arbitrary code execution from plugin manifests. That threat no longer applies because:

1. Install commands originate from `platforms.config.json` inside the agent-plugin npm package, not from plugin author-controlled `plugin.json` files.
2. A compromised `platforms.config.json` requires a supply chain attack on the agent-plugin package itself. This is the same threat model as any npm package (postinstall scripts, bin entries). No additional attack surface is created.
3. Plugin authors cannot inject install commands. The `systemDependencies` field was removed from the plugin manifest schema entirely.

**Non-interactive mode**: When `--ci` or `--yes` flags are passed, all missing dependencies from `platforms.config.json` are auto-confirmed. The `--ci` flag implies the CI environment has all dependencies pre-installed or the user accepts the install plan. If a dependency install fails in CI mode, the entire operation rolls back (Decision 3 rollback).

**Dependency tracking in lockfile**:

```json
{
  "installedDeps": [
    { "name": "brew", "method": "curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh | bash", "source": "platforms.config.json", "installedAt": "2026-03-07T10:00:00Z" },
    { "name": "claude", "method": "brew install claude", "source": "platforms.config.json", "installedAt": "2026-03-07T10:00:05Z" },
    { "name": "node", "method": "brew install node", "source": "platforms.config.json", "installedAt": "2026-03-07T10:00:10Z" }
  ]
}
```

Note: `installedDeps` is a top-level lockfile field, not per-plugin. Dependencies are package-level concerns (from `platforms.config.json`), not plugin-level.

On uninstall, the tool presents a reverse multiselect of dependencies it installed, letting the user choose which to remove. Dependencies shared across active platform configurations are flagged but not blocked from removal.

### Decision 3: Six-Phase Installation Flow

Installation proceeds in 6 sequential phases. Each phase is a checkpoint. On failure, rollback reverses completed phases in reverse order. Phases 1-3 are read-only (detection, selection, resolution) and make no changes to the filesystem or configuration. Only Phases 4-6 require rollback.

```text
Phase 1: DETECT
  Scan for all 7 platforms using platforms.config.json (ANALYSIS-029).
  Detection uses binary existence check + config directory check, run in parallel.
  Output: list of detected platforms with their config file paths.

Phase 2: SELECT
  Present @clack/prompts multiselect showing:
  - Detected platforms: pre-checked (selected by default)
  - Undetected platforms: unchecked but selectable (selecting triggers dependency install)
  Output: user's platform selection.

Phase 3: RESOLVE
  Fetch the plugin package from its source (npm, git, local path).
  Validate plugin.json schema via Zod (ADR-003 Decision 5).
  Determine everything needed for the selected platforms:
  - Missing platform CLIs for selected undetected platforms (from platforms.config.json)
  - Missing package manager dependencies in the chain (e.g., brew needed to install claude)
  - Files to write (namespaced per ADR-003 Decision 1)
  - MCP server entries to merge (colon-separated namespaced keys per ADR-009 Decision 2, e.g., `db-tools:postgres`)
  - Hook contributions to merge (overlay/recompute per ADR-003 Decision 2)
  Note: all dependency definitions come from platforms.config.json, not plugin.json.
  Output: complete install plan.

Phase 4: CONFIRM
  If missing dependencies exist: present @clack/prompts multiselect with all
  missing deps selected by default. User can deselect any they prefer to
  install manually. Message: "These required dependencies are missing and
  need to be installed. Unselect any you don't want the package to install
  for you."
  Then present @clack/prompts confirm showing the full install plan:
  - N files to install across M platforms
  - N MCP server entries to add
  - N hook contributions to merge
  - N dependencies to install (user-confirmed subset)
  In CI mode (--ci or --yes): auto-confirmed.
  Output: user confirmation.

Phase 5: APPLY
  Before any modifications: create lockfile backup (.bak) if lockfile exists.
  Execute the install plan in order:
  1. Install missing dependencies (platform CLIs, system tools)
  2. Copy/transform plugin files to target directories (atomic writes)
  3. Merge MCP server entries into platform config files
  4. Merge hook contributions via overlay/recompute
  Each operation recorded in an in-memory transaction log.
  Pre-merge snapshots of all platform config files taken before modification.
  Output: all changes applied.

Phase 6: RECORD
  Write lockfile with all tracking data:
  - Plugin metadata (name, version, source, scope)
  - Installed components per platform
  - MCP keys added per platform config file
  - Hook contributions per platform
  - Installed dependencies
  - File inventory with SHA-256 hashes
  Update lockfile backup (.bak) to reflect final state.
  Remove staging directory (if used).
  Output: installation complete.
```

**Rollback on failure**: If any phase fails, rollback executes completed phases in reverse:

| Failed Phase | Rollback Actions |
|---|---|
| Phase 6 (Record) | Restore lockfile from `.bak` (created in Phase 5) |
| Phase 5 (Apply) | Restore config snapshots, delete installed files. Dependency uninstall is best-effort and may fail (no mainstream package manager supports reliable rollback). |
| Phase 4 (Confirm) | User cancelled; no changes made |
| Phase 3 (Resolve) | Clean up fetched package; no changes to project (read-only phase) |
| Phase 2 (Select) | User cancelled; no changes made (read-only phase) |
| Phase 1 (Detect) | No changes made (read-only phase) |

**Progress reporting**: Each phase reports via @clack/prompts spinner (interactive) or plain text lines (CI mode), per ADR-007 Decision 7.

### Decision 4: Interactive Upgrade Flow

Upgrade uses an interactive flow with atomic replace per plugin.

**Upgrade command behavior** (`agent-plugin upgrade`):

1. Scan all installed plugins from the lockfile.
2. Check latest versions from each plugin's original source (npm registry, git repo, local path).
3. Present `@clack/prompts` multiselect showing plugins with newer versions available:

```text
Select plugins to upgrade:
  [x] @scope/linting-plugin  1.2.0 -> 2.0.0
  [x] @scope/docs-plugin     0.9.1 -> 0.9.2
  [ ] @scope/test-plugin      1.0.0 -> 1.1.0
```

1. Each selected plugin upgrades via install-then-remove (ANALYSIS-027 recommendation):
   a. Install new version to staging area (full 6-phase flow from Decision 3, writing to temporary paths)
   b. Swap: atomically replace old files with new files, update MCP entries, recompute hook contributions
   c. Remove old version artifacts no longer referenced
   d. If upgrade fails mid-way, the old version remains intact (no window with no plugin)
**Version pinning**: `agent-plugin upgrade @scope/plugin@2.0.0` upgrades to a specific version. Without a version specifier, upgrades to latest.

**Downgrade warning**: If the specified version is older than installed, warn and proceed only with `--force`.

**Breaking change detection**: Compare old and new manifests for removed components, removed MCP servers, and changed system dependencies. Warn before proceeding.

**Non-interactive mode**: `--ci` or `--yes` auto-selects all available upgrades. `agent-plugin upgrade @scope/plugin --ci` upgrades a specific plugin without prompts.

### Uninstall Flow

Uninstall reverses the install flow. The command `agent-plugin uninstall <plugin-name>` executes these steps:

1. **Locate**: Read the plugin's entry from `plugin-lock.json`. Error if the plugin is not installed.
2. **Remove files**: Delete all files listed in the lockfile's file inventory for this plugin (namespaced directories).
3. **Remove MCP entries**: Delete all colon-namespaced MCP server keys added by this plugin from platform config files.
4. **Remove hook contributions**: Remove this plugin's hook contributions from the lockfile and recompute merged hooks (overlay/recompute per ADR-003 Decision 2).
5. **Offer dependency uninstall**: If dependencies were installed by agent-plugin for this plugin (tracked in `installedDeps`), present a `@clack/prompts` multiselect (none selected by default) offering to uninstall them. Dependencies still required by other installed plugins are flagged. Dependency uninstall is best-effort (see NEG-004).
6. **Update lockfile**: Remove the plugin entry from `plugin-lock.json`. Write lockfile atomically.

In non-interactive mode (`--ci` or `--yes`), skip the dependency uninstall prompt (dependencies are kept unless `--remove-deps` is passed).

Decision 5 (CLI generation) has been extracted to [[ADR-011 Auto-Generated CLI from MCP Tools]].

## Consequences

### Positive

- **POS-001**: Project scope default provides per-project reproducibility. Team members get identical plugin environments by sharing `plugin-lock.json`.
- **POS-002**: Interactive fallback when no project is detected prevents silent failures and guides the user to the correct scope.
- **POS-003**: Dependency installation with confirmation eliminates the friction of manual dependency setup while preserving user consent. Users see the full plan before any system modification.
- **POS-004**: 6-phase install flow with checkpoint-based rollback prevents partial installs from corrupting project state. Pre-merge snapshots enable clean config restoration. Atomicity applies to file writes and config merges; dependency rollback is best-effort (see NEG-004).
- **POS-005**: Interactive upgrade with multiselect gives users control over which plugins to update and surfaces breaking changes before they take effect.
- **POS-006**: Atomic replace for upgrades eliminates the merge conflict class entirely. No incremental patching, no diff computation, no user modification preservation (managed files are not user-editable).
- **POS-007**: All installed dependencies tracked in lockfile enables clean uninstall without orphaned system tools.

### Negative

- **NEG-001**: Dependency installation increases the tool's attack surface compared to check-only. However, install commands come from the trusted agent-plugin package (`platforms.config.json`), not from plugin authors. The remaining risk is a supply chain attack on the agent-plugin npm package itself, which is the same risk as any npm dependency with postinstall scripts. Mitigation: user sees the full install plan via multiselect before confirmation; `platforms.config.json` is code-reviewed as part of the package.
- **NEG-002**: 6 install phases add implementation complexity. Each phase needs its own rollback handler. Estimated 2000-3000 lines of TypeScript across install, uninstall, and upgrade commands (ANALYSIS-027 estimate).
- **NEG-003**: Atomic replace for upgrades re-installs all files even when only one file changed. For large plugins, this is slower than incremental patching. Acceptable tradeoff: AI agent plugins are typically less than 1 MB total.
- **NEG-004**: Phase 5 rollback for system dependencies is not atomic. Zero mainstream package managers (npm, pip, cargo, brew) implement reliable uninstall rollback. `brew uninstall` may fail if other packages depend on the target. Dependency uninstall is best-effort; the tool logs failures and advises the user to clean up manually.

## Alternatives Considered

### Install Scope: Global Default (Rejected)

- Good, because simpler mental model for users who install system-wide tools
- Bad, because violates per-project reproducibility principle. Two projects would share all plugins.
- Bad, because contradicts npm, mise, proto, asdf conventions that all default to project scope
- Bad, because global installs modify user-level config files shared across all projects

### Dependency Policy: Check-Only, Never Auto-Install (Rejected)

ANALYSIS-027 initially recommended this approach based on supply chain security concerns. The debate (DEBATE-ADR-010, P0-2) raised it again with a CVSS 9.1 rating for arbitrary code execution from plugin manifests.

- Good, because zero risk of installing malicious dependencies
- Bad, because forces users to manually install each missing dependency, reading and following printed instructions
- Bad, because breaks the flow: user runs `agent-plugin install`, gets a list of missing deps, leaves the terminal to install them, then re-runs the command
- Rejected because the revised approach eliminates the CVSS 9.1 concern entirely: dependency definitions come from the trusted agent-plugin package (`platforms.config.json`), not from plugin author manifests. The `systemDependencies` field was removed from `plugin.json`. The remaining attack vector (compromised `platforms.config.json`) is equivalent to any npm package running arbitrary code via postinstall scripts. Multiselect confirmation with all deps visible provides user agency.

### Dependency Policy: Silent Auto-Install (Rejected)

- Bad, because modifies the user's system without consent
- Bad, because 500,000+ malicious packages detected in 2024 makes supply chain risk real
- Bad, because violates principle of least surprise

## Implementation Notes

- **IMP-001**: Platform detection (Phase 1) runs binary checks and config directory checks in parallel using `Promise.allSettled()` for maximum speed. A platform is "detected" when both the binary exists AND the config directory exists.
- **IMP-002**: The `platforms.config.json` registry (ANALYSIS-029) provides all platform-specific paths, binary names, and config key mappings. No platform knowledge is hardcoded in install logic.
- **IMP-003**: Upgrade's breaking change detection compares component lists between old and new manifests. Removed components are flagged. Renamed components are treated as remove + add (namespace handles this cleanly per ADR-003).
- **IMP-004**: Dependency install commands are defined in `platforms.config.json` alongside platform detection data. Each platform entry specifies its binary name, detection method, and install command per OS/package-manager combination. This is trusted code shipped with the agent-plugin package, not user-supplied input.
- **IMP-005**: Package manager dependency chain resolution: if a platform dependency requires a package manager (e.g., `brew install claude` requires Homebrew), and that package manager is also missing, both appear in the missing dependencies list. The chain is resolved from `platforms.config.json` dependency metadata.
- **IMP-006**: Install-time dependency prompt uses `@clack/prompts` multiselect with all missing deps selected by default. The user can deselect any dependency to handle it manually. Deselected required dependencies cause the install to abort with an actionable message.
- **IMP-007**: Uninstall-time dependency prompt uses a reverse `@clack/prompts` multiselect showing only dependencies that were installed by agent-plugin (tracked in `plugin-lock.json` `installedDeps`). None are selected by default. The user selects which to remove.
- **IMP-008**: Pre-install integrity verification (CWE-494 mitigation). Plugin source integrity MUST be verified before extraction/installation:
  - **npm**: Verify tarball integrity against the registry `dist.integrity` field (SHA-512) before extraction. Reject packages where the downloaded tarball hash does not match the registry-published hash.
  - **git**: Pin to commit SHA in the lockfile after first install. On upgrade, verify the resolved commit SHA matches the expected tag/branch target. Warn if the tag has been force-pushed (SHA changed for same tag).
  - **local**: Compute and store a content hash (SHA-256 of all plugin files) on first install. On subsequent installs or upgrades, warn if the hash has changed since last recorded state.

## Confirmation

Implementation compliance will be verified through:

- [ ] Project scope is the default; `--global` flag switches to user scope
- [ ] No-project fallback prompts in interactive mode, errors in CI mode
- [ ] Dependency installation requires explicit user confirmation via @clack/prompts
- [ ] All 6 install phases execute in order with progress reporting
- [ ] Rollback reverses all changes when any phase fails (tested with induced failures at each phase)
- [ ] Upgrade multiselect shows correct version comparisons
- [ ] All installed dependencies tracked in lockfile and removable on uninstall
- [ ] Plugin source integrity verified before extraction (npm SHA-512, git commit SHA, local content hash)
- [ ] MCP server entries use colon-namespaced keys per ADR-009 Decision 2
- [ ] Uninstall removes all installed artifacts (files, MCP entries, hook contributions, lockfile entry)

## Reversibility Assessment

- [x] **Rollback capability**: All decisions are reversible without data loss. Install scope can be changed. Dependency policy can be tightened to check-only.
- [x] **Vendor lock-in**: No new vendor dependencies introduced. @clack/prompts (already adopted in ADR-006), gunshi (already adopted in ADR-006), and lockfile format (already defined in ADR-003) are the same dependencies.
- [x] **Exit strategy**: If dependency installation is problematic, revert to check-only policy.
- [x] **Legacy impact**: Greenfield project. No existing installations to migrate.
- [x] **Data migration**: Lockfile schema changes use integer `lockfileVersion` migration pipeline (ADR-003 Decision 3).

## References

- **REF-001**: ANALYSIS-027 Installation Mechanics (install scope patterns, system dependency management, 5-phase flow, upgrade mechanics)
- **REF-002**: ANALYSIS-029 Platform Config Registry (platforms.config.json, MCP key namespacing)
- **REF-007**: ADR-009 Platform Detection and Config Registry (MCP key namespacing with colon separator, Decision 2)
- **REF-003**: ADR-003 Conflict Resolution and Namespacing (always-namespace, overlay/recompute hooks, lockfile design, sanitization pipeline)
- **REF-004**: ADR-005 Runtime and Distribution Strategy (Bun runtime, distribution channels)
- **REF-005**: ADR-007 CLI Architecture and Interaction Model (command tree, three-tier input resolution, @clack/prompts mapping)
- **REF-006**: ADR-011 Auto-Generated CLI from MCP Tools (extracted from this ADR's original Decision 5)

## Observations

- [decision] Project scope is the default install target; `--global` flag required for user-wide installation, matching npm/mise/proto/asdf conventions #scope #installation
- [decision] No-project fallback: interactive mode prompts via @clack/prompts confirm; non-interactive mode errors with flag hint (exit code 2) #scope #three-tier
- [decision] Agent-plugin CAN install platform CLIs and system dependencies with explicit user confirmation via @clack/prompts multiselect. Dependencies defined by agent-plugin package (platforms.config.json), NOT by plugin authors. systemDependencies field removed from plugin.json. CVSS 9.1 concern (P0-2) mitigated by trusted source. #dependencies #user-consent #security
- [decision] All installed dependencies tracked in plugin-lock.json installedDeps (top-level, not per-plugin) for clean uninstall via reverse multiselect prompt #dependencies #lockfile
- [decision] 6-phase install flow: Detect, Select, Resolve, Confirm, Apply, Record. Checkpoint-based rollback reverses completed phases on failure #installation #atomicity
- [decision] Platform detection uses binary + config directory dual check in parallel via platforms.config.json registry #detection #platforms
- [decision] Upgrade is interactive: multiselect of plugins with newer versions, atomic replace per plugin (remove old, install new), rollback to previous on failure #upgrade #atomic-replace
- [risk] Dependency installation increases attack surface vs check-only. Mitigated by: (1) install commands from trusted package code not plugin manifests, (2) multiselect prompt showing exact commands, (3) remaining risk equivalent to any npm package with postinstall scripts #dependencies #security
- [decision] CLI generation content extracted to ADR-011 per debate P0-1 consensus (5/6 reviewers agreed on split) #architecture #extraction
- [constraint] Phase 5 dependency rollback is best-effort, not atomic. Zero mainstream package managers support reliable uninstall rollback. Atomicity claim scoped to file writes and config merges only. #rollback #honesty
- [decision] Pre-install integrity verification required: npm SHA-512 against registry dist.integrity, git commit SHA pinning in lockfile, local content hash on first install (CWE-494 mitigation) #security #integrity
- [decision] MCP server entries use colon-separated namespaced keys per ADR-009 Decision 2 (e.g., db-tools:postgres) #mcp #namespacing
- [decision] Upgrade uses install-then-remove order per ANALYSIS-027 to avoid a window with no plugin #upgrade #reliability
- [decision] Uninstall flow specified: remove files, remove MCP entries, recompute hooks, offer dep uninstall, update lockfile #uninstall #lifecycle
- [fact] Phases 1-3 are read-only (no filesystem or config changes); only Phases 4-6 require rollback #phases #simplification
- [decision] Lockfile backup (.bak) created before Phase 5 modifications, not in Phase 6 #rollback #timing

## Relations

- depends_on [[ADR-003 Conflict Resolution and Namespacing]]
- depends_on [[ADR-005 Runtime and Distribution Strategy]]
- relates_to [[ADR-007 CLI Architecture and Interaction Model]]
- relates_to [[ANALYSIS-027-installation-mechanics]]
- relates_to [[ANALYSIS-029-platform-config-registry]]
- depends_on [[ADR-009 Platform Detection and Config Registry]]
- relates_to [[ADR-011 Auto-Generated CLI from MCP Tools]]
