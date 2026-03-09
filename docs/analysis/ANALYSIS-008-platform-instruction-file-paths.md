---
title: ANALYSIS-008 Platform Instruction File Paths
type: note
permalink: analysis/analysis-008-platform-instruction-file-paths-1
tags:
- analysis
- platforms
- file-paths
- instructions
- skills
- agents
- cross-platform
---

> **Platform Scope Change**: Per ADR-002 Amendment #1 (2026-03-09), supported platforms reduced from 7 to 4: Claude Code, Cursor, GitHub Copilot, Kiro. OpenCode, Amp, and Windsurf were dropped due to incomplete content type coverage. References to dropped platforms in this note are historical only.

# ANALYSIS-008 Platform Instruction File Paths

## 1. Objective and Scope

**Objective**: What are the exact file paths each of the 7 target platforms reads for custom prompts, agent definitions, skills, hooks, MCP configuration, and general instructions?

**Scope**: 7 target platforms (Claude Code, Cursor, GitHub Copilot CLI, Kiro, OpenCode, Amp, Windsurf). Research conducted March 2026 using official documentation.

## 2. Context

The agent-plugin project needs to generate platform-specific instruction files during installation. Each platform reads different file paths for different extension types. This analysis maps every known file path per platform so the plugin generator can write to the correct locations.

The AGENTS.md open standard (adopted by 20+ platforms, 60,000+ repos) provides a cross-platform fallback. The `.agents/` directory convention is emerging as a cross-platform skill and check location.

## 3. Approach

**Methodology**: Web research against official documentation for each platform. Fetched actual docs pages for file path specifics.

**Tools Used**: WebSearch (15+ queries), WebFetch (12 pages including official docs for all 7 platforms).

**Limitations**: Some platforms evolve rapidly. Enterprise-only features may have additional paths not documented publicly.

## 4. Data and Analysis

---

### Platform 1: Claude Code (Anthropic)

#### Instruction/Rules Files

| Scope | Path | Format | Notes |
|---|---|---|---|
| Project | `CLAUDE.md` (root) | Markdown | Primary instruction file |
| Project | `.claude/CLAUDE.md` | Markdown | Alternative location |
| Subdirectory | `<dir>/CLAUDE.md` | Markdown | Scoped to directory |
| User global | `~/.claude/CLAUDE.md` | Markdown | All projects |
| Personal (not committed) | `CLAUDE.local.md` | Markdown | Add to .gitignore |

#### Skills

| Scope | Path |
|---|---|
| Project | `.claude/skills/<name>/SKILL.md` |
| User global | `~/.claude/skills/<name>/SKILL.md` |

#### Agents

| Scope | Path |
|---|---|
| Project | `.claude/agents/<name>.md` |
| User global | `~/.claude/agents/<name>.md` |

#### Commands

| Scope | Path |
|---|---|
| Project | `.claude/commands/<name>.md` |

Note: Commands and skills are merged. Both `.claude/commands/review.md` and `.claude/skills/review/SKILL.md` create `/review`.

#### Hooks

| Scope | Path |
|---|---|
| Project | `.claude/settings.json` (hooks section) |
| User global | `~/.claude/settings.json` (hooks section) |

Events: PreToolUse, PostToolUse, Notification, UserPromptSubmit, Stop, and 7 more (12 total).

#### MCP Configuration

| Scope | Path |
|---|---|
| Project | `.mcp.json` (root) |

#### Plugin Manifest

| Scope | Path |
|---|---|
| Plugin root | `.claude-plugin/plugin.json` |

---

### Platform 2: Cursor (Anysphere)

#### Instruction/Rules Files

| Scope | Path | Format | Notes |
|---|---|---|---|
| Project (modern) | `.cursor/rules/<name>.mdc` | MDC (Markdown + metadata) | Recommended format |
| Project (modern alt) | `.cursor/rules/<name>.md` | Markdown | Also supported |
| Project (AGENTS.md) | `AGENTS.md` (root) | Markdown | Cross-platform standard |
| Subdirectory | `<dir>/AGENTS.md` | Markdown | Scoped to directory |
| Project (legacy) | `.cursorrules` (root) | Text | Deprecated, still supported |
| User global | Cursor Settings > Rules | UI-defined | No file path |

#### Skills

| Scope | Path |
|---|---|
| Project (preferred) | `.agents/skills/<name>/SKILL.md` |
| Project | `.cursor/skills/<name>/SKILL.md` |
| Project (compat) | `.claude/skills/<name>/SKILL.md` |
| Project (compat) | `.codex/skills/<name>/SKILL.md` |
| User global | `~/.cursor/skills/<name>/SKILL.md` |
| User global (compat) | `~/.claude/skills/<name>/SKILL.md` |
| User global (compat) | `~/.codex/skills/<name>/SKILL.md` |

