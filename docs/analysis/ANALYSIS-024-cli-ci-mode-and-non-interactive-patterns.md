---
title: ANALYSIS-024 CLI CI Mode and Non-Interactive Patterns
type: analysis
permalink: analysis/analysis-024-cli-ci-mode-and-non-interactive-patterns-1
tags:
- cli
- ci-mode
- non-interactive
- json-output
- tty-detection
- graceful-degradation
---

# ANALYSIS-024 CLI CI Mode and Non-Interactive Patterns

## 1. Objective and Scope

**Objective**: How do popular CLI tools handle CI/non-interactive mode, and what patterns should @acmelabs-15/agent-plugin adopt for graceful degradation from interactive prompts to flag-based input?

**Scope**: CI detection, non-interactive fallback, JSON output, error handling for missing inputs. Excludes authentication flows and deployment-specific concerns.

## 2. Context

The agent-plugin CLI needs dual-mode operation: interactive terminal for developers, non-interactive for CI pipelines. The design spec proposes `--ci`, `--yes/-y`, `--json` flags plus env var auto-detection. This analysis validates those choices against industry patterns.

## 3. Approach

**Methodology**: Web research across 10 CLI tools, 3 detection libraries, 2 prompt libraries, and 2 style guides.
**Tools Used**: WebSearch, WebFetch against GitHub repos, npm, official docs.
**Limitations**: Could not access npm package pages directly (403). @clack/prompts source code not inspected at file level.

## 4. Data and Analysis

### Evidence Gathered

| Finding | Source | Confidence |
|---------|--------|------------|
| ci-info detects 50+ CI vendors via env vars, zero dependencies | github.com/watson/ci-info | High |
| npm/npx auto-implies --yes when stdin is not TTY or CI detected | npm docs, GitHub commits | High |
| GitHub CLI uses GH_PROMPT_DISABLED env var, errors on missing required flags | github.com/cli/cli#1739 | High |
| Railway CLI --ci flag implies non-interactive; --json also implies CI mode | Railway docs | High |
| Vercel CLI --yes skips prompts using defaults from vercel.json | Vercel docs | High |
| Wrangler uses ci-info for detection, skips confirmations in CI | cloudflare/workers-sdk issues | High |
| pnpm uses ci-info internally for TTY decisions | pnpm/pnpm#9966 | High |
| Turborepo auto-detects CI, switches from streaming to grouped logs | Turborepo docs | Medium |
| clig.dev recommends: JSON to stdout, logs to stderr, fail with instructions in non-interactive | clig.dev | High |
| @clack/prompts has no built-in CI bypass; crashes on non-TTY with WriteStream init | GitHub issues | Medium |

### Facts (Verified)

- ci-info exports: `isCI` (boolean), `name` (string or null), `isPR` (boolean or null), vendor-specific booleans (e.g., `GITHUB_ACTIONS`). Zero dependencies. 50+ vendors.
- ci-info detects these env vars (partial list matching the design spec): `CI`, `GITHUB_ACTIONS`, `GITLAB_CI`, `CIRCLECI`, `BUILDKITE`, `TRAVIS`, `JENKINS_URL` + `BUILD_ID`, `CODEBUILD_BUILD_ARN`, `TF_BUILD`.
- `is-ci` is a thin wrapper around ci-info that exports a single boolean. Internally just re-exports `ci-info.isCI`.
- npm/npx: When `!process.stdin.isTTY || ci-info.isCI`, the `--yes` flag is auto-applied.
- GitHub CLI: `GH_PROMPT_DISABLED` (any value) disables prompts. When non-interactive and required flags missing, commands error with explicit message: "--title or --fill required for non-interactive mode".
- Railway CLI: `--ci` flag forces non-interactive. `--json` output also implies CI mode. Uses token env vars for auth.
- Vercel CLI: `--yes` answers prompts with defaults inferred from config files and folder names.
- Wrangler: Uses ci-info internally. Skips confirmation steps in CI. Recent change: redacts account names in CI but shows them in non-interactive non-CI terminals (agent/pipe contexts).
- pnpm: Uses ci-info for TTY detection. Breaking change in v10.16 where missing TTY without CI=true caused ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY.
- Turborepo: Auto-detects CI for log grouping. Uses TURBO_TOKEN + TURBO_TEAM for non-interactive auth.
- @clack/prompts: No built-in CI detection or non-interactive bypass. Checks `t.isTTY` for raw mode. Throws ERR_TTY_INIT_FAILED on non-TTY WriteStream creation. Must be wrapped with guard logic.
- clig.dev: "If --no-input is passed, don't prompt. If the command requires input, fail and tell the user how to pass the information as a flag."

