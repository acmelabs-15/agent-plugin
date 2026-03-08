---
title: ANALYSIS-005-claude-code-plugin-format
type: analysis
permalink: analysis/analysis-005-claude-code-plugin-format
tags:
- plugin-format
- claude-code
- research
- manifest
- architecture
---

# ANALYSIS-005 Claude Code Plugin Format

## 1. Objective and Scope

**Objective**: Document the complete Claude Code plugin format, manifest schema, directory conventions, component types, namespacing rules, marketplace integration, and lifecycle. Identify concepts to borrow for `@acmelabs-15/agent-plugin`.

**Scope**: Claude Code plugin system as of version 1.0.33+. Covers plugin.json manifest, directory structure, all component types (skills, agents, hooks, commands, MCP, LSP), marketplace.json format, namespacing, installation lifecycle, and real-world examples from the official marketplace.

## 2. Context

We are building a cross-platform AI agent plugin manager. Our design decision is that a plugin is a BUNDLE containing many skills, many agents, many prompts, many hooks, and probably a single MCP. This mirrors the Claude Code plugin model. This analysis provides the reference data for our plugin format design.

## 3. Approach

**Methodology**: Systematic documentation extraction from official Claude Code docs plus inspection of real plugin source code.

**Tools Used**: WebFetch on 6 documentation pages, git sparse-checkout of `anthropics/claude-code` repo for 4 real plugins.

**Sources consulted**:

- `https://code.claude.com/docs/en/plugins` (create plugins guide)
- `https://code.claude.com/docs/en/plugins-reference` (full technical reference)
- `https://code.claude.com/docs/en/discover-plugins` (install and lifecycle)
- `https://code.claude.com/docs/en/plugin-marketplaces` (marketplace format)
- `https://code.claude.com/docs/en/skills` (skill component format)
- `https://code.claude.com/docs/en/sub-agents` (agent component format)
- `https://code.claude.com/docs/en/hooks` (hook component format)
- `anthropics/claude-code` GitHub repo (4 real plugins inspected)

**Limitations**: No access to Claude Code source code (closed source). Schema validated against docs and real plugins only.

## 4. Plugin Manifest Schema (plugin.json)

Location: `.claude-plugin/plugin.json` inside the plugin root directory.

The manifest is optional. If omitted, Claude Code auto-discovers components in default locations and derives the plugin name from the directory name.

### Required Fields

| Field | Type | Description | Constraints |
|-------|------|-------------|-------------|
| `name` | string | Unique identifier and namespace prefix | kebab-case, no spaces |

### Metadata Fields (All Optional)

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `version` | string | Semantic version (MAJOR.MINOR.PATCH) | `"2.1.0"` |
| `description` | string | Brief explanation shown in plugin manager | `"Deployment automation tools"` |
| `author` | object | Author info: `{name, email?, url?}` | `{"name":"Dev Team","email":"dev@co.com"}` |
| `homepage` | string | Documentation URL | `"https://docs.example.com"` |
| `repository` | string | Source code URL | `"https://github.com/user/plugin"` |
| `license` | string | SPDX license identifier | `"MIT"`, `"Apache-2.0"` |
| `keywords` | string[] | Discovery tags | `["deployment","ci-cd"]` |

### Component Path Fields (All Optional)

| Field | Type | Description | Default Location |
|-------|------|-------------|------------------|
| `commands` | string or string[] | Additional command files/directories | `commands/` |
| `agents` | string or string[] | Additional agent files | `agents/` |
| `skills` | string or string[] | Additional skill directories | `skills/` |
| `hooks` | string, string[], or object | Hook config paths or inline config | `hooks/hooks.json` |
| `mcpServers` | string, string[], or object | MCP config paths or inline config | `.mcp.json` |
| `lspServers` | string, string[], or object | LSP config paths or inline config | `.lsp.json` |
| `outputStyles` | string or string[] | Output style files/directories | N/A |

### Path Behavior Rules

