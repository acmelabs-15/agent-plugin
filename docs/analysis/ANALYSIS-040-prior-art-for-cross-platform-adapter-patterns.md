---
title: ANALYSIS-040 Prior Art for Cross-Platform Adapter Patterns
type: analysis
permalink: analysis/analysis-040-prior-art-for-cross-platform-adapter-patterns-1
tags:
- adapter-pattern
- cross-platform
- prior-art
- research
- architecture
---

> **Platform Scope Change**: Per ADR-002 Amendment #1 (2026-03-09), supported platforms reduced from 7 to 4: Claude Code, Cursor, GitHub Copilot, Kiro. OpenCode, Amp, and Windsurf were dropped due to incomplete content type coverage. References to dropped platforms in this note are historical only.

# ANALYSIS-040 Prior Art for Cross-Platform Adapter Patterns

## 1. Objective and Scope

**Objective**: What existing solutions solve "one canonical definition, multiple platform-specific outputs"? Which patterns best fit our plugin system where a canonical plugin.json transforms into 7 AI platform-specific configurations?

**Scope**: 10 domains researched: Terraform providers, Kubernetes/Helm, cross-platform UI frameworks, configuration management (Ansible), build systems (CMake/Gradle), design token systems (Style Dictionary), API schema tools (OpenAPI/Protobuf), AI-specific prior art, plugin adapter patterns (ESLint/Babel/PostCSS), and the platform adapter pattern itself. Excludes implementation design (deferred to architect).

## 2. Context

The agent-plugin system (ADR-001) defines plugins in a canonical plugin.json with skills, agents, prompts, hooks, and MCP servers. ADR-009 established a data-driven platform registry (platforms.config.json) for file paths, binary detection, and MCP config locations. ANALYSIS-014 recommended a hybrid D+C pattern (adapter with layered overrides) for platform-specific configuration. This analysis examines prior art from 10 domains to validate, refine, or challenge those decisions.

The core problem: 7 AI platforms (Claude Code, Cursor, Windsurf, Kiro, OpenCode, Amp, Copilot CLI) each expect slightly different config formats. Differences span file locations, field naming, frontmatter schemas, event name formatting, and MCP root keys. The canonical form must transform into each platform's native format.

## 3. Approach

**Methodology**: Web research across 16 queries covering architecture documentation, open-source implementations, and community analysis. Cross-referenced with project's existing analyses (ANALYSIS-014, ANALYSIS-029, ADR-009).

**Tools Used**: WebSearch (16 queries), Brain MCP (3 prior analysis reads), codebase analysis (7 existing docs)

**Limitations**: Some tools are evolving rapidly (AGENTS.md spec, OpenAgentsControl compatibility layer still in development). AI coding platform configs change frequently. Flutter's rendering pipeline and Bazel's platform rules were not deeply researched due to lower relevance to JSON config transformation.

## 4. Data and Analysis

### 4.1 Pattern Taxonomy

Research reveals 5 distinct architectural patterns used for "one definition, many outputs":

| Pattern | Core Mechanism | Examples |
|---------|---------------|----------|
| **Schema-First Code Generation** | IDL/schema compiled to platform-specific code | Protobuf, OpenAPI, GraphQL |
| **Token/Data Transform Pipeline** | Data tokens processed through sequential transforms | Style Dictionary, PostCSS, Babel |
| **Provider/Adapter Registry** | Pluggable providers implement a shared interface | Terraform, Ansible modules |
| **Template Rendering** | Templates + values = platform-specific output | Helm, CMake, OpenAPI Generator |
| **Abstraction Layer with Platform Bridge** | Shared abstraction layer, platform-specific bridge code | React Native, Flutter |

### 4.2 Terraform Providers

**How it works**: Users declare resources in HCL. Each cloud provider implements a Go plugin (provider) that maps HCL resources to platform-specific API calls. The Terraform Plugin SDK defines the provider contract: schema definition, CRUD operations, state management.

**Key architecture**:
- Resources follow the naming and structure of the target API
- Provider block sets defaults (region, credentials)
- Resource blocks inherit or override provider defaults
- Schema types are mapped between HCL types and API types
- Provider is a plugin binary communicating via gRPC

**What we can learn**:
- The provider interface is the contract. All providers implement the same lifecycle methods (Create, Read, Update, Delete). Our platform adapters could implement a similar interface (Transform, Validate, Write).
- Schema follows the API. Terraform does not abstract away cloud differences. It exposes each provider's native fields. This contrasts with our cross-platform concept mapping (ANALYSIS-014) where we abstract loading strategy across platforms.
- Provider defaults with resource overrides is exactly the layered pattern (Pattern C) that ANALYSIS-014 recommended.

