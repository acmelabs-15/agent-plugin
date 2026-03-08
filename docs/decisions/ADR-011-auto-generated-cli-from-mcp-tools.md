---
title: ADR-011 Auto-Generated CLI from MCP Tools
status: accepted
date: 2026-03-07
type: decision
permalink: decisions/adr-011-auto-generated-cli-from-mcp-tools
tags:
- decision
- cli
- mcp
- auto-generation
- command-grouping
- agent-plugin
---

# ADR-011 Auto-Generated CLI from MCP Tools

## Status

**Accepted**

**Date**: 2026-03-07
**Decision-makers**: Peter Kloss
**Consulted**: Architect agent, Analyst agent, Critic agent, Security agent, Independent Thinker agent, Advisor agent
**Informed**: All project contributors

## Context and Problem Statement

ADR-010 governs the installation lifecycle for `@acmelabs-15/agent-plugin`. This ADR was originally Decision 5 within ADR-010, but was extracted into a standalone ADR based on debate consensus (P0-1: 5 of 6 reviewers agreed on split). The rationale for extraction:

1. CLI generation is a different concern with different stakeholders, change frequency, and risk profile than install lifecycle.
2. 4 of 7 P0 issues on ADR-010 were CLI-specific (command grouping algorithm, MCP server lifecycle, CLI symlink security, Windows distribution).
3. ADR-003 was previously blocked for the same overloading problem (combining too many concerns in one ADR).
4. CLI generation has independent utility and could become a standalone package.

Plugin authors who build MCP servers already define tool schemas with names, descriptions, and typed parameters. Generating a CLI from these schemas avoids duplicating command definitions. ANALYSIS-030 researched the `cli` field design, command grouping, install-time selection, and runtime behavior.

How should the optional `cli` field in plugin.json work, and how are MCP tool schemas translated to CLI commands?

## Decision Drivers

- Schema reuse: MCP tool schemas already contain names, descriptions, and typed parameters needed for CLI generation
- Author experience: zero-config CLI generation reduces plugin authoring friction
- Optional functionality: not all plugins need a CLI (MCP-only plugins, skills-only plugins)
- Consistent interaction: auto-generated CLI follows the same three-tier input resolution as the main CLI (ADR-007 Decision 4)
- Install-time customization: users should control which CLI commands are installed

## Considered Options

- Option A: No CLI generation, MCP-only plugins
- Option B: Optional auto-generated CLI from MCP tool schemas with author override (chosen)
- Option C: Mandatory CLI for all plugins

## Decision Outcome

Chosen option: "Optional auto-generated CLI from MCP tool schemas with author override", because it reuses existing MCP tool schemas to eliminate command definition duplication while keeping CLI generation optional for plugins that do not need it.

### Decision 1: Auto-Generated CLI from MCP Tools

The optional `cli` field in plugin.json supports three modes:

| `cli` Value | Behavior |
|---|---|
| `"auto"` | Auto-generate CLI from MCP tool schemas |
| `"./path/to/binary"` | Symlink author-provided binary |
| Omitted | No CLI installed |

**Auto-generation mode** (`"auto"`):

When `cli` is `"auto"`, the plugin manager reads the MCP server's tool definitions at install time and generates gunshi command definitions. Each MCP tool becomes a CLI subcommand.

**Command mapping**: For v1, auto-generation produces flat subcommands with no auto-grouping heuristic. Underscore-separated MCP tool names are converted to kebab-case.

| MCP Tool Name | CLI Command (v1 flat) |
|---|---|
| `write_note` | `plugin-name write-note` |
| `edit_note` | `plugin-name edit-note` |
| `delete_note` | `plugin-name delete-note` |
| `search` | `plugin-name search` |

**Command grouping** is author-specified only via `cli.groups`:

```json
{
  "cli": {
    "mode": "auto",
    "groups": {
      "note": ["write_note", "edit_note", "delete_note", "read_note"],
      "search": ["search", "list_directory"]
    }
  }
}
```

