---
title: ANALYSIS-036 AI Platform Plugin Standards Survey
type: analysis
permalink: analysis/analysis-036-ai-platform-plugin-standards-survey-1
tags:
- platforms
- standards
- plugins
- research
- cross-platform
---

> **Platform Scope Change**: Per ADR-002 Amendment #1 (2026-03-09), supported platforms reduced from 7 to 4: Claude Code, Cursor, GitHub Copilot, Kiro. OpenCode, Amp, and Windsurf were dropped due to incomplete content type coverage. References to dropped platforms in this note are historical only.

# ANALYSIS-036 AI Platform Plugin Standards Survey

## 1. Objective and Scope

**Objective**: Map the current plugin, extension, and agent configuration models across eight major AI coding assistant platforms to identify convergence patterns, shared standards, and platform-specific approaches.

**Scope**: Claude Code, Cursor, GitHub Copilot, Google Gemini Code Assist, OpenAI Codex CLI, Windsurf, Amazon Q Developer, JetBrains AI Assistant. Covers plugin architecture, instruction/rules files, MCP support, agent configuration, and distribution models. Research conducted March 2026.

## 2. Context

The AI coding assistant market has fragmented into 8+ major platforms, each with proprietary configuration formats. Two cross-platform standards have emerged: MCP (Model Context Protocol) for tool integration, and AGENTS.md for agent instructions. This survey captures the state of convergence as of March 2026.

## 3. Approach

**Methodology**: Web research across official documentation, blog posts, GitHub repositories, and community discussions. Direct documentation fetch from primary sources.

**Tools Used**: WebSearch, WebFetch across official docs sites for each platform.

**Limitations**: Some enterprise features behind paywalls could not be verified. Rapidly evolving landscape means details may shift within weeks.

## 4. Platform-by-Platform Analysis

---

### 4.1 Claude Code (Anthropic)

**Directory Structure**: `.claude/` for project-level configuration.
- `.claude/settings.json` -- project settings
- `.claude/commands/` -- slash commands (markdown files)
- `.claude/agents/` -- custom agent definitions
- `.claude/skills/` -- agent skills with SKILL.md files
- `.mcp.json` -- MCP server configuration at project root
- `CLAUDE.md` -- project-level instructions (root)

**Plugin System**: Full plugin architecture with `.claude-plugin/plugin.json` manifest.
- Manifest fields: `name`, `description`, `version`, `author`, `homepage`, `repository`, `license`
- Plugin directory contains: `commands/`, `agents/`, `skills/`, `hooks/`, `.mcp.json`, `.lsp.json`, `settings.json`
- Skills use `SKILL.md` files with YAML frontmatter (`description`, `disable-model-invocation`)
- Namespaced skills prevent conflicts: `/plugin-name:skill-name`
- `settings.json` in plugin can set default agent via `agent` key

**Distribution Model**: Plugin marketplaces.
- Official Anthropic marketplace at `claude.ai/settings/plugins/submit`
- `marketplace.json` defines marketplace metadata
- Install via `/plugin install` command or `--plugin-dir` for local development
- Community registry at `claude-plugins.dev`

**Hooks System**: Event-driven automation.
- Defined in `hooks/hooks.json` within plugin or `settings.json` standalone
- Lifecycle events: `PostToolUse`, etc.
- Hooks receive JSON on stdin, can run shell commands

**MCP Integration**: First-class support via `.mcp.json` at project root or within plugins.

**AGENTS.md Support**: Also reads AGENTS.md in addition to CLAUDE.md.

**LSP Integration**: `.lsp.json` for language server configuration in plugins.

---

### 4.2 Cursor

**Directory Structure**: `.cursor/` for project configuration.
- `.cursor/rules/*.mdc` or `.cursor/rules/*.md` -- project rules
- `.cursor/mcp.json` -- MCP server configuration
- `.cursorrules` (legacy, root file) -- deprecated in favor of `.cursor/rules/`

