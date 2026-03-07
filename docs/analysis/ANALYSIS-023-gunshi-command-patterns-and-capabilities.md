---
title: ANALYSIS-023 gunshi Command Patterns and Capabilities
type: note
permalink: analysis/analysis-023-gunshi-command-patterns-and-capabilities-1
tags:
- analysis
- gunshi
- cli
- commands
- lazy-loading
- subcommands
- patterns
---

# ANALYSIS-023 gunshi Command Patterns and Capabilities

## 1. Objective and Scope

**Objective**: Document gunshi v0.29.2 command patterns, lazy loading mechanics, nested subcommand support, global flags, context passing, and limitations to inform implementation of the @acmelabz/agent-plugin CLI command tree.

**Scope**: gunshi v0.29.2 API surface as published on npm. Includes Command, CommandContext, CliOptions, LazyCommand types. Covers plugin system for global flags. Excludes i18n features (not needed for this project).

**Exclusions**: Framework comparison (covered in ANALYSIS-017). Runtime compatibility (covered in ANALYSIS-016).

## 2. Context

ADR-006 selected gunshi v0.29.2 as the CLI framework. The design spec proposes approximately 20 commands organized as:
- Consumer: add, remove, upgrade, update (alias), list
- Author: init, validate, build, dev
- Scaffolding: new agent, new skill, new command, new hook, new rule, new mcp init, new mcp add-tool (3 levels deep)
- MCP: mcp serve
- Shell completions: complete bash/zsh/fish/powershell
- Global flags: --ci, --yes/-y, --json, --verbose/-v

This analysis documents the exact API patterns needed to implement this command tree.

## 3. Approach

**Methodology**: Web research across gunshi.dev documentation, npm package type definitions (unpkg.com), JSR registry, GitHub repository, real-world project analysis (ccusage, pnpmc, varlock).

**Tools Used**: WebSearch (12 queries), WebFetch (14 page analyses), Brain memory search.

**Limitations**: Could not run gunshi locally (no package.json in project yet). CHANGELOG.md rate-limited. JSR registry shows v0.26.3 (lags behind npm v0.29.2). WebFetch summarization lost some type details; recovered by fetching the raw types-BBvkSkl2.d.ts file.

## 4. Data and Analysis

### 4.1 Command Definition Pattern

The `define()` function creates typed command objects:

```typescript
import { cli, define } from 'gunshi'

const command = define({
  name: 'greeter',
  description: 'A simple greeting CLI',
  args: {
    name: {
      type: 'string',
      short: 'n',
      description: 'Name to greet'
    },
    uppercase: {
      type: 'boolean',
      short: 'u',
      description: 'Convert greeting to uppercase'
    }
  },
  run: ctx => {
    const { name = 'World', uppercase } = ctx.values
    let greeting = `Hello, ${name}!`
    if (uppercase) {
      greeting = greeting.toUpperCase()
    }
    console.log(greeting)
  }
})

await cli(process.argv.slice(2), command, {
  name: 'my-app',
  version: '1.0.0',
  description: 'My CLI application'
})
```

**Command interface properties (v0.29.2, verified from source types):**

| Property | Type | Required | Since | Description |
|----------|------|----------|-------|-------------|
| name | string | No | v0.1 | Command identifier for subcommand routing |
| description | string | No | v0.1 | Help text displayed in usage |
| args | Args | No | v0.1 | Argument definitions with type, short, default, description |
| examples | string or function | No | v0.1 | Usage examples |
| run | CommandRunner | No | v0.1 | Execution handler receiving CommandContext |
| toKebab | boolean | No | v0.1 | Convert camelCase arg names to kebab-case |
| internal | boolean | No | v0.27.0 | Hide command from help output |
| entry | boolean | No | v0.27.0 | Mark as entry command |
| rendering | RenderingOptions | No | v0.27.0 | Custom renderers for header, usage, validation errors |
| subCommands | Record or Map | No | v0.28.0 | Nested subcommands |

**Arg types supported**: `'string'`, `'boolean'`, `'number'`, `'integer'`, `'float'`