- Custom paths supplement default directories. They do not replace them.
- All paths must be relative to plugin root and start with `./`.
- Multiple paths can be specified as arrays.

### Environment Variables

| Variable | Description |
|----------|-------------|
| `${CLAUDE_PLUGIN_ROOT}` | Absolute path to plugin installation directory. Required in hooks, MCP configs, and scripts. |
| `${CLAUDE_SKILL_DIR}` | Directory containing the skill's SKILL.md file |
| `${CLAUDE_SESSION_ID}` | Current session ID |

### Complete Example

```json
{
  "name": "enterprise-plugin",
  "version": "2.1.0",
  "description": "Enterprise workflow automation tools",
  "author": {"name": "Dev Team", "email": "dev@co.com"},
  "homepage": "https://docs.example.com/plugin",
  "repository": "https://github.com/author/plugin",
  "license": "MIT",
  "keywords": ["workflow", "automation"],
  "commands": ["./custom/commands/special.md"],
  "agents": "./custom/agents/",
  "skills": "./custom/skills/",
  "hooks": "./config/hooks.json",
  "mcpServers": "./mcp-config.json",
  "outputStyles": "./styles/",
  "lspServers": "./.lsp.json"
}
```

## 5. Directory Structure

### Standard Layout

```
plugin-name/
+-- .claude-plugin/           # Metadata directory (only plugin.json goes here)
|   +-- plugin.json           # Plugin manifest
+-- commands/                 # Slash commands as Markdown files (legacy)
|   +-- status.md
|   +-- logs.md
+-- agents/                   # Subagent definitions as Markdown files
|   +-- security-reviewer.md
|   +-- performance-tester.md
+-- skills/                   # Agent Skills (subdirectories with SKILL.md)
|   +-- code-reviewer/
|   |   +-- SKILL.md          # Required entrypoint
|   |   +-- reference.md      # Optional supporting file
|   |   +-- scripts/          # Optional scripts
|   +-- pdf-processor/
|       +-- SKILL.md
+-- hooks/                    # Hook configurations
|   +-- hooks.json            # Main hook config
+-- settings.json             # Default settings (only "agent" key supported)
+-- .mcp.json                 # MCP server definitions
+-- .lsp.json                 # LSP server configurations
+-- scripts/                  # Hook and utility scripts
+-- LICENSE
+-- CHANGELOG.md
+-- README.md
```

### Critical Rule

Components (commands, agents, skills, hooks) MUST be at the plugin root. Only plugin.json belongs inside `.claude-plugin/`. Placing components inside `.claude-plugin/` is the most common mistake.

### File Locations Reference

| Component | Default Location | File Format |
|-----------|------------------|-------------|
| Manifest | `.claude-plugin/plugin.json` | JSON |
| Commands | `commands/` | Markdown files (.md) |
| Skills | `skills/<name>/SKILL.md` | Markdown with YAML frontmatter |
| Agents | `agents/` | Markdown with YAML frontmatter |
| Hooks | `hooks/hooks.json` | JSON |
| MCP Servers | `.mcp.json` | JSON (standard MCP config format) |
| LSP Servers | `.lsp.json` | JSON |
| Settings | `settings.json` | JSON (only `agent` key supported) |
| Output Styles | (custom path) | Markdown |

## 6. Component Types

### 6.1 Skills (Primary user-facing component)

**Location**: `skills/<skill-name>/SKILL.md`
**Alternate location**: `commands/<name>.md` (legacy, still supported)
**Format**: Markdown with YAML frontmatter

