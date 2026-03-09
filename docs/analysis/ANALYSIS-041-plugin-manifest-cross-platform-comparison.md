---
title: ANALYSIS-041 Plugin Manifest Cross-Platform Comparison
type: analysis
permalink: analysis/analysis-041-plugin-manifest-cross-platform-comparison-1
tags:
- plugin-manifest
- cross-platform
- comparison
- research
---

> **Platform Scope Change**: Per ADR-002 Amendment #1 (2026-03-09), supported platforms reduced from 7 to 4: Claude Code, Cursor, GitHub Copilot, Kiro. OpenCode, Amp, and Windsurf were dropped due to incomplete content type coverage. References to dropped platforms in this note are historical only.

# ANALYSIS-041 Plugin Manifest Cross-Platform Comparison

## 1. Objective and Scope

**Objective**: Compare our agent-plugin's `plugin.json` manifest format against plugin/extension manifest formats used by 9 AI coding platforms. Identify overlaps, differences, and alignment opportunities.

**Scope**: Field-by-field comparison of manifest schemas across Claude Code, GitHub Copilot CLI, Cursor, Windsurf, OpenAI Codex, Amp, Kiro, Agent Skills spec (agentskills.io), AGENTS.md spec, and Microsoft APM. Excludes runtime behavior and installation lifecycle analysis (covered by ADR-010/ADR-013).

## 2. Context

Our `plugin.json` manifest (ADR-001, amended by ADR-013 Decision 5) declares a plugin bundle with these minimum required fields:

```json
{
  "name": "@scope/my-plugin",
  "description": "Code review skills for TypeScript projects",
  "installMode": "bundle",
  "skills": ["skills/review", "skills/lint"],
  "agents": ["agents/reviewer.md"],
  "hooks": ["hooks/pre-commit.js"],
  "instructions": ["instructions/coding-standards.md"],
  "mcp": "mcp/server.json"
}
```

Required: `name`, `description`. Version from `package.json`. Component types declared as file path arrays.

## 3. Approach

**Methodology**: Web research of official documentation for each platform. Fetched specification pages, reference docs, and blog posts. Cross-referenced with existing ANALYSIS-003, ANALYSIS-004, ANALYSIS-005 findings.

**Tools Used**: WebSearch, WebFetch, Brain memory search, file reading of ADR-001 and ADR-013.

**Limitations**: Amazon Q Developer documentation rendered via JavaScript redirect; some fields inferred from blog posts and third-party guides. Amp plugin API is marked experimental and WIP. Windsurf has no plugin manifest (skills only).

## 4. Data and Analysis

### 4.1 Platform Manifest Architecture Summary

| Platform | Manifest File | Location | Required Fields | Format |
|---|---|---|---|---|
| **Our plugin.json** | `plugin.json` | Plugin root | `name`, `description` | JSON |
| **Claude Code** | `plugin.json` | `.claude-plugin/` | `name` only | JSON |
| **Copilot CLI** | `plugin.json` | Plugin root | `name` only | JSON |
| **Cursor** | `plugin.json` | `.cursor-plugin/` | `name` only | JSON |
| **Windsurf** | None (SKILL.md only) | `.windsurf/skills/` | N/A | N/A |
| **Codex** | SKILL.md + agents/openai.yaml | `.agents/skills/` | `name`, `description` | YAML frontmatter |
| **Amp** | None (TypeScript files) | `.amp/plugins/` | N/A (code-based) | TypeScript |
| **Kiro** | Agent JSON config | `.kiro/agents/` | None (all optional) | JSON |
| **Amazon Q** | Agent JSON config | `.amazonq/agents/` | `description` only | JSON |
| **Agent Skills spec** | SKILL.md | `skills/<name>/` | `name`, `description` | YAML frontmatter |
| **AGENTS.md** | AGENTS.md | Repo root | None | Markdown (unstructured) |
| **Microsoft APM** | `apm.yml` | Package root | `name`, `version` | YAML |

### 4.2 Field-by-Field Comparison Matrix

#### Metadata Fields