**Rules System**: MDC format with YAML frontmatter.
- Frontmatter fields: `description`, `globs` (file patterns), `alwaysApply` (boolean)
- Four rule types: Always Apply, Apply Intelligently (agent decides), Apply to Specific Files (glob), Apply Manually (@rule-name)
- Precedence: Team Rules > Project Rules > User Rules
- `.cursorrules` does NOT load in agent mode; must use `.cursor/rules/*.mdc`

**MCP Integration**: Full MCP support.
- Configuration in `.cursor/mcp.json`
- One-click MCP server installation with OAuth support
- Supports STDIO, SSE, and Streamable HTTP transports

**Plugin Model**: No dedicated plugin system beyond VS Code extensions.
- Relies on MCP servers as the extension mechanism
- Background Agents for parallel autonomous coding (0.50 release)
- BugBot for automated PR review

**AGENTS.md Support**: Reads `AGENTS.md` in project root and subdirectories (nested support).

---

### 4.3 GitHub Copilot (CLI and IDE)

**Instruction Files**:
- `.github/copilot-instructions.md` -- repository-level custom instructions
- `AGENTS.md` -- supported since August 2025 for coding agent
- `.agent.md` files -- custom agent definitions

**Plugin System (CLI)**: Full plugin architecture.
- `plugin.json` manifest with fields: `name`, `description`, `version`, `author`, `agents`, `skills`, `commands`, `hooks`, `mcpServers`, `lspServers`
- Agents defined as `.agent.md` files with YAML frontmatter
- Agent frontmatter: `description` (required), `name`, `target` (vscode/github-copilot), `tools`, `disable-model-invocation`, `user-invocable`, `mcp-servers`, `metadata`
- Tool aliases: `execute`, `read`, `edit`, `search`, `agent`, `web`, `todo`
- Agent ID derived from filename (e.g., `reviewer.agent.md` -> `reviewer`)
- Max 30,000 characters for agent prompt content

**Distribution**: Plugin install from marketplaces, GitHub repos, git URLs, or local paths.
- `marketplace.json` in `.github/plugin/` or `.claude-plugin/`
- Plugins installed to `~/.copilot/state/installed-plugins/`
- Supports `install`, `uninstall`, `list`, `update`, `enable`, `disable`

**MCP Integration**: Built-in GitHub MCP server plus custom MCP servers.
- Coding agent can use MCP servers configured via `copilot-setup-steps.yml`
- Agent mode in IDE extends via MCP

**Coding Agent**: Autonomous execution in GitHub Actions-based CI environment.
- Separate from local agent mode in VS Code/JetBrains

---

### 4.4 Google Gemini Code Assist

**Configuration Directory**: `~/.gemini/settings.json` for user-level settings.
- `.gemini/styleguide.md` in repository root for code review customization (GitHub)
- No formal `.gemini/rules/` directory; rules set via IDE settings

**Rules System**: Settings-based.
- Rules added via IDE settings: `Geminicodeassist: Rules`
- Natural language rules applied to every prompt
- `styleguide.md` for repository-specific GitHub code review behavior (no schema, free-form)

**MCP Integration**: Supported since October 2025.
- Configured in `~/.gemini/settings.json`
- Must be manually added (no command palette install)
- Migration from Tool Calling API to MCP required by March 2026

**Agent Mode**: Multi-file editing with built-in tools and MCP integration.
- Human-in-the-Loop (HiTL) support
- Gemini 2.5 Pro and Flash models

**Plugin Model**: No dedicated plugin system.
- Extends via MCP servers only
- Code customization (Enterprise) for private codebase suggestions

**AGENTS.md Support**: Supported via Gemini CLI.

---

### 4.5 OpenAI Codex CLI

**Instruction Files**: `AGENTS.md` as primary instruction format.
- Discovery hierarchy: Global (`~/.codex/AGENTS.md`) -> Project root -> Subdirectories toward CWD
- `AGENTS.override.md` at any level supersedes `AGENTS.md`
- Files concatenated from root downward; closer files override earlier guidance
- Max combined size: 32 KiB (`project_doc_max_bytes`)
- Fallback filenames configurable via `project_doc_fallback_filenames`

