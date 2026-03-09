---
title: ADR-014 Explicit Installation Model and Content Features
type: decision
permalink: decisions/adr-014-explicit-installation-model-and-content-features
tags:
- installation
- distribution
- features
- manifest
- lockfile
- commands
- architecture
- content-types
---

# ADR-014 Explicit Installation Model and Content Features

---
status: "accepted"
date: "2026-03-09"
decision-makers: "Peter Kloss, Agent Plugin Core Team"
consulted: "Architect agent, Analyst agent, Independent Thinker agent, High-Level Advisor agent"
informed: "All project contributors"
---

## Status

**Accepted** (2026-03-09) — Round 2: 5 Accept + 1 D&C

Supersedes ADR-013 (npm-Package Distribution and Revised Command Tree). ADR-013 remains accepted as a historical record. This ADR captures the design pivot that occurred immediately after ADR-013 acceptance.

## Context and Problem Statement

ADR-013 adopted the TanStack Intent model: plugins ship as npm packages, bun handles all package management, and `agent-plugin install` auto-wires content via a `postinstall` lifecycle hook. The user accepted ADR-013 (Round 2: 5 Accept + 1 D&C) then immediately pivoted to a different model after reviewing the Vercel Skills CLI approach.

The pivot was driven by three concerns:

1. **Implicit wiring is opaque.** The postinstall hook fires silently during `bun add/remove/update`. Users cannot see what changed in their AI platform configs without running `agent-plugin list` after every dependency operation.

2. **No feature selection.** ADR-013's `installMode: bundle | collection` binary was already flagged as insufficient by 2 of 6 ADR-001 debate reviewers. The postinstall model wires everything unconditionally. Users cannot select which skills, agents, or hooks they want from a plugin.

3. **Source resolution UX matters.** ADR-013 delegates source resolution to bun, which supports git repos (`github:owner/repo`, `git+https://`) and local paths (`file:../`) in addition to npm. However, bun requires protocol prefixes (`github:`, `git+`, `file:`) and resolves sources as npm dependencies into `node_modules`. The Vercel Skills model handles source resolution directly with simpler syntax (bare `owner/repo`) and without creating npm dependency entries. Direct source resolution also enables fetching without side effects on `package.json` or `bun.lockb`.

The Vercel Skills CLI demonstrated that an explicit `add/remove/update` command model with a custom lockfile provides better UX: the user sees exactly what happens, selects features interactively, and the lockfile enables team synchronization via git.

How should `@acmelabs-15/agent-plugin` install plugins, manage state, select features, and structure its content model?

## Decision Drivers

- **Explicit over implicit**: Users should invoke a command and see what it does, not discover changes after a silent postinstall hook
- **Feature selection**: Irrelevant skills in agent context consume context window tokens and degrade agent behavior (ANALYSIS-043). Users need per-feature granularity.
- **Source flexibility**: npm packages, git repos (owner/repo shorthand, full URLs), and local paths must all work as plugin sources (matching Vercel Skills)
- **Team synchronization**: A lockfile checked into version control enables `agent-plugin install` to restore the exact plugin set for all team members
- **No unnecessary state files**: The lockfile and platform configs are sufficient. No separate config file.
- **Platform convergence**: Three AI platforms (Claude Code, Copilot CLI, Cursor) independently converged on `plugin.json` as the manifest filename. The manifest location should follow Claude Code's `.claude-plugin/plugin.json` convention for disambiguation.
- **Content type completeness**: The 6-type model from ADR-001 (skills, agents, prompts, hooks, commands, mcpServers) is incomplete. Rules, AGENTS.md, and CLI are missing. Prompts and instructions need reclassification.

## Considered Options

- Option A: Keep ADR-013 postinstall model with feature selection bolted on
- Option B: Explicit add/remove/update commands with lockfile and features model (chosen)
- Option C: Hybrid model with postinstall for wiring and explicit commands for feature selection

## Pros and Cons of Rejected Options

### Option A: Keep ADR-013 Postinstall Model

- Good, because bun handles all package management (zero custom source resolution)
- Good, because postinstall fires automatically on dependency changes
- Bad, because wiring happens silently with no user visibility
- Bad, because no mechanism for feature selection without a separate state file
- Bad, because requires bun protocol prefixes for non-npm sources (`github:`, `git+`, `file:`) and creates npm dependency entries for all plugins
- Bad, because `installMode: bundle | collection` binary was already flagged as insufficient

### Option C: Hybrid (Postinstall Wiring + Explicit Feature Selection)

- Good, because retains automatic wiring from ADR-013
- Bad, because two installation paths create confusion ("did postinstall wire it, or did I select it?")
- Bad, because feature selection state must be persisted somewhere, contradicting ADR-013's stateless principle
- Bad, because postinstall still wires everything by default, requiring a second step to prune features

## Decision Outcome

Chosen option: "Option B: Explicit add/remove/update commands with lockfile and features model", because it provides full user control over installation, feature selection, and team synchronization while supporting all source types. Eight interconnected decisions implement this model.

### Decision 1: Explicit Add/Remove/Update Model

The `agent-plugin add <source>` command fetches a plugin from any supported source, runs a feature wizard, and installs content directly to AI platform configuration locations. This replaces ADR-013's postinstall auto-wiring model.

**Source types supported** (by agent-plugin, not bun):