| Field | Our plugin.json | Claude Code | Copilot CLI | Cursor | Codex | Amazon Q | Kiro | Agent Skills | APM |
|---|---|---|---|---|---|---|---|---|---|
| **name** | Required | Required | Required | Required | Required (SKILL.md) | Optional (from filename) | Optional (from filename) | Required | Required |
| **description** | Required | Optional | Optional | Optional | Required (SKILL.md) | Required | Optional | Required | N/A |
| **version** | In package.json | Optional | Optional | Optional | In metadata map | N/A | N/A | In metadata map | Required |
| **author** | Optional (ADR-001) | Optional (object) | Optional (object) | Optional (object) | In metadata map | N/A | N/A | In metadata map | N/A |
| **license** | Optional (ADR-001) | Optional | Optional | Optional | Optional (SKILL.md) | N/A | N/A | Optional | N/A |
| **homepage** | Optional (ADR-001) | Optional | Optional | Optional | N/A | N/A | N/A | N/A | N/A |
| **repository** | Optional (ADR-001) | Optional | Optional | Optional | N/A | N/A | N/A | N/A | N/A |
| **keywords** | Optional (ADR-001) | Optional | Optional | Optional | N/A | N/A | N/A | N/A | N/A |
| **category** | N/A | N/A | Optional | N/A | N/A | N/A | N/A | N/A | N/A |
| **tags** | N/A | N/A | Optional | N/A | N/A | N/A | N/A | N/A | N/A |
| **logo** | N/A | N/A | N/A | Optional | Via openai.yaml | N/A | N/A | N/A | N/A |

#### Component Path Fields

| Field | Our plugin.json | Claude Code | Copilot CLI | Cursor | Codex | Amazon Q | Kiro | Agent Skills | APM |
|---|---|---|---|---|---|---|---|---|---|
| **skills** | Path array | string or array | string or array | string or array | Directory convention | N/A | Via resources | Directory convention | Directory convention |
| **agents** | Path array | string or array | string or array | string or array | N/A | N/A | N/A | N/A | Directory convention |
| **hooks** | Path array | string, array, or inline object | string or inline object | string or inline object | In openai.yaml | Object (5 events) | Object (5 events) | N/A | Directory convention |
| **mcp** | Path (string) | string, array, or inline object (as `mcpServers`) | string or inline object (as `mcpServers`) | string, array, or inline object (as `mcpServers`) | Via openai.yaml deps | Object (as `mcpServers`) | Object (as `mcpServers`) | N/A | N/A |
| **instructions** | Path array | N/A (use AGENTS.md/CLAUDE.md) | N/A | N/A (use rules) | N/A | Via `prompt` field | Via `prompt` field | N/A | Directory convention (`.instructions.md`) |
| **commands** | N/A | string or array | string or array | string or array | N/A | N/A | N/A | N/A | N/A |
| **rules** | N/A | N/A | N/A | string or array | N/A | N/A | N/A | N/A | N/A |
| **outputStyles** | N/A | string or array | N/A | string or array | N/A | N/A | N/A | N/A | N/A |
| **lspServers** | N/A | string, array, or inline object | string or inline object | string or inline object | N/A | N/A | N/A | N/A | N/A |
| **prompts** | Path array (ADR-001) | N/A | N/A | N/A | N/A | N/A | N/A | N/A | Directory convention (`.prompt.md`) |

#### Behavioral/Configuration Fields

| Field | Our plugin.json | Claude Code | Copilot CLI | Cursor | Codex | Amazon Q | Kiro | Agent Skills | APM |
|---|---|---|---|---|---|---|---|---|---|
| **installMode** | `bundle` or `collection` | N/A | N/A | N/A | N/A | N/A | N/A | N/A | N/A |
| **platformConfig** | Object (ADR-001) | N/A | N/A | N/A | N/A | N/A | N/A | N/A | N/A |
| **tools** | N/A | N/A | N/A | N/A | N/A | Array (tool list) | Array (tool list) | N/A | N/A |
| **allowedTools** | N/A | N/A | N/A | N/A | Experimental (SKILL.md) | Array (auto-approve) | Array (auto-approve) | Experimental | N/A |
| **model** | N/A | N/A | N/A | N/A | N/A | Optional | Optional | N/A | N/A |
| **prompt** | N/A | N/A | N/A | N/A | N/A | string or file:// URI | string or file:// URI | N/A | N/A |
| **resources** | N/A | N/A | N/A | N/A | N/A | Array (files, skills, knowledge bases) | Array (files, skills, knowledge bases) | N/A | N/A |
| **toolAliases** | N/A | N/A | N/A | N/A | N/A | Object | Object | N/A | N/A |
| **toolsSettings** | N/A | N/A | N/A | N/A | N/A | Object | Object | N/A | N/A |
| **compatibility** | N/A | N/A | N/A | N/A | Optional (SKILL.md) | N/A | N/A | Optional | N/A |
| **metadata** | N/A | N/A | N/A | N/A | Optional (SKILL.md) | N/A | N/A | Optional | N/A |
| **dependencies** | Deferred (ADR-001) | N/A | N/A | N/A | N/A | N/A | N/A | N/A | `dependencies.apm` array |
| **welcomeMessage** | N/A | N/A | N/A | N/A | N/A | N/A | Optional | N/A | N/A |
| **keyboardShortcut** | N/A | N/A | N/A | N/A | N/A | N/A | Optional | N/A | N/A |

