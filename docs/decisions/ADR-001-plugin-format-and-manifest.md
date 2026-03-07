---
title: ADR-001 Plugin Format and Manifest
type: decision
permalink: decisions/adr-001-plugin-format-and-manifest-1
tags:
- architecture
- plugin-format
- manifest
- decision
- plugin-json
---

# ADR-001 Plugin Format and Manifest

---
title: "ADR-001: Plugin Format and Manifest"
status: "Accepted"
date: "2026-03-07"
authors: "Agent Plugin Core Team"
tags: ["architecture", "plugin-format", "manifest", "decision", "plugin-json"]
supersedes: ""
superseded_by: ""
---

## Status

**Accepted**

## Context

We are designing the plugin format for `@acmelabz/agent-plugin`, a cross-platform AI agent plugin manager targeting 7 platforms: Claude Code, Cursor, GitHub Copilot CLI, Kiro, OpenCode, Amp, and Windsurf.

Three reference systems were analyzed to inform this decision:

- **Vercel Skills** (ANALYSIS-003): Skill-centric format with metadata and execution patterns
- **TanStack Intent** (ANALYSIS-004): Intent-based plugin architecture with declarative configuration
- **Claude Code Plugin Format** (ANALYSIS-005): Bundle model with optional manifest and nested directory structure

Key forces at play:

- **Cross-platform compatibility**: Plugins must work across 7 different AI agent platforms with varying native plugin formats
- **Developer experience**: Authors need a clear, predictable structure that is easy to scaffold and maintain
- **Component flexibility**: Plugins may contain skills, agents, prompts, hooks, and MCP server configurations in various combinations
- **Installation UX**: Users need control over which components to install when components are independent
- **Conflict management**: Multiple plugins may provide overlapping components that require intelligent resolution

Claude Code's bundle model demonstrated the right conceptual approach (one plugin, many components), but its implementation had documented issues: the nested `.claude-plugin/plugin.json` path was error-prone, and the optional manifest created ambiguity for cross-platform tooling.

## Decision

We adopt a **bundle model** with a **mandatory root-level manifest** (`plugin.json`) and **conventional component directories**.

### 1. Plugin = Bundle Model

A plugin is a bundle that can contain many skills, many agents, many prompts, many hooks, and typically a single MCP server configuration. This follows Claude Code's bundle model conceptually but corrects its implementation issues.

### 2. Directory Structure

```text
my-plugin/
  plugin.json          # REQUIRED - root level, not nested
  skills/              # Skill definitions
  agents/              # Agent definitions
  prompts/             # Prompt templates
  hooks/               # Lifecycle hooks
  mcp/                 # MCP server configuration
```

The manifest lives at the plugin root. This avoids the error-prone nested path (`.claude-plugin/plugin.json`) documented in Claude Code's own materials.

The `prompts/` directory is a novel extension not present in any of the three analyzed reference systems (Vercel Skills, TanStack Intent, Claude Code). We include it because prompts are a distinct content type in AI agent workflows: reusable system prompts, templates with variable interpolation, and context-setting instructions. These are neither skills (which include execution logic and acceptance criteria) nor agents (which define personas and routing rules). Treating prompts as a first-class component type enables sharing and versioning of prompt libraries independently of the skills or agents that consume them.

### 3. plugin.json Is Always Required

Every plugin must have a `plugin.json` manifest. Unlike Claude Code where the manifest is optional, we require it because cross-platform metadata, version information, and component declarations are essential for a multi-platform plugin manager.

### 4. Minimum Required Fields

```json
{
  "name": "@scope/plugin-name",
  "version": "1.0.0",
  "description": "What this plugin does"
}
```

Three fields are required: `name`, `version`, and `description`. These match npm `package.json` conventions that developers already know.

### Schema Evolution Strategy

No `formatVersion` or `manifest_version` field. Research (ANALYSIS-007) found that 70% of config formats (package.json, tsconfig, Cargo.toml, composer.json) handle evolution without format version fields. Schema evolution is managed through:

1. **Additive changes only**: New optional fields are added without breaking existing manifests
2. **Ignore unknown fields**: Older plugin manager versions silently ignore fields they don't recognize
3. **Detect missing required fields**: Newer plugin manager versions detect when a manifest is missing fields that have become required and provide actionable error messages
4. **`doctor` command**: A diagnostic command (`agent-plugin doctor`) validates installed plugin manifests against the current schema, identifying issues and suggesting fixes
5. **Auto-detection on upgrade/update**: The `upgrade` and `update` commands detect schema mismatches automatically and trigger migration (auto-migration for non-breaking changes, migration wizard via @clack/prompts when user input is required)
6. **Migration support**: When schema changes require user input, the plugin manager runs a migration wizard (via @clack/prompts) to guide the author through updates. Non-interactive migrations are applied automatically.

### Why JSON for the Manifest

