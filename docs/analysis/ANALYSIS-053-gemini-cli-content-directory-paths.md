---
title: ANALYSIS-053 Gemini CLI Content Directory Paths
type: analysis
permalink: analysis/analysis-053-gemini-cli-content-directory-paths
tags:
- gemini-cli
- platform-detection
- content-paths
- configuration
---

# ANALYSIS-053 Gemini CLI Content Directory Paths

## Observations

- [fact] Gemini CLI binary command name is `gemini`, installed via `npm install -g @google/gemini-cli` or `brew install gemini-cli` #platform-detection
- [fact] Main user config directory is `~/.gemini/` overridable via `GEMINI_CLI_HOME` env var #configuration
- [fact] Project config directory is `.gemini/` at project root #configuration
- [fact] Context files default to `GEMINI.md` but configurable via `context.fileName` in settings.json to read arrays like `["AGENTS.md", "CLAUDE.md", "GEMINI.md"]` #context
- [fact] Gemini CLI does NOT read CLAUDE.md by default; requires explicit `context.fileName` configuration #cross-platform
- [fact] Skills use SKILL.md schema in `.gemini/skills/` (project) and `~/.gemini/skills/` (user) with `.agents/skills/` alias taking precedence #skills
- [fact] Agents (subagents) are markdown files with YAML frontmatter in `.gemini/agents/*.md` (project) and `~/.gemini/agents/*.md` (user), requires `experimental.enableAgents: true` #agents
- [fact] Custom commands are TOML files in `.gemini/commands/` (project) and `~/.gemini/commands/` (user), path separators become colons for namespacing #commands
- [fact] Hooks are defined in `settings.json` under the `hooks` key, not in separate files #hooks
- [fact] MCP servers configured in `settings.json` under `mcpServers` key, same format as other platforms #mcp
- [decision] Extensions use `gemini-extension.json` manifest, installed via `gemini extensions install <github-url>`, stored in `~/.gemini/extensions/` #extensions

## Relations

- extends [[ANALYSIS-039 Cross Platform Mcp And Instruction Formats]]
- relates_to [[agent-plugin spec]]