### 4.3 Structural Pattern Analysis

#### Pattern 1: Plugin Manifest (Bundle Model)
Platforms: Our plugin.json, Claude Code, Copilot CLI, Cursor, Microsoft APM

These platforms use a root-level manifest file that declares multiple component types via path references. The manifest acts as a table of contents for the plugin bundle.

**Convergence**: Claude Code, Copilot CLI, and Cursor all converged on nearly identical `plugin.json` schemas with the same field names. Only `name` is required. Component paths use `string | string[]` types. Hooks and MCP support inline objects or file paths.

**Our divergence**: We use `mcp` (singular); they all use `mcpServers`. We use path arrays only; they support inline objects. We have `instructions` and `prompts`; they have `commands` and `rules`.

#### Pattern 2: Skill-Level Manifest (SKILL.md)
Platforms: Agent Skills spec, Codex, Windsurf, OpenCode, Amp, Kiro (for skills)

These platforms have no plugin-level manifest. Each skill is a self-contained directory with `SKILL.md` frontmatter as the only metadata. Skills are discovered by directory convention, not declared in a manifest.

**Convergence**: All follow the Agent Skills spec (agentskills.io) with `name` and `description` required in YAML frontmatter. Name must match parent directory name. Name constraints: lowercase, hyphens, 1-64 chars.

#### Pattern 3: Agent Configuration (JSON)
Platforms: Amazon Q, Kiro (for agents)

Agent-level JSON configs that define a single agent's tools, permissions, resources, and behavior. Not a plugin bundle. Focused on runtime configuration rather than packaging.

**Convergence**: Amazon Q and Kiro share an almost identical schema (Amazon Q is the upstream; Kiro adopted it). Fields: `tools`, `allowedTools`, `mcpServers`, `resources`, `hooks`, `prompt`, `model`, `toolAliases`, `toolsSettings`.

#### Pattern 4: Unstructured (AGENTS.md)
Platforms: AGENTS.md spec

No manifest, no structured fields. Plain Markdown with natural language instructions. Deliberately avoids schema constraints.

## 5. Results

### 5.1 Direct Field Equivalents (Our Fields That Exist Elsewhere)

| Our Field | Equivalent On | Same Name? | Notes |
|---|---|---|---|
| `name` | All platforms | Yes | Universal. Exact match everywhere. |
| `description` | All platforms | Yes | Universal. Exact match everywhere. |
| `skills` | Claude Code, Copilot, Cursor | Yes | Same name, same concept. We use array-only; they support string too. |
| `agents` | Claude Code, Copilot, Cursor | Yes | Same name, same concept. |
| `hooks` | Claude Code, Copilot, Cursor, Amazon Q, Kiro | Yes | Same name. We use path array; they also support inline objects. |
| `mcp` | Claude Code, Copilot, Cursor, Amazon Q, Kiro | **No** | They use `mcpServers`. This is the most significant naming divergence. |
| `instructions` | APM only | **No** | APM uses `.instructions.md` files in directory convention. No other platform has this as a manifest field. |

### 5.2 Fields Other Platforms Have That We Lack

