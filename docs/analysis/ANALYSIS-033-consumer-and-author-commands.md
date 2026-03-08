---
title: ANALYSIS-033 Consumer and Author Commands
type: analysis
permalink: analysis/analysis-033-consumer-and-author-commands-1
tags:
- commands
- consumer
- author
- cli
- spec-analysis
- group-7
---

# ANALYSIS-033 Consumer and Author Commands

## 1. Objective and Scope

**Objective**: Catalog every consumer and author command from design spec Sections 13-14. Compare against existing ADR decisions. Identify conflicts, gaps, and dependencies.

**Scope**: Consumer commands (add, remove, upgrade, list) and author commands (init, validate, build, dev, publish). Content-type command groups (skill, agent, mcp, command, hook, instruction) are covered only where they interact with consumer/author commands. MCP tool mappings included. Scaffolding wizard internals excluded (covered in ANALYSIS-031).

## 2. Context

The design spec predates the ADR process. ADRs 001-012 formalized decisions that refined, amended, or rejected spec proposals. This analysis identifies where the spec and ADRs agree, diverge, or leave gaps.

Key prior decisions constraining this analysis:
- ADR-001: plugin.json mandatory, 5-component model (commands added as 6th by ADR-012)
- ADR-002: No hosted registry, no `publish` command, multi-source distribution
- ADR-003: Always-namespace with colon separator, no conflict resolution flags needed
- ADR-007: Command tree, global flags, three-tier input resolution, @clack/prompts mapping
- ADR-009: Platform detection via platforms.config.json, colon-namespaced MCP keys
- ADR-010: 6-phase install lifecycle, project-scope default, dependency installation with confirmation, upgrade via atomic replace
- ADR-011: Auto-generated CLI from MCP tools, MCP daemon lifecycle
- ADR-012: Content-type command groups replace `new` subtree, creator skills for eval/improve

## 3. Approach

**Methodology**: Line-by-line comparison of spec Sections 13-14 against ADR decisions. Each command cataloged with flags, interactive flow, CI behavior, and MCP tool mapping. Conflicts and gaps flagged with severity.

**Tools Used**: Brain memory read (ADRs 001-012), design spec file read

**Limitations**: No access to the spec's Section 11 (CLI architecture overview) which may contain additional flag details. Spec uses old naming conventions (acmelabz.json, fastmcp, new subtree) that ADRs have superseded.

## 4. Data and Analysis

### Section 13: Consumer Commands

#### 4.1 `add <source>`

**Spec definition (10 steps)**:
1. Parse source string to ParsedSource
2. Check if already installed, offer reinstall
3. Fetch manifest from npm/github/local
4. Show package contents summary (p.note())
5. Check system dependencies, progress bar installation
6. Grouped prompt: scope + platforms (p.group())
7. Detect conflicts, resolution prompt
8. Stage content to temp dir
9. Install to each platform with progress bar
10. Record in database

**Spec CI mode**: All prompts skipped, uses `--scope`, `--platform`, `--conflict` flags. Defaults: scope=global, platforms=all detected compatible, conflict=overwrite.

**ADR-010 6-phase model**: Detect, Select, Resolve, Confirm, Apply, Record

**Comparison**:

| Aspect | Spec | ADR Decision | Status |
|---|---|---|---|
| Flow structure | 10 informal steps | 6 formal phases with rollback | [RESOLVED] ADR-010 supersedes |
| Scope default | Not specified in steps, CI default=global | Project scope default, --global flag | [CONFLICT] Spec CI default is global; ADR-010 default is project |
| Platform flag name | `--platform` (singular) | `--platforms` (plural, ADR-007 Decision 5) | [CONFLICT] Spec uses singular |
| Conflict flag | `--conflict` flag exists | Removed entirely (ADR-003 always-namespace) | [CONFLICT] Spec has dead flag |
| Conflict detection step | Step 7: detect conflicts, resolution prompt | Not needed per ADR-003 | [CONFLICT] Spec step is obsolete |
| Dependency handling | Step 5: progress bar installation (implicit auto) | Multiselect confirmation, user chooses which to install | [CONFLICT] Spec skips user consent |
| Staging | Step 8: stage to temp dir | Phase 5 Apply: atomic writes, pre-merge snapshots | [RESOLVED] ADR-010 is more detailed |
| State storage | "Record in database" | JSON lockfile (plugin-lock.json) | [CONFLICT] Spec says "database" |
| Reinstall check | Step 2: offer reinstall | Not explicitly specified in ADR-010 | [GAP] ADR-010 silent on reinstall |
| Package summary | Step 4: p.note() summary | Not specified in ADR-010 | [GAP] UX detail not in ADR |
| Integrity verification | Not mentioned | IMP-008: SHA-512 for npm, commit SHA for git, content hash for local | [GAP] Spec missing security step |

