---
title: ANALYSIS-001 Agent Plugin Foundation and Vision
type: note
permalink: analysis/analysis-001-agent-plugin-foundation-and-vision
tags:
- analysis
- agent-plugin
- foundation
- vision
- ecosystem
---

# ANALYSIS-001 Agent Plugin Foundation and Vision

## 1. Objective and Scope

**Objective**: Validate the foundational design of `@acmelabs-15/agent-plugin` against the current AI coding agent ecosystem. Determine competitive positioning, naming fitness, three-audience model viability, and self-bootstrapping risk profile.

**Scope**: CLI plugin ecosystem landscape, three-audience model validation, naming analysis, self-bootstrapping assessment. Excludes individual dependency evaluations (gunshi, drizzle-orm, etc.) which are separate research tasks.

## 2. Context

The `@acmelabs-15/agent-plugin` project proposes a CLI + embedded MCP server + skill system for managing AI agent plugins across 5 coding platforms (Claude Code, Cursor, Windsurf, Codex, Gemini CLI). Three target audiences: Consumers (install plugins), Authors (create/publish), AI Assistants (operate via MCP tools). Self-bootstrapping: uses itself to create its own content.

The project enters a market that has evolved rapidly between late 2024 and early 2026, with multiple established competitors and a maturing plugin/skills ecosystem.

## 3. Approach

**Methodology**: Web research across official documentation, GitHub repositories, npm registry, community discussions, and industry analysis.
**Tools Used**: WebSearch (12 queries), WebFetch (5 page analyses), npm CLI (2 package lookups), file reading.
**Limitations**: Could not access GitHub rate-limited pages. Some competitor download/usage statistics unavailable.

## 4. Data and Analysis

### 4.1 CLI Plugin Ecosystem Landscape

#### Claude Code Plugin System (Anthropic)

Claude Code has a mature, first-party plugin system launched in public beta early 2026. Key facts:

- [fact] Plugin components: skills, agents, hooks, MCP servers, LSP servers, commands, settings
- [fact] Directory structure: `.claude-plugin/plugin.json` manifest at root; `skills/`, `agents/`, `commands/`, `hooks/` directories at plugin root
- [fact] Marketplace system: Official Anthropic marketplace with 36+ curated plugins (9,000+ total as of Feb 2026)
- [fact] Distribution: GitHub repos, git URLs, local paths, npm packages, pip packages all supported as plugin sources
- [fact] Enterprise features: private marketplaces, managed restrictions (`strictKnownMarketplaces`), auto-updates
- [fact] CLI: `/plugin install`, `/plugin marketplace add`, `/plugin validate`, `/plugin enable/disable`
- [fact] Namespacing: Plugin skills prefixed with plugin name (e.g., `/my-plugin:hello`)

Source: <https://code.claude.com/docs/en/plugins>, <https://code.claude.com/docs/en/discover-plugins>, <https://code.claude.com/docs/en/plugin-marketplaces>

#### Vercel `npx skills` (Direct Competitor)

- [fact] 8,800 GitHub stars, 731 forks, 220 commits. Maintained by Guillermo Rauch (rauchg). Published 4 days ago (v1.4.4).
- [fact] Supports 40+ agents including Claude Code, Cursor, Codex, Windsurf, Gemini CLI, GitHub Copilot, Roo, OpenCode, and more.
- [fact] Skills-only scope: SKILL.md files with YAML frontmatter. No MCP servers, no hooks, no agents, no commands.
- [fact] Installation: symlinks or copies. GitHub, GitLab, local paths. No npm package distribution for individual skills.
- [fact] CLI commands: `add`, `list`, `find`, `remove`, `check`, `update`, `init`
- [fact] Zero dependencies. Lightweight. No build step.

Source: <https://github.com/vercel-labs/skills>, npm registry

#### Alternative Installers

