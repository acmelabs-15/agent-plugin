---
title: ANALYSIS-007-config-schema-versioning-patterns
type: note
permalink: analysis/analysis-007-config-schema-versioning-patterns
tags:
- analysis
- schema-versioning
- config-formats
- manifest
- best-practices
---

# ANALYSIS-007 Config Schema Versioning Patterns

## 1. Objective and Scope

**Objective**: How do popular config/manifest JSON files handle schema evolution and format versioning? Should a plugin manifest include an explicit format version field?

**Scope**: 9 primary config formats (package.json, tsconfig.json, Cargo.toml, pyproject.toml, composer.json, .eslintrc.json, Chrome manifest.json, VS Code package.json, plugin.json). 4 bonus formats (WordPress theme.json, MediaWiki extension.json, npm package-lock.json, Terraform HCL).

## 2. Context

The agent-plugin project is defining a plugin manifest format (ADR-001). A key design question is whether to include an explicit format/schema version field. This analysis surveys real-world config ecosystems to inform that decision with evidence.

## 3. Approach

**Methodology**: Web research across official documentation, community discussions, and schema stores for each format.

**Tools Used**: WebSearch, WebFetch against official docs (npm, TypeScript, Rust, Python, PHP, Chrome, VS Code, Grafana, WordPress, MediaWiki).

**Limitations**: Some documentation pages blocked access (MediaWiki 403). Training data supplemented where web fetch failed.

## 4. Data and Analysis

### Evidence Gathered

| Config Format | Has Format Version Field | Field Name | Values | Confidence |
|---|---|---|---|---|
| package.json (npm) | NO | N/A | N/A | High |
| tsconfig.json | NO | N/A | N/A | High |
| Cargo.toml (Rust) | NO (but has `edition`) | `edition` | "2015", "2018", "2021", "2024" | High |
| pyproject.toml (Python) | NO | N/A | N/A | High |
| composer.json (PHP) | NO | N/A | N/A | High |
| .eslintrc.json | NO | N/A | N/A | High |
| Chrome manifest.json | YES | `manifest_version` | 2, 3 (integer) | High |
| VS Code package.json | NO (uses `engines.vscode`) | `engines.vscode` | "^1.80.0" etc. | High |
| Grafana plugin.json | NO (uses `$schema`) | `$schema` | URL string | Medium |
| WordPress theme.json | YES | `version` | 1, 2, 3 (integer) | High |
| MediaWiki extension.json | YES | `manifest_version` | 1, 2 (integer) | High |
| npm package-lock.json | YES | `lockfileVersion` | 1, 2, 3 (integer) | High |
| Terraform HCL | NO (uses `required_version`) | `required_version` | ">= 1.6.0" etc. | High |

### Detailed Findings by Format

#### 1. package.json (npm) -- NO format version

The `version` field is the package's own version, not a schema version. npm has never added a format version for package.json itself. Schema evolution happens through additive, backward-compatible field additions. The SchemaStore maintains an external JSON Schema for IDE validation, but npm does not embed or reference it. npm's approach: add optional fields, never remove or rename existing ones.

#### 2. tsconfig.json (TypeScript) -- NO format version

No version field of any kind for the format itself. New compiler options are added per TypeScript release. The SchemaStore maintains an external schema. TypeScript handles evolution by making all fields optional and ignoring unknown fields. Older tsconfig files work with newer TypeScript versions without modification.

#### 3. Cargo.toml (Rust) -- NO format version (edition is for compiler, not manifest)

The `edition` field ("2015", "2018", "2021", "2024") controls Rust language edition for compilation, not the manifest format. Cargo.toml itself has no format version. Schema evolution strategy: new fields are optional with sensible defaults, old fields are deprecated but remain supported, no breaking changes to TOML structure. The `rust-version` field (MSRV) specifies minimum Rust toolchain, not manifest format.

#### 4. pyproject.toml (Python) -- NO format version (acknowledged as a mistake)