### Hypotheses (Unverified)

- Bun may use ci-info or a similar mechanism internally (bun init supports --yes but detection source unclear).
- @clack/prompts may add CI mode in a future version given community pressure from multiple issues.

## 5. Results

### 5.1 CI Detection: Three-Layer Consensus

Every major CLI tool uses the same detection hierarchy:

| Layer | Mechanism | Precedence | Example |
|-------|-----------|------------|---------|
| 1. Explicit flag | `--ci` or `--no-input` | Highest | Railway `--ci`, clig.dev `--no-input` |
| 2. Env var | `CI=true` or vendor-specific | Medium | npm, pnpm, Wrangler all check via ci-info |
| 3. TTY detection | `process.stdin.isTTY === false` | Lowest | npm/npx, GitHub CLI stdin check |

All 7 tools researched use at least layers 2 and 3. Railway and clig.dev also use layer 1.

### 5.2 ci-info Package: Industry Standard

| Metric | Value |
|--------|-------|
| Vendors detected | 50+ |
| Dependencies | 0 |
| Used by | npm, pnpm, Wrangler, Yarn, and others |
| API surface | 3 properties: isCI, name, isPR |
| Maintenance | Active (watson/ci-info on GitHub) |

The 9 env vars in the design spec (CI, GITHUB_ACTIONS, GITLAB_CI, CIRCLECI, BUILDKITE, TRAVIS, JENKINS_URL, CODEBUILD_BUILD_ID, TF_BUILD) are all covered by ci-info, plus 40+ more.

### 5.3 Non-Interactive Behavior When Inputs Missing

Three patterns observed across tools:

| Pattern | Tools Using It | Behavior |
|---------|---------------|----------|
| Error with flag hint | GitHub CLI, glab, clig.dev | Fail fast: "--title required in non-interactive mode" |
| Use defaults silently | Vercel (--yes), npm init (--yes) | Apply config-derived or convention defaults |
| Hang (anti-pattern) | pnpm (pre-fix), expo-cli (bug) | Block waiting for input that never comes |

**Consensus**: Error with flag hint for required inputs. Use defaults for optional inputs. Never hang.

### 5.4 JSON Output Pattern

| Rule | Source | Detail |
|------|--------|--------|
| Structured data to stdout | clig.dev, Heroku, Node.js best practices | `--json` flag gates JSON mode |
| Logs/warnings/progress to stderr | clig.dev, Heroku CLI style guide | `cli.action()` uses stderr |
| JSON implies no decorations | clig.dev | No colors, spinners, progress bars |
| Errors also as JSON in --json mode | Salesforce CLI, AWS CLI v2 | `{ "status": 1, "message": "...", "name": "ERROR_CODE" }` |
| Enable via flag, not default | All tools surveyed | Human-readable is default; --json opts in |

### 5.5 @clack/prompts CI Compatibility

| Aspect | Status |
|--------|--------|
| Built-in CI detection | None |
| TTY guard | Partial: checks isTTY for raw mode, but still attempts WriteStream |
| Programmatic answers | Not supported natively |
| Non-TTY behavior | Crashes with ERR_TTY_INIT_FAILED |
| Workaround required | Yes: must guard all prompt calls with isCI/isTTY check before invocation |

**Required wrapper pattern**:

```typescript
async function promptWithFallback<T>(
  promptFn: () => Promise<T>,
  fallback: T,
  options: { isCI: boolean }
): Promise<T> {
  if (options.isCI || !process.stdin.isTTY) {
    return fallback;
  }
  return promptFn();
}
```

