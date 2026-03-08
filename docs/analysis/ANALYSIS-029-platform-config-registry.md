---
title: ANALYSIS-029 Platform Config Registry
type: analysis
permalink: analysis/analysis-029-platform-config-registry
tags:
- platform-config
- registry
- platforms-config-json
- cross-platform
- agent-plugin
---

# ANALYSIS-029 Platform Config Registry

## Context

During Group 4 discussion (Source and Platform, Sections 7-9), the team decided that all platform-specific mapping should live in a single static data file rather than being embedded in plugin.json or scattered across code. This separates platform knowledge from plugin authoring and from runtime logic.

The original ANALYSIS-027 proposed platform adapters in code. The discussion refined this: the adapter logic reads from a data file (`platforms.config.json`) so adding a new platform requires zero code changes.

## Design

### platforms.config.json

A static JSON data file at the agent-plugin project root. Contains all platform-specific knowledge:

- **Config paths per OS**: where each platform stores its MCP config file (macOS, Linux, Windows)
- **Binary names**: CLI binary names for platform detection (e.g., `cursor`, `code`, `windsurf`)
- **MCP config locations**: project-level and user-level config file paths for each platform
- **Content directory mappings**: where each platform expects skills, agents, prompts to be placed
- **Detection methods**: binary name + config directory path for dual-detection strategy
- **Root key differences**: `mcpServers` vs `mcp` vs `amp.mcpServers` (3 variants across 7 platforms)

### Plugin Author Experience

Plugin authors write platform-agnostic `plugin.json`. No `platforms` block is needed or allowed (removed per ADR-001 decision). The plugin author declares skills, agents, prompts, hooks, and MCP servers without knowing platform-specific paths.

### Install-Time Resolution

At install time, agent-plugin reads both files:

1. `plugin.json` -- what the plugin provides (platform-agnostic)
2. `platforms.config.json` -- where and how to write content per platform

The install flow (6-phase: Detect, Select, Resolve, Confirm, Apply, Record) uses platforms.config.json during the Detect phase (which platforms are installed) and the Apply phase (where to write files and config entries).

### MCP Key Namespacing

MCP server entries use colon separator: `plugin-name:server-name`. This matches Claude Code's internal convention (`plugin:<name>:<server>`), avoids JSON Pointer (RFC 6901) escaping conflicts with slash, and aligns with ADR-003's colon convention for component identifiers. More readable than the double-dash format (`agentplugin--plugin-name--server-name`) proposed in ANALYSIS-027.

### Adding a New Platform

Adding support for a new platform requires only adding an entry to `platforms.config.json`. No code changes, no new adapter functions, no new module. The entry specifies:

- Platform name and display label
- Binary name for detection
- Config directory path per OS for detection
- MCP config file paths (project and user scope)
- MCP root key (`mcpServers`, `servers`, etc.)
- Content directories for skills, agents, prompts

## Observations

- [decision] All platform-specific mapping consolidated into `platforms.config.json` at project root as a pure static JSON data file #architecture #data-driven
- [decision] Plugin authors write platform-agnostic plugin.json with no platforms block. Agent-plugin reads both files at install time. #authoring #simplification
- [decision] MCP key namespacing uses colon separator (`plugin-name:server-name`), matching Claude Code internal convention and ADR-003 component identifier pattern #mcp #namespacing
- [decision] Adding a new platform = adding a JSON entry, zero code changes required #extensibility #data-driven
- [fact] 7 target platforms use 3 different JSON root keys for MCP server config: mcpServers (5), mcp (1: OpenCode), amp.mcpServers (1: Amp). All captured in platforms.config.json. #platform-diversity
- [fact] platforms.config.json contains: config paths per OS, binary names, MCP config locations, content directory mappings, detection methods, root key differences #registry-contents
- [insight] Separating platform knowledge from code enables community-contributed platform support without pull requests to runtime logic #extensibility

## Relations

- extends [[ANALYSIS-027-installation-mechanics]]
- extends [[ANALYSIS-026-platform-detection-and-mapping]]
- relates_to [[ANALYSIS-014-platform-config-patterns]]
- relates_to [[ADR-002-target-platforms-and-audiences]]
- relates_to [[ADR-003-conflict-resolution-and-namespacing]]

- [decision] Maintenance strategy resolved: platforms.config.json is bundled inside the @acmelabs-15/agent-plugin npm package and ships with every release. Platform path changes (e.g., Kiro migrating from .amazonq/ to .kiro/) are published as new package versions. Users receive updates through normal npm/bun update flows. No remote fetching or dynamic update mechanism needed. #maintenance #npm-distribution #p0-resolution

- [fact] envOverrides field added to platforms.config.json schema: lists env vars to check for non-standard binary/config paths per platform (OpenCode, Amp, XDG-compliant platforms) #env-overrides #schema
- [fact] OpenCode uses non-standard MCP format (command array, environment key). formatTransformer field in platforms.config.json dispatches to named transformer function in runtime code. #opencode #format-divergence
- [fact] Claude Code supports both project-level (.mcp.json) and user-level (~/.claude.json) MCP configuration. Both paths captured in platforms.config.json. #claude-code #mcp-scope
