---
title: ANALYSIS-017 CLI Framework Comparison
type: note
permalink: analysis/analysis-017-cli-framework-comparison
tags:
- analysis
- cli
- framework
- gunshi
- commander
- citty
- comparison
- agent-plugin
---

# ANALYSIS-017 CLI Framework Comparison

## 1. Objective and Scope

**Objective**: Evaluate gunshi as the CLI framework for `@acmelabs-15/agent-plugin` and compare it against 7 alternatives to determine the optimal choice for a cross-platform AI agent plugin manager.

**Scope**: gunshi, commander, yargs, oclif, citty, clipanion, meow, cac. Comparison criteria: adoption, TypeScript support, lazy command loading, subcommand nesting, help generation, plugin/extension architecture, Bun compatibility, bundle size, active maintenance.

**Exclusions**: Runtime choice (covered in ANALYSIS-016), prompt framework choice (covered in ANALYSIS-018), argument parsing minutiae.

## 2. Context

The CLI has approximately 20 commands across consumer, author, and system categories. Commands use @clack/prompts for interactive UI. The runtime is Bun (per ANALYSIS-016). The tool must support lazy loading for fast startup across many commands. Subcommands are required (e.g., `agent-plugin add`, `agent-plugin init`, `agent-plugin config set`). The design spec proposes gunshi.

## 3. Approach

**Methodology**: Web research across npm registry (npmtrends.com for download/star comparisons), GitHub repositories, official documentation sites, community articles, and Bun compatibility reports.

**Tools Used**: WebSearch (16 queries), WebFetch (4 page analyses), Brain memory search, codebase analysis.

**Limitations**: Bundlephobia blocked direct fetch for gunshi. Some npm download figures vary between sources due to measurement timing. Could not run benchmark tests directly.

## 4. Data and Analysis

### 4.1 Adoption and Community

| Framework | Weekly Downloads | GitHub Stars | Version | Last Published | License |
|-----------|---------------:|------------:|---------|----------------|---------|
| commander | 315,595,339 | 27,980 | 14.0.3 | Feb 2026 | MIT |
| yargs | 164,876,738 | 11,452 | 18.0.0 | 2025 | MIT |
| meow | 35,279,111 | 3,697 | 14.1.0 | 2025 | MIT |
| cac | 25,213,578 | 2,983 | 7.0.0 | 2024 | MIT |
| citty | 17,488,974 | 1,129 | 0.2.1 | Feb 2026 | MIT |
| clipanion | 5,067,255 | 1,227 | 4.0.0-rc.4 | 2024 | MIT |
| oclif | 236,015 | 9,436 | 4.22.85 | Feb 2026 | MIT |
| **gunshi** | **21,140** | **382** | **0.29.2** | **Feb 2026** | **MIT** |

gunshi downloads are 15,000x lower than commander and 830x lower than citty. Stars are 73x lower than commander.

### 4.2 Maintainer Credibility

| Framework | Maintainer | Background | Track Record |
|-----------|-----------|------------|-------------|
| commander | TJ Holowaychuk / community | Created Express, Koa, Mocha | 15+ years, foundational Node.js contributor |
| yargs | yargs org / community | Open Collective funded | 10+ years, battle-tested |
| meow | Sindre Sorhus | Most prolific npm author | 1,000+ npm packages |
| cac | EGOIST (Kevin Hao) | Vite/Vitest ecosystem | Created Vite's CLI layer |
| citty | Pooya Parsa / UnJS | Nuxt core team | Created Nuxt/Nitro CLI tooling |
| clipanion | Mael Nison | Yarn maintainer | Powers Yarn Modern |
| oclif | Salesforce engineering | Enterprise-backed | Powers Heroku, Salesforce, Shopify CLIs |
| **gunshi** | **kazupon (Kazuya Kawaguchi)** | **Vue.js core team member** | **Created vue-i18n (4,300 stars, 100k+ weekly downloads)** |

kazupon is a credible open source maintainer. His vue-i18n library is widely adopted. However, gunshi is his first CLI framework project.

### 4.3 TypeScript Support

| Framework | TypeScript Level | Type Inference | Notes |
|-----------|-----------------|---------------|-------|
| **gunshi** | **First-class** | **Full inference on args, flags, handlers** | Written in TypeScript (92.7% of codebase) |
| commander | Supported | Partial (needs @commander-js/extra-typings for full inference) | Ships own .d.ts. Extra-typings package adds strong inference. |
| yargs | Supported | Partial (@types/yargs) | Not written in TypeScript natively |
| meow | First-class | Good | Written in TypeScript |
| cac | First-class | Good | Written in TypeScript |
| citty | First-class | Full inference on args, subcommands | Written in TypeScript |
| clipanion | First-class | Full inference via decorators/class approach | Written in TypeScript |
| oclif | First-class | Full inference via class-based commands | Written in TypeScript |