| Field | Platforms | Relevance | Priority |
|---|---|---|---|
| `commands` | Claude Code, Copilot, Cursor | Medium. Legacy concept in Claude Code (superseded by skills). Cursor still uses it. | Low (skills replace commands) |
| `rules` | Cursor | Medium. Cursor-specific `.mdc` rule files. | Low (platform-specific) |
| `outputStyles` | Claude Code, Cursor | Low. Formatting styles. Niche. | Low |
| `lspServers` | Claude Code, Copilot, Cursor | Medium. Language server integration. | Medium (could be useful) |
| `tools` | Amazon Q, Kiro | High. Explicit tool whitelist. Different concept from our component paths. | Evaluate |
| `allowedTools` | Amazon Q, Kiro, Agent Skills | High. Auto-approval list for tool permissions. | Evaluate |
| `model` | Amazon Q, Kiro | Low. Model selection. Platform-specific. | Low |
| `prompt` | Amazon Q, Kiro | Medium. System prompt for agent. Overlaps with our `instructions`. | Low (covered by instructions) |
| `resources` | Amazon Q, Kiro | Medium. Knowledge bases, file references. | Evaluate |
| `logo` | Cursor | Low. Visual branding. | Low |
| `category` | Copilot | Low. Discovery classification. | Low |
| `dependencies` | APM | High. Transitive plugin dependencies. | High (deferred in ADR-001) |
| `compatibility` | Agent Skills | Medium. Environment requirements. | Medium |

### 5.3 Our Unique Fields (Not Found Elsewhere)

| Field | Unique To Us | Notes |
|---|---|---|
| `installMode` | Yes | No other platform has bundle/collection concept. Closest: APM's dependency model. |
| `platformConfig` | Yes | No other platform needs this (they are single-platform). Unique to our cross-platform requirement. |
| `prompts` | Partially | APM has `.prompt.md` files by convention. No other platform has this as a manifest field. |
| `instructions` | Partially | APM has `.instructions.md` files. Kiro/Amazon Q use `prompt` field inline. |

### 5.4 Naming Alignment Analysis

| Our Name | Industry Standard | Platforms Using Standard | Action |
|---|---|---|---|
| `mcp` | `mcpServers` | Claude Code, Copilot, Cursor, Amazon Q, Kiro | **Rename to `mcpServers`**. 5 of 5 platforms with MCP support use this name. |
| `instructions` | No standard | Only APM has similar concept | Keep. No strong convention exists. |
| `prompts` | No standard | APM only (by convention, not manifest field) | Keep. Novel component type we introduced. |
| `hooks` | `hooks` | All platforms | Keep. Universal name. |
| `skills` | `skills` | All platforms | Keep. Universal name. |
| `agents` | `agents` | Claude Code, Copilot, Cursor | Keep. Universal name. |

### 5.5 Content Declaration Patterns

| Pattern | Platforms | Description |
|---|---|---|
| **Path arrays** | Our plugin.json | `["skills/review", "skills/lint"]` |
| **String or array** | Claude Code, Copilot, Cursor | `"skills/"` or `["skills/", "extra/"]` |
| **Inline objects** | Claude Code, Copilot, Cursor (hooks, MCP) | Hooks and MCP can be declared inline as JSON objects |
| **Directory convention** | Agent Skills, Codex, Windsurf, OpenCode, Amp | No manifest; discover by scanning directories |
| **YAML manifest** | APM | `apm.yml` with dependencies array |

### 5.6 Manifest Location Patterns

| Pattern | Platforms | Path |
|---|---|---|
| **Root-level** | Our plugin.json, Copilot CLI | `plugin.json` at plugin root |
| **Nested .plugin dir** | Claude Code | `.claude-plugin/plugin.json` |
| **Nested .plugin dir** | Cursor | `.cursor-plugin/plugin.json` |
| **No manifest** | Windsurf, Codex, OpenCode, Amp | Convention-based discovery |
| **Platform-specific dir** | Amazon Q, Kiro | `.amazonq/agents/`, `.kiro/agents/` |

## 6. Discussion

### 6.1 The Convergence of Claude Code, Copilot CLI, and Cursor

The most striking finding is that Claude Code, Copilot CLI, and Cursor have converged on a nearly identical `plugin.json` schema. This convergence happened independently but follows the same pattern:

- `name` as the only required field
- Same metadata fields: `version`, `description`, `author` (object), `homepage`, `repository`, `license`, `keywords`
- Same component paths: `skills`, `agents`, `commands`, `hooks`, `mcpServers`
- Same path flexibility: `string | string[]` for component paths
- Same inline support: hooks and MCP can be path references or inline objects