When `cli.groups` is provided, grouped tools become nested subcommands (`plugin-name note write`, `plugin-name note edit`, `plugin-name search`). Tools not listed in any group become top-level flat commands.

**Validation rules for `cli.groups`**: No duplicate tool assignments across groups. All tool names must match MCP server tools (unknown names rejected at install time). Empty groups rejected.

Auto-grouping heuristics may be added in a future version based on real-world plugin naming data from 10+ plugins.

**Parameter type mapping**:

| MCP Schema Type | CLI Flag Type | Behavior |
|-----------------|---------------|----------|
| `string` | `--flag value` | Direct string |
| `number` / `integer` | `--flag 42` | Parsed as number |
| `boolean` | `--flag` / `--no-flag` | Boolean flag |
| `enum` (via `enum` keyword) | `--flag option` | Validated against allowed values |
| `array` | `--flag a --flag b` | Repeated flag (gunshi `multiple: true`) |
| `object` | `--flag '{"key":"val"}'` | JSON string, parsed by CLI before forwarding to MCP |
| Nested / `anyOf` / `oneOf` | Not supported in v1 | Tool excluded from auto-generation with warning at install time |

**Install-time command selection**: For v1, CLI installation is all-or-nothing: all auto-generated commands are installed. Per-group command selection may be added in a future version based on user demand.

**CLI runtime behavior**:

- Binary symlinked to `~/.local/bin/plugin-name` (XDG-compliant, typically on PATH)
- Symlink tracked in `plugin-lock.json`, removed on uninstall
- Post-install, if `~/.local/bin` is not detected on the user's PATH, emit a `@clack/prompts` warning with shell-specific instructions (e.g., `export PATH="$HOME/.local/bin:$PATH"` for bash/zsh)
- CLI connects to the plugin's MCP server (see Decision 2 for server lifecycle)
- Missing required parameters trigger `@clack/prompts` interactive input at runtime (per ADR-007 Decision 4 three-tier resolution)
- In CI mode (`--ci`), missing required params produce an error with flag hints

**Binary name safety**:

A hardcoded denylist of system-critical binary names prevents plugin CLIs from shadowing them: `git`, `ssh`, `sudo`, `su`, `curl`, `wget`, `node`, `bun`, `deno`, `python`, `python3`, `bash`, `zsh`, `sh`, `brew`, `apt`, `dnf`, `npm`, `npx`, `env`, `which`, `chmod`, `chown`, `rm`, `cp`, `mv`, `ls`, `cat`, `kill`, `docker`, `kubectl`.

Before symlinking, check if `plugin-name` resolves to an existing binary via `which`. If it does, require explicit user confirmation with warning. In CI mode (`--ci`), refuse to shadow existing binaries unless `--allow-shadow` is explicitly passed.

**Custom CLI mode** (`"./path/to/binary"`):

The plugin manager resolves the binary path using `realpathSync()` and verifies the resolved path starts with the plugin's installation directory. Absolute paths, paths containing `..` components, and symlinks resolving outside the plugin directory are rejected with an error at install time. No auto-generation occurs. The validated binary is symlinked to `~/.local/bin/plugin-name` and tracked in `plugin-lock.json`.

**Upgrade preservation**: User's command selections are stored in `plugin-lock.json` and preserved across upgrades. Only newly added commands (from new MCP tools in the updated plugin) are included automatically.

### Decision 2: Built-in MCP Server Lifecycle Commands

The auto-generated CLI includes a built-in `mcp` command group for every plugin with an MCP server. These commands are always present regardless of `cli` mode (auto or custom binary) and are not generated from tool schemas.

| Command | Behavior |
|---------|----------|
| `plugin-name mcp start` | Start MCP server as background daemon (PID file tracked in `~/.local/share/agent-plugin/pids/plugin-name.pid`) |
| `plugin-name mcp stop` | Graceful shutdown via SIGTERM to daemon instance |
| `plugin-name mcp restart` | Graceful restart: start new instance, verify ready, then stop old |
| `plugin-name mcp status` | Show running state, PID, uptime, transport mode |