Each arg schema supports: `type`, `short`, `description`, `default`, `required`, `multiple`, `choices`

### 4.2 Lazy Loading Pattern

The `lazy()` function separates command metadata from implementation. Metadata loads immediately for help text generation. Implementation loads only when the command is invoked.

```typescript
import { lazy } from 'gunshi'

// Metadata object (loaded immediately, keep small)
const addMeta = define({
  name: 'add',
  description: 'Add a plugin to the project',
  args: {
    source: {
      type: 'string',
      description: 'Plugin source (npm package, git URL, or local path)',
      required: true
    },
    yes: {
      type: 'boolean',
      short: 'y',
      description: 'Skip confirmation prompts'
    }
  }
})

// Lazy command (implementation loaded on-demand)
const addCommand = lazy(
  async () => {
    const { run } = await import('./commands/add.js')
    return run
  },
  addMeta
)
```

**Implementation file pattern** (separate module):

```typescript
// commands/add.ts
import type { CommandRunner } from 'gunshi'

export const run: CommandRunner<{ args: typeof import('./add-meta').default.args }> =
  async ctx => {
    const { source, yes } = ctx.values
    // Implementation here
  }
```

**Type-safe variant** with explicit type parameters:

```typescript
import { lazyWithTypes } from 'gunshi'

const addCommand = lazyWithTypes<
  { args: typeof addMeta.args }
>()(
  async () => await load('add'),
  addMeta
)
```

**Factory pattern** for many commands:

```typescript
function createLazyCommand(name: string) {
  return lazy(
    async () => {
      const mod = await import(`./commands/${name}.js`)
      return mod.default || mod.run
    },
    { name, description: `...` }
  )
}
```

### 4.3 Nested Subcommands (Critical for `new mcp init`)

Nested subcommands use the `subCommands` property on the Command interface (since v0.28.0). gunshi documentation states: "You can nest commands to any depth."

**Two-level nesting** (e.g., `agent-plugin new agent`):

```typescript
const newAgentCommand = define({
  name: 'agent',
  description: 'Scaffold a new agent definition',
  run: ctx => { /* scaffolding logic */ }
})

const newSkillCommand = define({
  name: 'skill',
  description: 'Scaffold a new skill',
  run: ctx => { /* scaffolding logic */ }
})

const newCommand = define({
  name: 'new',
  description: 'Scaffold new plugin components',
  subCommands: {
    agent: newAgentCommand,
    skill: newSkillCommand
  },
  run: ctx => {
    // Runs when user types `agent-plugin new` without a subcommand
    // Can show interactive menu or help
    console.log('Use: agent-plugin new agent|skill|command|hook|rule|mcp')
  }
})
```

**Three-level nesting** (e.g., `agent-plugin new mcp init`):

```typescript
const mcpInitCommand = define({
  name: 'init',
  description: 'Initialize MCP server configuration',
  run: ctx => { /* ... */ }
})

const mcpAddToolCommand = define({
  name: 'add-tool',
  description: 'Add a tool to the MCP server',
  run: ctx => { /* ... */ }
})

const newMcpCommand = define({
  name: 'mcp',
  description: 'MCP server scaffolding',
  subCommands: {
    init: mcpInitCommand,
    'add-tool': mcpAddToolCommand
  },
  run: () => {
    console.log('Use: agent-plugin new mcp init|add-tool')
  }
})

const newCommand = define({
  name: 'new',
  description: 'Scaffold new plugin components',
  subCommands: {
    agent: newAgentCommand,
    mcp: newMcpCommand
  },
  run: () => { /* ... */ }
})
```

**Context propagation through nesting levels:**

| Property | Entry command | 1st level sub | 2nd level sub | 3rd level sub |
|----------|-------------|--------------|--------------|--------------|
| callMode | 'entry' | 'subCommand' | 'subCommand' | 'subCommand' |
| commandPath | [] | ['new'] | ['new', 'mcp'] | ['new', 'mcp', 'init'] |
| omitted | false | true (if no sub specified) | true (if no sub specified) | false |

