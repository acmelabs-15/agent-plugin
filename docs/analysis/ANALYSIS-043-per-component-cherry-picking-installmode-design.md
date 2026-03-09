---
title: ANALYSIS-043 Per-Component Cherry-Picking InstallMode Design
type: analysis
permalink: analysis/analysis-043-per-component-cherry-picking-installmode-design
tags:
- installMode
- cherry-picking
- component-selection
- plugin-format
- architecture
- UX
---

# ANALYSIS-043 Per-Component Cherry-Picking InstallMode Design

## 1. Objective and Scope

**Objective**: Evaluate whether the current `installMode` binary (`bundle` | `collection`) should be replaced or extended with a per-component cherry-picking model that lets users select individual skills, agents, hooks, commands, instructions, and MCP servers from within a single plugin.

**Scope**: Granularity options with tradeoffs, UX flow design, dependency/safety implications, interaction with ADR-013 npm distribution model, and proposed `installMode` redesign. Excludes implementation-level code design and security model changes (ADR-004 scope).

## 2. Context

ADR-001 defines `installMode` as a binary choice:

- `bundle`: install all components or nothing. No chooser UI.
- `collection`: components are independent. Installer presents a chooser UI so users pick which to install.

ADR-013 pivoted to npm-package distribution: plugins are npm packages installed via `bun add`, and `agent-plugin install` wires their content to platform configs. The npm package downloads ALL components; cherry-picking happens at the wiring stage, not the download stage.

The user proposes a Vercel-inspired model where individual components from ANY type are independently selectable. Example: a plugin has 5 skills, 3 agents, 2 hooks. User wants skill-1, skill-3, agent-2, and hook-1 but NOT the rest.

### Current `installMode` Gap

ADR-001's own "Future Considerations" section identifies this gap:

> "A plugin with 3 independent skills that all require the same MCP server has no way to express 'install any skill, but MCP is mandatory.'"

