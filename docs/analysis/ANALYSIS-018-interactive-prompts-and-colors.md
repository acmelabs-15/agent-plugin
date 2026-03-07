---
title: ANALYSIS-018-interactive-prompts-and-colors
type: note
permalink: analysis/analysis-018-interactive-prompts-and-colors
tags:
- cli
- prompts
- colors
- terminal
- research
- dependencies
---

# ANALYSIS-018 Interactive Prompts and Terminal Colors

## 1. Objective and Scope

**Objective**: Evaluate interactive CLI prompt frameworks and terminal color libraries for the @acmelabz/agent-plugin CLI tool. Determine best fit for a cross-platform AI agent plugin manager with ~20 commands, wizard flows, and CI/non-interactive mode requirements.

**Scope**: Prompt frameworks (@clack/prompts, @inquirer/prompts, prompts, enquirer) and color libraries (picocolors, chalk, kleur, colorette, ansis). Excludes terminal UI frameworks (ink, blessed) and argument parsers.

## 2. Context

The CLI needs polished interactive UI for plugin management workflows (init, configure, install). It must support non-interactive mode for CI pipelines (ADR-002 requirement). It runs on Bun runtime and ships as an npm package. Visual quality matters because this tool represents the first user touchpoint for the plugin ecosystem.

## 3. Approach

**Methodology**: Web research of npm download statistics, GitHub repository data, Bun compatibility issues, API documentation, and community benchmarks.

**Tools Used**: WebSearch, WebFetch (bomb.sh docs)

**Limitations**: npm.js blocked direct page fetching (403). Some download numbers are approximate due to varying measurement dates. Bun compatibility status for some packages could not be verified to the exact current version.

## 4. Data and Analysis

---

### Part 1: Interactive Prompt Frameworks

#### Comparison Table

