---
title: ANALYSIS-039 Cross-Platform MCP and Instruction Formats
type: analysis
permalink: analysis/analysis-039-cross-platform-mcp-and-instruction-formats-1
tags:
- mcp
- instructions
- rules
- cross-platform
- configuration
- research
---

# ANALYSIS-039 Cross-Platform MCP and Instruction Formats

## 1. Objective and Scope

**Objective**: Map the MCP server configuration formats and instruction/rules file systems across all major AI coding assistant platforms to identify commonalities, differences, and convergence patterns.

**Scope**: Claude Code, Cursor, Windsurf, GitHub Copilot (VS Code + Coding Agent + CLI), OpenAI Codex CLI, JetBrains AI Assistant, Zed, Gemini CLI. Covers MCP config structure and instruction/rules files. Excludes runtime behavior and performance analysis.

## 2. Context

The Model Context Protocol (MCP) has become the standard interface for connecting AI coding assistants to external tools. As of March 2026, every major AI coding platform supports MCP. AGENTS.md was contributed to the Linux Foundation's Agentic AI Foundation in December 2025 alongside MCP and goose, signaling industry convergence on open standards.

## 3. Approach

**Methodology**: Web research of official documentation, GitHub repos, and community guides for each platform.
**Tools Used**: WebSearch, WebFetch against official docs sites.
**Limitations**: JetBrains project-level config file naming (.jb-mcp.json) could not be verified from primary docs alone. Some platforms are evolving rapidly and details may shift between minor releases.

## 4. Data and Analysis

---

## Part 1: MCP Server Configuration

### 4.1 Configuration File Locations

| Platform | Project-Level Config | User/Global Config |
|---|---|---|
| **Claude Code** | `.mcp.json` (project root, version-controlled) | `~/.claude.json` (mcpServers field) |
| **Cursor** | `.cursor/mcp.json` | `~/.cursor/mcp.json` |
| **Windsurf** | None (global only) | `~/.codeium/windsurf/mcp_config.json` |
| **VS Code (Copilot)** | `.vscode/mcp.json` | User profile via command palette |
| **GitHub Copilot Coding Agent** | Repository Settings UI on GitHub.com | N/A (cloud-hosted) |
| **GitHub Copilot CLI** | `.copilot/mcp-config.json` | `~/.copilot/mcp-config.json` |
| **OpenAI Codex CLI** | `.codex/config.toml` (trusted projects) | `~/.codex/config.toml` |
| **JetBrains AI Assistant** | Settings UI or project-level file | `~/Library/Application Support/JetBrains/<IDE>/mcp.json` (macOS) |
| **Zed** | `settings.json` (workspace) | `~/.config/zed/settings.json` |
| **Gemini CLI** | `.gemini/settings.json` | `~/.gemini/settings.json` |

### 4.2 JSON Structure Comparison

**The Canonical Format** (used by Claude Code, Cursor, Windsurf, JetBrains, GitHub Copilot CLI):

```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "your-token"
      }
    }
  }
}
```

**VS Code diverges** with a different top-level key and additional features:

```json
{
  "inputs": [
    {
      "type": "promptString",
      "id": "api-key",
      "description": "API Key",
      "password": true
    }
  ],
  "servers": {
    "server-name": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@example/mcp-server"],
      "env": {
        "API_KEY": "${input:api-key}"
      },
      "sandboxEnabled": true,
      "sandbox": {
        "filesystem": {
          "allowWrite": ["${workspaceFolder}"]
        },
        "network": {
          "allowedDomains": ["api.example.com"]
        }
      }
    }
  }
}
```

Key VS Code differences:
- Top-level key is `servers` not `mcpServers`
- `inputs` array for secure credential prompting with `${input:id}` substitution
- `sandboxEnabled` and `sandbox` for filesystem/network isolation (macOS/Linux)
- `envFile` field for loading env from a file
- `dev` field for watch mode and debugging

**OpenAI Codex CLI uses TOML** instead of JSON:

```toml
[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]

[mcp_servers.github.env]
GITHUB_TOKEN = "your-token"
```

