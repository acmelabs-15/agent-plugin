---
title: ANALYSIS-042 Native Platform Plugin Installation Feasibility
type: note
permalink: analysis/analysis-042-native-platform-plugin-installation-feasibility-1
tags:
- native-plugin
- platform-installation
- feasibility
- hybrid-approach
- architecture
---

> **Platform Scope Change**: Per ADR-002 Amendment #1 (2026-03-09), supported platforms reduced from 7 to 4: Claude Code, Cursor, GitHub Copilot, Kiro. OpenCode, Amp, and Windsurf were dropped due to incomplete content type coverage. References to dropped platforms in this note are historical only.

# ANALYSIS-042 Native Platform Plugin Installation Feasibility

## 1. Objective and Scope

**Objective**: Evaluate whether `@acmelabs-15/agent-plugin` should install plugins as native platform plugins (using each platform's own plugin infrastructure) on platforms that support them, instead of or in addition to config wiring.

**Scope**: Claude Code, Cursor, GitHub Copilot CLI, and Microsoft APM (platforms with native plugin support). Excludes Windsurf, Codex, Kiro, Amazon Q, OpenCode, and Amp (wiring-only platforms with no plugin manifest system). Research conducted March 2026.

## 2. Context

ADR-013 established that `agent-plugin install` is a "wiring tool": it scans `node_modules` for packages containing `plugin.json`, then writes their declared content (skills, agents, hooks, instructions, MCP entries) into platform-specific config files. This works universally across all 7 target platforms.

The alternative under evaluation: on platforms that natively support plugins (Claude Code, Cursor, Copilot CLI), install as a real native plugin using the platform's own `.claude-plugin/`, `.cursor-plugin/`, or `.github/plugin/` directory conventions. This would make the plugin visible in the platform's plugin management UI, enable marketplace integration, and leverage the platform's own install/uninstall/update lifecycle.

ANALYSIS-036 found that only 2 of 8 surveyed platforms have full plugin manifest systems (Claude Code, Copilot CLI). Cursor added its plugin system in February 2026 (v2.5). Microsoft APM is an emerging package manager with transitive dependency resolution.

## 3. Approach

**Methodology**: Web research on native plugin installation mechanisms for Claude Code, Cursor, Copilot CLI, and Microsoft APM. Cross-referenced with existing project analyses (ANALYSIS-005, ANALYSIS-036, ANALYSIS-040, ANALYSIS-041). Evaluated against current architecture (ADR-001, ADR-003, ADR-009, ADR-013).

**Tools Used**: WebSearch (10 queries), Brain MCP (8 note reads), file analysis (5 docs files)

**Limitations**: Cursor plugin API is new (Feb 2026) and documentation is incomplete. Claude Code plugin cache behavior has known issues (stale cache after updates). Microsoft APM is still emerging with limited production usage data. No access to platform source code.

## 4. Data and Analysis

### 4.1 Claude Code Native Plugin Installation

**Current mechanism**: `claude plugin add <source>` where source can be a marketplace reference, GitHub repo, Git URL, npm package, pip package, or local path.

**Directory structure**: `.claude-plugin/plugin.json` manifest at plugin root. Components (skills/, agents/, hooks/, commands/) at plugin root, NOT inside `.claude-plugin/`.

**Cache system**: Plugins copied to `~/.claude/plugins/cache/` keyed by name and version. Cache invalidation based on version string only. Local development plugins have known stale-cache issues (GitHub issue #15642).

**Installation scopes**: user (default, `~/.claude/settings.json`), project (`.claude/settings.json`), local (`.claude/settings.local.json`), managed (admin-controlled).

**Lifecycle commands**: `install`, `uninstall`, `enable`, `disable`, `update`, `validate`, `/reload-plugins`.

**Programmatic access**: `claude --plugin-dir ./path` for local testing. No documented API for third-party tools to programmatically register plugins in Claude Code's settings.

**Key finding**: Claude Code's plugin install copies files to its cache and registers the plugin in `settings.json`. A third-party tool could theoretically write to `~/.claude/plugins/cache/` and modify `settings.json` directly, but this is undocumented and fragile. The `claude plugin add` CLI command is the supported mechanism, and it requires running Claude Code itself.

### 4.2 Cursor Native Plugin Installation

**Current mechanism**: Cursor 2.5 (Feb 2026) introduced plugins. Install via `/add-plugin` in editor or browse `cursor.com/marketplace`. No external CLI command documented for plugin installation.

**Directory structure**: `.cursor-plugin/plugin.json` manifest. Structure mirrors Claude Code closely.

**Local development**: Community-documented workaround involves running a shell script (`install-plugin.sh`) and restarting Cursor. No hot-reload. No watch mode.

**Key finding**: Cursor's plugin system is 3 weeks old as of this analysis. No programmatic install API exists. Installation requires the Cursor editor to be running. A third-party tool cannot install native Cursor plugins without manipulating Cursor's internal state.

### 4.3 GitHub Copilot CLI Native Plugin Installation

**Current mechanism**: `copilot plugin install <source>` where source is a marketplace plugin, GitHub repo (`owner/repo`), or local path.

**Directory structure**: `plugin.json` at root, or in `.github/plugin/`, or in `.claude-plugin/` (all three locations accepted).

**Storage**: Installed to `~/.copilot/state/installed-plugins/PLUGIN-NAME/`.

**Marketplace**: Marketplace registration via `copilot plugin marketplace add <owner/repo>` or local path.

**Key finding**: Copilot CLI has the most straightforward programmatic install path. The `copilot plugin install ./local-path` command accepts local directories. A third-party tool could write plugin files to disk and invoke `copilot plugin install` programmatically. This is the most feasible native install target among the 3 platforms.

### 4.4 Microsoft APM

**Current mechanism**: `apm install <package>` installs skills, prompts, instructions, and agents. Creates `apm.yml` on first install. Supports transitive dependencies.

**Output**: Installs content under `.github/instructions/` for VS Code editor integration.

**Key finding**: APM is a package manager, not a platform plugin system. It is structurally similar to what agent-plugin does: resolve packages and wire content into platform-specific locations. APM is a peer/competitor, not a target for native installation.

### 4.5 Manifest Format Compatibility

Our `plugin.json` diverges from platform-native `plugin.json` in specific ways:

| Field | Our plugin.json | Claude Code | Copilot CLI | Cursor |
|---|---|---|---|---|
| Location | Plugin root | `.claude-plugin/` | Root or `.github/plugin/` | `.cursor-plugin/` |
| Required fields | `name`, `description` | `name` only | `name` only | `name` only |
| MCP field name | `mcp` (renaming to `mcpServers`) | `mcpServers` | `mcpServers` | `mcpServers` |
| `instructions` | Supported | Not in schema | Not in schema | Not in schema |
| `prompts` | Supported | Not in schema | Not in schema | Not in schema |
| `installMode` | Supported | Not in schema | Not in schema | Not in schema |
| `platformConfig` | Supported | Not in schema | Not in schema | Not in schema |
| `version` | In package.json | In plugin.json | In plugin.json | In plugin.json |

A native install would require generating a platform-specific `plugin.json` from our canonical `plugin.json`. Our unique fields (`installMode`, `platformConfig`, `instructions`, `prompts`) would be stripped. The `version` field would need to be injected from `package.json`.

### 4.6 What Native Installation Provides vs Wiring

| Capability | Native Plugin | Config Wiring |
|---|---|---|
| Platform UI visibility | Plugin appears in platform's plugin list | Content appears but plugin is invisible as a unit |
| Uninstall tracking | Platform tracks what to remove | agent-plugin must track via scan-diff-reconcile |
| Enable/disable toggle | Platform provides toggle | No equivalent (must unwire/rewire) |
| Auto-update | Platform can auto-update from marketplace | agent-plugin install re-scans on postinstall |
| Marketplace discovery | Visible in marketplace if published there | Not visible in any marketplace |
| Namespace handling | Platform auto-namespaces (plugin-name:component) | agent-plugin namespaces via ADR-003 |
| Version management | Platform tracks versions | bun.lockb tracks npm package version |
| Hook lifecycle | Platform manages hook registration | agent-plugin writes hooks to platform config |
| MCP server lifecycle | Platform auto-starts MCP servers | agent-plugin writes MCP config; platform starts |

## 5. Results

### 5.1 Platform-by-Platform Feasibility Assessment

| Platform | Native Install Feasible? | Mechanism | Effort | Risk |
|---|---|---|---|---|
| **Claude Code** | Partially | Write to cache + modify settings.json, OR invoke `claude plugin add` | High | Undocumented internals, cache invalidation issues, version sync |
| **Copilot CLI** | Yes | Invoke `copilot plugin install ./path` | Medium | Requires Copilot CLI installed, subprocess management |
| **Cursor** | No (currently) | No programmatic install path exists | N/A | Plugin system is 3 weeks old, API likely to change |
| **Microsoft APM** | No (different model) | APM is a peer tool, not a target platform | N/A | Competing, not complementary |

### 5.2 Architectural Requirements for Native Install

To support native plugin installation, agent-plugin would need:

1. **Platform-specific plugin.json generator**: Transform our canonical `plugin.json` into each platform's native format. Strip unsupported fields. Inject `version` from `package.json`. Place manifest in correct location (`.claude-plugin/`, `.cursor-plugin/`, `.github/plugin/`, or root).

2. **Directory structure mapper**: Create platform-expected directory layout. Claude Code expects components at plugin root alongside `.claude-plugin/`. Copilot CLI accepts multiple manifest locations.

3. **Platform CLI invocation layer**: Call `claude plugin add` or `copilot plugin install` as subprocesses. Handle authentication, error codes, timeout. Detect if platform CLI is available.

4. **Dual-path install logic**: For each platform, decide: native install or config wiring? Need fallback from native to wiring if native install fails.

5. **Sync mechanism**: Keep native plugin version in sync with npm package version. When `bun update` changes the package, the native plugin must also update.

6. **Uninstall coordination**: When a package is removed from `node_modules`, both the wired config entries AND the native plugin must be removed.

### 5.3 Comparison: Native vs Wiring vs Hybrid

| Dimension | Native Only | Wiring Only (Current) | Hybrid |
|---|---|---|---|
| Platform coverage | 2 of 7 (Claude Code, Copilot CLI) | 7 of 7 | 7 of 7 |
| Implementation complexity | High (per-platform CLI integration) | Medium (scan-diff-reconcile) | Very High (both paths + decision logic) |
| Maintenance burden | High (track platform CLI changes) | Medium (track config format changes) | Very High (maintain both) |
| User experience on supported platforms | Better (native UI, enable/disable, marketplace) | Functional but invisible | Best of both |
| User experience consistency | Inconsistent (native on 2, nothing on 5) | Consistent across all 7 | Inconsistent (native on 2, wiring on 5) |
| Uninstall reliability | High (platform tracks ownership) | Medium (scan-diff-reconcile, stateless) | Complex (two removal paths) |
| Version sync | Hard (npm version vs native plugin version) | Simple (npm version is single source of truth) | Hard (same as native) |
| CI/CD compatibility | Low (requires platform CLI in CI) | High (just file manipulation) | Mixed |
| Error surface | Large (subprocess failures, auth, timeouts) | Small (file read/write) | Very large |
| Lines of code estimate | 1500-2500 for 2 platforms | 800-1200 for all 7 | 2300-3700 combined |
| Testing complexity | High (mock platform CLIs) | Low (mock filesystem) | Very high |

## 6. Discussion

### The Core Tradeoff

Native plugin installation provides real benefits on the 2 platforms that support it: UI visibility, enable/disable toggle, marketplace potential, and platform-managed lifecycle. These are genuine UX improvements for users of Claude Code and Copilot CLI.

The cost is substantial. Supporting native install on 2 platforms while maintaining wiring for all 7 roughly triples the installation logic. Every change to the install flow must be tested against both paths. Platform CLI updates can break the native install path at any time. Version synchronization between npm packages and native plugins creates a new class of bugs.

### Why Wiring Is Architecturally Sound

ADR-013's wiring model has a structural advantage: it treats all platforms identically. The scan-diff-reconcile loop reads `node_modules`, compares against wired state in platform configs, and reconciles. This loop is the same for Claude Code and Windsurf. The platform adapter layer (ADR-009, ANALYSIS-040) handles format differences. Adding platform 8 is a data entry in `platforms.config.json`, not new code.

Native install breaks this uniformity. Claude Code's native install uses subprocess invocation. Copilot CLI's uses a different subprocess. Cursor has no mechanism. Each platform requires custom integration code that cannot be abstracted behind a shared interface because the installation mechanisms are fundamentally different (cache copy vs CLI invocation vs editor-internal).

### The Marketplace Question

The strongest argument for native install is marketplace visibility. A plugin installed natively can appear in Claude Code's `/plugin list` and potentially in the marketplace. A wired plugin is invisible as a unit; its skills, hooks, and MCP entries exist in platform configs but there is no "plugin" entity the user can manage.

Counter-argument: our wiring approach already namespaces components as `plugin-name:component-name` (ADR-003). Users see the plugin name in every component reference. The `agent-plugin list` command shows all discovered and wired plugins. The UX gap is real but bounded.

### Claude Code Lifecycle Hooks Are Still Missing

GitHub issue #11240 documents that Claude Code currently lacks plugin install/uninstall lifecycle hooks. Plugins cannot run setup scripts on install or cleanup scripts on uninstall. This reduces the value of native installation: even as a native plugin, the install lifecycle is limited.

### Cursor Is Too Early

Cursor's plugin system shipped February 17, 2026. It is 3 weeks old. No programmatic install API. No CLI command. Community workarounds involve manual file copying and editor restarts. Investing in native Cursor plugin support now would be building on sand.

### APM Is a Competitor, Not a Target

Microsoft APM solves a similar problem to agent-plugin: resolving packages and installing AI agent content into platform-specific locations. It is not a platform that accepts plugins. Supporting APM as a distribution channel would mean publishing agent-plugin packages in APM format, which is a distribution strategy question separate from native installation.

### The Timing Factor

All 3 native plugin systems are less than 6 months old. Claude Code plugins launched in public beta early 2026. Copilot CLI plugins launched with GA in February 2026. Cursor plugins launched February 2026. Plugin APIs are unstable. Building tight integration with unstable APIs creates high maintenance burden.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Continue with wiring-only approach for v1.0 | Covers 7/7 platforms. Avoids 2-3x complexity for 2-platform benefit. Unstable native APIs create maintenance risk. | Already designed |
| P1 | Design adapter interface to support native install as future extension | The platform adapter (ADR-009) should have an abstract `install()` method that defaults to wiring but can be overridden with native install for specific platforms. This keeps the door open without implementing it now. | Low |
| P1 | Track Claude Code plugin CLI stability | Monitor whether `claude plugin add` becomes a stable, documented API for third-party tools. Watch for lifecycle hooks (issue #11240). | Low (tracking) |
| P2 | Prototype Copilot CLI native install as first candidate | Copilot CLI has the most feasible native install path (`copilot plugin install ./path`). A prototype would validate the effort and reveal integration issues. | Medium |
| P2 | Monitor Cursor plugin API evolution | Cursor's plugin system is too new for integration. Revisit when programmatic install mechanism exists. | Low (tracking) |
| P3 | Evaluate native install for v2.0 based on platform API stability | After 6-12 months, native plugin APIs will stabilize. The cost-benefit calculation improves when APIs are stable and documented for third-party use. | Deferred |

## 8. Conclusion

**Verdict**: Defer native plugin installation to v2.0+

**Confidence**: High

**Rationale**: Native plugin installation provides genuine UX benefits on 2 of 7 platforms but roughly triples implementation complexity, creates version synchronization challenges, and depends on platform APIs that are less than 6 months old. The wiring approach covers all 7 platforms uniformly. The adapter interface can be designed now to accommodate native install later without architectural changes.

### Adoption Status (2026-03-09)

This recommendation was adopted. ADR-014 uses direct file-based installation (writing content to platform config locations) rather than native platform plugin APIs. The deferral to v2.0+ remains the current plan. Note: ADR-013 (referenced in Context section) was superseded by ADR-014 on the same day it was accepted. The wiring model changed from ADR-013's `node_modules` scan-diff-reconcile to ADR-014's explicit `agent-plugin add/remove/update` with `.agent-lock.json` lockfile, but the core recommendation from this analysis (defer native install) applies equally to both models.

### User Impact

- **What changes for you**: No changes to the current architecture. The adapter interface should include an extension point for native install. This is a 1-day design task, not an implementation task.
- **Effort required**: Zero for v1.0. Medium (2-4 weeks) if native install is added in v2.0 for Claude Code + Copilot CLI.
- **Risk if ignored**: Missing marketplace visibility on Claude Code and Copilot CLI. This is a real but bounded gap. Users can still use `agent-plugin list` to manage plugins. Platform-native `enable/disable` toggle is unavailable; users must use `agent-plugin install` to rewire.

## 9. Appendices

### Sources Consulted

- [Claude Code Plugins Reference](https://code.claude.com/docs/en/plugins-reference)
- [Claude Code Plugin Marketplaces](https://code.claude.com/docs/en/plugin-marketplaces)
- [Claude Code Plugin Cache Issue #15642](https://github.com/anthropics/claude-code/issues/15642)
- [Claude Code Plugin Lifecycle Hooks Issue #11240](https://github.com/anthropics/claude-code/issues/11240)
- [Cursor Plugins Documentation](https://cursor.com/docs/plugins)
- [Cursor Plugins Building Reference](https://cursor.com/docs/plugins/building)
- [Cursor Plugin Marketplace](https://cursor.com/marketplace)
- [Writing and Testing Cursor Plugins Locally (Tajzich, Mar 2026)](https://medium.com/@v.tajzich/how-to-write-and-test-cursor-plugins-locally-the-part-the-docs-dont-tell-you-4eee705d7f76)
- [GitHub Copilot CLI Plugin Reference](https://docs.github.com/en/copilot/reference/cli-plugin-reference)
- [Finding and Installing Plugins for Copilot CLI](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-finding-installing)
- [Creating Copilot CLI Plugins](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-creating)
- [Microsoft APM GitHub](https://github.com/microsoft/apm)
- [Claude Code Plugin Dev Blog (Somethinghitme)](https://somethinghitme.com/2026/01/31/creating-local-claude-code-plugins/)
- [Claude Code Plugins README (anthropics/claude-code)](https://github.com/anthropics/claude-code/blob/main/plugins/README.md)
- [Cursor Marketplace Blog](https://cursor.com/blog/marketplace)

### Data Transparency

- **Found**: Native plugin installation mechanisms for Claude Code, Copilot CLI, Cursor. Plugin.json schema differences. Cache behavior. Lifecycle command surfaces. Storage locations. Known issues with cache invalidation. Community workarounds for local development.
- **Not Found**: Cursor programmatic install API (confirmed: does not exist yet). Claude Code documented API for third-party plugin registration (confirmed: undocumented). Copilot CLI plugin install error codes and edge cases. Exact lines-of-code estimates for implementation (estimated from comparable adapter work). Performance impact of subprocess invocation during install.

## Observations

- [fact] Only 2 of 7 target platforms (Claude Code, Copilot CLI) have programmatically accessible native plugin install mechanisms; Cursor's plugin system is 3 weeks old with no external API #platform-support #feasibility
- [fact] Claude Code copies plugins to ~/.claude/plugins/cache/ keyed by name and version; cache invalidation is version-only, causing known stale-cache issues for local development (GitHub #15642) #claude-code #cache
- [fact] Copilot CLI accepts local path installation via `copilot plugin install ./path` and stores at ~/.copilot/state/installed-plugins/PLUGIN-NAME/, making it the most feasible native install target #copilot-cli #feasibility
- [fact] Claude Code lacks plugin install/uninstall lifecycle hooks (GitHub #11240), limiting the value of native installation even on this platform #claude-code #lifecycle
- [decision] Recommend deferring native plugin installation to v2.0+; wiring approach covers 7/7 platforms uniformly with 2-3x less complexity #architecture #deferral
- [insight] Native install would require maintaining two parallel installation paths (native + wiring fallback) with version synchronization between npm packages and platform plugins, roughly tripling install logic #complexity #tradeoff
- [insight] Microsoft APM is a peer/competitor tool solving similar cross-platform wiring problems, not a platform target for native plugin installation #microsoft-apm #competitive
- [risk] All 3 native plugin systems are less than 6 months old with unstable APIs; building integration now creates high maintenance burden as APIs evolve #stability #timing
- [technique] Design adapter interface with abstract install() method defaulting to wiring, overridable per-platform for future native install support without architectural changes #extensibility #adapter-pattern
- [insight] Strongest argument for native install is marketplace visibility and platform UI integration; strongest argument against is implementation complexity and API instability #tradeoff #ux

## Relations

- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-003 Conflict Resolution and Namespacing]]
- relates_to [[ADR-009 Platform Detection and Config Registry]]
- relates_to [[ADR-013 npm-Package Distribution and Revised Command Tree]]
- relates_to [[ANALYSIS-005 Claude Code Plugin Format]]
- relates_to [[ANALYSIS-036 AI Platform Plugin Standards Survey]]
- relates_to [[ANALYSIS-040 Prior Art for Cross-Platform Adapter Patterns]]
- relates_to [[ANALYSIS-041 Plugin Manifest Cross-Platform Comparison]]
- relates_to [[ADR-014 Explicit Installation Model and Content Features]]