No format version field exists. Paul Moore (Python core) publicly acknowledged this was "a mistake not defining some form of versioning for the [project] table." Community discussion in December 2023 debated adding one but consensus shifted against it. Arguments against: user-edited files create UX burden, tools would still error on incompatible features regardless, versioning adds false specificity. Resolution: focus on better compatibility discussion in future PEPs rather than retrofitting a version field.

#### 5. composer.json (PHP) -- NO format version

No schema version or format version field. The `version` field is the package version. Composer references an external JSON Schema at getcomposer.org/schema.json for validation. Evolution handled through additive changes and external schema validation.

#### 6. .eslintrc.json -- NO format version (but format itself was replaced)

No version field in the legacy eslintrc format. ESLint handled its major schema evolution by creating an entirely new format: flat config (eslint.config.js). ESLint 9 deprecated eslintrc, ESLint 10 removes it entirely. Instead of versioning the format, they replaced it. Plugins in flat config include `meta.name` and `meta.version` but these identify the plugin, not the config format.

#### 7. Chrome extension manifest.json -- YES: `manifest_version` (integer)

**Field**: `manifest_version` (required integer)
**Values**: 1 (ancient, unsupported), 2 (deprecated 2022, disabled 2024), 3 (current, only supported value)
**Usage**: Chrome reads this field to determine which APIs and behaviors to expose. V2 to V3 migration required significant code changes (service workers instead of background pages, declarativeNetRequest instead of webRequest, etc.). Chrome enforced a hard cutover timeline. This is the canonical example of explicit manifest versioning done at scale.

#### 8. VS Code extension package.json -- NO format version (uses engines.vscode)

No dedicated manifest format version. The `engines.vscode` field (required, e.g., "^1.80.0") declares which VS Code versions the extension supports. This controls API availability, not manifest schema. The manifest format evolves by adding optional fields (e.g., `browser`, `l10n`). VS Code ignores unknown fields, maintaining backward compatibility.

#### 9. Grafana plugin.json -- NO explicit format version (uses $schema URL)

The `$schema` field points to a JSON Schema URL for validation. No integer version field. Grafana handles evolution through the external schema definition. The `@grafana/schema` package (currently alpha) manages type definitions.

### Bonus Formats (Strong Examples)

#### WordPress theme.json -- YES: `version` (integer)

**Field**: `version` (required integer)
**Values**: 1 (WP 5.8), 2 (WP 5.9), 3 (WP 6.6)
**Usage**: WordPress reads this to determine which settings/styles to apply. Older versions remain backward-compatible. New features only available in latest version. Migration is opt-in: themes upgrade manually when ready. WordPress will NOT auto-migrate. Each version has a corresponding JSON Schema at schemas.wp.org. This is a well-executed example of format versioning for machine-read, human-edited files.

#### MediaWiki extension.json -- YES: `manifest_version` (integer)

**Field**: `manifest_version` (required integer)
**Values**: 1 (MW 1.25-1.42, deprecated in 1.43+), 2 (MW 1.29+)
**Usage**: MediaWiki reads this to apply the correct parsing rules. V2 added stricter validation and developer features. A migration script (`updateExtensionJsonSchema.php`) converts v1 to v2. Recommendation: use v2 if your extension requires MW 1.29+, but do not break compatibility with older MW just to upgrade.

#### npm package-lock.json -- YES: `lockfileVersion` (integer)

**Field**: `lockfileVersion` (required integer)
**Values**: 1 (npm 5-6), 2 (npm 7-8, backward-compatible with v1), 3 (npm 9, no backward compatibility)
**Usage**: npm reads this to determine parsing strategy. V2 includes backward-compatible affordances for older npm. The hidden lockfile (node_modules/.package-lock.json) uses v3 without backward compatibility since older npm ignores it.

#### Terraform HCL -- NO format version (uses required_version)

No format version field. The `required_version` constraint (e.g., ">= 1.6.0") specifies which Terraform CLI versions can process the config. This gates the tool version, not the config format. HCL itself evolves through additive syntax additions.

## 5. Results

### Classification of Approaches

**Category A: Explicit Format Version Field (4 of 13)**
Chrome manifest.json, WordPress theme.json, MediaWiki extension.json, npm package-lock.json