| Tool | Stars | Scope | Unique Feature |
|------|-------|-------|----------------|
| `npx skills` (Vercel) | 8,800 | Skills only | 40+ agent support, largest community |
| `npx add-skill` | Unknown | Skills only | Zero deps, auto-detection |
| `npx openskills` | Unknown | Skills only | Universal SKILL.md installer |
| `npx ai-agent-skills` | Unknown | Skills only | "Homebrew for AI Agent Skills" |
| `npm-agentskills` | Unknown | npm bundled | Bundles skills into npm packages |

Source: npm registry, GitHub

#### AWS Agent Plugins

- [fact] AWS launched `awslabs/agent-plugins` with a `deploy-on-aws` plugin. Supports Claude Code marketplace format and Cursor.
- [fact] Uses Claude Code's native marketplace system for distribution (`/plugin marketplace add awslabs/agent-plugins`).
- [fact] Five-step workflow: analyze, recommend, estimate costs, generate IaC, deploy with confirmation.
- [fact] Demonstrates enterprise-grade plugin authoring pattern.

Source: <https://github.com/awslabs/agent-plugins>, <https://aws.amazon.com/blogs/developer/introducing-agent-plugins-for-aws/>

#### Platform Plugin Systems

| Platform | Plugin System | Config Location | MCP Support | Marketplace |
|----------|--------------|-----------------|-------------|-------------|
| Claude Code | Full plugin system (skills, agents, hooks, MCP, LSP, commands) | `.claude-plugin/plugin.json` | Yes (bundled in plugins) | Official + custom marketplaces |
| Cursor | MCP-centric, rules files | `.cursor/mcp.json`, `.cursor/rules/` | Yes (stdio, SSE, streamable HTTP) | Cursor Marketplace (extensions) |
| Codex | Skills + MCP + plugins | `.codex/config.toml`, `.agents/skills/` | Yes (stdio, streamable HTTP) | Local marketplace + built-in installer |
| Windsurf | MCP + Cascade rules | `mcp_config.json` | Yes (with toggle UI) | No dedicated marketplace |
| Gemini CLI | Extensions (skills, MCP, commands, themes, hooks, sub-agents) | `gemini-extension.json` | Yes | Extension browse/install |

Source: Platform documentation, multiple search results

#### MCP Registries

7+ registries exist for MCP server distribution:

| Registry | Scale | Key Feature |
|----------|-------|-------------|
| Official MCP Registry | Preview | Canonical, verified |
| Glama.ai | ~10,000 servers | Largest, hosted gateway |
| MCP.so | Thousands | Usage-based rankings |
| Smithery.ai | Large | Early mover, hosted |
| MCP-Get | Monitoring | Uptime tracking |
| OpenTools | Curated | Production-ready focus |
| Mastra | Meta-aggregation | Cross-registry view |

Ecosystem grew from ~100 servers (Nov 2024) to 16,670+ (Sep 2025). At least half of early registries have gone to dead domains or pivoted.

Source: <https://nordicapis.com/7-mcp-registries-worth-checking-out/>

### 4.2 Three-Audience Model Validation

**CLI + API + AI Interface Pattern**

Evidence of precedent:

- [fact] FastMCP (Python) explicitly supports CLI usage patterns alongside MCP server hosting. Documentation at <https://gofastmcp.com/patterns/cli> shows dual-interface patterns.
- [fact] Codex runs as both a CLI tool and an MCP server, enabling orchestration via the Agents SDK.
- [fact] `mcp-cli` tools (multiple implementations) serve both human operators (interactive shells) and AI systems (JSON-RPC protocols) simultaneously.
- [fact] No tool found that combines all three audiences (Consumer CLI + Author CLI + AI MCP) in a single package for plugin management.

**Assessment**: The three-audience model is architecturally sound. Each audience pair has precedent: CLI+MCP (FastMCP, Codex), CLI+AI (all coding agent CLIs), Consumer+Author (npm, pip). The full triangle is novel and represents a genuine differentiator.

### 4.3 Naming Analysis

**Current name**: `@acmelabs-15/agent-plugin` (scoped) / `agent-plugin` (unscoped)