**Transport model**: The MCP server supports two modes:

- **stdio** (default): Claude Code and other MCP clients spawn the server as a child process via stdio transport. Each client manages its own instance. The CLI's `mcp start/stop/restart` commands do NOT affect stdio instances. They are managed by their parent process.
- **daemon**: The CLI `mcp start` command launches the server as a standalone background process. The daemon uses the same server binary but runs independently of any MCP client. CLI tool commands connect to the daemon if running, or start a per-invocation instance if not.

**Restart safety**: `mcp restart` only affects the daemon instance started via `mcp start`. It never touches stdio instances managed by Claude Code or other MCP clients. This means restarting via CLI cannot break an active Claude Code session.

**PID file management**: Daemon PID tracked at `~/.local/share/agent-plugin/pids/plugin-name.pid`. On `mcp start`, verify no existing daemon is running. On `mcp stop`, send SIGTERM and clean up PID file. Orphan detection: on any CLI invocation, check PID file. If PID exists but process is dead, clean up stale PID file.

**Server lifecycle for CLI tool commands**: When running a tool command (e.g., `plugin-name write-note`), the CLI checks for a running daemon first. If a daemon is running (PID file valid), connect to it. If no daemon, start a per-invocation MCP server via `Bun.spawn` with stdio transport, execute the command, then terminate the server on CLI exit.

**Crash recovery**: If per-invocation server crashes, CLI emits error with stderr from server process. No automatic retry.

**Signal handling**: CLI wrapper propagates SIGINT/SIGTERM to child MCP server process via process group.

**Concurrent invocations**: Multiple CLI invocations can share the daemon. Without daemon, each gets its own per-invocation server.

## Trust Model

Installing a plugin with `cli: "auto"` or `cli: "./path"` grants the plugin author the ability to execute code with the user's permissions when the MCP server is started (either via auto-start or `mcp start`). This is inherent to the MCP server model. MCP servers are not sandboxed. This trust boundary is surfaced during install-time confirmation (ADR-010 Phase 4). Users accept this risk when confirming installation. This is consistent with all current MCP clients (Claude Code, Cursor, Windsurf, etc.).

## Consequences

### Positive

- **POS-001**: Auto-generated CLI reuses MCP tool schemas, eliminating command definition duplication. Authors with MCP servers get a CLI for free.
- **POS-002**: Three-mode `cli` field (auto, path, omitted) covers the full spectrum from zero-config to full custom CLI.
- **POS-003**: Author override via `cli.groups` provides explicit control over command grouping when flat subcommands are insufficient.
- **POS-004**: Built-in `mcp` command group provides daemon lifecycle management for all plugins with MCP servers.

### Negative

- **NEG-001**: v1 auto-generated CLI produces flat subcommands without grouping. Authors who want grouped commands must specify `cli.groups` explicitly. This is intentionally conservative. Heuristic grouping is deferred until real-world plugin naming patterns are observed.
- **NEG-002**: Per-invocation MCP server startup adds cold-start latency (~1-3s depending on server complexity). Users can eliminate this latency by running `plugin-name mcp start` to start a persistent daemon.
- **NEG-003**: ADR-011 applies to Unix-like systems (macOS, Linux) for v1. Windows CLI distribution is out of scope for v1 and will be addressed in a future ADR.
- **NEG-004**: MCP servers execute with full user permissions. No sandboxing or capability restriction is applied. This is an inherent property of the MCP server model shared by all current MCP clients, but means plugin authors are trusted with the same access level as any locally installed software.

## Prior Art

| Tool | Approach | Why Not Sufficient |
|------|----------|--------------------|
| MCPShim | Daemon + CLI, auto-discovers tools | Generic caller, no per-plugin binary with plugin-specific command tree |
| mcp-cli | Bun-based `call server tool` pattern | Requires knowing server name and tool name. No standalone binary |
| FastMCP CLI | Python, auto-installs with FastMCP | Tied to FastMCP framework (Python only) |
| mcptools | Go, shell mode + proxy | Inspector/debugger, not end-user CLI |

