---
title: ADR-012 Scaffolding and Content Management
type: decision
permalink: decisions/adr-012-scaffolding-and-content-management
status: accepted
date: 2026-03-07
decision-makers:
- Peter Kloss (Project Lead)
consulted:
- Architect Agent
- Analyst Agent
informed:
- Implementer Team
- QA
tags:
- scaffolding
- content-management
- wizards
- creator-skills
- cli-architecture
---

# ADR-012 Scaffolding and Content Management

## Context and Problem Statement

The agent-plugin CLI needs scaffolding wizards for creating plugin content (skills, agents, MCP servers, commands, hooks, instructions) and lifecycle commands for managing that content post-creation. The design spec (Section 12) defines wizard flows for 6 content types. Research (ANALYSIS-031) evaluated template strategies, @clack/prompts group() patterns, and prior art from Anthropic's creator skills.

Three Anthropic creator skills (skill-creator, mcp-builder, agent-creator) offer iterative development lifecycles -- eval/improve loops -- that go beyond simple scaffolding. How should the CLI structure its content creation commands, template rendering, wizard flows, and iterative improvement capabilities?

## Decision Drivers

- Content types span 6 categories with different complexity levels
- ADR-007 defined a `new` subtree that proved insufficiently discoverable
- Three Anthropic creator skills provide eval/improve workflows worth adapting
- CLI must serve three interfaces: interactive prompts, CI flags, MCP tool parameters
- Zero new dependencies preferred per ADR-006 dependency minimalism
- Self-bootstrapping principle requires agent-plugin to use its own capabilities
- Instructions (formerly "rules") need platform-aware merge behavior at install time

## Considered Options

- **Option A: Content-type command groups with bundled creator skills** -- Each content type gets a top-level command group; creator skills provide eval/improve
- **Option B: Retain `new` subtree with creator skills as separate plugins** -- Keep ADR-007 `new` tree; creator skills installed separately
- **Option C: Single `create` command with type flag** -- Flat `agent-plugin create --type skill`

## Decision Outcome

Chosen option: **Option A -- Content-type command groups with bundled creator skills**, because it maximizes discoverability via tab completion, keeps related operations grouped, and bundles the eval/improve lifecycle as a first-class capability rather than an afterthought.

This ADR contains 11 formal decisions governing the scaffolding and content management subsystem.

---

## Decision 1: Content-Type Command Groups Replace `new` Tree

**Amends**: ADR-007 Decision 1 (`new` subtree)

The `new` command tree from ADR-007 is replaced by content-type command groups. Each content type gets its own top-level command group with subcommands.

Full command tree:

```text
agent-plugin
├── Consumer: add, remove, upgrade, list
├── Author: init, validate, build, dev
├── skill: create, remove, list, eval, improve
├── agent: create, remove, list, eval, improve
├── mcp: create, create-tool, remove, remove-tool, list, eval, improve
├── command: create, remove, list
├── hook: create, remove, list
├── instruction: create, remove, list, eval, improve
├── complete <shell>
└── help
```

Key design choices:

- `create`/`remove` for content operations (not `add`/`remove` to avoid collision with top-level plugin install `add` command)
- `eval`/`improve` subcommands only for content types backed by creator skills: skill, agent, mcp, instruction
- `rules/` directory renamed to `instructions/`. Instructions merge into AGENTS.md (6 of 7 supported platforms) and CLAUDE.md (Claude Code) at install time

## Decision 2: Template Rendering Strategy

Use `yaml` 2.x `stringify()` for YAML frontmatter in markdown content (agents, skills, commands, instructions). Use tagged template literals for TypeScript code generation (MCP servers, tools). This adds zero new dependencies.

Rejected alternatives:

| Alternative | Reason Rejected |
|---|---|
| Handlebars | Overkill for structured output; adds dependency |
| EJS | Security risk from template injection; adds dependency |
| gray-matter | CVE-2025-64718 vulnerability; blocked by ADR-006 |

## Decision 3: Schema-First Architecture (Consistency Loop)

Each wizard's inputs are defined as a single Zod v4 schema. All three interfaces derive from this schema:

1. **Interactive prompts** -- Schema fields map to @clack/prompts questions
2. **CI flags** -- Schema fields map to CLI flags with validation
3. **MCP tool parameters** -- Schema fields map to JSON Schema tool input

Verified compatible: Zod v4 `toJSONSchema()` output targets JSON Schema Draft 2020-12, matching `@modelcontextprotocol/sdk` tool input schema requirements. Minimum versions: `@modelcontextprotocol/sdk` >= 1.23.0, `zod` >= 4.1.13. See ANALYSIS-032 for full compatibility assessment.

This maintains the self-bootstrapping consistency loop: the same schema validates user input regardless of entry point. Drift between CLI, CI, and MCP interfaces becomes structurally impossible.