**Applicability score**: 7/10. The provider interface pattern is directly applicable. The "follow the API" philosophy conflicts with our cross-platform abstraction goal.

### 4.3 Style Dictionary (Design Tokens)

**How it works**: Design tokens are defined once in JSON/YAML. Style Dictionary processes them through a transform pipeline to generate platform-specific outputs (CSS variables, Swift constants, Android XML, Flutter Dart).

**Key architecture**:
- Tokens are canonical data: `{ "color": { "primary": { "value": "#0066FF" } } }`
- Transforms are functions: `(token) => platformSpecificValue`
- Transforms are grouped into transform groups (e.g., "css", "ios", "android")
- Each platform registers: transforms (value conversion), formats (output file structure), actions (side effects like file writing)
- Pipeline: Parse tokens -> Apply transforms -> Format output -> Write files

**Critical insight -- the W3C Design Tokens spec (2025.10)**:
- First stable version of vendor-neutral token format
- Reference implementations in Style Dictionary, Tokens Studio, Terrazzo
- Proves that "canonical definition with platform transforms" scales to industry standard

**What we can learn**:
- Transform groups are the key abstraction. Instead of per-field adapters, group all transforms for a platform. Our "claude-code" transform group maps loading strategy, field names, and file paths together.
- Transforms are composable. A chain of small transforms (rename field, convert type, set default) is more maintainable than monolithic per-platform converters.
- The pipeline pattern (parse -> transform -> format -> write) maps directly to our install flow (Resolve -> Transform -> Apply -> Record).

**Applicability score**: 9/10. Highest applicability. Our problem is structurally identical: canonical JSON tokens transformed to platform-specific configs. Style Dictionary's transform group + pipeline architecture is the closest match.

### 4.4 OpenAPI Generator

**How it works**: One OpenAPI spec (YAML/JSON) generates client SDKs, server stubs, and documentation for 100+ languages. Generators are template-based: Mustache/Handlebars templates produce platform-specific code.

**Key architecture**:
- OpenAPI 2.0 and 3.x documents normalize into a single internal API model
- Each generator creates a data structure from the normalized model
- Templates (Mustache) render platform-specific output from the data structure
- Adapter pattern for template engines: `TemplatingEngineAdapter` interface allows swapping Mustache for Handlebars or custom engines

**What we can learn**:
- Normalization step is critical. OpenAPI normalizes different spec versions into one internal model before generators touch it. We should normalize plugin.json + platformConfig overrides into one resolved model before platform adapters transform it.
- Template-based generation works for "mostly text with some variable substitution." Our SKILL.md frontmatter generation is exactly this: template with platform-specific field values injected.
- Pluggable generators via interface. Adding a new language = implementing the generator interface + writing templates.

**Applicability score**: 7/10. Template rendering is relevant for SKILL.md generation. The normalization pattern is valuable. Less applicable for JSON config merging (MCP server entries).

### 4.5 Protocol Buffers

**How it works**: Define data structures once in `.proto` IDL. The `protoc` compiler generates code for 12+ languages. Each language plugin implements serialization/deserialization for that language's idioms.

**Key architecture**:
- Schema-first: `.proto` is the single source of truth
- Compiler plugins: each language has a `protoc` plugin that generates code
- Forward/backward compatibility via field numbering (fields can be added/removed without breaking)
- Cross-language type mapping: proto types map to each language's native types

**What we can learn**:
- Schema versioning via field numbering. Our plugin.json schema can version fields similarly: new fields get new numbers, removed fields are reserved. Maps to ADR-001's schema versioning strategy.
- Compiler plugin architecture. The protoc plugin interface (`CodeGeneratorRequest -> CodeGeneratorResponse`) is a clean model for our platform adapters: `CanonicalPlugin -> PlatformOutput`.

**Applicability score**: 6/10. The compiler plugin interface is elegant but over-engineered for JSON config transformation. Field numbering for schema evolution is a useful technique.

### 4.6 Ansible Modules

**How it works**: Playbooks declare tasks in YAML. Each task uses a module (e.g., `apt`, `yum`, `service`) that handles platform-specific execution. The same playbook runs on Debian (apt), RHEL (yum), and macOS (brew) by selecting the correct module based on gathered facts.

**Key architecture**:
- Facts gathered at runtime: OS, distro, version, architecture
- Module abstraction: `package` module delegates to `apt`/`yum`/`brew` based on facts
- Jinja2 templates for platform-conditional config rendering
- Roles bundle related tasks for reuse

