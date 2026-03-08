---
title: ANALYSIS-004-tanstack-intent-deep-dive
type: analysis
permalink: analysis/analysis-004-tanstack-intent-deep-dive
tags:
- analysis
- tanstack-intent
- format-specification
- skill-md
- cli-design
- agent-plugin
- npm-distribution
- staleness-detection
---

# ANALYSIS-004 TanStack Intent Deep Dive

## 1. Objective and Scope

**Objective**: Analyze TanStack Intent (v0.0.12) to document its complete architecture, SKILL.md format, CLI surface, source resolution, state tracking, and unique patterns. Identify what `@acmelabs-15/agent-plugin` should borrow versus what Vercel already covers.

**Scope**: Full source code analysis of `packages/intent/` (11 TypeScript source files, 5 meta-skills, 3 workflow templates, validation scripts). Excludes TanStack ecosystem libraries (Query, Router, Form) as none have published skills yet.

## 2. Context

TanStack Intent takes a fundamentally different approach from Vercel's `skills` CLI. Where Vercel is a "skill installer" (fetch skills from GitHub/GitLab/npm and place them in agent directories), TanStack Intent is a "skill authoring and distribution toolkit" for library maintainers. Skills ship inside npm packages alongside the library code itself.

Repository: <https://github.com/TanStack/intent> (MIT license)
Version analyzed: 0.0.12 (March 2026, 130 commits)
Package: `@tanstack/intent` on npm
Single runtime dependency: `yaml` (^2.7.0)
Build tool: `tsdown` (^0.19.0)

## 3. Approach

**Methodology**: Direct source code analysis via GitHub API and raw file fetching. Read all 11 TypeScript source files in `src/`. Read all 5 meta-skill SKILL.md files in `meta/`. Analyzed 3 GitHub Actions workflow templates. Reviewed validation scripts, package.json, README, CONTRIBUTING.md, and documentation overview.

**Tools Used**: GitHub API (`gh api`), WebFetch (raw source files), Brain MCP (context retrieval).

**Limitations**: No TanStack library has published skills yet (TanStack Query, Router, Form all return 404 for `skills/` directory). Project is at v0.0.12, still pre-1.0. Community adoption data unavailable.

## 4. Data and Analysis

### 4.1 Project Purpose -- Fundamentally Different from Vercel

| Dimension | Vercel `skills` | TanStack `intent` |
|-----------|-----------------|-------------------|
| Primary user | Skill consumer (developer) | Skill author (library maintainer) |
| Distribution | GitHub repos, GitLab, well-known URLs | npm packages (skills travel with code) |
| Versioning | GitHub tree SHA hashing | npm semver (skills version with library) |
| Installation | Copy/symlink to agent directories | Already in `node_modules/` |
| State tracking | Lock files (JSON) | No lock file needed |
| Staleness | Check GitHub tree SHA changes | Compare source doc SHA vs skill metadata |
| Feedback | None | Structured feedback via GitHub Issues |
| Authoring | `init` creates template | 3-phase AI-guided scaffold (domain discovery, tree generation, skill generation) |

### 4.2 The IntentConfig Format

Skills are declared in `package.json` under an `intent` key:

```json
{
  "name": "@tanstack/query",
  "version": "5.0.0",
  "intent": {
    "version": 1,
    "repo": "TanStack/query",
    "docs": "https://tanstack.com/query/latest/docs",
    "requires": ["@tanstack/query-core"]
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | number | Yes | Must be `1`. Protocol version. |
| `repo` | string | Yes | GitHub `owner/repo` shorthand. |
| `docs` | string | Yes | Documentation URL for the library. |
| `requires` | string[] | No | Other intent-enabled packages this depends on. |

Validation in `scanner.ts`: `validateIntentField()` enforces version=1, string repo, string docs. Falls back to `deriveIntentConfig()` which extracts from standard package.json `repository` and `homepage` fields.

### 4.3 SKILL.md Frontmatter Specification

TanStack Intent extends Vercel's basic `name`/`description` frontmatter with additional fields:

```yaml
---
name: tanstack-query/core/live-queries
description: >
  Agent-facing routing description packed with keywords.
