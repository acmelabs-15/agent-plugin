---
title: ADR-003 Conflict Resolution and Namespacing
type: decision
permalink: decisions/adr-003-conflict-resolution-and-namespacing-1
tags:
- architecture
- conflict-resolution
- namespacing
- hooks
- lockfile
- platform-config
- sanitization
---

# ADR-003 Conflict Resolution and Namespacing

## Status

**Accepted** (revised 2026-03-07 after unanimous adr-review feedback)

**Date**: 2026-03-07
**Authors**: Peter Kloss
**Consulted**: adr-review panel (architect, critic, independent-thinker, security, analyst, high-level-advisor)
**Informed**: All project contributors

## Context and Problem Statement

`@acmelabz/agent-plugin` is a cross-platform AI agent plugin manager. Plugins are bundles containing multiple component types (skills, agents, prompts, hooks, MCP). When installing plugins from different sources, four problems must be solved:

1. **Naming collisions**: Multiple plugins may define components with the same name (e.g., two plugins both provide a skill called `code-review`). Without disambiguation, the second install silently overwrites the first.
2. **Hook merging**: Hook components have different collision semantics than named components. Multiple hooks on the same event are additive, not conflicting. Blocking hooks (e.g., Claude Code PreToolUse) require a resolution rule when plugins disagree.
3. **Platform configuration**: Frontmatter fields vary across 7 target platforms. A universal superset would include unsupported fields on some platforms. Authors need a way to declare platform-specific settings without knowing each platform's field names.
4. **State tracking**: Plugin updates and removals require knowing which installed components belong to which plugin, what files were modified, and whether the state file is corrupted.

These concerns are tightly coupled: the namespacing strategy determines whether conflict resolution prompts are needed, which affects state tracking complexity, which affects the lockfile design.

## Decision Drivers

- Eliminate user-facing conflict resolution prompts (CI pipelines, Docker builds, MCP server audience cannot use interactive prompts per ADR-002)
- Support both `bundle` and `collection` installModes from ADR-001 without special-casing
- Deterministic, order-independent hook merging across all platforms
- Cross-platform compatibility (Windows NTFS, macOS, Linux) for all naming conventions
- Defense against plugin-authored injection (CWE-94, prompt injection, prototype pollution)
- Lockfile must survive corruption without data loss

## Considered Options

- **Option A: Always-namespace** (auto-prefix every component with `plugin-name:component-name`)
- **Option B: User-choice conflict resolution** (interactive prompt asking user which component to rename)
- **Option C: Blanket conflict blocking** (`conflicts` field in manifest, fail on match)
- **Option D: No namespacing** (flat names, first-installed wins)
- **Option E: Slash separator** (`plugin-name/component-name`)

## Decision Outcome

**Chosen option: Option A (always-namespace)**, because it eliminates conflict resolution entirely. Every installed component is automatically prefixed with `plugin-name:component-name`. No user prompts, no rename tracking, no cross-reference updates, no state store for mappings. Works identically for bundle and collection installModes, in CI and interactive contexts, across all 7 target platforms.

This ADR covers 6 interconnected decisions:

### Decision 1: Always-Namespace with Colon Separator

Every installed component is named `plugin-name:component-name`, following the Claude Code convention. This is automatic and unconditional.

**The colon is a logical-only identifier.** It appears in component registries, help text, and cross-references. It NEVER appears in filenames. Windows NTFS forbids colons in filenames. macOS Finder replaces them. Claude Code itself uses colon as a display convention, not in filenames on disk.

**Name validation**: Both plugin names and component names must match kebab-case: `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`. This prevents namespace spoofing (a plugin named `foo:bar` would be ambiguous), path traversal via names, and special character injection.

### Decision 2: Hook Merge via Overlay/Recompute Pattern

Hook merging uses the overlay/recompute pattern (ANALYSIS-010). Each plugin's hook contributions are stored as a dedicated section within the lockfile (`plugin-lock.json`). On install, update, or uninstall, ALL plugin hook sections are read and merged into the platform's configuration using `deepmerge` (ANALYSIS-012).

**Merge semantics**:

- Objects merge recursively (event groups contain matcher groups)
- Arrays concatenate (hook command lists append, never replace)
- Booleans use **strictest wins** via `deepmerge` `customMerge` option: if ANY plugin blocks an action, the action is blocked. This matches Claude Code's native behavior where any PreToolUse hook returning "deny" blocks the tool call.