**What we can learn**:
- Fact gathering = platform detection. Our dual detection (ADR-009) is analogous to Ansible's `setup` module that gathers OS facts.
- The `package` meta-module pattern: one abstract task that dispatches to platform-specific implementations. Our `loadingStrategy: "always"` is a meta-concept that dispatches to `alwaysApply: true` (Cursor) or `inclusion: always` (Kiro).
- Jinja2 conditional rendering: `{% if ansible_os_family == 'Debian' %}` maps to our `{% if platform == 'cursor' %}` for frontmatter generation.

**Applicability score**: 7/10. The meta-module dispatch pattern validates our cross-platform concept mapping. The fact-gathering parallel is already implemented (ADR-009).

### 4.7 Helm Charts

**How it works**: Helm charts define Kubernetes resources as Go templates. Values files customize the templates. Different environments (dev, staging, prod) use different values files to produce environment-specific manifests.

**Key architecture**:
- `values.yaml`: default configuration
- `-f production.yaml`: override file for specific environment
- Go template functions transform values into resource specs
- Priority: later values files override earlier ones
- Umbrella charts compose sub-charts for complex applications

**What we can learn**:
- Values override cascade: `default values.yaml` -> `environment values.yaml` -> `--set` flags. Maps directly to our resolution order: Agent Skills standard fields -> adapter defaults -> manifest platformConfig -> per-component overrides (ANALYSIS-014 Section 5).
- Template functions for transformation: `{{ .Values.service.port | quote }}` maps to our field transformation (`loadingStrategy: "always"` -> `alwaysApply: true`).

**Applicability score**: 6/10. The override cascade validates our layered approach. Template functions are useful for frontmatter generation. Less applicable because Helm targets one platform (Kubernetes) with environment variations, not fundamentally different platforms.

### 4.8 CMake

**How it works**: CMake generates native build files (Makefiles, Visual Studio projects, Xcode projects, Ninja files) from a single `CMakeLists.txt`. Toolchain files describe the target platform.

**Key architecture**:
- Meta-build system: does not build directly, generates platform-native build files
- Toolchain abstraction: compiler, linker, flags differ per platform
- System introspection: detects compilers, features, libraries at configure time
- INTERFACE targets represent abstract dependencies

**What we can learn**:
- Meta-tool pattern: CMake does not compile code; it generates config files for tools that do. agent-plugin does not run AI agents; it generates config files for platforms that do. This is an exact structural match.
- Toolchain files are analogous to our platforms.config.json: external data files that describe the target platform's capabilities and paths.

**Applicability score**: 7/10. The meta-tool philosophy validates our approach. Toolchain files validate platforms.config.json.

### 4.9 Gradle / Kotlin Multiplatform

**How it works**: Kotlin Multiplatform compiles shared Kotlin code to JVM, JS, Native, iOS, Android, and WASM targets. Each target has platform-specific source sets. Build variants combine build types with product flavors.

**Key architecture**:
- Shared source set (`commonMain`) contains platform-agnostic code
- Platform source sets (`iosMain`, `androidMain`) contain platform-specific code
- `expect`/`actual` declarations: shared code declares an interface (`expect`), platform code provides implementation (`actual`)
- Build variants: cross-product of build type and product flavor

**What we can learn**:
- expect/actual pattern: declare abstract platform concepts in the canonical form, implement concrete translations per platform. Our `loadingStrategy` is an `expect` declaration; each platform adapter provides the `actual` field mapping.
- Source sets for platform-specific overrides: `commonMain` = shared plugin content, `claudeCodeMain` = Claude Code-specific frontmatter extensions.

**Applicability score**: 6/10. The expect/actual metaphor is useful for thinking about cross-platform concepts. Build variants are less relevant since our "variants" are different platforms, not debug/release.

### 4.10 React Native

**How it works**: Components written in JavaScript/React compile to native iOS (UIKit) and Android (View system) widgets. A bridge layer handles communication between JS and native code.

**Key architecture**:
- Shared component tree in JavaScript
- Bridge translates to platform-native rendering
- `Platform.select({ ios: value, android: value })` for platform-specific values
- Platform-specific file extensions: `Component.ios.js`, `Component.android.js`
- Abstraction layer hides platform differences; native modules expose platform-specific capabilities

**What we can learn**:
- `Platform.select()` pattern: simple conditional mapping object. Our platform adapter could use the same pattern: `platformSelect({ "claude-code": { model: "opus" }, cursor: { alwaysApply: true } })`.
- Platform-specific file extensions: `skill.claude-code.md` and `skill.cursor.mdc` as override files. This is an alternative to the `platforms` frontmatter block in ANALYSIS-014.
- The bridge as a thin translation layer, not a reimplementation.