## Decision 4: Multi-Group Wizard Flow

Sequential `group()` calls from @clack/prompts for multi-group wizards with `p.log.step()` separators between groups.

| Wizard | Groups | Structure |
|---|---|---|
| agent create | 4 | Identity, Model Config, Skills, Advanced |
| mcp create | 2 | Server Identity, Initial Tool |
| All others | 1 | Single group |

Dynamic loops (e.g., environment variable collection in MCP create) run as imperative code outside `group()` since group() does not support dynamic field counts.

## Decision 5: Manifest Auto-Update

All `create` wizards auto-update plugin.json after file generation:

1. Read manifest via `Bun.file().json()`
2. Add entry to the appropriate manifest array
3. Write back with `JSON.stringify(manifest, null, 2)`

The `validate` command detects drift between filesystem and manifest.

Content-type to manifest array mapping:

| Content Type | Manifest Array | Notes |
|---|---|---|
| skill | skills[] | File + manifest entry |
| agent | agents[] | File + manifest entry |
| mcp | mcp[] | File + manifest entry |
| command | commands[] | File + manifest entry |
| hook | hooks[] | Manifest-only, no files generated |
| instruction | instructions[] | File + manifest entry |

Exception: `mcp create-tool` does NOT update the manifest. It only patches the MCP server's index.ts.

## Decision 6: MCP Server Scaffolding

Generated MCP server code uses `@modelcontextprotocol/sdk` (not fastmcp, per ADR-006). Output structure:

```text
mcp/{server-name}/
├── index.ts          # McpServer + StdioServerTransport + marker comments
└── tools/
    └── {tool-name}.ts  # Individual tool file
```

Design constraints:

- No biome.json generated per MCP server (project-level linting only)
- No per-MCP package.json (single workspace root)
- Marker comments in index.ts delineate tool import/registration sections
- `mcp create-tool` patches index.ts between marker comments to add new tools
- Separate tool files for organizational clarity

## Decision 7: Bundled Creator Skills

Four creator skills are bundled with agent-plugin:

| Content Type | Creator Skill | Source | Adaptation |
|---|---|---|---|
| Skill | skill-creator | anthropics/skills | Python to Bun TypeScript |
| Agent | agent-creator | anthropics/claude-code/plugins/plugin-dev | Python to Bun TypeScript |
| MCP | mcp-builder | anthropics/skills | Python to Bun TS, fastmcp to @modelcontextprotocol/sdk |
| Instruction | instruction-evaluator | Original | agent-plugin exclusive |

Adaptation principles:

- Python MCP servers converted to Bun TypeScript
- fastmcp references replaced with @modelcontextprotocol/sdk
- Python-only MCP scaffolding removed (TypeScript only)
- Output aligned with ADR-001 plugin.json manifest format
- Logic and workflow stay faithful to Anthropic originals where applicable

The instruction-evaluator is original to agent-plugin. It scores and optimizes CLAUDE.md/AGENTS.md content, detects anti-patterns in instruction files, and provides improvement suggestions.

Self-bootstrapping: agent-plugin installs itself as a plugin, making these creator skills available through the standard eval/improve commands.

## Decision 8: Interactive Improve Preview

All `improve` commands display an interactive diff/preview before applying changes:

- Color-coded diff showing before/after per change
- "Why" annotation explaining the reasoning for every proposed change
- Three apply modes:
  - **Apply all**: Accept all changes at once
  - **Review one-by-one**: Step through changes individually (similar to `git add -p`)
  - **Skip**: Reject all changes

CI and MCP behavior:

| Interface | Behavior |
|---|---|
| CI with `--apply` | Auto-accept all changes |
| CI with `--dry-run` | Output JSON diff, exit 0 |
| MCP | Return structured proposed changes object |

## Decision 9: Remove Confirmation

All `remove` commands require `p.confirm()` displaying:

- List of files that will be deleted
- Manifest entries that will be removed
- Any dependent content that references the item

CI mode requires explicit `--yes` flag to bypass confirmation. MCP interface returns a confirmation prompt structure for the caller to handle.

## Decision 10: Wizard Simplifications

Three simplifications from the original design spec:

1. **Emoji prefix DROPPED** from agent create wizard. Agent names use plain text identifiers.
2. **autoLoadAgents DROPPED** from skill create wizard. Instead, `agent create` shows a multiselect of available skills for explicit binding.
3. **Skill frontmatter aligned** with Claude Code SKILL.md fields:
   - Core fields (always prompted): `name`, `description`, `argument-hint`
   - Advanced fields (behind "Configure advanced options?" confirm): `disable-model-invocation`, `user-invocable`, `allowed-tools`, `model`, `context`, `agent`, `hooks`