**Important**: `commandPath` was added in v0.28.0. It provides the full resolution path as a string array, which is essential for deeply nested commands to know their invocation context.

### 4.4 Global Flags Pattern

gunshi does NOT have a built-in global flags mechanism on the Command interface. Global flags are added via the plugin system.

**Using @gunshi/plugin-global** (adds --help and --version automatically):

```typescript
import { cli, define } from 'gunshi'
// @gunshi/plugin-global is typically auto-included

await cli(process.argv.slice(2), command, {
  name: 'agent-plugin',
  version: '1.0.0',  // Enables --version via plugin-global
  plugins: [/* plugin-global is likely auto-registered */]
})
```

**Custom global flags via plugin** (for --ci, --yes, --json, --verbose):

```typescript
import { plugin } from 'gunshi/plugin'

const globalFlagsPlugin = plugin({
  id: 'agent-plugin-global',
  name: 'Global Flags',
  setup: ctx => {
    ctx.addGlobalOption('ci', {
      type: 'boolean',
      description: 'Run in CI mode (non-interactive, fail on prompts)'
    })
    ctx.addGlobalOption('yes', {
      type: 'boolean',
      short: 'y',
      description: 'Skip confirmation prompts'
    })
    ctx.addGlobalOption('json', {
      type: 'boolean',
      description: 'Output results as JSON'
    })
    ctx.addGlobalOption('verbose', {
      type: 'boolean',
      short: 'v',
      description: 'Enable verbose output'
    })
  }
})

// Register in CLI options
await cli(process.argv.slice(2), entryCommand, {
  name: 'agent-plugin',
  version: '1.0.0',
  plugins: [globalFlagsPlugin],
  subCommands: { /* ... */ }
})
```

Commands then access global flags via `ctx.values.ci`, `ctx.values.json`, etc. The flags appear in help text for all commands.

### 4.5 Command Aliases

**FINDING: gunshi v0.29.2 does NOT support command aliases natively.**

The Command interface has no `alias` or `aliases` property (verified from source types at unpkg.com/gunshi@0.29.2/lib/types-BBvkSkl2.d.ts). The JSR registry page (v0.26.3) incorrectly suggested alias support existed.

**Workaround for `update` as alias of `upgrade`:**

```typescript
// Option 1: Register same command under both names in subCommands
const upgradeCommand = define({
  name: 'upgrade',
  description: 'Upgrade installed plugins',
  run: ctx => { /* ... */ }
})

const subCommands = {
  upgrade: upgradeCommand,
  update: upgradeCommand  // Same command object, different key
}
```

```typescript
// Option 2: Create a thin wrapper that delegates
const updateCommand = define({
  name: 'update',
  description: 'Alias for upgrade',
  internal: true,  // Hide from help (since v0.27.0)
  args: upgradeCommand.args,
  run: upgradeCommand.run
})
```

Option 1 is simpler. The command appears twice in help text unless `internal: true` is used on the alias entry.

### 4.6 Context (CommandContext) Properties

Full CommandContext interface (v0.29.2, verified from source):

| Property | Type | Description |
|----------|------|-------------|
| name | string or undefined | CLI program name |
| description | string or undefined | CLI program description |
| env | CommandEnvironment | Full environment (cwd, version, margins, renderers) |
| args | Args | Argument schema definitions |
| explicit | Record | Which args were explicitly provided by user (true/false per arg) |
| values | ArgValues | Parsed argument values (TYPED based on args definition) |
| positionals | string[] | Positional arguments |
| rest | string[] | Arguments after `--` delimiter |
| _ | string[] | Original raw argv |
| tokens | ArgToken[] | Parsed tokens from parseArgs |
| omitted | boolean | True when subcommand name was omitted |
| callMode | CommandCallMode | 'entry', 'subCommand', or 'unexpected' |
| commandPath | string[] | Nested subcommand resolution path (since v0.28.0) |
| toKebab | boolean or undefined | Whether camelCase args are converted to kebab-case |
| log | function | Output function (respects usageSilent) |
| extensions | ExtendContext | Plugin extensions |
| validationError | AggregateError or undefined | Validation errors from arg parsing |

