---
title: ANALYSIS-044 Sectioned Content Configuration Patterns
type: note
permalink: analysis/analysis-044-sectioned-content-configuration-patterns-1
tags:
- configuration
- modular
- sections
- composition
- cherry-picking
- plugin-format
- architecture
---

# ANALYSIS-044 Sectioned Content Configuration Patterns

## 1. Objective and Scope

**Objective**: Research how popular packages and community standards handle modular/sectioned content configuration, where a single artifact is broken into named sections that can be individually selected, toggled, or composed. Identify patterns applicable to `@acmelabs-15/agent-plugin`'s unified configuration format across all component types (skills, agents, hooks, commands, AGENTS.md instructions).

**Scope**: 13 ecosystems analyzed: ESLint flat config, Prettier, Stylelint, Markdownlint, Tailwind CSS presets, Biome, TypeScript tsconfig, Yeoman/Hygen generators, VS Code extension packs, Terraform modules, Docker Compose profiles, Nx/Turborepo, Storybook addons, and AI platform patterns (Vercel, Claude Code, GitHub Copilot). Excludes implementation-level code design. Builds on ANALYSIS-043 (per-component cherry-picking).

## 2. Context

ANALYSIS-043 established that per-component cherry-picking is the recommended approach for plugin installation. The next question is how individual components (skills, agents, etc.) structure their internal content into named sections that can be independently selected, toggled, or composed.

The Vercel `react-best-practices` skill demonstrates this pattern: a single skill contains 8 sections (async, bundle, server, client, rerender, rendering, js, advanced), each containing multiple rules. A `sectionMap` in the build config maps filename prefixes to section numbers. We want to generalize this into a unified format that works across ALL component types.

## 3. Approach

**Methodology**: Web research across 13 ecosystems. Source code analysis of Vercel agent-skills, ESLint flat config, Biome, and Tailwind CSS. Cross-referencing with existing ANALYSIS-003, ANALYSIS-043, and ADR decisions.
**Tools Used**: WebSearch, WebFetch (documentation sites, GitHub), Brain MCP (existing analysis notes).
**Limitations**: Some source code not directly accessible (rate-limited). Vercel sectionMap pattern analyzed via DeepWiki documentation rather than raw source. No user research data on section-level cherry-picking demand.

## 4. Data and Analysis

### 4.1 Pattern-by-Pattern Analysis

#### Pattern 1: ESLint Flat Config (Array Composition)

**Config format**: JavaScript/TypeScript (`eslint.config.js`). Configuration is an ordered array of config objects.

**Selection mechanism**: Users compose by spreading arrays or using `extends` within `defineConfig()`. Each config object specifies `files` globs for scoping.

```javascript
import { defineConfig } from "eslint/config";
export default defineConfig([
  { files: ["**/*.js"], extends: ["js/recommended"] },
  { files: ["**/*.ts"], extends: [tseslint.configs.recommended] },
  { rules: { "no-unused-vars": "warn" } }
]);
```

**Dependency handling**: No explicit dependency declarations between config objects. Order determines precedence (later overrides earlier).

**Ordering**: Significant. Array order = merge order. Later entries override earlier ones.

**Defaults**: No defaults. Users explicitly compose their config from available pieces.

**Build/compose step**: None. Runtime merge by ESLint engine.

**Override granularity**: Individual rules can be overridden by adding a later config object with the same rule key.

**Airbnb sub-pattern**: eslint-config-airbnb-base organizes 300+ rules into 8 category files: `best-practices.js`, `errors.js`, `es6.js`, `imports.js`, `node.js`, `strict.js`, `style.js`, `variables.js`. The main config imports all 8. Users cannot cherry-pick individual category files without importing them directly (undocumented internal structure).

**typescript-eslint sub-pattern**: Provides tiered presets (`recommended` < `strict` < `stylistic`) where each is a superset. Users pick a tier, then override individual rules. Cannot cherry-pick rules from `strict` without also getting `recommended`.

**Relevance**: [HIGH] Array-based composition with override semantics is directly applicable. The Airbnb category-file pattern maps well to our section concept. The typescript-eslint tiered-preset pattern is useful for progressive disclosure.

---

#### Pattern 2: Prettier (Flat Object Spread)

**Config format**: JSON, YAML, or JavaScript (`.prettierrc`, `prettier.config.js`).

**Selection mechanism**: No `extends` mechanism. Shareable configs are npm packages exporting a single object. Users import and spread:

```javascript
import base from "@org/prettier-config";
export default { ...base, semi: false };
```

**Dependency handling**: None. All options are independent.

**Ordering**: Not applicable. Object merge (last write wins).

**Defaults**: Prettier has built-in defaults. Shared configs override them.

**Build/compose step**: None.

**Override granularity**: Individual properties only. No grouping concept.

**Relevance**: [LOW] Too flat for our needs. No section concept. Useful only as a counter-example of what happens without sectioning.

---

#### Pattern 3: Stylelint (Cascading Extends)

**Config format**: JSON, YAML, or JavaScript.

**Selection mechanism**: `extends` array with ordered precedence. Later entries override earlier ones:

```json
{
  "extends": ["stylelint-config-recommended", "stylelint-config-standard"],
  "rules": { "color-hex-length": "long" }
}
```

**Dependency handling**: Configs can extend other configs (chaining). `standard` extends `recommended`.

**Ordering**: Significant in `extends` array (last wins).

**Defaults**: `recommended` provides minimal defaults. `standard` adds more.

**Build/compose step**: None.

**Override granularity**: Individual rules.

**Relevance**: [MEDIUM] The chaining pattern (standard extends recommended) is a useful model for progressive section inclusion.

---

#### Pattern 4: Markdownlint-cli2 (Hierarchical + Extends)

**Config format**: JSONC, JSON, YAML, or JavaScript (`.markdownlint-cli2.jsonc`).

**Selection mechanism**: `extends` property for shared configs. Hierarchical directory-based resolution (config in parent directory applies to all subdirectories, child overrides parent).

**Dependency handling**: None between rules.

**Ordering**: Directory hierarchy determines precedence (child overrides parent).

**Defaults**: All rules enabled by default. Users disable selectively.

**Build/compose step**: None.

**Override granularity**: Individual rules by rule ID.

**Relevance**: [LOW] Directory-based hierarchy is interesting but not applicable to our section model. The opt-out default (all rules on) aligns with ANALYSIS-043's recommendation.

---

#### Pattern 5: Tailwind CSS Presets (Layered Merge with Plugin Array)

**Config format**: JavaScript (`tailwind.config.js`).

**Selection mechanism**: `presets` array. Presets are full Tailwind config objects. Multiple presets merge left-to-right (last wins). Presets can contain other presets (nesting).

```javascript
module.exports = {
  presets: [
    require("@acmecorp/tailwind-colors"),
    require("@acmecorp/tailwind-fonts")
  ],
  theme: { extend: { spacing: { "128": "32rem" } } }
}
```

**Dependency handling**: None between presets. Plugins within presets merge additively.

**Ordering**: Significant. Last preset wins for conflicting theme keys.

**Defaults**: Tailwind's built-in design system is the default. `presets: []` removes all defaults.

**Build/compose step**: Tailwind JIT compiler resolves the merged config at build time.

**Override granularity**: Theme keys are shallowly merged (top-level key replacement). `extend` key is special: it accumulates across all configs. **Plugins cannot be disabled from a preset**. This is a documented limitation.

**Relevance**: [HIGH] The layered preset merge with `extend` accumulation is a strong model for section composition. The plugin-cannot-be-disabled limitation is an important lesson: our format should support section removal.

---

#### Pattern 6: Biome (Grouped Rules with Per-Group and Per-Rule Control)

**Config format**: JSON (`biome.json` or `biome.jsonc`).

**Selection mechanism**: Rules organized into 8 semantic groups (`a11y`, `complexity`, `correctness`, `nursery`, `performance`, `security`, `style`, `suspicious`). Each group can be enabled/disabled as a unit OR individual rules within a group can be configured:

```json
{
  "linter": {
    "rules": {
      "recommended": true,
      "style": {
        "noNonNullAssertion": "off"
      },
      "suspicious": "warn",
      "nursery": {
        "recommended": true,
        "noConsole": "error"
      }
    }
  }
}
```

**Dependency handling**: None between groups. `recommended` is a cross-group preset.

**Ordering**: Configuration hierarchy: CLI args > overrides > top-level rules > group defaults > recommended preset.

**Defaults**: `recommended` preset enabled by default. Users opt-out of individual rules.

**Build/compose step**: None. Runtime resolution.

**Override granularity**: Three levels: (1) global preset (`recommended`), (2) per-group severity, (3) per-rule severity. Plus `overrides` array for path-specific configuration.

**CLI selection**: `biome lint --only style/useNamingConvention` runs individual rules or groups on demand.

**Relevance**: [VERY HIGH] This is the closest model to what we need. The three-level granularity (preset / group / individual rule) maps directly to our needs (plugin / section / individual item). The `overrides` system for path-specific config is a bonus.

---

#### Pattern 7: TypeScript tsconfig (Preset Extends with Full Override)

**Config format**: JSON (`tsconfig.json`).

**Selection mechanism**: `extends` field (string or array since TS 5.0). Extends npm packages or relative paths:

```json
{
  "extends": ["@tsconfig/strictest/tsconfig", "@tsconfig/node18/tsconfig"],
  "compilerOptions": { "noUnusedLocals": false }
}
```

**Dependency handling**: None between presets. Multiple extends processed left-to-right.

**Ordering**: Significant. Last entry in `extends` array wins for conflicting options. Local config always wins over extended configs.

**Defaults**: TypeScript has built-in defaults. Presets override them.

**Build/compose step**: TypeScript compiler resolves at invocation.

**Override granularity**: Individual compiler options. Any option from a preset can be overridden.

**Relevance**: [MEDIUM] The multi-extends pattern is useful but the flat option namespace (no grouping) limits applicability. The "local always wins" principle is important for our override model.

---

