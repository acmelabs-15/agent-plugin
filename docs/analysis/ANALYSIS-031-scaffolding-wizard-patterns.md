---
title: ANALYSIS-031 Scaffolding Wizard Patterns
type: analysis
permalink: analysis/analysis-031-scaffolding-wizard-patterns
tags:
- scaffolding
- wizards
- cli
- clack-prompts
- code-generation
- templates
---

# ANALYSIS-031 Scaffolding Wizard Patterns

## 1. Objective and Scope

**Objective**: Define the complete scaffolding wizard architecture for the `new` command tree. This covers what each wizard asks, what it generates, how it integrates with @clack/prompts `group()`, how it works in CI and MCP modes, what template strategy to use, and whether wizards should auto-update `plugin.json`.

**Scope**: All 7 scaffolding wizards (`new agent`, `new skill`, `new command`, `new hook`, `new rule`, `new mcp init`, `new mcp add-tool`). Excludes the `init` command (project-level scaffolding) and consumer commands.

## 2. Context

The design spec (Section 12) defines wizard flows for each content type. ADR-007 establishes the three-tier input resolution pattern (interactive via @clack/prompts, CI via flags, MCP via tool parameters). ADR-001 defines the plugin bundle model with `plugin.json` manifest and conventional directories (`agents/`, `skills/`, `commands/`, `hooks/`, `mcp/`, `rules/`). ADR-003 mandates always-namespace with colon separator and kebab-case validation.

The self-bootstrapping architecture (Section 16) requires that CLI wizards, MCP tools, and AI skills ask identical questions and produce identical output. This creates a "consistency loop" where a shared library layer sits beneath all three interfaces.

### Prior Art Reviewed

- Yeoman: Full project scaffolding, generator ecosystem, Inquirer-based prompts
- Plop: Micro-generator with Handlebars templates, Inquirer prompts
- Hygen: Lightweight code generation with EJS templates, frontmatter-based metadata
- create-t3-app: Modular scaffolding with component selection, no template engine (direct file manipulation)
- @modelcontextprotocol/create-server: MCP server scaffold with fastmcp, zod, placeholder tools
- Claude Code skill-creator: SKILL.md with YAML frontmatter, progressive disclosure structure

## 3. Approach

**Methodology**: Cross-referenced design spec Section 12 against ADR-001, ADR-003, ADR-007, and the author project layout (Section 17). Evaluated template engine options against project constraints (Bun runtime, yaml 2.x with manual frontmatter parser for frontmatter, TypeScript strict mode). Analyzed @clack/prompts `group()` API for multi-step wizard compatibility.

**Tools Used**: Design spec, ADR documents, web research on scaffolding tools and @clack/prompts

**Limitations**: No access to the actual @clack/prompts source code for `group()` beyond documentation. Bun compatibility of template engines not verified at runtime.

## 4. Data and Analysis

### 4.1 Content Types Requiring Scaffolding

7 content types map to 7 wizard commands:

| Wizard | Output Type | Files Generated | Manifest Update |
|---|---|---|---|
| `new agent` | Markdown + frontmatter | 1 `.md` file | `content.agents[]` |
| `new skill` | Directory + SKILL.md | 1-3 items (SKILL.md + optional scripts/, references/) | `content.skills[]` |
| `new command` | Markdown + frontmatter | 1 `.md` file | `content.commands[]` |
| `new hook` | Manifest-only entry | 0 files (hook config in manifest) | `content.hooks[]` |
| `new rule` | Markdown | 1 `.md` file | `content.rules[]` |
| `new mcp init` | TypeScript project | 4+ files (index.ts, tools/*.ts, package.json, biome.json) | `content.mcp[]` |
| `new mcp add-tool` | TypeScript file + patch | 1 file + 1 patch (tools/*.ts + index.ts import) | None (MCP config unchanged) |

### 4.2 Wizard Input Specifications

#### `new agent` (4-group wizard)

**Group 1: Identity**

| Field | Prompt Type | Default | Validation | CI Flag |
|---|---|---|---|---|
| name | `p.text()` | none | `^[a-z][a-z0-9-]*$`, unique | `--name` |
| description | `p.text()` | none | 10+ chars | `--description` |
| model | `p.autocomplete()` | claude-sonnet-4 | known models or custom | `--model` |
| customModel | `p.text()` | none | non-empty (conditional) | `--model` (same flag) |

**Group 2: Capabilities**

| Field | Prompt Type | Default | Source | CI Flag |
|---|---|---|---|---|
| tools | `p.autocompleteMultiselect()` | Read, Grep, Glob | static list | `--tools` (comma-sep) |
| mcpTools | `p.autocompleteMultiselect()` | none | `discoverMcpServers()` | `--mcp-tools` (comma-sep) |
| skills | `p.autocompleteMultiselect()` | none | `discoverContent()` | `--skills` (comma-sep) |
| emojiPrefix | `p.text()` | none | none | `--emoji` |

**Group 3: Behavior**

| Field | Prompt Type | Default | CI Flag |
|---|---|---|---|
| delegationPattern | `p.select()` | specialist | `--delegation` |
| canSpawnSubagents | `p.confirm()` | false | `--spawn-subagents` |
| instructions | `p.text()` | none | `--instructions` |

**Group 4: Hooks (optional)**

| Field | Prompt Type | Default | CI Flag |
|---|---|---|---|
| addSessionStart | `p.confirm()` | false | `--session-start-hook` |
| addPostToolUse | `p.confirm()` | false | `--post-tool-hook` |
| postToolUseMatcher | `p.text()` | none (conditional) | `--post-tool-matcher` |
| addQualityGate | `p.confirm()` | false | `--quality-gate-hook` |

**Output**: Agent `.md` file via yaml 2.x frontmatter serializer (manual parser + Zod v4 validation). ~~gray-matter.stringify()~~ was disqualified per ADR-012 (CVE-2025-64718).

#### `new skill` (single group)

| Field | Prompt Type | Default | Validation | CI Flag |
|---|---|---|---|---|
| name | `p.text()` | none | `^[a-z][a-z0-9-]*$`, unique | `--name` |
| description | `p.text()` | none | non-empty | `--description` |
| includeScripts | `p.confirm()` | false | none | `--scripts` |
| includeReferences | `p.confirm()` | false | none | `--references` |
| autoLoadAgents | `p.autocompleteMultiselect()` | none | discovered agents | `--agents` (comma-sep) |

**Output**: Directory `{contentDir}/skills/{name}/` containing `SKILL.md` with progressive disclosure structure (Overview, When to Use, Instructions, Examples), optional `scripts/` and `references/` subdirectories.

#### `new command` (single group)

| Field | Prompt Type | Default | Validation | CI Flag |
|---|---|---|---|---|
| name | `p.text()` | none | `^[a-z][a-z0-9-]*$`, unique | `--name` |
| description | `p.text()` | none | non-empty | `--description` |
| allowedTools | `p.autocompleteMultiselect()` | none | static list | `--tools` (comma-sep) |
| acceptArguments | `p.confirm()` | false | none | `--arguments` |

**Output**: Command `.md` file.

#### `new hook` (single group with conditionals)

| Field | Prompt Type | Default | Validation | CI Flag |
|---|---|---|---|---|
| event | `p.autocomplete()` | none | known events list | `--event` |
| matcher | `p.text()` | none (conditional) | valid glob pattern | `--matcher` |
| hookType | `p.select()` | command | command or prompt | `--type` |
| shellCommand | `p.text()` | none (conditional) | non-empty | `--command` |
| promptText | `p.text()` | none (conditional) | non-empty | `--prompt` |

Known events: `PreToolUse`, `PostToolUse`, `PermissionRequest`, `UserPromptSubmit`, `Stop`, `SessionStart`, `SessionEnd`, `Notification`.

**Output**: No file created. Hook entry added to `plugin.json` `content.hooks[]`.

#### `new rule` (single group)

| Field | Prompt Type | Default | Validation | CI Flag |
|---|---|---|---|---|
| name | `p.text()` | none | `^[a-z][a-z0-9-]*$` | `--name` |
| description | `p.text()` | none | non-empty | `--description` |
| content | `p.text()` | template | none | `--content` |

**Output**: Rule `.md` file.

#### `new mcp init` (2-group wizard)

**Group 1: Identity**

| Field | Prompt Type | Default | Validation | CI Flag |
|---|---|---|---|---|
| serverName | `p.text()` | none | `^[a-z][a-z0-9-]*$`, unique | `--name` |
| description | `p.text()` | none | non-empty | `--description` |
| transport | `p.select()` | stdio | stdio or sse | `--transport` |
| initialTools | `p.text()` | none | comma-separated kebab-case names | `--tools` (comma-sep) |
| port | `p.text()` | 3000 (conditional) | 1-65535 | `--port` |

**Group 2: Environment**

| Field | Prompt Type | Default | Validation | CI Flag |
|---|---|---|---|---|
| defineEnvVars | `p.confirm()` | false | none | `--env` (key=value pairs) |
| envVarName | `p.text()` (loop) | none | `^[A-Z_][A-Z0-9_]*$` | (part of --env) |
| envVarDefault | `p.text()` (loop) | none | none | (part of --env) |

**Output**: Directory `{contentDir}/mcp/{name}/` containing:
- `index.ts` (fastmcp server entry with tool imports and registrations)
- `tools/{tool}.ts` (one per initial tool with zod schema and execute stub)
- `package.json` (fastmcp + zod dependencies)
- `biome.json` (formatter/linter config)

#### `new mcp add-tool` (single group)

| Field | Prompt Type | Default | Validation | CI Flag |
|---|---|---|---|---|
| server | `p.select()` | auto (if 1 server) | discovered servers | `--server` |
| toolName | `p.text()` | none | `^[a-z][a-z0-9-]*$`, unique in server | `--name` |
| description | `p.text()` | none | non-empty | `--description` |
| parameters | `p.text()` | none | `name:type` format | `--params` (comma-sep) |
| returnType | `p.select()` | string | string, json, array, void | `--return-type` |

Valid parameter types: `string`, `number`, `boolean`, `string[]`, `number[]`.

**Output**:
- `tools/{name}.ts` with zod schema and execute stub
- Patched `index.ts`: import line added, `server.addTool()` registration appended

### 4.3 @clack/prompts group() Integration

The `group()` function accepts an object of prompt functions. Each function receives `results` from prior prompts, enabling conditional logic:

```typescript
const identity = await group({
  name: () => text({
    message: "Agent name",
    validate: (v) => kebabCaseRegex.test(v) ? undefined : "Use kebab-case",
  }),
  description: () => text({
    message: "Description (helps orchestrator delegate)",
    validate: (v) => v.length >= 10 ? undefined : "Must be 10+ characters",
  }),
  model: () => autocomplete({
    message: "Model",
    options: modelOptions,
  }),
  customModel: ({ results }) =>
    results.model === "custom"
      ? text({ message: "Custom model string" })
      : undefined,
});
```

**Design consideration**: The spec defines 4 groups for `new agent` and 2 groups for `new mcp init`. All other wizards use a single group. There are two approaches to multi-group wizards:

**Option A: Sequential group() calls**

```typescript
const g1 = await group({ /* identity */ });
const g2 = await group({ /* capabilities, using g1 context */ });
const g3 = await group({ /* behavior */ });
const g4 = await group({ /* hooks */ });
const result = { ...g1, ...g2, ...g3, ...g4 };
```

Pros: Clear visual separation, can show section headers between groups.
Cons: 4 awaits per wizard, cancellation requires guards at each group boundary.

**Option B: Single group() with visual separators**

Use `p.log.step()` or `p.note()` between logical sections within one `group()`.

Pros: One cancellation guard, simpler code.
Cons: `group()` does not support interspersed non-prompt calls. Each property must be a prompt function.

**Recommendation**: Option A (sequential group() calls). The spec explicitly defines numbered groups. Each group() call maps to one section. Insert `p.log.step("Capabilities")` between group calls for visual structure. This matches the spec's mental model and allows `p.note()` summaries between sections.

### 4.4 Three-Tier Mode Implementation

Per ADR-007 Decision 4, every wizard input site needs a three-branch guard. The shared library layer implements this:

```typescript
// Shared resolver pattern
async function resolveInput<T>(opts: {
  flag: T | undefined;
  prompt: () => Promise<T>;
  mcpParam: T | undefined;
  mode: "interactive" | "ci" | "mcp";
  flagName: string;
  options?: string[];
}): Promise<T> {
  if (opts.flag !== undefined) return opts.flag;
  if (opts.mode === "interactive") return opts.prompt();
  if (opts.mode === "ci") {
    throw new CliError(`Missing --${opts.flagName}. Required in non-interactive mode.`, 2);
  }
  // MCP mode
  throw new McpError("MISSING_INPUT", `Missing ${opts.flagName}`, opts.options);
}
```

**CI mode**: Every wizard field maps to a CLI flag (see tables above). All flags are required in CI mode. No defaults are assumed because wizard defaults are interactive-only conveniences.

**MCP mode**: Every wizard field maps to a Zod-typed parameter on the corresponding MCP tool. The MCP tool schema mirrors the wizard exactly:

```typescript
// MCP tool: acmelabs_new_agent
const params = z.object({
  name: z.string().regex(/^[a-z][a-z0-9-]*$/).describe("Agent name in kebab-case"),
  description: z.string().min(10).describe("Agent purpose for orchestrator delegation"),
  model: z.string().describe("Model identifier"),
  tools: z.array(z.string()).optional().describe("Allowed tools"),
  mcpTools: z.array(z.string()).optional().describe("MCP tools from project servers"),
  skills: z.array(z.string()).optional().describe("Skills to auto-load"),
  delegationPattern: z.enum(["orchestrator", "specialist", "reviewer", "independent"]).optional(),
  canSpawnSubagents: z.boolean().optional().default(false),
  instructions: z.string().optional(),
  // ... hooks
});
```

### 4.5 Template Strategy

Four options evaluated:

| Option | Engine | Pros | Cons |
|---|---|---|---|
| A: Handlebars | handlebars (plop uses this) | Logic-less, well-known, helpers | Extra dependency, overkill for simple interpolation |
| B: EJS | ejs (hygen uses this) | Inline JS logic, lightweight | Security risk with eval, extra dependency |
| C: Tagged template literals | None (native TS) | Zero dependencies, type-safe, Bun-native | Verbose for complex templates, no partial support |
| ~~D: gray-matter.stringify()~~ | ~~gray-matter~~ | ~~Already in dependency stack, handles frontmatter natively~~ | **DISQUALIFIED**: CVE-2025-64718 in pinned js-yaml@^3.13.1 dependency, 5+ years inactive maintainer. See ADR-012. |
| D (revised): yaml 2.x + manual parser | yaml 2.x + 5-10 line manual frontmatter parser + Zod v4 | Actively maintained, no CVEs, small footprint, Zod validates structure | Requires writing a small parser (~10 lines) instead of using a library |

**Recommendation**: Option C + D (revised) combined. Use yaml 2.x with a manual frontmatter parser and Zod v4 validation for all markdown content types (agents, skills, commands, rules). Use tagged template literals for TypeScript code generation (MCP server files). This requires only yaml 2.x as a new dependency (already needed for frontmatter parsing elsewhere). gray-matter was originally recommended here but was disqualified during ADR-006 review due to CVE-2025-64718 in its pinned js-yaml@^3.13.1 dependency. See ADR-012 for the full disqualification rationale and replacement strategy.

**Rationale**: The templates are simple. Agent markdown is frontmatter + body text. Skill SKILL.md is frontmatter + section headings. MCP TypeScript files are import statements + function calls. None of these require loops, partials, or conditional blocks that would justify a template engine. The manual frontmatter parser is approximately 5-10 lines of code and pairs with Zod v4 for schema validation, providing stronger type guarantees than gray-matter offered. Create-t3-app validates this approach: it generates an entire Next.js project without a template engine, using direct file manipulation.

**Implementation pattern for markdown content** (updated per IMP-007/ADR-012: gray-matter replaced by yaml 2.x + manual parser):

```typescript
import { stringify } from "yaml";
// Manual frontmatter serializer (~5 lines, replaces gray-matter.stringify())
function stringifyFrontmatter(body: string, data: Record<string, unknown>): string {
  return `---\n${stringify(data).trimEnd()}\n---\n\n${body}`;
}