**Applicability score**: 5/10. The `Platform.select()` pattern is a useful API design idea. The bridge concept validates the thin adapter layer. Less applicable because React Native bridges complex rendering, not JSON config transformation.

### 4.11 Babel Transform Pipeline

**How it works**: JavaScript source code is parsed into an AST, transformed by plugins in sequence, and generated back to source code. Presets bundle related plugins.

**Key architecture**:
- Three phases: Parse -> Transform -> Generate
- Plugins are visitor-pattern functions that modify the AST
- Presets group plugins for common workflows
- Plugin ordering: first to last. Preset ordering: reversed.
- Composable transforms: each plugin makes one small change

**What we can learn**:
- Sequential transform pipeline. Our platform adapter could be a pipeline: `canonicalPlugin -> resolveOverrides -> mapConcepts -> translateFields -> formatOutput -> writeFiles`. Each step is a small, testable transform.
- Presets as platform profiles. A "claude-code" preset bundles all transforms needed for Claude Code output. A "cursor" preset bundles Cursor-specific transforms.
- Execution ordering matters. Transforms that set defaults must run before transforms that map fields.

**Applicability score**: 7/10. The pipeline architecture is directly applicable. Presets-as-platform-profiles is a clean abstraction.

### 4.12 PostCSS

**How it works**: CSS is parsed into an AST. Plugins transform the AST sequentially. The modified AST is stringified back to CSS.

**Key architecture**:
- Tokenizer -> Parser -> AST -> Plugin transforms -> Stringifier
- Plugins execute in the order they are added
- Each plugin receives the full AST and can modify any node
- Standard plugin factory pattern: `module.exports = (opts) => ({ postcssPlugin: 'name', Rule(rule) { ... } })`

**What we can learn**:
- The Stringifier concept: a dedicated component that converts the internal model back to the output format. Our platform adapter needs a stringifier per platform: the component that writes the platform's native format (YAML frontmatter for SKILL.md, JSON for MCP config, MDC format for Cursor rules).
- Plugin ordering determines output. Platform-specific transforms must run after cross-platform concept mapping.

**Applicability score**: 6/10. The stringifier concept is useful. The sequential plugin pattern validates the pipeline approach.

### 4.13 ESLint/Prettier Integration

**How it works**: Two independent tools (linter and formatter) with conflicting rules are reconciled through adapter packages. `eslint-config-prettier` disables conflicting ESLint rules. `eslint-plugin-prettier` runs Prettier as an ESLint rule.

**Key architecture**:
- Shared config packages published as npm modules
- Config extends/overrides pattern: base config + tool-specific overrides
- eslint-config-prettier acts as a conflict resolver between two platforms

**What we can learn**:
- Conflict resolution through disabling incompatible features. When a canonical concept has no equivalent on a platform, the adapter should explicitly skip it (not error).
- Shared config packages as npm modules. Platform adapter configs could be published as separate npm packages (`@acmelabs-15/adapter-claude-code`, `@acmelabs-15/adapter-cursor`).

**Applicability score**: 4/10. The conflict resolution pattern is relevant. Publishing adapters as packages is a future consideration. The ESLint/Prettier problem (reconciling two tools) is different from our problem (generating for many targets).

### 4.14 AI-Specific Prior Art

**AGENTS.md (Open Standard)**:
- Emerged from OpenAI Codex, Google Jules, Cursor, Factory collaboration
- Stewarded by Agentic AI Foundation (Linux Foundation)
- Used by 60k+ open-source projects
- Standard Markdown, no structured fields. Agent parses free text.
- Portable across all major AI coding tools
- Limitation: no structured config, no transform pipeline. It is instructions-only.

**AgentRuleGen (agentrulegen.com)**:
- Web tool that generates rules for Cursor, Claude Code, Copilot, Windsurf
- "Generate rules once and export to multiple formats"
- Proves market demand for cross-platform config generation
- Limitation: web-only, no CLI, no plugin system, no MCP support

**ClaudeMDEditor (claudemdeditor.com)**:
- Visual editor for CLAUDE.md, .cursorrules, and other AI config files
- Auto-finds AI config files across repos
- Manages skills, rules, and agent files across projects
- Limitation: editor only, does not generate or transform

**CC-Switch**:
- Desktop app for unified AI assistant config management
- One-click provider switching for Claude Code, Codex, Gemini CLI
- Unified MCP server management
- Limitation: switching tool, not a plugin system

