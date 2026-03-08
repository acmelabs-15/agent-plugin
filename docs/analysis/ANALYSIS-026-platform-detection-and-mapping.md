---
title: ANALYSIS-026 Platform Detection and Mapping
type: analysis
permalink: analysis/analysis-026-platform-detection-and-mapping-1
tags:
- analysis
- platform-detection
- cross-platform
- config-paths
- binary-detection
- installation
- agent-plugin
---

# ANALYSIS-026 Platform Detection and Mapping

## 1. Objective and Scope

**Objective**: How should the agent-plugin CLI detect which of the 7 target platforms are installed, and where should it write plugin content for each platform on each OS?

**Scope**: 7 target platforms (Claude Code, Cursor, GitHub Copilot CLI, Kiro, OpenCode, Amp, Windsurf). Three operating systems (macOS, Linux, Windows). Detection methods, config directory paths, content subdirectory mapping, and MCP configuration paths.

## 2. Context

The agent-plugin CLI must detect installed platforms at runtime and install plugin content to the correct locations. Each platform uses different directory structures, config file formats, and extension mechanisms. Prior analysis (ANALYSIS-002, ANALYSIS-008) established the capability matrix and instruction file paths. This analysis adds the detection layer and maps config/settings paths that ANALYSIS-008 did not cover.

ADR-005 selected Bun as the runtime, so detection uses `Bun.spawn` for binary checks.

## 3. Approach

**Methodology**: Web research against official documentation, GitHub repos, install scripts, community forums, and package registries for all 7 platforms. Cross-referenced binary names, config paths, and install methods across macOS, Linux, and Windows.

**Tools Used**: WebSearch (25+ queries), Brain memory search (ANALYSIS-002, ANALYSIS-008, DEBATE-ADR-002).

**Limitations**: Some Windows-specific paths for Kiro IDE and Amp are not fully documented publicly. Enterprise/managed deployment paths are excluded. Paths evolve as platforms release updates.

## 4. Data and Analysis

### 4.1 Platform Detection Methods

For each platform, detection should use a two-step approach: (1) binary check via `which`/`where`, and (2) config directory existence check. Both signals together provide high confidence.

#### Detection Summary Table

| Platform | Binary Name(s) | Config Dir Check (macOS/Linux) | Config Dir Check (Windows) | Detection Confidence |
|---|---|---|---|---|
| Claude Code | `claude` | `~/.claude/` | `%USERPROFILE%\.claude\` | High (binary + dir) |
| Cursor | `cursor` | `~/.config/Cursor/` (Linux), `~/Library/Application Support/Cursor/` (macOS) | `%APPDATA%\Cursor\` | High (binary + dir) |
| GitHub Copilot CLI | `copilot`, `gh copilot` | `~/.copilot/` | `%USERPROFILE%\.copilot\` | High (binary + dir) |
| Kiro (CLI) | `kiro-cli` | `~/.kiro/` | `%USERPROFILE%\.kiro\` | High (binary + dir) |
| OpenCode | `opencode` | `~/.config/opencode/` | `%USERPROFILE%\.config\opencode\` | High (binary + dir) |
| Amp | `amp` | `~/.config/amp/` | `$XDG_CONFIG_HOME/amp/` or `~/.config/amp/` | High (binary + dir) |
| Windsurf | `windsurf` | `~/.codeium/windsurf/` | `%USERPROFILE%\.codeium\windsurf\` | High (binary + dir) |

### 4.2 Binary Detection Details

#### Claude Code

| Property | Value |
|---|---|
| Binary name | `claude` |
| npm package | `@anthropic-ai/claude-code` |
| Native install path (macOS/Linux) | `~/.local/bin/claude` (symlink to `~/.local/share/claude/versions/<ver>`) |
| Homebrew path (macOS) | `/opt/homebrew/bin/claude` (ARM), `/usr/local/bin/claude` (Intel) |
| npm global path | Varies by system |
| Windows | `%USERPROFILE%\.local\bin\claude` or npm global |

#### Cursor

| Property | Value |
|---|---|
| Binary name | `cursor` |
| macOS path | `/usr/local/bin/cursor` (added via Command Palette: "Install 'cursor' command") |
| Linux path | `~/.local/bin/cursor` |
| Windows path | Added to PATH during install; `%LOCALAPPDATA%\Programs\cursor\cursor.exe` |
| Note | Shell command must be explicitly installed by user via Command Palette |

#### GitHub Copilot CLI

| Property | Value |
|---|---|
| Binary name | `copilot` (standalone) |
| Alternative | `gh copilot` (via GitHub CLI extension) |
| npm package | `@github/copilot` |
| Homebrew | `brew install github/copilot/copilot` |
| WinGet | `winget install GitHub.CopilotCLI` |
| Install script | Custom PREFIX to `$PREFIX/bin/` |

#### Kiro

| Property | Value |
|---|---|
| CLI binary name | `kiro-cli` (not `kiro`) |
| IDE binary name | `kiro` (desktop app, not typically in PATH) |
| Homebrew | `brew install --cask kiro-cli` |
| Linux install | `curl -fsSL https://cli.kiro.dev/install \| bash` |
| AppImage | Portable format for Linux |
| Prerequisite | glibc 2.34+ |