#### Agents (Subagents)

| Scope | Path |
|---|---|
| Project | `.cursor/agents/<name>.md` |
| Project (compat) | `.claude/agents/<name>.md` |
| Project (compat) | `.codex/agents/<name>.md` |
| User global | `~/.cursor/agents/<name>.md` |
| User global (compat) | `~/.claude/agents/<name>.md` |
| User global (compat) | `~/.codex/agents/<name>.md` |
| Background output | `~/.cursor/subagents/` |

Precedence: `.cursor/` > `.claude/` > `.codex/` when names conflict.

#### MCP Configuration

| Scope | Path |
|---|---|
| Project | `.cursor/mcp.json` |

---

### Platform 3: GitHub Copilot CLI (Microsoft/GitHub)

#### Instruction/Rules Files

| Scope | Path | Format | Notes |
|---|---|---|---|
| Repository | `.github/copilot-instructions.md` | Markdown | Primary repo-wide instructions |
| Path-specific | `.github/instructions/<name>.instructions.md` | Markdown + frontmatter | Glob-scoped with `applyTo` |
| Project (AGENTS.md) | `AGENTS.md` (root) | Markdown | Cross-platform standard |
| Project (compat) | `CLAUDE.md` (root) | Markdown | Claude Code compatibility |
| Project (compat) | `GEMINI.md` (root) | Markdown | Gemini compatibility |
| User global | `$HOME/.copilot/copilot-instructions.md` | Markdown | User-level |
| Custom dirs | `COPILOT_CUSTOM_INSTRUCTIONS_DIRS` env var | Env var | Comma-separated paths |

#### Skills

| Scope | Path |
|---|---|
| Project | `.github/skills/<name>/SKILL.md` |
| Project (compat) | `.claude/skills/<name>/SKILL.md` |
| User global | `~/.copilot/skills/<name>/SKILL.md` |
| User global (compat) | `~/.claude/skills/<name>/SKILL.md` |

#### Agents

| Scope | Path |
|---|---|
| Project | `.github/agents/<name>.agent.md` |
| User global | `~/.copilot/agents/<name>.agent.md` |

Note: User-level agents override project-level agents with same name.

#### Hooks

| Scope | Path |
|---|---|
| Project | `.github/hooks/*.json` |

#### MCP Configuration

| Scope | Path |
|---|---|
| User global | `~/.copilot/mcp-config.json` |

#### General Config

| Scope | Path |
|---|---|
| User global | `~/.copilot/config.json` |
| Project | `.copilot/settings.json` |
| Personal (not committed) | `.copilot/settings.local.json` |

---

### Platform 4: Kiro (AWS)

#### Instruction/Rules Files (Steering)

| Scope | Path | Format | Notes |
|---|---|---|---|
| Workspace | `.kiro/steering/<name>.md` | Markdown + frontmatter | Inclusion modes via frontmatter |
| Workspace (AGENTS.md) | `AGENTS.md` (root) | Markdown | Always included, no inclusion modes |
| User global | `~/.kiro/steering/<name>.md` | Markdown | All workspaces |
| User global (AGENTS.md) | `~/.kiro/steering/AGENTS.md` | Markdown | Global AGENTS.md |

Foundational steering files (auto-generated): `product.md`, `tech.md`, `structure.md`.

#### Skills

| Scope | Path |
|---|---|
| Workspace | `.kiro/skills/<name>/SKILL.md` |
| User global | `~/.kiro/skills/<name>/SKILL.md` |

Workspace skills take precedence over global when names conflict.

#### Agents (Custom Subagents)

| Scope | Path | Format |
|---|---|---|
| Workspace | `.kiro/agents/<name>.json` | JSON |
| User global | `~/.kiro/agents/<name>.json` | JSON |

Note: Kiro agents use JSON configuration, not Markdown. Prompts referenced via `file://` URIs.

#### Hooks

| Scope | Path |
|---|---|
| Agent config | Defined in agent JSON config `hooks` field |
| Global hooks | Not yet supported (feature request open) |

#### MCP Configuration

| Scope | Path |
|---|---|
| Workspace | `.kiro/settings/mcp.json` |
| User global | `~/.kiro/settings/mcp.json` |

---

### Platform 5: OpenCode (Community)

#### Instruction/Rules Files