DEBATE-ADR-001 also flagged `installMode` binary as insufficient (Critic P1-4, Independent Thinker challenge #4). ANALYSIS-041 confirmed that `installMode` has no equivalent on any other platform, making it a novel concept with no ecosystem precedent to validate.

## 3. Approach

**Methodology**: Cross-ecosystem comparison (VS Code extension packs, Homebrew Bundle, apt meta-packages, Arch Linux package groups, Vercel Skills, TanStack Intent). Analysis of existing ADR decisions and debate findings. Evaluation against ADR-013 npm distribution model.

**Tools Used**: Brain MCP (existing analysis notes), WebSearch (ecosystem comparisons), file reading (ADRs and session notes).

**Limitations**: No user research data on how often consumers want partial plugin installation. Frequency estimate is based on ecosystem patterns, not measured demand.

## 4. Data and Analysis

### 4.1 Cross-Ecosystem Precedent for Cherry-Picking

| System | Selection Model | Granularity | UX Pattern |
|---|---|---|---|
| **VS Code Extension Packs** | All-or-nothing install, individual uninstall after | Pack level | Install pack, then disable individual extensions |
| **Homebrew Bundle** | All-or-nothing, skip via env vars | Per-formula | `$HOMEBREW_BUNDLE_BREW_SKIP=pkg1,pkg2` |
| **apt meta-packages** | Install all deps, remove individuals after | Per-package | Install meta-package, `apt remove` individuals |
| **Arch Linux pacman groups** | Interactive numbered selection at install time | Per-package | `pacman -S group` shows numbered list, user enters `1 3 5` or `^2` to exclude |
| **Vercel Skills** | Per-skill selection from a source repo | Per-skill only | `--skill` flag or interactive multiselect when source has multiple skills |
| **TanStack Intent** | All-or-nothing (skills travel with library) | Package level | No selection. Skills are a feature of the npm package. |
| **Claude Code Plugins** | All-or-nothing | Plugin level | No component-level selection within a plugin. |

**Key finding**: Only Arch Linux pacman provides true interactive cherry-picking at install time. VS Code and apt use a "install all, remove later" pattern. Vercel provides skill-level selection but only for skills (not agents, hooks, etc.). No system offers cross-type component cherry-picking from a single package.

### 4.2 Granularity Options

Three granularity levels are possible:

**Option A: Per-Plugin (current `bundle` default)**

- User installs plugin, gets everything.
- Simple. No selection UI. No state tracking of partial installs.
- Works for tightly-coupled plugins where components depend on each other.
- Loses: user cannot skip components they do not want.

**Option B: Per-Component-Type (e.g., "all skills" or "all agents")**

- User selects which component types to wire: "I want skills and agents from this plugin, but not hooks."
- Medium complexity. 6 type-level checkboxes.
- Fails for: "I want skill-1 and skill-3 but not skill-2."

**Option C: Per-Component (individual skill-1, agent-2, hook-1)**

- User selects individual components across all types.
- Maximum flexibility. Handles all use cases.
- Highest complexity: grouped multiselect UI, dependency tracking, state persistence.
- This is the user's proposal.

### 4.3 Component Type Safety Analysis

Not all component types are equally safe to cherry-pick:

| Component Type | Cherry-Pick Safety | Reasoning |
|---|---|---|
| **Skills** | [PASS] Safe | Skills are self-contained instruction documents. Skipping one does not break others. Vercel Skills validates this model with 8,800+ GitHub stars. |
| **Agents** | [PASS] Safe | Agents are persona definitions. Independent by nature. |
| **Commands** | [PASS] Safe | Commands are user-invoked operations. Independent. |
| **Instructions** | [WARNING] Conditional | Instructions modify platform instruction files (AGENTS.md, CLAUDE.md). A skill may assume certain instructions are present. Skipping instructions could silently degrade skill behavior. |
| **Hooks** | [WARNING] Risky | Hooks affect platform behavior globally. ADR-003 uses overlay/recompute merge. Skipping a hook that a skill expects (e.g., pre-commit validation) creates a silent gap. Plugin authors may assume all hooks are present. |
| **MCP Servers** | [FAIL] Unsafe to skip | MCP servers provide tools that skills and agents consume. Skipping the MCP server while installing a skill that calls its tools causes runtime errors. This is the exact gap ADR-001 Future Considerations identified. |

### 4.4 The Dependency Problem

Per-component cherry-picking introduces an intra-plugin dependency graph:

```text
skill-1 ──requires──> mcpServers (tools X, Y)
skill-2 ──requires──> mcpServers (tool Z)
skill-3 ──standalone──> (no deps)
agent-1 ──requires──> instruction-1
hook-1 ──standalone──> (no deps)
```

If user selects skill-1 but deselects mcpServers, skill-1 breaks at runtime. Three approaches to handle this:

**Approach 1: Ignore dependencies (Vercel model)**
Let users select anything. No validation. If something breaks, user figures it out. Simple to implement. Poor user experience for complex plugins.

**Approach 2: Declare and enforce dependencies (npm model)**
Add `requires` field to component frontmatter. During selection UI, auto-include required components and gray them out. Prevents broken installs. Requires plugin authors to declare dependencies correctly.

**Approach 3: Always-include certain types**
MCP servers and instructions are always wired if any component from the plugin is selected. Only skills, agents, commands, and hooks are individually selectable. No dependency declaration needed. Simpler than approach 2 but less flexible.

### 4.5 Interaction with ADR-013 npm Model

ADR-013 established:

1. Plugins are npm packages. `bun add` downloads everything.
2. `agent-plugin install` scans `node_modules` and wires components to platform configs.
3. Wiring state is derived from `node_modules` + platform config files. No separate state store.

Per-component cherry-picking impacts this model:

**Problem**: ADR-013 Decision 4 derives wired state by scanning platform configs for namespaced entries. If user cherry-picks, `agent-plugin install` needs to know which components were selected to avoid re-wiring deselected components on next run.

**Solution**: Selection state must be persisted. Two options:

| Storage | Pros | Cons |
|---|---|---|
| **In platform config files** (via comments/markers) | No new state file. State lives where wiring lives. | Fragile. Users edit config files. Comments get stripped. |
| **In a selection manifest** (`.agent-plugin/selections.json`) | Explicit. Survives config file edits. Diffable. VCS-friendly. | New state file. Contradicts ADR-013 "no separate state store" principle. |

The selection manifest approach is more robust. The file would be small:

```json
{
  "@scope/my-plugin": {
    "include": ["skills/review", "skills/lint", "agents/reviewer.md"],
    "exclude": ["skills/debug", "hooks/pre-commit.js"]
  }
}
```

This creates a minimal exception to ADR-013's stateless principle. The selection manifest is not a lockfile. It records user intent, not installed state.

### 4.6 UX Flow Design

Based on the Arch Linux pacman pattern (the only ecosystem with true interactive cherry-picking), the recommended UX is a **grouped multiselect**:

```text
$ agent-plugin install

Found @scope/code-tools (5 skills, 3 agents, 2 hooks, 1 MCP server)

Select components to install:
  Skills
    [x] review          Review TypeScript code for patterns
    [x] lint            ESLint integration for agent workflows
    [ ] debug           Debug assistance (requires MCP server)
    [x] test-gen        Generate unit test scaffolds
    [ ] refactor        Automated refactoring suggestions

  Agents
    [ ] reviewer        Code review persona
    [x] tester          Test engineering persona
    [ ] debugger        Debug investigation persona

  Hooks
    [x] pre-commit      Run lint before commit
    [ ] post-install    Configure workspace after install

  MCP Servers (required by: review, lint, debug)
    [x] code-tools-mcp  Code analysis tools [auto-selected]

  Instructions (always included)
    [x] coding-standards  TypeScript coding standards
```

Design decisions in this UX:

1. **Grouped by type** for scannability (not a flat list of 11+ items).
2. **MCP servers auto-selected** when any component that requires them is selected.
3. **Instructions always included** (convention: instructions set context that all components may assume).
4. **"required by" annotation** shows dependency information inline.
5. **All components checked by default** (opt-out, not opt-in). Most users want everything. Cherry-picking is the exception.

For **CI mode** (`--ci`): use `--include` and `--exclude` flags:

```bash
agent-plugin install --include "skills/review,skills/lint" --exclude "hooks/*"
```

For **MCP mode**: return component list as structured data. Agent selects via tool parameters.

### 4.7 Proposed `installMode` Redesign

Replace the binary `bundle` | `collection` with a single-field approach. Remove `installMode` entirely.

**Rationale**: The distinction between `bundle` and `collection` was designed for a pre-ADR-013 world where `collection` meant choosing between whole plugins, not components within a plugin. With npm distribution, the plugin is always a single npm package. The question is always "which components from this plugin do you want wired?"

**Proposed behavior**:

| Scenario | Behavior |
|---|---|
| Plugin has 1 component per type | Wire everything automatically. No selection UI. |
| Plugin has multiple components in any type | Present grouped multiselect. All checked by default. |
| User passes `--all` flag | Wire everything. No selection UI. |
| User passes `--include` / `--exclude` | Apply filters. No interactive UI. |
| CI mode (`--ci`) without include/exclude | Wire everything (equivalent to `--all`). |

This eliminates `installMode` as a manifest field. The decision of whether to show a selection UI becomes runtime behavior based on component count, not a static author declaration.

**Counter-argument**: Plugin authors may want to FORCE all-or-nothing for tightly-coupled plugins. A skill that depends on a hook that depends on an MCP server should not be separable.

**Response**: Handle this via intra-component `requires` declarations, not `installMode`. If all components require each other, the selection UI auto-selects everything and grays out deselection. The author achieves all-or-nothing without a separate field.

### 4.8 Alternative: Keep `installMode` but Add a Third Value

If removing `installMode` is too aggressive, add a third value:

```json
{
  "installMode": "bundle" | "selectable" | "collection"
}
```

- `bundle`: all-or-nothing (no UI). Current behavior preserved.
- `selectable`: per-component cherry-picking with grouped multiselect. New.
- `collection`: deprecated or repurposed.

The `collection` value was originally intended for choosing between whole plugins. With ADR-013's npm model, this concept no longer applies. `collection` could be deprecated.

### 4.9 Impact of Per-Component Selection on `agent-plugin install` Wiring

The current `install` command (ADR-013 Decision 4) does scan-diff-reconcile:

1. Scan: walk `node_modules` for `plugin.json`.
2. Diff: compare discovered vs currently wired.
3. Reconcile: add/remove/update.

With cherry-picking, step 2 changes. The diff must account for:

- Components in `node_modules` that the user intentionally excluded.
- Components that were selected but are no longer in `node_modules` (plugin updated, component removed).
- New components added to a plugin in an update that the user has not yet selected.

The selection manifest (`.agent-plugin/selections.json`) resolves this. The diff becomes:

```text
wired_state = scan platform configs
desired_state = scan node_modules FILTERED BY selections.json
diff = desired_state - wired_state
```

On `bun update @scope/plugin`, if the updated plugin adds new components, `agent-plugin install` detects unknown components and re-prompts:

```text
@scope/code-tools updated: 2 new components found
  [ ] skills/new-skill    New skill added in v2.1.0
  [ ] agents/new-agent    New agent added in v2.1.0
```

## 5. Results

| Finding | Source | Confidence |
|---|---|---|
| No ecosystem provides cross-type per-component cherry-picking from a single package | VS Code, Homebrew, apt, Arch Linux, Vercel, TanStack analysis | High |
| Arch Linux pacman groups provide the closest UX precedent (numbered selection at install) | Arch Wiki | High |
| MCP servers are unsafe to skip when skills depend on their tools | ADR-001 Future Considerations, logical analysis | High |
| Instructions are conditionally safe to skip (may degrade skill behavior silently) | Logical analysis of instruction/skill relationship | Medium |
| Skills, agents, and commands are safe to cherry-pick independently | Vercel Skills validates skill-level independence | High |
| Selection state requires persistence outside platform config files | ADR-013 stateless model analysis | High |
| `installMode: "collection"` no longer serves its original purpose under ADR-013 | ADR-001 vs ADR-013 comparison | High |
| Grouped multiselect with opt-out pattern is the preferred UX | Arch Linux precedent + @clack/prompts capability | Medium |

### Facts (Verified)

- Vercel Skills supports per-skill selection via `--skill` flag and interactive multiselect. It does NOT handle agents, hooks, or other types.
- VS Code extension packs are all-or-nothing at install time. Individual extensions can be disabled/uninstalled after pack install.
- Arch Linux `pacman -S group` presents a numbered list and supports range selection (`1-5`), exclusion (`^2`), and individual picks (`1 3 5`).
- Homebrew Bundle uses env vars (`$HOMEBREW_BUNDLE_BREW_SKIP`) for exclusion, not interactive selection.
- apt meta-packages install all dependencies; individual packages can be removed afterward without removing the meta-package.
- ADR-013 declared `agent-plugin install` stateless: no separate state file, truth derived from `node_modules` + platform configs.
- ADR-001 DEBATE flagged `installMode` binary as insufficient (Critic P1-4, Independent Thinker #4, 2 of 6 reviewers).

### Hypotheses (Unverified)

- Most plugin consumers (estimated 80%+) will want all components from a plugin. Cherry-picking is the minority use case. No user research data to confirm.
- Plugin authors rarely create tightly-coupled cross-type dependencies (skill requiring specific hook). No ecosystem data on coupling frequency.
- A selection manifest file will not create confusion alongside `bun.lockb` and platform config files. Needs user testing.

## 6. Discussion

### 6.1 The Core Tension

The proposal sits at the intersection of two competing values:

1. **Simplicity**: plugins are bundles. Install everything. Authors test everything together. No partial state to debug.
2. **Control**: users want only what they need. A plugin with 5 skills has 2 relevant ones. Installing 3 irrelevant skills adds noise to their agent context.

The second point matters more for AI agent plugins than for traditional software. Irrelevant skills in an agent's context consume context window tokens and may cause the agent to invoke the wrong skill. This is a stronger argument for cherry-picking than exists in any of the analyzed ecosystems.

### 6.2 The ADR-013 Alignment

ADR-013 made `agent-plugin install` a stateless scan-diff-reconcile loop. Per-component selection introduces state (the selection manifest). This is a material change to ADR-013's design philosophy.

However, the state is minimal and has a clear analog: `.gitignore` files tell git which files to exclude from tracking. `.agent-plugin/selections.json` tells `agent-plugin install` which components to exclude from wiring. The metaphor is consistent.

### 6.3 Recommended Path

Replace the `installMode` field with runtime behavior + optional `requires` declarations:

1. **Drop `installMode` from `plugin.json`**. The field has no ecosystem precedent (ANALYSIS-041 confirmed), was flagged by 2 of 6 reviewers as insufficient, and its `collection` value no longer applies under ADR-013.

2. **Add optional `requires` field to component frontmatter**. Components declare which other components they need:

```yaml
---
name: review
description: Review TypeScript code
type: skill
requires:
  - mcpServers/code-tools-mcp
---
```

3. **`agent-plugin install` presents grouped multiselect** when a plugin has multiple components. All checked by default. MCP servers and instructions checked and locked when required by a selected component.

4. **Selection state persisted in `.agent-plugin/selections.json`**. Checked into VCS. Minimal file. Amends ADR-013 to add this one piece of state.

5. **`--all` flag skips selection UI**. For users and CI who want everything.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Remove `installMode` field from `plugin.json` | No ecosystem precedent. `collection` value is dead under ADR-013. Binary was already flagged as insufficient by 2/6 debate reviewers. | Low (schema change) |
| P0 | Add optional `requires` field to component frontmatter | Enables intra-plugin dependency declaration. Prevents broken installs when cherry-picking. Solves the ADR-001 "skills + mandatory MCP" gap. | Medium (frontmatter schema + validation) |
| P1 | Implement grouped multiselect UI in `agent-plugin install` | @clack/prompts already adopted. Grouped by component type. All checked by default (opt-out). Arch Linux pacman validates the pattern. | Medium (UI implementation) |
| P1 | Create `.agent-plugin/selections.json` for selection persistence | Needed for idempotent `agent-plugin install` with cherry-picking. Amends ADR-013 minimally. Check into VCS. | Low (JSON read/write) |
| P1 | Auto-select MCP servers when required by any selected component | MCP servers are unsafe to skip. `requires` field drives this. Gray out deselection in UI. | Low (dependency resolution logic) |
| P2 | Always-include instructions by default | Instructions set context assumptions for skills. Users can explicitly deselect but should be warned. | Low (default behavior) |
| P2 | Add `--include` / `--exclude` flags for CI mode | Non-interactive cherry-picking for CI pipelines and scripting. | Low (flag parsing) |
| P2 | Re-prompt on plugin update when new components are detected | When `bun update` adds new components, `agent-plugin install` should ask about them rather than silently ignoring or auto-including. | Medium (update detection logic) |

## 8. Conclusion

**Verdict**: Proceed with per-component cherry-picking model. Remove `installMode`. Add `requires` for dependency safety.

**Confidence**: High

**Rationale**: The `installMode` binary was already identified as insufficient by ADR-001 debate reviewers. The `collection` value lost its meaning under ADR-013's npm model. Per-component selection is more important for AI agent plugins than traditional software because irrelevant components consume context window tokens and degrade agent behavior. The `requires` field and always-include conventions for MCP/instructions solve the safety gap.

### Adoption Status (2026-03-09)

ADR-014 adopted the core recommendations from this analysis:

- `installMode` replaced by a features model with per-component granularity (ADR-014 D5)
- `requires` field added for intra-plugin dependency declaration (ADR-014 D5)
- Grouped multiselect UI retained as the interactive selection mechanism (ADR-014 D7 Step 5)
- Selection state persisted in `.agent-lock.json` lockfile (ADR-014 D2), not `.agent-plugin/selections.json`
- Three selection mechanisms defined: section-based (markdown headers), code regions, file-based (ADR-014 D5)
- Mandatory PoC validation gate added: build markdown section extraction PoC before implementing other mechanisms

The ADR-013 npm/wiring model referenced throughout this analysis was superseded by ADR-014's explicit `agent-plugin add/remove/update` model on the same day ADR-013 was accepted.

### User Impact

- **What changes for you**: `installMode` field removed from `plugin.json`. On `agent-plugin add`, you see a features wizard with per-component selection. All features enabled by default. MCP servers auto-selected when needed. Selections saved in `.agent-lock.json` (commit to your repo).
- **Effort required**: Schema change to `plugin.json` (P0). Features wizard UI (P1). Lockfile (P1). Total estimated: 3-5 days implementation.
- **Risk if ignored**: Users install plugins with 10+ skills when they need 2, polluting their agent context window with 8 irrelevant instruction documents. No way to express "install skill X but MCP is required" without `requires` field.

## 9. Appendices

### Cross-Ecosystem Sources

- [VS Code Extension Packs](https://code.visualstudio.com/blogs/2017/03/07/extension-pack-roundup)
- [Homebrew Bundle](https://docs.brew.sh/Brew-Bundle-and-Brewfile)
- [apt Meta-Packages](https://help.ubuntu.com/community/MetaPackages)
- [Arch Linux pacman groups](https://wiki.archlinux.org/title/Meta_package_and_package_group)
- [Vercel Skills CLI](https://github.com/vercel-labs/skills)
- [TanStack Intent](https://tanstack.com/intent/latest)

### Data Transparency

- **Found**: Cross-ecosystem cherry-picking patterns across 6 systems. ADR-001 debate consensus on `installMode` insufficiency. ADR-013 stateless model constraints. ANALYSIS-041 confirmation that `installMode` has no ecosystem equivalent.
- **Not Found**: User research data on cherry-picking frequency. Real-world data on intra-plugin component coupling patterns. Performance impact of irrelevant skills on agent context windows (claimed but not measured).

## Observations

- [decision] Recommend removing `installMode` field from plugin.json entirely: no ecosystem precedent, `collection` value dead under ADR-013, binary flagged insufficient by 2/6 debate reviewers #installMode #removal
- [decision] Recommend adding optional `requires` field to component frontmatter for intra-plugin dependency declaration #requires #dependencies
- [decision] Recommend grouped multiselect UI (by component type) with opt-out pattern (all checked by default) following Arch Linux pacman precedent #UX #cherry-picking
- [decision] Recommend `.agent-plugin/selections.json` for persisting user component selections (minimal amendment to ADR-013 stateless principle) #state #selections
- [decision] MCP servers must be auto-selected and non-deselectable when any component declares them in `requires` #mcpServers #safety
- [decision] Instructions should be always-included by default with explicit deselection warning #instructions #defaults
- [fact] No ecosystem provides cross-type per-component cherry-picking from a single package; this is novel design #precedent
- [fact] Arch Linux pacman groups is the closest UX precedent for interactive numbered/range selection at install time #precedent #pacman
- [fact] Irrelevant skills in agent context consume context window tokens, making cherry-picking more important for AI plugins than traditional software #motivation
- [insight] ADR-001 DEBATE already flagged installMode binary as insufficient (Critic P1-4, Independent Thinker challenge #4) #validation
- [insight] ANALYSIS-041 confirmed installMode has zero equivalents across 9 analyzed AI platforms #validation
- [risk] Selection manifest creates state in a system ADR-013 designed to be stateless; this is a philosophical trade-off #stateless #tradeoff
- [risk] Intra-plugin dependency declarations (requires field) add authoring burden and may not be declared correctly by plugin authors #authoring #risk

## Relations

- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-013 npm-Package Distribution and Revised Command Tree]]
- relates_to [[ADR-003 Conflict Resolution and Namespacing]]
- relates_to [[ANALYSIS-041 Plugin Manifest Cross-Platform Comparison]]
- relates_to [[ANALYSIS-003 Vercel Skills Format Deep Dive]]
- relates_to [[ANALYSIS-034 Skill Versioning Models Comparison]]
- relates_to [[DEBATE-ADR-001 Plugin Format and Manifest]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]
- leads_to [[ADR-014 Explicit Installation Model and Content Features]]