| Criteria | @clack/prompts | @inquirer/prompts | prompts (terkelg) | enquirer |
|---|---|---|---|---|
| **Version** | 1.1.0 | 8.3.0 | 2.4.2 | 2.4.1 |
| **Weekly Downloads** | ~2.5M | ~13.6M | ~25M (est.) | ~21.7M |
| **GitHub Stars** | 7.5k (bombshell-dev/clack) | 18.2k (SBoudrias/Inquirer.js) | 8.7k | 7.3k |
| **Last Published** | Mar 2026 (2 days ago) | Feb 2026 (11 days ago) | 2022 (4 years ago) | 2023 (3 years ago) |
| **TypeScript** | Native (written in TS) | Native (written in TS) | @types/prompts | @types/enquirer |
| **ESM Support** | Yes (dual CJS/ESM) | Yes (ESM) | CJS only | CJS only |
| **Dependencies** | @clack/core, picocolors, sisteransi, is-unicode-supported | Multiple @inquirer/* packages | kleur, sisteransi | ansi-colors |
| **License** | MIT | MIT | MIT | MIT |

#### Feature Comparison

| Feature | @clack/prompts | @inquirer/prompts | prompts | enquirer |
|---|---|---|---|---|
| text/input | Yes | Yes | Yes | Yes |
| password | Yes (masked) | Yes | Yes (invisible) | Yes |
| confirm | Yes | Yes | Yes (toggle) | Yes |
| select | Yes | Yes | Yes | Yes |
| multiselect | Yes | Yes (checkbox) | Yes | Yes |
| grouped multiselect | Yes (groupMultiselect) | No | No | No |
| autocomplete/search | No | Yes (search prompt) | Yes (autocomplete) | Yes |
| editor | No | Yes | No | No |
| spinner | Yes (built-in) | No (separate package) | No | No |
| tasks/progress | Yes (tasks()) | No | No | No |
| group/wizard flow | Yes (group()) | No built-in | No built-in | No built-in |
| intro/outro framing | Yes | No | No | No |
| note/log display | Yes (note, log.*) | No | No | No |
| AbortController | Yes (v1.1.0) | Yes | No | No |
| custom I/O streams | Yes (v1.1.0) | Yes | Yes | No |
| theming | Opinionated (consistent) | Yes (theme object) | No | Yes |
| visual polish | High (animated, styled) | Medium | Low-Medium | Medium |

#### Bun Compatibility Assessment

| Package | Bun Status | Issues | Severity |
|---|---|---|---|
| @clack/prompts | Partial, regressions | EPERM stdin in Bun 1.3.2, stuck with multiple prompts | HIGH |
| @inquirer/prompts | Improved since Bun 1.0.36 | Earlier AsyncResource issues, mostly resolved | MEDIUM |
| prompts | Generally works | Some readline crashes with heavy use (issue #10844) | LOW-MEDIUM |
| enquirer | Unknown/untested | No specific Bun issues found | UNKNOWN |

#### CI/Non-Interactive Mode

| Package | CI Support | Implementation |
|---|---|---|
| @clack/prompts | No built-in | Must detect CI env vars and skip prompts manually. Custom I/O streams (v1.1.0) enable testing. |
| @inquirer/prompts | No built-in | Same manual detection pattern needed. |
| prompts | Partial | onCancel handler, can inject answers programmatically |
| enquirer | Partial | Can set initial values, skip() method on some prompts |

None of the frameworks provide native CI/non-interactive mode. All require wrapping logic: detect `process.env.CI` or `--yes` flag, then use defaults instead of prompting.

#### Wizard Flow Evaluation

@clack/prompts has the strongest wizard flow support through `group()`:

```typescript
const config = await group({
  name: () => text({ message: 'Plugin name?' }),
  platform: ({ results }) => select({
    message: `Configure ${results.name}?`,
    options: [{ value: 'claude', label: 'Claude' }]
  }),
  confirm: () => confirm({ message: 'Create plugin?' })
}, {
  onCancel: () => { cancel('Setup cancelled'); process.exit(0); }
});
```

@inquirer/prompts requires manual sequential await calls with no built-in grouping. prompts supports similar sequential flows but without the visual framing (intro/outro/note).

---

### Part 2: Terminal Color Libraries

#### Comparison Table

| Criteria | picocolors | chalk | kleur | colorette | ansis |
|---|---|---|---|---|---|
| **Version** | 1.1.1 | 5.6.2 | 4.1.5 | 2.0.20 | 4.2.0 |
| **Weekly Downloads** | ~93.8M | ~349.5M | ~38.8M | ~41.7M | ~1M (est.) |
| **Bundle Size** | ~2.6 kB (min) | ~5 kB (v5) | ~3.4 kB | ~2.8 kB | ~3.8 kB |
| **node_modules Size** | 6.4 kB | ~40 kB (v5) | ~21 kB | ~14 kB | ~15 kB |
| **ESM Support** | Dual CJS/ESM | ESM only (v5) | Dual CJS/ESM | Dual CJS/ESM | Dual CJS/ESM |
| **TypeScript** | Built-in types | Built-in types | Built-in types | Built-in types | Built-in types |
| **Dependencies** | 0 | 0 (v5) | 0 | 0 | 0 |
| **Last Published** | 2024 | 2025 | 2023 | 2023 | 2025 (5 months ago) |
| **GitHub Stars** | ~3k | ~21.8k | ~1.7k | ~1.6k | ~500 |
| **License** | ISC | MIT | MIT | MIT | ISC |

#### Feature Comparison

| Feature | picocolors | chalk (v5) | kleur | colorette | ansis |
|---|---|---|---|---|---|
| Basic 16 colors | Yes | Yes | Yes | Yes | Yes |
| 256 colors | No | Yes | No | No | Yes |
| Truecolor (RGB/Hex) | No | Yes (hex, rgb) | No | No | Yes |
| Named truecolors | No | No | No | No | Yes (orange, pink, etc.) |
| Chained API | No | Yes | Yes | No | Yes |
| Nested styles | No (breaks) | Yes | Yes | No | Yes |
| Color detection | Basic (NO_COLOR, FORCE_COLOR) | Yes (automatic) | Basic | Basic | Yes (auto fallback) |
| Bun compatible | Yes | Yes | Yes | Yes | Yes (explicit) |
| Deno compatible | Partial | Yes | Yes | Partial | Yes (explicit) |
| Browser console | No | No | No | No | Yes |

#### Performance Rankings (benchmarks from ansis repo)

| Scenario | Fastest | Second | Third |
|---|---|---|---|
| Single style (e.g. red) | picocolors | ansis | kleur |
| Two+ styles (e.g. red + bold) | ansis | kleur | chalk |
| Complex nested styles | ansis | chalk | kleur |
| Real-world mixed usage | ansis ~= picocolors | kleur | chalk |

---

## 5. Results

### Prompt Framework Facts

- @clack/prompts is the only framework with built-in spinner, tasks, group(), intro/outro, and note display. Zero other frameworks bundle this complete CLI UX toolkit.
- @clack/prompts v1.1.0 shipped March 2026 with AbortController and custom I/O streams, indicating active maintenance.
- Bun compatibility for @clack/prompts has 4 open issues on oven-sh/bun (issues #3099, #4835, #7033, #24615). The EPERM regression in Bun 1.3.2 is unconfirmed resolved.
- prompts (terkelg) and enquirer have not shipped a release in 3-4 years. Both are effectively unmaintained.
- @inquirer/prompts has the largest prompt type catalog (input, select, checkbox, confirm, search, password, editor, rawlist, expand) but lacks visual framing and progress indicators.
- No prompt framework provides built-in CI/non-interactive mode. All require manual wrapping.

### Color Library Facts

- picocolors is 2.6 kB minified with zero dependencies, the smallest option, but lacks 256/truecolor, chaining, and proper nested style handling.
- ansis is 3.8 kB with zero dependencies and provides the richest feature set: truecolor, 256 colors, chaining, nesting, auto color detection with fallback, and explicit Bun/Deno/browser support.
- chalk v5 is ESM-only, which complicates CJS consumers. The project suffered a supply chain attack in September 2025 (npm hijack affecting 2B weekly downloads).
- kleur and colorette are both unmaintained (last published 2023) and lack advanced color features.
- picocolors is already a transitive dependency of @clack/prompts (used internally).

## 6. Discussion

### Prompt Framework

@clack/prompts is the clear leader for this use case. The built-in group() API maps directly to wizard flows (plugin init, platform config). The spinner and tasks APIs handle long-running operations (install, build). The intro/outro/note/log APIs provide consistent visual framing across all 20 commands.

The Bun compatibility risk is real but concentrated on the Bun side (regressions in Bun's stdin handling), not in @clack itself. The custom I/O streams added in v1.1.0 provide a path for testing and potential CI workarounds.

@inquirer/prompts is the strongest alternative. It has more prompt types (editor, search, rawlist) and a larger ecosystem. However, it requires assembling spinner, progress, and visual framing from separate packages, increasing integration complexity.

prompts and enquirer are eliminated by maintenance status alone. Both are 3-4 years without a release.

### Color Library

For a CLI with ~20 commands, the choice depends on whether you need just basic formatting or richer color support.

picocolors is sufficient if your color needs are limited to basic styling (red errors, green success, bold headers). It is already a transitive dependency via @clack/prompts, so adding it directly costs zero additional bytes.

ansis is the better choice if you want truecolor, 256 colors, chained API, or browser console support. The 1.2 kB size premium over picocolors buys substantially more capability. Its explicit Bun compatibility claim is a differentiator.

chalk is eliminated by its ESM-only constraint (complicates dual-format publishing) and the September 2025 supply chain attack (trust concern). kleur and colorette are eliminated by maintenance status.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|---|---|---|---|
| P0 | Use @clack/prompts for interactive CLI | Best wizard flow support (group), built-in spinner/tasks/framing, active maintenance, TypeScript native | Low (direct API mapping to use cases) |
| P0 | Implement CI/non-interactive wrapper | No framework provides this. Build a thin layer: detect CI env, --yes flag, use defaults | Low (utility function) |
| P1 | Use picocolors for terminal colors | Already a transitive dep of @clack/prompts, zero additional cost, sufficient for CLI status/error coloring | Trivial |
| P1 | Monitor Bun + @clack/prompts compatibility | Track oven-sh/bun issues #24615 and #7033. Pin Bun version in CI to known-good version (1.3.0 or earlier) | Ongoing |
| P2 | Evaluate ansis if color needs grow | If truecolor branding, 256-color themes, or browser console output become requirements, swap picocolors for ansis (3.8 kB, API-compatible pattern) | Low |

## 8. Conclusion

**Verdict**: Proceed with @clack/prompts + picocolors

**Confidence**: High (for prompts), High (for colors)

**Rationale**: @clack/prompts is the only framework that bundles wizard flows, spinners, tasks, and visual framing in a single actively-maintained TypeScript package. picocolors adds zero cost as a transitive dependency and covers basic color needs.

### User Impact

- **What changes for you**: Plugin init, configure, and install commands get polished interactive flows with spinners and progress. CI pipelines get non-interactive mode via wrapper.
- **Effort required**: Low for initial integration. CI wrapper is a utility function. Bun compatibility requires version pinning until upstream fixes land.
- **Risk if ignored**: Without a prompt framework decision, each command implements its own UX pattern, leading to inconsistent user experience across 20+ commands.

### Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Bun stdin regressions break @clack/prompts | Medium | High | Pin Bun version, test on CI with known-good version, custom I/O streams as fallback |
| @clack/prompts goes unmaintained | Low | Medium | v1.1.0 just shipped. @clack/core allows building custom prompts. @inquirer/prompts is drop-in alternative. |
| picocolors insufficient for branding needs | Low | Low | Swap to ansis (3.8 kB, same API pattern) |
| Supply chain attack on dependencies | Low | High | Lock dependency versions, use npm audit, consider socket.dev monitoring |

## 9. Appendices

### Observations

- [fact] @clack/prompts v1.1.0 has 2.5M weekly downloads, 7.5k GitHub stars, and shipped March 2026 #cli #prompts
- [fact] @clack/prompts is the only framework with built-in group(), tasks(), spinner(), intro/outro, note(), and log.* APIs #cli #ux
- [fact] Bun has 4 open issues affecting @clack/prompts stdin handling, most recent EPERM regression in Bun 1.3.2 #bun #risk
- [risk] No prompt framework provides built-in CI/non-interactive mode; all require manual wrapping #ci #non-interactive
- [fact] picocolors is 2.6 kB minified, zero dependencies, already a transitive dep of @clack/prompts #colors #size
- [fact] ansis is 3.8 kB with truecolor, 256 colors, chaining, auto color detection, and explicit Bun/Deno/browser support #colors #features
- [risk] chalk v5 suffered a supply chain attack in September 2025 affecting packages with 2B weekly downloads #security #supply-chain
- [insight] prompts (terkelg) and enquirer are effectively abandoned with no releases in 3-4 years #maintenance
- [fact] @inquirer/prompts has 13.6M weekly downloads and the widest prompt type catalog but lacks visual framing and progress #prompts
- [insight] @clack/prompts group() API maps directly to multi-step wizard flows needed for plugin init and platform config #wizard #ux

### Sources Consulted

- [npm: @clack/prompts](https://www.npmjs.com/package/@clack/prompts)
- [npm: @inquirer/prompts](https://www.npmjs.com/package/@inquirer/prompts)
- [npm: inquirer](https://www.npmjs.com/package/inquirer)
- [npm: prompts](https://www.npmjs.com/package/prompts)
- [npm: enquirer](https://www.npmjs.com/package/enquirer)
- [npm: picocolors](https://www.npmjs.com/package/picocolors)
- [npm: chalk](https://www.npmjs.com/package/chalk)
- [npm: ansis](https://www.npmjs.com/package/ansis)
- [npm: kleur](https://www.npmjs.com/package/kleur)
- [npm: colorette](https://www.npmjs.com/package/colorette)
- [Bombshell Clack Docs](https://bomb.sh/docs/clack/packages/prompts/)
- [GitHub: bombshell-dev/clack](https://github.com/bombshell-dev/clack)
- [GitHub: SBoudrias/Inquirer.js](https://github.com/SBoudrias/Inquirer.js)
- [GitHub: webdiscus/ansis](https://github.com/webdiscus/ansis)
- [GitHub: alexeyraspopov/picocolors](https://github.com/alexeyraspopov/picocolors)
- [Bun issue #24615: EPERM stdin with @clack/prompts](https://github.com/oven-sh/bun/issues/24615)
- [Bun issue #7033: @clack/prompts stuck with multiple prompts](https://github.com/oven-sh/bun/issues/7033)
- [Bun issue #4835: Terminal interaction breaks](https://github.com/oven-sh/bun/issues/4835)
- [Bun issue #4787: Inquirer.js doesn't work with Bun](https://github.com/oven-sh/bun/issues/4787)
- [npm supply chain attack September 2025](https://thehackernews.com/2025/09/20-popular-npm-packages-with-2-billion.html)

### Data Transparency

- **Found**: Download counts, GitHub stars, version history, Bun issue reports, API documentation, bundle sizes, feature lists, benchmark data
- **Not Found**: Exact resolution status of Bun 1.3.2 EPERM issue (may be fixed in later Bun versions but unconfirmed). Exact weekly downloads for prompts (terkelg) and ansis (estimates used). Enquirer Bun compatibility (no data).

## Relations

- relates_to [[ADR-002 CI Non-Interactive Mode]]