We use JSON for `plugin.json` because: (a) universal tooling support across all 7 target platforms, each of which already parses JSON natively; (b) unambiguous parsing with no implicit type coercion (no YAML "Norway problem" where `NO` becomes `false`); (c) native to the JavaScript/TypeScript ecosystem where most AI agent tooling is built; (d) JSON Schema enables IDE autocomplete, inline validation, and documentation generation from the schema definition. TOML and YAML were considered but rejected: TOML lacks nested object ergonomics needed for component declarations, and YAML's implicit typing creates correctness risks in automated pipelines.

### 5. installMode Field

```json
{
  "installMode": "bundle" | "collection"
}
```

- **`bundle`** (default): All components are interdependent. Install all or nothing. No chooser UI.
- **`collection`**: Components are independent. The installer presents a chooser UI so users pick which components to install.

### 6. Core Optional Fields

```json
{
  "author": "string",
  "license": "string",
  "repository": "string",
  "homepage": "string",
  "keywords": ["string"]
}
```

All plugins are inherently cross-platform. The plugin manager handles translation to each platform's native format, so there is no `platforms` field. Every plugin can be installed on all 7 supported platforms (Claude Code, Cursor, GitHub Copilot CLI, Kiro, OpenCode, Amp, Windsurf).

**Deferred fields**: Plugin dependencies, conflicts, and source provenance are deferred to a future ADR (not yet written). These are load-bearing architectural concepts requiring dedicated design -- dependency resolution ordering, conflict detection algorithms, and provenance verification each warrant their own decision record.

Plus component path declarations (see below).

### 7. Component Path Declarations

Skills, agents, hooks, prompts, and MCP server configs are declared as paths in `plugin.json`:

```json
{
  "skills": ["skills/research", "skills/code-review"],
  "agents": ["agents/analyst.md"],
  "hooks": ["hooks/pre-commit.js"],
  "prompts": ["prompts/system.md"],
  "mcp": "mcp/server.json"
}
```

### 8. Intelligent Conflict Resolution

Different strategies per component type rather than blanket blocking:

- **Skills**: Namespace prefixing or user-prompted rename on conflict
- **Hooks**: Merged execution (run both, combine outputs)
- **Other types**: Strategy details TBD in implementation

## Consequences

### Positive

- **POS-001**: Mandatory manifest provides reliable metadata for cross-platform installation, registry indexing, and dependency resolution
- **POS-002**: Root-level `plugin.json` eliminates the nested path confusion documented in Claude Code's format
- **POS-003**: `installMode` gives plugin authors explicit control over whether their components are presented as a unit or a menu, creating appropriate UX for both tightly-coupled and loosely-coupled plugins
- **POS-004**: Minimum required fields (`name`, `version`, `description`) match npm conventions, reducing cognitive load for JavaScript/TypeScript developers
- **POS-005**: Intelligent per-type conflict resolution preserves user work and plugin functionality instead of failing on first conflict

### Negative

- **NEG-001**: Authors must always create a `plugin.json` even for simple single-skill plugins (mitigated by scaffolding wizard)
- **NEG-002**: The installer must implement per-type conflict resolution strategies, adding complexity to the installation pipeline
- **NEG-003**: Two `installMode` values create two distinct user experiences that must both be tested and documented
- **NEG-004**: Conventional directories (`skills/`, `agents/`, etc.) impose structure that may feel rigid for plugins with unconventional layouts

## Alternatives Considered

### Optional Manifest (Claude Code Model)

- **ALT-001**: **Description**: Follow Claude Code's approach where `plugin.json` is optional, with component auto-discovery based on file conventions
- **ALT-001**: **Rejection Reason**: Cross-platform metadata (platform targets, version, dependencies) is essential for a multi-platform plugin manager. Without a manifest, the installer cannot determine platform compatibility or resolve dependencies reliably

### Nested Manifest Path (.claude-plugin/plugin.json)

- **ALT-002**: **Description**: Place the manifest inside a `.claude-plugin/` subdirectory, matching Claude Code's directory structure
- **ALT-002**: **Rejection Reason**: Claude Code's own documentation identifies this nested path as error-prone. A root-level manifest is more discoverable and aligns with established conventions (package.json, pyproject.toml, Cargo.toml)

### Type-Specific Entry Files Only (No Bundle)

- **ALT-003**: **Description**: Each component type uses its own entry file (e.g., `SKILL.md` per skill) with no overarching plugin manifest or bundle concept
- **ALT-003**: **Rejection Reason**: Plugins frequently need to bundle multiple interdependent components (a skill that requires a specific MCP server, an agent that references specific prompts). Without a bundle model, expressing these dependencies is impossible

### Simple Conflicts Field (Block on Conflict)

- **ALT-004**: **Description**: A `conflicts` field in the manifest that blocks installation entirely when any conflict is detected
- **ALT-004**: **Rejection Reason**: Blanket blocking is too aggressive. Skills can be namespaced, hooks can be merged, and prompts can be appended. Per-type resolution preserves user intent and maximizes plugin compatibility

