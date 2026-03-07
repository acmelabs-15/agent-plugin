---
title: ANALYSIS-020 Frontmatter and Markdown Processing Libraries
type: note
permalink: analysis/analysis-020-frontmatter-and-markdown-processing
tags:
- analysis
- frontmatter
- markdown
- libraries
- dependencies
- bun
- typescript
---

# ANALYSIS-020 Frontmatter and Markdown Processing Libraries

## 1. Objective and Scope

**Objective**: Evaluate frontmatter parsing and markdown processing libraries for `@acmelabs-15/agent-plugin`, determining which libraries to adopt and whether full markdown processing is needed.

**Scope**: Frontmatter parsers (gray-matter, front-matter, manual YAML parsing). Markdown processors (micromark, remark/unified, markdown-it, marked). Bundle size, maintenance, security, Bun compatibility, TypeScript support. Excludes HTML rendering frameworks.

## 2. Context

The tool parses SKILL.md and AGENT.md files with YAML frontmatter containing metadata (name, description, type, requires, sources, platformConfig). The manifest itself is JSON (plugin.json, per ADR-001), but component files use markdown with YAML frontmatter. Vercel Skills uses gray-matter for this purpose (ANALYSIS-003). Runtime is Bun (ANALYSIS-016). Validation uses Zod v4.

Key question: Does this project need full markdown AST processing, or just frontmatter extraction plus pass-through of markdown body text?

## 3. Approach

**Methodology**: Web research across npm registry, GitHub repositories, bundlephobia, npm-compare, package.json inspection, security advisory databases, and community discussions.
**Tools Used**: WebSearch (18 queries), WebFetch (8 pages), Brain memory search, GitHub raw content fetch.
**Limitations**: Bundlephobia uses client-side rendering so exact minified+gzipped sizes could not be scraped programmatically. Sizes are gathered from GitHub discussions, README claims, and community benchmarks. Some figures are approximate.

## 4. Data and Analysis

### 4.1 Frontmatter Library Comparison

| Metric | gray-matter 4.0.3 | front-matter 4.0.2 | Manual (yaml 2.x + split) |
|--------|-------------------|--------------------|-----------------------------|
| Weekly downloads | ~1.9M | ~5.8M | yaml: ~82M, js-yaml: ~136M |
| GitHub stars | 4,381 | 695 | yaml: 1,604 |
| Last published | 5 years ago | 6 years ago | yaml 2.8.2: active (2025) |
| Dependencies | 4 (js-yaml@^3.13.1, kind-of, section-matter, strip-bom-string) | 1 (js-yaml@^3.13.1) | 0 (yaml pkg is self-contained) |
| TypeScript types | Included (gray-matter.d.ts) | Included (index.d.ts) | Included natively (yaml 2.x) |
| Module format | CJS only (no ESM exports) | CJS only (no ESM exports) | ESM native (yaml 2.x) |
| YAML spec | YAML 1.1 via js-yaml 3.x | YAML 1.1 via js-yaml 3.x | YAML 1.2 (yaml 2.x) |
| Multi-format support | YAML, JSON, TOML, Coffee, custom | YAML only | YAML only (unless you add more) |
| Bun compatible | Yes (CJS, Bun handles interop) | Yes (CJS, Bun handles interop) | Yes (ESM native) |
| Unpacked size | 11.5 kB | 11.3 kB | yaml: ~300 kB (but tree-shakeable) |
| License | MIT | MIT | ISC |

### 4.2 Security Analysis: js-yaml Dependency

Both gray-matter and front-matter pin js-yaml@^3.13.1. This version range has known issues:

| Issue | Affected Versions | Status |
|-------|-------------------|--------|
| Code injection via load() function | js-yaml < 3.13.1 | Fixed in 3.13.1 (gray-matter uses safeLoad) |
| Prototype pollution in merge (<<) | js-yaml < 4.1.1 | NOT fixed in 3.x branch |
| CVE-2025-64718 | js-yaml < 4.1.0 | NOT fixed in 3.x branch |