**Configuration**: TOML format at `~/.codex/config.toml` (global) and `.codex/config.toml` (project).
- Model/provider settings, sandbox mode, approval policies
- Network permissions with domain allowlists/denylists
- Agent definitions under `agents.<name>` with descriptions and config paths
- Feature flags under `features.*` namespace
- Profile-scoped configuration under `profiles.<name>`

**Skills System**: Directory-based skill discovery.
- Locations scanned (priority order): `.agents/skills` (CWD), parent dirs, repo root, `~/.agents/skills` (user), `/etc/codex/skills` (admin), built-in
- `SKILL.md` required with `name` and `description` frontmatter
- Optional `agents/openai.yaml` for extended metadata (interface, policy, dependencies)
- Implicit invocation when task description matches skill description
- Explicit invocation via `/skills` command or `$` mention syntax

**MCP Integration**: Full support.
- Config in `[mcp_servers.<id>]` tables in config.toml
- CLI management: `codex mcp add <name> -- <command>`
- Tool filtering with enabled/disabled lists
- OAuth scopes and startup timeouts

**Plugin System**: Emerging plugin system.
- Loads skills, MCP entries, and app connectors from config or local marketplace
- Install endpoint for enabling plugins from app server

**AGENTS.md**: Origin platform for the standard. Most mature implementation.

---

### 4.6 Windsurf (Codeium / Cognition AI)

**Directory Structure**: `.windsurf/rules/` for workspace rules.
- `.windsurfrules` (legacy, project root) -- still supported
- `.windsurf/rules/*.md` -- current format
- `~/.codeium/windsurf/memories/global_rules.md` -- global rules (6,000 char limit)

**Rules System**: Markdown files with optional YAML frontmatter.
- Frontmatter field: `trigger` with values: `always_on`, `model_decision`, `glob`, `manual`
- `globs` field required for glob trigger mode
- Workspace rules: 12,000 character limit each
- Global rules: 6,000 character limit
- Root-level AGENTS.md is always-on; subdirectory AGENTS.md auto-applies via glob

**Enterprise Rules**: System-level read-only rules via OS paths.
- macOS: `/Library/Application Support/Windsurf/rules/*.md`
- Linux: `/etc/windsurf/rules/*.md`
- Windows: `C:\ProgramData\Windsurf\rules\*.md`
- MDM policy deployment supported

**MCP Integration**: Full MCP support.
- Streamable HTTP transport (replaced SSE)
- MCP authentication (replaced access tokens)
- MCP prompts support
- Toggle MCPs on/off in Cascade header
- @ mention MCPs in Cascade

**Agent (Cascade)**: Two modes -- Code (file modifications) and Chat (Q&A).
- No dedicated plugin system beyond MCP servers

**AGENTS.md Support**: Full support at root and subdirectory levels.

**Ownership Note**: Acquired by Cognition AI (makers of Devin) for ~$250M in December 2025.

---

### 4.7 Amazon Q Developer

**Directory Structure**: `.amazonq/` for project configuration.
- `.amazonq/rules/*.md` -- project rules
- `.amazonq/default.json` -- local MCP configuration
- `~/.aws/amazonq/default.json` -- global MCP configuration
- `~/.aws/amazonq/prompts/` -- reusable custom prompts

**Rules System**: Markdown files in `.amazonq/rules/`.
- Plain markdown, no required frontmatter or schema
- Rules toggled on/off per chat session via UI
- Auto-loaded on first interaction within project
- No priority levels, pattern matching, or conditional application documented

**MCP Integration**: Full support since mid-2025.
- STDIO and HTTP transports
- Tool permissions: Ask, Always Allow, Deny (per tool)
- Enterprise governance: admin can disable MCP or specify allowed server whitelist
- Legacy `mcp.json` support alongside `default.json`
- Local (`.amazonq/default.json`) overrides global (`~/.aws/amazonq/default.json`)

**Custom Prompts**: CLI prompt management system.
- Stored in `~/.aws/amazonq/prompts/` as markdown
- Reusable across conversations and projects
- MCP prompts also supported via CLI