**Spec flags for `add`**:

| Flag | Spec | ADR-007 Decision 5 |
|---|---|---|
| `--scope` | Present | Present (global or project) |
| `--platform` | Present (singular) | `--platforms` (plural, comma-separated) |
| `--conflict` | Present | Removed (ADR-003) |

**MCP tool mapping**: Spec Section 15 defines `acmelabz_add` as the MCP tool. ADR-007 Decision 4 requires three-tier input resolution. Parameters match wizard inputs per ADR-012 Decision 3 schema-first architecture.

#### 4.2 `remove [source]`

**Spec definition (5 steps)**:
1. Load installed sources from database
2. If source specified: find it. If not: p.autocompleteMultiselect() for >5 sources, p.multiselect() for <=5
3. Confirm removal
4. Remove with progress bar (per-source)
5. Check for orphaned dependencies, offer uninstall via p.multiselect()

**ADR-010 uninstall flow (6 steps)**:
1. Locate: Read plugin entry from plugin-lock.json
2. Remove files: Delete files from lockfile inventory
3. Remove MCP entries: Delete colon-namespaced keys from platform configs
4. Remove hook contributions: Recompute merged hooks
5. Offer dependency uninstall: Multiselect (none selected by default)
6. Update lockfile: Atomic write

**Comparison**:

| Aspect | Spec | ADR Decision | Status |
|---|---|---|---|
| Terminology | "remove" command | ADR-010 text says "uninstall" in flow heading, but ADR-007 command tree says "remove" | [WARNING] Mixed terminology in ADR-010 |
| Source selection UX | autocompleteMultiselect for >5, multiselect for <=5 | Not specified in ADR-010 | [GAP] ADR-010 does not specify selection UX for multi-remove |
| Dep uninstall defaults | Not specified | None selected by default | [RESOLVED] ADR-010 specifies |
| State storage | "database" | plugin-lock.json | [CONFLICT] Same as add |
| MCP entry cleanup | Not mentioned | Explicit step: delete colon-namespaced keys | [GAP] Spec misses MCP cleanup |
| Hook recompute | Not mentioned | Explicit step: overlay/recompute | [GAP] Spec misses hook recompute |

**Spec CI mode**: Not explicitly stated for remove. ADR-010 specifies: skip dep uninstall prompt unless `--remove-deps` is passed.

**MCP tool mapping**: Spec Section 15 defines `acmelabz_remove`.

#### 4.3 `upgrade [source[@version]]`

**Spec definition**:

*Specific source*:
1. Parse source (may include @version pin)
2. If version pinned: upgrade to exact version
3. If not: check npm registry for latest
4. Show diff, confirm, perform upgrade

*All sources (no argument)*:
1. Check all installed sources in parallel with progress bar
2. Show up-to-date sources
3. p.autocompleteMultiselect() for upgradable sources (all pre-selected)
4. Show summary in p.note()
5. Confirm, upgrade with progress bar

**Upgrade mechanics**: Remove old content, install new content, update database entry.

**ADR-010 Decision 4 upgrade flow**:
1. Scan all installed plugins from lockfile
2. Check latest versions from original sources
3. Present multiselect showing current->latest versions
4. Each plugin: install-then-remove (install new to staging, swap atomically, remove old)

**Comparison**:

| Aspect | Spec | ADR Decision | Status |
|---|---|---|---|
| Upgrade order | Remove old, install new | Install new first, then remove old | [CONFLICT] ADR-010 reverses the order for safety |
| Version pinning | `@version` in source string | --version flag (ADR-007 Decision 5) AND inline @version | [RESOLVED] Both supported |
| Downgrade handling | Not mentioned | Warn and require --force | [GAP] Spec silent on downgrade |
| Breaking changes | Not mentioned | Compare old/new manifests, warn on removed components | [GAP] Spec silent on breaking changes |
| All-sources check | "in parallel with progress bar" | Matches ADR-010 | [RESOLVED] |
| Show up-to-date | Spec explicitly shows them | Not specified in ADR-010 | [GAP] UX detail |
| State storage | "database entry" | plugin-lock.json | [CONFLICT] Same naming issue |

**Spec flags for `upgrade`**: `--version` (from ADR-007 Decision 5).

**MCP tool mapping**: Spec Section 15 defines `acmelabz_upgrade`.

#### 4.4 `list`

**Spec definition**:
- Interactive: p.box() per source showing version, scope, platforms, content counts, install/update dates
- CI/JSON: structured JSON array of all sources

**ADR coverage**: ADR-007 Decision 7 maps p.box() to the list command. ADR-007 Decision 2 defines --json flag behavior.

**Comparison**:

| Aspect | Spec | ADR Decision | Status |
|---|---|---|---|
| Interactive display | p.box() per source | Confirmed in ADR-007 component mapping | [RESOLVED] |
| JSON output | Structured JSON array | --json flag produces JSON envelope per ADR-007 Decision 10 | [RESOLVED] |
| Data fields | version, scope, platforms, content counts, install/update dates | Not enumerated in ADR | [GAP] Exact fields not specified in ADR |

**MCP tool mapping**: Spec Section 15 defines `acmelabz_list`.

### Section 14: Author Commands

#### 4.5 `init`

**Spec definition**:

*Fresh project (no manifest)*:
- Grouped prompts: name, description, version, platforms, content types, content dir, dist dir, dependencies
- Scaffolds: acmelabz.json, .acmelabz/project.json, content directories with template files, package.json (if none), .gitignore additions
- Progress bar for scaffolding steps

*Existing project*:
- Shows current config in p.note()
- Offers multiselect of what to update: metadata, platforms, content types, dependencies, scan for undeclared content
- Updates manifest and project config

**ADR coverage**: ADR-007 component mapping confirms p.group() for init. ADR-012 does not detail `init` wizard fields (focuses on content-type wizards).

**Comparison**:

| Aspect | Spec | ADR Decision | Status |
|---|---|---|---|
| Manifest filename | acmelabz.json | plugin.json (ADR-001) | [CONFLICT] Spec uses old name |
| Project config | .acmelabz/project.json | Not addressed in any ADR | [GAP] No ADR covers project config |
| Platforms prompt | Asks which platforms to target | ADR-001 Decision 6: all plugins are inherently cross-platform, no platforms field | [CONFLICT] Spec asks for platforms; ADR says all platforms always |
| Dependencies prompt | Asks about dependencies | ADR-001: dependencies deferred to future ADR | [GAP] No dependency management ADR exists |
| Existing project update | Multiselect of sections to update | Not specified in any ADR | [GAP] Update mode not formalized |
| Content dir prompt | Asks for content directory path | Not specified in ADR | [GAP] |
| Dist dir prompt | Asks for dist directory path | ADR-007 Decision 8: dist dir must not equal content dir | [RESOLVED] Validation exists |

**Spec flags**: None specified for init in CI mode. This is a gap: init in CI mode needs all parameters as flags.

**MCP tool mapping**: No MCP tool for init in Section 15. Spec defines `acmelabz_project_info` for reading project state but no tool for creating projects.

#### 4.6 `validate`

**Spec definition (5 steps)**:
1. Load manifest
2. Run validation rules (all via Zod internally)
3. Check file path references exist on disk
4. Discover undeclared content
5. Report errors/warnings with fix suggestions
6. Exit code 1 if errors (for CI)

**ADR coverage**: ADR-007 Decision 8 defines 16 Zod validation rules. ADR-012 Decision 5 specifies the `validate` command detects drift between filesystem and manifest.