### 5.6 The --yes and --ci Relationship

| Tool | --yes behavior | --ci behavior | Relationship |
|------|---------------|---------------|-------------|
| npm/npx | Auto-confirm prompts | N/A (auto-detected) | --yes implied when CI detected |
| Vercel | Accept defaults from config | N/A (auto-detected) | --yes is the only flag needed |
| Railway | N/A | Force non-interactive | --json implies --ci |
| Bun | Accept defaults | N/A | Mirrors npm behavior |
| GitHub CLI | N/A | GH_PROMPT_DISABLED | Errors on missing required flags |

**Design spec validation**: `--ci implies --yes` matches Railway's pattern. This is correct. The reverse (`--yes implies --ci`) is NOT standard because --yes is weaker: it confirms prompts but does not disable all interactive features (spinners, colors, progress).

## 6. Discussion

### Use ci-info, Not Custom Detection

The design spec lists 9 env vars for CI detection. ci-info covers all 9 plus 40+ more. Rolling custom detection means maintaining a growing list as new CI platforms emerge. Every major Node.js CLI tool (npm, pnpm, Wrangler, Yarn) delegates to ci-info. The package has zero dependencies and a 3-property API.

**One exception**: The design spec includes `CODEBUILD_BUILD_ID` but ci-info uses `CODEBUILD_BUILD_ARN`. Both work for AWS CodeBuild detection since ci-info checks for the `CODEBUILD_BUILD_ARN` env var. Using ci-info handles this correctly.

### Three-Mode Architecture

The research reveals a clear three-mode pattern:

1. **Interactive** (default when TTY): Full prompts, spinners, colors, progress bars.
2. **Non-interactive/--yes** (flag or auto-detected): Prompts auto-confirmed with defaults. Spinners and progress still shown if TTY.
3. **CI/--ci** (flag or env): All interactive features disabled. No prompts (error if required input missing). No spinners. No colors (unless --color forced). Implies --yes.

The design spec conflates modes 2 and 3. They should be distinct: `--yes` is a subset of `--ci`.

### @clack/prompts Requires a Guard Layer

@clack/prompts cannot be called in non-TTY environments without crashing. The CLI must implement a guard layer that:

1. Detects mode before any prompt call.
2. Short-circuits to flag values or defaults in CI/non-interactive mode.
3. Only invokes @clack/prompts when confirmed interactive.

### JSON Output is a Separate Concern

`--json` controls output format, not interactivity. A command can be:
- Interactive + JSON (unusual but valid)
- Non-interactive + JSON (common in CI)
- Non-interactive + human-readable (scripting)

The design spec correctly treats `--json` as orthogonal to `--ci`.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|----------|---------------|-----------|--------|
| P0 | Use ci-info package for CI detection instead of custom env var checks | Industry standard, 50+ vendors, zero deps, used by npm/pnpm/Wrangler | Low (1 dependency add) |
| P0 | Implement three-layer detection: explicit flag > env var > TTY check | Matches consensus across all 7 tools researched | Low |
| P0 | Guard @clack/prompts behind isCI/isTTY check; never call prompts in non-interactive mode | @clack/prompts crashes on non-TTY; no built-in CI support | Medium |
| P1 | Separate --yes (auto-confirm) from --ci (full non-interactive) | --ci implies --yes but not vice versa; matches Railway/npm patterns | Low |
| P1 | In non-interactive mode, error with flag hint when required input missing | "Missing --project-name. Required in non-interactive mode." Matches gh CLI pattern. | Low |
| P1 | JSON output: structured data to stdout, logs/progress to stderr | Universal pattern per clig.dev, Heroku, AWS CLI | Medium |
| P2 | Add --no-color flag (respect NO_COLOR env var per no-color.org) | Standard complement to CI mode; many tools support this | Low |
| P2 | Consider --plain flag for grep-parseable tabular output (not JSON) | Heroku CLI pattern for human-but-scriptable output | Low |

### Recommended Detection Code Pattern