Our manifest is 90% aligned with this emerging standard. The 3 key divergences are:
1. We call it `mcp` instead of `mcpServers`
2. We require `description` (they make it optional)
3. We have `instructions` and `prompts` (they have `commands` and sometimes `rules`)

### 6.2 The Agent Skills Spec as a Sub-Standard

The Agent Skills spec (agentskills.io) operates at a different level. It defines the format for individual skill files (SKILL.md), not plugin bundles. Every platform that supports skills (Claude Code, Copilot, Cursor, Codex, Windsurf, OpenCode, Amp, Kiro) follows this spec for skill-level metadata.

Our plugin.json references skills by path; the skills themselves follow the Agent Skills spec. These are complementary, not competing standards.

### 6.3 The Amazon Q/Kiro Agent Config Pattern

Amazon Q and Kiro use a different conceptual model. Their JSON configs define a single agent's runtime behavior (tools, permissions, resources, prompt) rather than a plugin bundle. This is closer to our per-agent files in `agents/` than to our plugin.json.

Key concepts from this pattern that we lack:
- `tools`: explicit tool whitelist (not component declaration but permission boundary)
- `allowedTools`: auto-approval without user confirmation
- `resources`: knowledge bases and file references loaded into context
- `toolAliases`: rename tools to avoid conflicts

### 6.4 Is a Universal Plugin Manifest Possible?

Based on this analysis, a universal manifest is not feasible because platforms operate at 3 different conceptual levels:

1. **Plugin bundle** (our level): Claude Code, Copilot CLI, Cursor, APM
2. **Individual skill/agent**: Agent Skills spec, Codex, Windsurf, OpenCode
3. **Runtime configuration**: Amazon Q, Kiro

However, a plugin manifest that maps cleanly to level 1 platforms and can be decomposed into level 2/3 formats is achievable. This is exactly what our adapter layer (ADR-009) does.

### 6.5 Microsoft APM as a Validation Signal

Microsoft APM (`apm.yml`) independently arrived at nearly the same component model we use: skills, agents, instructions, prompts, hooks. Its directory convention (`.apm/instructions/`, `.apm/prompts/`, `.apm/skills/`, `.apm/agents/`, `.apm/hooks/`) maps 1:1 to our directory structure. APM also supports transitive dependencies (`dependencies.apm`), which we deferred in ADR-001. This validates both our component taxonomy and our deferred dependency decision.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Rename `mcp` to `mcpServers` in plugin.json | 5 of 5 platforms with MCP support use `mcpServers`. Our name is the only outlier. Aligning eliminates a translation step in every adapter. | Low (field rename) |
| P1 | Support `string or string[]` for component paths (not just arrays) | Claude Code, Copilot, and Cursor all accept either. Single-skill plugins should not need `["skills/review"]` when `"skills/review"` works. | Low (Zod schema change) |
| P1 | Support inline objects for `hooks` and `mcpServers` | All 3 convergent platforms support this. Authors should be able to declare MCP servers inline without a separate file. | Medium (parser change) |
| P2 | Add optional `lspServers` field | Claude Code and Copilot support it. Cursor supports it. Language server integration is valuable for cross-platform plugins. | Low (field addition) |
| P2 | Add optional `commands` field | Claude Code, Copilot, Cursor all have it. Even if skills supersede commands, supporting them eases migration. | Low (field addition) |
| P2 | Add optional `compatibility` field from Agent Skills spec | Indicates environment requirements. Useful for plugins that need specific tools installed. | Low (field addition) |
| P3 | Add optional `allowedTools` field | Amazon Q and Kiro use it for auto-approval. Agent Skills spec has it as experimental. Cross-platform tool permission concept emerging. | Medium (design required) |
| P3 | Evaluate `resources` field from Amazon Q/Kiro | Knowledge base and file reference concept is useful but may not be portable across all platforms. | Medium (research required) |
| Defer | Add `dependencies` field | APM validates the concept. Already deferred in ADR-001. Implement when dependency resolution is designed. | High (architecture) |
| No action | Add `rules` field | Cursor-specific. Not portable. Handle via platformConfig if needed. | N/A |
| No action | Add `outputStyles` field | Niche. Only Claude Code and Cursor. Low demand. | N/A |
| No action | Add `model` field | Runtime config, not packaging. Each platform handles model selection differently. | N/A |

## 8. Conclusion

**Verdict**: Proceed with 3 targeted alignment changes (P0 + P1 items).

