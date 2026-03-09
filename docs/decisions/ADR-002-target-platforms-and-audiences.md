---
title: ADR-002 Target Platforms and Audiences
type: decision
permalink: decisions/adr-002-target-platforms-and-audiences-1
status: Accepted
date: '2026-03-07'
authors: Peter Kloss
tags:
- architecture
- decision
- platforms
- audiences
- cross-platform
- mcp
- distribution

---

# ADR-002 Target Platforms and Audiences

## Status

**Accepted**

## Context

We are designing `@acmelabs-15/agent-plugin`, a cross-platform AI agent plugin manager. The AI coding agent ecosystem is fragmented across 16+ platforms, each with varying levels of extensibility. ANALYSIS-002 evaluated these platforms against 6 required capabilities: prompts, skills, agents, hooks, MCPs, and running agents in parallel.

The market lacks a unified plugin management solution that works across platforms. Developers building AI-assisted workflows are locked into single-platform tooling, and there is no standard way to package, distribute, and install agent capabilities across the ecosystem.

Key forces at play:

- **Market fragmentation**: 16+ coding agent platforms exist, each with proprietary extension mechanisms
- **Cross-platform demand**: Teams use multiple AI tools and need portable agent capabilities
- **Three distinct user groups**: End users who install plugins, authors who create them, and AI assistants that operate them at runtime
- **No existing standard**: No unified format for packaging agent prompts, skills, hooks, MCPs, and sub-agent configurations
- **Self-bootstrapping risk**: If the manifest structure is not designed early, retrofitting it later causes breaking changes across the ecosystem

## Decision

### 1. Platform Selection Criteria

A platform qualifies as a target if it supports ALL 6 required capabilities:

- **Prompts**: Custom system/user prompts or instruction files
- **Skills**: Reusable tool definitions or callable capabilities
- **Agents**: Custom sub-agent or persona definitions
- **Hooks**: Lifecycle hooks or event-driven automation
- **MCPs**: Model Context Protocol server integration
- **Parallel agents**: Ability to run multiple agents concurrently

This is a hard requirement. Platforms missing any capability are excluded entirely. There is no graceful degradation tier — partial support creates silent failures that are worse than no support.

### 2. Seven Primary Target Platforms (6/6 Capabilities)

The following platforms meet all 6 criteria and receive full first-class support:

| Platform | Vendor | Instruction Files | Skills Path | Agent Definitions |
|:--|:--|:--|:--|:--|
| Claude Code | Anthropic | `CLAUDE.md` | `.claude/skills/` | `.claude/agents/*.md` |
| Cursor | Anysphere | `.cursor/rules/*.mdc`, `AGENTS.md` | `.cursor/skills/`, `.claude/skills/`, `.agents/skills/` | `.cursor/agents/*.md` |
| GitHub Copilot | Microsoft/GitHub | `.github/copilot-instructions.md`, `CLAUDE.md` | `.github/skills/`, `.claude/skills/` | `.github/agents/*.agent.md` |
| Kiro | AWS | `.kiro/steering/*.md`, `AGENTS.md` | `.kiro/skills/` | `.kiro/agents/*.json` |

**Cross-platform coverage**: `AGENTS.md` is read by all 4 supported platforms. `CLAUDE.md` is read by 3/4. `.claude/skills/` is read by 3/4. `.agents/skills/` is read by 3/4 and emerging as a standard.

**General fallback**: For platforms without well-defined paths, the plugin manager uses `.agents/` directory and `AGENTS.md` as the fallback convention.

### 3. Excluded Platforms

Platforms scoring below 6/6 are not supported. Notable exclusions:

| Platform | Score | Missing Capability |
|:--|:--|:--|
| Codex CLI (OpenAI) | 5.5/6 | Hooks are notification-only (cannot block/modify) |
| Cline | 5.5/6 | No custom sub-agent definitions |
| Gemini CLI (Google) | 5.5/6 | Sub-agents run sequentially only |

These platforms may be added in the future if they close their capability gaps. The 6/6 threshold is non-negotiable — partial support creates silent failures that are worse than explicit non-support.