Codex-specific TOML fields:
- `startup_timeout_sec` (default: 10)
- `tool_timeout_sec` (default: 60)
- `enabled` (boolean, disable without removing)
- `required` (boolean, fail startup if init fails)
- `enabled_tools` (allow list)
- `disabled_tools` (deny list)
- `bearer_token_env_var` (for HTTP auth)
- `env_http_headers` (headers from env vars)
- `cwd` (working directory)

**Zed uses a different key** in settings.json:

```json
{
  "context_servers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "@example/server"],
      "env": {}
    }
  }
}
```

Note: Zed is moving toward the standardized `mcpServers` format (PR #33539).

**Gemini CLI uses `mcpServers`** in settings.json but adds unique fields:

```json
{
  "mcpServers": {
    "github": {
      "httpUrl": "https://api.githubcopilot.com/mcp/",
      "headers": {
        "Authorization": "Bearer $GITHUB_TOKEN"
      },
      "timeout": 5000,
      "trust": true,
      "description": "GitHub MCP server"
    }
  }
}
```

Gemini-specific fields: `httpUrl` (distinct from `url` for SSE), `timeout`, `trust`, `description`.

### 4.3 Concrete Cross-Platform Example: SQLite MCP Server

**Claude Code (.mcp.json)**:
```json
{
  "mcpServers": {
    "sqlite": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sqlite", "/path/to/db.sqlite"],
      "env": {}
    }
  }
}
```

**Cursor (.cursor/mcp.json)**:
```json
{
  "mcpServers": {
    "sqlite": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sqlite", "/path/to/db.sqlite"],
      "env": {}
    }
  }
}
```

**Windsurf (~/.codeium/windsurf/mcp_config.json)**:
```json
{
  "mcpServers": {
    "sqlite": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sqlite", "/path/to/db.sqlite"],
      "env": {}
    }
  }
}
```

**VS Code (.vscode/mcp.json)**:
```json
{
  "servers": {
    "sqlite": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sqlite", "/path/to/db.sqlite"]
    }
  }
}
```

**OpenAI Codex CLI (.codex/config.toml)**:
```toml
[mcp_servers.sqlite]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-sqlite", "/path/to/db.sqlite"]
```

**JetBrains (via Settings UI or mcp.json)**:
```json
{
  "mcpServers": {
    "sqlite": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sqlite", "/path/to/db.sqlite"]
    }
  }
}
```

**Zed (settings.json)**:
```json
{
  "context_servers": {
    "sqlite": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sqlite", "/path/to/db.sqlite"],
      "env": {}
    }
  }
}
```

**Gemini CLI (.gemini/settings.json)**:
```json
{
  "mcpServers": {
    "sqlite": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-sqlite", "/path/to/db.sqlite"]
    }
  }
}
```

### 4.4 Transport Types Comparison

| Platform | stdio | SSE | HTTP/Streamable HTTP | Notes |
|---|---|---|---|---|
| Claude Code | Yes | Yes (deprecated) | Yes (recommended) | OAuth 2.0 for HTTP |
| Cursor | Yes | Yes | Yes | OAuth support |
| Windsurf | Yes | Yes | Yes (`serverUrl`) | Uses `serverUrl` not `url` |
| VS Code | Yes | Yes | Yes | `type` field required |
| Copilot Coding Agent | Yes (`local`/`stdio`) | Yes | Yes | `type` field required |
| Codex CLI | Yes | N/A | Yes (`url`) | Distinct `url` for HTTP |
| JetBrains | Yes | Yes (legacy) | Yes (`url`) | Streamable HTTP preferred |
| Zed | Yes | N/A | Yes (`url`) | Transitioning format |
| Gemini CLI | Yes | Yes (`url`) | Yes (`httpUrl`) | Separate fields for SSE vs HTTP |

### 4.5 HTTP/Remote Server Configuration Differences

**Claude Code**:
```json
{ "type": "http", "url": "https://example.com/mcp", "headers": { "Authorization": "Bearer token" } }
```

**VS Code**:
```json
{ "type": "http", "url": "https://example.com/mcp", "headers": { "Authorization": "Bearer ${input:token}" } }
```

**Windsurf**:
```json
{ "serverUrl": "https://example.com/mcp", "headers": { "API_KEY": "value" } }
```

**Codex CLI (TOML)**:
```toml
url = "https://example.com/mcp"
bearer_token_env_var = "MY_TOKEN"
http_headers = { "X-Custom" = "value" }
```

**Gemini CLI**:
```json
{ "httpUrl": "https://example.com/mcp", "headers": { "Authorization": "Bearer $TOKEN" }, "timeout": 5000 }
```

### 4.6 Environment Variable Handling

| Platform | Env Syntax | Variable Expansion |
|---|---|---|
| Claude Code | `"env": { "KEY": "value" }`, `${VAR}` in .mcp.json | `${VAR}`, `${VAR:-default}` in command, args, env, url, headers |
| Cursor | `"env": { "KEY": "value" }` | Standard env block |
| Windsurf | `"env": { "KEY": "value" }` | `${env:VAR}` interpolation in all fields |
| VS Code | `"env": { "KEY": "${input:id}" }` | `${input:id}` for prompted values, `${workspaceFolder}` predefined vars |
| Codex CLI | `[mcp_servers.name.env]` | `bearer_token_env_var`, `env_http_headers`, `env_vars` |
| JetBrains | `"env": { "KEY": "value" }` | Standard env block |
| Zed | `"env": {}` | Standard env block |
| Gemini CLI | `"env": { "KEY": "$SHELL_VAR" }` | Shell-style `$VAR` expansion at runtime |

### 4.7 Key MCP Config Answers

**Q1: Is the MCP server config structure identical across platforms?**
No, but it is very close. 6 of 8 platforms use the `mcpServers` wrapper key with identical inner structure (`command`, `args`, `env`). VS Code uses `servers` instead. Zed uses `context_servers` (migrating to `mcpServers`). Codex CLI uses TOML instead of JSON.

**Q2: What are the actual differences?**
- **Wrapper key**: `mcpServers` (6 platforms) vs `servers` (VS Code) vs `context_servers` (Zed)
- **File format**: JSON (7 platforms) vs TOML (Codex CLI)
- **File location**: Varies significantly per platform
- **Type field**: VS Code and Copilot Coding Agent require explicit `type` field; others infer from presence of `command` vs `url`
- **Remote URL field**: `url` (most), `serverUrl` (Windsurf), `httpUrl` (Gemini CLI for streamable HTTP)
- **Platform-specific fields**: VS Code has `sandbox`, `inputs`, `envFile`, `dev`; Codex has timeouts, tool filtering, `required`; Gemini has `trust`, `description`; Copilot Agent has `tools` array
- **Env var expansion syntax**: `${VAR}` (Claude), `${input:id}` (VS Code), `${env:VAR}` (Windsurf), `$VAR` (Gemini)

**Q3: Is there a canonical MCP config format?**
The `mcpServers` JSON format with `command`/`args`/`env` for stdio and `url`/`headers` for HTTP is the de facto standard. It originated from Claude Desktop and has been adopted by 6+ platforms. VS Code's `servers` key is the notable outlier but shares the same inner structure.

**Q4: How does each platform discover MCP servers?**
- **Claude Code**: `.mcp.json` (project), `~/.claude.json` (user/local), managed-mcp.json (org). Precedence: local > project > user.
- **Cursor**: `.cursor/mcp.json` (project), `~/.cursor/mcp.json` (global).
- **Windsurf**: `~/.codeium/windsurf/mcp_config.json` only (no project-level).
- **VS Code**: `.vscode/mcp.json` (workspace), user profile, devcontainer.json.
- **Copilot Coding Agent**: GitHub.com repository settings UI.
- **Copilot CLI**: `.copilot/mcp-config.json` (project), `~/.copilot/mcp-config.json` (global).
- **Codex CLI**: `.codex/config.toml` (project, trusted), `~/.codex/config.toml` (global).
- **JetBrains**: IDE Settings UI, global config in IDE config directory.
- **Zed**: `settings.json` (workspace and user).
- **Gemini CLI**: `.gemini/settings.json` (project), `~/.gemini/settings.json` (global).

---

## Part 2: Instructions / Rules Files

### 4.8 Platform-by-Platform Instruction Format

#### Claude Code: CLAUDE.md

**Files and locations**:
| Scope | Location | Shared |
|---|---|---|
| Managed policy | `/Library/Application Support/ClaudeCode/CLAUDE.md` (macOS) | All users |
| Project | `./CLAUDE.md` or `./.claude/CLAUDE.md` | Team (VCS) |
| User | `~/.claude/CLAUDE.md` | All projects |
| Local | `./CLAUDE.local.md` | Just you |

**Format**: Plain Markdown. No required frontmatter. No required structure.

**Rules directory**: `.claude/rules/*.md` files with optional YAML frontmatter:
```yaml
---
paths:
  - "src/api/**/*.ts"
---
```
Rules without `paths` frontmatter load unconditionally. Path-scoped rules load when Claude reads matching files.

**Loading behavior**: Walks up directory tree from CWD. All ancestor CLAUDE.md files load at launch. Subdirectory CLAUDE.md files load on demand. Precedence: local > project > user. Imports via `@path/to/file` syntax (max 5 hops deep).

**Size recommendation**: Under 200 lines per file. First 200 lines of auto-memory MEMORY.md loaded at startup.

**Key features**: `/init` command generates initial file. Auto-memory system (Claude writes its own notes). `claudeMdExcludes` setting to skip irrelevant files in monorepos. User-level rules in `~/.claude/rules/`.

#### Cursor: .cursor/rules/

**Files and locations**:
- `.cursor/rules/*.md` or `.cursor/rules/*.mdc` (project, version-controlled)
- User rules via Settings UI

**Format**: Markdown with YAML frontmatter:
```yaml
---
description: "Service definition patterns"
globs: ["src/services/**"]
alwaysApply: false
---

- Use internal RPC pattern for services
- Employ snake_case for service naming
```

**Rule types** (set via frontmatter):
| Type | Frontmatter | Behavior |
|---|---|---|
| Always Apply | `alwaysApply: true` | Active in every session |
| Apply Intelligently | `description` set, no globs | Agent decides relevance |
| Apply to Specific Files | `globs` set | Triggered on file match |
| Apply Manually | No frontmatter flags | Invoked via `@rule-name` |

**Legacy format**: `.cursorrules` file in project root (still functional but deprecated as of v2.2).

**Precedence**: Team Rules > Project Rules > User Rules.

**Also supports**: `AGENTS.md` files.

#### Windsurf: .windsurf/rules/

**Files and locations**:
| Scope | Location | Limit |
|---|---|---|
| Global | `~/.codeium/windsurf/memories/global_rules.md` | 6,000 chars |
| Workspace | `.windsurf/rules/*.md` | 12,000 chars per file |
| System (Enterprise) | `/Library/Application Support/Windsurf/rules/*.md` (macOS) | Read-only |
| Legacy | `.windsurfrules` (project root) | Deprecated |

**Format**: Markdown with YAML frontmatter for workspace rules:
```yaml
---
trigger: glob
globs: **/*.test.ts
---
```

**Activation modes**:
| Mode | Trigger Value | Behavior |
|---|---|---|
| Always On | `always_on` | Included in every prompt |
| Model Decision | `model_decision` | Description shown; full content loaded when relevant |
| Glob | `glob` | Activated on matching files |
| Manual | `manual` | Requires `@rule-name` mention |

**Also supports**: `AGENTS.md` files.

**Key constraints**: 100 total tool limit. Combined global + local rules must not exceed 12,000 characters.

#### GitHub Copilot: copilot-instructions.md

**Files and locations**:
- `.github/copilot-instructions.md` (repository-wide)
- `.github/instructions/*.instructions.md` (path-specific)
- `AGENTS.md` / `CLAUDE.md` / `GEMINI.md` (agent instructions, experimental in VS Code)

**Format**: Markdown with optional YAML frontmatter for path-specific files:
```yaml
---
applyTo: "**/*.py"
excludeAgent: "code-review"
---
```

**Frontmatter fields**:
- `applyTo` (glob pattern for file targeting)
- `excludeAgent` (`"code-review"` or `"coding-agent"`)
- `name` (display name, optional, VS Code only)
- `description` (tooltip, optional, VS Code only)

**Precedence**: Personal > Repository > Organization.

**Platform support varies**:
| Platform | Repo-wide | Path-specific | Agent files |
|---|---|---|---|
| GitHub.com | Yes | Yes | Yes |
| VS Code | Yes | Yes | Yes |
| Visual Studio | Yes | Yes | No |
| JetBrains | Yes | No | No |
| Xcode/Eclipse | Yes | No | No |

**VS Code specific**: `chat.instructionsFilesLocations` setting controls which directories are scanned. Supports `.claude/rules` files (using `paths` instead of `applyTo`). `/init` command generates initial file.

#### OpenAI Codex CLI: AGENTS.md

**Files and locations**:
- `~/.codex/AGENTS.md` or `~/.codex/AGENTS.override.md` (global)
- `AGENTS.md` or `AGENTS.override.md` (any directory in project)

**Format**: Plain Markdown. No frontmatter. No required structure.

**Discovery**:
1. Global: checks `AGENTS.override.md` first, then `AGENTS.md` in `~/.codex/`
2. Project: walks directory tree from git root to CWD
3. At each directory: `AGENTS.override.md` > `AGENTS.md` > fallback names
4. One file per directory, closest file takes precedence

**Fallback filenames** (configurable in config.toml):
```toml
project_doc_fallback_filenames = ["TEAM_GUIDE.md", ".agents.md"]
```

**Size limit**: 32 KiB default (`project_doc_max_bytes`).

**Injection**: Each file becomes a user-role message starting with `# AGENTS.md instructions for <directory>`. Root-to-leaf order.

**Rules system**: Separate `.rules` files in `.codex/rules/` using Starlark syntax for execution policy (allow/prompt/forbidden). This is for command approval, not instruction formatting.

#### Gemini CLI: GEMINI.md

**Files and locations**:
- `~/.gemini/GEMINI.md` (global)
- `GEMINI.md` (project root and subdirectories)

**Format**: Plain Markdown. No frontmatter.

**Loading**: Three-tier hierarchy:
1. Global (`~/.gemini/GEMINI.md`)
2. Workspace (project root and ancestors)
3. Just-in-Time (scanned when tools access directories)

All discovered files are concatenated and sent with each prompt.

**Import syntax**: `@file.md` for relative/absolute imports.

**Customizable context filename** in `.gemini/settings.json`:
```json
{
  "context": {
    "fileName": ["AGENTS.md", "CONTEXT.md", "GEMINI.md"]
  }
}
```

**Also supports**: AGENTS.md files (configurable as context filename).

**Commands**: `/memory show`, `/memory refresh`, `/memory add <text>`, `/init`.

#### Zed

Zed reads `AGENTS.md` files. No Zed-specific instruction file format documented.

### 4.9 Cross-Platform Instruction Format Comparison

| Feature | Claude Code | Cursor | Windsurf | GitHub Copilot | Codex CLI | Gemini CLI |
|---|---|---|---|---|---|---|
| **Primary file** | CLAUDE.md | .cursor/rules/*.md | .windsurf/rules/*.md | copilot-instructions.md | AGENTS.md | GEMINI.md |
| **Format** | Markdown | Markdown + YAML FM | Markdown + YAML FM | Markdown + YAML FM | Plain Markdown | Plain Markdown |
| **Frontmatter** | paths (rules only) | description, globs, alwaysApply | trigger, globs | applyTo, excludeAgent | None | None |
| **File scoping** | paths glob | globs array | glob trigger | applyTo glob | Directory nesting | Directory nesting |
| **Rule types** | Always + path-scoped | 4 types (always, auto, glob, manual) | 4 modes (always_on, model_decision, glob, manual) | Always + path-specific | Always (position-based) | Always (concatenated) |
| **Project scope** | ./CLAUDE.md | .cursor/rules/ | .windsurf/rules/ | .github/copilot-instructions.md | ./AGENTS.md | ./GEMINI.md |
| **User scope** | ~/.claude/CLAUDE.md | Settings UI | ~/.codeium/windsurf/memories/global_rules.md | Personal settings | ~/.codex/AGENTS.md | ~/.gemini/GEMINI.md |
| **Org/managed** | /Library/.../CLAUDE.md | Team rules | Enterprise rules | Org instructions | N/A | N/A |
| **Directory walking** | Yes (ancestors + subdirs) | No | No | No | Yes (root to CWD) | Yes (3-tier) |
| **Import syntax** | @path/to/file | @file-reference | N/A | N/A | N/A | @file.md |
| **Size limits** | ~200 lines recommended | N/A | 6K-12K chars | N/A | 32 KiB | N/A |
| **Auto-generation** | /init | N/A | N/A | /init | N/A | /init |
| **AGENTS.md support** | No (uses CLAUDE.md) | Yes | Yes | Yes | Yes (primary) | Yes (configurable) |

### 4.10 AGENTS.md as Emerging Standard

AGENTS.md was released by OpenAI in August 2025 and contributed to the Linux Foundation's Agentic AI Foundation in December 2025. As of March 2026:

**Platforms that read AGENTS.md**:
- OpenAI Codex CLI (primary format)
- GitHub Copilot (VS Code, Coding Agent)
- Cursor (alongside .cursor/rules/)
- Windsurf (alongside .windsurf/rules/)
- Gemini CLI (configurable filename)
- Zed
- 20+ other tools (Aider, Warp, Devin, Factory, goose, etc.)

**Platforms that do NOT read AGENTS.md**:
- Claude Code (uses CLAUDE.md exclusively)
- JetBrains AI Assistant (uses .github/copilot-instructions.md)

**Key characteristic**: AGENTS.md is intentionally minimal. Plain markdown, no frontmatter, no special syntax. This simplicity drives adoption but limits per-file scoping and conditional activation.

### 4.11 Cross-Platform Analysis Answers

**Q1: What is the common core of instruction files?**
All instruction files are Markdown containing natural language guidance for AI. The common content includes: project overview, build/test commands, coding standards, architecture notes, and workflow instructions. Every platform treats these as context injected into the AI's prompt.

**Q2: What platform-specific metadata exists?**
- Cursor: `description`, `globs`, `alwaysApply` frontmatter
- Windsurf: `trigger`, `globs` frontmatter
- GitHub Copilot: `applyTo`, `excludeAgent` frontmatter
- Claude Code: `paths` frontmatter (rules only)
- Codex/Gemini/AGENTS.md: No metadata

**Q3: Could a single canonical instruction be transformed into each format?**
Yes, with limitations. A core Markdown instruction file could be:
1. Used directly as AGENTS.md / GEMINI.md (plain markdown)
2. Wrapped with frontmatter for Cursor (`globs`, `alwaysApply`)
3. Wrapped with frontmatter for Windsurf (`trigger`, `globs`)
4. Wrapped with frontmatter for Copilot (`applyTo`)
5. Used directly as CLAUDE.md or placed in `.claude/rules/` with `paths`

The transformation is straightforward: add platform-specific YAML frontmatter to the same Markdown body. File-scoping is the main divergence point.

**Q4: Is there convergence happening?**
Yes, significant convergence:
- AGENTS.md adoption across 60,000+ repos and 20+ tools
- `mcpServers` JSON format used by 6+ platforms for MCP config
- All platforms use Markdown for instructions
- Glob-based file scoping appearing in 4 platforms (Cursor, Windsurf, Copilot, Claude Code)
- VS Code now reads Claude Code's `.claude/rules/` files
- Gemini CLI can be configured to read AGENTS.md as its primary file
- Zed migrating to standardized `mcpServers` key

## 5. Results

### MCP Configuration Findings

- 8 of 8 platforms share the same core fields for stdio servers: `command`, `args`, `env`
- 6 of 8 platforms use the `mcpServers` JSON wrapper key
- 7 of 8 platforms use JSON; 1 uses TOML (Codex CLI)
- The stdio server config block is copy-pasteable between Claude Code, Cursor, Windsurf, JetBrains, Copilot CLI, and Gemini CLI without modification
- Remote/HTTP config diverges more: different URL field names, auth mechanisms, and header syntax

### Instruction File Findings

- All 8 platforms use Markdown as the base format
- 4 platforms support YAML frontmatter for conditional activation
- 6 platforms support AGENTS.md
- 3 platforms have their own named files (CLAUDE.md, GEMINI.md, .cursorrules)
- Directory-walking discovery is used by 3 platforms (Claude Code, Codex CLI, Gemini CLI)

## 6. Discussion

**MCP config has effectively standardized.** The `mcpServers` + `command`/`args`/`env` pattern is the de facto standard. VS Code's `servers` key and Codex's TOML format are the outliers, but the inner structure is identical. A tool that generates MCP configs could target 6 platforms with one JSON template plus simple transformations for VS Code (rename key) and Codex (convert to TOML).

**Instruction files are converging but not yet standardized.** AGENTS.md is the closest to a cross-platform standard, adopted by 60,000+ repos. However, it lacks the conditional activation features (glob-based scoping, AI-decided relevance) that platform-specific formats offer. The practical approach for projects is to maintain both an AGENTS.md (for broad compatibility) and platform-specific rule files (for advanced features).

**VS Code is becoming a universal reader.** VS Code now reads `.github/copilot-instructions.md`, `.github/instructions/*.instructions.md`, `AGENTS.md`, `CLAUDE.md`, and `.claude/rules/` files. This positions VS Code as the most format-agnostic platform.

**The frontier is conditional activation.** The most sophisticated feature across instruction formats is conditional rules: Cursor's 4-type system, Windsurf's activation modes, and Copilot's `applyTo` targeting. AGENTS.md's simplicity (no metadata) is both its strength (adoption) and weakness (no per-file scoping).

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | For any plugin targeting cross-platform MCP config, use `mcpServers` JSON as the canonical format | 6 of 8 platforms use it directly | Low |
| P0 | Include an AGENTS.md in all repositories | Read by 20+ tools, zero-overhead format | Low |
| P1 | Generate VS Code format by renaming `mcpServers` to `servers` and adding `type` field | Simple transformation covers the main outlier | Low |
| P1 | For instruction files targeting multiple platforms, maintain a core Markdown body with platform-specific frontmatter wrappers | Maximum reuse with platform-specific features | Medium |
| P2 | Generate Codex TOML from the canonical JSON MCP config | JSON-to-TOML conversion is mechanical | Low |
| P2 | Monitor Zed's migration to `mcpServers` format (PR #33539) | Will reduce outlier platforms to 2 (VS Code, Codex) | None |

## 8. Conclusion

**Verdict**: Proceed with cross-platform tooling using `mcpServers` JSON as canonical MCP format and Markdown as canonical instruction format.

**Confidence**: High

**Rationale**: MCP config has effectively standardized around the `mcpServers` JSON pattern with minor per-platform variations. Instruction files are converging via AGENTS.md adoption and VS Code's multi-format reader support. The differences are mechanical transformations, not fundamental incompatibilities.

### User Impact

- **What changes for you**: A single MCP config can target 6+ platforms with zero or minimal transformation. A single instruction file body can serve all platforms with frontmatter wrappers.
- **Effort required**: Low for MCP configs (copy-paste works for 6 platforms). Medium for instruction files (need frontmatter per platform for advanced features).
- **Risk if ignored**: Maintaining separate configs per platform increases drift risk and maintenance burden as the ecosystem continues to consolidate.

## Observations

- [fact] 6 of 8 platforms use `mcpServers` JSON wrapper key for MCP configuration #mcp #cross-platform
- [fact] All 8 platforms share identical inner structure for stdio MCP servers: command, args, env #mcp #standardization
- [fact] AGENTS.md adopted by 60,000+ repos and 20+ tools as of March 2026 #agents-md #adoption
- [fact] VS Code uses `servers` not `mcpServers` as top-level key, diverging from the de facto standard #vscode #mcp
- [fact] OpenAI Codex CLI is the only platform using TOML instead of JSON for MCP configuration #codex #toml
- [decision] The mcpServers JSON format is the de facto canonical MCP config format #mcp #standardization
- [insight] Instruction file convergence is happening through AGENTS.md adoption and VS Code multi-format reading #instructions #convergence
- [insight] Conditional activation (glob-based scoping) is the main differentiation between platform instruction formats #rules #differentiation
- [fact] AGENTS.md was contributed to Linux Foundation Agentic AI Foundation in December 2025 alongside MCP #agents-md #governance
- [fact] Claude Code is the only major platform that does not read AGENTS.md files #claude-code #agents-md

## Relations

- relates_to [[ADR-007 Memory-First Architecture]]
- relates_to [[SESSION-2026-03-07_01 Agent Plugin Spec Ideation]]

## 9. Appendices

### Sources Consulted

- Claude Code MCP Docs: https://code.claude.com/docs/en/mcp
- Claude Code Memory Docs: https://code.claude.com/docs/en/memory
- Cursor Rules Docs: https://cursor.com/docs/context/rules
- Cursor MCP Docs: https://cursor.com/docs/context/mcp
- Windsurf MCP Docs: https://docs.windsurf.com/windsurf/cascade/mcp
- Windsurf Memories Docs: https://docs.windsurf.com/windsurf/cascade/memories
- VS Code MCP Docs: https://code.visualstudio.com/docs/copilot/customization/mcp-servers
- VS Code MCP Reference: https://code.visualstudio.com/docs/copilot/reference/mcp-configuration
- VS Code Custom Instructions: https://code.visualstudio.com/docs/copilot/customization/custom-instructions
- GitHub Copilot Instructions: https://docs.github.com/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot
- GitHub Copilot Coding Agent MCP: https://docs.github.com/copilot/how-tos/agents/copilot-coding-agent/extending-copilot-coding-agent-with-mcp
- OpenAI Codex MCP Docs: https://developers.openai.com/codex/mcp/
- OpenAI Codex Config Reference: https://developers.openai.com/codex/config-reference
- OpenAI Codex AGENTS.md Guide: https://developers.openai.com/codex/guides/agents-md/
- OpenAI Codex Rules: https://developers.openai.com/codex/rules/
- JetBrains MCP Docs: https://www.jetbrains.com/help/ai-assistant/mcp.html
- JetBrains MCP Config: https://www.jetbrains.com/help/ai-assistant/configure-an-mcp-server.html
- Zed MCP Docs: https://zed.dev/docs/ai/mcp
- Gemini CLI Docs: https://geminicli.com/docs/
- Gemini CLI GEMINI.md: https://geminicli.com/docs/cli/gemini-md/
- Gemini CLI MCP: https://geminicli.com/docs/tools/mcp-server/
- AGENTS.md Official Site: https://agents.md/
- Linux Foundation AAIF Announcement: https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation
- InfoQ AGENTS.md Article: https://www.infoq.com/news/2025/08/agents-md/

### Data Transparency

- **Found**: Complete MCP config schemas for all 8 platforms. Complete instruction file formats for all platforms. AGENTS.md adoption data and governance structure.
- **Not Found**: Exact JetBrains project-level config file name (`.jb-mcp.json` mentioned in one source but not confirmed in official docs). Zed instruction file format (appears to only use AGENTS.md). Exact character/token limits for Cursor and Copilot instruction files. Claude Code's internal AGENTS.md reading behavior (confirmed it does NOT read AGENTS.md).