#### OpenCode

| Property | Value |
|---|---|
| Binary name | `opencode` |
| npm package | `opencode-ai` |
| Homebrew | `brew install opencode` |
| Default install dir | `$HOME/.opencode/bin` (install script) |
| Env overrides | `OPENCODE_INSTALL_DIR`, `XDG_BIN_DIR` |
| Binary override | `OPENCODE_BIN_PATH` env var |

#### Amp

| Property | Value |
|---|---|
| Binary name | `amp` |
| npm package | `@sourcegraph/amp` |
| pnpm (recommended) | `pnpm add -g @sourcegraph/amp` |
| Homebrew | Also available |
| IDE mode | `amp --ide` (WebSocket-based editor integration) |

#### Windsurf

| Property | Value |
|---|---|
| Binary name | `windsurf` |
| macOS path | `/usr/local/bin/windsurf` (installed via Command Palette) |
| macOS app binary | `/Applications/Windsurf.app/Contents/Resources/app/bin/windsurf` |
| Linux | Installed via package manager or .deb/.rpm |
| Windows | `%LOCALAPPDATA%\Programs\Windsurf\windsurf.exe` |

### 4.3 Global Config Directory Paths Per Platform

#### macOS

| Platform | Global Config Directory | Settings File |
|---|---|---|
| Claude Code | `~/.claude/` | `~/.claude/settings.json` |
| Cursor | `~/Library/Application Support/Cursor/User/` | `settings.json` in that dir |
| Copilot CLI | `~/.copilot/` | `~/.copilot/config.json` |
| Kiro | `~/.kiro/` | `~/.kiro/settings/mcp.json` |
| OpenCode | `~/.config/opencode/` | `~/.config/opencode/opencode.json` |
| Amp | `~/.config/amp/` | `~/.config/amp/settings.json` |
| Windsurf | `~/.codeium/windsurf/` | `~/.codeium/windsurf/mcp_config.json` |

#### Linux

| Platform | Global Config Directory | Settings File |
|---|---|---|
| Claude Code | `~/.claude/` | `~/.claude/settings.json` |
| Cursor | `~/.config/Cursor/User/` | `settings.json` in that dir |
| Copilot CLI | `~/.copilot/` | `~/.copilot/config.json` |
| Kiro | `~/.kiro/` | `~/.kiro/settings/mcp.json` |
| OpenCode | `~/.config/opencode/` | `~/.config/opencode/opencode.json` |
| Amp | `~/.config/amp/` | `~/.config/amp/settings.json` |
| Windsurf | `~/.codeium/windsurf/` | `~/.codeium/windsurf/mcp_config.json` |