| Scope | Path | Format | Notes |
|---|---|---|---|
| Project | `AGENTS.md` (root) | Markdown | Primary |
| Project (compat) | `CLAUDE.md` (root) | Markdown | Fallback, disableable |
| Config-defined | `opencode.json` `instructions` array | Globs, paths, URLs | Most flexible |
| User global | `~/.config/opencode/AGENTS.md` | Markdown | All sessions |
| User global (compat) | `~/.claude/CLAUDE.md` | Markdown | Fallback, disableable |

Compatibility env vars: `OPENCODE_DISABLE_CLAUDE_CODE`, `OPENCODE_DISABLE_CLAUDE_CODE_PROMPT`, `OPENCODE_DISABLE_CLAUDE_CODE_SKILLS`.

#### Skills

| Scope | Path |
|---|---|
| Project | `.opencode/skills/<name>/SKILL.md` |
| Project (compat) | `.claude/skills/<name>/SKILL.md` |
| Project (compat) | `.agents/skills/<name>/SKILL.md` |
| User global | `~/.config/opencode/skills/<name>/SKILL.md` |
| User global (compat) | `~/.claude/skills/<name>/SKILL.md` |
| User global (compat) | `~/.agents/skills/<name>/SKILL.md` |

OpenCode traverses upward from CWD to git root, loading skills along the way.

#### Agents

| Scope | Path | Format |
|---|---|---|
| Project | `.opencode/agents/<name>.md` | Markdown |
| User global | `~/.config/opencode/agents/<name>.md` | Markdown |
| Config-defined | `opencode.json` `agents` section | JSON |

#### Commands

| Scope | Path |
|---|---|
| Project | `.opencode/commands/<name>.md` |
| User global | `~/.config/opencode/commands/<name>.md` |

#### General Config

| Scope | Path |
|---|---|
| Project | `opencode.json` |
| User global | `~/.config/opencode/opencode.json` |
| TUI | `tui.json` / `~/.config/opencode/tui.json` |

Additional directories: `.opencode/modes/`, `.opencode/plugins/`, `.opencode/themes/`, `.opencode/tools/`.

---

### Platform 6: Amp (Sourcegraph)

#### Instruction/Rules Files

| Scope | Path | Format | Notes |
|---|---|---|---|
| Project | `AGENTS.md` (root) | Markdown | Searched up to $HOME |
| Project (alt) | `AGENT.md` (root) | Markdown | Alternative naming |
| Project (compat) | `CLAUDE.md` (root) | Markdown | Legacy naming |
| Subdirectory | `<dir>/AGENTS.md` | Markdown | Scoped to subtree |
| User global | `~/.config/AGENTS.md` | Markdown | Global |
| User global | `~/.config/amp/AGENTS.md` | Markdown | Amp-specific global |

#### Skills

| Scope | Path |
|---|---|
| Project | `.agents/skills/<name>/SKILL.md` |
| Project (compat) | `.claude/skills/<name>/SKILL.md` |
| User global | `~/.config/agents/skills/<name>/SKILL.md` |
| User global | `~/.config/amp/skills/<name>/SKILL.md` |
| User global (compat) | `~/.claude/skills/<name>/SKILL.md` |

Skills can bundle MCP servers via `<skill-dir>/mcp.json`.

#### Code Review Checks

| Scope | Path |
|---|---|
| Project-wide | `.agents/checks/<name>.md` |
| Directory-scoped | `<dir>/.agents/checks/<name>.md` |

#### Workspace Settings

| Scope | Path |
|---|---|
| Project | `.amp/settings.json` |
| User global | `~/.config/amp/settings.json` |
| Enterprise (macOS) | `/Library/Application Support/ampcode/managed-settings.json` |
| Enterprise (Linux) | `/etc/ampcode/managed-settings.json` |

---

### Platform 7: Windsurf (Codeium)

#### Instruction/Rules Files

| Scope | Path | Format | Notes |
|---|---|---|---|
| Project (rules) | `.windsurf/rules/<name>.md` | Markdown | Recommended format |
| Project (AGENTS.md) | `AGENTS.md` (root) | Markdown | Always-on rule |
| Subdirectory | `<dir>/AGENTS.md` | Markdown | Auto-scoped glob rule |
| Project (legacy) | `.windsurfrules` (root) | Text | Legacy, still supported |
| Project (compat) | `.cursorrules` (root) | Text | Cursor compatibility fallback |
| User global | `~/.codeium/windsurf/memories/global_rules.md` | Markdown | All workspaces |

#### Skills