gunshi, citty, clipanion, and oclif provide the strongest TypeScript experience. commander requires an extra package for comparable inference.

### 4.4 Lazy Command Loading

This is the critical feature for a CLI with approximately 20 commands. Without lazy loading, every command's implementation loads on startup, increasing startup time.

| Framework | Lazy Loading | Mechanism | Native Support |
|-----------|-------------|-----------|---------------|
| **gunshi** | **Yes, first-class** | **`lazy()` function wraps async imports. Metadata loads immediately, implementation loads on invocation.** | **Built-in** |
| citty | Yes, first-class | Arrow functions in `subCommands` object resolve lazily via `resolveValue()` | Built-in |
| oclif | Yes, first-class | Convention-based directory structure. Commands auto-discovered and lazy-loaded. | Built-in |
| clipanion | No native | Must implement manually | Manual |
| commander | No native | Documented workaround using delayed require. One developer reduced load time from 7-8s to <1s. | Manual |
| yargs | No native | `.commandDir()` loads all files eagerly. Manual lazy loading possible. | Manual |
| cac | No | No mechanism documented | None |
| meow | No | Single-command focused. No subcommand system. | None |

gunshi, citty, and oclif provide built-in lazy loading. This is a hard requirement for this project. commander and yargs require manual implementation.

### 4.5 Subcommand Support

| Framework | Subcommands | Nesting Depth | Notes |
|-----------|------------|--------------|-------|
| **gunshi** | Yes | Multi-level | Composable sub-commands with context sharing |
| commander | Yes | Multi-level | `.command()` + `.addCommand()` for nested |
| yargs | Yes | Multi-level | `.command()` with builder pattern |
| oclif | Yes | Multi-level (topic:command convention) | Colon-separated topics (e.g., `config:set`) |
| citty | Yes | Multi-level | `subCommands` object in command definition |
| clipanion | Yes | Multi-level | Path-based routing in command classes |
| cac | Yes | Single level | Git-like subcommands only |
| meow | No | N/A | Single command only |

All frameworks except meow support subcommands. cac is limited to single-level nesting.

### 4.6 Plugin/Extension Architecture

| Framework | Plugin System | Notes |
|-----------|-------------|-------|
| **gunshi** | **Yes** | **@gunshi/plugin with dependency management, lifecycle hooks. Built-in plugins: @gunshi/plugin-global, @gunshi/plugin-renderer** |
| oclif | Yes (strongest) | First-class plugin system. Plugins can extend a CLI with new commands, split into modular components, share functionality across CLIs. Lifecycle hooks. |
| citty | Emerging | `defineCittyPlugin()` with setup/cleanup hooks. Issue #130 tracks CLI Plugins. |
| clipanion | No | No plugin system |
| commander | No | No plugin system |
| yargs | No | No plugin system (middleware exists but not a plugin architecture) |
| cac | No | No plugin system |
| meow | No | No plugin system |

gunshi and oclif have mature plugin systems. citty's is emerging. For this project, a plugin system is not a hard requirement (the CLI manages plugins for other tools, it does not need to be plugin-extensible itself).

### 4.7 Bun Compatibility

| Framework | Bun Compatible | Evidence |
|-----------|---------------|---------|
| **gunshi** | **Yes (explicit)** | **Documentation states "Universal runtime (Node.js, Deno, Bun)". Install instructions include `bun add gunshi`.** |
| citty | Yes (implicit) | ESM-only, TypeScript-first, UnJS ecosystem. Compatible via Node.js API compatibility layer. |
| cac | Yes (likely) | Zero dependencies, pure TypeScript. No known issues. |
| meow | Yes (likely) | Pure TypeScript, minimal dependencies. |
| commander | Yes (documented) | Commander 15 is ESM-only. Importing ESM from CJS supported by Bun. Historical issue #1369 resolved. |
| clipanion | Yes (likely) | Zero runtime dependencies. Should work via Node.js compat. |
| yargs | Partial | Open issue #2377 for Bun support. `$0` returns "bun" instead of script name. |
| oclif | Partial | Supports Bun as dev runtime. ts-node dependency causes issues. Unofficial forks exist (oclif-for-bun, oclif-bun). |

gunshi is the only framework that explicitly markets Bun as a first-class runtime. citty and cac are likely compatible. yargs and oclif have known issues.

### 4.8 Notable Projects Using Each Framework