## Decision 11: eval/improve Trust Model and Execution Boundaries

Creator skill `eval` and `improve` commands execute bundled analysis scripts converted from Anthropic's Python originals to Bun TypeScript. These are first-party code that analyzes user content but does not execute user-authored code.

Trust model:

| Aspect | Specification |
|---|---|
| eval execution | Runs bundled Bun TypeScript analysis scripts. Read-only analysis of user content. Outputs structured report. |
| improve execution | Proposes changes via interactive diff preview (Decision 8). No mutations without user confirmation. |
| HTML eval viewer | Local HTTP server via `Bun.serve()` for visual reports. Bun-native only (no Node.js, no Python). |
| Shared viewer | All eval subcommands (skill, agent, mcp) use the same viewer infrastructure for consistent visual feedback. |
| CI `--apply` | Auto-accepts all proposed improve changes. Equivalent to "Apply all" mode. |
| CI `--dry-run` | Outputs JSON diff only, exit 0. No mutations. |
| MCP interface | Returns structured proposed changes object for caller to handle. |

Faithfulness principle: creator skills stay as close to Anthropic implementations as possible. Exceptions limited to:

- Python scripts converted to Bun TypeScript
- fastmcp references replaced with @modelcontextprotocol/sdk
- Minor verbiage tweaks for alignment and consistency across all creator skills

---

## Consequences

### Positive

- Content-type command groups are more discoverable than a flat `new` tree; tab completion exposes all operations per type
- Schema-first architecture prevents drift between CLI, CI, and MCP interfaces
- Zero new dependencies for template rendering (yaml 2.x already in dependency tree, tagged template literals are native)
- Creator skills provide iterative eval/improve development lifecycle beyond one-shot scaffolding
- Instruction eval/improve is a unique differentiator not available in competing tools
- Interactive improve preview prevents surprise changes to user content

### Negative

- **NEG-001**: ADR-007 command tree must be amended to reflect content-type groups
- **NEG-002**: Four creator skills represent significant implementation scope (estimated 3-4 weeks). Mitigated: creator skills are independently shippable and may be phased.
- **NEG-003**: Python-to-TypeScript conversion of Anthropic creator skills may introduce behavioral differences from originals
- **NEG-004**: Marker comment patching in index.ts is fragile if authors modify generated code outside markers

### Implementation Notes

- **IMP-001**: ADR-007 command tree section must be updated to match Decision 1
- **IMP-002**: ADR-001 must add `instructions/` directory to the plugin structure (replacing `rules/`)
- **IMP-003**: Eval HTML report viewer must be converted for Bun's serve capabilities
- **IMP-004**: Instruction eval scoring rubric to be developed during feature spec phase
- **IMP-005**: ADR-001 must be amended to restore `commands` as a 6th component type (currently excluded as "legacy slash-command pattern"). Commands are not unique to Claude Code -- all target platforms have command equivalents.
- **IMP-006**: Creator skills are independently shippable and may be phased. Sequencing deferred to project planning. IMP-004 (instruction-evaluator scoring rubric) is a gate before that creator skill ships.
- **IMP-007**: ANALYSIS-031 must be updated to replace all gray-matter references with yaml 2.x (gray-matter disqualified by CVE-2025-64718 per ADR-006).

### Confirmation

Implementation and compliance will be confirmed through:

```text
Confirmation Checklist

- [ ] Content-type command groups registered in CLI framework (Decision 1)
- [ ] `new` subtree removed from ADR-007 and CLI (Decision 1)
- [ ] `rules/` renamed to `instructions/` in plugin structure (Decision 1)
- [ ] YAML frontmatter uses yaml 2.x stringify, no Handlebars/EJS/gray-matter (Decision 2)
- [ ] TypeScript generation uses tagged template literals only (Decision 2)
- [ ] Each wizard has a single Zod v4 schema driving all three interfaces (Decision 3)
- [ ] Multi-group wizards use sequential group() with p.log.step() separators (Decision 4)
- [ ] All create wizards auto-update plugin.json manifest (Decision 5)
- [ ] MCP servers use @modelcontextprotocol/sdk, not fastmcp (Decision 6)
- [ ] All four creator skills operational and self-bootstrapped (Decision 7)
- [ ] Improve commands show interactive diff with "Why" annotations (Decision 8)
- [ ] Remove commands require p.confirm() or --yes flag (Decision 9)
- [ ] Skill frontmatter aligns with Claude Code SKILL.md fields (Decision 10)
- [ ] MCP lifecycle commands (start/stop/restart/status) NOT in author CLI (Decision 1, P0-1)
- [ ] Manifest arrays use flat naming (skills[], not content.skills[]) (Decision 5, P0-2)
- [ ] ADR-001 amended to restore commands as 6th component type (IMP-005, P0-3)
- [ ] Zod v4 + MCP SDK version requirements met (Decision 3, P0-7)
- [ ] eval/improve trust model documented and followed (Decision 11, P0-6)
- [ ] HTML eval viewer uses Bun.serve(), shared across all eval subcommands (Decision 11, P0-6)
```

