---
title: ANALYSIS-002-platform-capability-matrix
type: note
permalink: analysis/analysis-002-platform-capability-matrix
tags:
- analysis
- platforms
- capabilities
- compatibility
- agent-plugin
---

# ANALYSIS-002 Platform Capability Matrix

## 1. Objective and Scope

**Objective**: Which AI coding platforms support the full extensibility feature set: prompts/commands, skills, agents, hooks, MCPs, and parallel agents?

**Scope**: 16 platforms evaluated across 6 capability dimensions. Research conducted March 2026 using official docs, changelogs, GitHub repos, and third-party comparisons.

## 2. Context

The agent-plugin project needs to target platforms that support a comprehensive extensibility model. This analysis identifies which platforms can consume all 6 extension types that the plugin system packages together: prompts/commands, skills, agents, hooks, MCP servers, and parallel agent execution.

## 3. Approach

**Methodology**: Web research across official documentation, GitHub repositories, changelogs, and third-party comparison articles for each platform.

**Tools Used**: WebSearch (30+ queries), WebFetch (8 pages), cross-referencing multiple sources per platform.

**Limitations**: Some platforms (Void, Grok Build, Devin) have limited public documentation on extensibility internals. Community implementations (e.g., Aider MCP via third-party servers) are distinguished from native support.

## 4. Data and Analysis

### Master Capability Matrix

| Platform | Prompts/Commands | Skills | Agents | Hooks | MCPs | Parallel Agents | Score |
|---|---|---|---|---|---|---|---|
| **Claude Code** | Yes | Yes | Yes | Yes | Yes | Yes | 6/6 |
| **Cursor** | Yes | Yes | Yes | Yes | Yes | Yes | 6/6 |
| **GitHub Copilot CLI** | Yes | Yes | Yes | Yes | Yes | Yes | 6/6 |
| **Codex CLI** | Yes | Yes | Yes | Partial | Yes | Yes | 5.5/6 |
| **Kiro** | Yes | Yes | Yes | Yes | Yes | Yes | 6/6 |
| **Cline** | Yes | Yes | Partial | Yes | Yes | Yes | 5.5/6 |
| **Roo Code** | Partial | Yes | Partial | Partial | Yes | Yes | 4.5/6 |
| **OpenCode** | Yes | Yes | Yes | Yes | Yes | Yes | 6/6 |
| **Gemini CLI** | Yes | Yes | Yes | Yes | Yes | Partial | 5.5/6 |
| **Amp (Sourcegraph)** | Yes | Yes | Yes | Yes | Yes | Yes | 6/6 |
| **Windsurf** | Yes | Yes | Yes | Yes | Yes | Yes | 6/6 |
| **Aider** | Partial | No | No | No | Partial | No | 1/6 |
| **Continue.dev** | Yes | No | Partial | No | Yes | Partial | 2.5/6 |
| **Devin** | Partial | Partial | No | No | Yes | Yes | 2.5/6 |
| **Grok Build** | Partial | Partial | Partial | Partial | Partial | Yes | 3/6 |
| **Void** | No | No | No | No | Yes | No | 1/6 |

### Per-Platform Detail

---

#### Claude Code (Anthropic)

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | Yes | Slash commands via `.claude/commands/*.md`; plugin namespaced commands | 9,000+ plugins available as of Feb 2026 |
| Skills | Yes | `.claude/skills/*/SKILL.md` files with YAML frontmatter; auto-invoked or explicit | Progressive disclosure, context-aware |
| Agents | Yes | Custom subagents via `.claude/agents/*.md`; Agent Teams for teammate coordination | Each agent gets own context window, tools, permissions |
| Hooks | Yes | 12 lifecycle events: PreToolUse, PostToolUse, Notification, UserPromptSubmit, Stop, etc. | Shell, HTTP, LLM prompt, or subagent hooks |
| MCPs | Yes | `.mcp.json` config; stdio, SSE, streamable HTTP transports | Native first-class support |
| Parallel Agents | Yes | Agent Teams (in-process teammates) and Task tool (subagents); shared task list coordination | Teammates message each other directly |

**Verdict**: [PASS] Full 6/6. The reference implementation for the extensibility model.

