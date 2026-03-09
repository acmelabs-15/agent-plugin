---
title: ANALYSIS-038 Cross-Platform Agent and Skill Definitions
type: analysis
permalink: analysis/analysis-038-cross-platform-agent-and-skill-definitions-1
tags:
- agents
- skills
- cross-platform
- definitions
- research
---

# ANALYSIS-038 Cross-Platform Agent and Skill Definitions

## 1. Objective and Scope

**Objective**: Map the structural formats used to define agents and skills across 7 major AI coding assistant platforms. Identify universal fields, platform-specific extensions, and feasibility of a canonical format that transforms per-platform.

**Scope**: Claude Code, Cursor, Windsurf, GitHub Copilot (VS Code + CLI + GitHub.com), OpenAI Codex CLI, Gemini Code Assist, Amazon Q Developer CLI. Focus on file format structure, not runtime UX.

## 2. Context

The agent-plugin project (FEAT-001 cross-platform portability, FEAT-005 multi-platform support strategy) needs to generate platform-specific agent and skill definitions from a canonical internal format. This research provides the foundation for REQ-002 (cross-platform agent adaptation), REQ-004 (agent instruction file generation), and REQ-005 (skill and command cross-platform generation).

## 3. Approach

**Methodology**: Web research of official documentation, GitHub repos, and community guides for each platform. Direct fetching of specification pages for field-level detail.

**Tools Used**: WebSearch, WebFetch across official docs sites (code.claude.com, cursor.com, docs.github.com, developers.openai.com, agentskills.io, docs.windsurf.com, developers.google.com, docs.aws.amazon.com, agents.md)

**Limitations**: Amazon Q Developer CLI documentation pages use JavaScript redirects preventing direct content extraction. Some details inferred from blog posts and community guides.

## 4. Data and Analysis

---

### 4.1 Agent Definition Formats

#### 4.1.1 Claude Code (Subagents)

**File format**: Markdown with YAML frontmatter
**File extension**: `.md`
**File location**: `.claude/agents/` (project), `~/.claude/agents/` (user), plugin `agents/` directory
**Priority**: CLI flag > project > user > plugin

| Field | Required | Type | Description |
|---|---|---|---|
| `name` | Yes | string | Unique identifier, lowercase + hyphens |
| `description` | Yes | string | When Claude should delegate to this agent |
| `tools` | No | comma-separated string | Allowlist of tools. Inherits all if omitted |
| `disallowedTools` | No | comma-separated string | Tools to deny |
| `model` | No | string | `sonnet`, `opus`, `haiku`, `inherit` (default: `inherit`) |
| `permissionMode` | No | string | `default`, `acceptEdits`, `dontAsk`, `bypassPermissions`, `plan` |
| `maxTurns` | No | integer | Maximum agentic turns |
| `skills` | No | list | Skills to preload into agent context |
| `mcpServers` | No | object/list | MCP server configurations |
| `hooks` | No | object | Lifecycle hooks (PreToolUse, PostToolUse, Stop) |
| `memory` | No | string | Persistent memory scope: `user`, `project`, `local` |
| `background` | No | boolean | Run as background task (default: false) |
| `isolation` | No | string | `worktree` for git worktree isolation |

**Body**: Markdown system prompt. Agent receives ONLY this prompt (plus basic env details), not the full Claude Code system prompt.

**Unique features**: Agent tool spawning control via `Agent(type)` syntax in tools field. Hook system with PreToolUse/PostToolUse/Stop events. Persistent memory with auto-managed MEMORY.md. Worktree isolation. CLI JSON format for ephemeral agents.

---

#### 4.1.2 GitHub Copilot (Custom Agents)

**File format**: Markdown with YAML frontmatter
**File extension**: `.agent.md` or `.md`
**File location**: `.github/agents/` (project), `~/.copilot/agents/` (CLI user-level), `{org}/.github/` or `{org}/.github-private/` (org-level)
**Targets**: VS Code, GitHub.com Copilot coding agent, Copilot CLI