Skills follow the [Agent Skills](https://agentskills.io) open standard.

#### SKILL.md Frontmatter Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `name` | No | string | Display name. Defaults to directory name. Lowercase, hyphens, max 64 chars. |
| `description` | Recommended | string | What the skill does. Claude uses this for auto-invocation decisions. |
| `argument-hint` | No | string | Autocomplete hint. Example: `[issue-number]` |
| `disable-model-invocation` | No | bool | If true, only user can invoke via `/name`. Default: false. |
| `user-invocable` | No | bool | If false, hidden from `/` menu. Default: true. |
| `allowed-tools` | No | string | Comma-separated tools allowed without permission. Example: `Read, Grep, Glob` |
| `model` | No | string | Model override for this skill. |
| `context` | No | string | Set to `fork` to run in isolated subagent context. |
| `agent` | No | string | Subagent type when `context: fork`. Options: `Explore`, `Plan`, `general-purpose`, or custom. |
| `hooks` | No | object | Hooks scoped to this skill's lifecycle. |

#### String Substitutions

| Variable | Description |
|----------|-------------|
| `$ARGUMENTS` | All arguments passed after skill name |
| `$ARGUMENTS[N]` or `$N` | Specific argument by 0-based index |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `${CLAUDE_SKILL_DIR}` | Directory containing SKILL.md |
| `` !`command` `` | Preprocessing: shell command output injected before Claude sees content |

#### Skill Directory Structure

```
my-skill/
+-- SKILL.md           # Main instructions (required)
+-- reference.md       # Optional reference material
+-- examples/          # Optional examples
+-- scripts/           # Optional executable scripts
```

#### Example: Real Plugin Skill (commit-commands)

```yaml
---
allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git commit:*)
description: Create a git commit
---

## Context
- Current git status: !`git status`
- Current git diff: !`git diff HEAD`
- Current branch: !`git branch --show-current`
- Recent commits: !`git log --oneline -10`

## Your task
Based on the above changes, create a single git commit.
```

### 6.2 Agents (Subagents)

**Location**: `agents/<name>.md`
**Format**: Markdown with YAML frontmatter (frontmatter = config, body = system prompt)

#### Agent Frontmatter Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `name` | Yes | string | Unique identifier. Lowercase, hyphens. |
| `description` | Yes | string | When Claude should delegate. Include examples. |
| `tools` | No | string/string[] | Allowlist of tools. Inherits all if omitted. |
| `disallowedTools` | No | string/string[] | Denylist of tools. |
| `model` | No | string | `sonnet`, `opus`, `haiku`, or `inherit` (default) |
| `permissionMode` | No | string | `default`, `acceptEdits`, `dontAsk`, `bypassPermissions`, `plan` |
| `maxTurns` | No | int | Maximum agentic turns. |
| `skills` | No | string[] | Skills to preload into agent context. |
| `mcpServers` | No | object | MCP servers available to this agent. |
| `hooks` | No | object | Lifecycle hooks scoped to this agent. |
| `memory` | No | string | Persistent memory scope: `user`, `project`, `local`. |
| `background` | No | bool | Always run as background task. Default: false. |
| `isolation` | No | string | Set to `worktree` for git worktree isolation. |
| `color` | No | string | UI background color identifier. |

#### Example: Real Plugin Agent (code-reviewer from pr-review-toolkit)

```yaml
---
name: code-reviewer
description: Use this agent when you need to review code...
model: opus
color: green
---

You are an expert code reviewer...
```

### 6.3 Hooks

**Location**: `hooks/hooks.json` or inline in plugin.json
**Format**: JSON

#### Hook Events (17 total)

| Event | When It Fires | Matcher Input |
|-------|---------------|---------------|
| `SessionStart` | Session begins/resumes | How started: `startup`, `resume`, `clear`, `compact` |
| `UserPromptSubmit` | User submits prompt | No matcher |
| `PreToolUse` | Before tool execution (can block) | Tool name |
| `PermissionRequest` | Permission dialog shown | Tool name |
| `PostToolUse` | After successful tool use | Tool name |
| `PostToolUseFailure` | After failed tool use | Tool name |
| `Notification` | Notification sent | Notification type |
| `SubagentStart` | Subagent spawned | Agent type name |
| `SubagentStop` | Subagent completes | Agent type name |
| `Stop` | Claude finishes responding | No matcher |
| `TeammateIdle` | Agent team teammate about to idle | No matcher |
| `TaskCompleted` | Task marked completed | No matcher |
| `InstructionsLoaded` | CLAUDE.md loaded | No matcher |
| `ConfigChange` | Config file changes | Config source |
| `WorktreeCreate` | Git worktree created | No matcher |
| `WorktreeRemove` | Git worktree removed | No matcher |
| `PreCompact` | Before context compaction | Trigger: `manual`, `auto` |
| `SessionEnd` | Session terminates | Why ended |

#### Hook Handler Types

| Type | Description |
|------|-------------|
| `command` | Execute shell command. Receives JSON on stdin. |
| `prompt` | Evaluate prompt with LLM. Uses `$ARGUMENTS` placeholder. |
| `agent` | Run agentic verifier with tools for complex verification. |

#### Hook Configuration Example

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ${CLAUDE_PLUGIN_ROOT}/hooks/security_reminder_hook.py"
          }
        ]
      }
    ]
  }
}
```

### 6.4 MCP Servers

**Location**: `.mcp.json` at plugin root, or inline in plugin.json
**Format**: Standard MCP server configuration

```json
{
  "mcpServers": {
    "plugin-database": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/db-server",
      "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config.json"],
      "env": {"DB_PATH": "${CLAUDE_PLUGIN_ROOT}/data"},
      "cwd": "${CLAUDE_PLUGIN_ROOT}"
    }
  }
}
```

Plugin MCP servers start automatically when the plugin is enabled.

### 6.5 LSP Servers

**Location**: `.lsp.json` at plugin root, or inline in plugin.json
**Format**: JSON mapping language names to server configs

#### Required Fields

| Field | Description |
|-------|-------------|
| `command` | LSP binary to execute (must be in PATH) |
| `extensionToLanguage` | Maps file extensions to language identifiers |

#### Optional Fields

| Field | Description |
|-------|-------------|
| `args` | Command-line arguments |
| `transport` | `stdio` (default) or `socket` |
| `env` | Environment variables |
| `initializationOptions` | Options for server init |
| `settings` | Workspace settings |
| `workspaceFolder` | Workspace folder path |
| `startupTimeout` | Max startup wait (ms) |
| `shutdownTimeout` | Max shutdown wait (ms) |
| `restartOnCrash` | Auto-restart on crash |
| `maxRestarts` | Max restart attempts |

### 6.6 Settings

**Location**: `settings.json` at plugin root
**Format**: JSON. Only `agent` key currently supported.

```json
{
  "agent": "security-reviewer"
}
```

This activates a plugin agent as the main thread agent.

## 7. Namespacing Rules

### Plugin Namespace

The plugin `name` field is the namespace. All components are prefixed:

| Component | Standalone | Plugin |
|-----------|-----------|--------|
| Skill/Command | `/hello` | `/plugin-name:hello` |
| Agent | `my-agent` | `plugin-name:my-agent` |

### Name Constraints

- kebab-case only (lowercase letters, numbers, hyphens)
- No spaces
- Must be unique within installation scope

### Namespace Purpose

Prevents conflicts when multiple plugins define components with the same base name. The colon (`:`) is the namespace separator.

### Precedence (highest to lowest)

1. Enterprise/managed
2. Personal (`~/.claude/`)
3. Project (`.claude/`)
4. Plugin (namespaced, so no conflicts with above)

## 8. Marketplace Integration

### marketplace.json Schema

Location: `.claude-plugin/marketplace.json` in repository root.

#### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Marketplace identifier (kebab-case) |
| `owner` | object | `{name: string, email?: string}` |
| `plugins` | array | List of plugin entries |

#### Optional Metadata

| Field | Type | Description |
|-------|------|-------------|
| `$schema` | string | JSON schema URL |
| `version` | string | Marketplace version |
| `description` | string | Marketplace description |
| `metadata.pluginRoot` | string | Base directory for relative source paths |

#### Plugin Entry Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `name` | Yes | string | Plugin identifier |
| `source` | Yes | string or object | Where to fetch plugin |
| `description` | No | string | Plugin description |
| `version` | No | string | Plugin version |
| `author` | No | object | Author info |
| `homepage` | No | string | Documentation URL |
| `repository` | No | string | Source code URL |
| `license` | No | string | SPDX license |
| `keywords` | No | string[] | Discovery tags |
| `category` | No | string | Plugin category |
| `tags` | No | string[] | Searchability tags |
| `strict` | No | bool | If true (default), plugin.json is authority. If false, marketplace entry is authority. |
| `commands` | No | string/string[] | Custom command paths |
| `agents` | No | string/string[] | Custom agent paths |
| `hooks` | No | string/object | Hook config |
| `mcpServers` | No | string/object | MCP config |
| `lspServers` | No | string/object | LSP config |

#### Plugin Source Types

| Source | Format | Key Fields |
|--------|--------|------------|
| Relative path | `"./plugins/my-plugin"` | N/A |
| GitHub | object | `source: "github"`, `repo`, `ref?`, `sha?` |
| Git URL | object | `source: "url"`, `url` (must end .git), `ref?`, `sha?` |
| Git subdirectory | object | `source: "git-subdir"`, `url`, `path`, `ref?`, `sha?` |
| npm | object | `source: "npm"`, `package`, `version?`, `registry?` |
| pip | object | `source: "pip"`, `package`, `version?`, `registry?` |

#### Reserved Marketplace Names

`claude-code-marketplace`, `claude-code-plugins`, `claude-plugins-official`, `anthropic-marketplace`, `anthropic-plugins`, `agent-skills`, `life-sciences`

### Real Marketplace Example (official)

```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "claude-code-plugins",
  "version": "1.0.0",
  "description": "Bundled plugins for Claude Code",
  "owner": {"name": "Anthropic", "email": "support@anthropic.com"},
  "plugins": [
    {
      "name": "commit-commands",
      "description": "Commands for git commit workflows",
      "version": "1.0.0",
      "author": {"name": "Anthropic"},
      "source": "./plugins/commit-commands",
      "category": "productivity"
    }
  ]
}
```

## 9. Plugin Lifecycle

### Installation

| Method | Command | Details |
|--------|---------|---------|
| From marketplace | `claude plugin install <name>@<marketplace>` | Copies to cache |
| Local testing | `claude --plugin-dir ./my-plugin` | Loads directly, no copy |
| CLI flags | `--scope user\|project\|local` | Controls settings file |

### Installation Scopes

| Scope | Settings File | Shareable |
|-------|---------------|-----------|
| `user` (default) | `~/.claude/settings.json` | No |
| `project` | `.claude/settings.json` | Yes (via VCS) |
| `local` | `.claude/settings.local.json` | No (gitignored) |
| `managed` | Admin-controlled | Yes (read-only) |

### Plugin Cache

Marketplace plugins are copied to `~/.claude/plugins/cache`. Path traversal outside plugin root is blocked. Symlinks are honored during copy.

### Lifecycle Commands

| Command | Action |
|---------|--------|
| `claude plugin install <plugin>` | Install from marketplace |
| `claude plugin uninstall <plugin>` | Remove (aliases: `remove`, `rm`) |
| `claude plugin enable <plugin>` | Re-enable disabled plugin |
| `claude plugin disable <plugin>` | Disable without uninstalling |
| `claude plugin update <plugin>` | Update to latest version |
| `claude plugin validate .` | Validate plugin structure |
| `/reload-plugins` | Apply plugin changes without restart |

### Auto-Update

Official marketplaces have auto-update enabled by default. Third-party marketplaces default to disabled. Controlled via environment variables: `DISABLE_AUTOUPDATER`, `FORCE_AUTOUPDATE_PLUGINS`.

### Version Resolution

Version in `plugin.json` takes priority over marketplace entry version. Cache paths are version-dependent. Same version = skipped update.

## 10. Real Plugin Examples

### Example 1: commit-commands (Minimal, commands only)

```
commit-commands/
+-- .claude-plugin/
|   +-- plugin.json    # {"name":"commit-commands","version":"1.0.0",...}
+-- commands/
|   +-- commit.md
|   +-- commit-push-pr.md
|   +-- clean_gone.md
+-- README.md
```

### Example 2: pr-review-toolkit (Agents + commands)

```
pr-review-toolkit/
+-- .claude-plugin/
|   +-- plugin.json
+-- agents/
|   +-- code-reviewer.md
|   +-- silent-failure-hunter.md
|   +-- code-simplifier.md
|   +-- comment-analyzer.md
|   +-- pr-test-analyzer.md
|   +-- type-design-analyzer.md
+-- commands/
|   +-- review-pr.md
+-- README.md
```

### Example 3: security-guidance (Hooks only)

```
security-guidance/
+-- .claude-plugin/
|   +-- plugin.json
+-- hooks/
|   +-- hooks.json
|   +-- security_reminder_hook.py
```

### Example 4: plugin-dev (Skills + agents + commands, no plugin.json)

```
plugin-dev/
+-- agents/
|   +-- agent-creator.md
|   +-- skill-reviewer.md
|   +-- plugin-validator.md
+-- commands/
|   +-- create-plugin.md
+-- skills/
|   +-- command-development/
|   |   +-- SKILL.md
|   |   +-- references/
|   |   +-- examples/
|   +-- hook-development/
|   |   +-- SKILL.md
|   |   +-- references/
|   |   +-- examples/
|   |   +-- scripts/
|   +-- plugin-structure/
|   |   +-- SKILL.md
|   |   +-- references/
|   |   +-- examples/
|   +-- agent-development/
|   |   +-- SKILL.md
|   |   +-- references/
|   |   +-- examples/
|   |   +-- scripts/
|   +-- mcp-integration/
|   |   +-- SKILL.md
|   |   +-- references/
|   |   +-- examples/
|   +-- plugin-settings/
|       +-- SKILL.md
|       +-- references/
|       +-- examples/
|       +-- scripts/
+-- README.md
```

This plugin has no `.claude-plugin/plugin.json`. It relies on the marketplace entry with `strict: false` (implied) for all metadata.

## 11. Concepts to Borrow for @acmelabs-15/agent-plugin

### Borrow Directly

| Concept | Why | Adaptation Notes |
|---------|-----|------------------|
| Bundle model (plugin = many components) | Matches our design decision exactly | Our plugin contains skills, agents, prompts, hooks, MCP |
| Convention-over-configuration discovery | Reduces boilerplate. Default locations auto-discovered. | Use same directory names: `skills/`, `agents/`, `hooks/` |
| Manifest at `.claude-plugin/plugin.json` | Separates metadata from content | Consider `plugin.json` or `plugin.yaml` at root instead (simpler) |
| Namespace via plugin name + colon separator | Prevents multi-plugin conflicts | Adopt `plugin-name:component-name` pattern |
| SKILL.md with YAML frontmatter | Simple, human-readable, version-controllable | Core skill format |
| Agent as markdown file with frontmatter | Simple agent definition | Extend with platform-specific fields |
| hooks.json with event/matcher/handler model | Clean separation of concerns | Adapt events to our lifecycle |
| `${PLUGIN_ROOT}` variable | Portable paths regardless of install location | Rename to `${AGENT_PLUGIN_ROOT}` or similar |
| Marketplace as JSON catalog | Clean distribution model | Build our registry format on this |
| Installation scopes (user/project/local) | Matches real-world usage patterns | Adopt directly |
| Plugin cache with copy semantics | Security + isolation | Adopt directly |
| Semantic versioning with cache-busting | Update detection | Adopt directly |
| `settings.json` for plugin defaults | Configurable behavior | Extend beyond just `agent` key |

### Modify or Extend

| Concept | Why Modify | Our Approach |
|---------|-----------|--------------|
| Manifest location `.claude-plugin/plugin.json` | Nested directory is error-prone (common mistake per docs) | Consider `plugin.json` at plugin root |
| `commands/` vs `skills/` distinction | Legacy confusion | Unify under `skills/` only |
| Hook types (command, prompt, agent) | We need platform-agnostic hooks | Abstract hook handler types |
| LSP servers in plugin | Too Claude-Code-specific | Drop LSP. Replace with generic "services" concept. |
| `outputStyles` component | Niche, Claude-Code-specific | Drop or generalize to "themes" |
| `settings.json` limited to `agent` key | Too restrictive | Support full plugin configuration |
| Plugin source types (npm, pip, github, git-subdir) | Good breadth. Missing container registries. | Add OCI/container source type |

### Skip (Too Platform-Specific)

| Concept | Why Skip |
|---------|----------|
| `${CLAUDE_SESSION_ID}` | Claude-Code-specific runtime |
| `` !`command` `` preprocessing syntax | Tied to Claude Code skill runner |
| `context: fork` / subagent spawning | Runtime behavior, not format |
| `permissionMode` values | Claude-Code-specific permission model |
| `disable-model-invocation` | Claude-Code-specific invocation model |
| `background` / `isolation: worktree` | Claude-Code-specific execution model |

### Key Design Insights from Claude Code

1. **Optional manifest works**: Claude Code proves that convention-based discovery with an optional manifest is viable. Plugin can work with zero config if files are in the right places.

2. **Frontmatter over JSON for components**: Skills and agents use YAML frontmatter in markdown files. This is more human-friendly than separate JSON configs per component.

3. **Single namespace separator**: The colon (`:`) cleanly separates plugin name from component name. Simple and effective.

4. **Strict vs non-strict mode**: The marketplace `strict` flag lets either the plugin manifest or the marketplace entry be the authority. This flexibility supports both self-describing plugins and curator-controlled distribution.

5. **Real plugins are simple**: All 13 official plugins use only 1-3 component types each. Most use just commands + agents. The format supports complexity but real usage is simple.

6. **Cache + copy semantics**: Plugins are copied to a local cache on install. This prevents mutation and provides security isolation. The tradeoff is that symlinks are needed for shared resources.

7. **Version drives update detection**: If version string does not change, update is skipped. This is simple and deterministic.

## Observations

- [fact] Claude Code plugin manifest requires only the `name` field. All other fields are optional. #plugin-format
- [fact] Plugin components are auto-discovered in conventional directories (skills/, agents/, hooks/, commands/). Manifest can override or supplement with custom paths. #convention-over-configuration
- [fact] The namespace separator is colon (`:`). Plugin `my-plugin` with skill `hello` becomes `/my-plugin:hello`. #namespacing
- [technique] Marketplace uses `strict` boolean (default true) to control whether plugin.json or marketplace entry is the authority for component definitions. #marketplace
- [fact] Plugins are cached at `~/.claude/plugins/cache/` after installation. Path traversal outside plugin root is blocked. #security
- [decision] Skills use YAML frontmatter in Markdown files as the component definition format. 10 frontmatter fields documented. #skill-format
- [fact] Agents use YAML frontmatter in Markdown files with 13 frontmatter fields. The markdown body is the system prompt. #agent-format
- [fact] 17 hook event types exist covering the full session lifecycle. Hook handlers can be command, prompt, or agent type. #hooks
- [insight] Real plugins from the official marketplace are simple. 13 plugins inspected, none uses all component types. Most use 1-3 types. #real-world-usage
- [fact] Plugin source types in marketplace: relative path, GitHub, Git URL, git-subdir, npm, pip. 6 total. #distribution

## Relations

- relates_to [[ADR-003 Adapter Implementation Decisions]]
- relates_to [[TASK-008 Create Go Claude Code Adapter]]
- relates_to [[TASK-013 Remove Apps Claude Plugin]]
