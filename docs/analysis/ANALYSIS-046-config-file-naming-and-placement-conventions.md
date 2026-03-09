---
title: ANALYSIS-046 Config File Naming and Placement Conventions
type: note
permalink: analysis/analysis-046-config-file-naming-and-placement-conventions-1
tags:
- configuration
- naming-conventions
- dotfiles
- project-config
- agent-plugin
- ecosystem-research
---

# ANALYSIS-046 Config File Naming and Placement Conventions

## 1. Objective and Scope

**Objective**: Which config file naming pattern and placement convention should `@acmelabs-15/agent-plugin` use for its project-scoped configuration file? This file stores which optional features the user selected when installing plugins to a specific project.

**Scope**: JS/TS ecosystem config naming patterns, platform directory conventions, the `.agents/` standard scope, modern trends (ESLint flat config, Biome, rc fatigue), config file discovery patterns, and AI-specific tooling conventions. Project-scoped only; no user/global scope needed.

## 2. Context

`@acmelabs-15/agent-plugin` is a cross-platform AI agent plugin manager targeting 7 platforms (ADR-002). During `agent-plugin install`, users select optional features (sections, components) from each plugin. The project needs a config file to persist these selections so that subsequent commands (`update`, `uninstall`, `doctor`) know what was installed and how.

Key constraints:
- Project-scoped only (no global/user scope needed)
- Stores feature selections, not behavioral configuration
- Must coexist with existing project files (package.json, tsconfig.json, AGENTS.md, platform dirs)
- Tool name is `@acmelabs-15/agent-plugin`, CLI binary is `agent-plugin`

Related decisions: ADR-001 (plugin format), ADR-009 (platform config registry), ADR-010 (installation lifecycle), ANALYSIS-044 (sectioned content patterns).

## 3. Approach

**Methodology**: Web research across 20+ tools and ecosystems. Fetched primary documentation for AGENTS.md spec, ESLint flat config, Biome, Deno config hell analysis, Node.js tooling issue #79. Cross-referenced with 7 AI coding platforms' config conventions.

**Tools Used**: WebSearch (12 queries), WebFetch (4 page analyses), Brain MCP (existing project notes).

**Limitations**: Node.js tooling issue #79 (.config/ subdirectory proposal) was closed without consensus. No authoritative industry standard exists for config placement. Download/adoption statistics for .config/ directory pattern unavailable.

## 4. Data and Analysis

### 4.1 Config Naming Pattern Inventory

| Pattern | Examples | Who Uses It | When Introduced |
|---------|----------|-------------|-----------------|
| `.foorc` / `.foorc.json` | `.eslintrc`, `.prettierrc`, `.babelrc`, `.lintstagedrc` | ESLint (legacy), Prettier, Babel, lint-staged | 1990s (Unix rc convention) |
| `foo.config.js` / `.ts` | `eslint.config.js`, `tailwind.config.js`, `vite.config.ts`, `commitlint.config.js` | ESLint v9+, Tailwind, Vite, Commitlint, Next.js | 2020s trend |
| `foo.json` at root | `biome.json`, `tsconfig.json`, `turbo.json` | Biome, TypeScript, Turborepo | Various |
| `.foo/config.json` | `.vscode/settings.json`, `.husky/pre-commit` | VS Code, Husky | 2000s+ |
| `.foo/` platform dir | `.claude/`, `.cursor/`, `.kiro/`, `.windsurf/` | Claude Code, Cursor, Kiro, Windsurf | 2024-2025 |
| `package.json` field | `"eslintConfig"`, `"prettier"`, `"lint-staged"` | ESLint (legacy), Prettier, lint-staged | 2010s |

### 4.2 When Directory vs Root File

Research reveals clear decision criteria for when tools use a directory vs a root-level file:

**Use a directory (`.foo/`) when:**
- Multiple related files are needed (VS Code: settings.json, extensions.json, launch.json, tasks.json)
- Tool stores state, cache, or generated files alongside config (Husky: hook scripts)
- Tool has hierarchical config (skills, agents, commands within the directory)
- Tool is a platform/IDE with broad scope (Claude Code, Cursor, VS Code)

**Use a root file when:**
- Configuration is a single concern (Biome: lint + format rules in one file)
- File is consumed by multiple tools (tsconfig.json: TypeScript, bundlers, editors)
- No ancillary files needed
- Tool is a utility, not a platform

**Key finding**: The directory pattern correlates with tool scope. Platforms and IDEs use directories. Single-purpose utilities use root files. The threshold is roughly "does the tool store more than configuration?" If it stores state, scripts, or generated content, use a directory.