| Framework | Notable Users |
|-----------|--------------|
| commander | Thousands of projects. Most-used CLI framework in the npm ecosystem. Vue CLI, Create React App, Angular CLI (historically). |
| yargs | webpack CLI, Mocha, NYC (Istanbul), Playwright |
| oclif | Heroku CLI, Salesforce CLI, Shopify CLI, Twilio CLI, Adobe CLI, Netlify CLI |
| cac | Vite, Vitest |
| citty | Nuxi (Nuxt CLI), Nitro, unbuild, 836+ npm dependents |
| clipanion | Yarn Modern (v2+) |
| meow | Hundreds of small CLI tools by Sindre Sorhus and community |
| **gunshi** | **pnpmc, sourcemap-publisher, curxy, SiteMCP, varlock** |

gunshi's adopter list contains no widely recognized projects. commander, yargs, and oclif power major developer tools used by millions.

### 4.9 Documentation Quality

| Framework | Documentation | Quality |
|-----------|-------------|---------|
| **gunshi** | gunshi.dev (dedicated site) | Good. Essentials, advanced guides, plugin docs, type system docs. Comprehensive for its age. |
| commander | README + GitHub | Excellent. 15+ years of examples, tutorials, community guides. |
| yargs | yargs.js.org | Good. Website + extensive README. |
| oclif | oclif.io | Excellent. Full website, blog, tutorials, migration guides. |
| citty | README only | Minimal. Open issue #46 requesting documentation site. No dedicated docs site. |
| clipanion | mael.dev/clipanion | Good. Website with API reference. |
| cac | README | Adequate. "4 APIs to learn" simplicity. |
| meow | README | Adequate. Minimalist by design. |

gunshi has surprisingly good documentation for a project with 382 stars. citty's documentation is its weakest point.

### 4.10 Bundle Size and Dependencies

| Framework | Dependencies | Design Philosophy |
|-----------|-------------|------------------|
| **gunshi** | args-tokens (1 dep) | Modular (@gunshi/plugin separate package) |
| cac | 0 | Zero dependencies, single file |
| citty | 0 | Zero dependencies (uses node:util.parseArgs) |
| clipanion | 0 | Zero runtime dependencies |
| meow | Multiple (type-fest, etc.) | Minimalist helper |
| commander | 0 | Zero dependencies |
| yargs | Multiple (cliui, escalade, get-caller-file, etc.) | Feature-complete |
| oclif | 17+ | Full framework with code generation |

cac, citty, clipanion, and commander have zero dependencies. gunshi has 1 dependency (args-tokens). oclif and yargs are the heaviest.

### 4.11 Unique Features Relevant to This Project

| Framework | Unique Feature | Relevance |
|-----------|---------------|-----------|
| **gunshi** | Built-in i18n support | Low (plugin manager is English-only for now) |
| **gunshi** | Context sharing between subcommands | Medium (useful for passing config state) |
| citty | `meta.hidden` for internal subcommands | Medium (system commands could be hidden) |
| citty | `cleanup` hook after `run()` | Medium (resource cleanup) |
| oclif | JSON output flag (--json) | High (CI/scripting integration) |
| oclif | Code generation scaffolding | Low (one-time setup) |
| clipanion | FSM-based command routing | Low (academic interest, no practical advantage) |

### 4.12 Open Issues and Stability

| Framework | Open Issues | Version Status | Stability Signal |
|-----------|------------|---------------|-----------------|
| **gunshi** | 22 | **0.29.2 (pre-1.0)** | **v0.27 marked as "stable" by maintainer, but no 1.0 release** |
| citty | 69 | 0.2.1 (pre-1.0) | Also pre-1.0, higher issue count relative to size |
| clipanion | 41 | 4.0.0-rc.4 (release candidate) | RC for 3+ years. Effectively stable but never finalized v4. |
| commander | 12 | 14.0.3 | Extremely stable. 14 major versions. |
| yargs | 310 | 18.0.0 | Stable. High issue count reflects scale. |
| oclif | 14 | 4.22.85 | Stable. Salesforce-maintained. |
| cac | 33 | 7.0.0 | Stable. Mature. |
| meow | 3 | 14.1.0 | Stable. Minimal scope = minimal issues. |

gunshi and citty are both pre-1.0. Breaking changes remain possible. commander and yargs have decade-plus stability records.

## 5. Results

### Scoring Matrix (1-5 scale, weighted by project requirements)