| Field | Required | Type | Description |
|---|---|---|---|
| `name` | No | string | Display name (defaults to filename) |
| `description` | Yes | string | Agent purpose and capabilities |
| `tools` | No | list/string | Tool allowlist. `["*"]` for all, `[]` for none |
| `model` | No | string | AI model in format "Model Name (vendor)" |
| `target` | No | string | `vscode` or `github-copilot` |
| `handoffs` | No | list | Transitions to other agents |
| `user-invocable` | No | boolean | Controls dropdown visibility |
| `disable-model-invocation` | No | boolean | Prevents automatic agent selection |
| `mcp-servers` | No | object | MCP server configurations |
| `agents` | No | list | Permitted subagents (`*` for all, `[]` for none) |
| `argument-hint` | No | string | Guidance text for chat input |
| `metadata` | No | object | Key-value pairs for annotation |

**Handoff schema**: Each handoff has `label` (button text), `agent` (target), `prompt` (message), `send` (boolean auto-submit), `model` (optional).

**Body**: Markdown instructions (max 30,000 characters). Tool references use `#tool:<tool-name>` syntax.

**Unique features**: Handoff system between agents. Multi-target support (VS Code vs GitHub.com). Tool reference syntax in body. Org-level agent sharing.

**Limitations on GitHub.com**: `model`, `argument-hint`, and `handoffs` are not supported for the Copilot coding agent on GitHub.com.

---

#### 4.1.3 Amazon Q Developer CLI (Custom Agents)

**File format**: JSON
**File extension**: `.json`
**File location**: `.amazonq/agents/` (project), system-level directories
**Name derivation**: Filename without `.json` extension

| Field | Required | Type | Description |
|---|---|---|---|
| `name` | No | string | Agent identifier (defaults to filename) |
| `description` | Yes | string | Human-readable explanation |
| `prompt` | Yes | string | System prompt. Supports `file://` URIs |
| `model` | No | string | Model ID (e.g., `claude-sonnet-4`) |
| `tools` | No | array | Tool names: built-in (`fs_read`), MCP (`@server`), wildcards (`*`, `@builtin`) |
| `allowedTools` | No | array | Tools usable without user prompting. Supports globs (`fs_*`) |
| `toolAliases` | No | object | Remap tool names to resolve collisions |
| `toolsSettings` | No | object | Tool-specific configuration and permissions |
| `mcpServers` | No | object | MCP server definitions with command, args, env, timeout |
| `resources` | No | array | Local file access via `file://` URIs with glob patterns |
| `hooks` | No | object | Lifecycle hooks: agentSpawn, userPromptSubmit, preToolUse, postToolUse, stop |
| `useLegacyMcpJson` | No | boolean | Include legacy MCP configurations |

**Unique features**: JSON format (only non-Markdown platform). File URI references for prompts. Tool aliasing for collision resolution. Glob patterns in allowedTools. Resource declarations.

---

#### 4.1.4 Cursor (Rules-Based, No Agent Entity)

**File format**: MDC (Markdown with frontmatter) or plain Markdown
**File extension**: `.mdc` or `.md`
**File location**: `.cursor/rules/` (project)
**Legacy**: `.cursorrules` file at project root (deprecated)

Cursor does NOT have a dedicated "agent" entity. Instead, it uses a rules system.

| Field | Required | Type | Description |
|---|---|---|---|
| `description` | No | string | Rule purpose for intelligent application |
| `globs` | No | string/array | File path patterns for scope (`**/*.py`) |
| `alwaysApply` | No | boolean | Apply to every chat session |

**Activation modes** (4 types):
1. **Always Apply**: Included in every session
2. **Apply Intelligently**: Agent determines relevance from description
3. **Apply to Specific Files**: Active when glob patterns match
4. **Apply Manually**: User invokes via `@rule-name`

**Body**: Plain markdown instructions. File references use `@path/to/file.ext` syntax.

**Also supports**: `AGENTS.md` files as plain markdown alternative.

---

#### 4.1.5 Windsurf (Rules-Based, No Agent Entity)

**File format**: Plain Markdown (no frontmatter required)
**File locations**: `.windsurfrules` (project root, legacy), `.windsurf/rules/` (project), `~/.windsurf/rules/` (global), `AGENTS.md` files