| Scope | Path |
|---|---|
| Project | `.windsurf/skills/<name>/SKILL.md` |
| Project (compat) | `.agents/skills/<name>/SKILL.md` |
| Project (compat) | `.claude/skills/<name>/SKILL.md` |
| User global | `~/.codeium/windsurf/skills/<name>/SKILL.md` |
| User global (compat) | `~/.agents/skills/<name>/SKILL.md` |
| User global (compat) | `~/.claude/skills/<name>/SKILL.md` |
| Enterprise (macOS) | `/Library/Application Support/Windsurf/skills/` |
| Enterprise (Linux) | `/etc/windsurf/skills/` |

#### MCP Configuration

| Scope | Path |
|---|---|
| User global | `~/.codeium/windsurf/mcp_config.json` |

---

## 5. Results

### Cross-Platform File Path Summary

| Extension Type | Claude Code | Cursor | Copilot CLI | Kiro | OpenCode | Amp | Windsurf |
|---|---|---|---|---|---|---|---|
| **Primary instructions** | `CLAUDE.md` | `.cursor/rules/*.mdc` | `.github/copilot-instructions.md` | `.kiro/steering/*.md` | `AGENTS.md` | `AGENTS.md` | `.windsurf/rules/*.md` |
| **AGENTS.md support** | No (uses CLAUDE.md) | Yes | Yes | Yes | Yes | Yes | Yes |
| **Skills directory** | `.claude/skills/` | `.agents/skills/` | `.github/skills/` | `.kiro/skills/` | `.opencode/skills/` | `.agents/skills/` | `.windsurf/skills/` |
| **Agents directory** | `.claude/agents/` | `.cursor/agents/` | `.github/agents/` | `.kiro/agents/` | `.opencode/agents/` | N/A (Task tool) | N/A (no file-based) |
| **Hooks config** | `.claude/settings.json` | Built into skills | `.github/hooks/*.json` | Agent JSON config | Plugin-based | Settings | Cascade settings |
| **MCP config** | `.mcp.json` | `.cursor/mcp.json` | `~/.copilot/mcp-config.json` | `.kiro/settings/mcp.json` | `opencode.json` | `.amp/settings.json` | `~/.codeium/windsurf/mcp_config.json` |

### Cross-Platform Compatibility Paths Read

Many platforms read paths from other platforms as fallbacks:

| Path | Read By |
|---|---|
| `AGENTS.md` (root) | Cursor, Copilot CLI, Kiro, OpenCode, Amp, Windsurf (6 of 7) |
| `CLAUDE.md` (root) | Claude Code, Copilot CLI, OpenCode, Amp (4 of 7) |
| `.claude/skills/<name>/SKILL.md` | Claude Code, Cursor, Copilot CLI, OpenCode, Amp, Windsurf (6 of 7) |
| `.agents/skills/<name>/SKILL.md` | Cursor, OpenCode, Amp, Windsurf (4 of 7) |
| `.claude/agents/<name>.md` | Claude Code, Cursor (2 of 7) |

### Recommended Fallback Strategy

For maximum cross-platform coverage with minimum files:

1. **`AGENTS.md`** in project root: Read by 6 of 7 platforms (all except Claude Code which uses CLAUDE.md)
2. **`CLAUDE.md`** in project root: Read by 4 of 7 platforms (Claude Code primary, plus Copilot CLI, OpenCode, Amp as compat)
3. **`.agents/skills/<name>/SKILL.md`**: Read by 4 of 7 platforms directly
4. **`.claude/skills/<name>/SKILL.md`**: Read by 6 of 7 platforms (highest skill path coverage)

## 6. Discussion

The `.claude/skills/` path has the highest cross-platform reach for skills (6 of 7 platforms). This is because Claude Code established the Agent Skills standard early, and other platforms added compatibility.

The `.agents/` directory is the emerging cross-platform convention. Cursor, OpenCode, Amp, and Windsurf all read `.agents/skills/`. Amp uses `.agents/checks/` for code review. This directory is the best candidate for the general fallback.

`AGENTS.md` is the most universal instruction file. 6 of 7 target platforms read it. Claude Code is the outlier, using `CLAUDE.md` exclusively.

Agent definition formats diverge the most across platforms. Claude Code and Cursor use Markdown files. Copilot CLI uses `.agent.md` extension. Kiro uses JSON. Amp and Windsurf do not have file-based agent definitions.

