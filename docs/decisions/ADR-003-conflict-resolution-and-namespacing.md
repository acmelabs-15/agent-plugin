---
title: ADR-003 Conflict Resolution and Namespacing
type: decision
permalink: decisions/adr-003-conflict-resolution-and-namespacing-1
tags:
- architecture
- conflict-resolution
- namespacing
- plugin-install
---

# ADR-003 Conflict Resolution and Namespacing

---
title: "ADR-003: Conflict Resolution and Namespacing"
status: "Accepted"
date: "2026-03-07"
authors: "Peter Kloss"
tags: ["architecture", "conflict-resolution", "namespacing", "plugin-install"]
supersedes: ""
superseded_by: ""
---

## Status

**Accepted**

## Context

`@acmelabz/agent-plugin` is a cross-platform AI agent plugin manager. Plugins are bundles containing multiple component types (skills, agents, prompts, hooks, MCP). When installing plugins from different sources, conflicts can arise between components from different plugins or with already-installed components.

Key forces at play:

- Multiple plugins may define components with the same name (e.g., two plugins both provide a skill called `code-review`)
- The system targets multiple AI coding platforms (Claude Code, Cursor, Windsurf, etc.), each with its own conventions for naming and frontmatter
- Component cross-references within a plugin must remain valid after conflict resolution renames
- Hook components have different collision semantics than named components — multiple hooks on the same event are additive, not conflicting
- Plugin updates and removals must work correctly even after conflict-resolution renames
- Frontmatter fields vary across target platforms; a universal superset would include unsupported fields on some platforms

## Decision

Five interconnected decisions were made to address conflict resolution, namespacing, and cross-platform component formatting:

### 1. Colon Namespace Separator

Installed components use the `plugin-name:component-name` pattern (borrowed from Claude Code conventions). Platform adapters translate this canonical format to whatever separator or naming pattern the target platform expects.

### 2. Intelligent Conflict Resolution Per Component Type

Conflict handling is differentiated by component type:

- **Name collisions (skills, agents, prompts, MCP):** The installer asks the user to prefix or rename the colliding component. The user chooses WHICH component to modify — the already-installed one OR the incoming one. Cross-references within the affected plugin's files are updated automatically.
- **Hook event collisions:** Hooks are merged — both hooks run and their outputs are combined. The user can configure execution order.

### 3. Rename Tracking

When conflict resolution renames a component, the installer stores the `original-name → installed-name` mapping in the state store. This mapping enables correct behavior for:

- Plugin updates (matching incoming components to their renamed installed counterparts)
- Plugin removals (knowing which installed components belong to a plugin)
- Cross-reference management (maintaining internal consistency)

### 4. Cross-Platform Component Format

Plugin source files use a cross-platform core frontmatter schema with these fields:

- `name` — component identifier
- `description` — human-readable summary
- `type` — component type (skill, agent, prompt, hook, mcp)
- `requires` — dependency declarations
- `sources` — source file references

An optional `platformConfig` section holds platform-specific overrides (e.g., `claude-code.allowed-tools`, `cursor.applyTo`). This section is preserved in the plugin source but selectively applied during installation.

### 5. Platform-Aware Frontmatter Generation

When installing to a specific platform, the installer:

- ONLY emits frontmatter fields that the target platform supports
- But includes ALL fields that the platform supports
- The plugin's source frontmatter contains the superset; the installer strips and maps per platform

This avoids polluting platform-specific files with unrecognized fields while maximizing the metadata available to the target platform's tooling.

## Consequences

### Positive

- **POS-001**: Colon separator provides clear, unambiguous namespacing that avoids conflicts with directory separators across all platforms
- **POS-002**: Per-component-type conflict resolution matches real-world semantics — name collisions are conflicts, hook overlaps are additive
- **POS-003**: User choice during conflict resolution preserves agency and avoids surprising renames of already-working components
- **POS-004**: Rename tracking creates a reliable audit trail enabling correct update, removal, and cross-reference operations
- **POS-005**: Cross-platform frontmatter schema enables plugin portability while respecting each platform's field support

### Negative