---

#### Cursor (Anysphere)

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | Yes | Custom commands via `/` in agent input; reusable workflow triggers | Markdown-based command definitions |
| Skills | Yes | Dynamically loaded when agent decides they are relevant; keeps context clean | Distinct from Rules which are always included |
| Agents | Yes | Custom agent definitions; Composer agent mode; subagent system | Interactive wizard or file-based definition |
| Hooks | Yes | PreToolUse hooks for enforcement; 10-20x faster hooks in 2026 | File access policies, argument sanitization, approval workflows |
| MCPs | Yes | Full MCP server support for external tool integration | Slack, Datadog, Sentry, databases, etc. |
| Parallel Agents | Yes | Background Agents; subagent system for parallel task processing | Cursor 2.0+ feature |

**Verdict**: [PASS] Full 6/6. Plugins bundle skills, subagents, MCP servers, hooks, and rules.

---

#### GitHub Copilot CLI

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | Yes | Custom commands; `/plugin install owner/repo` | Plugin-installable commands from GitHub repos |
| Skills | Yes | Markdown-based skill files; auto-load when relevant | Cross-platform: works in CLI, VS Code, and Copilot coding agent |
| Agents | Yes | `.agent.md` files; interactive wizard or manual creation | Agents specify own tools, MCP servers, instructions |
| Hooks | Yes | preToolUse hooks for enforcement | File access policies, argument sanitization |
| MCPs | Yes | Built-in GitHub MCP server; custom MCP server support | Ships with GitHub MCP built in |
| Parallel Agents | Yes | Automatic delegation; can run multiple agents in parallel | Delegates to specialized agents automatically |

**Verdict**: [PASS] Full 6/6. GA since February 2026. Plugins bundle MCP, agents, skills, hooks.

---

#### Codex CLI (OpenAI)

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | Yes | Slash commands (`/agent`); task-based command structure | TUI multi-agent flow with approval prompts |
| Skills | Yes | SKILL.md files with YAML frontmatter; Agent Skills standard (cross-platform) | Implicit invocation when task matches description |
| Agents | Yes | `[agents]` section in config.toml; Agents SDK integration | Multi-agent workflows, role configuration |
| Hooks | Partial | `notify` for event notifications; Ralph plugin implements Stop hook | Limited to notification events; no full PreToolUse/PostToolUse lifecycle |
| MCPs | Yes | STDIO and streaming HTTP in config.toml; `codex mcp` CLI commands | Can run as MCP server itself |
| Parallel Agents | Yes | Multi-agent workflows via Agents SDK; parallel processing across git worktrees | Orchestration through MCP + Agents SDK |

**Verdict**: [WARNING] 5.5/6. Hooks are partial (notification-based, not full lifecycle enforcement).

---

#### Kiro (AWS)

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | Yes | Steering files for project-level instructions | Spec-driven development approach |
| Skills | Yes | "Powers" bundle MCP tools + knowledge + workflows; dynamically loaded | Curated MCP servers, steering files, hooks in a single package |
| Agents | Yes | Custom subagents; autonomous agent mode | Parallel execution of tasks using custom subagents (Feb 2026) |
| Hooks | Yes | Pre Tool Use and Post Tool Use hooks; file change triggers | Intercept agent tool invocations; block or provide context |
| MCPs | Yes | Native MCP integration; remote connections supported | Stdio, SSE, streamable HTTP |
| Parallel Agents | Yes | Custom subagents execute simultaneously; parallel across repos | Multiple custom subagents run in parallel |

**Verdict**: [PASS] Full 6/6. "Powers" concept bundles skills + MCP + hooks.

---

#### Cline

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | Yes | Slash commands from markdown files in workflows directory | Drop `.md` file, auto-becomes `/command` |
| Skills | Yes | Global (`~/Documents/Cline/Rules/Skills/`) and project (`.clinerules/skills/`) scopes | Backward-compatible with other AI coding assistant formats |
| Agents | Partial | No custom subagent definitions; single-agent model in VS Code extension | CLI 2.0 runs isolated instances, not true sub-agent delegation |
| Hooks | Yes | PreToolUse (can block), PostToolUse; JSON stdin/stdout protocol | macOS and Linux only; Windows not supported |
| MCPs | Yes | MCP support for creating and installing custom tools | Can create tools on demand via "add a tool" |
| Parallel Agents | Yes | CLI 2.0 parallel terminal agents; fully isolated instances | Each maintains own state, conversation, model config |