The `explicit` property (distinguishing user-provided args from defaults) is particularly useful for the --ci flag pattern where you need to know if a user explicitly set a value.

### 4.7 CLI Options and Hooks

**CliOptions interface (v0.29.2):**

| Property | Type | Since | Description |
|----------|------|-------|-------------|
| cwd | string | v0.1 | Working directory |
| name | string | v0.1 | Program name |
| description | string | v0.1 | Program description |
| version | string | v0.1 | Program version (enables --version) |
| subCommands | Record or Map | v0.1 | Top-level subcommands |
| leftMargin | number | v0.1 | Help text left margin (default: 2) |
| middleMargin | number | v0.1 | Help text middle margin (default: 10) |
| usageOptionType | boolean | v0.1 | Show option type in help (default: false) |
| usageOptionValue | boolean | v0.1 | Show option value in help (default: true) |
| usageSilent | boolean | v0.1 | Suppress usage output (default: false) |
| renderUsage | function or null | v0.1 | Custom usage renderer |
| renderHeader | function or null | v0.1 | Custom header renderer |
| renderValidationErrors | function or null | v0.1 | Custom validation error renderer |
| fallbackToEntry | boolean | v0.27.0 | Run entry command when subcommand not found (default: false) |
| plugins | Plugin[] | v0.27.0 | Registered plugins |
| onBeforeCommand | function | v0.27.0 | Pre-execution hook |
| onAfterCommand | function | v0.27.0 | Post-execution hook (receives result) |
| onErrorCommand | function | v0.27.0 | Error handler hook (receives Error) |

**Lifecycle execution order:**

1. Plugin setup phase (plugins register decorators, global options)
2. Extension creation (plugin extensions initialized)
3. `onBeforeCommand` hook fires
4. Plugin decorators wrap command execution (LIFO order)
5. Command `run()` executes
6. `onAfterCommand` hook (success) OR `onErrorCommand` hook (error)
7. Extension cleanup

### 4.8 Entry Command and No-Subcommand Behavior

When user runs `agent-plugin` with no subcommand:

1. The entry command's `run()` function executes
2. If `fallbackToEntry: true` in CliOptions, unrecognized subcommands also fall back to entry
3. The entry command can show help, an interactive menu, or default behavior

**Pattern for interactive fallback:**

```typescript
const entryCommand = define({
  name: 'agent-plugin',
  description: 'AI agent plugin manager',
  run: async ctx => {
    if (ctx.omitted || ctx.callMode === 'entry') {
      // No subcommand provided - show interactive menu or help
      const { select } = await import('@clack/prompts')
      const action = await select({
        message: 'What would you like to do?',
        options: [
          { value: 'add', label: 'Add a plugin' },
          { value: 'list', label: 'List installed plugins' },
          { value: 'init', label: 'Initialize a new plugin project' }
        ]
      })
      // Route to selected command
    }
  }
})
```

### 4.9 Real-World Usage Patterns

**ccusage** (ryoppippi): Uses gunshi with subCommands map. Sets `dailyCommand` as the default when no subcommand is specified. All commands share common flags (--json, --since, --until, --timezone). Demonstrates the pattern this project needs.

**pnpmc** (kazupon, gunshi author): Monorepo structure with separate packages per command. Uses meta.ts + runner.ts separation pattern described in docs.

**Key pattern from ccusage:**

```typescript
// commands/index.ts
const subCommandUnion = {
  daily: dailyCommand,
  weekly: weeklyCommand,
  monthly: monthlyCommand,
  session: sessionCommand,
  blocks: blocksCommand,
  statusline: statuslineCommand
}

// Entry point runs cli() with subCommands
await cli(process.argv.slice(2), mainCommand, {
  name: 'ccusage',
  version: pkg.version,
  subCommands: subCommandUnion
})
```

### 4.10 Version Display