| Source | Example | Resolution |
|---|---|---|
| npm package | `agent-plugin add @scope/plugin-name` | npm registry fetch |
| GitHub shorthand | `agent-plugin add owner/repo` | GitHub API |
| Full git URL | `agent-plugin add https://github.com/owner/repo.git` | git clone |
| Local path | `agent-plugin add ./path/to/plugin` | filesystem read |

This follows the Vercel Skills CLI model where `npx skills` fetches skills from git repos and copies content to platform-specific directories.

**Consumer commands**:

| Command | Behavior |
|---|---|
| `add <source>` | Fetch plugin, run feature wizard, install content to platform configs, update lockfile |
| `remove [plugin]` | Remove plugin content from platform configs and lockfile. Multiselect picker if no argument. |
| `update [plugin]` | Check for updates. Show updatable plugins with current and latest versions. Apply selected updates. |
| `install` | Restore all plugins from `.agent-lock.json` (team sync via git, analogous to `npm install` from package-lock.json) |
| `list` | Show installed plugins with source, version, and feature selections |

**Content placement**: Content is placed DIRECTLY to platform configuration locations via platform adapters (ADR-009). No symlinks. No canonical intermediate directory. The platform config files are the single location for installed content.

**Contrast with ADR-013**: ADR-013 eliminated consumer commands entirely and delegated to `bun add/remove/update` with a postinstall hook. This ADR reinstates consumer commands as the primary installation interface.

### Decision 2: .agent-lock.json Lockfile

A lockfile named `.agent-lock.json` lives in the consuming project root. It tracks all installed plugins with enough information to reproduce the exact installation state.

**Schema**:

```json
{
  "version": 1,
  "plugins": {
    "@scope/plugin-name": {
      "source": "@scope/plugin-name",
      "sourceType": "npm",
      "version": "1.2.3",
      "hash": "sha256:abc123...",
      "features": ["async", "bundle", "server"],
      "installedAt": "2026-03-09T12:00:00Z",
      "updatedAt": "2026-03-09T12:00:00Z"
    },
    "owner/repo": {
      "source": "owner/repo",
      "sourceType": "git",
      "ref": "abc123def456789",
      "hash": "sha256:def456...",
      "features": ["code-review", "lint"],
      "installedAt": "2026-03-09T14:00:00Z",
      "updatedAt": "2026-03-09T14:00:00Z"
    }
  }
}
```

**Fields per plugin entry**:

| Field | Type | Description |
|---|---|---|
| `source` | string | Original source identifier as provided to `add` |
| `sourceType` | `"npm" \| "git" \| "local"` | How the source was resolved |
| `version` | string \| null | Resolved version for npm sources (e.g., `"1.2.3"`). Null for local sources. |
| `ref` | string \| null | Resolved commit SHA for git sources (e.g., `"abc123def"`). Null for npm and local sources. |
| `hash` | string | SHA-256 hash of all content files listed in the manifest, computed deterministically (sorted file paths, concatenated content) |
| `features` | string[] | Selected feature names from the feature wizard |
| `installedAt` | string | ISO 8601 timestamp of initial installation |
| `updatedAt` | string | ISO 8601 timestamp of last update |

**Team synchronization**: `agent-plugin install` (no arguments) reads `.agent-lock.json` and restores all plugins with their recorded feature selections. This enables team sync by committing `.agent-lock.json` to version control. Analogous to `npm install` restoring from `package-lock.json`.

**No consumer-side config file**: The lockfile and platform config files are the only state. No `.agent-plugin/config.json` or similar file. This reverses ANALYSIS-046's recommendation of `.agent-plugin/config.json`. The lockfile's `features` array captures what ANALYSIS-046 proposed storing in a config file.

**What this supersedes**:
- ADR-013 Decision 6: `bun.lockb` as the only lockfile. This ADR adds `.agent-lock.json` as plugin-specific state.
- ADR-003 Decision 3: `plugin-lock.json` custom lockfile. The schema and purpose differ significantly. `.agent-lock.json` tracks feature selections, not overlay contributions.

### Decision 3: .agent-plugin/plugin.json Manifest Location

The plugin manifest lives at `.agent-plugin/plugin.json` within the plugin package, not at the package root.

**Rationale**:
- `plugin.json` alone is too generic a filename. Many tools could use it. The `.agent-plugin/` directory provides unambiguous disambiguation.
- Claude Code uses `.claude-plugin/plugin.json` as its plugin manifest location. Following this convention aligns with the dominant AI platform.
- 3 of 3 AI platforms that have plugin manifest formats (Claude Code, Copilot CLI, Cursor) converged on `plugin.json` as the filename. The directory wrapper is our differentiation.

**plugin.json name vs package.json name**: These are intentionally independent (Option C from the design discussion). The `plugin.json` `name` field is the plugin identity used for namespacing, display in `agent-plugin list`, and component prefixing (`plugin-name:component-name`). It does not need to match `package.json` `name`, which is the npm package identifier. A plugin distributed as `@scope/my-helpers` can declare its plugin name as `my-helpers` without the scope prefix.

**What this supersedes**: ADR-001 Decision 2 placed the manifest at the plugin root as `plugin.json`. This ADR moves it into `.agent-plugin/plugin.json`.

### Decision 4: Content Type Taxonomy

Eight content types replace ADR-001's original six:

| Content Type | Directory/File | Description | Selection Mechanism |
|---|---|---|---|
| **Skills** | `skills/` (one SKILL.md per skill) | Agent capabilities with instructions and acceptance criteria | Section-based (markdown headers) |
| **Agents** | `agents/` | Agent persona definitions with routing rules | Section-based (markdown headers) |
| **Hooks** | `hooks/` | Lifecycle hooks (pre-commit, post-install, etc.) | Section-based (code regions) |
| **Commands** | `commands/` | User-invoked CLI operations (distinct from agent-invoked skills) | Section-based (code regions) |
| **Rules** | `rules/` | File-based prompt content (coding standards, guidelines) | File-based (include/exclude) |
| **MCP** | `mcp/` | MCP server configurations | No features (always installed) |
| **AGENTS.md** | `AGENTS.md` | Plugin-wide instructions file (replaces `instructions/` directory) | Section-based (markdown headers) |
| **CLI** | (custom) | Optional custom CLI extensions (deferred — see below) | N/A (deferred) |

**Changes from ADR-001's 6-type model**:

| ADR-001 Type | ADR-014 Replacement | Reason |
|---|---|---|
| `prompts/` | `rules/` | "Prompts" is ambiguous in AI contexts. "Rules" better describes file-based content that sets standards and guidelines. |
| `instructions/` | `AGENTS.md` | Plugin-wide instructions belong in a single AGENTS.md file (adopted by 6/7 platforms, 60K+ GitHub repos, Linux Foundation governance). A directory of instruction files is unnecessary. |
| (none) | `CLI` | Optional content type for plugins that ship custom CLI extensions. **Deferred**: CLI content type is listed for taxonomy completeness but is not specified in this ADR. Schema, directory convention, and installation behavior will be defined in a future ADR if demand emerges. Not part of MVP. |
| (none) | `commands/` elevated | Commands were added as a 6th type in ADR-001 Amendment 4 (ADR-012). This ADR confirms them as first-class with section-based features. |

### Decision 5: Features Model

The features model replaces ADR-001's `installMode: bundle | collection` binary with granular feature selection at two scopes.

**Two scopes**:

1. **Plugin-level features**: Span multiple components. Declared in the top-level `features` object in `plugin.json`. Example: a "strict" feature that enables additional rules AND additional hook checks.

2. **Component-level features**: Apply to a single component. Declared in the per-component `features` mapping that maps feature names to section arrays within the component.

**Top-level feature declaration** (in `plugin.json`):

```json
{
  "features": {
    "strict": {
      "description": "Enable strict coding standards and additional hook checks",
      "default": false,
      "requires": []
    },
    "typescript": {
      "description": "TypeScript-specific rules and patterns",
      "default": true,
      "requires": []
    },
    "testing": {
      "description": "Test writing assistance and coverage hooks",
      "default": true,
      "requires": ["typescript"]
    }
  }
}
```

**Feature declaration fields**:

| Field | Type | Description |
|---|---|---|
| `description` | string | One-line description shown in the feature wizard |
| `default` | boolean | Whether the feature is selected by default in the wizard |
| `requires` | string[] | Other feature names that must be selected if this feature is selected |

**Per-component feature mapping** (in `plugin.json`):

```json
{
  "skills": [
    {
      "path": "skills/code-review",
      "features": {
        "typescript": ["## TypeScript Patterns", "## Type Safety"],
        "testing": ["## Test Coverage"]
      }
    }
  ]
}
```

The per-component `features` object maps feature names to arrays of section identifiers. The section identifiers depend on the content type's selection mechanism (see below).

**Three selection mechanisms by content type**:

**1. Section-based for markdown content** (skills, agents, AGENTS.md):

Sections are identified by standard markdown headers. The parser uses `remark` / `unified` / `mdast-util-heading-range` to extract or exclude sections by header text.

Example SKILL.md:

```markdown
## Core Rules

These rules always apply.

## TypeScript Patterns

Use strict type checking...

## Type Safety

Avoid `any` type...

## Test Coverage

Ensure 80% branch coverage...
```

When a user deselects the "testing" feature, the "## Test Coverage" section is excluded from the compiled output written to platform configs.

**2. Section-based for code content** (hooks, commands):

Code files use `// #region feature:NAME` and `// #endregion feature:NAME` markers. A custom parser (estimated 50-100 lines) extracts or excludes regions by feature name.

Example hook:

```javascript
export function preCommit(files) {
  // Core checks always run
  validateFileNames(files);

  // #region feature:typescript
  checkTypeErrors(files.filter(f => f.endsWith('.ts')));
  // #endregion feature:typescript

  // #region feature:strict
  enforceNamingConventions(files);
  checkCyclomaticComplexity(files);
  // #endregion feature:strict
}
```

**3. File-based for rules**:

Individual rule files in `rules/` are included or excluded based on feature selection. The per-component `features` mapping lists file paths instead of section headers:

```json
{
  "rules": {
    "path": "rules/",
    "features": {
      "typescript": ["rules/ts-strict.md", "rules/ts-patterns.md"],
      "testing": ["rules/test-coverage.md"]
    }
  }
}
```

**MCP has NO features**: MCP server configurations are always installed as-is. They cannot be partially installed because MCP servers expose tools as an atomic unit. Skipping part of an MCP server breaks the tool contract.

**Feature wizard**: During `agent-plugin add`, a multiselect wizard (via `@clack/prompts`) presents all declared features with their descriptions and defaults. Features with `requires` dependencies are auto-selected when their dependent feature is selected.

