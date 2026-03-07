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
**Branch:** main
**Starting Commit:** 84f8511 first commit
**Objective:** Work through the `@acmelabz/agent-plugin` comprehensive design specification using the ideation workflow, conducting web research, creating ADRs for architectural decisions, and producing feature specs in the features/ directory

---

## Acceptance Criteria

- [ ] Complete research on all major spec areas (CLI framework, dependencies, platform support, data storage, MCP server, scaffolding wizards)
- [ ] Create ADRs for key architectural decisions identified in the spec
- [ ] Create feature specs in features/ directory following FEAT-NNN template structure
- [ ] All research findings saved as Brain memory notes
- [ ] Session note kept current with all touched files, commits, memory notes, work log

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
- [decision] Package name: `@acmelabz/agent-plugin` (scoped) -- need to register `acmelabz` npm org at npmjs.com #naming
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
- [decision] Intelligent conflict resolution: name collisions (any type) prompt user to prefix or rename, choosing which to modify (existing or incoming), with cross-reference updates. Hook event collisions merge (run both, combine outputs, configurable order). #conflict-resolution
- [decision] Colon namespace separator: plugin-name:component-name pattern for installed components. Platform adapters translate as needed. #namespacing
- [decision] Component format: cross-platform core frontmatter (name, description, type, requires, sources) + platformConfig section for platform-specific overrides #component-format
- [decision] Platform-aware frontmatter generation: on install, emit ONLY fields the target platform supports but include ALL supported fields. Source frontmatter is the superset. #platform-adaptation
- [decision] Track renamed components: installer stores original-name to installed-name mapping for updates, removals, and cross-reference management #conflict-tracking
- [decision] No graceful degradation tier -- only 6/6 platforms supported. Codex CLI, Cline, Gemini CLI excluded. Partial support creates silent failures worse than no support. #platforms #no-degradation
- [decision] No publish command -- no registry means nothing to publish to. Distribution is handled by making the plugin available at any supported source (GitHub, GitLab, npm, local path). #distribution #no-publish
- [decision] Source versioning: all source types support optional version specifier (e.g., owner/repo@v1.2.0, @scope/pkg@^2.0.0). No version = latest. Git sources use tags/releases, npm uses semver. #distribution #versioning
- [decision] Installed version tracking: plugin manager records installed version in state store for accurate upgrade/update behavior #state-management #versioning
- [decision] Invalid version handling: detect invalid version, display available versions via @clack/prompts select picker (including latest) for user to choose #ux #versioning
- [decision] Platform instruction file paths fully specified for all 7 platforms (ANALYSIS-008). Fallback convention: .agents/ directory and AGENTS.md for platforms without well-defined paths #platforms #instruction-files
- [fact] AGENTS.md read by 6/7 platforms, CLAUDE.md by 4/7, .claude/skills/ by 6/7, .agents/skills/ by 4/7 and emerging as standard #platforms #cross-platform-coverage
- [decision] NEG-006 added to ADR-002: instruction file modification is an attack vector, deferred to ADR-004 #security
- [decision] NEG-007 added to ADR-002: no centralized vetting in multi-source model, deferred to ADR-004 #security

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

**Current Position:** Phase 1 > Group 1 > ADR-002 P1 issue resolution (P1-7 of 12)

### Phase 1: Research and Discovery

#### Group 1: Foundation and Vision (Sections 1-3) -- RESEARCH COMPLETE, ADRs IN REVIEW

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
- [x] Naming: CONFIRMED @acmelabz/agent-plugin (scoped). Need to register acmelabz npm org.
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
- [ ] ADR-002 P1-7: Self-bootstrapping acceptance criteria -- IN PROGRESS
- [ ] ADR-002 P1-8: Three-audience bloat risk
- [ ] ADR-002 P1-9: Trust-on-first-use sources
- [ ] ADR-002 P1-10: MCP metadata access control
- [ ] ADR-002 P1-11: Instruction file management feasibility
- [ ] ADR-002 P1-12: Bidirectional ADR-002/ADR-003 link
- [x] ADR-003 created
- [ ] ADR-003 adr-review debate not yet run (after ADR-002 P1s complete)

#### Group 2: Core Technology Stack (Section 4 -- Dependencies) -- STARTING

Background research running: [[ANALYSIS-006-runtime-cli-framework-validation-stack]] (Bun vs Node/Deno, gunshi vs alternatives, zod vs alternatives)

Each dependency is a SUGGESTION from the spec that needs research and validation:

- [ ] Runtime: Bun (vs Node.js, Deno)
- [ ] CLI framework: gunshi (vs commander, yargs, oclif, citty, clipanion)
- [ ] Validation: zod (vs valibot, arktype, typebox)
- [ ] Interactive prompts: @clack/prompts (vs inquirer, prompts, enquirer)
- [ ] Shell completions: @bomb.sh/tab (vs alternatives)
- [ ] Frontmatter parsing: gray-matter (vs alternatives)
- [ ] Markdown processing: micromark + mdast (vs unified/remark, markdown-it)
- [ ] Database: drizzle-orm + SQLite (vs better-sqlite3, prisma, json files)
- [ ] Full-text search: @orama/orama (vs flexsearch, lunr, minisearch)
- [ ] Semantic search: @huggingface/transformers (vs alternatives, or skip entirely)
- [ ] MCP framework: fastmcp (vs @modelcontextprotocol/sdk)
- [ ] File watching: watcher (vs chokidar, node:fs.watch)
- [ ] Colors: picocolors (vs chalk, kleur)