**Order independence**: Plugin hook sections are sorted alphabetically by plugin name before merging. Installation order is irrelevant. `deepmerge.all([sortedPluginHooks], hookMergeOptions)` produces identical output for identical plugin sets.

**User-owned hooks**: On first plugin install, existing hooks are snapshotted as "user-owned" within the lockfile. Plugin hook contributions layer on top of the user snapshot. User hooks are never lost during merge or unmerge.

**Unmerge**: Remove the plugin's hook section from the lockfile, recompute merged result from remaining plugin sections. Clean removal without touching other plugins' contributions.

**Composability constraint**: Strictest-wins means a restrictive plugin can block actions that a permissive plugin requires. For example, a security plugin blocking `Bash` tool calls would prevent a deployment plugin from functioning. Users must understand that installing a restrictive plugin affects all other plugins' capabilities. Mitigation is scoped to ADR-004 (per-plugin hook scope, user override, or conflict notification).

**Security**: Hook merge security (per-hook user consent for blocking hooks, CWE-94 mitigation) is deferred to ADR-004. Hook commands validated by the sanitization pipeline (Decision 5) are stored in the lockfile but are NOT executable until ADR-004 establishes the consent policy.

### Decision 3: JSON Lockfile with Atomic Writes

Plugin state is tracked in a single lockfile (ANALYSIS-011). No separate state directory is needed — hook overlay data, plugin inventory, and component registry all live in the lockfile.

**Location**:

- **Project-scoped** (default): `./plugin-lock.json` at project root, alongside `plugin.json`
- **User-scoped** (global install): `~/.config/agent-plugin/plugin-lock.json` (XDG-compliant)

**Format**: JSON with `lockfileVersion` (integer, starting at 1) for schema evolution. Contains plugin inventory, component registry per plugin, hook contributions per plugin per platform, user-owned hook snapshot, and file modification history.

```json
{
  "lockfileVersion": 1,
  "_integrity": "sha256-...",
  "userHooks": { "claude-code": { "...snapshotted user hooks..." } },
  "plugins": {
    "linting-plugin": {
      "version": "1.2.0",
      "components": { "skills": ["code-review"], "hooks": ["pre-tool"] },
      "hooks": {
        "claude-code": {
          "PreToolUse": [{ "matcher": "Bash", "command": "lint-check" }]
        }
      }
    }
  }
}
```

**Atomic writes**: The `atomically` npm package writes to a temp file then renames atomically.

**Corruption recovery**: The lockfile is a cache, not a source of truth. Installed plugin files on disk ARE the source of truth. Recovery is layered:

1. Try `JSON.parse()`
2. On failure, try `plugin-lock.json.bak` (backup from last successful write, same directory)
3. On failure, re-derive from disk scan of installed plugins

**Integrity**: SHA-256 hash stored in `_integrity` field (self-excluded from hash computation). Detects external modification. Warn-only, does not block operations.

**Schema migration**: Integer `lockfileVersion` with sequential migration pipeline. Each migration transforms v(N) to v(N+1). Matches npm's proven model.

**File permissions**: The lockfile and its backup are written with mode 600. The XDG config directory (`~/.config/agent-plugin/`) is created with mode 700.

### Decision 4: Hybrid Platform Configuration (Adapter + Layered Overrides)

Platform-specific configuration uses a hybrid D+C pattern (ANALYSIS-014): adapter translation for cross-platform concepts, with layered overrides for platform-specific capabilities.

**4-level resolution order** (higher overrides lower):

| Level | Source | Example |
|-------|--------|---------|
| 1 | Agent Skills standard fields | `name`, `description`, `allowed-tools` |
| 2 | Adapter default mapping | `loadingStrategy: "always"` maps to `alwaysApply: true` on Cursor |
| 3 | Manifest-level `platformConfig` in plugin.json | `platformConfig.claude-code.model: "sonnet"` |
| 4 | Per-component `platforms` block in SKILL.md frontmatter | `platforms.claude-code.model: "opus"` |

**Cross-platform concepts** (3 abstract fields that map across all 7 platforms):

- `loadingStrategy`: always, file-conditional, manual, auto-detect
- `filePatterns`: glob patterns for conditional loading
- `approvedTools`: tool allowlist (maps to `allowed-tools` on Claude Code, ignored on platforms without support)

80% of plugins need only levels 1-2 (standard fields + adapter). 15% add level 3 (manifest platformConfig). 5% use level 4 (per-component overrides).

### Decision 5: Sanitization Pipeline