### 4.3 ESLint Flat Config Migration: Why the Shift

ESLint moved from `.eslintrc` (JSON/YAML rc file) to `eslint.config.js` (JavaScript config file) in v9.0.0. The reasons documented by the ESLint team:

1. **Performance**: Single file eliminates directory-tree traversal. `.eslintrc` required checking every directory from the linted file up to root for additional config files. Flat config reads one file.
2. **Programmability**: JavaScript config enables dynamic imports, conditional logic, and composition. JSON/YAML rc files cannot express conditionals.
3. **Direct imports**: Plugins referenced by JavaScript import instead of string names. Eliminates the "resolve by name" indirection that caused dependency confusion.
4. **Reduced formats**: One format (JS) instead of 5 (.eslintrc, .eslintrc.json, .eslintrc.yaml, .eslintrc.yml, .eslintrc.js). Reduces cognitive load and documentation surface.
5. **Better defaults**: `ecmaVersion` defaults to "latest" in flat config. Old system required explicit setting.

**Relevance to agent-plugin**: Our config stores data (feature selections), not behavior rules. We do not need programmability, dynamic imports, or conditional logic. The ESLint migration rationale does not apply to our use case. A JSON file is appropriate.

### 4.4 Biome: Why Root File, Not Directory

Biome uses `biome.json` at project root. Key design choices:

- Single file for all concerns (linting, formatting, import sorting)
- Hierarchical discovery: Biome walks parent directories to find config, enabling monorepo support
- No ancillary files needed (no scripts, state, or generated content)
- Simplicity: one file, one format, one location

Biome's choice validates that single-purpose tools with pure-data config belong at the root, not in a directory.

### 4.5 Config File Fatigue ("Root Clutter")

The Node.js community has documented the problem of config file proliferation:

- **Node.js tooling issue #79**: "The creeping scourge of tooling config files in project root directories." Proposed `.config/` subdirectory convention. Closed without consensus.
- **Deno blog**: Documented a Next.js project with 30 config files in the root directory. Deno's answer: zero-config with smart defaults.
- **Hacker News discussion**: Community split between "move configs to .config/" and "fix file explorers to auto-group."
- **Outcome**: No `.config/` standard emerged. Individual tools optionally support alternative locations. The ecosystem fragmented rather than converging.

**Relevance**: Adding another root-level config file contributes to this problem. A directory approach groups agent-plugin files but adds a directory entry. The tradeoff is 1 root file vs 1 root directory.

### 4.6 The `.agents/` Standard: Scope Analysis

