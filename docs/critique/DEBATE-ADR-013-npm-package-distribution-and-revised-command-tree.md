---
title: DEBATE-ADR-013 npm-Package Distribution and Revised Command Tree
type: critique
tags:
- adr-review
- debate
- npm-distribution
- commands
- simplification
- ADR-013
- agent-plugin
permalink: critique/debate-adr-013-npm-package-distribution-and-revised-command-tree
---

# DEBATE-ADR-013 npm-Package Distribution and Revised Command Tree

## ADR Under Review

ADR-013 npm-Package Distribution and Revised Command Tree

## Round 1 Results

**Outcome**: 3 P0 issues, 11 P1 issues. No agent accepted as-is.

Round 1: 0 Accept, 0 D&C, 6 Needs Revision, 0 Block

### Agent Verdicts

| Agent | Verdict | P0 | P1 | P2 |
|-------|---------|----|----|-----|
| Architect | NEEDS REV | 1 | 4 | 1 |
| Critic | NEEDS REV | 1 | 4 | 2 |
| Independent Thinker | NEEDS REV | 1 | 3 | 1 |
| Security | NEEDS REV | 1 | 2 | 3 |
| Analyst | NEEDS REV | 1 | 3 | 1 |
| Advisor | NEEDS REV | 0 | 2 | 1 |

## P0 Issues (Consolidated)

### P0-1: postinstall Does NOT Fire on `bun remove` (Critic, Independent-Thinker, Analyst)

- ADR Decision 3 claims: "Bun triggers postinstall for all dependency changes"
- Review agents extrapolated from npm/yarn behavior and concluded this was incorrect
- npm v7+ dropped uninstall lifecycle scripts; Yarn does not fire postinstall on remove

**[RESOLVED] Empirical testing disproved the concern.**

Bun (v1.3.8) was tested on 2026-03-08. Results:

| Operation | postinstall Fires? | Verified |
|-----------|-------------------|----------|
| `bun add` | YES | Confirmed |
| `bun remove` | YES | Confirmed |
| `bun update` | YES | Confirmed |
| `bun install` (no args) | YES | Confirmed |

Bun runs the project's own lifecycle scripts on ALL package manager operations, unlike npm/yarn which only run them on install. The `--ignore-scripts` flag on `bun remove` (documented at bun.sh/docs/pm/cli/remove) confirms this: it exists to suppress the default behavior of running scripts.

**ADR-013 Decision 3's claim is correct for Bun.** The review agents' concern was based on npm/yarn behavior that does not apply to Bun. No ADR change needed for P0-1.

### P0-2: Hook Merge Without Lockfile (Architect, Critic)

- ADR-003 Decision 2 defines overlay/recompute pattern for hook merging
- That pattern stores per-plugin hook contributions in `plugin-lock.json`
- ADR-013 Decision 6 eliminates `plugin-lock.json`
- Without per-plugin hook tracking, uninstalling a plugin cannot cleanly remove its hook contributions from merged platform configs
- Stateless wiring (Decision 4) works for 1:1 component mappings but hooks use overlay/recompute which requires knowing each plugin's individual contributions

**[RESOLVED] Full recompute from node_modules eliminates need for lockfile state.**

ANALYSIS-035 evaluated 6 hook merge strategies. Full recompute (Strategy 1) selected:

- Each plugin's `plugin.json` in `node_modules` declares its hook contributions
- On `agent-plugin install`, scan all plugin.json files, compute merged hook state from scratch, write result
- On remove (plugin absent from `node_modules`), its hooks are naturally absent from the recomputed merge
- **node_modules IS the state** -- no per-plugin tracking file needed
- Performance: under 20ms for 10 plugins (top-level dir scan only, no deep traversal)
- Requires `agent-plugin install` runs after removal (connects to P0-1: postinstall fires on `bun remove`)

### P0-3: Supply Chain -- Unrestricted Scanning + Auto MCP Wiring (Security)

- `agent-plugin install` scans ALL packages in `node_modules` for `plugin.json`
- Any npm package (including transitive dependencies) can include a `plugin.json`
- MCP server entries contain `command` fields specifying executables to run
- Automatic wiring via postinstall silently grants command execution privileges
- CWE-829 (Inclusion of Functionality from Untrusted Control Sphere) + CWE-94 (Code Injection)
- Risk Score: 8/10
- Old model (ADR-010 Phase 4: CONFIRM) had explicit user confirmation step; ADR-013 removes it
**Resolution needed: Allowlist mechanism restricting which packages are eligible for wiring (e.g., only direct dependencies)**

## P1 Issues (Consolidated, Deduplicated)

