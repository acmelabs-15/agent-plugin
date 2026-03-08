---
title: ANALYSIS-030 Auto-Generated CLI from MCP Tools
type: analysis
permalink: analysis/analysis-030-auto-generated-cli-from-mcp-tools
tags:
- cli
- mcp
- auto-generation
- clack-prompts
- agent-plugin
---

# ANALYSIS-030 Auto-Generated CLI from MCP Tools

## Context

Plugin authors who build MCP servers already define tool schemas with names, descriptions, and typed parameters. Generating a CLI from these schemas avoids duplicating command definitions. This analysis covers the optional `cli` field in plugin.json and how auto-generation works.

## The `cli` Field in plugin.json

Three modes:

1. **`"auto"`** -- auto-generates a CLI from the plugin's MCP tool schemas. No author code required.
2. **`"./path/to/binary"`** -- author provides a custom CLI binary. Plugin manager symlinks it.
3. **Omitted** -- no CLI generated. Plugin operates as MCP-only.

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "cli": "auto"
}
```

## Auto-Generation from MCP Tool Schemas

When `cli` is `"auto"`, the plugin manager reads the MCP server's tool definitions at install time and generates gunshi command definitions. Each MCP tool becomes a CLI subcommand.

### Command Grouping via Prefix Convention

MCP tools sharing a common prefix (separated by underscores) become subcommands under a group:

| MCP Tool Name | CLI Command |
|---------------|-------------|
| `write_note` | `plugin-name memory write` |
| `edit_note` | `plugin-name memory edit` |
| `delete_note` | `plugin-name memory delete` |
| `search` | `plugin-name search` |
| `list_directory` | `plugin-name list-directory` |

Grouping rules:

- Tools with shared prefix (e.g., `write_note`, `edit_note`, `delete_note` all share the implicit "note" concept) are grouped by the plugin manager's prefix detection
- Single-word tools become top-level commands
- Author can override via `cli.groups` in plugin.json

### Author Override for Grouping

```json
{
  "cli": {
    "mode": "auto",
    "groups": {
      "memory": ["write_note", "edit_note", "delete_note", "read_note"],
      "search": ["search", "list_directory"]
    }
  }
}
```

This gives authors full control over how tools map to CLI command groups without writing CLI code.

## Install-Time Command Selection

At install time, the plugin manager presents a @clack/prompts multiselect showing all detected command groups. All groups are selected by default. Users can toggle off groups they do not need.

```
◆ Select command groups to install
│ ◻ memory (write, edit, delete, read)
│ ◻ search (search, list-directory)
│ ◻ config (get, set, reset)
└
```

User selections are stored in plugin-lock.json. On upgrade, existing selections are preserved -- only newly added groups prompt the user.

## CLI Runtime Behavior

- CLI binary is symlinked to `~/.local/bin/plugin-name` (XDG-compliant, typically on PATH)
- Symlink tracked in plugin-lock.json for clean uninstall
- CLI connects to the plugin's running MCP server via stdio
- If MCP server is not running, CLI auto-starts it before executing the command
- Missing required parameters trigger @clack/prompts interactive input (p.text, p.select as appropriate based on schema type)
- In CI mode (--ci flag), missing required params produce an error with flag hints instead of interactive prompts

## Custom CLI Alternative

When `cli` is a path string (e.g., `"./bin/my-cli"`), the plugin manager:

1. Validates the binary exists at the specified path within the plugin
2. Symlinks it to `~/.local/bin/plugin-name`
3. Tracks symlink in plugin-lock.json
4. Does not auto-generate any commands

This supports authors who need CLI behavior beyond what auto-generation provides.

## Observations

- [decision] Optional `cli` field in plugin.json with three modes: "auto" (generate from MCP schemas), path string (custom binary), or omitted (no CLI) #cli #manifest
- [technique] Auto-generation infers command grouping from MCP tool name prefixes using underscore convention, reducing author boilerplate to zero for standard cases #auto-generation #dx
- [decision] Install-time @clack/prompts multiselect lets users pick which command groups to install, all selected by default. Selections persisted in plugin-lock.json and preserved across upgrades #ux #install
- [fact] CLI binary symlinked to ~/.local/bin/plugin-name (XDG-compliant). MCP server auto-started if not running when CLI invoked. Missing required params trigger interactive @clack/prompts input #runtime #integration
- [technique] Author can override auto-grouping via `cli.groups` object in plugin.json, mapping group names to arrays of MCP tool names #override #flexibility
- [decision] Custom CLI via path string alternative for authors needing behavior beyond auto-generation. Same symlink and tracking mechanics apply #custom-cli
- [insight] MCP tool schemas already contain names, descriptions, and typed parameters -- all the metadata needed to generate CLI commands without duplication #schema-reuse

## Relations

- extends [[ANALYSIS-027-installation-mechanics]] (CLI symlinks use same install/uninstall tracking)
- extends [[ANALYSIS-029-platform-config-registry]] (CLI generation reads MCP tool schemas from platform config)
- implements [[ADR-003-conflict-resolution-and-namespacing]] (CLI binary namespaced as plugin-name in ~/.local/bin)
- relates_to [[ADR-007-cli-architecture-and-interaction-model]] (auto-generated CLI follows same three-tier input resolution and global flags)
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]] (Decision 11 in Group 4 discussion)

- [decision] CLI generation decisions now codified in ADR-011 (extracted from ADR-010 Decision 5) per debate P0-1 consensus #adr-011 #extraction
- relates_to [[ADR-011 Auto-Generated CLI from MCP Tools]]