**Plugin Model**: No dedicated plugin/extension architecture.
- Extends via MCP servers and IDE plugins (VS Code, JetBrains)

**AGENTS.md Support**: Not documented.

---

### 4.8 JetBrains AI Assistant

**Directory Structure**: `.aiassistant/rules/` for project rules.
- `.aiassistant/rules/*.md` -- rule files
- MCP configuration via IDE Settings > Tools > AI Assistant > Model Context Protocol

**Rules System**: Markdown files with five activation modes.
- **Always**: Applied to all chat sessions automatically
- **Manually**: Activated via `@rule:` or `#rule:` syntax
- **By Model Decision**: AI determines relevance based on Instruction field
- **By File Patterns**: Triggered when files match patterns (e.g., `*.kt`, `src/**`)
- **Off**: Inactive

**MCP Integration**: Supported since IntelliJ IDEA 2025.1.
- Configuration via Settings UI or import from Claude Desktop config
- STDIO, Streamable HTTP, and SSE (legacy) transports
- Global or project-specific server levels
- "Brave Mode" for running commands without confirmation

**Built-in MCP Server**: JetBrains IDEs (2025.2+) expose an MCP server with 24 tools.
- File operations, code analysis, project introspection, terminal, VCS
- External clients (Claude Desktop, VS Code) can connect to IDE tools

**Prompt Library**: UI-based prompt customization.
- Custom prompts created via Settings > Tools > AI Assistant > Prompt Library
- Rules auto-added to default chat prompts
- Recommend duplicating rules in Prompt Library for consistent application

**Plugin Model**: Standard JetBrains plugin marketplace.
- AI Assistant itself is a plugin
- No dedicated AI agent plugin format beyond rules and MCP

**AGENTS.md Support**: Not explicitly documented.

---

## 5. Cross-Platform Convergence Analysis

### 5.1 AGENTS.md as Emerging Standard

**Status**: De facto industry standard. 60,000+ repositories as of early 2026.

**Governance**: Stewarded by Agentic AI Foundation under Linux Foundation.

**Supported Platforms (confirmed)**:
| Platform | AGENTS.md Support | Notes |
|---|---|---|
| OpenAI Codex CLI | Yes (origin platform) | Full hierarchy with override mechanism |
| GitHub Copilot | Yes (since Aug 2025) | Coding agent and CLI |
| Cursor | Yes | Root and subdirectory, nested support |
| Windsurf | Yes | Root and subdirectory |
| Claude Code | Yes | In addition to CLAUDE.md |
| Google Gemini CLI | Yes | Via Gemini CLI |
| Amazon Q Developer | Not documented | |
| JetBrains AI Assistant | Not documented | |

**Key Characteristics**:
- Plain markdown, no required schema
- Nested files: subdirectory AGENTS.md overrides parent
- Concatenated into prompt context
- Acts as "README for agents" -- build steps, test commands, conventions

### 5.2 MCP as Universal Protocol

**Status**: Industry standard. 97M+ monthly SDK downloads. Donated to Linux Foundation (December 2025).

**Support Matrix**:
| Platform | MCP Support | Transport Types | Config Location |
|---|---|---|---|
| Claude Code | Yes | STDIO, SSE, HTTP | `.mcp.json` |
| Cursor | Yes | STDIO, SSE, Streamable HTTP | `.cursor/mcp.json` |
| GitHub Copilot | Yes | STDIO, HTTP | `.agent.md` frontmatter, plugin config |
| Gemini Code Assist | Yes (Oct 2025) | STDIO, HTTP | `~/.gemini/settings.json` |
| OpenAI Codex CLI | Yes | STDIO | `.codex/config.toml` |
| Windsurf | Yes | STDIO, Streamable HTTP | IDE settings |
| Amazon Q Developer | Yes (mid-2025) | STDIO, HTTP | `.amazonq/default.json` |
| JetBrains AI | Yes (2025.1) | STDIO, Streamable HTTP, SSE | IDE Settings UI |