MCP configuration paths are completely platform-specific with no cross-platform convention.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Generate both `AGENTS.md` and `CLAUDE.md` at project root | Covers all 7 platforms for instruction files | Low |
| P0 | Write skills to `.claude/skills/` | Read by 6 of 7 platforms | Low |
| P1 | Also write skills to `.agents/skills/` | Emerging standard; 4 platforms read it natively | Low |
| P1 | Generate platform-specific instruction files per target | `.cursor/rules/`, `.kiro/steering/`, `.windsurf/rules/`, `.github/copilot-instructions.md` | Medium |
| P2 | Generate platform-specific agent definitions | Format differs: MD vs JSON vs .agent.md | Medium |
| P2 | Generate platform-specific MCP configs | Every platform uses different path and format | High |

## 8. Conclusion

**Verdict**: Proceed with dual-file instruction strategy (AGENTS.md + CLAUDE.md) and `.claude/skills/` as primary skill path.

**Confidence**: High

**Rationale**: Official documentation confirms all file paths. The `.claude/skills/` path provides 6/7 platform coverage. Adding `.agents/skills/` symlink or copy brings the remaining platform (Windsurf native) into coverage. AGENTS.md + CLAUDE.md covers all 7 platforms for instructions.

### User Impact

- **What changes for you**: The plugin generator must write to 2-3 locations per extension type to achieve cross-platform coverage.
- **Effort required**: Low for instructions and skills (known paths). Medium for agents (format differs). High for MCP (completely divergent).
- **Risk if ignored**: Writing to only one platform's paths limits plugin reach to 1 of 7 platforms.

## 9. Appendices

### Observations

- [fact] AGENTS.md is read by 6 of 7 target platforms (all except Claude Code) #cross-platform #instructions
- [fact] .claude/skills/ path is read by 6 of 7 target platforms for Agent Skills #skills #portability
- [fact] .agents/skills/ is read by 4 of 7 platforms: Cursor, OpenCode, Amp, Windsurf #agents-directory #emerging
- [fact] Claude Code is the only target platform that does not read AGENTS.md, requiring separate CLAUDE.md #claude-code
- [fact] Agent definition formats diverge: Markdown (Claude Code, Cursor), .agent.md (Copilot CLI), JSON (Kiro), none (Amp, Windsurf) #agents #fragmentation
- [fact] MCP configuration paths are entirely platform-specific with no cross-platform convention #mcp #divergence
- [decision] Dual-file strategy (AGENTS.md + CLAUDE.md) provides 100% instruction coverage across all 7 platforms #strategy
- [insight] .claude/skills/ has higher cross-platform reach (6/7) than .agents/skills/ (4/7) due to Claude Code's early standard adoption #skills #adoption
- [risk] Agent and MCP config generation requires per-platform adapters due to format divergence #implementation #complexity

### Sources Consulted

- Claude Code Docs: <https://code.claude.com/docs/en/skills>
- Claude Code Showcase: <https://github.com/ChrisWiles/claude-code-showcase>
- Cursor Rules Docs: <https://cursor.com/docs/context/rules>
- Cursor Skills Docs: <https://cursor.com/docs/context/skills>
- Cursor Subagents Docs: <https://cursor.com/docs/context/subagents>
- Copilot CLI Custom Instructions: <https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-custom-instructions>
- Copilot CLI Custom Agents: <https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-custom-agents-for-cli>
- Copilot CLI Skills: <https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-skills>
- Kiro Steering: <https://kiro.dev/docs/steering/>
- Kiro Skills: <https://kiro.dev/docs/skills/>
- Kiro Agent Config Reference: <https://kiro.dev/docs/cli/custom-agents/configuration-reference/>
- OpenCode Config: <https://opencode.ai/docs/config/>
- OpenCode Rules: <https://opencode.ai/docs/rules/>
- OpenCode Skills: <https://opencode.ai/docs/skills/>
- OpenCode Agents: <https://opencode.ai/docs/agents/>
- Amp Manual: <https://ampcode.com/manual>
- Windsurf AGENTS.md: <https://docs.windsurf.com/windsurf/cascade/agents-md>
- Windsurf Skills: <https://docs.windsurf.com/windsurf/cascade/skills>
- AGENTS.md Standard: <https://agents.md/>

### Data Transparency

- **Found**: All primary instruction file paths, skill paths, agent paths, and MCP config paths for all 7 platforms from official documentation
- **Not Found**: Cursor hooks file path (integrated into skills, no separate file); Windsurf hooks file path (Cascade settings, not file-based); Amp agent definition files (uses Task tool, not file-based agents)

## Relations

- extends [[ANALYSIS-002-platform-capability-matrix]]
- implements [[REQ-004 Agent Instruction File Generation]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]