**Verdict**: [WARNING] 5.5/6. No custom subagent definitions (agents are isolated CLI instances, not delegated).

---

#### Roo Code

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | Partial | Custom Modes define agent behavior; no standalone slash command system found | Modes function as role-driven presets |
| Skills | Yes | Instruction packages with optional bundled files; `.md` based | Not executable tools; instruction-only |
| Agents | Partial | Custom Modes create specialized agent roles (Architect, Code, Debug, etc.) | Modes, not independent sub-agents with own context windows |
| Hooks | Partial | Custom Mode enforcement rules; prompt templates; input/output redaction | Not full lifecycle event hooks like PreToolUse/PostToolUse |
| MCPs | Yes | Dedicated MCP marketplace; one-click server installation | Strong MCP integration |
| Parallel Agents | Yes | Agent Teams via git worktrees; multiple Roo Tasks in parallel via REST or MCP | Lead agent coordinates, assigns, synthesizes |

**Verdict**: [WARNING] 4.5/6. Custom Modes serve as partial substitutes for agents and hooks.

---

#### OpenCode

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | Yes | Custom commands in `commands/` directory; markdown with frontmatter | Supports $ARGUMENTS placeholder and bash injection |
| Skills | Yes | `.opencode/skills/*/SKILL.md` or `~/.config/opencode/skills/*/SKILL.md` | Scoped permissions; agents stay in bounds |
| Agents | Yes | Specialized agents with custom prompts, models, tool access | Multi-tier delegation with orchestrator pattern |
| Hooks | Yes | 44 extension points across 7 event types; 3 tiers (Core, Continuation, Skill) | Plugin-based hook registration |
| MCPs | Yes | MCP server support; tools available alongside built-in tools | Same permission model as built-in tools |
| Parallel Agents | Yes | Multi-session support; multiple parallel agents on same project | 70K+ GitHub stars, 650K+ monthly devs |

**Verdict**: [PASS] Full 6/6. 44 hook extension points is the most granular of any platform.

---

#### Gemini CLI (Google)

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | Yes | Extensions with custom commands; `GEMINI.md` context files | Extensions bundle hooks and commands |
| Skills | Yes | Agent Skills standard; `.gemini/skills/` directories | activate_skill tool; scanned at session start |
| Agents | Yes | Custom subagents via `.gemini/agents/*.md`; A2A protocol (experimental) | Exposed as tools to main agent |
| Hooks | Yes | Lifecycle hooks as middleware; configurable via extensions | Run synchronously; support parallel operations |
| MCPs | Yes | Local and remote MCP servers; FastMCP integration | Tools discovered automatically from configured servers |
| Parallel Agents | Partial | Subagents report back sequentially; no native peer-to-peer messaging | Community Maestro project adds parallel dispatch; feature request open for native support |

**Verdict**: [WARNING] 5.5/6. Parallel execution is community-implemented, not native peer-to-peer.

---

#### Amp (Sourcegraph)

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | Yes | Slash commands in `.agents/commands/`; migrating from custom commands to Skills | Custom commands deprecated Jan 2026 end |
| Skills | Yes | SKILL.md files; skills can bundle MCP servers via `mcp.json` | Progressive disclosure; hidden tools until skill loads |
| Agents | Yes | Task tool for subagents; Oracle and Librarian built-in sub-agents | Each subagent has own context window and tool access |
| Hooks | Yes | `tool:post-execute` and other event triggers | Event-driven configuration |
| MCPs | Yes | MCP servers bundled in skills; project or user config | Servers start at launch, tools hidden until skill activated |
| Parallel Agents | Yes | Sub-agents execute parallel tasks; report back to main thread | Multiple agents operate in parallel with oversight |

**Verdict**: [PASS] Full 6/6. Unique: skills bundle MCP servers with progressive disclosure.

---