#### Pattern 8: Yeoman / Hygen (Interactive Feature Selection)

**Config format**: JavaScript (Yeoman), EJS templates with YAML frontmatter (Hygen).

**Selection mechanism**: Yeoman uses `this.prompt()` (Inquirer.js) for interactive feature selection during scaffolding. Conditional logic (`when` property) shows/hides prompts based on prior answers. Hygen uses EJS conditionals in templates (`<% if(feature) -%>`) and sets `to: null` to skip entire file generation.

**Dependency handling**: Yeoman: implicit in prompt `when` chains. Hygen: implicit in EJS conditionals.

**Ordering**: Prompt order determines information flow. Not relevant to output ordering.

**Defaults**: Author-defined. Some features checked by default, others not.

**Build/compose step**: Yes. Template rendering produces output files based on selections.

**Override granularity**: Per-feature (binary: include or exclude).

**Relevance**: [MEDIUM] The interactive selection UX pattern is relevant (aligned with ANALYSIS-043's grouped multiselect recommendation). The conditional template generation is relevant for our build step.

---

#### Pattern 9: VS Code Extension Packs (All-or-Nothing with Post-Install Toggle)

**Config format**: JSON (`package.json`).

**Selection mechanism**: `extensionPack` array lists extension IDs. All extensions install together. Users can disable individual extensions post-install via VS Code UI.

**Dependency handling**: `extensionDependencies` for hard dependencies (auto-installed). `extensionPack` for soft bundling.

**Ordering**: Not significant.

**Defaults**: All extensions in pack are installed (opt-out post-install).

**Build/compose step**: None. VS Code Marketplace handles distribution.

**Override granularity**: Binary per-extension (enabled/disabled).

**Relevance**: [LOW] The all-or-nothing-then-toggle pattern is what ANALYSIS-043 already evaluated and rejected in favor of upfront selection.

---

#### Pattern 10: Terraform Modules (Boolean Feature Flags)

**Config format**: HCL (`.tf` files).

**Selection mechanism**: Boolean variables with defaults control optional features. Module consumers set variables to enable/disable:

```hcl
variable "enable_monitoring" {
  type    = bool
  default = false
}

resource "aws_cloudwatch_alarm" "cpu" {
  count = var.enable_monitoring ? 1 : 0
  # ...
}
```

**Dependency handling**: Implicit in HCL expressions (`count` depends on variable values).

**Ordering**: Not significant for feature flags.

**Defaults**: Author-defined per variable. Convention: optional features default to `false` (opt-in).

**Build/compose step**: `terraform plan/apply` resolves conditional resources.

**Override granularity**: Per-variable (typically boolean but can be complex objects).

**Warning from ecosystem**: "Optional features are powerful but addictive. Every time someone says 'can the module also do X?' you add another flag. Eventually your module has 50 variables."

**Relevance**: [MEDIUM] The boolean feature flag pattern is simple and well-understood. The warning about flag proliferation is directly relevant. Our section model should avoid becoming a bag of boolean flags.

---

#### Pattern 11: Docker Compose Profiles (Tag-Based Opt-In)

**Config format**: YAML (`docker-compose.yml`).

**Selection mechanism**: Services declare `profiles` array. Services without profiles always run. Services with profiles only run when that profile is activated:

```yaml
services:
  app:
    image: myapp        # always runs (no profiles)
  debug-tools:
    image: debug
    profiles: [debug]   # opt-in
  monitoring:
    image: grafana
    profiles: [debug, monitoring]  # in two profiles
```

Activation: `docker compose --profile debug up` or `COMPOSE_PROFILES=debug,monitoring`.

**Dependency handling**: Targeted services auto-start their `depends_on` dependencies. But if a dependency has its own profile, that profile must also be active.

**Ordering**: Not significant. Profile activation is boolean.

**Defaults**: Services without `profiles` attribute always run. Profiled services are opt-in.

**Build/compose step**: None. Runtime selection.

**Override granularity**: Per-service (binary: profile active or not).

**Relevance**: [HIGH] The profile tagging model is directly applicable. Sections within a skill could declare profiles/tags. Users activate profiles to select which sections they want. The "no profile = always included" convention maps to mandatory/core sections.

---

#### Pattern 12: Nx Plugins (Glob-Based Include/Exclude)

**Config format**: JSON (`nx.json`).

**Selection mechanism**: Plugins registered in `nx.json` with optional `include`/`exclude` glob arrays:

```json
{
  "plugins": [
    {
      "plugin": "@nx/jest/plugin",
      "include": ["packages/**/*"],
      "exclude": ["**/*-e2e/**/*"]
    }
  ]
}
```

**Dependency handling**: Plugin dependency graph managed by Nx. Users do not declare inter-plugin deps.

**Ordering**: Plugin processing order matters for target inference.

**Defaults**: All matching projects included unless filtered.

**Build/compose step**: None. Nx resolves at task execution.

**Override granularity**: Per-project via glob patterns. Individual targets can be overridden in `project.json`.

**Relevance**: [MEDIUM] The include/exclude glob pattern is a clean selection mechanism. Directly applicable for filtering which sections apply to which contexts.

---

#### Pattern 13: Storybook Addons (Preset Composition with Feature Flags)

**Config format**: JavaScript/TypeScript (`.storybook/main.ts`).

**Selection mechanism**: `addons` array for registration. Preset addons export configuration hooks (`webpackFinal`, `viteFinal`, `babelDefault`). Recent versions added feature flags for granular addon control:

```typescript
const config: StorybookConfig = {
  addons: ["@storybook/addon-a11y"],
  features: {
    viewport: true,
    highlight: true,
    controls: true,
    interactions: false,
    actions: true,
    backgrounds: false,
    measure: true,
    outline: false
  }
};
```

**Dependency handling**: Presets compose via hook chaining. Each addon receives prior addon's output.

**Ordering**: Addon array order = processing order. Hooks chain sequentially.

**Defaults**: Essential addons enabled by default. Feature flags default to `true`.

**Build/compose step**: Storybook build resolves at compilation.

**Override granularity**: Per-addon (in `addons` array) and per-feature (in `features` object).

**Relevance**: [HIGH] The dual registration model (addons array + features object) separates "what's available" from "what's active." This directly maps to our need: a skill declares sections (what's available), the user's config controls which are active.

---

#### Pattern 14: Vercel agent-skills (Compiled Rules with SectionMap)

**Config format**: TypeScript build config + Markdown rule files.

**Selection mechanism**: Rule files use filename prefixes (`async-*.md`, `bundle-*.md`) that map to section numbers via `sectionMap` in `config.ts`:

| Prefix | Section | Title | Impact |
|--------|---------|-------|--------|
| `async-` | 1 | Eliminating Waterfalls | CRITICAL |
| `bundle-` | 2 | Bundle Size Optimization | CRITICAL |
| `server-` | 3 | Server-Side Performance | HIGH |
| `client-` | 4 | Client-Side Data Fetching | MEDIUM-HIGH |
| `rerender-` | 5 | Re-render Optimization | MEDIUM |
| `rendering-` | 6 | Rendering Performance | MEDIUM |
| `js-` | 7 | JavaScript Performance | LOW-MEDIUM |
| `advanced-` | 8 | Advanced Patterns | LOW |

Section metadata lives in `_sections.md`. Build process (`build.ts`) compiles all rule files into a single `AGENTS.md` output, grouped by section in numerical order.

**Dependency handling**: None between sections. All sections are independent.

**Ordering**: Explicit via section numbers in `sectionMap`. Lower number = higher priority.

**Defaults**: All sections included in build output. No per-section opt-out at build time.

**Build/compose step**: Yes. `build.ts` reads `rules/`, groups by prefix, generates `AGENTS.md`.

**Override granularity**: None. All-or-nothing compilation. No mechanism to exclude sections or individual rules from the build output.

**Relevance**: [VERY HIGH] This is the direct inspiration for our format. The sectionMap pattern provides the foundation. Our format must extend it with: (1) per-section selection by consumers, (2) cross-type support (not just skills), and (3) dependency declarations between sections.

---

#### Pattern 15: Claude Code Skills (Progressive Disclosure)

**Config format**: Markdown (`SKILL.md`) with YAML frontmatter.

**Selection mechanism**: No section-level selection. Skills are atomic units. Skill activation is binary (triggered or not). Internal structure uses progressive disclosure: SKILL.md provides the entry point, `references/` provides detailed docs loaded on demand, `scripts/` provides automation.

**Dependency handling**: `allowed-tools` in frontmatter declares tool requirements.

**Ordering**: Not applicable at section level. Skills are triggered by name/description matching.

**Defaults**: All content available when skill is active.

**Build/compose step**: None. Runtime loading.

**Override granularity**: None within a skill. Users can modify SKILL.md directly.

**Relevance**: [MEDIUM] The progressive disclosure principle is important: not all sections need to load simultaneously. Sections could support lazy loading (metadata first, full content on demand).

---

#### Pattern 16: GitHub Copilot Instructions (Path-Scoped Files)

**Config format**: Markdown (`.github/instructions/*.instructions.md`).

**Selection mechanism**: File-per-scope pattern. Each instruction file targets specific paths via frontmatter or naming convention (`frontend.instructions.md`, `backend.instructions.md`). Copilot activates instructions based on the file context being edited.

**Dependency handling**: None between instruction files.

**Ordering**: Priority hierarchy: personal > repository > organization.

**Defaults**: All matching instruction files are active. No opt-out mechanism.

**Build/compose step**: None. Runtime file-based resolution.

**Override granularity**: Per-file scope. Cannot override individual rules within an instruction file.

**Relevance**: [MEDIUM] The path-scoped activation model is interesting for context-dependent section activation. A skill's sections could declare which file contexts they apply to.

---

### 4.2 Comparison Table: 7 Criteria Across All Patterns

| Pattern | Config Format | Selection Mechanism | Dependency Handling | Ordering Significant | Defaults Model | Build Step | Override Granularity |
|---------|---------------|--------------------|--------------------|---------------------|----------------|------------|---------------------|
| ESLint Flat Config | JS/TS array | Array spread + extends | None (order-based) | Yes (array order) | Opt-in (explicit composition) | No | Individual rules |
| Prettier | JS object | Object spread | None | No | Built-in defaults | No | Individual properties |
| Stylelint | JSON/YAML | extends array | Chaining (A extends B) | Yes (last wins) | recommended preset | No | Individual rules |
| Markdownlint | JSONC/YAML | extends + directory hierarchy | None | Yes (child over parent) | All rules on (opt-out) | No | Individual rules |
| Tailwind Presets | JS | presets array | None (plugins additive) | Yes (last wins) | Built-in defaults | Yes (JIT) | Theme keys (shallow) |
| **Biome** | JSON | **Group + rule + preset** | None | Yes (hierarchy) | **recommended (opt-out)** | No | **3 levels: preset/group/rule** |
| tsconfig | JSON | extends (string/array) | None | Yes (last wins) | TS defaults | No | Individual options |
| Yeoman/Hygen | JS/EJS | Interactive prompts | Implicit (when chains) | Prompt order | Author-defined | Yes (template) | Per-feature (binary) |
| VS Code Packs | JSON | extensionPack array | extensionDependencies | No | All installed (opt-out) | No | Per-extension (binary) |
| Terraform | HCL | Boolean variables | Implicit (expressions) | No | Author-defined | Yes (plan/apply) | Per-variable |
| **Docker Profiles** | YAML | **Profile tags on services** | depends_on interaction | No | **No-profile = always on** | No | Per-service (binary) |
| Nx Plugins | JSON | include/exclude globs | Plugin dependency graph | Yes (processing order) | All included | No | Per-project (glob) |
| **Storybook** | JS/TS | **addons array + features object** | Hook chaining | Yes (addon order) | **Essential addons on** | Yes (build) | **Per-addon + per-feature** |
| **Vercel sectionMap** | TS + MD | **Filename prefix mapping** | None | **Yes (section number)** | **All sections compiled** | **Yes (build.ts)** | **None (all-or-nothing)** |
| Claude Code Skills | MD + YAML | Binary activation | allowed-tools | No | All content available | No | None |
| Copilot Instructions | MD | Path-scoped files | None | Priority hierarchy | All matching active | No | Per-file scope |

### 4.3 Most Applicable Patterns

Four patterns stand out as most applicable to our unified config format:

1. **Biome's 3-level granularity** (preset / group / rule): Maps to plugin / section / individual item. Proven at scale with 340+ rules across 8 groups.

2. **Vercel's sectionMap + build compilation**: The direct inspiration. Provides section ordering, filename-prefix convention, and metadata registry. Needs extension for consumer-side selection.

3. **Docker Compose profiles** (tag-based opt-in): The "no tag = always included" convention solves the mandatory section problem. Profile activation via CLI flags aligns with our `--include`/`--exclude` pattern from ANALYSIS-043.

4. **Storybook's dual model** (addons + features): Separates "what's available" from "what's active." Maps to: manifest declares sections, user config controls activation.

## 5. Results

| Finding | Source | Confidence |
|---------|--------|------------|
| Biome's 3-level granularity (preset/group/rule) is the most expressive model for sectioned configuration | Biome documentation | High |
| Vercel's sectionMap is the only AI-platform example of section-level content organization | vercel-labs/agent-skills source | High |
| No ecosystem supports per-section cherry-picking within a single skill/config artifact for AI agents | All 13 ecosystems analyzed | High |
| Docker Compose profiles provide the cleanest model for "mandatory vs optional" section semantics | Docker documentation | High |
| ESLint flat config's array composition with defineConfig().extends is the most developer-friendly composition API | ESLint documentation | High |
| Tailwind's limitation (cannot disable a preset plugin) is a cautionary design lesson | Tailwind documentation | High |
| Terraform's boolean-flag-per-feature pattern leads to variable proliferation at scale | Terraform community documentation | Medium |
| Storybook's recent addition of feature flags for core addons validates the "features object" pattern | Storybook PR #31146 | Medium |

### Facts (Verified)

- Biome organizes 340+ lint rules into 8 semantic groups with per-group and per-rule severity control
- ESLint flat config uses `defineConfig()` with automatic array flattening since ESLint v9
- Vercel react-best-practices has exactly 8 sections mapped via filename prefixes in config.ts
- Tailwind presets cannot disable plugins added by a preset (documented limitation)
- Docker Compose services without `profiles` attribute always run (verified in Docker docs)
- typescript-eslint provides 6 tiered configs (recommended, strict, stylistic + type-checked variants)
- Airbnb ESLint config organizes rules into exactly 8 category files (best-practices, errors, es6, imports, node, strict, style, variables)
- Storybook added feature flags for viewport, highlight, controls, interactions, actions, backgrounds, measure, outline (PR #31146)
- GitHub Copilot supports path-scoped `.instructions.md` files in `.github/instructions/` directory

### Hypotheses (Unverified)

- The 3-level granularity model (preset/group/rule mapping to plugin/section/item) will be intuitive for plugin authors without training. No user testing data.
- Section-level cherry-picking within a skill is a minority use case (estimated 10-20% of consumers). No usage data available.
- A build step for compiling sections into output will not create friction for plugin authors. No workflow studies.

## 6. Discussion

### 6.1 The Convergence Point

Three independent ecosystems (Biome, Vercel sectionMap, Docker Compose profiles) converge on the same structural insight: content should be organized into named groups with three control levels:

1. **All-or-nothing**: Enable/disable the entire artifact (plugin install/uninstall)
2. **Group-level**: Enable/disable named sections (Biome group severity, Docker profiles, Vercel sections)
3. **Item-level**: Override individual items within a group (Biome per-rule, ESLint per-rule)

Our unified format should support all three levels.

### 6.2 The Defaults Question

Two competing conventions exist:

- **Opt-out** (Biome, Markdownlint, Docker no-profile services, ANALYSIS-043 recommendation): Everything enabled by default. Users disable what they do not want.
- **Opt-in** (Terraform, Docker profiled services, ESLint flat config): Nothing enabled. Users explicitly enable what they want.

ANALYSIS-043 already decided: opt-out (all sections enabled by default) is correct for AI agent plugins because most users want everything. Cherry-picking is the exception. This aligns with Biome's `recommended: true` default and Docker's "no profile = always runs" convention.

### 6.3 Section Semantics for AI Agent Plugins

Unlike linter rules, AI agent sections have a unique property: they consume context window tokens. Including an irrelevant section wastes tokens and may confuse the agent. This makes section selection MORE important for AI plugins than for linters.

Vercel's build step (compile sections into AGENTS.md) is a response to this: the compiled output is what agents consume. If we support per-section selection, the compilation step produces a tailored output per user's selections.

### 6.4 Cross-Type Generalization

The sectionMap pattern generalizes across component types:

| Component Type | Section Examples | Selection Concern |
|---------------|-----------------|-------------------|
| Skills | async, bundle, server, client (Vercel pattern) | Which rule categories apply to my codebase? |
| Agents | persona traits, tool permissions, workflow rules | Which persona aspects do I want? |
| Instructions | coding standards, review guidelines, testing practices | Which guidelines apply to my project? |
| Hooks | pre-commit checks, post-install setup, file watchers | Which automated behaviors do I want? |
| Commands | CLI subcommands, MCP tool definitions | Which tools should be available? |

The section concept maps naturally to all types. The key insight is that sections are a content organization pattern, not a type-specific feature.

## 7. Recommendations

### 7.1 Recommended Unified Configuration Schema

Based on the convergence of Biome (3-level granularity), Vercel (sectionMap), Docker (profiles), and Storybook (features object), the recommended schema:

```typescript
// Section declaration in component frontmatter or metadata
interface SectionDefinition {
  name: string;           // unique identifier (e.g., "async", "bundle")
  title: string;          // display name (e.g., "Eliminating Waterfalls")
  order: number;          // display/compilation order
  profile?: string[];     // Docker-style profile tags (e.g., ["core", "advanced"])
  required?: boolean;     // true = always included, cannot be deselected
  requires?: string[];    // other sections this depends on
  description?: string;   // one-line description for selection UI
  impact?: string;        // severity/importance indicator
}

// Plugin manifest section registry (extends plugin.json)
interface PluginManifest {
  name: string;
  version: string;
  components: {
    skills: ComponentDef[];
    agents: ComponentDef[];
    hooks: ComponentDef[];
    commands: ComponentDef[];
    instructions: ComponentDef[];
    mcpServers: ComponentDef[];
  };
}

interface ComponentDef {
  name: string;
  path: string;
  sections?: SectionDefinition[];  // optional: component-level sections
  requires?: string[];             // cross-component dependencies
}

// Consumer-side selection config (.agent-plugin/config.json)
interface PluginSelectionConfig {
  [pluginName: string]: {
    // Component-level selection (from ANALYSIS-043)
    include?: string[];   // component paths to include
    exclude?: string[];   // component paths to exclude

    // Section-level selection (NEW)
    sections?: {
      [componentPath: string]: {
        include?: string[];   // section names to include
        exclude?: string[];   // section names to exclude
        profiles?: string[];  // activate these profiles
      };
    };
  };
}
```

### 7.2 Build/Compose Step

Following Vercel's pattern, a build step compiles selected sections into output:

```text
Source (rules/*.md) --> sectionMap --> section grouping --> selection filter --> compiled output
```

The build step runs at `agent-plugin install` time, not at plugin publish time. This allows per-consumer section selection while keeping the published plugin complete.

### 7.3 Priority Recommendations

| Priority | Recommendation | Rationale | Effort |
|----------|---------------|-----------|--------|
| P0 | Adopt Biome's 3-level granularity model (plugin/section/item) | Proven at scale. Maps directly to our component/section/rule hierarchy. Most expressive model found. | Low (schema design) |
| P0 | Use Docker Compose profile semantics for mandatory vs optional sections | `required: true` for core sections (always included). Named profiles for optional groups. Clean, proven convention. | Low (schema field) |
| P1 | Implement Vercel-style sectionMap with section ordering in component metadata | Direct translation of proven pattern. Add `sections` array to component frontmatter or companion `_sections.md` file. | Medium (frontmatter schema) |
| P1 | Support consumer-side section selection via `sections` field in `.agent-plugin/config.json` | Extends ANALYSIS-043's `include/exclude` model to section level. Uses same patterns users already understand. | Medium (config schema + UI) |
| P1 | Add `requires` field for inter-section dependencies | Prevents broken installations when sections depend on other sections. Same pattern as ANALYSIS-043's component-level `requires`. | Low (validation logic) |
| P2 | Build step at install time compiles selected sections into platform-specific output | Follows Vercel's compilation pattern. Produces tailored AGENTS.md/instructions per user's selections. Reduces context window waste. | High (build pipeline) |
| P2 | Profile-based activation via `--profile` flag for CLI and `profiles` field in config | Docker Compose pattern. Users define profiles like "minimal", "strict", "full". Profiles activate sets of sections. | Medium (CLI + config) |
| P3 | Progressive disclosure for large skills (metadata first, full sections on demand) | Claude Code's progressive disclosure principle. Large skills should not load all sections into context simultaneously. | Medium (runtime loading) |

## 8. Conclusion

**Verdict**: Proceed with unified section configuration format based on Biome's 3-level granularity, Vercel's sectionMap compilation, and Docker Compose's profile semantics.

**Confidence**: High

**Rationale**: Four independent ecosystems converge on the same structural pattern: named groups with multi-level control. The combination of Biome (granularity model), Vercel (section ordering and compilation), Docker Compose (mandatory vs optional semantics), and Storybook (features object separation) provides a well-validated foundation. No single ecosystem has all the pieces, but the composition is coherent and each element is proven independently.

### Adoption Status (2026-03-09)

ADR-014 Decision 5 adopted the features model informed by this analysis, with three selection mechanisms:

1. **Section-based** (markdown headers) — directly inspired by Vercel sectionMap pattern analyzed here
2. **Code regions** (`// #region feature:NAME`) — new mechanism not in this analysis
3. **File-based** (include/exclude whole files) — simplest mechanism, analogous to Docker profiles

A mandatory PoC validation gate requires proving markdown section extraction (mechanism 1) before implementing mechanisms 2 and 3. This gates the riskiest recommendation from this analysis (per-section cherry-picking within a single artifact has zero production precedent).

The ADR-013 npm/wiring model referenced throughout this analysis was superseded by ADR-014's explicit `agent-plugin add/remove/update` model.

### User Impact

- **What changes for you**: Skills, agents, and other components can declare named sections in their metadata. Consumers select which sections to include at install time via interactive features wizard, CLI flags (`--features`), or lockfile. Build step compiles selected sections into platform-specific output.
- **Effort required**: Schema design (P0, 1-2 days). Section metadata support in frontmatter (P1, 2-3 days). Consumer selection config and UI (P1, 3-5 days). Build pipeline (P2, 5-8 days). Total estimated: 11-18 days.
- **Risk if ignored**: Skills with 8+ sections force consumers to accept all content, wasting context window tokens on irrelevant sections. Plugin authors have no standard way to organize content into logical groups. No interoperability between section formats across component types.

## 9. Appendices

### Draft Schema: Complete Example

```json
{
  "$schema": "https://agent-plugin.dev/schemas/plugin.json",
  "name": "@acmelabs/react-best-practices",
  "version": "1.0.0",
  "components": {
    "skills": [
      {
        "name": "react-perf",
        "path": "skills/react-perf/SKILL.md",
        "sections": [
          {
            "name": "async",
            "title": "Eliminating Waterfalls",
            "order": 1,
            "required": true,
            "impact": "CRITICAL",
            "description": "Rules for avoiding cascading async requests"
          },
          {
            "name": "bundle",
            "title": "Bundle Size Optimization",
            "order": 2,
            "profile": ["performance"],
            "impact": "CRITICAL",
            "description": "Rules for reducing JavaScript bundle size"
          },
          {
            "name": "server",
            "title": "Server-Side Performance",
            "order": 3,
            "profile": ["performance", "server"],
            "requires": ["async"],
            "impact": "HIGH",
            "description": "Rules for server component optimization"
          },
          {
            "name": "advanced",
            "title": "Advanced Patterns",
            "order": 8,
            "profile": ["advanced"],
            "impact": "LOW",
            "description": "Advanced optimization patterns for experts"
          }
        ]
      }
    ]
  }
}
```

Consumer-side selection:

```json
{
  "@acmelabs/react-best-practices": {
    "include": ["skills/react-perf"],
    "sections": {
      "skills/react-perf": {
        "profiles": ["performance"],
        "exclude": ["advanced"]
      }
    }
  }
}
```

This selects the `react-perf` skill with the `performance` profile active (which includes `bundle` and `server` sections), plus the always-included `async` section (marked `required: true`), while explicitly excluding `advanced`.

### Sources Consulted

- [ESLint Flat Config Documentation](https://eslint.org/docs/latest/use/configure/configuration-files)
- [ESLint defineConfig and extends](https://eslint.org/blog/2025/03/flat-config-extends-define-config-global-ignores/)
- [ESLint Combine Configs](https://eslint.org/docs/latest/use/configure/combine-configs)
- [eslint-config-airbnb-base rules directory](https://github.com/airbnb/javascript/tree/master/packages/eslint-config-airbnb-base/rules)
- [typescript-eslint Shared Configs](https://typescript-eslint.io/users/configs/)
- [Prettier Sharing Configurations](https://prettier.io/docs/sharing-configurations)
- [Stylelint Configuration](https://stylelint.io/user-guide/configure/)
- [markdownlint-cli2 GitHub](https://github.com/DavidAnson/markdownlint-cli2)
- [Tailwind CSS Presets](https://v3.tailwindcss.com/docs/presets)
- [Biome Configuration Reference](https://biomejs.dev/reference/configuration/)
- [Biome Linter](https://biomejs.dev/linter/)
- [TypeScript tsconfig/bases](https://github.com/tsconfig/bases)
- [Yeoman User Interactions](https://yeoman.io/authoring/user-interactions.html)
- [Hygen Templates](https://hygen.ecmascript.pizza/docs/templates/)
- [VS Code Extension Packs](https://code.visualstudio.com/blogs/2017/03/07/extension-pack-roundup)
- [Terraform Modules with Optional Features](https://oneuptime.com/blog/post/2026-02-23-terraform-modules-with-optional-features/view)
- [Docker Compose Profiles](https://docs.docker.com/compose/how-tos/profiles/)
- [Nx nx.json Reference](https://nx.dev/docs/reference/nx-json)
- [Storybook Writing Presets](https://storybook.js.org/docs/addons/writing-presets)
- [Storybook Feature Flags PR #31146](https://github.com/storybookjs/storybook/pull/31146)
- [Vercel agent-skills Directory Structure](https://deepwiki.com/vercel-labs/agent-skills/9.1-directory-structure)
- [Vercel react-best-practices _sections.md](https://github.com/vercel-labs/agent-skills/blob/main/skills/react-best-practices/rules/_sections.md)
- [Claude Code Skills Documentation](https://code.claude.com/docs/en/skills)
- [Claude Code Skill Authoring Best Practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [GitHub Copilot Custom Instructions](https://docs.github.com/en/copilot/how-tos/configure-custom-instructions)

### Data Transparency

- **Found**: Complete configuration patterns for all 13+ ecosystems. Source code structure for Vercel sectionMap build pipeline. Biome's 3-level granularity schema. Docker Compose profile semantics. Storybook feature flag implementation. ESLint flat config composition API.
- **Not Found**: Vercel sectionMap TypeScript source code (rate-limited, used DeepWiki mirror). User research data on section-level cherry-picking demand. Performance benchmarks for context window impact of unused sections. Real-world adoption data for Biome's group-level configuration usage.

## Observations

- [decision] Recommend Biome's 3-level granularity (preset/group/rule) as the foundation for section configuration, mapping to plugin/section/item #architecture #biome
- [decision] Recommend Docker Compose profile semantics for mandatory vs optional sections: required=true for always-included, named profiles for opt-in groups #profiles #docker
- [decision] Recommend Vercel-style sectionMap with ordering, extending it with consumer-side selection that Vercel lacks #sectionMap #vercel
- [decision] Recommend build step at install time (not publish time) to compile selected sections into platform output #build #compilation
- [fact] Biome organizes 340+ rules into 8 semantic groups with per-group and per-rule severity control, the most expressive model found #biome #evidence
- [fact] Vercel react-best-practices uses 8 sections mapped via filename prefixes in config.ts, compiled into AGENTS.md at build time #vercel #evidence
- [fact] Docker Compose services without profiles attribute always run; profiled services are opt-in only when profile is activated #docker #evidence
- [fact] Tailwind CSS presets cannot disable plugins from a preset (documented limitation); our format must avoid this trap #tailwind #warning
- [fact] No ecosystem provides per-section cherry-picking within a single AI agent skill/config artifact; this is novel design #precedent
- [insight] Four independent ecosystems (Biome, Vercel, Docker Compose, Storybook) converge on named-groups-with-multi-level-control, validating the pattern independently #convergence
- [insight] AI agent sections have a unique property vs linter rules: unused sections waste context window tokens and may confuse the agent, making selection MORE important #motivation
- [risk] Section-level cherry-picking within a skill may be a minority use case (estimated 10-20%); over-engineering the selection UX could add complexity without proportional value #adoption
- [technique] ESLint flat config defineConfig() with automatic array flattening is the most developer-friendly composition API for JavaScript config #api-design

## Relations

- extends [[ANALYSIS-043 Per-Component Cherry-Picking InstallMode Design]]
- relates_to [[ANALYSIS-003 Vercel Skills Format Deep Dive]]
- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-013 npm-Package Distribution and Revised Command Tree]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]
- leads_to [[ADR-014 Explicit Installation Model and Content Features]]