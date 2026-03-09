---
title: SESSION-2026-03-07_01-agent-plugin-spec-ideation
type: session
permalink: sessions/session-2026-03-07-01-agent-plugin-spec-ideation
tags:
- session
- '2026-03-07'
- agent-plugin
- ideation
- spec-review
---

# SESSION-2026-03-07_01 Agent Plugin Spec Ideation

**Status:** IN_PROGRESS
**Branch:** ideation/agent-plugin-spec
**Starting Commit:** 84f8511 first commit
**Current Commit:** 79b429d feat: add ADR-014 (accepted) and complete decision audit reconciliation
**Objective:** Work through the `@acmelabs-15/agent-plugin` comprehensive design specification using the ideation workflow, conducting web research, creating ADRs for architectural decisions, and producing feature specs in the features/ directory

---

## Acceptance Criteria

- [~] Complete research on all major spec areas (CLI framework, dependencies, platform support, data storage, MCP server, scaffolding wizards) -- ~85% complete. Groups 1-7 done. Groups 8-9 partially remaining (4 gaps open).
- [~] Create ADRs for key architectural decisions identified in the spec -- 10 active ADRs created (ADR-001 through ADR-014, excluding ADR-004). ADR-004 (security) still needed. 3 ADRs superseded (008, 010, 013).
- [ ] Create feature specs in features/ directory following FEAT-NNN template structure -- Phase 3 (not started)
- [x] All research findings saved as Brain memory notes -- 47 analysis notes created (ANALYSIS-001 through 047)
- [x] Session note kept current with all touched files, commits, memory notes, work log

---

## Session Start Protocol (BLOCKING)

| Req Level | Step | Status | Evidence |
|-----------|------|--------|----------|
| MUST | Initialize Brain MCP | [x] | bootstrap_context called, agent-plugin project active |
| MUST | Create session log | [x] | SESSION-2026-03-07_01 created via mcp session tool |
| SHOULD | Search relevant memories | [x] | No prior memories found for agent-plugin project |
| SHOULD | Verify git status | [x] | Branch: main, Commit: 84f8511, clean working tree |

---

## Key Decisions