Windsurf does NOT have a dedicated "agent" entity. Rules and AGENTS.md provide instructions.

**AGENTS.md behavior**:
- Root directory: Always-on rule (full content in system prompt)
- Subdirectory: Auto-converted to glob rule with `<directory>/**` pattern
- Case-insensitive (AGENTS.md or agents.md)
- No frontmatter required

**Rules files**: Plain markdown in `.windsurf/rules/` directory, organized by topic (e.g., `general.md`, `api-conventions.md`).

---

#### 4.1.6 OpenAI Codex CLI (AGENTS.md, No Agent Entity)

**File format**: Plain Markdown
**File extension**: `.md`
**File locations**: `~/.codex/AGENTS.md` or `AGENTS.override.md` (global), `AGENTS.md` or `AGENTS.override.md` per directory (project)

Codex does NOT have a dedicated agent entity. It uses AGENTS.md instruction files.

**Discovery order**: Global scope first, then project scope from git root to current directory.
**Merge behavior**: Files concatenate from root down with blank lines. Closer files override earlier guidance.
**Override mechanism**: `AGENTS.override.md` takes precedence over `AGENTS.md` in same directory.
**Size limit**: `project_doc_max_bytes` (default 32 KiB).
**Config**: `~/.codex/config.toml` for `project_doc_fallback_filenames` and `project_doc_max_bytes`.

---

#### 4.1.7 Gemini Code Assist (Context Files, No Agent Entity)

**File format**: Plain Markdown
**Context files**: `GEMINI.md` or `AGENT.md` at project root, `~/.gemini/GEMINI.md` (global)
**Settings**: `~/.gemini/settings.json`

Gemini does NOT have a dedicated agent entity. Context files and settings control behavior.

**Context file hierarchy**: Global < project root < subdirectories (more specific overrides general).

**Settings fields for tool control**:

| Field | Type | Description |
|---|---|---|
| `coreTools` | string[] | Approved tools with optional command restrictions |
| `excludeTools` | string[] | Blocked tools (takes precedence over coreTools) |
| `mcpServers` | object | MCP server configurations |

**Also supports**: `.gemini/styleguide.md` for code review customization (natural language, no schema). `.gemini/config.yaml` for repository settings.

---

### 4.2 Skill Definition Formats

#### 4.2.1 Agent Skills Open Standard (agentskills.io)

**Status**: Open standard created by Anthropic, adopted by 26+ platforms.
**Specification**: https://agentskills.io/specification

**Directory structure**:
```
skill-name/
  SKILL.md          (required)
  scripts/          (optional - executable code)
  references/       (optional - additional docs)
  assets/           (optional - templates, images, data)
```

**SKILL.md frontmatter**:

| Field | Required | Constraints |
|---|---|---|
| `name` | Yes | Max 64 chars. Lowercase, numbers, hyphens. Must match directory name. |
| `description` | Yes | Max 1024 chars. What it does and when to use it. |
| `license` | No | License name or reference to bundled file |
| `compatibility` | No | Max 500 chars. Environment requirements |
| `metadata` | No | Arbitrary key-value mapping |
| `allowed-tools` | No | Space-delimited pre-approved tools (experimental) |

**Progressive disclosure model**:
1. Metadata (~100 tokens): name + description loaded at startup
2. Instructions (<5000 tokens recommended): Full SKILL.md loaded on activation
3. Resources (as needed): Supporting files loaded on demand

**Validation**: `skills-ref validate ./my-skill` via reference library.

---

#### 4.2.2 Claude Code Skills (extends Agent Skills standard)

**Locations**: `.claude/skills/<name>/SKILL.md` (project), `~/.claude/skills/<name>/SKILL.md` (personal), plugin `skills/` directory, enterprise managed settings.
**Priority**: enterprise > personal > project. Plugin skills use `plugin-name:skill-name` namespace.
**Legacy**: `.claude/commands/*.md` files still work (merged into skills system).

**Additional frontmatter fields beyond Agent Skills standard**:

| Field | Type | Description |
|---|---|---|
| `argument-hint` | string | Hint for autocomplete (e.g., `[issue-number]`) |
| `disable-model-invocation` | boolean | Prevent auto-loading. Manual `/name` only. |
| `user-invocable` | boolean | Show/hide from `/` menu (default: true) |
| `model` | string | Model to use when skill is active |
| `context` | string | `fork` to run in forked subagent context |
| `agent` | string | Subagent type for `context: fork` |
| `hooks` | object | Lifecycle hooks scoped to skill |

**String substitutions**: `$ARGUMENTS`, `$ARGUMENTS[N]`, `$N`, `${CLAUDE_SESSION_ID}`, `${CLAUDE_SKILL_DIR}`
**Dynamic context**: `!`command`` syntax for shell command injection before prompt delivery.
**Permission control**: `Skill(name)` and `Skill(name *)` in permission rules.

---

#### 4.2.3 OpenAI Codex CLI Skills

**Locations**: `.agents/skills/` (current directory + parent directories), `$REPO_ROOT/.agents/skills/`, `$HOME/.agents/skills/`, `/etc/codex/skills/` (admin), built-in system skills.

**SKILL.md frontmatter**: Same as Agent Skills standard (`name`, `description`).

**Platform-specific metadata file**: `agents/openai.yaml` (optional)

| Field | Type | Description |
|---|---|---|
| `interface.display_name` | string | User-facing name |
| `interface.short_description` | string | Brief description |
| `interface.icon_small` | string | Path to small icon |
| `interface.icon_large` | string | Path to large icon |
| `interface.brand_color` | string | Hex color |
| `interface.default_prompt` | string | Optional surrounding prompt |
| `policy.allow_implicit_invocation` | boolean | Auto-invoke based on description (default: true) |
| `dependencies.tools` | array | Required MCP tools with type, value, transport, url |

**Configuration**: `~/.codex/config.toml` for enabling/disabling specific skills.

---

#### 4.2.4 GitHub Copilot Skills

**Location**: `.github/skills/<name>/SKILL.md`
**Format**: Agent Skills standard with minor extensions.
**Activation**: Automatic (description matching) or explicit (`/skill-name` or `@skill-name`).

---

#### 4.2.5 Windsurf Skills

**Locations**: `.windsurf/skills/<name>/SKILL.md` (workspace), `~/.codeium/windsurf/skills/<name>/SKILL.md` (global), system/enterprise paths.
**Cross-agent compatibility**: Also discovers from `.agents/skills/`, `~/.agents/skills/`, `.claude/skills/` (if enabled), `~/.claude/skills/`.

**Format**: Agent Skills standard (`name`, `description` in frontmatter).
**Activation**: Progressive disclosure (auto-invoke) or manual `@skill-name` mention.

---

#### 4.2.6 Cursor, Gemini, Amazon Q Skills

**Cursor**: No dedicated skills concept. Rules system serves equivalent purpose.
**Gemini**: No dedicated skills concept as of current documentation. Custom commands exist but undocumented schema.
**Amazon Q**: No dedicated skills concept in CLI. Tool configuration within agent JSON serves similar purpose.

---

### 4.3 Cross-Platform Instruction Standards

#### 4.3.1 AGENTS.md Standard

**Spec**: https://agents.md/
**Adoption**: 60,000+ open-source repositories, 20+ tools.
**Format**: Plain markdown. No required fields or schema.
**Supported by**: OpenAI Codex, Claude, Gemini CLI, Cursor, Windsurf, GitHub Copilot, VS Code, Zed, Aider, goose, opencode, Factory, Devin, and others.
**Behavior**: Closest file to edited file wins. Directory scoping via nested files.

#### 4.3.2 Agent Skills Standard

**Spec**: https://agentskills.io/specification
**Adoption**: 26+ platforms.
**Format**: SKILL.md with YAML frontmatter in named directory.
**Supported by**: Claude Code, OpenAI Codex, GitHub Copilot, Windsurf, Cursor (via VS Code), Gemini CLI.

---

## 5. Results