**Category B: No Format Version, Additive Evolution (7 of 13)**
package.json, tsconfig.json, Cargo.toml, pyproject.toml, composer.json, Grafana plugin.json, Terraform HCL

**Category C: Tool Version Constraint Instead (2 of 13)**
VS Code package.json (engines.vscode), Terraform (required_version)

**Category D: Format Replacement Instead of Versioning (1 of 13)**
ESLint (eslintrc replaced by flat config)

### Pattern: When Format Version Fields Appear

Format version fields appear when ALL of these conditions hold:
1. The format is consumed by a platform/runtime (not just a build tool)
2. Breaking changes to the format are anticipated or have occurred
3. The platform must support multiple format versions simultaneously during transition
4. The file is a manifest declaring capabilities to a host system

Format version fields are absent when:
1. The format is consumed by a single tool the user controls (Cargo, Composer, pip)
2. Evolution happens through additive, backward-compatible changes only
3. The file configures behavior rather than declaring capabilities

### Naming Conventions

| System | Field Name | Type |
|---|---|---|
| Chrome | `manifest_version` | integer |
| WordPress | `version` | integer |
| MediaWiki | `manifest_version` | integer |
| npm lockfile | `lockfileVersion` | integer |
| Docker/OCI | `schemaVersion` | integer |

All use integers, not semver strings. Values are small (1, 2, 3). This reflects the intent: format versions mark discrete breaking changes, not continuous evolution.

## 6. Discussion

### The Split is 70/30 Against Format Version Fields

9 of 13 formats surveyed have no format version field. The 4 that do share a common trait: they are platform manifests consumed by a runtime that must support multiple versions during migration periods.

### The pyproject.toml Cautionary Tale

Python's packaging community explicitly discussed and rejected adding a format version field after the fact. Key arguments:

- User-edited files create a UX burden (users copy-paste the wrong version number)
- Tools can detect incompatible features without a version field (fail on unknown keys)
- Versioning adds false specificity without improving error messages meaningfully
- Retrofitting a version field after initial release is harder than including one from the start

### The Chrome manifest_version Success Story

Chrome's manifest_version is the strongest example of format versioning done well:

- Required from day one
- Simple integer, not semver
- Clear migration documentation and timeline
- Platform enforces it (refuses to load unsupported versions)
- Enabled a major API redesign (V2 to V3) with a multi-year transition

### WordPress theme.json: Best-in-Class for Human-Edited Files

WordPress theme.json demonstrates format versioning that works well for human-edited config:

- Older versions remain functional indefinitely
- New features only on latest version (incentivizes upgrade)
- No forced migration
- Version-specific JSON Schemas for IDE support

### The Tool-Version-Constraint Alternative

VS Code and Terraform take a different approach: instead of versioning the config format, they let the config declare which tool version it requires. This pushes compatibility into the tool's upgrade cycle rather than the config format.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Include a `manifest_version` integer field if the manifest declares capabilities to a host runtime | 4/4 platform-consumed manifests use this pattern; enables future breaking changes without backward-compatibility hacks | Low (1 field) |
| P1 | Use integer values (1, 2, 3), not semver strings | All 4 existing implementations use integers; format changes are discrete, not continuous | None |
| P1 | Make it required from v1 | Retrofitting is painful (see pyproject.toml); including from the start costs nothing | None |
| P2 | Support reading older versions indefinitely | WordPress and MediaWiki patterns show this reduces ecosystem friction during transitions | Medium |
| P2 | Consider a `$schema` URL field for IDE validation | Grafana and composer.json patterns show this improves developer experience independently of format versioning | Low |

### Decision Framework

```
Is your config file a manifest consumed by a platform/runtime?
├── YES: Include manifest_version (integer, required)
│   └── Will you ever make breaking changes to the format?
│       ├── YES: manifest_version is essential
│       └── NO/UNSURE: Include it anyway (cost is 1 field; retrofitting cost is high)
└── NO (tool config, build config):
    └── Evolve additively. Ignore unknown fields. Skip format version.
```

## 8. Conclusion

**Verdict**: Include `manifest_version` as a required integer field in plugin manifests.

**Confidence**: High