**OpenAgentsControl**:
- AI agent framework with planned compatibility layer
- Targets Cursor, Claude Code, Windsurf, Copilot, Codeium, Tabnine
- Features: common agent/context interface format, translation layer, slash command translation, skill/ability mapping
- Limitation: compatibility layer still in development. OpenCode integration stabilizing first.

**AI Rules Fragmentation (2026 landscape)**:
- NIST launched AI Agent Standards Initiative (Feb 2026)
- Agentic AI Foundation (Anthropic, OpenAI, Block) consolidates MCP and AGENTS.md
- Sourcegraph's AGENT.md proposes universal file format
- Each platform still uses different files: CLAUDE.md, .cursorrules, .windsurfrules, .github/copilot-instructions.md, .kiro/steering/

**Key finding**: No existing tool does what agent-plugin proposes: a full plugin manager with canonical manifest, platform adapters, MCP server management, and conflict resolution. AgentRuleGen and ClaudeMDEditor solve adjacent problems (rule generation, config editing) but not the full install/update/uninstall lifecycle. OpenAgentsControl is the closest competitor but its compatibility layer is incomplete.

### 4.15 The W3C Design Tokens + MCP Parallel

The most striking parallel in the research is between Design Tokens standardization and AI agent config standardization:

| Aspect | Design Tokens (2015-2025) | AI Agent Config (2024-2026) |
|--------|--------------------------|---------------------------|
| Problem | Design values differ across iOS, Android, Web | Agent instructions differ across Claude, Cursor, Windsurf |
| Canonical format | W3C Design Tokens spec (JSON) | AGENTS.md / Agent Skills spec (Markdown + YAML) |
| Transform tool | Style Dictionary | (our agent-plugin) |
| Platform adapters | CSS, Swift, XML, Dart transforms | Claude Code, Cursor, Kiro adapters |
| Industry body | W3C Design Tokens Community Group | Agentic AI Foundation (Linux Foundation) |
| Maturity | Stable spec (2025.10), 10 years | Early standardization, 2 years |

This parallel validates our architecture. Style Dictionary solved the same structural problem for a different domain and succeeded at industry scale.

## 5. Results

### Pattern Applicability Ranking

| Rank | Pattern | Source | Score | Why |
|------|---------|--------|-------|-----|
| 1 | Token Transform Pipeline | Style Dictionary | 9/10 | Structurally identical problem: canonical JSON -> platform-specific outputs via composable transforms |
| 2 | Provider Interface + Schema | Terraform | 7/10 | Provider contract (Transform/Validate/Write) directly applicable; schema-follows-API conflicts with abstraction goal |
| 3 | Sequential Transform Pipeline | Babel, PostCSS | 7/10 | Pipeline architecture (parse -> transform -> format -> write) maps to install flow; presets-as-platforms |
| 4 | Meta-Build Generator | CMake | 7/10 | Meta-tool philosophy (generate configs, not run agents) validates approach; toolchain files validate platforms.config.json |
| 5 | Module Dispatch + Facts | Ansible | 7/10 | Meta-module dispatch validates cross-platform concept mapping; fact gathering validates platform detection |
| 6 | Normalize + Template | OpenAPI Generator | 7/10 | Normalization step before generation is critical; template rendering for SKILL.md frontmatter |
| 7 | Values Override Cascade | Helm | 6/10 | Override cascade validates layered resolution order |
| 8 | Expect/Actual Declarations | Kotlin Multiplatform | 6/10 | Conceptual model for abstract concepts with concrete platform implementations |
| 9 | Compiler Plugin Interface | Protobuf | 6/10 | Clean input/output contract; over-engineered for JSON transforms |
| 10 | Platform Bridge | React Native | 5/10 | Platform.select() API design idea; bridge concept validates thin adapter |

### Evidence Gathered

| Finding | Source | Confidence |
|---------|--------|-----------|
| Transform pipeline (parse -> transform -> format -> write) is the dominant pattern for canonical-to-platform conversion | Style Dictionary, Babel, PostCSS, OpenAPI Generator | High |
| Data-driven platform registry (external file, not code) is validated by CMake toolchains, Terraform provider schemas, Style Dictionary platform configs | CMake, Terraform, Style Dictionary | High |
| Layered override cascade (defaults -> manifest -> per-component) is used by Terraform, Helm, JetBrains, and Nx | 4 independent systems | High |
| No existing tool provides full plugin manager lifecycle (install/update/uninstall) for AI agent configs across platforms | Market survey of 5 AI-specific tools | High |
| AGENTS.md and Agent Skills spec are converging as canonical formats but lack structured transform pipeline | AGENTS.md spec, Agent Skills spec | Medium |
| OpenAgentsControl is the closest competitor but compatibility layer is incomplete | GitHub issue #141, project README | Medium |
| Cross-platform concept mapping (abstract concepts dispatched to platform fields) is validated by Ansible modules, Kotlin expect/actual, React Native Platform.select | 3 independent systems | High |