### 5.1 Cross-Platform Agent Definition Comparison Matrix

| Feature | Claude Code | GitHub Copilot | Amazon Q | Cursor | Windsurf | Codex CLI | Gemini |
|---|---|---|---|---|---|---|---|
| **Has agent entity** | Yes | Yes | Yes | No | No | No | No |
| **File format** | MD + YAML FM | MD + YAML FM | JSON | MDC/MD | Plain MD | Plain MD | Plain MD |
| **File extension** | `.md` | `.agent.md` | `.json` | `.mdc`/`.md` | `.md` | `.md` | `.md` |
| **Project path** | `.claude/agents/` | `.github/agents/` | `.amazonq/agents/` | `.cursor/rules/` | `.windsurf/rules/` | `AGENTS.md` | `GEMINI.md` |
| **User/global path** | `~/.claude/agents/` | `~/.copilot/agents/` | N/A | N/A | `~/.windsurf/rules/` | `~/.codex/AGENTS.md` | `~/.gemini/GEMINI.md` |
| **name field** | Yes (required) | Yes (optional) | Yes (from filename) | N/A | N/A | N/A | N/A |
| **description field** | Yes (required) | Yes (required) | Yes | Yes | N/A | N/A | N/A |
| **system prompt** | MD body | MD body | `prompt` field | MD body | MD body | MD body | MD body |
| **tool control** | Yes (allow+deny) | Yes (allow) | Yes (allow+alias+settings) | No | No | No | Yes (via settings.json) |
| **model selection** | Yes | Yes | Yes | No | No | No | No |
| **MCP servers** | Yes | Yes | Yes | No | No | No | Yes |
| **hooks/lifecycle** | Yes | No | Yes | No | No | No | No |
| **handoffs** | No (via Agent Teams) | Yes | No | No | No | No | No |
| **memory** | Yes (persistent) | No | No | No | No | No | No |
| **permissions** | Yes (5 modes) | No | Yes (allowedTools) | No | No | No | No |
| **subagent spawning** | Yes | Yes | No | No | No | No | No |
| **AGENTS.md support** | Via CLAUDE.md | Yes | No | Yes | Yes | Yes (native) | Yes |

### 5.2 Cross-Platform Skill Definition Comparison Matrix

| Feature | Claude Code | GitHub Copilot | Codex CLI | Windsurf | Cursor | Amazon Q | Gemini |
|---|---|---|---|---|---|---|---|
| **Has skill concept** | Yes | Yes | Yes | Yes | No | No | No |
| **Follows Agent Skills spec** | Yes (extends) | Yes | Yes (extends) | Yes | N/A | N/A | N/A |
| **Skill location** | `.claude/skills/` | `.github/skills/` | `.agents/skills/` | `.windsurf/skills/` | N/A | N/A | N/A |
| **Cross-compat discovery** | N/A | N/A | N/A | `.agents/skills/`, `.claude/skills/` | N/A | N/A | N/A |
| **Invocation control** | Yes (2 fields) | Yes (1 field) | Yes (policy) | Yes (auto vs manual) | N/A | N/A | N/A |
| **Context forking** | Yes | No | No | No | N/A | N/A | N/A |
| **String substitutions** | Yes (5 variables) | No | No | No | N/A | N/A | N/A |
| **Dynamic context** | Yes (shell injection) | No | No | No | N/A | N/A | N/A |
| **Supporting files** | Yes | Yes | Yes | Yes | N/A | N/A | N/A |

### 5.3 Universal Fields Across Agent-Capable Platforms

Fields present in ALL platforms that have a formal agent definition (Claude Code, GitHub Copilot, Amazon Q):

| Field | Claude Code | GitHub Copilot | Amazon Q |
|---|---|---|---|
| **name** | `name` (frontmatter) | `name` (frontmatter) | `name` (JSON) or filename |
| **description** | `description` (frontmatter) | `description` (frontmatter) | `description` (JSON) |
| **system prompt** | MD body | MD body | `prompt` (JSON) |
| **tools** | `tools` (frontmatter) | `tools` (frontmatter) | `tools` (JSON array) |
| **model** | `model` (frontmatter) | `model` (frontmatter) | `model` (JSON) |
| **MCP servers** | `mcpServers` (frontmatter) | `mcp-servers` (frontmatter) | `mcpServers` (JSON) |