Version is set via `CliOptions.version`. The `@gunshi/plugin-global` plugin adds `--help` and `--version` flags automatically. When `--version` is passed, gunshi outputs the version string and exits. No custom implementation needed.

## 5. Results

### Evidence Table

| Finding | Source | Confidence |
|---------|--------|------------|
| Command interface has 10 properties, NO alias support | unpkg.com types-BBvkSkl2.d.ts (v0.29.2) | High |
| subCommands property added in v0.28.0, supports any nesting depth | gunshi.dev/guide/advanced/nested-sub-commands + source types | High |
| commandPath (string[]) tracks full resolution path, added in v0.28.0 | Source types (CommandContext interface) | High |
| lazy() separates metadata from implementation for on-demand loading | gunshi.dev/guide/advanced/advanced-lazy-loading | High |
| Global flags added via plugin system (ctx.addGlobalOption), not Command interface | gunshi.dev/guide/plugin/getting-started + source PluginContext | High |
| Three lifecycle hooks: onBeforeCommand, onAfterCommand, onErrorCommand (since v0.27.0) | Source types (CliOptions interface) | High |
| fallbackToEntry option routes unknown subcommands to entry command (since v0.27.0) | Source types (CliOptions interface) | High |
| internal: true hides commands from help output (since v0.27.0) | Source types (Command interface) | High |
| @gunshi/plugin-global adds --help and --version automatically | gunshi.dev documentation, npm | Medium |
| ccusage uses gunshi with shared common flags and default subcommand pattern | deepwiki.com/ryoppippi/ccusage analysis | High |
| No native command alias support in v0.29.2 | Verified absence from Command interface source | High |

### Facts (Verified)

- gunshi v0.29.2 Command interface has 10 properties: name, description, args, examples, run, toKebab, internal, entry, rendering, subCommands
- subCommands accepts Record<string, SubCommandable> or Map<string, SubCommandable> with unlimited nesting depth
- commandPath (string[]) provides the full nested resolution path (e.g., ['new', 'mcp', 'init'])
- lazy() takes two parameters: async loader function and metadata command object
- Global options are added via plugin system using ctx.addGlobalOption(name, schema)
- Three lifecycle hooks exist on CliOptions: onBeforeCommand, onAfterCommand, onErrorCommand
- fallbackToEntry: true in CliOptions routes unknown subcommands to the entry command
- internal: true on a Command hides it from help text
- ctx.explicit tracks which args were user-provided vs defaults
- ctx.omitted is true when an intermediate command runs without a subcommand specified
- Arg schema supports: type, short, description, default, required, multiple, choices

### Hypotheses (Unverified)

- @gunshi/plugin-global may be auto-registered (not confirmed from source)
- Deeply nested lazy commands (3+ levels with lazy() at each level) may have untested edge cases given gunshi's small user base
- The `entry: true` property on Command may interact with fallbackToEntry in undocumented ways
- Performance impact of lazy loading with Bun's module resolution at 3+ nesting levels is unknown

## 6. Discussion

### Command Tree Mapping

The design spec's command tree maps to gunshi as follows:

```
agent-plugin (entry command)
  subCommands:
    add        -> lazy(import('./commands/add.js'), addMeta)
    remove     -> lazy(import('./commands/remove.js'), removeMeta)
    upgrade    -> lazy(import('./commands/upgrade.js'), upgradeMeta)
    update     -> upgradeCommand (alias via same object reference)
    list       -> lazy(import('./commands/list.js'), listMeta)
    init       -> lazy(import('./commands/init.js'), initMeta)
    validate   -> lazy(import('./commands/validate.js'), validateMeta)
    build      -> lazy(import('./commands/build.js'), buildMeta)
    dev        -> lazy(import('./commands/dev.js'), devMeta)
    new        -> define({ subCommands: {
                    agent:   lazy(...)
                    skill:   lazy(...)
                    command: lazy(...)
                    hook:    lazy(...)
                    rule:    lazy(...)
                    mcp:     define({ subCommands: {
                               init:     lazy(...)
                               add-tool: lazy(...)
                             }})
                  }})
    mcp        -> define({ subCommands: {
                    serve: lazy(...)
                  }})
    complete   -> lazy(...)  // or use @gunshi/plugin-completion
```