1. **ANALYSIS-034 recommendation contradiction** (Advisor, Independent-Thinker, Analyst, Critic) -- ANALYSIS-034 explicitly recommended AGAINST TanStack model. ADR cites it as supporting evidence while rejecting its verdict. Must document why recommendation was overridden.
2. **Husky uses `prepare`, not `postinstall`** (Independent-Thinker, Analyst) -- Modern Husky v5+ uses `prepare` hook. `postinstall` runs when project is installed as a dependency; `prepare` only runs in local dev context. ADR misidentifies the hook Husky uses.
3. **Missing MADR Confirmation section** (Architect) -- Required by MADR 4.0 template. Every other ADR in this project includes one.
4. **Missing Pros/Cons for rejected options** (Architect, Critic) -- Options A (status quo) and C (hybrid) lack structured analysis. Required by MADR 4.0.
5. **Dev workflow gap** (Architect) -- No file-watching replacement for plugin authors. `bun add file:../my-plugin` installs once but does not watch for changes. Authors must manually re-run `agent-plugin install` after every edit.
6. **"60% complexity reduction" unsourced** (Critic, Analyst) -- Figure not present in ANALYSIS-034. Rhetorical estimate, not engineering measurement.
7. **plugin.json `name` vs package.json `name` dual-source** (Critic) -- Same dual-maintenance problem that removing `version` solved. Must define precedence or matching requirement.
8. **Path validation for plugin.json content** (Security) -- Content arrays contain file paths. No validation against `../` traversal, absolute paths, or symlinks outside package directory. CWE-22.
9. **Wiring change output/diff visibility** (Security) -- postinstall output may be suppressed by package manager. User has no visibility into what was wired. Need at minimum a summary output.
10. **`bun update` postinstall behavior unverified** (Independent-Thinker, Analyst) -- Bun docs do not explicitly confirm `bun update` triggers project's own `postinstall`.
11. **Isolated linker compatibility** (Analyst) -- Bun workspace projects default to isolated linker (pnpm-style with `.bun/` store and symlinks). Scanning logic must follow symlinks or enumerate from `package.json` dependencies.

## P2 Issues

1. **Collection installMode interaction** (Architect, Critic) -- No invocation point for collection-mode chooser UI after consumer commands removed.
2. **Audit trail loss** (Advisor, Independent-Thinker) -- NEG-005 acknowledged but unmitigated. No way to answer "what was wired at time T?"
3. **`list` command semantics unclear** (Architect, Critic) -- Does it show discovered (node_modules), wired (platform configs), or both?
4. **`prompts/` component type missing** (Critic) -- ADR-001 has 5 types including prompts; plugin.json example omits it.
5. **Performance of node_modules scanning** (Analyst) -- 500+ dependency projects require full directory traversal on every postinstall. No short-circuit optimization specified.
6. **Smart merge edge cases underspecified** (Independent-Thinker, Analyst) -- Shell parsing for existing postinstall scripts with `||`, `;`, env vars, `sh -c` wrappers.
7. **Global scope mechanics incomplete** (Architect) -- Where globally installed packages live in bun, how `init` handles global scope.

## Agent Conflict: Scope Split

| Agent | Position |
|-------|----------|
| Architect | Consider splitting D2 and D5 |
| Critic | Split into 2-3 ADRs (strong) |
| Independent Thinker | D2 and D3 could be separate |
| **High-Level-Advisor** | **DO NOT SPLIT (tie-breaker)** |
| Analyst | No strong opinion |
| Security | No split needed |

**Resolution**: High-level-advisor ruling (tie-breaker authority): Keep bundled. The 6 decisions are causally linked. D1 necessitates D2 (no consumer commands), D2 necessitates D3 (postinstall hooks), D1 necessitates D6 (no custom lockfile), D5 follows from D1, D4 implements D1. Single enforcement criterion covers all six: "Does this code do package management? Delete it."

## Proposed P0 Resolutions

### P0-1 Resolution: Acknowledge `bun remove` Gap

- Correct Decision 3 to state that postinstall fires on `bun install`, `bun add`, and `bun update` but NOT on `bun remove`
- Document that `agent-plugin install` must be run manually after `bun remove` to reconcile stale wiring
- Note that stale wiring self-heals on the next `bun install`/`bun add` operation
- Consider `prepare` hook instead of `postinstall` per Husky v5+ pattern (resolves P1-2 simultaneously)

### P0-2 Resolution: Hook Contributions via Runtime Recomputation

- Each plugin's `plugin.json` in `node_modules` declares its hook contributions
- On `agent-plugin install`, scan all plugin.json files, compute the merged hook state from scratch, and write it
- On remove (when plugin is absent from `node_modules`), the missing plugin's hooks are absent from the recomputed merge
- No per-plugin state file needed -- the plugin.json files in node_modules ARE the state
- Requires that `agent-plugin install` runs after removal (connects to P0-1)