| Factor | Finding |
|--------|---------|
| npm `agent-plugin` | Taken. Unrelated Vue.js template package (22 versions, low usage). |
| npm `@acmelabs-15/agent-plugin` | Available. |
| Competitor naming | Vercel uses "skills", Codex uses "skills", Claude Code uses "plugins", AWS uses "agent-plugins" |
| Industry terminology | "plugin" used by Claude Code and AWS. "skill" used by Vercel, Codex, Gemini CLI. "extension" used by Gemini CLI and Cursor. |

**Naming risks**:

- [risk] "agent-plugin" is generic. AWS already uses "agent-plugins" (awslabs/agent-plugins). Collision risk in search results and mindshare.
- [risk] The term "plugin" aligns with Claude Code's ecosystem but not with Vercel/Codex/Gemini which use "skills" or "extensions".
- [risk] The project aims to be cross-platform, but "plugin" is Claude Code-specific terminology. Other platforms call the same concept different things.

**Naming alternatives worth considering**:

- `agent-kit` -- toolbox framing, platform-neutral
- `agent-craft` -- authoring-focused
- `agentpkg` -- package manager framing (like `winget`, `pkg`)
- `plugsmith` -- plugin smithy/forge metaphor
- Keep `@acmelabs-15/agent-plugin` -- scoped name avoids npm collision, "plugin" is the most comprehensive term (Claude Code plugins include skills, agents, hooks, MCP, LSP)

### 4.4 Self-Bootstrapping Assessment

**Precedent in software**: Self-hosting compilers (Rust, Go, TypeScript, OCaml). The pattern dates to 1962 (LISP at MIT).

**Benefits**:

- [fact] Serves as a non-trivial integration test of the tool's own capabilities (dogfooding).
- [fact] Forces the plugin manifest format to be expressive enough for real-world use from day 1.
- [fact] Demonstrates credibility: "we trust our own tool."
- [fact] Compiler bootstrapping proves: improvements to the tool automatically improve the tool's own development workflow.

**Risks**:

- [risk] Chicken-and-egg problem: the tool cannot manage its own plugins until it exists. Requires a bootstrap phase where initial content is manually created.
- [risk] Circular dependency in CI/CD: building the tool requires the tool. Compiler ecosystems solve this with multi-stage bootstrapping (Rust has 4 bootstrap stages).
- [risk] Trusting Trust attack vector: a compromised bootstrap could persist through self-hosted builds. Lower risk for a plugin manager than a compiler, but still a supply chain consideration.
- [risk] Complexity tax: self-bootstrapping adds constraints to every architectural decision ("will this break our own bootstrapping?").
- [risk] Phase dependency: the spec lists self-bootstrapping as Phase 4 of 5. This means Phases 1-3 must ship without self-bootstrapping, then Phase 4 retrofits it. Retrofitting self-hosting is harder than designing for it from the start.

**Mitigation strategies from compiler ecosystems**:

- Stage 0 bootstrap with minimal manual setup
- Automated reproducible builds that verify stage N output matches stage N-1
- Clear separation between "bootstrap mode" and "self-hosted mode"

## 5. Results

### Competitive Position Matrix

| Capability | `@acmelabs-15/agent-plugin` (planned) | `npx skills` (Vercel) | Claude Code Plugins | Codex Skills | Gemini CLI Extensions |
|-----------|-------------------------------------|----------------------|--------------------|--------------|-----------------------|
| Cross-platform | 5 platforms | 40+ agents | Claude Code only | Codex only | Gemini CLI only |
| Skills | Yes | Yes | Yes | Yes | Yes |
| MCP servers | Yes (embedded) | No | Yes (bundled) | Yes (config) | Yes (bundled) |
| Agents | Yes | No | Yes | No | Yes |
| Hooks | Yes | No | Yes | No | Yes |
| Commands | Yes | No | Yes | No | Yes |
| Scaffolding wizard | Yes | `init` only | No | No | No |
| SQLite data store | Yes | No | No | No | No |
| Semantic search | Yes | No | No | No | No |
| AI-operable (MCP) | Yes | No | No | No | No |
| Self-bootstrapping | Yes | No | No | No | No |
| Package distribution | npm, GitHub, local | GitHub, GitLab, local | npm, GitHub, git, pip, local | git, built-in installer | Extensions registry |
| Marketplace | Planned | No | Official + custom | Local + built-in | Browse + install |