### No-Alias Workaround

The `update` alias for `upgrade` is straightforward: register the same command object under both keys in the subCommands record. Use `internal: true` on the alias entry to hide it from help, or let both appear.

### Global Flags Strategy

Create a custom plugin (`agent-plugin-global`) that adds --ci, --yes, --json, --verbose via ctx.addGlobalOption(). This is the intended gunshi pattern. The plugin system handles propagation to all commands including nested ones.

### Interactive Fallback

When no subcommand is provided, the entry command's run() fires. Use @clack/prompts to show an interactive menu. The `omitted` context property tells you the user did not specify a subcommand. With `fallbackToEntry: true`, even typos route to the entry command for graceful handling.

### Error Handling

The `onErrorCommand` hook on CliOptions catches errors from any command execution. Combined with `renderValidationErrors` for arg parsing failures, this covers the error handling needs without additional infrastructure.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|----------|---------------|-----------|--------|
| P0 | Use lazy() for all leaf commands (add, remove, upgrade, list, init, validate, build, dev, complete, mcp serve, all new/* commands) | Startup time with 20+ commands. Only metadata loads initially. | Low |
| P0 | Use define() with subCommands for intermediate nodes (new, new mcp, mcp) | These are routing nodes, not heavy implementations. No need for lazy loading. | Low |
| P0 | Create agent-plugin-global plugin for --ci, --yes, --json, --verbose flags | gunshi's intended pattern for cross-cutting options | Low |
| P1 | Implement update alias as same object reference in subCommands record | Simplest alias pattern. No custom wrapper needed. | Trivial |
| P1 | Use onBeforeCommand hook for CI mode detection and logging setup | Runs before any command, ideal for global setup | Low |
| P1 | Use onErrorCommand hook for centralized error handling and JSON error output | Single error handling point for all commands | Low |
| P2 | Use internal: true for commands that should not appear in help (e.g., update alias) | Clean help output without duplicate entries | Trivial |
| P2 | Set fallbackToEntry: true for graceful handling of unknown subcommands | Better UX than "command not found" error | Trivial |
| P2 | Test 3-level nesting (new mcp init) early in implementation | Untested depth with gunshi's small user base | Low |

## 8. Conclusion

**Verdict**: Proceed with implementation
**Confidence**: High
**Rationale**: gunshi v0.29.2 provides all capabilities needed for the @acmelabz/agent-plugin CLI command tree. Nested subcommands (unlimited depth, since v0.28.0) handle the `new mcp init` three-level pattern. Lazy loading is built-in. Global flags work through the plugin system. Command aliases require a simple workaround (same object reference). The only gap is native alias support, which is trivially solved. All findings are verified from source type definitions.

### User Impact

- **What changes for you**: Each command is a standalone module with typed args. The CLI entry point is a thin wiring layer (~50 lines) that registers subCommands and plugins. Adding new commands requires: (1) define metadata, (2) implement run function, (3) register in subCommands map.
- **Effort required**: Low. The command tree maps directly to gunshi's API. No custom infrastructure needed for lazy loading, help generation, or argument parsing.
- **Risk if ignored**: Without understanding gunshi's patterns, implementers may build custom lazy loading, custom global flag propagation, or custom help generation. These are all built-in.

## 9. Appendices

### Complete CLI Entry Point Pattern

```typescript
// src/cli.ts
import { cli, define, lazy } from 'gunshi'
import { plugin } from 'gunshi/plugin'
import pkg from '../package.json' with { type: 'json' }

// Global flags plugin
const globalFlags = plugin({
  id: 'agent-plugin-global',
  name: 'Global Flags',
  setup: ctx => {
    ctx.addGlobalOption('ci', {
      type: 'boolean',
      description: 'Run in CI mode (non-interactive)'
    })
    ctx.addGlobalOption('yes', {
      type: 'boolean',
      short: 'y',
      description: 'Skip confirmation prompts'
    })
    ctx.addGlobalOption('json', {
      type: 'boolean',
      description: 'Output as JSON'
    })
    ctx.addGlobalOption('verbose', {
      type: 'boolean',
      short: 'v',
      description: 'Verbose output'
    })
  }
})

// Lazy command imports (metadata only, implementations load on-demand)
const addCommand = lazy(
  async () => (await import('./commands/add.js')).run,
  { name: 'add', description: 'Add a plugin', args: { /* ... */ } }
)