**Verdict**: MCP is universal. All 8 platforms support it. Configuration format varies per platform.

### 5.3 Shared Plugin Manifest Format

Two platforms have converged on nearly identical `plugin.json` formats:

| Field | Claude Code | GitHub Copilot CLI |
|---|---|---|
| `name` | Required | Required |
| `description` | Optional | Optional |
| `version` | Optional | Optional |
| `author` | Optional (object) | Optional (object) |
| `agents` | Directory | Directory |
| `skills` | Directory | Directory |
| `hooks` | File/object | File/object |
| `mcpServers` | `.mcp.json` | Config |
| `lspServers` | `.lsp.json` | Config |
| `commands` | Directory | Directory |

No other platform has adopted this manifest format. Codex CLI has an emerging plugin system but uses TOML configuration.

### 5.4 Rules/Instructions File Comparison

| Platform | File Location | Format | Activation Modes | Pattern Matching |
|---|---|---|---|---|
| Claude Code | `CLAUDE.md`, `.claude/` | Markdown | Always on | No |
| Cursor | `.cursor/rules/*.mdc` | MDC (frontmatter+MD) | Always, Intelligent, Glob, Manual | Yes (globs) |
| Copilot | `.github/copilot-instructions.md` | Markdown | Always on | No |
| Gemini | IDE settings, `.gemini/styleguide.md` | Text/Markdown | Always on | No |
| Codex | `AGENTS.md` hierarchy | Markdown | Always on (hierarchical) | No |
| Windsurf | `.windsurf/rules/*.md` | Markdown (frontmatter) | Always, Model Decision, Glob, Manual | Yes (globs) |
| Amazon Q | `.amazonq/rules/*.md` | Markdown | Toggle on/off | No |
| JetBrains | `.aiassistant/rules/*.md` | Markdown | Always, Manual, Model Decision, File Patterns, Off | Yes (patterns) |

**Common Pattern**: Markdown as universal format. Three platforms (Cursor, Windsurf, JetBrains) support conditional activation via file pattern matching.

### 5.5 Standards Bodies and Working Groups

| Organization | Initiative | Focus | Status (March 2026) |
|---|---|---|---|
| Linux Foundation | Agentic AI Foundation | MCP + AGENTS.md governance | Active, stewarding both standards |
| NIST | AI Agent Standards Initiative | Interoperability, security, identity | Launched Feb 2026, RFI due March 2026 |
| W3C | AI Agent Protocol Community Group | Agent protocol standards | Active |
| Google + 50 partners | Agent2Agent (A2A) Protocol | Agent-to-agent communication | Linux Foundation project since June 2025 |

---

## 6. Results Summary

### Platform Maturity Ranking (Plugin/Extension Ecosystem)

| Rank | Platform | Plugin System | Rules System | MCP | AGENTS.md | Distribution |
|---|---|---|---|---|---|---|
| 1 | Claude Code | Full (plugin.json, marketplace) | CLAUDE.md + skills | Yes | Yes | Marketplace |
| 2 | GitHub Copilot CLI | Full (plugin.json, marketplace) | .agent.md + instructions | Yes | Yes | Marketplace, GitHub repos |
| 3 | OpenAI Codex CLI | Emerging (skills, TOML config) | AGENTS.md hierarchy | Yes | Yes (origin) | Local marketplace |
| 4 | Cursor | None (MCP only) | MDC rules (4 modes) | Yes | Yes | N/A |
| 5 | Windsurf | None (MCP only) | Frontmatter rules (4 modes) | Yes | Yes | N/A |
| 6 | JetBrains AI | JetBrains marketplace | Rules (5 modes) | Yes | No | JetBrains marketplace |
| 7 | Amazon Q | None | Simple markdown rules | Yes | No | N/A |
| 8 | Gemini Code Assist | None | IDE settings + styleguide.md | Yes | Partial (CLI) | N/A |

### Convergence Score by Feature