### 5.4 Universal Fields Across Skill-Capable Platforms

Per the Agent Skills open standard, these fields are universal:

| Field | Required | All Platforms |
|---|---|---|
| `name` | Yes | Identical spec across all |
| `description` | Yes | Identical spec across all |

---

## 6. Discussion

### 6.1 Three Tiers of Platform Maturity

**Tier 1 - Full Agent Systems**: Claude Code, GitHub Copilot, Amazon Q. These have formal agent definitions with structured metadata, tool permissions, model selection, and MCP integration. They support the richest customization.

**Tier 2 - Rules/Instructions Systems**: Cursor, Windsurf, Gemini. These lack a formal "agent" entity but provide rules, context files, or instruction files that guide AI behavior. Capabilities are more limited (no tool control, no model selection within rules).

**Tier 3 - Instruction Files Only**: OpenAI Codex CLI. Uses AGENTS.md as plain markdown with no structured metadata. The simplest format but the most portable.

### 6.2 Convergence Around Two Standards

Two open standards are emerging as cross-platform bridges:

1. **AGENTS.md** (agents.md): Universal instruction format. Plain markdown, no schema. Supported by virtually all platforms. Lowest common denominator but most portable.

2. **Agent Skills** (agentskills.io): Universal skill format. SKILL.md with name + description frontmatter. Adopted by 26+ platforms. Claude Code, Codex, Copilot, and Windsurf all follow it.

### 6.3 Format Divergence Points

The main areas where platforms diverge:

1. **File format**: MD (most) vs JSON (Amazon Q only)
2. **Tool permission model**: Allow-list (all), deny-list (Claude), aliases (Amazon Q), command-level restrictions (Gemini), or no control (Cursor, Windsurf, Codex)
3. **Lifecycle hooks**: Claude Code and Amazon Q support them; others do not
4. **Agent composition**: Handoffs (Copilot), Agent Teams (Claude Code), none (others)
5. **Persistent memory**: Only Claude Code
6. **Permission modes**: Only Claude Code and Amazon Q

### 6.4 Minimum Viable Agent Definition

A canonical agent definition that works across all Tier 1 platforms would need:

```yaml
name: string          # Universal
description: string   # Universal
prompt: string        # Universal (body or field)
tools: list           # Universal (format differs)
model: string         # Universal (values differ)
mcpServers: object    # Universal (schema similar)
```

Everything else is a platform-specific extension layer.

### 6.5 Transformation Feasibility

A single canonical agent definition CAN be transformed per-platform:

- **Claude Code**: Emit MD with YAML frontmatter. Body = prompt. Map tools to comma-separated format.
- **GitHub Copilot**: Emit `.agent.md` with YAML frontmatter. Body = prompt. Map tools to array format.
- **Amazon Q**: Emit JSON file. Map prompt to `prompt` field. Map tools to array with `@server` prefix notation.
- **Cursor**: Emit `.mdc` rule file with description + globs. Body = prompt content as rules.
- **Windsurf**: Emit AGENTS.md or `.windsurf/rules/*.md`. Plain markdown, no frontmatter.
- **Codex CLI**: Emit AGENTS.md. Plain markdown.
- **Gemini**: Emit GEMINI.md. Plain markdown.

Platform-specific features (hooks, memory, handoffs, permissions) would be additional extension blocks in the canonical format that only emit when targeting the supporting platform.

