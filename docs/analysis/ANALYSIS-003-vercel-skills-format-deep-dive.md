---
title: ANALYSIS-003-vercel-skills-format-deep-dive
type: note
permalink: analysis/analysis-003-vercel-skills-format-deep-dive
tags:
- analysis
- vercel-skills
- format-specification
- skill-md
- cli-design
- agent-plugin
---

# ANALYSIS-003 Vercel Skills Format Deep Dive

## 1. Objective and Scope

**Objective**: Reverse-engineer the Vercel `skills` package (v1.4.4) to document its complete SKILL.md format specification, source resolution patterns, installation mechanics, state tracking, and CLI surface. Identify what `@acmelabs-15/agent-plugin` should borrow versus extend.

**Scope**: Full source code analysis of `vercel-labs/skills` repository. Example skills from `vercel-labs/agent-skills`. Excludes competitive analysis (covered in ANALYSIS-001).

## 2. Context

Vercel's `npx skills` CLI is the most popular skills installer in the AI agent ecosystem (8,800 GitHub stars, 40+ agents supported). It defines the de facto SKILL.md format. We decided to create our own plugin format but borrow concepts from Vercel's standard where sensible, extending the pattern for agents, prompts, hooks, and MCP servers.

Repository: <https://github.com/vercel-labs/skills> (MIT license)
Version analyzed: 1.4.4 (March 2026)
Package: `skills` on npm (also aliased as `add-skill`)

## 3. Approach

**Methodology**: Direct source code analysis via GitHub API. Read all TypeScript source files in `src/`. Fetched 3 example SKILL.md files from `vercel-labs/agent-skills`. Analyzed README documentation.
**Tools Used**: GitHub API (`gh api`), WebFetch (README), Brain MCP (context retrieval).
**Limitations**: Could not access `add.ts` fully (the main add command orchestration). Telemetry server-side logic not visible.

## 4. Complete SKILL.md Format Specification

### 4.1 Frontmatter Fields (YAML)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Unique identifier for the skill. Must be a string (YAML auto-parsing of numbers/booleans rejected). Used as directory name after sanitization. |
| `description` | string | Yes | Brief explanation of skill purpose. Used in listings and search. |
| `license` | string | No | License identifier (e.g., "MIT"). Not enforced by CLI. |
| `metadata` | object | No | Arbitrary key-value metadata. |
| `metadata.internal` | boolean | No | When `true`, hides skill from normal discovery. Only visible when `INSTALL_INTERNAL_SKILLS=1` is set. |
| `metadata.author` | string | No | Skill author identifier. |
| `metadata.version` | string | No | Semantic version string. |
| `metadata.argument-hint` | string | No | Hint for expected arguments (e.g., `<file-or-pattern>`). |

Parsing uses `gray-matter` library. Validation is minimal: only `name` and `description` are required, both must be strings.

### 4.2 Body Structure (Markdown)

No enforced body structure. The `npx skills init` command generates this template:

```markdown
---
name: my-skill
description: A brief description of what this skill does
---

# my-skill

Instructions for the agent to follow when this skill is activated.

## When to use

Describe when this skill should be used.

## Instructions

1. First step
2. Second step
3. Additional steps as needed
```