### Key Quantified Findings

- Claude Code plugin ecosystem: 9,000+ plugins, 36 in official marketplace
- Vercel skills: 8,800 GitHub stars, 40+ agents supported, 55 npm versions
- MCP registries: 7+ active, 16,670+ servers listed
- Platform coverage gap: No single tool manages full plugin lifecycle (skills + agents + hooks + MCP + commands) across multiple platforms

## 6. Discussion

### The Ecosystem Gap

The current landscape splits into two camps:

1. **Skills-only installers** (`npx skills`, `add-skill`, `openskills`): Cross-platform but limited to SKILL.md files. No MCP, hooks, agents, or commands. Lightweight but narrow.

2. **Platform-native plugin systems** (Claude Code plugins, Codex skills, Gemini extensions): Full-featured but locked to one platform. Rich component model but no portability.

`@acmelabs-15/agent-plugin` targets the gap between these camps: full-featured plugin management that works across platforms. This is a real gap. No existing tool occupies this space.

### Competitive Threat Assessment

The primary threat is not any single competitor but ecosystem lock-in. Claude Code's plugin system is comprehensive and well-designed. If Claude Code dominates the market, cross-platform plugin management becomes less valuable. The value proposition depends on multi-platform development remaining common.

Vercel's `npx skills` has strong momentum (8,800 stars) but is intentionally limited to skills. It does not aspire to manage MCP servers, hooks, or agents. The two tools could coexist: `npx skills` for simple skill distribution, `@acmelabs-15/agent-plugin` for full plugin lifecycle management.

### Three-Audience Model

The model is sound but unprecedented at this scope. The closest precedent is Codex running as both CLI and MCP server. The risk is scope creep: serving three audiences means three sets of UX requirements, three documentation paths, and three testing surfaces.

### Self-Bootstrapping Verdict

Viable but should be designed in from Phase 1, not retrofitted in Phase 4. The spec's phased approach creates a risk: the manifest format and plugin structure decisions made in Phases 1-3 may not accommodate self-bootstrapping requirements discovered in Phase 4. Recommendation: define the self-bootstrapping manifest structure in Phase 1, even if the runtime support ships later.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|----------|---------------|-----------|--------|
| P0 | Keep `@acmelabs-15/agent-plugin` scoped name | Unscoped `agent-plugin` is taken. Scoped name is available and avoids AWS `agent-plugins` confusion. | None |
| P0 | Design self-bootstrapping manifest format in Phase 1 | Retrofitting in Phase 4 risks breaking changes. Compiler bootstrapping literature shows early design prevents costly rework. | Low |
| P1 | Position as "full plugin manager" not "skills installer" | Differentiates from Vercel skills ecosystem. Claude Code's "plugin" terminology validates this framing. | Low |
| P1 | Prioritize Claude Code + Codex + Cursor in Phase 1 | These 3 have the largest user bases and the most mature plugin/MCP infrastructure. Windsurf and Gemini CLI can follow. | Medium |
| P1 | Study Claude Code marketplace.json schema closely | It is the most complete plugin distribution format. Compatibility or interop with it adds immediate value. | Medium |
| P2 | Consider interop with `npx skills` | Vercel's skills ecosystem has 8,800+ stars. Being able to install skills from that ecosystem adds instant value. | Medium |
| P2 | Evaluate whether embedded MCP server is the right AI interface | Every platform already has MCP support. An MCP server that manages plugins is novel but adds operational complexity. | Medium |

## 8. Conclusion

**Verdict**: Proceed with refined scope
**Confidence**: High
**Rationale**: A genuine gap exists between skills-only installers and platform-locked plugin systems. The three-audience model is novel and defensible. Self-bootstrapping is viable with early design.

### User Impact