All plugin-authored content passes through a layered validation pipeline before installation (ANALYSIS-013):

| Layer | Purpose | Implementation |
|-------|---------|---------------|
| 1. Schema validation | Reject wrong types, missing fields, unexpected keys | Zod v4 `.strict()` mode |
| 2. Value constraints | Length limits, character restrictions, format (semver, kebab-case) | Zod `.max()`, `.regex()`, `.refine()` |
| 3. Content sanitization | Strip dangerous patterns from text fields | Zod `.transform()` + `validator.js` |
| 4. Path validation | Reject traversal, absolute paths, disallowed extensions | `path.resolve()` + containment check |
| 5. Hook command validation | Parse commands, reject operators (pipes, redirects, chains) | `shell-quote` AST inspection |
| 6. Output escaping | Context-specific escaping for JSON, YAML, markdown, shell | Platform adapters |

**Security-critical fields** (`permissionMode`, `allowed-tools` at the plugin level) are NEVER accepted from plugin manifests. They are user-controlled only.

**Package selection**: Zod v4 (65M weekly downloads, TypeScript-first, 14x faster string parsing in v4), `shell-quote` (22.7M weekly downloads), `validator` (15.5M weekly downloads). No HTML sanitization libraries needed (output targets AI agents, not browsers).

### Decision 6: Forward Reference to ADR-004

Hook merge security, per-hook user consent for blocking hooks, and CWE-94 mitigation for hook command execution are scoped to ADR-004. This ADR defines the merge mechanics; ADR-004 defines the security policy governing what hooks are allowed to do.

## Consequences

### Positive

- **POS-001**: Always-namespace eliminates conflict resolution UI, rename tracking, cross-reference updates, and interactive prompts. Reduces installer complexity by removing 3 subsystems.
- **POS-002**: Always-namespace works identically for CI, Docker, MCP server, and interactive audiences. No `--strategy` flag needed.
- **POS-003**: Overlay/recompute hook merge provides inherent provenance (lockfile section = owner), clean unmerge (remove section + recompute), and deterministic output regardless of installation order. Single lockfile eliminates separate state directory.
- **POS-004**: Strictest-wins for blocking hooks matches Claude Code's native behavior and follows the Microsoft Intune security policy precedent.
- **POS-005**: JSON lockfile as re-derivable cache eliminates single point of failure. Lockfile corruption does not orphan installed plugins.
- **POS-006**: Atomic writes via `atomically` prevent corruption from crashes or power loss.
- **POS-007**: 4-level platform config resolution lets 80% of plugins use 3 abstract fields while giving 5% access to full platform-specific capabilities.
- **POS-008**: Zod v4 schema validation with strict mode rejects unknown keys, providing prototype pollution protection at the schema level.
- **POS-009**: Colon-is-logical-only avoids all filesystem escaping issues across Windows, macOS, and Linux.
- **POS-010**: Kebab-case name validation prevents namespace spoofing, path traversal via names, and YAML key-value ambiguity.

### Negative

- **NEG-001**: Always-namespace produces longer component names. `my-linting-plugin:code-review` is 32 characters vs `code-review` at 11.
- **NEG-002**: Platform adapter layer must maintain a field mapping table for each of 7 platforms. This is an ongoing maintenance burden as platforms evolve.
- **NEG-003**: Overlay/recompute overwrites manual edits to the merged hooks output. Users must edit their own hooks (preserved via snapshot in lockfile), not the merged output.
- **NEG-004**: Custom prompt injection detection (regex-based) catches obvious attacks but can be evaded by sophisticated adversaries. Medium confidence for this vector.
- **NEG-005**: No existing npm package combines JSON merging + provenance tracking + clean unmerge. The overlay system requires custom implementation (estimated medium effort per ANALYSIS-010).
- **NEG-006**: The `deepmerge` package (v4.3.1) has not published a new release in 3 years. This could indicate stability or slow abandonment. `deepmerge-ts` is the actively maintained TypeScript alternative if migration becomes necessary.
- **NEG-007**: Strictest-wins hook merge can make plugins non-composable when their security postures conflict. A restrictive plugin blocking `Bash` tool calls would prevent a deployment plugin from functioning. Mitigation scoped to ADR-004 (per-plugin hook scope, user override, or conflict notification).

## Pros and Cons of the Options

### Option A: Always-Namespace (Chosen)

Auto-prefix every installed component with `plugin-name:component-name`.

