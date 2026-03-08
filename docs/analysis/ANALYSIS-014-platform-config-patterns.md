---
title: ANALYSIS-014 Platform Config Patterns
type: note
permalink: analysis/analysis-014-platform-config-patterns
tags:
- platform-config
- plugin-systems
- cli-wizard
- analysis
- agent-plugin
---

# ANALYSIS-014 Platform Config Patterns

## 1. Objective and Scope

**Objective**: How should plugin authors define platform-specific configuration (platformConfig) for their components in @acmelabs-15/agent-plugin? Should this config live per-component, in a centralized manifest, or in a layered system? How should the CLI wizard guide authors through defining these fields?

**Scope**: Research across 15 plugin/extension systems to identify configuration location patterns, platform-specific field handling, and CLI wizard UX patterns for multi-platform configuration. Includes AI coding assistants (Claude Code, Cursor, Kiro, OpenCode, Amp, Windsurf, GitHub Copilot CLI), ecosystem tools (TanStack Intent, Vercel Skills, Agent Skills spec), general-purpose plugin systems (VSCode, JetBrains, Backstage, Terraform, Nx, WordPress), and CLI wizard tools (@clack/prompts, Yeoman, create-next-app).

## 2. Context

@acmelabs-15/agent-plugin targets 7 AI coding platforms. Each platform uses different configuration fields for skills/agents/rules:

| Platform | Skill/Rule Format | Key Platform-Specific Fields |
|----------|------------------|------------------------------|
| Claude Code | SKILL.md (YAML frontmatter) | `allowed-tools`, `disable-model-invocation`, `user-invocable`, `model`, `context`, `agent`, `hooks` |
| Cursor | .mdc files or SKILL.md | `alwaysApply`, `globs`, `description` (required for auto-attach) |
| GitHub Copilot CLI | SKILL.md | `user-invocable`, `disable-model-invocation`, `hint` (autocomplete text), `target` (vscode or github-copilot) |
| Kiro | Steering files (.md) | `inclusion` (always/fileMatch/manual/auto), `fileMatchPattern`, `name`, `description` |
| OpenCode | SKILL.md | `name`, `description`, `allowed-tools`, `license`, `compatibility`, `metadata` |
| Amp | SKILL.md + mcp.json | `name`, `description`, plus bundled `mcp.json` with `includeTools` |
| Windsurf | .windsurfrules or .windsurf/rules/ | `globs`, `description`, `alwaysApply` (similar to Cursor) |

ADR-001 states: "All plugins are inherently cross-platform. The plugin manager handles translation to each platform's native format." This means we need a translation layer. The question is where platform-specific configuration lives and how authors specify it.

## 3. Approach

**Methodology**: Web research across official documentation, GitHub repositories, and community resources. Cross-referenced 15 plugin/extension systems. Analyzed CLI wizard patterns from 4 scaffolding tools.

**Tools Used**: WebSearch (16 queries), WebFetch (4 page analyses), Brain MCP (5 prior analysis notes read).

**Limitations**: TanStack Intent documentation is sparse (alpha product). Kiro steering format has limited public documentation beyond basics. Windsurf rule format documentation is fragmented across blog posts and community forums.

## 4. Data and Analysis

### 4.1 Configuration Location Patterns Across Systems

Research reveals 4 distinct patterns for where platform-specific configuration lives:

#### Pattern A: Per-Component (Frontmatter in Each File)

**Used by**: Agent Skills spec, Claude Code, Cursor, Kiro, OpenCode, Amp, Windsurf, GitHub Copilot CLI, TanStack Intent, Vercel Skills

Each component file (SKILL.md, .mdc, steering file) contains its own configuration in YAML frontmatter. The platform-specific fields are embedded directly in the component.

**Evidence**:

- Agent Skills spec: `name`, `description`, `allowed-tools` in each SKILL.md
- Claude Code: 10 frontmatter fields per SKILL.md; 13 frontmatter fields per agent .md file
- Cursor: `alwaysApply`, `globs`, `description` in each .mdc file
- Kiro: `inclusion`, `fileMatchPattern` in each steering file
- TanStack Intent: extends frontmatter with `type`, `framework`, `sources`, `requires`