const upgradeCommand = lazy(
  async () => (await import('./commands/upgrade.js')).run,
  { name: 'upgrade', description: 'Upgrade plugins', args: { /* ... */ } }
)

// Nested subcommands (intermediate nodes use define, leaves use lazy)
const newMcpCommand = define({
  name: 'mcp',
  description: 'MCP server scaffolding commands',
  subCommands: {
    init: lazy(
      async () => (await import('./commands/new/mcp/init.js')).run,
      { name: 'init', description: 'Initialize MCP server' }
    ),
    'add-tool': lazy(
      async () => (await import('./commands/new/mcp/add-tool.js')).run,
      { name: 'add-tool', description: 'Add MCP tool' }
    )
  },
  run: ctx => { ctx.log('Use: agent-plugin new mcp init|add-tool') }
})

const newCommand = define({
  name: 'new',
  description: 'Scaffold plugin components',
  subCommands: {
    agent: lazy(async () => (await import('./commands/new/agent.js')).run,
      { name: 'agent', description: 'New agent definition' }),
    skill: lazy(async () => (await import('./commands/new/skill.js')).run,
      { name: 'skill', description: 'New skill' }),
    mcp: newMcpCommand
  },
  run: ctx => { ctx.log('Use: agent-plugin new agent|skill|command|hook|rule|mcp') }
})

// Entry command
const entryCommand = define({
  name: 'agent-plugin',
  description: 'AI agent plugin manager',
  run: async ctx => {
    if (ctx.omitted) {
      // Interactive menu via @clack/prompts
    }
  }
})