```text
$ agent-plugin add @scope/code-tools

Select features:
  [x] typescript    TypeScript-specific rules and patterns
  [x] testing       Test writing assistance and coverage hooks
  [ ] strict        Enable strict coding standards and additional hook checks

  testing requires: typescript (auto-selected)
```

**Validation strategy**: The features model combines patterns from 4 ecosystems (Biome, Vercel, Docker Compose, Storybook) into a novel system. ANALYSIS-043 confirms no production system provides cross-type per-component cherry-picking from a single package. Before implementing all three selection mechanisms, a proof-of-concept must validate the markdown section extraction mechanism:

1. Build a PoC for skills content (markdown section extraction using `remark`/`unified`/`mdast-util-heading-range`)
2. Test with a real plugin containing 3+ features mapped to 5+ sections
3. Measure: author friction (setup time), extraction accuracy, edge cases (nested headers, ambiguous boundaries)
4. Only proceed to implement code region parsing and file-based selection after the markdown PoC validates the approach

**What this supersedes**: ADR-001 Decision 5 (`installMode: bundle | collection`). The features model provides granular selection that the bundle/collection binary could not express.

### Decision 6: Revised Command Tree

```text
agent-plugin
  Consumer Commands
    add <source>            Fetch plugin, feature wizard, install content
    remove [plugin]         Remove plugin (multiselect if no arg)
    update [plugin]         Check for updates, show versions, apply
    install                 Restore all from .agent-lock.json (team sync)
    list                    Show installed plugins with details

  Author Commands
    create                  Scaffold a new plugin project
    validate                Check .agent-plugin/plugin.json and content integrity
    build                   Build plugin for distribution

  MCP Server
    mcp serve               Start embedded MCP tool server

  Content-Type Groups
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
    rule
      create                Scaffold a new rule
      remove                Remove a rule from plugin
      list                  List rules in plugin
```

**Eliminated commands**:

| Command | Status | Reason |
|---|---|---|
| `init` | Eliminated | ADR-013's postinstall lifecycle hook model is superseded. No hooks to wire into package.json. |
| `deinit` | Eliminated | No init means no deinit. |
| `dev` | Eliminated | ADR-013 already removed this. Not needed for explicit add/remove model. |
| `complete` | Eliminated | Shell completions deferred. Not MVP. |
| `publish` | Eliminated | No registry (ADR-002). Distribution via npm publish, git push, or local path. |

**What this supersedes**: ADR-013 Decision 2 (revised command tree). ADR-013 eliminated consumer commands (add, remove, upgrade) in favor of bun. This ADR reinstates consumer commands (add, remove, update) and eliminates the init/deinit pair.

### Decision 7: Add Command Flow (8 Steps)

The `agent-plugin add <source>` command executes 8 sequential steps:

**Step 1: Resolve source**

Parse the source argument to determine type (npm, git, local). For npm sources, fetch the package. For git sources, clone the repository. For local paths, resolve to absolute path.

**Step 2: Read manifest**

Read `.agent-plugin/plugin.json` from the resolved source. If the file does not exist, abort with an error message: "No .agent-plugin/plugin.json found in source."

**Step 3: Validate manifest**

Validate the manifest against the plugin.json Zod schema (ADR-007 Decision 8). Required fields: `name`, `description`. Abort on validation failure with specific error messages.

**Step 4: Detect installed AI platforms**

Run platform detection (ADR-009): binary existence check + config directory check via `platforms.config.json`. Present detected platforms. Skip platforms that are not installed.

**Step 5: Feature wizard**

If the plugin declares features (Decision 5), present a `@clack/prompts` multiselect with:
- All features listed with descriptions
- Default selections pre-checked
- Features with `requires` dependencies auto-selected when their parent is selected
- `--all` flag skips the wizard and selects all features
- `--features <list>` flag specifies features explicitly (comma-separated), skipping the wizard
- `--ci` mode selects defaults without prompting. Use `--features` with `--ci` to override defaults in CI/Docker environments.

If the plugin declares no features, skip this step (all content installed).

**Step 6: Parse and compile content**

For each component in the manifest, apply the appropriate selection mechanism:
- **Markdown content** (skills, agents, AGENTS.md): Parse with `remark`/`unified`. Extract sections matching selected features using `mdast-util-heading-range`. Compile into output markdown.
- **Code content** (hooks, commands): Parse `// #region feature:NAME` markers. Include only regions matching selected features. Compile into output code.
- **Rules**: Include only rule files mapped to selected features.
- **MCP**: Include as-is (no feature filtering).

**Step 7: Write to platform configs**

For each detected platform, use the platform adapter (ADR-009) to write compiled content to platform-specific locations. Two install types:

| Install Type | Description | Examples |
|---|---|---|
| File-based | Copy content files to platform directories | Skills to `.claude/skills/`, agents to `.agents/` |
| Config-based | Modify JSON/YAML config files | MCP entries to `.claude/settings.json`, `.cursor/mcp.json` |

Platform adapters read the base content and merge platform-specific metadata from the `platforms` field in plugin.json (Decision 8).

**Hook merge semantics**: Hooks from multiple plugins are installed as independent entries. Each plugin's hooks are written to separate files or separate config entries namespaced by plugin name (e.g., `hooks/plugin-name--pre-commit.js`). No cross-plugin hook merging is performed. This differs from ADR-003 Decision 2's overlay/recompute model and ADR-013's full-recompute model, both of which merged hooks from multiple sources into a single entry. The explicit model avoids merge conflicts by keeping hooks isolated per plugin. Platform adapters that require a single hook entry point (e.g., Claude Code's `hooks` array in `settings.json`) add one entry per plugin hook.