#### Group 3: CLI Architecture (Sections 5-6) -- NOT STARTED

- [ ] Command tree structure
- [ ] @clack/prompts usage patterns
- [ ] CI mode and non-interactive fallbacks

#### Group 4: Source and Platform (Sections 7-9) -- NOT STARTED

- [ ] Source resolution (npm, GitHub, local)
- [ ] Platform support model and detection
- [ ] Installation mechanics (scope, conflicts, hook merging, MCP merging)
- [ ] Manifest format (acmelabz.json vs package.json field vs other approaches)

#### Group 5: Data and Storage (Sections 10-11) -- NOT STARTED

- [ ] Storage approach (SQLite vs JSON lockfile vs other)
- [ ] Schema design
- [ ] Search integration approach

#### Group 6: Scaffolding Wizards (Section 12) -- NOT STARTED

- [ ] Wizard design patterns
- [ ] Content type scaffolding (agent, skill, command, hook, rule, MCP)

#### Group 7: Commands (Sections 13-14) -- NOT STARTED

- [ ] Consumer commands (add, remove, upgrade, list)
- [ ] Author commands (init, validate, build, dev)

#### Group 8: MCP and Self-Bootstrap (Sections 15-18) -- NOT STARTED

- [ ] Embedded MCP server architecture
- [ ] Self-bootstrapping approach
- [ ] Project context system

#### Group 9: Polish (Sections 19-22) -- NOT STARTED

- [ ] Shell completions
- [ ] Bun-native APIs
- [ ] Markdown processing pipeline
- [ ] Implementation phasing

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
- [ ] [research] Runtime/CLI/validation stack -- [[ANALYSIS-006-runtime-cli-framework-validation-stack]] #dependencies (background, in progress)

### Group 1 Discussion

- [x] [decision] Cross-platform full lifecycle management confirmed as target positioning #market
- [x] [decision] Platform criteria locked: prompts + skills + agents + hooks + MCPs + parallel agents #platforms
- [x] [decision] All 7 full-support platforms targeted #platforms
- [x] [decision] Platform instruction file management required on install #new-requirement
- [x] [decision] Full three-audience model confirmed (Consumer + Author + AI) #audiences
- [x] [decision] Package name @acmelabz/agent-plugin confirmed, npm org registration needed #naming
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
- [ ] P1-7 through P1-12: IN PROGRESS #review

### ADR-003 Status

- [x] [adr] ADR-003 Conflict Resolution and Namespacing created #architecture
- [ ] adr-review debate not yet run (pending ADR-002 P1 completion) #review

### File Name Fixes

- [x] [fix] Renamed ADR-001, ADR-002, ADR-003 from space-separated to kebab-case file names #naming
- [x] [fix] Renamed ANALYSIS-008 from space-separated to kebab-case file name #naming
- [x] [fix] Deleted premature ADR-001-target-platforms-and-selection-criteria #cleanup

---

## Requirements Context

From /Users/peter.kloss/Downloads/agent-plugin-design-spec.md:

**Package:** @acmelabz/agent-plugin
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
| created | [[ANALYSIS-006-runtime-cli-framework-validation-stack]] | IN_PROGRESS (background) |
| created | [[ANALYSIS-007-config-schema-versioning-patterns]] | COMPLETE |
| created | [[ANALYSIS-008-platform-instruction-file-paths]] | COMPLETE |
| created | [[ADR-001-plugin-format-and-manifest]] | ACCEPTED |
| created | [[ADR-002-target-platforms-and-audiences]] | ACCEPTED (P1 in progress) |
| created | [[ADR-003-conflict-resolution-and-namespacing]] | ACCEPTED (pending review) |
| created | [[DEBATE-ADR-001-plugin-format-and-manifest]] | COMPLETE (2 Accept, 3 D&C, 1 Block) |
| created | [[DEBATE-ADR-002-target-platforms-and-audiences]] | COMPLETE (0 Accept, 4 D&C, 1 Needs Revision) |
| deleted | ADR-001-target-platforms-and-selection-criteria | Premature duplicate |
| deleted | CRIT-001 ADR-001 Plugin Format and Manifest Review | Ad-hoc, replaced by adr-review skill |
| deleted | REVIEW-ADR-002-target-platforms-and-audiences | Ad-hoc, replaced by adr-review skill |
| deleted | 6x REVIEW-*-ADR-002 individual notes | Consolidated into DEBATE-ADR-002 |
| renamed | ADR-001, ADR-002, ADR-003, ANALYSIS-008 | From space-separated to kebab-case file names |

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

---

## Session End Protocol (BLOCKING)

| Req Level | Step | Status | Evidence |
|-----------|------|--------|----------|
| MUST | Update session status to COMPLETE | [ ] | |
| MUST | Update Brain memory with learnings | [ ] | |
| MUST | Run markdownlint | [ ] | |
| MUST | Commit all changes | [ ] | |