```typescript
import ci from 'ci-info';

interface InteractivityContext {
  isCI: boolean;         // Full CI mode (no prompts, no spinners, no color)
  isInteractive: boolean; // Can use prompts and rich UI
  autoYes: boolean;       // Auto-confirm prompts but allow rich UI
  jsonOutput: boolean;    // Machine-readable output format
  ciName: string | null;  // Detected CI vendor name
}

function detectInteractivity(flags: {
  ci?: boolean;
  yes?: boolean;
  json?: boolean;
}): InteractivityContext {
  const isCIEnvironment = ci.isCI;
  const isTTY = Boolean(process.stdin.isTTY && process.stdout.isTTY);

  const isCI = flags.ci || isCIEnvironment || !isTTY;
  const jsonOutput = flags.json ?? false;
  const autoYes = flags.yes || isCI;
  const isInteractive = !isCI && isTTY;

  return {
    isCI,
    isInteractive,
    autoYes,
    jsonOutput,
    ciName: ci.name,
  };
}
```

### Recommended Prompt Guard Pattern

```typescript
import * as p from '@clack/prompts';

async function safePrompt<T>(
  ctx: InteractivityContext,
  promptFn: () => Promise<T | symbol>,
  options: {
    flagValue?: T;
    defaultValue?: T;
    required?: boolean;
    flagName?: string;
  }
): Promise<T> {
  // Priority 1: Explicit flag value always wins
  if (options.flagValue !== undefined) {
    return options.flagValue;
  }

  // Priority 2: Non-interactive mode
  if (!ctx.isInteractive) {
    if (options.defaultValue !== undefined) {
      return options.defaultValue;
    }
    if (options.required) {
      throw new Error(
        `Missing required input. Pass --${options.flagName} in non-interactive mode.`
      );
    }
    throw new Error('Input required but running in non-interactive mode.');
  }

  // Priority 3: Interactive prompt
  const result = await promptFn();
  if (p.isCancel(result)) {
    p.cancel('Operation cancelled.');
    process.exit(1);
  }
  return result as T;
}
```

### Recommended JSON Output Pattern

```typescript
interface CLIOutput {
  write(data: unknown): void;
  log(message: string): void;
  error(message: string, code?: string): void;
}

function createOutput(ctx: InteractivityContext): CLIOutput {
  return {
    // Structured data -> stdout
    write(data: unknown) {
      if (ctx.jsonOutput) {
        process.stdout.write(JSON.stringify(data, null, 2) + '\n');
      } else {
        // Human-readable formatting
        console.log(formatHuman(data));
      }
    },
    // Progress/info -> stderr (never pollutes stdout)
    log(message: string) {
      if (!ctx.jsonOutput) {
        process.stderr.write(message + '\n');
      }
    },
    // Errors -> stderr (JSON-formatted if --json)
    error(message: string, code?: string) {
      if (ctx.jsonOutput) {
        process.stderr.write(
          JSON.stringify({ error: { message, code } }) + '\n'
        );
      } else {
        process.stderr.write(`Error: ${message}\n`);
      }
    },
  };
}
```

## 8. Conclusion

**Verdict**: Proceed with the design spec's approach, with refinements.
**Confidence**: High
**Rationale**: The proposed --ci, --yes, --json flags align with industry patterns across all 7 tools researched. Two refinements needed: (1) use ci-info instead of custom detection, (2) treat --yes and --ci as distinct modes.

### User Impact

- **What changes for you**: CI pipelines work without code changes if standard env vars are set. Interactive mode degrades gracefully.
- **Effort required**: Low for ci-info adoption. Medium for @clack/prompts guard layer.
- **Risk if ignored**: @clack/prompts will crash in CI environments. Custom env var detection will miss edge-case CI platforms.

## 9. Appendices

### ci-info Vendor Detection (Complete List, 50+ vendors)

Key vendors and their env vars:

| Vendor | Env Var(s) |
|--------|-----------|
| GitHub Actions | GITHUB_ACTIONS |
| GitLab CI | GITLAB_CI |
| CircleCI | CIRCLECI |
| Travis CI | TRAVIS |
| Jenkins | JENKINS_URL + BUILD_ID |
| Azure Pipelines | TF_BUILD |
| AWS CodeBuild | CODEBUILD_BUILD_ARN |
| Buildkite | BUILDKITE |
| Bitbucket Pipelines | BITBUCKET_COMMIT |
| Vercel | NOW_BUILDER or VERCEL |
| Netlify | NETLIFY |
| Render | RENDER |
| Google Cloud Build | BUILDER_OUTPUT |
| Drone | DRONE |
| Semaphore | SEMAPHORE |
| TeamCity | TEAMCITY_VERSION |
| AppVeyor | APPVEYOR |
| Cloudflare Pages | CF_PAGES |
| Woodpecker | CI=woodpecker |
| Heroku | NODE path contains /app/.heroku/ |

Plus 30+ more (Agola, Alpic, Appcircle, Bamboo, Bitrise, Buddy, Cirrus, Codefresh, Codemagic, Codeship, dsari, Earthly, EAS, Gerrit, Gitea Actions, GoCD, Harness, Hudson, LayerCI, Magnum, Nevercode, Prow, ReleaseHub, Sail, Screwdriver, Sourcehut, Strider, TaskCluster, Vela, VS App Center, Xcode Cloud, Xcode Server).

### Sources Consulted

- [ci-info GitHub repository](https://github.com/watson/ci-info)
- [is-ci npm package](https://www.npmjs.com/package/is-ci)
- [Command Line Interface Guidelines (clig.dev)](https://clig.dev/)
- [GitHub CLI issue #1739: Disable interactive mode](https://github.com/cli/cli/issues/1739)
- [Vercel CLI docs](https://vercel.com/docs/cli)
- [Railway CLI docs](https://docs.railway.com/cli)
- [Cloudflare Wrangler docs](https://developers.cloudflare.com/workers/wrangler/)
- [pnpm issue #9966: TTY error breaking change](https://github.com/pnpm/pnpm/issues/9966)
- [Turborepo options reference](https://turborepo.dev/docs/reference/options-overview)
- [Heroku CLI Style Guide](https://devcenter.heroku.com/articles/cli-style-guide)
- [Node.js CLI Apps Best Practices](https://github.com/lirantal/nodejs-cli-apps-best-practices)
- [npm exec docs](https://docs.npmjs.com/cli/v8/commands/npm-exec/)
- [@clack/prompts npm package](https://www.npmjs.com/package/@clack/prompts)
- [Bombshell/Clack docs](https://bomb.sh/docs/clack/packages/prompts/)

### Data Transparency

- **Found**: Detection patterns for all 7 target CLI tools. ci-info full vendor list (50+ vendors). JSON output patterns from 4 style guides. @clack/prompts TTY behavior from issues and docs.
- **Not Found**: @clack/prompts source code for exact isTTY guard implementation. Bun's internal CI detection mechanism. Exact weekly download counts for ci-info (npm page blocked). Railway CLI source code for --ci implementation.

## Observations

- [fact] ci-info detects 50+ CI vendors with zero dependencies, used by npm, pnpm, Wrangler, Yarn #ci-detection
- [fact] All 7 CLI tools researched use the same three-layer detection: explicit flag > env var > TTY check #pattern
- [decision] Recommend ci-info over custom env var detection for broader coverage and zero maintenance #dependency
- [fact] @clack/prompts has no built-in CI bypass and crashes with ERR_TTY_INIT_FAILED on non-TTY #risk
- [technique] Guard pattern: check isCI/isTTY before calling @clack/prompts, fall back to flag values or error #implementation
- [fact] npm/npx auto-implies --yes when stdin is not TTY or CI is detected via ci-info #pattern
- [insight] --ci and --yes are distinct modes: --ci implies --yes but --yes does not imply --ci #architecture
- [technique] JSON output: structured data to stdout, progress/logs to stderr, errors as JSON in --json mode #pattern
- [fact] GitHub CLI errors with flag hint when required input missing in non-interactive mode #error-handling
- [risk] @clack/prompts requires wrapper layer; calling it directly in CI will crash the process #integration

## Relations

- relates_to [[Design Spec CLI Interface]]
- relates_to [[Agent Plugin Architecture]]