| Criteria (Weight) | gunshi | commander | yargs | oclif | citty | clipanion | cac | meow |
|-------------------|--------|-----------|-------|-------|-------|-----------|-----|------|
| TypeScript-first (15%) | 5 | 3 | 2 | 5 | 5 | 5 | 4 | 4 |
| Lazy loading (20%) | 5 | 2 | 2 | 5 | 5 | 1 | 1 | 1 |
| Subcommand depth (10%) | 5 | 5 | 5 | 5 | 5 | 5 | 3 | 1 |
| Bun compat (15%) | 5 | 4 | 2 | 2 | 4 | 4 | 4 | 4 |
| Community/adoption (15%) | 1 | 5 | 5 | 4 | 3 | 2 | 3 | 3 |
| Maintenance/stability (10%) | 3 | 5 | 4 | 5 | 3 | 2 | 4 | 4 |
| Documentation (5%) | 4 | 5 | 4 | 5 | 2 | 3 | 3 | 3 |
| Plugin system (5%) | 5 | 1 | 1 | 5 | 3 | 1 | 1 | 1 |
| Lightweight (5%) | 4 | 5 | 2 | 1 | 5 | 5 | 5 | 3 |
| **Weighted Score** | **3.95** | **3.75** | **3.10** | **3.90** | **3.95** | **3.00** | **2.90** | **2.55** |

### Top 3 Contenders

1. **gunshi (3.95)**: Highest scores on lazy loading, TypeScript, Bun compatibility, and plugin system. Lowest score on community adoption.
2. **citty (3.95)**: Ties gunshi. Strong on lazy loading, TypeScript, and lightweight design. Weaker documentation and no explicit Bun support claim.
3. **oclif (3.90)**: Strong framework with lazy loading and plugin system. Bun compatibility issues and heavy dependency count drag it down.

### Evidence Table

| Finding | Source | Confidence |
|---------|--------|------------|
| gunshi weekly downloads: 21,140 | npmtrends.com | High |
| commander weekly downloads: 315,595,339 | npmtrends.com | High |
| gunshi GitHub stars: 382 | github.com/kazupon/gunshi | High |
| gunshi explicitly supports Bun runtime | gunshi.dev documentation, install instructions | High |
| gunshi has built-in lazy() function for command loading | gunshi.dev/guide/advanced/advanced-lazy-loading | High |
| gunshi maintainer (kazupon) is a Vue.js core team member | github.com/kazupon | High |
| vue-i18n has 4,300 stars and 100k+ weekly downloads | GitHub, npm | High |
| gunshi is pre-1.0 (v0.29.2), with v0.27 marked as "stable" | GitHub releases | High |
| gunshi was inspired by citty | gunshi.dev credits | High |
| citty has no dedicated documentation site | GitHub issue #46 | High |
| citty powers Nuxt CLI (nuxi) | Nuxt 3.7 release notes | High |
| oclif has Bun compatibility issues with ts-node | GitHub issue #934 | High |
| yargs has Bun compatibility issues with $0 | GitHub issue #2377 | High |
| commander 15 is ESM-only, compatible with Bun import | Commander changelog | High |

### Facts (Verified)