- **What changes for you**: Clarity on competitive positioning and naming. Evidence-based confidence that the project fills a real gap. Specific risks identified for self-bootstrapping and phasing.
- **Effort required**: Naming decision is immediate (low effort). Self-bootstrapping manifest design should be pulled into Phase 1 (medium effort). Core implementation plan is validated.
- **Risk if ignored**: Without competitive awareness, the project risks building features already well-served by Claude Code's native plugin system or Vercel's skills ecosystem. Without early self-bootstrapping design, Phase 4 becomes a breaking-change risk.

## Observations

- [fact] Claude Code has a comprehensive first-party plugin system with 9,000+ plugins and official marketplace with enterprise features #ecosystem
- [fact] Vercel's `npx skills` has 8,800 GitHub stars and supports 40+ agents but is limited to SKILL.md files only #competitor
- [fact] No existing tool manages the full plugin lifecycle (skills + agents + hooks + MCP + commands) across multiple coding platforms #market-gap
- [fact] The unscoped npm name `agent-plugin` is taken by an unrelated Vue.js package; `@acmelabs-15/agent-plugin` is available #naming
- [fact] AWS uses the term "agent-plugins" for their similar cross-platform plugin project, creating naming collision risk #naming
- [decision] Three-audience model (CLI + Author CLI + AI MCP) is novel and has partial precedents but no full implementation exists #architecture
- [risk] Self-bootstrapping as Phase 4 retrofit creates breaking-change risk; should be designed in Phase 1 #architecture
- [insight] The market splits into skills-only cross-platform tools and full-featured platform-locked systems, leaving a gap for full-featured cross-platform management #market-gap
- [risk] Claude Code ecosystem dominance could reduce the value of cross-platform plugin management if market consolidates around one platform #market-risk

## Relations

- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]
- relates_to [[agent-plugin-design-spec]]

## 9. Appendices

### Sources Consulted

- Claude Code Plugin Docs: <https://code.claude.com/docs/en/plugins>
- Claude Code Discover Plugins: <https://code.claude.com/docs/en/discover-plugins>
- Claude Code Marketplace Guide: <https://code.claude.com/docs/en/plugin-marketplaces>
- Vercel Skills GitHub: <https://github.com/vercel-labs/skills>
- Vercel Skills npm: <https://www.npmjs.com/package/skills>
- AWS Agent Plugins: <https://github.com/awslabs/agent-plugins>
- AWS Blog Post: <https://aws.amazon.com/blogs/developer/introducing-agent-plugins-for-aws/>
- Codex Skills Docs: <https://developers.openai.com/codex/skills/>
- Codex MCP Docs: <https://developers.openai.com/codex/mcp>
- Gemini CLI Extensions: <https://geminicli.com/docs/extensions/>
- Gemini CLI Skills: <https://geminicli.com/docs/cli/skills/>
- Nordic APIs MCP Registries: <https://nordicapis.com/7-mcp-registries-worth-checking-out/>
- Glama MCP State 2025: <https://glama.ai/blog/2025-12-07-the-state-of-mcp-in-2025>
- Awesome Claude Code Plugins: <https://github.com/ccplugins/awesome-claude-code-plugins>
- Awesome Claude Code: <https://github.com/hesreallyhim/awesome-claude-code>
- add-skill.org: <https://add-skill.org/>
- npm-agentskills: <https://github.com/onmax/npm-agentskills>
- Wikipedia Bootstrapping Compilers: <https://en.wikipedia.org/wiki/Bootstrapping_(compilers)>
- FastMCP CLI Patterns: <https://gofastmcp.com/patterns/cli>
- Cursor MCP Docs: <https://cursor.com/docs/context/mcp>
- Codex Changelog: <https://developers.openai.com/codex/changelog/>

### Data Transparency

- **Found**: Detailed plugin system architectures for all 5 target platforms. Competitor feature sets, star counts, and distribution models. MCP registry landscape. npm name availability. Self-bootstrapping precedents.
- **Not Found**: Vercel skills download counts (npm weekly downloads not retrieved). Cursor Marketplace total extension count. Windsurf internal plugin/skills roadmap. Gemini CLI extension registry size. Market share data across coding agent platforms.