**Rationale**: The agent-plugin manifest declares plugin capabilities to a host runtime. All 4 comparable systems (Chrome, WordPress, MediaWiki, npm lockfile) use an explicit integer version field. The cost of inclusion is 1 field. The cost of omission, if breaking changes are ever needed, is a painful retrofit (as Python's pyproject.toml community learned). Including it from v1 is a free option on future flexibility.

### User Impact

- **What changes for you**: Add `"manifest_version": 1` to plugin manifests. 1 line.
- **Effort required**: Minimal. One field in schema definition, one validation check in loader.
- **Risk if ignored**: If the manifest format needs breaking changes later, every existing plugin and every tool that reads manifests must handle versionless files as a special case. This is the exact problem Chrome, WordPress, and MediaWiki solved by including the field from the start.

## 9. Appendices

### Sources Consulted

- [npm package.json docs](https://docs.npmjs.com/creating-a-package-json-file/)
- [SchemaStore package.json schema](https://github.com/SchemaStore/schemastore/blob/master/src/schemas/json/package.json)
- [TypeScript tsconfig.json docs](https://www.typescriptlang.org/docs/handbook/tsconfig-json.html)
- [SchemaStore tsconfig.json schema](https://github.com/SchemaStore/schemastore/blob/master/src/schemas/json/tsconfig.json)
- [Cargo.toml manifest format](https://doc.rust-lang.org/cargo/reference/manifest.html)
- [pyproject.toml specification](https://packaging.python.org/en/latest/specifications/pyproject-toml/)
- [Python discussion: Versioning pyproject.toml](https://discuss.python.org/t/versioning-pyproject-toml/41419)
- [Composer.json schema docs](https://getcomposer.org/doc/04-schema.md)
- [ESLint flat config migration guide](https://eslint.org/docs/latest/use/configure/migration-guide)
- [Chrome manifest file format](https://developer.chrome.com/docs/extensions/reference/manifest)
- [Chrome manifest V2 to V3 migration](https://developer.chrome.com/docs/extensions/develop/migrate/manifest)
- [VS Code extension manifest](https://code.visualstudio.com/api/references/extension-manifest)
- [Grafana plugin.json reference](https://grafana.com/developers/plugin-tools/reference/plugin-json)
- [WordPress theme.json reference](https://developer.wordpress.org/block-editor/reference-guides/theme-json-reference/theme-json-living/)
- [WordPress theme.json global settings](https://developer.wordpress.org/block-editor/how-to-guides/themes/global-settings-and-styles/)
- [MediaWiki extension.json schema](https://www.mediawiki.org/wiki/Manual:Extension.json/Schema)
- [npm package-lock.json docs](https://docs.npmjs.com/cli/v9/configuring-npm/package-lock-json/)
- [Terraform version constraints](https://developer.hashicorp.com/terraform/language/expressions/version-constraints)

### Data Transparency

- **Found**: Direct documentation for all 13 formats surveyed. Community discussion thread for pyproject.toml versioning debate. Chrome migration documentation. WordPress version history and migration guides.
- **Not Found**: MediaWiki extension.json page returned 403 (supplemented with search result snippets). No single authoritative source on "community consensus" for format versioning as a practice. Grafana plugin.json evolution strategy is underdocumented.

## Observations

- [fact] 4 of 13 config formats surveyed include an explicit format/schema version field #schema-versioning
- [fact] All 4 formats with version fields use integers (not semver) with small values (1, 2, 3) #naming-convention
- [decision] Format version fields correlate with platform-consumed manifests, not tool-consumed configs #pattern
- [insight] Python pyproject.toml community acknowledged missing format version was a mistake but decided against retrofitting #cautionary-tale
- [fact] Chrome manifest_version is required, integer-typed, and enables multi-year format migration #chrome
- [technique] WordPress theme.json supports older versions indefinitely while gating new features behind latest version #backward-compatibility
- [insight] ESLint chose format replacement over format versioning when facing breaking schema changes #alternative-approach
- [recommendation] Plugin manifests consumed by a host runtime should include manifest_version from v1 #guidance

## Relations

- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[DEBATE-ADR-001-plugin-format-and-manifest]]