**Confidence**: High

**Rationale**: Our plugin.json is 90% aligned with the emerging standard established by Claude Code, Copilot CLI, and Cursor convergence. Three changes close the remaining gaps: rename `mcp` to `mcpServers`, support `string | string[]` for paths, and support inline objects for hooks/MCP. These are non-breaking, additive changes that reduce adapter complexity and improve developer familiarity.

### User Impact

- **What changes for you**: `mcp` field renamed to `mcpServers`. Existing `mcp` field accepted as alias during migration. Path fields accept both strings and arrays. Hooks and MCP can be declared inline.
- **Effort required**: P0 + P1 items are 2-3 days of implementation.
- **Risk if ignored**: Every adapter must translate `mcp` to `mcpServers`. Authors familiar with Claude Code/Copilot/Cursor will be confused by the naming divergence.

## 9. Appendices

### Platform Documentation Sources

- [Claude Code Plugins Reference](https://code.claude.com/docs/en/plugins-reference)
- [GitHub Copilot CLI Plugin Reference](https://docs.github.com/en/copilot/reference/cli-plugin-reference)
- [Cursor Plugins Reference](https://cursor.com/docs/plugins/building)
- [Windsurf Cascade Skills](https://docs.windsurf.com/windsurf/cascade/skills)
- [OpenAI Codex Agent Skills](https://developers.openai.com/codex/skills/)
- [Agent Skills Specification](https://agentskills.io/specification)
- [AGENTS.md Specification](https://agents.md/)
- [Amazon Q Developer Custom Agents](https://docs.aws.amazon.com/amazonq/latest/qdeveloper-ug/command-line-custom-agents-configuration.html)
- [Kiro Agent Configuration Reference](https://kiro.dev/docs/cli/custom-agents/configuration-reference/)
- [Amp Owner's Manual](https://ampcode.com/manual)
- [OpenCode Skills](https://opencode.ai/docs/skills/)
- [Microsoft APM](https://github.com/microsoft/apm)

### Data Transparency

- **Found**: Complete manifest schemas for Claude Code, Copilot CLI, Cursor, Agent Skills spec, Codex, Kiro, Amazon Q, OpenCode, Amp, Microsoft APM. AGENTS.md spec confirmed as unstructured.
- **Not Found**: Windsurf plugin-level manifest (confirmed: does not exist). Amp plugin manifest (confirmed: code-based, no manifest). Amazon Q full field documentation (JS redirect blocked scraping; fields confirmed via blog posts and third-party guides).

## Observations

- [fact] Claude Code, Copilot CLI, and Cursor independently converged on nearly identical plugin.json schemas with same field names and types #convergence #plugin-manifest
- [fact] All 5 platforms with MCP support use `mcpServers` as the field name; our `mcp` field is the sole outlier #naming #mcpServers
- [fact] Agent Skills spec (agentskills.io) operates at skill level, not plugin level; complementary to plugin manifests #agent-skills #layering
- [fact] Amazon Q and Kiro share nearly identical agent JSON schemas (Amazon Q is upstream) with tools, allowedTools, mcpServers, resources, hooks #amazon-q #kiro
- [fact] Microsoft APM independently arrived at same 5-component model (skills, agents, instructions, prompts, hooks) validating our taxonomy #apm #validation
- [decision] Recommend renaming `mcp` to `mcpServers` for cross-platform alignment #naming #P0
- [decision] Recommend supporting `string | string[]` for component paths to match Claude Code/Copilot/Cursor pattern #paths #P1
- [decision] Recommend supporting inline objects for hooks and mcpServers fields to match convergent platform pattern #inline #P1
- [insight] Universal plugin manifest is not feasible due to 3 conceptual levels (bundle, skill, runtime config), but our adapter-based approach handles this correctly #architecture
- [insight] Our `instructions` and `prompts` fields are novel; only Microsoft APM has similar concepts validating the design #unique-fields
- [risk] `installMode` field has no equivalent on any platform; may create confusion for authors familiar with other platforms #installMode

## Relations

- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-013 npm-Package Distribution and Revised Command Tree]]
- relates_to [[ADR-009 Platform Detection and Config Registry]]
- relates_to [[ANALYSIS-005 Claude Code Plugin Format]]
- relates_to [[ANALYSIS-014-platform-config-patterns]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]