### P0-3 Resolution: Direct Dependencies Only + Wiring Summary -- AGREED

- Restrict scanning to packages listed in `package.json` `dependencies` and `devDependencies` (not transitive)
- This matches TanStack Intent's actual behavior
- Always output a human-readable summary of wiring changes (new MCP servers, removed entries, etc.)
- Add path validation for all plugin.json content declarations (resolves P1-8)
- **Agreed with user on 2026-03-08. Not yet applied to ADR-013.**

## Round 2 Results

**Outcome**: Consensus reached. 5 Accept, 1 Disagree-and-Commit. 0 P0, 0 P1 issues.

Round 2: 5 Accept, 1 D&C, 0 Needs Revision, 0 Block

### Agent Verdicts

| Agent | Verdict | P0 | P1 | P2 |
|-------|---------|----|----|-----|
| Architect | ACCEPT | 0 | 0 | 1 |
| Security | ACCEPT | 0 | 0 | 1 |
| Advisor | ACCEPT | 0 | 0 | 1 |
| Analyst | ACCEPT | 0 | 0 | 1 |
| Critic | ACCEPT | 0 | 0 | 3 |
| Independent Thinker | D&C | 0 | 0 | 1 |

### Consolidated P2 Issues (Non-Blocking)

1. **installMode in example** (Architect, Critic) -- `installMode: "bundle"` in plugin.json example (Decision 5) is undocumented. Remove or define.
2. **ANALYSIS-035 reference label** (Critic) -- REF-006 labels it "Full Recompute Performance Benchmark" but actual title is "Hook Merging Strategies Without Custom Lockfile."
3. **ADR-010 supersession precision** (Critic) -- ADRs Affected table says "All 4 decisions" superseded while IMP-005 reuses platform detection logic from Decision 3 Phase 1.
4. **prompts/ content type** (Security) -- ADR-012 defines prompts/ but plugin.json content arrays in Decision 4 omit it.
5. **trustedDependencies naming** (Analyst) -- Bun field name should be verified against current docs at implementation time.
7. **Debate log stale observations** (Independent Thinker) -- Lines 130-136 and 162 contain pre-empirical-testing text that contradicts the resolved findings.

## D&C Reservations

### Independent Thinker

1. **Standalone content authors**: ANALYSIS-034 recommended against TanStack model. The override rationale (ANALYSIS-033's 12 conflicts) is documented and defensible, but the tightly-coupled versioning model breaks down when plugin content diverges from library code. For greenfield with library-maintainer-authored plugins, acceptable. If the ecosystem evolves toward standalone content authors (skills not tied to libraries), this ADR will need revisiting. Commits to executing Option B as specified.

2. **Bun postinstall stability**: Bun's postinstall-on-remove behavior is non-standard relative to npm/yarn. Empirical verification on v1.3.8 is sufficient today, but this behavior has no specification guaranteeing stability. If Bun aligns with npm semantics in a future release, automatic unwiring on `bun remove` breaks. Graceful degradation (manual `agent-plugin install`) is adequate mitigation. Commits to the postinstall approach as specified.

## Observations

- [fact] Round 1 produced unanimous NEEDS_REVISION with 3 consolidated P0 issues across 6 agents #adr-review #adr-013
- [fact] postinstall does NOT fire on bun remove -- verified via Bun docs, npm lifecycle spec, Yarn issue #3628, Bun GitHub issue #4959 #bun #lifecycle
- [risk] Unrestricted node_modules scanning enables supply chain attacks via malicious plugin.json in transitive dependencies #security #supply-chain
- [risk] Hook overlay/recompute pattern from ADR-003 depends on eliminated lockfile for per-plugin tracking #hooks #state
- [insight] ANALYSIS-034 recommended against TanStack model but team adopted it after ANALYSIS-033 revealed 12 conflicts #decision-process
- [insight] Modern Husky v5+ uses prepare hook, not postinstall -- prepare only runs in local dev context #husky #lifecycle
- [decision] High-level-advisor tie-breaker: keep 6 decisions bundled, causally linked, single enforcement criterion #scope

## Relations

- reviews [[ADR-013 npm-Package Distribution and Revised Command Tree]]
- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-003 Conflict Resolution and Namespacing]]
- relates_to [[ADR-007 CLI Architecture and Interaction Model]]
- relates_to [[ADR-008 Source Resolution and Package Validation]]
- relates_to [[ADR-010 Installation Lifecycle]]
- relates_to [[ADR-012 Scaffolding and Content Management]]
- relates_to [[ANALYSIS-033 Consumer and Author Commands]]
- relates_to [[ANALYSIS-034 Skill Versioning Models Comparison]]