Real-world skills (Vercel's own) use richer structures:

- **Section: When to Apply/Use** -- trigger conditions for the skill
- **Section: Rule Categories** -- priority tables with impact ratings
- **Section: Quick Reference** -- condensed rule listings
- **Section: How to Use** -- step-by-step execution instructions
- **Supporting files**: `rules/` directory with individual rule files, `AGENTS.md` compiled document, `metadata.json` for build metadata

### 4.3 Multi-File Skills

Skills are directories, not single files. The SKILL.md is the entry point, but a skill directory can contain:

- `SKILL.md` (required, the entry point)
- `rules/*.md` (individual rule files referenced by SKILL.md)
- `AGENTS.md` (compiled full document)
- `metadata.json` (build metadata with version, organization, references)
- Any other supporting files

Files excluded during installation: `metadata.json`, `_*` prefixed files, `.git/` directories.

### 4.4 Name Sanitization

Skill names are sanitized to kebab-case for filesystem safety:

```
input -> lowercase -> replace [^a-z0-9._] with hyphens -> trim dots/hyphens from edges -> limit 255 chars
```

Fallback: `unnamed-skill` if result is empty. Path traversal validated at multiple levels.

## 5. Source Resolution Patterns

### 5.1 Source Types (ParsedSource)

The `parseSource()` function in `source-parser.ts` resolves 7 source types:

| Type | Input Pattern | Example | Resolution |
|------|--------------|---------|------------|
| `github` | `owner/repo` shorthand | `vercel-labs/agent-skills` | `https://github.com/owner/repo.git` |
| `github` | `owner/repo@skill` | `vercel-labs/agent-skills@react-best-practices` | GitHub URL + skillFilter |
| `github` | `owner/repo/path` | `vercel-labs/agent-skills/skills/deploy` | GitHub URL + subpath |
| `github` | GitHub URL with tree path | `https://github.com/owner/repo/tree/main/path` | GitHub URL + ref + subpath |
| `github` | `github:` prefix | `github:owner/repo` | Strips prefix, recurses |
| `gitlab` | GitLab URL | `https://gitlab.com/owner/repo` | GitLab URL + `.git` |
| `gitlab` | `gitlab:` prefix | `gitlab:owner/repo` | Converts to full URL |
| `gitlab` | GitLab tree URL | `gitlab.com/owner/repo/-/tree/branch/path` | GitLab URL + ref + subpath |
| `git` | Direct git URL | `https://example.com/repo.git` | Passed through |
| `local` | Absolute/relative path | `./my-skills`, `/path/to/skills` | `resolve()` to absolute |
| `well-known` | HTTP(S) non-git URL | `https://example.com` | Checks `/.well-known/skills/index.json` |

### 5.2 Source Aliases

Hardcoded aliases for common shorthand:

```typescript
const SOURCE_ALIASES: Record<string, string> = {
  'coinbase/agentWallet': 'coinbase/agentic-wallet-skills',
};
```

### 5.3 Well-Known Skills Provider (RFC 8615)

Organizations can publish skills at `https://example.com/.well-known/skills/`. The provider fetches `index.json` containing:

```json
{
  "skills": [
    {
      "name": "skill-id",
      "description": "What it does",
      "files": ["SKILL.md", "rules/rule-1.md"]
    }
  ]
}
```

### 5.4 Provider Registry

Extensible provider system with `HostProvider` interface:

```typescript
interface HostProvider {
  readonly id: string;
  readonly displayName: string;
  match(url: string): ProviderMatch;
  fetchSkill(url: string): Promise<RemoteSkill | null>;
  toRawUrl(url: string): string;
  getSourceIdentifier(url: string): string;
}
```

Providers registered in a singleton `ProviderRegistry`. Current providers: well-known. GitHub and GitLab handled via `simple-git` cloning, not the provider registry.

### 5.5 Claude Code Plugin Manifest Integration

If `.claude-plugin/marketplace.json` or `.claude-plugin/plugin.json` exists in a repository, skills declared in those manifests are discovered:

```json
{
  "metadata": { "pluginRoot": "./plugins" },
  "plugins": [
    {
      "name": "my-plugin",
      "source": "./my-plugin",
      "skills": ["./skills/review", "./skills/test"]
    }
  ]
}
```

All paths must start with `./` (Claude Code convention). Path traversal validated.

## 6. Installation Mechanics

### 6.1 Two-Tier Directory Architecture

**Canonical location**: `.agents/skills/<skill-name>/` (project) or `~/.agents/skills/<skill-name>/` (global). This is the single source of truth.

**Agent-specific locations**: Symlinks from agent directories to the canonical location. Example: `.claude/skills/<skill-name>/` symlinks to `.agents/skills/<skill-name>/`.

### 6.2 Installation Modes

| Mode | Behavior | When Used |
|------|----------|-----------|
| `symlink` (default) | Copy to canonical, symlink from agent dirs | Default for all installs |
| `copy` | Direct copy to each agent directory | Fallback when symlinks fail, or `--copy` flag |

Symlink fallback: If symlink creation fails (permissions, Windows without developer mode), automatically falls back to copy mode per agent.

### 6.3 Agent Path Mapping

40+ agents mapped with `AgentConfig`:

```typescript
interface AgentConfig {
  name: string;
  displayName: string;
  skillsDir: string;           // Project-level path (e.g., '.claude/skills')
  globalSkillsDir: string;     // Global path (e.g., '~/.claude/skills')
  detectInstalled: () => Promise<boolean>;  // Checks if agent is installed
  showInUniversalList?: boolean;
}
```

Agent detection: checks for config directory existence (e.g., `~/.claude` for Claude Code, `~/.cursor` for Cursor).

**Universal agents**: Agents using `.agents/skills` as their `skillsDir` (Amp, Cline, Codex, Cursor, Gemini CLI, GitHub Copilot, Kimi CLI, OpenCode, Replit). These share the canonical directory and skip symlink creation.

### 6.4 Conflict Handling

On reinstall/update, the skill directory is cleaned and recreated:

```typescript
await rm(path, { recursive: true, force: true });
await mkdir(path, { recursive: true });
```

No merge. Full replacement. Handles ELOOP (circular symlinks), stale symlinks, and existing directories.

### 6.5 Security Measures

- Path traversal prevention via `isPathSafe()` checks
- Subpath sanitization via `sanitizeSubpath()` rejecting `..` segments
- Name sanitization limiting characters and length
- Plugin manifest path validation (must start with `./`)
- Private repo detection via GitHub API

### 6.6 Config File Modification

The CLI does NOT modify any agent configuration files. It only places files in the expected skill directories. Each agent is responsible for discovering and loading skills from its own directory.

Exception: Kiro CLI requires manual configuration after install (adding resources to `.kiro/agents/<agent>.json`).

## 7. State Tracking

### 7.1 Global Lock File: `~/.agents/.skill-lock.json`

Tracks all installed skills globally. NOT checked into version control.

```json
{
  "version": 3,
  "skills": {
    "react-best-practices": {
      "source": "vercel-labs/agent-skills",
      "sourceType": "github",
      "sourceUrl": "https://github.com/vercel-labs/agent-skills.git",
      "skillPath": "skills/react-best-practices/SKILL.md",
      "skillFolderHash": "<GitHub tree SHA>",
      "installedAt": "2026-03-01T00:00:00.000Z",
      "updatedAt": "2026-03-05T00:00:00.000Z",
      "pluginName": "optional-plugin-group"
    }
  },
  "dismissed": {
    "findSkillsPrompt": true
  },
  "lastSelectedAgents": ["claude-code", "cursor"]
}
```

Key design choices:

- Version field with backwards-incompatible wipe (v3 added `skillFolderHash`)
- `skillFolderHash` uses GitHub Trees API SHA (changes when any file in folder changes)
- `lastSelectedAgents` remembers user's agent selection for next install
- `dismissed` tracks UI prompt dismissals

### 7.2 Local Lock File: `skills-lock.json` (project root)

Designed to be checked into version control. Intentionally minimal and timestamp-free to minimize merge conflicts.

```json
{
  "version": 1,
  "skills": {
    "react-best-practices": {
      "source": "vercel-labs/agent-skills",
      "sourceType": "github",
      "computedHash": "<SHA-256 of all file contents>"
    }
  }
}
```

Key design choices:

- No timestamps (reduces merge conflicts)
- Skills sorted alphabetically when written (deterministic output)
- Hash computed from actual file contents on disk (not GitHub API)
- Enables `npx skills install` to restore skills from lock file (like `npm install`)

### 7.3 node_modules Integration

The `sync.ts` module crawls `node_modules/` for packages containing SKILL.md files. Discovered skills are installed with `sourceType: "node_modules"`. This enables npm-based skill distribution.

## 8. CLI Command Surface

| Command | Aliases | Description |
|---------|---------|-------------|
| `npx skills add <source>` | -- | Install skills from source |
| `npx skills list` | `ls` | List installed skills |
| `npx skills find [query]` | -- | Interactive skill search via skills.sh API |
| `npx skills remove [skills]` | `rm` | Remove installed skills |
| `npx skills init [name]` | -- | Create SKILL.md template |
| `npx skills check` | -- | Check for available updates |
| `npx skills update` | -- | Update all skills to latest |
| `npx skills install` | -- | Restore skills from skills-lock.json |
| `npx skills sync` | -- | Sync skills from node_modules |

### Add Command Flags

| Flag | Short | Description |
|------|-------|-------------|
| `--global` | `-g` | Install to user directory |
| `--agent <agents...>` | `-a` | Target specific agents |
| `--skill <skills...>` | `-s` | Install specific skills by name |
| `--list` | `-l` | List available skills without installing |
| `--copy` | -- | Copy files instead of symlinking |
| `--yes` | `-y` | Skip confirmation prompts |
| `--all` | -- | Install all skills to all agents |

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `INSTALL_INTERNAL_SKILLS` | Show/install internal skills when set to `1` or `true` |
| `DISABLE_TELEMETRY` | Disable anonymous usage telemetry |
| `DO_NOT_TRACK` | Alternative telemetry disable |
| `SKILLS_API_URL` | Override skills.sh search API endpoint |
| `GITHUB_TOKEN` / `GH_TOKEN` | GitHub authentication for API calls |
| `CODEX_HOME` | Override Codex config directory |
| `CLAUDE_CONFIG_DIR` | Override Claude config directory |

## 9. Technical Architecture

### 9.1 Build System

- TypeScript compiled with `obuild` (v0.4.22)
- Zero runtime dependencies (all devDependencies bundled at build time)
- Key build deps: `gray-matter` (frontmatter parsing), `simple-git` (git operations), `@clack/prompts` (interactive UI), `picocolors` (terminal colors), `xdg-basedir` (XDG paths)
- Node.js >= 18 required

### 9.2 Skill Discovery Priority

When scanning a repository for skills:

1. Check if direct path contains SKILL.md (return immediately unless `fullDepth`)
2. Search 30+ priority directories (`.claude/skills/`, `.agents/skills/`, `skills/`, etc.)
3. Check Claude Code plugin manifests (`.claude-plugin/marketplace.json`, `plugin.json`)
4. Fallback: recursive search up to 5 levels deep
5. Deduplicate by skill name (first found wins)

### 9.3 Update Detection

Uses GitHub Trees API to fetch folder SHAs. Compares stored `skillFolderHash` against current tree SHA. Any file change in the skill directory triggers an update.

## 10. What to BORROW for @acmelabs-15/agent-plugin

### 10.1 SKILL.md Format (BORROW directly)

| Element | Borrow | Rationale |
|---------|--------|-----------|
| YAML frontmatter with `name` + `description` required | Yes | De facto standard. 8,800+ stars validate this. |
| `metadata` object for extensible fields | Yes | Clean separation of required vs optional. |
| `metadata.internal` for hidden plugins | Yes | Useful for internal/development plugins. |
| Markdown body as free-form instructions | Yes | Agents parse markdown natively. No need for custom format. |
| Multi-file skill directories | Yes | SKILL.md as entry point, supporting files alongside. |
| kebab-case name sanitization | Yes | Filesystem safety pattern is well-tested. |

### 10.2 Source Resolution (BORROW with extensions)

| Element | Borrow | Extend |
|---------|--------|--------|
| `owner/repo` GitHub shorthand | Yes | -- |
| `owner/repo@skill` skill filter | Yes | Extend to `owner/repo@plugin-type/name` |
| GitLab URL support | Yes | -- |
| Local path support | Yes | -- |
| Well-known provider pattern | Yes | Extend to `/.well-known/plugins/` |
| Provider registry interface | Yes | Add npm, pip providers |
| Source aliases | Yes | -- |

### 10.3 Installation Mechanics (BORROW with modifications)

| Element | Borrow | Modify |
|---------|--------|--------|
| Canonical directory + symlinks | Yes | Change canonical from `.agents/` to our own root |
| Two-scope model (project + global) | Yes | -- |
| Symlink with copy fallback | Yes | -- |
| Agent detection via config directory | Yes | -- |
| Path traversal security | Yes | -- |
| Clean-and-replace on reinstall | Yes | Add merge option for config files |

### 10.4 State Tracking (BORROW with extensions)

| Element | Borrow | Extend |
|---------|--------|--------|
| Global lock file concept | Yes | Use SQLite instead of JSON for query capability |
| Local lock file for VCS | Yes | Extend with plugin type and platform fields |
| Content hashing for update detection | Yes | -- |
| `lastSelectedAgents` memory | Yes | -- |
| Timestamp-free local lock (merge-friendly) | Yes | -- |

## 11. What to EXTEND Beyond Vercel's Scope

### 11.1 Additional Plugin Types

Vercel handles SKILL.md only. We need parallel formats:

| Plugin Type | Entry File | Frontmatter Extensions |
|-------------|-----------|----------------------|
| Skill | SKILL.md | Borrow Vercel format directly |
| Agent | AGENT.md | Add `persona`, `activation`, `tools`, `handoff` fields |
| Prompt | PROMPT.md | Add `variables`, `model`, `temperature` fields |
| Hook | HOOK.md | Add `trigger`, `event`, `phase` fields |
| MCP | MCP.md | Add `transport`, `tools`, `resources`, `config` fields |

### 11.2 Richer Frontmatter

Fields Vercel does not have but we need:

| Field | Type | Purpose |
|-------|------|---------|
| `type` | enum | Plugin type discriminator (skill/agent/prompt/hook/mcp) |
| `platforms` | string[] | Supported platforms (claude-code, cursor, codex, etc.) |
| `dependencies` | string[] | Other plugins this requires |
| `conflicts` | string[] | Incompatible plugins |
| `hooks` | object | Lifecycle hooks (pre-install, post-install, etc.) |
| `config` | object | User-configurable settings with defaults |
| `version` | semver | Moved from metadata to top-level, enforced |
| `author` | string | Moved from metadata to top-level |
| `allowed-tools` | string[] | Tools this plugin needs access to |

### 11.3 SQLite State Store

Replace Vercel's JSON lock files with SQLite for:

- Query installed plugins by type, platform, source
- Track dependency graphs
- Store configuration overrides
- Support the MCP server read interface
- Full-text search across installed plugin metadata

### 11.4 MCP Server Interface

Vercel has no AI-operable interface. We add:

- `plugin.list` tool -- query installed plugins
- `plugin.search` tool -- search registry
- `plugin.install` tool -- install via MCP
- `plugin.configure` tool -- adjust settings

### 11.5 Platform-Aware Installation

Vercel copies the same SKILL.md to every agent directory. We need:

- Platform-specific configuration generation (e.g., MCP config for Cursor vs Claude Code)
- Platform capability detection (does this platform support hooks? MCP? etc.)
- Conditional content in plugin files based on target platform

## 12. Key Design Insights from Vercel's Code

### 12.1 Zero Dependencies at Runtime

All dependencies are devDependencies bundled at build. This is critical for `npx` usage where install time matters. We should match this.

### 12.2 Agent Detection is Heuristic

Detection checks for config directory existence (e.g., `~/.claude` exists means Claude Code is installed). This is fragile but practical. No agent provides a reliable "am I installed?" API.

### 12.3 Universal Agent Pattern

Agents sharing `.agents/skills/` as their skill directory avoid redundant symlinks. This pattern reduces disk usage and simplifies the mental model. Our canonical directory should follow this pattern.

### 12.4 Lock File Versioning Strategy

Backwards-incompatible changes wipe the lock file and start fresh. No migration code. Simple but loses installation history. Our SQLite approach should support migrations.

### 12.5 Telemetry Envelope

Telemetry tracks installs, skill sources, and agent selections. The `skills.sh` API provides search functionality. We should plan for similar analytics if we build a registry.

## Observations

- [fact] SKILL.md format requires exactly 2 frontmatter fields: `name` (string) and `description` (string); all other fields are optional #format
- [fact] Vercel uses a two-tier installation: canonical `.agents/skills/` directory with symlinks to agent-specific directories #installation
- [fact] Two lock files exist: global `~/.agents/.skill-lock.json` (not in VCS) and project `skills-lock.json` (designed for VCS) #state-tracking
- [fact] Source resolution supports 7 input patterns: GitHub shorthand, GitHub URL, GitLab URL, git URL, local path, well-known URL, and `@skill` filter syntax #resolution
- [fact] The CLI has zero runtime dependencies; all 6 devDependencies (gray-matter, simple-git, clack, picocolors, xdg-basedir, obuild) are bundled at build time #architecture
- [decision] Our format should borrow SKILL.md frontmatter conventions (name, description, metadata) and extend with type, platforms, dependencies, and config fields #format-design
- [insight] Vercel's provider registry pattern (HostProvider interface) is extensible and should be adopted for our plugin source resolution #architecture
- [insight] The well-known skills protocol (RFC 8615 style) enables organizations to self-host skills without GitHub; we should extend this for full plugin discovery #distribution
- [technique] Name sanitization pattern (lowercase, replace non-alphanumeric, trim edges, 255 char limit) is battle-tested and should be reused #security
- [risk] Vercel's lock file versioning wipes state on incompatible changes; our SQLite approach must support proper migrations to preserve install history #state-tracking

## Relations

- relates_to [[ANALYSIS-001-agent-plugin-foundation-and-vision]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]

## 13. Appendices

### 13.1 Sources Consulted

- Vercel Skills GitHub: <https://github.com/vercel-labs/skills> (src/ directory, all .ts files)
- Vercel Agent Skills: <https://github.com/vercel-labs/agent-skills> (example SKILL.md files)
- skills.sh: <https://skills.sh> (search API referenced in find.ts)
- agentskills.io: referenced in README as specification site

### 13.2 Data Transparency

- **Found**: Complete source code for all core modules (skill-lock.ts, local-lock.ts, source-parser.ts, installer.ts, agents.ts, skills.ts, plugin-manifest.ts, types.ts, constants.ts, providers/). 3 complete SKILL.md examples. Full README documentation. package.json.
- **Not Found**: Complete `add.ts` main flow (partial via install.ts). Telemetry server implementation. skills.sh API documentation. Hugging Face provider implementation (referenced but not in providers/ directory). Total download/install counts.