### 4. Three-Audience Model in a Single Package

The package `@acmelabs-15/agent-plugin` serves three audiences through a single distribution:

- **Consumer CLI**: End users install plugins via `agent-plugin install owner/repo`. Manages dependencies, updates platform instruction files, handles version management.
- **Author CLI**: Plugin creators scaffold, build, and validate plugins via `agent-plugin init`, `agent-plugin build`, `agent-plugin validate`. Provides templates, linting, and packaging. There is no `publish` command — distribution is handled by making the plugin available at any supported source (GitHub repo, GitLab repo, npm package, local path).
- **AI Assistants**: At runtime, an embedded MCP server exposes installed plugin metadata to AI agents. Agents query available skills, prompts, and configurations through MCP tool calls. This is the key differentiator versus competitors that only address human users.

The architecture uses shared core logic with thin audience-specific layers. No separate packages.

### 5. Package Identity

- **npm package name**: `@acmelabs-15/agent-plugin` (scoped under the `acmelabs-15` npm organization)
- **Action required**: Register the `acmelabs-15` organization at npmjs.com before first publish

### 6. Platform Instruction File Management

On plugin install, the tool MUST:

- **Create** platform-specific instruction files if they do not exist (e.g., `CLAUDE.md`, `.cursorrules`)
- **Update** existing instruction files without destroying current content (append/merge, never overwrite)
- **Add** installed agent/skill/prompt/hook/MCP entries with usage guidance so both humans and AI assistants understand what was installed and how to use it

### 7. Distribution Model

- **Own format**: A custom manifest and packaging format, borrowing concepts from the Vercel skills standard but extended to cover agents, prompts, hooks, and MCPs
- **No hosted registry**: Plugins are sourced directly, not from a centralized registry
- **Multiple source types supported**:
  - GitHub shorthand: `owner/repo`
  - Full GitHub URL: `https://github.com/owner/repo`
  - GitLab URL: `https://gitlab.com/owner/repo`
  - Local path: `./my-plugin` or `/absolute/path`
  - npm package: `@scope/package-name`
- **Source versioning**: All source types support an optional version specifier. If no version is specified, the latest version is used. Examples:
  - `owner/repo@v1.2.0` or `owner/repo@latest`
  - `@scope/package-name@^2.0.0`
  - Git-based sources resolve versions via tags/releases; npm sources use npm semver
- **Invalid version handling**: If a user specifies a version that doesn't exist, the plugin manager identifies the error, displays available versions, and presents a @clack/prompts select picker listing valid versions (including `latest`) so the user can choose
- **Installed version tracking**: The plugin manager records the installed version of each plugin in its state store. This enables accurate `upgrade`/`update` behavior — the manager knows what's currently installed and can diff against what's available.

### 8. Self-Bootstrapping Strategy

- **Phase 1**: Design the manifest structure with self-bootstrap in mind. The manifest format must be capable of describing `agent-plugin` itself as a plugin.
- **Phase 4**: Ship the self-bootstrap runtime. The tool can install and manage itself through its own plugin system.
- **Acceptance Criteria**: (a) `agent-plugin install acmelabs-15/agent-plugin` works — the plugin.json describes the tool itself as a valid plugin. (b) The tool's own skills, agents, prompts, and hooks are authored as plugin components using its own format, dogfooding the full authoring pipeline.
- **Rationale**: Design-first prevents retrofit breaking changes. Shipping the runtime later avoids premature complexity. This is a deliberate split: the manifest is stable from day one, the runtime matures over 3 phases before self-hosting.

## Consequences

### Positive

- **POS-001**: Cross-platform reach across 4 supported platforms with full capability support covers the majority of the AI coding agent market
- **POS-002**: The three-audience model in a single package simplifies distribution, versioning, and dependency management compared to a multi-package approach
- **POS-003**: The embedded MCP server for AI assistants is a unique differentiator that no competitor currently offers, enabling AI-native plugin discovery and usage
- **POS-004**: Own format with borrowed concepts gives full control over the plugin specification without being constrained by upstream changes to Vercel or Claude Code ecosystems
- **POS-005**: Design-first self-bootstrapping prevents the manifest format from requiring breaking changes when the runtime ships in Phase 4