| Feature | Platforms Supporting | Convergence Level |
|---|---|---|
| MCP integration | 8/8 (100%) | Universal |
| AGENTS.md support | 6/8 (75%) | Strong |
| Markdown-based rules | 8/8 (100%) | Universal |
| Plugin manifest (plugin.json) | 2/8 (25%) | Low |
| Marketplace distribution | 3/8 (38%) | Low |
| Glob/pattern-based rule activation | 3/8 (38%) | Low |
| Skills/agent definitions | 4/8 (50%) | Moderate |
| LSP integration | 2/8 (25%) | Low |
| Hooks/event system | 2/8 (25%) | Low |

---

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Target MCP as primary extension protocol | 100% platform coverage, Linux Foundation governance, 97M+ monthly SDK downloads | Low -- already standard |
| P0 | Support AGENTS.md as cross-platform instruction format | 75% coverage, growing rapidly, governed by Agentic AI Foundation | Low -- plain markdown |
| P1 | Adopt Claude Code/Copilot CLI plugin.json as reference manifest | Only 2 platforms but both are market leaders with nearly identical formats | Medium -- spec alignment |
| P1 | Implement glob-based rule activation | 3/8 platforms support it; pattern emerging in Cursor, Windsurf, JetBrains | Medium |
| P2 | Monitor NIST AI Agent Standards Initiative | Federal standards body entering the space; RFI due March 2026 | Low -- tracking only |
| P2 | Watch A2A Protocol for agent-to-agent communication needs | Complements MCP; 50+ industry partners; Linux Foundation governance | Low -- future relevance |

---

## 8. Conclusion

**Verdict**: Proceed with MCP + AGENTS.md as the dual-standard foundation. Align plugin manifest with Claude Code/Copilot CLI format.

**Confidence**: High

**Rationale**: MCP has achieved universal adoption across all 8 platforms. AGENTS.md covers 6 of 8 with strong growth trajectory. The plugin.json format used by Claude Code and GitHub Copilot CLI shows the most mature plugin architecture and represents the leading edge of where the industry is heading. No competing plugin manifest standard exists.

### User Impact

- **What changes for you**: Any plugin built on MCP + AGENTS.md + plugin.json will work across the majority of AI coding platforms with minimal adaptation.
- **Effort required**: MCP integration is table stakes. AGENTS.md is zero-effort (plain markdown). Plugin manifest alignment requires matching the Claude Code/Copilot CLI schema.
- **Risk if ignored**: Building on a proprietary format risks lock-in to a single platform while the industry converges on shared standards.

---

## 9. Appendices

### Key Differences: Platform-Specific Config Directories

```
Claude Code:    .claude/          + CLAUDE.md
Cursor:         .cursor/          + .cursorrules (legacy)
Copilot:        .github/          + copilot-instructions.md
Gemini:         .gemini/          + ~/.gemini/settings.json
Codex CLI:      .codex/           + AGENTS.md
Windsurf:       .windsurf/        + .windsurfrules (legacy)
Amazon Q:       .amazonq/         + ~/.aws/amazonq/
JetBrains AI:   .aiassistant/     + IDE Settings
```

### MCP Configuration Format Comparison

```
Claude Code:    .mcp.json (JSON)
Cursor:         .cursor/mcp.json (JSON)
Copilot CLI:    .agent.md frontmatter (YAML) or plugin config
Gemini:         ~/.gemini/settings.json (JSON)
Codex CLI:      .codex/config.toml (TOML)
Windsurf:       IDE settings (JSON)
Amazon Q:       .amazonq/default.json (JSON)
JetBrains:      IDE Settings UI (JSON import)
```

### Sources Consulted