**Comparison**:

| Aspect | Spec | ADR Decision | Status |
|---|---|---|---|
| Validation engine | Zod internally | Zod schemas per ADR-007 Decision 8 | [RESOLVED] |
| Path reference check | Step 3: verify paths exist | ADR-012 Decision 5: validate detects drift | [RESOLVED] |
| Undeclared content | Step 4: discover undeclared | ADR-012 Decision 5: validate detects drift | [RESOLVED] |
| Fix suggestions | Step 5: report with fix suggestions | Not specified in ADR | [GAP] Fix suggestion format undefined |
| Exit codes | 1 if errors | ADR-007: 1=runtime error, 2=usage error | [WARNING] Validation errors may need exit code 1 or 2 depending on context |

**MCP tool mapping**: Spec Section 15 defines `acmelabz_validate`.

#### 4.7 `build`

**Spec definition (6 steps)**:
1. Validate first (fail fast)
2. Clean dist directory
3. Copy content files with progress bar
4. Copy skill subdirectories (scripts/, references/)
5. Rewrite manifest paths to dist-relative
6. Sync version to package.json

**ADR coverage**: No ADR specifically covers the build command in detail.

**Comparison**:

| Aspect | Spec | ADR Decision | Status |
|---|---|---|---|
| Validate-first | Step 1 | Not in ADR but consistent with ADR-007 fail-fast principle | [RESOLVED] |
| Manifest path rewrite | Step 5 | Not specified in any ADR | [GAP] Path rewriting logic undefined |
| Version sync | Step 6: sync to package.json | Spec Section 14 publish also syncs to 3 files | [WARNING] Two commands modify version |
| Dist structure | Copy content + skill subdirs | Not specified in ADR | [GAP] Build output structure undefined |
| Progress reporting | Progress bar | ADR-007 component mapping: p.progress() for building | [RESOLVED] |

**MCP tool mapping**: No MCP tool for build in Section 15. Build is a local authoring operation.

#### 4.8 `dev`

**Spec definition (6 steps)**:
1. Grouped prompt: platforms + scope
2. Initial install with p.taskLog()
3. Start watcher on content directory (recursive, 300ms debounce)
4. Also watch acmelabz.json
5. On change: validate, install, log result
6. Ctrl+C to stop

**ADR coverage**: ADR-007 Decision 7 maps p.taskLog() to initial install in dev command.

**Comparison**:

| Aspect | Spec | ADR Decision | Status |
|---|---|---|---|
| Watch target | acmelabz.json | Should be plugin.json per ADR-001 | [CONFLICT] Spec uses old name |
| Platforms prompt | Asks which platforms | Consistent with ADR-010 Phase 2 Select | [RESOLVED] |
| Scope prompt | Asks scope | Consistent with ADR-010 Decision 1 | [RESOLVED] |
| Debounce | 300ms | Not specified in ADR | [GAP] Implementation detail |
| Watcher library | Not specified | Not specified in ADR | [GAP] No dependency decision for file watching |

**MCP tool mapping**: No MCP tool for dev. Dev is an interactive local-only operation.

#### 4.9 `publish`

**Spec definition (6 steps)**:
1. Validate
2. Version bump: p.select() (none, patch, minor, major, custom)
3. Update version in: acmelabz.json, package.json, .acmelabz/project.json
4. Build
5. Confirm publish
6. Run `bun publish --access public` with p.taskLog()

**ADR coverage**: ADR-002 Decision 4 explicitly states "There is no `publish` command -- distribution is handled by making the plugin available at any supported source." ADR-007 Decision 1 confirms: "Rejected: `publish` command (ADR-002: no hosted registry)."

**Comparison**:

| Aspect | Spec | ADR Decision | Status |
|---|---|---|---|
| Command existence | publish command exists | **REJECTED** by ADR-002 and ADR-007 | [CONFLICT] MAJOR: Spec defines a command ADRs explicitly rejected |
| Version bump | Part of publish flow | No ADR covers version bump as standalone | [GAP] Version bump functionality has no home |
| npm publish | bun publish --access public | ADR-002: plugins distributed via any source, not npm-only | [CONFLICT] Spec assumes npm-only publishing |