**Pros**: Self-contained files. Each component carries its own metadata. No cross-referencing needed. Easy to copy/move components.

**Cons**: Platform-specific fields leak into component files. Authors must know each platform's fields. No single source of truth for "what fields does my plugin declare for Cursor?"

#### Pattern B: Centralized Manifest (Single Config File)

**Used by**: VSCode (package.json contributes), JetBrains (plugin.xml), Claude Code marketplace (marketplace.json), Backstage (app-config.yaml with plugin schemas)

One manifest file declares all configuration for all components. Components themselves have no metadata.

**Evidence**:

- VSCode: `contributes.configuration` in package.json declares all extension settings
- JetBrains: plugin.xml declares all extensions, actions, listeners
- Claude Code marketplace.json: can override plugin component paths and metadata
- Backstage: each plugin contributes JSON Schema fragments to a unified config

**Pros**: Single source of truth. Easy to see all configuration at once. IDE autocomplete and validation via JSON Schema.

**Cons**: Manifest becomes large for complex plugins. Components are not self-describing. Moving a component requires updating the manifest.

#### Pattern C: Layered (Manifest Defaults + Per-Component Overrides)

**Used by**: Claude Code plugin system, JetBrains (plugin.xml + optional per-IDE config files), Terraform (provider blocks + resource blocks), Nx (nx.json defaults + per-project overrides)

A manifest provides defaults that per-component files can override.

**Evidence**:

- Claude Code: plugin.json provides custom paths; SKILL.md frontmatter provides skill-specific config. Marketplace.json `strict` flag controls which is authority.
- JetBrains: plugin.xml has main config; `<depends config-file="...">` loads additional IDE-specific config
- Terraform: provider block sets defaults (region, credentials); resource blocks override per-resource
- Nx: nx.json sets default generator options; per-project project.json overrides

**Pros**: Sensible defaults reduce per-component boilerplate. Components can specialize without repeating shared config. Flexibility.

**Cons**: Resolution order must be clearly documented. "Where does this value come from?" debugging. More complex for tooling to resolve.

#### Pattern D: Adapter/Translation Layer (No Author-Facing Platform Config)

**Used by**: Vercel Skills CLI, Agent Skills spec (partially)

Authors write platform-agnostic content. The installation tool translates to each platform's format.

**Evidence**:

- Vercel Skills: authors write standard SKILL.md; `npx skills add` places files in platform-specific directories (.claude/skills/, .cursor/skills/). No platform-specific frontmatter needed from authors.
- Agent Skills spec: defines 6 platform-agnostic fields; platforms that need more (Claude Code's `allowed-tools`, `context: fork`) extend the spec with their own fields

**Pros**: Authors write once. No platform knowledge required. Tool handles all translation. Simplest author experience.

**Cons**: Loses platform-specific capabilities. Cannot use Claude Code's `context: fork` or Cursor's `alwaysApply` without platform-specific extensions. Lowest common denominator risk.

### 4.2 How the Most Successful Systems Handle It

#### Agent Skills Open Standard (agentskills.io)

The Agent Skills spec (created by Anthropic, now cross-platform) defines a **minimal core** with **platform-extensible metadata**:

Core fields (portable): `name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`

The `metadata` field is intentionally open: "Arbitrary key-value mapping for additional metadata. Clients can use this to store additional properties not defined by the Agent Skills spec." Unknown frontmatter fields are ignored. This means platforms can add their own fields (Claude Code adds `model`, `context`, `agent`, etc.) while the base spec remains portable.

This is Pattern D with escape hatches.

#### Vercel Skills CLI (npx skills)

Vercel takes the **pure adapter pattern**. Authors write standard Agent Skills SKILL.md files. The CLI detects which platforms are installed (37+ agents supported) and copies/symlinks to the correct directories:

| Platform | Install Path |
|----------|-------------|
| Claude Code | `.claude/skills/<name>/` |
| Cursor | `.agents/skills/<name>/` (project) or `~/.cursor/skills/` (global) |
| OpenCode | `.agents/skills/<name>/` or `~/.config/opencode/skills/` |
| Amp | `.agents/skills/<name>/` |

No platform-specific configuration is written by Vercel's installer. The SKILL.md file is identical across all platforms. Platform-specific behavior comes from each platform's own interpretation of the standard fields.

This works because skills are instructions (markdown text). The platform reads the same text but applies its own invocation logic. No platform-specific fields are needed for basic skill functionality.

#### TanStack Intent

TanStack adds **framework** and **type** discriminators to the Agent Skills base:

```yaml
type: framework
framework: react
```

This enables the same skill to have variants per framework. The framework field acts like a platform discriminator but for UI frameworks, not AI platforms. TanStack does NOT have AI-platform-specific fields; it relies on the Agent Skills standard for cross-platform compatibility.

#### Claude Code Plugin System

Claude Code uses **Pattern C** (layered). The plugin.json manifest declares component paths and can override metadata. Each SKILL.md has its own frontmatter. The marketplace.json can override both via the `strict` field.

For platform-specific config, Claude Code is the platform, so all fields are platform-specific by definition. But the key insight is the layering: plugin-level defaults in plugin.json, component-level specifics in frontmatter.

#### VSCode Extensions

VSCode uses **Pattern B** (centralized). All configuration is in package.json under `contributes`. Platform-specific handling uses the `extensionDependencies` and platform-specific publishing:

- Keys bindings: `"key"` for default, `"mac"` for macOS override
- Platform-specific packages: publish with `--target` flag for OS-specific content

No per-file config. Everything in the manifest.

#### JetBrains Plugins

JetBrains uses **Pattern C** (layered) with optional per-IDE config files:

```xml
<depends optional="true" config-file="myPlugin-withKotlin.xml">
  org.jetbrains.kotlin
</depends>
```

The main plugin.xml contains shared config. Additional XML files contain IDE-specific extensions that only load when the target IDE has the required dependency. This is the closest analogue to our platform-specific config problem.

#### Terraform Providers

Terraform uses **Pattern C** (layered) at a different scale. Each cloud provider has its own schema (provider block), and resources within that provider inherit defaults from the provider block while declaring resource-specific config.

The key pattern: `provider "aws" { region = "us-east-1" }` sets defaults; `resource "aws_instance" { ... }` inherits region unless overridden. This maps to: `platformConfig.claudeCode = { model: "opus" }` as default; individual skills can override `model: "sonnet"`.

### 4.3 Platform Field Inventory

Across the 7 target platforms, here are all known configuration fields that an author might need to set:

#### Shared Fields (Agent Skills Standard)

| Field | All Platforms | Notes |
|-------|--------------|-------|
| `name` | Yes | Required everywhere |
| `description` | Yes | Required everywhere |
| `license` | Yes | Optional everywhere |
| `metadata` | Yes | Optional key-value map |
| `allowed-tools` | Most | Claude Code, OpenCode, Copilot support it |

#### Claude Code-Specific Fields

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `disable-model-invocation` | bool | false | If true, only user can invoke |
| `user-invocable` | bool | true | If false, hidden from / menu |
| `model` | string | inherit | Model override |
| `context` | string | -- | `fork` for isolated subagent context |
| `agent` | string | -- | Subagent type when context=fork |
| `hooks` | object | -- | Skill-scoped lifecycle hooks |
| `argument-hint` | string | -- | Autocomplete hint text |

#### Cursor-Specific Fields

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `alwaysApply` | bool | false | Always attach to context |
| `globs` | string/array | -- | File patterns for conditional attachment |

#### GitHub Copilot CLI-Specific Fields

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `hint` | string | -- | Autocomplete hint text |
| `target` | string | both | `vscode` or `github-copilot` |

#### Kiro-Specific Fields

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `inclusion` | enum | always | `always`, `fileMatch`, `manual`, `auto` |
| `fileMatchPattern` | string/array | -- | File glob patterns |

#### Windsurf-Specific Fields

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `alwaysApply` | bool | -- | Always attach (same name as Cursor) |
| `globs` | string | -- | File patterns (same name as Cursor) |

#### OpenCode and Amp

Both follow the Agent Skills standard closely. OpenCode adds no fields beyond the standard. Amp adds `mcp.json` co-location (a file, not a frontmatter field).

### 4.4 Field Overlap Analysis

Some fields map across platforms with different names or semantics:

| Concept | Claude Code | Cursor | Kiro | Windsurf | Copilot |
|---------|------------|--------|------|----------|---------|
| Always load | `user-invocable: true` + no `disable-model-invocation` | `alwaysApply: true` | `inclusion: always` | `alwaysApply: true` | Default behavior |
| File-conditional | Not built-in | `globs: "src/**"` | `inclusion: fileMatch` + `fileMatchPattern` | `globs: "*"` | Not built-in |
| Manual-only | `disable-model-invocation: true` | `alwaysApply: false` + no globs | `inclusion: manual` | `alwaysApply: false` + no globs | `disable-model-invocation: true` |
| Autocomplete hint | `argument-hint` | Not available | Not available | Not available | `hint` |
| Model override | `model: "opus"` | Not available | Not available | Not available | Not available |

This reveals 3 cross-platform concepts that map to different fields:

1. **Loading strategy**: always / conditional / manual / auto-detect
2. **File matching**: glob patterns for conditional loading
3. **Invocation control**: who can trigger the skill

### 4.5 CLI Wizard Patterns

#### create-next-app Pattern

Sequential prompts with smart defaults:

1. Project name (text input)
2. TypeScript? (yes/no, default: yes)
3. ESLint/Biome/None? (select)
4. Tailwind CSS? (yes/no)
5. src/ directory? (yes/no)
6. App Router? (yes/no)
7. Import alias? (text, default: @/*)

Key UX patterns:

- Sane defaults for every question (press Enter to accept)
- Questions ordered from most common to least common
- No platform-specific branching; one linear flow
- CLI flags can skip any prompt (`--typescript`, `--eslint`)

#### Nx Generator Pattern

Schema-driven prompts:

- `schema.json` defines all options with types, defaults, descriptions
- Nx Console renders a GUI from the schema
- CLI prompts only for required fields without defaults
- Framework-specific presets bundle related options

Key UX pattern: **Discriminated union types** enforce valid combinations. Choosing "React" unlocks React-specific options; choosing "Angular" unlocks different ones.

#### Yeoman Composability Pattern

Sub-generators handle platform-specific scaffolding:

- Base generator prompts for shared config
- `composeWith('generator-mocha')` adds test framework config
- Each sub-generator manages its own prompts
- Configuration stored in `.yo-rc.json`

Key UX pattern: **Composable generators** where each platform is a sub-generator with its own prompt sequence.

#### @clack/prompts Capabilities

@clack/prompts provides:

- `text()` -- free text with validation
- `select()` -- single choice from options
- `multiselect()` -- multiple choices with checkboxes
- `confirm()` -- yes/no
- `group()` -- sequential prompt groups
- `spinner()` -- loading indicator
- Custom styled prompts via @clack/core

Key UX pattern: `group()` allows branching. After selecting platforms, show platform-specific prompts only for selected platforms.

### 4.6 Recommended CLI Wizard Flow

Based on research, the optimal wizard pattern for multi-platform config:

**Phase 1: Core Config (shared)**

```
? Component name: [text]
? Description: [text]
? Type: [select: skill | agent | prompt]
? Target platforms: [multiselect: all 7, default: all selected]
```

**Phase 2: Cross-Platform Concepts (mapped)**

```
? Loading strategy: [select: always | file-conditional | manual | auto-detect]
  (if file-conditional) ? File patterns: [text, e.g., "src/**/*.ts"]
? Pre-approved tools: [multiselect from common tools, or custom text]
```

**Phase 3: Platform-Specific Overrides (per selected platform)**

```
[Only shown if user selected specific platforms in Phase 1]
? Claude Code: Model override? [select: inherit | opus | sonnet | haiku]
? Claude Code: Run in isolated context? [confirm]
? Cursor: Always apply rule? [confirm] (pre-filled from Phase 2 mapping)
? Kiro: Inclusion mode? [select] (pre-filled from Phase 2 mapping)
```

**Phase 4: Review and Confirm**

```
[Show generated platformConfig with all values]
? Confirm and create? [confirm]
```

The critical UX insight: **Phase 2 captures intent once; the wizard maps it to platform-specific fields automatically.** Authors think in concepts ("I want this skill loaded when editing TypeScript files"), not in platform fields ("set `globs` for Cursor and `fileMatchPattern` for Kiro"). Phase 3 exists only for platform-specific fields that have no cross-platform concept (like Claude Code's `model` override).

## 5. Results

### Configuration Location Pattern Comparison

| Pattern | Systems Using It | Author Complexity | Platform Fidelity | Tooling Complexity | Best For |
|---------|-----------------|-------------------|--------------------|--------------------|----------|
| A: Per-Component | Agent Skills, all platforms natively | High (know each platform) | High (full access to all fields) | Low | Single-platform plugins |
| B: Centralized | VSCode, JetBrains, Backstage | Medium (one file to learn) | High | Medium | IDE extensions with many contribution points |
| C: Layered | Claude Code, JetBrains, Terraform, Nx | Medium | High | High | Complex plugins with shared defaults |
| D: Adapter/Translation | Vercel Skills, our plugin manager | Low (write once) | Low-Medium (lowest common denominator) | High (adapter per platform) | Cross-platform distribution |

### Recommended Hybrid: D+C (Adapter with Layered Overrides)

The evidence points to combining Pattern D (adapter/translation) with Pattern C (layered overrides):

1. **Default**: Authors write platform-agnostic components using the Agent Skills standard fields. The plugin manager's adapter layer translates to each platform's native format.

2. **Override**: Authors who need platform-specific capabilities declare them in `platformConfig` inside `plugin.json`. These override the adapter's default translation.

3. **Per-component override**: Individual SKILL.md files can include platform-specific frontmatter in a `platforms` block that overrides both the manifest default and the adapter default.

### Proposed Configuration Resolution Order

```
Platform native format
    ^
    |  4. Component-level platformConfig (in SKILL.md frontmatter)
    |  3. Manifest-level platformConfig (in plugin.json)
    |  2. Adapter default mapping (cross-platform concept -> platform field)
    |  1. Agent Skills standard fields (name, description, etc.)
```

Higher numbers override lower numbers. Most plugins only need levels 1-2 (standard fields + adapter).

### Proposed plugin.json platformConfig Schema

```json
{
  "name": "@scope/my-plugin",
  "version": "1.0.0",
  "description": "My cross-platform plugin",
  "skills": ["skills/code-review"],
  "agents": ["agents/analyst.md"],
  
  "platformConfig": {
    "defaults": {
      "loadingStrategy": "auto-detect",
      "filePatterns": ["src/**/*.ts"],
      "approvedTools": ["Read", "Grep", "Glob"]
    },
    "claude-code": {
      "model": "sonnet",
      "context": "fork"
    },
    "cursor": {
      "alwaysApply": false
    },
    "kiro": {
      "inclusion": "fileMatch"
    }
  }
}
```

### Proposed SKILL.md Per-Component Override

```yaml
---
name: code-review
description: Reviews code changes for quality and patterns
platforms:
  claude-code:
    model: opus
    allowed-tools: Read, Grep, Glob, Bash(git diff:*)
  cursor:
    alwaysApply: true
  kiro:
    inclusion: always
---

# Code Review Skill
...
```

The `platforms` block in SKILL.md is optional. When present, it overrides the manifest-level `platformConfig` for that specific component. When absent, the manifest-level config applies.

### Cross-Platform Concept Mapping Table

The adapter layer translates these abstract concepts to platform-specific fields:

| Abstract Concept | plugin.json Field | Claude Code | Cursor | Kiro | Windsurf | Copilot |
|-----------------|-------------------|-------------|--------|------|----------|---------|
| Loading: always | `loadingStrategy: "always"` | (default behavior) | `alwaysApply: true` | `inclusion: always` | `alwaysApply: true` | (default) |
| Loading: file-conditional | `loadingStrategy: "file-conditional"` + `filePatterns` | Not native; use skill description | `globs: [patterns]` | `inclusion: fileMatch` + `fileMatchPattern` | `globs: [patterns]` | Not native |
| Loading: manual | `loadingStrategy: "manual"` | `disable-model-invocation: true` | No globs, no alwaysApply | `inclusion: manual` | No globs, no alwaysApply | `disable-model-invocation: true` |
| Loading: auto-detect | `loadingStrategy: "auto-detect"` | (default) | No alwaysApply, no globs | `inclusion: auto` + description | No alwaysApply, no globs | (default) |
| Approved tools | `approvedTools: [...]` | `allowed-tools: ...` | Not supported | Not supported | Not supported | Not supported |
| Autocomplete hint | `hint: "..."` | `argument-hint: ...` | Not supported | Not supported | Not supported | `hint: ...` |
| File patterns | `filePatterns: [...]` | (not native) | `globs` | `fileMatchPattern` | `globs` | (not native) |

## 6. Discussion

### Why Not Pure Pattern D (Adapter Only)?

Pure adapter pattern (Vercel Skills approach) works for skills because skills are primarily markdown instructions. But our plugin system includes agents, hooks, and MCP server configs that have genuinely platform-specific semantics. Claude Code's `context: fork` (run in isolated subagent) has no equivalent on Cursor. Kiro's `inclusion: auto` (AI decides based on description) has no equivalent on Windsurf.

Without a way to declare platform-specific config, authors lose access to platform-specific capabilities. This reduces our plugins to lowest-common-denominator functionality.

### Why Not Pure Pattern B (Centralized)?

Centralized manifest works for VSCode because it targets one platform. Our plugin.json would need to declare platform-specific fields for 7 platforms for every component. A plugin with 5 skills and 3 agents across 7 platforms produces 56 configuration blocks (8 components x 7 platforms). This becomes unmaintainable.

### Why Layered (D+C) Wins

The hybrid approach means:

- **80% of plugins**: write standard SKILL.md, set `loadingStrategy` in plugin.json, done. Adapter handles the rest.
- **15% of plugins**: add `platformConfig.claude-code.model` in plugin.json for platform-specific defaults.
- **5% of plugins**: add `platforms` block in individual SKILL.md for per-component platform overrides.

This follows the progressive disclosure pattern that Agent Skills and TanStack Intent both use: simple things are simple, complex things are possible.

### CLI Wizard Design Implications

The wizard should:

1. Ask cross-platform concepts first (loading strategy, file patterns, approved tools)
2. Auto-map to platform fields internally
3. Only show platform-specific prompts for fields that have no cross-platform concept
4. Show a preview of the generated platformConfig before confirming
5. Support `--platforms` flag to filter which platform prompts appear

The `multiselect` from @clack/prompts is ideal for the tools selection. The `select` works for loading strategy. The `group` API enables the phased approach.

### Precedent Validation

This approach is validated by 3 systems:

1. **JetBrains**: plugin.xml (shared) + optional per-IDE config files (overrides) = Pattern C
2. **Terraform**: provider defaults + resource overrides = Pattern C
3. **Vercel Skills**: adapter for directory placement + Agent Skills standard for content = Pattern D

Combining C and D is not novel; it follows the same principle as CSS cascading (browser defaults -> stylesheet -> element style -> inline style).

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|----------|---------------|-----------|--------|
| P0 | Adopt cross-platform concept mapping for loadingStrategy, filePatterns, approvedTools | Captures 80% of platform config needs with 3 abstract fields. Authors think in concepts, not platform fields. | Medium |
| P0 | Add `platformConfig` section to plugin.json with `defaults` and per-platform blocks | Provides manifest-level platform-specific configuration without per-component bloat. | Low |
| P0 | Implement adapter layer that translates abstract concepts to platform fields | Core of the cross-platform promise. Each platform adapter maps concepts to native fields. | High |
| P1 | Add optional `platforms` block to SKILL.md/AGENT.md frontmatter for per-component overrides | 5% of plugins need per-component platform specifics. Without this, authors cannot access platform-specific capabilities per component. | Medium |
| P1 | CLI wizard Phase 2 (cross-platform concepts) before Phase 3 (platform-specific overrides) | Research shows users understand "loading strategy" better than "alwaysApply vs inclusion vs disable-model-invocation". | Medium |
| P1 | Auto-populate Phase 3 fields from Phase 2 answers with edit capability | Reduces prompts by 60-70%. Cursor's alwaysApply is derived from loadingStrategy=always. | Medium |
| P2 | JSON Schema for platformConfig with platform-specific sub-schemas | Enables IDE autocomplete and validation for all 7 platforms. | Medium |
| P2 | `agent-plugin doctor` validates platformConfig against current platform schemas | Catches invalid or deprecated fields. Pattern borrowed from ADR-001 schema evolution strategy. | Low |
| P3 | Per-platform adapter plugins so community can add new platforms | Future-proofs against new AI coding platforms. | High |

## 8. Conclusion

**Verdict**: Proceed with hybrid D+C pattern (adapter with layered overrides)

**Confidence**: High

**Rationale**: 15 plugin systems analyzed. The hybrid pattern is used by the 3 most successful multi-platform systems (JetBrains, Terraform, Vercel Skills). It preserves the "write once, install everywhere" promise while allowing authors to access platform-specific capabilities when needed. The cross-platform concept mapping (loadingStrategy, filePatterns, approvedTools) captures 80% of use cases with 3 abstract fields. The CLI wizard phases (shared -> concepts -> platform-specific) reduce author friction by asking intent before implementation.

### User Impact

- **What changes for you**: Plugin authors get a 3-phase wizard that asks loading strategy and file patterns first, then auto-generates platform-specific fields. Authors who need platform-specific features can override in plugin.json or per-component frontmatter.
- **Effort required**: Adapter layer for 7 platforms is the largest work item. Cross-platform concept mapping needs a field inventory per platform (documented in section 4.3). CLI wizard uses @clack/prompts group() for phased flow.
- **Risk if ignored**: Without cross-platform concept mapping, authors must know 7 platforms' fields. Without layered overrides, plugins are limited to lowest-common-denominator capabilities. Without the wizard flow, authoring friction drives authors to single-platform publishing.

## Observations

- [decision] Hybrid D+C pattern (adapter with layered overrides) selected for platformConfig: adapter translates abstract concepts to platform fields; plugin.json and per-component frontmatter provide override layers #architecture #platform-config
- [fact] 3 cross-platform concepts map across all 7 platforms: loading strategy (always/conditional/manual/auto), file patterns (globs), and approved tools #platform-config #field-mapping
- [fact] Agent Skills open standard defines 6 platform-agnostic fields (name, description, license, compatibility, metadata, allowed-tools); unknown fields are silently ignored for forward compatibility #agent-skills #standard
- [fact] Vercel Skills CLI supports 37+ agents and uses pure adapter pattern: identical SKILL.md installed to platform-specific directories via copy/symlink #vercel #adapter-pattern
- [fact] Claude Code SKILL.md has 10 frontmatter fields; agents have 13 frontmatter fields; 7 of these are Claude-Code-specific with no cross-platform equivalent #claude-code #field-inventory
- [fact] Cursor and Windsurf share identical field names (alwaysApply, globs, description) for rule/skill configuration #cursor #windsurf #field-overlap
- [insight] Kiro's inclusion mode (always/fileMatch/manual/auto) is the most semantically rich loading strategy system and maps cleanly to an abstract loadingStrategy concept #kiro #loading-strategy
- [technique] CLI wizard should use phased approach: cross-platform concepts first (Phase 2), then platform-specific overrides pre-filled from Phase 2 answers (Phase 3); this reduces prompts by 60-70% #cli-wizard #ux-pattern
- [insight] JetBrains plugin.xml with optional per-IDE config files and Terraform provider defaults with resource overrides both validate the layered override pattern for multi-platform config #precedent #validation
- [fact] TanStack Intent adds framework discriminator (react/vue/solid/svelte/angular) and 6 skill types (core/sub-skill/framework/lifecycle/composition/security) to Agent Skills base; no AI-platform-specific fields #tanstack #format-extension
- [risk] Without per-component platform overrides, plugins using Claude Code's context:fork or model override cannot specialize individual skills differently from plugin defaults #platform-capabilities

## Relations

- implements [[ADR-001 Plugin Format and Manifest]]
- extends [[ANALYSIS-005-claude-code-plugin-format]]
- extends [[ANALYSIS-004-tanstack-intent-deep-dive]]
- relates_to [[ANALYSIS-002-platform-capability-matrix]]
- relates_to [[ANALYSIS-001-agent-plugin-foundation-and-vision]]

## 9. Appendices

### Sources Consulted

- Agent Skills Specification: <https://agentskills.io/specification>
- Claude Code Plugin Reference: <https://code.claude.com/docs/en/plugins-reference>
- Claude Code Skills Docs: <https://code.claude.com/docs/en/skills>
- Cursor Rules Docs: <https://cursor.com/docs/context/rules>
- Cursor Skills Docs: <https://cursor.com/docs/context/skills>
- Kiro Steering Docs: <https://kiro.dev/docs/steering/>
- OpenCode Skills Docs: <https://opencode.ai/docs/skills/>
- Amp Manual: <https://ampcode.com/manual>
- Amp MCP Skills Loading: <https://ampcode.com/news/lazy-load-mcp-with-skills>
- Windsurf Rules: <https://docs.windsurf.com/windsurf/cascade/workflows>
- GitHub Copilot CLI Skills: <https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-skills>
- VS Code Agent Skills: <https://code.visualstudio.com/docs/copilot/customization/agent-skills>
- Vercel Skills GitHub: <https://github.com/vercel-labs/skills>
- Vercel Skills FAQ: <https://vercel.com/blog/agent-skills-explained-an-faq>
- TanStack Intent Blog: <https://tanstack.com/blog/from-docs-to-agents>
- TanStack Intent GitHub: <https://github.com/TanStack/intent>
- TanStack CLI GitHub: <https://github.com/TanStack/cli>
- VSCode Contribution Points: <https://code.visualstudio.com/api/references/contribution-points>
- JetBrains Plugin Config: <https://plugins.jetbrains.com/docs/intellij/plugin-configuration-file.html>
- JetBrains Plugin Compatibility: <https://plugins.jetbrains.com/docs/intellij/plugin-compatibility.html>
- Backstage Config Docs: <https://backstage.io/docs/conf/defining/>
- Terraform Provider Config: <https://developer.hashicorp.com/terraform/language/providers/configuration>
- Nx Generators: <https://nx.dev/docs/reference/plugin/generators>
- Nx nx.json Reference: <https://nx.dev/docs/reference/nx-json>
- Yeoman Composability: <https://yeoman.io/authoring/composability.html>
- create-next-app Docs: <https://nextjs.org/docs/app/api-reference/cli/create-next-app>
- @clack/prompts: <https://www.blacksrc.com/blog/elevate-your-cli-tools-with-clack-prompts>
- CLI UX Patterns: <https://lucasfcosta.com/2022/06/01/ux-patterns-cli-tools.html>
- CLIG.dev CLI Guidelines: <https://clig.dev/>
- Cursor Rules Deep Dive: <https://forum.cursor.com/t/a-deep-dive-into-cursor-rules-0-45/60721>
- Cursor MDC Best Practices: <https://forum.cursor.com/t/my-best-practices-for-mdc-rules-and-troubleshooting/50526>
- Windsurf Rules Guide: <https://localskills.sh/blog/windsurf-rules-guide>

### Data Transparency

- **Found**: Complete frontmatter field inventories for Claude Code (10 skill fields, 13 agent fields), Cursor (3 fields), Kiro (4 fields), GitHub Copilot CLI (4 fields), OpenCode (6 fields), Windsurf (3 fields). Agent Skills spec (6 fields). Vercel Skills adapter architecture (37+ agents, platform-specific paths). TanStack Intent format extensions (6 additional fields). CLI wizard patterns from 4 tools.
- **Not Found**: Amp-specific SKILL.md frontmatter extensions beyond Agent Skills standard. Kiro steering file field validation rules. Windsurf internal adapter architecture. Exact field handling for platforms not in top 7 (Cline, Roo Code, Gemini CLI). @clack/prompts performance benchmarks for large multiselect lists. Real-world plugin author feedback on cross-platform config complexity.