---

## Pros and Cons of the Options

### Option A: Content-type command groups with bundled creator skills (Chosen)

- Good, because tab completion exposes all operations per content type
- Good, because eval/improve lifecycle is first-class, not bolted on
- Good, because self-bootstrapping principle is maintained
- Good, because schema-first eliminates interface drift
- Neutral, because command count increases from ~12 to ~35 total subcommands
- Bad, because amends ADR-007 command tree
- Bad, because four creator skills add 3-4 weeks of implementation scope

### Option B: Retain `new` subtree with creator skills as separate plugins

- Good, because no ADR-007 amendment needed
- Good, because creator skills developed independently on their own timeline
- Bad, because `new skill` vs `skill eval` splits related operations across command trees
- Bad, because discoverability suffers (user must know to look in two places)
- Bad, because separate plugin installation adds friction to developer experience

### Option C: Single `create` command with type flag

- Good, because minimal command surface (`agent-plugin create --type skill`)
- Good, because simple to implement
- Bad, because no natural home for lifecycle commands (eval, improve, list, remove)
- Bad, because tab completion cannot expose type-specific subcommands
- Bad, because forces all content types into identical command signature

---

## Reversibility Assessment

- [x] **Rollback capability**: Command groups can be reorganized without data loss; generated files use standard formats
- [x] **Vendor lock-in**: No vendor lock-in. Creator skills are bundled, not SaaS-dependent
- [ ] **Exit strategy**: N/A -- no external service dependency
- [x] **Legacy impact**: ADR-007 `new` subtree users must update muscle memory; `rules/` to `instructions/` rename requires migration
- [x] **Data migration**: Reversing this decision does not orphan or corrupt data; generated files remain valid

---

## More Information

Anthropic creator skill sources:

- skill-creator: `anthropics/skills` repository
- mcp-builder: `anthropics/skills` repository
- agent-creator: `anthropics/claude-code/plugins/plugin-dev`

The instruction-evaluator has no Anthropic source. It is original to agent-plugin and scores instruction quality against platform-specific best practices (CLAUDE.md structure, AGENTS.md conventions, token efficiency).

Design spec Section 12 contains the original wizard flow diagrams. ANALYSIS-031 contains template strategy evaluation data.

---

## Observations

- [decision] Content-type command groups replace ADR-007 new subtree for improved discoverability #cli-architecture #scaffolding
- [decision] Template rendering uses yaml 2.x stringify and tagged template literals with zero new dependencies #templates #content-management
- [decision] Schema-first architecture derives all three interfaces from a single Zod v4 schema per wizard #consistency #wizards
- [decision] Four creator skills bundled: skill-creator, agent-creator, mcp-builder, instruction-evaluator #creator-skills
- [fact] MCP server scaffolding uses @modelcontextprotocol/sdk per ADR-006 with marker comments for create-tool patching #mcp #scaffolding
- [risk] Marker comment patching in generated index.ts is fragile if authors modify code outside markers #fragility
- [risk] Python-to-TypeScript conversion of 3 Anthropic creator skills may introduce behavioral differences #creator-skills
- [constraint] gray-matter blocked by CVE-2025-64718 per ADR-006 dependency policy #security
- [fact] rules/ directory renamed to instructions/ with platform-aware merge into AGENTS.md or CLAUDE.md at install time #instructions
- [decision] All improve commands show interactive diff with Why annotations before applying changes #improve-preview
- [decision] eval/improve executes bundled first-party Bun TypeScript scripts, not user code. HTML viewer via Bun.serve() #trust-model #security
- [fact] Zod v4 toJSONSchema() verified compatible with MCP SDK tool input schemas. Min versions: SDK >= 1.23.0, Zod >= 4.1.13 #compatibility
- [decision] Commands restored as content type, amending ADR-001 5-component to 6-component model #commands #plugin-format

## Relations

- amends [[ADR-007 CLI Architecture and Interaction Model]]
- amends [[ADR-001 Plugin Format and Manifest]]
- depends_on [[ADR-006 Core Dependency Stack]]
- relates_to [[ADR-011 Auto-Generated CLI from MCP Tools]]
- relates_to [[ADR-010 Installation Lifecycle]]
- relates_to [[ANALYSIS-031 Scaffolding Wizard Patterns]]
- relates_to [[ANALYSIS-022 @clack/prompts API Surface and Gaps]]
- relates_to [[ANALYSIS-032 MCP SDK Bun Runtime Compatibility]]