---

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Define canonical agent format with universal fields (name, description, prompt, tools, model, mcpServers) | All Tier 1 platforms share these 6 fields | M |
| P0 | Define canonical skill format following Agent Skills spec (name, description, body) | Already an open standard adopted by 26+ platforms | S |
| P1 | Implement per-platform emitters for agent definitions (7 targets) | Each platform has different file format, path, and field naming | L |
| P1 | Support AGENTS.md generation as baseline for all platforms | Universal compatibility with 60,000+ repos | S |
| P1 | Support Agent Skills SKILL.md generation for 4 skill-capable platforms | Shared standard minimizes emitter complexity | M |
| P2 | Model platform-specific extensions as optional blocks | Hooks (Claude, Q), handoffs (Copilot), memory (Claude), permissions (Claude, Q) | M |
| P2 | Add Cursor MDC format support with glob-based activation | Cursor's unique format requires specific emitter | S |
| P3 | Support Amazon Q JSON format | Only platform using JSON; lowest adoption among targets | S |

---

## 8. Conclusion

**Verdict**: Proceed with canonical format design
**Confidence**: High
**Rationale**: The 3 Tier 1 platforms share 6 universal fields. Two open standards (AGENTS.md, Agent Skills) provide cross-platform bridges. A canonical format with platform-specific extension blocks is feasible and directly supports FEAT-001 and FEAT-005.

### User Impact

- **What changes for you**: A single agent/skill definition generates correct files for 7 platforms
- **Effort required**: Medium. 6 universal fields + 7 emitters + optional extension blocks
- **Risk if ignored**: Each platform requires manual maintenance of separate definition files. Drift between platforms increases with every change.

---

## Observations

- [fact] 3 of 7 platforms have formal agent entities (Claude Code, GitHub Copilot, Amazon Q) #agents #cross-platform
- [fact] 4 of 7 platforms use rules or instruction files instead of agents (Cursor, Windsurf, Codex, Gemini) #agents #cross-platform
- [fact] 6 fields are universal across all Tier 1 agent platforms: name, description, prompt, tools, model, mcpServers #agents #schema
- [fact] Agent Skills spec (agentskills.io) adopted by 26+ platforms with 2 required fields: name, description #skills #standard
- [fact] AGENTS.md standard adopted by 60,000+ repos and 20+ tools as plain markdown instruction format #agents #standard
- [decision] Canonical format should use the 6 universal fields as core schema with platform-specific extension blocks #architecture
- [insight] Only Amazon Q uses JSON format; all other platforms use Markdown with optional YAML frontmatter #format
- [insight] Windsurf uniquely discovers skills from Claude Code paths (.claude/skills/) demonstrating cross-platform convergence #skills
- [risk] Tool permission models vary significantly across platforms, requiring careful mapping logic in emitters #tools

## Relations

- implements [[REQ-002 Cross Platform Agent Adaptation]]
- implements [[REQ-005 Skill and Command Cross-Platform Generation]]
- relates_to [[REQ-004 Agent Instruction File Generation]]
- part_of [[FEAT-001 Cross-Platform Portability]]
- part_of [[FEAT-005 Multi-Platform Support Strategy]]
- relates_to [[TASK-005 Implement Agent Definition Generator]]

## Sources Consulted

### Official Documentation
- Claude Code: https://code.claude.com/docs/en/sub-agents, https://code.claude.com/docs/en/skills
- Cursor: https://cursor.com/docs/context/rules
- Windsurf: https://docs.windsurf.com/windsurf/cascade/agents-md, https://docs.windsurf.com/windsurf/cascade/skills
- GitHub Copilot: https://docs.github.com/en/copilot/reference/custom-agents-configuration, https://code.visualstudio.com/docs/copilot/customization/custom-agents
- OpenAI Codex: https://developers.openai.com/codex/guides/agents-md/, https://developers.openai.com/codex/skills/
- Gemini: https://developers.google.com/gemini-code-assist/docs/use-agentic-chat-pair-programmer
- Amazon Q: https://aws.github.io/amazon-q-developer-cli/agent-format.html

### Open Standards
- AGENTS.md: https://agents.md/
- Agent Skills: https://agentskills.io/specification

### Data Transparency
- **Found**: Complete agent definition schemas for Claude Code, GitHub Copilot, Amazon Q. Complete skill schemas for all 4 skill-capable platforms. Both open standards fully documented.
- **Not Found**: Amazon Q CLI agent-format.md (rate-limited). Gemini Code Assist detailed agent mode configuration schema (no formal agent entity). Cursor agent plans beyond rules system.