This is the single largest conflict between spec and ADRs. The spec includes a full `publish` command workflow. ADRs 002 and 007 explicitly rejected it. The version bump and build functionality may need a standalone `version` command or be folded into `build`.

### Section 15: MCP Tool Mapping Summary

**Spec defines these MCP tools**:

| Tool | Maps To | Status |
|---|---|---|
| `acmelabz_list` | `list` command | [RESOLVED] |
| `acmelabz_add` | `add` command | [RESOLVED] |
| `acmelabz_remove` | `remove` command | [RESOLVED] |
| `acmelabz_upgrade` | `upgrade` command | [RESOLVED] |
| `acmelabz_project_info` | Project metadata query | [RESOLVED] |
| `acmelabz_project_content` | Content listing | [RESOLVED] |
| `acmelabz_project_mcps` | MCP server listing | [RESOLVED] |
| `acmelabz_validate` | `validate` command | [RESOLVED] |
| `acmelabz_new_agent` | `agent create` (ADR-012) | [WARNING] Spec uses old `new` naming |
| `acmelabz_new_skill` | `skill create` (ADR-012) | [WARNING] Spec uses old `new` naming |
| `acmelabz_new_command` | `command create` (ADR-012) | [WARNING] Spec uses old `new` naming |
| `acmelabz_new_hook` | `hook create` (ADR-012) | [WARNING] Spec uses old `new` naming |
| `acmelabz_new_rule` | `instruction create` (ADR-012) | [WARNING] Spec uses old name "rule" |
| `acmelabz_new_mcp_init` | `mcp create` (ADR-012) | [WARNING] Spec uses old `new` naming |
| `acmelabz_new_mcp_add_tool` | `mcp create-tool` (ADR-012) | [WARNING] Spec uses old `new` naming |

**Missing MCP tools** (functionality exists in ADRs but no spec MCP tool):
- No MCP tool for `init` (project creation)
- No MCP tool for `build`
- No MCP tool for `dev`
- No MCP tool for content-type `remove` subcommands
- No MCP tool for content-type `list` subcommands
- No MCP tool for `eval` or `improve` (ADR-012 creator skills)

**Tool naming convention**: Spec uses `acmelabz_` prefix. This needs updating to `acmelabs_15_` or a shorter chosen prefix after the org rename from acmelabz to acmelabs-15.

### Cross-Cutting Concerns

#### 4.10 Manifest Filename Discrepancy

The spec consistently uses `acmelabz.json` as the manifest filename. ADR-001 established `plugin.json`. This affects:
- `init` command (scaffolds wrong filename)
- `dev` command (watches wrong file)
- `build` command (reads/rewrites wrong file)
- `publish` command (updates wrong file)
- `validate` command (loads wrong file)

All 5 author commands reference the wrong manifest filename. This is a systematic spec-vs-ADR divergence.

#### 4.11 "Database" vs Lockfile

The spec uses "database" to describe state storage in 3 places (add step 10, remove step 1, upgrade mechanics). ADRs established plugin-lock.json as a JSON lockfile. The spec may have been written before the lockfile decision, or it may have envisioned SQLite/similar. ADR-003 settled this as JSON lockfile with atomic writes.

#### 4.12 Content-Type Command Interaction with Consumer/Author Commands

ADR-012 Decision 1 defined content-type command groups (skill, agent, mcp, command, hook, instruction) each with create/remove/list subcommands. These interact with consumer/author commands:

| Interaction | Description |
|---|---|
| Content create -> validate | Creating content should auto-update manifest (ADR-012 Decision 5). validate detects drift. |
| Content create -> build | New content must be included in build output. |
| Content create -> dev | File watcher picks up new files and auto-installs. |
| Content remove -> validate | Removing content should auto-update manifest. |
| add -> content list | After add, content-type list shows installed plugin content. |
| remove -> content list | After remove, content-type list no longer shows removed content. |

No conflicts identified. The content-type commands operate on author-side content. Consumer add/remove operate on installed plugin packages. These are orthogonal concerns.