- Good, because eliminates conflict resolution UI, rename tracking, and cross-reference updates (3 fewer subsystems)
- Good, because works for CI, Docker, MCP, and interactive audiences without configuration
- Good, because works identically for bundle and collection installModes
- Good, because deterministic: same plugin always produces same installed names
- Neutral, because longer component names (32 chars vs 11 in the example above)
- Bad, because users cannot install a component under a shorter custom name

### Option B: User-Choice Conflict Resolution (Rejected)

Ask the user which component to rename when a collision is detected.

- Good, because preserves user agency (user chooses which component to modify)
- Bad, because fails in CI pipelines, Docker builds, and MCP server contexts (no interactive terminal)
- Bad, because requires rename tracking state store (original-name to installed-name mappings)
- Bad, because requires cross-reference updates when a component is renamed (AST-aware parsing across YAML+Markdown+JSON with no off-the-shelf solution)
- Bad, because 50-component plugin with 8 conflicts produces 8 sequential prompts (prompt fatigue)
- Bad, because installMode interaction undefined (does 1 conflict in a 15-component bundle fail the entire bundle?)

### Option C: Blanket Conflict Blocking (Rejected)

Define a `conflicts` array in the manifest. Installation fails if any listed plugin is already installed.

- Bad, because too blunt. Does not handle simple name collisions between otherwise compatible plugins.
- Bad, because forces plugin authors to predict all future conflicts at authoring time.

### Option D: No Namespacing (Rejected)

Install components with original names. First-installed wins.

- Bad, because collision risk grows linearly with installed plugin count.
- Bad, because no disambiguation path for users who need both components.

### Option E: Slash Separator (Rejected)

Use `plugin-name/component-name` as the namespace pattern.

- Bad, because slash collides with directory separators on Unix-like systems.
- Bad, because causes ambiguity in file path resolution on some platforms.

## Implementation Notes

- **IMP-001**: The colon separator MUST be logical-only. Installed files use the component name portion only (or a platform-appropriate path structure). The full `plugin:component` identifier appears in registries, CLI output, and configuration cross-references.
- **IMP-002**: Name validation regex `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$` applies to both plugin names and component names independently. Validation runs at manifest parse time via Zod schema.
- **IMP-003**: Hook contributions are stored per-plugin within the lockfile under `plugins.{plugin-name}.hooks.{platform}`. No separate overlay files or state directory needed.
- **IMP-004**: The `deepmerge` `customMerge` option dispatches per-key: boolean fields named `blocking` or `deny` use logical OR (strictest wins); all other fields use default deep merge. Arrays always concatenate via `arrayMerge: (target, source) => [...target, ...source]`.
- **IMP-005**: Project-scoped lockfile at `./plugin-lock.json` (project root). User-scoped lockfile at `~/.config/agent-plugin/plugin-lock.json` (XDG). Backup at `plugin-lock.json.bak` in the same directory.
- **IMP-006**: Lockfile writes use `atomically` (npm package). Reads validate `_integrity` hash and `lockfileVersion`. Migration functions transform v(N) to v(N+1).
- **IMP-007**: Platform adapters implement a concept-to-field mapping table. The adapter for Cursor maps `loadingStrategy: "always"` to `alwaysApply: true`. The adapter for Kiro maps it to `inclusion: "always"`. Each adapter is a separate module.
- **IMP-008**: Zod v4 schemas define the exact shape of `plugin.json`, component frontmatter, and `platformConfig`. All use `.strict()` mode. The `__proto__`, `constructor`, and `prototype` keys are explicitly rejected via `.refine()`.
- **IMP-009**: Hook commands are parsed with `shell-quote` at install time. Commands containing pipes (`|`), redirects (`>`, `<`), subshells (`$()`, backticks), command chains (`;`, `&&`, `||`), or environment variable references (`$VAR`) are rejected.
- **IMP-010**: Deterministic JSON serialization uses sorted keys and 2-space indentation for both the lockfile and hook merge output. This minimizes git diff noise.

## Reversibility Assessment