- [Claude Code Plugin Docs](https://code.claude.com/docs/en/plugins)
- [Claude Code Plugin GitHub](https://github.com/anthropics/claude-code/blob/main/plugins/README.md)
- [Cursor Rules Docs](https://cursor.com/docs/context/rules)
- [Cursor MCP Docs](https://docs.cursor.com/context/model-context-protocol)
- [GitHub Copilot CLI Plugin Reference](https://docs.github.com/en/copilot/reference/cli-plugin-reference)
- [GitHub Copilot Custom Agents](https://docs.github.com/en/copilot/reference/custom-agents-configuration)
- [GitHub Copilot MCP Docs](https://docs.github.com/en/copilot/tutorials/enhance-agent-mode-with-mcp)
- [Gemini Code Assist Agent Mode](https://developers.google.com/gemini-code-assist/docs/use-agentic-chat-pair-programmer)
- [OpenAI Codex AGENTS.md](https://developers.openai.com/codex/guides/agents-md/)
- [OpenAI Codex Skills](https://developers.openai.com/codex/skills/)
- [OpenAI Codex Config Reference](https://developers.openai.com/codex/config-reference)
- [OpenAI Codex MCP](https://developers.openai.com/codex/mcp/)
- [Windsurf Cascade Docs](https://docs.windsurf.com/windsurf/cascade/memories)
- [Amazon Q MCP Configuration](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/mcp-ide.html)
- [Amazon Q Project Rules](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/context-project-rules.html)
- [JetBrains MCP Integration](https://www.jetbrains.com/help/ai-assistant/mcp.html)
- [JetBrains Project Rules](https://www.jetbrains.com/help/ai-assistant/configure-project-rules.html)
- [AGENTS.md Official Site](https://agents.md/)
- [AGENTS.md Cross-Tool Guide](https://smartscope.blog/en/generative-ai/github-copilot/github-copilot-agents-md-guide/)
- [NIST AI Agent Standards Initiative](https://www.nist.gov/caisi/ai-agent-standards-initiative)
- [A2A Protocol](https://a2aprotocol.ai/)
- [MCP Wikipedia](https://en.wikipedia.org/wiki/Model_Context_Protocol)
- [Anthropic MCP Announcement](https://www.anthropic.com/news/model-context-protocol)

### Data Transparency

- **Found**: Plugin architectures, rules systems, MCP configuration, AGENTS.md adoption data, standards body initiatives, distribution models, configuration formats for all 8 platforms
- **Not Found**: Exact download/usage numbers per platform, enterprise pricing details for some platforms, internal roadmaps, draft specifications for NIST initiative

## Observations

- [fact] MCP is supported by all 8 surveyed platforms as of March 2026, making it the only universal standard in the AI coding assistant space #standards #mcp
- [fact] AGENTS.md is supported by 6 of 8 platforms (75%) with 60,000+ GitHub repos adopting it, governed by Agentic AI Foundation under Linux Foundation #standards #agents-md
- [fact] Only Claude Code and GitHub Copilot CLI have full plugin manifest systems (plugin.json) with marketplace distribution; their formats are nearly identical #plugins #architecture
- [decision] MCP + AGENTS.md represent the dual-standard foundation that any cross-platform plugin system should target #standards #strategy
- [fact] All 8 platforms use markdown as the base format for rules/instructions files, though activation mechanisms differ (always-on vs glob-based vs model-decision) #rules #convergence
- [fact] NIST launched the AI Agent Standards Initiative in February 2026 focusing on interoperability, security, and identity standards for AI agents #standards #governance
- [insight] The plugin ecosystem is bifurcated: 2 platforms have full plugin architectures (Claude Code, Copilot CLI), while 6 platforms rely on MCP servers as their primary extension mechanism #plugins #architecture
- [fact] Three platforms (Cursor, Windsurf, JetBrains) support glob/pattern-based conditional rule activation with frontmatter metadata, representing an emerging pattern #rules #activation
- [risk] No unified MCP configuration format exists across platforms; each uses different file locations and formats (JSON, TOML, YAML, IDE settings) #mcp #fragmentation
- [fact] Google A2A Protocol (Agent-to-Agent) complements MCP for inter-agent communication, with 50+ technology partners and Linux Foundation governance since June 2025 #protocols #a2a

## Relations

- relates_to [[ADR-002 Target Platforms And Audiences]]
- relates_to [[ANALYSIS-029 Platform Config Registry]]
- relates_to [[ANALYSIS-027 Installation Mechanics]]