## 6. Discussion

### Key Question 1: Best pattern for JSON config transformation?

**Answer**: Style Dictionary's token transform pipeline. Our problem is structurally identical: canonical JSON data transformed to multiple platform-specific output formats via composable transforms. The pipeline architecture (parse canonical -> resolve overrides -> map concepts -> translate fields -> format output -> write files) is the recommended approach.

### Key Question 2: Should adapters be code (functions) or data (JSON schemas)?

**Answer**: Both. This is not an either/or question. The research shows successful systems use data for mapping and code for transformation:

- **Data (platforms.config.json)**: File paths, binary names, MCP root keys, content directories. Already decided in ADR-009.
- **Code (transform functions)**: Field name mapping (`loadingStrategy` -> `alwaysApply`), type conversion (`string[]` -> `comma-separated string`), conditional logic (`if platform lacks feature, skip`), format transformation (OpenCode's non-standard MCP format).

Style Dictionary uses this exact split: token files are data, transforms are code. Terraform does the same: resource schemas are data, CRUD operations are code.

The data file answers "where?" The transform functions answer "how?"

### Key Question 3: How to handle platform features with no equivalent?

Three strategies from prior art:

1. **Skip silently** (Agent Skills spec): Unknown frontmatter fields are ignored. If canonical form includes `model: opus` and Cursor has no model override, the field is silently dropped.
2. **Warn explicitly** (ESLint-config-prettier approach): Log a warning that `model` is Claude Code-specific and will be ignored on Cursor. Helps authors understand platform limitations.
3. **Degrade gracefully** (React Native): Provide a fallback. If the concept has no direct mapping, use the closest equivalent or embed in description text.

Recommended: **Strategy 2 (warn explicitly)** during development/authoring, **Strategy 1 (skip silently)** during install. Authors should know what they lose on each platform. End users should not see warnings for features that simply do not exist on their platform.

### Key Question 4: What handles "90% the same, 10% different" best?

**Answer**: The layered override pattern (Pattern C from ANALYSIS-014) combined with the transform pipeline. The 90% common case flows through the default pipeline unchanged. The 10% different case uses:

- platformConfig overrides in plugin.json (manifest-level)
- `platforms` block in SKILL.md frontmatter (component-level)
- Platform-specific transform functions for field mapping

This matches Style Dictionary (common tokens with platform-specific transforms), Terraform (provider defaults with resource overrides), and Helm (default values with environment overrides).

### Key Question 5: What minimizes work to add a new platform?

**Answer**: Three tiers of effort based on prior art:

| Tier | Effort | When | What to do |
|------|--------|------|-----------|
| Standard platform | Add JSON entry | Platform uses standard MCP format and Agent Skills fields | Add entry to platforms.config.json |
| Custom fields platform | Add JSON entry + field mapping | Platform has unique frontmatter fields (like Kiro's `inclusion`) | Add entry + add field mapping to concept-to-platform map |
| Custom format platform | Add JSON entry + field mapping + format transformer | Platform uses non-standard MCP format (like OpenCode) | Add entry + mapping + write transformer function |

ADR-009 already established Tier 1. This analysis adds Tier 2 (field mapping as data) and Tier 3 (format transformer as code).

### Validation of Existing Decisions

| Decision | Validated By | Verdict |
|----------|-------------|---------|
| platforms.config.json (data-driven registry) | CMake toolchains, Terraform provider schemas, Style Dictionary platform configs | [PASS] Strongly validated |
| Hybrid D+C pattern (adapter + layered overrides) | Terraform, Helm, JetBrains, Nx all use layered overrides | [PASS] Strongly validated |
| Cross-platform concept mapping (loadingStrategy, filePatterns) | Ansible meta-modules, Kotlin expect/actual, React Native Platform.select | [PASS] Validated |
| Plugin authors write platform-agnostic plugin.json | Style Dictionary (authors write tokens, not platform code), AGENTS.md (one file for all platforms) | [PASS] Validated |

No existing decision needs revision based on this research. The prior art strengthens confidence in all four decisions.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|----------|---------------|-----------|--------|
| P0 | Model the platform adapter as a transform pipeline: Resolve -> Transform -> Format -> Write | Style Dictionary, Babel, PostCSS all use this pattern. Composable, testable, extensible. | Medium |
| P0 | Split adapter logic into data (field mapping tables) and code (transform functions) | Every successful system uses this split. Data for "what maps where," code for "how to convert." | Medium |
| P1 | Implement cross-platform concept map as a static data table, not embedded in code | Concept mapping (loadingStrategy -> alwaysApply/inclusion) should be data-driven like platforms.config.json. Enables community additions without code changes. | Medium |
| P1 | Add platform-specific field mapping as a second data file (or section in platforms.config.json) alongside paths/detection | Currently platforms.config.json covers paths and detection. Field mapping (frontmatter schemas, concept translations) is the missing piece. | Medium |
| P2 | Implement "warn on author, skip on install" for unsupported platform features | ESLint-config-prettier validates the warn approach. Authors need feedback; end users need clean installs. | Low |
| P2 | Consider preset/profile pattern for common platform combinations | Babel presets group transforms. A "full-stack" preset targeting Claude Code + Cursor + Copilot bundles all three adapters. | Low |
| P3 | Explore community-contributed platform adapters as separate npm packages | Style Dictionary and Terraform both support third-party platform plugins. Enables platforms 8+ without core package changes. | High |

## 8. Conclusion

**Verdict**: Proceed. Existing architecture decisions are strongly validated by prior art.

**Confidence**: High

**Rationale**: 10 domains analyzed, 5 independent systems validate the data-driven registry pattern, 4 validate the layered override pattern, and 3 validate cross-platform concept mapping. Style Dictionary's token transform pipeline is the closest structural match to our problem and should be the primary architectural reference. No existing tool provides the full plugin manager lifecycle for AI agent configs, confirming this is a genuine gap in the market.

### User Impact

- **What changes for you**: No changes to existing decisions. This research validates ADR-009 (platforms.config.json), ANALYSIS-014 (hybrid D+C pattern), and the cross-platform concept mapping approach. The recommended transform pipeline architecture (Resolve -> Transform -> Format -> Write) provides a concrete implementation model.
- **Effort required**: Primary implementation work is the transform pipeline and field mapping data tables. Style Dictionary's architecture provides a well-documented reference implementation.
- **Risk if ignored**: Without the transform pipeline pattern, adapter logic becomes monolithic per-platform functions that are hard to test and extend. Without the data/code split, adding platforms requires code changes instead of data entries.

## Observations

- [fact] Style Dictionary's token transform pipeline (parse -> transform -> format -> write) is structurally identical to our canonical plugin-to-platform conversion problem, scoring 9/10 on applicability #prior-art #style-dictionary #transform-pipeline
- [fact] 5 independent systems (CMake toolchains, Terraform provider schemas, Style Dictionary platform configs, Helm values, Nx defaults) validate data-driven platform registries over code-based adapters #prior-art #data-driven #validation
- [fact] 4 independent systems (Terraform provider/resource defaults, Helm values cascade, JetBrains plugin.xml + per-IDE configs, Nx nx.json + project.json) validate the layered override pattern for multi-platform config #prior-art #layered-overrides
- [fact] No existing tool provides full plugin manager lifecycle (install/update/uninstall) for AI agent configs across platforms. AgentRuleGen, ClaudeMDEditor, CC-Switch, and OpenAgentsControl solve adjacent problems. #market-gap #competitive-landscape
- [fact] W3C Design Tokens spec reached stable v1 (2025.10) after 10 years, proving the canonical-definition-to-platform-outputs pattern scales to industry standard #design-tokens #w3c #validation
- [decision] Recommended architecture: transform pipeline (Resolve -> Transform -> Format -> Write) with data/code split: data for field mappings and paths, code for transform functions and format converters #architecture #transform-pipeline
- [insight] The W3C Design Tokens (2015-2025) and AI Agent Config (2024-2026) standardization trajectories are structurally parallel: same problem (cross-platform consistency), same solution shape (canonical format + transform tools), different domains #parallel #standardization
- [insight] Platform features with no cross-platform equivalent should warn during authoring (helps plugin authors understand limitations) and skip silently during install (clean experience for end users) #degradation-strategy #ux
- [technique] Three tiers of platform addition effort: Tier 1 (standard format) = JSON entry only, Tier 2 (custom fields) = JSON entry + field mapping data, Tier 3 (custom format) = JSON entry + mapping + code transformer #extensibility #tiered-effort
- [insight] The Terraform "schema follows the API" philosophy conflicts with cross-platform concept abstraction but is correct for platform-specific override blocks: those blocks should use platform-native field names, not abstractions #terraform #naming

## Relations

- extends [[ANALYSIS-014 Platform Config Patterns]]
- extends [[ANALYSIS-029 Platform Config Registry]]
- relates_to [[ADR-009 Platform Detection and Config Registry]]
- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-002 Target Platforms and Audiences]]