### Negative

- **NEG-001**: Supporting 4 platforms increases the testing and maintenance surface area (4 platforms × 8 content types = 32+ test combinations)
- **NEG-002**: Platform instruction file management (create/update without breaking) is inherently fragile and will require per-platform parsing logic for each instruction file format
- **NEG-003**: No hosted registry means discovery depends entirely on GitHub/GitLab URLs and word-of-mouth, limiting organic plugin ecosystem growth
- **NEG-004**: Own format means no existing tooling ecosystem to leverage. All validation, linting, and IDE support must be built from scratch
- **NEG-005**: The 6/6 capability requirement is strict. Platforms that add capabilities over time require re-evaluation for inclusion
- **NEG-006**: Instruction file modification is an attack vector. Deferred to ADR-004
- **NEG-007**: No centralized vetting in multi-source model. Deferred to ADR-004

## Alternatives Considered

### Fewer Target Platforms

- **ALT-001**: **Description**: Target only 2-3 platforms (e.g., Claude Code, Cursor, Copilot) to reduce scope and ship faster
- **ALT-001**: **Rejection Reason**: The market is multi-platform and fragmenting further. Cross-platform support is the primary differentiator of this tool. Narrowing the target list undermines the core value proposition.

### Consumer and Author Only (No MCP Server)

- **ALT-002**: **Description**: Build only the Consumer CLI and Author CLI without an embedded MCP server for AI assistants
- **ALT-002**: **Rejection Reason**: The MCP server is the key differentiator versus existing plugin managers. Without it, `agent-plugin` is just another CLI tool. The MCP server enables AI agents to discover and operate plugins autonomously, which is the unique value proposition.

### Separate Packages per Audience

- **ALT-003**: **Description**: Publish `@acmelabs-15/agent-plugin-cli` (consumer), `@acmelabs-15/agent-plugin-author` (author), and `@acmelabs-15/agent-plugin-mcp` (AI assistants) as independent packages
- **ALT-003**: **Rejection Reason**: The three audiences share substantial core logic (manifest parsing, source resolution, platform detection). Separate packages would duplicate this logic, complicate versioning, and fragment the user experience. A single package with thin audience layers is simpler.

### Import from Vercel/Claude Code Ecosystems

- **ALT-004**: **Description**: Adopt the Vercel skills standard or Claude Code agent format directly instead of creating a custom format
- **ALT-004**: **Rejection Reason**: Adopting an upstream format creates a dependency on that ecosystem's evolution. Vercel's format covers skills but not agents, hooks, or MCPs. Claude Code's format is Anthropic-specific. Own format with borrowed concepts provides full coverage and vendor independence.

### Self-Bootstrap from Phase 1

- **ALT-005**: **Description**: Build the self-bootstrap runtime immediately so the tool dogfoods its own plugin system from the start
- **ALT-005**: **Rejection Reason**: Too much upfront complexity. The plugin system, manifest format, and source resolution all need to stabilize before the tool can reliably manage itself. Premature self-hosting risks circular dependency bugs and blocks progress on core features.

### Self-Bootstrap in Phase 4 Only (No Early Design)

- **ALT-006**: **Description**: Defer all self-bootstrapping considerations to Phase 4, designing the manifest without self-hosting constraints
- **ALT-006**: **Rejection Reason**: Retrofitting self-bootstrap capability into a manifest format designed without it risks breaking changes for all existing plugins. The design-first approach ensures the manifest is self-bootstrap-compatible from day one without shipping the runtime prematurely.

## Implementation Notes