**Step 8: Update lockfile**

Write or update the plugin entry in `.agent-lock.json` with source, sourceType, version/ref, hash, selected features, and timestamps.

**Output**: Print a summary of what was installed:

```text
Installed @scope/code-tools
  Features: typescript, testing
  Platforms: Claude Code, Cursor, Kiro
  Skills: 3 installed
  Hooks: 1 installed
  Rules: 4 installed
  MCP: 1 server configured
```

### Decision 8: Platform-Specific Metadata

Plugin content files (SKILL.md, agent definitions, hook implementations) are platform-agnostic. Platform-specific metadata lives in `plugin.json` per component via a `platforms` field.

**Per-component platform metadata**:

```json
{
  "skills": [
    {
      "path": "skills/code-review",
      "platforms": {
        "claude-code": {
          "allowed-tools": ["Read", "Grep", "Glob", "Bash"],
          "model": "sonnet"
        },
        "cursor": {
          "alwaysApply": false
        }
      }
    }
  ]
}
```

**How platform adapters use this**:

1. Read the base content file (e.g., `skills/code-review/SKILL.md`).
2. Read the platform-specific metadata from `plugin.json`.
3. Merge: generate platform-specific output by combining base content with platform metadata.
4. Write to the platform's expected location and format.

**Two install types**:

| Type | Mechanism | Content Types |
|---|---|---|
| **File-based** | Copy compiled content files to platform directories | Skills, agents, rules, AGENTS.md |
| **Config-based** | Modify platform JSON/YAML config files (add entries, merge objects) | MCP servers, hooks (platform-specific hook configs) |

This separates content authoring (platform-agnostic) from content delivery (platform-specific), keeping plugin authors focused on content while the platform adapter layer handles translation.

## Complete plugin.json Schema Example

The following example demonstrates all decisions in this ADR working together:

```json
{
  "name": "code-quality",
  "description": "Code quality skills, rules, and hooks for TypeScript projects",

  "features": {
    "typescript": {
      "description": "TypeScript-specific rules and patterns",
      "default": true,
      "requires": []
    },
    "testing": {
      "description": "Test writing assistance and coverage hooks",
      "default": true,
      "requires": ["typescript"]
    },
    "strict": {
      "description": "Strict coding standards and additional checks",
      "default": false,
      "requires": []
    }
  },

  "skills": [
    {
      "path": "skills/code-review",
      "features": {
        "typescript": ["## TypeScript Patterns", "## Type Safety"],
        "testing": ["## Test Coverage"],
        "strict": ["## Strict Rules"]
      },
      "platforms": {
        "claude-code": {
          "allowed-tools": ["Read", "Grep", "Glob", "Bash"]
        },
        "cursor": {
          "alwaysApply": false
        }
      }
    },
    {
      "path": "skills/refactoring",
      "features": {
        "typescript": ["## Type Refactoring"],
        "strict": ["## Advanced Refactoring"]
      }
    }
  ],

  "agents": [
    {
      "path": "agents/reviewer.md",
      "features": {
        "strict": ["## Strict Review Criteria"]
      }
    }
  ],

  "hooks": [
    {
      "path": "hooks/pre-commit.js",
      "features": {
        "typescript": ["typescript"],
        "strict": ["strict"]
      }
    }
  ],

  "rules": {
    "path": "rules/",
    "features": {
      "typescript": ["rules/ts-strict.md", "rules/ts-patterns.md"],
      "testing": ["rules/test-coverage.md", "rules/test-naming.md"],
      "strict": ["rules/naming-conventions.md", "rules/complexity-limits.md"]
    }
  },

  "mcpServers": {
    "code-analysis": {
      "command": "node",
      "args": ["mcp/server.js"]
    }
  },

  "agentsMd": "AGENTS.md"
}
```

**Key schema points**:

- `features` (top-level): Declares available features with descriptions, defaults, and dependencies. Drives the feature wizard.
- `skills[].features`: Maps feature names to markdown header arrays. Sections under these headers are included when the feature is selected.
- `hooks[].features`: Maps feature names to region name arrays. Code regions with `// #region feature:NAME` markers are included when the feature is selected.
- `rules.features`: Maps feature names to file path arrays. Listed files are included when the feature is selected.
- `mcpServers`: No features. Always installed as declared.
- `skills[].platforms`: Platform-specific metadata merged by platform adapters during installation.
- `agentsMd`: Path to the plugin's AGENTS.md file. Installed as-is (with feature-based section selection if features are mapped).

## Consequences

### Positive