## 9. Appendices

### Sources Consulted

- Terraform Provider Design Principles: https://developer.hashicorp.com/terraform/plugin/best-practices/hashicorp-provider-design-principles
- Terraform Provider Configuration: https://developer.hashicorp.com/terraform/language/providers/configuration
- Terraform Custom Provider Architecture: https://shadow-soft.com/content/terraform-provider-architecture
- Style Dictionary GitHub: https://github.com/style-dictionary/style-dictionary
- W3C Design Tokens Specification (2025.10): https://www.w3.org/community/design-tokens/2025/10/28/design-tokens-specification-reaches-first-stable-version/
- Design Token-Based UI Architecture (Martin Fowler): https://martinfowler.com/articles/design-token-based-ui-architecture.html
- OpenAPI Generator GitHub: https://github.com/OpenAPITools/openapi-generator
- OpenAPI Generator Templates: https://openapi-generator.tech/docs/templating/
- Protocol Buffers Overview: https://protobuf.dev/overview/
- Ansible Architecture: https://spacelift.io/blog/ansible-architecture
- Helm Chart Template Guide: https://helm.sh/docs/chart_template_guide/
- CMake Cross-Platform Build: https://www.studyplan.dev/cmake/cross-platform-cmake
- CMake Toolchains: https://cmake.org/cmake/help/latest/manual/cmake-toolchains.7.html
- Kotlin Multiplatform DSL Reference: https://www.jetbrains.com/help/kotlin-multiplatform-dev/multiplatform-dsl-reference.html
- Gradle Build Variants: https://developer.android.com/build/build-variants
- React Native Architecture: https://reactnative.dev/architecture/overview
- Babel Plugins: https://babeljs.io/docs/plugins
- PostCSS Architecture: https://postcss.org/docs/postcss-architecture
- Adapter Pattern in TypeScript: https://refactoring.guru/design-patterns/adapter/typescript/example
- AGENTS.md Spec: https://agents.md/
- AGENTS.md GitHub: https://github.com/agentsmd/agents.md
- OpenAI Codex AGENTS.md Guide: https://developers.openai.com/codex/guides/agents-md/
- AgentRuleGen: https://www.agentrulegen.com/
- ClaudeMDEditor: https://www.claudemdeditor.com/
- OpenAgentsControl GitHub: https://github.com/darrenhinde/OpenAgentsControl
- OpenAgentsControl Compatibility Layer Issue: https://github.com/darrenhinde/OpenAgentsControl/issues/141
- AI Agent Rule Files Fragmentation: https://www.everydev.ai/p/blog-ai-coding-agent-rules-files-fragmentation-formats-and-the-push-to-standardize
- AI Coding Agent Rules Directory: https://aicodingrules.org/
- NIST AI Agent Standards Initiative: https://www.nist.gov/news-events/news/2026/02/announcing-ai-agent-standards-initiative-interoperable-and-secure
- Cursor Rules vs CLAUDE.md vs Copilot Instructions: https://www.agentrulegen.com/guides/cursorrules-vs-claude-md
- Unified Tool Calling Architecture: https://www.scalekit.com/blog/unified-tool-calling-architecture-langchain-crewai-mcp
- Agent Protocol for LLM Agents (LangChain): https://blog.langchain.com/agent-protocol-interoperability-for-llm-agents/
- MCP as Universal Bridge: https://engini.io/blog/model-context-protocol-mcp-the-universal-bridge-for-ai-agents-to-tools-data/

### Data Transparency

- **Found**: Complete architecture documentation for Terraform providers, Style Dictionary transforms, OpenAPI Generator templates, Protobuf compiler plugins, Ansible module dispatch, Helm values cascade, CMake toolchains, Kotlin Multiplatform source sets, React Native bridge, Babel/PostCSS pipelines. Five AI-specific tools surveyed (AgentRuleGen, ClaudeMDEditor, CC-Switch, OpenAgentsControl, AGENTS.md spec). W3C Design Tokens stable spec. NIST AI Agent Standards Initiative.
- **Not Found**: Flutter rendering pipeline details (lower priority). Bazel platform-specific build rules (lower relevance). OpenAgentsControl compatibility layer implementation details (still in development). Performance benchmarks for Style Dictionary at scale (thousands of tokens). Detailed comparison of OpenAPI Generator template engines. Quantitative adoption metrics for AGENTS.md beyond "60k+ projects."