#### 4.13 Watcher Dependency Gap

The `dev` command requires a file system watcher with recursive watching and debounce. No ADR addresses which watcher to use. Options:
- `node:fs.watch` (built into Bun, recursive support varies by OS)
- `chokidar` (mature, but adds a dependency against ADR-006 minimalism)
- Bun's native `Bun.file().watch()` capabilities

This needs a decision or at minimum an implementation note.

#### 4.14 MCP Server Technology Discrepancy

Spec Section 15 states the embedded MCP server uses "fastmcp ^3.0.0". ADR-006 and ADR-012 explicitly rejected fastmcp in favor of `@modelcontextprotocol/sdk`. This affects the `mcp serve` command implementation but not the command interface.

## 5. Results

### Conflict Summary

| ID | Severity | Spec Says | ADR Says | Resolution Needed |
|---|---|---|---|---|
| C-001 | P0 | `publish` command exists with full workflow | Rejected by ADR-002 and ADR-007 | Drop publish; decide where version bump lives |
| C-002 | P1 | Manifest is `acmelabz.json` | Manifest is `plugin.json` (ADR-001) | Spec references need updating |
| C-003 | P1 | `--conflict` flag on add | Removed by ADR-003 (always-namespace) | Drop flag |
| C-004 | P1 | Upgrade order: remove old then install new | ADR-010: install new then remove old | Follow ADR-010 |
| C-005 | P2 | `--platform` (singular) | `--platforms` (plural, ADR-007) | Use plural |
| C-006 | P2 | CI default scope=global | ADR-010 default scope=project | Follow ADR-010 |
| C-007 | P2 | "database" for state storage | plugin-lock.json (ADR-003) | Use lockfile |
| C-008 | P2 | Embedded MCP uses fastmcp | ADR-006/012: @modelcontextprotocol/sdk | Use SDK |
| C-009 | P2 | init asks for target platforms | ADR-001: all plugins inherently cross-platform | Drop platforms prompt from init |
| C-010 | P2 | MCP tools use `acmelabz_new_*` naming | ADR-012: content-type command groups replace `new` tree | Rename tools |
| C-011 | P2 | rules/new rule | ADR-012: instructions/instruction create | Use new naming |
| C-012 | P2 | Dependency auto-install without explicit consent | ADR-010: multiselect confirmation required | Follow ADR-010 |

### Gap Summary

| ID | Severity | Description | Needs ADR? |
|---|---|---|---|
| G-001 | P1 | Version bump functionality has no command after publish rejection | Decision needed |
| G-002 | P1 | File watcher dependency for dev command undecided | Implementation note or ADR-006 amendment |
| G-003 | P1 | init CI mode flags not specified | ADR-007 amendment or new ADR |
| G-004 | P2 | .acmelabz/project.json not covered by any ADR | Decision needed |
| G-005 | P2 | Reinstall detection/flow not in ADR-010 | ADR-010 amendment |
| G-006 | P2 | Build output structure and path rewriting not formalized | Implementation note |
| G-007 | P2 | MCP tools missing for init, build, content remove/list, eval/improve | ADR-011 or MCP tool ADR |
| G-008 | P2 | Validate exit code semantics (1 vs 2) need clarification | ADR-007 amendment |
| G-009 | P3 | List command data fields not enumerated in ADR | Implementation note |
| G-010 | P3 | Fix suggestion format for validate not defined | Implementation note |
| G-011 | P3 | MCP tool name prefix needs update after org rename | Naming decision |

## 6. Discussion

### The Publish Command Problem

The most significant finding is C-001. The spec defines a full `publish` workflow (validate, version bump, build, npm publish). ADR-002 and ADR-007 rejected this command because the project uses a no-registry distribution model. Plugins reach users via any supported source (npm, GitHub, GitLab, local path), not a centralized registry.

However, the spec's `publish` command contains 2 useful sub-workflows:
1. **Version bump**: Select bump type, update version across 3 files
2. **Build + validate + npm publish**: Sequenced publishing pipeline