## Implementation Notes

- **IMP-001**: The scaffolding wizard (`create-plugin` or equivalent) must generate a valid `plugin.json` with required fields pre-populated, reducing the burden of the mandatory manifest requirement
- **IMP-002**: Component path declarations in `plugin.json` should support both file paths and directory paths (directories imply all matching files within)
- **IMP-003**: The `installMode: "collection"` chooser UI must clearly communicate component descriptions, dependencies between components (if any), and default selections
- **IMP-004**: Conflict resolution strategies should be implemented as pluggable handlers per component type, allowing future extension without modifying core installer logic
- **IMP-005**: Validation tooling (`plugin lint` or equivalent) should verify manifest schema, check that declared component paths exist, and warn about missing optional fields
- **IMP-006**: Security controls (path traversal prevention, hook isolation, MCP approval gates, manifest integrity verification) are deferred to [[ADR-004 Plugin Security Model]]. This ADR intentionally omits security constraints to avoid coupling format decisions with security policy decisions

## References

- **REF-001**: ANALYSIS-003 Vercel Skills format deep dive
- **REF-002**: ANALYSIS-004 TanStack Intent deep dive
- **REF-003**: ANALYSIS-005 Claude Code plugin format analysis
- **REF-004**: npm package.json specification (field naming conventions)
- **REF-005**: Claude Code plugin documentation (nested path issues)

## Future Considerations

- **Hybrid installMode**: `installMode` may need a third mode or intra-plugin component dependency declarations for hybrid plugins that combine independent skills with a shared MCP server. Currently, `bundle` forces all-or-nothing and `collection` implies full independence. A plugin with 3 independent skills that all require the same MCP server has no way to express "install any skill, but MCP is mandatory."
- **5-component model scope**: The 5-component model (skills, agents, prompts, hooks, mcp) is a deliberate subset of Claude Code's 7-type system. We omit commands (legacy slash-command pattern being replaced by skills), lspServers (platform-specific, not portable), and outputStyles (niche formatting concern). If future platforms introduce new component types that are genuinely cross-platform, the open component path declaration pattern in Section 7 can accommodate them without schema changes.
- **Platform translation layer**: How `plugin.json` maps to each platform's native format (Claude Code's SKILL.md, Cursor's rules files, etc.) requires a dedicated ADR. This ADR defines the canonical format; translation is a separate concern.

## Observations
- [decision] Plugin = bundle model: one plugin contains many skills, agents, prompts, hooks, and typically one MCP server #plugin-format #architecture
- [decision] plugin.json is mandatory at the plugin root directory, not nested #manifest #cross-platform
- [decision] Minimum required fields: name, version, description — matching npm package.json conventions #manifest
- [decision] No formatVersion field — schema evolution via additive changes, unknown field tolerance, doctor command, and migration wizards (ANALYSIS-007 research: 70% of config formats handle evolution without version fields) #manifest #versioning
- [decision] installMode field distinguishes bundle (all-or-nothing) from collection (user picks components) #installation-ux
- [decision] Intelligent per-type conflict resolution instead of blanket blocking #conflict-resolution
- [decision] Component paths declared in manifest for skills, agents, hooks, prompts, and MCP #manifest #component-discovery
- [decision] JSON chosen over YAML/TOML for manifest: universal tooling, unambiguous parsing, JSON Schema support #manifest #format
- [decision] prompts/ is a novel first-class component type not in reference systems, distinct from skills and agents #plugin-format
- [decision] All plugins are inherently cross-platform; no platforms field needed; plugin manager handles translation to all 7 target platforms #cross-platform
- [decision] dependencies, conflicts, and source provenance deferred to future ADRs as load-bearing architectural concepts #deferred
- [decision] Security controls deferred to ADR-004 Plugin Security Model #security #deferred
- [constraint] Authors must always create plugin.json; scaffolding wizard mitigates this burden #developer-experience
- [requirement] Installer must implement per-type conflict resolution strategies #installer
- [insight] Claude Code's nested manifest path (.claude-plugin/plugin.json) is documented as error-prone by Claude Code itself #reference-analysis
- [insight] Hybrid plugins (independent skills + shared MCP) expose a gap in the bundle/collection binary that may need future resolution #installMode
- [fact] Seven target platforms: Claude Code, Cursor, GitHub Copilot CLI, Kiro, OpenCode, Amp, Windsurf #cross-platform
- [fact] 5-component model is a deliberate subset of Claude Code's 7-type system, omitting commands, lspServers, outputStyles #plugin-format

## Relations

- relates_to [[ANALYSIS-003 Vercel Skills Format Deep Dive]]
- relates_to [[ANALYSIS-004 TanStack Intent Deep Dive]]
- relates_to [[ANALYSIS-005 Claude Code Plugin Format]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]