#### Windows

| Platform | Global Config Directory | Settings File |
|---|---|---|
| Claude Code | `%USERPROFILE%\.claude\` | `settings.json` |
| Cursor | `%APPDATA%\Cursor\User\` | `settings.json` |
| Copilot CLI | `%USERPROFILE%\.copilot\` | `config.json` |
| Kiro | `%USERPROFILE%\.kiro\` | `settings\mcp.json` |
| OpenCode | `%USERPROFILE%\.config\opencode\` | `opencode.jsonc` |
| Amp | `%USERPROFILE%\.config\amp\` (or `$XDG_CONFIG_HOME/amp/`) | `settings.json` |
| Windsurf | `%USERPROFILE%\.codeium\windsurf\` | `mcp_config.json` |

### 4.4 MCP Configuration Paths

MCP server configuration is the most divergent across platforms. Every platform uses a different file path and format.

| Platform | MCP Config Path (Project) | MCP Config Path (Global) | Format |
|---|---|---|---|
| Claude Code | `.mcp.json` (project root) | N/A (project-level only) | JSON: `{ "mcpServers": { "name": { "command": "...", "args": [...] } } }` |
| Cursor | `.cursor/mcp.json` | N/A (project-level only) | JSON: `{ "mcpServers": { ... } }` |
| Copilot CLI | `.copilot/settings.json` (project) | `~/.copilot/mcp-config.json` | JSON: MCP config format |
| Kiro | `.kiro/settings/mcp.json` | `~/.kiro/settings/mcp.json` | JSON: `{ "mcpServers": { ... } }` |
| OpenCode | `opencode.json` (mcp section) | `~/.config/opencode/opencode.json` | JSON: inline in main config |
| Amp | `.amp/settings.json` (mcpServers key) | `~/.config/amp/settings.json` | JSON: `{ "amp.mcpServers": { ... } }` |
| Windsurf | N/A (global only) | `~/.codeium/windsurf/mcp_config.json` | JSON: `{ "mcpServers": { ... } }` |

### 4.5 Project-Level Content Directories

This table maps where each content type should be written per platform. Data sourced from ANALYSIS-008 with additions.

| Content Type | Claude Code | Cursor | Copilot CLI | Kiro | OpenCode | Amp | Windsurf |
|---|---|---|---|---|---|---|---|
| **Rules/Instructions** | `CLAUDE.md` | `.cursor/rules/<name>.mdc` | `.github/copilot-instructions.md` | `.kiro/steering/<name>.md` | `AGENTS.md` | `AGENTS.md` | `.windsurf/rules/<name>.md` |
| **Skills** | `.claude/skills/<name>/SKILL.md` | `.cursor/skills/<name>/SKILL.md` | `.github/skills/<name>/SKILL.md` | `.kiro/skills/<name>/SKILL.md` | `.opencode/skills/<name>/SKILL.md` | `.agents/skills/<name>/SKILL.md` | `.windsurf/skills/<name>/SKILL.md` |
| **Agents** | `.claude/agents/<name>.md` | `.cursor/agents/<name>.md` | `.github/agents/<name>.agent.md` | `.kiro/agents/<name>.json` | `.opencode/agents/<name>.md` | N/A (Task tool) | N/A (no file-based) |
| **Commands** | `.claude/commands/<name>.md` | N/A (via rules) | N/A (via agents) | N/A (via steering) | `.opencode/commands/<name>.md` | `.agents/commands/<name>.md` | N/A (via rules) |
| **Hooks** | `.claude/settings.json` | Built into skills | `.github/hooks/*.json` | Agent JSON config | Plugin-based | Settings-based | Cascade settings |
| **MCP** | `.mcp.json` | `.cursor/mcp.json` | `~/.copilot/mcp-config.json` | `.kiro/settings/mcp.json` | `opencode.json` | `.amp/settings.json` | `~/.codeium/windsurf/mcp_config.json` |

### 4.6 Cross-Platform Compatibility Paths

Several paths are read by multiple platforms (from ANALYSIS-008):

| Path | Read By (count) |
|---|---|
| `AGENTS.md` (root) | Cursor, Copilot CLI, Kiro, OpenCode, Amp, Windsurf (6/7) |
| `CLAUDE.md` (root) | Claude Code, Copilot CLI, OpenCode, Amp (4/7) |
| `.claude/skills/<name>/SKILL.md` | Claude Code, Cursor, Copilot CLI, OpenCode, Amp, Windsurf (6/7) |
| `.agents/skills/<name>/SKILL.md` | Cursor, OpenCode, Amp, Windsurf (4/7) |

### 4.7 Platform Capability Support Matrix

Cross-referenced with ANALYSIS-002 to confirm which content types each platform supports:

| Platform | Rules | Skills | Agents | Commands | Hooks | MCPs |
|---|---|---|---|---|---|---|
| Claude Code | Yes | Yes | Yes | Yes | Yes | Yes |
| Cursor | Yes | Yes | Yes | Partial | Yes | Yes |
| Copilot CLI | Yes | Yes | Yes | Partial | Yes | Yes |
| Kiro | Yes | Yes | Yes (JSON) | Partial | Yes | Yes |
| OpenCode | Yes | Yes | Yes | Yes | Yes | Yes |
| Amp | Yes | Yes | Partial | Deprecated | Yes | Yes |
| Windsurf | Yes | Yes | Partial | Partial | Yes | Yes |

### 4.8 Parallel Detection Implementation

Using Bun's native APIs, detection of all 7 platforms can run in parallel with timeouts.

**Recommended pattern using `Bun.spawn` + `Promise.allSettled`**:

```typescript
// Pseudocode for parallel platform detection
interface DetectionResult {
  platform: string;
  detected: boolean;
  binaryPath: string | null;
  configDir: string | null;
  method: "binary" | "directory" | "both";
}

async function detectPlatform(
  binaryName: string,
  configDirs: string[]
): Promise<DetectionResult> {
  // 1. Binary check with Bun.spawn + timeout
  const whichCmd = process.platform === "win32" ? "where" : "which";
  const proc = Bun.spawn([whichCmd, binaryName], {
    timeout: 3_000, // 3-second timeout per ADR spec
  });
  const binaryFound = (await proc.exited) === 0;

  // 2. Config directory existence check
  const dirFound = configDirs.some(
    (dir) => fs.existsSync(expandPath(dir))
  );

  return {
    platform: binaryName,
    detected: binaryFound || dirFound,
    binaryPath: binaryFound ? (await new Response(proc.stdout).text()).trim() : null,
    configDir: dirFound ? configDirs.find(d => fs.existsSync(expandPath(d))) ?? null : null,
    method: binaryFound && dirFound ? "both" : binaryFound ? "binary" : "directory",
  };
}

// Run all 7 detections in parallel
const results = await Promise.allSettled([
  detectPlatform("claude", ["~/.claude"]),
  detectPlatform("cursor", ["~/.config/Cursor", "~/Library/Application Support/Cursor"]),
  detectPlatform("copilot", ["~/.copilot"]),
  detectPlatform("kiro-cli", ["~/.kiro"]),
  detectPlatform("opencode", ["~/.config/opencode"]),
  detectPlatform("amp", ["~/.config/amp"]),
  detectPlatform("windsurf", ["~/.codeium/windsurf"]),
]);
```

**Key implementation details**:

1. `Bun.spawn` supports a native `timeout` option (milliseconds). Timed-out processes receive SIGTERM by default.
2. `Promise.allSettled` returns all results regardless of individual failures. One platform timing out does not block others.
3. On Windows, use `where` instead of `which` for binary detection.
4. Config directory check provides a fallback when the binary is not in PATH (common for GUI-installed editors like Cursor, Windsurf).
5. `AbortController` can be used as an alternative to the timeout option for finer-grained control.

**Timeout rationale**: 3 seconds per binary check is generous. `which`/`where` commands complete in under 50ms on typical systems. The timeout guards against edge cases like network-mounted home directories or stale PATH entries.

### 4.9 Config File Safe Merge Strategy

MCP configuration requires merging into existing JSON files without losing user customizations. Recommended approach:

1. **Read** existing config file (if it exists)
2. **Parse** as JSON
3. **Deep merge** plugin's MCP server entries under the `mcpServers` key
4. **Namespace** all entries under plugin name (per ADR-003 always-namespace): `"pluginName/serverName": { ... }`
5. **Write** back with original formatting preserved (detect indent style)
6. **Track** installed entries in a manifest for clean uninstall

Collision risk is minimal because ADR-003 mandates namespacing. A plugin named `my-plugin` writes MCP servers as `my-plugin/server-name`, which cannot conflict with user-defined servers.

## 5. Results

### Evidence Table

| Finding | Source | Confidence |
|---|---|---|
| Claude Code binary is `claude`, config at `~/.claude/` | Official docs (code.claude.com), GitHub issues | High |
| Cursor binary is `cursor`, config at `~/Library/Application Support/Cursor/` (macOS) | Official docs (cursor.com), community forum | High |
| Copilot CLI binary is `copilot`, config at `~/.copilot/` | GitHub docs, changelog | High |
| Kiro CLI binary is `kiro-cli` (not `kiro`), config at `~/.kiro/` | Official docs (kiro.dev), Homebrew formulae | High |
| OpenCode binary is `opencode`, config at `~/.config/opencode/` | Official docs (opencode.ai), GitHub repo | High |
| Amp binary is `amp`, config at `~/.config/amp/` | npm package, official manual | High |
| Windsurf binary is `windsurf`, config at `~/.codeium/windsurf/` | Official docs (docs.windsurf.com) | High |
| `Bun.spawn` supports native `timeout` option | Bun API reference (bun.com/reference) | High |
| `Promise.allSettled` returns all results regardless of failures | JavaScript standard | High |
| `which` (Unix) / `where` (Windows) detects binary in PATH | OS standard | High |
| Windsurf Windows config path at `%USERPROFILE%\.codeium\` | Windsurf docs, community guides | Medium |
| Amp Windows config path at `%USERPROFILE%\.config\amp\` | Amp manual, XDG convention | Medium |
| Kiro Windows config path at `%USERPROFILE%\.kiro\` | GitHub issues, install docs | Medium |

### Platform Detection Reliability Assessment

| Platform | Binary in PATH likelihood | Config Dir Exists likelihood | Overall Detection Reliability |
|---|---|---|---|
| Claude Code | High (CLI-first tool) | High (always created on first run) | High |
| Cursor | Medium (requires manual PATH install) | High (created on install) | High (dir fallback covers) |
| Copilot CLI | High (CLI-first, Homebrew/npm/WinGet) | High (created on first run) | High |
| Kiro CLI | High (CLI install adds to PATH) | High (migrated from .amazonq) | High |
| OpenCode | High (CLI-first, Homebrew/npm) | High (XDG standard location) | High |
| Amp | High (npm global install) | High (XDG standard location) | High |
| Windsurf | Medium (requires manual PATH install) | High (created on install) | High (dir fallback covers) |

## 6. Discussion

### Detection Strategy: Binary + Directory is Required

Binary-only detection fails for GUI-installed editors (Cursor, Windsurf, Kiro IDE) where the user may not have added the shell command to PATH. Directory-only detection may produce false positives if config directories exist from a previous uninstalled version. The two-step approach (binary OR directory) with separate confidence indicators gives the installer enough information to proceed.

### XDG Compliance Split

Platforms split into two groups:

- **XDG-compliant** (use `~/.config/` and `~/.local/share/`): OpenCode, Amp, Copilot CLI (with XDG_CONFIG_HOME override)
- **Custom paths**: Claude Code (`~/.claude/`), Cursor (Application Support), Kiro (`~/.kiro/`), Windsurf (`~/.codeium/`)

The installer must handle both patterns. On Windows, none follow XDG; each platform uses its own convention.

### MCP Configuration Is the Hardest Problem

Every platform uses a different MCP config file path and format. Some embed MCP config inside their main settings file (Amp, OpenCode), some use a dedicated file (Claude Code, Cursor, Kiro, Copilot CLI, Windsurf). The installer needs 7 separate MCP config writers.

### Agent Definition Format Divergence

Agent definitions have the widest format divergence:

- Markdown files: Claude Code, Cursor, OpenCode
- `.agent.md` extension: Copilot CLI
- JSON files: Kiro
- Not file-based: Amp (Task tool), Windsurf (no custom agent files)

This means the agent content type requires per-platform adapters or is skipped for platforms that do not support file-based agents.

### Cross-Platform Shortcut: `.claude/skills/` Has 6/7 Coverage

Writing skills to `.claude/skills/<name>/SKILL.md` covers 6 of 7 platforms through compatibility paths. Only Windsurf requires its own path (`.windsurf/skills/`). However, using the platform-native path is recommended for first-class support and future compatibility.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Implement dual detection (binary + config directory) for all 7 platforms | Binary-only misses GUI editors; directory-only may false-positive | Low |
| P0 | Use `Promise.allSettled` with `Bun.spawn({ timeout: 3000 })` for parallel detection | All 7 platforms detected in under 3 seconds total | Low |
| P0 | Use `which` on macOS/Linux, `where` on Windows for binary detection | OS-native binary lookup | Low |
| P0 | Create a platform adapter interface with per-platform implementations | Each platform has unique paths, formats, and capabilities | Medium |
| P1 | Write skills to platform-native paths, with `.claude/skills/` as cross-platform fallback | Native path ensures first-class support; fallback covers compat | Low |
| P1 | Implement safe JSON merge for MCP config files with namespaced keys | Prevents overwriting user customizations per ADR-003 | Medium |
| P1 | Track installed files in a manifest per plugin for clean uninstall | Required for `uninstall` command to reverse changes | Medium |
| P2 | Support XDG_CONFIG_HOME and platform-specific env var overrides | Respects user customization for OpenCode, Amp, Copilot CLI | Low |
| P2 | Detect Kiro IDE separately from Kiro CLI (different binaries: `kiro` vs `kiro-cli`) | Some users have IDE but not CLI, or vice versa | Low |

## 8. Conclusion

**Verdict**: Proceed with implementation of platform detection and content mapping.

**Confidence**: High

**Rationale**: All 7 platforms have documented binary names, config directories, and content paths. Detection via binary check + directory existence provides reliable results across all three operating systems. The parallel detection pattern using `Bun.spawn` and `Promise.allSettled` aligns with the Bun runtime decision (ADR-005) and meets the 3-second timeout spec.

### User Impact

- **What changes for you**: The installer runs 7 parallel detection checks in under 3 seconds, then writes plugin content to the correct platform-specific directories. No manual platform selection needed.
- **Effort required**: 7 platform adapter implementations (one per platform). Each adapter encapsulates detection logic, content paths, and config file formats.
- **Risk if ignored**: Writing to wrong paths causes plugins to not load. Missing detection causes platforms to be skipped during install.

## 9. Appendices

### Observations

- [fact] All 7 target platforms have distinct binary names: claude, cursor, copilot, kiro-cli, opencode, amp, windsurf #binary-detection
- [fact] GUI-installed editors (Cursor, Windsurf) may not have binaries in PATH, requiring config directory fallback detection #detection-reliability
- [fact] MCP configuration paths are completely platform-specific with 7 different file locations and 3 format variations #mcp #divergence
- [fact] .claude/skills/ path is read by 6 of 7 platforms for skill content, providing cross-platform fallback #skills #portability
- [fact] Bun.spawn supports native timeout option (milliseconds) with SIGTERM on expiry #bun #detection
- [decision] Dual detection (binary + directory) recommended for reliability across GUI and CLI platforms #detection-strategy
- [decision] Promise.allSettled with 3-second timeout per check enables parallel detection of all 7 platforms #parallel-detection
- [insight] XDG-compliant platforms (OpenCode, Amp) use ~/.config/ while others use custom paths (~/.claude, ~/.kiro, ~/.codeium) #config-paths
- [risk] Agent definition formats diverge across 4 different formats plus 2 platforms with no file-based agents #agents #fragmentation
- [technique] Safe JSON merge with namespaced keys prevents MCP config collisions during install #mcp #merge
- [decision] Dual detection confirmed for v1 over binary-only simplification (debate P0-7). Low implementation cost, GUI editor support from day one. --platform flag added as override for CI and edge cases where detection fails. #platform-detection #p0-resolution

### Sources Consulted

- Claude Code Install FAQ: <https://claudelog.com/faqs/where-is-claude-code-installed/>
- Claude Code Setup Docs: <https://code.claude.com/docs/en/setup>
- Cursor CLI Installation: <https://cursor.com/docs/cli/installation>
- Cursor Config Discussion: <https://forum.cursor.com/t/where-is-the-default-environment-settings-path/17269>
- Cursor Settings Location Analysis: <https://www.jackyoustra.com/blog/cursor-settings-location>
- GitHub Copilot CLI Docs: <https://docs.github.com/en/copilot/how-tos/set-up/install-copilot-cli>
- Copilot CLI Config Reference: <https://docs.github.com/en/copilot/how-tos/copilot-cli/set-up-copilot-cli/configure-copilot-cli>
- Copilot CLI Config File Locations: <https://inventivehq.com/knowledge-base/copilot/where-configuration-files-are-stored>
- Kiro CLI Installation: <https://kiro.dev/docs/cli/installation/>
- Kiro Homebrew Formula: <https://formulae.brew.sh/cask/kiro-cli>
- OpenCode CLI Docs: <https://opencode.ai/docs/cli/>
- OpenCode DeepWiki (Binary Detection): <https://deepwiki.com/sst/opencode/7.2-authentication-system>
- OpenCode Install Script: <https://github.com/anomalyco/opencode/blob/dev/install>
- Amp npm Package: <https://www.npmjs.com/package/@sourcegraph/amp>
- Amp Owner's Manual: <https://ampcode.com/manual>
- Amp CLI Guide: <https://github.com/sourcegraph/amp-examples-and-guides/blob/main/guides/cli/README.md>
- Windsurf Docs: <https://docs.windsurf.com/>
- Windsurf PATH Setup: <https://dilsayar.com/how-to-add-windsurf-to-path-on-mac-with-zsh-and-fish-shells/>
- Bun.spawn API Reference: <https://bun.com/reference/bun/spawn>
- Bun Child Process Docs: <https://bun.com/docs/runtime/child-process>

### Data Transparency

- **Found**: Binary names, install methods, config directory paths (all 3 OSes) for all 7 platforms. MCP config file paths and formats. Bun.spawn timeout and AbortController APIs. Cross-platform compatibility paths (AGENTS.md, .claude/skills/).
- **Not Found**: Exact Windows paths for Amp IDE mode (XDG convention assumed). Windsurf Windows %APPDATA% vs %USERPROFILE% distinction (community reports suggest %USERPROFILE%). Kiro IDE Windows config path (inferred from Linux/macOS pattern). Enterprise deployment paths for all platforms.

## Relations

- extends [[ANALYSIS-008 Platform Instruction File Paths]]
- extends [[ANALYSIS-002 Platform Capability Matrix]]
- relates_to [[DEBATE-ADR-002-target-platforms-and-audiences]]
- implements [[ADR-005 Bun Runtime Selection]]