- **IMP-001**: Begin with platform adapter interfaces for all 7 primary targets. Each adapter handles instruction file detection, parsing, and safe update for its platform.
- **IMP-002**: No graceful degradation adapters. Only 6/6 platforms are supported. Platforms that close capability gaps can be added via a new platform adapter.
- **IMP-003**: Register the `acmelabs-15` npm organization at npmjs.com before any publish operations.
- **IMP-004**: Design the manifest schema in Phase 1 with explicit fields for self-description (the manifest must be able to describe `agent-plugin` itself as a valid plugin).
- **IMP-005**: Build the MCP server as an embedded component that starts automatically when AI assistants query for plugin metadata. No separate process management required.
- **IMP-006**: Instruction file update logic uses a hybrid managed-section pattern (ANALYSIS-009): HTML comment markers (`<!-- BEGIN AGENT-PLUGIN:name -->` / `<!-- END AGENT-PLUGIN:name -->`) for shared files (CLAUDE.md, AGENTS.md), dedicated per-plugin files for per-file platforms (Cursor .mdc, Kiro steering, Windsurf rules, Copilot CLI path-specific). Tool-owned templates render content from plugin metadata; authors never write raw instruction content. Content sanitization mandatory (3 CVEs prove instruction file injection is a real attack vector).

## References

- [[ANALYSIS-002 Platform Capability Matrix]] — 16+ platform evaluation with 6-capability scoring
- [[ANALYSIS-001 Agent Plugin Foundation and Vision]] — foundational research and vision document
- Vercel Skills Standard — concepts borrowed for manifest design
- Model Context Protocol Specification — MCP server integration standard
- [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]] — session where these decisions were discussed and locked

## Observations

- [decision] 4 primary target platforms selected based on full content type support: Claude Code, Cursor, GitHub Copilot, Kiro (reduced from 7 per Amendment #1) #platforms #cross-platform
- [decision] No graceful degradation tier — platforms below 6/6 are excluded entirely; partial support creates silent failures worse than no support #platforms #no-degradation
- [decision] Three-audience model in single package: Consumer CLI, Author CLI, AI Assistants via embedded MCP server #architecture #audiences
- [decision] Package scoped as @acmelabs-15/agent-plugin on npm #distribution #npm
- [decision] Own plugin format borrowing from Vercel skills standard, no hosted registry, multiple source types #distribution #format
- [decision] Platform instruction files (CLAUDE.md, .cursorrules, etc.) managed non-destructively on install #platforms #instruction-files
- [decision] Self-bootstrap: design manifest in Phase 1, ship runtime in Phase 4 to prevent retrofit breaking changes #architecture #self-bootstrap
- [requirement] All 6 capabilities required for primary platform support: prompts, skills, agents, hooks, MCPs, parallel agents #platforms #criteria
- [constraint] Must register acmelabs-15 npm org before first publish #npm #prerequisite
- [insight] MCP server for AI assistants is the key differentiator versus competitor plugin managers #differentiation #mcp

## Relations

- relates_to [[ANALYSIS-001 Agent Plugin Foundation and Vision]]
- relates_to [[ANALYSIS-002 Platform Capability Matrix]]
- relates_to [[ADR-003-conflict-resolution-and-namespacing]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]
- relates_to [[ANALYSIS-008-platform-instruction-file-paths]]
- relates_to [[ANALYSIS-009-instruction-file-update-patterns]]
- relates_to [[ANALYSIS-014-platform-config-patterns]]

### Amendment #1: Platform Reduction to 4 Supported Platforms (2026-03-09)

The original 7 target platforms are reduced to 4. Only platforms that support ALL required content types (Skills, Agents, Commands, Rules/Instructions, AGENTS.md, Hooks, MCP) are supported.

**Supported platforms**:
1. Claude Code (Anthropic)
2. Cursor (Anysphere)
3. GitHub Copilot (Microsoft/GitHub)
4. Kiro (AWS)

**Dropped platforms** (insufficient content type coverage):
- OpenCode: No file-based hooks (plugin-based only)
- Amp: No file-based agent definitions, no hooks system, commands deprecated in favor of skills
- Windsurf: No file-based agent definitions

**Rationale**: The plugin manager installs 7 content types to platform directories. If a platform doesn't support a content type natively, the installed content would be ignored. Rather than partial support with confusing "skipped" messages, we support platforms that can use everything we install.

**Re-evaluation criteria**: If a dropped platform adds the missing content types, it can be re-added by updating `platforms.config.json` with an entry for that platform. No code changes needed (ADR-009 D3 data-driven design).