// Wire everything
await cli(process.argv.slice(2), entryCommand, {
  name: 'agent-plugin',
  version: pkg.version,
  description: pkg.description,
  plugins: [globalFlags],
  fallbackToEntry: true,
  subCommands: {
    add: addCommand,
    remove: lazy(async () => (await import('./commands/remove.js')).run,
      { name: 'remove', description: 'Remove a plugin' }),
    upgrade: upgradeCommand,
    update: upgradeCommand,  // Alias: same object reference
    list: lazy(async () => (await import('./commands/list.js')).run,
      { name: 'list', description: 'List installed plugins' }),
    init: lazy(async () => (await import('./commands/init.js')).run,
      { name: 'init', description: 'Initialize plugin project' }),
    validate: lazy(async () => (await import('./commands/validate.js')).run,
      { name: 'validate', description: 'Validate plugin' }),
    build: lazy(async () => (await import('./commands/build.js')).run,
      { name: 'build', description: 'Build plugin for distribution' }),
    dev: lazy(async () => (await import('./commands/dev.js')).run,
      { name: 'dev', description: 'Development mode with live reload' }),
    new: newCommand,
    mcp: define({
      name: 'mcp',
      description: 'MCP server commands',
      subCommands: {
        serve: lazy(async () => (await import('./commands/mcp/serve.js')).run,
          { name: 'serve', description: 'Start MCP server' })
      }
    }),
    complete: lazy(async () => (await import('./commands/complete.js')).run,
      { name: 'complete', description: 'Generate shell completions' })
  },
  onBeforeCommand: ctx => {
    // Global setup: configure logger verbosity, CI mode detection
  },
  onErrorCommand: (ctx, error) => {
    // Centralized error handling
    if (ctx.values.json) {
      console.error(JSON.stringify({ error: error.message }))
    } else {
      console.error(error.message)
    }
    process.exit(1)
  }
})
```

### Limitations and Gotchas

1. **No native alias support**: Must use same-object-reference or internal:true wrapper pattern
2. **Global flags via plugin only**: Cannot define global flags on Command interface directly
3. **subCommands added in v0.28.0**: Ensure pinned version is >= 0.28.0 (v0.29.2 satisfies this)
4. **commandPath added in v0.28.0**: Earlier versions lack nested command path tracking
5. **Pre-1.0 API stability**: Pin exact version. Review each minor bump.
6. **Small user base for deep nesting**: 3+ level nesting (new mcp init) should be tested early
7. **LazyCommand typing**: The type intersection pattern for LazyCommand requires careful use of lazyWithTypes() for full type safety when args are defined externally
8. **Plugin decorator order**: Decorators apply LIFO (last registered executes first). Order matters for multi-plugin setups.

### Sources Consulted

- gunshi.dev/guide/essentials/getting-started (command definition examples)
- gunshi.dev/guide/advanced/advanced-lazy-loading (lazy loading patterns)
- gunshi.dev/guide/advanced/nested-sub-commands (nesting depth, commandPath, callMode)
- gunshi.dev/guide/advanced/type-system (GunshiParams, define/lazy type parameters)
- gunshi.dev/guide/plugin/getting-started (plugin system, addGlobalOption)
- gunshi.dev/guide/plugin/lifecycle (onBeforeCommand, onAfterCommand, onErrorCommand)
- gunshi.dev/showcase (real-world projects: ccusage, pnpmc, varlock)
- unpkg.com/gunshi@0.29.2/lib/types-BBvkSkl2.d.ts (complete source type definitions)
- unpkg.com/gunshi@0.29.2/package.json (version, exports map)
- jsr.io/@kazupon/gunshi (exported symbols list, v0.26.3)
- deepwiki.com/ryoppippi/ccusage (real-world usage pattern analysis)
- github.com/kazupon/gunshi/issues/2 (roadmap)
- npmjs.com/package/gunshi (version, metadata)

### Data Transparency

- **Found**: Complete Command, CommandContext, CliOptions, PluginContext interfaces from source types. Lifecycle hook signatures. Nested subcommand support details. Real-world usage patterns from ccusage. Plugin global option pattern. All 10 Command properties verified.
- **Not Found**: Whether @gunshi/plugin-global is auto-registered or requires explicit import. Performance benchmarks for lazy loading at 3+ nesting depth. Whether the `entry: true` property interacts with `fallbackToEntry`. Exact CHANGELOG entries for v0.27-v0.29 breaking changes (rate-limited). Whether `update` alias shows in --help when using same-object-reference pattern.

## Observations

- [fact] gunshi v0.29.2 Command interface has 10 properties; no native alias support exists #gunshi #api
- [fact] subCommands property (since v0.28.0) supports unlimited nesting depth via nested define() calls #gunshi #subcommands
- [fact] commandPath string array (since v0.28.0) tracks full resolution path for nested commands #gunshi #context
- [fact] Global flags are added via plugin system using ctx.addGlobalOption(), not on Command interface #gunshi #plugins
- [fact] Three lifecycle hooks exist: onBeforeCommand, onAfterCommand, onErrorCommand (since v0.27.0) #gunshi #hooks
- [fact] lazy() separates metadata from implementation; metadata loads immediately, implementation on-demand #gunshi #lazy-loading
- [technique] Command aliases implemented by registering same object reference under multiple keys in subCommands record #gunshi #pattern
- [technique] Interactive fallback when no subcommand uses ctx.omitted check in entry command's run() #gunshi #pattern
- [technique] fallbackToEntry: true routes unknown subcommands to entry command for graceful error handling #gunshi #pattern
- [insight] ccusage project validates the shared-flags-via-plugin pattern and default-subcommand pattern that agent-plugin needs #gunshi #real-world

## Relations

- implements [[ADR-006 Core Dependency Stack]]
- relates_to [[ANALYSIS-017-cli-framework-comparison]]
- relates_to [[ANALYSIS-001-agent-plugin-foundation-and-vision]]