- **POS-001**: Users see exactly what happens during installation. `agent-plugin add` prints a summary of installed content, selected features, and targeted platforms. No silent postinstall side effects.
- **POS-002**: Feature selection reduces context window waste. A plugin with 8 features and 5 skills can be installed with only the 2 relevant features, preventing 6 irrelevant instruction sections from consuming agent context tokens (ANALYSIS-043 motivation).
- **POS-003**: `.agent-lock.json` enables deterministic team synchronization. `agent-plugin install` restores the exact plugin set with the same feature selections across all team members.
- **POS-004**: Three source types (npm, git, local) with simple syntax (bare `owner/repo`, `./path`) provide better UX than ADR-013's bun-delegated model, which required protocol prefixes and created npm dependency entries.
- **POS-005**: No consumer-side config file simplifies the state model. Two sources of truth: `.agent-lock.json` (what was installed) and platform configs (where it was installed). No third file.
- **POS-006**: The features model replaces the insufficient `installMode: bundle | collection` binary with granular, dependency-aware feature selection. This resolves the ADR-001 Future Considerations gap ("skills + mandatory MCP").
- **POS-007**: `.agent-plugin/plugin.json` manifest location provides unambiguous disambiguation. No collision with other tools that might use a root-level `plugin.json`.
- **POS-008**: Eight content types (up from 6) cover all AI agent content patterns. AGENTS.md replaces the instructions/ directory, aligning with the AGENTS.md standard (60K+ repos, Linux Foundation governance).

### Negative

- **NEG-001**: Custom source resolution is reinstated. ADR-013 eliminated this by delegating to bun. This ADR requires agent-plugin to handle npm registry fetch, git clone, and local path resolution. Estimated 500-1000 lines of TypeScript (significantly less than ADR-008/010's 2000-3000 lines because git clone and npm fetch are simpler than ADR-008's full archive extraction pipeline).
- **NEG-002**: The features model adds authoring complexity. Plugin authors must declare features, map them to sections, and use `// #region` markers in code. Authors of simple plugins can skip features entirely (no features = all content installed).
- **NEG-003**: Three selection mechanisms (markdown headers, code regions, file-based) create a learning curve. Each content type uses a different mechanism. This is inherent to the content types having different structures.
- **NEG-004**: `.agent-lock.json` is a custom lockfile. ADR-013 eliminated custom lockfiles. This ADR reintroduces one, though simpler than ADR-003's `plugin-lock.json` (no overlay contributions, no lockfileVersion migration pipeline).
- **NEG-005**: The `add` command requires internet access for npm and git sources. `install` from lockfile also requires internet to re-fetch sources. Local path sources work offline.

### Implementation Notes

- **IMP-001**: Source resolution must handle npm registry fetch (GET `https://registry.npmjs.org/@scope/package`), git clone (Bun.spawn `git clone --depth 1 --no-recurse-submodules`), and local path resolution (`path.resolve()`). Security constraints:
  - **CWE-22 (Path Traversal)**: Reject `../`, absolute paths, and symlinks outside the plugin directory for ALL path fields in plugin.json (skills[].path, agents[].path, hooks[].path, rules.path, rules.features.*[], mcpServers.*.args[], agentsMd). Validate at Zod schema level and at runtime before file reads.
  - **CWE-494 (Download Integrity)**: Verify npm tarball integrity via registry-provided shasum. For git sources, pin to resolved commit SHA in the lockfile `ref` field.
  - **CWE-78 (Command Injection)**: For MCP server entries, display the exact `command` and `args` values in the installation summary and require explicit user confirmation. Consider an allowlist of safe commands (`node`, `bun`, `npx`, `python`, `deno`).
  - **Git clone safety**: Use `--depth 1 --no-recurse-submodules` by default. Set `GIT_CONFIG_NOSYSTEM=1` to prevent `.gitattributes` filter execution during clone.
- **IMP-002**: The markdown section parser uses `remark` + `unified` + `mdast-util-heading-range`. These are already in the dependency stack for other markdown processing. The parser extracts or excludes sections by matching heading text against the features mapping.
- **IMP-003**: The code region parser is a custom implementation (estimated 50-100 lines). It scans for `// #region feature:NAME` and `// #endregion feature:NAME` markers and includes or excludes the enclosed code based on feature selection.
- **IMP-004**: The feature wizard dependency resolution (the `requires` field) uses topological sort. If feature A requires feature B, selecting A auto-selects B. Deselecting B auto-deselects A with a warning.
- **IMP-005**: Platform adapters (ADR-009) already handle file-based and config-based writes. The `add` command compiles content, then calls the adapter layer's write methods. No changes to the adapter interface needed.
- **IMP-006**: The `remove` command must reverse all platform writes. For file-based content, delete files. For config-based content, remove entries from JSON/YAML configs. The lockfile provides the list of installed components for accurate removal.
- **IMP-007**: The `update` command checks the source for a newer version. For npm: compare registry version to lockfile version. For git: compare HEAD SHA to lockfile hash. For local: compare content hash. If newer, re-run the add flow with existing feature selections preserved.

### Confirmation

Implementation compliance will be verified through:

- **Code review**: PR reviewers confirm the add/remove/update command implementations follow the 8-step flow, feature wizard uses `@clack/prompts`, and content is placed directly to platform locations (no symlinks).
- **Integration tests**: Test suite covers all three source types, feature selection with dependencies, markdown section extraction, code region extraction, file-based rule selection, lockfile read/write, and team sync via `install`.
- **Architecture audit**: Post-implementation review confirms no postinstall hooks, no init/deinit commands, no consumer-side config file beyond `.agent-lock.json`.
- **Schema validation**: Zod schema for `plugin.json` validates all fields including features, per-component features mappings, and platform metadata.

## ADRs Affected