type: core
library: @tanstack/query
library_version: "5.0.0"
framework: react
requires:
  - tanstack-query/core
sources:
  - "TanStack/query:docs/guides/queries.md"
  - "TanStack/query:src/queryClient.ts"
metadata:
  version: '1.0'
  category: meta-tooling
  input_artifacts:
    - skills/_artifacts/skill_tree.yaml
  output_artifacts:
    - SKILL.md
  skills:
    - skill-tree-generator
---
```

| Field | Type | Required | Vercel Has? | Description |
|-------|------|----------|-------------|-------------|
| `name` | string | Yes | Yes | Hierarchical: `library/domain/skill-name` |
| `description` | string | Yes | Yes | Dense routing key for agent loading |
| `type` | enum | No | No | `core`, `sub-skill`, `framework`, `lifecycle`, `composition`, `security` |
| `library` | string | No | No | npm package name |
| `library_version` | string | No | No | Target library version |
| `framework` | enum | No | No | `react`, `vue`, `solid`, `svelte`, `angular` |
| `requires` | string[] | No | No | Prerequisite skill names |
| `sources` | string[] | No | No | `Owner/repo:relative-path` format |
| `metadata` | object | No | Yes | Extensible key-value pairs |

Key differences from Vercel:

1. Hierarchical naming: `library/domain/skill-name` vs flat `skill-name`
2. Skill types: 6 types vs none
3. Source tracking: explicit `sources` array linking to upstream docs/code
4. Library versioning: `library_version` field
5. Framework discriminator: allows framework-specific skill variants
6. No `internal` flag (Vercel-specific)

### 4.4 Skill Body Structure

The generate-skill meta-skill prescribes a standard body structure:

1. **Dependency Note** (framework/sub-skills only) -- references parent skill
2. **Setup** -- complete, copy-pasteable initialization code
3. **Core Patterns** -- 2-4 primary usage patterns with complete code blocks
4. **Common Mistakes** -- minimum 3 entries with priority (CRITICAL/HIGH/MEDIUM), wrong/correct code pairs, failure mechanism, and source reference
5. **References** -- links to reference files when skill exceeds 500 lines

Line budget: 500 lines maximum per SKILL.md. Enforced by `validate` command.

### 4.5 Artifacts System

Skills generate 3 artifacts during scaffolding, stored in `skills/_artifacts/`:

| Artifact | Format | Purpose |
|----------|--------|---------|
| `domain_map.yaml` | YAML | Structured domain inventory with skills, failure modes, tensions, gaps |
| `skill_spec.md` | Markdown | Human-readable summary with tables |
| `skill_tree.yaml` | YAML | Hierarchical skill dependency tree |

These artifacts are excluded from npm publish via `"!skills/_artifacts"` in `files` array. They serve as input for the generate-skill meta-skill and are validated during `intent validate`.

### 4.6 CLI Command Surface

| Command | Description | Vercel Equivalent |
|---------|-------------|-------------------|
| `intent list [--json]` | Scan node_modules for intent-enabled packages | `skills list` |
| `intent meta [name]` | List/print meta-skills for authoring | None |
| `intent validate [dir]` | Validate SKILL.md files against format spec | None |
| `intent install` | Print skill-to-task mapping prompt | `skills add` (but different) |
| `intent scaffold` | Print AI-guided scaffolding prompt | `skills init` (but richer) |
| `intent stale [--json]` | Check skills for staleness | `skills check` |
| `intent add-library-bin` | Generate bin/intent.js bridge file | None |
| `intent edit-package-json` | Wire package.json for skill publishing | None |
| `intent setup-github-actions` | Copy CI workflow templates | None |

Two binaries are published:

- `intent` -- full CLI (cli.ts)
- `intent-library` -- lightweight CLI for libraries that bundle intent (intent-library.ts, only `list` and `install`)

### 4.7 Source Resolution -- npm-First

TanStack Intent does NOT clone Git repositories. It scans `node_modules/` directly.

Scanner flow (`scanner.ts`):

1. Detect package manager (pnpm-lock.yaml, bun.lockb, yarn.lock, package-lock.json)
2. Read all package.json files in `node_modules/`
3. Check for `intent` field or derive from `repository`/`homepage` fields
4. Discover `skills/` directory within each package
5. Parse SKILL.md frontmatter for each skill
6. Topologically sort packages by `requires` dependencies
7. Return `ScanResult` with packages, skills, and warnings

Library scanner (`library-scanner.ts`) does the same but from the library's own perspective (for maintainers), walking dependencies that have an `intent` bin entry.

### 4.8 Staleness Detection

Staleness is tracked via source document changes, not content hashing.

`staleness.ts` flow:

1. Read `sync-state.json` from skills directory (stores source SHAs)
2. Fetch current npm version via registry API
3. Classify version drift: major, minor, patch, or null
4. Compare stored source SHAs against current source file SHAs
5. Generate `StalenessReport` per package with per-skill `needsReview` flags

The `skill-staleness-check` meta-skill classifies impact:

- **No impact**: typo/comment/test changes -- skip
- **Version bump only**: metadata change -- update frontmatter
- **Content update**: API/behavior change -- surgical rewrite
- **Breaking change**: API removal -- full rewrite plus document old pattern

### 4.9 Feedback System

A complete feedback loop from consumers to maintainers:

1. **FeedbackPayload**: structured data (skill, package, task, whatWorked, whatFailed, missing, selfCorrections, userRating)
2. **Secret scanning**: 7 regex patterns check for leaked tokens before submission
3. **Submission**: GitHub Issue via `gh issue create` (fallback: file output, stdout)
4. **Meta-feedback**: separate payload for meta-skill effectiveness (domain-discovery, tree-generator, generate-skill, skill-staleness-check)
5. **Frequency control**: `intent.config.json` sets feedback frequency (always, every-N, never)
6. **Privacy**: explicit rules for private repos to strip project-specific details

### 4.10 Meta-Skills (AI-Guided Authoring)

5 meta-skills ship in `meta/` directory. These are SKILL.md files designed to be loaded into an AI agent conversation to guide the scaffolding process.

| Meta-Skill | Purpose | Output |
|------------|---------|--------|
| `domain-discovery` | 5-phase methodology: autonomous scan, maintainer interview, deep read, detail interview, finalize | `domain_map.yaml`, `skill_spec.md` |
| `tree-generator` | Generate hierarchical skill tree from domain map | `skill_tree.yaml` |
| `generate-skill` | Generate SKILL.md files from artifacts + source docs | Individual SKILL.md files |
| `skill-staleness-check` | Evaluate skills when source files change | Updated SKILL.md files, PRs |
| `feedback-collection` | 4-phase feedback: automated signal collection, human interview, document generation, submission | GitHub Issues |

The scaffold command outputs a sequential prompt that chains domain-discovery, tree-generator, and generate-skill with mandatory human review gates between phases.

### 4.11 CI/CD Integration

3 workflow templates in `meta/templates/workflows/`:

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `validate-skills.yml` | PR modifying `skills/**` | Run `intent validate` on changed skill dirs |
| `check-skills.yml` | Release published | Run `intent stale`, open PR for stale skills |
| `notify-playbooks.yml` | (not analyzed) | Notification system |

`setup-github-actions` copies these templates with variable substitution (`{{PACKAGE_NAME}}`, `{{REPO}}`, `{{DOCS_PATH}}`, `{{SRC_PATH}}`).

### 4.12 Packaging for Library Maintainers

`setup.ts` provides 3 commands to prepare libraries for skill distribution:

1. **add-library-bin**: Creates `bin/intent.js` shim that proxies to `@tanstack/intent` CLI
2. **edit-package-json**: Adds `"skills"` and `"bin"` to `files` array, excludes `"!skills/_artifacts"`, wires bin entry
3. **setup-github-actions**: Copies workflow templates with variable substitution

This enables `npx @tanstack/query intent list` after installing TanStack Query.

## 5. Results

### 5.1 Architecture Comparison Matrix

| Feature | Vercel Skills | TanStack Intent | Our Plugin Manager |
|---------|--------------|----------------|--------------------|
| Content types | SKILL.md only | SKILL.md only | SKILL.md, AGENT.md, PROMPT.md, HOOK.md, MCP.md |
| Distribution | Git clone, well-known, npm scan | npm packages (primary) | Git, npm, well-known, registry |
| Versioning | GitHub tree SHA | npm semver | npm semver + content hash |
| State tracking | JSON lock files | None (npm manages) | SQLite |
| Installation | Copy/symlink to agent dirs | Already in node_modules | Copy/symlink to agent dirs |
| Staleness | Tree SHA comparison | Source doc SHA + npm version drift | Both approaches |
| Validation | Minimal (name + description) | Rich (500 line limit, path matching, artifact checks, packaging warnings) | Rich validation |
| Authoring | `init` template | 3-phase AI-guided scaffold | TBD |
| Feedback | None | GitHub Issues with secret scanning | TBD |
| CI/CD | None | 3 workflow templates | TBD |
| Agent support | 40+ agents mapped | 5 agents in types (claude-code, cursor, copilot, codex, other) | 7 target platforms |
| Skill types | None | 6 types (core, sub-skill, framework, lifecycle, composition, security) | TBD |
| Source tracking | None | `sources` array with `Owner/repo:path` | TBD |
| Dependencies | Skill-level: none | Package-level: `requires` array | Plugin-level dependencies |
| Line limit | None | 500 lines per SKILL.md | TBD |

### 5.2 Unique Patterns in TanStack Intent (Not in Vercel)

| Pattern | Description | Value |
|---------|-------------|-------|
| npm-native distribution | Skills travel inside npm packages | High -- eliminates separate install step |
| Source provenance tracking | `sources` array links skills to upstream docs/code | High -- enables staleness detection |
| Hierarchical skill naming | `library/domain/skill-name` | Medium -- organizes large skill sets |
| 6 skill types | core, sub-skill, framework, lifecycle, composition, security | High -- enables type-specific validation |
| Artifact system | domain_map.yaml, skill_spec.md, skill_tree.yaml | Medium -- structured authoring |
| AI-guided scaffolding | 5 meta-skills for scaffolding process | High -- reduces authoring friction |
| Structured feedback loop | Consumer-to-maintainer feedback via GitHub Issues | Medium -- closes feedback loop |
| Secret scanning | 7 regex patterns before feedback submission | Medium -- prevents credential leaks |
| Topological skill sorting | `requires` dependencies topologically sorted | Low -- relevant for ordered loading |
| 500-line budget | Enforced line limit per skill | Medium -- prevents bloat |
| Packaging validation | Checks package.json for correct files/bin/devDependencies | Medium -- catches distribution errors |
| CI workflow templates | Ready-made GitHub Actions for validation and staleness | Medium -- reduces setup friction |
| Cross-model compatibility rules | Explicit guidelines for multi-agent consumption | High -- directly relevant to us |

## 6. Discussion

### 6.1 Two Complementary Models

Vercel and TanStack solve different halves of the skills ecosystem:

- **Vercel**: "I want to install skills written by others into my project"
- **TanStack**: "I want to ship skills alongside my npm library"

Our plugin manager must support both models. A developer might:

1. Install a standalone skill from GitHub (Vercel model)
2. Get skills automatically from npm dependencies (TanStack model)
3. Install an agent or MCP server from a registry (our extension)

### 6.2 Source Provenance is a Strong Pattern

TanStack's `sources` array (`"Owner/repo:docs/path.md"`) enables automated staleness detection without content hashing. When the source document changes, the skill is flagged for review. This is more precise than Vercel's tree SHA approach (which triggers on any file change in the skill directory).

Our format should adopt this pattern. It allows CI pipelines to automatically detect when plugins need updates.

### 6.3 Skill Types Enable Smarter Validation

TanStack's 6 skill types (core, sub-skill, framework, lifecycle, composition, security) allow type-specific validation rules:

- Framework skills must have `requires` (dependency on core skill)
- Security skills use checklist body format
- Sub-skills must have at least one source

Our plugin types (skill, agent, prompt, hook, mcp) serve a similar purpose but at a different level. We could adopt TanStack's sub-typing within our skill type.

### 6.4 The Meta-Skills Pattern is Novel

TanStack ships 5 SKILL.md files designed to be loaded into AI agent conversations to guide the scaffolding process. This is a "skills about creating skills" approach. The domain-discovery meta-skill enforces mandatory human interview gates between autonomous phases.

This pattern could inform how we approach plugin authoring guidance in our system.

### 6.5 Feedback Loop is Missing from Vercel

TanStack's structured feedback (skill, package, task, whatWorked, whatFailed, missing, selfCorrections, userRating) with secret scanning and GitHub Issue submission is a capability Vercel lacks entirely. It closes the loop between consumers and maintainers.

### 6.6 Pre-1.0 Risk

TanStack Intent is at v0.0.12 with 130 commits. No TanStack library has published skills yet (Query, Router, Form all return 404). The format specification exists primarily in the meta-skills, not in a standalone spec document. Adoption is unproven.

## 7. Recommendations

### 7.1 What to BORROW from TanStack Intent

| Priority | Element | Rationale |
|----------|---------|-----------|
| P0 | Source provenance (`sources` array) | Enables automated staleness detection for our plugins |
| P0 | Hierarchical naming (`library/domain/skill-name`) | Essential for organizing large plugin sets across 7 platforms |
| P0 | Skill type discriminator (`type` field) | Enables type-specific validation; maps to our plugin types |
| P1 | Cross-model compatibility rules | We target 7 platforms; these rules prevent agent-specific markup |
| P1 | 500-line budget enforcement | Prevents context window bloat; enforces conciseness |
| P1 | Framework discriminator (`framework` field) | Useful for framework-specific plugin variants |
| P1 | `requires` dependency chain | Enables prerequisite ordering for plugin loading |
| P2 | npm-native scanning model | Support discovering plugins from node_modules |
| P2 | Structured feedback payload format | Close consumer-to-author feedback loop |
| P2 | Secret scanning patterns | Apply to any user-submitted content |
| P2 | CI workflow templates | Provide ready-made validation workflows |
| P3 | Meta-skills authoring pattern | Consider for plugin authoring guidance |
| P3 | Artifact system (domain_map, skill_tree) | Complex; defer unless authoring tooling is prioritized |

### 7.2 What NOT to Borrow

| Element | Reason |
|---------|--------|
| npm-only distribution | We must support Git, registries, and local sources too |
| No lock file | We need state tracking for multi-source, multi-platform installs |
| `intent` key in package.json | Our plugins are standalone; we should not require library package.json changes |
| Single content type (SKILL.md only) | We support 5 plugin types |
| Library-centric worldview | TanStack assumes you are a library maintainer; we serve plugin consumers too |

### 7.3 Merged Format Proposal

Combining the strongest patterns from both Vercel and TanStack:

```yaml
---
name: my-org/auth-flow
description: >
  Dense agent-facing routing description with keywords.
type: skill
subtype: core
version: "1.0.0"
author: my-org
license: MIT
platforms: [claude-code, cursor, copilot]
framework: react
requires:
  - my-org/auth-core
sources:
  - "my-org/my-lib:docs/auth.md"
  - "my-org/my-lib:src/auth.ts"
dependencies:
  - my-org/some-other-plugin
conflicts:
  - old-auth-plugin
config:
  timeout: 30
  retries: 3
metadata:
  internal: false
  category: authentication
---
```

This format:

- Keeps Vercel's `name`/`description` as required fields
- Adds TanStack's `type`, `sources`, `framework`, `requires`
- Adds our extensions: `platforms`, `dependencies`, `conflicts`, `config`
- Uses `subtype` for TanStack's skill type taxonomy within our broader `type` system

## 8. Conclusion

**Verdict**: Proceed -- incorporate TanStack patterns into our format specification.

**Confidence**: High

**Rationale**: TanStack Intent solves a complementary problem to Vercel skills. Its source provenance, hierarchical naming, skill types, and cross-model compatibility rules fill gaps in Vercel's approach. The combination of both systems' patterns gives us a strong foundation for our multi-platform, multi-type plugin format.

### User Impact

- **What changes for you**: Our plugin format gains source tracking, hierarchical naming, type-specific validation, and cross-platform compatibility rules. Plugins become more discoverable and maintainable.
- **Effort required**: P0 items (source provenance, hierarchical naming, type discriminator) should be incorporated into the format spec immediately. P1 items during implementation. P2-P3 items are backlog candidates.
- **Risk if ignored**: Without source provenance, we cannot detect stale plugins. Without hierarchical naming, large plugin sets become unmanageable. Without cross-model rules, plugins break on specific platforms.

## Observations

- [fact] TanStack Intent (v0.0.12) has 1 runtime dependency (yaml ^2.7.0) vs Vercel's 0 runtime deps (all bundled); both minimize install footprint #architecture
- [fact] Skills ship inside npm packages under a `skills/` directory; no separate installation step needed; npm semver handles versioning #distribution
- [fact] The `intent` key in package.json requires exactly 3 fields: version (must be 1), repo (string), docs (string); optional requires array #format
- [fact] 6 skill types defined: core, sub-skill, framework, lifecycle, composition, security; framework type requires `requires` field #format
- [fact] Source provenance tracked via `sources` array in frontmatter using `Owner/repo:relative-path` format; enables automated staleness detection #staleness
- [fact] 500-line limit per SKILL.md enforced by validate command; excess moves to `references/` directory #validation
- [fact] 5 meta-skills ship in `meta/` directory for AI-guided authoring: domain-discovery, tree-generator, generate-skill, skill-staleness-check, feedback-collection #authoring
- [decision] Borrow source provenance, hierarchical naming, skill types, cross-model rules from TanStack; do not borrow npm-only distribution or no-lock-file approach #format-design
- [insight] Vercel and TanStack solve complementary problems: Vercel is a skill installer, TanStack is a skill authoring/distribution toolkit; our system must support both models #architecture
- [insight] The meta-skills pattern (skills about creating skills) with mandatory human interview gates is a novel approach to reducing authoring friction that could inform our plugin authoring story #authoring
- [technique] Secret scanning with 7 regex patterns (GitHub tokens, Stripe keys, AWS access keys, PEM keys, JWTs, Bearer tokens, generic secrets) applied before feedback submission #security
- [risk] TanStack Intent is pre-1.0 (v0.0.12); no TanStack library has published skills yet; format specification exists only in meta-skills, not standalone spec #adoption

## Relations

- relates_to [[ANALYSIS-003-vercel-skills-format-deep-dive]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]
- extends [[ANALYSIS-001-agent-plugin-foundation-and-vision]]

## 9. Appendices

### 9.1 Sources Consulted

- TanStack Intent GitHub: <https://github.com/TanStack/intent> (all source files in packages/intent/src/)
- Package README: <https://github.com/TanStack/intent/blob/main/packages/intent/README.md>
- Root README: <https://github.com/TanStack/intent/blob/main/README.md>
- Documentation overview: <https://github.com/TanStack/intent/blob/main/docs/overview.md>
- Contributing guide: <https://github.com/TanStack/intent/blob/main/CONTRIBUTING.md>
- Meta-skills: 5 SKILL.md files in packages/intent/meta/
- Workflow templates: 3 YAML files in packages/intent/meta/templates/workflows/
- Validation script: scripts/validate-skills.ts
- Package.json: packages/intent/package.json

### 9.2 File Inventory Analyzed

**Source files (11)**: cli.ts, display.ts, feedback.ts, index.ts, intent-library.ts, library-scanner.ts, scanner.ts, setup.ts, staleness.ts, types.ts, utils.ts

**Meta-skills (5)**: domain-discovery/SKILL.md, feedback-collection/SKILL.md, generate-skill/SKILL.md, skill-staleness-check/SKILL.md, tree-generator/SKILL.md

**Workflow templates (3)**: check-skills.yml, notify-playbooks.yml, validate-skills.yml

### 9.3 Data Transparency

- **Found**: Complete source code for all 11 modules. All 5 meta-skill SKILL.md files. All 3 workflow templates. Package.json with full configuration. README and documentation. Contributing guide. Validation script. Complete repo file tree (93 files).
- **Not Found**: No published skills in any TanStack library (Query, Router, Form all 404). No npm download counts for @tanstack/intent. No documentation site content beyond overview.md. notify-playbooks.yml content not analyzed. No test file content analyzed (8 test files exist).