- [decision] Using ideation workflow for nuanced, collaborative spec review #workflow
- [decision] Delegate ALL work to specialized Brain agents (no direct implementation by orchestrator) #delegation
- [decision] Agent Teams not available; using subagent delegation instead #constraint
- [decision] Feature specs follow FEAT-NNN structure from FEAT-003-filtering template #template
- [decision] All research must include community best practices via web research #research-standard
- [decision] Design spec is a requirements document with suggestions, NOT pre-made decisions #process
- [decision] Go over every decision one at a time with user before committing to ADRs #process
- [decision] ADRs only created AFTER decisions are locked in through discussion #process
- [decision] Use brain adr-review skill (6-agent debate protocol) for ALL ADR reviews, not ad-hoc agents #process
- [decision] Per-ADR workflow: research, discuss, create ADR, run adr-review, save debate log to critique/DEBATE-ADR-NNN-name, resolve P0/P1 with user, then next ADR #process
- [decision] Cross-platform full lifecycle management confirmed as target #positioning
- [decision] Platform criteria: must support prompts, skills, agents, hooks, MCPs, AND running agents in parallel #platform-criteria
- [decision] Target ALL 7 full-support platforms: Claude Code, Cursor, GitHub Copilot CLI, Kiro, OpenCode, Amp, Windsurf #platforms
- [decision] Must update platform-specific instruction files (CLAUDE.md, AGENTS.md, etc.) on install without breaking existing content #platform-instructions
- [decision] Full three-audience model (Consumer CLI + Author CLI + AI MCP) in single package -- MCP server is key differentiator, shared core logic with thin audience-specific layers #audiences
- [decision] Package name: `@acmelabs-15/agent-plugin` (scoped) -- need to register `acmelabs-15` npm org at npmjs.com #naming
- [decision] Self-bootstrapping: design manifest structure in Phase 1, ship runtime in Phase 4 #self-bootstrap
- [decision] Own plugin format, no hosted registry. Sources: GitHub shorthand (owner/repo), full GitHub URL, GitLab URL, local path, npm package #distribution
- [decision] Borrow SKILL.md standard concepts from Vercel skills, extend pattern to agents/prompts/hooks/MCPs #format-standard
- [decision] Plugin = bundle containing many skills, many agents, many prompts, many hooks, probably single MCP #bundle-model
- [decision] Plugin directory structure: plugin.json at root (not nested), skills/, agents/, prompts/, hooks/, mcp/ directories #structure
- [decision] plugin.json is ALWAYS required (not optional like Claude Code) #manifest
- [decision] Minimum required fields: name, version, description #manifest
- [decision] Component declarations in plugin.json (skills, agents, hooks, prompts, mcp paths) #manifest
- [decision] installMode: "bundle" or "collection" -- bundle = all-or-nothing, collection = pick and choose #install-mode
- [decision] Core optional fields: author, license, repository, homepage, keywords #manifest
- [decision] platforms field REMOVED from plugin.json -- all plugins are inherently cross-platform, plugin manager handles translation to all 7 platforms #manifest #cross-platform
- [decision] formatVersion REMOVED from plugin.json -- schema evolution via additive changes, unknown field tolerance, doctor command, upgrade/update auto-detection, migration wizards via clack/prompts (ANALYSIS-007: 70% of config formats handle evolution without version fields) #manifest #schema-evolution
- [decision] Component format: cross-platform core frontmatter (name, description, type, requires, sources) + platformConfig section for platform-specific overrides #component-format
- [decision] Platform-aware frontmatter generation: on install, emit ONLY fields the target platform supports but include ALL supported fields. Source frontmatter is the superset. #platform-adaptation
- [decision] No graceful degradation tier -- only 6/6 platforms supported. Codex CLI, Cline, Gemini CLI excluded. Partial support creates silent failures worse than no support. #platforms #no-degradation
- [decision] No publish command -- no registry means nothing to publish to. Distribution is handled by making the plugin available at any supported source (GitHub, GitLab, npm, local path). #distribution #no-publish
- [decision] Source versioning: all source types support optional version specifier (e.g., owner/repo@v1.2.0, @scope/pkg@^2.0.0). No version = latest. Git sources use tags/releases, npm uses semver. #distribution #versioning
- [decision] Installed version tracking: plugin manager records installed version in state store for accurate upgrade/update behavior #state-management #versioning
- [decision] Invalid version handling: detect invalid version, display available versions via @clack/prompts select picker (including latest) for user to choose #ux #versioning
- [decision] Platform instruction file paths fully specified for all 7 platforms (ANALYSIS-008). Fallback convention: .agents/ directory and AGENTS.md for platforms without well-defined paths #platforms #instruction-files
- [fact] AGENTS.md read by 6/7 platforms, CLAUDE.md by 4/7, .claude/skills/ by 6/7, .agents/skills/ by 4/7 and emerging as standard #platforms #cross-platform-coverage
- [decision] NEG-006 added to ADR-002: instruction file modification is an attack vector, deferred to ADR-004 #security
- [decision] NEG-007 added to ADR-002: no centralized vetting in multi-source model, deferred to ADR-004 #security
- [decision] Self-bootstrapping = self-install + dogfood: agent-plugin install acmelabs-15/agent-plugin works AND tool's own content uses plugin format #self-bootstrap
- [decision] Three-audience bloat is not a real concern for CLI tools -- gunshi lazy loading handles it naturally #bloat
- [decision] Instruction file updates must be templated/controlled, not AI freestyle -- content derived from author metadata with sanitization #instruction-files
- [decision] Always-namespace adopted: auto-prefix every installed component with plugin-name:component-name (Claude Code model). Eliminates user-choice conflict resolution, rename tracking, state store for mappings, and cross-reference updates #namespacing #simplification
- [decision] Colon separator is logical identifier only, NEVER in filenames (Windows forbids, macOS replaces). Plugin/component names validated as kebab-case #namespacing #validation
- [decision] Hook merge: strictest wins for blocking hooks. Overlay/recompute pattern -- hook contributions stored per-plugin within the lockfile, merged deterministically on demand. Installation order irrelevant #hooks #merge
- [decision] NEG-007: Strictest-wins can make plugins non-composable when security postures conflict. Mitigation scoped to ADR-004 #hooks #composability
- [decision] Hook execution blocked until ADR-004 consent model is implemented -- hooks are stored but NOT executable until security policy exists #security #hooks
- [decision] State store: single lockfile at project root (plugin-lock.json) or user home (~/.config/agent-plugin/plugin-lock.json for XDG). No .agent-plugins/ state directory. Lockfile is cache, not source of truth #state-management
- [decision] Hook overlays stored as sections within the lockfile, not separate files. Simplifies design: one state file, no separate state directory #state-management #simplification
- [decision] Lockfile corruption recovery: re-derive from installed plugin manifests on disk. doctor command handles proactive health checks #reliability
- [decision] File permissions: lockfile mode 600, XDG config directory mode 700 #security #permissions
- [decision] Hook merge security deferred to ADR-004 (forward reference from ADR-003) #security #deferred
- [decision] P1-7/P1-9 injection sanitization covered by ANALYSIS-009 mandate + ANALYSIS-013 research. Implementation detail for feature spec, not ADR-level #sanitization
- [decision] platformConfig: Hybrid D+C pattern adopted (adapter + layered overrides). 4-level resolution: Agent Skills standard fields → adapter concept mapping → plugin.json platformConfig → per-component platforms block. 3 cross-platform concepts: loadingStrategy, filePatterns, approvedTools. 80% of plugins only need levels 1-2 (estimated). #platform-config #architecture
- [decision] Zod v4 (full, not Mini) as org-wide validation standard. Bundle size irrelevant for CLI/backend. Better DX via method chaining, IntelliSense, and built-in English error messages. #validation #org-standard
- [decision] ADR-001 Section 9 updated to reference ADR-003 as authoritative for conflict resolution #cross-adr-consistency
- [decision] Command tree finalized: publish removed (no registry), new mcp init uses @modelcontextprotocol/sdk + zod, completions via @gunshi/plugin-completion, 7 platforms per ADR-002 #command-tree
- [decision] upgrade only (update alias dropped): convention is upgrade = install newer versions. No alias to avoid npm-style naming confusion. #command-alias
- [decision] --conflict flag REMOVED: always-namespace (ADR-003) eliminates file conflicts between plugins. No user choice needed. #conflict-resolution #simplification
- [decision] picocolors REMOVED: @clack/prompts v1.1.0 replaced picocolors with node:util styleText. Use styleText directly for terminal colors. Zero dependencies. #colors #dependency-change
- [decision] CI detection via ci-info package (zero deps, 50+ CI vendors). Three-layer precedence: --ci flag > env var > TTY check. #ci-mode
- [decision] --ci and --yes are DISTINCT: --yes auto-confirms but keeps visual feedback (spinners, colors, progress). --ci is full non-interactive (implies --yes, no spinners, no colors). #ci-mode #flags
- [decision] 5 global flags: --ci, --yes/-y, --json, --verbose/-v, --quiet/-q. Output stack: quiet (errors only) < default < verbose. --json overrides all for structured output. #global-flags
- [decision] Three-tier input resolution pattern: (1) Interactive → @clack/prompts select/multiselect, (2) CI → error with flag hint, (3) MCP → error response with options list for agent self-correction. Core architectural principle. #input-resolution #three-tier
- [decision] Context-aware interactive fallback: no-subcommand shows p.select() menu. plugin.json present → author commands first. Otherwise consumer commands first. #interactive-ux
- [decision] 16 validation rules for p.text() inputs: 3 naming, 3 uniqueness, 3 paths/patterns (glob not regex for hook matchers), 3 misc, 4 source format validation (npm, git HTTPS, git shorthand, local path) #validation
- [decision] Bun ADOPTED as runtime: 4-8x faster CLI startup (8-15ms vs 40-120ms), native TypeScript, Anthropic-owned (acquired Oven Dec 2025), Claude Code proves production viability. Dual distribution: npm primary + optional compiled binaries. #runtime #adopted
- [decision] @clack/prompts ADOPTED for interactive CLI: built-in wizard flows via group(), spinners, visual framing. Bun stdin risk (GitHub issues #4835, #3099, #7033) noted but mitigatable via testing/fallback. #interactive-prompts #adopted
- [decision] picocolors REMOVED: @clack/prompts v1.1.0 replaced it with node:util styleText. Use styleText directly. #colors #removed
- [decision] gunshi ADOPTED as CLI framework: built-in lazy loading (critical for 20+ commands), explicit Bun support, full TypeScript inference. kazupon (Vue.js core team) as maintainer. citty as documented fallback (similar API). Pin exact version (pre-1.0). #cli-framework #adopted
- [decision] @modelcontextprotocol/sdk ADOPTED for MCP server (skip fastmcp): official SDK with Bun support, fastmcp wraps it anyway (adds overhead), fastmcp value-adds (auth, CORS) irrelevant for stdio embedded server. Direct v2 upgrade path. #mcp #adopted
- [decision] chokidar v5 ADOPTED for file watching (watcher as fallback): 123M weekly downloads, zero native deps, ESM-only. Only used for author dev command. @parcel/watcher broken on Bun, Bun fs.watch has recursive watching bugs. #file-watching #adopted
- [decision] yaml 2.x ADOPTED for frontmatter parsing (gray-matter DISQUALIFIED due to CVE-2025-64718 in pinned js-yaml@^3.13.1, inactive maintainer 5 years). Manual 5-10 line parser + Zod v4 validation. Zero dependencies. #frontmatter #adopted
- [decision] Markdown processing REMOVED from MVP -- no markdown-to-HTML rendering needed. Tool extracts frontmatter and passes body as-is. Add micromark only if concrete use case emerges later. #markdown #removed
- [decision] drizzle-orm + SQLite REMOVED -- JSON lockfile (ADR-003) sufficient for 5-20 plugins. bun:sqlite built-in if ever needed at scale. #database #removed
- [decision] @orama/orama full-text search REMOVED permanently -- Array.filter() on name/description/tags sufficient. No plugin manager bundles search engines. #search #removed
- [decision] @huggingface/transformers semantic search REMOVED permanently -- over-engineering for CLI plugin manager. AI assistants via MCP already have semantic understanding. #search #removed
- [decision] plugin.json only for manifest discovery in sources -- no package.json field fallback (ADR-001 already mandates plugin.json at root) #manifest-discovery
- [decision] Bare owner/repo ambiguity: smart detection (check filesystem first, fall back to GitHub). Matches Go's approach. #source-resolution #smart-detection
- [decision] Bun.semver ADOPTED for semver comparison (production-ready since Nov 2023, passes node-semver test suite, satisfies+order cover our needs). node-semver NOT needed. #semver #bun
- [decision] tar npm package ADOPTED for archive extraction (15+ years battle-tested, built-in strip:1, streaming, security hardened). Bun.Archive too young (2 months, no strip-components, memory-buffered). #extraction #tar
- [decision] Bun.write with response.arrayBuffer() workaround for downloads (known hanging bug with Response streaming, PR in progress). Acceptable for <10MB plugins. #downloads #bun-workaround
- [decision] Platform detection: dual strategy (binary check + config directory existence). Both signals required for high confidence. Binary-only fails for GUI editors without CLI. Directory-only false-positives from leftover configs. #platform-detection #dual-detection
- [decision] Detection runs in parallel via Promise.allSettled with Bun.spawn({ timeout: 3000 }). Uses which on macOS/Linux, where on Windows. #platform-detection #parallel
- [decision] MCP key namespacing: colon separator (`plugin-name:server-name`), matching Claude Code internal convention and ADR-003 component identifier pattern. Replaces slash and double-dash from earlier analysis. #mcp #namespacing
- [decision] Install scope: project scope default. When no project detected and --global not passed, prompt user via @clack/prompts confirm. Error in non-interactive mode. #install-scope #ux
- [decision] System/platform dependencies: agent-plugin CAN install platform CLIs and system deps with explicit user confirmation via @clack/prompts. All installed deps tracked in plugin-lock.json for uninstall. #dependencies #system-deps
- [decision] Installation flow: 6-phase model (Detect, Select, Resolve, Confirm, Apply, Record). User selects which platforms to install to via @clack/prompts multiselect. Missing platforms treated as installable deps. #installation #6-phase
- [decision] Upgrade strategy: interactive upgrade flow -- scan installed plugins, check latest versions, present @clack/prompts multiselect showing `plugin-name current -> latest`. User picks which to upgrade. Atomic replace with rollback on failure. #upgrade #interactive
- [decision] Platform config mapping: all platform-specific mapping lives in `platforms.config.json` at project root. Plugin authors write platform-agnostic plugin.json. Agent-plugin reads both at install time. Pure data file, not code. #platform-config #registry
- [decision] Optional CLI generation from MCP tools: plugin.json `cli` field ("auto" | path | omitted). v1 produces flat subcommands (no auto-grouping heuristic). Author `cli.groups` for explicit grouping. All-or-nothing install for v1. Built-in `mcp` command group (start/stop/restart/status) for daemon lifecycle. Daemon/stdio transport model: stdio for MCP clients (Claude Code), daemon for CLI. `mcp restart` never touches stdio instances. Binary name denylist (31 entries). Custom binary path containment via realpathSync(). Parameter type mapping (7 types). Trust model documented. Symlinked to ~/.local/bin, tracked in plugin-lock.json. #cli #mcp #auto-generation #daemon-lifecycle
- [decision] MCP key separator: colon (`plugin-name:server-name`), matching Claude Code internal convention. Replaces slash from earlier analysis. #mcp #namespacing
- [decision] platforms.config.json: bundled in npm package, updated via version bumps. Contains all platform-specific mapping. Pure data file. #platform-config
- [decision] Dual platform detection kept for v1 with --platform flag override. Binary check + config directory check via Promise.allSettled with 3s timeout. #platform-detection
- [decision] CLI generation extracted from ADR-010 into ADR-011 per debate consensus (5/6 reviewers agreed on split) #architecture #extraction
- [decision] Dependency model: deps defined by agent-plugin package (platforms.config.json), not plugin authors. Multiselect prompt for missing deps including package managers. Tracked in lockfile for uninstall reverse multiselect. #dependencies
- [decision] MCP daemon lifecycle commands: built-in start/stop/restart/status for every plugin with MCP server. Restart safety: never touches stdio instances managed by Claude Code. PID file tracking at ~/.local/share/agent-plugin/pids/. #mcp #daemon-lifecycle
- [decision] MCP tool registry: on-demand for v1 (read from MCP servers transiently). No persistent tool schema storage in lockfile. Defer persistence to v2 if cross-plugin discovery latency is a problem. #mcp #data-storage
- [decision] Dual-location manifest: NO. plugin.json only (ADR-001 mandate). No embedded "agentPlugin" field in package.json. Simplifies manifest discovery and validation. #manifest #simplification
- [decision] Content-type command groups replace ADR-007 `new` subtree: skill/agent/mcp/command/hook/instruction each get create/remove/list subcommands. eval/improve only for types with creator skills. #command-tree #scaffolding
- [decision] Template rendering: yaml 2.x stringify for YAML frontmatter + tagged template literals for TypeScript code. Zero new dependencies. gray-matter DISQUALIFIED (CVE-2025-64718). #templates #zero-deps
- [decision] Schema-first architecture: single Zod v4 schema per wizard drives all 3 interfaces (interactive, CI, MCP). Prevents interface drift. #schema-first #consistency
- [decision] Bundled creator skills: skill-creator, agent-creator, mcp-builder (adapted from Anthropic), instruction-evaluator (original). Self-bootstrapping via self-install. #creator-skills #self-bootstrap
- [decision] eval/improve split: eval = read-only analysis with report, improve = interactive diff preview with Why annotations and 3 apply modes. #eval-improve #lifecycle
- [decision] rules/ renamed to instructions/. Instructions merge into AGENTS.md (6/7 platforms) + CLAUDE.md at install time. #instructions #rename
- [decision] MCP server scaffolding: @modelcontextprotocol/sdk with separate tool files and marker comments for create-tool patching. No per-MCP biome.json or package.json. #mcp #scaffolding
- [decision] eval/improve trust model: executes bundled first-party Bun TypeScript scripts (not user code). HTML eval viewer via Bun.serve(). Shared viewer infrastructure across skill/agent/mcp eval. #trust-model #security
- [decision] Faithfulness principle: creator skills stay close to Anthropic originals. Exceptions: Python→Bun TS conversion, fastmcp→@modelcontextprotocol/sdk, minor verbiage alignment. #faithfulness #creator-skills
- [decision] Commands restored as 6th content type (amending ADR-001). Commands are cross-platform, not Claude Code legacy. #commands #manifest
- [decision] Zod v4 + MCP SDK verified compatible on Bun. Min versions: SDK >= 1.23.0, Zod >= 4.1.13. MCP SDK works 100% on Bun (zero Node.js dependency). #compatibility #bun
- [decision] npm-package distribution model adopted (TanStack Intent style): plugins are npm packages installed via `bun add @scope/plugin`. Bun handles ALL package management (version resolution, lockfile bun.lockb, source resolution, integrity verification, upgrades). agent-plugin becomes purely a "wiring tool" bridging npm packages to AI platform configs. #distribution #architecture #major-pivot
- [decision] Consumer commands eliminated: `add` REMOVED (use `bun add`), `remove` REMOVED (use `bun remove`), `upgrade` REMOVED (use `bun update`). Bun handles package management natively. #commands #simplification
- [decision] `init` redesigned to Husky model: `agent-plugin init` wires lifecycle hooks into package.json (`"postinstall": "agent-plugin install"`). Smart merge with existing scripts. `agent-plugin deinit` reverses cleanly. postinstall covers add/remove/update since bun triggers it for all dependency changes. #init #lifecycle-hooks
- [decision] `install` command is core wiring operation: scans node_modules for packages containing plugin.json, diffs discovered plugins vs currently wired platforms, reconciles (adds new, removes deleted, updates changed). #install #core-command
- [decision] `create` command ADDED: scaffolds a new plugin project (like `bun create`). #create #scaffolding
- [decision] `deinit` command ADDED: reverse of init, removes lifecycle hooks from package.json. #deinit #lifecycle-hooks
- [decision] `dev` command REMOVED: not needed with lifecycle hook model. #dev #removed
- [decision] `complete` command REMOVED: shell completions not needed. #completions #removed
- [decision] `publish` command CONFIRMED REMOVED: no registry, no version command. Version management is author's responsibility using standard tools (bun, npm). validate can warn about plugin.json/package.json inconsistencies. #publish #removed
- [decision] version field removed from plugin.json required fields: package.json version is authoritative (controlled by npm/bun). plugin.json minimum required fields reduced to just `name` and `description`. plugin.json becomes primarily a content declaration manifest (skills[], agents[], hooks[], instructions[], commands[], mcp[]). #manifest #simplification
- [decision] ADR-008 (Source Resolution) FULLY SUPERSEDED: bun handles all source resolution natively. #adr-008 #superseded
- [decision] ADR-010 (Installation Lifecycle) MOSTLY SUPERSEDED: 6-phase model replaced by `bun add` + `agent-plugin install` wiring. #adr-010 #superseded
- [decision] ADR-003 Decision 3 (lockfile) SUPERSEDED: bun.lockb replaces plugin-lock.json. #adr-003 #lockfile #superseded
- [decision] ADR-001 needs amendment: version removed from required fields, minimum fields now just name + description. #adr-001 #amendment
- [decision] Bun postinstall fires on ALL operations (add, remove, update, install) -- empirically verified on v1.3.8. Unlike npm/yarn which skip remove. #bun #lifecycle #verified
- [decision] Full recompute for hook merging: scan all plugin.json files in node_modules, recompute merged hook state from scratch. node_modules IS the state. Under 20ms for 10 plugins. #hooks #merge #stateless
- [decision] Supply chain: restrict node_modules scanning to direct dependencies in package.json only (not transitive). Matches TanStack Intent's actual behavior. #security #supply-chain
- [decision] Rename `mcp` to `mcpServers` in plugin.json -- de facto standard across 6/8 AI platforms, copy-pasteable stdio config #manifest #cross-platform
- [decision] Support `string | string[]` for content paths in plugin.json manifest -- matching Claude Code/Cursor/Copilot convergence pattern #manifest #cross-platform
- [decision] Support inline objects for hooks/mcpServers in plugin.json (not just file paths) #manifest #flexibility
- [decision] Commands are first-class content type, NOT legacy. User-invoked (distinct from agent-invoked skills). #commands #content-types
- [decision] ADR-013 ACCEPTED (Round 2: 5 Accept + 1 D&C) but then SUPERSEDED by design pivot #adr-013 #accepted #superseded
- [decision] DESIGN PIVOT: explicit `agent-plugin add/remove/update` commands replace ADR-013's npm-dependency + postinstall auto-wiring model. User prefers Vercel Skills approach. #distribution #major-pivot
- [decision] ADR-014 to be created to supersede ADR-013's installation model (keep ADR-013 accepted, option C) #adr-impact
- [decision] `.agent-lock.json` lockfile in consuming project: tracks source, sourceType, hash, features, timestamps per plugin. Enables team sync via `agent-plugin install` (restore from lockfile). #lockfile #state-management
- [decision] No consumer-side config file: `.agent-lock.json` + platform configs are the only state. No `.agent-plugin/config.json`. #state-management #simplification
- [decision] Features model: two scopes -- plugin-level (span multiple components) and component-level (single component). Top-level `features` declaration with description, default, requires fields. Per-component `features` mapping features to sections. #features #content-model
- [decision] Section-based feature mechanism for markdown content (skills, agents, commands, AGENTS.md): sections mapped by markdown headers parsed via remark/unified/mdast-util-heading-range. #features #sections #markdown
- [decision] Section-based feature mechanism for code content (hooks): `// #region feature:NAME` / `// #endregion feature:NAME` markers. Custom ~50-100 line parser. #features #sections #code
- [decision] File-based feature mechanism for rules only: individual rule files included/excluded based on feature selection. #features #rules
- [decision] MCP has NO features -- always installed as-is. #mcp #features
- [decision] Plugin manifest: `.agent-plugin/plugin.json` in dedicated directory within plugin package. Follows Claude Code `.claude-plugin/plugin.json` convention. Directory provides disambiguation for generic "plugin.json" name. 3/3 AI platforms (Claude Code, Copilot CLI, Cursor) converged on plugin.json. #manifest #naming
- [decision] Content types: Skills (SKILL.md per skill), Agents, Hooks, Commands, Rules, MCP, AGENTS.md (replaces instructions/), CLI (optional custom impl). #content-types
- [decision] AGENTS.md replaces instructions/ as content type. Plugin-wide instructions via AGENTS.md file. Rules are separate file-based content type. #content-types #instructions
- [decision] Platform-specific metadata lives in plugin.json per component `platforms` field, NOT in content files. Content files are platform-agnostic. #platform-config #content-model
- [decision] Source types: npm packages, git repos (owner/repo shorthand, full URL), local paths. Same as Vercel Skills. #sources #distribution
- [decision] Add flow: resolve source → read .agent-plugin/plugin.json → validate with Zod → detect platforms → feature wizard (clack multiselect) → parse/compile content → write to platforms → update .agent-lock.json → summary. #add-flow #installation
- [decision] Revised command tree: add/remove/update/install/list (consumer) + create/validate/build (author) + mcp serve + content-type groups (skill/agent/mcp/command/hook/rule CRUD) #command-tree
- [decision] `install` command restores all plugins from `.agent-lock.json` (team sync via git, analogous to `npm install` from package-lock.json) #install #team-sync
- [decision] Content-type groups use plural names: skills, agents, commands, hooks, rules. mcp stays singular (abbreviation). Matches CLI conventions (rails, docker, gh). #command-tree #naming
- [decision] `eval` subcommand renamed to `analyze`: clearer intent, avoids JavaScript eval() ambiguity. Read-only quality analysis producing structured report. #commands #rename
- [decision] `improve` subcommand replaced by `analyze --fix` flag: follows universal CLI convention (ESLint --fix, Prettier --write, Biome --fix). Same interactive diff preview behavior. Reduces command tree by 3 entries. #commands #simplification
- [decision] MCP tool catalog deferred to creator skill evaluation: derivation order is creator skills → CLI wizards → MCP tools. Not all CLI commands become MCP tools. Excluded: mcp serve (circular), build (long-running), analyze (invokes AI, circular), create project scaffold (heavy wizard). ~20-25 MCP tools estimated. #mcp #tool-catalog #sequencing
- [fact] No cross-platform AI agent plugin manager exists -- this is confirmed whitespace opportunity (ANALYSIS-036) #market-gap
- [fact] MCP is universal standard: 8/8 platforms support it, mcpServers JSON format identical across 6/8 #mcp #universal
- [fact] AGENTS.md adopted by 6/8 platforms, 60K+ GitHub repos, Linux Foundation governance #standards
- [fact] Style Dictionary is closest prior art for cross-platform adapter pattern (9/10 applicability score) #prior-art #adapters
- [fact] Plugin manifest convergence: Claude Code, Copilot CLI, Cursor independently converged on nearly identical plugin.json schemas #manifest #convergence

---

## Ideation Workflow

### Workflow Per ADR

1. Research topic area (spawn analyst agents)
2. Discuss findings with user one decision at a time
3. Create ADR once decisions are locked in
4. Run brain:adr-review skill (6-agent debate protocol)
5. Save consolidated debate log to critique/DEBATE-ADR-NNN-name
6. Resolve all P0 and P1 issues with user
7. Only then move to the next ADR

### Phases

**Phase 1**: Research and Discovery -- research each group, discuss, create ADRs, run adr-review debates

**Phase 2**: Validation and Consensus -- strategic fit, assumption challenge, completeness review, roadmap priority

**Phase 3**: Feature Specifications -- create feature specs in features/ directory following FEAT-003-filtering template structure:

```text
features/FEAT-NNN-name/
  FEAT-NNN-name.md           # Main feature spec
  requirements/REQ-NNN-NNN   # Per requirement
  design/DESIGN-NNN          # Architecture docs
  tasks/TASK-NNN             # Implementation tasks
```

Template reference: /Users/peter.kloss/Documents/examples/docs/features/FEAT-003-filtering

**Phase 4**: Epic and PRD Creation

**Phase 5**: Implementation Plan Review

---

## Ideation Workflow Status

**Current Position:** Phase 1 ~85% COMPLETE. Groups 1-7 COMPLETE. ADR-014 ACCEPTED (Round 2: 5 Accept + 1 D&C), superseding ADR-013. Decision Audit Reconciliation COMPLETE. Gap Analysis COMPLETE (9 gaps identified, 4 already resolved via existing ADRs, 5 require action). Group 8 IN PROGRESS (3 items narrower than originally scoped). Group 9 MOSTLY DECIDED (3/4 items already covered by existing ADRs/decisions). Phases 2-5 NOT STARTED.

**Remaining Phase 1 Work:**

- Group 8: 2 items need action (author project config details, project context system). MCP tool catalog deferred to creator skill evaluation.
- Group 9: 1 item needs action (implementation phasing)
- 9 identified gaps to resolve (5 actionable, 4 pre-resolved)
- ADR-004 Plugin Security Model creation (CRITICAL: 11 items blocked across 4 active ADRs)
- Stale ADR body text updates (9 instances across active ADRs)
- ADR-006 amendment needed (remark/unified deps from ADR-014 D5)

### Phase 1: Research and Discovery

#### Group 1: Foundation and Vision (Sections 1-3) -- COMPLETE

Research completed:

- [x] [[ANALYSIS-001-agent-plugin-foundation-and-vision]]
- [x] [[ANALYSIS-002-platform-capability-matrix]]
- [x] [[ANALYSIS-003-vercel-skills-format-deep-dive]]
- [x] [[ANALYSIS-004-tanstack-intent-deep-dive]]
- [x] [[ANALYSIS-005-claude-code-plugin-format]]
- [x] [[ANALYSIS-007-config-schema-versioning-patterns]]
- [x] [[ANALYSIS-008-platform-instruction-file-paths]]
- [x] All format discussions complete (plugin.json, installMode, conflicts, namespacing, frontmatter)

Discussion topics completed (one at a time):

- [x] Market positioning: confirmed cross-platform full lifecycle management
- [x] Platform criteria: must support prompts + skills + agents + hooks + MCPs + parallel agents
- [x] Platform list: ALL 7 full-support platforms (Claude Code, Cursor, Copilot CLI, Kiro, OpenCode, Amp, Windsurf)
- [x] Platform instruction file management on install (new requirement)
- [x] Three-audience model: CONFIRMED full three-audience model (Consumer CLI + Author CLI + AI MCP) in single package
- [x] Naming: CONFIRMED @acmelabs-15/agent-plugin (scoped). Need to register acmelabs-15 npm org.
- [x] Self-bootstrapping: CONFIRMED design manifest structure in Phase 1, ship runtime in Phase 4
- [x] Interop: Own format, no hosted registry. Multiple source types. Borrow concepts from Vercel skills SKILL.md standard, extend to agents/prompts/hooks/MCPs
- [x] Deep research: ANALYSIS-003, ANALYSIS-004, ANALYSIS-005 complete
- [x] Format decisions synthesized: plugin.json manifest, installMode, conflict resolution, namespacing, component frontmatter, platform-aware generation, rename tracking
- [x] formatVersion removed after ANALYSIS-007 research (70% of config formats handle evolution without version fields)
- [x] platforms field removed -- all plugins are cross-platform
- [x] No publish command -- no registry
- [x] Source versioning, invalid version handling, installed version tracking
- [x] Platform instruction file paths for all 7 platforms (ANALYSIS-008) + fallback convention

ADR status:

- [x] ADR-001 created, debated via adr-review skill
- [x] DEBATE-ADR-001 saved (2 Accept, 3 D&C, 1 Block)
- [x] ADR-001 P0 resolutions applied: platforms field removed, formatVersion removed, schema evolution strategy added
- [x] ADR-001 COMPLETE
- [x] ADR-002 created, debated via adr-review skill
- [x] DEBATE-ADR-002 saved (0 Accept, 4 D&C, 1 Needs Revision)
- [x] ADR-002 P0-1 resolved: No graceful degradation -- only 6/6 platforms
- [x] ADR-002 P0-2 resolved: No publish command
- [x] ADR-002 P0-3 resolved: NEG-006 added (instruction file injection, deferred to ADR-004)
- [x] ADR-002 P0-4 resolved: NEG-007 added (no centralized vetting, deferred to ADR-004)
- [x] ADR-002 P1-1 resolved: Duplicate ADR already deleted
- [x] ADR-002 P1-4 resolved: Instruction file paths filled in for all 7 platforms + fallback
- [x] ADR-002 P1-5 resolved: Moot -- degradation tier removed
- [ ] ADR-002 P1-2 skipped: Audience priority ordering deferred to Phase 4 (Epic/PRD)
- [ ] ADR-002 P1-3 skipped: Phased delivery deferred to Phase 4 (Epic/PRD)
- [ ] ADR-002 P1-6 skipped: skills.sh -- differentiation already clear
- [x] ADR-002 P1-7 resolved: Self-bootstrapping = self-install + dogfood. Acceptance criteria added to ADR-002.
- [x] ADR-002 P1-8 resolved: Not a real concern for CLI tools. gunshi lazy loading handles it.
- [x] ADR-002 P1-9 skipped: Covered by NEG-006/NEG-007 (ADR-004 scope)
- [x] ADR-002 P1-10 skipped: Covered by NEG-006/NEG-007 (ADR-004 scope)
- [x] ADR-002 P1-11: Instruction file update patterns -- ANALYSIS-009 COMPLETE
- [x] ADR-002 P1-12 resolved: Bidirectional link added to ADR-002 Relations
- [x] ADR-003 created
- [x] ADR-003 adr-review debate: UNANIMOUS NEEDS REVISION (0 Accept, 0 D&C, 6 Needs Revision)
- [x] DEBATE-ADR-003 saved (7 P0, 14 P1, 8 P2)
- [x] ADR-003 P0 Core: Always-namespace adopted (Claude Code model)
- [x] ADR-003 P0-1: Keep as one ADR (not split)
- [x] ADR-003 P0-2: MOOT (always-namespace is automatic, no interactive prompts needed)
- [x] ADR-003 P0-3: MOOT (no renames with always-namespace)
- [x] ADR-003 P0-4: MOOT (always-namespace works for both installModes)
- [x] ADR-003 P0-5: Colon logical-only + kebab-case name validation
- [x] ADR-003 P0-6: Strictest wins for blocking hooks + ANALYSIS-010/012 research
- [x] ADR-003 P0-7: JSON lockfile, re-derive on corruption
- [x] ADR-003 P1-1: MOOT (always-namespace adopted)
- [x] ADR-003 P1-2: Resolved -- overlay/recompute pattern (order-independent)
- [x] ADR-003 P1-3: COVERED by P0-5 (colon logical-only)
- [x] ADR-003 P1-4: RESOLVED -- hybrid D+C pattern with cross-platform concepts + per-platform blocks
- [x] ADR-003 P1-5: MOOT (no renames)
- [x] ADR-003 P1-6: COVERED (colon logical-only, low migration cost)
- [x] ADR-003 P1-7: COVERED by ANALYSIS-009 + ANALYSIS-013 sanitization
- [x] ADR-003 P1-8: COVERED by P0-7 (re-derive from disk)
- [x] ADR-003 P1-9: COVERED by ANALYSIS-009 + ANALYSIS-013 sanitization
- [x] ADR-003 P1-10: MOOT (no prompts with always-namespace)
- [x] ADR-003 P1-11: RESOLVED -- both plugin.json (defaults) AND per-component (overrides), adapter as base
- [x] ADR-003 P1-12: MOOT (no renames)
- [x] ADR-003 P1-13: RESOLVED -- 4-level resolution order (standard < adapter < manifest < component)
- [x] ADR-003 P1-14: Forward reference to ADR-004 added
- [x] ADR-003 ALL P0s RESOLVED, ALL P1s RESOLVED
- [x] ADR-003 REWRITTEN by architect agent incorporating all P0/P1 resolutions into 6-decision structure
- [x] ADR-003 adr-review Round 2: CONSENSUS REACHED (3 Accept, 3 D&C)
- [x] ADR-003 Round 2 P1-R2-1 resolved: ADR-001 Section 9 updated to reference ADR-003
- [x] ADR-003 Round 2 P1-R2-2 resolved: Lockfile at project root (plugin-lock.json) or user home (~/.config/agent-plugin/), no .agent-plugins/ directory
- [x] ADR-003 Round 2 P1-R2-3: deepmerge customMerge dispatch -- noted for implementation (path-context-aware)
- [x] ADR-003 Round 2 P1-R2-4 resolved: NEG-007 added (strictest-wins composability constraint)
- [x] ADR-003 Round 2 P1-R2-5 resolved: File permissions added (lockfile 600, XDG dir 700)
- [x] ADR-003 Round 2 P1-R2-6 resolved: Hook execution blocked until ADR-004 consent model
- [x] ADR-003 COMPLETE

#### Group 2: Core Technology Stack (Section 4 -- Dependencies) -- COMPLETE

Background research running: [[ANALYSIS-006-runtime-cli-framework-validation-stack]] (Bun vs Node/Deno, gunshi vs alternatives, zod vs alternatives)

Each dependency is a SUGGESTION from the spec that needs research and validation:

- [x] Runtime: Bun ADOPTED (vs Node.js, Deno) -- 4-8x faster startup, native TypeScript, Anthropic-owned. Dual distribution: npm primary + optional compiled binaries. [[ANALYSIS-016-bun-runtime-assessment]] #decided
- [x] CLI framework: gunshi ADOPTED (vs commander, yargs, oclif, citty, clipanion) -- built-in lazy loading, explicit Bun support, full TypeScript inference. citty as documented fallback. [[ANALYSIS-017-cli-framework-comparison]] #decided
- [x] Validation: zod v4 full ADOPTED (decided during ADR-003 review) -- org-wide standard, bundle size irrelevant for CLI #decided
- [x] Interactive prompts: @clack/prompts ADOPTED -- built-in wizard flows (group()), spinners, visual framing. Bun stdin risk mitigatable. [[ANALYSIS-018-interactive-prompts-and-colors]] #decided
- [x] Shell completions: gunshi plugin-completion ADOPTED, supports bash/zsh/fish/powershell. [[ANALYSIS-017-cli-framework-comparison]] #decided
- [x] Frontmatter parsing: yaml 2.x ADOPTED (gray-matter DISQUALIFIED: CVE-2025-64718 in js-yaml@^3.13.1, inactive 5 years). Manual 5-10 line parser + Zod validation. [[ANALYSIS-020-frontmatter-and-markdown-processing]] #decided
- [x] Markdown processing: REMOVED -- no markdown-to-HTML needed at launch. Tool extracts frontmatter and passes body as-is to platform adapters. Add micromark later only if concrete use case emerges. [[ANALYSIS-020-frontmatter-and-markdown-processing]] #decided
- [x] Database: REMOVED -- JSON lockfile (ADR-003) is sufficient. drizzle-orm dropped. bun:sqlite available built-in if ever needed later. [[ANALYSIS-021-data-storage-and-search]] #decided
- [x] Full-text search: REMOVED permanently -- Array.filter() on name/description/tags sufficient for 5-20 plugins. @orama/orama dropped. [[ANALYSIS-021-data-storage-and-search]] #decided
- [x] Semantic search: REMOVED permanently -- over-engineering for a plugin manager. AI assistants via MCP already have semantic understanding. @huggingface/transformers dropped. [[ANALYSIS-021-data-storage-and-search]] #decided
- [x] MCP framework: @modelcontextprotocol/sdk ADOPTED (skip fastmcp) -- official SDK, Bun-supported, no wrapper overhead, direct v2 upgrade path. [[ANALYSIS-019-mcp-framework-and-file-watching]] #decided
- [x] File watching: chokidar v5 ADOPTED (watcher as fallback) -- 123M weekly downloads, 0 native deps, ESM-only. Only used for author dev command. [[ANALYSIS-019-mcp-framework-and-file-watching]] #decided
- [x] Colors: picocolors REMOVED -- @clack/prompts v1.1.0 replaced with node:util styleText. Use styleText directly. [[ANALYSIS-018-interactive-prompts-and-colors]] #decided #superseded

#### Group 3: CLI Architecture (Sections 5-6) -- COMPLETE

Research completed:

- [x] [[ANALYSIS-022-clack-prompts-api-surface-and-gaps]] -- All 19 APIs exist in v1.1.0, 6 added in v1.0.0
- [x] [[ANALYSIS-023-gunshi-command-patterns-and-capabilities]] -- lazy loading, 3-level nesting, plugin system for globals
- [x] [[ANALYSIS-024-cli-ci-mode-and-non-interactive-patterns]] -- ci-info recommended, three-layer detection

Decisions made:

- [x] Command tree: `publish` removed (ADR-002), `new mcp init` uses @modelcontextprotocol/sdk + zod (ADR-006), completions via @gunshi/plugin-completion (ADR-006), 7 platforms per ADR-002 #decided
- [x] Command alias: `upgrade` only (update alias dropped to avoid npm naming confusion) #decided
- [x] `--conflict` flag REMOVED: always-namespace (ADR-003) prevents file conflicts. Reinstall/upgrade overwrites own files. #decided
- [x] picocolors REMOVED from dependency stack: @clack/prompts v1.1.0 replaced it with node:util styleText. Use styleText directly. #decided
- [x] CI detection: Use `ci-info` package (zero deps, 50+ vendors). Three-layer: flag > env var > TTY check. #decided
- [x] `--ci` vs `--yes` are DISTINCT flags: --yes auto-confirms (keeps spinners/colors), --ci is full non-interactive (implies --yes). #decided
- [x] Global flags: 5 total (--ci, --yes/-y, --json, --verbose/-v, --quiet/-q). Flag interaction matrix defined. --quiet wins over --verbose. #decided
- [x] Command-specific flags: add (--scope, --platforms, no --conflict), upgrade (--version). --platforms is plural. #decided
- [x] Three-tier input resolution: Interactive → @clack/prompts select/multiselect; CI → error with flag hint; MCP → error with options list for agent self-correction. #decided
- [x] Interactive fallback: No-subcommand shows context-aware p.select() menu. Author project (plugin.json present) → author commands first. #decided
- [x] 16 validation rules: 3 naming, 3 uniqueness, 3 paths/patterns (glob not regex), 3 misc, 4 source format (npm/git HTTPS/git shorthand/local) #decided
- [x] @clack/prompts: 12 APIs tested on Bun 1.3.8 (PASS), 6 v1.0.0 additions source-confirmed (Bun verification pending) #verified
- [x] MCP error response schema: 5 typed error codes, CWE-209 sanitization, options for agent self-correction #decided
- [x] JSON response envelope: ok/data/error with semver stability guarantee #decided
- [x] Exit codes: 0 success, 1 runtime error, 2 usage error #decided
- [x] Hook matchers restricted to glob patterns (ReDoS prevention, CWE-1333) #decided
- [x] p.password() removed from component mapping (no use case) #decided

ADR status:

- [x] ADR-007 created, debated via adr-review skill (6-agent debate)
- [x] DEBATE-ADR-007 saved (Round 1: 6/6 ACCEPT_WITH_CONDITIONS; 4 P0, 19 P1)
- [x] ADR-007 P0/P1 resolutions applied (MCP schema, JSON envelope, 12/12 count, source taxonomy, flag matrix, glob matchers, exit codes, picocolors note, update alias dropped, testing matrix, p.password() removed)
- [x] ADR-007 Round 2: UNANIMOUS ACCEPT (6/6), 3 D&C from independent thinker
- [x] ADR-007 COMPLETE

#### Group 4: Source and Platform (Sections 7-9) -- COMPLETE

Research completed:

- [x] [[ANALYSIS-025-source-resolution-patterns]] -- npm/GitHub/local resolution, semver (Bun.semver), staging, manifest discovery
- [x] [[ANALYSIS-026-platform-detection-and-mapping]] -- detection patterns, config dirs, content mapping for all 7 platforms
- [x] [[ANALYSIS-027-installation-mechanics]] -- install scope, MCP merging, dependency management, uninstall/upgrade

All 11 discussion topics decided:

- [x] Manifest discovery: plugin.json only (no package.json fallback, per ADR-001)
- [x] Bare `owner/repo` ambiguity: smart detection (check filesystem first, fall back to GitHub)
- [x] Bun built-in API reliability: Bun.semver (use) + tar npm package (instead of Bun.Archive) + Bun.write arrayBuffer workaround
- [x] Platform detection: dual (binary + config dir) in parallel via Promise.allSettled + Bun.spawn with 3s timeout
- [x] MCP key namespacing: colon separator (`plugin-name:server-name`), matching Claude Code internal convention and ADR-003 component identifier pattern
- [x] Install scope: project scope default. No project detected and no --global flag = prompt user via @clack/prompts confirm. Error in non-interactive mode.
- [x] System/platform dependencies: agent-plugin CAN install platform CLIs and system deps with explicit user confirmation via @clack/prompts. Tracked in plugin-lock.json for uninstall.
- [x] Installation flow: 6-phase model (Detect, Select, Resolve, Confirm, Apply, Record). User selects platforms via multiselect. Missing platforms treated as installable deps.
- [x] Upgrade strategy: interactive -- scan installed, check versions, multiselect `plugin current -> latest`, atomic replace with rollback on failure.
- [x] Platform config mapping: all platform-specific mapping in `platforms.config.json` at project root. Plugin authors write platform-agnostic plugin.json. Pure data file.
- [x] Optional CLI generation from MCP tools: `cli` field in plugin.json ("auto" | path | omitted). v1: flat subcommands, author `cli.groups` for grouping, all-or-nothing install. Built-in `mcp` start/stop/restart/status commands. Daemon/stdio transport. Binary safety denylist. Parameter type mapping. Trust model.

ADR status:

- [x] ADR-008 Source Resolution and Package Validation: created, debated (Round 1: 3 Accept, 3 D&C), P0/P1 fixes applied, ACCEPTED
- [x] ADR-009 Platform Detection and Config Registry: created, debated (Round 1: NEEDS REVISION), P0/P1 fixes applied, Round 2 (5 Accept, 1 D&C), ACCEPTED
- [x] ADR-010 Installation Lifecycle: created, debated (Round 1: NEEDS REVISION with 1 BLOCK), P0/P1 fixes applied, CLI generation extracted to ADR-011, Round 2 (5 Accept, 1 D&C), ACCEPTED
- [x] ADR-011 Auto-Generated CLI from MCP Tools: created (extracted from ADR-010 Decision 5), debated (Round 1: unanimous NEEDS REVISION), 7 P0s resolved (flat commands, MCP lifecycle, binary safety, trust model, path containment, prior art, type mapping), Round 2 (5 Accept, 1 D&C), ACCEPTED
- [x] DEBATE-ADR-008, DEBATE-ADR-009, DEBATE-ADR-010, DEBATE-ADR-011 all saved to critique/
- [x] Group 4 COMPLETE: All 4 ADRs accepted

#### Group 5: Data and Storage (Sections 10-11) -- COMPLETE (covered by existing decisions)

- [x] Storage approach: JSON lockfile adopted in ADR-003 Decision 3. SQLite/drizzle-orm removed (ANALYSIS-021). bun:sqlite as deferred upgrade path.
- [x] Schema design: lockfile schema in ADR-003 (plugin inventory, component registry, hook overlays, file modification history). ADR-010 adds installedDeps. ADR-011 adds CLI command selections.
- [x] Search: Array.filter() sufficient for 5-20 plugins (ANALYSIS-021). @orama/orama deferred to Phase 4. @huggingface/transformers permanently removed.
- [x] MCP tool registry: on-demand (read from MCP servers transiently, not persisted in lockfile). Avoids staleness risk. Defer persistence to v2 if latency is a problem.
- [x] Dual-location manifest: NO. plugin.json only (ADR-001). No embedded package.json field. Simplifies discovery and validation.

#### Group 6: Scaffolding Wizards (Section 12) -- COMPLETE

Research completed:

- [x] [[ANALYSIS-031-scaffolding-wizard-patterns]] -- Template strategies, @clack/prompts group() patterns, wizard specs for all 7 content types
- [x] [[ANALYSIS-032-mcp-sdk-bun-runtime-compatibility]] -- Verified MCP SDK + Zod v4 work 100% on Bun runtime

All decisions discussed one at a time with user:

- [x] Template rendering: yaml 2.x stringify + tagged template literals (zero new deps, gray-matter disqualified by CVE-2025-64718)
- [x] Schema-first architecture: single Zod v4 schema per wizard drives interactive prompts, CI flags, and MCP tool parameters
- [x] Multi-group wizard flow: sequential group() calls with p.log.step() separators (agent create: 4 groups, mcp create: 2 groups)
- [x] Manifest auto-update: all create wizards auto-update plugin.json with flat array naming matching ADR-001
- [x] MCP server scaffolding: @modelcontextprotocol/sdk with separate tool files and marker comments for create-tool patching
- [x] Content-type command groups replace ADR-007 `new` subtree (skill/agent/mcp/command/hook/instruction with subcommands)
- [x] Bundled creator skills: skill-creator, agent-creator, mcp-builder (adapted from Anthropic sources), instruction-evaluator (original)
- [x] eval/improve split: eval = read-only analysis with report, improve = interactive diff preview with apply modes
- [x] Instructions replace rules: rules/ → instructions/, merge into AGENTS.md (6/7 platforms) + CLAUDE.md
- [x] Wizard simplifications: emoji prefix dropped, autoLoadAgents dropped, skill frontmatter aligned with Claude Code SKILL.md fields
- [x] Interactive improve preview: color-coded diff, "Why" annotations, three apply modes (apply all, review one-by-one, skip)
- [x] Remove confirmation: p.confirm() showing files, manifest entries, and dependent content. --yes flag for CI.
- [x] Trust model: eval/improve executes bundled first-party Bun TypeScript scripts (not user code). HTML viewer via Bun.serve(). Shared viewer across all eval subcommands.
- [x] Faithfulness principle: creator skills stay close to Anthropic implementations. Exceptions: Python→Bun TS, fastmcp→SDK, minor verbiage tweaks.
- [x] Commands restored as content type: ADR-001 amendment needed (IMP-005) to add commands as 6th component
- [x] Zod v4 + MCP SDK verified compatible: min SDK >= 1.23.0, Zod >= 4.1.13 (ANALYSIS-032)

ADR status:

- [x] ADR-012 Scaffolding and Content Management created (11 decisions)
- [x] ADR-012 adr-review Round 1: unanimous NEEDS REVISION (0 Accept, 6 Needs Rev). 7 consolidated P0 issues.
- [x] P0-1 resolved: MCP lifecycle commands (start/stop/restart/status) removed from author CLI (consumer-only, ADR-011)
- [x] P0-2 resolved: content.* nesting replaced with flat array names matching ADR-001
- [x] P0-3 resolved: Commands kept as content type, IMP-005 tracks ADR-001 amendment
- [x] P0-4 resolved: IMP-006 notes creator skills independently shippable, phasing deferred to project plan
- [x] P0-5 resolved: IMP-007 tracks ANALYSIS-031 gray-matter cleanup
- [x] P0-6 resolved: Decision 11 added with trust model, HTML viewer, faithfulness principle
- [x] P0-7 resolved: Verified compatible, minimum versions documented, ANALYSIS-032 created
- [x] ADR-012 adr-review Round 2: ACCEPTED (5 Accept, 1 D&C). All P0s resolved.
- [x] DEBATE-ADR-012 saved with both rounds
- [x] ADR-012 COMPLETE

#### Group 7: Commands and npm Distribution (Sections 13-14) -- COMPLETE

Research completed:

- [x] [[ANALYSIS-033-consumer-and-author-commands]] -- Consumer/author command analysis, npm-package distribution model research
- [x] [[ANALYSIS-034-skill-versioning-models-comparison]] -- TanStack Intent model, npm-package distribution patterns
- [x] [[ANALYSIS-035-hook-merging-strategies-without-custom-lockfile]] -- 6 strategies compared, full recompute recommended (<20ms for 10 plugins)
- [x] [[ANALYSIS-036-ai-platform-plugin-standards-survey]] -- MCP 8/8, AGENTS.md 6/8, no cross-platform plugin manager exists
- [x] [[ANALYSIS-037-cross-platform-hook-configuration-analysis]] -- 6 universal hook events, naming divergence, converging stdin/stdout/exit-code protocol
- [x] [[ANALYSIS-038-cross-platform-agent-and-skill-definitions]] -- 6 universal agent fields, 3 platform tiers
- [x] [[ANALYSIS-039-cross-platform-mcp-and-instruction-formats]] -- MCP stdio config copy-pasteable across 6/8 platforms, instruction file divergence
- [x] [[ANALYSIS-040-prior-art-for-cross-platform-adapter-patterns]] -- Style Dictionary closest match (9/10), transform pipeline validated
- [x] [[ANALYSIS-041-plugin-manifest-cross-platform-comparison]] -- 90% aligned with Claude Code/Copilot/Cursor convergence

Major architectural pivot: npm-package distribution model adopted (TanStack Intent style)

- [x] Consumer commands (add, remove, upgrade) ELIMINATED -- bun handles package management natively
- [x] `init` redesigned to Husky model: lifecycle hooks in package.json (`"postinstall": "agent-plugin install"`)
- [x] `install` is core wiring command: scan node_modules, diff plugins, reconcile platform configs
- [x] `create` command ADDED: scaffold new plugin project (like `bun create`)
- [x] `deinit` command ADDED: reverse of init, removes lifecycle hooks
- [x] `dev` command REMOVED: not needed with lifecycle hook model
- [x] `complete` command REMOVED: shell completions not needed
- [x] `publish` command CONFIRMED REMOVED: no registry, no version command
- [x] version field removed from plugin.json required fields (package.json is authoritative)
- [x] plugin.json minimum required fields reduced to just `name` and `description`
- [x] ADR-008 to be fully superseded (bun handles source resolution)
- [x] ADR-010 to be mostly superseded (6-phase model replaced by bun add + install wiring)
- [x] ADR-003 Decision 3 lockfile to be superseded (bun.lockb replaces plugin-lock.json)
- [x] ADR-001 to be amended (version removed from required fields)
- [x] ADR-013 created (npm-Package Distribution and Revised Command Tree, 6 decisions)
- [x] ADR-013 adr-review Round 1: unanimous NEEDS REVISION (0 Accept, 6 Needs Rev). 3 P0, 11 P1, 7 P2.
- [x] DEBATE-ADR-013 saved to critique/
- [x] ADR-013 adr-review Round 2: ACCEPTED (5 Accept, 1 D&C). 0 P0, 0 P1, 7 P2 (non-blocking).
- [x] ADR-013 status changed from "proposed" to "accepted" (2026-03-09)
- [x] ADR-013 COMPLETE (but to be superseded by ADR-014 due to design pivot)
- [x] Design pivot: explicit add/remove/update commands replace postinstall auto-wiring
- [x] Features model, .agent-plugin/plugin.json manifest, .agent-lock.json lockfile, content types, section mechanisms, command tree all settled
- [x] ANALYSIS-047 Project Bootstrapping captured as pending workstream

ADR-013 P0 resolutions (applied in Round 2 revision):

- [x] P0-1 RESOLVED: Bun postinstall empirically verified on v1.3.8 -- fires on ALL operations (add, remove, update, install). ADR claim is correct. No change needed.
- [x] P0-2 RESOLVED: Full recompute from plugin.json files in node_modules. No lockfile needed for hook tracking. ANALYSIS-035 validates <20ms for 10 plugins.
- [x] P0-3 RESOLVED: Restrict scanning to direct dependencies in package.json only (not transitive). Resolves supply chain risk.

ADR-013 P1 resolutions (all resolved via Round 2 revision):

- [x] P1-1 agreed: Add acknowledgment that ANALYSIS-034 recommended against TanStack model, document why overridden
- [x] P1-2: Husky uses `prepare` not `postinstall` -- corrected in ADR-013 revision
- [x] P1-3: Missing MADR Confirmation section -- added in Round 2
- [x] P1-4: Missing Pros/Cons for rejected options -- added in Round 2
- [x] P1-5: Dev workflow gap -- addressed (file watching not needed with explicit add model)
- [x] P1-6: "60% complexity reduction" unsourced -- removed rhetorical estimate
- [x] P1-7: plugin.json name vs package.json name dual-source -- precedence defined
- [x] P1-8: Path validation for plugin.json content -- validation rules added
- [x] P1-9: Wiring change output/diff visibility -- summary output required
- [x] P1-10: `bun update` postinstall behavior -- empirically verified on v1.3.8
- [x] P1-11: Isolated linker compatibility -- scanning follows symlinks

ADR-013 Round 2 (convergence vote):

- [x] Round 2: CONSENSUS REACHED (5 Accept, 1 D&C). 0 P0, 0 P1. 7 P2 (non-blocking).
- [x] ADR-013 status changed from "proposed" to "accepted" (2026-03-09)
- [x] DEBATE-ADR-013 updated with Round 2 verdicts, P2 issues, D&C reservations
- [x] Independent Thinker D&C: standalone content authors concern, Bun postinstall stability concern

Cross-platform alignment decisions (from ANALYSIS-036 through 041):

- [x] Rename `mcp` to `mcpServers` in plugin.json (de facto standard across 6/8 platforms)
- [x] Support `string | string[]` for paths in manifest (matching Claude Code/Cursor/Copilot patterns)
- [x] Support inline objects for hooks/mcpServers in addition to file paths
- [x] Commands confirmed as first-class content type (NOT legacy, distinct from agent-invoked skills)
- [x] Rules vs AGENTS.md scope: rules are file-based (included/excluded per feature), AGENTS.md is section-based
- [x] installMode replaced by features model: plugin-level and component-level features with section-based cherry-picking
- [x] Native platform plugin installation: deferred to v2 (explicit add/remove/update model first)

Major design pivot (post ADR-013 acceptance):

- [x] Moved AWAY from ADR-013's npm-dependency + postinstall auto-wiring model
- [x] Adopted explicit `agent-plugin add/remove/update` command model (Vercel Skills style)
- [x] ADR-014 to be created to supersede ADR-013's installation model
- [x] `.agent-lock.json` lockfile in consuming project tracks installed plugins (source, sourceType, hash, features, timestamps)
- [x] `agent-plugin install` restores from lockfile (team sync via git)
- [x] Vercel Skills CLI deeply analyzed (lock files, installer, remove, list commands)
- [x] Features model designed: two scopes (plugin-level, component-level), section-based mechanism for markdown+code, file-based for rules, none for MCP
- [x] Plugin manifest: `.agent-plugin/plugin.json` (follows Claude Code `.claude-plugin/plugin.json` convention, directory provides disambiguation)
- [x] Content types: Skills, Agents, Hooks, Commands, Rules, MCP, AGENTS.md (replaces instructions/), CLI (optional)
- [x] Section parsing: markdown headers via remark/unified/mdast-util-heading-range for markdown, `// #region feature:NAME` for code files
- [x] Platform-specific metadata in plugin.json per component `platforms` field, NOT in content files
- [x] No consumer-side config file: `.agent-lock.json` + platform configs are the only state
- [x] Source types: npm packages, git repos (owner/repo, full URL), local paths
- [x] Add flow: resolve source → read plugin.json → detect platforms → feature wizard → parse/compile content → write to platforms → update lockfile → summary
- [x] ANALYSIS-047 created as pending workstream: project bootstrapping (Biome, Bun, Turbo, releases, CI/CD, GitHub config, testing)

#### Decision Audit Reconciliation -- COMPLETE

Source: `docs/analysis/RECONCILIATION-2026-03-09-decision-audit.md`
6 parallel extraction agents read 22,969 lines of conversation log, reconciliation agent resolved reversals chronologically.

**P0 Tasks (Critical -- design decisions exist but have no ADR)**:

- [x] P0-1: Create ADR-014 (785 lines, 8 decisions). adr-review completed: Round 2 consensus (5 Accept + 1 D&C). Status: Accepted.
- [x] P0-2: ADR-001 updated with amendments #6 (manifest location), #7 (content types 6→8), #8 (installMode→features). Duplicate Section 9 removed. ADR-014 relation added.
- [x] P0-3: ADR-013 status changed to "superseded" with full explanation referencing ADR-014.

**P1 Tasks (Missing analysis/documentation)**:

- [x] P1-1: ANALYSIS-043 updated with "Adoption Status" section noting ADR-014 adopted recommendations. ADR-014 relation added.
- [x] P1-2: ANALYSIS-044 updated with "Adoption Status" section noting ADR-014 D5 adopted section-based selection. ADR-014 relation added.
- [x] P1-3: Covered by ADR-014 D2 (lockfile schema). Standalone note not needed.
- [x] P1-4: Covered by ADR-014 D3/D4/D8 (plugin.json schema, content types, platform metadata). Standalone note not needed.
- [x] P1-5: Covered by ADR-014 D7 (8-step add flow). Standalone note not needed.
- [ ] P1-6: Vercel Skills CLI research note (nice to have, not blocking)
- [x] P1-7: ADR-003 Decision 3 supersession note updated to reference ADR-014 (.agent-lock.json, independent per-plugin hooks). ADR-014 relation added.
- [x] P1-8: ANALYSIS-042 updated with "Adoption Status" section confirming deferral. ADR-014 relation added.

**P2 Tasks (Housekeeping)**:

- [x] P2-1: Update session note with reconciliation results and completion status (this update)
- [x] P2-2: Commit all changes (committed as 7ccbbb1 and subsequent commits)
- [x] P2-3: ADR-008/010 supersession chain update -- ADR-008 SUPERSEDED by ADR-014 (via ADR-013), ADR-010 SUPERSEDED by ADR-014 (via ADR-013), ADR-013 SUPERSEDED by ADR-014. Chain: ADR-008/010 → ADR-013 → ADR-014.

#### Comprehensive Gap Analysis -- COMPLETE (2026-03-09)

Source: 3 parallel 🧠:analyst agents performed exhaustive cross-reference:

1. **Spec Inventory Agent**: Read all 22 sections of design spec, produced 594 decision points with S{N}-{M} identifiers
2. **ADR Audit Agent**: Read all 13 ADRs exhaustively (every decision, deferral, forward reference, amendment, supersession)
3. **Cross-Reference Agent**: Section-by-section verdicts + 9 specific gaps identified

**ADR Status Summary (13 ADRs):**

| ADR | Status | Notes |
|-----|--------|-------|
| ADR-001 | ACCEPTED | 8 amendments applied. Body text stale (still references root manifest, installMode). |
| ADR-002 | ACCEPTED | npm org registration still open (IMP-003). |
| ADR-003 | ACCEPTED (partially superseded) | D2/D3 superseded by ADR-014. Body text references plugin-lock.json (now .agent-lock.json). |
| ADR-004 | **DOES NOT EXIST** | Forward-referenced by ADR-002, ADR-003, ADR-006, ADR-014. **11 security items blocked.** |
| ADR-005 | ACCEPTED | @clack/prompts blocking gate partially resolved. |
| ADR-006 | ACCEPTED | **Needs amendment**: remark/unified/mdast-util-heading-range deps from ADR-014 D5 not listed. |
| ADR-007 | ACCEPTED | Command tree body stale (references pre-pivot commands). |
| ADR-008 | **SUPERSEDED** | By ADR-014 via ADR-013. Bun handles source resolution. |
| ADR-009 | ACCEPTED | References unspecified lockfile format. |
| ADR-010 | **SUPERSEDED** | By ADR-014 via ADR-013. 6-phase model replaced. |
| ADR-011 | ACCEPTED | Depends on superseded ADR-010 for installation context. |
| ADR-012 | ACCEPTED | References instructions/ (now rules/ per ADR-014). |
| ADR-013 | **SUPERSEDED** | By ADR-014. Historical record. |
| ADR-014 | ACCEPTED | Round 2: 5 Accept + 1 D&C. Current authoritative design. |

**Supersession Chains:**

- ADR-008 → ADR-013 → ADR-014
- ADR-010 → ADR-013 → ADR-014
- ADR-001 D5 (installMode) → ADR-014 D5 (features)
- ADR-003 D2/D3 (lockfile) → ADR-014 D2/D7 (.agent-lock.json)

**9 Identified Gaps:**

| Gap ID | Severity | Description | Status |
|--------|----------|-------------|--------|
| GAP-1 | LOW | Install scope (project vs global) -- spec S9 default scope unclear | PRE-RESOLVED: Decided in Group 4 discussions (project default, prompt when no project, error in CI). Tracked in ADR-010/ANALYSIS-027. |
| GAP-2 | MEDIUM | MCP tool catalog -- spec S15 lists 14 tools, no ADR specifies which tools | DEFERRED: Tool catalog depends on creator skill evaluation. Derivation order: skills → CLI → MCP tools. ADR-014 Amendment #2 records this. |
| GAP-3 | LOW | MCP naming prefix -- spec says `ap:` prefix for tool names | PRE-RESOLVED: Colon separator adopted (ADR-003, ADR-009) with `plugin-name:server-name` pattern |
| GAP-4 | **CRITICAL** | ADR-004 Plugin Security Model -- 11 items blocked across 4 ADRs | OPEN: Hook execution blocked, instruction file injection deferred, consent model missing |
| GAP-5 | LOW | Hook event types -- spec S9 lists specific events | PRE-RESOLVED: ANALYSIS-037 identified 6 universal hook events across platforms |
| GAP-6/7 | MEDIUM | Author project config -- spec S17 Biome, testing, monorepo layout | OPEN: ANALYSIS-047 created but not started. Biome, testing, monorepo decisions pending. |
| GAP-8 | MEDIUM | Project context system -- spec S18 state detection, config resolution | OPEN: No ADR covers project context detection and workspace awareness |
| GAP-9 | LOW | Implementation phasing -- spec S22 proposes 5 phases | OPEN: Naturally deferred to Phase 4 (Epic/PRD). ADR-002 P1-2/P1-3 also deferred. |

**GAP-4 Detail: ADR-004 Plugin Security Model (CRITICAL)**

ADR-004 is forward-referenced by 4 active ADRs but does not exist:

- **ADR-002**: NEG-006 (instruction file injection attack vector), NEG-007 (no centralized vetting in multi-source model)
- **ADR-003**: NEG-007 (strictest-wins composability constraint), P1-14 (hook security forward reference), Decision 2 (hook execution BLOCKED until ADR-004 consent model)
- **ADR-006**: Deferred security items referencing ADR-004
- **ADR-014**: D6 (security and trust model forward references)

Items blocked until ADR-004 exists:

1. Hook execution (stored but NOT executable)
2. Instruction file injection mitigation
3. Multi-source vetting model
4. Content hashing/integrity
5. MCP access control
6. Consent model for plugin permissions
7. Strictest-wins composability resolution
8. Shell-quote sanitization enforcement
9. Trust boundaries between plugins
10. Privilege escalation prevention
11. Supply chain verification beyond direct deps

**Stale ADR Body Text (9 instances):**

These ADRs have correct amendments/status but body text not updated to reflect changes:

| ADR | Stale Content | Correct Source |
|-----|---------------|----------------|
| ADR-001 | References `plugin.json` at root (not `.agent-plugin/plugin.json`) | ADR-014 D3, Amendment #6 |
| ADR-001 | References `installMode` field | ADR-014 D5, Amendment #8 |
| ADR-001 | Lists 6 content types (not 8) | ADR-014 D4, Amendment #7 |
| ADR-003 | References `plugin-lock.json` | ADR-014 D2 (.agent-lock.json) |
| ADR-007 | Command tree includes pre-pivot commands (add/remove/upgrade) | ADR-014 D1 (revised command tree) |
| ADR-009 | References unspecified lockfile | ADR-014 D2 (.agent-lock.json) |
| ADR-011 | References ADR-010 installation context | ADR-010 superseded by ADR-014 |
| ADR-012 | References `instructions/` directory | ADR-014 D4 (rules/ + AGENTS.md) |
| ADR-006 | Missing remark/unified/mdast-util-heading-range | ADR-014 D5 section-based parsing |

**Pending ADR Amendment:**

- [ ] ADR-006 needs amendment to add remark/unified/mdast-util-heading-range as dependencies (required by ADR-014 D5 section-based feature parsing)

**Open IMP-NNN Items (~60 across active ADRs):**

Implementation notes tracked within ADRs. Not blocking Phase 1 ideation but must be resolved before implementation. Key examples:

- IMP-003 (ADR-002): Register `acmelabs-15` npm org at npmjs.com
- IMP-005 (ADR-012): ADR-001 amendment to add commands as content type (DONE via Amendment #7)
- IMP-006 (ADR-012): Creator skills independently shippable, phasing deferred
- IMP-007 (ADR-008/012): Various cleanup items (gray-matter refs, zip slip, integrity verification)

#### Group 8: MCP and Self-Bootstrap (Sections 15-18) -- IN PROGRESS (~60% decided)

Spec sections 15-18 cover embedded MCP server, self-bootstrapping, author project layout, and project context system. Gap analysis shows most architectural decisions already made via ADR-014 and earlier ADRs; remaining items are narrower than originally scoped.

Already decided (covered by existing ADRs/decisions):

- [x] MCP server framework: @modelcontextprotocol/sdk adopted (ADR-006, ANALYSIS-019). Skip fastmcp. Direct Bun support verified (ANALYSIS-032).
- [x] MCP server transport: stdio for AI platform clients, daemon for CLI (ADR-011)
- [x] MCP tool naming: colon separator `plugin-name:server-name` (ADR-003, ADR-009)
- [x] MCP daemon lifecycle: built-in start/stop/restart/status commands (ADR-011)
- [x] Self-bootstrapping strategy: design manifest Phase 1, ship runtime Phase 4. Self-install + dogfood (ADR-002 acceptance criteria).
- [x] Plugin manifest location: `.agent-plugin/plugin.json` (ADR-014 D3)
- [x] Content types: Skills, Agents, Hooks, Commands, Rules, MCP, AGENTS.md, CLI (ADR-014 D4)
- [x] Author project layout: `.agent-plugin/plugin.json` at root, content in skills/, agents/, hooks/, commands/, cli/, mcp/, AGENTS.md, rules/ (ADR-014 D3/D4)

Remaining items needing research/decisions:

- [~] **GAP-2**: MCP tool catalog -- DEFERRED to creator skill evaluation. Derivation order: creator skills → CLI wizards → MCP tools. ADR-014 Amendment #2 records this decision. ~20-25 MCP tools estimated from 32-command CLI tree (excluding mcp serve, build, analyze, create project scaffold).
- [ ] **GAP-6/7**: Author project config details -- spec Section 17 describes Biome config, testing setup, monorepo layout. No ADR covers this. ANALYSIS-047 created as pending workstream but not started. Needs: research + decisions on project scaffolding details.
- [ ] **GAP-8**: Project context system -- spec Section 18 describes project state detection, config resolution, workspace awareness. No ADR covers this. Needs: research on how agent-plugin detects project context (plugin.json presence, platform detection scope, workspace root finding).

#### Group 9: Polish (Sections 19-22) -- MOSTLY DECIDED (~75% decided)

Spec sections 19-22 cover shell completions, Bun-native APIs, markdown processing, and implementation phasing. Gap analysis shows 3/4 items already decided.

Already decided (covered by existing ADRs/decisions):

- [x] Shell completions: @gunshi/plugin-completion adopted for bash/zsh/fish/powershell (ADR-006, ADR-007). @bomb.sh/tab from spec replaced.
- [x] Bun-native APIs: Bun.semver adopted, Bun.write for downloads (with arrayBuffer workaround), tar npm package for extraction (Bun.Archive too young). Platform detection via Bun.spawn. (ADR-005, ADR-008, ANALYSIS-028)
- [x] Markdown processing: REMOVED from MVP. No markdown-to-HTML rendering needed. Frontmatter: yaml 2.x + Zod v4 validation. Body passed as-is. gray-matter disqualified (CVE). (ADR-006, ANALYSIS-020). NOTE: ADR-014 D5 introduces remark/unified/mdast-util-heading-range for section-based feature parsing -- this is NOT the same as the spec's markdown processing pipeline (which was about rendering).

Remaining items needing research/decisions:

- [ ] **GAP-9**: Implementation phasing -- spec Section 22 proposes 5 phases. No ADR covers phasing. Deferred ADR-002 items P1-2 (audience priority) and P1-3 (phased delivery) are relevant. Needs: phasing plan aligned with revised architecture (ADR-014 model vs spec's original model). This is also a natural Phase 4 (Epic/PRD) deliverable.

### Remaining Phase 1 Work (Sequenced)

**Step 1: Resolve open gaps (Groups 8 + 9 remaining items)**

These are the specific items that need research and/or decisions before Phase 1 is complete:

1. [x] **GAP-2** (MEDIUM): MCP tool catalog -- DEFERRED to creator skill evaluation (ADR-014 Amendment #2). Derivation: creator skills → CLI wizards → MCP tools. Not blocked for Phase 1 completion.
2. [ ] **GAP-8** (MEDIUM): Project context system -- research spec S18 (state detection, config resolution, workspace awareness). Decide how agent-plugin detects project context. May need analysis note + ADR decisions.
3. [ ] **GAP-6/7** (MEDIUM): Author project config -- advance ANALYSIS-047 workstream (Biome, testing, monorepo layout, CI/CD). Research + decisions needed.
4. [ ] **GAP-9** (LOW): Implementation phasing -- can be partially deferred to Phase 4 (Epic/PRD), but high-level phasing alignment with ADR-014 architecture should be confirmed.

**Step 2: Create ADR-004 Plugin Security Model (CRITICAL)**

5. [ ] **GAP-4** (CRITICAL): Research security model requirements. 11 items blocked across 4 active ADRs. Must cover:
   - Hook execution consent model (currently BLOCKED)
   - Instruction file injection mitigation (NEG-006 from ADR-002)
   - Multi-source vetting model (NEG-007 from ADR-002)
   - Content hashing and integrity verification
   - MCP access control
   - Plugin permission boundaries
   - Trust model for third-party plugins
   - Supply chain verification
   - Shell-quote sanitization enforcement
   - Privilege escalation prevention
   - Strictest-wins composability resolution

**Step 3: ADR housekeeping (non-blocking but important)**

6. [ ] ADR-006 amendment: add remark, unified, mdast-util-heading-range as dependencies (required by ADR-014 D5)
7. [ ] Stale ADR body text updates (9 instances -- see gap analysis table above)
8. [ ] IMP-007 (ADR-012): Update ANALYSIS-031 to remove stale gray-matter references
9. [ ] P1-6 (Reconciliation): Vercel Skills CLI research note (nice to have)

**Step 4: Phase 1 completion criteria**

All of the above resolved → Phase 1 COMPLETE. Ready for Phase 2.

### Phase 2: Validation and Consensus -- NOT STARTED

- [ ] Strategic fit assessment (high-level-advisor)
- [ ] Assumption challenge (independent-thinker)
- [ ] Research completeness review (critic)
- [ ] Roadmap priority assessment (roadmap)

### Phase 3: Feature Specifications -- NOT STARTED

- [ ] Create feature specs in features/ directory following FEAT-003-filtering template
- [ ] Each feature gets: FEAT-NNN-name.md + requirements/ + design/ + tasks/
- [ ] Template: /Users/peter.kloss/Documents/examples/docs/features/FEAT-003-filtering

### Phase 4: Epic and PRD Creation -- NOT STARTED

- [ ] Epic creation (roadmap)
- [ ] PRD creation (explainer)
- [ ] Task generation (task-generator)

### Phase 5: Implementation Plan Review -- NOT STARTED

- [ ] Architecture review (architect)
- [ ] DevOps review (devops)
- [ ] Security review (security)
- [ ] QA review (qa)

---

## Work Log

### Session Initialization

- [x] [fact] Set agent-plugin as active Brain project #bootstrap
- [x] [fact] Created session via mcp session tool #session-lifecycle
- [x] [fact] Read complete design spec (22 sections, ~1200 lines) from /Users/peter.kloss/Downloads/agent-plugin-design-spec.md #spec-review
- [x] [fact] Read session template from prior AG Grid project #template
- [x] [fact] Read FEAT-003 feature spec template (FEAT.md + requirements/ + design/ + tasks/) #template

### Phase 1: Research and Discovery

- [x] [research] Foundation and vision analysis -- [[ANALYSIS-001-agent-plugin-foundation-and-vision]] #ecosystem #vision #naming
- [x] [research] Platform capability matrix -- [[ANALYSIS-002-platform-capability-matrix]] #platforms
- [x] [research] Vercel skills format deep dive -- [[ANALYSIS-003-vercel-skills-format-deep-dive]] #format #skills
- [x] [research] TanStack Intent deep dive -- [[ANALYSIS-004-tanstack-intent-deep-dive]] #format #authoring
- [x] [research] Claude Code plugin format -- [[ANALYSIS-005-claude-code-plugin-format]] #format #bundle-model
- [x] [research] Config schema versioning patterns -- [[ANALYSIS-007-config-schema-versioning-patterns]] #versioning #manifest
- [x] [research] Platform instruction file paths -- [[ANALYSIS-008-platform-instruction-file-paths]] #platforms #file-paths
- [x] [research] Runtime/CLI/validation stack -- [[ANALYSIS-006-runtime-cli-framework-validation-stack]] #dependencies (stale, superseded by ANALYSIS-016 through 021)

### Group 1 Discussion

- [x] [decision] Cross-platform full lifecycle management confirmed as target positioning #market
- [x] [decision] Platform criteria locked: prompts + skills + agents + hooks + MCPs + parallel agents #platforms
- [x] [decision] All 7 full-support platforms targeted #platforms
- [x] [decision] Platform instruction file management required on install #new-requirement
- [x] [decision] Full three-audience model confirmed (Consumer + Author + AI) #audiences
- [x] [decision] Package name @acmelabs-15/agent-plugin confirmed, npm org registration needed #naming
- [x] [decision] Self-bootstrapping: design Phase 1, ship Phase 4 #architecture
- [x] [decision] Own format, no registry, multiple source types #distribution
- [x] [decision] Plugin = bundle model with plugin.json manifest #format
- [x] [decision] installMode, conflict resolution, namespacing, frontmatter decisions synthesized #format
- [x] [decision] platforms field removed from plugin.json #manifest
- [x] [decision] formatVersion removed after ANALYSIS-007 research #manifest
- [x] [decision] No publish command #distribution
- [x] [decision] Source versioning, invalid version handling, installed version tracking #versioning
- [x] [decision] Platform instruction file paths for all 7 platforms + fallback convention #platforms
- [x] [fix] Deleted premature ADR-001 (created before user discussion) #process-correction

### ADR-001 Review and Updates

- [x] [adr] ADR-001 Plugin Format and Manifest created #architecture
- [x] [review] brain:adr-review skill run -- DEBATE-ADR-001 saved (2 Accept, 3 D&C, 1 Block) #review
- [x] [fix] P0 resolutions applied to ADR-001 #review
- [x] [fix] platforms field removed from ADR-001 optional fields (user decision) #manifest
- [x] [fix] formatVersion removed from ADR-001 required fields (user decision after ANALYSIS-007) #manifest
- [x] [fix] Schema evolution strategy section added to ADR-001 #manifest
- [x] [fact] ADR-001 status: ACCEPTED, COMPLETE #status

### ADR-002 Review and Updates

- [x] [adr] ADR-002 Target Platforms and Audiences created #architecture
- [x] [review] brain:adr-review skill run -- 6 individual review agents completed #review
- [x] [review] 6 individual REVIEW-*-ADR-002 notes deleted, consolidated into DEBATE-ADR-002 (0 Accept, 4 D&C, 1 Needs Revision) #review
- [x] [fix] P0-1 resolved: No graceful degradation -- Excluded Platforms section replaces degradation tier #platforms
- [x] [fix] P0-2 resolved: No publish command -- Author CLI description updated #distribution
- [x] [fix] P0-3 resolved: NEG-006 added (instruction file injection, deferred to ADR-004) #security
- [x] [fix] P0-4 resolved: NEG-007 added (no centralized vetting, deferred to ADR-004) #security
- [x] [fix] P1-1 resolved: Duplicate ADR-001-target-platforms-and-selection-criteria deleted #cleanup
- [x] [fix] P1-4 resolved: Platform table updated with actual instruction file paths for all 7 platforms #platforms
- [x] [fix] P1-5 resolved: Moot -- degradation tier removed #platforms
- [ ] P1-2 skipped: Audience priority ordering deferred to Phase 4 #deferred
- [ ] P1-3 skipped: Phased delivery deferred to Phase 4 #deferred
- [ ] P1-6 skipped: skills.sh differentiation already clear #deferred
- [x] [fix] P1-7 resolved: Self-bootstrapping acceptance criteria added (self-install + dogfood) #self-bootstrap
- [x] [fix] P1-8 resolved: Bloat not a real concern for CLI tools, gunshi lazy loading sufficient #accepted
- [x] [fix] P1-9 skipped: Content hashing covered by NEG-006/NEG-007 (ADR-004 scope) #security
- [x] [fix] P1-10 skipped: MCP access control covered by NEG-006/NEG-007 (ADR-004 scope) #security
- [x] [fix] P1-11: Instruction file update patterns -- ANALYSIS-009 COMPLETE #research
- [x] [fix] P1-12 resolved: Bidirectional ADR-002/ADR-003 link added #cleanup

### ADR-003 Review and P0/P1 Resolution

- [x] [adr] ADR-003 Conflict Resolution and Namespacing created #architecture
- [x] [review] brain:adr-review debate run -- UNANIMOUS NEEDS REVISION (0 Accept, 0 D&C, 6 Needs Revision) #review
- [x] [review] DEBATE-ADR-003 saved (7 P0, 14 P1, 8 P2) #review
- [x] [decision] P0 Core: Always-namespace adopted -- eliminates decisions 2+3 from original ADR #simplification
- [x] [decision] P0-1: Keep as one ADR, not split #structure
- [x] [decision] P0-2/3/4: MOOT with always-namespace #simplification
- [x] [decision] P0-5: Colon logical-only + kebab-case name validation #namespacing
- [x] [decision] P0-6: Strictest wins for blocking hooks, research spawned (ANALYSIS-010, 012) #hooks
- [x] [decision] P0-7: JSON lockfile at .agent-plugins/plugin-lock.json, re-derive on corruption #state
- [x] [decision] P1-2: Overlay/recompute pattern for hook merge ordering #hooks
- [x] [decision] P1-7/P1-9: Injection sanitization covered by ANALYSIS-009 + ANALYSIS-013 #security
- [x] [decision] P1-14: Forward reference to ADR-004 for hook security #deferred
- [x] [decision] P1-4/P1-11/P1-13: platformConfig resolved -- hybrid D+C with 4-level resolution (ANALYSIS-014) #platform-config
- [x] [decision] ADR-003 fully rewritten incorporating all P0/P1 resolutions #architecture

### Background Research Spawned

- [x] [research] ANALYSIS-009 Instruction File Update Patterns -- COMPLETE #instruction-files
- [x] [research] ANALYSIS-010 Hook Merge/Unmerge Patterns -- COMPLETE (overlay/recompute recommended) #hooks
- [x] [research] ANALYSIS-011 Lockfile Management Patterns -- COMPLETE (atomically + integer versioning + re-derive) #state-management
- [x] [research] ANALYSIS-012 JSON Config Merge Patterns -- COMPLETE (deepmerge + customMerge + Zod validation) #json-merge
- [x] [research] ANALYSIS-013 Input Sanitization Patterns -- COMPLETE (Zod v4 + shell-quote + validator) #security
- [x] [research] ANALYSIS-014 Platform Config Patterns -- COMPLETE (hybrid D+C, 4-level resolution, cross-platform concepts) #platform-config

### ADR-003 Round 2 Review and P1 Resolutions

- [x] [review] ADR-003 adr-review Round 2 (convergence check on rewritten ADR) -- 6 agents spawned in parallel #review
- [x] [review] Round 2 result: CONSENSUS REACHED (3 Accept, 3 D&C) -- Architect Accept, Critic Accept, Independent Thinker D&C, Security D&C, Analyst D&C, High-Level Advisor Accept #consensus
- [x] [review] DEBATE-ADR-003 updated with Round 2 verdicts, new issues, dissent record #review
- [x] [fix] P1-R2-1: ADR-001 Section 9 updated to reference ADR-003 as authoritative for conflict resolution #cross-adr
- [x] [decision] P1-R2-2: Lockfile at project root (plugin-lock.json) for project scope, ~/.config/agent-plugin/ for user scope. No .agent-plugins/ directory. #state-management
- [x] [decision] P1-R2-2: Hook overlays stored as sections within the lockfile, not separate files #simplification
- [x] [fix] P1-R2-3: customMerge dispatch noted for implementation (path-context-aware, not global name matching) #implementation-note
- [x] [fix] P1-R2-4: NEG-007 added to ADR-003 (strictest-wins composability constraint) #consequences
- [x] [fix] P1-R2-5: File permissions added to ADR-003 Decision 3 (lockfile 600, XDG dir 700) #security
- [x] [fix] P1-R2-6: Hook execution constraint added to ADR-003 Decision 2 (stored but NOT executable until ADR-004) #security
- [x] [fix] ADR-003 duplicate Decision 2/3 sections removed (Brain MCP replace_section artifact) #cleanup
- [x] [fix] ADR-003 inline Decision 2/3 text updated to match simplified lockfile model #cleanup
- [x] [update] ADR-001 POS-005 and observation updated to reference ADR-003 always-namespace #cross-adr

### Group 2: Core Technology Stack Research and Decisions

- [x] [research] Bun runtime assessment -- [[ANALYSIS-016-bun-runtime-assessment]] COMPLETE #runtime
- [x] [decision] Bun ADOPTED as runtime (user accepted) #runtime
- [x] [research] Interactive prompts and colors -- [[ANALYSIS-018-interactive-prompts-and-colors]] COMPLETE #prompts #colors
- [x] [decision] @clack/prompts ADOPTED for interactive CLI (user accepted) #prompts
- [x] [decision] picocolors initially adopted, later REMOVED in Group 3 (replaced by node:util styleText in @clack/prompts v1.1.0) #colors #superseded
- [x] [research] CLI framework comparison -- [[ANALYSIS-017-cli-framework-comparison]] COMPLETE #cli-framework
- [x] [research] MCP framework and file watching -- [[ANALYSIS-019-mcp-framework-and-file-watching]] COMPLETE #mcp #file-watching
- [x] [research] Shell completions, frontmatter, markdown, data storage, search -- COMPLETE (ANALYSIS-020, 021) #remaining-deps
- [x] [adr] ADR-005 Runtime and Distribution Strategy created #architecture
- [x] [fix] ADR-005 updated for Bun-only distribution: removed npx/Node.js consumer support, IMP-005, IMP-010, NEG-004 per user instruction #bun-only
- [x] [adr] ADR-006 Core Dependency Stack created #architecture
- [x] [review] ADR-006 adr-review debate: 6 agents completed (3 P0, 11 P1, 13 P2). DEBATE-ADR-006 created. #review
- [x] [fix] ADR-006 P0/P1 resolutions applied: validator CVE pin, supply chain controls, clack gate, version pinning, MADR frontmatter, cold start alignment, ADR-003 boundary, MCP SDK pin, Zod peer dep, governance policy, hook exec reference, gunshi estimate revised #adr-006-fixes
- [x] [test] @clack/prompts Bun 1.3.8 compatibility test: ALL PASS -- setRawMode works, all APIs available, EPERM regression from 1.3.2 fixed. Pin Bun >= 1.3.8. #clack #bun-compat #verified
- [x] [review] ADR-005 Round 2: UNANIMOUS ACCEPT (6/6). All P0/P1 resolved. #convergence #accepted
- [x] [review] ADR-006 Round 2: UNANIMOUS ACCEPT (6/6). All P0/P1 resolved. @gunshi/plugin-completion "latest" vs pinning policy (P2) noted for implementation. #convergence #accepted
- [x] [update] DEBATE-ADR-005 and DEBATE-ADR-006 updated with Round 2 verdicts and consensus status #debate-logs

### Group 3: CLI Architecture Research and ADR-007

- [x] [research] @clack/prompts API surface -- [[ANALYSIS-022-clack-prompts-api-surface-and-gaps]] COMPLETE #clack #api
- [x] [research] gunshi command patterns -- [[ANALYSIS-023-gunshi-command-patterns-and-capabilities]] COMPLETE #gunshi #cli
- [x] [research] CLI CI mode patterns -- [[ANALYSIS-024-cli-ci-mode-and-non-interactive-patterns]] COMPLETE #ci #patterns
- [x] [test] @clack/prompts v1.0.0+ APIs: 12 tested on Bun 1.3.8 (ALL PASS), 6 v1.0.0 additions source-confirmed #clack #bun-compat
- [x] [decision] picocolors REMOVED: @clack/prompts v1.1.0 uses node:util styleText instead #dependency-change
- [x] [decision] ci-info adopted for CI detection (zero deps, 50+ vendors) #ci
- [x] [decision] update alias dropped -- upgrade only #command-tree
- [x] [adr] ADR-007 CLI Architecture and Interaction Model created (10 decisions) #architecture
- [x] [adr] ADR-006 amended: picocolors removed, ci-info added, total deps 13 (9 new + 4 from ADR-003) #amendment
- [x] [review] ADR-007 adr-review Round 1: 6/6 ACCEPT_WITH_CONDITIONS (4 P0, 19 P1, 19 P2) #review
- [x] [fix] P0-A: MCP error response Zod schema added (Decision 9) #p0
- [x] [fix] P0-B: Common JSON response envelope added (Decision 10) #p0
- [x] [fix] P0-C: Component count corrected (12 tested + 6 pending) #p0
- [x] [fix] P0-D: Source format taxonomy added (4 types, rules 13-16) #p0
- [x] [fix] P1: Flag interaction matrix, glob hook matchers, MCP sanitization, exit codes, picocolors note, update alias, testing matrix, p.password() removed #p1
- [x] [review] ADR-007 Round 2: UNANIMOUS ACCEPT (6/6), 3 D&C from independent thinker #convergence
- [x] [fix] 6x individual REVIEW-ADR-007-* notes deleted, all content in DEBATE-ADR-007 #cleanup
- [x] [fix] DEBATE-ADR-007 renamed from space-separated to kebab-case #naming
- [x] [decision] User prefers over-specification in ADRs to prevent implementation assumptions #user-preference #adr-style
- [x] [constraint] ADR review agents must NOT create individual REVIEW notes. All agent responses go into the single DEBATE-ADR-NNN note only #adr-review #convention
- [x] [fact] ADR-007 now has 10 decisions: original 8 + Decision 9 (MCP error schema) + Decision 10 (JSON envelope) #adr-007

### File Name Fixes

- [x] [fix] Renamed ADR-001, ADR-002, ADR-003 from space-separated to kebab-case file names #naming
- [x] [fix] Renamed ANALYSIS-008 from space-separated to kebab-case file name #naming
- [x] [fix] Deleted premature ADR-001-target-platforms-and-selection-criteria #cleanup
- [x] [fix] ADR-005 file renamed to kebab-case (ADR-005-runtime-and-distribution-strategy) #naming
- [x] [fix] ADR-006 file renamed to kebab-case (ADR-006-core-dependency-stack) #naming
- [x] [fix] ANALYSIS-020 file renamed to kebab-case (ANALYSIS-020-frontmatter-and-markdown-processing) #naming

### Group 4: Source and Platform Research and Decisions

- [x] [research] Source resolution patterns -- [[ANALYSIS-025-source-resolution-patterns]] COMPLETE #source-resolution
- [x] [research] Platform detection and mapping -- [[ANALYSIS-026-platform-detection-and-mapping]] COMPLETE #platform-detection
- [x] [research] Installation mechanics -- [[ANALYSIS-027-installation-mechanics]] COMPLETE #installation
- [x] [decision] plugin.json only for manifest discovery (no package.json field fallback) #manifest-discovery
- [x] [fix] ANALYSIS-026 renamed from space-separated to kebab-case #naming
- [x] [decision] Bare owner/repo: smart detection (filesystem first, then GitHub) #source-resolution
- [x] [decision] Bun.semver adopted, tar npm package for extraction, Bun.write arrayBuffer workaround for downloads #bun-apis
- [x] [decision] Platform detection: dual (binary + config dir) via Promise.allSettled with 3s timeout #platform-detection
- [x] [decision] MCP key namespacing: colon separator (`plugin-name:server-name`), matching Claude Code internal convention and ADR-003 component identifier pattern #mcp-namespacing
- [x] [decision] Install scope: project default, prompt when no project detected, error in CI #install-scope
- [x] [decision] System deps: agent-plugin CAN install with user confirmation, tracked in plugin-lock.json #system-deps
- [x] [decision] Installation flow: 6-phase (Detect, Select, Resolve, Confirm, Apply, Record) with platform multiselect #installation
- [x] [decision] Upgrade: interactive multiselect `plugin current -> latest`, atomic replace with rollback #upgrade
- [x] [decision] Platform config mapping: platforms.config.json at project root, pure data file #platform-config
- [x] [decision] Optional CLI generation from MCP tools: `cli` field in plugin.json ("auto" | path | omitted). v1: flat subcommands, author `cli.groups` for grouping. Built-in `mcp` start/stop/restart/status commands. Daemon/stdio transport. #cli #mcp
- [x] [decision] All 11 Group 4 discussion topics decided #group-4-complete
- [x] [decision] MCP key separator changed from slash to colon based on debate P0: JSON Pointer RFC 6901 conflicts, Claude Code internal convention, ADR-003 precedent #mcp-namespacing #p0-resolution
- [x] [decision] platforms.config.json maintenance: bundled in npm package, updated via normal version bumps. No remote fetching. #platform-config #p0-resolution
- [x] [decision] Dual detection kept for v1. Added --platform flag as override for CI and edge cases. #platform-detection #p0-resolution
- [x] [decision] CLI generation split from ADR-010 into ADR-011 per debate P0-1 consensus (5/6 reviewers) #cli #architecture #p0-resolution
- [x] [decision] Dependency auto-install kept. Deps defined by agent-plugin package (platforms.config.json), not plugin authors. Tracked for uninstall. #dependencies #p0-resolution

### Group 4: ADR Creation, Debate, and Acceptance

- [x] [adr] ADR-008 Source Resolution and Package Validation created #architecture
- [x] [review] ADR-008 adr-review Round 1: ACCEPTED (3 Accept, 3 D&C). 3 P0 issues. #review
- [x] [fix] ADR-008 P0/P1 resolutions: memory argument contradiction dropped, zip slip IMP-007 added, integrity verification IMP-008 added, go-getter citation corrected, observation categories fixed #p0-resolution
- [x] [adr] ADR-009 Platform Detection and Config Registry created #architecture
- [x] [review] ADR-009 adr-review Round 1: NEEDS REVISION (0 Accept, 1 Needs Rev, 4 D&C). 7 P0 issues. #review
- [x] [fix] ADR-009 P0 resolutions: colon separator (P0-1), qualified "zero code changes" (P0-2), envOverrides added (P0-3), maintenance strategy added (P0-4), Claude Code user-scoped MCP (P0-5), phantom "servers" key removed (P0-6), --platform flag added (P0-7) #p0-resolution
- [x] [review] ADR-009 Round 2: ACCEPTED (5 Accept, 1 D&C). No remaining P0s. #convergence
- [x] [adr] ADR-010 Installation Lifecycle created (originally included CLI generation) #architecture
- [x] [review] ADR-010 adr-review Round 1: NEEDS REVISION with 1 BLOCK (0 Accept, 1 Block, 1 Needs Rev, 3 D&C). 7 P0 issues. #review
- [x] [fix] ADR-010 P0 resolutions: CLI generation extracted to ADR-011 (P0-1), dependency policy revised to trusted source (P0-2), rollback documented as best-effort (P0-5), integrity verification added (P0-6), MCP namespace codified as colon (P0-7) #p0-resolution
- [x] [review] ADR-010 Round 2: ACCEPTED (5 Accept, 1 D&C). Security D&C: 4 P1 reservations. #convergence
- [x] [adr] ADR-011 Auto-Generated CLI from MCP Tools created (extracted from ADR-010 Decision 5) #architecture
- [x] [review] ADR-011 adr-review Round 1: unanimous NEEDS REVISION (0 Accept, 6 Needs Rev). 7 P0 issues. #review
- [x] [fix] ADR-011 P0 resolutions: flat subcommands for v1 (P0-1), MCP daemon lifecycle commands added (P0-2), binary name denylist (P0-3), trust model section (P0-4), path containment validation (P0-5), prior art section (P0-6), parameter type mapping (P0-7) #p0-resolution
- [x] [decision] MCP daemon lifecycle: built-in start/stop/restart/status commands for every plugin with MCP server. Daemon/stdio transport model. Restart safety: never touches Claude Code stdio instances. #mcp #daemon-lifecycle
- [x] [review] ADR-011 Round 2: ACCEPTED (5 Accept, 1 D&C). Independent Thinker D&C: daemon transport protocol must be specified before daemon ships. #convergence
- [x] [fix] All debate log filenames aligned: CRIT-NNN renamed to DEBATE-ADR-NNN convention for ADR-008, 009, 010 #naming
- [x] [fix] Debate log frontmatter fixed for ADR-001 through 007: titles using spaces, type changed to critique, permalinks corrected #naming
- [x] [fact] All 4 Group 4 ADRs accepted: ADR-008, ADR-009, ADR-010, ADR-011 #complete

### Group 5: Data and Storage Quick-Pass

- [x] [analysis] Group 5 quick-pass: 9/11 spec topics already decided by ADR-003, ADR-010, ADR-011, ANALYSIS-021 #data-storage
- [x] [decision] MCP tool registry: on-demand for v1, not persisted in lockfile. Avoids schema staleness. #mcp #data-storage
- [x] [decision] Dual-location manifest rejected: plugin.json only, no package.json embedded field #manifest #simplification
- [x] [fact] Group 5 COMPLETE: all data storage decisions covered by existing ADRs #complete

### Group 7: Commands Research and Decisions

- [x] [research] Consumer and author commands -- [[ANALYSIS-033-consumer-and-author-commands]] COMPLETE #commands #distribution
- [x] [research] Skill versioning models comparison -- [[ANALYSIS-034-skill-versioning-models-comparison]] COMPLETE #versioning #npm
- [x] [decision] MAJOR ARCHITECTURAL PIVOT: npm-package distribution model adopted (TanStack Intent style). Plugins are npm packages installed via `bun add @scope/plugin`. Bun handles ALL package management. agent-plugin becomes purely a "wiring tool" bridging npm packages to AI platform configs. #distribution #architecture
- [x] [decision] Consumer commands (add, remove, upgrade) ELIMINATED -- bun handles package management natively #commands #simplification
- [x] [decision] `init` redesigned to Husky model: `agent-plugin init` wires lifecycle hooks into package.json (`"postinstall": "agent-plugin install"`). Smart merge with existing scripts. postinstall covers add/remove/update since bun triggers it for all dependency changes. #init #lifecycle-hooks
- [x] [decision] `install` is core wiring command: scan node_modules for packages with plugin.json, diff vs currently wired platforms, reconcile (add new, remove deleted, update changed) #install #core-command
- [x] [decision] `create` command ADDED: scaffolds new plugin project (like `bun create`) #create #scaffolding
- [x] [decision] `deinit` command ADDED: reverse of init, removes lifecycle hooks from package.json #deinit #lifecycle-hooks
- [x] [decision] `dev` command REMOVED: not needed with lifecycle hook model #dev #removed
- [x] [decision] `complete` command REMOVED: shell completions not needed #completions #removed
- [x] [decision] `publish` command CONFIRMED REMOVED: no registry. No version command either. Version management is author's responsibility using standard tools (bun, npm). validate can warn about inconsistencies. #publish #removed
- [x] [decision] version field removed from plugin.json required fields: package.json version is authoritative. plugin.json minimum required fields reduced to `name` and `description`. plugin.json becomes content declaration manifest (skills[], agents[], hooks[], instructions[], commands[], mcp[]). #manifest #simplification
- [x] [decision] ADR-008 (Source Resolution) to be FULLY SUPERSEDED: bun handles all source resolution #adr-impact
- [x] [decision] ADR-010 (Installation Lifecycle) to be MOSTLY SUPERSEDED: 6-phase model replaced by bun add + agent-plugin install wiring #adr-impact
- [x] [decision] ADR-003 Decision 3 (lockfile) to be SUPERSEDED: bun.lockb replaces plugin-lock.json #adr-impact
- [x] [decision] ADR-001 to be amended: version removed from required fields #adr-impact
- [x] [fix] ANALYSIS-033, ANALYSIS-034 renamed from space-separated to kebab-case file names #naming

### Group 7 (cont'd): ADR-013 Debate and Cross-Platform Research

- [x] [adr] ADR-013 npm-Package Distribution and Revised Command Tree created (6 decisions) #architecture
- [x] [review] ADR-013 adr-review Round 1: unanimous NEEDS REVISION (0 Accept, 6 Needs Rev). 3 P0, 11 P1, 7 P2. #review
- [x] [review] DEBATE-ADR-013 saved with Round 1 results, P0 proposed resolutions, agent conflict resolution #debate-log
- [x] [test] Bun v1.3.8 postinstall empirical test: postinstall fires on ALL operations (add, remove, update, install). Disproves P0-1 concern. #bun #verified
- [x] [research] Hook merging without lockfile -- [[ANALYSIS-035-hook-merging-strategies-without-custom-lockfile]] COMPLETE. Full recompute recommended (<20ms). #hooks
- [x] [decision] P0-1 RESOLVED: ADR-013 Decision 3 claim is correct for Bun (unlike npm/yarn). No ADR change needed. #p0-resolution
- [x] [decision] P0-2 RESOLVED: Full recompute from plugin.json files in node_modules. node_modules IS the state. No separate hook tracking needed. #p0-resolution
- [x] [decision] P0-3 RESOLVED: Restrict scanning to direct dependencies in package.json only (not transitive). Supply chain risk mitigated. #p0-resolution
- [x] [decision] P1-1 agreed: Add acknowledgment that ANALYSIS-034 recommended against TanStack model, document why overridden after ANALYSIS-033 revealed 12 conflicts #p1-resolution
- [x] [done] P1-2 through P1-11: ALL resolved in ADR-013 Round 2 revision #adr-013
- [x] [done] P0 resolutions applied to ADR-013 file #adr-013
- [x] [done] ADR-013 Round 2 convergence vote: 5 Accept + 1 D&C = CONSENSUS #adr-013
- [x] [research] AI platform plugin standards survey -- [[ANALYSIS-036-ai-platform-plugin-standards-survey]] COMPLETE #cross-platform
- [x] [research] Cross-platform hook configuration -- [[ANALYSIS-037-cross-platform-hook-configuration-analysis]] COMPLETE #hooks #cross-platform
- [x] [research] Cross-platform agent and skill definitions -- [[ANALYSIS-038-cross-platform-agent-and-skill-definitions]] COMPLETE #agents #skills #cross-platform
- [x] [research] Cross-platform MCP and instruction formats -- [[ANALYSIS-039-cross-platform-mcp-and-instruction-formats]] COMPLETE #mcp #instructions #cross-platform
- [x] [research] Prior art for cross-platform adapter patterns -- [[ANALYSIS-040-prior-art-for-cross-platform-adapter-patterns]] COMPLETE #adapters #cross-platform
- [x] [research] Plugin manifest cross-platform comparison -- [[ANALYSIS-041-plugin-manifest-cross-platform-comparison]] COMPLETE #manifest #cross-platform
- [x] [fix] ANALYSIS-035 through ANALYSIS-041 renamed from space-separated to kebab-case file names #naming
- [x] [decision] Rename `mcp` to `mcpServers` in plugin.json (de facto standard across 6/8 platforms) #manifest #alignment
- [x] [decision] Support `string | string[]` for content paths in manifest (matching Claude Code/Cursor/Copilot patterns) #manifest #alignment
- [x] [decision] Support inline objects for hooks/mcpServers (not just file paths) #manifest #alignment
- [x] [decision] Commands confirmed as first-class content type (user-invoked, distinct from agent-invoked skills) #commands #content-types
- [x] [fact] Bun Plugin API (`Bun.plugin()`) is bundler/runtime only, cannot hook into package manager operations. No PM hook surface exists. Feature request #8062 open. #bun #limitation
- [x] [fact] Style Dictionary is closest prior art for cross-platform adapter (9/10 applicability). Transform pipeline: Resolve, Map concepts, Translate fields, Format output, Write files. #prior-art
- [x] [fact] 6 universal hook events across platforms: Pre Tool Use, Post Tool Use, User Prompt Submit, Session Start, Stop/Complete, Session End #hooks #universal
- [x] [fact] 6 universal agent fields: name, description, prompt, tools, model, mcpServers #agents #universal
- [x] [resolved] Native platform plugin installation: deferred to v2 (explicit add/remove/update model first) #architecture
- [x] [resolved] installMode replaced by features model: plugin-level and component-level features with section-based cherry-picking #architecture
- [x] [resolved] Rules vs AGENTS.md: rules are file-based (per feature), AGENTS.md is section-based. instructions/ renamed to AGENTS.md content type #content-types
- [x] [done] ADR amendments: ADR-001 updated with amendments #6 (manifest location), #7 (content types 6→8), #8 (installMode→features). ADR-003 D2/D3 supersession noted. #amendments
- [x] [done] ADR supersessions: ADR-008 SUPERSEDED → ADR-013 → ADR-014. ADR-010 SUPERSEDED → ADR-013 → ADR-014. ADR-013 SUPERSEDED by ADR-014. #supersessions
- [x] [done] ADR-014 created: explicit add/remove/update model, 8 decisions, accepted Round 2 (5 Accept + 1 D&C) #adr-014

### Group 7 (cont'd): ADR-013 Round 2 and Design Pivot

- [x] [review] ADR-013 Round 2 convergence vote: 6 agents spawned in parallel (architect, critic, independent-thinker, security, analyst, high-level-advisor) #review
- [x] [review] Round 2 result: CONSENSUS REACHED (5 Accept, 1 D&C) -- Architect Accept, Security Accept, Advisor Accept, Analyst Accept, Critic Accept, Independent Thinker D&C #consensus
- [x] [review] DEBATE-ADR-013 updated with Round 2 verdicts, 7 P2 issues (non-blocking), D&C reservations #review
- [x] [fix] ADR-013 status changed from "proposed" to "accepted" (2026-03-09, Round 2: 5 Accept + 1 D&C) #status
- [x] [decision] MAJOR DESIGN PIVOT: user rethought installation model, moved AWAY from ADR-013's npm-dependency + postinstall toward explicit `agent-plugin add/remove/update` commands (Vercel Skills style) #pivot
- [x] [decision] ADR impact approach: Option C -- keep ADR-013 accepted, create ADR-014 to supersede it #adr-impact
- [x] [research] Vercel Skills CLI deeply analyzed via GitHub API: lock files (global + project), installer, remove, list commands. Two repos: vercel-labs/skills (CLI) and vercel-labs/agent-skills (content). #vercel-skills
- [x] [decision] `.agent-lock.json` lockfile in consuming project: source, sourceType, hash, features, timestamps per plugin. Team sync via `agent-plugin install`. #lockfile
- [x] [decision] No consumer-side config file: `.agent-lock.json` + platform configs are the only state #simplification
- [x] [decision] Features model designed: plugin-level (span multiple components) + component-level (single component). Top-level `features` declaration with description, default, requires. Per-component features mapping to sections. #features
- [x] [decision] Section-based feature mechanism for markdown (skills, agents, commands, AGENTS.md): standard markdown headers parsed by remark/unified/mdast-util-heading-range #sections #markdown
- [x] [decision] Section-based feature mechanism for code (hooks): `// #region feature:NAME` / `// #endregion feature:NAME`. Custom parser. #sections #code
- [x] [decision] File-based feature mechanism for rules only: individual files included/excluded per feature #rules
- [x] [decision] MCP has NO features -- always installed as-is #mcp
- [x] [decision] Plugin manifest naming: `.agent-plugin/plugin.json` (follows Claude Code `.claude-plugin/plugin.json` convention). 3/3 AI platforms converged on plugin.json. Directory provides disambiguation. #manifest
- [x] [decision] Content types: Skills, Agents, Hooks, Commands, Rules, MCP, AGENTS.md (replaces instructions/), CLI (optional) #content-types
- [x] [decision] Platform-specific metadata in plugin.json per component `platforms` field, NOT in content files #platform-config
- [x] [decision] Source types: npm packages, git repos (owner/repo, full URL), local paths #sources
- [x] [decision] Revised command tree: add/remove/update/install/list (consumer) + create/validate/build (author) + mcp serve + content-type CRUD groups #command-tree
- [x] [decision] Add end-to-end flow: resolve source → read plugin.json → validate Zod → detect platforms → feature wizard → parse/compile → write platforms → update lockfile → summary #add-flow
- [x] [decision] plugin.json schema drafted with features, skills, hooks, rules, mcpServers, agentsMd sections #schema
- [x] [decision] Plugin package structure settled: .agent-plugin/plugin.json, skills/, agents/, hooks/, commands/, cli/, mcp/, AGENTS.md, rules/, package.json #structure
- [x] [workstream] ANALYSIS-047 created: project bootstrapping and DX infrastructure (Biome, Bun, Turbo, releases, CI/CD, GitHub config, testing) #pending
- [ ] [pending] IMP-007 (from ADR-012): Update ANALYSIS-031 to remove stale gray-matter references #cleanup

### Decision Audit Reconciliation

- [x] [analysis] 6 parallel extraction agents read 22,969 lines of conversation log #reconciliation
- [x] [adr] ADR-014 Explicit Installation Model and Content Features created (8 decisions, 785 lines) #architecture
- [x] [review] ADR-014 adr-review Round 1: NEEDS REVISION. Round 2: ACCEPTED (5 Accept + 1 D&C). #review
- [x] [review] DEBATE-ADR-014 saved with both rounds #debate-log
- [x] [fix] ADR-001 updated with amendments #6 (manifest location), #7 (content types), #8 (installMode→features) #amendments
- [x] [fix] ADR-013 status changed to "superseded" with ADR-014 explanation #supersession
- [x] [fix] ADR-008, ADR-010 supersession chain: → ADR-013 → ADR-014 #supersession
- [x] [fix] ANALYSIS-043 updated with ADR-014 adoption status #analysis-update
- [x] [fix] ANALYSIS-044 updated with ADR-014 adoption status #analysis-update
- [x] [fix] ADR-003 Decision 3 supersession note updated for ADR-014 #cross-adr
- [x] [fix] ANALYSIS-042 updated with deferral adoption status #analysis-update

### Comprehensive Gap Analysis (2026-03-09)

- [x] [analysis] 3 parallel 🧠:analyst agents: spec inventory (594 decision points), ADR audit (all 13 ADRs), cross-reference (9 gaps) #gap-analysis
- [x] [fact] Phase 1 assessed at ~85% complete: Groups 1-7 done, Group 8 ~60%, Group 9 ~75% #status
- [x] [fact] 9 gaps identified: 4 pre-resolved by existing decisions, 5 require action #gaps
- [x] [fact] GAP-4 is CRITICAL: ADR-004 Plugin Security Model forward-referenced by 4 active ADRs, 11 items blocked #security #critical
- [x] [fact] 9 stale ADR body text instances identified (amendments correct, body text not updated) #staleness
- [x] [fact] ADR-006 needs amendment for remark/unified/mdast-util-heading-range deps from ADR-014 D5 #amendment-needed
- [x] [fact] ~60 open IMP-NNN implementation notes across active ADRs, not blocking Phase 1 #implementation-notes

### GAP-2 Resolution: MCP Tool Catalog and Command Tree Updates (2026-03-09)

- [x] [analysis] Read spec Section 15 (14 MCP tools in 3 categories) and ADR-014 Decision 6 (35 CLI commands) #gap-2
- [x] [decision] `eval` subcommand renamed to `analyze` (clearer, avoids JS eval() ambiguity) #commands #rename
- [x] [decision] `improve` subcommand replaced by `analyze --fix` flag (ESLint/Prettier/Biome convention) #commands #simplification
- [x] [decision] Content-type groups use plural names: skills, agents, commands, hooks, rules (mcp stays singular) #commands #naming
- [x] [decision] MCP tool catalog deferred to creator skill evaluation. Derivation order: creator skills → CLI wizards → MCP tools #mcp #sequencing
- [x] [decision] Not all CLI commands become MCP tools. Excluded: mcp serve, build, analyze, create project scaffold (~20-25 tools from 32 commands) #mcp #curation
- [x] [fix] ADR-014 Decision 6 command tree updated with plural names, analyze/analyze --fix #adr-014 #amendment
- [x] [fix] ADR-014 Amendment #1 (command naming) and Amendment #2 (MCP catalog deferral) added #adr-014
- [x] [fix] ADR-012 Amendment #1 added (eval→analyze, improve→analyze --fix cross-reference) #adr-012

### Organization Rename and Path Migration

- [x] [fact] Organization renamed from acmelabz to acmelabs-15, npm scope @acmelabs-15
- [x] [fact] Git remote updated to <https://github.com/acmelabs-15/agent-plugin>
- [x] [fact] 26 docs files updated (66 replacements), committed as 7ccbbb1
- [x] [fact] Local directory moved from /Users/peter.kloss/Dev/acmelabz/agent-plugin to /Users/peter.kloss/Dev/acmelabs-15/agent-plugin
- [x] [fact] Brain MCP project config recreated: code_path and memories_path updated to new location (CODE mode, docs/)
- [ ] [pending] Clean up old /Users/peter.kloss/Dev/acmelabz/ directory (may have hidden files)
- [x] [done] Restart Claude Code from new working directory #resolved

### Group 6: Scaffolding Wizards Research and ADR-012

- [x] [research] Scaffolding wizard patterns -- [[ANALYSIS-031-scaffolding-wizard-patterns]] COMPLETE #scaffolding #wizards
- [x] [research] MCP SDK Bun runtime compatibility -- [[ANALYSIS-032-mcp-sdk-bun-runtime-compatibility]] COMPLETE #mcp-sdk #bun
- [x] [research] Anthropic creator skills found: skill-creator (anthropics/skills), mcp-builder (anthropics/skills), agent-creator (anthropics/claude-code/plugins/plugin-dev) #creator-skills
- [x] [fact] No Anthropic tool exists for CLAUDE.md/AGENTS.md evaluation -- opportunity for original instruction-evaluator #differentiator
- [x] [decision] 16 decisions discussed one at a time with user across template rendering, schema-first, wizards, creator skills, eval/improve, instructions, trust model #group-6
- [x] [adr] ADR-012 Scaffolding and Content Management created (11 decisions) #architecture
- [x] [review] ADR-012 adr-review Round 1: unanimous NEEDS REVISION (0 Accept, 6 Needs Rev). 7 P0 issues consolidated. #review
- [x] [fix] P0-1: MCP lifecycle commands removed from mcp group (consumer-only per ADR-011) #p0-resolution
- [x] [fix] P0-2: content.* nesting replaced with flat array names matching ADR-001 #p0-resolution
- [x] [fix] P0-3: Commands kept, IMP-005 tracks ADR-001 amendment (6th component type) #p0-resolution
- [x] [fix] P0-4: IMP-006 added (creator skills independently shippable, phasing deferred to project plan) #p0-resolution
- [x] [fix] P0-5: IMP-007 added (ANALYSIS-031 gray-matter cleanup) #p0-resolution
- [x] [fix] P0-6: Decision 11 added (trust model, HTML viewer, faithfulness principle) #p0-resolution
- [x] [fix] P0-7: Verified Zod v4 + MCP SDK compatible. Min versions added. ANALYSIS-032 created. #p0-resolution
- [x] [review] ADR-012 adr-review Round 2: ACCEPTED (5 Accept, 1 D&C). All P0s resolved. #convergence
- [x] [review] DEBATE-ADR-012 saved with Round 1 + Round 2 results #debate-log
- [x] [fact] Independent Thinker D&C reservations: faithfulness subordination, IMP-007 scope, marker comment error handling #d&c
- [x] [fix] 23 analysis note titles fixed from kebab-case to Title Case in frontmatter #naming
- [x] [fix] Duplicate ANALYSIS-031 (space-separated filename) deleted #cleanup
- [x] [fix] DEBATE-ADR-012 renamed from CRIT-012 to DEBATE-ADR-NNN convention, then from space-separated to kebab-case #naming
- [x] [fact] ADR-012 status: ACCEPTED, COMPLETE #status

---

## Requirements Context

From /Users/peter.kloss/Downloads/agent-plugin-design-spec.md:

**Package:** @acmelabs-15/agent-plugin
**Runtime:** Bun | **Language:** TypeScript (strict mode) | **License:** MIT

**Three Audiences:**

1. Consumers -- install AI agent plugins to coding platforms
2. Library Authors -- scaffold, validate, build plugin packages (no publish)
3. AI Assistants -- operate same workflows via embedded MCP server

**Major Areas (22 sections):**

1. Identity and naming
2. Vision (CLI + MCP server + skill system)
3. Audiences and workflows
4. Dependency stack (gunshi, @clack/prompts, gray-matter, drizzle-orm, orama, fastmcp, zod, etc.)
5. CLI architecture (gunshi-based command tree)
6. @clack/prompts usage (full v1.1.0 API)
7. Source resolution (npm, GitHub, local)
8. Platform support (7 platforms with 6/6 capabilities)
9. Installation mechanics (scope, conflicts, hook merging, MCP merging, dependencies)
10. Data storage (SQLite via drizzle-orm replacing JSON lockfile)
11. Manifest format (plugin.json at root)
12. Scaffolding wizard specs (agent, skill, command, hook, rule, MCP)
13. Consumer commands (add, remove, upgrade, list)
14. Author commands (init, validate, build, dev)
15. Embedded MCP server (fastmcp, 14 tools)
16. Self-bootstrapping architecture
17. Author project layout
18. Project context system
19. Shell completions (@bomb.sh/tab)
20. Bun-native APIs
21. Markdown processing pipeline (gray-matter + micromark)
22. Implementation phases (5 phases)

---

## Files Touched

### Brain Memory Notes

| Action | Note | Status |
|--------|------|--------|
| created | [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]] | IN_PROGRESS |
| created | [[ANALYSIS-001-agent-plugin-foundation-and-vision]] | COMPLETE |
| created | [[ANALYSIS-002-platform-capability-matrix]] | COMPLETE |
| created | [[ANALYSIS-003-vercel-skills-format-deep-dive]] | COMPLETE |
| created | [[ANALYSIS-004-tanstack-intent-deep-dive]] | COMPLETE |
| created | [[ANALYSIS-005-claude-code-plugin-format]] | COMPLETE |
| created | [[ANALYSIS-006-runtime-cli-framework-validation-stack]] | STALE (superseded by ANALYSIS-016 through 021) |
| created | [[ANALYSIS-007-config-schema-versioning-patterns]] | COMPLETE |
| created | [[ANALYSIS-008-platform-instruction-file-paths]] | COMPLETE |
| created | [[ADR-001-plugin-format-and-manifest]] | ACCEPTED |
| created | [[ADR-002-target-platforms-and-audiences]] | ACCEPTED |
| created | [[ADR-003-conflict-resolution-and-namespacing]] | ACCEPTED |
| created | [[DEBATE-ADR-001-plugin-format-and-manifest]] | COMPLETE (2 Accept, 3 D&C, 1 Block) |
| created | [[DEBATE-ADR-002-target-platforms-and-audiences]] | COMPLETE (0 Accept, 4 D&C, 1 Needs Revision) |
| deleted | ADR-001-target-platforms-and-selection-criteria | Premature duplicate |
| deleted | CRIT-001 ADR-001 Plugin Format and Manifest Review | Ad-hoc, replaced by adr-review skill |
| deleted | REVIEW-ADR-002-target-platforms-and-audiences | Ad-hoc, replaced by adr-review skill |
| deleted | 6x REVIEW-*-ADR-002 individual notes | Consolidated into DEBATE-ADR-002 |
| renamed | ADR-001, ADR-002, ADR-003, ANALYSIS-008 | From space-separated to kebab-case file names |
| created | [[ANALYSIS-009-instruction-file-update-patterns]] | COMPLETE |
| created | [[ANALYSIS-010-hook-merge-unmerge-patterns]] | COMPLETE |
| created | [[DEBATE-ADR-003-conflict-resolution-and-namespacing]] | COMPLETE (Round 1: unanimous Needs Revision; Round 2: CONSENSUS 3 Accept + 3 D&C) |
| created | [[ANALYSIS-011-lockfile-management-patterns]] | COMPLETE |
| created | [[ANALYSIS-012-json-config-merge-patterns]] | COMPLETE |
| created | [[ANALYSIS-013-input-sanitization-patterns]] | COMPLETE |
| created | [[ANALYSIS-014-platform-config-patterns]] | COMPLETE |
| created | [[CRIT-003 ADR-003 Round 2 Convergence Review]] | COMPLETE (critic agent saved) |
| created | [[ANALYSIS-015-ADR-003-convergence-round2-validation]] | COMPLETE (analyst agent saved) |
| updated | [[ADR-001-plugin-format-and-manifest]] | Section 9 superseded by ADR-003, POS-005 updated, observation updated |
| updated | [[ADR-003-conflict-resolution-and-namespacing]] | Decisions 2+3 rewritten (lockfile model, composability, permissions, hook execution gate) |
| updated | [[DEBATE-ADR-003-conflict-resolution-and-namespacing]] | Round 2 verdicts, new P1/P2 issues, dissent record, consensus status |
| created | [[ADR-005-runtime-and-distribution-strategy]] | ACCEPTED (Round 2: Unanimous Accept 6/6) |
| created | [[ADR-006-core-dependency-stack]] | ACCEPTED (Round 2: Unanimous Accept 6/6) |
| created | [[DEBATE-ADR-005-runtime-and-distribution-strategy]] | Round 1 + Round 2 verdicts |
| created | [[DEBATE-ADR-006-core-dependency-stack]] | Round 1 + Round 2 verdicts |
| created | [[ANALYSIS-016-bun-runtime-assessment]] | COMPLETE |
| created | [[ANALYSIS-017-cli-framework-comparison]] | COMPLETE |
| created | [[ANALYSIS-018-interactive-prompts-and-colors]] | COMPLETE |
| created | [[ANALYSIS-019-mcp-framework-and-file-watching]] | COMPLETE |
| created | [[ANALYSIS-020-frontmatter-and-markdown-processing]] | COMPLETE |
| created | [[ANALYSIS-021-data-storage-and-search]] | COMPLETE |
| created | [[ANALYSIS-022-clack-prompts-api-surface-and-gaps]] | COMPLETE |
| created | [[ANALYSIS-023-gunshi-command-patterns-and-capabilities]] | COMPLETE |
| created | [[ANALYSIS-024-cli-ci-mode-and-non-interactive-patterns]] | COMPLETE |
| created | [[ADR-007-cli-architecture-and-interaction-model]] | ACCEPTED (Round 2: Unanimous Accept 6/6) |
| created | [[DEBATE-ADR-007-cli-architecture-and-interaction-model]] | COMPLETE |
| updated | [[ADR-006-core-dependency-stack]] | Amended: picocolors removed, ci-info added, total deps updated to 13 |
| deleted | 6x REVIEW-ADR-007-* individual notes | Consolidated into DEBATE-ADR-007 |
| created | [[ANALYSIS-025-source-resolution-patterns]] | COMPLETE |
| created | [[ANALYSIS-026-platform-detection-and-mapping]] | COMPLETE |
| created | [[ANALYSIS-027-installation-mechanics]] | COMPLETE |
| renamed | ANALYSIS-026 | From space-separated to kebab-case file name |
| updated | [[ANALYSIS-027-installation-mechanics]] | Decisions revised: 6-phase install, slash MCP namespacing, interactive upgrade, system deps with confirmation |
| created | [[ANALYSIS-029-platform-config-registry]] | COMPLETE |
| created | [[ANALYSIS-030-auto-generated-cli-from-mcp-tools]] | COMPLETE |
| created | [[ANALYSIS-028-bun-builtin-api-reliability]] | COMPLETE |
| created | [[ADR-008-source-resolution-and-package-validation]] | ACCEPTED (Round 1: 3 Accept, 3 D&C) |
| created | [[ADR-009-platform-detection-and-config-registry]] | ACCEPTED (Round 2: 5 Accept, 1 D&C) |
| created | [[ADR-010-installation-lifecycle]] | ACCEPTED (Round 2: 5 Accept, 1 D&C) |
| created | [[ADR-011-auto-generated-cli-from-mcp-tools]] | ACCEPTED (Round 2: 5 Accept, 1 D&C) |
| created | [[DEBATE-ADR-008-source-resolution-and-package-validation]] | COMPLETE (Round 1: 3 Accept, 3 D&C) |
| created | [[DEBATE-ADR-009-platform-detection-and-config-registry]] | COMPLETE (Round 1: Needs Revision; Round 2: 5 Accept, 1 D&C) |
| created | [[DEBATE-ADR-010-installation-lifecycle-and-cli-generation]] | COMPLETE (Round 1: Needs Revision with 1 Block; Round 2: 5 Accept, 1 D&C) |
| created | [[DEBATE-ADR-011-auto-generated-cli-from-mcp-tools]] | COMPLETE (Round 1: Needs Revision; Round 2: 5 Accept, 1 D&C) |
| updated | [[ANALYSIS-027-installation-mechanics]] | Dependency policy revised |
| updated | [[ANALYSIS-029-platform-config-registry]] | Separator changed to colon, maintenance strategy added |
| updated | [[ANALYSIS-030-auto-generated-cli-from-mcp-tools]] | CLI decisions now in ADR-011 |
| renamed | DEBATE-ADR-008, 009, 010 | From CRIT-NNN to DEBATE-ADR-NNN convention |
| fixed | DEBATE-ADR-001 through 007 | Frontmatter: titles with spaces, type changed to critique, permalinks fixed |
| renamed | ADR-005, ADR-006 | From space-separated to kebab-case file names |
| renamed | ANALYSIS-020 | From space-separated to kebab-case file name |
| created | [[ANALYSIS-031-scaffolding-wizard-patterns]] | COMPLETE |
| created | [[ANALYSIS-032-mcp-sdk-bun-runtime-compatibility]] | COMPLETE |
| created | [[ADR-012-scaffolding-and-content-management]] | ACCEPTED (Round 2: 5 Accept, 1 D&C) |
| created | [[DEBATE-ADR-012-scaffolding-and-content-management]] | COMPLETE (Round 1: unanimous Needs Revision; Round 2: 5 Accept + 1 D&C) |
| deleted | ANALYSIS-031 Scaffolding Wizard Patterns (space-separated) | Duplicate of kebab-case version |
| deleted | CRIT-012 ADR-012 Scaffolding Debate Log | Wrong naming convention, replaced by DEBATE-ADR-012 |
| fixed | 23 analysis notes (ANALYSIS-001 through 030) | Frontmatter titles fixed from kebab-case to Title Case |
| renamed | DEBATE-ADR-012 | From CRIT-012 to DEBATE-ADR-012, then space-separated to kebab-case |
| created | [[ANALYSIS-033-consumer-and-author-commands]] | COMPLETE |
| created | [[ANALYSIS-034-skill-versioning-models-comparison]] | COMPLETE |
| renamed | ANALYSIS-033, ANALYSIS-034 | From space-separated to kebab-case file names |
| created | [[ADR-013-npm-package-distribution-and-revised-command-tree]] | ACCEPTED (Round 2: 5 Accept + 1 D&C). To be superseded by ADR-014. |
| updated | [[ADR-013-npm-package-distribution-and-revised-command-tree]] | ACCEPTED (2026-03-09, Round 2: 5 Accept + 1 D&C) |
| updated | [[DEBATE-ADR-013-npm-package-distribution-and-revised-command-tree]] | COMPLETE (Round 1 + Round 2 verdicts, P2 issues, D&C reservations) |
| created | [[ANALYSIS-047-project-bootstrapping-and-dx-infrastructure]] | PENDING (future workstream) |
| created | [[ANALYSIS-035-hook-merging-strategies-without-custom-lockfile]] | COMPLETE |
| created | [[ANALYSIS-036-ai-platform-plugin-standards-survey]] | COMPLETE |
| created | [[ANALYSIS-037-cross-platform-hook-configuration-analysis]] | COMPLETE |
| created | [[ANALYSIS-038-cross-platform-agent-and-skill-definitions]] | COMPLETE |
| created | [[ANALYSIS-039-cross-platform-mcp-and-instruction-formats]] | COMPLETE |
| created | [[ANALYSIS-040-prior-art-for-cross-platform-adapter-patterns]] | COMPLETE |
| created | [[ANALYSIS-041-plugin-manifest-cross-platform-comparison]] | COMPLETE |
| renamed | ANALYSIS-035 through ANALYSIS-041 | From space-separated to kebab-case file names |
| created | [[ADR-014-explicit-installation-model-and-content-features]] | ACCEPTED (Round 2: 5 Accept + 1 D&C). Supersedes ADR-013. |
| created | [[DEBATE-ADR-014-explicit-installation-model-and-content-features]] | COMPLETE (Round 1 + Round 2 verdicts) |
| created | [[RECONCILIATION-2026-03-09-decision-audit]] | COMPLETE (6 extraction agents, reconciliation) |
| updated | [[ADR-001-plugin-format-and-manifest]] | Amendments #6 (manifest location), #7 (content types 6→8), #8 (installMode→features). ADR-014 relation added. |
| updated | [[ADR-013-npm-package-distribution-and-revised-command-tree]] | Status changed to "superseded" with ADR-014 explanation |
| updated | [[ADR-003-conflict-resolution-and-namespacing]] | Decision 3 supersession note updated for ADR-014 |
| updated | [[ANALYSIS-042-native-platform-plugin-installation]] | Adoption status section added, ADR-014 relation |
| updated | [[ANALYSIS-043-cross-platform-content-model-analysis]] | Adoption status section added, ADR-014 relation |
| updated | [[ANALYSIS-044-feature-selection-mechanism-analysis]] | Adoption status section added, ADR-014 relation |
| updated | [[ADR-014-explicit-installation-model-and-content-features]] | Amendment #1 (plural names, analyze/analyze --fix), Amendment #2 (MCP catalog deferral), Decision 6 command tree updated |
| updated | [[ADR-012-scaffolding-and-content-management]] | Amendment #1 (eval→analyze, improve→analyze --fix cross-reference) |

### Code Files

| File | Context |
|------|---------|
| (none yet) | No code changes in this ideation session |

---

## Observations

- [fact] Design spec is comprehensive at 22 sections covering CLI, MCP, scaffolding, platform support, data storage, and self-bootstrapping #scope
- [fact] Spec pre-selects specific dependencies (gunshi, @clack/prompts, drizzle-orm, orama, fastmcp, etc.) -- each needs research validation #dependencies
- [fact] Self-bootstrapping is a key architectural constraint: the tool uses itself to create its own content #architecture
- [fact] 7 platforms qualify with full 6/6 support: Claude Code, Cursor, Copilot CLI, Kiro, OpenCode, Amp, Windsurf #platforms
- [fact] 3 platforms near-complete at 5.5/6: Codex CLI (hooks notification-only), Cline (no custom sub-agents), Gemini CLI (sequential sub-agents) #platforms
- [fact] Claude Code has 9,000+ plugins and official marketplace; Vercel npx skills has 8,800 GitHub stars for skills-only #ecosystem
- [fact] No existing tool manages full plugin lifecycle across multiple platforms -- this is the gap #market-gap
- [fact] February 2026 saw 6 platforms ship parallel agent execution within a 2-week window #industry-trend
- [fact] AGENTS.md read by 6/7, CLAUDE.md by 4/7, .claude/skills/ by 6/7, .agents/skills/ by 4/7 #cross-platform-coverage
- [fact] 70% of config formats (package.json, tsconfig, Cargo.toml, etc.) handle schema evolution without format version fields #versioning
- [fact] Phase 1 is ~85% complete after 7 Groups + Decision Audit + Gap Analysis #progress
- [fact] ADR-004 Plugin Security Model is CRITICAL gap: 11 items blocked across 4 active ADRs, hook execution literally blocked until it exists #security #critical
- [fact] 9 stale ADR body text instances identified where amendments are correct but inline text not updated #staleness
- [fact] 594 decision points identified across 22 spec sections via comprehensive inventory #scope
- [fact] ADR-014 is now the authoritative design document, superseding ADR-013, ADR-008, ADR-010 #architecture
- [constraint] Agent Teams not working -- using subagent delegation for all specialist work #process
- [constraint] Must manage platform-specific instruction files (CLAUDE.md, AGENTS.md, etc.) without breaking existing content #requirement
- [insight] Each platform has different mechanisms for the same 6 capabilities -- the package must abstract these differences #architecture
- [insight] Agent Skills (SKILL.md with YAML frontmatter) is the most portable extension format across platforms #portability
- [insight] .claude/skills/ has highest cross-platform reach (6/7) due to Claude Code early standard adoption #skills

## Relations

- relates_to [[ANALYSIS-001-agent-plugin-foundation-and-vision]]
- relates_to [[ANALYSIS-002-platform-capability-matrix]]
- relates_to [[ANALYSIS-003-vercel-skills-format-deep-dive]]
- relates_to [[ANALYSIS-004-tanstack-intent-deep-dive]]
- relates_to [[ANALYSIS-005-claude-code-plugin-format]]
- relates_to [[ANALYSIS-006-runtime-cli-framework-validation-stack]]
- relates_to [[ANALYSIS-007-config-schema-versioning-patterns]]
- relates_to [[ANALYSIS-008-platform-instruction-file-paths]]
- relates_to [[ADR-001-plugin-format-and-manifest]]
- relates_to [[ADR-002-target-platforms-and-audiences]]
- relates_to [[ADR-003-conflict-resolution-and-namespacing]]
- relates_to [[DEBATE-ADR-001-plugin-format-and-manifest]]
- relates_to [[DEBATE-ADR-002-target-platforms-and-audiences]]
- relates_to [[DEBATE-ADR-003-conflict-resolution-and-namespacing]]
- relates_to [[ANALYSIS-009-instruction-file-update-patterns]]
- relates_to [[ANALYSIS-010-hook-merge-unmerge-patterns]]
- relates_to [[ANALYSIS-011-lockfile-management-patterns]]
- relates_to [[ANALYSIS-012-json-config-merge-patterns]]
- relates_to [[ANALYSIS-013-input-sanitization-patterns]]
- relates_to [[ANALYSIS-014-platform-config-patterns]]
- relates_to [[ANALYSIS-015-ADR-003-convergence-round2-validation]]
- relates_to [[ANALYSIS-016-bun-runtime-assessment]]
- relates_to [[ANALYSIS-017-cli-framework-comparison]]
- relates_to [[ANALYSIS-018-interactive-prompts-and-colors]]
- relates_to [[ANALYSIS-019-mcp-framework-and-file-watching]]
- relates_to [[ANALYSIS-020-frontmatter-and-markdown-processing]]
- relates_to [[ANALYSIS-021-data-storage-and-search]]
- relates_to [[ANALYSIS-022-clack-prompts-api-surface-and-gaps]]
- relates_to [[ANALYSIS-023-gunshi-command-patterns-and-capabilities]]
- relates_to [[ANALYSIS-024-cli-ci-mode-and-non-interactive-patterns]]
- relates_to [[ADR-005-runtime-and-distribution-strategy]]
- relates_to [[ADR-006-core-dependency-stack]]
- relates_to [[ADR-007-cli-architecture-and-interaction-model]]
- relates_to [[DEBATE-ADR-005-runtime-and-distribution-strategy]]
- relates_to [[DEBATE-ADR-006-core-dependency-stack]]
- relates_to [[DEBATE-ADR-007-cli-architecture-and-interaction-model]]
- relates_to [[ANALYSIS-025-source-resolution-patterns]]
- relates_to [[ANALYSIS-026-platform-detection-and-mapping]]
- relates_to [[ANALYSIS-027-installation-mechanics]]
- relates_to [[ANALYSIS-029-platform-config-registry]]
- relates_to [[ANALYSIS-030-auto-generated-cli-from-mcp-tools]]
- relates_to [[ADR-011-auto-generated-cli-from-mcp-tools]]
- relates_to [[ANALYSIS-028-bun-builtin-api-reliability]]
- relates_to [[ADR-008-source-resolution-and-package-validation]]
- relates_to [[ADR-009-platform-detection-and-config-registry]]
- relates_to [[ADR-010-installation-lifecycle]]
- relates_to [[DEBATE-ADR-008-source-resolution-and-package-validation]]
- relates_to [[DEBATE-ADR-009-platform-detection-and-config-registry]]
- relates_to [[DEBATE-ADR-010-installation-lifecycle-and-cli-generation]]
- relates_to [[DEBATE-ADR-011-auto-generated-cli-from-mcp-tools]]
- relates_to [[ANALYSIS-031-scaffolding-wizard-patterns]]
- relates_to [[ANALYSIS-032-mcp-sdk-bun-runtime-compatibility]]
- relates_to [[ADR-012-scaffolding-and-content-management]]
- relates_to [[DEBATE-ADR-012-scaffolding-and-content-management]]
- relates_to [[ANALYSIS-033-consumer-and-author-commands]]
- relates_to [[ANALYSIS-034-skill-versioning-models-comparison]]
- relates_to [[ADR-013-npm-package-distribution-and-revised-command-tree]]
- relates_to [[DEBATE-ADR-013-npm-package-distribution-and-revised-command-tree]]
- relates_to [[ANALYSIS-035-hook-merging-strategies-without-custom-lockfile]]
- relates_to [[ANALYSIS-036-ai-platform-plugin-standards-survey]]
- relates_to [[ANALYSIS-037-cross-platform-hook-configuration-analysis]]
- relates_to [[ANALYSIS-038-cross-platform-agent-and-skill-definitions]]
- relates_to [[ANALYSIS-039-cross-platform-mcp-and-instruction-formats]]
- relates_to [[ANALYSIS-040-prior-art-for-cross-platform-adapter-patterns]]
- relates_to [[ANALYSIS-041-plugin-manifest-cross-platform-comparison]]
- relates_to [[ANALYSIS-047-project-bootstrapping-and-dx-infrastructure]]
- relates_to [[ADR-014-explicit-installation-model-and-content-features]]
- relates_to [[DEBATE-ADR-014-explicit-installation-model-and-content-features]]
- relates_to [[RECONCILIATION-2026-03-09-decision-audit]]
- relates_to [[ANALYSIS-042-native-platform-plugin-installation]]
- relates_to [[ANALYSIS-043-cross-platform-content-model-analysis]]
- relates_to [[ANALYSIS-044-feature-selection-mechanism-analysis]]

---

## Session End Protocol (BLOCKING)

| Req Level | Step | Status | Evidence |
|-----------|------|--------|----------|
| MUST | Update session status to COMPLETE | [ ] | |
| MUST | Update Brain memory with learnings | [ ] | |
| MUST | Run markdownlint | [ ] | |
| MUST | Commit all changes | [ ] | |