agent-plugin's CLI generation differs from these tools by producing a standalone per-plugin binary with plugin-specific command names. Users run `brain write-note` not `mcp-cli call brain write_note`. This is a developer experience decision: plugin CLIs should feel like native tools, not generic MCP callers.

## Alternatives Considered

### No CLI Generation, MCP-Only (Rejected)

- Good, because simpler implementation
- Bad, because MCP servers require an MCP client to invoke. Users without an AI coding platform cannot use the plugin's functionality.
- Bad, because CLI access to MCP tools is a common developer workflow (testing, scripting, debugging)

### Mandatory CLI for All Plugins (Rejected)

- Bad, because forces CLI generation for plugins that are MCP-only by design
- Bad, because some plugins provide only skills/agents/prompts with no MCP server

## Implementation Notes

- **IMP-001**: CLI binary generation uses gunshi's lazy command loading to keep startup fast. Each command group is a separate module loaded on demand.
- **IMP-002**: MCP server lifecycle follows Decision 2. Per-invocation server uses `Bun.spawn` with stdio transport, started on CLI invocation and terminated on CLI exit. Daemon mode uses PID file tracking at `~/.local/share/agent-plugin/pids/plugin-name.pid`.
- **IMP-003**: ADR-011 applies to Unix-like systems (macOS, Linux) for v1. Windows CLI distribution is out of scope for v1 and will be addressed in a future ADR.
- **IMP-004**: Server startup timeout: 10s default, configurable via plugin.json.
- **IMP-005**: Orphaned process detection via stale PID file cleanup. On any CLI invocation, check PID file. If PID exists but process is dead, clean up stale PID file.
- **IMP-006**: Signal propagation: SIGINT/SIGTERM forwarded to child server process via process group.
- **IMP-007**: Binary name denylist of system-critical binaries checked before symlinking. `which`-based detection for non-denylisted collisions with user confirmation. CI mode refuses shadow unless `--allow-shadow` passed.
- **IMP-008**: Path containment validation using `realpathSync()` boundary check for custom CLI binary paths. Rejects absolute paths, `..` components, and symlinks resolving outside plugin directory.
- **IMP-009**: Tools with unsupported parameter types (nested objects, `anyOf`/`oneOf` combinators) are excluded from auto-generation and logged as a warning at install time. Authors can use custom CLI mode for these tools.
- **IMP-010**: Post-install PATH detection. If `~/.local/bin` is not on the user's PATH, emit a `@clack/prompts` warning with shell-specific instructions.

## Confirmation

Implementation compliance will be verified through:

- [ ] Auto-generated CLI produces flat subcommands from MCP tool schemas
- [ ] Author override via `cli.groups` produces expected nested command structure
- [ ] Validation rejects duplicate tool assignments, unknown tool names, and empty groups
- [ ] `mcp start/stop/restart/status` commands manage daemon lifecycle correctly
- [ ] Daemon PID file tracking and orphan detection works correctly
- [ ] CLI connects to running daemon or starts per-invocation server
- [ ] `mcp restart` does not affect stdio instances managed by MCP clients
- [ ] Binary name denylist prevents shadowing system-critical binaries
- [ ] `which`-based collision detection prompts user confirmation
- [ ] Custom CLI binary path containment validated via `realpathSync()`
- [ ] Parameter type mapping covers string, number, boolean, enum, array, object
- [ ] Tools with unsupported parameter types excluded with install-time warning
- [ ] CI mode produces error with flag hints for missing required params
- [ ] CI mode refuses binary shadowing unless `--allow-shadow` passed
- [ ] Post-install PATH detection warns if `~/.local/bin` not on PATH
- [ ] Trust boundary surfaced during install-time confirmation

## Reversibility Assessment