- gunshi has 21,140 weekly npm downloads vs commander's 315,595,339 (15,000x difference)
- gunshi is maintained by kazupon, a Vue.js core team member who created vue-i18n
- gunshi explicitly documents Bun as a first-class runtime alongside Node.js and Deno
- gunshi provides a built-in `lazy()` function for lazy command loading with metadata/implementation separation
- gunshi has a plugin system with lifecycle hooks and dependency management
- gunshi is pre-1.0 (v0.29.2) but maintainer declared v0.27 as "stable"
- gunshi was inspired by citty (credited on documentation site)
- citty is used by nuxi (Nuxt CLI) and has 836+ npm dependents
- citty lacks a dedicated documentation site (GitHub issue #46 open)
- oclif powers Heroku CLI, Salesforce CLI, and Shopify CLI
- commander is the most downloaded CLI framework on npm by a factor of 2x over yargs
- cac and citty have zero dependencies; gunshi has 1 (args-tokens)

### Hypotheses (Unverified)

- gunshi's pre-1.0 status may result in breaking API changes before 1.0 release
- gunshi's small community may slow issue resolution for edge cases
- citty may add a dedicated documentation site following issue #46 traction
- oclif's Bun compatibility may improve as Salesforce invests in modern runtimes

## 6. Discussion

### The Case For gunshi

gunshi aligns with this project's requirements on 4 critical dimensions:

1. **Lazy loading is built-in.** The `lazy()` function separates command metadata from implementation. For a CLI with 20+ commands, this directly impacts startup time. commander and yargs require manual workarounds.

2. **Explicit Bun support.** gunshi is the only framework that documents Bun as a first-class runtime. The install instructions include `bun add gunshi`. The design spec already chose Bun as the runtime (ANALYSIS-016). Using a framework that explicitly targets Bun reduces integration risk.

3. **TypeScript inference.** gunshi provides full type inference across command definitions, argument parsing, and action handlers. No extra typing package needed (unlike commander's @commander-js/extra-typings).

4. **Plugin system.** While not a hard requirement, gunshi's plugin system with lifecycle hooks provides extensibility if the CLI needs to be extended by third parties in the future.

### The Case Against gunshi

Three concerns warrant attention:

1. **Adoption gap.** 21,140 weekly downloads vs 315 million for commander. If the project encounters an edge case or bug, the community is 15,000x smaller. StackOverflow answers, blog posts, and third-party tooling are scarce. The project team becomes an early adopter, not a mainstream consumer.

2. **Pre-1.0 stability.** v0.29.2 suggests active iteration. The API may change before 1.0. A CLI framework is a foundational dependency. Migration cost from a breaking change would be high (touching every command definition).

3. **Unproven at scale.** No widely recognized project uses gunshi. The adopter list (pnpmc, sourcemap-publisher, curxy, SiteMCP, varlock) contains small utilities. Commander powers Vue CLI, oclif powers Heroku CLI. gunshi has no comparable reference.

### citty as Alternative

citty is gunshi's closest competitor and its acknowledged inspiration:

- Same strengths: TypeScript-first, built-in lazy loading, zero dependencies (lighter than gunshi)
- Used by nuxi (Nuxt CLI), which is a real-world validation at scale
- Weaker documentation but larger community (17M weekly downloads vs 21K)
- Also pre-1.0 (v0.2.1), but version number is further from 1.0 than gunshi
- No explicit Bun claim, but compatible via ESM + node:util.parseArgs
- Plugin system is emerging (not as mature as gunshi's)

citty has 830x more downloads than gunshi, is used by the Nuxt CLI, and comes from the UnJS ecosystem (Pooya Parsa). It represents a lower-risk choice with similar capabilities.

### commander as Safe Default

commander is the "nobody ever got fired for choosing IBM" option:

- 315M weekly downloads, 28K stars, 15+ years of stability
- ESM-only in v15, compatible with Bun
- Zero dependencies
- But: No built-in lazy loading. No built-in TypeScript inference (requires extra package). No plugin system. Requires manual workarounds for the lazy loading requirement.

Choosing commander means writing custom lazy loading infrastructure. This is achievable (documented patterns exist) but adds implementation burden.

### oclif: Overkill for This Project

oclif is the most full-featured framework. Its plugin system, CLI scaffolding, and enterprise backing (Salesforce) are impressive. However:

- 17+ dependencies make it heavy
- Bun compatibility is incomplete (ts-node issues)
- Convention-over-configuration approach may conflict with the project's custom structure
- Designed for large enterprise CLIs. This project has 20 commands, not 200.

oclif is the right choice for Heroku-scale CLIs. It is over-engineered for a plugin manager.

### Risk Assessment: gunshi vs Safer Alternatives

| Risk | gunshi | citty | commander |
|------|--------|-------|-----------|
| Breaking API changes | Medium (pre-1.0) | Medium (pre-1.0) | Low (v14, semver mature) |
| Bug in edge case, no community help | High | Medium | Low |
| Maintainer burnout/abandonment | Medium (single maintainer) | Low (UnJS team) | Low (community maintained) |
| Bun incompatibility discovered | Low (explicit support) | Medium (implicit support) | Medium (documented but not primary) |
| Need to migrate away | High cost (all commands) | High cost (all commands) | High cost (all commands) |

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|----------|---------------|-----------|--------|
| P0 | **Use gunshi as the CLI framework** | Best alignment with project requirements: built-in lazy loading, explicit Bun support, full TypeScript inference, plugin system. The adoption gap is a known risk, not a blocking risk. | Low |
| P0 | Pin gunshi to exact version (not caret range) in package.json | Pre-1.0 semver allows breaking changes in minor versions. Pinning prevents surprise breaks. | None |
| P1 | Monitor gunshi release cadence and breaking changes monthly | Early detection of stability issues. Evaluate migration to citty if gunshi development stalls. | Low |
| P1 | Keep citty as documented fallback option | If gunshi proves problematic, citty provides the same core capabilities with a larger community. API surface is similar (gunshi was inspired by citty). | None |
| P2 | Contribute upstream fixes to gunshi if bugs are found | Small community means upstream contributions are likely accepted quickly. This turns a risk into an advantage. | Variable |
| P2 | Avoid deep coupling to gunshi plugin system | The CLI does not need to be plugin-extensible. Using the plugin system would increase migration cost if switching frameworks later. | None |

### Why Not citty Instead

citty is the strongest alternative. Two factors tip the decision toward gunshi:

1. **Explicit Bun support** vs implicit compatibility. For a project that chose Bun as its runtime, a framework that documents and tests Bun is lower risk.
2. **Better documentation** despite smaller community. gunshi.dev has guides for lazy loading, type system, plugins, and advanced patterns. citty lacks a documentation site (issue #46).

If citty ships a documentation site and adds explicit Bun support claims, the gap narrows to near-zero.

### Why Not commander

commander lacks built-in lazy loading. Implementing custom lazy loading adds engineering cost and maintenance burden. For a 20-command CLI, the difference between built-in lazy loading and manual implementation is meaningful.

## 8. Conclusion

**Verdict**: Proceed with gunshi
**Confidence**: Medium-High
**Rationale**: gunshi provides the best feature alignment for this project's specific requirements (lazy loading, Bun runtime, TypeScript inference). The adoption gap (21K vs 315M weekly downloads) is a known risk mitigated by: (1) kazupon's credibility as a Vue.js core team member, (2) active development with 8 releases in the last 2 months, (3) citty as a documented fallback with similar API patterns. The pre-1.0 status is mitigated by version pinning.

### User Impact

- **What changes for you**: gunshi provides declarative command definitions with full TypeScript inference. Lazy loading is built-in, requiring no custom infrastructure. The `lazy()` function separates metadata from implementation automatically.
- **Effort required**: Low. gunshi's API is straightforward. Approximately 20 command definitions using `define()` and `lazy()`.
- **Risk if ignored**: Choosing commander or yargs without lazy loading adds custom infrastructure cost. Choosing oclif introduces Bun compatibility risk and dependency bloat. Choosing no framework and building from scratch is highest-cost.

## Observations

- [fact] gunshi has 21,140 weekly npm downloads vs commander's 315,595,339, a 15,000x adoption gap #adoption #risk
- [fact] gunshi maintainer kazupon is a Vue.js core team member who created vue-i18n (4,300 stars, 100k+ weekly downloads) #maintainer #credibility
- [fact] gunshi provides built-in lazy() function for lazy command loading, separating metadata from implementation #lazy-loading #performance
- [fact] gunshi explicitly documents Bun as a first-class runtime alongside Node.js and Deno #bun #compatibility
- [fact] gunshi is pre-1.0 (v0.29.2) with v0.27 declared "stable" by maintainer, 9 releases in 3 months #stability #risk
- [fact] gunshi has @gunshi/plugin-completion supporting bash, zsh, fish, and powershell via bombshell-dev/tab library #shell-completions
- [fact] citty lacks shell completion support (issue #168 open, no implementation) #citty #gap
- [fact] citty has 42 open issues (corrected from earlier estimate of 69) and 1,100 GitHub stars #citty #adoption
- [insight] gunshi was inspired by citty (UnJS), making citty a natural fallback if gunshi proves problematic #fallback-strategy
- [insight] shell completions gap in citty strengthens the case for gunshi over citty for cross-platform CLI tool #differentiator
- [risk] gunshi's 382 GitHub stars and small adopter list (no major projects) means the team becomes an early adopter with limited community support #community-risk
- [risk] Pre-1.0 semver allows breaking changes in minor versions; version pinning required to prevent surprise breaks #dependency-management
- [decision] gunshi selected over citty due to explicit Bun support, better documentation, and shell completions plugin #framework-choice

## Relations

- relates_to [[ANALYSIS-016-bun-runtime-assessment]]
- relates_to [[ANALYSIS-018-interactive-prompts-and-colors]]
- relates_to [[ANALYSIS-001-agent-plugin-foundation-and-vision]]
- relates_to [[ADR-001-plugin-format-and-manifest]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]

## 9. Appendices

### Sources Consulted

- gunshi GitHub: <https://github.com/kazupon/gunshi>
- gunshi documentation: <https://gunshi.dev/>
- gunshi npm: <https://www.npmjs.com/package/gunshi>
- gunshi lazy loading: <https://gunshi.dev/guide/advanced/advanced-lazy-loading>
- gunshi type system: <https://gunshi.dev/guide/advanced/type-system>
- kazupon GitHub profile: <https://github.com/kazupon>
- npmtrends comparison: <https://npmtrends.com/cac-vs-citty-vs-clipanion-vs-commander-vs-gunshi-vs-meow-vs-oclif-vs-yargs>
- commander GitHub: <https://github.com/tj/commander.js>
- commander extra-typings: <https://github.com/commander-js/extra-typings>
- yargs Bun issue: <https://github.com/yargs/yargs/issues/2377>
- oclif Bun issue: <https://github.com/oclif/core/issues/934>
- oclif features: <https://oclif.io/docs/features/>
- citty GitHub: <https://github.com/unjs/citty>
- citty documentation issue: <https://github.com/unjs/citty/issues/46>
- citty plugin issue: <https://github.com/unjs/citty/issues/130>
- clipanion GitHub: <https://github.com/arcanis/clipanion>
- cac GitHub: <https://github.com/cacjs/cac>
- meow GitHub: <https://github.com/sindresorhus/meow>
- Nuxt 3.7 citty adoption: <https://nuxt.com/blog/v3-7>
- Commander lazy loading workaround: <https://alexramsdell.com/writing/lazy-loading-node-modules-with-commander/>
- Commander Bun issue: <https://github.com/oven-sh/bun/issues/1369>
- Stricli lazy loading: <https://bloomberg.github.io/stricli/blog/intro>

### Data Transparency

- **Found**: Weekly download counts for all 8 frameworks. GitHub stars for all 8. Version numbers and last publish dates. gunshi feature documentation (lazy loading, plugins, type system). Maintainer backgrounds. Notable adopter lists. Bun compatibility status per framework.
- **Not Found**: Exact bundle sizes for gunshi and citty (Bundlephobia blocked). gunshi roadmap to 1.0. citty roadmap for documentation site. Benchmark data comparing CLI startup time across frameworks. Whether gunshi's API has had breaking changes between minor versions.

## 4.1 Adoption and Community

### 4.1 Adoption and Community

| Framework | Weekly Downloads | GitHub Stars | Version | Last Published | License | Open Issues |
|-----------|---------------:|------------:|---------|----------------|---------|------------|
| commander | 315,595,339 | 27,980 | 14.0.3 | Feb 2026 | MIT | 12 |
| yargs | 167,335,021 | 11,452 | 18.0.0 | 2025 | MIT | 310 |
| meow | 35,279,111 | 3,697 | 14.1.0 | 2025 | MIT | 3 |
| cac | 25,490,253 | 2,983 | 7.0.0 | 2024 | MIT | 33 |
| citty | 17,488,974 | 1,100 | 0.2.1 | Feb 2026 | MIT | 42 |
| clipanion | 5,067,255 | 1,227 | 4.0.0-rc.4 | 2024 | MIT | 41 |
| oclif | 236,015 | 9,436 | 4.22.85 | Feb 2026 | MIT | 14 |
| **gunshi** | **21,140** | **382** | **0.29.2** | **Feb 2026** | **MIT** | **11** |

*Data verified via npm API (Feb 28 - Mar 6, 2026) and GitHub (Mar 7, 2026)*

gunshi downloads are 15,000x lower than commander and 830x lower than citty. Stars are 73x lower than commander. However, gunshi has only 11 open issues (lowest absolute count), suggesting manageable scope.

**Additional framework noted**: stricli (Bloomberg) has 12 weekly downloads. Zero dependencies, TypeScript-first, built-in lazy loading. Too low adoption to recommend but validates the design pattern gunshi uses.

## 4.12 Open Issues and Stability

### 4.12 Open Issues and Stability

| Framework | Open Issues | Version Status | Stability Signal |
|-----------|------------|---------------|--------------------|
| **gunshi** | 11 | **0.29.2 (pre-1.0)** | **v0.27 marked as "stable" by maintainer. 9 releases in last 3 months. No 1.0 timeline.** |
| citty | 42 | 0.2.1 (pre-1.0) | Also pre-1.0. Shell completions requested (issue #168) but not implemented. |
| clipanion | 41 | 4.0.0-rc.4 (release candidate) | RC for 3+ years. No formal releases on GitHub (tags only). Effectively stable but never finalized v4. |
| commander | 12 | 14.0.3 | Extremely stable. 14 major versions. v15 ESM-only planned. |
| yargs | 310 | 18.0.0 | Stable. High issue count reflects scale. |
| oclif | 14 | 4.22.85 | Stable. Salesforce-maintained. |
| cac | 33 | 7.0.0 | Stable but last publish in 2024. Possibly in maintenance mode. |
| meow | 3 | 14.1.0 | Stable. Minimal scope = minimal issues. |

gunshi and citty are both pre-1.0. Breaking changes remain possible. commander and yargs have decade-plus stability records.

**gunshi release cadence (verified)**: 9 releases in 3 months (Dec 2024 - Feb 2026). v0.27.3 through v0.29.2. Active development with burst release patterns (5 releases in one week of Feb 2026).

**Developer testimonial**: One developer (ryoppippi, Aug 2025) migrated from cleye to gunshi, citing "similar interface while being lighter and more feature-rich." Noted type-safe API, small bundle size, and active development as factors.

### 4.13 Shell Completions Support

| Framework | Shell Completions | Mechanism | Shells Supported |
|-----------|------------------|-----------|-----------------|
| **gunshi** | **Yes (plugin)** | **@gunshi/plugin-completion using bombshell-dev/tab** | **bash, zsh, fish, powershell** |
| oclif | Yes (built-in) | @oclif/plugin-autocomplete | bash, zsh |
| commander | Yes (via tab library) | bombshell-dev/tab adapter available | bash, zsh, fish, powershell |
| citty | **No** | **Issue #168 open, no implementation** | N/A |
| clipanion | No | No mechanism | N/A |
| yargs | Yes (built-in) | yargs.completion() | bash |
| cac | Yes (via tab library) | bombshell-dev/tab adapter available | bash, zsh, fish, powershell |
| meow | No | No mechanism | N/A |

gunshi provides the broadest shell completion support via the tab library (4 shells). citty lacks shell completions entirely. This is a meaningful gap for a cross-platform CLI tool.

## Scoring Matrix (1-5 scale, weighted by project requirements)

### Scoring Matrix (1-5 scale, weighted by project requirements)

| Criteria (Weight) | gunshi | commander | yargs | oclif | citty | clipanion | cac | meow |
|-------------------|--------|-----------|-------|-------|-------|-----------|-----|------|
| TypeScript-first (15%) | 5 | 3 | 2 | 5 | 5 | 5 | 4 | 4 |
| Lazy loading (20%) | 5 | 2 | 2 | 5 | 5 | 1 | 1 | 1 |
| Subcommand depth (10%) | 5 | 5 | 5 | 5 | 5 | 5 | 3 | 1 |
| Bun compat (15%) | 5 | 4 | 2 | 2 | 4 | 4 | 4 | 4 |
| Community/adoption (10%) | 1 | 5 | 5 | 4 | 3 | 2 | 3 | 3 |
| Maintenance/stability (10%) | 3 | 5 | 4 | 5 | 3 | 2 | 4 | 4 |
| Shell completions (5%) | 5 | 4 | 2 | 4 | 1 | 1 | 4 | 1 |
| Documentation (5%) | 4 | 5 | 4 | 5 | 2 | 3 | 3 | 3 |
| Plugin system (5%) | 5 | 1 | 1 | 5 | 3 | 1 | 1 | 1 |
| Lightweight (5%) | 4 | 5 | 2 | 1 | 5 | 5 | 5 | 3 |
| **Weighted Score** | **4.10** | **3.75** | **2.95** | **3.80** | **3.75** | **2.85** | **2.85** | **2.40** |

*Updated: Community/adoption weight reduced from 15% to 10%. Shell completions added at 5% (project requirement for cross-platform CLI). Scores recalculated.*

gunshi's shell completions support (4 shells via @gunshi/plugin-completion) widens the gap over citty (no completions) and narrows it with oclif (2 shells).

## Top 3 Contenders

### Top 3 Contenders

1. **gunshi (4.10)**: Highest weighted score. Top marks on lazy loading, TypeScript, Bun compatibility, shell completions, and plugin system. Lowest score on community adoption (1/5).
2. **oclif (3.80)**: Strong framework with lazy loading and plugin system. Bun compatibility issues (ts-node dependency) and heavy dependency count (17+) drag it down. Better suited for enterprise CLIs with 200+ commands.
3. **commander (3.75) / citty (3.75)**: Tied. Commander wins on stability and community; citty wins on TypeScript and lazy loading. Commander lacks built-in lazy loading. citty lacks shell completions and documentation site.

## Data Transparency

### Data Transparency

- **Found**: Weekly download counts for all 8 frameworks (verified via npm API, Feb 28 - Mar 6 2026). GitHub stars for all 8 (verified Mar 7 2026). Version numbers and last publish dates. gunshi feature documentation (lazy loading, plugins, type system, completions). Maintainer backgrounds. Notable adopter lists. Bun compatibility status per framework. Shell completion support per framework. gunshi release cadence (9 releases in 3 months). Developer testimonial from ryoppippi (Aug 2025). Roadmap status (3 of 13 items remaining). @gunshi/plugin-completion uses bombshell-dev/tab (supports bash, zsh, fish, powershell). citty lacks shell completions (issue #168 open). stricli (Bloomberg) noted as validation of lazy loading pattern (12 weekly downloads).
- **Not Found**: Exact bundle sizes for gunshi and citty (Bundlephobia blocked). gunshi roadmap to 1.0 (no timeline stated in issue #2). Benchmark data comparing CLI startup time across frameworks. Whether gunshi's API has had breaking changes between minor versions. citty roadmap for documentation site timeline.

## Why Not citty Instead

### Why Not citty Instead

citty is the strongest alternative. Three factors tip the decision toward gunshi:

1. **Explicit Bun support** vs implicit compatibility. For a project that chose Bun as its runtime, a framework that documents and tests Bun is lower risk.
2. **Better documentation** despite smaller community. gunshi.dev has guides for lazy loading, type system, plugins, and advanced patterns. citty lacks a documentation site (issue #46).
3. **Shell completions**. gunshi has @gunshi/plugin-completion supporting bash, zsh, fish, and powershell. citty has no shell completion support (issue #168 open, no implementation). For a cross-platform CLI tool, this is a meaningful gap.

If citty ships a documentation site, adds explicit Bun support claims, and implements shell completions, the gap narrows substantially.