| ADR | Relationship | What Changes |
|---|---|---|
| ADR-013 | Superseded | All 6 decisions superseded: npm-dependency model (D1), revised command tree without consumer commands (D2), postinstall lifecycle hooks (D3), install as scan-diff-reconcile (D4), simplified plugin.json (D5), bun.lockb as only lockfile (D6). ADR-013 remains accepted as historical record. |
| ADR-001 | Amended | Decision 2: manifest location changes from root-level `plugin.json` to `.agent-plugin/plugin.json`. Decision 5: `installMode` replaced by features model. Decision 7: content types change from 6 to 8 (prompts/ becomes rules/, instructions/ becomes AGENTS.md, CLI added). |
| ADR-003 | Amended | Decision 3: `plugin-lock.json` superseded by `.agent-lock.json` with different schema and purpose. |
| ADR-008 | Partially restored | Source resolution is reinstated for npm, git, and local sources, though simpler than ADR-008's 3-decision pipeline. ADR-008 remains superseded by ADR-013; this ADR implements source resolution independently. |
| ADR-009 | Unchanged | Platform detection and config registry remain valid. Platform adapters used by the add command for content placement. |
| ADR-010 | Remains superseded | Installation lifecycle remains superseded by ADR-013. This ADR does not restore the 6-phase model. |
| ADR-007 | Partially affected | Command tree (Decision 1) changes: consumer commands restored, init/deinit eliminated. |
| ADR-011 | Unchanged | MCP tool auto-generation remains valid. |
| ADR-012 | Partially affected | Content-type command groups updated: `instruction` subcommands become `rule` subcommands. |

## Reversibility Assessment

- [x] **Rollback capability**: All decisions can be rolled back without data loss. Reverting to ADR-013's postinstall model is additive work (restore init/deinit, remove add/remove/update). Feature selections in `.agent-lock.json` can be ignored.
- [x] **Vendor lock-in**: No new vendor lock-in. Source resolution uses standard protocols (npm registry HTTP API, git clone, filesystem). No proprietary APIs.
- [x] **Exit strategy**: If the explicit model proves too complex, the postinstall model (ADR-013) can be restored. The two models share the same platform adapter layer (ADR-009) and content format.
- [x] **Legacy impact**: No existing users. Greenfield project.
- [x] **Data migration**: `.agent-lock.json` is a new file. No migration from existing state needed.

## Vendor Lock-in Assessment

**Dependency**: remark/unified ecosystem for markdown parsing
**Lock-in Level**: Low

### Lock-in Indicators

- [ ] Proprietary APIs without standards-based alternatives
- [ ] Data formats that require conversion to export
- [ ] Licensing terms that restrict migration
- [x] Integration depth that increases switching cost: markdown AST processing is tightly coupled to the remark/unified API
- [ ] Team training investment

### Exit Strategy

**Trigger conditions**: remark/unified becomes unmaintained or introduces breaking changes.
**Migration path**: Replace with any markdown parser that produces an AST (marked, micromark, markdown-it). The section extraction logic depends on heading-range identification, which any AST parser can provide.
**Estimated effort**: 1-2 days to swap the parser. The feature mapping schema and section identifiers (heading text strings) remain unchanged.
**Data export**: All data is in standard formats (JSON lockfile, markdown content files, JSON manifest).

### Accepted Trade-offs

The remark/unified ecosystem is the most widely used markdown processing toolkit in JavaScript (500M+ weekly npm downloads across the unified ecosystem). The risk of abandonment is low. The section extraction logic is a thin wrapper (~50 lines) around `mdast-util-heading-range`, making replacement straightforward.

## References

- [[ANALYSIS-033 Consumer and Author Commands]] — 12 conflicts between spec and ADRs in the consumer command layer
- [[ANALYSIS-034 Skill Versioning Models Comparison]] — TanStack Intent, Vercel Skills, Claude Code Plugins; Vercel model adopted for explicit add/remove
- [[ANALYSIS-035 Hook Merging Strategies Without Custom Lockfile]] — full recompute performance validation
- [[ANALYSIS-043 Per-Component Cherry-Picking InstallMode Design]] — features model rationale, cross-ecosystem cherry-picking analysis
- [[ANALYSIS-044 Sectioned Content Configuration Patterns]] — 13 ecosystems analyzed, Biome/Vercel/Docker/Storybook convergence on section-based selection
- [[ANALYSIS-046 Config File Naming and Placement Conventions]] — recommended .agent-plugin/config.json, overridden by Decision 2's "no config file" approach
- Vercel Skills CLI (`npx skills`) — prior art for explicit add command, git source resolution, and `.skills-lock.json` lockfile
- AGENTS.md Specification — Linux Foundation AAIF governance, 60K+ repos, 6/7 platform adoption
- [[ADR-013 npm-Package Distribution and Revised Command Tree]] — superseded design

## More Information

### Why the Pivot from ADR-013

ADR-013 was accepted after a thorough 6-agent debate (Round 2: 5 Accept + 1 Disagree-and-Commit). The design was sound for its stated goals. The pivot occurred because the user's priorities shifted after seeing the Vercel Skills CLI in action:

1. **Vercel Skills demonstrates that explicit commands work.** `npx skills` fetches from git repos, presents a skill selector, and copies content to platform directories. No postinstall hooks, no package manager dependency. Users understand what happened because they invoked the command.

2. **Feature selection cannot be bolted onto postinstall.** ADR-013's stateless model (derive state from `node_modules` + platform configs) has no place to store feature selections. Adding a state file for selections contradicts the stateless principle that justified the postinstall model.

3. **Direct source resolution provides better UX.** ADR-013 delegated to bun, which supports git and local sources but requires protocol prefixes (`github:`, `file:`) and creates npm dependency entries. The Vercel model uses bare `owner/repo` syntax and fetches without side effects on `package.json`.