function generateAgent(input: AgentInput): string {
  const frontmatter = {
    name: input.name,
    description: input.description,
    model: input.model,
    tools: input.tools,
    mcpTools: input.mcpTools,
    skills: input.skills,
    delegationPattern: input.delegationPattern,
    canSpawnSubagents: input.canSpawnSubagents,
  };

  const body = `# ${input.name}

${input.instructions || "<!-- Add agent instructions here -->"}
`;

  return stringifyFrontmatter(body, frontmatter);
}
```

**Implementation pattern for TypeScript content**:

```typescript
function generateMcpTool(input: ToolInput): string {
  const params = input.parameters
    .map((p) => `  ${p.name}: z.${mapType(p.type)}().describe("${p.name}")`)
    .join(",\n");

  return `import { z } from "zod";
import type { FastMCP } from "fastmcp";

export function register(server: FastMCP) {
  server.addTool({
    name: "${input.name}",
    description: "${input.description}",
    parameters: z.object({
${params}
    }),
    execute: async (args) => {
      // TODO: Implement ${input.name}
      return "";
    },
  });
}
`;
}
```

### 4.6 Manifest Auto-Update

**Question**: Should wizards auto-update `plugin.json` after generating files?

**Analysis**: The spec (Section 12) explicitly states that `new agent` output includes "Updated `acmelabz.json` manifest (agents array + any hooks)". The `new mcp init` output includes "Updated `acmelabz.json` manifest (mcp array)". This is stated for every wizard that creates files.

| Wizard | Manifest Field Updated |
|---|---|
| `new agent` | `content.agents[]` + optionally `content.hooks[]` |
| `new skill` | `content.skills[]` |
| `new command` | `content.commands[]` |
| `new hook` | `content.hooks[]` (this is the only output) |
| `new rule` | `content.rules[]` |
| `new mcp init` | `content.mcp[]` |
| `new mcp add-tool` | None (tools are within an existing MCP server) |

**Recommendation**: Auto-update `plugin.json` for all wizards. The `validate` command can detect drift between manifest and disk. Requiring manual manifest updates after every scaffold would create friction and a common source of validation errors. The manifest update is idempotent: add a new entry to the appropriate array if it does not already exist.

**Implementation**: Read `plugin.json` via `Bun.file().json()`, add the new content entry, write back with `Bun.write()`. Use `JSON.stringify(manifest, null, 2)` to preserve formatting with 2-space indentation.

### 4.7 Project Context Discovery

Per Section 18, wizards that reference existing project content use two discovery functions:

- **`discoverContent()`**: Scans content directories, returns declared and undeclared content. Used by `new agent` (to offer skills for auto-load), `new skill` (to offer agents for linking).
- **`discoverMcpServers()`**: Scans `{contentDir}/mcp/*/index.ts`, returns server metadata. Used by `new agent` (MCP tool discovery), `new mcp add-tool` (server selection).

Discovery runs before prompts so that `p.autocompleteMultiselect()` can present real options. In CI mode, the `--skills` and `--mcp-tools` flags accept names that are validated against discovered content.

### 4.8 File Generation Summary

#### Agent `.md`

```markdown
---
name: code-reviewer
description: Reviews pull requests for code quality and standards
model: claude-sonnet-4
tools:
  - Read
  - Grep
  - Glob
mcpTools:
  - brain:search
skills:
  - code-standards
delegationPattern: specialist
canSpawnSubagents: false
emoji: "\U0001F50D"
---

# code-reviewer

<!-- Agent instructions go here -->
```

#### Skill directory

```text
skills/code-review/
  SKILL.md
  scripts/      (optional)
  references/   (optional)
```

SKILL.md structure (progressive disclosure):

```markdown
---
name: code-review
description: Guidelines for reviewing code quality
---

# code-review

## Overview

<!-- What this skill does -->

## When to Use

<!-- Trigger conditions -->

## Instructions

<!-- Step-by-step instructions -->

## Examples

<!-- Usage examples -->
```

#### Command `.md`

```markdown
---
name: review-pr
description: Step-by-step PR review workflow
tools:
  - Read
  - Bash
acceptArguments: true
---

# /review-pr

<!-- Command instructions -->
```

#### Hook (manifest entry only)

```json
{
  "event": "PostToolUse",
  "matcher": "Write|Edit",
  "hook": {
    "type": "command",
    "command": "biome check --write $CLAUDE_FILE_PATHS"
  }
}
```

#### Rule `.md`

```markdown
---
name: code-conventions
description: Code style and naming conventions
---

# code-conventions

<!-- Rule content -->
```

#### MCP server directory

```text
mcp/my-server/
  index.ts          (fastmcp entry, tool imports, server.start())
  tools/
    my-tool.ts      (zod schema + execute stub)
  package.json      (fastmcp + zod deps)
  biome.json        (formatter config)
```

`index.ts` template:

```typescript
import { FastMCP } from "fastmcp";
import { registerMyTool } from "./tools/my-tool.js";

const server = new FastMCP({
  name: "my-server",
  version: "0.1.0",
});

registerMyTool(server);

server.start({ transportType: "stdio" });
```

`tools/my-tool.ts` template:

```typescript
import { z } from "zod";
import type { FastMCP } from "fastmcp";

export function registerMyTool(server: FastMCP) {
  server.addTool({
    name: "my-tool",
    description: "TODO: Add description",
    parameters: z.object({
      query: z.string().describe("Search query"),
    }),
    execute: async (args) => {
      // TODO: Implement my-tool
      return "";
    },
  });
}
```

#### MCP add-tool (file + patch)

New file: `tools/{name}.ts` (same pattern as above).
Patch to `index.ts`: Add import line and `register{Name}(server)` call before `server.start()`.

### 4.9 Env Variable Handling in `new mcp init`

The env var collection loop in the spec uses `p.confirm()` + `p.text()` in a while-loop pattern:

```typescript
const envVars: Array<{ name: string; defaultValue: string }> = [];
let addMore = await confirm({ message: "Define environment variables?" });
while (addMore) {
  const name = await text({
    message: "Env var name",
    validate: (v) => /^[A-Z_][A-Z0-9_]*$/.test(v) ? undefined : "Use UPPER_SNAKE_CASE",
  });
  const defaultValue = await text({ message: `Default value for ${name}` });
  envVars.push({ name, defaultValue });
  addMore = await confirm({ message: "Add another?" });
}
```

This loop cannot be expressed inside `group()` because `group()` requires a fixed set of prompt keys. The loop runs after the Group 2 header.

In CI mode, env vars are passed as: `--env API_KEY=sk-xxx --env DB_URL=postgres://...`
In MCP mode: `envVars: [{ name: "API_KEY", defaultValue: "sk-xxx" }]`

### Evidence Gathered

| Finding | Source | Confidence |
|---|---|---|
| 7 content types need scaffolding | Design spec Section 12 | High |
| `new agent` uses 4 prompt groups | Design spec Section 12 | High |
| `new mcp init` uses 2 prompt groups | Design spec Section 12 | High |
| All other wizards use single group | Design spec Section 12 | High |
| ~~gray-matter handles frontmatter serialization~~ gray-matter disqualified (CVE-2025-64718); replaced by yaml 2.x + manual parser per ADR-012 | Design spec Section 21, ADR-012 | High |
| Wizards must auto-update plugin.json | Design spec Section 12 (explicit in output) | High |
| group() supports conditional prompts via results | @clack/prompts docs | High |
| group() cannot contain loops | @clack/prompts API shape (fixed keys) | High |
| create-t3-app uses no template engine | Web research | High |
| fastmcp uses server.addTool() pattern | Web research, design spec | High |
| Plop uses Handlebars, Hygen uses EJS | Web research | High |
| Template engines add unnecessary deps for this use case | Analysis | Medium |

### Facts (Verified)

- The design spec defines exact prompt types and validation for all 7 wizards
- ~~gray-matter.stringify() can serialize frontmatter + body for all markdown content types~~ gray-matter disqualified (CVE-2025-64718 in js-yaml@^3.13.1); yaml 2.x with a manual frontmatter parser replaces it per ADR-012
- @clack/prompts group() accepts an object of prompt functions with access to prior results
- group() cannot handle dynamic loops (env var collection requires a separate while-loop)
- The spec mandates `plugin.json` auto-update for all file-generating wizards
- `discoverContent()` and `discoverMcpServers()` provide context for autocomplete options
- ADR-007 mandates every prompt site implements the three-tier guard (interactive/CI/MCP)
- 16 Zod validation rules (ADR-007 Decision 8) cover all wizard input validation

### Hypotheses (Unverified)

- Tagged template literals will be sufficient for all MCP TypeScript generation without a template engine (requires implementation validation)
- The `index.ts` patch for `new mcp add-tool` can use string manipulation (regex-based insertion before `server.start()`) without an AST parser (edge cases with complex index.ts files untested)
- Sequential `group()` calls with `p.log.step()` between them will produce acceptable visual output (no Bun runtime test)

## 5. Results

### Content Type Coverage

All 7 content types from the design spec are accounted for. Each maps to exactly one `new` subcommand. The `new mcp` namespace contains 2 subcommands (`init` and `add-tool`), consistent with ADR-007 Decision 1.

### Wizard Complexity Distribution

| Complexity | Wizards | Prompt Count |
|---|---|---|
| Multi-group (4 groups) | `new agent` | 14 prompts |
| Multi-group (2 groups) | `new mcp init` | 7 prompts + env var loop |
| Single group with conditionals | `new hook` | 5 prompts (2 conditional) |
| Single group | `new skill`, `new command`, `new rule`, `new mcp add-tool` | 3-5 prompts each |

### Template Strategy Decision

Use yaml 2.x with manual frontmatter parser + Zod v4 validation for markdown, tagged template literals for TypeScript. ~~gray-matter~~ disqualified per ADR-012 (CVE-2025-64718).

### Manifest Update Decision

Auto-update `plugin.json` for all wizards. This matches spec requirements and reduces author friction.

## 6. Discussion

### The Consistency Loop Constraint

The self-bootstrapping architecture creates a hard constraint: the wizard prompt sequence, the MCP tool parameter schema, and the skill documentation must stay synchronized. This means:

1. Wizard prompt definitions should be derived from a shared schema (Zod object)
2. MCP tool parameters should use the same Zod schema
3. The skill's `wizard-questions.md` reference file should be generated from the schema
4. Changes to any wizard must propagate to all three interfaces

This suggests a schema-first approach where each wizard's input is defined as a Zod schema once, and the interactive prompts, CI flags, and MCP parameters are all derived from that schema.

### group() Limitations

Two limitations surfaced during analysis:

1. **No loops in group()**: The env var collection in `new mcp init` requires a while-loop outside group(). This breaks the clean group-per-section model. The workaround is to run Group 2 as imperative code after the group() call.

2. **No non-prompt calls between group keys**: You cannot insert `p.log.step()` inside a group() to create visual sections. Visual separators must go between sequential group() calls.

### index.ts Patching Risk

The `new mcp add-tool` command must patch an existing `index.ts` file. This is fragile if authors modify the generated `index.ts` beyond the template structure. Two mitigation strategies:

1. **Marker comments**: Generate `index.ts` with `// --- IMPORTS ---` and `// --- REGISTRATIONS ---` markers. The patcher inserts at these markers.
2. **AST-based patching**: Use a TypeScript AST parser to find the correct insertion points. Higher reliability but adds complexity.

Recommendation: Start with marker comments. They are explicit, zero-dependency, and match hygen's "injection" pattern. Document that removing markers breaks `add-tool` and require manual registration.

### Naming Convention Enforcement

ADR-003 mandates kebab-case for all component names. The validation regex `^[a-z][a-z0-9-]*$` is shared across all wizards. When content is installed, names are prefixed with `plugin-name:` (the namespace). The wizard does not add this prefix -- it generates content with bare names. Namespacing happens at install time.

### Hook Wizard is File-less

The `new hook` wizard is unique: it generates no files on disk. It only modifies `plugin.json`. This means:

- No template is needed
- The wizard output is a JSON object appended to `content.hooks[]`
- The `validate` command cannot verify hooks by checking file paths (unlike all other content types)
- Hooks are the only content type where `plugin.json` is the sole artifact

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Use schema-first design: define each wizard's inputs as a Zod schema, derive prompts/flags/MCP params from it | Maintains consistency loop, reduces drift between 3 interfaces | Medium |
| P0 | Use yaml 2.x + manual frontmatter parser + Zod v4 for markdown, template literals for TypeScript | ~~gray-matter~~ disqualified (CVE-2025-64718, ADR-012); yaml 2.x is actively maintained, parser is ~10 lines | Low |
| P0 | Auto-update plugin.json in all wizards | Spec requirement, reduces validation errors from manual manifest edits | Low |
| P1 | Use sequential group() calls for multi-group wizards (agent, mcp init) | Matches spec's group structure, allows visual separators between groups | Low |
| P1 | Use marker comments in generated index.ts for add-tool patching | Simple, zero-dependency, explicitly documented limitation | Low |
| P1 | Implement env var loop as imperative code outside group() | group() cannot express dynamic loops | Low |
| P2 | Generate wizard-questions.md from Zod schemas for the authoring skill | Closes the consistency loop for AI-driven content creation | Medium |
| P2 | Add --dry-run flag to all new subcommands | Shows what would be created without writing files; useful for CI validation | Low |

## 8. Conclusion

**Verdict**: Proceed with implementation. No blocking unknowns remain.

**Confidence**: High

**Rationale**: The design spec fully defines all 7 wizard flows with explicit prompt types, validation rules, and output files. The template strategy (yaml 2.x + manual frontmatter parser + template literals) adds only yaml 2.x as a dependency. gray-matter was the original recommendation but was disqualified during ADR-006 review due to CVE-2025-64718 (see ADR-012). The @clack/prompts group() API supports the multi-step flows with known workarounds for its limitations (loops, visual separators). The three-tier mode pattern from ADR-007 applies uniformly to all wizard inputs.

### User Impact

- **What changes for you**: Plugin authors get guided wizards that produce valid, manifest-registered content in one command. AI assistants use identical MCP tools for the same result.
- **Effort required**: 7 wizard implementations, each following the shared resolver pattern. The `new agent` wizard is the largest (14 prompts, 4 groups). The `new rule` wizard is the smallest (3 prompts).
- **Risk if ignored**: Without scaffolding wizards, authors must manually create content files with correct frontmatter, register them in plugin.json, and follow naming conventions. This creates a high error rate for new authors and breaks the self-bootstrapping consistency loop.

## 9. Appendices

### Open Questions

1. **Emoji input in terminals**: The spec shows `p.text()` for emoji prefix in `new agent`. Not all terminals render emoji input correctly. Should this use `p.select()` with common emoji options plus a custom option?

2. **Skill auto-load linkage**: When `new skill` asks "which agents should auto-load this skill", what mechanism links them? The spec does not define whether this modifies the agent's frontmatter or adds a field to the skill's metadata.

3. **Template evolution**: If the generated file templates change in future versions, how do existing projects upgrade their scaffolded content? The `init` command handles project-level migration, but individual content files have no migration path.

4. **MCP tool return type**: The `new mcp add-tool` spec asks for a return type (string, json, array, void). FastMCP's `execute` function always returns a string. The return type may be aspirational metadata rather than enforced at runtime.

5. **Content dir resolution**: The spec references `{contentDir}` (default `src/`). This comes from `.acmelabz/project.json`. Wizards must load project context before determining output paths. What happens if project context is unavailable (no init done)?

### Sources Consulted

- Design spec Section 12 (Scaffolding Wizard Specifications)
- Design spec Section 16 (Self-Bootstrapping Architecture)
- Design spec Section 17 (Author Project Layout)
- Design spec Section 18 (Project Context System)
- Design spec Section 21 (Markdown Processing Pipeline)
- ADR-001 Plugin Format and Manifest
- ADR-003 Namespace Strategy (via ADR-001 references)
- ADR-007 CLI Architecture and Interaction Model
- ANALYSIS-022 @clack/prompts API Surface and Gaps
- @clack/prompts documentation (bomb.sh/docs)
- Plop.js documentation (plopjs.com)
- Hygen documentation (github.com/jondot/hygen)
- FastMCP documentation (github.com/punkpeye/fastmcp)
- Claude Code Skills documentation (code.claude.com/docs/en/skills)
- @modelcontextprotocol/create-server patterns

### Data Transparency

- **Found**: Complete wizard input specs from design spec, @clack/prompts group() API behavior, template engine comparisons, fastmcp tool registration patterns, Claude Code SKILL.md format
- **Not Found**: Bun runtime verification of sequential group() visual output, actual @clack/prompts group() source code, internal create-t3-app template generation implementation details

## Observations

- [fact] 7 scaffolding wizards cover all content types: agent, skill, command, hook, rule, mcp init, mcp add-tool #scaffolding
- [fact] new agent is the largest wizard with 14 prompts across 4 groups; new rule is the smallest with 3 prompts #scaffolding #complexity
- [decision] Template strategy: yaml 2.x + manual frontmatter parser + Zod v4 for markdown content, tagged template literals for TypeScript code. ~~gray-matter~~ disqualified per ADR-012 (CVE-2025-64718) #templates #dependencies
- [decision] Sequential group() calls for multi-group wizards with p.log.step() visual separators between groups #clack-prompts #ux
- [decision] Auto-update plugin.json for all file-generating wizards as specified in design spec #manifest #automation
- [technique] Schema-first design: define Zod schema once, derive interactive prompts, CI flags, and MCP params from it #consistency-loop #architecture
- [technique] Marker comments in generated index.ts enable add-tool patching without AST parsing #code-generation
- [constraint] group() cannot express dynamic loops; env var collection requires imperative code outside group() #clack-prompts #limitation
- [risk] index.ts patching is fragile if authors remove marker comments; manual registration fallback needed #mcp #maintenance
- [insight] new hook is the only wizard that produces no files on disk; it modifies plugin.json exclusively #hooks #uniqueness
- [decision] IMP-007/ADR-012: all gray-matter references marked as disqualified (CVE-2025-64718 in js-yaml@^3.13.1, 5+ years inactive maintainer). Replaced by yaml 2.x with manual 5-10 line frontmatter parser + Zod v4 validation. Historical references preserved with strikethrough notation. #security #dependencies #imp-007

## Relations

- depends_on [[ADR-007 CLI Architecture and Interaction Model]]
- depends_on [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ADR-003 Conflict Resolution and Namespacing]]
- relates_to [[ANALYSIS-022 @clack/prompts API Surface and Gaps]]
- relates_to [[ANALYSIS-023 gunshi Command Patterns and Capabilities]]
- relates_to [[ANALYSIS-020 Frontmatter and Markdown Processing]]
- depends_on [[ADR-012 Markdown Processing Pipeline]] (gray-matter disqualification, yaml 2.x replacement)