The AGENTS.md / `.agents/` standard (now under Linux Foundation's AAIF governance):

**What AGENTS.md is**: A single markdown file at the project root providing instructions to AI coding agents. Adopted by 60,000+ repos. Supported by 8+ platforms.

**What `.agents/` contains**: The `.agents/skills/` directory stores skills (SKILL.md + optional scripts/references). This is the standard location for agent skills across Codex, Cursor, OpenCode, and Amp.

**Scope boundary**: AGENTS.md and `.agents/` are scoped to agent instructions, skills, and behavioral guidance. The standard does not cover:
- Tool configuration (no config.json equivalent)
- Plugin state or lockfiles
- Installation metadata
- Feature selection records

**Key finding**: `.agents/` is an instruction/behavior directory, not a configuration directory. Storing `agent-plugin` config inside `.agents/` would be a scope violation. The standard explicitly separates instructions (in `.agents/`) from tool configuration (tool-specific locations like `.codex/config.toml`, `.claude/settings.json`).

Evidence: Codex stores its config in `~/.codex/config.toml`, not in `.agents/`. Claude Code stores config in `.claude/settings.json`, not in `.agents/`. Each tool has its own config location separate from the shared `.agents/` standard.

### 4.7 AI-Specific Tooling Conventions

| Tool | Config Location | What It Stores |
|------|----------------|----------------|
| Claude Code | `.claude/settings.json` | Project-level settings, allowed tools |
| Cursor | `.cursor/mcp.json` | MCP server config |
| Windsurf | `.windsurfrules` | Project-specific rules |
| Cline | `.clinerules/` | Rules directory with multiple files |
| Codex (OpenAI) | `~/.codex/config.toml` | User-level tool config |
| Vercel Skills | `.skills-lock.json` (root) | Lock file for installed skills |
| GitHub Copilot | `.github/copilot-instructions.md` | Instructions |

**Pattern**: AI platforms use platform-specific directories (`.claude/`, `.cursor/`, `.kiro/`). Cross-platform tools like Vercel Skills use root-level files (`.skills-lock.json`). No AI tool stores its config inside `.agents/`.

### 4.8 Config File Discovery: cosmiconfig Pattern

cosmiconfig (70M+ weekly npm downloads) defines the standard search pattern for JS tools:

For a tool named "myapp", cosmiconfig searches:
1. `package.json` `"myapp"` field
2. `.myapprc` / `.myapprc.json` / `.myapprc.yaml` / `.myapprc.js`
3. `.config/myapprc` / `.config/myapprc.json`
4. `myapp.config.js` / `myapp.config.ts`

The search walks up the directory tree from the working directory to root.

**Relevance**: cosmiconfig's multi-location search is designed for tools that need to support diverse user preferences. Our tool has a single canonical location (project root) and does not need directory-tree traversal. The cosmiconfig pattern is overkill for a project-scoped-only config.

### 4.9 Vercel Skills Precedent

Vercel Skills (`npx skills`) is the closest comparable tool: a cross-platform agent skill installer. Its config approach:
- `.skills-lock.json` at project root for lockfile (what was installed)
- No separate config file for feature selections
- Skills are copied to platform-specific directories

This precedent validates root-level placement for plugin installation metadata.

## 5. Results

### Candidate Evaluation

| Candidate | Pattern | Pros | Cons | Fit Score |
|-----------|---------|------|------|-----------|
| `.agent-plugin/config.json` | Platform dir | Groups all agent-plugin files. Room for future files (lockfile, cache). Clear ownership. | Adds a directory to root. Longer path. No precedent for this specific name. | 7/10 |
| `agent-plugin.config.json` | Root file | Follows `foo.config.json` convention. Single file, clear naming. Discoverable. | Adds to root clutter. Hyphens in filename (valid but unusual for config files). | 6/10 |
| `.agentrc` / `.agentrc.json` | rc file | Familiar Unix convention. Short name. | "agent" is generic (conflicts possible). rc convention is declining (ESLint moved away). Does not clearly identify the tool. | 3/10 |
| `agent.config.json` | Root file | Short. Follows `foo.config.json` pattern. | "agent" is too generic. Conflicts with other agent tools. | 3/10 |
| `.agents/plugins/config.json` | Inside .agents standard | Leverages existing .agents/ directory. | Scope violation: .agents/ is for instructions/skills, not tool config. Breaks standard scope boundary. | 2/10 |
| `package.json` field | Embedded | No new files. | Coupling to npm. Not all projects have package.json. Non-Node projects excluded. | 2/10 |

### Trend Analysis: Where the Ecosystem Is Moving

| Direction | Evidence | Confidence |
|-----------|----------|------------|
| Away from rc files | ESLint dropped .eslintrc in v9. New tools (Biome, Turbo) never used rc pattern. | High |
| Toward `foo.config.js` for programmable config | ESLint, Tailwind, Vite, Next.js all use this. But only for config that benefits from JS programmability. | High |
| Toward `foo.json` for data config | Biome, TypeScript, Turborepo use plain JSON at root for pure-data config. | High |
| Platform directories for broad-scope tools | Claude Code, Cursor, VS Code all use `.foo/` directories. But these are IDEs/platforms, not utilities. | High |
| `.config/` subdirectory | Proposed but never achieved consensus. No major tool adopted it as primary location. | Low |
| Package.json embedding declining | Vue CLI, ESLint, Prettier all moved toward standalone files. Less config in package.json over time. | Medium |

### When to Use Directory vs Root File (Decision Framework)

```
Does the tool need to store multiple related files?
  YES --> Directory (.foo/)
  NO  --> Does the config benefit from JS programmability?
            YES --> foo.config.js at root
            NO  --> foo.json at root
```

For agent-plugin: We store a single config file (feature selections). No scripts, no generated files, no state. A root-level JSON file is the appropriate choice by this framework. However, if we anticipate a lockfile or cache file in the future, a directory provides room to grow.

## 6. Discussion

### The Directory Question

The strongest argument for `.agent-plugin/` is future expansion. ADR-010 (installation lifecycle) mentions lockfile tracking. If a lockfile is needed alongside config, a directory groups related files cleanly.

The strongest argument against is that directories signal platform-level scope (Claude Code, VS Code, Cursor), while `agent-plugin` is a utility. Adding a directory for a single file feels premature.

### The `.agents/` Question

Placing config inside `.agents/` would be a scope violation. The `.agents/` standard covers instructions and skills. Every AI platform stores its own config separately from `.agents/`:
- Codex: `~/.codex/config.toml`
- Claude Code: `.claude/settings.json`
- Cursor: `.cursor/mcp.json`

Following this precedent, agent-plugin should store its config separately from `.agents/`.

### The Naming Question

`agent-plugin.config.json` follows the `foo.config.json` convention used by modern tools. The hyphen in the name is unusual (most tools are single words: `biome.json`, `turbo.json`) but not invalid. The full tool name provides unambiguous identification.

A shorter name like `agent.config.json` risks collision. "Agent" is now a generic term used by many tools.

### Recommendation: `.agent-plugin/config.json`

Despite the single-file-for-now situation, the directory approach wins for three reasons:

1. **Future lockfile**: ADR-010's installation lifecycle tracks installed components. A lockfile (`.agent-plugin/lock.json` or similar) is likely. The directory provides a natural home.

2. **Precedent alignment**: The tool manages installations across multiple platforms, similar to how `.vscode/` manages VS Code workspace config. The scope is broader than a single config value.

3. **Root clutter reduction**: One directory entry is less visual noise than multiple root files if a lockfile is added later. The `.` prefix keeps it out of `ls` output by default.

4. **Clear ownership**: `.agent-plugin/` unambiguously belongs to the `agent-plugin` tool. No collision risk with other tools.

The `config.json` filename inside the directory is deliberately simple. It does not need the tool name in the filename because the directory already provides namespacing.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|----------|---------------|-----------|--------|
| P0 | Use `.agent-plugin/config.json` for project-scoped config | Directory provides room for lockfile, clear ownership, reduces root clutter vs multiple files. Follows `.vscode/`, `.husky/` pattern for tools with multiple concerns. | Low |
| P0 | Do NOT place config inside `.agents/` | `.agents/` standard scope is instructions/skills only. Every AI platform stores tool config separately from `.agents/`. Scope violation. | N/A |
| P1 | Use JSON format, not JS/TS | Config stores data (feature selections), not behavior. No programmability needed. JSON provides schema validation, IDE autocomplete, and cross-language parsing. | N/A |
| P1 | Add `.agent-plugin/` to `.gitignore` guidance with opt-in tracking | Feature selections may be personal preference (like VS Code settings). Document that teams can choose to commit or gitignore. | Low |
| P2 | Define JSON Schema for config.json from day one | Enables IDE autocomplete, validation, and forward-compatible schema evolution (ADR-001 pattern). | Low |
| P2 | Plan lockfile as `.agent-plugin/lock.json` | Separate from config to allow independent versioning. Config = what user chose. Lock = what was installed (hashes, versions, paths). | Low |

## 8. Conclusion

**Verdict**: Proceed with `.agent-plugin/config.json`

**Confidence**: High

**Rationale**: The directory pattern (`.agent-plugin/`) is appropriate because the tool will likely need multiple files (config + lockfile). The `.agents/` standard is scoped to instructions/skills and should not contain tool config. Root-level `agent-plugin.config.json` would work for a single file but does not scale to lockfile + cache without contributing to root clutter. The directory name uses the full tool name to avoid collision with the generic term "agent."

### User Impact

- **What changes for you**: Running `agent-plugin install` creates `.agent-plugin/config.json` in your project root. This file records which features you selected. Add `.agent-plugin/` to `.gitignore` for personal preferences, or commit it for team-shared defaults.
- **Effort required**: Low. Single directory with one JSON file. JSON Schema for validation.
- **Risk if ignored**: Without a defined convention, config placement becomes ad-hoc. Changing placement after release is a breaking change requiring migration tooling.

## 9. Appendices

### Sources Consulted

- [ESLint Configuration Migration Guide](https://eslint.org/docs/latest/use/configure/migration-guide)
- [ESLint Flat Config Rollout Plans](https://eslint.org/blog/2023/10/flat-config-rollout-plans/)
- [ESLint New Config System Part 2](https://eslint.org/blog/2022/08/new-config-system-part-2/)
- [Biome Configuration Reference](https://biomejs.dev/reference/configuration/)
- [Configure Biome Guide](https://biomejs.dev/guides/configure-biome/)
- [cosmiconfig GitHub](https://github.com/cosmiconfig/cosmiconfig)
- [cosmiconfig npm](https://www.npmjs.com/package/cosmiconfig)
- [Node.js Tooling Issue #79: Config File Proliferation](https://github.com/nodejs/tooling/issues/79)
- [Node.js Tooling Issue #71: Config File Recommendations](https://github.com/nodejs/tooling/issues/71)
- [Deno Blog: Node.js Config Hell](https://deno.com/blog/node-config-hell)
- [AGENTS.md Specification](https://agents.md/)
- [AGENTS.md OpenAI Codex Guide](https://developers.openai.com/codex/guides/agents-md/)
- [OpenAI Codex Agent Skills](https://developers.openai.com/codex/skills/)
- [Linux Foundation AAIF Announcement](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)
- [Claude Code Settings Documentation](https://code.claude.com/docs/en/settings)
- [Cursor MCP Configuration](https://cursor.com/docs/context/mcp)
- [VS Code User and Workspace Settings](https://code.visualstudio.com/docs/configure/settings)
- [Hacker News: Config File Proliferation Discussion](https://news.ycombinator.com/item?id=24066748)
- [npm package.json Documentation](https://docs.npmjs.com/cli/v11/configuring-npm/package-json/)
- [lint-staged GitHub](https://github.com/lint-staged/lint-staged)
- [Tailwind CSS Configuration](https://v3.tailwindcss.com/docs/configuration)

### Data Transparency

- **Found**: Complete config naming patterns for 20+ tools. ESLint flat config migration rationale (4 blog posts, official docs). Biome config placement reasoning. cosmiconfig search pattern (70M+ weekly downloads). AGENTS.md spec scope (instructions/skills only, no tool config). AI platform config locations for 7 platforms. Node.js tooling issue #79 outcome (no consensus). Deno config hell statistics (30 files in one project root). Vercel Skills lockfile precedent (.skills-lock.json).
- **Not Found**: Adoption statistics for `.config/` subdirectory pattern. Quantitative data on how many projects use each naming convention. User research on config file discoverability preferences. Whether any AI tool stores config inside `.agents/` (none found; all use separate locations).

## Observations

- [decision] .agent-plugin/config.json recommended as project-scoped config location: directory provides room for lockfile, clear ownership, reduces root clutter #config-placement #naming-convention
- [fact] AGENTS.md and .agents/ directory are scoped to instructions and skills only. No AI platform stores tool config inside .agents/. Codex uses ~/.codex/config.toml, Claude Code uses .claude/settings.json, Cursor uses .cursor/mcp.json #agents-standard #scope-boundary
- [fact] ESLint migrated from .eslintrc to eslint.config.js for programmability, performance (single file vs directory traversal), and direct imports. This rationale does not apply to pure-data config files #eslint #flat-config #migration-rationale
- [fact] Node.js tooling issue #79 proposed .config/ subdirectory convention to reduce root clutter. Closed without consensus. No major tool adopted it as primary location #config-fatigue #no-standard
- [fact] Deno blog documented 30 config files in a single Next.js project root, calling it "config hell." Deno's answer: zero-config with smart defaults #config-proliferation #ecosystem-problem
- [fact] cosmiconfig (70M+ weekly downloads) searches .foorc, .config/foorc, foo.config.js, and package.json fields. Multi-location search is standard for tools supporting diverse user preferences #cosmiconfig #discovery-pattern
- [fact] Vercel Skills (closest comparable tool) uses .skills-lock.json at project root for lockfile, validating root-level placement for installation metadata #vercel-skills #precedent
- [insight] Directory pattern (.foo/) correlates with tool scope: platforms/IDEs use directories (VS Code, Claude Code, Cursor), single-purpose utilities use root files (Biome, TypeScript, Turbo) #directory-vs-root #decision-framework
- [insight] rc file convention is declining: ESLint dropped .eslintrc in v9, new tools (Biome, Turbo) never used rc pattern. Modern trend is foo.config.js for programmable config and foo.json for data config #naming-trends #rc-decline
- [risk] Placing config inside .agents/ would violate the standard's scope boundary. If the AAIF governance later formalizes this boundary, retroactive migration would be needed #scope-violation #standards-risk
- [constraint] Tool name contains hyphen (agent-plugin), making config file naming slightly unusual: agent-plugin.config.json vs the more common single-word pattern (biome.json, turbo.json) #naming #hyphenated-tool-name

## Relations

- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-009 Platform Detection and Config Registry]]
- relates_to [[ADR-010 Installation Lifecycle]]
- extends [[ANALYSIS-044 Sectioned Content Configuration Patterns]]
- relates_to [[ANALYSIS-014 Platform Config Patterns]]
- relates_to [[ANALYSIS-011 Lockfile Management Patterns]]