- [x] **Rollback capability**: CLI generation can be disabled per plugin by removing the `cli` field from plugin.json. Existing symlinks are removed on uninstall.
- [x] **Vendor lock-in**: No new vendor dependencies. gunshi (already adopted in ADR-006) generates commands. @clack/prompts (already adopted in ADR-006) provides interactive input.
- [x] **Exit strategy**: If auto-generated CLI proves insufficient, authors switch to custom CLI mode (`"./path/to/binary"`). If CLI generation is dropped entirely, plugins continue to work as MCP-only.
- [x] **Legacy impact**: Greenfield project. No existing CLI installations to migrate.
- [x] **Data migration**: Command selections in `plugin-lock.json` are a subset of lockfile data. Lockfile schema changes use integer `lockfileVersion` migration pipeline (ADR-003 Decision 3).

## References

- **REF-001**: ANALYSIS-030 Auto-Generated CLI from MCP Tools (cli field modes, flat command mapping, install-time selection)
- **REF-002**: ADR-010 Installation Lifecycle (install flow phases, lockfile tracking, upgrade mechanics)
- **REF-003**: ADR-007 CLI Architecture and Interaction Model (command tree, three-tier input resolution, @clack/prompts mapping)
- **REF-004**: ADR-006 Core Dependency Stack (gunshi, @clack/prompts adoption)
- **REF-005**: DEBATE-ADR-010 Installation Lifecycle and CLI Generation (P0-1 consensus for extraction)
- **REF-006**: DEBATE-ADR-011 Auto-Generated CLI Debate (P0 issue resolutions for grouping, lifecycle, security, trust, type mapping)

## Observations

- [decision] Optional `cli` field in plugin.json with three modes: "auto" (generate from MCP schemas), path string (custom binary), or omitted (no CLI) #cli #manifest
- [decision] v1 auto-generated CLI produces flat subcommands without auto-grouping heuristic. Author override via `cli.groups` is the only grouping mechanism #cli #auto-generation
- [decision] Built-in `mcp` command group (start/stop/restart/status) for daemon lifecycle management, always present regardless of cli mode #cli #mcp-lifecycle
- [decision] Daemon and stdio transport modes are independent. `mcp restart` never touches stdio instances managed by MCP clients #mcp #safety
- [technique] Parameter type mapping from MCP schema types to CLI flag types. Unsupported types (nested, anyOf, oneOf) exclude the tool with install-time warning #cli #type-mapping
- [fact] Binary name denylist prevents shadowing system-critical binaries. `which`-based collision detection requires user confirmation #cli #security
- [fact] Custom CLI binary path containment via `realpathSync()` boundary check. Rejects paths resolving outside plugin directory #cli #security
- [risk] MCP servers execute with full user permissions. No sandboxing applied. Consistent with all current MCP clients but plugin authors are fully trusted #trust #security
- [risk] Per-invocation MCP server startup adds ~1-3s cold-start latency. Daemon mode eliminates this #cli #performance
- [insight] Prior art tools (MCPShim, mcp-cli, FastMCP CLI, mcptools) are generic callers. agent-plugin produces standalone per-plugin binaries that feel like native tools #cli #dx
- [decision] v1 scoped to Unix-like systems (macOS, Linux). Windows CLI distribution deferred to future ADR #cli #scope
- [decision] Extracted from ADR-010 Decision 5 per debate P0-1 consensus: 5 of 6 reviewers agreed CLI generation is an independent concern #architecture #extraction

## Relations

- depends_on [[ADR-010 Installation Lifecycle]]
- depends_on [[ADR-007 CLI Architecture and Interaction Model]]
- relates_to [[ANALYSIS-030-auto-generated-cli-from-mcp-tools]]
- relates_to [[ADR-006 Core Dependency Stack]]
- relates_to [[DEBATE-ADR-010 Installation Lifecycle and CLI Generation]]
- relates_to [[DEBATE-ADR-011 Auto-Generated CLI Debate]]