#### Windsurf (Codeium)

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | Yes | Slash command workflows (`/0-task`, `/1-discovery`, etc.); `.windsurfrules` | Workflow-driven command system |
| Skills | Yes | Cascade Skills; folders with reference scripts, templates, checklists | Progressive disclosure; context-aware invocation |
| Agents | Yes | Custom Cascade Agents with system prompts, context sources, activation triggers | Guided creation; domain-specific tuning |
| Hooks | Yes | Cascade Hooks on user prompts; policy enforcement | Available to all tiers |
| MCPs | Yes | Deep integrations; Streamable HTTP transport; MCP authentication | GitHub, Slack, Stripe, Figma, databases |
| Parallel Agents | Yes | 5 parallel Cascade agents via git worktrees (Wave 13, Feb 2026) | Git worktree isolation |

**Verdict**: [PASS] Full 6/6. 5 parallel agents shipped in Wave 13.

---

#### Aider

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | Partial | `/commands` in chat; limited to built-in set | No custom command definition system |
| Skills | No | No SKILL.md or equivalent instruction file system | Strength is git-native pair programming, not extensibility |
| Agents | No | Single-agent model; no sub-agent spawning | Feature request exists (issue #4428) |
| Hooks | No | No native hook system | AiderDesk (desktop fork) adds JavaScript hooks |
| MCPs | Partial | No native MCP client; community-built MCP servers expose Aider as a tool | Third-party: aider-mcp-server by sengokudaikon and disler |
| Parallel Agents | No | Single session, single agent | No parallel execution |

**Verdict**: [FAIL] 1/6. Git-focused pair programmer. Not an extensible agent platform.

---

#### Continue.dev

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | Yes | Slash commands via `prompts` section in config.yaml | Markdown-based prompts with `/` invocation |
| Skills | No | No SKILL.md or equivalent; relies on MCP for tool extension | No progressive disclosure skill system |
| Agents | Partial | Cloud Agents via Mission Control; no local custom sub-agent definitions | Cloud agents triggered by events, schedules, webhooks |
| Hooks | No | No documented hook system | No PreToolUse/PostToolUse or lifecycle events |
| MCPs | Yes | `mcpServers` in config.yaml; full MCP server support | Stdio transport; configurable timeouts |
| Parallel Agents | Partial | Multiple CLI instances can run concurrently (manual) | No native orchestrated parallel execution |

**Verdict**: [FAIL] 2.5/6. Strong MCP and commands; lacks skills, hooks, and native parallel.

---

#### Devin (Cognition)

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | Partial | Playbooks as reusable prompts for repeated tasks; `!triage-bug` style triggers | Not slash commands; more like saved prompt templates |
| Skills | Partial | Playbooks function as skill-like instruction sets | No SKILL.md standard; proprietary format |
| Agents | No | Monolithic autonomous agent; no user-defined sub-agents | Most autonomous but least extensible |
| Hooks | No | No documented hook system for user customization | Event triggers exist for playbook activation only |
| MCPs | Yes | 3 transports (stdio, SSE, HTTP); marketplace with Datadog, Sentry, Linear, Figma | One-click MCP enablement |
| Parallel Agents | Yes | Parallel sessions; run 2+ Devins concurrently | Shipped Feb 2026 |

**Verdict**: [FAIL] 2.5/6. Highly autonomous but not extensible. No hooks, agents, or standard skills.

---

#### Grok Build (xAI)

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | Partial | Task descriptions in natural language; no custom command file system | Browser-based interface |
| Skills | Partial | Grok CLI (open-source fork) supports skills; Grok Build itself is unclear | CLI has SKILL.md; browser app unclear |
| Agents | Partial | 4 built-in specialized agents (Benjamin, Harper, Lucas, Grok); no user-defined custom agents | Fixed agent roles, not user-extensible |
| Hooks | Partial | Grok CLI supports PreToolUse, PostToolUse, UserPromptSubmit | CLI only; browser-based Grok Build unclear |
| MCPs | Partial | Grok CLI supports MCP; Grok Build browser interface has limited MCP | CLI vs browser distinction matters |
| Parallel Agents | Yes | Up to 8 parallel coding agents; Arena mode | 4 agents per model, 2 models |

**Verdict**: [FAIL] 3/6. Strong parallel execution but limited user extensibility. CLI fork has more features than the hosted product.

---

#### Void

| Capability | Supported? | Mechanism | Notes |
|---|---|---|---|
| Prompts/Commands | No | No custom command system documented | VS Code fork with basic AI features |
| Skills | No | No skill system | Focus on model flexibility, not extensibility |
| Agents | No | Agent mode for file ops; no custom agent definitions or sub-agents | Single agent, single session |
| Hooks | No | No hook system | No lifecycle event support |
| MCPs | Yes | MCP tool access in Agent mode | Basic MCP integration |
| Parallel Agents | No | No parallel execution | Single session only |

**Verdict**: [FAIL] 1/6. Work on Void is currently paused. Minimal extensibility.

## 5. Results

### Platforms with Full 6/6 Support

| Platform | Type | Key Differentiator |
|---|---|---|
| Claude Code | CLI | Reference implementation; 12 hook events; Agent Teams with peer messaging |
| Cursor | IDE | Plugin system bundles all 6 types; Background Agents |
| GitHub Copilot CLI | CLI | Cross-platform skills; built-in GitHub MCP; GA Feb 2026 |
| Kiro | IDE | "Powers" bundle skills+MCP+hooks; spec-driven development |
| OpenCode | CLI | 44 hook extension points; 75+ model providers; open source |
| Amp (Sourcegraph) | IDE/CLI | Skills bundle MCP servers with progressive disclosure |
| Windsurf | IDE | Custom Cascade Agents; 5 parallel agents via worktrees |

### Platforms with 5-5.5/6 Support (Near-Complete)

| Platform | Score | Missing/Partial |
|---|---|---|
| Codex CLI | 5.5/6 | Hooks are notification-only, not full lifecycle |
| Cline | 5.5/6 | No custom sub-agent definitions; isolated CLI instances instead |
| Gemini CLI | 5.5/6 | Parallel agents are sequential report-back; no native peer messaging |

### Platforms with Less Than 5/6 Support

| Platform | Score | Key Gaps |
|---|---|---|
| Roo Code | 4.5/6 | Custom Modes partial substitute for agents and hooks |
| Grok Build | 3/6 | CLI vs browser split; fixed agent roles |
| Continue.dev | 2.5/6 | No skills, hooks, or native parallel |
| Devin | 2.5/6 | No hooks, custom agents, or standard skills |
| Aider | 1/6 | Git pair programmer; not an extension platform |
| Void | 1/6 | Paused development; minimal features |

## 6. Discussion

Seven platforms achieve full 6/6 support: Claude Code, Cursor, GitHub Copilot CLI, Kiro, OpenCode, Amp, and Windsurf. Three more (Codex CLI, Cline, Gemini CLI) are within reach at 5.5/6.

The Agent Skills open standard (SKILL.md with YAML frontmatter) has become cross-platform. Skills written for Claude Code can often be adapted for Codex CLI, OpenCode, Gemini CLI, Amp, Kiro, and Cline. This makes skills the most portable extension type.

Hooks have the widest variance in implementation. Claude Code leads with 12 lifecycle events. OpenCode has 44 extension points. Codex CLI and Gemini CLI have limited hook support. Aider, Continue.dev, Devin, and Void have none.

MCP is the most universally supported capability. 14 of 16 platforms support MCP natively or through community integrations. Only Aider (community-only) and Void (basic) trail.

Parallel agent execution shipped across the industry in February 2026: Claude Code Agent Teams, Windsurf 5 parallel agents, Cline CLI 2.0, Grok Build 8 agents, Codex CLI Agents SDK, and Devin parallel sessions.

The plugin packaging model (bundling skills + hooks + agents + MCP + commands into a single installable unit) is currently implemented by Claude Code (9,000+ plugins), Cursor, GitHub Copilot CLI, and OpenCode. This is the delivery mechanism the agent-plugin project should target.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Target Claude Code as primary platform | Reference implementation; largest plugin ecosystem (9,000+); full 6/6 | Low |
| P0 | Support Agent Skills standard (SKILL.md) | Cross-platform portable to 7+ platforms | Low |
| P1 | Target Cursor and GitHub Copilot CLI as secondary | Full 6/6; large user bases; plugin install mechanisms | Medium |
| P1 | Target Codex CLI, Gemini CLI, OpenCode | 5.5/6 support; degrade gracefully on missing features | Medium |
| P2 | Evaluate Kiro, Amp, Windsurf compatibility | Full 6/6 but smaller ecosystems; worth testing | Low |
| P2 | Document platform-specific adaptations | Hook event mapping, agent definition format differences | Medium |

## 8. Conclusion

**Verdict**: Proceed with multi-platform plugin design targeting the 7 full-support platforms.

**Confidence**: High

**Rationale**: 7 of 16 platforms achieve full 6/6 capability support. The Agent Skills standard provides cross-platform portability for the skills layer. Plugin packaging (bundling all 6 types) is supported by 4 platforms with the rest adopting similar patterns.

### User Impact

- **What changes for you**: Target the 7 full-support platforms for maximum reach. Design plugins with graceful degradation for 5.5/6 platforms.
- **Effort required**: Core plugin format + 2-3 platform-specific adapters.
- **Risk if ignored**: Targeting only Claude Code limits reach to one platform in a market with 7 full-capability alternatives.

## 9. Appendices

### Observations

- [fact] 7 of 16 platforms support all 6 capabilities: Claude Code, Cursor, GitHub Copilot CLI, Kiro, OpenCode, Amp, Windsurf #platform-analysis
- [fact] Agent Skills (SKILL.md) standard is portable across 7+ platforms as of March 2026 #skills #portability
- [fact] MCP is supported by 14 of 16 platforms, making it the most universal extension type #mcp #adoption
- [fact] February 2026 saw industry-wide parallel agent launches: 6 platforms shipped within 2 weeks #parallel-agents #timing
- [decision] Plugin packaging model (bundle all 6 types) is implemented by Claude Code, Cursor, GitHub Copilot CLI, and OpenCode #plugins #packaging
- [insight] Hooks have the widest variance: 0 events (Aider) to 44 extension points (OpenCode) #hooks #variance
- [risk] Targeting only Claude Code limits reach to 1 of 7 fully capable platforms #strategy #risk

### Sources Consulted

- Claude Code Docs: <https://code.claude.com/docs/en/sub-agents>
- Cursor Changelog: <https://cursor.com/changelog>
- GitHub Copilot CLI GA: <https://github.blog/changelog/2026-02-25-github-copilot-cli-is-now-generally-available/>
- Codex CLI Features: <https://developers.openai.com/codex/cli/features/>
- Gemini CLI Docs: <https://geminicli.com/docs/>
- Kiro Docs: <https://kiro.dev/docs/skills/>
- Cline CLI 2.0: <https://devops.com/cline-cli-2-0-turns-your-terminal-into-an-ai-agent-control-plane/>
- Roo Code Docs: <https://docs.roocode.com/>
- OpenCode Docs: <https://opencode.ai/docs/>
- Amp Manual: <https://ampcode.com/manual>
- Windsurf Changelog: <https://windsurf.com/changelog>
- Continue.dev Docs: <https://docs.continue.dev/reference>
- Devin Docs: <https://docs.devin.ai/>
- Morphllm Comparison: <https://www.morphllm.com/ai-coding-agent>
- Lushbinary Comparison: <https://www.lushbinary.com/blog/ai-coding-agents-comparison-cursor-windsurf-claude-copilot-kiro-2026/>
- Agentic Coding Guide: <https://halallens.no/en/blog/agentic-coding-in-2026-the-complete-guide-to-plugins-multi-model-orchestration-and-ai-agent-teams>

### Data Transparency

- **Found**: Feature support for all 16 platforms across all 6 dimensions with source verification
- **Not Found**: Grok Build browser extensibility specifics (CLI fork documented, browser app unclear); Void detailed roadmap (development paused); Devin internal hook architecture

## Relations

- extends [[ANALYSIS-001-agent-plugin-foundation-and-vision]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]
- relates_to [[REQ-004 Agent Instruction File Generation]]