The pivot preserves ADR-013's insights about simplification (eliminated custom archive extraction, reduced command surface) while restoring the user control that the postinstall model sacrificed.

### Relationship Between Features and installMode

ADR-001 Decision 5 defined `installMode: bundle | collection`:
- `bundle`: Install all components or nothing.
- `collection`: Present a chooser UI for individual components.

This binary was insufficient because it operated at the component level (whole skills, whole agents) and had no mechanism for intra-component selection. The features model operates at the section level within components, providing finer granularity.

A plugin with no `features` declaration behaves like the old `bundle` mode: all content is installed. A plugin with `features` provides the selection that `collection` promised but with feature-level (not component-level) granularity and dependency tracking via `requires`.

ANALYSIS-043 documented that no ecosystem provides cross-type per-component cherry-picking from a single package. The features model is novel but builds on proven patterns: Biome's 3-level granularity (preset/group/rule), Vercel's sectionMap compilation, and Docker Compose's profile semantics (ANALYSIS-044).

### Content Type Evolution

| Version | Content Types | Count |
|---|---|---|
| ADR-001 (original) | skills, agents, prompts, hooks, commands, mcpServers | 6 |
| ADR-001 + ADR-012 amendment | skills, agents, prompts, hooks, commands, mcpServers | 6 (commands elevated) |
| ADR-014 (this) | skills, agents, hooks, commands, rules, mcp, AGENTS.md, CLI | 8 |

The shift from `prompts/` to `rules/` reflects ecosystem terminology. "Prompts" in AI contexts refers to user input messages. "Rules" better describes the content type's purpose: file-based coding standards, guidelines, and conventions that inform agent behavior.

The shift from `instructions/` to `AGENTS.md` follows the AGENTS.md standard. A single AGENTS.md file at the plugin root replaces a directory of instruction files. This aligns with how 6 of 7 target platforms consume plugin-wide instructions.

## Observations

- [decision] Explicit add/remove/update commands replace ADR-013's postinstall auto-wiring model. User invokes commands and sees what happens. Vercel Skills CLI is the primary prior art. #installation #explicit #vercel
- [decision] .agent-lock.json lockfile in consuming project tracks source, sourceType, hash, features, timestamps per plugin. Enables team sync via `agent-plugin install`. No consumer-side config file. #lockfile #state
- [decision] Manifest location moved to .agent-plugin/plugin.json. Directory provides disambiguation. Follows Claude Code .claude-plugin/plugin.json convention. 3/3 AI platforms converged on plugin.json filename. #manifest #location
- [decision] 8 content types replace 6: Skills, Agents, Hooks, Commands, Rules (was prompts/), MCP, AGENTS.md (was instructions/), CLI (new, optional). #content-types #taxonomy
- [decision] Features model replaces installMode binary. Two scopes: plugin-level features (span components) and component-level features (section mappings). Three mechanisms: markdown headers (remark/unified), code regions (#region markers), file-based (rules). MCP has no features. #features #selection
- [decision] Revised command tree: add/remove/update/install/list (consumer) + create/validate/build (author) + mcp serve + content-type CRUD groups. init/deinit eliminated. #commands #tree
- [decision] 8-step add flow: resolve source, read manifest, validate with Zod, detect platforms, feature wizard, parse/compile content, write to platforms, update lockfile. #add-flow #installation
- [decision] Platform metadata in plugin.json per component via platforms field. Content files are platform-agnostic. Platform adapter merges metadata during write. Two install types: file-based (copy) and config-based (modify JSON/YAML). #platforms #adapters
- [fact] ADR-013 was accepted (Round 2: 5 Accept + 1 D&C) then immediately superseded by this design pivot #history #pivot
- [fact] No ecosystem provides cross-type per-component cherry-picking from a single package (ANALYSIS-043). The features model is novel design building on Biome, Vercel, Docker Compose, and Storybook patterns. #novelty #research
- [insight] Implicit postinstall wiring sacrifices visibility for convenience. For AI agent plugins, users need to see what skills and rules enter their agent context. Explicit commands preserve this visibility. #rationale #transparency
- [risk] Custom source resolution reinstated at 500-1000 lines (less than ADR-008's 2000-3000 but still custom code). #complexity #tradeoff
- [risk] Three selection mechanisms (markdown headers, code regions, file-based) create a learning curve for plugin authors #authoring #complexity

## Relations

- supersedes [[ADR-013 npm-Package Distribution and Revised Command Tree]]
- amends [[ADR-001 Plugin Format and Manifest]]
- amends [[ADR-003 Conflict Resolution and Namespacing]]
- depends_on [[ADR-009 Platform Detection and Config Registry]]
- relates_to [[ADR-007 CLI Architecture and Interaction Model]]
- relates_to [[ADR-012 Scaffolding and Content Management]]
- relates_to [[ANALYSIS-033 Consumer and Author Commands]]
- relates_to [[ANALYSIS-034 Skill Versioning Models Comparison]]
- relates_to [[ANALYSIS-035 Hook Merging Strategies Without Custom Lockfile]]
- relates_to [[ANALYSIS-043 Per-Component Cherry-Picking InstallMode Design]]
- relates_to [[ANALYSIS-044 Sectioned Content Configuration Patterns]]
- relates_to [[ANALYSIS-046 Config File Naming and Placement Conventions]]