Option A: Add a standalone `version` command for bumping and fold build validation into `build --check`. npm publishing is left to the author's own workflow.
Option B: Restore `publish` but scope it to "prepare for distribution" rather than "publish to registry". It runs validate + build + version bump but does NOT run `bun publish`.
Option C: Keep `build` and add `--bump` flag to it.

Each option has tradeoffs. This needs a formal decision.

### The Database vs Lockfile Terminology

The spec's use of "database" in 3 places suggests early design may have considered SQLite or similar for state storage. ADR-003 settled on a JSON lockfile. The spec also mentions "drizzle content index" and "orama search index" in Section 12 wizard outputs, which implies richer state storage than a lockfile provides. These references may be vestigial from a more complex architecture that was simplified during ADR discussions.

### Init Command Complexity

The `init` command has two modes (fresh project vs existing project update) with significant interactive flows. No ADR covers init in detail. The platforms prompt conflicts with ADR-001's cross-platform-by-default stance. The dependencies prompt has no backing ADR. The .acmelabz/project.json config file is not documented in any ADR.

This suggests `init` needs its own ADR or a significant expansion of ADR-012.

### MCP Tool Coverage Gaps

Section 15 defines 15 MCP tools. ADR-012 added eval/improve workflows and content-type remove/list subcommands that have no MCP tool equivalents. The MCP tool naming still uses the old `acmelabz_new_*` pattern instead of content-type groups. A comprehensive MCP tool inventory needs updating.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Formally decide what replaces `publish` command (version bump + distribution prep) | Spec defines a full workflow that ADRs rejected. The functionality gap is real. | 1 ADR |
| P1 | Create ADR or decision record for `init` command details | init has 2 modes, conflicts with ADR-001 on platforms, references undocumented .acmelabz/project.json | 1 ADR |
| P1 | Decide file watcher strategy for `dev` command | dev command requires recursive file watching with debounce. No dependency decision exists. | ADR-006 amendment |
| P1 | Update MCP tool inventory to match ADR-012 command groups | 7 scaffolding tools use obsolete `new_*` naming. Missing tools for eval/improve, content remove/list. | ADR-011 amendment |
| P2 | Add reinstall detection to ADR-010 | Spec step 2 of `add` checks for existing install. ADR-010 is silent on this. | ADR-010 amendment |
| P2 | Clarify validate exit codes (1 vs 2) | ADR-007 defines 1=runtime, 2=usage. Validation errors could be either. | ADR-007 amendment |
| P2 | Document build output structure | Path rewriting, dist directory layout, and version sync are spec-only details. | Implementation note |
| P3 | Update MCP tool prefix after org rename | acmelabz_ prefix is stale. Need acmelabs_15_ or shortened prefix. | Naming decision |

## 8. Conclusion

**Verdict**: Proceed with ADR creation for identified gaps
**Confidence**: High
**Rationale**: 12 conflicts identified between spec and ADRs, all resolvable. 11 gaps identified, 3 at P1 severity requiring formal decisions before implementation. The spec is a useful starting point but has diverged from ADR decisions in naming, command existence, and interaction patterns.

### User Impact

- **What changes for you**: The publish command will not exist. A replacement for version bumping is needed. All other consumer/author commands are well-defined across spec and ADRs with minor reconciliation needed.
- **Effort required**: 2-3 ADRs to close P0 and P1 gaps. Remaining items are implementation notes or minor ADR amendments.
- **Risk if ignored**: Implementers will face ambiguity on publish replacement, init behavior, dev watcher choice, and MCP tool naming. These will cause rework if decided ad-hoc during implementation.

## 9. Appendices

### Complete Command Inventory (Reconciled)