- [x] **Rollback capability**: Switching from always-namespace to a different strategy requires renaming installed components, but no data loss occurs. The lockfile tracks all installed components.
- [x] **Vendor lock-in**: No new vendor lock-in. All dependencies (`deepmerge`, `atomically`, `zod`, `shell-quote`, `validator`) are MIT/ISC licensed open-source packages with alternatives.
- [x] **Exit strategy**: `deepmerge` can be replaced with `deepmerge-ts` (same API pattern). `atomically` can be replaced with `write-file-atomic` (npm's standard). Zod can be replaced with AJV + manual transforms.
- [x] **Legacy impact**: Existing plugins (none yet, greenfield) are unaffected. First-mover advantage: no migration needed.
- [x] **Data migration**: The lockfile re-derives from disk. Changing the lockfile schema requires only a new migration function, not data conversion.

## Confirmation

Implementation compliance will be confirmed via:

1. Unit tests verifying always-namespace naming for every component type
2. Property-based tests verifying hook merge determinism (same overlays, different processing orders, identical output)
3. Integration tests verifying lockfile corruption recovery (delete lockfile, re-derive from disk, compare with original)
4. Zod schema tests verifying rejection of `__proto__`, `constructor`, `prototype` keys
5. Shell-quote tests verifying rejection of operator-containing hook commands

## References

- **REF-001**: Claude Code component naming conventions (colon separator pattern as display convention)
- **REF-002**: ANALYSIS-010 Hook Merge/Unmerge Patterns (overlay/recompute architecture, 40+ sources)
- **REF-003**: ANALYSIS-011 Lockfile Management Patterns (`atomically`, integer versioning, re-derive on corruption)
- **REF-004**: ANALYSIS-012 JSON Config Merge Patterns (`deepmerge` selection, strictest-wins via `customMerge`)
- **REF-005**: ANALYSIS-013 Input Sanitization Patterns (Zod v4 + shell-quote + validator pipeline)
- **REF-006**: ANALYSIS-014 Platform Config Patterns (hybrid D+C, 4-level resolution, cross-platform concepts)
- **REF-007**: ADR-001 Plugin Format and Manifest (bundle/collection installModes, plugin.json schema)
- **REF-008**: ADR-002 Target Platforms and Audiences (7 platforms, 3 audience types including CI/MCP)
- **REF-009**: DEBATE-ADR-003 (unanimous Needs Revision, 7 P0 + 14 P1 issues, all resolved in this revision)

## Observations

- [decision] Always-namespace adopted: auto-prefix every component with plugin-name:component-name, eliminating conflict resolution UI, rename tracking, and cross-reference updates #namespacing #simplification
- [decision] Colon is logical-only identifier, never in filenames; kebab-case validation enforced via regex for both plugin and component names #namespacing #cross-platform
- [decision] Overlay/recompute pattern for hook merging: each plugin's hooks stored as sections within the lockfile, merged deterministically via deepmerge with customMerge for strictest-wins booleans #hooks #architecture
- [decision] Single lockfile at project root (plugin-lock.json) or user home (~/.config/agent-plugin/plugin-lock.json) with atomically for atomic writes, integer lockfileVersion for schema evolution, re-derive from disk on corruption. No separate state directory needed. #lockfile #state-management
- [risk] Strictest-wins hook merge can make plugins non-composable when security postures conflict; mitigation scoped to ADR-004 #hooks #composability
- [decision] Hybrid D+C platformConfig with 4-level resolution: standard fields, adapter mapping, manifest platformConfig, per-component platforms block #platform-config #cross-platform
- [decision] Sanitization via Zod v4 strict mode + shell-quote + validator in a 6-layer pipeline from schema validation through output escaping #security #validation
- [decision] Hook merge security deferred to ADR-004: per-hook user consent for blocking hooks, CWE-94 mitigation #security #forward-reference
- [constraint] Colon forbidden in filenames (NTFS, Finder); plugin/component names validated as kebab-case to prevent namespace spoofing #cross-platform #filesystem
- [fact] Overlay/recompute is the dominant pattern across 12 production systems: systemd, Kustomize, Docker Compose, NixOS, Terraform, Git config #prior-art
- [risk] deepmerge v4.3.1 unchanged for 3 years; deepmerge-ts is the actively maintained alternative if migration needed #maintenance #dependency
- [insight] Always-namespace eliminates 3 subsystems (conflict resolution UI, rename tracking, cross-reference updates) that the original ADR-003 required #simplification

## Relations

- extends [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-002 Target Platforms and Audiences]]
- relates_to [[ANALYSIS-010-hook-merge-unmerge-patterns]]
- relates_to [[ANALYSIS-011-lockfile-management-patterns]]
- relates_to [[ANALYSIS-012-json-config-merge-patterns]]
- relates_to [[ANALYSIS-013-input-sanitization-patterns]]
- relates_to [[ANALYSIS-014-platform-config-patterns]]
- relates_to [[DEBATE-ADR-003-conflict-resolution-and-namespacing]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]