gray-matter has an open PR (#137) to update to js-yaml v4 that has NOT been merged. The maintainer (jonschlinkert) has not been active. This means both gray-matter and front-matter ship with a js-yaml version that has a known prototype pollution vulnerability.

The `yaml` npm package (eemeli/yaml v2.x) does NOT have these vulnerabilities. It implements YAML 1.2, is ESM-native, and includes TypeScript types.

### 4.3 Markdown Library Comparison

| Metric | micromark 4.x | remark (unified) | markdown-it 14.x | marked 17.x |
|--------|--------------|-------------------|-------------------|-------------|
| Weekly downloads | ~7M (via remark) | remark-parse: ~19.7M | ~15.7M | ~26.6M |
| GitHub stars | ~1,800 | remark: 8,732 | 21,030 | 36,579 |
| Last published | 2 years ago | 2 years ago | 17 hours ago (active) | 4 hours ago (active) |
| Dependencies | 17 (micromark-util-*) | Many (unified + micromark + mdast) | 4 (entities, linkify-it, mdurl, punycode.js) | 0 |
| TypeScript | Yes (ESM, included) | Yes (ESM, included) | Via @types/markdown-it | Built-in |
| Module format | ESM only | ESM only | CJS + ESM | CJS + ESM |
| Bundle (gzipped) | ~14 kB | ~30+ kB (parse+stringify+unified) | ~33 kB | ~12-14 kB |
| CommonMark compliant | Yes (100%) | Yes (via micromark) | Yes (essentially) | No |
| Security | Safe by default | Safe by default | Safe by default | NOT safe by default |
| Plugin ecosystem | Extensions (syntax-level) | Large (200+ plugins) | Large (plugins) | Growing (extensions) |
| Primary use case | Low-level parsing, markdown-to-HTML | AST transformations, content pipelines | Markdown-to-HTML with plugins | Fast markdown-to-HTML |
| Bun compatible | Yes (ESM) | Yes (ESM) | Yes | Yes |

### 4.4 Does This Project Need Full Markdown Processing?

Analysis of how markdown content is used in `@acmelabs-15/agent-plugin`:

| Use Case | What's Needed | Full Parser Required? |
|----------|---------------|----------------------|
| Parse SKILL.md frontmatter | Extract YAML between `---` delimiters | No -- frontmatter library or manual split |
| Parse AGENT.md frontmatter | Extract YAML between `---` delimiters | No -- same as above |
| Read markdown body content | Pass raw markdown text to platform adapters | No -- string after frontmatter extraction |
| Generate CLI help text | Display plugin/skill descriptions | No -- plain text from frontmatter fields |
| Render documentation | Possible future feature | Maybe -- but not in MVP |
| Extract section headings | Possible for structured skill parsing | Maybe -- regex or simple heading extraction sufficient |
| Validate markdown structure | Possible future linting | Maybe -- but separate concern from core |

**Verdict**: The project does NOT need a full markdown parser for its core functionality. The primary operation is frontmatter extraction. The markdown body is passed through as-is to platform adapters (Claude Code, Cursor, etc.) which handle their own markdown rendering. A full markdown AST parser would be over-engineering.

If section extraction becomes necessary later (e.g., extracting "When to Use" sections from SKILL.md), simple regex or line-by-line parsing of headings is sufficient. A full CommonMark parser is not justified.

## 5. Results

### Evidence Table

| Finding | Source | Confidence |
|---------|--------|------------|
| gray-matter pins js-yaml@^3.13.1 with known prototype pollution CVE | GitHub package.json, CVE databases | High |
| front-matter also pins js-yaml@^3.13.1 with same vulnerability | GitHub package.json | High |
| gray-matter has an unmerged PR (#137) for js-yaml v4, maintainer inactive | GitHub PR #137, issue #136 | High |
| yaml 2.x (eemeli) is ESM-native, YAML 1.2, no known vulnerabilities | npm registry, GitHub | High |
| Vercel Skills uses gray-matter for SKILL.md parsing | ANALYSIS-003 source code analysis | High |
| Bun handles CJS interop transparently for both gray-matter and front-matter | Bun documentation | High |
| marked is NOT safe by default for HTML output | macwright.com analysis, documentation | High |
| micromark is ~14 kB gzipped, smallest CommonMark parser | micromark README, GitHub | High |
| This project does not need full markdown AST processing for core functionality | ADR-001, ANALYSIS-003, use case analysis | High |
| Manual frontmatter parsing is 5-10 lines of code | Implementation complexity assessment | Medium |

### Facts (Verified)

- gray-matter 4.0.3 has 4 dependencies, CJS only, TypeScript types included, last published 5 years ago
- front-matter 4.0.2 has 1 dependency (js-yaml), CJS only, TypeScript types included, last published 6 years ago
- Both gray-matter and front-matter depend on js-yaml@^3.x which has CVE-2025-64718 (prototype pollution)
- yaml 2.8.2 (eemeli/yaml) is ESM-native, zero external dependencies, TypeScript types built-in, actively maintained
- Vercel Skills (the reference implementation) uses gray-matter for SKILL.md frontmatter parsing
- marked 17.x has zero dependencies but is not safe by default for HTML rendering
- micromark is the smallest 100% CommonMark compliant parser at ~14 kB gzipped
- The project's core use case is frontmatter extraction, not markdown-to-HTML conversion

### Hypotheses (Unverified)

- Manual frontmatter parsing with yaml 2.x may have better long-term maintainability than depending on unmaintained gray-matter
- Section heading extraction (if needed later) can be done with regex without a full parser
- The yaml package tree-shakes well in Bun's bundler despite its 300 kB unpacked size

## 6. Discussion

### The gray-matter Problem

gray-matter is the most popular and feature-rich frontmatter parser. It supports multiple formats (YAML, JSON, TOML, Coffee) and is used by Astro, Gatsby, VitePress, and Vercel Skills. However, it has three problems for this project:

1. **Security**: Pins js-yaml@^3.13.1. The js-yaml 3.x branch has a known prototype pollution vulnerability (CVE-2025-64718). An open PR to update to js-yaml v4 has been pending since at least 2022 and the maintainer appears inactive.

2. **Module format**: CJS only with no ESM exports field. Bun handles CJS interop, but ESM-native is cleaner for a new project.

3. **Maintenance**: Last published 5 years ago. 4 dependencies that add complexity for limited benefit (kind-of, section-matter, strip-bom-string are not needed for simple YAML frontmatter).

### The front-matter Alternative

front-matter is lighter (1 dependency vs 4) but has the same js-yaml@^3.x vulnerability. It also has not been published in 6 years. Choosing it over gray-matter saves 3 transitive dependencies but does not solve the security or maintenance problems.

### The Manual Parsing Option

Frontmatter parsing is simple. The algorithm is:

1. Check if string starts with `---\n`
2. Find the next `---\n`
3. Parse the content between delimiters as YAML
4. Return parsed data + remaining content

This is 5-10 lines of code. Using `yaml` 2.x (eemeli/yaml) as the YAML engine provides:
- YAML 1.2 compliance (vs 1.1 for js-yaml 3.x)
- ESM native (no CJS interop needed)
- TypeScript types built in
- Zero external dependencies
- Active maintenance (82M weekly downloads)
- No known CVEs

The trade-off is owning the frontmatter splitting logic. However, this logic is trivial and well-understood. The edge cases (empty frontmatter, no frontmatter, BOM handling) are straightforward.

### Markdown Processing: Not Needed Now

The project extracts frontmatter from SKILL.md/AGENT.md files and passes the markdown body as-is to platform adapters. There is no markdown-to-HTML rendering in the core tool. There is no AST transformation needed. The CLI help text uses frontmatter fields (name, description), not parsed markdown.

Adding micromark, remark, markdown-it, or marked would add 14-33 kB of gzipped dependencies for functionality the project does not use. If markdown processing becomes necessary later (documentation generation, structured section extraction), it can be added as a separate concern.

### Why Not micromark Even as a Future Hedge?

micromark is excellent (smallest CommonMark parser, safe, extensible). But adding it now would be speculative dependency accumulation. The project should adopt a markdown parser only when a concrete use case requires it. At that point, micromark (for simple HTML output) or remark (for AST transformations) would be the right choices. marked is disqualified due to being unsafe by default.

## 7. Recommendations

| Priority | Recommendation | Rationale | Effort |
|----------|---------------|-----------|--------|
| P0 | Use manual frontmatter parsing with `yaml` 2.x (eemeli/yaml) | Eliminates js-yaml 3.x CVE, ESM-native, TypeScript built-in, zero transitive deps, actively maintained | Low (5-10 lines) |
| P0 | Do NOT add gray-matter | Unmaintained (5 years), js-yaml CVE, CJS-only, 4 unnecessary dependencies | None |
| P0 | Do NOT add any markdown processing library in MVP | No use case requires markdown-to-HTML or AST in the core tool | None |
| P1 | Create a `parseFrontmatter()` utility function with Zod validation | Combine YAML parsing with Zod v4 schema validation in one function | Low |
| P2 | If markdown processing is needed later, evaluate micromark (HTML output) or remark (AST) | micromark: 14 kB, CommonMark, safe. remark: plugin ecosystem. Do NOT use marked (unsafe by default). | Future |

## 8. Conclusion

**Verdict**: Proceed with manual frontmatter parsing using `yaml` 2.x
**Confidence**: High
**Rationale**: gray-matter and front-matter both depend on a vulnerable js-yaml 3.x. Manual parsing is trivial (5-10 lines). The `yaml` package is ESM-native, TypeScript-native, YAML 1.2 compliant, and actively maintained with 82M weekly downloads. No markdown processing library is needed for the project's core functionality.

### User Impact

- **What changes for you**: One dependency (`yaml`) instead of gray-matter's 5 (gray-matter + 4 transitive). No known CVEs. ESM-native. TypeScript types included.
- **Effort required**: Low. Write a ~10-line `parseFrontmatter()` utility. Validate output with Zod schema.
- **Risk if ignored**: Using gray-matter introduces a known prototype pollution vulnerability (CVE-2025-64718) via js-yaml 3.x, an unmaintained dependency chain, and CJS interop overhead.

## 9. Appendices

### Frontmatter Parsing Reference Implementation

```typescript
import { parse as parseYaml } from 'yaml';

interface ParsedFrontmatter<T = Record<string, unknown>> {
  data: T;
  content: string;
}

function parseFrontmatter<T = Record<string, unknown>>(input: string): ParsedFrontmatter<T> {
  const trimmed = input.replace(/^\uFEFF/, ''); // Strip BOM
  if (!trimmed.startsWith('---\n') && !trimmed.startsWith('---\r\n')) {
    return { data: {} as T, content: trimmed };
  }
  const end = trimmed.indexOf('\n---', 3);
  if (end === -1) {
    return { data: {} as T, content: trimmed };
  }
  const yamlStr = trimmed.slice(4, end);
  const content = trimmed.slice(end + 4).replace(/^\r?\n/, '');
  const data = parseYaml(yamlStr) as T;
  return { data: data ?? {} as T, content };
}
```

### Sources Consulted

- gray-matter GitHub: https://github.com/jonschlinkert/gray-matter
- gray-matter package.json: https://raw.githubusercontent.com/jonschlinkert/gray-matter/master/package.json
- gray-matter js-yaml update PR #137: https://github.com/jonschlinkert/gray-matter/pull/137
- gray-matter js-yaml update issue #136: https://github.com/jonschlinkert/gray-matter/issues/136
- front-matter GitHub: https://github.com/jxson/front-matter
- front-matter package.json: https://raw.githubusercontent.com/jxson/front-matter/master/package.json
- yaml (eemeli/yaml) docs: https://eemeli.org/yaml/
- yaml npm: https://www.npmjs.com/package/yaml
- js-yaml CVE: https://security.snyk.io/package/npm/js-yaml
- micromark GitHub: https://github.com/micromark/micromark
- remark GitHub: https://github.com/remarkjs/remark
- marked npm: https://www.npmjs.com/package/marked
- markdown-it GitHub: https://github.com/markdown-it/markdown-it
- "Don't use marked" article: https://macwright.com/2024/01/28/dont-use-marked
- npm-compare frontmatter: https://npm-compare.com/front-matter,gray-matter,yaml-front-matter
- npm-compare markdown: https://npm-compare.com/markdown-it,marked,remark,remark-parse,unified
- Bun CJS interop: https://bun.sh/blog/commonjs-is-not-going-away
- Vercel Skills source analysis: ANALYSIS-003

### Data Transparency

- **Found**: Package versions, dependency trees, CVE data, download counts, GitHub stars, module formats, TypeScript support, Bun compatibility, Vercel Skills reference implementation details, bundle size approximations.
- **Not Found**: Exact bundlephobia minified+gzipped sizes (site uses client-side rendering, blocked by WebFetch). Exact tree-shaken size of yaml 2.x in Bun's bundler. Whether gray-matter maintainer plans to merge js-yaml v4 PR.

## Observations

- [decision] Manual frontmatter parsing with yaml 2.x (eemeli/yaml) recommended over gray-matter or front-matter #frontmatter #dependencies
- [decision] No markdown processing library needed for MVP -- core use case is frontmatter extraction, not HTML rendering #markdown #scope
- [fact] gray-matter 4.0.3 pins js-yaml@^3.13.1 which has CVE-2025-64718 (prototype pollution), unmaintained for 5 years #security #risk
- [fact] front-matter 4.0.2 has the same js-yaml@^3.x vulnerability, unmaintained for 6 years #security
- [fact] yaml 2.8.2 (eemeli/yaml) is ESM-native, TypeScript-native, YAML 1.2, zero external deps, 82M weekly downloads #dependency-choice
- [fact] Vercel Skills reference implementation uses gray-matter for SKILL.md parsing (ANALYSIS-003) #reference
- [fact] Manual frontmatter parsing is 5-10 lines of code: split on --- delimiters, parse YAML, return data + content #implementation
- [risk] marked (17.x) is not safe by default for HTML rendering -- disqualified if markdown-to-HTML is needed later #security
- [insight] micromark (~14 kB gzipped) or remark (plugin ecosystem) are the correct future choices if markdown processing becomes necessary #future
- [constraint] Both gray-matter and front-matter are CJS-only; Bun handles interop but ESM-native is preferred for new projects #module-format

## Relations

- relates_to [[ADR-001 Plugin Format and Manifest]]
- relates_to [[ANALYSIS-003 Vercel Skills Format Deep Dive]]
- relates_to [[ANALYSIS-016 Bun Runtime Assessment]]
- relates_to [[SESSION-2026-03-07_01-agent-plugin-spec-ideation]]