```text
agent-plugin
├── Consumer Commands
│   ├── add <source>           [ADR-007, ADR-010] Install plugin to platforms
│   │   Flags: --scope, --platforms, --global
│   ├── remove [source]        [ADR-007, ADR-010] Uninstall plugin(s)
│   │   Flags: --remove-deps, --yes
│   ├── upgrade [source[@ver]] [ADR-007, ADR-010] Check/apply updates
│   │   Flags: --version, --force
│   └── list                   [ADR-007] Show installed plugins
│       Flags: --json
├── Author Commands
│   ├── init                   [ADR-007] Initialize or update project
│   │   Flags: UNDEFINED for CI mode
│   ├── validate               [ADR-007, ADR-012] Check manifest integrity
│   │   Flags: (none specific)
│   ├── build                  [ADR-007] Build for distribution
│   │   Flags: UNDEFINED
│   └── dev                    [ADR-007] Watch + auto-install locally
│       Flags: --scope, --platforms
├── Content-Type Commands      [ADR-012]
│   ├── skill: create, remove, list, eval, improve
│   ├── agent: create, remove, list, eval, improve
│   ├── mcp: create, create-tool, remove, remove-tool, list, eval, improve
│   ├── command: create, remove, list
│   ├── hook: create, remove, list
│   └── instruction: create, remove, list, eval, improve
├── MCP Server                 [ADR-007]
│   └── mcp serve              Start embedded MCP tool server
├── Shell Completions          [ADR-007]
│   └── complete <shell>       Generate completion script
└── help                       Auto-generated by gunshi
```

**NOT included** (rejected by ADR-002/007): `publish`

### Sources Consulted

- Design spec Sections 13, 14, 15 (lines 750-913)
- Design spec Section 12 (lines 610-748) for wizard cross-references
- ADR-001 Plugin Format and Manifest
- ADR-002 Target Platforms and Audiences
- ADR-003 Conflict Resolution and Namespacing (referenced, not fully read)
- ADR-007 CLI Architecture and Interaction Model
- ADR-009 Platform Detection and Config Registry (referenced)
- ADR-010 Installation Lifecycle
- ADR-011 Auto-Generated CLI from MCP Tools
- ADR-012 Scaffolding and Content Management
- ANALYSIS-031 Scaffolding Wizard Patterns (referenced)

### Data Transparency

- **Found**: Complete spec text for Sections 13-14. Full content of ADRs 001, 002, 007, 010, 011, 012. MCP tool definitions from Section 15.
- **Not Found**: ADR-004 (Plugin Security Model), ADR-003 full text (used summary from other ADRs), Section 11 of spec (CLI architecture overview which may contain additional flag definitions). No existing analysis of consumer/author commands to compare against.

## Observations

- [fact] Design spec Section 13 defines 4 consumer commands (add, remove, upgrade, list) with 10-step add flow, 5-step remove flow, dual-mode upgrade, and formatted list #consumer #commands
- [fact] Design spec Section 14 defines 5 author commands (init, validate, build, dev, publish) with dual-mode init and 6-step publish flow #author #commands
- [fact] 12 conflicts identified between spec and ADRs, ranging from P0 (publish command rejection) to P2 (naming discrepancies) #conflicts #spec-vs-adr
- [fact] 11 gaps identified in ADR coverage, 3 at P1 severity: publish replacement, init details, dev watcher dependency #gaps
- [decision] Spec's publish command is rejected by ADR-002 and ADR-007. Version bump functionality needs a new home. #publish #conflict
- [problem] Spec uses acmelabz.json throughout; ADR-001 established plugin.json. All 5 author commands reference the wrong filename. #manifest #naming
- [problem] Spec uses "database" for state storage in 3 places; ADR-003 established JSON lockfile (plugin-lock.json) #state #naming
- [insight] The init command has the largest ADR gap: 2 modes, conflicts with ADR-001 on platforms, references undocumented .acmelabz/project.json #init #gap
- [insight] MCP tool inventory needs updating: 7 tools use obsolete new_* naming, missing tools for eval/improve and content remove/list #mcp-tools #gap
- [risk] Implementing without resolving the publish gap will cause ad-hoc decisions during development #planning #risk

## Relations

- relates_to [[ADR-007 CLI Architecture and Interaction Model]]
- relates_to [[ADR-010 Installation Lifecycle]]
- relates_to [[ADR-011 Auto-Generated CLI from MCP Tools]]
- relates_to [[ADR-012 Scaffolding and Content Management]]
- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-002 Target Platforms and Audiences]]
- relates_to [[ANALYSIS-031 Scaffolding Wizard Patterns]]