- **NEG-001**: Installer must implement an interactive conflict resolution UI (via `@clack/prompts` or similar), adding complexity to the install flow
- **NEG-002**: State store must persist and maintain original-name to installed-name mappings, adding a stateful dependency
- **NEG-003**: Platform adapter layer must maintain knowledge of each platform's supported frontmatter fields, creating an ongoing maintenance burden as platforms evolve
- **NEG-004**: Cross-reference update logic adds complexity — when a component is renamed, all references within that plugin's files must be found and updated
- **NEG-005**: Colon character may need escaping in some shell contexts or platform configurations

## Alternatives Considered

### Blanket Conflict Blocking via `conflicts` Field

- **ALT-001**: **Description**: Define a `conflicts` array in the manifest that lists incompatible plugins. Installation fails if any listed plugin is already installed.
- **ALT-002**: **Rejection Reason**: Too blunt. Does not handle the common case of simple name collisions between otherwise compatible plugins. Forces plugin authors to predict all future conflicts at authoring time.

### No Namespacing (Flat Names)

- **ALT-003**: **Description**: Install components with their original names, no prefix or namespace. First-installed wins.
- **ALT-004**: **Rejection Reason**: Collision risk grows linearly with the number of installed plugins. No disambiguation path for users who need both components.

### Slash Separator for Namespacing

- **ALT-005**: **Description**: Use `plugin-name/component-name` as the namespace pattern instead of colon.
- **ALT-006**: **Rejection Reason**: Slash collides with directory separators on Unix-like systems and could cause ambiguity in file path resolution on some platforms.

### Vercel Minimal Frontmatter (name + description only)

- **ALT-007**: **Description**: Follow Vercel's approach of minimal frontmatter with only `name` and `description` fields.
- **ALT-008**: **Rejection Reason**: Loses structured metadata (type, requires, sources) that tooling can use for dependency resolution, validation, and cross-referencing.

### Claude Code Full Platform-Specific Frontmatter

- **ALT-009**: **Description**: Use Claude Code's complete frontmatter schema as the universal format for all platforms.
- **ALT-010**: **Rejection Reason**: Many Claude Code-specific fields (e.g., `allowed-tools`) have no equivalent on other platforms. Emitting them would pollute files with unrecognized fields and create confusion.

## Implementation Notes

- **IMP-001**: The interactive conflict resolution UI should use `@clack/prompts` for consistent cross-platform terminal interaction, presenting clear choices when collisions are detected
- **IMP-002**: The state store mapping (original-name to installed-name) should be persisted alongside the plugin registry, likely in a JSON file within the agent-plugin data directory
- **IMP-003**: Platform adapters should implement a `supportedFields()` method that returns the set of frontmatter fields the platform recognizes, enabling the installer to filter appropriately
- **IMP-004**: Cross-reference updates during renames should use AST-aware or frontmatter-aware parsing rather than naive string replacement to avoid false positives
- **IMP-005**: Hook merge ordering should default to installation order (first-installed runs first) with explicit override via user configuration

## References

- **REF-001**: Claude Code component naming conventions (colon separator pattern)
- **REF-002**: Vercel Skills format analysis (minimal frontmatter approach)
- **REF-003**: Related ADRs: ADR-001 (plugin format), ADR-002 (target platforms)

## Observations

- [decision] Colon separator chosen as namespace delimiter for installed components #namespacing #convention
- [decision] Conflict resolution is type-aware: named components prompt user choice, hooks merge additively #conflict-resolution
- [decision] User selects which component to rename during collision, preserving agency #user-experience
- [decision] Rename mappings persisted in state store for update/removal correctness #state-management
- [decision] Cross-platform core frontmatter uses name, description, type, requires, sources with optional platformConfig #frontmatter #cross-platform
- [decision] Platform-aware installation emits only supported fields per target platform #platform-adapter
- [constraint] Interactive UI required for conflict resolution increases installer complexity #trade-off
- [constraint] Platform adapter layer must track each platform's supported frontmatter fields #maintenance
- [risk] Colon character may require escaping in certain shell or platform contexts #namespacing

## Relations

- extends [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-002 Target Platforms and Audiences]]
- relates_to [[ANALYSIS-003-vercel-skills-format-deep-dive]]
- relates_to [[ANALYSIS-005-claude-code-plugin-format]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]